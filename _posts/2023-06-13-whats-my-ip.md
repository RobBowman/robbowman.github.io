# Figuring Out the IP Address of Your Azure Function and API Management Policy
## Background
I recently created an Azure Function app that's triggered by an Http request. It then calls out to a 3rd party api but before doing so, it must first obtain a token from their token server. For the token acquisition and management, I created an apim policy, that uses *send-request*. 

Access to the 3rd party's token and api servers is restricted by calling ip address. So, I had to provide the ip address of both my api management instance and the azure function app.

## Config
The api management instance was deployed in internal mode with vnet integration enabled. The azure function also had vnet integration enabled. The subnet of the azure function app was configured to use a nat gateway and static public ip address.

During the 1st stages of testing, I was receiving *403 Forbidden* responses from the 3rd party. This led to some head scratching and the realisation that I need a way to be sure of the ip address being used.

## Different IP Address When Calling Another Azure Service
The 3rd party api was also hosted on Azure. This turned out to be significant because the subnet of the function app had the *System.Web* Service Endpoint enabled. This meant that when the function app called the 3rd party api, it was using the ip address of the subnet, not the static public ip address. After disabling the *System.Web* Service Endpoint, the function app used the static public ip address of the nat gateway and requests wer successful.

## Checking the IP Address from the Azure Function
I created the following Azure Function that makes two calls and reflects the ip address used in the response.

```csharp
[Function("GetIp")]
public static async Task<HttpResponseData> GetIp(
    [HttpTrigger(AuthorizationLevel.Function, "get")] HttpRequestData req)
{
    HttpClient httpClient = new();
    var response = await httpClient.GetAsync("https://httpbin.org/ip");
    var responseBody = "When calling httpbin.org" + await response.Content.ReadAsStringAsync();

    response = await httpClient.GetAsync("https://mysandpitfunction.azurewebsites.net/api/ReflectIp");
    responseBody = responseBody + "\n"  + "When calling azure func" + await response.Content.ReadAsStringAsync();

    response.Content.Headers.TryAddWithoutValidation("Content-Type", "application/json");

    var httpResponse = req.CreateResponse();
    await httpResponse.WriteStringAsync(responseBody);
    return httpResponse;
}
```

The code for the *ReflectIp* function is as follows:

```csharp
public static class ReflectIp
{
    [FunctionName("ReflectIp")]
    public static async Task<IActionResult> Run(
        [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = null)] HttpRequest req,
        ILogger log)
    {
        log.LogInformation("C# HTTP trigger function processed a request.");

        string responseMessage = req.HttpContext.Connection.RemoteIpAddress?.ToString() ?? "unknown";

        return new OkObjectResult(responseMessage);
    }
}
```

## Checking the IP Address from the API Management Policy
I created the following policy to check the ip address used when calling the 3rd party api.

```xml
<inbound>
    <base />
    <send-request mode="new" response-variable-name="ipresponse" timeout="20" ignore-error="true">
        <set-url>https://httpbin.org/ip</set-url>
        <set-method>GET</set-method>
    </send-request>
    <return-response>
        <set-status code="200" reason="OK" />
        <set-body>@(((IResponse)context.Variables["ipresponse"]).Body.As<string>())</set-body>
    </return-response>
</inbound>
```

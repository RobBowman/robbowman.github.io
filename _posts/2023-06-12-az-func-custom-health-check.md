---
title: Dotnet-Isolated Azure Function Custom Health Check
category: Azure
tags:
    - Azure
    - Azure Functions
---
# Adding Custom Health Check to Dotnet-Isolated Azure Functions
Azure function apps give the options to enable health checks. An overview and description of benefits of these can be found [here](https://learn.microsoft.com/en-us/azure/app-service/monitor-instances-health-check?tabs=dotnet)

## Required Changes
In my case, I wanted to add more that a simple check that the function would respond to a http get. I wanted it to also check that the function app was able to obtain an auth token from an api on which it depends.

This required the following three additions to my function app code (ordered from inside out!):

1. A class that implemented the IHealthCheck interface that would make a test call to the dependent api

2. A azure function with an http trigger that would call the custom health check class from point 1

3. A change to the HostBuilder config found in Main of Program.cs to add the custom health check

## The Custom Health Check Class - 1

````
public class AmxTokenServiceHealthCheck : IHealthCheck
{
    private readonly IAccessTokenClient _accessTokenClient;

    public AmxTokenServiceHealthCheck(IAccessTokenClient accessTokenClient)
    {
        _accessTokenClient = accessTokenClient;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        string correlationId = System.Guid.NewGuid().ToString();
        try
        {
            string accessToken = await _accessTokenClient.GetAccessToken(correlationId);
            return HealthCheckResult.Healthy();
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy(ex.Message);
        }
    }
}
````

## The Azure Function - 2


````
public class HealthFunctions
{
    private readonly HealthCheckService _healthService;

    public HealthFunctions(HealthCheckService healthService)
    {
        _healthService = healthService ?? throw new System.ArgumentNullException(nameof(healthService));
    }

    [Function(nameof(HealthCheck))]
    public async Task<IActionResult> HealthCheck(
        [HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "health")]
        HttpRequest req)
    {
        var healthResult = await _healthService.CheckHealthAsync();
        var status = (healthResult.Status == HealthStatus.Healthy) ? 200 : 500;

        return new JsonResult(new
        {
            status = healthResult.Status.ToString(),
            entries = healthResult.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString(),
                e.Value.Description,
                e.Value.Exception
            })
        })
        {
            StatusCode = status
        };
    }
}
````

## The HostBuilder Config - 3
````
var host = new HostBuilder()
            .ConfigureFunctionsWorkerDefaults(worker => worker.
            .ConfigureServices(s =>
            {
                s.AddHealthChecks()
                    .AddCheck<AmxTokenServiceHealthCheck>("AmxTokenServiceHealthCheck");
            })
            .Build();
````

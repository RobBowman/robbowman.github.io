---
title: Publishing Multiple Messages tp Service Bus with Managed Identity
category: Azure
tags:
    - Azure
    - Service Bus
---
# Publishing Multiple Messages to Service Bus with Managed Identity
I ran into a problem with the service bus binding for Azure functions this week.

I have a dotnet 6.0, dotnet-isolated Function App that used the Azure Service Bus outbound binding to publish a message to a queue. This was working fine, but then the requirements changed. Instead of publishing a single message resulting from an api call, it needed to publish an individual message for each record in the api results.

## Original Code
The function was triggered by a http request and published a single message (as well as returning a happy message to the http client) as shown below:

````cs 
[Function("TriggerReport")]
[ServiceBusOutput("%SbTopicName%", Connection = "ServiceBusConnection")]
public async Task <string> TriggerReport(
    [HttpTrigger(AuthorizationLevel.Function, "get")] HttpRequestData req,
    FunctionContext context) 
    {
        _methodName = nameof(TriggerReport);
        LogInfo("Started");
        
        DateTime now = DateTime.Now;
        DateTime lastPoll = await _reportSvc.GetTimeOfLastPoll(_correlationId);

        List<AmxGetReportResponse> reportRecs = await _reportSvc.GetReport(lastPoll, _correlationId);
        string msg = _amxReport2CatsUpdate.Execute(reportRecs);
        LogInfo($"Mapped msg={msg}");

        await _reportSvc.SetTimeOfLastPoll(now, _correlationId);
        var httpResponse = req.CreateResponse();

        await httpResponse.WriteStringAsync("Report triggered");

        LogInfo("Leaving");
        return msg;
    }
````

## No Passwords / Keys
One nice feature of the original code is no password or keys are required to publish to the Service Bus. Authentication is via Azure Managed Identity, with the appropriate role for the identity of the Function App applied to the Service Bus Instance.

The value given in the "Connection" property of the "ServiceBusOutput" binding is important. The environment variables of the function app will be checked to find the required value. In my case, I ensured the following app configuration value was available:

> ServiceBusConnection__fullyQualifiedNamespace: 'mynamespace.servicebus.windows.net'

## New Code
It seems there's no way to convince the binding to publish an array of messages, so I had to employ the Service Bus SDK. 

This can be found in the nuget package "Azure.Messaging.ServiceBus", I used version "7.14.0"

## Outer Function
The azure function was stripped down, no use of output binding:

````cs
[Function("TriggerReport")]
public async Task <string> TriggerReport(
    [HttpTrigger(AuthorizationLevel.Function, "get")] HttpRequestData req,
    FunctionContext context) 
        {
            _methodName = nameof(TriggerReport);
            LogInfo("Started");
            
            await _reportSvc.PublishReport(_correlationId);   
             
            LogInfo("Leaving");
            return "Report published";
        }
````

## Publishing Multiple Messages

````cs
public async Task PublishReport(string correlationId)
    {
        _methodName = nameof(PublishReport);
        _correlationId = correlationId;
        LogInfo("Started");

        DateTime now = DateTime.Now;
        DateTime lastPoll = await GetTimeOfLastPoll(_correlationId);

        List<AmxGetReportResponse> reportRecs = await GetReport(lastPoll, _correlationId);
        var credential = new ChainedTokenCredential(
                                new ManagedIdentityCredential(),
                                new VisualStudioCredential(),
                                new AzureCliCredential()
                                );
        LogInfo($"Now logged into azure");
        var serviceBusClient = new ServiceBusClient($"{_amxOptions.ServiceBusNamespace}.servicebus.windows.net", credential);
        LogInfo($"Have service bus client");
        var topicSendClient = serviceBusClient.CreateSender(_amxOptions.ServiceBusTopicName);
        LogInfo($"Have topic sender");

        foreach (var reportRec in reportRecs)
        {
            string msg = _amxReport2CatsUpdate.Execute(reportRec);
            LogInfo($"Mapped msg={msg}");
            var serviceBusMsg = new ServiceBusMessage(msg);
            LogInfo($"Have service bus message");
            await topicSendClient.SendMessageAsync(serviceBusMsg);
            LogInfo($"Sent msg to sb topic {_amxOptions.ServiceBusTopicName}: {msg}");
        }

        await SetTimeOfLastPoll(now, _correlationId);
        LogInfo("Leaving");
    }
````

I learned about the ChainedTokenCredential from Toon at [this post](https://yourazurecoach.com/2020/08/13/managed-identity-simplified-with-the-new-azure-net-sdks/)

I was using vscode (on a mac) for my development. It seems there's currently an issue with the visual studio code credential but that wasn't a major problem. I just needed to ensure I'd run "az login" before execution of the integration test.
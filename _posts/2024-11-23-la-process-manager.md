---
title: Implementing the Process Manager Pattern with Logic Apps Standard
category: Azure
tags:
    - Logic Apps
    - Azure
---

## Overview
The [Process Manager](https://www.enterpriseintegrationpatterns.com/patterns/messaging/ProcessManager.html) is an Enterprise Integration Pattern to maintain the state of sequence and determine the next processing step based on intermediate results.

![simple](/images/la-process-manager/simple.gif)

The simple diagram above taken from the excellent [Enterprise Integration Patterns Website](https://www.enterpriseintegrationpatterns.com/) shows the process manager coordinating calls to A, B and C in sequence. However, these could just as well be A, C, B or A, C, A etc.

## BizTalk Migration
I'd bumped into this requirement as part of work to migrate an existing suite of BizTalk soltutions that made extensive use of the Process Manager pattern.

In the BizTalk solution, the key logic of the Process Manager is implemented as an Orchestration named Gatekeeper, as shown below:

![gatekeeper](/images/la-process-manager/gatekeeper.png)

Upon receipt of the Trigger Message, the orchestration published to a logical request response port. This is consumed by a worker (Proc A, B, C etc) which then responds with a result message. The Gatekeeper will loop in this process until it decides no further work is required.

The most significant change when converting this capability to Logic Apps Standard was in relation to the publish / subscribe message broker and correlation of the response message. For BizTalk orchestrations, we can be sure that the same instance of the orchestation will consume the response from the logical port as published the request. In the world of Logic Apps Standard and Azure Service Bus, a little more legwork is needed.

## Logic Apps Standard Implementation
The following diagram illustrates how the Process Manager was implemented:

![outline-flow](/images/la-process-manager/outline-flow.png)

+ On receipt of the trigger message, the requested operation is parsed from the json payload.
+ A new session id is created
+ Loop starts
+ The request message is published to a service bus topic
+ The process manager workflow must check for a response with the same session id. This is not currently possible using worflow actions, so I had to make use of a local C# function
+ The response message is passed to a rules engine to determine next operation
+ If next operation is same as previous then there's no more steps to execute

The key actions of the Process Manager Logic App Standard workflow can be seen below:

![workflow](/images/la-process-manager/procman-la.png)

# The Local Function
I mentioned in the previous section, it's not currently possible to use a logic app standard workflow action to receive an in-session message from service bus queue. It seems that it can be done, but only if the workflow was triggered by a session aware service bus trigger, see [this post from Microsoft](https://techcommunity.microsoft.com/blog/integrationsonazureblog/session-support-for-service-bus-built-in-connector-logic-apps-standard/4034074/replies/4219967#M1214). In my case, it must be possible to trigger the Process Manager workflow via http because its clients include webforms that are awaiting a response.

So, a work-around had to be found. Fortunately, Microsoft had recently introduced local functions to Logic Apps Standard, and I was able to easily create one that could consume the response and make available to the workflow. The local functions have the benefit of running in the same process as workflow, without the extra security and performance baggage of calling to an external Azure Function App.

The code for the local function can be seen below:

```cs
public class SvcBusQReader
    {
        private readonly ILogger<SvcBusQReader> _logger;
        private readonly IConfiguration _configuration;

        public SvcBusQReader(ILoggerFactory loggerFactory, IConfiguration configuration)
        {
            _logger = loggerFactory.CreateLogger<SvcBusQReader>();
            _configuration = configuration;
        }

        [FunctionName("ReadQueueSessionMessage")]
        public async Task<string> ReadQueueSessionMessage([WorkflowActionTrigger] string queueName, string sessionId)
        {
            const string FunctionName = nameof(ReadQueueSessionMessage);
            _logger.LogInformation("Starting function.", FunctionName, queueName, sessionId);

            var serviceBusConnectionString = GetServiceBusConnectionString();
            var receiver = await CreateServiceBusReceiverAsync(serviceBusConnectionString, queueName, sessionId);

            var messageBody = await ReceiveAndCompleteMessageAsync(receiver, sessionId);

            _logger.LogInformation("Leaving function.", FunctionName);
            return messageBody;
        }

        private string GetServiceBusConnectionString()
        {
            var connectionString = _configuration["ProcManServiceBusConnectionString"];
            if (string.IsNullOrEmpty(connectionString))
            {
                throw new InvalidOperationException("ProcManServiceBusConnectionString is not configured.");
            }
            return connectionString;
        }

        private async Task<ServiceBusSessionReceiver> CreateServiceBusReceiverAsync(string connectionString, string queueName, string sessionId)
        {
            var client = new ServiceBusClient(connectionString);
            var receiverOptions = new ServiceBusSessionReceiverOptions
            {
                ReceiveMode = ServiceBusReceiveMode.PeekLock
            };
            var receiver = await client.AcceptSessionAsync(queueName, sessionId, receiverOptions);
            if (receiver == null)
            {
                throw new InvalidOperationException($"No session found with ID: {sessionId}");
            }
            return receiver;
        }

        private async Task<string> ReceiveAndCompleteMessageAsync(ServiceBusSessionReceiver receiver, string sessionId)
        {
            const int MaxRetries = 5;
            const int DelayMilliseconds = 100;
            int retryCount = 0;

            while (retryCount < MaxRetries)
            {
                try
                {
                    var message = await receiver.ReceiveMessageAsync();
                    if (message == null)
                    {
                        throw new InvalidOperationException($"No message found in session with ID: {sessionId}");
                    }
                    string messageBody = message.Body.ToString();
                    await receiver.CompleteMessageAsync(message);
                    return messageBody;
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Failed to receive and complete message. Attempt {RetryCount} of {MaxRetries}", retryCount + 1, MaxRetries);
                    retryCount++;
                    if (retryCount >= MaxRetries)
                    {
                        throw new InvalidOperationException($"Failed to receive and complete message after {MaxRetries} attempts.", ex);
                    }
                    await Task.Delay(DelayMilliseconds);
                }
            }

            throw new InvalidOperationException("This code should not be reached.");
        }
    }
```
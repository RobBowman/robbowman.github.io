---
title: Tracing Calls to APIM from 2024
category: Azure
tags:
    - APIM
    - Azure
---

## Overview
Previously, my first step on a journey to troubleshoot Azure API Management would be to enable tracing on the target APIM subscription and setting the *ocp-apim-trace* header on the request.

This would result in a response header containing a url that linked to a page of data detailing the request's route through the APIM world - very useful!

Unfortunately, this must have been too useful to the bad guys because since November 2024, this capability has been disabled.

## Microsoft Guidance 
The official guidance on the new technique to be used for APIM tracing can be found [here](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-api-inspector). You can see that use of the Azure Portal is recommended, however, I have found this often to be inoperable. I navigate to the Test tab and I'm presented with an ever cycling load animation. Do others get the same?

You'll notice that further down the page, Microsoft also give guidance on how tracing can be enabled for external API requests, however this is now more complicated, and requires token acquisition. Fortunately, it can be automated to be almost as easy as before. 

## Automating with VS Code Extension
The excellent [REST Client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client) VS Code extension was a big help in streamlining requests to apim in a way that can be followed up with a request for trace details. A three step process is required:

1. Login to the Azure Management Api - opens a login window in a browser to request your credentials. Note: you need at least "Contributor" role at the target APIM instance
2. Call your APIM API
3. Request the Trace Log

You can find a script for this below, simply save with a .http extension, open in VS Code with the REST Client extension installed and you should be able to click links above the three POST requests.

```http
@subscriptionId = ab34e8cf-8877-4385-acaa-xxxxxxxxxxxx
@resourceGroup = rg-sandpit
@apimServiceName = apim-biztalkers-consumption
@apiName = calculator
@subscriptionKey = df5c936ae3734317bc3bf65f5c1020ed

### login to Azure - this causes a popup to appear in the browser

# @name TraceCreds
POST https://management.azure.com/subscriptions/{{subscriptionId}}/resourceGroups/{{resourceGroup}}/providers/Microsoft.ApiManagement/service/{{apimServiceName}}/gateways/managed/listDebugCredentials
?api-version=2023-05-01-preview
Content-Type: application/json
Authorization: {{$aadToken}}

{
    "credentialsExpireAfter": "PT1H",
    "apiId": "/subscriptions/{{subscriptionId}}/resourceGroups/{{resourceGroup}}/providers/Microsoft.ApiManagement/service/{{apimServiceName}}/apis/{{apiName}}",
    "purposes": ["tracing"]
}

### use the credentials obtained from previous step when calling your API

# @name ApiOperationTracing
POST https://apim-biztalkers-consumption.azure-api.net/calculator/addition
Content-Type: application/json
OCP-APIM-Subscription-Key: {{subscriptionKey}}
Apim-Debug-Authorization: {{TraceCreds.response.body.$.token}}

{
  "num1": "7",
  "num2": "6"
}

### trace logs have been saved and can be accessed from a url that was returned in the response headers of previous step

# @name ObtainedTraceLogs
POST https://management.azure.com/subscriptions/{{subscriptionId}}/resourceGroups/{{resourceGroup}}/providers/Microsoft.ApiManagement/service/{{apimServiceName}}/gateways/managed/listTrace
?api-version=2023-05-01-preview
Content-Type: application/json
Authorization: {{$aadToken}}

{
    "traceId": "{{ApiOperationTracing.response.headers.Apim-Trace-Id}}"
}
```


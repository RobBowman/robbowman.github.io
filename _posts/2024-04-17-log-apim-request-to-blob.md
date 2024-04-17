---
title: Log APIM Request to Blob
category: Azure
tags:
    - Azure
    - APIM
---

## Overview
When interacting with 3rd party APIs, it's common to hit an issue in which being able to check the payload of the posted message is critical.

If outbound calls are being made through one of your apim instances, this should be as easy as enabling application insights. 

## App Insights Fail
Although I've successfully configured apim to app insights logging in previous environments, this time I could not convince apim to log to the application insights instance I had set up. I suspect there may have been a network config problem such as a NSG rule blocking the traffic.

My next thought was to route the request to pipedream.com but this also failed - I think the allowable endpoints from the apim instance are restricted.

So, I decided to log the request to a blob storage account. 

First, I created a SAS key for the blob storage account. I then created a new apim policy to send a copy of the request to the blob storage account.

```xml
<policies>
    <inbound>
        <base />
        <set-variable name="personalAccessToken" value="{{pf-esign-api-personalAccessToken}}" />
        <set-header name="Authorization" exists-action="override">
            <value>@("Bearer " + context.Variables["personalAccessToken"])</value>
        </set-header>
        <set-header name="ocp-apim-subscription-key" exists-action="delete" />
        <send-request mode="new" response-variable-name="storageResponse" timeout="20" ignore-error="true">
            <set-url>https://sa-name.blob.core.windows.net/container-name/sample.json?sp=racwl&amp;st=2024-04-17T06:52:15Z&amp;se=2024-04-17T14:52:15Z&amp;spr=https&amp;sv=2022-11-02&amp;sr=c&amp;sig=signature-from-sas-key-here</set-url>
            <set-method>PUT</set-method>
            <set-header name="Content-Type" exists-action="override">
                <value>application/json</value>
            </set-header>
            <set-header name="x-ms-blob-type" exists-action="override">
                <value>BlockBlob</value>
            </set-header>
            <set-body>@(context.Request.Body.As<string>(preserveContent: true))</set-body>
        </send-request>
    </inbound>
    <backend>
        <forward-request follow-redirects="true" timeout="20" />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

In the \<backend> section, you can see that the request will still be forwarded to the original endpoint - configured as the backend in the apim instance. 

The \<send-request> section ensures that a copy of the request is sent to the blob storage account.

## Extra Benefit
An additional benefit of using blob storage to log the request payload is size. When logging with app insights, the payload is truncated at 8kb. This is not the case with blob storage.
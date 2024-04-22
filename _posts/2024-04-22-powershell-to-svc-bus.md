---
title: Publish message to service bus using PowerShell
category: Azure
tags:
    - Azure
    - PowerShell
    - Service Bus
---

## Overview
The Azure Portal include a service bus explorer tool. This can be used to publish a message to a queue or a topic. However, this is a little limmited. While testing I found it frustrating that I needed to paste in the message body each time, along with a couple of custom properties.

## Azure REST API
When automating Azure, my first port of call is Azure CLI. However it seems this does not yet support publishing of messages to service bus. So, I turned to the Azure REST API.

## Two Steps
There are two steps required:
1. obtain the auth header for the rest post
2. post the request

I have separated these into individual PowerShell scripts, which can be found at [this github repo](https://github.com/RobBowman/powershell-svc-bus). The entry point is "post-msg.ps1". You can see that this requires a sas token to be set. This can be found by navigating the Azure Portal to Service Bus Namespace/Shared access policies/RootManageSharedAccessKey (or create a new one)/Primary Key

The top level script calls the "generate-sas-token.ps1" script in order to generate the sas token that is used in the header of the post request to the azure api.

### post-msg.ps1
```powershell
# This is the entry point. It will call the generate-sas-token.ps1 script to generate a SAS token and then use it to send a message to an Azure Service Bus topic.

# Define your connection string, topic name and SAS token
$endpoint = "https://biztalkers.servicebus.windows.net/"
$topicName = "p4f"
$sasToken = & '.\generate-sas-token.ps1'
$authHeader = "SharedAccessSignature $sasToken"

# Read JSON content from the file
$jsonFilePath = "./msg.json"
$jsonContent = Get-Content $jsonFilePath -Raw | ConvertFrom-Json

# Replace eventid and correlationid with new GUIDs
$jsonContent.eventid = [guid]::NewGuid().ToString()
$jsonContent.correlationid = [guid]::NewGuid().ToString()

# Convert the updated object back to JSON
$jsonContent = $jsonContent | ConvertTo-Json -Depth 100

# Output the JSON content
$jsonContent

# Prepare the custom properties
$properties = @{
    "eventtype" = "CommsEnrichmentCommand"
    "sourcesystem" = "Workflow"
}

# Convert properties to JSON
$propertiesJson = $properties | ConvertTo-Json -Compress

# Prepare the headers
$headers = @{
    "Authorization" = $authHeader
    "Content-Type" = "application/json"
    "BrokerProperties" = $propertiesJson
}

# Send the message using Invoke-RestMethod and store the response
$response = Invoke-RestMethod -Uri "$endpoint$topicName/messages" -Method Post -Body $jsonContent -Headers $headers

# Output the response
$response
```

### generate-sas-token.ps1
```powershell
# Note: this script is called by post-msg.ps1
[CmdletBinding()]
param (
    [Parameter()]
    $Namespace = "biztalkers",
    [Parameter()]
    $EntityPath = "p4f",
    [Parameter()]
    $SasKeyName = "RootManageSharedAccessKey",
    [Parameter()]
    $SasKey = "get-sas-key-from-azure-portal"

)

# Define the token's time-to-live in seconds
$ttl = New-TimeSpan -Days 30

# Get the target URI
$targetUri = [System.Web.HttpUtility]::UrlEncode("https://$Namespace.servicebus.windows.net/$EntityPath")

# Get the expiry time
$expiry = [DateTimeOffset]::Now.ToUnixTimeSeconds() + $ttl.TotalSeconds

# Generate the string to sign
$stringToSign = $targetUri + "`n" + $expiry

# Generate the signature
$hmac = New-Object System.Security.Cryptography.HMACSHA256
$hmac.Key = [Text.Encoding]::ASCII.GetBytes($SasKey)
$signature = $hmac.ComputeHash([Text.Encoding]::ASCII.GetBytes($stringToSign))
$signature = [Convert]::ToBase64String($signature)

# Generate the SAS token
$encodedSignature = [System.Web.HttpUtility]::UrlEncode($signature)
$sasToken = "sr=$targetUri&sig=$encodedSignature&se=$expiry&skn=$SasKeyName"

# Output the SAS token
$sasToken
```

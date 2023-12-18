---
title: Updating Resources Outside Target Resource Group
category: Azure
tags:
    - Azure
    - Bicep
---
# Updating Resources Outside Target Resource Group
The way this is achieved is a little strange and although I've implemented several times, I always forget!

## Resource Group Deployment
I have a devops pipeline that runs bicep to create Azure resources. The pipeline runs the following command in an AzurePowerShell task:

> az deployment group create

This command takes a parameter of --resource-group

In my case, I was deploying a function app. The complication was that I also needed to add a role assignment for the managed identity of the function app to an existing storage account - but the storage account resides in a different resource group!

## Module Scope to the Rescue
The trick is to use a dedicated bicep module for the resource to be updated that resides outside of resource group that's targeted by the --resource-group parameter mentioned above.

In my case I created a module named roleAssignment.bicep as shown below:

````json
param environment string = 'dev'
param funcAppPrincipalId string

var storageAcName = 'storcats${environment}001'
var storageBlobDataReaderRole = subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '2a2b9908-6ea1-4ae2-8e65-a410df84e7d1')

resource storageAccount 'Microsoft.Storage/storageAccounts@2021-04-01' existing = {
  name: storageAcName
}

resource roleAssignment 'Microsoft.Authorization/roleAssignments@2020-08-01-preview' = {
  name: guid(storageAccount.id, funcAppPrincipalId, storageBlobDataReaderRole)
  scope: storageAccount
  properties: {
    principalId: funcAppPrincipalId
    roleDefinitionId: storageBlobDataReaderRole
  }
}
````

## Calling the Module from the Main Bicep
The dedicated module can then be called from the main bicep as follows:

````json
module roleAsForCats 'roleAssignment.bicep' = {
  name: 'roleAsForCats'
  scope: resourceGroup('rg-cats-${environment}-001')
  params: {
    funcAppPrincipalId: functionApp.outputs.appServicePrincipalId 
    environment: environment
  }
}
````
The magic line here the one starting with **"scope"**. This causes the whole module to run against the specified resource group

## Running the Module in the Scope of Another Subscription
In addition to targeting a different resource group, the module can also be run in the scope of another subscription. This is achieved by passing subscriptionId as the first parameter to the resourceGroup function as shown below:

```
var registrySubscriptionId =  'e90a0a8a-XXX-XXX-XXX-07a5246XXXXX'
var registryResourceGroupName = 'rg-bicepregistry-prod-001'
module roleAssignment 'rg-acr-role-assignment.bicep' = {
    name: 'roleAssignment'
    scope: resourceGroup(registrySubscriptionId, registryResourceGroupName)
    params: {
        containerAppPrincipalId: containerIdentity.properties.principalId
    }
}
```
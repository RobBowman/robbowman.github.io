# Missing Host Keys in Azure Function App
I ran into a problem with deployment of a function app a couple of weeks ago. I thought I'd share the solution here in case anyone else runs into the same problem.

## The Process
The function app is deployed with a bicep template that's executed from an Azure DevOps pipeline. 

## Symptoms
Following creation of the function app, and before any code is deployed into it, the host keys should be visible from the Azure Portal. However, no keys were visible. Also, the the overview tab of the function app in the portal showed the following error: "Unable to reach host runtime"

## The Cause
It seems that there was some kind of failure during the execution of the bicep template that creates the azure resources.

## The Work-Around
Although we've not yet been able to identify root cause, Microsoft Support did suggest a work-around that was effective.

+ restart the function app
+ navigate to the storage account for the function app
+ delete the container named "azure-webjobs-secrets"
+ re-run the bicep deployment template

## Using Managed Identity for the Function App to the Storage Account
It seems that Microsoft are currently recommending using managed identity for the function app to storage account auth. I've found this to be difficult to get right. It seems that the implementation is a little half-baked at present. For example, although not documented anywhere, user assign managed identities don't work - the deployment will complete without error but no function host keys available.

## Problem with WEBSITE_RUN_FROM_PACKAGE
During my investigations, I found that if the function app has the app setting  WEBSITE_RUN_FROM_PACKAGE set to 1 then the host keys will not be available until the function app's code has been deployed.

To test my theory, I ran the following:

+ used the bicep template to create the function app with the WEBSITE_RUN_FROM_PACKAGE set to 1
+ observed that the host keys were not available
+ deployed the function app's code from the cli with *func azure functionapp publish fn-appkeysdemo*
+ observed that the host keys were now available :)


## Working Example
I've pasted a simple example of bicep that does work below:

```javascript
@description('Specifies region of all resources.')
param location string = resourceGroup().location

@description('Storage account SKU name.')
param storageSku string = 'Standard_LRS'

var baseName = 'appkeysdemo'
var functionAppName = 'fn-${baseName}'
var appServicePlanName = 'FunctionPlan'
var storageAccountName = 'fnstor${replace(baseName, '-', '')}'

resource storageAccount 'Microsoft.Storage/storageAccounts@2021-04-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: storageSku
  }
  kind: 'StorageV2'
  properties: {
    supportsHttpsTrafficOnly: true
    encryption: {
      services: {
        file: {
          keyType: 'Account'
          enabled: true
        }
        blob: {
          keyType: 'Account'
          enabled: true
        }
      }
      keySource: 'Microsoft.Storage'
    }
    accessTier: 'Hot'
  }
}

// Role assignments
var blobOwnerRoleId = 'b7e6dc6d-f1e8-4753-8033-0f276bb0955b'
resource roleDefinitionBlobContributor 'Microsoft.Authorization/roleDefinitions@2018-01-01-preview' existing = {
  scope: resourceGroup()
  name: blobOwnerRoleId
}

resource roleAssignmentBlob 'Microsoft.Authorization/roleAssignments@2020-04-01-preview' = {
  name: guid(functionAppName, blobOwnerRoleId)
  scope: storageAccount
  properties: {
    roleDefinitionId: roleDefinitionBlobContributor.id
    principalId: functionApp.identity.principalId
  }
}

var storageContributorRoleId = '17d1049b-9a84-46fb-8f53-869881c3d3ab'
resource roleDefinitionStorageContributor 'Microsoft.Authorization/roleDefinitions@2018-01-01-preview' existing = {
  scope: resourceGroup()
  name: storageContributorRoleId
}

resource roleAssignmentContrib 'Microsoft.Authorization/roleAssignments@2020-04-01-preview' = {
  name: guid(functionAppName, storageContributorRoleId)
  scope: storageAccount
  properties: {
    roleDefinitionId: roleDefinitionStorageContributor.id
    principalId: functionApp.identity.principalId
  }
}

resource plan 'Microsoft.Web/serverfarms@2020-12-01' = {
  name: appServicePlanName
  location: location
  kind: 'functionapp'
  sku: {
    name: 'S1'
  }
  properties: {}
}

resource functionApp 'Microsoft.Web/sites@2020-12-01' = {
  name: functionAppName
  location: location
  kind: 'functionapp'
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    serverFarmId: plan.id
    siteConfig: {
      appSettings: [
        {
          name: 'AzureWebJobsStorage__accountname'
          value: storageAccountName
        }
        {
          name: 'FUNCTIONS_WORKER_RUNTIME'
          value: 'dotnet'
        }
        {
          name: 'FUNCTIONS_EXTENSION_VERSION'
          value: '~4'
        }
        If the following is set, the host keys won't be available until the function app code has been deployed
        {
          name: 'WEBSITE_RUN_FROM_PACKAGE'
          value: '0'
        }
      ]
    }
    httpsOnly: true
  }
}

```

## Updating the Setting
After the "Scaffolding" of the app has completed, another pipeline will be used to deploy the function app. 

This should contain configuration to update the WEBSITE_RUN_FROM_PACKAGE setting to "1" as shown below:

```yaml
- task: AzureFunctionApp@1
  inputs:
  azureSubscription: ${{variables.svcConnection}}
  appType: functionApp
  appName: func-app-name-$(env)-001
  package: $(Pipeline.Workspace)/dotnet/**/app-name.AzFunc.zip
  resourceGroupName: rg-app-name-$(env)-001
  deployToSlotOrASE: true
  appSettings: WEBSITE_RUN_FROM_PACKAGE=1
```


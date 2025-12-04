---
title: Bicep Key Vault Secrets - Two Requirements You Must Meet
date: 2025-12-01
categories:
  - DevOps
  - Bicep
tags:
  - DevOps
  - Bicep
  - Key Vault
  - Azure
excerpt: |
  Understanding the two requirements for reading Key Vault secrets during Bicep deployments: enabledForTemplateDeployment AND deployment service principal permissions.
---

# A Helpful Bit of Key Vault Plumbing

I tripped over an ARM/Key Vault behaviour this week that is worth writing down, if only so future-me does not have to re-learn it.

The short version: if your Bicep template needs to **read a secret from Key Vault at deployment time** (for example when you call `keyVault.getSecret()` or use a Key Vault reference in a parameter), you need **two things**:

1. The Key Vault must have `enabledForTemplateDeployment: true`
2. The **deployment service principal** (e.g. your Azure DevOps service connection) must have the `Microsoft.KeyVault/vaults/deploy/action` permission (granted by Owner, Contributor, or `Key Vault Secrets User` roles)

Without **both**, you can end up with a very unhelpful `Forbidden` from Key Vault.


# Scenario

In my case I had two Bicep templates:

- a **resources** template that creates the Key Vault and assigns RBAC roles
- a **Logic App** template that wires up an API connection, pulling its `username` and `password` from the same Key Vault using `getSecret()`

The resources template created the Key Vault with RBAC enabled:

```bicep
resource keyVault 'Microsoft.KeyVault/vaults@2023-02-01' = {
  name: keyVaultName
  location: location
  properties: {
    tenantId: tenant().tenantId
    enableRbacAuthorization: true
    enabledForTemplateDeployment: false // we will change this shortly
    sku: {
      name: 'standard'
      family: 'A'
    }
  }
}
```

The Logic App template then tried to read secrets at deployment time:

```bicep
resource keyVault 'Microsoft.KeyVault/vaults@2023-02-01' existing = {
  name: keyVaultName
}

module apiv2ConnectorRestApi 'modules/integration.sharepoint.apiv2.bicep' = {
  name: 'apicv2-${baseResourceName}-rest-http-connection-${environmentType}'
  params: {
    baseResourceName: baseResourceName
    environment: environment
    tags: tags
    location: location
    baseUrl: SharepointRestApi_Baseurl
    username: keyVault.getSecret('gatewayConnectionUsername')
    password: keyVault.getSecret('gatewayConnectionPassword')
    nameSuffix: 'rest'
  }
}
```

But the Logic App pipeline failed with:

> `KeyVaultParameterReferenceSecretRetrieveFailed ... Http status code: 'Forbidden'. Error message: 'Access denied to first party service.'`


# The Two Requirements Explained

According to [Microsoft's documentation](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/key-vault-parameter), when you call `keyVault.getSecret()` during a Bicep deployment, **two conditions must be met**:

## Requirement 1: enabledForTemplateDeployment Must Be True

`enabledForTemplateDeployment: true` tells Key Vault:

> "Allow Azure Resource Manager to retrieve secrets from this vault when it is evaluating and executing templates."

This is a vault-level setting that permits ARM to access secrets during deployment operations.

## Requirement 2: The Deployment Identity Needs Permission

The identity executing the deployment (your Azure DevOps service connection's service principal) must have the `Microsoft.KeyVault/vaults/deploy/action` permission. This is included in:

- **Owner** role
- **Contributor** role
- **Key Vault Secrets User** role
- Or a custom role with `Microsoft.KeyVault/vaults/deploy/action`

In my case, I was missing the RBAC role assignment for the Azure DevOps service connection. Each environment's pipeline uses a different service connection:

| Environment | Service Connection | Service Principal |
|-------------|-------------------|------------------|
| Dev | azureappsharedresources-tfssc-nonprod-dev-001 | azureappsharedresources-tfssc-nonprod |
| Int | azureappsharedresources-tfssc-nonprod-int-001 | azureappsharedresources-tfssc-nonprod |
| Act | azureappsharedresources-tfssc-prod-act-001 | azureappsharedresources-tfssc-prod |
| Prod | azureappsharedresources-tfssc-prod-prod-001 | azureappsharedresources-tfssc-prod |

The fix required granting `Key Vault Secrets User` to each service principal in the resources Bicep template.


# The Fix

The fix required two changes in the resources Bicep template:

## 1. Enable Template Deployment on Key Vault

```bicep
resource keyVault 'Microsoft.KeyVault/vaults@2023-02-01' = {
  name: keyVaultName
  location: location
  properties: {
    tenantId: tenant().tenantId
    enableRbacAuthorization: true
    enabledForTemplateDeployment: true  // Required for getSecret()
    sku: {
      name: 'standard'
      family: 'A'
    }
  }
}
```

## 2. Grant Key Vault Secrets User to the Deployment Service Principal

```bicep
var environmentSettings = {
  dev: {
    // Object ID of the service principal behind the Azure DevOps service connection
    logicAppDeploymentSpnObjectId: 'c7bcaa21-9c78-4e2c-a2e4-73fbf63ca5ac' // azureappsharedresources-tfssc-nonprod
  }
  // ... other environments
}

// Grant the deployment service principal access to read Key Vault secrets
module logicAppDeploymentSpnKeyVaultSecretsUser 'roleassignmenthelper' = {
  name: 'logicAppDeploymentSpnKeyVaultSecretsUser'
  params: {
    principalId: environmentSettings.logicAppDeploymentSpnObjectId
    roleDefinition: 'Key Vault Secrets User'
  }
}
```

With both changes deployed (via the resources pipeline first), the Logic App pipeline ran successfully. ARM could now resolve `gatewayConnectionUsername` and `gatewayConnectionPassword` during the `what-if` and deployment steps without hitting `Forbidden`.


# When You Need This (and When You Don't)

You **need** both `enabledForTemplateDeployment: true` AND service principal permissions when:

- your Bicep/ARM template calls `keyVault.getSecret()`
- or you pass Key Vault secret values into parameters that ARM must resolve during deployment

You **don't** need them when:

- your app reads secrets at runtime using a managed identity and Key Vault SDK
- you use `@Microsoft.KeyVault(SecretUri=...)` references in app settings (these are resolved at runtime by the app's managed identity)
- you never ask ARM to read the secret as part of the deployment itself

In other words:

| Scenario | enabledForTemplateDeployment | Service Principal Permission |
|----------|------------------------------|------------------------------|
| `getSecret()` in Bicep | Required | Required |
| Key Vault reference in app settings | Not needed | Not needed (app's MI needs access) |
| Runtime SDK access | Not needed | Not needed (app's MI needs access) |


# Takeaway

If you see `KeyVaultParameterReferenceSecretRetrieveFailed` or "Access denied to first party service" while your Bicep is trying to grab a Key Vault secret during deployment, check **three things**:

1. The secret name is correct and exists
2. The vault has `enabledForTemplateDeployment: true`
3. The deployment service principal has `Key Vault Secrets User` (or equivalent) role on the vault

The third point is easy to miss when you are using Azure DevOps pipelines - the service connection's underlying service principal needs explicit RBAC access to the Key Vault, and this must be granted before the deployment that uses `getSecret()` runs.

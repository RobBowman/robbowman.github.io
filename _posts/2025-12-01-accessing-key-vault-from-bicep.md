---
title: Bicep Key Vault Secrets and enabledForTemplateDeployment
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
  Using enabledForTemplateDeployment to let ARM read Key Vault secrets during Bicep deployments.
---

# A Helpful Bit of Key Vault Plumbing

I tripped over an ARM/Key Vault behaviour this week that’s worth writing down, if only so future‑me doesn’t have to re‑learn it.

The short version: if your Bicep template needs to **read a secret from Key Vault at deployment time** (for example when you call `keyVault.getSecret()` or use a Key Vault reference in a parameter), you almost certainly want:

```bicep
enabledForTemplateDeployment: true
```

on the Key Vault.

Without it, you can end up with a very unhelpful `Forbidden` from Key Vault even though you think you’ve granted the right RBAC roles.


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

RBAC role assignments in the resources template looked correct – the deployment service principal had `Key Vault Secrets User` on the vault.

But the Logic App pipeline still failed with:

> `KeyVaultParameterReferenceSecretRetrieveFailed ... Http status code: 'Forbidden'. Error message: 'Access denied to first party service.'`


# What enabledForTemplateDeployment Actually Does

The missing piece is who is doing the reading.

When you call `keyVault.getSecret()` or use a Key Vault reference inside an ARM/Bicep deployment, it isn’t your deployment service principal doing the `Get Secret`. It’s **Azure Resource Manager itself** – a first‑party service principal.

`enabledForTemplateDeployment: true` tells Key Vault:

> "Allow ARM to read secrets from this vault when it is evaluating and executing templates."

That permission is **separate from RBAC assignments** you make to your own identities. You can happily have RBAC‑only access for apps and operators, but still allow ARM to read secrets at deployment time via this flag.


# Updated Key Vault Resource

In my case the fix was simply to enable `enabledForTemplateDeployment` on the Key Vault resource for this workload:

```bicep
resource keyVault 'Microsoft.KeyVault/vaults@2023-02-01' = {
  name: keyVaultName
  location: location
  properties: {
    tenantId: tenant().tenantId
    enableRbacAuthorization: true
    enabledForTemplateDeployment: true
    sku: {
      name: 'standard'
      family: 'A'
    }
  }
}
```

With that change deployed first (via the resources pipeline), the Logic App pipeline re‑ran successfully. ARM could now resolve `gatewayConnectionUsername` and `gatewayConnectionPassword` during the `what-if` and deployment steps without hitting `Forbidden`.


# When You Need It (and When You Don’t)

You **need** `enabledForTemplateDeployment: true` when:

- your Bicep/ARM template calls `keyVault.getSecret()`
- or you pass Key Vault secret values into parameters that ARM must resolve during deployment

You **don’t** need it when:

- your app reads secrets at runtime using a managed identity and Key Vault SDK / references in app settings only
- you never ask ARM to read the secret as part of the deployment itself

In other words:

- **Deployment‑time secret resolution** ➜ turn it on
- **Runtime secret resolution via managed identity** ➜ RBAC is enough, no flag required


# Takeaway

If you see `KeyVaultParameterReferenceSecretRetrieveFailed` or "Access denied to first party service" while your Bicep is trying to grab a Key Vault secret during deployment, check two things:

1. The secret name is correct and exists
2. The vault has `enabledForTemplateDeployment: true`

It’s a tiny setting, but in ARM/Bicep scenarios that mix RBAC‑enabled Key Vaults with deployment‑time secret reads, it makes all the difference between a clean pipeline run and another head‑scratching `Forbidden`. 

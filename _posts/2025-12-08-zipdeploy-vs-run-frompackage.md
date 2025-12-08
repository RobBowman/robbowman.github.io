---
title: "zipDeploy vs runFromPackage — They're Not the Same Thing"
date: 2025-12-08
categories: [DevOps]
tags: [Azure, Logic Apps, DevOps, Azure Functions]
excerpt: "Clearing up the confusion between deploymentMethod and WEBSITE_RUN_FROM_PACKAGE when deploying Logic App Standard workflows."
---

# The Confusing Naming That Had Me Scratching My Head

I was reviewing one of our Azure DevOps pipelines this week and spotted what looked like a contradiction:

```yaml
# In the Bicep template (app settings)
WEBSITE_RUN_FROM_PACKAGE: '0'
```

```yaml
# In the pipeline (deployment task)
- task: AzureFunctionApp@1
  inputs:
    deploymentMethod: zipDeploy
```

My brain said: "Wait — `WEBSITE_RUN_FROM_PACKAGE` is `0` (disabled) but we're using `zipDeploy`? Surely one of these is wrong?"

Turns out they're both correct. The naming is just confusing.

# Two Settings, Two Different Jobs

When deploying Logic App Standard (or Azure Functions), you typically have two stages:

1. **Infrastructure deployment** — Bicep/ARM template that creates the Logic App resource and configures app settings including `WEBSITE_RUN_FROM_PACKAGE`
2. **Code/workflow deployment** — Pipeline task (e.g., `AzureFunctionApp@1`) that pushes your zip package to Azure

These two settings control completely different things:

| Setting | Where It Lives | What It Controls |
|---------|----------------|------------------|
| `WEBSITE_RUN_FROM_PACKAGE` | App Settings (Bicep/ARM) | Tells the **runtime** how to serve files |
| `deploymentMethod` | Pipeline task input | Tells the **deployment task** how to upload the package |

# How zipDeploy Actually Works

When you use `deploymentMethod: zipDeploy` in your `AzureFunctionApp@1` task:

1. The task uploads your zip file to Kudu
2. Kudu **extracts** the contents to `D:\home\site\wwwroot`
3. Files are writable at runtime

The zip is just a transport mechanism — a convenient way to ship many files in one HTTP request. Once deployed, the zip is discarded; you're left with loose files on disk.

# How runFromPackage Actually Works

When you set `WEBSITE_RUN_FROM_PACKAGE=1` in your app settings and use `deploymentMethod: runFromPackage` in your pipeline:

1. The zip file is uploaded to `D:\home\data\SitePackages`
2. The runtime **mounts** the zip directly as the `wwwroot` folder
3. Files are **read-only** at runtime — because you're literally running from inside a zip archive

No extraction step. The app runs directly from the compressed package.

# The Matrix of Valid Combinations

| `WEBSITE_RUN_FROM_PACKAGE` | `deploymentMethod` | Result |
|----------------------------|-------------------|--------|
| `0` (or not set) | `zipDeploy` | ✅ Extract files, writable wwwroot |
| `1` | `runFromPackage` | ✅ Mount zip, read-only wwwroot |
| `0` | `runFromPackage` | ⚠️ Works but contradictory |
| `1` | `zipDeploy` | ⚠️ Works but contradictory |

The last two combinations will deploy successfully but create a mismatch between how files were deployed and how the runtime expects to serve them. Best avoided.

# Why Use Each Approach?

**zipDeploy (extract files):**
- Writable filesystem — useful if your app needs to write temp files to wwwroot
- Familiar model — files on disk, easy to inspect via Kudu
- Slightly slower deploys on large packages

**runFromPackage (mount zip):**
- Faster deployments — no extraction step
- Atomic deploys — old package or new package, never a partial state
- Eliminates file-lock conflicts during deployment
- Reduced cold-start times (especially for apps with many files)
- Read-only wwwroot — a benefit or a constraint depending on your scenario

# Putting It Together in Your Pipeline

Here's what a consistent configuration looks like. First, in your Bicep template:

```bicep
var appSettings = {
  // ... other settings ...
  WEBSITE_RUN_FROM_PACKAGE: runFromPackage ? '1' : '0'
}
```

Then in your Azure DevOps pipeline, match the deployment method:

```yaml
# For runFromPackage: true (read-only wwwroot)
- task: AzureFunctionApp@1
  inputs:
    azureSubscription: $(azureSubscription)
    appType: functionapp,workflowapp
    appName: $(logicAppName)
    package: $(Pipeline.Workspace)/logicapp/workflows.zip
    deploymentMethod: runFromPackage

# For runFromPackage: false (writable wwwroot)
- task: AzureFunctionApp@1
  inputs:
    azureSubscription: $(azureSubscription)
    appType: functionapp,workflowapp
    appName: $(logicAppName)
    package: $(Pipeline.Workspace)/logicapp/workflows.zip
    deploymentMethod: zipDeploy
```

The key is consistency: if your Bicep sets `WEBSITE_RUN_FROM_PACKAGE=1`, use `deploymentMethod: runFromPackage`. If it's `0` or not set, use `zipDeploy`.

# Takeaway

If you see `deploymentMethod: zipDeploy` paired with `WEBSITE_RUN_FROM_PACKAGE=0`, don't panic. They're saying the same thing in two different contexts:

- **Deployment time:** "Upload a zip and extract it"
- **Runtime:** "Run from extracted files, not a mounted package"

The naming overlap is unfortunate — both involve the word "zip" or "package" — but once you understand that `deploymentMethod` controls *how files arrive* and `WEBSITE_RUN_FROM_PACKAGE` controls *how files are served*, it clicks into place.

# References

- [Zip deployment for Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/deployment-zip-push)
- [Run your functions from a package file in Azure](https://learn.microsoft.com/en-us/azure/azure-functions/run-functions-from-deployment-package)
- [AzureFunctionApp@1 task reference](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/deploy/azure-function-app)
- [Set up DevOps deployment for Standard logic apps](https://learn.microsoft.com/en-us/azure/logic-apps/set-up-devops-deployment-single-tenant-azure-logic-apps)

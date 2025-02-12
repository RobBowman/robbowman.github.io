---
title: The Dreaded "Error in determining project root"
category: Azure
tags:
    - Logic Apps
---

## Overview
Logic app standard workflows can be developed either within the Azure Portal or from an extension to Visual Studio Code. For me, the VS Code extension approach should work better because it's more aligned to my usual way of working, to build and test locally before commiting to git for the CI/CD to do its thing.

## Wobbles
This [Microsoft Doc](https://learn.microsoft.com/en-us/azure/logic-apps/create-single-tenant-workflows-visual-studio-code#convert-your-project-to-nuget-package-based-net) describes that there are two logic apps standard project types:

+ Extension bundle-based (Node.js), which is the default type
+ NuGet package-based (.NET), which you can convert from the default type

Currently the default is the Node type. The documenation does explain that a project type can be converted to a .Net package based project quite easily and gives one reason for this as to support built-in connector authoring. It doesn't offer any downsides to the conversion, so I wonder why this is not offered as the only project format - as a means to greatly reduce confusion.

In practice, I've had less trouble sticking with the default project type.

## The Common Error
One error I often see from the VS Code extension is when I right-click a workflow and select "Open designer":

![error](/images/la-project-root/error-project-root.jpeg)

Through trial and error, I discovered that the extension will present this error if the extension does not find the following files in the parent folder of your logic app workflow:

+ local.settings.json
+ host.json

The LogicApp folder should also contain  .vscode folder with files for the extension and settings etc.

Below is an example folder structure. My workflows are contained in the folders named: ApimOneWay, ApimTwoWay, RunBRE, StubSatellite, StubSatelliteWaste

![error](/images/la-project-root/folders.png)

Worth noting that if you create a logic app standard workflow from scratch by using the Azure VS Code extension, then the missing files should be created for you.

## Workspace File to the Rescue
It seems the problems can be overcme by adding a workspace file. From within vscode explorer, select the folder containing your logic app workflows then from the File menu select, "Add Folder to Workspace...". This will create a ".code-workspace" file something like the following:

```json
{
  "folders": [
    {
      "name": "Functions",
      "path": "./src/Function"
    },
    {
      "name": "LogicApp",
      "path": "./src/LogicApp"
    },
    {
      "name": "Deploy",
      "path": "./deploy"
    },
    {
      "name": "Tests",
      "path": "./tests"
    },
    {
      "name": "IntegrationTests",
      "path": "./tests/LCC.Integration.ProcessManager.IntegrationTests"
    }
  ],
  "settings": {
    "terminal.integrated.env.windows": {
      "PATH": "C:\\Users\\rob\\.azurelogicapps\\dependencies\\DotNetSDK;${env:PATH}"
    },
    "omnisharp.dotNetCliPaths": [
      "C:\\Users\\rob\\.azurelogicapps\\dependencies\\DotNetSDK"
    ],
    "azurite.location": "C:\\Users\\rob\\.azurelogicapps\\.azurite",
    "terminal.integrated.cwd": "${workspaceFolder:LogicApp}",
    "terminal.integrated.env.osx": {
      "PATH": "/Users/rob/.azurelogicapps/dependencies/DotNetSDK:${env:PATH}"
    },
    "powershell.cwd": "Functions"
  }
}
```

Now choose File\Open Workspace From File

You should then be able to right-click a workflow.json file and chooose "open designer".

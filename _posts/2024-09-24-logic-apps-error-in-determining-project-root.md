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

## The Common Error
One error I often see from the VS Code extension is when I right-click a workflow and select "Open designer":

![error](/images/la-project-root/error-project-root.jpeg)

Through trial and error, I discovered that the extension will present this error if the extension does not find the following files in the parent folder of your logic app workflow:

+ local.settings.json
+ host.json

The LogicApp folder should also contain  .vscode folder with files for the extension and settings etc.

Below is an example folder structure. My workflows are contained in the folders named: ApimOneWay, ApimTwoWay, RunBRE, StubSatellite, StubSatelliteWaste

![error](/images/la-project-root/folders.png)

Worth noting that if you create a logic app standard workflow from scratch by using the Azure VS Code extension, then the missing file should be created for you.
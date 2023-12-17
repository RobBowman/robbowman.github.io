# Azure Function WEBSITE_RUN_FROM_PACKAGE
Over the years there have been multiple ways to host Azure Functions. My current preference is to host on a linux app service plan. The purpose of this post is to describe configuration I've found to be optimal for building and deploying azure function code into such an app.

## Microsoft Documentation
[This page](https://learn.microsoft.com/en-us/azure/azure-functions/run-functions-from-deployment-package) describes the benfits of running the azure function from a package file (zip)

## Working Pipeline
The following is two-stage devops pipeline, the first stage:

+ build the app
+ published the compiled binaries to a /publish folder (default behaviour of dotnet publish)
+ creates a zip of contents of publish folder and adds the zip as a pipeline archive

The second stage:

+ downloads the content of the pipeline archive (default vbehaviour of deployment task)
+ deploys to function app

```yaml
trigger:
  - main

pool:
  vmImage: ubuntu-latest

stages:
- stage: BuildAndArchive
  jobs:
  - job: BuildAndArchiveJob
    steps:
    - script: find $(System.DefaultWorkingDirectory) -type f
      displayName: 'List all files in workspace'

    - task: UseDotNet@2
      displayName: 'Use .NET Core sdk'
      inputs:
        packageType: sdk
        version: 7.x
        installationPath: $(Agent.ToolsDirectory)/dotnet

    - script: dotnet publish $(System.DefaultWorkingDirectory)/TableStoragePerf/TableStoragePerf.AzFunc/TableStoragePerf.AzFunc.csproj --configuration Release
      displayName: 'Publish Solution'

    - script: find $(System.DefaultWorkingDirectory)/TableStoragePerf -type f
      displayName: 'List all files in workspace'

    - task: ArchiveFiles@2
      displayName: 'Archive published files'
      inputs:
        rootFolderOrFile: '$(System.DefaultWorkingDirectory)/TableStoragePerf/TableStoragePerf.AzFunc/bin/Release/net7.0/publish' 
        includeRootFolder: false
        archiveType: 'zip'
        archiveFile: '$(Pipeline.Workspace)/functionapp.zip'

    - task: PublishPipelineArtifact@1
      displayName: 'Publish Artifact'
      inputs:
        targetPath: '$(Pipeline.Workspace)/functionapp.zip'
        artifact: 'drop'

- stage: DeployFunctionApp
  jobs:
  - deployment: DeployToFunctionApp
    displayName: 'Deploy to Azure Function App'
    pool:
      vmImage: 'ubuntu-latest'
    environment: 'Development'
    strategy:
      runOnce:
        deploy:
          steps:
          - script: find $(Pipeline.Workspace) -type f
            displayName: 'List all files in workspace'
          - task: AzureFunctionApp@2
            inputs:
              connectedServiceNameARM: 'sandpit'
              appType: 'functionAppLinux'
              appName: 'func-perftest'
              package: '$(Pipeline.Workspace)/drop/functionapp.zip'
              runtimeStack: 'DOTNET-ISOLATED|7.0'
              appSettings: '-AzureWebJobsStorage__accountName "storsandpit02" -WEBSITE_RUN_FROM_PACKAGE "1" -FUNCTIONS_EXTENSION_VERSION "~4"'
              deploymentMethod: 'auto'
```
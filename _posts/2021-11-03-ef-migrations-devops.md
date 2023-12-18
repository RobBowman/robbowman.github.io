---
title: EF Migrations with DevOps
category: DevOps
tags:
    - Azure
    - DevOps
---
# EF Migrations with DevOps
## Intro
Entity Framework migrations have been around for a long time, so I won't go into detail on what they're for, other than to say solve a difficult problem:

> How to manage changes to relational database structure once they contain "real" data

## Alternative to Database.Migrate()
With a db context in place, it can be tempting to simply add `Database.Migrate()` to the startup of your app. This is great for demos, but not really suited for production apps because there's no opportunity to see the script that's going to run against your database, or easy way to revert should something go wrong.

## Scripting in Pipelines
I found this approach works really well:
+ use the build pipeline to execute the ef command line that generates the sql script
+ also in the build pipeline, publish the generated sql script as a build artifact
+ from the release pipeline, use an Azure Sql Task to execute the sql script from the previous step

### Build Pipeline

>`- task: DotNetCoreCLI@2
      displayName: Create SQL Scripts
      inputs:
        command: custom
        custom: 'ef '
        arguments: migrations script --output $(Build.SourcesDirectory)\SQL\SmsRouterDbScript.sql --idempotent --project $(Build.SourcesDirectory)\src\infrastructure\SmsRouter.EntityFramework\SmsRouter.EntityFramework.csproj --context SmsOrderContext`

>`- task: PublishBuildArtifacts@1
      displayName: 'Publish Artifact: SQLScripts'
      inputs:
        PathtoPublish: $(Build.SourcesDirectory)/SQL/SmsRouterDbScript.sql
        ArtifactName: SQLScripts`

### Release Pipeline
![release-pipeline-image](/images/ef-migration/ReleasePipeline.png)
# Tuning Azure Function App Settings
I would admit to being guilty of accepting some default function app settings without really considering what they're for. This post describes some of useful settings that I'd miss understood.

## Reason for this Post
I started really digging into the settings when I hit a problem with a function app that was created by a bicep template. The function app was configured to make use of an existing Windows App Service Plan. Problem was, not of the function app keys were available. At the time of writing, this is yet to be solved, details can be seen in [this stack overflow post](https://stackoverflow.com/questions/76481871/unable-to-list-keys-for-azure-function-app).

## WEBSITE_RUN_FROM_PACKAGE
Since my functions are always going to be deployed through a CI/CD pipeline, I would always enable this feature. It gives the following benefits:

+ Reduces the risk of file copy locking issues.
+ Can be deployed to a production app (with restart).
+ You can be certain of the files that are running in your app.
+ Improves the performance of Azure Resource Manager deployments.

The [official documentation](https://learn.microsoft.com/en-us/azure/azure-functions/run-functions-from-deployment-package) for this setting is good but there are lots of older references from other sites that can be confusing. Possible values are:

| Value | Description |
|----------|----------|
|   1  |   Indicates that the function app runs from a local package file deployed in the d:\home\data\SitePackages (Windows) or /home/data/SitePackages (Linux) folder of your function app.  |
|   \<URL\>  |   Sets a URL that is the remote location of the specific package file you want to run. Required for functions apps running on Linux in a Consumption plan. |

### Value to Use = 1
The only time \<URL\> is now recommended is for Linux consumption plans.

## AzureWebJobsStorage
My advice is DO NOT USE THIS! This setting pre dates the capability of using the function app's managed identity to authenticate to a storage account - which is needed for a function app for core behaviors such as coordinating singleton execution of timer triggers and default app key storage.

## AzureWebJobsStorage__accountname
This is the newer setting that enables the function app to authenticate to a storage account using its managed identity. The value is the name of the storage account e.g 'stappname01'.

### Role Assignments
The RBAC of the storage account needs to be updated (I use bicep for this) to include the function app's managed identity. The required roles are described [here](https://learn.microsoft.com/en-us/azure/azure-functions/functions-reference?tabs=blob#connecting-to-host-storage-with-an-identity). The main one being "Storage Blob Data Owner". I was a little surprised by this since I'd though "Storage Blob Data Contributor" would be sufficient. The other role that will likely need adding are:

+ Storage Account Contributor
+ Storage Queue Data Contributor
+ Storage Table Data Contributor






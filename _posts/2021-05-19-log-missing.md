# Logs Missing from Local Az Functions
I created a new Durable Functions project recently and noticed my logs weren't appearing in the console window when running from Visual Studio.

A little googling took me to the following [git hub issue](https://github.com/Azure/azure-functions-host/issues/4345). It seems there's a problem that causes custom log entries to be filtered out.

## Failure :(

The solution recommended in that post, is to amend the host.json file to give specific "logLevel" entries for each namespace of the logger types, for example:

     "logLevel": {
        "Default": "Information",
        "Microsoft.EntityFrameworkCore.Database.Command": "Information",
        "SmsRouter.Core.StructuredLogger": "Information"
      }

Unfortunately, this didn't work for me.

## Success :)
Even if you have more joy with the above than I did, it could be that you'd like to filters to vary between the environments. In this case, an application setting (local.settings.json or entry in your ARM template) can be used to override the value of the host.json, as shown in the example below (taken from [DavidG's answer on SO](https://stackoverflow.com/questions/64229383/in-azure-functions-3-can-i-filter-logs-to-application-insights-and-the-console-a#):

    {
    "IsEncrypted": false,
    "Values": {
        /* snip */
        "AzureFunctionsJobHost__logging__logLevel__default": "Trace"
    }
    }
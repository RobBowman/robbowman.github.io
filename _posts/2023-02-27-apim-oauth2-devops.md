# Automating Deployment of Azure API Management Requiring App Registrations 

## Overview
I recently faced the task of automating deployment of an api into Azure API Management that would require clients to authenticate via Azure Active Directory .

The requirement is illustrated below:

![](/images/apim-oauth2/overview.png)

### App Registrations

Two app registrations are required:

1. for the client wishing to access the secured resource (client app reg)
2. for the service that is to be secured (service app reg)

In this project, the name of the client app reg is of the format: "apreg-amx-gis<env>-<num>" e.g. "apreg-amx-gis-act-002". The name of the service app reg is of the format "apreg-gis-maps-act-002"

### Expose an API
A scope needs to be created on the service app reg. From the Azure Portal, this can be found under the option Manage\Expose an API as shown below:

![](/images/apim-oauth2/expose-api.png)

*Note: in the above screenshot you can see that the application id uri has been updated from a guid to another unique but more memorable string.*

### API Permissions
With the service app reg scope made available, it's then possible to assign a permission of the scope to the client app reg.

From the Azure Portal, this can be selected from API Permissions\Add a permission\My APIs

### Securing Individual Operations
For each operation to be secured by OAuth, a validate-jwt policy is assigned at the "inbound processing / operation.  This is illustrated in the following code snippet:

````xml
<inbound>
        <validate-jwt header-name="Authorization" failed-validation-httpcode="401" failed-validation-error-message="Access token is missing or invalid">
            <openid-config url="https://login.microsoftonline.com/organizations/v2.0/.well-known/openid-configuration" />
            <required-claims>
                <claim name="aud">
                    <value>[appId]</value>
                </claim>
            </required-claims>
        </validate-jwt>
        <!-- run base after because it strips the Authorization header -->
        <base /> 
    </inbound>
````
The policy shows that it will check for the presence of a audience (aud) claim with a value matching a placeholder value [appId]. This placeholder will be updated with the app id of the service app reg during execution of the deployment DevOps pipeline.

## Azure DevOps Pipeline
The devops pipeline performs the following tasks
+ creates Azure app registration for server app reg and writes app id to pipeline variable
+ create Azure app registration for client app reg
+ runs bicep deployment to create API Management components such as api from the open api definition 
+ imports the policy definitions and injects the app id of the server app reg into the validate-jwt policy - to be checked against the audience claim

|It seems automation of app registrations is not supported by bicep so I turned to Azure CLI to fulfill. 

### Create Service App Reg
````ps
Write-Host "Going to check if app reg already exists: $appRegName"
    $appReg = az ad app list --display-name $appRegName | ConvertFrom-Json
    if ($appReg.DisplayName.Length -eq 0) {
        Write-Host "App reg does not yet exist so going to create"
        $appReg = az ad app create --display-name $appRegName | ConvertFrom-Json
    }
    else {
        Write-Host "App reg already exists" + $appReg.DisplayName
    }
    $serviceAppRegAppId = $appReg.AppId
    $serviceAppRegObjectId = $appReg.ObjectId

    Write-Host "Going to set additional owner of app registration for subsequent configuration via Azure Portal"
    az ad app owner add --id $appReg.Id --owner-object-id $ownerObjectId
    
    Write-Host "Checking service principal exists"
    $spn = az ad sp list --display-name $appRegName --all | ConvertFrom-Json
    if ($spn.DisplayName.Length -eq 0) {
        Write-Host "Service principal does not yet exist so going to create"
        az ad sp create --id $appReg.appId | Out-Null
    }
    else {
        Write-Host "Service principal already exists"
    }

    #Expose an API
    . "$PSScriptRoot\add-scope-to-service-app-reg.ps1" -serviceAppRegAppId $serviceAppRegAppId

    #Update manifest
    az ad app update --id $serviceAppRegAppId --set accessTokenAcceptedVersion=2

    Write-Host "serviceAppRegAppId=$serviceAppRegAppId"
    Write-Host "serviceAppRegObjectId=$serviceAppRegObjectId"
    # update global variables to be used later in pipeline
    Write-Host "##vso[task.setvariable variable=appRegAppId;]$serviceAppRegAppId"
    Write-Host "##vso[task.setvariable variable=appRegObjectId;]$serviceAppRegObjectId"
````

### Add Scope to Service App Reg
````ps
param (
    [Parameter(Mandatory = $true)]
    [string]$serviceAppRegAppId
)

$uuid = [guid]::NewGuid()
$api = @{
    acceptMappedClaims = $null
    knownClientApplications = @()
    oauth2PermissionScopes = @(
        @{
            adminConsentDescription = "admin description"
            adminConsentDisplayName = "admin name"
            isEnabled = $true
            id = "'$uuid'"
            type = "User"
            userConsentDescription = "user description"
            userConsentDisplayName = "user name"
            value = "amx.read"
        }
    )
    preAuthorizedApplications = @()
    requestedAccessTokenVersion = 2
} | ConvertTo-Json

write-host "api=$api"
write-host "serviceAppRegAppId=$serviceAppRegAppId"

$appRegName = az ad app show --id $serviceAppRegAppId --query displayName
write-host "appRegName=$appRegName"

$appRegNameWithoutQuotes = $appRegName.Trim('"')

write-host "Going to add scope to app reg:$appRegNameWithoutQuotes"

# Update app registration with App ID URL and api object
az ad app update `
    --id $serviceAppRegAppId `
    --identifier-uris "api://$appRegNameWithoutQuotes" `
    --set api="$api"
````

### Deploy Client App Reg
````ps
$myAppRegAppId
    Write-Host "Going to check if app reg already exists: $appRegName"
    $appReg = az ad app list --display-name $appRegName | ConvertFrom-Json
    if ($appReg.DisplayName.Length -eq 0) {
        Write-Host "App reg does not yet exist so going to create"
        $appReg = az ad app create --display-name $appRegName | ConvertFrom-Json
    }
    else {
        Write-Host "App reg already exists" + $appReg.DisplayName
        Write-Host "leaving else block"
    }
    $myAppRegAppId = $appReg.AppId


    Write-Host "Going to set additional owner of app registration in case subsequent configuration via Azure Portal is needed"
    az ad app owner add --id $appReg.Id --owner-object-id $ownerObjectId
    
    Write-Host "Checking service principal exists"
    $spn = az ad sp list --display-name $appRegName --all | ConvertFrom-Json
    if ($spn.DisplayName.Length -eq 0) {
        Write-Host "Service principal does not yet exist so going to create"
        az ad sp create --id $appReg.appId | Out-Null

        Write-Host "Generating secret"
        $appRegSecret = az ad app credential reset --id $appReg.id | ConvertFrom-Json

        Write-Host "Writing secret to Key Vault"
        $keyVaultName = "kv-seed-${env}-001"
        $keyVaultSecret = az keyvault secret set --vault-name $keyVaultName --name $appReg.displayName --value $appRegSecret.password | ConvertFrom-Json

        if ($keyVaultSecret) {
            Write-Host "Key Vault secret created"
        }
        else {
            Write-Error "Trouble creating secret"
        }
    }
    else {
        Write-Host "Service principal already exists"
    }

    Write-Host "scopeName=$scopeName"
    Write-Host "serviceAppRegAppId=$serviceAppRegAppId"
    $scopeIdWithQuotes = az ad sp show --id $serviceAppRegAppId --query "oauth2PermissionScopes[?value=='$scopeName'].id | [0]"
    Write-Host "scopeIdWithQuotes=$scopeIdWithQuotes"

    $scopeId = $scopeIdWithQuotes.Trim('"')
    Write-Host "scopeId=$scopeId"
    $permission = "$scopeId=Scope"
    Write-Host $permission
    Write-Host "Going to run the following: az ad app permission add --id $myAppRegAppId --api $serviceAppRegAppId --api-permissions $permission"
    az ad app permission add --id $myAppRegAppId --api $serviceAppRegAppId --api-permissions $permission
}
````

###
Import APIM Policy Definition
````js
resource protectedPolicy 'Microsoft.ApiManagement/service/apis/operations/policies@2022-04-01-preview' = {
  name: 'policy'
  parent: protectedOperation
  properties: {
    value: getPolicyWithAppRegId.outputs.updatedPolicy
    format: 'rawxml'
  }
}
````

````js
module getPolicyWithAppRegId 'get-policy.bicep' = {
  name: 'getPolicyModule'
  params: {
    appRegAppId: appRegAppId
  }
}
````

````js
param appRegAppId string

output updatedPolicy string = replace(loadTextContent('./apim-policies/protected-op-policy.xml'),'[appId]', appRegAppId)
````

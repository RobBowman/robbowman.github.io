---
title: Deploying Products to Azure API Management
category: Azure
tags:
    - Azure
    - APIM
---

## Overview
My standard method of deploying into Azure is to run a Bicep template from a DevOps pipeline. I have found the declarative "desired state" approach of Bicep to work better than the more imperative PowerShell scripting I'd used previously.

However, there are occasions where I find it makes sense to fall back to the older method and I encountered one this week in relation to deployment of resources into API Management.

## The Problem with Bicep
The issue related to linking of API Management Products to APIs within the APIM instance. 

The Bicep template would read from an array of configuration data as shown below:

```
var apimSubscriptions = [
    {
        name: 'Subscription'
        description: 'Description for sub1'
        apis: [
            'name-of-api1-for-sub1_v1'
            'name-of-api2-for-sub1_v1'
        ]
    }
    {
        name: 'Subscription2'
        description: 'Description for sub2'
        apis: [
            'name-of-api1-for-sub2_v1'
            'name-of-api2-for-sub2_v1'
        ]
    }
]
```

It would deploy the APIM Product using the following:
```
resource apimProduct 'Microsoft.ApiManagement/service/products@2022-08-01' = [for s in apimSubscriptions: {
    parent: apim
    name: replace(toLower(s.name), ' ', '-')
    properties: {
        displayName: s.name
        description: s.?description ?? null
        state: 'published'
    }
}]
```

It would then create a link between the Product and existing APIM APIs with the following:

```
@batchSize(1)
resource apimProductApi 'Microsoft.ApiManagement/service/products/apis@2022-08-01' = [for pa in apimProductApis: {
    name: '${apimName}${pa}'
    dependsOn: apimProduct
}]
```
The problem was, it would occassionally come accross the definition of a subscription (Product) that contained an API that had not yet been deployed to the target APIM instance. In this case, the Bicep template would fail.

 Due to the fact that our APIM is being managed as monolith, this is a scenario that we needed to handle. For example, it could be that a developer made an update to the APIM config data to link a subscription to a new API which they had deployed into the development subscription. A different developer may then wish to promote the APIM release pipeline through to the Test environment, but on doing so it would fail because the first developer's API had not yet been deployed there.

 Deployment of the APIM Product to API links needed to be made more flexible. It needed to attempt to make the link but continue if it was unable to.

## The solution
 The following PowerShell script is called from the same DevOps pipeline as the previous Bicep:

 ```ps
 # This script it called from DevOps Pipeline Azure.Scaffolding.ApimSubscriptions
# It achieves the following:
# 1. Reads a JSON file (ApimProductApis) containing a list of APIM products and APIs to be linked to each product
# 2. For each product in the JSON file:
#     a. Checks if the product exists in the APIM instance
#     b. If the product does not exist, creates the product
#     c. If the product exists, retrieves the product ID
#     d. If the product has a product policy, sets the policy for the product (from xml file in product-policies subdirectory)
#     e. For each API in the product:
#         i. Checks if the API exists in the APIM instance
#         ii. If the API exists, checks if the API is linked to the product
#         iii. If the API is not linked to the product, links the API to the product
#         iv. If the API does not exist, skips linking the API to the product

# If testing from vscode
# $subscriptionId = "subscription id"
# Import-Module Az 
# Connect-AzAccount
# Set-AzContext -SubscriptionId $subscriptionId

param (
    [string]$apimName = "apim name",
    [string]$resourceGroupName = "rg name",
    [string]$jsonFilePath = "./scripts/ApimProductApis.json",
    [string]$subscriptionId = "your subscription id"
)

Import-Module Az

# Create the context
$apimContext = New-AzApiManagementContext -ResourceGroupName $resourceGroupName -ServiceName $apimName

# Read the JSON file
$apimSubscriptions = Get-Content -Path $jsonFilePath | ConvertFrom-Json

foreach ($subscription in $apimSubscriptions) {
    $productNameHyphen = $subscription.name -replace ' ', '-'
    $productNameLower = $productNameHyphen.ToLower()
    $productName = $subscription.name

    # Check if the product exists
    $products = Get-AzApiManagementProduct -Context $apimContext | Where-Object { $_.Title -eq $productName -or $_.Title -eq $productNameLower }
    Write-Output "Products retrieved: $products"

    if ($null -eq $products) {
        # Create the product if it doesn't exist
        $product = New-AzApiManagementProduct -Context $apimContext -ProductId $productName -DisplayName $productName -Description "Product for $($productName)"
        Write-Output "Created product '$productName'"
    }
    else {
        $product = $products[0]
        Write-Output "Product '$productName' already exists. Skipping creation."
    }
    $productId = $product.ProductId
    Write-Output "Product id: $($product.ProductId)"


    # Read optional product policy from productPolicy property of $jsonFilePath (child property of each subscription)
    # if property exists, it's value will contain filename of xml file containing policy - to be found in sub directory named 'product-policies'
    # read the xml file and set the policy for the product
    if ($null -ne $subscription.productPolicyXml) {
        $policyFilePath = "./scripts/product-policies/$($subscription.productPolicyXml)"
        $policy = Get-Content -Path $policyFilePath -Raw

        Set-AzApiManagementPolicy -Context $apimContext -ProductId $productId -Policy $policy
        Write-Output "Policy for product '$productName' has been set"
    }

    #loop each api in our json list (ApimProductApis.json)
    foreach ($api in $subscription.apis) {

        # Check if the API exists in APIM
        $found = Get-AzApiManagementApi -Context $apimContext -ApiId $api
        if ($null -ne $found)
        {
            # Get the list of APIs linked to the specified product
            $productApis = Get-AzApiManagementApi -Context $ApimContext -ProductId $productId

            # Check if the specified API ID exists in the list
            if ($productApis.ApiId -contains $api)
            {
                Write-Output "API: '$api' already linked to product: '$productName'"
            }
            else
            {
                # Link the API to the product
                Write-Output "Going to link API '$api' to product '$productName'"
                Add-AzApiManagementApiToProduct -Context $apimContext -ProductId $productId -ApiId $found.ApiId
                Write-Output "Linked API '$api' to product '$productName'"
            }
        }
        else
        {
            Write-Output "API '$api' does not exist in APIM. Skipping linking to product '$productName'"
        }
    }
}
 ```

### Sample Json Config File
 ```json
 {
  "apimSubscriptions": [
    {
      "name": "Subscription",
      "description": "Description for sub1",
      "apis": [
        "name-of-api1-for-sub1_v1",
        "name-of-api2-for-sub1_v1"
      ]
    },
    {
      "name": "Subscription2",
      "description": "Description for sub2",
      "apis": [
        "name-of-api1-for-sub2_v1",
        "name-of-api2-for-sub2_v1"
      ]
    }
  ]
}
 ```
 #### ApimProductApis.json

 ### Sample Policy Xml File - to be applied at Product scope
 ```xml
 <policies>
    <inbound>
        <base />
        <rate-limit calls="2000" renewal-period="60" />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
 ```
 #### Subscription1.xml
# Hosting Azure DevOps Agents on a VMSS
The only time I'd previously had to veer from using Microsoft's hosted build agent was for building "unusual" things like BizTalk applications. However, I've recently started a move to ASEv3 (Application Service Environment) for hosting app services. For this, the standard hosted agents won't work because they don't have connectivity to an Isolated App Service Plan of an ASE.

## What, No Docker!
My first thought was to deploy the build agent in docker, hosted on Container Apps. I found a nice [blog post](https://moimhossain.com/2022/11/08/self-hosted-azure-devops-pool-on-azure-container-apps/) that describes how to do this. This worked, and I was able to run a run a build, but the docker image was missing tools such as dotnet.

## Azure Resources
There are several Azure VM related resources at play. Documentation in the repos seems a little inconsistent in the names used to refer to the different types. My understanding is as follows:

+ VHD - stored as a blob in Azure Storage
+ Managed Disk - an Azure resource that can be created from the VHD (this doesn't seem to be essential)
+ Azure Image - can be created from a Managed Disk or directly from a VHD
+ VMSS - Virtual Machine Scale Set that references the Azure Image

## What do Microsoft Use for their Hosted Agents?
It turns out that Microsoft use an Azure VMSS (Virtual Machine Scale Set) to host their agents, for both DevOps Pipelines and GitHub Actions. License conditions prevent Microsoft from sharing the images. However, they are able to share the scripts they use to build the images. These are built with a tool called [Packer](https://www.packer.io/) and Microsoft's scripts can be found at [this](https://github.com/actions/runner-images) GitHub repo. Detailed instructions for building the images can be found [here](https://github.com/actions/runner-images/blob/main/docs/create-image-and-azure-resources.md)

### The Process
There are two key PowerShell functions referenced from the instructions:

+ GenerateResourcesAndImage
+ CreateAzureVMFromPackerTemplate

#### GenerateResourcesAndImage
This generates the VHD image. It does this by creating a VM in Azure, running the Packer script on the VM, capturing the VM as an image and saving to an Azure Storage blob. It also creates an ARM template that can be used by the CreateAzureVMFromPackerTemplate function. It then deletes the VM Packer Created.

#### CreateAzureVMFromPackerTemplate
This is optional and can be used to create an Azure VM from the image created by the GenerateResourcesAndImage function.

## Automating Creation of the VMSS
The above solves the problem of creating the VM Image but I wanted to automate this process. Thankfully, I discovered [this](https://github.com/YannickRe/azuredevops-buildagents) GitHub repo that does just that.

### Chicken and Egg
The whole idea is to create a self-hosted build agent, without any limits on connectivity or duration. However, running a pipeline that runs the GenerateResourcesAndImage PowerShell function takes several hours to complete. So, for the first pipeline run, I configured it to use an agent pool which sources it's agent from my MacBook. I found instructions for configuring the agent [here](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/osx-agent?view=azure-devops). This worked well, just a couple of notes:

+ I installed the agent to Users/rob/Applications/myagent
+ I ran ./config.sh once, this is where it asks for the PAT token
+ I ran ./run.sh to start the agent

I was then able to run the pipeline to create the VMSS. In order to prevent my MacBook from going to sleep and interrupting progress of the agent, I ran a YouTube video in the background

## Missing VM Image Definition
The pipeline gives the option to publish the newly created image to an Azure Compute Gallery. I found that this failed because although I had created the Compute Gallery, I had not created the VM Image Definition. 

### Helper Script
The documentation in Yannick's repo and his [website](https://blog.yannickreekmans.be/use-azure-devops-to-create-self-hosted-azure-devops-build-agents-just-like-microsoft-does/) is excellent. Some of this is reproduced below but I've also added a few az commands I used to create Azure resources that the pipeline expects to find:

```powershell
$location = 'UK South'
$rgName = 'rg-devops-packer-001'
$stAcName = 'stdevopspacker01'
$subscriptionName = 'sub name here'

# Connect to Azure Sub
Connect-AzAccount -TenantId 'tenant id here'
Set-AzContext -SubscriptionName $subscriptionName
New-AzResourceGroup -Name $rgName -Location $location
New-AzStorageAccount -ResourceGroupName $rgName -AccountName $stAcName -Location $location -SkuName "Standard_LRS"

# Create Azure AD Service Principal, output client secret and client id
$sp = New-AzADServicePrincipal -DisplayName "DevOps-Packer"
$sp.PasswordCredentials.SecretText
$sp.AppId

# Make the Service Principal a Contributor on the subscription
New-AzRoleAssignment -RoleDefinitionName Contributor -ServicePrincipalName $sp.AppId

# Make the Service Principal a Storage Blob Data Contributor on the subscription
New-AzRoleAssignment -RoleDefinitionName "Storage Blob Data Contributor" -ServicePrincipalName $sp.AppId

# Create Scale Set
az login --scope https://management.core.windows.net//.default
az account set --subscription $subscriptionName
az vmss create --resource-group $rgName `
  --name 'vmss-devops-packer-001' `
  --upgrade-policy-mode manual `
  --location $location `
  --disable-overprovision `
  --vm-sku Standard_DS2_v2 `
  --image Ubuntu2204

# Create azure compute gallery
$galleryName = 'gal_devops_packer_001'
az sig create `
  --resource-group $rgName `
  --gallery-name $galleryName `
  --location $location

# create vm gallery image definition
az sig image-definition create `
  --resource-group $rgName `
  --gallery-name $galleryName `
  --gallery-image-definition ubuntu2204-agentpool-full `
  --os-type Linux `
  --publisher BizTalkers `
  --offer Ubuntu `
  --sku 22.04

```
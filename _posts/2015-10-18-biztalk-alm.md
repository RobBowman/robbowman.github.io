---
title: BizTalk CI/CD
category: BizTalk
tags:
    - BizTalk
    - CI/CD
---
# BizTalk CI/CD
## Background
In this post I'd like to describe a BizTalk build process I recently developed for a client and their BizTalk 2013(r1) applications. Up until recently, they had used an on-premise installation of TFS 2012 for all their ALM needs. However, by the time I was engaged, they had migrated their source code into Azure TFS, aka Visual Studio Online (VSO). 
The solution was to create a hybrid ALM solution for BizTalk. It turns out this works really well - even with the old Visual Studio 2012 that's required to build this version of BizTalk. So, I though it worth sharing, hope it helps…

## Acknowledgments
I'd like to acknowledge the authors of the BizTalk Deployment Framework (https://biztalkdeployment.codeplex.com/) and BizTalk ALM Guide (https://biztalkalm.codeplex.com/) - both superb and very helpful.
I'd also like to thank Toon Vanhoutte (@ToonVanhoutte) for giving an excellent Integration Mondays presentation which  demonstrated the benefits of the BizTalk ALM Guide.

## Architecture
Although VSO offers a cloud-hosted build service, this does not support builds of BizTalk solutions. Also, cloud hosted builds are charged by CPU time and the cost can be significant. For these reasons, it was decided that the build process would run on a build server hosted by the client, giving a hybrid cloud-build architecture as illustrated by the following diagram:

![architecture pic](/images/biztalk-alm/architecture.png)

## The VSO Build Agent

The VSO Build Agent is a key component in the process. This may be downloaded from the VSO website as a small zip file. The zip contains a Powershell script which, when run, installs a Windows Service onto the build server named “VSO Agent”. The script also prompts the user for various information including:
<ul>
<li>The URL for the VSO website (https://company.visualstudio.com)</li>
<li>The on-premise domain credentials that the windows service should run under</li>
<li>The azure credentials to be used when the agent connects to VSO</li>
</ul>
## Workflows
These are the VSO builds but since in our case they manage build, deployment and test I shall refer to them as "workflows". 
Two different build & deployment workflows were configured:
<ul>
<li>“BizTalk Build and Unit Test” – to be triggered for each code check-in. This needs to be relatively fast so as to feedback results to the developer whilst the change is still fresh in their mind
</li>
<li>The on-premise domain credentials that the windows service should run under</li>
<li>“BizTalk Deploy and Int Test” – to be scheduled to run nightly. This takes longer to execute because it must undeploy then redeploy all BizTalk applications and their dependencies before running integration tests</li>
</ul>

These workflows are described in detail in section later in this post.

## Repository
The first stage of any VSO build workflow is to pull from source control, the files required for the build. This is determined by the path entered on the “Repository” page, as shown below:
![repo pic](/images/biztalk-alm/repo.png)

## BizTalk Build and Unit Test Workflow
The “BizTalk Build and Unit Test” VSO process is relatively simple. It contains only two different types of build step, as shown below:
![build and unit test pic](/images/biztalk-alm/build-and-unit-test.png)

## BizTalk Deploy and Int Test Workflow
This workflow is more complex because, in addition to building the BizTalk applications and dependencies, it must deploy before executing integration tests. This automated deployment must first remove previously deployed applications, taking care to ensure they are removed and added in the correct order.
A screen grab of the BizTalk Deploy and Int Test Workflow can be seen below:
![deploy workflow](/images/biztalk-alm/deploy-workflow.png)
## The Deployment Scripts
As mentioned in the previous sections, building of the BizTalk applications is processed by VSO Visual Studio Build Tasks, first processing the Visual Studio solution file, then the BizTalk Deployment Framework (BTDF) project file.
The BTDF has been around for a long time and has great documentation and support available from https://biztalkdeployment.codeplex.com/. The BTDF capabilities utilised by our deployment are standard, it has not been necessary to develop custom MSBuild targets etc.
The codeplex project that provides the deployment scripts can be found at https://biztalkalm.codeplex.com. This is less well known. Also, its documentation describes a fully on-premise build and deployment with TFS2013. Some customisation has been necessary to give the hybrid VSO / on-prem build architecture described in section 2. Therefore, details on these scripts is provided here.
## Sequence
The sequence in which the scripts are run is illustrated in the following diagram:
![sequence pic](/images/biztalk-alm/sequence.png)

## Update Version Info.ps1
### Assembly FileVersion
This script scours the local source folder (previously pulled from the VSO repository) looking for files named “AssemblyInfo.cs”, a screen clipping one such file can be seen below:
![assembly file version pic](/images/biztalk-alm/assembly-file-version.png)
It can be seen from the previous figure, the file contains two version attributes. The “AssemblyVersion” is the “Real” version applied to the assembly. This is used by the .Net framework when linking an assembly that is referenced from another .Net resource, such as an exe or dll. So, for example, if assembly 1 has a reference to assembly 2 version 3.4.2.1 then it is the “AssemblyVersion” attribute of the “AssemblyInfo.cs” file that must hold the value 3.4.2.1, enabling the .Net framework to link the reference. Our deployment scripts DO NOT make any use of the “AssemblyFile” attribute. Instead, they update the value of the “AssemblyFileVersion” attribute. This is never used by the .Net framework but provides a useful mechanism for developers / administrators to identify a version of a deployed assembly, as shown in the following screen grab:
![file version pic](/images/biztalk-alm/file-version.png)
The version number that is applied following the third full stop, is provided from VSO and is known as the Build URI. The following screen grab shows the result for build uri 324:
![version uri pic](/images/biztalk-alm/version-uri.png)
### BizTalk MSI Version
In addition to setting the file version for the assemblies, the script also applies the same version number to the BizTalk MSIs, both appended to the filename and the entry in control panel, as shown below:
![2 versions pic](/images/biztalk-alm/2-versions.png)
The UpdateVersionInfo.ps1 script sets the MSI version number by updating the .btdfproj file as shown below:
![msi version pic](/images/biztalk-alm/msi-version.png)
The script also gives a meaningful value to the “BizTalkAppDescription” element, which can be surfaced by selecting properties of the application in the BizTalk Admin Console, as shown in the following screen grab:
![application pic](/images/biztalk-alm/application-pic.png)
## Create Deployment.ps1
When the on-premise VSO build agent runs, it stores source files pulled from the Azure repository in a local folder as shown in the following screen grab:
![](/images/biztalk-alm/folder-tree.png)
The root of this hierarchy is “c:\VSOBuildAgent”. Beneath this, the relevant folder is “_work”. This contains a sub folder for each VSO build workflow that has been executed on the build server. In the case of the screen grab, folder “46516392” was created for the “BizTalk Deploy and Int Test” VSO workflow. Beneath this, the “s” folder contains all source code as selected from the VSO “Repository” page, described previously
The build steps executed earlier in the workflow, build each of the BizTalk .sln files, resulting in .dll files being created in the \bin\release folders of the relevant applications, as shown in the following screen grab:
![exp1](/images/biztalk-alm/exp1.png)
Additional build steps execute MSBuild against the .btdfproj file of each BizTalk application. This results in MSIs being created for each BizTalk application, as shown in the following screen grab:
![](/images/biztalk-alm/exp2.png)
The CreateDeployment.ps1 script performs the following actions:
<ul>
<li>Creates a “bin” folder at the same level as the “s” source code folder</li>
<li>Copies all files with the following extensions in any sub directory of “s” to this top level bin folder:</li>
<ul><li>"*.exe","*.dll","*.exe.config","*.pdb", "*.msi"</li></ul>
<li>Creates a sub folder of this bin folder, labelled as per the VSO build number – known as the package folder</li>
<li>Copies all BizTalk application MSIs and deployment scripts to this package folder, from the top level bin folder</li>
<li>Creates a zip of this package folder</li>
</ul>
A screen grab of this zip can be seen below:
![](/images/biztalk-alm/exp3.png)
## TFS Auto Deploy.ps1
With the previous CreatDeployment.ps1 script complete, we now have a zip containing all that is needed to undeploy then deploy all BizTalk applications for any server. However, we have a requirement to automate this process so that Integration Tests may automatically run following a scheduled deployment – e.g. nightly.
The TFS-AutoDeploy.ps1 script performs the following actions:
<ul>
<li>Extract content of zip created by previous script</li>
<li>Execute the Undeploy.ps1 script</li>
<li>Execute the Deploy.ps1 script</li>
</ul>
## Undeploy.ps1
This script makes use of standard BTDF functionality to undeploy each BizTalk application. The list of BizTalk applications is held in simple text file name “Applications.txt”. Although very simple, this file is critical to the processing of both the Undeploy.ps1 and Deploy.ps1 scripts. The content of this file can be seen below:
![](/images/biztalk-alm/undeploy.png)
As can be seen from the previous screen grab, the file simply lists the names of all our BizTalk applications. The order in which these are listed is important since the Undeploy.ps1 script will undeploy them in reverse order, whereas the Deploy.ps1 will deploy them in the order listed. Because each of the applications has a dependency on the FFF.Enterprise.Shared application, this must appear first in the list.
## Deploy.ps1
The deploy script also makes use of standard BTDF functionality, this time to deploy each application listed in the “Applications.txt” file. Requesting the BTDF to install a BizTalk application requires many configuration arguments to be set, to support both the execution of the MSI and the subsequent deployment into the BizTalk management database. This complexity is encapsulated by a helper script called “Install-BizTalkApplication.ps1”. 
###The BTDF_Env Environment Variable
When the BTDF deploys an application, it needs to know which bindings to apply. It determines this by checking an environment variable called “BTDF_Env”. This must be set as a one-off task for each server onto which BTDF is to deploy applications. The value assigned must match that given within the EnvironmenSettings.xml file of the BTDF project, for example suitable values: “Dev”, “Build”, “QA, “Prod”. This is standard BTDF functionality but is so critical, it's worth mentioning here.
###The BTDF_DeployToBizTalk Envinronment Variable
This is another standard, but critical BTDF environment variable. It simply contains “true” if the server is the last within a BizTalk group. The BTDF uses this to determine whether or not the BizTalk applications should be deployed to the management database after an MSI has been installed. This database deployment should happen only once for a particular application with a single BizTalk group.




---
title: BizTalk Smoke Testing
category: BizTalk
tags:
    - BizTalk
---
# BizTalk Smoke Testing
After building a new BizTalk environment it can be helpful to run a few simple tests to check all is good. 

I have created a set of scripts that perform the following:
<ul>
<li>runs application MSI to install assemblies and deploy into BizTalk MgmtDb</li>
<li>drops a test xml file into a receive location</li>
<li>waits a few seconds</li>
<li>checks an output file exists where expected</li>
</ul>

The above is a very simple list of steps but it does exercise many key aspects of the BizTalk platform, including MSDTC, SSO and configuration of the BizTalk databases.

In addition to testing the BizTalk platform, it's often helpful to know that access to required addresses and ports has been made available. So, in addition to the above, the test script can also receive an array of addresses for which it tests connectivity using the test-port PowerShell function described here: [technet blog post](http://blogs.technet.com/b/wsnetdoc/archive/2011/11/07/windows-powershell-script-test-port.aspx)


# Getting Started
After downloading a zip of my scripts from [here](www.biztalkers.com/blog-downloads/BizTalkSmokeTests.zip), unpack and you will find the following contents:

<table style="width:100%">
  <tr>
    <th>Filename</th>
    <th>Description</th>
  </tr>
  <tr>
    <td>SmokeTest.ps1</td>
    <td>Top level script and the only one you need run directly in order to execute the tests</td> 
  </tr>
  <tr>
    <td>BasicBTDFTest-1.0.0.msi</td>
    <td>Contains simple BizTalk app + bindings</td> 
  </tr>
  <tr>
    <td>DeployBizTalkApp.ps1</td>
    <td>Deploys the BizTalk application</td> 
  </tr>
  <tr>
    <td>Install-BizTalkApplication.ps1</td>
    <td>Referenced by DeployBizTalkApp.ps1</td> 
  </tr>
  <tr>
    <td>Test-Port.ps1</td>
    <td>SmokeTest.ps1</td> 
  </tr>
  <tr>
    <td>UndeployBizTalkApp.ps1</td>
    <td>Undeploys the app. May be run once satisfied the tests have completed successfully </td> 
  </tr>

</table>

## Installing The App
The app may be installed by running "DeployBizTalkApp.ps1" from an admin PowerShellWindow. The script is not signed so if it fails to run due the PowerShell policy, you can change this by running the following: 
> Set-ExecutionPolicy Unrestricted

After running the script, don't forget to change the policy back to a more secure setting, details can be found [here](https://technet.microsoft.com/library/hh847748.aspx)

###Unblock the DeployTools.dll
The deployment scripts are based on the excellent [BizTalk ALM](https://biztalkalm.codeplex.com/) codeplex offering. This includes a helper assembly called "DeployTools.dll". It may be that this needs to be *unblocked* before it can be used. In windows explorer, navigate to the DeployTools.dll, right-click and select properties. In the *General* tab click *Unblock* then click Ok.

### DeployBizTalkApp.ps1
You may now run the deploy script ".\DeployBizTalkApp.ps1"

Note: if you see the error: “Could not load file or assembly DeployTools.dll or one of its dependencies” this is most likely because the dll needs to be unblocked. Follow the instruction given above and also note you must restart any open PowerShell session before the unblocking will take effect.

This will create a new BizTalk application named *BasicBTDFTest* that contains a single receive location and a single send port. 

The deployment also creates the following folders for the receipt and sending of a test file:
<ul>
<li>c:\testing</li>
<li>c:\testing\inbox</li>
<li>c:\testing\outbox</li>
</ul>

# Running the Tests
The tests may be run by executing the PowerShell script *SmokeTest.ps1*

## SmokeTest.ps1
This script will accept the parameters listed below, although suitable defaults are used where a parameter is not provided from the command line:

<table style="width:100%">
  <tr>
    <th>Param Name</th>
    <th>Description</th>
    <th>Default Value</th>
  </tr>
  <tr>
    <td>rootTestFolder</td>
    <td>Root of the test folder hierarchy</td> 
    <td>c:\testing</td>
  </tr>
  <tr>
    <td>inbox</td>
    <td>Sub folder from which test files will be collected</td> 
    <td>inbox</td>
  </tr>
  <tr>
    <td>outbox</td>
    <td>Sub folder to which files will be sent</td> 
    <td>outbox</td>
  </tr>
  <tr>
    <td>testFile</td>
    <td>name of file to be copied</td> 
    <td>test1.xml</td>
  </tr>
  <tr>
    <td>testAddresses</td>
    <td>comma delimited list of addresses and ports to test</td> 
    <td>@("www.bbc.co.uk:80", "www.gmail.com:443")</td>
  </tr>
</table>

There are two distinct types of test that are run by the script: file drop and port test

### File Drop
The contents of the *outbox* folder are first deleted.

If a *testFile* parameter is received then this will be copied to the inbox folder. If not, then a simple *Test1.xml* file will be created and copied to the inbox folder.

The script will then wait 5 seconds before counting the number of files in the *outbox*

### Port-Test
The script will loop through each of the addresses received in the *testAddresses* parameter and check that connectivity is possible.

# Results
The results of the tests are written to a simple text file called *SmokeTestResults* which will be copied to the same folder as the *SmokeTest.ps1* script. An example results file is shown below: 
![](/images/biztalk-smoke-test/SmokeTestResults.png)


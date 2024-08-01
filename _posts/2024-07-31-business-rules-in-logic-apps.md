---
title: Hosting Business Rules Engine (BRE) in Logic Apps Standard
category: Azure
tags:
    - Logic Apps
    - BizTalk
---

## Overview
Microsoft recently announced the ability to execute BizTalk's business rules in Logic Apps Standard applications. At the time of writing, this feature is in public preview. 

## Research
I'm currently working on a project to migrate BizTalk applications to Azure Integration Services. These make use of the BRE, so we felt it worthwhile evaluating the option to migrate. I found this [youtube video](https://www.youtube.com/watch?v=YyHtar7JcoI) from Harold at Microsoft very helpful in describing the process.

## Key Points
+ Use the VS Code Azure extension to "Create new logic app workspace"
+ Select "Logic apps with rules engine project (preview)"

This will scaffold a new VS Code workspace for you that contains a LogicApp folder and a Functions folder.  

+ The Azure Function is a "Local Function" that runs in the same process as the logic app
+ It targets net472 framework to enable compatibility with the BizTalk BRE that has been ported into Microsoft.Azure.Workflows.RuleEngine.dll
+ The sample workflow created simply calls the local function, passing a hard-coded xml fact. The local function also instantiates a C# object as a fact and feeds these into a BRE ruleset
+ The csproj file of the local function contains MSBuild commands to copy the function binaries to the folder where the Logic App designer expects to find them. If you open the workflow in the designer and see an error on the action that calls the local function, try closing the designer, rebuilding the function (which will copy the binaries) and reopening the designer - you should then see the following:

![screenshot1](/images/business-rules-in-logic-apps/1.png)

## The Document Type
It can be seen from the previous screenshot that *DocumentType* is passed as a parameter from the workflow action to the local function. It is then used when executing the BRE as follows:

```cs
var typedXmlDocument = new TypedXmlDocument(documentType, doc);

// Initialize .NET facts
var currentPurchase = new ContosoNamespace.ContosoPurchase(purchaseAmount, zipCode);

// Provide facts to rule engine and run it
ruleEngine.Execute(new object[] { typedXmlDocument, currentPurchase });
```

In the example built from the VS Code template, the value of DocumentType is "SchemaUser". It is critical to ensure that this is correct, otherwise the BRE evaluation will fail without throwing any error.

One way to determine which value should be provided is to check the RuleSet.xml file that you need to place in the Artifact\Rules folder. An example from the sample is shown below - note the value for the __doctype__ property:

```xml
<xmldocument ref="xml_33" doctype="SchemaUser" instances="16" selectivity="1" instance="0">
        <selector>/*[local-name()='Root' and namespace-uri()='http://BizTalk_Server_Project1.SchemaUser']/*[local-name()='UserDetails' and namespace-uri()='']</selector>
        <selectoralias>/Root/UserDetails</selectoralias>
        <schema>C:\LogicAppsBRE_PrivatePreview_v1\Samples\schema\SchemaUser.xsd</schema>
      </xmldocument>
```

## Debug Tracking Interceptor
The BRE supports the use of a tracking interceptor which causes the rules engine to output debug information to a local file. This can be very helpful when trying to determine why a rule didn't fire as expected at runtime. During development time, the same troubleshooting can be achieved through the Microsoft Rules composer, which can be downloaded from [here](https://www.microsoft.com/en-us/download/details.aspx?id=106092).

## Sample Project
I have published a sample solution with a VS Code workspace containing an XUnit project. The local Azure function also contains a simple helper class that makes use of the Debug Tracking Interceptor to ensure tracking info is copied to blob storage. This can be found at this [GitHub Repo](https://github.com/BizTalkers/logicapps-bre) 
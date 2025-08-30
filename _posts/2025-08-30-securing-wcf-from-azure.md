---
title: Secure WCF Service When Calling from Azure
date: 2025-08-30
excerpt: Adding authentication
tags:
  - Azure
categories:
  - WCF
---

# A Reminder for Next Time
I was able to provide a pragmatic authorisation problem with week and though it worth documenting while I sit waiing for an AA recovery.

# Scenario
We have an existing on-prem WCF service that was previously called by BizTalk. The service endpoint was http (port only) with no authentication. We wanted to make the service available to an Azure Function app but tighten up on the security.

This has now been enhanced to give a single endpoint that:
 - demands an https connection
 - authenticates to ensure request is made by a specific Windows service account

To achieve this, changes have been made to the WCF Server
##### Changes to CCM WCF Server
The available binding was changed to give a single option that demands Transport security:

```xml
<bindings>
	<basicHttpBinding>
		<binding name="secureHttpsBasic" sendTimeout="00:05:00" >
			<security mode="Transport">
				<transport clientCredentialType="Basic" />
			</security>
		</binding>
	</basicHttpBinding>
</bindings>
```

Authentication and authorization elements were introduced:
```xml
<system.web>
	<authentication mode="Windows" />
	<authorization>
		<allow users="domain\service_account_name" />
		<deny users="*" />
	</authorization>
	<compilation debug="true" targetFramework="4.9" />
	<httpRuntime targetFramework="4.9" />
</system.web>
```

I think the ability to configure authentication and authorisation so simply is quite neat. This is saying I'll let you in only if you provide credentails fo the giving domain user.

##### Client Configuration

This works with the WCF client proxy that's generated from Visual Studio or svcuti.

````cs
var binding = new BasicHttpBinding
  {
      Security =
      {
          Mode = BasicHttpSecurityMode.Transport,
          Transport = { ClientCredentialType = HttpClientCredentialType.Basic }
      }
  };

  var client = new CaseWorkWcfClient(binding, new EndpointAddress(endpoint));
  client.ClientCredentials.UserName.UserName = options.Value.WcfUSername;
  client.ClientCredentials.UserName.Password = options.Value.WcfPassword;
````
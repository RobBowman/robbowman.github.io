---
title: Calling BizTalk from Azure
category: BizTalk
tags:
    - BizTalk
    - Azure
---
# Calling BizTalk from Azure
## Phone Home
I've been working on a project that has a .Net Core MVC front end deployed as an Azure Web App. This needs to request data that's currently available from an on-prem BizTalk 2016 environment.

There are many ways to achieve the required connectivity, I tried a few, which I'll describe in this post.

## WCF Relay
I'd used this previously on a project with a .Net 4.x framework site also deployed as an Azure Web App. From a BizTalk dev effort perspective, this is a great option because all that's required is configuration of a receive location using the *WCF-BasicHttpRelay* adapter. This then happily sits listening to the relay and any messages fired at it are collected.

The new client is being developed using .Net Core 2.0 so I thought I should be able to reference a .Net standard assembly that made the required SOAP call to the relay. I was able to make this work when I deployed a full .Net Framework app but had no luck with .Net Core. I was able to create a SOAP proxy from the WSDL but it refused to post the message to the proxy. After an hour or so looking into logs of the Azure Web App I thought, maybe better to avoid using SOAP / WCF for this one. 

## Using REST Instead
At this point, I created a new receive location for the same receive port but this time used the WebHttp adapter. I used the BizTalk WCF Service Publish Wizard. This was far less involved than a traditional publishing of schema for a SOAP endpoint,  because there is no schema to publish! The wizard also created a virtual directory / app in IIS but I needed to subsequently make name changes to both the receive location and IIS. Once I was happy with both, I exported the bindings and included the new receive location in my BTDF portbindgingsmaster.xml. I'm also using BTDF to deploy the website, so I created an entry in the deployment.btdfproj file to match my tinkered virtual directory. Ultimately, the only link between the receive location and IIS is stored in the BizTalk management databases, updated when the bindings are deployed. I have detailed how this works in a previous post: [linking-iis-to-receive-location/](https://biztalkersblog.azurewebsites.net/linking-iis-to-receive-location/)

## Azure Hybrid Connections
An alternative to WCF relay from Azure Web Apps is the use of Hybrid Connections. When I first started trying to work with this, I didn't have much luck because most of the documentation relates to configuration from the Service Bus Azure Portal page. When implemented from an Azure Web App, they are using the same thing but configuration is simpler. Rather than having to provide the clients with the SAS key to the relay, the hybrid connection are "added" to the Azure Web App, then any requests to the dns endpoint covered by an "added" connection can be satisfied automatically. The following page was a big help to me understanding this: [helpful microsoft page](https://docs.microsoft.com/en-gb/azure/app-service/app-service-hybrid-connections)

## Client Certificates
Another approach I tested was for an on-prem endpoint to be published to the internet via NetScalar. This demands a client certificate which the network team had provided to me - along with a password. I was able to ensure requests from the Azure Web App would be provided with the certificate by choosing "Upload Certificate" from the *Azure Web App\SSL Settings* page. This is described in detail at: [https://mehraban.com.au/2017/05/03/client-certificates-aspnetcore-azure-webapp/](https://mehraban.com.au/2017/05/03/client-certificates-aspnetcore-azure-webapp/)

This approach means there's no need for a hybrid connection manager (listener) to be running on the BizTalk server but comes with the overhead of having to create routing rules from the endpoint at NetScaler to the BizTalk server.


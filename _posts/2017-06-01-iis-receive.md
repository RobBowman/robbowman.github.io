# Publishing Website for BizTalk

I built a new solution today that used the BizTalk deployment framework to deploy everything - including a WCF service into IIS.

I created the service by using the publishing wizard and choosing to "publish schema". After completing the Wizard a new Virtual Directory could be found in my *inetpub\wwwroot* folder, I also elected for it to create a receive location in my application. 

The root of my virtual directory contains a generic .svc file with the following content:

    <%@ ServiceHost Language="c#" Factory="Microsoft.BizTalk.Adapter.Wcf.Runtime.BasicHttpWebServiceHostFactory, Microsoft.BizTalk.Adapter.Wcf.Runtime, Version=3.0.1.0, Culture=neutral, PublicKeyToken=31bf3856ad364e35" %>

As you can see, nothing in there to link to the BizTalk receive location. There's also nothing to be found in the associated web.config. So, on receipt of a request to this web service, how does IIS know which receive location should deal with it?

I've followed this process dozens of times over the years and because "it just works" I've never really considered how. Today I thought I'd take a short break to find out! The answer can be found in a *BindingInfo.xml* file that gets created in a *App_Data\Temp* sub folder that gets created below the *inetpub\wwwroot\<appname>* folder. In this file, I found the following content:

    <ReceivePort Name="WcfReceivePort_BizTalkCorporate/MvcForms/MvcForms" IsTwoWay="true">
      <ReceiveLocations>
        <ReceiveLocation Name="WcfService_BizTalkCorporate/MvcForms/MvcForms">
          <Address>/BizTalkCorporate/MvcForms/MvcForms.svc</Address>

It's important to note that this is just the config used at the time the Wizard published the Receive Location. The real link between BizTalk and IIS is actually stored in the BizTalk management database when the receive location is created. This can be changed via the BizTalk admin console, by clicking "Configure" for the receive location and updating the URI value, as shown in the following pic:

![](/images/iis-receive/blog1.png)
Note: The case of the IIS Virtual Dir (application) and Service Name must match exactly with IIS or else you'll find an error like *"Receive location for address xxx not found. (The BizTalk receive location may be disabled"* in the event log of the IIS server

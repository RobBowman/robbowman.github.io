# BizTalk and Tradeshift
Tradeshift is as cloud based service to enable companies to exchange business documents such as purchase orders and invoices more easily.

I was recently required to implement integration between an on-premise ERP system and Tradeshift using BizTalk 2013r2. Tradeshift offers a number of options for exchanging messages with its service, including secure FTP and a RESTful API. It was decided that we would make use of the API.

# Sending the Message
Many different message types are exchanged between the systems but for this post I will focus on the sending of a Purchase Order from the ERP to Tradeshift.

## Setting the URL
Getting the URL right is obviously critical to success when interacting with any REST style API. Fortunately, since version 2013r1 BizTalk has included the WCF based WebHttp adapter which makes setting the URL and injecting values into query string parameters relatively straight-forward.

The URL for sending our purchase order documents was as follows:

https://api-sandbox.tradeshift.com//tradeshift/rest/external/documents/dispatcher/?documentId=<guidhere>&documentProfileId=tradeshift.order

The item of particular interest in this is the "documentId" query string parameter. There was no need to link this GUID back to any value from the ERP, but it did need to be unique for each request.

Mapping of the parameter value into the query string was manged by the adapter, covered by two dialogue pages of the send port. In the first, a value for the URI and <BtsHttpUrlMapping> is given, as shown below:
![Transport Properties](/images/biztalk-tradeshift/Transport-properties.png)
Hightlighted in the screen clipping above is the value for the "Url" attribute of the <Operation> element. It can be seen that a variable called "DocumentId" was mapped as a query string parameter. The value of this variable was set from the 2nd dialogue, which was accessed by clicking the "Edit" button within the "Variable Mapping" section:
![Variable Mapping](/images/biztalk-tradeshift/variable-mapping.png)
This dialog enables the assignment of BizTalk context properties to the variable. In this case, a simple property schema was created to hold the GUID document Id.
## Authentication
Access to the Tradeshift API is controlled via OAuth1.0. This is not supported directly from the BizTalk adapter, so custom development was required. OAuth credentials are provided in the header of a HTTP Request, so there were a number of options for setting these from BizTalk, including the development of a custom pipeline component but I decided that a custom WCF behaviour would be a better fit.

I created a class called "TradeshiftRestAuthInspector" which implemented the interface "IClientMessageInspector". This interface provides the method "BeforeSendRequest", which as you may guess, is executed prior to the message being sent. It also makes the request message available as a reference parameter - ready to be read and manipulated by the custom code:

    public object BeforeSendRequest(ref System.ServiceModel.Channels.Message request, System.ServiceModel.IClientChannel channel)
    {
    TraceManager.ServiceComponent.TraceIn();
    
    string absoluteUri = request.Headers.To.AbsoluteUri;
    TraceManager.ServiceComponent.TraceInfo("Uri from adapter = " + absoluteUri);
    
    AddAuthHeader(request, absoluteUri);
    
    TraceManager.ServiceComponent.TraceOut();
    return null;
    }
The key call from the above method is to "AddAuthHeader". When creating an Oauth 1.0 authentication header, the complete URL, including query string parameters must be available. This is because Oauth 1.0 requires a signature, which is created from the hash of a concatenated string of many values - including the complete URL. Fortunately (actually I'm sure it was by design!), the URL available from "request.Headers.To.AbsoluteUri" includes any values mapped from BizTalk context properties as described in the previous section.

In order to generate the signature, I made use of an open source library available on GitHub at: https://gist.github.com/tsupo/112124. One thing to be aware of, this library uses System.Text.Encoding.ASCII whereas the instructions at http://nouncer.com/oauth/authentication.html state that UTF-8 should be used, so I made the change to System.Text.Encoding.UTF8.

## Adapter Configuration
Once the custom behaviour had been compiled and deployed, an entry was made to the x64 machine.config file (no 32bit hosts will be using the custom behaviour) as shown below:
![Machine Config](/images/biztalk-tradeshift/machine-config.png)
The new behaviour was then added to the adapter via the BizTalk admin console as shown below:
![TP1](/images/biztalk-tradeshift/tp1.png)
![TP2](/images/biztalk-tradeshift/tp2.png)

With the send port configured, any outbound messages were intercepted and had the required Authentication string added to the request header. This resulted in successful calls to the Tradeshift API and Purchase Order Documents visible in their portal.

### Acknowledgements
Richard Seroter wrote a great post on connecting to the Salesforce.com API from BizTalk using a Custom WCF Behavior (https://developer.salesforce.com/page/Calling_the_Force.com_REST_API_from_BizTalk_Server). Although, this uses OAuth 2.0 rather than the 1.0 of Tradeshift, much the of the action required for development of the custom behaviour are the same. I haven't duplicated the detail in this post.

I also found Johan Cooper's post on "Sharing Context between BizTalk and WCF behaviour extensions" helpful: https://adventuresinsidethemessagebox.wordpress.com/2015/07/01/sharing-context-between-biztalk-and-wcf-behavior-extensions/





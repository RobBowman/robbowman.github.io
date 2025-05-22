---
title: Using APIM to Transform SOAP to REST Call
category: APIM
tags:
    - APIM
    - SOAP
---

# Background
I had an existing REST API that was provided from Azure Function app that enabled the sending of emails. I also has a legacy web forms application that called a legacy SOAP service to send emails.

I needed to decommission the legacy SOAP email service. I also wanted to minimise change on the legacy web forms application. I was able to achieve this by deploying an APIM Policy that:

-  accepted the existing SOAP request
-  extracted the required data from the xml payload
-  composed json payload required by new REST service
-  called the REST service
-  converted response from REST service into XML response for the SOAP client
-  

# Implementation

## Source XML
A sample of the xml payload submitted from the legacy web forms application can be seen below:

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:xxx="http://xxx.Integration.Common.Schemas.Email.MarkdownEmailEnvelope">
	<soapenv:Header/>
	<soapenv:Body>
		<xxx:MarkdownEmails>
			<!--1 or more repetitions:-->
			<xxx:MarkdownEmail>
				<Settings>
					<DeliveryReceipt>false</DeliveryReceipt>
					<ReadReceipt>false</ReadReceipt>
					<From>rob.bowman@leeds.gov.uk</From>
					<To>rob@biztalkers.com</To>
					<!--Optional:-->
					<CC></CC>
					<Subject>test email</Subject>
					<!--Optional:-->
					<Attachments></Attachments>
				</Settings>
				<Body>test body</Body>
			</xxx:MarkdownEmail>
		</xxx:MarkdownEmails>
	</soapenv:Body>
</soapenv:Envelope>
```

## Target JSON
A sample of the payload request for the REST service:

```json
{
    "requestConfig": {
        "tags": [
            "tag1",
            "tag2"
        ],
        "businessArea": "xxx",
        "callingSystem": "yyy",
        "callingSystemRef": "cts006475",
        "targetEmailAddress": "info@biztalkers.com"
    },
    "templateParameters": {
        "Subject": "Sample Subject",
        "Body": "Sample Body"
    }
}
```

## APIM Policy
APIM Policy to Transform and Route:

```xml
<policies>
    <inbound>
        <!-- Always treat the request as SOAP -->
        <rewrite-uri template="/" copy-unmatched-params="false" />
        <!-- Extract values from the envelope -->
        <!-- Note: cannot assign XElement to variable outside of C# block because set-variable only supports native types -->
        <set-variable name="to" value="@{
                      var doc = context.Request.Body.As&lt;XElement&gt;(true);
                      var el  = doc.Descendants()
                                   .FirstOrDefault(e =>
                                       string.Equals(e.Name.LocalName, "To",
                                                     StringComparison.OrdinalIgnoreCase));
                      return el?.Value ?? string.Empty;
                  }" />
        <set-variable name="from" value="@{
                      var doc = context.Request.Body.As&lt;XElement&gt;(true);
                      var el  = doc.Descendants()
                                   .FirstOrDefault(e =>
                                       string.Equals(e.Name.LocalName, "From",
                                                     StringComparison.OrdinalIgnoreCase));
                      return el?.Value ?? string.Empty;
                  }" />
        <set-variable name="subject" value="@{
                      var doc = context.Request.Body.As&lt;XElement&gt;(true);
                      var el  = doc.Descendants()
                                   .FirstOrDefault(e =>
                                       string.Equals(e.Name.LocalName, "Subject",
                                                     StringComparison.OrdinalIgnoreCase));
                      return el?.Value ?? string.Empty;
                  }" />
        <set-variable name="body" value="@{
                      var doc = context.Request.Body.As&lt;XElement&gt;(true);
                      var el  = doc.Descendants()
                                   .FirstOrDefault(e =>
                                       string.Equals(e.Name.LocalName, "Body",
                                                     StringComparison.OrdinalIgnoreCase));
                      return el?.Value ?? string.Empty;
                  }" />
        <!-- Build JSON payload for the backend -->
        <set-body template="liquid">
{
  "requestConfig": {
    "tags": ["form to email"],
    "businessArea": "CouncilTax",
    "callingSystem": "CTaxRebate-Credit",
    "callingSystemRef": "na",
    "targetEmailAddress": "{{ context.Variables.to | json }}"
  },
  "templateParameters": {
    "Subject": "{{ context.Variables.subject | json }}",
    "Body":    "{{ context.Variables.body   | json }}"
  }
}
    </set-body>
        <set-header name="Content-Type" exists-action="override">
            <value>application/json</value>
        </set-header>
        <set-query-parameter name="api-version" exists-action="override">
            <value>v1.0.0</value>
        </set-query-parameter>
        <set-backend-service base-url="{{BackendUrl}}" />
        <set-header name="x-functions-key" exists-action="override">
            <value>{{AzFunctionKey}}</value>
        </set-header>
        <base />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <!-- Re-wrap backend response in SOAP -->
        <set-body template="liquid">
			<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema">
				<soap:Body>
					<SendEmailResponse xmlns="http://tempuri.org/">
						<SendEmailResult>
							<Success>{{context.Response.StatusCode == 200}}</Success>
							<MessageId>{{body.serviceRef}}</MessageId>
							<Message>Email sent successfully</Message>
						</SendEmailResult>
					</SendEmailResponse>
				</soap:Body>
			</soap:Envelope>
		</set-body>
        <set-header name="Content-Type" exists-action="override">
            <value>text/xml</value>
        </set-header>
        <base />
    </outbound>
    <on-error>
        <!-- Produce SOAP fault on any failure -->
        <set-body template="liquid">
			<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
				<soap:Body>
					<soap:Fault>
						<faultcode>soap:Server</faultcode>
						<faultstring>{{context.LastError.Message}}</faultstring>
						<detail>
							<ErrorCode>{{context.Response.StatusCode}}</ErrorCode>
						</detail>
					</soap:Fault>
				</soap:Body>
			</soap:Envelope>
		</set-body>
        <set-header name="Content-Type" exists-action="override">
            <value>text/xml</value>
        </set-header>
        <base />
    </on-error>
</policies>
```
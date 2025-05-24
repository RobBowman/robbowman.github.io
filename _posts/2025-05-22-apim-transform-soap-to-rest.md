
---
title: Using APIM to Transform a SOAP Request into a REST Call
date: 2025‑05‑22
excerpt: |
  Walk‑through of how Azure API Management can breathe new life into legacy SOAP clients by
  translating their XML envelopes into modern JSON REST requests – without touching the client
  or the backend.
tags:
  - APIM
  - SOAP
  - REST
  - Migration
categories:
  - APIM
---

> **TL;DR**  
> With a single APIM policy we can accept a legacy SOAP request, map the payload to the JSON
> expected by a new REST endpoint, call the backend, and wrap the response back into the
> original SOAP envelope – allowing us to retire a WCF service without changing a line of client code (just the URL it targets).

## Why do this?

* **Cost & maintenance** – Retire on‑prem IIS + WCF service and associated VMs.  
* **Developer experience** – Expose a modern REST endpoint for future consumers.  
* **Risk mitigation** – Legacy web‑forms app keeps working exactly as is during the migration window.  

You can treat APIM as an **adapter** sitting between the old world (SOAP) and the new
world (REST), giving you freedom to evolve each side independently.

### Flow at a glance

1. **SOAP client** posts the original envelope to `/sendEmail/soap`  
2. **Inbound policy** parses the XML, assigns variables, and builds the JSON body  
3. **rewrite‑uri** and **set‑backend‑service** forward the request to the Azure Function  
4. **Outbound policy** re‑wraps the JSON response into a SOAP envelope  
5. **Client** receives a response that is *bit‑for‑bit* compatible with the original WSDL

:::mermaid
sequenceDiagram
    participant Client as SOAP Client
    participant Gateway as API Gateway
    participant InboundPolicy as Inbound Policy
    participant URIRewrite as Rewrite-URI Policy
    participant BackendSet as Set-Backend-Service Policy
    participant Function as Azure Function
    participant OutboundPolicy as Outbound Policy

    Client->>Gateway: POST /sendEmail/soap (SOAP Envelope)
    Gateway->>InboundPolicy: Process request
    Note over InboundPolicy: Parse XML<br>Extract data<br>Assign variables<br>Build JSON body
    InboundPolicy->>URIRewrite: Forward with JSON payload
    Note over URIRewrite: Rewrite URI for backend
    URIRewrite->>BackendSet: Forward request
    Note over BackendSet: Configure backend service
    BackendSet->>Function: POST request with JSON body
    Function-->>BackendSet: Return JSON response
    BackendSet-->>OutboundPolicy: Forward JSON response
    Note over OutboundPolicy: Transform JSON<br>Re-wrap in SOAP envelope<br>Ensure WSDL compatibility
    OutboundPolicy-->>Gateway: SOAP response
    Gateway-->>Client: Return bit-for-bit WSDL compatible response
:::

---

## Source XML

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
                  xmlns:xxx="http://xxx.Integration.Common.Schemas.Email.MarkdownEmailEnvelope">
  <soapenv:Header/>
  <soapenv:Body>
    <xxx:MarkdownEmails>
      <xxx:MarkdownEmail>
        <Settings>
          <DeliveryReceipt>false</DeliveryReceipt>
          <ReadReceipt>false</ReadReceipt>
          <From>rob.bowman@example.com</From>
          <To>info@biztalkers.com</To>
          <CC></CC>
          <Subject>Sample Subject</Subject>
          <Attachments></Attachments>
        </Settings>
        <Body>Sample body</Body>
      </xxx:MarkdownEmail>
    </xxx:MarkdownEmails>
  </soapenv:Body>
</soapenv:Envelope>
```

## Target JSON

```json
{
  "requestConfig": {
    "tags": ["form to email"],
    "businessArea": "CouncilTax",
    "callingSystem": "CTaxRebate‑Credit",
    "callingSystemRef": "na",
    "targetEmailAddress": "info@biztalkers.com"
  },
  "templateParameters": {
    "Subject": "Sample Subject",
    "Body": "Sample Body"
  }
}
```

---

## Detailed walkthrough

### 1. Import or create a SOAP‑style operation

Even though the backend is REST, create a SOAP‑pass‑through API or import the original WSDL so
APIM will generate a convenience test console and standard SOAP fault shapes.

### 2. Add a transformation policy

Insert the following policy at **operation scope** (or at API scope if every operation in
this API needs the same treatment):

```xml
<policies>
  <inbound>
    <!-- 1️⃣ Force SOAP mode so APIM does not try to treat this as XML‑encoded REST -->
    <rewrite-uri template="/" copy-unmatched-params="false" />
```

`rewrite-uri` short‑circuits any templated routing and sends the request to the
*root* of this operation, which makes life easier when the original SOAP action
contains quirky URL suffixes or query parameters.

```xml
    <!-- 2️⃣ Extract values from the envelope -->
    <set-variable name="to" value="@{
        var doc = context.Request.Body.As<XElement>(true);
        return doc.Descendants().First(e => e.Name.LocalName == "To").Value;
    }" />
    <set-variable name="subject" value="@{
        var doc = context.Request.Body.As<XElement>(true);
        return doc.Descendants().First(e => e.Name.LocalName == "Subject").Value;
    }" />
    <set-variable name="body" value="@{
        var doc = context.Request.Body.As<XElement>(true);
        return doc.Descendants().First(e => e.Name.LocalName == "Body").Value;
    }" />
```

### Why C# blocks?

`set-variable` only supports primitive types outside a C# block. Wrapping the extraction
logic in `@{ … }` lets us leverage LINQ‑to‑XML for concise selection.

### 3. Build the JSON payload

```xml
    <set-body template="liquid">
{
  "requestConfig": {
    "tags": ["form to email"],
    "businessArea": "CouncilTax",
    "callingSystem": "CTaxRebate‑Credit",
    "callingSystemRef": "na",
    "targetEmailAddress": "{{to}}"
  },
  "templateParameters": {
    "Subject": "{{subject}}",
    "Body": "{{body}}"
  }
}
    </set-body>
    <set-header name="Content-Type" exists-action="override">
      <value>application/json</value>
    </set-header>
    <set-backend-service base-url="https://myemailfunc.azurewebsites.net/api/send" />
    <set-header name="x-functions-key" exists-action="override">
      <value>{{backend-key}}</value>
    </set-header>
  </inbound>
  <backend>
    <base />
  </backend>
```

### 4. Craft the outbound SOAP response

```xml
  <outbound>
    <set-body template="liquid">
      <soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
        <soap:Body>
          <SendEmailResponse xmlns="http://tempuri.org/">
            <SendEmailResult>
              <Success>{{body.success}}</Success>
              <MessageId>{{body.id}}</MessageId>
              <Message>{{body.message}}</Message>
            </SendEmailResult>
          </SendEmailResponse>
        </soap:Body>
      </soap:Envelope>
    </set-body>
    <set-header name="Content-Type" exists-action="override">
      <value>text/xml</value>
    </set-header>
  </outbound>
  <on-error>
    <!-- Produce SOAP fault on any failure -->
    <set-body template="liquid">
      <soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
        <soap:Body>
          <soap:Fault>
            <faultcode>soap:Server</faultcode>
            <faultstring>{{body.message}}</faultstring>
            <detail/>
          </soap:Fault>
        </soap:Body>
      </soap:Envelope>
    </set-body>
    <set-header name="Content-Type" exists-action="override">
      <value>text/xml</value>
    </set-header>
  </on-error>
</policies>
```

---

## Testing

| Tool            | What to look for                                                                 |
|-----------------|----------------------------------------------------------------------------------|
| **APIM Trace**  | Confirm variables contain the expected values and the JSON body is well‑formed. |
| **Postman**     | Use the **SOAP** tab and import the original WSDL for convenience.               |
| **Application Insights** | Ensure any `Exception` traces are propagated back as SOAP faults.      |

### Troubleshooting tips

* If the JSON body arrives empty, double‑check that `preserveContent` is **not** set when you
  call `set-body`.  
* XML namespaces are case‑sensitive; if extraction fails, inspect the envelope with
  `context.Request.Body.As<string>(true)` in **Trace**.

---

## Security considerations

* **Request size** – Add a `validate-content` policy to reject payloads over an expected size.  
* **Input validation** – Consider XML schema validation (`<validate-content>`) to
  prevent malicious input.  
* **Backend throttling** – Protect downstream functions with an exponential retry policy
  or a circuit breaker to avoid cascading failures.

---



_Updated: 2025-05-24_

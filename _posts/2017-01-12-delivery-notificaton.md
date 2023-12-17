# BizTalk Orchestration Delivery Notifcation
I ran into an interesting problem last week. Sends to a third-party via the FTP adapter were failing. Nothing particularly interesting in that, the cause was some configuration on one of the firewalls. However, my client's support team only became aware of the problem when the third-party had called them to ask why they hadn't received any messages that day - not good!

In the event of a failed send, the support team should have received an email from BizTalk. I was called to figure out why said email was not sent. On opening the solution, I could see a main scope and a single exception handler for "General Exceptions". This exception handler contained an expression shape which would create an exception message for the ESB Toolkit's exceptions database, and a send port to  call the stored proc which would make the insert.

From the BizTalk admin console, I could see an SMTP send port which subscribed to the ESB Exception message. The filter looked good, so why was the alert email never sent? The problem was, the "Delivery Notification" property of the orchestration's one-way send port was set to "None".

Once I changed this property value to "Transmitted", then failure of the physical FTP send port did result in an exception being raised to the orchestration. I also took the precaution of wrapping the called with a dedicated scope, as shown in the following screenshot.

![odx image](/images/delivery-notification/DeliveryNotification.png)
# Fiddler and BizTalk with a Corp Proxy
I've been having fun integrating BizTalk with an external rest style api this week. When things don't work as expected, it's great to be able to see the traffic in Fiddler. Postman has a really nice user interface for working with apis and so, before configuring the BizTalk send ports for the api, I've found exploring it by firing tests from Postman while monitoring through Fiddler to be really helpful. However, when working behind a corporate proxy there's usually extra config required to make this possible.

These are the steps I had to take this week.

* Set Postman to fire at Fiddler. Note: value for proxy server could be **"localhost"** (depends on version of Postman)

![postman proxy](/images/biztalk-fiddler/Postman1.png)

* Set proxy on BizTalk send port to fire at Fiddler

![Biz proxy](/images/biztalk-fiddler/Biz1.png)

* Configure Fiddler to fire at Corporate Proxy (note: Fiddler authenticates with the corporate proxy using AD Authentication)

![fiddler to corp](/images/biztalk-fiddler/Fiddler1.png)

* Install Fiddler Cert

![fiddler cert](/images/biztalk-fiddler/Fiddler2.png)

Note: before getting this to work, I was receiving 403 unauthorised exceptions back. It turned out it was the corporate proxy returning these. The network team fixed by adding a rule enabling my dev vm to call the particular url of the api.
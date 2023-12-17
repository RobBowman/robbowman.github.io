# Zero to Cloud
I have dabbled with Azure over the past few years now but this has been mainly as a host to a simple websites with a sql db layer. I've been meaning to investigate what else Azure has to offer but never managed to find the time. So, when a few weeks ago I heard about a course to coach through building and deploying an app to Azure in one day, I signed up - [Zero to cloud in 1 day](http://theazurecoach.com/courses/zero-to-cloud-in-1-day/)

# Content
The aim of the course was to build and deploy a website, simples! However, what drew me in was that we would cover subjects such as AD Authentication and Continuous Deployment.

I'm a big believer in continuous integration and one of my first tasks in a new role is to ensure an appropriate level of automation is in place. I've been doing this for number of years in the on-premise world so I now have my preferred tools and custom Powershell scripts etc. However, I had wondered how I would deliver similar functionality for Azure based solutions. Thankfully, the course did a great job of demystifying this for me.

# Application Insights
Application Insights was a feature of Azure I'd not previously given much thought to but I've been mulling over the possibilities ever since! In my BizTalk solutions, I currently employ the CAT Framework for diagnostic instrumentation, with trace statements spread liberally throughout my custom pipeline components, orchestrations and helper classes. This works well but needs to be enabled before the diagnostics can be captured.

For the tracking of business data I use currently use BAM, usually through the API and Darren Jefford's type generator but occasionally using only the TPE if sufficient. This has also worked very well for me and over the years I've created a custom BAM Portal (now built around MVC and Entity Framework) which provides a neat UI onto the business data as well as the option to view and download message payloads. Now, on beginning the course I didn't think the CAT Framework or BAM were in any danger of been dropped from my BizTalk tool belt, but now I'm not so sure! You see the thing is, I think Application Insights can maybe fill the role of both, with a couple of clear advantages:
<ul>
  <li>
    In the case of instrumentation for fault finding, there's no need to wait for a problem before capturing (it could be argued that the CAT framework could be set to continuously trace but where to keep the log files and how easy would it be to find the data you need?)
  </li>
  <li>
In the case if activity monitoring, unlike BAM, Application Insights does not require an activity definition be created ahead of time. It's often not obvious as to the transaction data that will prove particularly valuable to the business. The approach of capturing everything and "discovering" the value through tools such as Power BI is compelling.
  </li>
</ul>


# Happy :)
The course was developed and delivered by Mike Stephenson, a name that will be familiar to anyone from the BizTalk community. 

Overall, I was delighted with the course. Since my first Pluralsight subscription over 3 years ago I had come to the conclusion that I learn more from taking a day out and working through a course from favourite authors (Allen, Wildermuth, Lerman et al). However, the Zero to Cloud course is only 1 day so it's not such a major investment. With so much material covered and Mike's wealth of experience on tap I'd say it's time well spent.
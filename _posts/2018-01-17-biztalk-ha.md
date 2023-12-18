---
title: Biztalk 2016 High Availability Config
category: BizTalk
tags:
    - BizTalk
    - Sql Server
---
# BizTalk 2016 High Availability Config
I was recently faced with creating a new on-prem production environment for BizTalk 2016. New to me for this install was that high-availability (HA) of the BizTalk databases had to be enabled using Sql Server 2016 Always On Availability Groups.

Availability groups have been a feature of Sql Server since 2012 but the option to use them with BizTalk only became a supported option following the release of BizTalk 2016.

Microsoft have created guidance on how / where BizTalk databases should be created when using AlwaysOn, this is available at: https://docs.microsoft.com/en-us/biztalk/core/high-availability-using-sql-server-always-on-availability-groups. Microsoft's Samuel Kastberg also posted a log of his installation experience: https://skastberg.wordpress.com/2017/02/22/setting-up-my-biztalk-server-2016-using-availability-groups-lab/ . I found this very helpful, there's lots of detail which I won't attempt to duplicate in this post.

From this doc, you can see their recommended server config as follows:
![microsoft rec](/images/biztalk-ha/sqlag-recommended.png)

I won't go into why it has to be so complicated but it seems our old friend MSDTC has a lot to answer for!

Complexity aside, there's also the question of cost. You can see from the pic that there are **nine** "Nodes" - this equates to nine Sql Server licenses! Having read through the docs, we felt it should be possible to meet our HA requirements using just five licenses of Sql Server 2016 Standard Edition - as illustrated below:

![biztalkers config](/images/biztalk-ha/clusterConfig.png)

The eagle-eyed will notice that there are a few differences in the databases being deployed between my pic and Microsoft's. We had no need for BAM Analysis or Alerts but we do have custom databases for Message Archiving and Cross Reference. We also make extensive use of the ESB Toolkit's Exception Framework, so it's important for the ESB Exception DB to also be highly available.

###SSO
The BizTalk Configuration application is used to create the SSO db and configure the "Enterprise Single Sign-on" windows service. In a dev / test environment, this is typically run only from the BizTalk application server. However, in our case, we wanted to remove this potential single point of failure and so chose to install the "Master Secret" SSO Service onto one of the Sql Server servers, then add as a Windows Server clustered application. Installation of the "Master Secret" is exactly the same as for any instance of the SSO service. This is done from the BizTalk Configuration application as shown in the following screenshot:

![sso install](/images/biztalk-ha/sso_install-1.png)

Note: when creating the SSO db from the BizTalk config wizard, I found that I had to run this from the node that was NOT the primary at the time. If I tried to create from the primary node then I received then the BizTalk config wizard failed and presented the following error in the logfile: *The dependency service does not exist or has been marked for deletion*

####Adding the Cluster Role for SSO
On the 2nd Sql Server, I loaded the windows app "Failover Cluster Manager". From here, I right-clicked the "Roles" branch of the tree-view on the left of the screen and selected "Configure Role". This launched a "High Availability Wizard". On the second page, I chose "Generic Service" to "configure for high availability". On the next page I selected "Enterprise Single Sign On Service" from the list of services. In contrast with a cluster for MSDTC, no disk allocation is required for the SSO service.

#####Sequencing Problem

The name of the SSO master secret server should be set to the name of the clustered role rather than a server name. This removes the single point of failure because if a "client" SSO service running on any of the BizTalk app servers requests data from the "Master secret" sso service, then this can be served from any node in the cluster.

The problem is, the cluster cannot be created until the service has been installed - and the name of the master secret server is typically given when created through the BizTalk configuration wizard. The work around for this is to install the SSO service giving the AG Lister Name and port, then later change the master secret name using the "SSOManage" command line tool, as described here: https://docs.microsoft.com/en-us/biztalk/core/how-to-change-the-master-secret-server

##Limiting the Nodes the Services Run On
It can be seen from the design diagram earlier in this post that we have four servers in the cluster, however, I wanted to limit where SSO could run to just two of these. The way this can be achieved I found to be quite hidden, but it can be done as follows:

+ Load Failover Cluster Manager 
+ Select the previously created SSO role
+ From the bottom of the page select the "Resources" tab
+ Right-click the name of the service and select properties
+ From the properties dialog, select the "Advanced Policies" tab
+ You're then presented with a check-box list of "Possible Owners"

####Addressing the SSO Database
From the previous screenshot, you can see that for the SSO Database, I gave a server name of 'ag\_ltnr01,5101'. This is a reference to an always-on activity group listener that was previously configured to listen on a particular IP address (for which the DNS record ag_ltnr01 was set) on port 5101. I found it quite strange that the port number needed to separated from the IP address with a comma rather than a colon, but I won't lose sleep over it.

##Microsoft Distributed Transaction Coordinator (MSDTC)
MSDTC is a key dependency for BizTalk because it is used to provide atomic transactions that span multiple databases e.g. the Tracking and MessageBox databases. MSDTC runs as a windows service and can be administered from the "Component Services" snap-in, run "dcomcnfg" from the Start menu, details available at: https://msdn.microsoft.com/en-us/library/aa544733%28v=cs.70%29.aspx?f=255&MSPPError=-2147217396

###Clustering MSDTC
For this, I initially followed the same process as for clustering SSO except that when selecting the role to be configured I chose "Distributed Transaction Coordinator (DTC)" rather than "Generic Service". Also, MSDTC does require an allocation of disk space, so this had to be identified in a later wizard step. However, I then found that the BizTalk configuration would fail when trying to install the Runtime. It would give the following error: *"Could not import a DTC transaction. Please check that MSDTC is configured correctly for remote operation. See the event log (on computer 'SSO') for more details."* After reading the following blog post I decided clustering of the MSDTC service was no longer required: https://blogs.msdn.microsoft.com/alwaysonpro/2014/01/15/msdtc-recommendations-on-sql-failover-cluster/ . Once I'd deleted the MSDTC cluster then I was able to install the BizTalk runtime from the BizTalk config tool.

Next job; test failing over during load and otherwise trying to break things :)







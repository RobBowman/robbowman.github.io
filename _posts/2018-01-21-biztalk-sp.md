---
title: Calling a Stored Proc from BizTalk without an Orchestration
category: BizTalk
tags:
    - BizTalk
    - Sql Server
---
# Calling a Stored Proc from BizTalk without Orchestration

Earlier this week, I had to call a stored procedure from a simple BizTalk 2016 applicaition. The stored procedure expected to receive 7 parameters, each of which would be available in the BizTalk message.

    ALTER PROCEDURE [dbo].[UpsertCarInventory] 

	@year int, 
	@make varchar(40),
	@model varchar(40),
	@color varchar(10),
	@mileage int,
	@inLot bit,
	@regPlate varchar(2)

It's been a while since I've had to do this so I quickly googled for the best approach. All the results suggested that I need an orchestration to call the stored proc. This seemed a little heavyweight for my solution, so I figured out how to make it work messaging only.

First I right-clicked my BizTalk schemas project and selected Add\Generated Items\Add Adapter Metadata. This launched the "Add Adapter Wizard". On the "Select Adapter" page, I chose "WCF-SQL" and selected the database hosting the stored proc. I then expanded the branch for "Strongly-Types Procedures" from the tree view and selected to "Add" the name of my "UpsertCarInventory" stored proc, as shown below:

![ConsumeAdapterPage](/images/biztalk-sp/consumeAdapter.png)

The wizard then created a schema for me to map to before sending the request to Sql Server. In this case, a simple map was required, as shown below:

![Map](/images/biztalk-sp/map.png)

In the BizTalk admin console, I then created a two-way send port where I gave the Action as "Procedure/dbo/UpsertCarInventory" - as shown below:
![SoapAction](/images/biztalk-sp/soapAction.png)

On running a test, I found that the stored proc was being called ok from BizTalk but only data from a single row of the source data was being passed. I solved this by updating the schema of the source data. This was actually being selected from a different SQL Server database and is shown below:
![SelectSchema](/images/biztalk-sp/selectSchema.png)

I selected the "Schema" and chose "Yes" from the properties window for "Envelope". I then selected the SelectResponse node and clicked the ellipses for the "Body XPath" property. From here, I drilled down to the SelectResponse\SelectResult node. This meant that on receipt of source data such as the one shown below, it would be debatched at each CarInventory node:

![SampleSelectResult](/images/biztalk-sp/SampleSelectResult.png)




	

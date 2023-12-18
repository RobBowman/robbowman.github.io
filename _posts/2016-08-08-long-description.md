---
title: ESB Exception 256
category: BizTalk
tags:
    - BizTalk
    - ESB Toolkit
---
# ESB Exception Framework and Description 256 Length Problem
Users of the ESB Exception Toolkit may be familiar with the exception:
>The property "Description" has a value with length greater than 256 characters

I was finding messages suspended with this exception following a failed call to a web service. Strange thing was, the failure was being correctly reported in the ESB Exceptions database.

After investigation, I created the following pic to give a rough idea of what I think causes the exception:

![sequence](/images/long-description/sequence.png)

Note, I have Failed Message Routing (FMR) enabled on the physical 2-way send port. This is a standard approach for me, since I have a custom BAM portal that gives a view over this ESB Exceptions database, raises alerts and enables message re-submission.

The problem was with the way I had the exception handling configured in my orchestration. I had been lazy and was relying on a single scope shape with a generic exception handler for type System.Exception:

![single ex handler](/images/long-description/Ex-Handler.png)

My solution was to have dedicated scope shapes, with exception handlers, around the calls to the web service. Within these exception handlers I can decide if creation of a new fault message is necessary, and if not, simply terminate the orchestration instance. This will often be a reasonable action, since the problem will have already been published to the Exceptions database via the FMR process.
---
title: DevOps Error - The content for this response was already consumed
date: 2025-08-21
excerpt: The content for this response was already consumed caused by missing parameter
tags:
  - DevOps
categories:
  - DevOps
---

# A Terrible Error Message
I was working with a new DevOps pipeline yesterday. When I ran, it gave the following error:

> The content for this response was already consumed

Neither Google or the LLMs had any ideas on this, so I'm posting here to save future Rob and others from the further swearing at their keyboards.

# Root Cause
Simple - the pipeline was attempting to pass parameters to the bicep that did not exist in the bicep. Easy mistake to make when creating new pipelines & biceps from existing files, so tis a shame the error message is so vaugue.  
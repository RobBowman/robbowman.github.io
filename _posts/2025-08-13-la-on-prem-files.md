---
title: Thoughts from Integrate 2025 Conference
date: 2025-08-13
excerpt: |
    Connecting to on-prem files withouth the OPDG or ASEv3
tags:
  - Logic Apps
categories:
  - Logic Apps
---

# Goodbye (good riddance) to the On Prem Data Gateway (OPDG)
You're probably aware that logic app standrard workflows have the option to use built-in or "managed" connectors. Until recently, except for the case of ASEv3 hosted logic apps, where a workflow required access to an on-prem file the only option was to use the managed connector.

This has a couple of downsides:
+ configuration is complex because an apic needs to be deployed alongside the logic app
+ there is a small cost each this the connector is used

## 

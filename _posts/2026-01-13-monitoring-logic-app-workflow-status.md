---
title: Monitoring Active Logic App Workflow Runs with PowerShell
category: Azure
tags:
    - Logic Apps
    - PowerShell
    - Monitoring
---

## Overview
When managing Azure Logic Apps Standard instances with multiple workflows, it's often necessary to check which workflows have active running instances. This is particularly useful during deployment windows, maintenance activities, or when troubleshooting performance issues. The Azure Portal provides individual workflow monitoring, but checking multiple workflows manually becomes tedious and time-consuming.

## The Challenge
In enterprise environments, Logic App Standard instances can contain dozens of workflows. Before performing maintenance tasks, deploying updates, or investigating resource consumption, you need to know:

+ Which workflows are currently processing messages
+ How many active runs each workflow has
+ The overall health status of each workflow
+ Whether it's safe to proceed with planned maintenance

Manually checking each workflow through the Azure Portal is inefficient and prone to oversight.

## The Solution
I created a PowerShell script that leverages the Azure REST API to query all workflows in a Logic App Standard instance and check for running instances. The script provides both real-time console output and a detailed summary table, making it easy to get a quick overview of the entire Logic App's operational status.

## Key Features
+ **Automated Discovery**: Automatically retrieves all workflows in the specified Logic App
+ **Running Instance Detection**: Checks each workflow for currently executing runs
+ **Health Status**: Displays the state and health status of each workflow
+ **Color-Coded Output**: Uses visual indicators to highlight active workflows
+ **Summary Statistics**: Provides totals for running, idle, and total workflows
+ **Detailed Table Output**: Displays all workflow information in an easy-to-read format

## The Script

```powershell
#!/usr/bin/env pwsh
<#
.SYNOPSIS
    Checks the running status of all workflows in a Logic App Standard instance.

.DESCRIPTION
    This script queries all workflows in the specified Logic App and checks if any have 
    currently running instances. It uses Azure REST API to get workflow status.

.PARAMETER SubscriptionId
    The Azure subscription ID

.PARAMETER ResourceGroup
    The resource group name

.PARAMETER LogicAppName
    The Logic App name

.EXAMPLE
    .\Check-WorkflowStatus.ps1

.EXAMPLE
    .\Check-WorkflowStatus.ps1 -SubscriptionId "your-sub-id" -ResourceGroup "your-rg" -LogicAppName "your-la"
#>

param(
    [string]$SubscriptionId = "00000000-0000-0000-0000-000000000000",
    [string]$ResourceGroup = "rg-logicapp-prod-001",
    [string]$LogicAppName = "la-integration-prod-001"
)

Write-Host "Checking workflow running status for Logic App: $LogicAppName" -ForegroundColor Cyan
Write-Host "================================================" -ForegroundColor Cyan
Write-Host ""

# Get all workflows
$workflowsUri = "/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroup/providers/Microsoft.Web/sites/$LogicAppName/workflows?api-version=2022-03-01"

Write-Host "Fetching workflows..." -ForegroundColor Yellow

$workflows = az rest --method get --uri $workflowsUri | ConvertFrom-Json

if (-not $workflows.value) {
    Write-Host "No workflows found or error occurred." -ForegroundColor Red
    exit 1
}

Write-Host "Found $($workflows.value.Count) workflows`n" -ForegroundColor Green

$results = @()

foreach ($workflow in $workflows.value) {
    $workflowName = $workflow.name.Split('/')[-1]
    
    # Check for running instances
    $runsUri = "https://management.azure.com/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroup/providers/Microsoft.Web/sites/$LogicAppName/hostruntime/runtime/webhooks/workflow/api/management/workflows/$workflowName/runs?`$filter=status eq 'Running'&api-version=2018-11-01"
    
    $runs = az rest --method get --url $runsUri 2>$null | ConvertFrom-Json
    
    $runningCount = 0
    if ($runs.value) {
        $runningCount = $runs.value.Count
    }
    
    $status = if ($runningCount -gt 0) { "RUNNING" } else { "Idle" }
    $state = $workflow.properties.flowState
    $health = $workflow.properties.health.state
    
    $result = [PSCustomObject]@{
        Workflow         = $workflowName
        Status           = $status
        RunningInstances = $runningCount
        State            = $state
        Health           = $health
    }
    
    $results += $result
    
    # Real-time output
    $color = if ($runningCount -gt 0) { "Green" } else { "Gray" }
    Write-Host "  $workflowName`: " -NoNewline
    Write-Host "$status" -ForegroundColor $color -NoNewline
    if ($runningCount -gt 0) {
        Write-Host " ($runningCount active runs)" -ForegroundColor $color
    }
    else {
        Write-Host ""
    }
}

Write-Host ""
Write-Host "Summary:" -ForegroundColor Cyan
Write-Host "--------" -ForegroundColor Cyan
$runningWorkflows = $results | Where-Object { $_.RunningInstances -gt 0 }
$idleWorkflows = $results | Where-Object { $_.RunningInstances -eq 0 }

Write-Host "  Running workflows: " -NoNewline
Write-Host "$($runningWorkflows.Count)" -ForegroundColor Green
Write-Host "  Idle workflows: " -NoNewline
Write-Host "$($idleWorkflows.Count)" -ForegroundColor Gray
Write-Host "  Total workflows: " -NoNewline
Write-Host "$($results.Count)" -ForegroundColor Cyan

# Output detailed table
Write-Host "`nDetailed Results:" -ForegroundColor Cyan
$results | Format-Table -AutoSize

# Optionally export to CSV
# $results | Export-Csv -Path "workflow-status-$(Get-Date -Format 'yyyyMMdd-HHmmss').csv" -NoTypeInformation
```

## How It Works

The script operates in several stages:

1. **Authentication**: Uses Azure CLI (`az rest`) which leverages your existing Azure authentication
2. **Workflow Discovery**: Calls the Azure REST API to retrieve all workflows in the Logic App
3. **Status Check**: For each workflow, queries the runs endpoint with a filter for "Running" status
4. **Result Aggregation**: Builds a collection of workflow status objects
5. **Output**: Displays both real-time console output and a formatted summary table

## Prerequisites

Before running the script, ensure you have:

+ Azure CLI installed and configured
+ Authenticated with `az login`
+ Appropriate permissions to read Logic App and workflow information
+ PowerShell Core (pwsh) installed

## Usage Scenarios

**Pre-Deployment Checks**  
Before deploying updates to a Logic App, run the script to verify no critical workflows are processing messages. This prevents interrupting active business processes.

**Troubleshooting Performance**  
If a Logic App is experiencing performance issues, quickly identify which workflows have the most active runs and may be contributing to resource contention.

**Monitoring During High Load**  
During peak processing times, monitor workflow activity to ensure messages are being processed and no workflows are stalled.

**Maintenance Windows**  
Before taking a Logic App offline for maintenance, confirm all workflows have completed their active runs.

## Benefits

+ **Time Savings**: Check all workflows in seconds rather than manually inspecting each one
+ **Visibility**: Get a complete picture of Logic App activity at a glance
+ **Automation**: Can be integrated into deployment pipelines or monitoring systems
+ **Documentation**: Export results to CSV for audit trails or reporting
+ **Flexibility**: Easily modify to check different subscription, resource groups, or Logic Apps

## Potential Enhancements

The script provides a solid foundation and could be extended with:

+ Parameter sets for checking multiple Logic Apps in one execution
+ Filtering to only show workflows with active runs
+ Integration with Azure Monitor for alerting
+ Automated execution on a schedule with results sent via email
+ Additional filters for workflow runs (failed, succeeded, waiting, etc.)

## Conclusion

Managing Azure Logic Apps Standard at scale requires good tooling. This PowerShell script provides a simple but effective way to monitor workflow activity across all workflows in a Logic App instance. By leveraging the Azure REST API and Azure CLI, it provides quick insights that would otherwise require tedious manual checking through the Azure Portal.

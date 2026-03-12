# CST8921 – Cloud Industry Trends
## Lab 7 – Serverless Computing
### Lab Report

**Student:** Kelvin Ngabo  
**Student ID:** 041196196

**Date:** March 12, 2026  
**Course:** CST8921 – Cloud Industry Trends  


## Introduction

This lab provided hands-on experience with serverless computing on Microsoft Azure. The objective was to build an event-driven, end-to-end telemetry pipeline that ingests wind turbine data, applies a health scoring rule inside an Azure Function, stores results in Azure Table Storage, and automatically triggers an email alert via a Logic App when a turbine is in an URGENT state.

Serverless computing eliminates the need to manage underlying server infrastructure. Azure Functions (FaaS – Function as a Service) allow developers to focus on individual units of logic that are triggered by events. Combined with Azure Event Hubs, Table Storage, and Logic Apps, this lab demonstrated how to build a fully automated, cloud-native monitoring system.

---

## Architecture Overview

```
Python Simulator (send_events.py)
          ↓  sends JSON telemetry events
Azure Event Hub (turbine-telemetry)
          ↓  triggers on each message
Azure Function (FunctionDWDumper)
          ↓  applies health scoring rule + writes row
Azure Table Storage (TurbineMetrics)
          ↓  polled every 1 minute
Logic App (la-smartturbine-nk07)
          ↓  checks for URGENT status
Email Alert → ngab0016@algonquinlive.com
```

**Health Scoring Rule:**
- If `WindSpeed > 15` AND `GeneratedPower < 5` → `Status = "URGENT"`
- Otherwise → `Status = "HEALTHY"`

**Resource Naming Convention:**

| Resource | Name |
|---|---|
| Resource Group | rg-SmartTurbine |
| Storage Account | stsmartturbngab |
| Event Hub Namespace | hubdatamigration-nk-07 |
| Event Hub | turbine-telemetry |
| Function App | func-smartturbine-nk07 |
| Logic App | la-smartturbine-nk07 |
| Table | TurbineMetrics |

---

## Phase 1 – Infrastructure Setup

### Step 1 – Azure Login & Account Verification

The Azure CLI was used to authenticate and verify the active subscription before creating any resources.

```bash
az login
az account show
```

**Screenshot 1 – Azure Login & Account Show**

![alt text](<Screenshot 2026-03-12 at 13.48.02.png>)

The output confirmed the active subscription: **MOC DS – 10295** under the **CloudLabsAI.com** tenant.

---

### Step 2 – Resource Group Creation

A dedicated resource group `rg-SmartTurbine` was created in the `eastus` region to contain all lab resources.

```bash
az group create -l eastus -n rg-SmartTurbine
```

**Screenshot 2 – Resource Group Created**

![alt text](<Screenshot 2026-03-12 at 13.52.25.png>)

The output confirmed `"provisioningState": "Succeeded"`.

---

### Steps 3–6 – Storage, Event Hub, Function App & Table

The remaining infrastructure was provisioned using Azure CLI commands. The storage account `stsmartturbngab` was created first, followed by the Event Hub namespace and hub, then the Function App, and finally the `TurbineMetrics` table.

**Screenshot 3 – Function App Created (state: Running)**

![alt text](<Screenshot 2026-03-12 at 14.10.16.png>)

**Screenshot 4 – Storage Key Retrieved & TurbineMetrics Table Created**

![alt text](<Screenshot 2026-03-12 at 14.12.33.png>)

The `az storage table create` command returned `"created": true`, confirming the NoSQL table was ready.

**Screenshot 5 – All Resources Listed (az resource list)**

![alt text](<Screenshot 2026-03-12 at 14.13.35.png>)

The resource list confirmed all required resources were present in `rg-SmartTurbine`:
- `stsmartturbngab` – Storage Account
- `hubdatamigration-nk-07` – Event Hub Namespace
- `func-smartturbine-nk07` – Function App
- `EastUSPlan` – App Service Plan (auto-created)

---

## Phase 2 – Building the Function

### Project Setup

Since the instructor-provided `FunctionDWDumper` project was unavailable, the project was created from scratch using Azure Functions Core Tools:

```bash
func init FunctionDWDumper --worker-runtime dotnet-isolated
func new --name FunctionDWDumper --template "EventHubTrigger"
dotnet add package Azure.Data.Tables
```

**Screenshot 6 – Function Project Created & Package Added**

![alt text](<Screenshot 2026-03-12 at 14.15.53.png>)

### Function Code

The function was implemented with three key responsibilities:

1. **Trigger** – Fires on each incoming message from the `turbine-telemetry` Event Hub
2. **Health Scoring** – Applies the URGENT/HEALTHY rule
3. **Table Storage Write** – Persists each telemetry record to `TurbineMetrics`

**Screenshot 7 – Function Code in VS Code (TableEntity block visible)**

![alt text](<Screenshot 2026-03-12 at 14.18.28.png>)

The critical health scoring logic implemented:

```csharp
string status = (telemetry.WindSpeed > 15 && telemetry.GeneratedPower < 5)
    ? "URGENT"
    : "HEALTHY";
```

Each entity was stored with `PartitionKey = DeviceId` and `RowKey = Timestamp`, along with `WindSpeed`, `GeneratedPower`, `TurbineSpeed`, and `Status` fields.

### Deployment & Configuration

The function was deployed using the Azure Functions Core Tools publish command. The EventHub connection string was then set as an application setting:

```bash
az functionapp config appsettings set \
  -g rg-SmartTurbine \
  -n func-smartturbine-nk07 \
  --settings "EventHubConnection=<CONNECTION_STRING>"
```

**Screenshot 8 – EventHubConnection App Setting Configured**

![alt text](<Screenshot 2026-03-12 at 14.25.28.png>)

**Screenshot 9 – Function App Environment Variables in Portal**

![alt text](<Screenshot 2026-03-12 at 14.28.29.png>)

The Portal confirmed both `AzureWebJobsStorage` and `EventHubConnection` were present as application settings.

---

## Phase 3 – Logic App Automation

### Logic App Creation

A Consumption-tier Logic App named `la-smartturbine-nk07` was created in the `rg-SmartTurbine` resource group via the Azure Portal.

**Screenshot 10 – Logic App Deployment Succeeded**

![alt text](<Screenshot 2026-03-12 at 14.31.17.png>)

### Workflow Design

The Logic App workflow was built in the Designer with the following steps:

**Trigger:** Recurrence – every 1 minute  
**Action 1:** Get entities (V2) from `TurbineMetrics` table with filter `Status eq 'URGENT'`  
**Action 2:** For each entity returned  
**Condition:** `items('For_each')['Status']` is equal to `URGENT`  
**True branch:** Send an email (V2) via Outlook.com  

**Screenshot 11 – Recurrence Trigger Configured**

![alt text](<Screenshot 2026-03-12 at 14.36.17.png>)

**Screenshot 12 – Get Entities Step with Filter Query**

![alt text](<Screenshot 2026-03-12 at 14.43.57.png>)

**Screenshot 13 – Full Workflow: Recurrence → Get Entities → For Each → Condition → Send Email**

![alt text](<Screenshot 2026-03-12 at 15.05.01.png>)

The email alert was configured with:
- **To:** ngab0016@algonquinlive.com
- **Subject:** `ALERT: Turbine Failure Detected!`
- **Body:** `Critical failure detected. Power output is too low for current wind speeds.`

---

## Phase 4 – Live Simulation & Monitoring

### Data Simulation

Since the `WindTurbineDataGenerator.exe` was not available, a Python script (`send_events.py`) was written to simulate telemetry events using the `azure-eventhub` SDK. The script sent 20 events, alternating between HEALTHY and URGENT scenarios:

- **HEALTHY:** `WindSpeed=8.0`, `GeneratedPower=12.5`
- **URGENT:** `WindSpeed=18.5`, `GeneratedPower=3.2`

**Screenshot 14 – Python Simulation Script Running (20 events sent)**

![alt text](<Screenshot 2026-03-12 at 15.39.46.png>)

### Function Invocation Monitoring

The Azure Function was monitored via the Invocations tab in the Portal.

**Screenshot 15 – Function Invocations List (Success and Error entries)**

![alt text](<Screenshot 2026-03-12 at 15.42.54.png>)

**Screenshot 16 – Invocation Details (Execution Log)**

![alt text](<Screenshot 2026-03-12 at 15.43.19.png>)

The log confirmed successful execution with the message: `Executed 'Functions.FunctionDWDumper' (Succeeded)`.

### Table Storage Validation

The `TurbineMetrics` table was inspected using the Azure Portal Storage Browser.

**Screenshot 17 – TurbineMetrics Table with Rows**

![alt text](<Screenshot 2026-03-12 at 15.44.30.png>)

The table showed 7 rows across three turbine devices (`turbine-001`, `turbine-002`, `turbine-003`) with correct timestamps and telemetry values.

**Screenshot 18 – URGENT Entity Detail (turbine-001)**

![alt text](<Screenshot 2026-03-12 at 15.45.14.png>)

The entity detail confirmed:
- `PartitionKey`: turbine-001
- `RowKey`: 2026-03-12T19:39:00Z
- `WindSpeed`: 18.5
- `GeneratedPower`: 3.2
- `Status`: **URGENT**

### Logic App Run History

**Screenshot 19 – Logic App Run History (Succeeded and Failed runs)**

![alt text](<Screenshot 2026-03-12 at 15.45.49.png>)

The run history showed multiple successful runs (15:30–15:38) confirming the Logic App triggered correctly on URGENT entities. Failed runs after 15:39 were caused by an incorrect condition expression that was subsequently fixed.

### Email Alert Confirmation

**Screenshot 20 – Alert Email Received in Junk Folder**

![alt text](<Screenshot 2026-03-12 at 15.56.32.png>)

Multiple alert emails with subject **"ALERT: Turbine Failure Detected!"** were received, confirming the end-to-end pipeline was fully operational.

---

## Findings & Analysis

### Serverless Benefits Observed

**1. Auto-scaling:** The Azure Function automatically handled all 20 incoming events without any manual infrastructure management. The Consumption plan scaled resources on demand.

**2. Event-driven execution:** The function only ran when messages arrived in the Event Hub — no resources were consumed while idle. This demonstrates the core FaaS benefit of pay-per-execution.

**3. Low operational overhead:** The entire pipeline from telemetry ingestion to email alert required zero server provisioning. All infrastructure was defined through CLI commands and Portal configuration.

**4. Integration simplicity:** Azure Logic Apps provided a no-code automation layer that connected Table Storage reads to email delivery through pre-built connectors.

### Health Scoring Rule Effectiveness

The rule `WindSpeed > 15 AND GeneratedPower < 5` proved effective at identifying turbine anomalies. In the simulation:
- 7 out of 20 events triggered URGENT status (turbine-001, every 3rd event)
- 13 events were classified as HEALTHY

This pattern reflects a realistic scenario where a minority of readings indicate critical conditions requiring immediate attention.

### Table Storage as a Telemetry Store

Azure Table Storage proved well-suited for this use case:
- **Low cost** compared to relational databases
- **Fast writes** — each function invocation completed in under 50ms for successful runs
- **Flexible schema** — no migration needed when adding new telemetry fields
- **PartitionKey/RowKey design** — using DeviceId + Timestamp enables efficient queries per device over time

---

## Challenges & Troubleshooting

### Challenge 1 – Event Hub Retention Error

**Problem:** The initial `az eventhubs eventhub create` command failed with:
> `The value '7' for MessageRetentionInDays is not valid for the Basic tier`

**Root Cause:** The Azure CLI defaulted to 7-day retention, which exceeds the Basic SKU's 1-day limit.

**Resolution:** Added `--retention-time-in-hours 24 --cleanup-policy Delete` flags to the create command.

**Learning:** The Basic Event Hub tier has strict limitations. For production workloads requiring longer retention, the Standard tier should be used.

---

### Challenge 2 – WindTurbineDataGenerator.exe Not Available

**Problem:** The instructor-provided data generator was not available.

**Resolution:** A Python script (`send_events.py`) was written using the `azure-eventhub` SDK to simulate telemetry events with both HEALTHY and URGENT scenarios. This achieved the same outcome as the provided tool.

**Learning:** Understanding the underlying protocol (AMQP via azure-eventhub) allowed for a flexible replacement solution.

---

### Challenge 3 – Logic App Condition Failing

**Problem:** The Logic App condition returned `No inputs / No outputs` and failed, preventing email alerts from sending.

**Root Cause:** The condition was using `Get entities result An entity Entity data` which returned a raw object rather than the `Status` string field directly.

**Resolution:** Replaced the condition expression with `items('For_each')['Status']` using the fx expression editor, which correctly accessed the Status field from each iterated entity.

**Learning:** When iterating over Table Storage results in a Logic App For Each loop, item fields must be accessed using the `items('ActionName')['FieldName']` expression syntax.

---

### Challenge 4 – Alert Emails Delivered to Junk

**Problem:** Alert emails were delivered to the Junk/Spam folder rather than the inbox.

**Root Cause:** The Logic App sent emails from an unverified sender address, which spam filters flagged.

**Resolution:** Located emails in the Junk Email folder. This is expected behaviour for automated system emails and would be resolved in production by configuring a verified sending domain.

---

## Resource deletion

![alt text](<Screenshot 2026-03-12 at 16.16.08.png>)


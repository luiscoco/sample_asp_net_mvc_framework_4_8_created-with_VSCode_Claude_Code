# Azure Application Insights — KQL Query Reference

This file documents how to run KQL queries against the `CodereMVCAppInsights` resource and provides a catalogue of the most useful queries for monitoring the application.

---

## Environment reference

| Item | Value |
|---|---|
| App Insights App ID | `b555c2cd-b2ce-4d7e-8b88-8bccbe74dea7` |
| Log Analytics Workspace ID | `8675ad26-45ec-4a6c-a56d-82038a06e6a8` |
| App URL | `https://codere-sample-mvc-legacy.azurewebsites.net` |

---

## How to run queries — three methods

### Method 1 — Azure Portal (interactive, visual)

1. Go to [portal.azure.com](https://portal.azure.com)
2. Navigate to **CodereSampleMVCLegacy** → **CodereMVCAppInsights**
3. Click **Logs** in the left menu
4. Paste any query and press **Run** or `Shift + Enter`

The portal provides a table view, chart visualisation, time range pickers, and the ability to pin results to dashboards.

---

### Method 2 — App Insights REST API via PowerShell (seconds delay)

This is the recommended method for scripting and automation. Data is available within seconds of ingestion.

```powershell
# One-time setup
$appId = "b555c2cd-b2ce-4d7e-8b88-8bccbe74dea7"
$token = az account get-access-token `
  --resource "https://api.applicationinsights.io" `
  --query accessToken --output tsv

# Reusable helper function
function Invoke-AIQuery {
    param([string]$Query, [string]$Timespan = "PT2H")
    $enc  = [uri]::EscapeDataString($Query)
    $resp = Invoke-RestMethod `
        -Uri "https://api.applicationinsights.io/v1/apps/$appId/query?query=$enc&timespan=$Timespan" `
        -Headers @{ Authorization = "Bearer $token" }
    $cols = $resp.tables[0].columns | ForEach-Object { $_.name }
    $resp.tables[0].rows | ForEach-Object {
        $row = $_; $obj = [ordered]@{}
        for ($i = 0; $i -lt $cols.Count; $i++) { $obj[$cols[$i]] = $row[$i] }
        [pscustomobject]$obj
    }
}

# Usage — paste any query from this file:
Invoke-AIQuery "requests | order by timestamp desc | take 10" | Format-Table
```

> **Token expiry:** Access tokens are valid for ~1 hour. Re-run the `az account get-access-token` line to refresh.

> **Timespan format:** ISO 8601 duration — `PT1H` = 1 hour, `PT24H` = 24 hours, `P7D` = 7 days, `P30D` = 30 days.

---

### Method 3 — Log Analytics workspace via Azure CLI (5–15 min lag)

Use this for queries that combine App Insights data with other Azure resources, or for scheduled alert rules. Table names differ from Method 2 (prefixed with `App`).

```powershell
az monitor log-analytics query `
  --workspace "8675ad26-45ec-4a6c-a56d-82038a06e6a8" `
  --timespan "PT2H" `
  --analytics-query "AppRequests | order by TimeGenerated desc | take 10" `
  --output table
```

---

## KQL language basics

KQL (Kusto Query Language) is a read-only pipe-based language. Each `|` passes the result of the previous operator into the next.

| Operator | Purpose | Example |
|---|---|---|
| `take N` | Return first N rows | `requests \| take 10` |
| `order by col desc` | Sort results | `requests \| order by timestamp desc` |
| `where condition` | Filter rows | `requests \| where resultCode == "404"` |
| `project col1, col2` | Select columns | `requests \| project name, duration` |
| `extend col = expr` | Add a computed column | `requests \| extend slow = duration > 500` |
| `summarize` | Aggregate | `requests \| summarize count() by name` |
| `count()` | Count rows in a group | used inside `summarize` |
| `avg(col)` | Average | `summarize avg(duration)` |
| `percentile(col, N)` | Nth percentile | `percentile(duration, 95)` |
| `countif(condition)` | Count matching rows | `countif(success == false)` |
| `bin(col, interval)` | Time bucket | `bin(timestamp, 1h)` |
| `round(val, digits)` | Round number | `round(duration, 1)` |
| `strcat(...)` | Concatenate strings | `strcat(tostring(n), "%")` |

---

## Query catalogue

### Requests

**Recent requests — last 10:**
```kusto
requests
| order by timestamp desc
| take 10
| project timestamp, name, resultCode, duration, success
```

**Requests per page — calls, average duration, failures:**
```kusto
requests
| summarize Calls=count(), AvgDuration=round(avg(duration),1), Failures=countif(success==false) by name
| order by Calls desc
```

**Response time percentiles (P50 / P95 / P99) by page:**
```kusto
requests
| summarize
    p50=round(percentile(duration,50),1),
    p95=round(percentile(duration,95),1),
    p99=round(percentile(duration,99),1)
  by name
| order by p95 desc
```

**Overall failure rate:**
```kusto
requests
| summarize Total=count(), Failed=countif(success==false)
| extend FailurePct=strcat(tostring(round(100.0*Failed/Total,1)), "%")
```

**Traffic by HTTP status code:**
```kusto
requests
| summarize Count=count() by resultCode
| order by Count desc
```

**Slowest individual requests:**
```kusto
requests
| where duration > 200
| order by duration desc
| take 10
| project timestamp, name, duration, resultCode, url=tostring(url)
```

**Hourly traffic and failure trend:**
```kusto
requests
| summarize Requests=count(), Failures=countif(success==false) by bin(timestamp, 1h)
| order by timestamp asc
```

**Failed requests only:**
```kusto
requests
| where success == false
| order by timestamp desc
| take 20
| project timestamp, name, resultCode, duration, url=tostring(url)
```

---

### Exceptions

**Recent server exceptions:**
```kusto
exceptions
| order by timestamp desc
| take 10
| project timestamp, type, outerMessage, method=tostring(parsejson(details)[0].parsedStack[0].method)
```

**Exception count by type:**
```kusto
exceptions
| summarize Count=count() by type
| order by Count desc
```

**Full stack trace for a specific exception:**
```kusto
exceptions
| where type contains "NullReference"
| take 1
| project timestamp, type, outerMessage, details
```

---

### Performance counters

**CPU and memory over time:**
```kusto
performanceCounters
| where name in ("% Processor Time", "Private Bytes", "Available Bytes")
| summarize avg(value) by bin(timestamp, 5m), name
| order by timestamp asc
```

---

### Page views (browser-side)

**Page views per page:**
```kusto
pageViews
| summarize Views=count(), AvgLoadTime=round(avg(duration),0) by name
| order by Views desc
```

**Browser load time percentiles:**
```kusto
pageViews
| summarize p50=percentile(duration,50), p95=percentile(duration,95) by name
| order by p95 desc
```

---

### Combined / overview

**Request volume, failure rate, and avg response time — last 24 hours:**
```kusto
requests
| where timestamp > ago(24h)
| summarize
    Requests   = count(),
    Failures   = countif(success==false),
    AvgMs      = round(avg(duration),1),
    P95Ms      = round(percentile(duration,95),1)
  by bin(timestamp, 1h)
| extend FailurePct = round(100.0*Failures/Requests,1)
| order by timestamp asc
```

**Top pages by user sessions (requires JS SDK page view tracking):**
```kusto
pageViews
| summarize Sessions=dcount(session_Id), Views=count() by name
| order by Sessions desc
```

---

## Log Analytics equivalents (Method 3)

When using `az monitor log-analytics query`, replace the table names as follows:

| App Insights API | Log Analytics workspace |
|---|---|
| `requests` | `AppRequests` |
| `exceptions` | `AppExceptions` |
| `dependencies` | `AppDependencies` |
| `pageViews` | `AppPageViews` |
| `traces` | `AppTraces` |
| `customEvents` | `AppEvents` |
| `performanceCounters` | `AppPerformanceCounters` |
| `timestamp` | `TimeGenerated` |

Example — recent requests via Log Analytics:
```powershell
az monitor log-analytics query `
  --workspace "8675ad26-45ec-4a6c-a56d-82038a06e6a8" `
  --timespan "PT2H" `
  --analytics-query "AppRequests | order by TimeGenerated desc | take 10 | project TimeGenerated, Name, ResultCode, DurationMs, Success" `
  --output table
```

> Allow **5–15 minutes** after traffic is generated before Log Analytics queries return data. Use the App Insights REST API (Method 2) for real-time results.

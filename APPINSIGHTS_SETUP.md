# Azure Application Insights — Setup Guide

This document describes every step and command used to add Azure Application Insights monitoring to the ASP.NET MVC Framework 4.8 application.

---

## Azure resources created

| Resource | Name | Type |
|---|---|---|
| Log Analytics Workspace | `CodereMVCLogAnalytics` | Microsoft.OperationalInsights/workspaces |
| Application Insights | `CodereMVCAppInsights` | Microsoft.Insights/components |

Both resources are in the same resource group, subscription, and region as the Web App:

| Setting | Value |
|---|---|
| Subscription | `e5bd93f3-dcd9-4833-a589-82e16245997c` |
| Resource group | `CodereSampleMVCLegacy` |
| Region | `westeurope` |

---

## What Application Insights monitors

Once set up, the following telemetry is collected automatically with no extra code:

| Category | What is tracked |
|---|---|
| **Server requests** | Every HTTP request: URL, method, response code, duration |
| **Exceptions** | Unhandled exceptions with full stack trace |
| **Dependencies** | Outbound calls to databases, HTTP APIs, Azure services |
| **Performance counters** | CPU, memory, and private bytes of the IIS process |
| **Page views** | Browser-side page loads (via JavaScript SDK) |
| **Client exceptions** | Unhandled JavaScript errors in the browser |
| **Heartbeat** | Periodic metadata about the Azure App Service host |
| **Live Metrics** | Real-time request/failure/latency stream (< 1 second lag) |

---

## Part 1 — Azure resources

### Step 1 — Create the Log Analytics Workspace

Modern Application Insights components are **workspace-based**: they store telemetry in a Log Analytics Workspace, which enables cross-resource queries and longer retention policies.

```powershell
az monitor log-analytics workspace create `
  --workspace-name CodereMVCLogAnalytics `
  --resource-group CodereSampleMVCLegacy `
  --location westeurope `
  --output table
```

### Step 2 — Get the workspace resource ID

The workspace ID is needed to link the App Insights component to it.

```powershell
$workspaceId = az monitor log-analytics workspace show `
  --workspace-name CodereMVCLogAnalytics `
  --resource-group CodereSampleMVCLegacy `
  --query id --output tsv
```

### Step 3 — Create the Application Insights component

```powershell
az monitor app-insights component create `
  --app CodereMVCAppInsights `
  --resource-group CodereSampleMVCLegacy `
  --location westeurope `
  --kind web `
  --application-type web `
  --workspace $workspaceId `
  --output table
```

> **`--kind web` and `--application-type web`**: These mark the component as a web application, which enables the correct set of default dashboards and alert rules in the Azure Portal.

> **`--workspace $workspaceId`**: Links to the Log Analytics Workspace. Without this, a classic (non-workspace-based) component is created, which Microsoft has deprecated.

> The `application-insights` CLI extension is installed automatically on first use.

### Step 4 — Retrieve the connection string

```powershell
az monitor app-insights component show `
  --app CodereMVCAppInsights `
  --resource-group CodereSampleMVCLegacy `
  --query connectionString --output tsv
```

The connection string looks like:
```
InstrumentationKey=fbba157c-05a6-489c-a5ec-1fec4e7bc502;IngestionEndpoint=https://westeurope-5.in.applicationinsights.azure.com/;LiveEndpoint=https://westeurope.livediagnostics.monitor.azure.com/;ApplicationId=b555c2cd-b2ce-4d7e-8b88-8bccbe74dea7
```

> **Connection string vs. Instrumentation Key**: The Instrumentation Key alone is deprecated. The full connection string is required for global deployments because the ingestion endpoint varies by region. Always use the connection string.

### Step 5 — Set the connection string as Azure Web App settings

Two app settings are set so the SDK can pick up the value automatically at runtime:

- `APPLICATIONINSIGHTS_CONNECTION_STRING` — read by the .NET server SDK
- `APPINSIGHTS_CONNECTIONSTRING` — read by the Razor layout for the JavaScript snippet

```powershell
$cs = "InstrumentationKey=fbba157c-05a6-489c-a5ec-1fec4e7bc502;IngestionEndpoint=https://westeurope-5.in.applicationinsights.azure.com/;LiveEndpoint=https://westeurope.livediagnostics.monitor.azure.com/;ApplicationId=b555c2cd-b2ce-4d7e-8b88-8bccbe74dea7"

az webapp config appsettings set `
  --name codere-sample-mvc-legacy `
  --resource-group CodereSampleMVCLegacy `
  --settings "APPLICATIONINSIGHTS_CONNECTION_STRING=$cs" "APPINSIGHTS_CONNECTIONSTRING=$cs"
```

> Azure Web App settings override values in `Web.config` and `ApplicationInsights.config` at runtime, so the connection string in those files is only used when running locally with IIS Express.

---

## Part 2 — Code changes

Five files were created or modified.

### Change 1 — NuGet package (`SampleMvcApp.csproj`)

Added `Microsoft.ApplicationInsights.Web` 2.22.0. This is the official Application Insights SDK for classic ASP.NET (not ASP.NET Core). It installs as a single package but pulls in several sub-packages automatically:

| Sub-package | Purpose |
|---|---|
| `Microsoft.ApplicationInsights` | Core SDK — telemetry client, configuration, channel |
| `Microsoft.AI.Web` | HTTP modules for request and exception tracking |
| `Microsoft.AI.DependencyCollector` | Tracks outbound SQL and HTTP calls |
| `Microsoft.AI.PerfCounterCollector` | Reads Windows performance counters |
| `Microsoft.AI.ServerTelemetryChannel` | Reliable channel with disk buffering and retry |
| `Microsoft.AI.WindowsServer` | Azure / App Service environment metadata |

```xml
<PackageReference Include="Microsoft.ApplicationInsights.Web" Version="2.22.0" />
```

The `ApplicationInsights.config` file also needs to be declared as content so it is included in the publish output:

```xml
<Content Include="ApplicationInsights.config" />
```

### Change 2 — `ApplicationInsights.config` (new file)

This XML file is read by the SDK at startup. It registers all telemetry modules, initializers, processors, and the telemetry channel.

Key sections:

```xml
<!-- Connection string (used when running locally) -->
<ConnectionString>InstrumentationKey=...;IngestionEndpoint=...;...</ConnectionString>

<!-- Initializers add properties to every telemetry item -->
<TelemetryInitializers>
  <Add Type="Microsoft.ApplicationInsights.Web.OperationNameTelemetryInitializer, Microsoft.AI.Web" />
  <Add Type="Microsoft.ApplicationInsights.Web.UserTelemetryInitializer, Microsoft.AI.Web" />
  <Add Type="Microsoft.ApplicationInsights.Web.SessionTelemetryInitializer, Microsoft.AI.Web" />
  <!-- ... more initializers ... -->
</TelemetryInitializers>

<!-- Modules auto-collect telemetry without any code -->
<TelemetryModules>
  <Add Type="Microsoft.ApplicationInsights.Web.RequestTrackingTelemetryModule, Microsoft.AI.Web" />
  <Add Type="Microsoft.ApplicationInsights.Web.ExceptionTrackingTelemetryModule, Microsoft.AI.Web" />
  <Add Type="Microsoft.ApplicationInsights.DependencyCollector.DependencyTrackingTelemetryModule, Microsoft.AI.DependencyCollector" />
  <Add Type="Microsoft.ApplicationInsights.Extensibility.PerfCounterCollector.PerformanceCollectorModule, Microsoft.AI.PerfCounterCollector" />
  <!-- ... more modules ... -->
</TelemetryModules>

<!-- Adaptive sampling limits data volume automatically -->
<TelemetrySinks>
  <Add Name="default">
    <TelemetryProcessors>
      <Add Type="Microsoft.ApplicationInsights.WindowsServer.TelemetryChannel.AdaptiveSamplingTelemetryProcessor, Microsoft.AI.ServerTelemetryChannel">
        <MaxTelemetryItemsPerSecond>5</MaxTelemetryItemsPerSecond>
      </Add>
    </TelemetryProcessors>
    <TelemetryChannel Type="Microsoft.ApplicationInsights.WindowsServer.TelemetryChannel.ServerTelemetryChannel, Microsoft.AI.ServerTelemetryChannel" />
  </Add>
</TelemetrySinks>
```

> **Why `ServerTelemetryChannel`?** Unlike the default in-memory channel, the server channel buffers telemetry to disk when the ingestion endpoint is unavailable, then replays it when connectivity is restored. This prevents data loss during brief network outages.

> **Why adaptive sampling?** The Free tier of App Insights has a 5 GB/month data cap. Adaptive sampling automatically reduces volume when traffic is high, keeping data costs in check without requiring manual configuration.

### Change 3 — `Web.config` (five sections updated)

**3a — appSettings:** Added the connection string so it is available to the Razor view engine when rendering the JavaScript snippet:

```xml
<add key="APPINSIGHTS_CONNECTIONSTRING"
     value="InstrumentationKey=fbba157c-05a6-489c-a5ec-1fec4e7bc502;..." />
```

> At runtime on Azure, this value is overridden by the Azure Web App application setting of the same name, so the config file value only applies when running locally.

**3b — HTTP modules (Fix required — not auto-added by PackageReference):**

With `PackageReference`, NuGet no longer runs `install.ps1` scripts, so the App Insights HTTP modules are never registered automatically. Without them, the SDK cannot intercept HTTP requests and the application crashes with a `FileLoadException` cascade on startup. Both sections must be present: `<system.web><httpModules>` for Classic mode and `<system.webServer><modules>` for Integrated mode (Azure uses Integrated):

```xml
<system.web>
  <httpModules>
    <add name="TelemetryCorrelationHttpModule"
         type="Microsoft.AspNet.TelemetryCorrelation.TelemetryCorrelationHttpModule, Microsoft.AspNet.TelemetryCorrelation" />
    <add name="ApplicationInsightsWebTracking"
         type="Microsoft.ApplicationInsights.Web.ApplicationInsightsHttpModule, Microsoft.AI.Web" />
  </httpModules>
</system.web>

<system.webServer>
  <modules>
    <remove name="TelemetryCorrelationHttpModule" />
    <add name="TelemetryCorrelationHttpModule"
         type="Microsoft.AspNet.TelemetryCorrelation.TelemetryCorrelationHttpModule, Microsoft.AspNet.TelemetryCorrelation"
         preCondition="integratedMode,managedHandler" />
    <remove name="ApplicationInsightsWebTracking" />
    <add name="ApplicationInsightsWebTracking"
         type="Microsoft.ApplicationInsights.Web.ApplicationInsightsHttpModule, Microsoft.AI.Web"
         preCondition="managedHandler" />
  </modules>
</system.webServer>
```

**3c — Polyfill assembly binding redirects (Fix required):**

`Microsoft.ApplicationInsights.Web` 2.22.0 ships with newer versions of five polyfill assemblies than those in the .NET Framework 4.8 GAC. Each mismatch causes a `FileLoadException` at startup. The actual versions were confirmed with:

```powershell
$bin = ".\publish\bin"
@("System.Diagnostics.DiagnosticSource","System.Runtime.CompilerServices.Unsafe",
  "System.Memory","System.Buffers","System.Numerics.Vectors") | ForEach-Object {
    $v = [System.Reflection.AssemblyName]::GetAssemblyName("$bin\$_.dll").Version
    Write-Host "$_  →  $v"
}
```

All five binding redirects added to the `<runtime><assemblyBinding>` section:

```xml
<dependentAssembly>
  <assemblyIdentity name="System.Diagnostics.DiagnosticSource" publicKeyToken="cc7b13ffcd2ddd51" />
  <bindingRedirect oldVersion="0.0.0.0-5.0.0.0" newVersion="5.0.0.0" />
</dependentAssembly>
<dependentAssembly>
  <assemblyIdentity name="System.Runtime.CompilerServices.Unsafe" publicKeyToken="b03f5f7f11d50a3a" />
  <bindingRedirect oldVersion="0.0.0.0-5.0.0.0" newVersion="5.0.0.0" />
</dependentAssembly>
<dependentAssembly>
  <assemblyIdentity name="System.Memory" publicKeyToken="cc7b13ffcd2ddd51" />
  <bindingRedirect oldVersion="0.0.0.0-4.0.1.1" newVersion="4.0.1.1" />
</dependentAssembly>
<dependentAssembly>
  <assemblyIdentity name="System.Buffers" publicKeyToken="cc7b13ffcd2ddd51" />
  <bindingRedirect oldVersion="0.0.0.0-4.0.3.0" newVersion="4.0.3.0" />
</dependentAssembly>
<dependentAssembly>
  <assemblyIdentity name="System.Numerics.Vectors" publicKeyToken="b03f5f7f11d50a3a" />
  <bindingRedirect oldVersion="0.0.0.0-4.1.4.0" newVersion="4.1.4.0" />
</dependentAssembly>
```

### Change 4 — `Views/Shared/_Layout.cshtml` (JavaScript SDK)

The server-side SDK only tracks server requests, exceptions, and dependencies. To also track **browser-side page views, client exceptions, and user sessions**, the Application Insights JavaScript SDK must be loaded on every page.

The layout was updated in two steps:

**Step A — Read the connection string server-side:**

```razor
@{
    var aiConnectionString = System.Configuration.ConfigurationManager
        .AppSettings["APPINSIGHTS_CONNECTIONSTRING"] ?? string.Empty;
}
```

**Step B — Inject the async SDK loader into `<head>`:**

```html
@if (!string.IsNullOrEmpty(aiConnectionString))
{
    <script type="text/javascript">
        var sdkInstance = "appInsightsSDK";
        window[sdkInstance] = "appInsights";
        var aiName = window[sdkInstance];
        var aisdk = window[aiName] || function (n) {
            var o = { config: n }, t = document, e = window, i = "script";
            setTimeout(function () {
                var e = t.createElement(i);
                e.src = n.url || "https://js.monitor.azure.com/scripts/b/ai.2.min.js";
                t.getElementsByTagName(i)[0].parentNode.appendChild(e);
            });
            try { o.cookie = t.cookie } catch (p) { }
            function s(n) {
                o[n] = function () {
                    var e = arguments;
                    o.queue.push(function () { o[n].apply(o, e); });
                };
            }
            o.queue = []; o.version = 2;
            for (var r = ["Event","PageView","Exception","Trace","DependencyData","Metric","PageViewPerformance"]; r.length;)
                s("track" + r.pop());
            s("setAuthenticatedUserContext"); s("clearAuthenticatedUserContext"); s("flush");
            return o;
        }({ connectionString: "@aiConnectionString" });
        window[aiName] = aisdk;
        if (aisdk.queue && aisdk.queue.length === 0) { aisdk.trackPageView({}); }
    </script>
}
```

> **How this snippet works:** It creates a stub `appInsights` object immediately so that any calls made before the SDK loads are queued. The actual SDK (`ai.2.min.js`) is loaded asynchronously from the Microsoft CDN and does not block page rendering. Once loaded, it replays the queue. Wrapping the injection in `@if (!string.IsNullOrEmpty(...))` means the snippet is suppressed entirely when no connection string is configured (e.g., during local development without App Insights).

---

## Part 3 — Redeploy

After the code changes, the app was rebuilt and redeployed following the same process as described in `DEPLOY_AZURE.md`, with one addition: `ApplicationInsights.config` must be copied to the publish folder.

```powershell
# 1. Publish
dotnet publish SampleMvcApp.csproj --configuration Release --output .\publish

# 2. Restructure (DLLs into bin\, copy web files)
New-Item -ItemType Directory -Force -Path .\publish\bin | Out-Null
Get-ChildItem .\publish\*.dll, .\publish\*.pdb | Move-Item -Destination .\publish\bin\ -Force
Copy-Item Web.config, Global.asax, ApplicationInsights.config -Destination .\publish\ -Force
Copy-Item Views, Content, Scripts, fonts -Destination .\publish\ -Recurse -Force

# 3. Zip and deploy
Compress-Archive -Path .\publish\* -DestinationPath deploy.zip -Force
az webapp deploy `
  --name codere-sample-mvc-legacy `
  --resource-group CodereSampleMVCLegacy `
  --src-path deploy.zip `
  --type zip
```

> **Note:** `dotnet publish` automatically copies `ApplicationInsights.config` to the DLL output directory (the SDK's MSBuild target handles this). When restructuring the publish folder, copy it explicitly from the project root to `publish\` so it sits next to `Web.config` in the web root — that is where IIS and the SDK expect it.

---

## Part 4 — Fixes applied during setup

### Fix 1 — HTTP modules not registered (app returns HTTP 500 immediately)

**Symptom:** After deploying with App Insights added, every request returns HTTP 500. Enabling `<customErrors mode="Off" />` reveals: `Could not load file or assembly 'System.Diagnostics.DiagnosticSource'`.

**Root cause:** `PackageReference` does not execute `install.ps1` scripts, so `TelemetryCorrelationHttpModule` and `ApplicationInsightsHttpModule` are never added to `Web.config`. The HTTP modules are the entry point for the SDK — without them, IIS tries to load App Insights assemblies through a different code path that fails due to version mismatches.

**Fix:** Manually register both modules in `Web.config` (see Change 3b above).

### Fix 2 — Polyfill assembly version cascade (five `FileLoadException` errors)

**Symptom:** After fixing the HTTP modules, the app returned HTTP 500 five more times, each time with a different `FileLoadException` for a polyfill assembly (`System.Diagnostics.DiagnosticSource`, `System.Runtime.CompilerServices.Unsafe`, `System.Memory`, `System.Buffers`, `System.Numerics.Vectors`).

**Root cause:** The App Insights SDK 2.22.0 ships with .NET 5-era polyfill assemblies. The .NET Framework 4.8 GAC has older versions of the same assemblies. IIS resolves the GAC version first, the versions don't match, and the CLR throws `FileLoadException`.

**Diagnosis approach:**
```powershell
# For each failing assembly, check the real version in bin\
[System.Reflection.AssemblyName]::GetAssemblyName(".\publish\bin\System.Diagnostics.DiagnosticSource.dll").Version
# → 5.0.0.0  (but 4.0.4.0 was requested → add redirect to 5.0.0.0)
```

**Fix:** Added five `<bindingRedirect>` entries to `Web.config` (see Change 3c above).

### Fix 3 — Log Analytics workspace has 5–15 minute ingestion lag

**Symptom:** Queries against `az monitor log-analytics query` returned empty results even after the app was confirmed to be sending telemetry.

**Root cause:** For workspace-based App Insights, telemetry flows: App → App Insights ingestion endpoint → Log Analytics workspace. The final hop to Log Analytics has a 5–15 minute propagation delay. The `az monitor log-analytics query` command targets the workspace directly and misses recent data.

**Fix:** Use the **App Insights REST API** (`api.applicationinsights.io`) for queries — it reflects ingested data within seconds. See `APPINSIGHTS_QUERIES.md` for working examples of both query methods.

---

## Verifying telemetry in the Azure Portal

1. Open the [Azure Portal](https://portal.azure.com)
2. Navigate to **CodereSampleMVCLegacy** → **CodereMVCAppInsights**
3. Browse to the deployed app at `https://codere-sample-mvc-legacy.azurewebsites.net` to generate traffic
4. In the App Insights blade, check:
   - **Overview** — live request rate, failure rate, response time
   - **Live Metrics** — real-time stream (< 1 second delay)
   - **Transaction search** — individual request and page view records
   - **Failures** — exception details with stack traces
   - **Performance** — response time breakdown by operation

Telemetry typically appears in the **Overview** within **1–2 minutes** of the first request. Log Analytics queries (`AppRequests` table) may take **5–15 minutes** to reflect new data.

---

## Running queries — two methods

### Method 1: App Insights REST API (real-time, seconds delay)

```powershell
$appId  = "b555c2cd-b2ce-4d7e-8b88-8bccbe74dea7"
$token  = az account get-access-token --resource "https://api.applicationinsights.io" --query accessToken --output tsv
$query  = "requests | order by timestamp desc | take 10 | project timestamp, name, resultCode, duration, success"
$enc    = [uri]::EscapeDataString($query)

Invoke-RestMethod `
  -Uri "https://api.applicationinsights.io/v1/apps/$appId/query?query=$enc&timespan=PT2H" `
  -Headers @{ Authorization = "Bearer $token" }
```

Table names used by this API: `requests`, `exceptions`, `dependencies`, `pageViews`, `traces`, `customEvents`.

### Method 2: Log Analytics workspace (5–15 min lag, use for dashboards/alerts)

```powershell
az monitor log-analytics query `
  --workspace "8675ad26-45ec-4a6c-a56d-82038a06e6a8" `
  --timespan "PT2H" `
  --analytics-query "AppRequests | order by TimeGenerated desc | take 10" `
  --output table
```

Table names in Log Analytics differ: `AppRequests`, `AppExceptions`, `AppDependencies`, `AppPageViews`, `AppTraces`, `AppEvents`.

> See `APPINSIGHTS_QUERIES.md` for a full set of KQL sample queries with real output.

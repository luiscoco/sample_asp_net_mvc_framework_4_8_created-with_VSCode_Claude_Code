# Azure Web Apps Deployment Guide — ASP.NET MVC Framework 4.8

This document describes every step and command required to deploy this application to **Azure Web Apps** using the **Free (F1) tier**.

---

## Azure environment details

| Setting | Value |
|---|---|
| Subscription ID | `e5bd93f3-dcd9-4833-a589-82e16245997c` |
| Resource group | `CodereSampleMVCLegacy` |
| Region | `westeurope` |
| App Service Plan | `CodereMVCFreePlan` |
| Web App name | `codere-sample-mvc-legacy` *(must be globally unique — change if taken)* |
| Runtime stack | `ASPNET:V4.8` |
| OS | Windows *(required — Linux does not support .NET Framework)* |
| SKU | `F1` (Free) |

---

## Prerequisites

- **Azure CLI** installed (`az --version` — tested with 2.80.0)
- **dotnet SDK** installed (`dotnet --version`)
- An active Azure subscription

---

## Step 1 — Log in to Azure

```powershell
az login
```

This opens a browser for interactive authentication. After sign-in, the CLI shows the list of available subscriptions.

---

## Step 2 — Set the active subscription

```powershell
az account set --subscription e5bd93f3-dcd9-4833-a589-82e16245997c
```

Verify it is set correctly:

```powershell
az account show --query "{Subscription:name, ID:id, State:state}" --output table
```

---

## Step 3 — Confirm the resource group exists

The resource group `CodereSampleMVCLegacy` was already provisioned. Verify it:

```powershell
az group show --name CodereSampleMVCLegacy --query "{Name:name, Location:location, State:properties.provisioningState}" --output table
```

If you ever need to create it from scratch:

```powershell
# Only run this if the group does not exist
az group create --name CodereSampleMVCLegacy --location westeurope
```

---

## Step 4 — Create the App Service Plan (Free tier)

Azure Web Apps run inside an **App Service Plan**, which defines the region, OS, and pricing tier.

> **F1 (Free) tier limitations:**
> - 60 minutes of CPU time per day
> - 1 GB storage
> - Shared infrastructure (not dedicated)
> - No custom domains
> - No SSL certificates
> - URL will be `https://<app-name>.azurewebsites.net`

```powershell
az appservice plan create `
  --name CodereMVCFreePlan `
  --resource-group CodereSampleMVCLegacy `
  --sku F1 `
  --location westeurope
```

> **Note:** The `--is-windows` flag was removed in Azure CLI 2.80.0. Windows is now the default OS when `--is-linux` is not present. Passing `--is-windows` causes an error on CLI 2.80.0 and later.

Verify the plan was created:

```powershell
az appservice plan show `
  --name CodereMVCFreePlan `
  --resource-group CodereSampleMVCLegacy `
  --query "{Name:name, SKU:sku.name, Status:status}" `
  --output table
```

---

## Step 5 — Create the Web App

The app name must be **globally unique** across all Azure customers because it becomes part of the public URL (`https://codere-sample-mvc-legacy.azurewebsites.net`). Change the name if it is already taken.

```powershell
az webapp create `
  --name codere-sample-mvc-legacy `
  --resource-group CodereSampleMVCLegacy `
  --plan CodereMVCFreePlan `
  --runtime "ASPNET:V4.8"
```

> **Why `--runtime "ASPNET:V4.8"`?**
> This selects the classic ASP.NET pipeline on IIS — the same runtime that hosts MVC 5 applications on-premises. To list all available Windows runtimes:
> ```powershell
> az webapp list-runtimes --os-type windows
> ```

After creation, Azure prints the default URL. You can also retrieve it with:

```powershell
az webapp show `
  --name codere-sample-mvc-legacy `
  --resource-group CodereSampleMVCLegacy `
  --query "defaultHostName" `
  --output tsv
```

---

## Step 6 — Publish the application locally

`dotnet publish` compiles the application and copies all NuGet DLL dependencies into the output folder.

```powershell
dotnet publish "C:\Codere Admira\sample_asp_net_mvc_framework_4_8\SampleMvcApp.csproj" `
  --configuration Release `
  --output "C:\Codere Admira\sample_asp_net_mvc_framework_4_8\publish"
```

> **Known limitation:** `Microsoft.NET.Sdk` (SDK-style projects) does not automatically copy web content files — Views, Content, Scripts, fonts, Web.config, and Global.asax are not included in the publish output. They must be added manually in the next step.

After publishing, the `publish\` folder contains only DLLs at the root. IIS expects them inside a `bin\` subfolder. Run this script to restructure the folder and copy all missing web files:

```powershell
$pub  = "C:\Codere Admira\sample_asp_net_mvc_framework_4_8\publish"
$proj = "C:\Codere Admira\sample_asp_net_mvc_framework_4_8"

# Move DLLs into bin\
New-Item -ItemType Directory -Force -Path "$pub\bin" | Out-Null
Get-ChildItem "$pub\*.dll","$pub\*.pdb" | Move-Item -Destination "$pub\bin\" -Force

# Copy web root files
Copy-Item "$proj\Web.config"                 "$pub\Web.config"                 -Force
Copy-Item "$proj\Global.asax"                "$pub\Global.asax"                -Force
Copy-Item "$proj\ApplicationInsights.config" "$pub\ApplicationInsights.config" -Force

# Copy web content folders
Copy-Item "$proj\Views"   "$pub\Views"   -Recurse -Force
Copy-Item "$proj\Content" "$pub\Content" -Recurse -Force
Copy-Item "$proj\Scripts" "$pub\Scripts" -Recurse -Force
Copy-Item "$proj\fonts"   "$pub\fonts"   -Recurse -Force
```

The final `publish\` structure should look like this:

```
publish\
  bin\                       ← SampleMvcApp.dll + all dependency DLLs
  Content\                   ← bootstrap.css, site.css
  Scripts\                   ← jquery-3.7.1.js, bootstrap.js
  fonts\                     ← Bootstrap Glyphicon fonts
  Views\                     ← All Razor views + Views\Web.config
  ApplicationInsights.config ← App Insights SDK configuration
  Web.config
  Global.asax
```

> **Why copy `ApplicationInsights.config` explicitly?** The App Insights MSBuild target does copy this file to the compiler output directory (`bin\`), but IIS expects it in the web root alongside `Web.config`. Always copy it from the project root to the publish root.

---

## Step 7 — Package the publish output as a ZIP

Azure's zip deployment API expects a single `.zip` file containing all the application files at the root level (not wrapped in a subfolder).

```powershell
Compress-Archive `
  -Path "C:\Codere Admira\sample_asp_net_mvc_framework_4_8\publish\*" `
  -DestinationPath "C:\Codere Admira\sample_asp_net_mvc_framework_4_8\deploy.zip" `
  -Force
```

> **Important:** The `-Path` uses `\*` (star) so the zip contains the files directly at the root, not inside a subfolder. If the DLL is at `subfolder\bin\SampleMvcApp.dll` instead of `bin\SampleMvcApp.dll`, IIS will not find the application.

---

## Step 8 — Deploy to Azure Web Apps

Use `az webapp deploy` to upload and extract the zip on the Azure side:

```powershell
az webapp deploy `
  --name codere-sample-mvc-legacy `
  --resource-group CodereSampleMVCLegacy `
  --src-path "C:\Codere Admira\sample_asp_net_mvc_framework_4_8\deploy.zip" `
  --type zip
```

This command:
1. Uploads the zip to the Kudu deployment engine running inside the Web App
2. Extracts it to the `wwwroot` folder (`D:\home\site\wwwroot` on Azure Windows)
3. Recycles the IIS application pool so the new DLL is loaded

Deployment takes 30–90 seconds. The command exits when it is complete.

---

## Step 9 — Verify the deployment

Open the application URL in a browser:

```powershell
Start-Process "https://codere-sample-mvc-legacy.azurewebsites.net"
```

Or check the app status from the CLI:

```powershell
az webapp show `
  --name codere-sample-mvc-legacy `
  --resource-group CodereSampleMVCLegacy `
  --query "{URL:defaultHostName, State:state, Runtime:siteProperties}" `
  --output table
```

To stream live application logs from the terminal (useful for diagnosing startup errors):

```powershell
az webapp log tail `
  --name codere-sample-mvc-legacy `
  --resource-group CodereSampleMVCLegacy
```

---

## Fixes discovered during deployment

### Fix A — `--is-windows` flag removed in Azure CLI 2.80.0

The flag `--is-windows` for `az appservice plan create` was deprecated and removed in Azure CLI 2.80.0. Windows is now the default OS when no `--is-linux` flag is passed. The working command omits `--is-windows` entirely. Passing it on CLI 2.80.0+ produces: `ERROR: unrecognized arguments: --is-windows`.

### Fix B — `dotnet publish` does not copy web content files

`Microsoft.NET.Sdk` (the SDK used by this project) only copies compiled DLLs to the publish output. It does not include Views, Content, Scripts, fonts, Web.config, Global.asax, or `ApplicationInsights.config`. These files must be copied manually after publish (see Step 6 above). This is a known difference from the classic Visual Studio publish experience.

### Fix C — Resource group did not exist

The resource group `CodereSampleMVCLegacy` was specified by the user but had not been provisioned yet. It was created with:
```powershell
az group create --name CodereSampleMVCLegacy --location westeurope
```

---

## Redeploy after code changes

Every time you change the application code, repeat Steps 6 → 7 → 8:

```powershell
$pub  = "C:\Codere Admira\sample_asp_net_mvc_framework_4_8\publish"
$proj = "C:\Codere Admira\sample_asp_net_mvc_framework_4_8"

# 1. Publish
Remove-Item $pub -Recurse -Force -ErrorAction SilentlyContinue
dotnet publish "$proj\SampleMvcApp.csproj" --configuration Release --output $pub

# 2. Restructure
New-Item -ItemType Directory -Force -Path "$pub\bin" | Out-Null
Get-ChildItem "$pub\*.dll","$pub\*.pdb" | Move-Item -Destination "$pub\bin\" -Force
Copy-Item "$proj\Web.config","$proj\Global.asax","$proj\ApplicationInsights.config" -Destination $pub -Force
Copy-Item "$proj\Views","$proj\Content","$proj\Scripts","$proj\fonts" -Destination $pub -Recurse -Force

# 3. Zip
$zip = "$proj\deploy.zip"
Remove-Item $zip -Force -ErrorAction SilentlyContinue
Compress-Archive -Path "$pub\*" -DestinationPath $zip -Force

# 4. Deploy
az webapp deploy `
  --name codere-sample-mvc-legacy `
  --resource-group CodereSampleMVCLegacy `
  --src-path $zip `
  --type zip
```

---

## Troubleshooting

### The app returns HTTP 500 after deployment

**Step 1 — Expose the full exception in the browser.** Add `<customErrors mode="Off" />` inside `<system.web>` in `Web.config`, copy it to the publish folder, rezip, and redeploy. The browser will then show the full exception message and stack trace instead of the generic error page.

```xml
<system.web>
  <customErrors mode="Off" />
  ...
</system.web>
```

> **Remove `customErrors mode="Off"` once the issue is diagnosed** — it exposes internal details publicly.

**Step 2 — Stream live logs from the CLI:**

```powershell
az webapp log config `
  --name codere-sample-mvc-legacy `
  --resource-group CodereSampleMVCLegacy `
  --application-logging filesystem `
  --level error

az webapp log tail `
  --name codere-sample-mvc-legacy `
  --resource-group CodereSampleMVCLegacy
```

**Step 3 — Check for assembly version mismatches.** The most common cause of 500 errors after adding new NuGet packages to a .NET Framework project is a `FileLoadException` where a DLL version in `bin\` doesn't match the binding redirect in `Web.config`. Diagnose with:

```powershell
# Check actual version of any DLL in bin\
[System.Reflection.AssemblyName]::GetAssemblyName(".\publish\bin\AssemblyName.dll").Version
```

Then update the `<bindingRedirect>` in `Web.config` to match. See **Fix 6** in `README.md` for the full list of redirects added for the App Insights polyfill assemblies.

### The app name `codere-sample-mvc-legacy` is already taken

Web App names are globally unique. If the name is taken, choose a different one and replace it in every command in this guide. Check availability first:

```powershell
az webapp show --name codere-sample-mvc-legacy --resource-group CodereSampleMVCLegacy 2>&1
# If it returns "ResourceNotFound", the name is available in your subscription
# (another tenant may still own it globally — the create command will tell you)
```

### Assembly version mismatch errors on Azure

Azure Web Apps include their own GAC (Global Assembly Cache). If you see binding errors for assemblies like `WebGrease` or `System.Web.Mvc`, make sure the `<bindingRedirect>` entries in `Web.config` match the versions of the DLLs in `bin\`. See the `README.md` for the fixes already applied.

---

## Clean up (delete all resources)

To remove everything and stop any potential billing:

```powershell
# Delete only the Web App
az webapp delete `
  --name codere-sample-mvc-legacy `
  --resource-group CodereSampleMVCLegacy

# Delete the App Service Plan
az appservice plan delete `
  --name CodereMVCFreePlan `
  --resource-group CodereSampleMVCLegacy `
  --yes

# Delete the entire resource group (removes everything inside it)
az group delete --name CodereSampleMVCLegacy --yes --no-wait
```

> The F1 free tier does not incur charges for compute, but the resource group and other services created inside it might. Always clean up resources you no longer need.

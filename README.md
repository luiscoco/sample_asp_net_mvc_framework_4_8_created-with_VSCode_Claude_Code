# Sample ASP.NET MVC — .NET Framework 4.8

A classic ASP.NET MVC 5 web application targeting .NET Framework 4.8, scaffolded manually using the SDK-style project format and the `dotnet` CLI.

---

## Why manual scaffolding?

The `dotnet new` CLI only provides templates for **ASP.NET Core** projects. There is no built-in template for classic ASP.NET MVC on .NET Framework. Visual Studio provides these templates through its own wizard, but the project was created here without Visual Studio using the SDK-style `.csproj` format.

---

## Project structure

```
SampleMvcApp.csproj          ← SDK-style project file targeting net48
Web.config                   ← ASP.NET runtime configuration and assembly binding redirects
Global.asax                  ← Application entry point directive
Global.asax.cs               ← MvcApplication class — wires up routes, filters, and bundles
App_Start/
  RouteConfig.cs             ← Defines the default {controller}/{action}/{id} route
  FilterConfig.cs            ← Registers HandleErrorAttribute globally
  BundleConfig.cs            ← Groups JS and CSS files into bundles for the browser
Controllers/
  HomeController.cs          ← Index, About, and Contact actions
Views/
  _ViewStart.cshtml          ← Sets the shared layout for all views
  Web.config                 ← Configures the Razor view engine and namespace imports
  Home/
    Index.cshtml
    About.cshtml
    Contact.cshtml
  Shared/
    _Layout.cshtml           ← Bootstrap 3 navbar shell used by all pages
    Error.cshtml             ← Displayed by HandleErrorAttribute on unhandled exceptions
Content/
  bootstrap.css
  bootstrap.min.css
  Site.css                   ← Custom application styles
Scripts/
  jquery-3.7.1.js / .min.js
  bootstrap.js / .min.js
fonts/                       ← Bootstrap 3 Glyphicons font files
```

---

## Steps followed to create the application

### 1. Created the SDK-style project file

Instead of using Visual Studio's wizard, the `.csproj` was written by hand using the modern **SDK-style** format. This format is much simpler than the classic MSBuild project format and works with the `dotnet` CLI.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net48</TargetFramework>
    <OutputType>Library</OutputType>
    ...
  </PropertyGroup>
</Project>
```

Key NuGet packages added:

| Package | Version | Purpose |
|---|---|---|
| `Microsoft.AspNet.Mvc` | 5.3.0 | The MVC 5 framework |
| `Microsoft.AspNet.Web.Optimization` | 1.1.3 | JS/CSS bundling and minification |
| `Newtonsoft.Json` | 13.0.3 | JSON serialization |
| `bootstrap` | 3.4.1 | CSS framework (NuGet reference only) |
| `jQuery` | 3.7.1 | JavaScript library (NuGet reference only) |

### 2. Created all MVC application files

The standard ASP.NET MVC project structure was created manually:

- `Global.asax` + `Global.asax.cs` — application startup
- `App_Start/RouteConfig.cs` — URL routing
- `App_Start/FilterConfig.cs` — global action filters
- `App_Start/BundleConfig.cs` — script and style bundles
- `Controllers/HomeController.cs` — three actions: Index, About, Contact
- Razor views for all three pages plus the shared layout
- `Views/Web.config` — required by the Razor engine to find MVC types
- `Web.config` — runtime settings, HTTP handlers, and assembly binding redirects

### 3. Added explicit framework assembly references

SDK-style projects targeting .NET Framework do not automatically include all `System.Web.*` assemblies the way a classic web project does. The following references had to be added explicitly to the `.csproj`:

```xml
<Reference Include="System.Web" />
<Reference Include="System.Web.ApplicationServices" />
<Reference Include="System.Web.Extensions" />
<Reference Include="System.Net.Http" />
<Reference Include="Microsoft.CSharp" />
```

`Microsoft.CSharp` is required because `ViewBag` uses the `dynamic` type, which depends on `CSharpArgumentInfo` at compile time.

---

## Fixes applied during development

### Fix 1 — IIS Express could not load `SampleMvcApp.MvcApplication`

**Error:** `No se pudo cargar el tipo 'SampleMvcApp.MvcApplication'`

**Cause:** The SDK-style project outputs the compiled DLL to `bin\Debug\net48\SampleMvcApp.dll` by default. IIS Express (and classic ASP.NET in general) expects the DLL to be in the `bin\` folder directly under the web root.

**Fix:** Added these properties to the `.csproj` to flatten the output path:

```xml
<OutputPath>bin\</OutputPath>
<AppendTargetFrameworkToOutputPath>false</AppendTargetFrameworkToOutputPath>
<AppendRuntimeIdentifierToOutputPath>false</AppendRuntimeIdentifierToOutputPath>
```

After rebuilding, the DLL was placed at `bin\SampleMvcApp.dll` and IIS Express could load it correctly.

### Fix 2 — WebGrease assembly version mismatch

**Error:** `No se puede cargar el archivo o ensamblado 'WebGrease, Version=1.6.5135.21930'`

**Cause:** The binding redirect in `Web.config` pointed to WebGrease version `1.6.5135.21930`, but the DLL actually installed by NuGet was version `1.5.2.14234`. The versions did not match, causing a `FileLoadException` at startup.

**Diagnosis:** Checked the real assembly version with PowerShell:
```powershell
[System.Reflection.AssemblyName]::GetAssemblyName(".\bin\WebGrease.dll").Version
# Output: 1.5.2.14234
```

**Fix:** Updated the binding redirect in `Web.config` to use the real version:

```xml
<dependentAssembly>
  <assemblyIdentity name="WebGrease" publicKeyToken="31bf3856ad364e35" />
  <bindingRedirect oldVersion="0.0.0.0-1.6.5135.21930" newVersion="1.5.2.14234" />
</dependentAssembly>
```

### Fix 3 — Scripts and Content folders missing (bundle engine crash)

**Error:** `Directory does not exist. Parameter name: directoryVirtualPath`

**Cause:** The `BundleConfig` referenced `~/Scripts/jquery-{version}.js`, `~/Scripts/bootstrap.js`, and `~/Content/bootstrap.css`, but the `Scripts\` and `Content\` folders did not exist.

With modern **PackageReference** (SDK-style projects), NuGet no longer copies content files (JS, CSS) into the project directory. This was a feature of the older `packages.config` workflow. The `bootstrap` and `jQuery` NuGet packages are effectively empty shells when used with PackageReference — they provide no actual files on disk.

**Fix:** Used `npm` to download the actual distribution files and copied them into the project manually:

```powershell
npm install --prefix "C:\Temp\mvc_libs" jquery@3.7.1 bootstrap@3.4.1

# Copy JS
Copy-Item "...\jquery\dist\jquery.js"        ".\Scripts\jquery-3.7.1.js"
Copy-Item "...\jquery\dist\jquery.min.js"    ".\Scripts\jquery-3.7.1.min.js"
Copy-Item "...\bootstrap\dist\js\bootstrap.js"     ".\Scripts\bootstrap.js"
Copy-Item "...\bootstrap\dist\js\bootstrap.min.js" ".\Scripts\bootstrap.min.js"

# Copy CSS
Copy-Item "...\bootstrap\dist\css\bootstrap.css"     ".\Content\bootstrap.css"
Copy-Item "...\bootstrap\dist\css\bootstrap.min.css" ".\Content\bootstrap.min.css"

# Copy fonts
Copy-Item "...\bootstrap\dist\fonts\*" ".\fonts\"
```

### Fix 4 — Modernizr not available

**Cause:** `Modernizr 2.8.3` is listed as a NuGet package but is not available on npm at that version. The `BundleConfig` referenced `~/Scripts/modernizr-*` which would have caused the same directory error as Fix 3.

**Fix:** Removed Modernizr from `BundleConfig.cs`, from the `_Layout.cshtml` render call, and from the `.csproj` package reference. Modernizr is a browser feature-detection library and is not required for the application to function.

---

## How to run the application

### Prerequisites

- [.NET Framework 4.8](https://dotnet.microsoft.com/download/dotnet-framework/net48) installed on the machine
- **IIS Express** (installed with Visual Studio, or standalone)

### Build

```powershell
dotnet build SampleMvcApp.csproj --configuration Debug
```

### Run with IIS Express

```powershell
& "C:\Program Files\IIS Express\iisexpress.exe" /path:"C:\Codere Admira\sample_asp_net_mvc_framework_4_8" /port:5000 /clr:v4.0
```

Then open `http://localhost:5000` in your browser.

Press `Q` + Enter in the terminal to stop IIS Express.

> **Note:** `dotnet run` does not work for .NET Framework ASP.NET MVC projects. The `dotnet` CLI can compile them, but it cannot host them. Classic ASP.NET MVC requires an IIS pipeline (IIS or IIS Express) to handle HTTP requests. `dotnet run` only provides a built-in host for ASP.NET Core projects.

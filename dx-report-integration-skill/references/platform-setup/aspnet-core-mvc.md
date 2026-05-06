# ASP.NET Core MVC — Complete Setup

## NuGet Packages

```
DevExpress.AspNetCore.Reporting    ← required
BuildBundlerMinifier               ← bundles node_modules into wwwroot
Microsoft.Web.LibraryManager.Build ← copies icon fonts via libman.json
```

## npm Packages (package.json)

```json
{
  "version": "1.0.0",
  "name": "asp.net",
  "private": true,
  "dependencies": {
    "bootstrap": "^4.3.1",
    "devextreme-dist": "25.2-stable",
    "@devexpress/analytics-core": "25.2-stable",
    "devexpress-reporting": "25.2-stable"
  }
}
```

Right-click `package.json` → **Restore Packages** (or `npm install`).

> **Build order:** Run `npm install` before `dotnet build`. The `libman.json` filesystem provider copies files from `node_modules/` — if packages are not restored first, libman fails with `LIB002 (could not resolve node_modules/...)` and the build stops.

## bundleconfig.json

```json
[
  {
    "outputFileName": "wwwroot/css/thirdparty.bundle.css",
    "inputFiles": [
      "node_modules/bootstrap/dist/css/bootstrap.min.css",
      "node_modules/devextreme-dist/css/dx.light.css"
    ],
    "minify": { "enabled": false, "adjustRelativePaths": false }
  },
  {
    "outputFileName": "wwwroot/css/viewer.part.bundle.css",
    "inputFiles": [
      "node_modules/@devexpress/analytics-core/dist/css/dx-analytics.common.css",
      "node_modules/@devexpress/analytics-core/dist/css/dx-analytics.light.css",
      "node_modules/devexpress-reporting/dist/css/dx-webdocumentviewer.css"
    ],
    "minify": { "enabled": false, "adjustRelativePaths": false }
  },
  {
    "outputFileName": "wwwroot/js/thirdparty.bundle.js",
    "inputFiles": [
      "node_modules/jquery/dist/jquery.min.js",
      "node_modules/knockout/build/output/knockout-latest.js",
      "node_modules/bootstrap/dist/js/bootstrap.min.js",
      "node_modules/devextreme-dist/js/dx.all.js"
    ],
    "minify": { "enabled": false },
    "sourceMap": false
  },
  {
    "outputFileName": "wwwroot/js/viewer.part.bundle.js",
    "inputFiles": [
      "node_modules/@devexpress/analytics-core/dist/js/dx-analytics-core.min.js",
      "node_modules/devexpress-reporting/dist/js/dx-webdocumentviewer.min.js"
    ],
    "minify": { "enabled": false },
    "sourceMap": false
  }
]
```

If you also include the End-User Report Designer, add a designer bundle. **Important:** `dx-analytics-core.js` must be the first entry in the designer bundle — it must not rely on a separate viewer bundle being loaded first.

```json
[
  {
    "outputFileName": "wwwroot/css/designer.part.bundle.css",
    "inputFiles": [
      "node_modules/@devexpress/analytics-core/dist/css/dx-analytics.common.css",
      "node_modules/@devexpress/analytics-core/dist/css/dx-analytics.light.css",
      "node_modules/devexpress-reporting/dist/css/dx-querybuilder.css",
      "node_modules/devexpress-reporting/dist/css/dx-reportdesigner.css"
    ],
    "minify": { "enabled": false, "adjustRelativePaths": false }
  },
  {
    "outputFileName": "wwwroot/js/designer.part.bundle.js",
    "inputFiles": [
      "node_modules/@devexpress/analytics-core/dist/js/dx-analytics-core.min.js",
      "node_modules/devexpress-reporting/dist/js/dx-querybuilder.min.js",
      "node_modules/devexpress-reporting/dist/js/dx-reportdesigner.min.js"
    ],
    "minify": { "enabled": false },
    "sourceMap": false
  }
]
```

The designer page must load the viewer bundle before the designer bundle. Reference order in the view:
```cshtml
<link rel="stylesheet" href="~/css/viewer.part.bundle.css" />
<link rel="stylesheet" href="~/css/designer.part.bundle.css" />
<script src="~/js/viewer.part.bundle.js"></script>
<script src="~/js/designer.part.bundle.js"></script>
```

## libman.json (icon fonts)

```json
{
  "version": "1.0",
  "defaultProvider": "filesystem",
  "libraries": [
    {
      "library": "node_modules/devextreme-dist/css/icons/",
      "destination": "wwwroot/css/icons",
      "files": ["dxicons.ttf", "dxicons.woff2", "dxicons.woff"]
    }
  ]
}
```

## Program.cs

```csharp
using DevExpress.AspNetCore;
using DevExpress.AspNetCore.Reporting;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllersWithViews();
builder.Services.AddDevExpressControls();                     // ← required
builder.Services.ConfigureReportingServices(configurator => {
    configurator.ConfigureWebDocumentViewer(viewerConfigurator => {
        viewerConfigurator.UseCachedReportSourceBuilder();    // ← recommended for production
    });
});

var app = builder.Build();
app.UseDevExpressControls();                                  // ← required, before UseStaticFiles
app.UseStaticFiles();
app.UseRouting();
app.MapDefaultControllerRoute();
app.Run();
```

`UseCachedReportSourceBuilder()` instructs the viewer to generate the document page-by-page into storage rather than fully in memory. Recommended for production and clustered/Azure deployments.

## Controllers/ReportingControllers.cs

```csharp
using DevExpress.AspNetCore.Reporting.WebDocumentViewer;
using DevExpress.AspNetCore.Reporting.WebDocumentViewer.Native.Services;

namespace WebApplication1.Controllers {
    public class CustomWebDocumentViewerController : WebDocumentViewerController {
        public CustomWebDocumentViewerController(
            IWebDocumentViewerMvcControllerService controllerService)
            : base(controllerService) { }
    }
}
```

If you also include the End-User Report Designer, add controllers for `ReportDesignerController` and `QueryBuilderController`.

## Views/_ViewImports.cshtml

```cshtml
@using DevExpress.AspNetCore
```

## Views/Shared/_Layout.cshtml (head section)

```cshtml
<head>
    <link rel="stylesheet" href="~/css/thirdparty.bundle.css" />
    <script src="~/js/thirdparty.bundle.js"></script>
</head>
```

## Viewer View (Views/Home/Viewer.cshtml)

### Bind directly to a report instance

```cshtml
@using WebApplication1.Reports

<link rel="stylesheet" href="~/css/viewer.part.bundle.css" />
<script src="~/js/viewer.part.bundle.js"></script>

@Html.DevExpress().WebDocumentViewer("DocumentViewer")
    .Height("calc(100vh - 60px)")
    .Bind(new SalesReport())
```

### Bind via controller + IReportProvider (recommended for dynamic reports)

```csharp
// HomeController.cs
public async Task<IActionResult> Viewer(
    [FromServices] IWebDocumentViewerClientSideModelGenerator modelGenerator,
    [FromQuery] string reportName = "SalesReport") {
    var model = await modelGenerator.GetModelAsync(
        reportName, WebDocumentViewerController.DefaultUri);
    return View(model);
}
```

```cshtml
@* Views/Home/Viewer.cshtml *@
@model DevExpress.XtraReports.Web.WebDocumentViewer.WebDocumentViewerModel

<link rel="stylesheet" href="~/css/viewer.part.bundle.css" />
<script src="~/js/viewer.part.bundle.js"></script>

@Html.DevExpress().WebDocumentViewer("DocumentViewer")
    .Height("calc(100vh - 60px)")
    .Bind(Model)
```

Register `IReportProvider` in `Program.cs`:
```csharp
builder.Services.AddScoped<IReportProvider, CustomReportProvider>();
```

## Troubleshooting

- **Razor helper chain renders as plain text** (`Height(...)` or `.Bind(...)` appears literally in the browser): wrap multi-line `@Html.DevExpress()` chains in `@(...)`:
  ```cshtml
  @(Html.DevExpress().WebDocumentViewer("DocumentViewer")
      .Height("calc(100vh - 60px)")
      .Bind(new SalesReport()))
  ```
  Single-line calls do not require the wrapper; multi-line chains always do.

- **`DevExpress.Analytics.Widgets is undefined`**: `dx-analytics-core.js` is not included in the designer bundle, or is loaded after `dx-querybuilder.js`. Ensure `dx-analytics-core.min.js` is the first entry in `designer.part.bundle.js`.

- **`DevExpress is not defined`**: `dx.all.js` (DevExtreme) is missing from `thirdparty.bundle.js` or loads after reporting scripts.

- **Blank viewer / 404 on DXXRDV**: `UseDevExpressControls()` missing from `Program.cs`, or placed after `UseStaticFiles()` — it must come before.
- **JavaScript errors**: npm not restored, or scripts not registered in correct order (`knockout` before `devextreme`, `thirdparty.bundle.js` before `viewer.part.bundle.js`).
- **Version mismatch errors**: npm package versions must match NuGet package versions. Enable Development Mode in `ConfigureReportingServices` to diagnose: `configurator.UseDevelopmentMode()`.
- **Report not found**: `IReportProvider.GetReport()` didn't find the name, or `ReportStorageWebExtension.GetData()` couldn't locate the file.

# Complete ASP.NET Core MVC Viewer Page

A complete working example showing an ASP.NET Core MVC app that displays a Document Viewer. Reports are loaded by name via `IReportProvider` — no End-User Designer, no `.repx` files needed.

## Project Files

### NuGet Packages (*.csproj)

```xml
<PackageReference Include="DevExpress.AspNetCore.Reporting" Version="25.2.*" />
<PackageReference Include="BuildBundlerMinifier" Version="3.2.*" />
<PackageReference Include="Microsoft.Web.LibraryManager.Build" Version="2.1.*" />
```

### package.json

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

Run `npm install` or right-click → Restore Packages.

### bundleconfig.json

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

### libman.json

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

### Program.cs

```csharp
using DevExpress.AspNetCore;
using DevExpress.AspNetCore.Reporting;
using DevExpress.XtraReports.Services;
using MyApp.Services;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllersWithViews();
builder.Services.AddDevExpressControls();
builder.Services.ConfigureReportingServices(configurator => {
    configurator.ConfigureWebDocumentViewer(viewerConfigurator => {
        viewerConfigurator.UseCachedReportSourceBuilder();
    });
});
builder.Services.AddScoped<IReportProvider, CustomReportProvider>();

var app = builder.Build();

app.UseDevExpressControls();   // ← before UseStaticFiles
app.UseStaticFiles();
app.UseRouting();
app.MapDefaultControllerRoute();
app.Run();
```

### Services/CustomReportProvider.cs

```csharp
using DevExpress.XtraReports.Services;
using DevExpress.XtraReports.UI;
using DevExpress.XtraReports.Web.ClientControls;

namespace MyApp.Services {
    public class CustomReportProvider : IReportProvider {
        public XtraReport GetReport(string id, ReportProviderContext context) =>
            id switch {
                "SalesReport"   => new Reports.SalesReport(),
                "InvoiceReport" => new Reports.InvoiceReport(),
                _ => throw new FaultException($"Report '{id}' not found.")
            };
    }
}
```

### Controllers/ReportingControllers.cs

```csharp
using DevExpress.AspNetCore.Reporting.WebDocumentViewer;
using DevExpress.AspNetCore.Reporting.WebDocumentViewer.Native.Services;

namespace MyApp.Controllers {
    public class CustomWebDocumentViewerController : WebDocumentViewerController {
        public CustomWebDocumentViewerController(
            IWebDocumentViewerMvcControllerService controllerService)
            : base(controllerService) { }
    }
}
```

### Controllers/HomeController.cs

```csharp
using DevExpress.AspNetCore.Reporting.WebDocumentViewer;
using DevExpress.XtraReports.Web.WebDocumentViewer;
using Microsoft.AspNetCore.Mvc;

namespace MyApp.Controllers {
    public class HomeController : Controller {

        // Option A: Bind viewer directly to a report instance in the view
        public IActionResult ViewerDirect() => View();

        // Option B: Bind viewer via model (supports dynamic report selection)
        public async Task<IActionResult> Viewer(
            [FromServices] IWebDocumentViewerClientSideModelGenerator modelGenerator,
            [FromQuery] string reportName = "SalesReport") {
            var model = await modelGenerator.GetModelAsync(
                reportName, WebDocumentViewerController.DefaultUri);
            return View(model);
        }
    }
}
```

### Views/_ViewImports.cshtml

```cshtml
@using MyApp
@using DevExpress.AspNetCore
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
```

### Views/Shared/_Layout.cshtml (relevant sections)

```cshtml
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <link rel="stylesheet" href="~/css/thirdparty.bundle.css" />
    @RenderSection("Styles", required: false)
</head>
<body>
    @RenderBody()
    <script src="~/js/thirdparty.bundle.js"></script>
    @RenderSection("Scripts", required: false)
</body>
```

### Views/Home/ViewerDirect.cshtml (Option A — direct bind)

```cshtml
@using MyApp.Reports

@{
    ViewData["Title"] = "Report Viewer";
}

@section Styles {
    <link rel="stylesheet" href="~/css/viewer.part.bundle.css" />
}
@section Scripts {
    <script src="~/js/viewer.part.bundle.js"></script>
}

@Html.DevExpress().WebDocumentViewer("DocumentViewer")
    .Height("calc(100vh - 60px)")
    .Bind(new SalesReport())
```

### Views/Home/Viewer.cshtml (Option B — model bind via IReportProvider)

```cshtml
@using DevExpress.XtraReports.Web.WebDocumentViewer
@model WebDocumentViewerModel

@{
    ViewData["Title"] = "Report Viewer";
}

@section Styles {
    <link rel="stylesheet" href="~/css/viewer.part.bundle.css" />
}
@section Scripts {
    <script src="~/js/viewer.part.bundle.js"></script>
}

@Html.DevExpress().WebDocumentViewer("DocumentViewer")
    .Height("calc(100vh - 60px)")
    .Bind(Model)
```

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Blank viewer or 404 on `DXXRDV` | `UseDevExpressControls()` missing or placed after `UseStaticFiles()` | Add before `UseStaticFiles()` in `Program.cs` |
| JavaScript errors in console | npm not restored, or wrong script order | Run `npm install`; ensure `knockout` is before `devextreme` in `bundleconfig.json` |
| Version mismatch errors | npm versions don't match NuGet versions | Keep both at `25.2.*`; enable `configurator.UseDevelopmentMode()` for diagnostics |
| Report not found | `IReportProvider` doesn't handle the name | Add the report name to the `switch` in `CustomReportProvider.cs` |
| Icon fonts missing | `libman.json` not restored | Run LibMan restore; verify `wwwroot/css/icons/` exists |

# ASP.NET Core Razor Pages — Complete Setup

## NuGet Packages

```
DevExpress.AspNetCore.Reporting
BuildBundlerMinifier
Microsoft.Web.LibraryManager.Build
```

## npm Packages (package.json)

```json
{
  "version": "1.0.0",
  "name": "asp.net",
  "private": true,
  "dependencies": {
    "bootstrap": "^5.3.3",
    "devextreme-dist": "25.2-stable",
    "@devexpress/analytics-core": "25.2-stable",
    "devexpress-reporting": "25.2-stable"
  }
}
```

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
    "outputFileName": "wwwroot/css/reporting.viewer.part.bundle.css",
    "inputFiles": [
      "node_modules/@devexpress/analytics-core/dist/css/dx-analytics.common.css",
      "node_modules/@devexpress/analytics-core/dist/css/dx-analytics.light.css",
      "node_modules/devexpress-reporting/dist/css/dx-webdocumentviewer.css"
    ],
    "minify": { "enabled": false, "adjustRelativePaths": false }
  },
  {
    "outputFileName": "wwwroot/js/vendor.js",
    "inputFiles": [
      "node_modules/knockout/build/output/knockout-latest.js"
    ],
    "minify": { "enabled": false }
  },
  {
    "outputFileName": "wwwroot/js/reporting.viewer.part.bundle.js",
    "inputFiles": [
      "node_modules/@devexpress/analytics-core/dist/js/dx-analytics-core.min.js",
      "node_modules/devexpress-reporting/dist/js/dx-webdocumentviewer.min.js"
    ],
    "minify": { "enabled": false },
    "sourceMap": false
  }
]
```

## libman.json

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

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddRazorPages();
builder.Services.AddDevExpressControls();   // ← required
builder.Services.AddMvcCore();              // ← required for reporting controllers

var app = builder.Build();
app.UseDevExpressControls();               // ← required, before UseStaticFiles
app.UseStaticFiles();
app.UseRouting();
app.MapRazorPages();
app.MapDefaultControllerRoute();           // ← required for reporting API endpoints
app.Run();
```

## Controllers/ReportingControllers.cs

```csharp
using DevExpress.AspNetCore.Reporting.WebDocumentViewer;
using DevExpress.AspNetCore.Reporting.WebDocumentViewer.Native.Services;

public class CustomWebDocumentViewerController : WebDocumentViewerController {
    public CustomWebDocumentViewerController(
        IWebDocumentViewerMvcControllerService controllerService)
        : base(controllerService) { }
}
```

## Pages/_ViewImports.cshtml

```cshtml
@using DevExpress.AspNetCore
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
```

## Pages/Shared/_Layout.cshtml (head section)

```cshtml
<head>
    <link rel="stylesheet" href="~/css/thirdparty.bundle.css" />
    <script src="~/js/vendor.js"></script>
</head>
```

## Pages/Viewer.cshtml

```cshtml
@page
@using WebApplication1.Reports

<link rel="stylesheet" href="~/css/reporting.viewer.part.bundle.css" />
<script src="~/js/reporting.viewer.part.bundle.js"></script>

@Html.DevExpress().WebDocumentViewer("DocumentViewer")
    .Height("calc(100vh - 60px)")
    .Bind(new SalesReport())
```

Note: for Razor Pages, the view file is `Pages/Viewer.cshtml` (not `Views/`). The `@page` directive makes it a Razor Page. Script/CSS links go directly in the page file, not only in `_Layout.cshtml`.

## Troubleshooting

- **404 on DXXRDV**: `AddMvcCore()` or `MapDefaultControllerRoute()` missing from `Program.cs`.
- **`vendor.js` missing**: Knockout must be included separately before DevExtreme scripts in Razor Pages applications.
- Same general troubleshooting as MVC applies (version mismatch, script order, `UseDevExpressControls()` placement).

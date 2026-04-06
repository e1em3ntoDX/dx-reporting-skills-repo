# Complete Blazor Viewer Pages

Two complete examples: one using the **native `DxReportViewer`** and one using the **JS-based `DxDocumentViewer`**. Both are for Blazor Server (Interactive Server render mode).

---

## Example A — Native DxReportViewer (Viewer Only)

The simplest setup. No MVC controllers needed. Customized via C#.

### NuGet

```
DevExpress.Blazor.Reporting.Viewer
```

### Program.cs

```csharp
using DevExpress.Blazor.Reporting;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddRazorComponents()
    .AddInteractiveServerComponents();
builder.Services.AddDevExpressServerSideBlazorReportViewer();
builder.WebHost.UseStaticWebAssets();

var app = builder.Build();
app.UseStaticFiles();
app.UseAntiforgery();
app.MapRazorComponents<App>().AddInteractiveServerRenderMode();
app.Run();
```

### Components/App.razor

```razor
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <base href="/" />
    @* Register DevExpress client scripts — REQUIRED *@
    @DxResourceManager.RegisterScripts()
    @* CSS theme — choose bootstrap 5 or fluent *@
    <link rel="stylesheet"
          href="_content/DevExpress.Blazor.Reporting.Viewer/css/dx-blazor-reporting-components.bs5.css" />
    <HeadOutlet />
</head>
<body>
    <Routes />
    <script src="_framework/blazor.web.js"></script>
</body>
</html>
```

### Components/_Imports.razor

```razor
@using Microsoft.AspNetCore.Components.Routing
@using Microsoft.AspNetCore.Components.Web
@using DevExpress.Blazor
@using DevExpress.Blazor.Reporting
```

### Components/Pages/ReportViewer.razor

```razor
@page "/viewer/{ReportName?}"
@rendermode InteractiveServer
@using DevExpress.XtraReports.UI

<h3>@(ReportName ?? "Report")</h3>

<div style="width:100%; height:calc(100vh - 80px)">
    <DxReportViewer @ref="reportViewer" />
</div>

@code {
    [Parameter] public string? ReportName { get; set; }
    DxReportViewer? reportViewer;

    protected override async Task OnAfterRenderAsync(bool firstRender) {
        if (!firstRender) return;
        XtraReport? report = (ReportName ?? "SalesReport") switch {
            "SalesReport"   => new SalesReport(),
            "InvoiceReport" => new InvoiceReport(),
            _ => null
        };
        if (report != null)
            await reportViewer!.OpenReportAsync(report);
    }
}
```

---

## Example B — JS-Based DxDocumentViewer + DxReportDesigner

Provides the full DevExtreme-based viewer UI including document map, search, and parameter panel. Also includes the End-User Report Designer. Customized via JavaScript.

### NuGet

```
DevExpress.Blazor.Reporting.JSBasedControls
DevExpress.AspNetCore.Reporting
```

### Program.cs

```csharp
using DevExpress.Blazor.Reporting;
using DevExpress.XtraReports.Web.Extensions;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddRazorComponents()
    .AddInteractiveServerComponents();
builder.Services.AddMvc();
builder.Services.AddDevExpressBlazorReporting();
builder.Services.ConfigureReportingServices(configurator => {
    configurator.ConfigureWebDocumentViewer(c => c.UseCachedReportSourceBuilder());
});
// Register report provider or storage for report name resolution
builder.Services.AddScoped<IReportProvider, CustomReportProvider>();

var app = builder.Build();
app.UseStaticFiles();
app.UseRouting();
app.UseDevExpressBlazorReporting();   // ← middleware for JS-based components
app.UseDevExpressControls();          // ← also required
app.UseAntiforgery();
app.UseEndpoints(e => e.MapControllers());   // ← required for viewer API endpoints
app.MapRazorComponents<App>().AddInteractiveServerRenderMode();
app.Run();
```

### Components/App.razor

```razor
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <base href="/" />
    @* Register DevExpress client scripts — REQUIRED *@
    @DxResourceManager.RegisterScripts()
    @* Optional: add customization JS (see viewer-toolbar-customization.md) *@
    @* @DxResourceManager.RegisterScripts(config =>
        config.Register(new DxResource("/js/viewer-customization.js", 900))) *@
    <HeadOutlet />
</head>
<body>
    <Routes />
    <script src="_framework/blazor.web.js"></script>
</body>
</html>
```

### Components/_Imports.razor

```razor
@using Microsoft.AspNetCore.Components.Routing
@using Microsoft.AspNetCore.Components.Web
@using DevExpress.Blazor
@using DevExpress.Blazor.Reporting
```

### Components/Pages/DocumentViewer.razor

```razor
@page "/docviewer"
@rendermode InteractiveServer

<DxDocumentViewer ReportName="SalesReport"
                  Height="calc(100vh - 60px)"
                  Width="100%">
    @* Optional JS-based customization callbacks *@
    @* <DxDocumentViewerCallbacks
        CustomizeExportOptions="ViewerCustomization.onCustomizeExportOptions" /> *@
</DxDocumentViewer>
```

### Components/Pages/ReportDesigner.razor

```razor
@page "/designer"
@rendermode InteractiveServer

<DxReportDesigner @ref="designer" />

@code {
    DxReportDesigner? designer;

    protected override async Task OnAfterRenderAsync(bool firstRender) {
        if (firstRender)
            await designer!.OpenReportAsync("SalesReport");
    }
}
```

### wwwroot/js/viewer-customization.js (optional)

```javascript
window.ViewerCustomization = {
    onCustomizeExportOptions: function(s, e) {
        // Hide XLS and RTF from the Export dropdown
        e.HideFormat(DevExpress.Reporting.Viewer.ExportFormatID.XLS);
        e.HideFormat(DevExpress.Reporting.Viewer.ExportFormatID.RTF);
    },
    onCustomizeMenuActions: function(s, e) {
        // Hide the Previous Page / Next Page buttons
        var prev = e.GetById(DevExpress.Reporting.Viewer.ActionId.PrevPage);
        if (prev) prev.visible = false;
    }
};
```

### Services/CustomReportProvider.cs

```csharp
using DevExpress.XtraReports.Services;
using DevExpress.XtraReports.UI;
using DevExpress.XtraReports.Web.ClientControls;

public class CustomReportProvider : IReportProvider {
    public XtraReport GetReport(string id, ReportProviderContext context) =>
        id switch {
            "SalesReport"   => new SalesReport(),
            "InvoiceReport" => new InvoiceReport(),
            _ => throw new FaultException($"Report '{id}' not found.")
        };
}
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Blank viewer box | `DxResourceManager.RegisterScripts()` missing from `App.razor` | Add it inside `<head>` |
| "Service not registered" at startup (native) | `AddDevExpressServerSideBlazorReportViewer()` missing | Add to `Program.cs` |
| "Service not registered" at startup (JS-based) | `AddDevExpressBlazorReporting()` missing | Add to `Program.cs` |
| 404 on viewer API calls (JS-based) | `UseDevExpressBlazorReporting()` or `MapControllers()` missing | Add both to `Program.cs` |
| Viewer not interactive / hydration errors | Missing `@rendermode InteractiveServer` on page | Add to top of razor page |
| CSS not applied | Missing CSS link in `App.razor` | Add `_content/DevExpress.Blazor.Reporting.Viewer/css/*.bs5.css` link |
| Report not found (JS-based) | `IReportProvider.GetReport()` doesn't match the `ReportName` | Add the name to the `switch` in `CustomReportProvider.cs` |

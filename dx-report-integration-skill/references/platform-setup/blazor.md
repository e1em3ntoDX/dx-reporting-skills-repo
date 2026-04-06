# Blazor Reporting — Setup Guide

Blazor has three distinct component families with different setup, capabilities, and registration requirements. Choose the right one before writing any code.

## Component Family Comparison

| Component | Type | Use When |
|---|---|---|
| `DxReportViewer` | **Native Blazor** | Viewer only. C# customization. Interactive Server or WebAssembly. |
| `DxDocumentViewer`, `DxReportDesigner`, `DxReportParametersPanel` | **JS-based (Server)** | Viewer + Designer + Parameters panel. JavaScript customization. Requires SignalR. Server render. |
| `DxWasmDocumentViewer`, `DxWasmReportDesigner`, `DxWasmReportParametersPanel` | **JS-based (WASM)** | Same as above but for WebAssembly render mode. Document generation on server. |

Reporting components require **interactive render mode** — they cannot run in static render mode.

---

## 1. Native DxReportViewer (Viewer Only)

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

// Choose one depending on render mode:
builder.Services.AddDevExpressServerSideBlazorReportViewer();   // Interactive Server
// builder.Services.AddDevExpressWebAssemblyBlazorReportViewer(); // Interactive WebAssembly

builder.WebHost.UseStaticWebAssets();

var app = builder.Build();
app.UseStaticFiles();
app.UseAntiforgery();
app.MapRazorComponents<App>().AddInteractiveServerRenderMode();
app.Run();
```

### _Imports.razor

```razor
@using DevExpress.Blazor
@using DevExpress.Blazor.Reporting
```

### App.razor

```razor
<head>
    @* Register DevExpress client scripts — REQUIRED *@
    @DxResourceManager.RegisterScripts()
    @* CSS — choose one theme *@
    <link rel="stylesheet"
          href="_content/DevExpress.Blazor.Reporting.Viewer/css/dx-blazor-reporting-components.bs5.css" />
</head>
```

### Viewer Page

```razor
@page "/viewer"
@rendermode InteractiveServer
@using DevExpress.Blazor.Reporting

<DxReportViewer @ref="reportViewer" />

@code {
    DxReportViewer? reportViewer;

    protected override async Task OnAfterRenderAsync(bool firstRender) {
        if (firstRender)
            await reportViewer!.OpenReportAsync(new SalesReport());
    }
}
```

For WebAssembly render mode, replace `@rendermode InteractiveServer` with `@rendermode InteractiveWebAssembly` and change the DI registration.

---

## 2. JS-Based Components — Blazor Server (InteractiveServer)

Provides `DxDocumentViewer`, `DxReportDesigner`, and `DxReportParametersPanel`. Requires MVC controllers for the viewer API.

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
builder.Services.AddDevExpressBlazorReporting();        // ← JS-based components
builder.Services.ConfigureReportingServices(configurator => {
    configurator.ConfigureWebDocumentViewer(c => c.UseCachedReportSourceBuilder());
});

var app = builder.Build();
app.UseStaticFiles();
app.UseRouting();
app.UseDevExpressBlazorReporting();                     // ← middleware
app.UseDevExpressControls();                            // ← also required
app.UseAntiforgery();
app.UseEndpoints(e => e.MapControllers());              // ← required for viewer API
app.MapRazorComponents<App>().AddInteractiveServerRenderMode();
app.Run();
```

### _Imports.razor

```razor
@using DevExpress.Blazor
@using DevExpress.Blazor.Reporting
```

### App.razor

```razor
<head>
    @DxResourceManager.RegisterScripts()
</head>
```

### Document Viewer Page

```razor
@page "/docviewer"
@rendermode InteractiveServer
@using DevExpress.Blazor.Reporting

<DxDocumentViewer ReportName="SalesReport" Height="calc(100vh - 60px)" Width="100%">
    <DxDocumentViewerCallbacks CustomizeExportOptions="ViewerCustomization.onCustomizeExportOptions" />
</DxDocumentViewer>
```

### Report Designer Page

```razor
@page "/designer"
@rendermode InteractiveServer
@using DevExpress.Blazor.Reporting

<DxReportDesigner @ref="designer" />

@code {
    DxReportDesigner? designer;

    protected override async Task OnAfterRenderAsync(bool firstRender) {
        if (firstRender)
            await designer!.OpenReportAsync("SalesReport");
    }
}
```

### Toolbar Customization (external JS file)

```javascript
// wwwroot/viewer-customization.js
window.ViewerCustomization = {
    onCustomizeExportOptions: function(s, e) {
        e.HideFormat(DevExpress.Reporting.Viewer.ExportFormatID.XLS);
    }
};
```

Register in `App.razor`:
```razor
@DxResourceManager.RegisterScripts(config =>
    config.Register(new DxResource("/viewer-customization.js", 900)))
```

---

## 3. JS-Based Components — Blazor WebAssembly (Interactive WebAssembly)

The WASM variant uses `DxWasmDocumentViewer` / `DxWasmReportDesigner`. Document generation always happens on the **server** (not in the browser). The Blazor WebAssembly project communicates with an ASP.NET Core backend via HTTP fetch.

### NuGet (Blazor Web App with WebAssembly interactivity)

| Project | Package |
|---|---|
| Server project | `DevExpress.AspNetCore.Reporting` |
| Client project | `DevExpress.Blazor.Reporting.JSBasedControls.WebAssembly` |

### Server Program.cs

```csharp
using DevExpress.AspNetCore;

builder.Services.AddControllersWithViews();
builder.Services.AddDevExpressControls();   // ← server-side
// ...
app.UseDevExpressControls();
app.MapControllers();
```

### Client Program.cs (WebAssembly)

```csharp
using DevExpress.Blazor.Reporting;

builder.Services.AddDevExpressBlazorReportingWebAssembly(configure => {
    configure.UseDevelopmentMode(); // ← add for development diagnostics
});
```

### Client _Imports.razor

```razor
@using DevExpress.Blazor.Reporting
@using DevExpress.Blazor
```

### Client App.razor

```razor
<HeadContent>
    @DxResourceManager.RegisterScripts()
</HeadContent>
```

### WASM Viewer Page (in the Client project)

```razor
@page "/viewer"
@using DevExpress.Blazor.Reporting

<DxWasmDocumentViewer ReportName="SalesReport" Height="800px" Width="100%" />
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Blank viewer | `DxResourceManager.RegisterScripts()` missing from `App.razor` | Add it to `<head>` |
| "Service not registered" at startup | Missing `AddDevExpressServerSideBlazorReportViewer()` or `AddDevExpressBlazorReporting()` | Add to `Program.cs` |
| 404 on reporting API calls (JS-based) | Missing `UseDevExpressBlazorReporting()` and/or `app.UseEndpoints(e => e.MapControllers())` | Add both to `Program.cs` |
| Components not found | Missing `@using DevExpress.Blazor.Reporting` in `_Imports.razor` | Add the namespace |
| `@rendermode` error | Page is using static render mode | Add `@rendermode InteractiveServer` to the page |
| CSS not loading | Missing link in `App.razor` | Add `_content/DevExpress.Blazor.Reporting.Viewer/css/...` link |

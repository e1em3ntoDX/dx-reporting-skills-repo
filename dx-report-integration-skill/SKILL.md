---
name: dx-report-integration
description: Expert skill for embedding DevExpress Reports into applications — NuGet packages, DI registration, viewer/designer wiring, async export, ReportStorageWebExtension, toolbar and parameter editor customization, security, and troubleshooting. Use whenever someone needs to add a DevExpress report viewer or designer to WinForms, WPF, ASP.NET Core MVC, Razor Pages, Angular, React, or Blazor; configure Program.cs for reporting; set up npm packages and bundleconfig.json; export reports to PDF/Excel/Word without a preview; configure ReportStorageWebExtension or IReportProvider; customize viewer toolbar, tab panel, search panel, or parameter editors; implement authorization or CSRF protection; debug errors like "viewer not rendering", "Unable to process binding", "Internal Server Error", "Report not found", 400/404/CORS errors; or upgrade DevExpress versions.
metadata:
  author: DevExpress Reports Team
  version: 1.0.0
---

You are an expert in integrating DevExpress Reports into .NET applications. You configure NuGet packages, DI services, client-side resources, viewers, designers, and export pipelines correctly for each platform.

## Using DevExpress Documentation MCP

If the DxDocs MCP server is available, use it to supplement this skill:

- **Search**: Use `devexpress_docs_search` with technology `"XtraReports"` (or `"AspNetCore"`, `"Blazor"`, etc.) and your question.
- **Fetch**: Use `devexpress_docs_get_content` with a documentation URL to get full article content.

When to use MCP vs. built-in references:
- **Built-in references**: Getting started, common patterns, key properties, troubleshooting covered in this skill.
- **MCP search**: Advanced scenarios not covered here, version-specific API changes, uncommon features.
- **Always MCP for**: Exact method signatures, event arguments, enum values, or edge cases when you are not 100% certain.

## 📦 NuGet Packages by Platform

| Platform | Required NuGet Package(s) |
|---|---|
| WinForms | `DevExpress.Win.Reporting` |
| WPF | `DevExpress.Wpf.Reporting` |
| ASP.NET Core MVC / Razor Pages | `DevExpress.AspNetCore.Reporting`, `BuildBundlerMinifier`, `Microsoft.Web.LibraryManager.Build` |
| Angular / React backend | `DevExpress.AspNetCore.Reporting` |
| **Linux / macOS** (any platform) | + `DevExpress.Drawing.Skia` — **required for non-Windows hosts** |
| Blazor Server — native viewer | `DevExpress.Blazor.Reporting.Viewer` |
| Blazor Server — JS-based viewer + designer | `DevExpress.Blazor.Reporting.JSBasedControls`, `DevExpress.AspNetCore.Reporting` |
| Blazor WebAssembly (client project) | `DevExpress.Blazor.Reporting.JSBasedControls.WebAssembly` |
| Blazor WebAssembly (server project) | `DevExpress.AspNetCore.Reporting` |

`ReportStorageWebExtension` and `IReportProvider` are **web-only** — for ASP.NET Core and Blazor. WinForms and WPF load report instances directly in code.

Report `.cs` / `.Designer.cs` files and `.repx` layout files are fully portable across all platforms.

## ⚙️ Mandatory Setup by Platform

Platform setup is required before any viewer or export will work. Missing these registrations causes the viewer to not render or throw service-not-found errors at runtime.

See `references/platform-setup/` for complete, copy-paste setup for each platform.

### WinForms — Minimal Setup

No `Program.cs` registration needed. Add the NuGet package and use directly:

```csharp
using DevExpress.XtraReports.UI;

using var printTool = new ReportPrintTool(new SalesReport());
printTool.ShowRibbonPreviewDialog(); // modal preview
// printTool.ShowRibbonPreview();    // non-modal
```

### WPF — Minimal Setup

```xml
<!-- MainWindow.xaml -->
xmlns:dxp="http://schemas.devexpress.com/winfx/2008/xaml/printing"

<dxp:DocumentPreviewControl x:Name="preview"
    RequestDocumentCreation="True"
    DocumentSource="{Binding Report}" />
```

```csharp
// Code-behind or ViewModel
public XtraReport Report { get; } = new SalesReport();
```

### ASP.NET Core MVC — 3 Required Steps

All three steps are mandatory. Missing any one causes the viewer to fail silently or throw 500 errors.

```csharp
// Program.cs — Step 1: Register services + recommended caching
// Required namespace for ConfigureReportingServices:
// using DevExpress.AspNetCore.Reporting;
builder.Services.AddDevExpressControls();
builder.Services.ConfigureReportingServices(configurator => {
    configurator.ConfigureWebDocumentViewer(c => c.UseCachedReportSourceBuilder());
});
// ...
var app = builder.Build();
// Step 2: Register middleware — must come BEFORE UseStaticFiles
app.UseDevExpressControls();
app.UseStaticFiles();
```

```json
// package.json — Step 3: npm packages (required for client-side viewer JS/CSS)
{
  "dependencies": {
    "devextreme-dist": "25.2-stable",
    "@devexpress/analytics-core": "25.2-stable",
    "devexpress-reporting": "25.2-stable",
    "bootstrap": "^4.3.1"
  }
}
```

Right-click `package.json` → Restore Packages (or `npm install`). Also create `bundleconfig.json` to bundle npm output into `wwwroot/`, and `libman.json` to copy icon fonts.

See `references/platform-setup/aspnet-core-mvc.md` for the complete setup: `bundleconfig.json`, `libman.json`, reporting controller, `_ViewImports`, `_Layout`, and viewer views.

See `references/platform-setup/aspnet-core-razor-pages.md` for the Razor Pages variant (requires `AddMvcCore()`, `vendor.js` for Knockout, `@page` directive in page files).

For Angular/React SPAs, the ASP.NET Core project acts as a backend-only API with CORS. See `references/platform-setup/angular-react.md`.

### Blazor — Choose the Right Component Family

Blazor has **three distinct component families** with different setups:

- **`DxReportViewer`** (native, viewer only) — uses `AddDevExpressServerSideBlazorReportViewer()`
- **`DxDocumentViewer` / `DxReportDesigner`** (JS-based, Server) — uses `AddDevExpressBlazorReporting()` + `UseDevExpressBlazorReporting()` + MVC controllers
- **`DxWasmDocumentViewer` / `DxWasmReportDesigner`** (JS-based, WebAssembly) — split NuGet across server/client projects

All Blazor components require interactive render mode and `DxResourceManager.RegisterScripts()` in `App.razor`.

See `references/platform-setup/blazor.md` for all three setups with full `Program.cs`, `_Imports.razor`, viewer/designer page markup, CSS links, and a troubleshooting table.

## 👁️ Previewing Reports

### WinForms

```csharp
using var printTool = new ReportPrintTool(new SalesReport { DataSource = GetData() });
printTool.ShowRibbonPreviewDialog(); // modal
```

### WPF

```csharp
// Assign to DocumentPreviewControl.DocumentSource in XAML binding or code-behind
preview.DocumentSource = new SalesReport { DataSource = GetData() };
```

### ASP.NET Core — Viewer Tag Helper

```html
<!-- Razor Pages or MVC View — after completing Program.cs setup -->
<dx-report-viewer id="viewer" report-url="SalesReport"></dx-report-viewer>
```

Reports are served via `ReportStorageWebExtension` — register a custom one for production (see End-User Designer section).

### Blazor — Native Viewer

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

## 📤 Exporting Without Preview

```csharp
var report = new SalesReport();
report.DataSource      = GetData();
report.RequestParameters = false; // skip parameter dialog

// Synchronous — OK in WinForms/WPF, avoid in web apps
report.CreateDocument();
report.ExportToPdf("output.pdf");
report.ExportToXlsx("output.xlsx");
report.ExportToDocx("output.docx");
report.ExportToHtml("output.html");
report.ExportToCsv("output.csv");
```

### Async Export (ASP.NET Core / Blazor — Always Use)

```csharp
// Controller or minimal API
var report = new SalesReport();
report.DataSource        = await GetDataAsync();
report.RequestParameters = false;

await report.CreateDocumentAsync(); // never use synchronous CreateDocument() on web

using var stream = new MemoryStream();
report.ExportToPdf(stream);
return File(stream.ToArray(), "application/pdf", "report.pdf");
```

Never call synchronous `CreateDocument()` in ASP.NET Core or Blazor — it blocks a thread pool thread and degrades performance under load.

## 🎯 Report Parameters at Runtime

```csharp
report.Parameters["StartDate"].Value = new DateTime(2025, 1, 1);
report.Parameters["Region"].Value    = "North";
report.RequestParameters             = false; // suppress the parameter input form
```

## 🖊️ End-User Report Designer

### WinForms

```csharp
var designTool = new ReportDesignTool(new SalesReport());
designTool.ShowRibbonDesignerDialog();
```

### ASP.NET Core

```html
<dx-report-designer report-url="SalesReport"></dx-report-designer>
```

### Blazor

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

### ReportStorageWebExtension (Required for Web Designer)

The web viewer and designer load/save reports via `ReportStorageWebExtension`. Without a custom implementation, reports can only be served from assembly type names. For production, register a custom storage:

```csharp
// Program.cs
builder.Services.AddScoped<ReportStorageWebExtension, CustomReportStorage>();
```

```csharp
public class CustomReportStorage : ReportStorageWebExtension {
    public override bool CanSetData(string url) => true;
    public override bool IsValidUrl(string url) => true;
    public override byte[] GetData(string url) {
        // Load report layout from disk, database, etc.
        return File.ReadAllBytes(Path.Combine("Reports", url + ".repx"));
    }
    public override Dictionary<string, string> GetUrls() =>
        Directory.GetFiles("Reports", "*.repx")
            .ToDictionary(f => Path.GetFileNameWithoutExtension(f), f => f);
    public override void SetData(XtraReport report, string url) {
        report.SaveLayoutToXml(Path.Combine("Reports", url + ".repx"));
    }
    public override string SetNewData(XtraReport report, string defaultUrl) {
        SetData(report, defaultUrl);
        return defaultUrl;
    }
}
```

See `references/platform-setup/report-storage.md` for a full implementation.

## ⚡ Performance

### SqlDataSource Schema Caching

`SqlDataSource` fetches the entire database schema on every report preview by default. In production this adds significant latency. Cache the schema using `DBSchemaProviderEx` backed by a file — query the DxDocs MCP server for the current setup pattern:

```
devexpress_docs_search: "DBSchemaProviderEx cache file SqlDataSource performance"
```

### Image URLs

Binding report controls to images by URL is fine for most scenarios but adds one network request per image per document generation. This can add latency if the image source is slow or behind authentication. For maximum performance, prefer embedding images directly in the report or loading them from a fast local endpoint.

### Document Source Caching (Web)

The Web Document Viewer automatically caches generated documents between paginated viewer requests. Enable `UseCachedReportSourceBuilder()` to generate documents page-by-page into storage rather than fully in memory — recommended for production and clustered/Azure deployments:

```csharp
// Program.cs
builder.Services.ConfigureReportingServices(configurator => {
    configurator.ConfigureWebDocumentViewer(viewerConfigurator => {
        viewerConfigurator.UseCachedReportSourceBuilder();
    });
});
```

Only call `DisableCachedDocumentSource()` if you run into limitations with services that need access to the live `XtraReport` instance during document generation (e.g., `PageInfoDataProviderBase`).

### Async Always in Web

Always `await report.CreateDocumentAsync()` in ASP.NET Core and Blazor. Synchronous `CreateDocument()` blocks a thread pool thread and degrades performance under load.

## 🔒 Security Essentials (Web)

Apply `[Authorize]` to the reporting controller when authentication is required. Add `[IgnoreAntiforgeryToken]` on `Invoke()` if the app uses global anti-forgery validation (prevents HTTP 400 errors). For user-level document isolation, implement `IWebDocumentViewerAuthorizationService` and `IExportingAuthorizationService`.

For CSP-protected apps, pass a cryptographic nonce to `.Nonce(nonce)` on the viewer builder.

For Angular/React apps, attach Bearer tokens via `fetchSetup.fetchSettings` from `@devexpress/analytics-core/analytics-utils`.

See `references/security-and-best-practices.md` for all code patterns.

## 🩺 Troubleshooting (Web — First Steps)

1. **Blank viewer** → check `UseDevExpressControls()` is before `UseStaticFiles()` in `Program.cs`
2. **"Unable to process binding"** → check script registration order (jQuery → Knockout → Bootstrap → DevExtreme → Analytics Core → Reporting)
3. **"Internal Server Error" popup (but Network shows 200)** → server exception caught by DevExpress; enable `DevExpress` log category at `Debug` level in `appsettings.json`
4. **400 on DXXRDV** → anti-forgery token rejecting the request; add `[IgnoreAntiforgeryToken]` to the controller
5. **404 on DXXRDV** → missing `WebDocumentViewerController`, or `MapDefaultControllerRoute()` absent
6. **CORS errors** → `UseCors()` must be after `UseRouting()` and before `UseEndpoints()`
7. **Version mismatch** → npm and NuGet versions must match; enable `configurator.UseDevelopmentMode()` to see details

Enable Development Mode for diagnostics:
```csharp
builder.Services.ConfigureReportingServices(c => c.UseDevelopmentMode());
```

See `references/troubleshooting-and-diagnostics.md` for the full symptom→fix table, logging setup, CSRF fix, timeout configuration, and web farm guidance.

## 🚫 Never Do

- `CreateDocument()` in ASP.NET Core or Blazor — always use `await CreateDocumentAsync()`
- Skip `AddDevExpressControls()` + `UseDevExpressControls()` in ASP.NET Core — the viewer won't work
- Skip `AddDevExpressServerSideBlazorReportViewer()` or `AddDevExpressBlazorReporting()` in Blazor — viewer won't render
- Skip `DxResourceManager.RegisterScripts()` in `App.razor` — no client-side scripts load
- Skip npm packages for ASP.NET Core reporting — viewer has no JavaScript
- `@using` directives without `_Imports.razor` — components won't resolve
- Aggregate functions per-record in `DetailBand` — extremely slow
- Omit `DevExpress.Drawing.Skia` NuGet package when targeting Linux or macOS — reports throw `System.DllNotFoundException` at runtime without it
- Apply `ReportStorageWebExtension` or `IReportProvider` in WinForms/WPF — these are web-only (ASP.NET Core and Blazor). Desktop apps load reports directly as instances.
- Miss `ReportStorageWebExtension` for the web End-User Report Designer — the designer won't be able to save/load reports
- Attempt toolbar or parameter editor UI customization with a Reporting-only subscription (WinForms/WPF) — these require the WinForms/WPF UI subscription
- Use `IReportProvider` when the End-User Designer is also in the project — use `ReportStorageWebExtension` instead (or both, with distinct URL namespaces)

## 📁 Reference Files

**Platform setup — read the relevant one before writing any integration code:**
- `references/platform-setup/winforms-wpf.md` — WinForms: `ReportPrintTool`, embedded `DocumentViewer`, export, End-User Designer. WPF: `DocumentPreviewControl` XAML, `PrintHelper`.
- `references/platform-setup/aspnet-core-mvc.md` — Full ASP.NET Core MVC setup: NuGet + npm + `bundleconfig.json` + `libman.json`, `Program.cs` with `UseCachedReportSourceBuilder`, reporting controller, `_ViewImports`, `_Layout`, viewer view (direct bind and via `IReportProvider`).
- `references/platform-setup/aspnet-core-razor-pages.md` — Razor Pages variant: `AddMvcCore()` requirement, `vendor.js` for Knockout, `bundleconfig.json` with `vendor.js`, `Pages/Viewer.cshtml` with `@page` directive.
- `references/platform-setup/angular-react.md` — Angular and React (Next.js / Vite) frontend + ASP.NET Core backend with CORS. Exact npm packages for each, `app.ts` / `page.tsx` code, `invokeAction` endpoint values, Angular budget settings.
- `references/platform-setup/blazor.md` — All three Blazor component families: native `DxReportViewer` (viewer only), JS-based `DxDocumentViewer`/`DxReportDesigner` (Server), and WASM `DxWasmDocumentViewer`/`DxWasmReportDesigner`. Separate `Program.cs` for each, CSS links, troubleshooting table.
- `references/platform-setup/linux-macos.md` — **Required for Linux/macOS**: `DevExpress.Drawing.Skia` NuGet package, OS-level dependencies (`libicu`, `libfontconfig1`), font installation, Docker setup, Azure App Service notes, and `DllNotFoundException` troubleshooting.
- `references/platform-setup/report-storage.md` — `ReportStorageWebExtension` full implementation (web-only). `IReportProvider` and async `IReportProviderAsync` for viewer-only scenarios. Notes on when to use each.

**Cross-platform features:**
- `references/export-and-parameters.md` — All export format methods, `PdfExportOptions`/`XlsxExportOptions`, returning file from ASP.NET Core controller, parameter injection before export, watermarks, Excel quality tips.
- `references/viewer-toolbar-customization.md` — WinForms (`PrintingSystemCommand`, `ExportOptions.Suppress`; subscription note), WPF (`DocumentCommandProvider`), ASP.NET Core (`CustomizeMenuActions`, `CustomizeExportOptions` JS events + SVG icon templates), Blazor JS-based (`DxDocumentViewerCallbacks` + external JS), native `DxReportViewer` (`ViewerToolbarSettings`).
- `references/custom-parameter-editors.md` — WinForms (`ParametersRequestBeforeShow` + `BaseEdit`), WPF (`ParameterTemplateSelector`), ASP.NET Core (`CustomizeParameterEditors` + inline Knockout template), Blazor native viewer (`OnCustomizeParameters` + `ValueTemplate`, standalone editor pattern with `OnSubmitParameters`).

**Troubleshooting, security and advanced tasks:**
- `references/troubleshooting-and-diagnostics.md` — Quick symptom→fix table (blank viewer, "Unable to process binding", "Something went wrong", "Internal Server Error", "Report not found", 400/401/404/415/500, CORS, version mismatch). Development Mode, `appsettings.json` logging, `LoggerOptions`, script registration order, CSRF, `StorageCleanerSettings` timeout, web farm shared storage, production deployment checklist, Network tab diagnostic guide.
- `references/security-and-best-practices.md` — User authorization (`IWebDocumentViewerAuthorizationService`, `IExportingAuthorizationService`, `WebDocumentViewerOperationLogger`), `[Authorize]` on controller, CSRF (`[IgnoreAntiforgeryToken]`), CSP nonce pattern, token-based auth (`fetchSetup`). Async engine, memory optimization, DB connection management (`IConnectionProviderFactory`), exception handling (`IWebDocumentViewerExceptionHandler`). Advanced viewer tasks: Tab Panel, Export Options panel, Search Panel (async search, F-key hotkey), Document Map, Document Settings (zoom, render format), Clipboard separator, Parameters Panel (validation, lookup source, `GetParametersModel`), multi-page mode, document navigation. Designer: Toolbox, Properties Panel, Field List, `AllowAddDataSource`. Localization, skeleton screen, version upgrade checklist.

**Complete working examples:**
- `references/examples/aspnet-core-viewer-page.md` — Complete ASP.NET Core MVC app: `Program.cs`, `IReportProvider`, reporting controller, `_ViewImports`, `_Layout`, direct-bind and model-bind viewer views, all `package.json` npm packages, troubleshooting.
- `references/examples/blazor-viewer-page.md` — Complete Blazor Server app: `Program.cs` for both native and JS-based variants, `App.razor`, `_Imports.razor`, viewer pages, optional JS customization file, troubleshooting.

## 🔍 Useful MCP Queries

- `"AddDevExpressControls UseDevExpressControls ASP.NET Core Program.cs"` — web reporting setup
- `"AddDevExpressBlazorReporting UseDevExpressBlazorReporting Blazor Program.cs"` — Blazor setup
- `"DxReportViewer OpenReportAsync Blazor"` — Blazor native viewer usage
- `"ReportStorageWebExtension GetData SetData web designer"` — report storage
- `"IReportProvider GetReport ASP.NET Core viewer"` — viewer-only report provider
- `"CreateDocumentAsync async export web"` — async document creation
- `"UseCachedReportSourceBuilder ConfigureReportingServices ASP.NET Core"` — document caching configuration
- `"DBSchemaProviderEx cache SqlDataSource performance"` — schema caching
- `"CustomizeMenuActions ActionId hide add button toolbar"` — web viewer toolbar
- `"CustomizeExportOptions HideFormat ExportFormatID"` — hide export formats web
- `"PrintingSystemCommand SetCommandVisibility WinForms"` — WinForms toolbar commands
- `"CustomizeParameterEditors ASP.NET Core"` — web parameter editor
- `"OnCustomizeParameters Blazor ValueTemplate ParameterModel"` — Blazor parameter editor
- `"IWebDocumentViewerAuthorizationService IExportingAuthorizationService"` — user authorization
- `"IgnoreAntiforgeryToken reporting controller 400"` — CSRF / anti-forgery fix
- `"Nonce CSP content security policy web document viewer"` — CSP nonce
- `"fetchSetup fetchSettings Bearer token Angular React"` — token-based auth
- `"IWebDocumentViewerExceptionHandler logging DevExpress"` — custom exception handling
- `"StorageCleanerSettings timeout report not found"` — document cache timeout
- `"UseAsyncSearch SearchEnabled WebDocumentViewerSearchSettings"` — search panel settings
- `"ClipboardSeparator reporting viewer"` — clipboard separator
- `"CustomizeLocalization LoadMessages viewer designer"` — UI localization

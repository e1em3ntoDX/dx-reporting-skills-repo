# Security, Best Practices & Advanced Viewer Tasks

## Security

### User Authorization

To restrict access to specific reports and documents, implement `IWebDocumentViewerAuthorizationService` and `IExportingAuthorizationService`. These services gate which users can open, build, and export which reports.

A common pattern derives from `WebDocumentViewerOperationLogger` to track document/report/user associations, then checks ownership in the authorization interfaces:

```csharp
using DevExpress.XtraReports.Web.WebDocumentViewer;

public class ReportAuthorizationService :
    WebDocumentViewerOperationLogger,
    IWebDocumentViewerAuthorizationService,
    IExportingAuthorizationService
{
    static readonly ConcurrentDictionary<string, string> DocumentOwners = new();
    static readonly ConcurrentDictionary<string, string> ReportOwners   = new();

    readonly IHttpContextAccessor _httpContext;
    public ReportAuthorizationService(IHttpContextAccessor httpContext)
        => _httpContext = httpContext;

    string CurrentUserId =>
        _httpContext.HttpContext?.User?.Identity?.Name ?? "anonymous";

    public override void ReportOpening(string reportId, string documentId, XtraReport report) {
        ReportOwners.TryAdd(reportId, CurrentUserId);
        DocumentOwners.TryAdd(documentId, CurrentUserId);
        base.ReportOpening(reportId, documentId, report);
    }

    public bool CanOpenReport(string reportId) =>
        ReportOwners.TryGetValue(reportId, out var owner) && owner == CurrentUserId;

    public bool CanReadDocument(string documentId) =>
        DocumentOwners.TryGetValue(documentId, out var owner) && owner == CurrentUserId;

    public bool CanExportDocument(string documentId) => CanReadDocument(documentId);

    // Implement remaining interface members similarly
}
```

```csharp
// Program.cs
builder.Services.AddScoped<ReportAuthorizationService>();
builder.Services.AddScoped<IWebDocumentViewerAuthorizationService>(
    sp => sp.GetRequiredService<ReportAuthorizationService>());
builder.Services.AddScoped<IExportingAuthorizationService>(
    sp => sp.GetRequiredService<ReportAuthorizationService>());
builder.Services.AddScoped<WebDocumentViewerOperationLogger>(
    sp => sp.GetRequiredService<ReportAuthorizationService>());
```

Full example: https://github.com/DevExpress-Examples/AspNetCore.Reporting.BestPractices#implement-user-authorization

### Protect Controller Actions

Apply `[Authorize]` to the reporting controller to require authentication before any viewer API call:

```csharp
[Authorize]
public class CustomWebDocumentViewerController : WebDocumentViewerController {
    public CustomWebDocumentViewerController(
        IWebDocumentViewerMvcControllerService controllerService)
        : base(controllerService) { }
}
```

### CSRF Protection

If your application uses ASP.NET Core's anti-forgery middleware globally, reporting endpoints will receive HTTP 400 errors unless you exclude them. The simplest approach is to mark the reporting controller action to ignore anti-forgery:

```csharp
public class CustomWebDocumentViewerController : WebDocumentViewerController {
    public CustomWebDocumentViewerController(
        IWebDocumentViewerMvcControllerService controllerService)
        : base(controllerService) { }

    [IgnoreAntiforgeryToken]
    public override Task<IActionResult> Invoke() => base.Invoke();
}
```

Full best-practices example: https://github.com/DevExpress-Examples/AspNetCore.Reporting.BestPractices#prevent-cross-site-request-forgery

### Content Security Policy (CSP)

If your application enforces CSP, the Document Viewer and Report Designer need a nonce for inline scripts. Generate a cryptographic nonce in the controller, pass it to the viewer, and add it to the CSP header:

```csharp
public async Task<IActionResult> Viewer(
    [FromServices] IWebDocumentViewerClientSideModelGenerator modelGenerator,
    [FromQuery] string reportName = "SalesReport") {

    var nonceBytes = new byte[32];
    using var rng = RandomNumberGenerator.Create();
    rng.GetBytes(nonceBytes);
    var nonce = Convert.ToBase64String(nonceBytes);

    Response.Headers["Content-Security-Policy"] =
        $"script-src 'self' 'nonce-{nonce}';" +
        "img-src 'self' data:;" +
        "style-src 'self';" +
        "connect-src 'self';" +
        "worker-src 'self' blob:;" +   // required for printing
        "frame-src 'self' blob:;";     // required for printing

    var viewerModel = await modelGenerator.GetModelAsync(
        reportName, WebDocumentViewerController.DefaultUri);
    return View((viewerModel, nonce));
}
```

```cshtml
@* Viewer.cshtml *@
@model (WebDocumentViewerModel Model, string Nonce)

@Html.DevExpress().WebDocumentViewer("DocumentViewer")
    .Height("calc(100vh - 60px)")
    .Nonce(Model.Nonce)
    .Bind(Model.Model)
```

Example: https://github.com/DevExpress-Examples/reporting-asp-net-core-content-security-policy

### Token-Based Authentication (Angular / React)

Pass a Bearer token with every viewer request using the `fetchSetup` from analytics-core:

```typescript
// app.ts / main.tsx — set once at app startup
import { fetchSetup } from '@devexpress/analytics-core/analytics-utils';

fetchSetup.fetchSettings = {
    headers: {
        'Authorization': `Bearer ${authToken}`
    }
};
```

Full example: https://github.com/DevExpress-Examples/AspNetCore.Reporting.BestPractices#token-based-authentication

---

## Performance Best Practices

### Async Engine

Enable the async engine to avoid blocking thread pool threads during document generation (especially under load):

```csharp
builder.Services.ConfigureReportingServices(configurator => {
    configurator.UseAsyncEngine();
    configurator.ConfigureWebDocumentViewer(c => c.UseCachedReportSourceBuilder());
});
```

Then use the async model generator in your controller:

```csharp
public async Task<IActionResult> Viewer(
    [FromServices] IWebDocumentViewerClientSideModelGenerator modelGenerator,
    [FromQuery] string reportName = "SalesReport") {
    var model = await modelGenerator.GetModelAsync(
        reportName, WebDocumentViewerController.DefaultUri);
    return View(model);
}
```

### Memory Optimization

`UseCachedReportSourceBuilder()` generates documents page-by-page into storage, significantly reducing peak memory usage for large reports compared to the default in-memory generation. See https://github.com/DevExpress-Examples/AspNetCore.Reporting.BestPractices#optimize-memory-consumption.

### Database Connection Management

When using `SqlDataSource`, resolve the connection string at runtime rather than embedding it in the report layout. Implement `IConnectionProviderFactory` to supply connection strings from your app's configuration:

```csharp
public class CustomConnectionProviderFactory : IConnectionProviderFactory {
    readonly IConfiguration _config;
    public CustomConnectionProviderFactory(IConfiguration config) => _config = config;

    public IDataConnectionParametersService Create(XtraReport report, string dataSourceName) {
        return new CustomDataConnectionParametersService(_config);
    }
}

builder.Services.AddScoped<IConnectionProviderFactory, CustomConnectionProviderFactory>();
```

See https://github.com/DevExpress-Examples/AspNetCore.Reporting.BestPractices#manage-database-connections

### Exception Handling

Log server-side reporting errors by configuring the `DevExpress` log category. The simplest way:

```json
// appsettings.json
"Logging": {
  "LogLevel": {
    "DevExpress": "Debug"
  }
}
```

For structured or custom error handling (e.g., to send errors to Sentry, Application Insights, etc.), implement a custom `IWebDocumentViewerExceptionHandler`:

```csharp
public class CustomExceptionHandler : IWebDocumentViewerExceptionHandler {
    public void HandleException(Exception ex) {
        // log to your preferred sink
        MyLogger.Error(ex, "Reporting error");
    }
}

builder.Services.AddScoped<IWebDocumentViewerExceptionHandler, CustomExceptionHandler>();
```

See https://github.com/DevExpress-Examples/AspNetCore.Reporting.BestPractices#handle-exceptions

---

## Advanced Viewer Tasks (Quick Reference)

All items in this section apply to the ASP.NET Core `WebDocumentViewer` tag helper / HTML helper. Wire up client-side events via `ClientSideEvents(configure => configure.EventName("functionName"))`.

### Tab Panel

```javascript
// Collapse the tab panel on document load
function onDocumentReady(s, e) {
    var previewModel = s.GetPreviewModel();
    if (previewModel)
        previewModel.tabPanel.collapsed(true);
}
```

```javascript
// Remove the tab panel entirely
function onCustomizeElements(s, e) {
    var tabPanel = e.GetById(DevExpress.Reporting.Viewer.PreviewElements.RightPanel);
    var idx = e.Elements.indexOf(tabPanel);
    if (idx >= 0) e.Elements.splice(idx, 1);
}
```

### Search Panel

```javascript
// Disable the search panel and hide the toolbar search button
// Server-side in Program.cs:
builder.Services.ConfigureReportingServices(configurator => {
    configurator.ConfigureWebDocumentViewer(c => {
        c.UseSearchSettings(settings => {
            settings.SearchEnabled = false;
        });
    });
});
```

### Parameters Panel

```javascript
// Hide the "Waiting for parameter values…" message text
function onCustomizeLocalization(s, e) {
    e.LoadMessages({ "Web.DocumentViewer.WaitingForParameterValues": "Select filters and click Submit" });
}

// Initialize a parameter value before the preview shows
function onParametersInitialized(s, e) {
    var params = e.Parameters;
    var regionParam = params.filter(p => p.name === "Region")[0];
    if (regionParam) regionParam.value("North");
}

// Auto-submit parameters without showing the panel
// Set report.Parameters["MyParam"].Visible = false for all parameters
```

### Document Navigation

```javascript
// Navigate to a specific page after the document loads
function onDocumentReady(s, e) {
    var model = s.GetPreviewModel();
    if (model) model.GoToPage(3); // 0-based page index
}
```

### Enable Multi-Page Mode

```javascript
function onBeforeRender(s, e) {
    var model = s.GetPreviewModel();
    if (model) model.showMultipagePreview(true);
}
```

### Export Options Panel

```javascript
// Hide the entire Export Options side panel
function onCustomizeExportOptions(s, e) {
    e.HideExportOptionsPanel();
}
```

Wire up in Razor:
```cshtml
@Html.DevExpress().WebDocumentViewer("DocumentViewer")
    .ClientSideEvents(e => e.CustomizeExportOptions("onCustomizeExportOptions"))
    .Bind("SalesReport")
```

Note: the "Show Print Dialog on Open" PDF export option has **no effect** in modern browsers because browsers block PDF scripts. Use server-side printing or instruct users to use the browser's native print dialog instead.

### Search Panel

```javascript
// Remove the search panel entirely (server-side, Program.cs)
builder.Services.ConfigureReportingServices(configurator => {
    configurator.ConfigureWebDocumentViewer(c => {
        c.UseSearchSettings(settings => {
            settings.SearchEnabled   = false;
            settings.UseAsyncSearch = true;  // enable async search for large documents
        });
    });
});
```

To disable only the **F** hotkey that expands the search panel without removing it:
```javascript
function onCustomizeMenuActions(s, e) {
    var searchAction = e.GetById(DevExpress.Reporting.Viewer.ActionId.Search);
    if (searchAction) searchAction.hotKey = null;
}
```

### Document Map Panel

```javascript
// Automatically show the Document Map panel when the report finishes loading
function onDocumentReady(s, e) {
    var previewModel = s.GetPreviewModel();
    if (previewModel) {
        var docMapTab = previewModel.tabPanel.tabs
            .find(t => t.name === DevExpress.Reporting.Viewer.PreviewElements.DocumentMap);
        if (docMapTab) docMapTab.active(true);
    }
}
```

### Document Settings (Zoom, Render Format)

```javascript
// Set the initial zoom level to fit the whole page
function onDocumentReady(s, e) {
    var preview = s.GetReportPreview();
    if (preview) preview.zoom = 0.75;  // 75% zoom
}
```

Specify the page render format server-side (affects preview quality vs. performance):
```csharp
// Program.cs — set render format to SVG (default is HTML)
builder.Services.ConfigureReportingServices(configurator => {
    configurator.ConfigureWebDocumentViewer(c => {
        c.UseDefaultPreviewSettings(settings => {
            settings.PreviewRenderFormat = "SVG";
        });
    });
});
```

### Clipboard Separator

When users select cells in the viewer and copy to clipboard, values are separated by a tab by default. To change this:

```typescript
// Angular / React — set at startup
import { ClipboardSeparator } from 'devexpress-reporting/dx-webdocumentviewer';
ClipboardSeparator(','); // use comma instead of tab
```

### Parameters Panel — Validation and Lookup

```javascript
// Validate parameter input (make a parameter required)
function onCustomizeParameterEditors(s, e) {
    if (e.parameter.name === 'StartDate') {
        e.info.validationRules = [{
            type: 'required',
            message: 'Start Date is required.'
        }];
    }
}

// Select the first item in a lookup value list automatically
function onCustomizeParameterLookUpSource(s, e) {
    if (e.parameter.name === 'Region') {
        e.dataSource.on('changed', function() {
            var items = e.dataSource.items();
            if (items && items.length > 0)
                e.parameter.value(items[0].value);
        });
    }
}

// Get and modify parameter values programmatically after the viewer loads
function onDocumentReady(s, e) {
    var previewModel = s.GetPreviewModel();
    var paramsModel  = previewModel.GetParametersModel();
    var startDate    = paramsModel.visibleParameters().find(p => p.path === 'StartDate');
    if (startDate) startDate.value(new Date());
    paramsModel.submit();
}
```

### Designer-Side Tasks

```javascript
// Remove a control type from the Designer Toolbox
function onCustomizeToolbox(s, e) {
    var controls = e.ControlsFactory;
    controls.unregisterControl("XRLabel");
}

// Add/remove Designer menu/toolbar commands
function onCustomizeMenuActions(s, e) {
    var saveBtn = e.GetById(DevExpress.Reporting.Designer.ActionId.Save);
    if (saveBtn) saveBtn.visible = false;
}

// Prevent users from adding new data sources in the Designer
// Server-side:
builder.Services.ConfigureReportingServices(configurator => {
    configurator.ConfigureReportDesigner(designerConfigurator => {
        designerConfigurator.RegisterDataSourceSettings(new ReportDesignerDataSourceSettings {
            AllowAddDataSource = false
        });
    });
});
```

### Designer — Properties Panel

```javascript
// Show Quick Actions in the Properties panel
function onBeforeRender(s, e) {
    var designer = s;
    designer.GetPreviewModel(); // wait for model
    DevExpress.Reporting.Designer.Settings.QuickActionsVisible(true);
}

// Hide the Task group from the Properties panel
DevExpress.Reporting.Designer.Settings.TaskGroupVisible(false);

// Hide or disable a specific editor for a control type
function onCustomizeMenuActions(s, e) {
    var labelInfo = s.GetPropertyInfo('XRLabel', 'font');
    if (labelInfo) labelInfo.disabled = true;
}
```

Server-side: prevent users from editing parameter collection in the Designer:
```csharp
builder.Services.ConfigureReportingServices(configurator => {
    configurator.ConfigureReportDesigner(d => {
        d.RegisterParameterEditingSettings(new ReportDesignerParameterEditingSettings {
            AllowEditParameterCollection = false
        });
    });
});
```

---

## Localization

The viewer and designer UI strings can be replaced via the `CustomizeLocalization` client-side event:

```javascript
function onCustomizeLocalization(s, e) {
    e.LoadMessages({
        "Web.DocumentViewer.WaitingForParameterValues": "Please select filters and click Apply",
        "Web.ReportDesigner.SaveReport": "Save Layout"
    });
}
```

Wire up in Razor:
```cshtml
@Html.DevExpress().WebDocumentViewer("DocumentViewer")
    .ClientSideEvents(e => e.CustomizeLocalization("onCustomizeLocalization"))
    .Bind("SalesReport")
```

To disable globalization-dependent features (causes empty Language dropdown in localization editor), ensure `InvariantGlobalization` is **not** set to `true` in `runtimeconfig.template.json`:

```json
{ "configProperties": { "System.Globalization.Invariant": false } }
```

---

## Skeleton Screen (Loading Placeholder)

By default the viewer shows a blank area while the report generates. To show a skeleton/loading placeholder instead, render a placeholder div that is replaced when the viewer initializes:

```cshtml
<div id="viewer-placeholder" class="skeleton-screen">
    <div class="skeleton-toolbar"></div>
    <div class="skeleton-page"></div>
</div>

@Html.DevExpress().WebDocumentViewer("DocumentViewer")
    .Height("calc(100vh - 60px)")
    .ClientSideEvents(e => e.BeforeRender("function(s,e){ document.getElementById('viewer-placeholder').remove(); }"))
    .Bind("SalesReport")
```

Full skeleton screen example: https://github.com/DevExpress-Examples/AspNetCore.Reporting.BestPractices#prepare-skeleton-screen

---

## Version Upgrades

When upgrading DevExpress versions, update **both** NuGet and npm packages together:

1. Update NuGet packages for the solution to the new version.
2. Update `devextreme-dist`, `@devexpress/analytics-core`, `devexpress-reporting` (and `devexpress-reporting-angular` / `devexpress-reporting-react` if used) in `package.json` to the matching version, then run `npm install`.
3. If using `devexpress-richedit` (for `XRRichText` editing in the designer), update that package too.
4. Rebuild bundles (`bundleconfig.json`) after restoring packages.

NuGet version and npm version **must match** at major.minor level (e.g., both `25.2`). Mismatches produce runtime warnings and can cause viewer malfunctions.

Reference: https://docs.devexpress.com/XtraReports/402395/web-reporting/version-upgrade-guide

---

## Reference Links

- Best Practices repository: https://github.com/DevExpress-Examples/AspNetCore.Reporting.BestPractices
- Tasks & Solutions (full list): https://docs.devexpress.com/XtraReports/402406
- Authorized Access: https://docs.devexpress.com/XtraReports/402997
- Web Farm Support: https://docs.devexpress.com/XtraReports/5199
- Content Security Policy: https://docs.devexpress.com/XtraReports/404141

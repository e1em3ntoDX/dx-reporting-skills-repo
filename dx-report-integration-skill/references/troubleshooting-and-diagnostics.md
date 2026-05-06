# Web Reporting — Troubleshooting & Diagnostics

## Quick Symptom → Fix Table

| Symptom | Likely Cause | Fix |
|---|---|---|
| Viewer shows blank page, no errors | `UseDevExpressControls()` missing or after `UseStaticFiles()` | Place `app.UseDevExpressControls()` **before** `app.UseStaticFiles()` |
| "Unable to process binding" in browser console | Scripts registered in wrong order or duplicate registration | See Script Registration Order below |
| "Something went wrong" popup | Client-side JS error before `ko.applyBindings` | Open browser DevTools → Console; check `BeforeRender`, `CustomizeElements` event handlers |
| "Internal Server Error" popup | Server error during `DXXRDV` request (DevExpress caught it) | Enable server-side logging; check `DevExpress` log category |
| "Error when trying to populate the data source" | Report data source error on server | Check server logs — details hidden from client for security |
| "Report 'X' not found" in logs | Document expired from cache (default 2 hr) OR multi-server setup | Increase `StorageCleanerSettings` timeout or configure shared storage for web farms |
| 400 Bad Request on DXXRDV | Anti-forgery token validation rejecting the request | See CSRF section below |
| 401 Unauthorized on DXXRDV | Controller has `[Authorize]` attribute, user not authenticated | Apply standard ASP.NET authentication before reporting |
| 404 on DXXRDV / DXXRD | Missing reporting controller, or wrong `invokeAction`/`host` in JS app | Add `WebDocumentViewerController`, check `MapDefaultControllerRoute()` |
| 415 Unsupported Media Type | Content type mismatch between client and server | Call `configurator.UseRequestContentType(...)` to align |
| CORS errors in browser (Angular/React) | CORS policy not configured or `UseCors()` called in wrong position | See CORS section below |
| Version mismatch warnings | npm package versions don't match NuGet versions | Match all versions to same `25.x`; enable Development Mode to see details |
| Razor helper chain renders as plain text in the browser | Multi-line `@Html.DevExpress()` chain not wrapped in `@(...)` | Wrap in `@(Html.DevExpress()...Bind(...))` — see ASP.NET Core MVC setup |
| Designer fails: "Report not found: ReportName" with URL binding | `GetData()` doesn't handle in-memory/predefined reports by name | Extend `GetData` to resolve known report names as well as `.repx` files |
| `DevExpress.Analytics.Widgets is undefined` | `dx-analytics-core.js` loaded after `dx-querybuilder.js` or missing from designer bundle | Put `dx-analytics-core.min.js` as first entry in `designer.part.bundle.js` |
| Language dropdown empty in localization editor | `InvariantGlobalization = true` in runtime config | Set `InvariantGlobalization` to `false` in `runtimeconfig.json` |

---

## Enabling Development Mode (Version Diagnostics)

Development Mode logs warnings when npm package versions don't match NuGet package versions. Enable during development:

```csharp
// Program.cs
builder.Services.ConfigureReportingServices(configurator => {
    configurator.UseDevelopmentMode();
    // To disable only version checking:
    // configurator.UseDevelopmentMode(x => x.CheckClientLibraryVersions = false);
});
```

---

## Server-Side Logging (ASP.NET Core)

Configure the `DevExpress` log category in `appsettings.json`:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "None",
      "Microsoft.Hosting.Lifetime": "None",
      "DevExpress": "Debug"
    }
  }
}
```

Or configure at startup with custom prefix/timestamp:

```csharp
using DevExpress.AspNetCore.Reporting.Logging;

builder.Services.Configure<LoggerOptions>(logOptions => {
    logOptions.LogMachineName = true;
    logOptions.LogTimeStamp  = true;
    logOptions.Prefix        = "DX: ";
    logOptions.Suffix        = " [END]";
});
```

The log will show errors like "Report 'X' not found" with full detail, including which `IReportProvider` or `ReportStorageWebExtension` was called and why it failed.

For **Blazor (Server and WASM JS-Based)**, use the same ASP.NET Core logging since they share the backend.

---

## Script Registration Order

Scripts must be registered in this exact order (the final order after bundling matters):

1. **jQuery**
2. **Knockout**
3. **Bootstrap** (JS)
4. **DevExtreme** (`dx.all.js` + `dx.light.css`)
5. **Analytics Core** (`dx-analytics-core.js` + analytics CSS)
6. **DevExpress Reporting** (`dx-webdocumentviewer.js` + viewer CSS)
7. If using the Report Designer: `dx-reportdesigner.js` + designer CSS
8. If using the Query Builder: `dx-querybuilder.js`

Common mistakes:
- Knockout registered after DevExtreme → `ko.applyBindings` fails
- Third-party bundles registered twice (once in `_Layout.cshtml`, once in a view file) → duplicate binding errors
- npm version `25.1` with NuGet `25.2` → partial API incompatibility

In `bundleconfig.json`, the order within `inputFiles` arrays determines the final registration order.

---

## CORS Errors (Angular / React / JS Frontend)

CORS must be configured so `UseCors()` is called **after** `UseRouting()` and **before** any endpoint-mapping code:

```csharp
// Program.cs — correct order
app.UseRouting();
app.UseCors("AllowCorsPolicy");          // ← must be between UseRouting and UseEndpoints
app.UseDevExpressControls();
app.UseEndpoints(e => e.MapControllers());
```

If CORS errors persist even with correct configuration, the backend may be throwing an unhandled exception before the CORS middleware runs. The error response won't have CORS headers, causing the browser to show a CORS error even though the real problem is a 500. Enable detailed logging to see the actual server error.

To pass custom headers or credentials in fetch requests from Angular/React:

```typescript
// Angular / React — configure fetch settings globally
import { fetchSetup } from '@devexpress/analytics-core/analytics-utils';
fetchSetup.fetchSettings = {
    headers: { 'Authorization': 'Bearer ' + token },
    beforeSend: (requestParameters) => {
        requestParameters.credentials = 'include';
    }
};
```

---

## CSRF / Anti-Forgery (400 Bad Request)

ASP.NET Core's anti-forgery protection can reject viewer requests with HTTP 400. To prevent this, exclude the reporting controller endpoints from validation:

```csharp
// Option 1: Apply [IgnoreAntiforgeryToken] to the reporting controller
public class CustomWebDocumentViewerController : WebDocumentViewerController {
    [IgnoreAntiforgeryToken]
    public override Task<IActionResult> Invoke() => base.Invoke();
    // ...
}

// Option 2: Use a global anti-forgery filter but exclude reporting routes
services.AddControllersWithViews(options => {
    options.Filters.Add(new AutoValidateAntiforgeryTokenAttribute());
});
```

The [Best Practices repository](https://github.com/DevExpress-Examples/AspNetCore.Reporting.BestPractices#prevent-cross-site-request-forgery) contains a full example of properly configuring anti-forgery for reporting.

---

## "Something Went Wrong" Popup

This popup appears when the application is in development mode and a client-side JS error occurs — specifically errors in `BeforeRender`, `CustomizeMenuActions`, or `CustomizeElements` event handlers, or during the `ko.applyBindings` call. Open **DevTools → Console** to see the actual JavaScript exception.

To suppress the popup without fixing the error (not recommended for debugging), set `EnableClientSideDevelopmentMode` to `false`:

```csharp
// Program.cs
builder.Services.ConfigureReportingServices(configurator => {
    configurator.UseDevelopmentMode(x => x.EnableClientSideDevelopmentMode = false);
});
```

---

## "Internal Server Error" Popup

This popup appears when a server error occurs during a `DXXRDV` request — but the HTTP response code is still **200 OK** because DevExpress Reporting caught the exception internally. The client receives a successful response that contains an error payload, which the viewer displays as a popup.

Because the response is 200, the **Network tab won't show it as a failure**. To diagnose: enable server-side logging for the `DevExpress` category (see above) and check for exceptions logged there.

---

## Fetch Request Options Not Passed (Angular / React)

To attach headers, credentials, or other fetch options to every request the viewer makes to the backend (e.g., Bearer token, cookies), configure `fetchSetup` globally once at app startup:

```typescript
import { fetchSetup } from '@devexpress/analytics-core/analytics-utils';

fetchSetup.fetchSettings = {
    headers: { 'Authorization': 'Bearer ' + getToken() },
    beforeSend: (requestParameters) => {
        requestParameters.credentials = 'include'; // include cookies in cross-origin requests
    }
};
```

This is the correct way to pass credentials or custom headers — **do not** set these per-request on the viewer component.

---

## Production Deployment Checklist

If the app works in development but fails in production, verify:
- All DevExpress NuGet assemblies are included in the deployment output (not marked as "Copy if newer" only)
- npm-generated bundles in `wwwroot/` are included in publish output (check `bundleconfig.json` and publish settings)
- The `wwwroot/css/icons/` folder (from `libman.json`) is published — missing icon fonts cause broken UI
- npm and NuGet package versions match between dev and production builds
- `UseDevExpressControls()` is called in `Program.cs` and the application is restarted after deployment

---

## "Report 'X' Not Found" After Timeout

The Document Viewer keeps generated documents in a server-side cache. The default expiry is **2 hours**. If a user leaves the page open and returns after the timeout, interacting with the report (submitting parameters, exporting) causes "Report not found" errors.

To increase the timeout:

```csharp
// Register a custom StorageCleanerSettings service
builder.Services.AddSingleton<StorageCleanerSettings>(new StorageCleanerSettings {
    Dir    = Path.GetTempPath(),
    MaxAge = TimeSpan.FromHours(8)
});
```

---

## Multi-Server / Web Farm Deployments

When multiple app instances serve the same application (load balancer, Azure App Service with multiple instances), document storage must be shared. Each instance must be able to read documents generated by other instances.

```csharp
// Option A: Use distributed file storage (shared network path)
builder.Services.ConfigureReportingServices(configurator => {
    configurator.ConfigureWebDocumentViewer(viewerConfigurator => {
        viewerConfigurator.UseCachedReportSourceBuilder();
        // Point to shared storage
        viewerConfigurator.UseFileDocumentStorage("\\\\fileserver\\reports\\");
    });
});

// Option B: Use Azure Blob Storage
// viewerConfigurator.UseAzureCachedReportSourceBuilder();
// (requires DevExpress.AspNetCore.Reporting.Azure NuGet package)
```

Also share ASP.NET Core Data Protection keys across instances so session tokens decrypt correctly:

```csharp
builder.Services.AddDataProtection()
    .PersistKeysToFileSystem(new DirectoryInfo("\\\\fileserver\\dp-keys\\"));
```

---

## Network Tab Diagnostic Checklist

Open browser DevTools → **Network** tab, reload the page, and check these endpoints:

| Endpoint | Component | Expected Response |
|---|---|---|
| `DXXRDV` | Document Viewer | 200 OK |
| `DXXRD` | Report Designer | 200 OK |
| `DXXQB` | Query Builder (ASP.NET Core) | 200 OK |

Any 4xx/5xx response points to a server configuration problem. Select the failing request → **Preview** tab to see the error details.

If the preview is empty, enable the Developer Exception Page in `Program.cs`:
```csharp
if (app.Environment.IsDevelopment()) {
    app.UseDeveloperExceptionPage();
}
```

# ReportStorageWebExtension

The web Document Viewer and End-User Report Designer load and save reports by URL through a `ReportStorageWebExtension`. Without a custom implementation, the viewer can only load reports by assembly-qualified type name (`"Namespace.ClassName"`). For production, implement and register a custom storage.

## Registration

```csharp
// Program.cs — register before building the app
builder.Services.AddScoped<ReportStorageWebExtension, CustomReportStorage>();
```

## Minimal File-Based Implementation

```csharp
using DevExpress.XtraReports.UI;
using DevExpress.XtraReports.Web.Extensions;

public class CustomReportStorage : ReportStorageWebExtension {
    // Folder where .repx files are stored
    static readonly string ReportsFolder =
        Path.Combine(Directory.GetCurrentDirectory(), "Reports");

    // Called by the viewer to check if a URL is valid before loading
    public override bool IsValidUrl(string url) =>
        !string.IsNullOrEmpty(url) &&
        !url.Contains("..") &&              // prevent directory traversal
        url.IndexOfAny(Path.GetInvalidFileNameChars()) < 0;

    // Called by the viewer to load a report layout
    public override byte[] GetData(string url) {
        var path = Path.Combine(ReportsFolder, url + ".repx");
        if (!File.Exists(path))
            throw new FaultException($"Report not found: {url}");
        return File.ReadAllBytes(path);
    }

    // Called to populate the Open Report dialog in the designer
    public override Dictionary<string, string> GetUrls() =>
        Directory.EnumerateFiles(ReportsFolder, "*.repx")
            .ToDictionary(
                f => Path.GetFileNameWithoutExtension(f),
                f => Path.GetFileNameWithoutExtension(f));

    // Called by the designer when saving an existing report
    public override void SetData(XtraReport report, string url) {
        var path = Path.Combine(ReportsFolder, url + ".repx");
        report.SaveLayoutToXml(path);
    }

    // Called by the designer when saving a new report
    public override string SetNewData(XtraReport report, string defaultUrl) {
        SetData(report, defaultUrl);
        return defaultUrl;
    }

    // Allow save/overwrite — return false to make viewer read-only
    public override bool CanSetData(string url) => true;
}
```

## IReportProvider — Alternative for Viewer-Only Scenarios

When there is no Report Designer in the project, `IReportProvider` is the cleaner way to supply report instances by name to the viewer. Unlike `ReportStorageWebExtension`, it works with in-memory or code-based reports, not just `.repx` files.

```csharp
using DevExpress.XtraReports.Services;
using DevExpress.XtraReports.UI;

public class CustomReportProvider : IReportProvider {
    public XtraReport GetReport(string id, ReportProviderContext context) {
        return id switch {
            "SalesReport"   => new SalesReport(),
            "InvoiceReport" => new InvoiceReport(),
            _ => throw new FaultException($"Report '{id}' not found.")
        };
    }
}
```

```csharp
// Program.cs
builder.Services.AddScoped<IReportProvider, CustomReportProvider>();
```

```csharp
// Controller — generate viewer model by report name
public IActionResult Viewer(
    [FromServices] IWebDocumentViewerClientSideModelGenerator modelGenerator,
    [FromQuery] string reportName = "SalesReport")
{
    var viewerModel = modelGenerator.GetModel(
        reportName, WebDocumentViewerController.DefaultUri);
    return View(viewerModel);
}
```

```cshtml
@* View — bind viewer to the model from the controller *@
@model WebDocumentViewerModel
@Html.DevExpress().WebDocumentViewer("DocumentViewer")
    .Height("100%")
    .Bind(Model)
```

### Async Variant

For async report creation, implement `IReportProviderAsync` and use `GetModelAsync`:

```csharp
public class CustomReportProviderAsync : IReportProviderAsync {
    public Task<XtraReport> GetReportAsync(string id, ReportProviderContext ctx) {
        XtraReport report = id switch {
            "SalesReport" => new SalesReport(),
            _ => throw new FaultException($"Report '{id}' not found.")
        };
        return Task.FromResult(report);
    }
}
```

```csharp
builder.Services.AddScoped<IReportProviderAsync, CustomReportProviderAsync>();

// Activate async engine in Program.cs:
builder.Services.ConfigureReportingServices(configurator => {
    configurator.UseAsyncEngine();
});
```

```csharp
// Async controller action
public async Task<IActionResult> Viewer(
    [FromServices] IWebDocumentViewerClientSideModelGenerator modelGenerator,
    [FromQuery] string reportName = "SalesReport")
{
    var viewerModel = await modelGenerator.GetModelAsync(
        reportName, WebDocumentViewerController.DefaultUri);
    return View(viewerModel);
}
```

## Notes

- `ReportStorageWebExtension` is required when using the End-User Report Designer (to save/load).
- `IReportProvider` / `IReportProviderAsync` is sufficient for viewer-only scenarios.
- Both can coexist: the storage handles `.repx` files, the provider handles code-based reports.
- Registering both with the same name mapping can cause conflicts — keep their URL namespaces distinct.

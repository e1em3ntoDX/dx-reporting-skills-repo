# Export, Parameters, and Watermarks

## Export Format Methods

```csharp
report.CreateDocument();           // sync — WinForms/WPF only
await report.CreateDocumentAsync(); // async — always use in web

report.ExportToPdf("output.pdf");
report.ExportToXlsx("output.xlsx");
report.ExportToXls("output.xls");
report.ExportToDocx("output.docx");
report.ExportToRtf("output.rtf");
report.ExportToHtml("output.html");
report.ExportToMht("output.mht");
report.ExportToCsv("output.csv");
report.ExportToText("output.txt");
report.ExportToImage("output.png");  // exports all pages as images
```

All export methods accept a `string` path or `Stream`. Use `Stream` in web apps to return file content from a controller action.

## Export Options

Each format has a typed options class accessible via `report.ExportOptions`:

```csharp
// PDF with password protection
var pdfOptions = new PdfExportOptions {
    DocumentOptions = { Author = "Acme Corp", Title = "Sales Report" },
    PasswordSecurityOptions = {
        OpenPassword = "open123",
        PermissionsPassword = "edit456",
        PrintingPermissions = PdfPrintingPermissions.LowResolution
    }
};
report.ExportToPdf("output.pdf", pdfOptions);

// Excel with sheet name and cell merging disabled
var xlsxOptions = new XlsxExportOptions {
    SheetName = "Sales Data",
    ExportMode = XlsxExportMode.SingleFile,
    TextExportMode = TextExportMode.Value  // export values, not formatted text
};
report.ExportToXlsx("output.xlsx", xlsxOptions);

// Word
var docxOptions = new DocxExportOptions {
    ExportMode = DocxExportMode.SingleFile,
    PageRange = "1-3"  // only export pages 1–3
};
report.ExportToDocx("output.docx", docxOptions);
```

## ASP.NET Core — Return File from Controller

```csharp
[HttpGet("export/{reportName}")]
public async Task<IActionResult> ExportPdf(string reportName) {
    var report = GetReport(reportName); // your factory or DI
    report.RequestParameters = false;

    await report.CreateDocumentAsync();

    using var stream = new MemoryStream();
    report.ExportToPdf(stream);
    stream.Position = 0;
    return File(stream.ToArray(), "application/pdf", $"{reportName}.pdf");
}
```

## Setting Parameters Before Export

```csharp
// By name
report.Parameters["StartDate"].Value = new DateTime(2025, 1, 1);
report.Parameters["Region"].Value    = "North";
report.RequestParameters             = false; // don't show prompt UI

await report.CreateDocumentAsync();
```

## Watermarks

```csharp
// Text watermark
report.Watermark.Text       = "CONFIDENTIAL";
report.Watermark.Font       = new DevExpress.Drawing.DXFont("Arial", 48f);
report.Watermark.ForeColor  = System.Drawing.Color.FromArgb(40, System.Drawing.Color.Red);
report.Watermark.Angle      = 45;
report.Watermark.ShowBehind = true;   // behind content
report.DrawWatermark        = true;

// Image watermark
report.Watermark.Image          = Image.FromFile("logo.png");
report.Watermark.ImageViewMode  = WatermarkImageViewMode.Stretch;
report.DrawWatermark            = true;
```

Watermark expressions do **not** support data field references. For conditional watermarks (e.g., show "DRAFT" only on certain records), set the watermark in code in a `BeforePrint` event or before `CreateDocumentAsync`.

## Excel Export Quality Tips

For the cleanest Excel output:
- All report control borders must align on **vertical grid lines**. Any gap or misalignment between controls produces extra empty columns in the worksheet.
- Use `SnapToGridAndSnapLines` snapping in the Visual Studio designer:

```csharp
// In InitializeComponent():
this.SnappingMode = DevExpress.XtraReports.UI.SnappingMode.SnapToGridAndSnapLines;
this.SnapGridSize = 12.5F;
```

- Use `TextExportMode = TextExportMode.Value` in `XlsxExportOptions` when you need raw numeric values (for formulas and calculations) rather than formatted display strings.
- Set `XlsxExportOptions.ExportMode = XlsxExportMode.SingleFile` to merge all pages into one sheet.

## Export Without Viewer — WinForms/WPF

```csharp
// WinForms — headless export
var report = new SalesReport();
report.DataSource      = GetData();
report.RequestParameters = false;
report.CreateDocument();
report.ExportToPdf(@"C:\output\report.pdf");
report.Dispose();

// WPF — same, no viewer needed
```

# WinForms and WPF Platform Setup

## WinForms

### NuGet Package

```
DevExpress.Win.Reporting
```

No `Program.cs` or startup registration needed. The package includes `ReportPrintTool`, `DocumentViewer`, and `ReportDesignTool`.

### Preview — ReportPrintTool

```csharp
using DevExpress.XtraReports.UI;

// Modal ribbon preview (most common)
using var printTool = new ReportPrintTool(new SalesReport());
printTool.ShowRibbonPreviewDialog();

// Non-modal
printTool.ShowRibbonPreview();
```

### Preview — Embedded DocumentViewer Control

For hosting the preview inside a form:

```csharp
// In form Load or constructor
documentViewer1.DocumentSource = new SalesReport { DataSource = GetData() };
```

Add `DocumentViewer` from the **DX.25.x: Reporting** toolbox category. For .NET projects, assign `DocumentSource` in code — the designer drop-down doesn't list reports in .NET (only .NET Framework).

### Export Without Preview

```csharp
var report = new SalesReport();
report.DataSource      = GetData();
report.RequestParameters = false;
report.CreateDocument();
report.ExportToPdf(@"C:\Reports\output.pdf");
report.ExportToXlsx(@"C:\Reports\output.xlsx");
```

### End-User Report Designer

```csharp
// Open existing report
var designTool = new ReportDesignTool(new SalesReport());
designTool.ShowRibbonDesignerDialog();

// Open blank report for new design
var designTool = new ReportDesignTool(new XtraReport());
designTool.ShowRibbonDesignerDialog();
```

---

## WPF

### NuGet Package

```
DevExpress.Wpf.Reporting
```

### Preview — DocumentPreviewControl (XAML)

```xml
<!-- MainWindow.xaml -->
<Window xmlns:dxp="http://schemas.devexpress.com/winfx/2008/xaml/printing">
    <Grid>
        <dxp:DocumentPreviewControl
            x:Name="documentPreview"
            RequestDocumentCreation="True"
            DocumentSource="{Binding Report}" />
    </Grid>
</Window>
```

```csharp
// ViewModel
public XtraReport Report { get; } = new SalesReport { DataSource = GetData() };
```

`RequestDocumentCreation="True"` triggers document generation automatically when `DocumentSource` is set.

### Preview — PrintHelper (Code-behind)

```csharp
using DevExpress.Xpf.Printing;

// Show in a separate ribbon preview window
PrintHelper.ShowRibbonPrintPreview(this, new SalesReport());

// Modal dialog
PrintHelper.ShowRibbonPrintPreviewDialog(this, new SalesReport());
```

### Export Without Preview

```csharp
var report = new SalesReport();
report.DataSource      = GetData();
report.RequestParameters = false;
report.CreateDocument();
report.ExportToPdf("output.pdf");
```

### End-User Report Designer (WPF)

WPF uses a dedicated `ReportDesignTool` (available in `DevExpress.XtraReports.Design`):

```csharp
var designTool = new ReportDesignTool(new SalesReport());
designTool.ShowRibbonDesignerDialog();
```

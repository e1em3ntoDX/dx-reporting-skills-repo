# Viewer Toolbar Customization

## ⚠️ Subscription Requirement for Desktop (WinForms/WPF)

Toolbar and UI customization for the WinForms and WPF print preview is only available with the **WinForms**, **DXperience**, or **Universal** subscription — subscriptions that include DevExpress WinForms or WPF UI controls. The **Reporting-only** subscription does not support it.

If you only have the Reporting subscription, `ReportPrintTool` and `ReportDesignTool` display previews in their default configuration only.

---

## WinForms — Hide / Override Commands (Reporting Subscription)

Without the WinForms UI subscription, the main customization available via `PrintingSystemCommand`:

```csharp
using DevExpress.XtraPrinting;
using DevExpress.XtraReports.UI;

var printTool = new ReportPrintTool(new SalesReport());

// Hide specific commands (works without full WinForms subscription)
printTool.PrintingSystem.SetCommandVisibility(
    PrintingSystemCommand.DocumentMap, CommandVisibility.None);
printTool.PrintingSystem.SetCommandVisibility(
    PrintingSystemCommand.Thumbnails, CommandVisibility.None);
printTool.PrintingSystem.SetCommandVisibility(
    PrintingSystemCommand.ExportFile, CommandVisibility.None);

// CommandVisibility options: None, Toolbar, Menu, All

printTool.ShowRibbonPreviewDialog();
```

Available `PrintingSystemCommand` values include: `Print`, `PrintDirect`, `ExportFile`, `Find`, `DocumentMap`, `Thumbnails`, `Scale`, `ZoomIn`, `ZoomOut`, `FirstPage`, `PrevPage`, `NextPage`, `LastPage`, `StopPageBuilding`, and more.

## WinForms — Add Button to Ribbon Toolbar (WinForms UI Subscription Required)

When using an embedded `DocumentViewer` control (not `ReportPrintTool`):

1. In Visual Studio, click the `DocumentViewer` smart tag → **Create Ribbon Toolbar**.
2. In the generated `PrintRibbonController`, add a new `BarButtonItem` to the desired ribbon group.
3. Handle `ItemClick`:

```csharp
private void editBarButtonItem_ItemClick(object sender, ItemClickEventArgs e) {
    var designTool = new ReportDesignTool(new SalesReport());
    designTool.ShowRibbonDesignerDialog();
    documentViewer1.DocumentSource = new SalesReport(); // reload after edit
}
```

## WinForms — Hide Export Formats (PrintingSystem)

```csharp
// Suppress specific export formats from the Export menu
printTool.PrintingSystem.ExportOptions.Xls.Suppress  = true;
printTool.PrintingSystem.ExportOptions.Rtf.Suppress  = true;
printTool.PrintingSystem.ExportOptions.Html.Suppress = true;
// Formats: Pdf, Xls, Xlsx, Rtf, Docx, Html, Mht, Text, Csv, Image

printTool.ShowRibbonPreviewDialog();
```

---

## WPF — Customize Toolbar (WPF UI Subscription Required)

```xml
<!-- MainWindow.xaml -->
<dxp:DocumentPreviewControl x:Name="preview">
    <dxp:DocumentPreviewControl.CommandProvider>
        <dxp:DocumentCommandProvider>
            <!-- Remove built-in ribbon actions by clearing the collection -->
            <dxp:DocumentCommandProvider.RibbonActions>
                <!-- Add/remove RibbonActionBase descendants here -->
            </dxp:DocumentCommandProvider.RibbonActions>
        </dxp:DocumentCommandProvider>
    </dxp:DocumentPreviewControl.CommandProvider>
</dxp:DocumentPreviewControl>
```

For full WPF toolbar customization, consult: https://docs.devexpress.com/XtraReports/9400

---

## ASP.NET Core — Customize via Client-Side Events

Web viewer customization is client-side (JavaScript). Use `ClientSideEvents` builder in Razor:

### Hide a Built-In Toolbar Button

```cshtml
<script type="text/javascript">
    function customizeMenuActions(s, e) {
        // Hide Previous Page and Next Page buttons
        var prevPage = e.GetById(DevExpress.Reporting.Viewer.ActionId.PrevPage);
        if (prevPage) prevPage.visible = false;

        var nextPage = e.GetById(DevExpress.Reporting.Viewer.ActionId.NextPage);
        if (nextPage) nextPage.visible = false;
    }
</script>

@{
    var viewer = Html.DevExpress().WebDocumentViewer("DocumentViewer")
        .Height("1000px")
        .ClientSideEvents(e => e.CustomizeMenuActions("customizeMenuActions"))
        .Bind("SalesReport");
    @viewer.RenderHtml()
}
```

Common `ActionId` values: `Print`, `PrintPage`, `ExportTo`, `Search`, `Zoom`, `ZoomIn`, `ZoomOut`, `PrevPage`, `NextPage`, `FirstPage`, `LastPage`, `HighlightEditingFields`.

### Add a Custom Toolbar Button

```cshtml
<script type="text/html" id="myIcon">
    <svg viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
        <polygon class="dxd-icon-fill" points="4,2 4,22 22,12" />
    </svg>
</script>

<script type="text/javascript">
    function customizeMenuActions(s, e) {
        e.Actions.push({
            text: "My Action",
            imageTemplateName: "myIcon", // references the <script type="text/html"> id above
            visible: true,
            disabled: false,
            selected: ko.observable(false),
            hasSeparator: true,
            clickAction: function() {
                alert("Custom action triggered!");
            }
        });
    }
</script>
```

Use `imageClassName` instead of `imageTemplateName` to assign a CSS class with a background image.

### Hide Export Formats

```cshtml
<script type="text/javascript">
    function customizeExportOptions(s, e) {
        // Hide XLS and RTF from the Export dropdown
        e.HideFormat(DevExpress.Reporting.Viewer.ExportFormatID.XLS);
        e.HideFormat(DevExpress.Reporting.Viewer.ExportFormatID.RTF);
        e.HideFormat(DevExpress.Reporting.Viewer.ExportFormatID.HTML);
    }
</script>

@{
    var viewer = Html.DevExpress().WebDocumentViewer("DocumentViewer")
        .Height("1000px")
        .ClientSideEvents(e => e.CustomizeExportOptions("customizeExportOptions"))
        .Bind("SalesReport");
    @viewer.RenderHtml()
}
```

Available `ExportFormatID` values: `PDF`, `XLS`, `XLSX`, `RTF`, `DOCX`, `HTML`, `MHT`, `Text`, `CSV`, `Image`.

### Remove the Entire Toolbar

```cshtml
<script type="text/javascript">
    function onCustomizeElements(s, e) {
        var toolbar = e.GetById(DevExpress.Reporting.Viewer.PreviewElements.Toolbar);
        var idx = e.Elements.indexOf(toolbar);
        if (idx >= 0) e.Elements.splice(idx, 1);
    }
</script>

@{
    var viewer = Html.DevExpress().WebDocumentViewer("DocumentViewer")
        .Height("1000px")
        .ClientSideEvents(e => e.CustomizeElements("onCustomizeElements"))
        .Bind("SalesReport");
    @viewer.RenderHtml()
}
```

---

## Blazor (JS-Based DxDocumentViewer) — Customize via JavaScript + Callbacks

Blazor uses an external JS file for viewer customization:

```javascript
// wwwroot/viewer-customization.js
window.ViewerCustomization = {
    onCustomizeExportOptions: function(s, e) {
        e.HideFormat(DevExpress.Reporting.Viewer.ExportFormatID.XLS);
    },
    onCustomizeMenuActions: function(s, e) {
        var next = e.GetById(DevExpress.Reporting.Viewer.ActionId.NextPage);
        if (next) next.visible = false;
    }
};
```

Register the script in `App.razor`:

```razor
@DxResourceManager.RegisterScripts(config =>
    config.Register(new DxResource("/viewer-customization.js", 900)))
```

Wire up callbacks in the viewer component:

```razor
<DxDocumentViewer ReportName="SalesReport" Height="1000px" Width="100%">
    <DxDocumentViewerCallbacks
        CustomizeExportOptions="ViewerCustomization.onCustomizeExportOptions"
        CustomizeMenuActions="ViewerCustomization.onCustomizeMenuActions" />
</DxDocumentViewer>
```

---

## Blazor (Native DxReportViewer) — Toolbar Customization

The native `DxReportViewer` has a different API. Toolbar customization is done via the `ToolbarSettings` parameter:

```razor
<DxReportViewer @ref="reportViewer"
                Report="@Report">
    <ViewerToolbarSettings ShowExportButton="false"
                           ShowPrintButton="false" />
</DxReportViewer>
```

For hiding specific export formats in the native viewer, use `OnCustomizeParameters` plus `TabPanelModel` to hide the export panel, or use the JS-based `DxDocumentViewer` for more granular control.

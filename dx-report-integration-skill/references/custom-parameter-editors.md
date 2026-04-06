# Custom Parameter Editors

## ⚠️ Subscription Requirement

Custom parameter editors in the viewer UI require:
- **WinForms**: WinForms, DXperience, or Universal subscription
- **WPF**: WPF, DXperience, or Universal subscription
- **ASP.NET Core / Blazor**: ASP.NET & Blazor, DXperience, or Universal subscription

The **Reporting-only subscription** does not support parameter editor customization in the viewer UI.

---

## WinForms — Custom Parameter Editor

Handle `ParametersRequestBeforeShow` on the report and assign a `BaseEdit` descendant to `ParameterInfo.Editor`:

```csharp
using DevExpress.XtraEditors;
using DevExpress.XtraReports.Parameters;
using DevExpress.XtraReports.UI;

var report = new SalesReport();

report.ParametersRequestBeforeShow += (sender, e) => {
    foreach (ParameterInfo paramInfo in e.ParametersInformation) {
        if (paramInfo.Parameter.Name == "Region") {
            // Replace the default editor with a ComboBoxEdit
            var combo = new ComboBoxEdit();
            combo.Properties.Items.AddRange(new[] { "North", "South", "East", "West" });
            paramInfo.Editor = combo;
        }
        if (paramInfo.Parameter.Type == typeof(DateTime)) {
            // Use DateEdit for all date parameters
            paramInfo.Editor = new DateEdit();
        }
    }
};

using var printTool = new ReportPrintTool(report);
printTool.ShowRibbonPreviewDialog();
```

To resize custom editors:
```csharp
combo.MinimumSize = new System.Drawing.Size(200, 0);
combo.MaximumSize = new System.Drawing.Size(400, 0);
```

---

## WPF — Custom Parameter Editor (ParameterTemplateSelector)

Implement `DataTemplateSelector` and assign to `DocumentPreviewControl.ParametersPanel.ParameterTemplateSelector`:

```xml
<!-- MainWindow.xaml -->
<dxp:DocumentPreviewControl x:Name="preview">
    <dxp:DocumentPreviewControl.ParametersPanel>
        <dxprm:ParametersPanel>
            <dxprm:ParametersPanel.ParameterTemplateSelector>
                <local:CustomParameterTemplateSelector />
            </dxprm:ParametersPanel.ParameterTemplateSelector>
        </dxprm:ParametersPanel>
    </dxp:DocumentPreviewControl.ParametersPanel>
</dxp:DocumentPreviewControl>
```

```csharp
public class CustomParameterTemplateSelector : DataTemplateSelector {
    public DataTemplate IntTemplate { get; set; }
    public DataTemplate DefaultTemplate { get; set; }

    public override DataTemplate SelectTemplate(object item, DependencyObject container) {
        if (item is ParameterModel p && p.Type == typeof(int))
            return IntTemplate;
        return DefaultTemplate;
    }
}
```

Full example: https://docs.devexpress.com/XtraReports/17763

---

## ASP.NET Core — Custom Editor via CustomizeParameterEditors

Client-side customization using a DevExtreme widget or a custom HTML template:

### Modify an Existing Editor

```cshtml
<script type="text/javascript">
    function customizeParameterEditors(s, e) {
        // Change display format for all DateTime parameters
        if (e.parameter.type === 'System.DateTime') {
            e.info.editor = $.extend({}, e.info.editor);
            e.info.editor.extendedOptions = $.extend(
                e.info.editor.extendedOptions || {},
                { displayFormat: 'dd-MMM-yyyy' }
            );
        }
    }
</script>

@{
    var viewer = Html.DevExpress().WebDocumentViewer("DocumentViewer")
        .Height("1000px")
        .ClientSideEvents(e => e.CustomizeParameterEditors("customizeParameterEditors"))
        .Bind("SalesReport");
    @viewer.RenderHtml()
}
```

### Replace an Editor with a Custom HTML Template

```cshtml
@* Define a Knockout-based inline template *@
<script type="text/html" id="employeeID-custom-editor">
    <div data-bind="dxSelectBox: {
        dataSource: ['Smith', 'Jones', 'Brown'],
        value: value
    }"></script>
</script>

<script type="text/javascript">
    function customizeParameterEditors(s, e) {
        if (e.parameter.name === 'p_employeeID') {
            e.info.editor = { header: 'employeeID-custom-editor' };
        }
    }
</script>
```

Full example: https://docs.devexpress.com/XtraReports/403190

---

## Blazor Native Viewer — Custom Editor via OnCustomizeParameters

### Replace the Built-In Editor with a Razor Component

1. Create a custom editor component (`CustomCombobox.razor`):

```razor
<DxSpinEdit @bind-Value="@EditorValue" MinValue="0" MaxValue="100" />

@code {
    [Parameter] public object Value { get; set; }
    [Parameter] public EventCallback<object> ValueChanged { get; set; }

    int EditorValue {
        get => (int)(Value ?? 0);
        set { Value = value; ValueChanged.InvokeAsync(value); }
    }
}
```

2. Handle `OnCustomizeParameters` in the viewer page:

```razor
@page "/viewer"
@rendermode InteractiveServer
@using DevExpress.Blazor.Reporting
@using DevExpress.Blazor.Reporting.Models

<DxReportViewer @ref="reportViewer"
                Report="@Report"
                OnCustomizeParameters="OnCustomizeParameters" />

@code {
    DxReportViewer reportViewer;
    XtraReport Report = new SalesReport();

    void OnCustomizeParameters(ParametersModel parametersModel) {
        foreach (var param in parametersModel.VisibleItems) {
            if (param.Type == typeof(int)) {
                param.ValueTemplate = @<CustomCombobox @bind-value="@param.Value" />;
            }
        }
    }
}
```

### Standalone Parameter Editor (Hide Built-In Panel)

When you want to build a custom parameter UI outside the viewer:

```razor
@page "/viewer"
@rendermode InteractiveServer
@using DevExpress.Blazor.Reporting
@using DevExpress.Blazor.Reporting.Models

<input @bind="paramValue" placeholder="Enter Order ID" />
<button @onclick="Submit">Apply</button>

<DxReportViewer @ref="reportViewer" Report="@Report" />

@code {
    DxReportViewer reportViewer;
    string paramValue = "1";
    IReport Report;

    protected override void OnAfterRender(bool firstRender) {
        if (firstRender) {
            var report = new SalesReport();
            report.Parameters["OrderId"].Visible = false; // hide from built-in panel
            Report = report;
        }
    }

    async Task Submit() {
        var report = new SalesReport();
        report.Parameters["OrderId"].Value = int.Parse(paramValue);
        report.Parameters["OrderId"].Visible = false;
        report.RequestParameters = false;
        report.CreateDocument();
        // Hide the built-in parameters tab
        reportViewer.TabPanelModel.Tabs[0].Visible = false;
        Report = report;
    }
}
```

For `ParametersModel.OnSubmitParameters()` approach (triggers rebuild without recreating the report):

```razor
@code {
    void OnCustomizeParameters(ParametersModel model) {
        var param = model.VisibleItems.FirstOrDefault(p => p.Name == "OrderId");
        if (param != null) {
            param.ValueTemplate = @<CustomCombobox @bind-value="@param.Value" />;
        }
    }

    async Task Submit() {
        await reportViewer.ParametersModel.OnSubmitParameters();
    }
}
```

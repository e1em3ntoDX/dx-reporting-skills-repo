---
name: dx-report-designer
description: Expert skill for creating and modifying DevExpress XtraReport layout code in Visual Studio — writing and editing *.Designer.cs files, InitializeComponent structure, bands, controls, expression bindings, styles, parameters, and data source wiring. Use this skill whenever someone asks to create a DevExpress report, add controls to a report, modify a *.Designer.cs file, fix designer-generated code, work with XRTable/XRLabel/XRPictureBox/bands, set up data binding expressions, define report parameters, or troubleshoot why a report won't open in the Visual Studio designer. Also use when someone asks about XtraReports InitializeComponent patterns, report band hierarchy, Criteria Language expressions, or summary functions.
metadata:
  author: DevExpress Reports Team
  version: 1.0.0
---

You are an expert in DevExpress XtraReports layout code. You write and modify `*.Designer.cs` files for `XtraReport` subclasses, following the exact serialization patterns the Visual Studio Report Designer produces and requires.

DevExpress Reports (`DevExpress.XtraReports.UI`) is a cross-platform .NET reporting library. The same `XtraReport` class runs on WinForms, WPF, ASP.NET Core, and Blazor — platform differences only affect the viewer layer.

## Using DevExpress Documentation MCP

If the DxDocs MCP server is available, use it to supplement this skill:

- **Search**: Use `devexpress_docs_search` with technology `"XtraReports"` and your question.
- **Fetch**: Use `devexpress_docs_get_content` with a documentation URL to get full article content.

When to use MCP vs. built-in references:
- **Built-in references**: Getting started, common patterns, key properties, troubleshooting covered in this skill.
- **MCP search**: Advanced scenarios not covered here, version-specific API changes, uncommon features.
- **Always MCP for**: Exact method signatures, event arguments, enum values, or edge cases when you are not 100% certain.

## 🔒 Designer File Rules

Every `XtraReport` subclass has two files — never mix their concerns:

| File | Contains |
|---|---|
| `ReportName.Designer.cs` | All layout: bands, controls, positions, styles, bindings, data sources |
| `ReportName.cs` | Business logic only: event handlers, runtime data wiring, constructors |

Never add layout code to the main `.cs` file. Adding controls, setting `LocationFloat`, assigning `ExpressionBindings`, or calling `Controls.Add()` outside `InitializeComponent()` in `*.Designer.cs` breaks the Visual Studio Report Designer — the report won't open visually. After every edit, verify the report still opens in the designer.

See `references/designer-file-patterns.md` for the complete `InitializeComponent()` structure, field declarations, and annotated code examples from real reports.

## ⚙️ InitializeComponent Order

`InitializeComponent()` must always follow this exact order:

1. **Local inline object declarations** — `XRSummary`, `SelectQuery`, `Column`, `QueryParameter`, `MasterDetailInfo`, `RelationColumnInfo` objects used directly in property assignments
2. **`new` calls for all fields** — bands, controls, styles, data sources, parameters declared as `private` fields
3. **`BeginInit()` on every `XRTable` and on `this`** — all at once, before any property assignments
4. **Property assignments and `AddRange` calls** — configure every control and band
5. **Report-level setup** — `Bands.AddRange`, `DataSource`, `DataMember`, `StyleSheet`, `Parameters`, `ComponentStorage.AddRange`, `RequestParameters`, `Version`
6. **`EndInit()` in the same order as `BeginInit()`**

`ComponentStorage.AddRange(...)` is mandatory — it registers the data source so the designer can serialize it.

Key imports:
```csharp
using DevExpress.XtraReports.UI;
using DevExpress.Drawing;   // DXFont — cross-platform font, not System.Drawing.Font
using DevExpress.Utils;     // PointFloat
```

## 🏗️ Band Hierarchy

| Band | Prints When | Typical Content |
|---|---|---|
| `TopMarginBand` | Top of every page | Top page margin — cannot be deleted |
| `ReportHeaderBand` | Once at start | Title, logo, KPI summary cards |
| `PageHeaderBand` | Top of every page | Column headers, company header |
| `GroupHeaderBand` | Before each group | Group field label |
| `DetailBand` | Once per data record | Row data — cannot be deleted |
| `DetailReportBand` | Per parent record | Master-detail child section |
| `GroupFooterBand` | After each group | Group subtotals |
| `ReportFooterBand` | Once at end | Grand totals |
| `PageFooterBand` | Bottom of every page | Page numbers, signatures |
| `BottomMarginBand` | Bottom of every page | Bottom page margin — cannot be deleted |
| `SubBand` | Below parent band | Optional/conditional sections |

Key band properties:
```csharp
this.Detail.KeepTogether = true;          // prevent row from splitting across pages
this.ReportFooter.PrintAtBottom = true;   // pin to bottom of last page
this.GroupHeader1.RepeatEveryPage = true; // repeat group header on every page
```

Column headers belong in `PageHeaderBand` or `GroupHeaderBand` with `RepeatEveryPage = true` — never only in `DetailBand`.

`GroupHeaderBand` and `GroupFooterBand` are always created and removed in matched pairs. `DetailBand`, `TopMarginBand`, and `BottomMarginBand` cannot be removed.

## 🎛️ Controls

Always use `XRTable` for tabular layouts — never place stacked `XRLabel` controls side by side for columns. Table cells adjust height together; separate labels produce uneven rows when content wraps.

| Control | Purpose |
|---|---|
| `XRLabel` | Text — static or data-bound. Use with `AllowMarkupText = true` for inline formatting. |
| `XRTable` / `XRTableRow` / `XRTableCell` | Tabular layout. Supports `EvenStyleName`/`OddStyleName`, `RowSpan`, `CanGrow`, `TextFitMode`. |
| `XRPictureBox` | Images — static or bound. Use with `ImageResources` for embedded SVG/bitmap assets. |
| `XRRichText` | RTF/complex HTML only. Ignores the `Font` property. `XRRichTextBox` is obsolete. |
| `XRBarCode` | Barcodes. Keep `Module` large enough to be reliably scanned. |
| `XRPageInfo` | Page number / date / user via `PageInfo` enum. |
| `XRSubreport` | Embedded child report. Size is irrelevant — only `LocationFloat` matters. Set child report `Margins=0`, `PaperKind=Custom`. |
| `XRCrossTab` / `XRPivotGrid` | Pivot. Prefer `XRPivotGrid` for richer customization. |
| `XRChart` | Chart with independent `DataSource`. |

Controls with independent data sources (set `DataSource`/`DataMember` directly, they do not inherit from the report root): `XRChart`, `XRCrossTab`, `XRPivotGrid`, `XRSparkline`, `DetailReportBand`.

Appearance properties set on a container (`Band`, `XRPanel`, `XRTable`) propagate to all children. Add `StylePriority.UseXxx = false` when a control overrides an inherited or style value.

Never overlap controls — overlapping produces incorrect output in certain export formats.

## 🔗 Data Binding

Always use `ExpressionBinding`. Never use `DataBindings.Add(...)` — the legacy API cannot coexist with expressions mode.

```csharp
// Field
cell.ExpressionBindings.Add(new ExpressionBinding("BeforePrint", "Text", "[ProductName]"));

// Multi-field — use FormatString() over string concatenation
cell.ExpressionBindings.Add(new ExpressionBinding("BeforePrint", "Text",
    "FormatString('{0}, {1}, {2}', [Address], [City], [Country])"));

// Parameter
label.ExpressionBindings.Add(new ExpressionBinding("BeforePrint", "Text", "?ReportTitle"));

// Conditional appearance
label.ExpressionBindings.Add(new ExpressionBinding("BeforePrint", "BackColor",
    "Iif([Amount] < 0, 'Red', 'Transparent')"));
```

## 📐 Criteria Language

Expressions use DevExpress Criteria Language — never C# syntax.

| Element | Syntax |
|---|---|
| Data field | `[FieldName]` |
| Child collection aggregate | `[RelationName][].Count()` |
| Parameter | `?ParameterName` |
| String constant | `'text'` (single quotes only) |
| Date constant | `#2024-01-01#` |
| Conditional | `Iif(cond, trueVal, falseVal)` |
| Multi-field format | `FormatString('{0}, {1}', [F1], [F2])` |
| Null check | `IsNull([Field])` |
| Named image | `[Images.ImageName]` |

Rules: single-quoted strings · `True`/`False` capitalized · no C# `? :` ternary · no double-quoted strings · no `&&`/`||`.

**Summary vs. Aggregate:** use `sumSum([Field])` / `sumCount([Id])` / `sumAvg([Price])` in footer bands — they support Group/Page/Report running scope. `Sum([Field])` aggregate functions apply to the entire data source, ignore `FilterString`, and are slow per-record. Never use aggregate functions in `DetailBand`.

See `references/expressions-and-summaries.md` for the full syntax reference and summary scope examples.

## 🎯 Parameters and Master-Detail

For date range parameters, use `RangeStartParameter`/`RangeEndParameter` with `RangeParametersSettings`. For master-detail `SqlDataSource`, use `MasterDetailInfo` and `RelationColumnInfo` to define query relations, then set `DetailReportBand.DataMember` to the relation path (e.g. `"Customers.CustomersOrders_1"`).

See `references/data-sources-and-parameters.md` for complete code examples.

## 🎨 Styles

Create `XRControlStyle` objects, register in `StyleSheet`, assign via `StyleName`. For alternating row colors, use `XRTable.EvenStyleName`/`OddStyleName` — no expressions needed.

## 📏 Formatting Standards

| Data Type | Alignment |
|---|---|
| Text, Notes, IDs, Phone, URL, Email | Left |
| Numeric, Currency, Percentage | Right |
| Date/Time, Boolean, Column headers, Images | Center |

Use `TextFormatString` for all number and date formatting: `"{0:C2}"`, `"{0:d}"`, `"{0:0%}"`.

For Excel export: align all control borders on vertical grid lines — gaps produce extra empty columns.

## 🚫 Never Do

- Layout code in `ReportName.cs` — all layout in `InitializeComponent()` in `*.Designer.cs`
- `DataBindings.Add(...)` — use `ExpressionBindings`
- Stacked `XRLabel` controls for columns — use `XRTable`
- `XRRichText` for simple formatted text — use `XRLabel` with `AllowMarkupText = true`
- `XRRichTextBox` — obsolete, use `XRRichText`
- C# syntax in expressions — use Criteria Language
- Overlapping controls
- Missing `BeginInit`/`EndInit` on any `XRTable` — all calls must happen together before/after all property assignments
- Missing `ComponentStorage.AddRange` for data sources
- Aggregate functions per-record in `DetailBand` — use summary functions in footer bands
- `XRCrossTab` when customization is insufficient — use `XRPivotGrid`

## 📁 Reference Files

- `references/designer-file-patterns.md` — complete `InitializeComponent()` skeleton, `BeginInit`/`EndInit` ordering, field declarations
- `references/expressions-and-summaries.md` — full Criteria Language syntax, summary scopes, `XRSummary` object, aggregate vs. summary
- `references/data-sources-and-parameters.md` — `SqlDataSource` setup, master-detail relations, range parameters, query `FilterString`
- `references/examples/master-detail-report.md` — annotated real-world master-detail report with grouping, summaries, and parameters
- `references/examples/table-report.md` — annotated invoice-style table report with multi-row cells, alternating styles, and subtotals

## 🔍 Useful MCP Queries

- `"Criteria Language syntax operators functions"` — expression syntax reference
- `"GroupHeaderBand RepeatEveryPage group fields"` — group band configuration

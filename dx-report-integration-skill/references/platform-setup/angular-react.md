# Angular / React — Document Viewer Integration

For SPA frameworks, the Document Viewer is split into two separate projects:

- **Backend** — ASP.NET Core API with CORS, serves reports and handles viewer API calls
- **Frontend** — Angular or React app, contains the viewer component

npm package versions on the frontend **must exactly match** NuGet package versions on the backend.

---

## Backend — ASP.NET Core (shared for Angular and React)

### Option A: DevExpress CLI Template (fastest)

```bash
dotnet new install DevExpress.AspNetCore.ProjectTemplates
dotnet new dx.aspnetcore.reporting.backend -n ServerApp --add-designer false
cd ServerApp
dotnet run
```

### Option B: Manual Setup

**NuGet:** `DevExpress.AspNetCore.Reporting`

**Program.cs:**

```csharp
using DevExpress.AspNetCore;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddCors(options => {
    options.AddPolicy("AllowCorsPolicy", policy => {
        // Allow all localhost ports during development
        policy.SetIsOriginAllowed(origin => new Uri(origin).Host == "localhost");
        policy.AllowAnyHeader();
        policy.AllowAnyMethod();
    });
});

builder.Services.AddControllersWithViews();
builder.Services.AddDevExpressControls();
builder.Services.ConfigureReportingServices(configurator => {
    configurator.ConfigureWebDocumentViewer(viewerConfigurator => {
        viewerConfigurator.UseCachedReportSourceBuilder();
    });
});

var app = builder.Build();

app.UseRouting();
app.UseCors("AllowCorsPolicy");      // ← after UseRouting, before UseEndpoints
app.UseDevExpressControls();

app.UseEndpoints(endpoints => {
    endpoints.MapControllerRoute(
        name: "default",
        pattern: "{controller=Home}/{action=Index}/{id?}");
});

app.Run();
```

**Controllers/ReportingControllers.cs:**

```csharp
using DevExpress.AspNetCore.Reporting.WebDocumentViewer;
using DevExpress.AspNetCore.Reporting.WebDocumentViewer.Native.Services;

public class CustomWebDocumentViewerController : WebDocumentViewerController {
    public CustomWebDocumentViewerController(
        IWebDocumentViewerMvcControllerService controllerService)
        : base(controllerService) { }
}
```

Register `IReportProvider` so reports can be loaded by name:

```csharp
// Program.cs
builder.Services.AddScoped<IReportProvider, CustomReportProvider>();
```

The viewer endpoint is at `http://localhost:5000/DXXRDV` (ASP.NET Core default).

---

## Angular Frontend

### Angular npm Packages

```bash
npm install devextreme@25.2-stable \
            devextreme-angular@25.2-stable \
            @devexpress/analytics-core@25.2-stable \
            devexpress-reporting-angular@25.2-stable
```

### app.ts (standalone components, Angular 17+)

```typescript
import { Component, ViewEncapsulation } from '@angular/core';
import { DxReportViewerModule } from 'devexpress-reporting-angular';

@Component({
  selector: 'app-root',
  encapsulation: ViewEncapsulation.None,   // ← required — prevents Angular from scoping CSS
  imports: [DxReportViewerModule],
  templateUrl: './app.html',
  styleUrls: [
    '../../node_modules/devextreme/dist/css/dx.light.css',
    '../../node_modules/@devexpress/analytics-core/dist/css/dx-analytics.common.css',
    '../../node_modules/@devexpress/analytics-core/dist/css/dx-analytics.light.css',
    '../../node_modules/devexpress-reporting/dist/css/dx-webdocumentviewer.css'
  ]
})
export class App {
  reportUrl: string = 'SalesReport';
  hostUrl: string = 'http://localhost:5000/';
  invokeAction: string = '/DXXRDV';   // ASP.NET Core backend
  // invokeAction: string = '/WebDocumentViewer/Invoke';  // ASP.NET MVC backend
}
```

### app.html

```html
<dx-report-viewer [reportUrl]="reportUrl" height="800px">
  <dxrv-request-options
    [invokeAction]="invokeAction"
    [host]="hostUrl">
  </dxrv-request-options>
</dx-report-viewer>
```

### angular.json — increase budget for large bundles

```json
"budgets": [
  {
    "type": "initial",
    "maximumWarning": "5mb",
    "maximumError": "10mb"
  }
]
```

Also add `"skipLibCheck": true` in `tsconfig.json` if you encounter Angular compilation errors with DevExpress types.

---

## React Frontend (Next.js)

### React npm Package

```bash
npm install devexpress-reporting-react@25.2-stable
```

`devexpress-reporting-react` includes all dependencies — do not separately install `devextreme` or `devexpress-reporting`.

### app/page.tsx (Next.js App Router)

```tsx
'use client';
import 'devextreme/dist/css/dx.light.css';
import '@devexpress/analytics-core/dist/css/dx-analytics.common.css';
import '@devexpress/analytics-core/dist/css/dx-analytics.light.css';
import 'devexpress-reporting/dist/css/dx-webdocumentviewer.css';
import ReportViewer, { RequestOptions } from 'devexpress-reporting-react/dx-report-viewer';

export default function App() {
  return (
    <ReportViewer reportUrl="SalesReport">
      <RequestOptions
        host="http://localhost:5000/"
        invokeAction="DXXRDV" />
    </ReportViewer>
  );
}
```

### React (Vite)

```bash
npm install devextreme@25.2-stable \
            @devexpress/analytics-core@25.2-stable \
            devexpress-reporting@25.2-stable \
            devexpress-reporting-react@25.2-stable
```

```tsx
import 'devextreme/dist/css/dx.light.css';
import '@devexpress/analytics-core/dist/css/dx-analytics.common.css';
import '@devexpress/analytics-core/dist/css/dx-analytics.light.css';
import 'devexpress-reporting/dist/css/dx-webdocumentviewer.css';
import ReportViewer, { RequestOptions } from 'devexpress-reporting-react/dx-report-viewer';

function App() {
  return (
    <ReportViewer reportUrl="SalesReport">
      <RequestOptions host="http://localhost:5000/" invokeAction="DXXRDV" />
    </ReportViewer>
  );
}
export default App;
```

---

## Common Troubleshooting (Angular / React)

- **Blank viewer / "Could not open report"**: Backend not running, wrong port in `host`, or CORS policy doesn't allow the frontend origin.
- **CORS errors in browser console**: `UseCors("AllowCorsPolicy")` must be called after `UseRouting()` and before `UseEndpoints()`/`MapControllers()` in `Program.cs`.
- **Version mismatch**: npm package versions must exactly match NuGet package versions. Enable `configurator.UseDevelopmentMode()` on the backend to see mismatch errors in the browser console.
- **`invokeAction` value**: Use `/DXXRDV` for ASP.NET Core, `/WebDocumentViewer/Invoke` for ASP.NET MVC.
- **Report not found**: Ensure backend `IReportProvider` or `ReportStorageWebExtension` maps the `reportUrl` string value to a report instance.

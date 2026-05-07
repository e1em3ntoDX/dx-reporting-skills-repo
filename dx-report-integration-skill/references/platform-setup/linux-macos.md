# Reporting on Linux and macOS

DevExpress Reports runs on Linux and macOS in .NET 6+ applications. Several additional packages and OS dependencies are required beyond the standard Windows setup.

## Required NuGet Package — Drawing Engine

Add `DevExpress.Drawing.Skia` to every project that runs reporting on a non-Windows host:

```xml
<PackageReference Include="DevExpress.Drawing.Skia" Version="25.2.*" />
```

This enables the Skia-based drawing engine. Without it, `System.DllNotFoundException` is thrown at runtime when the viewer or designer attempts to render a report (the native Windows GDI+ backend is unavailable on Linux/macOS).

If `System.Drawing.Common` v7 or later is present in the project, the Skia engine activates automatically. On earlier versions, the package enables it explicitly.

## Required NuGet Package — PDF Content Control

If your reports use the `XRPdfContent` control, also add:

```xml
<PackageReference Include="DevExpress.Pdf.SkiaRenderer" Version="25.2.*" />
```

On Linux, also add:

```xml
<PackageReference Include="SkiaSharp.NativeAssets.Linux" Version="*" />
```

## OS-Level Dependencies

Install these system packages before running the application.

### Debian / Ubuntu

```shell
apt-get update
apt-get install -y libc6 libicu-dev libfontconfig1
```

### Alpine

```shell
apk update && apk upgrade
apk add icu-libs icu-data-full fontconfig
```

### Fonts

Reports use fonts at render time. On Linux, fonts must be installed on the host OS:

```shell
# Copy font files to the system fonts folder
cp *.ttf /usr/share/fonts/
# Rebuild the fontconfig cache
fc-cache
```

macOS ships with system fonts and does not require `libcups` or additional font setup for common scenarios.

For Docker deployments, include the font installation and `fc-cache` step in your `Dockerfile`.

## Minimal Cross-Platform .csproj Template

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <!-- Core reporting -->
    <PackageReference Include="DevExpress.AspNetCore.Reporting" Version="25.2.*" />
    <!-- Required on Linux / macOS for the drawing engine -->
    <PackageReference Include="DevExpress.Drawing.Skia" Version="25.2.*" />
    <!-- Required only if reports embed PDF content via XRPdfContent -->
    <!-- <PackageReference Include="DevExpress.Pdf.SkiaRenderer" Version="25.2.*" /> -->
    <!-- Required on Linux when DevExpress.Pdf.SkiaRenderer is included -->
    <!-- <PackageReference Include="SkiaSharp.NativeAssets.Linux" Version="*" /> -->
  </ItemGroup>
</Project>
```

## Restore and Build Order

On non-Windows hosts the same build ordering applies as on Windows:

1. `npm install` (installs client-side packages into `node_modules/`)
2. `dotnet restore` (restores NuGet packages including `DevExpress.Drawing.Skia`)
3. `dotnet build` (triggers libman, which reads from `node_modules/`)

## Azure App Service (Linux)

When hosting on Azure App Service for Linux (or Azure Functions on Linux), enable the Skia rendering engine explicitly even if `DevExpress.Drawing.Skia` is installed:

```csharp
// Program.cs
DevExpress.Drawing.Internal.DXDrawingEngine.ForceSkia();
```

Or set the rendering engine property on print options where applicable.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `System.DllNotFoundException: DevExpress.Drawing.v25.2.Skia.dll` | `DevExpress.Drawing.Skia` NuGet package missing | Add `<PackageReference Include="DevExpress.Drawing.Skia" .../>` |
| `DllNotFoundException` for `libgdiplus` | GDI+ backend attempted on Linux without Skia | Same fix — add `DevExpress.Drawing.Skia` |
| Report renders blank or throws on PDF export | `XRPdfContent` without Skia renderer | Add `DevExpress.Pdf.SkiaRenderer` (+ `SkiaSharp.NativeAssets.Linux` on Linux) |
| Fonts not rendering / garbled text | Fonts missing from OS | Install fonts and run `fc-cache` |

---
title: "Shipping a .NET application with MakeNSIS — 1. From publish to Setup.exe"
description: "Build a reproducible Windows NSIS installer locally for a .NET application, using TidyMemo as a real-world example and without depending on GitHub Actions."
date: 2026-09-01T00:00:00+02:00
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [dotnet, csharp, avalonia, windows, nsis, packaging]
series: ["MakeNSIS for .NET"]
---

> **`dotnet publish` delivers files; an installer delivers a deployment experience. MakeNSIS turns that last mile into code: reproducible, versioned, and runnable both on a developer workstation and in a delivery pipeline.**

A desktop application is more than its executable. Someone must select a destination, copy a consistent set of files, create shortcuts, expose an uninstaller, and let Windows identify the product. A ZIP delegates those decisions to every user; an installer makes them explicit.

This first part builds a minimal installer without relying on any particular repository. It then applies the same concepts to [TidyMemo](https://github.com/agailloty/TidyMemo), a real Avalonia desktop application. Its GitHub automation appears only at the end, as a consumer of the local process rather than the starting point.

## What `makensis` does—and does not do

[NSIS](https://nsis.sourceforge.io/Docs/) is a scriptable Windows installer system. `makensis.exe` is its command-line compiler: it reads an `.nsi` script, embeds the selected files, and emits an installer executable.

It does not compile the .NET application. The boundary is deliberate:

```text
C# sources ── dotnet publish ──> publish directory
                                       │
.nsi script ─────── makensis ──────────┴──> Setup.exe
```

The .NET SDK owns the runtime, self-contained deployment, trimming, and AOT choices. NSIS owns Windows installation behavior. A `dotnet publish` failure is therefore not an NSIS failure, and vice versa.

## The NSIS mental model

An NSIS script is both declarative and imperative. Global instructions describe the package (`Name`, `OutFile`, and `InstallDir`). Page declarations define the visible wizard. `Section` blocks perform installation or removal. Finally, preprocessor instructions—prefixed with `!`—resolve constants, includes, and embedded files at compile time.

Keep two moments distinct:

- **at compile time**, `makensis` expands macros and embeds files selected by `File`;
- **at install time**, the generated `Setup.exe` uses variables such as `$INSTDIR`, writes to the target computer, and creates the uninstaller.

Consequently, `${PUBLISH_DIR}` can identify a build directory on the developer machine while `$INSTDIR` identifies the final directory on the user's computer.

## Install NSIS without a package manager

The only prerequisite beyond the .NET SDK is the official Windows NSIS installer. The [NSIS download page](https://nsis.sourceforge.io/Download) links to the executable for the stable release. At the time of writing, this is [NSIS 3.12—`nsis-3.12-setup.exe`](https://sourceforge.net/projects/nsis/files/NSIS%203/3.12/nsis-3.12-setup.exe/download), a download of about 1.6 MB. For durable documentation, always check the official page before pinning a version in automation.

1. Download the executable from the official page.
2. Run it and keep the default components, which include the compiler and include files.
3. Complete installation. No editor extension or package manager is required.

The usual location for a 32-bit installation on 64-bit Windows is `C:\Program Files (x86)\NSIS`, but robust automation should also account for a custom location:

```powershell
$candidates = @(
    (Join-Path ${env:ProgramFiles(x86)} 'NSIS\makensis.exe'),
    (Join-Path $env:ProgramFiles 'NSIS\makensis.exe')
)
$makeNsis = $candidates | Where-Object {
    Test-Path -LiteralPath $_ -PathType Leaf
} | Select-Object -First 1

if (-not $makeNsis) {
    throw 'makensis.exe was not found. Install NSIS from https://nsis.sourceforge.io/Download.'
}
& $makeNsis /VERSION
```

The final command should print the compiler version.

## First installer: start from a publish directory

Assume a desktop application named `MyApp`, with its project at `src/MyApp/MyApp.csproj`. Publish it for Windows first:

```powershell
dotnet publish src/MyApp/MyApp.csproj `
    --configuration Release `
    --runtime win-x64 `
    --self-contained true `
    --output publish/win-x64
```

Then create `packaging/windows/MyApp.nsi`:

```nsis
Unicode True
!include "MUI2.nsh"

!define APP_NAME "MyApp"
!define APP_EXE "MyApp.exe"

Name "${APP_NAME}"
OutFile "MyApp-Setup.exe"
InstallDir "$LOCALAPPDATA\Programs\${APP_NAME}"
RequestExecutionLevel user
SetCompressor /SOLID lzma

!insertmacro MUI_PAGE_DIRECTORY
!insertmacro MUI_PAGE_INSTFILES
!insertmacro MUI_UNPAGE_CONFIRM
!insertmacro MUI_UNPAGE_INSTFILES
!insertmacro MUI_LANGUAGE "English"

Section "Install"
  SetOutPath "$INSTDIR"
  File /r "..\..\publish\win-x64\*"
  WriteUninstaller "$INSTDIR\Uninstall.exe"
  CreateShortcut "$SMPROGRAMS\${APP_NAME}.lnk" "$INSTDIR\${APP_EXE}"
SectionEnd

Section "Uninstall"
  Delete "$SMPROGRAMS\${APP_NAME}.lnk"
  RMDir /r "$INSTDIR"
SectionEnd
```

Compile it from the project root:

```powershell
& $makeNsis 'packaging/windows/MyApp.nsi'
if ($LASTEXITCODE -ne 0) {
    throw "makensis failed with exit code $LASTEXITCODE."
}
```

This first script already contains the complete cycle: `File /r` embeds the publish output, `SetOutPath` selects its destination, `WriteUninstaller` generates the removal program, and the second section describes the inverse operation. It is intentionally minimal; registry metadata, icons, licensing, versioning, and injectable paths arrive with the real example.

## Case study: TidyMemo

TidyMemo is an open-source desktop application for organizing and processing photos and videos locally. Its UI uses Avalonia, its main project lives at `TidyMemo/TidyMemo.csproj`, and the repository contains `packaging/windows/TidyMemo.nsi`. We can now repeat the preceding process with real application constraints.

### .NET prerequisites for the example

The example reflects the TidyMemo version reviewed for this article: `net10.0`, the `win-x64` runtime, self-contained publishing, and `PublishAot` enabled in the project.

On Windows, check the SDK:

```powershell
dotnet --info
dotnet --list-sdks
```

### Publish the Windows application

From the TidyMemo repository root:

```powershell
$version = '2.5.0'

dotnet restore TidyMemo/TidyMemo.csproj
if ($LASTEXITCODE -ne 0) { throw 'dotnet restore failed.' }

dotnet publish TidyMemo/TidyMemo.csproj `
    --configuration Release `
    --runtime win-x64 `
    --self-contained true `
    --output publish/windows-x64 `
    -p:Version=$version `
    -p:PublishTrimmed=true
if ($LASTEXITCODE -ne 0) { throw 'dotnet publish failed.' }
```

The `win-x64` RID selects one architecture. `--self-contained true` includes the required .NET components, so users do not need to install that runtime separately, at the cost of a larger package. Trimming requires functional testing because indirectly reached code—for example through reflection—may be removed. TidyMemo already sets `PublishAot` in its project file; do not apply that choice to every Avalonia application without validating its dependencies.

Check the packaging contract:

```powershell
$publishDir = (Resolve-Path 'publish/windows-x64').Path
$appExe = Join-Path $publishDir 'TidyMemo.exe'
if (-not (Test-Path -LiteralPath $appExe -PathType Leaf)) {
    throw "The publish output does not contain $appExe"
}
```

### Evolve the minimal script

The real script is [`packaging/windows/TidyMemo.nsi`](https://github.com/agailloty/TidyMemo/blob/master/packaging/windows/TidyMemo.nsi). Build inputs are supplied on the command line:

```nsis
!ifndef APP_VERSION
  !define APP_VERSION "0.0.0"
!endif
!ifndef PUBLISH_DIR
  !error "PUBLISH_DIR must point to the dotnet publish directory"
!endif
!ifndef OUTPUT_FILE
  !define OUTPUT_FILE "TidyMemo-Setup.exe"
!endif
```

`!define` and `!ifndef` are NSIS preprocessor instructions. The script requires the only essential external input, `PUBLISH_DIR`, and defaults the others. Symbols passed with `/D` must appear before the `.nsi` file because `makensis` processes arguments in order.

The main configuration describes a per-user installation:

```nsis
Name "${APP_NAME} ${APP_VERSION}"
OutFile "${OUTPUT_FILE}"
InstallDir "$LOCALAPPDATA\Programs\${APP_NAME}"
InstallDirRegKey HKCU "${APP_REGISTRY_KEY}" "InstallLocation"
RequestExecutionLevel user
SetCompressor /SOLID lzma
```

This avoids elevation and is consistent with `HKCU` and `$LOCALAPPDATA`. Installing under Program Files normally calls for a per-machine strategy (`admin` and `HKLM`). Do not mix the models.

Modern UI 2 provides the wizard pages:

```nsis
!include "MUI2.nsh"
!insertmacro MUI_PAGE_WELCOME
!insertmacro MUI_PAGE_LICENSE "${REPOSITORY_ROOT}\LICENSE"
!insertmacro MUI_PAGE_DIRECTORY
!insertmacro MUI_PAGE_INSTFILES
!insertmacro MUI_PAGE_FINISH
```

The install section copies the exact `dotnet publish` output, writes an uninstaller, creates shortcuts, and registers the application with Windows:

```nsis
Section "TidyMemo" SEC_MAIN
  SetOutPath "$INSTDIR"
  File /r "${PUBLISH_DIR}\*"
  WriteUninstaller "$INSTDIR\Uninstall.exe"
  CreateDirectory "$SMPROGRAMS\${APP_NAME}"
  CreateShortcut "$SMPROGRAMS\${APP_NAME}\${APP_NAME}.lnk" "$INSTDIR\${APP_EXE}"
  WriteRegStr HKCU "${APP_REGISTRY_KEY}" "DisplayName" "${APP_NAME}"
  WriteRegStr HKCU "${APP_REGISTRY_KEY}" "UninstallString" '"$INSTDIR\Uninstall.exe"'
SectionEnd
```

`File /r` is evaluated at compile time. The files are embedded into `Setup.exe`; they are not read again on the target computer. At install time, `$INSTDIR` is the destination selected by the user.

### Compile Setup locally

GitHub is not involved at this point. The complete local command is:

```powershell
$ErrorActionPreference = 'Stop'
$version = '2.5.0'
$makeNsis = Join-Path ${env:ProgramFiles(x86)} 'NSIS\makensis.exe'
$publishDir = (Resolve-Path 'publish/windows-x64').Path
$repositoryRoot = (Resolve-Path '.').Path

New-Item -ItemType Directory -Force artifacts | Out-Null
$artifactsDir = (Resolve-Path 'artifacts').Path
$setup = Join-Path $artifactsDir 'TidyMemo-windows-x64-Setup.exe'

& $makeNsis `
    '/V3' `
    "/DAPP_VERSION=$version" `
    "/DPUBLISH_DIR=$publishDir" `
    "/DOUTPUT_FILE=$setup" `
    "/DREPOSITORY_ROOT=$repositoryRoot" `
    'packaging/windows/TidyMemo.nsi'

if ($LASTEXITCODE -ne 0) {
    throw "makensis failed with exit code $LASTEXITCODE."
}
if (-not (Test-Path -LiteralPath $setup -PathType Leaf)) {
    throw "The expected installer was not created: $setup"
}
Get-FileHash -Algorithm SHA256 -LiteralPath $setup
```

The [official `makensis` command-line reference](https://nsis.sourceforge.io/Docs/Chapter3.html) documents `/V0` through `/V4` for verbosity, `/Obuild.log` for log output, and `/Dname=value` for preprocessor symbols. Keep the PowerShell quotes around each argument so paths containing spaces reach `makensis` as one argument.

## One stable local command

For maintainable project documentation, put validation, `dotnet publish`, and `makensis` in `packaging/windows/package.ps1`, then expose one interface:

```powershell
./packaging/windows/package.ps1 -Version 2.5.0
```

Resolve paths from the script location instead of the caller's working directory:

```powershell
param(
    [Parameter(Mandatory)]
    [ValidatePattern('^\d+\.\d+\.\d+$')]
    [string] $Version
)

$repositoryRoot = (Resolve-Path (Join-Path $PSScriptRoot '../..')).Path
Push-Location $repositoryRoot
try {
    # restore, publish, checks, then makensis
}
finally {
    Pop-Location
}
```

The three-number constraint is significant. TidyMemo constructs `VIProductVersion "${APP_VERSION}.0"`, while a Windows version resource expects four numeric components. A value such as `2.5.0-beta.1` needs a separate numeric file version and display version.

## Verify the result

A successful build does not prove that the package behaves correctly. On a clean Windows VM or test machine:

1. launch `TidyMemo-windows-x64-Setup.exe`;
2. keep, then change, the proposed directory;
3. start the application from the Start menu;
4. check its entry in **Installed apps**;
5. reinstall the same version, then a newer one;
6. uninstall and inspect remaining files and shortcuts.

NSIS supports `/S` for silent installation, enabling a smoke test:

```powershell
$testDir = Join-Path $env:LOCALAPPDATA 'Programs\TidyMemo'
Start-Process -FilePath $setup -ArgumentList '/S' -Wait
if (-not (Test-Path (Join-Path $testDir 'TidyMemo.exe'))) {
    throw 'Silent installation could not be verified.'
}
Start-Process -FilePath (Join-Path $testDir 'Uninstall.exe') -ArgumentList '/S' -Wait
```

Run it in a disposable environment because it modifies the current profile. Installer option `/D=path` can override the destination, but NSIS requires it to be the final argument, absolute, and unquoted.

## Integrate the process into CI afterwards

Only after the local process is understood and verified should CI invoke it. TidyMemo's [Windows job](https://github.com/agailloty/TidyMemo/blob/master/.github/workflows/dotnet.yml) prepares .NET and NSIS, calculates a version, publishes the app, compiles the `.nsi`, and uploads artifacts. Those steps do not define how MakeNSIS works; they automate commands that already run on a Windows workstation.

The dependency should point one way: CI invokes local automation, rather than local automation emulating GitHub-specific variables. Part two will use this foundation to address upgrades, user data, signing, logs, and release checks.

### References

- [NSIS Users Manual](https://nsis.sourceforge.io/Docs/)
- [MakeNSIS command-line usage](https://nsis.sourceforge.io/Docs/Chapter3.html)
- [Modern UI 2 documentation](https://nsis.sourceforge.io/Docs/Modern%20UI%202/Readme.html)
- [TidyMemo workflow](https://github.com/agailloty/TidyMemo/blob/master/.github/workflows/dotnet.yml)
- [TidyMemo NSIS script](https://github.com/agailloty/TidyMemo/blob/master/packaging/windows/TidyMemo.nsi)

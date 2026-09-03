---
title: "Shipping a .NET application with MakeNSIS — 1. From publish to Setup.exe"
description: "Build a reproducible Windows NSIS installer locally for a .NET application, with a complete example and no PowerShell script."
date: 2026-09-01T00:00:00+02:00
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [dotnet, csharp, avalonia, windows, nsis, packaging]
series: ["MakeNSIS for .NET"]
---

> **A `dotnet publish` delivers files; an installer delivers a deployment experience. MakeNSIS turns that final step into versioned code. To get started, two commands and one `.nsi` file are enough.**

This guide starts with a .NET desktop application named `MyApp`. It requires no PowerShell, GitHub Actions, or previous knowledge of NSIS. By the end, you will have a `MyApp-Setup.exe` that installs and uninstalls cleanly.

A desktop application is more than its executable. Someone must choose where to copy it, create shortcuts, register it with Windows, plan upgrades, and make removal possible. Distributing a ZIP delegates those decisions to every user. An installer makes them once, explicitly and reproducibly.

## What MakeNSIS does—and does not do

[NSIS](https://nsis.sourceforge.io/Docs/) is a script-driven system for creating Windows installers. `makensis.exe` is its compiler: it reads an `.nsi` file, collects the selected files, and generates a new executable—the Setup.

MakeNSIS does not compile C# and knows nothing about NuGet, Avalonia, or the contents of a `.csproj`. Conversely, `dotnet publish` does not create Windows shortcuts and does not know how to uninstall the application. The boundary is intentionally clear:

```text
C# sources ── dotnet publish ──> publish directory
                                         │
.nsi script ─────── makensis ────────────┴──> Setup.exe
```

This separation also helps diagnosis. If `dotnet publish` fails, the problem belongs to the .NET build. If `makensis` cannot find a file or rejects an instruction, the problem belongs to packaging. If Setup is created but installs the application incorrectly, inspect the behavior described by the `.nsi` file.

## The mental model of an NSIS script

An `.nsi` file combines four kinds of instructions:

- global attributes describe the package, such as `Name`, `OutFile`, and `InstallDir`;
- page declarations compose the wizard shown to the user;
- `Section` blocks perform installation or removal operations;
- the preprocessor, identified by the `!` prefix, resolves constants, includes, and files to embed.

Most importantly, distinguish **compilation** from **installation**.

During compilation, `makensis.exe` reads the `.nsi` file. It expands macros, evaluates `${...}`, and embeds the files selected by `File`. The publish directory must therefore exist on the machine building Setup.

Later, the user runs the resulting Setup. Only then do the `Section` blocks run, `$INSTDIR` receive its value, files get extracted, and registry entries get written.

Therefore, `${PUBLISH_DIR}` represents a build directory on the developer's machine, while `$INSTDIR` represents the final directory on the user's machine. The syntax looks similar, but the values exist at different times and potentially on different machines.

## The complete path

```text
MyApp.csproj
    │
    └── dotnet publish ──> publish\win-x64\
                                  │
MyApp.nsi ─── makensis.exe ───────┴──> artifacts\MyApp-Setup.exe
```

The .NET SDK controls the runtime, self-contained deployment, trimming, and AOT. NSIS controls the installation directory, shortcuts, registry, and uninstallation.

## Prerequisites

You need Windows, the .NET SDK used by your project, and [NSIS](https://nsis.sourceforge.io/Download).

1. Install the .NET SDK and verify it with `dotnet --info`.
2. Install NSIS with the default components.
3. Check the compiler from `cmd.exe` or your IDE terminal:

```bat
"C:\Program Files (x86)\NSIS\makensis.exe" /VERSION
```

If NSIS is installed elsewhere, replace that path in the following commands. No state is shared between the code blocks: each command can be copied into a new terminal opened at the repository root.

## Project layout

```text
MyApp/
├── src/
│   └── MyApp/
│       └── MyApp.csproj
└── packaging/
    └── windows/
        └── MyApp.nsi
```

If your project lives elsewhere, only the `dotnet publish` command needs adapting.

## Step 1—publish the application

From the repository root:

```bat
dotnet publish src\MyApp\MyApp.csproj --configuration Release --runtime win-x64 --self-contained true --output publish\win-x64
```

Check that `publish\win-x64\MyApp.exe` exists. This directory is the contract between .NET and NSIS: the installer will embed all its contents.

`--self-contained true` avoids requiring a separate .NET runtime on the target machine, at the cost of a larger package. Do not add trimming or Native AOT immediately: these optimizations have their own prerequisites and require separate testing.

## Step 2—create the complete NSIS script

Create `packaging\windows\MyApp.nsi` with this content:

```nsis
Unicode True
!include "MUI2.nsh"

!define APP_NAME "MyApp"
!define APP_EXE "MyApp.exe"
!define APP_PUBLISHER "My Company"
!define APP_REGISTRY_KEY "Software\Microsoft\Windows\CurrentVersion\Uninstall\${APP_NAME}"

!ifndef APP_VERSION
  !define APP_VERSION "0.0.0"
!endif
!ifndef PUBLISH_DIR
  !error "PUBLISH_DIR must point to the directory produced by dotnet publish"
!endif
!ifndef OUTPUT_FILE
  !define OUTPUT_FILE "${APP_NAME}-Setup.exe"
!endif

Name "${APP_NAME} ${APP_VERSION}"
OutFile "${OUTPUT_FILE}"
InstallDir "$LOCALAPPDATA\Programs\${APP_NAME}"
InstallDirRegKey HKCU "${APP_REGISTRY_KEY}" "InstallLocation"
RequestExecutionLevel user
SetCompressor /SOLID lzma

VIProductVersion "${APP_VERSION}.0"
VIAddVersionKey /LANG=1033 "ProductName" "${APP_NAME}"
VIAddVersionKey /LANG=1033 "FileVersion" "${APP_VERSION}"
VIAddVersionKey /LANG=1033 "CompanyName" "${APP_PUBLISHER}"

!insertmacro MUI_PAGE_WELCOME
!insertmacro MUI_PAGE_DIRECTORY
!insertmacro MUI_PAGE_INSTFILES
!insertmacro MUI_PAGE_FINISH
!insertmacro MUI_UNPAGE_CONFIRM
!insertmacro MUI_UNPAGE_INSTFILES
!insertmacro MUI_LANGUAGE "English"

Section "Install"
  SetOutPath "$INSTDIR"
  File /r "${PUBLISH_DIR}\*"
  WriteUninstaller "$INSTDIR\Uninstall.exe"
  CreateDirectory "$SMPROGRAMS\${APP_NAME}"
  CreateShortcut "$SMPROGRAMS\${APP_NAME}\${APP_NAME}.lnk" "$INSTDIR\${APP_EXE}"

  WriteRegStr HKCU "${APP_REGISTRY_KEY}" "DisplayName" "${APP_NAME}"
  WriteRegStr HKCU "${APP_REGISTRY_KEY}" "DisplayVersion" "${APP_VERSION}"
  WriteRegStr HKCU "${APP_REGISTRY_KEY}" "Publisher" "${APP_PUBLISHER}"
  WriteRegStr HKCU "${APP_REGISTRY_KEY}" "DisplayIcon" "$INSTDIR\${APP_EXE}"
  WriteRegStr HKCU "${APP_REGISTRY_KEY}" "InstallLocation" "$INSTDIR"
  WriteRegStr HKCU "${APP_REGISTRY_KEY}" "UninstallString" '$\"$INSTDIR\Uninstall.exe$\"'
  WriteRegStr HKCU "${APP_REGISTRY_KEY}" "QuietUninstallString" '$\"$INSTDIR\Uninstall.exe$\" /S'
  WriteRegDWORD HKCU "${APP_REGISTRY_KEY}" "NoModify" 1
  WriteRegDWORD HKCU "${APP_REGISTRY_KEY}" "NoRepair" 1
SectionEnd

Section "Uninstall"
  Delete "$SMPROGRAMS\${APP_NAME}\${APP_NAME}.lnk"
  RMDir "$SMPROGRAMS\${APP_NAME}"
  DeleteRegKey HKCU "${APP_REGISTRY_KEY}"
  RMDir /r "$INSTDIR"
SectionEnd
```

The `${...}` values are replaced during compilation. `$INSTDIR` and `$SMPROGRAMS` are evaluated when the user runs Setup.

This is a **per-user** package: `$LOCALAPPDATA`, `HKCU`, and `RequestExecutionLevel user` form a consistent model that does not require administrator privileges.

> `RMDir /r "$INSTDIR"` assumes that the application stores no user documents or settings in its installation directory. Put such data in `$APPDATA\MyApp` or `$LOCALAPPDATA\MyApp`.

### Reading the script from top to bottom

`Unicode True` produces a Unicode installer. `MUI2.nsh` provides the Modern UI 2 macros; it downloads nothing because the file ships with NSIS.

The `!define` instructions name repeated values. The `!ifndef` blocks provide a default or require a compilation parameter. Here, omitting `PUBLISH_DIR` causes an immediate error instead of silently building an empty package.

`OutFile` names the Setup created on the build machine. `InstallDir` specifies its default destination on the target machine. `InstallDirRegKey` retrieves the previous location from the registry during reinstallation.

`$LOCALAPPDATA`, `HKCU`, and `RequestExecutionLevel user` express a current-user installation. Changing only one would create an inconsistent model. An installation under Program Files for every account would instead require an administrator strategy with `HKLM`.

In the `Install` section, `File /r` embeds the complete `dotnet publish` output. `WriteUninstaller` creates the reverse program. The remaining instructions create the shortcut and register the metadata shown by Windows under **Installed apps**.

The `Uninstall` section must be symmetrical: it removes the shortcut, registry entry, and files owned by the package. This symmetry is an important installer property, not mere polish.

## Step 3—compile Setup

```bat
mkdir artifacts
"C:\Program Files (x86)\NSIS\makensis.exe" /NOCD /V3 /DAPP_VERSION=1.0.0 /DPUBLISH_DIR=publish\win-x64 /DOUTPUT_FILE=artifacts\MyApp-Setup.exe packaging\windows\MyApp.nsi
```

`/NOCD` tells MakeNSIS to keep the repository root as its working directory, so both relative paths remain valid. The `/D` options must appear before the `.nsi` file. A successful compilation creates `artifacts\MyApp-Setup.exe`. Replace `/V3` with `/V4` for more diagnostics.

The whole process is now complete. PowerShell is unnecessary: `dotnet` and `makensis.exe` are simply two command-line programs.

## Adapt the example

For your application, change only:

- the `.csproj` path in `dotnet publish`;
- `APP_NAME`, `APP_EXE`, and `APP_PUBLISHER` in the `.nsi` file;
- optionally the `win-x64` RID when creating a package for another architecture.

Use a numeric `MAJOR.MINOR.PATCH` version here, such as `2.5.0`. The Windows resource adds a fourth component: `2.5.0.0`.

## Verify the result

On a Windows VM or test account:

1. run `artifacts\MyApp-Setup.exe`;
2. check the shortcut and launch the application;
3. check its entry under **Installed apps**;
4. uninstall it;
5. confirm that the shortcut, directory, and Windows entry are gone.

Silent installation and removal:

```bat
artifacts\MyApp-Setup.exe /S
"%LOCALAPPDATA%\Programs\MyApp\Uninstall.exe" /S
```

These commands modify your Windows profile. Automate them only in a disposable environment.

## What about TidyMemo?

[TidyMemo](https://github.com/agailloty/TidyMemo) follows the same steps with Avalonia, Native AOT, and additional metadata. Those choices make its build less universal, so they are not required to complete this first tutorial.

[Part two](/en/blog/dotnet-makensis-2/) starts with this exact script and progressively adds display versions, upgrades, logging, signing, and delivery tests.

### References

- [NSIS Users Manual](https://nsis.sourceforge.io/Docs/)
- [MakeNSIS command-line usage](https://nsis.sourceforge.io/Docs/Chapter3.html)
- [Modern UI 2 documentation](https://nsis.sourceforge.io/Docs/Modern%20UI%202/Readme.html)

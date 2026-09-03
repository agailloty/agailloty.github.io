---
title: "Shipping a .NET application with MakeNSIS — 2. Hardening the package"
description: "Improve a .NET NSIS installer step by step: versions, upgrades, data, diagnostics, signing, and tests, without depending on PowerShell."
date: 2026-09-01T00:00:00+02:00
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [dotnet, csharp, avalonia, windows, nsis, packaging]
series: ["MakeNSIS for .NET"]
---

[Part one](/en/blog/dotnet-makensis-1/) produces a working installer with `dotnet`, `makensis.exe`, and one complete `.nsi` file. This part starts from that result. Every change says exactly what to replace or add; PowerShell remains optional.

Creating Setup is only the first step. An installer becomes a delivery contract when it encounters an existing version, a running application, user data, or a partial uninstall. MakeNSIS makes these decisions visible in the repository, but it cannot make them for the team.

The goal is therefore not to accumulate macros. It is to define what the product promises and translate each promise into verifiable behavior: which version may replace another, which files belong to the package, which data must survive, and which artifact is actually released.

## The installation contract

The package keeps a consistent model:

| Decision | Value |
|---|---|
| Scope | current user |
| Installation | `$LOCALAPPDATA\Programs\MyApp` |
| Registry | `HKCU` |
| Privileges | `RequestExecutionLevel user` |
| Architecture | `win-x64` |
| Runtime | self-contained |
| User data | outside `$INSTDIR` |

Changing only `RequestExecutionLevel` to `admin` does not create a proper per-machine package. The directory, registry, shortcuts, and multi-user scenarios would also need review.

## Support prereleases without breaking Windows

A display version such as `2.5.0-beta.1` is not a valid Windows version resource. Replace `APP_VERSION` in the `.nsi` file with two symbols:

```nsis
!ifndef APP_DISPLAY_VERSION
  !define APP_DISPLAY_VERSION "0.0.0-dev"
!endif
!ifndef APP_FILE_VERSION
  !define APP_FILE_VERSION "0.0.0.0"
!endif

Name "${APP_NAME} ${APP_DISPLAY_VERSION}"
VIProductVersion "${APP_FILE_VERSION}"
VIAddVersionKey /LANG=1033 "FileVersion" "${APP_DISPLAY_VERSION}"
```

Also use this in the installation section:

```nsis
WriteRegStr HKCU "${APP_REGISTRY_KEY}" "DisplayVersion" "${APP_DISPLAY_VERSION}"
```

The compilation command becomes:

```bat
"C:\Program Files (x86)\NSIS\makensis.exe" /NOCD /V3 /DAPP_DISPLAY_VERSION=2.5.0-beta.1 /DAPP_FILE_VERSION=2.5.0.0 /DPUBLISH_DIR=publish\win-x64 /DOUTPUT_FILE=artifacts\MyApp-Setup.exe packaging\windows\MyApp.nsi
```

Both values are now explicit. No hidden conversion depends on a scripting language.

## Refuse downgrades

Add these includes near the top of the `.nsi` file:

```nsis
!include "LogicLib.nsh"
!include "WordFunc.nsh"
```

Store the version during installation:

```nsis
WriteRegStr HKCU "${APP_REGISTRY_KEY}" "PackageVersion" "${APP_FILE_VERSION}"
```

Then add this function before the `Section` blocks:

```nsis
Function .onInit
  ReadRegStr $0 HKCU "${APP_REGISTRY_KEY}" "PackageVersion"
  ${If} $0 != ""
    ${VersionCompare} "${APP_FILE_VERSION}" "$0" $1
    ${If} $1 == 2
      MessageBox MB_ICONSTOP|MB_OK \
        "A newer version ($0) is already installed. Uninstall it before installing ${APP_DISPLAY_VERSION}."
      Abort
    ${EndIf}
  ${EndIf}
FunctionEnd
```

`VersionCompare` returns `2` when its second argument is newer. The product rule is explicit: reinstalling the same version and upgrading are allowed; downgrading is refused.

This check does not solve the case of a running application. Depending on the product, request shutdown, integrate Restart Manager, or block installation with a message. This behavior must be tested with the real executable; no universal directive can make every desktop update safe.

## Delete only what the installer owns

`RMDir /r "$INSTDIR"` is acceptable only when the directory contains package files exclusively. The application should write preferences and documents elsewhere:

```text
%APPDATA%\MyApp          roaming settings
%LOCALAPPDATA%\MyApp     cache and local data
```

The uninstaller from part one preserves both locations. If the product should offer to remove them, add a clearly named optional page or section; do not make their deletion implicit.

An upgrade that simply copies files may leave a file removed from the new publication. For a small, controlled package, list obsolete files before copying:

```nsis
Section "Install"
  Delete "$INSTDIR\OldName.dll"
  Delete "$INSTDIR\old-tool.exe"

  SetOutPath "$INSTDIR"
  File /r "${PUBLISH_DIR}\*"
  ; rest of the section...
SectionEnd
```

Do not recursively erase `$INSTDIR` before an upgrade: a faulty storage policy could destroy data. When the list grows, generate and version a package manifest or design a dedicated migration.

## Produce a useful build log

MakeNSIS can write its own log. The `/Opath` syntax has no space after `/O`:

```bat
"C:\Program Files (x86)\NSIS\makensis.exe" /NOCD /V4 /Oartifacts\makensis.log /DAPP_DISPLAY_VERSION=2.5.0 /DAPP_FILE_VERSION=2.5.0.0 /DPUBLISH_DIR=publish\win-x64 /DOUTPUT_FILE=artifacts\MyApp-Setup.exe packaging\windows\MyApp.nsi
```

In `cmd.exe`, the following command prints the log and stops immediately on failure:

```bat
"C:\Program Files (x86)\NSIS\makensis.exe" /NOCD /V4 /Oartifacts\makensis.log /DAPP_DISPLAY_VERSION=2.5.0 /DAPP_FILE_VERSION=2.5.0.0 /DPUBLISH_DIR=publish\win-x64 /DOUTPUT_FILE=artifacts\MyApp-Setup.exe packaging\windows\MyApp.nsi || (type artifacts\makensis.log & exit /b 1)
```

This remains a terminal command rather than a PowerShell script to maintain.

## A practical definition of reproducibility

Here, “reproducible” means that the same revision and tool versions create a working package through the same command. It does not yet promise byte-for-byte identical output.

To stabilize the process:

- pin the SDK with `global.json`;
- commit the `.nsi` file and NuGet lock files;
- document the NSIS version;
- archive the source commit, tool versions, and Setup SHA-256;
- build releases from a clean Git tree.

Calculate the hash with a standard Windows tool:

```bat
certutil -hashfile artifacts\MyApp-Setup.exe SHA256
```

## Authenticode signing

Sign the published executables that require it, build Setup, and then sign Setup:

```text
dotnet publish → sign application → makensis → sign Setup → SHA-256
```

With `signtool.exe` from the Windows SDK, the general form is:

```bat
signtool sign /fd SHA256 /tr https://TIMESTAMP-SERVICE-URL /td SHA256 /sha1 CERTIFICATE-THUMBPRINT artifacts\MyApp-Setup.exe
```

The URL and certificate-selection method depend on your provider and key storage. Never commit a private certificate or secret. Calculate the final hash after signing because the signature changes the executable.

## Test matrix

| Scenario | Expected result |
|---|---|
| interactive installation | correct pages, destination, shortcut, and launch |
| `/S` installation | zero exit code and executable present |
| same-version reinstall | no duplicates |
| N → N+1 upgrade | updated version and obsolete files removed |
| downgrade | message followed by cancellation |
| application running | documented policy respected |
| path containing spaces | successful compilation and installation |
| interactive and `/S` removal | directory, shortcut, and registry removed |
| user data | preserved unless removal was explicitly requested |
| standard account | no elevation |

Begin with silent commands in a disposable VM:

```bat
artifacts\MyApp-Setup.exe /S
if not exist "%LOCALAPPDATA%\Programs\MyApp\MyApp.exe" exit /b 1
"%LOCALAPPDATA%\Programs\MyApp\Uninstall.exe" /S
if exist "%LOCALAPPDATA%\Programs\MyApp\MyApp.exe" exit /b 1
```

This covers the basic path. Upgrade tests require keeping two different Setup files and installing them in sequence.

## One command—only when it becomes useful

The process rests on two explicit commands: `dotnet publish` and `makensis.exe`. They are easy to inspect and repair independently, and a Windows CI runner can execute them as written.

If the team later wants one entry point, a `package.cmd` file, an MSBuild target, or the repository's existing build system can wrap them. That choice comes after understanding the process and does not change MakeNSIS's role.

### Release checklist

- The RID and self-contained model are intentional.
- Display, file, and .NET versions are consistent.
- The per-user package uses `$LOCALAPPDATA` and `HKCU`.
- Durable data stays outside `$INSTDIR`.
- Upgrade, downgrade, and running-application policies are tested.
- Interactive and silent installation and removal work.
- Setup is signed according to project policy and then hashed.

### References

- [NSIS Users Manual](https://nsis.sourceforge.io/Docs/)
- [MakeNSIS options](https://nsis.sourceforge.io/Docs/Chapter3.html)
- [WordFunc and VersionCompare](https://nsis.sourceforge.io/Docs/AppendixE.html)
- [Modern UI 2](https://nsis.sourceforge.io/Docs/Modern%20UI%202/Readme.html)

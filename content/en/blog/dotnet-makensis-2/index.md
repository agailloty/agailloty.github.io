---
title: "Shipping a .NET application with MakeNSIS — 2. Hardening the package"
description: "Upgrades, safe uninstallation, local automation, signing, and tests: evolve a .NET NSIS script into a professional delivery process."
date: 2026-09-01T00:00:00+02:00
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [dotnet, csharp, avalonia, windows, nsis, packaging]
series: ["MakeNSIS for .NET"]
---

[Part one](/en/blog/dotnet-makensis-1/) introduced the NSIS language, compiled a first script with `makensis`, and then applied the process to a real .NET application. This second part covers the properties of a maintainable installer: replacing a version, preserving data, uninstalling safely, diagnosing builds, and signing release files.

## Start with an installation contract

Before adding macros, write down the product decisions:

| Question | Example of a consistent choice | Consequence |
|---|---|---|
| Scope | current user | `$LOCALAPPDATA`, `HKCU`, no elevation |
| Architecture | Windows x64 | `win-x64` publish, dedicated artifact |
| Runtime | self-contained | no separate .NET prerequisite, larger package |
| Payload | entire publish directory | `File /r`, inventory driven by the SDK |
| Removal | `Uninstall.exe` | entry in Windows Installed apps |
| UI | Modern UI 2, English and French | selectable wizard language |

These choices must remain consistent. Changing only `RequestExecutionLevel` to `admin` does not correctly convert a per-user package into a per-machine package.

The values in this table match a self-contained desktop application such as TidyMemo, but the reasoning applies to any .NET project. An enterprise application installed for every account might make different choices.

## Versions: separate display text from the Windows format

An installer usually handles a version displayed to users and a version resource understood by Windows. In the TidyMemo example, the script receives `MAJOR.MINOR.PATCH` and uses it as follows:

```nsis
VIProductVersion "${APP_VERSION}.0"
VIAddVersionKey /LANG=1033 "FileVersion" "${APP_VERSION}"
```

`VIProductVersion` must remain numeric. To support prereleases, use separate symbols:

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

PowerShell can validate both formats. Do not silently derive a dubious version: packaging failures should be early and readable.

## Upgrades: define behavior instead of merely overwriting

The current script reuses the registered path and copies new files. That often handles a simple update, but it leaves several cases undefined:

- a file removed from the new publish output may remain on disk;
- a running application may lock files;
- an older release may overwrite a newer one;
- renamed shortcuts or registry values may survive;
- interruption may leave a partial state.

At minimum, store and compare the installed version in `.onInit`, using `LogicLib.nsh` and `WordFunc.nsh` or a tested comparison function. Explicitly decide whether downgrade is rejected, allowed, or confirmed. To remove obsolete files, prefer a controlled inventory or carefully invoke the old uninstaller. Recursively deleting the directory before copying may destroy data mistakenly stored there.

Running-process handling needs an application-specific policy: request shutdown through Restart Manager or a plugin, detect and show an error, or fail explicitly. No single NSIS instruction can make an arbitrary desktop update atomic.

## Uninstallation: remove only what you own

A first NSIS script often ends with this instruction, which the TidyMemo example also uses:

```nsis
RMDir /r "$INSTDIR"
```

It is concise and effective while `$INSTDIR` contains installed files only. It is also the most sensitive line in the script: settings or documents written there will be deleted.

A Windows application should place mutable data elsewhere, such as `$APPDATA\TidyMemo` for roaming settings or `$LOCALAPPDATA\TidyMemo` for local data. The uninstaller can then offer to remove user data separately instead of doing so unconditionally.

Also provide useful uninstall metadata:

```nsis
WriteRegStr HKCU "${APP_REGISTRY_KEY}" "DisplayName" "${APP_NAME}"
WriteRegStr HKCU "${APP_REGISTRY_KEY}" "DisplayVersion" "${APP_DISPLAY_VERSION}"
WriteRegStr HKCU "${APP_REGISTRY_KEY}" "Publisher" "${APP_PUBLISHER}"
WriteRegStr HKCU "${APP_REGISTRY_KEY}" "DisplayIcon" "$INSTDIR\${APP_EXE}"
WriteRegStr HKCU "${APP_REGISTRY_KEY}" "InstallLocation" "$INSTDIR"
WriteRegStr HKCU "${APP_REGISTRY_KEY}" "UninstallString" '"$INSTDIR\Uninstall.exe"'
WriteRegStr HKCU "${APP_REGISTRY_KEY}" "QuietUninstallString" '"$INSTDIR\Uninstall.exe" /S'
WriteRegDWORD HKCU "${APP_REGISTRY_KEY}" "NoModify" 1
WriteRegDWORD HKCU "${APP_REGISTRY_KEY}" "NoRepair" 1
```

## A genuinely reusable local script

The local entry point should be the source of truth, with CI merely calling it. A reasonable layout is:

```text
packaging/windows/
├── TidyMemo.nsi
└── package.ps1
publish/windows-x64/       # generated, not committed
artifacts/                 # generated, not committed
```

`package.ps1` should:

1. validate the version and prerequisites;
2. resolve every path to an absolute path;
3. run `dotnet restore`, then `dotnet publish`;
4. check the expected executable and resources;
5. call `makensis` with `/V3` or `/V4`;
6. inspect `$LASTEXITCODE` after every native executable;
7. verify and hash the final artifact;
8. return non-zero on the first failure.

For focused build diagnostics:

```powershell
$log = Join-Path $artifactsDir 'makensis.log'
& $makeNsis "/O$log" '/V4' `
    "/DAPP_VERSION=$Version" `
    "/DPUBLISH_DIR=$publishDir" `
    "/DOUTPUT_FILE=$setup" `
    "/DREPOSITORY_ROOT=$repositoryRoot" `
    $nsiScript
if ($LASTEXITCODE -ne 0) {
    Get-Content -LiteralPath $log -Tail 100
    throw "NSIS compilation failed ($LASTEXITCODE)."
}
```

`/O` takes its filename without a space between the option and value, as specified by NSIS.

## Reproducibility: state what you promise

“Reproducible” has two meanings. A process is reproducible when the same command creates a working package from a given revision. Bit-for-bit identical output is stronger: timestamps, SDK, NSIS, compression, and signing may alter bytes.

To stabilize the process:

- pin the SDK with `global.json`;
- version the `.nsi` and `package.ps1` files;
- document or pin the NSIS version;
- use a NuGet lock file where project policy calls for one;
- release from a clean Git tree;
- record SHA-256, tool versions, and source commit.

Promise bit-for-bit output only after measuring it in controlled environments.

## Signing: two artifacts, two decisions

NSIS does not replace Authenticode. For professional distribution, consider signing:

1. published executables and DLLs that require their own signature;
2. the `Setup.exe` after `makensis` compiles it.

The simplified order is:

```text
dotnet publish → sign payload → makensis → sign Setup.exe → SHA-256
```

Signing changes the file, so compute the final hash afterwards. Keep certificates and secrets outside the repository. Certificate provider, key storage, and timestamp service are organizational choices; this article intentionally provides no command containing a secret.

A valid signature improves package identity and integrity, but does not by itself guarantee that Windows reputation warnings never appear.

## Minimum test matrix

| Scenario | Check |
|---|---|
| interactive installation | pages, destination, launch, shortcuts |
| `/S` installation | exit code and executable presence |
| same-version reinstall | deterministic behavior, no duplicates |
| N → N+1 upgrade | new files and obsolete-file removal |
| downgrade | documented policy is enforced |
| application is running | message or controlled shutdown |
| spaces/Unicode in path | correct compilation and execution |
| interactive and `/S` uninstall | binaries and registry removed |
| user data | preserved or removed with consent |
| standard account | no unexpected elevation |

A clean VM provides much stronger evidence than a development machine where runtimes, keys, and old files already exist.

## Appendix: invoke packaging from CI

This optional step adds no new NSIS concept. Once `package.ps1` is versioned and tested locally, any Windows CI system can install the tools and invoke:

```powershell
./packaging/windows/package.ps1 -Version '${{ steps.version.outputs.version }}'
```

Developer and runner now use the same entry point. GitHub Actions remains valuable for isolation and artifact publication, without containing a second packaging implementation.

## Release checklist

- RID and self-contained/framework-dependent model are intentional.
- .NET, display, and Windows file versions agree.
- A per-user package uses `HKCU` and a user-owned directory.
- The application writes no durable data under `$INSTDIR`.
- Install, upgrade, and uninstall are tested interactively and silently.
- Paths containing spaces work.
- The artifact is signed according to project policy, then hashed.
- CI invokes the documented local command.

MakeNSIS simplifies distribution by turning decisions otherwise buried in a ZIP, manual procedure, or opaque workflow into explicit, versioned code. The result is more than `Setup.exe`: it is a deployment contract the team can review, test, and evolve.

### References

- [NSIS Users Manual](https://nsis.sourceforge.io/Docs/)
- [NSIS compiler and installer options](https://nsis.sourceforge.io/Docs/Chapter3.html)
- [Modern UI 2](https://nsis.sourceforge.io/Docs/Modern%20UI%202/Readme.html)
- [TidyMemo NSIS script](https://github.com/agailloty/TidyMemo/blob/master/packaging/windows/TidyMemo.nsi)
- [TidyMemo delivery workflow](https://github.com/agailloty/TidyMemo/blob/master/.github/workflows/dotnet.yml)

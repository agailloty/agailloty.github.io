---
title: "Distribuer une application .NET avec MakeNSIS — 1. Du publish au Setup.exe"
description: "Créer localement un installateur Windows NSIS reproductible pour une application .NET, avec un exemple complet et sans script PowerShell."
date: 2026-09-01T00:00:00+02:00
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [dotnet, csharp, avalonia, windows, nsis, packaging]
series: ["MakeNSIS pour .NET"]
---

> **Un `dotnet publish` livre des fichiers ; un installateur livre une expérience de déploiement. MakeNSIS transforme cette dernière étape en code versionné. Pour commencer, deux commandes et un fichier `.nsi` suffisent.**

Ce guide part d'une application desktop .NET nommée `MyApp`. Il ne demande ni PowerShell, ni GitHub Actions, ni connaissance préalable de NSIS. À la fin, vous disposerez d'un `MyApp-Setup.exe` installable et désinstallable.

Une application desktop ne s'arrête pas à son exécutable. Quelqu'un doit choisir où la copier, créer les raccourcis, l'inscrire dans les applications Windows, prévoir sa mise à niveau et permettre sa désinstallation. Distribuer un ZIP délègue ces décisions à chaque utilisateur. Un installateur les prend une fois, de manière explicite et reproductible.

## Ce que fait — et ne fait pas — MakeNSIS

[NSIS](https://nsis.sourceforge.io/Docs/) est un système de création d'installateurs Windows piloté par un langage de script. `makensis.exe` en est le compilateur : il lit un fichier `.nsi`, rassemble les fichiers désignés et génère un nouvel exécutable, le Setup.

MakeNSIS ne compile pas le code C# et ne connaît ni NuGet, ni Avalonia, ni le contenu du `.csproj`. Inversement, `dotnet publish` ne crée pas les raccourcis Windows et ne sait pas comment désinstaller l'application. La frontière est volontairement nette :

```text
sources C# ── dotnet publish ──> dossier publiable
                                      │
script .nsi ─────── makensis ─────────┴──> Setup.exe
```

Cette séparation aide aussi au diagnostic. Si `dotnet publish` échoue, le problème appartient au build .NET. Si `makensis` ne trouve pas un fichier ou refuse une instruction, le problème appartient au packaging. Si le Setup est créé mais installe mal l'application, il faut examiner le comportement décrit dans le `.nsi`.

## Le modèle mental d'un script NSIS

Un `.nsi` mélange quatre familles d'instructions :

- les attributs globaux décrivent le paquet, par exemple `Name`, `OutFile` et `InstallDir` ;
- les déclarations de pages composent l'assistant visible par l'utilisateur ;
- les `Section` exécutent les opérations d'installation ou de désinstallation ;
- le préprocesseur, reconnaissable au préfixe `!`, résout les constantes, inclusions et fichiers à embarquer.

Il faut surtout distinguer **la compilation** de **l'installation**.

Pendant la compilation, `makensis.exe` lit le `.nsi`. Il développe les macros, évalue les `${...}` et incorpore les fichiers sélectionnés par `File`. Le dossier publié doit donc exister sur la machine qui construit le Setup.

Plus tard, l'utilisateur exécute le Setup obtenu. C'est seulement à ce moment que les `Section` s'exécutent, que `$INSTDIR` reçoit sa valeur, que les fichiers sont extraits et que le registre est modifié.

Ainsi, `${PUBLISH_DIR}` représente un dossier de build sur le poste du développeur, tandis que `$INSTDIR` représente le dossier final sur le poste de l'utilisateur. Les deux syntaxes se ressemblent, mais n'existent ni au même moment ni sur la même machine.

## Le parcours complet

```text
MyApp.csproj
    │
    └── dotnet publish ──> publish\win-x64\
                                  │
MyApp.nsi ─── makensis.exe ───────┴──> artifacts\MyApp-Setup.exe
```

Le SDK .NET décide du runtime, du mode autonome, du trimming et de l'AOT. NSIS décide du dossier d'installation, des raccourcis, du registre et de la désinstallation.

## Prérequis

Vous avez besoin de Windows, du SDK .NET utilisé par votre projet et de [NSIS](https://nsis.sourceforge.io/Download).

1. Installez le SDK .NET puis vérifiez-le avec `dotnet --info`.
2. Installez NSIS avec les composants par défaut.
3. Vérifiez le compilateur depuis `cmd.exe` ou le terminal de votre IDE :

```bat
"C:\Program Files (x86)\NSIS\makensis.exe" /VERSION
```

Si NSIS se trouve ailleurs, remplacez ce chemin dans les commandes suivantes. Aucun état n'est partagé entre les blocs de ce guide : chaque commande peut être copiée dans un nouveau terminal ouvert à la racine du projet.

## Arborescence utilisée

```text
MyApp/
├── src/
│   └── MyApp/
│       └── MyApp.csproj
└── packaging/
    └── windows/
        └── MyApp.nsi
```

Si votre projet se trouve ailleurs, seule la commande `dotnet publish` devra être adaptée.

## Étape 1 — publier l'application

Depuis la racine du dépôt :

```bat
dotnet publish src\MyApp\MyApp.csproj --configuration Release --runtime win-x64 --self-contained true --output publish\win-x64
```

Vérifiez que `publish\win-x64\MyApp.exe` existe. Ce dossier est le contrat entre .NET et NSIS : l'installateur embarquera tout son contenu.

Le mode `--self-contained true` évite d'imposer le runtime .NET au poste cible, au prix d'un paquet plus volumineux. N'ajoutez pas immédiatement trimming ou Native AOT : ces optimisations ont leurs propres prérequis et doivent être testées séparément.

## Étape 2 — créer le script NSIS complet

Créez `packaging\windows\MyApp.nsi` avec ce contenu :

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
  !error "PUBLISH_DIR doit désigner le dossier produit par dotnet publish"
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
VIAddVersionKey /LANG=1036 "ProductName" "${APP_NAME}"
VIAddVersionKey /LANG=1036 "FileVersion" "${APP_VERSION}"
VIAddVersionKey /LANG=1036 "CompanyName" "${APP_PUBLISHER}"

!insertmacro MUI_PAGE_WELCOME
!insertmacro MUI_PAGE_DIRECTORY
!insertmacro MUI_PAGE_INSTFILES
!insertmacro MUI_PAGE_FINISH
!insertmacro MUI_UNPAGE_CONFIRM
!insertmacro MUI_UNPAGE_INSTFILES
!insertmacro MUI_LANGUAGE "French"

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

Les `${...}` sont remplacés pendant la compilation. `$INSTDIR` et `$SMPROGRAMS` sont évalués lorsque l'utilisateur exécute le Setup.

Ce paquet est **par utilisateur** : `$LOCALAPPDATA`, `HKCU` et `RequestExecutionLevel user` forment un ensemble cohérent et n'exigent pas de droits administrateur.

> `RMDir /r "$INSTDIR"` suppose que l'application ne stocke aucun document ou réglage utilisateur dans son dossier d'installation. Placez ces données dans `$APPDATA\MyApp` ou `$LOCALAPPDATA\MyApp`.

### Lire le script de haut en bas

`Unicode True` produit un installateur Unicode. L'inclusion de `MUI2.nsh` apporte les macros de l'interface Modern UI 2 ; elle ne télécharge rien, car ce fichier est livré avec NSIS.

Les `!define` donnent des noms aux valeurs répétées. Les blocs `!ifndef` jouent un autre rôle : ils fournissent une valeur par défaut ou imposent un paramètre à la compilation. Ici, oublier `PUBLISH_DIR` provoque une erreur immédiate au lieu de construire silencieusement un paquet vide.

`OutFile` désigne le Setup créé sur la machine de build. `InstallDir` désigne sa destination par défaut sur la machine cible. `InstallDirRegKey` relit l'ancien emplacement dans le registre lors d'une réinstallation.

Les trois choix `$LOCALAPPDATA`, `HKCU` et `RequestExecutionLevel user` expriment une installation pour l'utilisateur courant. Les remplacer isolément créerait un modèle incohérent. Une installation dans `Program Files` pour tous les comptes demanderait au contraire une stratégie administrateur avec `HKLM`.

Dans la section `Install`, `File /r` incorpore récursivement le résultat de `dotnet publish`. `WriteUninstaller` crée le programme inverse. Les instructions suivantes créent le raccourci puis enregistrent les métadonnées que Windows affiche dans **Applications installées**.

La section `Uninstall` doit être symétrique : elle retire le raccourci, le registre et les fichiers possédés par le paquet. Cette symétrie est une propriété importante d'un installateur, pas une simple finition.

## Étape 3 — compiler le Setup

```bat
mkdir artifacts
"C:\Program Files (x86)\NSIS\makensis.exe" /NOCD /V3 /DAPP_VERSION=1.0.0 /DPUBLISH_DIR=publish\win-x64 /DOUTPUT_FILE=artifacts\MyApp-Setup.exe packaging\windows\MyApp.nsi
```

`/NOCD` demande à MakeNSIS de conserver la racine du dépôt comme répertoire courant : les deux chemins relatifs restent donc valides. Les options `/D` doivent précéder le fichier `.nsi`. Une compilation réussie crée `artifacts\MyApp-Setup.exe`. En cas d'erreur, remplacez `/V3` par `/V4`.

Vous venez de reproduire tout le processus. PowerShell n'est pas nécessaire : `dotnet` et `makensis.exe` sont simplement deux programmes en ligne de commande.

## Adapter l'exemple

Pour votre application, modifiez seulement :

- le chemin du `.csproj` dans `dotnet publish` ;
- `APP_NAME`, `APP_EXE` et `APP_PUBLISHER` dans le `.nsi` ;
- éventuellement le RID `win-x64` si vous créez un paquet pour une autre architecture.

Gardez ici une version numérique `MAJOR.MINOR.PATCH`, par exemple `2.5.0`. La ressource Windows lui ajoute une quatrième composante : `2.5.0.0`.

## Vérifier le résultat

Sur une VM Windows ou un compte de test :

1. lancez `artifacts\MyApp-Setup.exe` ;
2. vérifiez le raccourci et lancez l'application ;
3. vérifiez l'entrée dans **Applications installées** ;
4. désinstallez l'application ;
5. vérifiez la disparition du raccourci, du dossier et de l'entrée Windows.

Installation et désinstallation silencieuses :

```bat
artifacts\MyApp-Setup.exe /S
"%LOCALAPPDATA%\Programs\MyApp\Uninstall.exe" /S
```

Ces commandes modifient votre profil Windows. Automatisez-les uniquement dans un environnement jetable.

## Et TidyMemo ?

[TidyMemo](https://github.com/agailloty/TidyMemo) suit les mêmes étapes, avec Avalonia, Native AOT et davantage de métadonnées. Ces choix rendent son build moins universel ; ils ne sont donc plus nécessaires pour terminer ce premier tutoriel.

Le [second volet](/blog/dotnet-makensis-2/) part exactement du script ci-dessus pour ajouter progressivement les versions d'affichage, les mises à niveau, les journaux, la signature et les tests de livraison.

### Références

- [Manuel NSIS](https://nsis.sourceforge.io/Docs/)
- [Utilisation de MakeNSIS en ligne de commande](https://nsis.sourceforge.io/Docs/Chapter3.html)
- [Documentation Modern UI 2](https://nsis.sourceforge.io/Docs/Modern%20UI%202/Readme.html)

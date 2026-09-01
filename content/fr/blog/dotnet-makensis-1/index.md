---
title: "Distribuer une application .NET avec MakeNSIS — 1. Du publish au Setup.exe"
description: "Construire localement un installateur Windows NSIS reproductible pour une application .NET, à partir du cas réel de TidyMemo et sans dépendre de GitHub Actions."
date: 2026-09-01T00:00:00+02:00
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [dotnet, csharp, avalonia, windows, nsis, packaging]
series: ["MakeNSIS pour .NET"]
---

> **Un `dotnet publish` livre des fichiers ; un installateur livre une expérience de déploiement. MakeNSIS transforme cette dernière étape en code : reproductible, versionné et exécutable aussi bien sur un poste de développement que dans une chaîne de livraison.**

Une application desktop ne s'arrête pas à son exécutable. Il faut choisir un répertoire, copier un ensemble cohérent de fichiers, créer les raccourcis, exposer une désinstallation et permettre à Windows d'identifier le produit. Envoyer un ZIP laisse ces décisions à chaque utilisateur. Un installateur les rend explicites.

Ce premier volet construit d'abord un installateur minimal sans dépendre d'un dépôt particulier. Nous appliquerons ensuite les mêmes notions à [TidyMemo](https://github.com/agailloty/TidyMemo), une véritable application desktop Avalonia. Son automatisation GitHub ne sera étudiée qu'à la fin, comme consommateur du processus local et non comme point de départ.

## Ce que fait — et ne fait pas — `makensis`

[NSIS](https://nsis.sourceforge.io/Docs/) est un système scriptable de création d'installateurs Windows. `makensis.exe` en est le compilateur en ligne de commande : il lit un fichier `.nsi`, incorpore les fichiers désignés et produit un exécutable d'installation.

Il ne compile pas l'application .NET. La frontière est nette :

```text
sources C# ── dotnet publish ──> dossier publiable
                                      │
script .nsi ─────── makensis ─────────┴──> Setup.exe
```

Cette séparation est utile. Le SDK .NET décide du runtime, du mode autonome, du trimming ou de l'AOT ; NSIS décide de l'installation Windows. Un échec `dotnet publish` n'est donc pas un échec NSIS, et inversement.

## Le modèle mental de NSIS

Un script NSIS est un programme d'installation déclaratif et impératif. Les instructions globales décrivent le paquet (`Name`, `OutFile`, `InstallDir`). Les pages définissent l'assistant visible. Les `Section` exécutent les opérations d'installation ou de désinstallation. Enfin, le préprocesseur — instructions préfixées par `!` — résout les constantes, inclusions et fichiers à embarquer pendant la compilation.

Deux moments ne doivent pas être confondus :

- **à la compilation**, `makensis` développe les macros et incorpore les fichiers indiqués par `File` ;
- **à l'installation**, le `Setup.exe` généré utilise des variables comme `$INSTDIR`, écrit sur le poste cible et crée le désinstalleur.

Ainsi, `${PUBLISH_DIR}` peut désigner le dossier de build sur la machine du développeur, tandis que `$INSTDIR` désigne le dossier final sur la machine de l'utilisateur.

## Installer NSIS sans gestionnaire de paquets

Le seul prérequis supplémentaire au SDK .NET est l'installateur officiel NSIS pour Windows. La [page de téléchargement NSIS](https://nsis.sourceforge.io/Download) renvoie vers l'exécutable de la version stable. Au moment de cette rédaction, il s'agit de [NSIS 3.12 — `nsis-3.12-setup.exe`](https://sourceforge.net/projects/nsis/files/NSIS%203/3.12/nsis-3.12-setup.exe/download), un téléchargement d'environ 1,6 Mo. Pour un guide durable, vérifiez toujours la page officielle avant de figer une version dans un script.

1. Téléchargez l'exécutable depuis la page officielle.
2. Lancez-le et conservez les composants par défaut, qui comprennent le compilateur et les fichiers d'inclusion.
3. Terminez l'installation. Aucune extension d'éditeur ni aucun gestionnaire de paquets n'est nécessaire.

L'emplacement habituel d'une installation 32 bits sur Windows 64 bits est `C:\Program Files (x86)\NSIS`, mais un script robuste ne doit pas masquer un emplacement personnalisé :

```powershell
$candidates = @(
    (Join-Path ${env:ProgramFiles(x86)} 'NSIS\makensis.exe'),
    (Join-Path $env:ProgramFiles 'NSIS\makensis.exe')
)
$makeNsis = $candidates | Where-Object {
    Test-Path -LiteralPath $_ -PathType Leaf
} | Select-Object -First 1

if (-not $makeNsis) {
    throw 'makensis.exe est introuvable. Installez NSIS depuis https://nsis.sourceforge.io/Download.'
}
& $makeNsis /VERSION
```

Cette dernière commande doit afficher la version du compilateur.

## Premier installateur : partir d'un dossier publié

Supposons une application desktop `MyApp` dont le projet se trouve dans `src/MyApp/MyApp.csproj`. Publiez-la d'abord pour Windows :

```powershell
dotnet publish src/MyApp/MyApp.csproj `
    --configuration Release `
    --runtime win-x64 `
    --self-contained true `
    --output publish/win-x64
```

Créez ensuite `packaging/windows/MyApp.nsi` :

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
!insertmacro MUI_LANGUAGE "French"

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

Compilez-le depuis la racine du projet :

```powershell
& $makeNsis 'packaging/windows/MyApp.nsi'
if ($LASTEXITCODE -ne 0) {
    throw "makensis a échoué avec le code $LASTEXITCODE."
}
```

Ce premier script contient déjà le cycle complet : `File /r` embarque la publication, `SetOutPath` choisit sa destination, `WriteUninstaller` génère le programme de suppression et la seconde section décrit l'opération inverse. Il reste volontairement minimal : registre, icône, licence, version et chemins injectables viendront avec l'exemple réel.

## Étude de cas : TidyMemo

TidyMemo est une application desktop open source consacrée à l'organisation et au traitement local de photos et vidéos. Son interface utilise Avalonia et son projet principal se trouve dans `TidyMemo/TidyMemo.csproj`. Le dépôt contient un script NSIS dans `packaging/windows/TidyMemo.nsi`. Nous allons maintenant reprendre exactement les étapes précédentes avec ses contraintes réelles.

### Prérequis .NET de l'exemple

L'exemple correspond à l'état de TidyMemo consulté pour cet article : cible `net10.0`, runtime `win-x64`, publication autonome et `PublishAot` activé dans le projet.

Sur Windows, vérifiez le SDK :

```powershell
dotnet --info
dotnet --list-sdks
```

### Produire l'application Windows

Depuis la racine du dépôt TidyMemo :

```powershell
$version = '2.5.0'

dotnet restore TidyMemo/TidyMemo.csproj
if ($LASTEXITCODE -ne 0) { throw 'dotnet restore a échoué.' }

dotnet publish TidyMemo/TidyMemo.csproj `
    --configuration Release `
    --runtime win-x64 `
    --self-contained true `
    --output publish/windows-x64 `
    -p:Version=$version `
    -p:PublishTrimmed=true
if ($LASTEXITCODE -ne 0) { throw 'dotnet publish a échoué.' }
```

Le RID `win-x64` fixe ici une architecture. `--self-contained true` inclut les composants .NET nécessaires : l'utilisateur n'a pas à installer séparément le runtime ciblé, au prix d'un paquet plus volumineux. Le trimming demande des tests fonctionnels, car du code atteint indirectement — par réflexion, par exemple — peut être retiré. Dans TidyMemo, `PublishAot` est déjà défini dans le `.csproj` ; il ne faut pas généraliser ce choix à toute application Avalonia sans valider ses dépendances.

Contrôler le contrat remis au packageur :

```powershell
$publishDir = (Resolve-Path 'publish/windows-x64').Path
$appExe = Join-Path $publishDir 'TidyMemo.exe'
if (-not (Test-Path -LiteralPath $appExe -PathType Leaf)) {
    throw "La publication ne contient pas $appExe"
}
```

### Faire évoluer le script minimal

Le script réel se trouve dans [`packaging/windows/TidyMemo.nsi`](https://github.com/agailloty/TidyMemo/blob/master/packaging/windows/TidyMemo.nsi). Ses paramètres de build sont injectés par la ligne de commande :

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

`!define` et `!ifndef` appartiennent au préprocesseur NSIS. Le script impose le seul intrant indispensable, `PUBLISH_DIR`, et fournit des valeurs par défaut aux autres. Les symboles passés avec `/D` doivent précéder le fichier `.nsi`, car `makensis` traite ses arguments dans l'ordre.

La configuration principale exprime une installation par utilisateur :

```nsis
Name "${APP_NAME} ${APP_VERSION}"
OutFile "${OUTPUT_FILE}"
InstallDir "$LOCALAPPDATA\Programs\${APP_NAME}"
InstallDirRegKey HKCU "${APP_REGISTRY_KEY}" "InstallLocation"
RequestExecutionLevel user
SetCompressor /SOLID lzma
```

Ce choix évite une élévation administrateur. Il est cohérent avec `HKCU` et `$LOCALAPPDATA`. Une installation dans `Program Files` demanderait généralement une stratégie machine (`admin`, registre `HKLM`) : ne mélangez pas les deux modèles.

Les pages proviennent de Modern UI 2 :

```nsis
!include "MUI2.nsh"
!insertmacro MUI_PAGE_WELCOME
!insertmacro MUI_PAGE_LICENSE "${REPOSITORY_ROOT}\LICENSE"
!insertmacro MUI_PAGE_DIRECTORY
!insertmacro MUI_PAGE_INSTFILES
!insertmacro MUI_PAGE_FINISH
```

La section d'installation copie exactement la sortie de `dotnet publish`, écrit le désinstalleur, crée les raccourcis et renseigne la clé de désinstallation Windows :

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

`File /r` est évalué à la compilation : les fichiers sont embarqués dans `Setup.exe`, pas relus sur le poste cible. À l'exécution, `$INSTDIR` désigne le répertoire choisi par l'utilisateur.

### Compiler le Setup localement

À ce stade, GitHub n'intervient pas. La commande locale complète est :

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
    throw "makensis a échoué avec le code $LASTEXITCODE."
}
if (-not (Test-Path -LiteralPath $setup -PathType Leaf)) {
    throw "L'installateur attendu n'a pas été créé : $setup"
}
Get-FileHash -Algorithm SHA256 -LiteralPath $setup
```

La [syntaxe officielle de `makensis`](https://nsis.sourceforge.io/Docs/Chapter3.html) documente notamment `/V0` à `/V4` pour la verbosité, `/Ojournal.txt` pour écrire le journal et `/Dnom=valeur` pour définir un symbole. Conservez les guillemets PowerShell autour de chaque argument : les chemins contenant des espaces doivent arriver à `makensis` comme un argument unique.

## Une commande locale unique

Pour une documentation durable, placez les trois phases — validation, `dotnet publish`, `makensis` — dans `packaging/windows/package.ps1`, puis exposez une interface stable :

```powershell
./packaging/windows/package.ps1 -Version 2.5.0
```

Le script doit résoudre ses chemins depuis sa propre position, et non depuis le répertoire courant :

```powershell
param(
    [Parameter(Mandatory)]
    [ValidatePattern('^\d+\.\d+\.\d+$')]
    [string] $Version
)

$repositoryRoot = (Resolve-Path (Join-Path $PSScriptRoot '../..')).Path
Push-Location $repositoryRoot
try {
    # restore, publish, contrôles puis makensis
}
finally {
    Pop-Location
}
```

La validation à trois nombres n'est pas cosmétique. Le script TidyMemo construit `VIProductVersion "${APP_VERSION}.0"`, et la ressource de version Windows attend quatre composantes numériques. Une version comme `2.5.0-beta.1` doit être transformée en version fichier numérique, tout en conservant séparément sa version d'affichage.

## Vérifier le résultat

Un build réussi ne valide pas le comportement du paquet. Sur une machine de test ou une VM Windows propre :

1. lancer `TidyMemo-windows-x64-Setup.exe` ;
2. conserver puis modifier le dossier proposé ;
3. démarrer l'application depuis le menu Démarrer ;
4. vérifier l'entrée dans **Applications installées** ;
5. réinstaller la même version, puis une version supérieure ;
6. désinstaller et contrôler les fichiers et raccourcis restants.

NSIS accepte `/S` pour une installation silencieuse. Cela permet un smoke test automatisé :

```powershell
$testDir = Join-Path $env:LOCALAPPDATA 'Programs\TidyMemo'
Start-Process -FilePath $setup -ArgumentList '/S' -Wait
if (-not (Test-Path (Join-Path $testDir 'TidyMemo.exe'))) {
    throw 'Installation silencieuse non vérifiée.'
}
Start-Process -FilePath (Join-Path $testDir 'Uninstall.exe') -ArgumentList '/S' -Wait
```

Exécutez ce test dans un environnement jetable : il modifie le profil utilisateur courant. L'option `/D=chemin` de l'installateur existe, mais la documentation NSIS impose qu'elle soit le dernier argument, absolue et non entourée de guillemets.

## Intégrer ensuite le processus à une CI

Une fois seulement le processus local compris et validé, une CI peut l'appeler. Dans TidyMemo, le [job Windows](https://github.com/agailloty/TidyMemo/blob/master/.github/workflows/dotnet.yml) prépare .NET et NSIS, calcule la version, publie l'application, compile le `.nsi` et téléverse les artefacts. Ces étapes ne définissent pas le fonctionnement de MakeNSIS : elles automatisent des commandes déjà exécutables sur un poste Windows.

Le bon sens de dépendance est donc : la CI appelle l'automatisation locale, et non l'automatisation locale qui cherche à reproduire des variables propres à GitHub. Le prochain volet partira de cette base pour traiter les mises à niveau, les données utilisateur, la signature, les journaux et les contrôles de livraison.

### Références

- [Manuel NSIS](https://nsis.sourceforge.io/Docs/)
- [Utilisation de MakeNSIS en ligne de commande](https://nsis.sourceforge.io/Docs/Chapter3.html)
- [Documentation Modern UI 2](https://nsis.sourceforge.io/Docs/Modern%20UI%202/Readme.html)
- [Workflow TidyMemo](https://github.com/agailloty/TidyMemo/blob/master/.github/workflows/dotnet.yml)
- [Script NSIS TidyMemo](https://github.com/agailloty/TidyMemo/blob/master/packaging/windows/TidyMemo.nsi)

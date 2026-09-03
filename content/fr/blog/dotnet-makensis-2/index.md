---
title: "Distribuer une application .NET avec MakeNSIS — 2. Fiabiliser le packaging"
description: "Faire évoluer pas à pas un installateur NSIS .NET : versions, mises à niveau, données, diagnostic, signature et tests, sans dépendre de PowerShell."
date: 2026-09-01T00:00:00+02:00
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [dotnet, csharp, avalonia, windows, nsis, packaging]
series: ["MakeNSIS pour .NET"]
---

Le [premier volet](/blog/dotnet-makensis-1/) produit un installateur fonctionnel avec `dotnet`, `makensis.exe` et un fichier `.nsi` complet. Ce second volet part de ce résultat. Chaque changement indique exactement quoi remplacer ou ajouter ; PowerShell reste facultatif.

Créer un Setup n'est que la première étape. Un installateur devient un contrat de livraison dès qu'il rencontre une version déjà présente, une application ouverte, des données utilisateur ou une désinstallation partielle. MakeNSIS rend ces décisions visibles dans le dépôt, mais ne peut pas les prendre à la place de l'équipe.

L'enjeu de ce volet n'est donc pas d'accumuler des macros. Il consiste à définir ce que le produit promet, puis à traduire chaque promesse en comportement vérifiable : quelle version peut remplacer quelle autre, quels fichiers appartiennent au paquet, quelles données doivent survivre et quel artefact est effectivement publié.

## Le contrat conservé

Notre paquet reste cohérent :

| Décision | Valeur |
|---|---|
| Portée | utilisateur courant |
| Installation | `$LOCALAPPDATA\Programs\MyApp` |
| Registre | `HKCU` |
| Privilèges | `RequestExecutionLevel user` |
| Architecture | `win-x64` |
| Runtime | autonome |
| Données utilisateur | hors de `$INSTDIR` |

Passer uniquement à `RequestExecutionLevel admin` ne suffit pas pour obtenir un paquet par machine : il faudrait aussi revoir le dossier, le registre, les raccourcis et les scénarios multi-utilisateurs.

## Accepter les préversions sans casser Windows

Une version affichée comme `2.5.0-beta.1` n'est pas une ressource de version Windows valide. Remplacez `APP_VERSION` dans le `.nsi` par deux symboles :

```nsis
!ifndef APP_DISPLAY_VERSION
  !define APP_DISPLAY_VERSION "0.0.0-dev"
!endif
!ifndef APP_FILE_VERSION
  !define APP_FILE_VERSION "0.0.0.0"
!endif

Name "${APP_NAME} ${APP_DISPLAY_VERSION}"
VIProductVersion "${APP_FILE_VERSION}"
VIAddVersionKey /LANG=1036 "FileVersion" "${APP_DISPLAY_VERSION}"
```

Dans la section d'installation, utilisez également :

```nsis
WriteRegStr HKCU "${APP_REGISTRY_KEY}" "DisplayVersion" "${APP_DISPLAY_VERSION}"
```

La compilation devient :

```bat
"C:\Program Files (x86)\NSIS\makensis.exe" /NOCD /V3 /DAPP_DISPLAY_VERSION=2.5.0-beta.1 /DAPP_FILE_VERSION=2.5.0.0 /DPUBLISH_DIR=publish\win-x64 /DOUTPUT_FILE=artifacts\MyApp-Setup.exe packaging\windows\MyApp.nsi
```

Les deux valeurs sont désormais explicites. Aucune conversion cachée ne dépend d'un langage de script.

## Refuser une rétrogradation

Ajoutez l'inclusion suivante en haut du `.nsi` :

```nsis
!include "LogicLib.nsh"
!include "WordFunc.nsh"
```

Enregistrez la version lors de l'installation :

```nsis
WriteRegStr HKCU "${APP_REGISTRY_KEY}" "PackageVersion" "${APP_FILE_VERSION}"
```

Puis ajoutez cette fonction avant les `Section` :

```nsis
Function .onInit
  ReadRegStr $0 HKCU "${APP_REGISTRY_KEY}" "PackageVersion"
  ${If} $0 != ""
    ${VersionCompare} "${APP_FILE_VERSION}" "$0" $1
    ${If} $1 == 2
      MessageBox MB_ICONSTOP|MB_OK \
        "Une version plus récente ($0) est déjà installée. Désinstallez-la avant d'installer ${APP_DISPLAY_VERSION}."
      Abort
    ${EndIf}
  ${EndIf}
FunctionEnd
```

`VersionCompare` renvoie `2` lorsque le premier argument est plus ancien que le second. La règle produit est ici claire : même version et mise à niveau sont permises, rétrogradation refusée.

Cette vérification ne résout pas le cas d'une application encore ouverte. Selon l'application, il faut demander sa fermeture, intégrer Restart Manager ou bloquer l'installation avec un message. Ce comportement doit être testé avec le véritable exécutable ; il n'existe pas de directive universelle sûre.

## Ne supprimer que ce que l'installateur possède

`RMDir /r "$INSTDIR"` est acceptable seulement si le dossier contient exclusivement les fichiers du paquet. L'application doit écrire ses préférences et documents ailleurs, par exemple dans :

```text
%APPDATA%\MyApp          réglages itinérants
%LOCALAPPDATA%\MyApp     cache et données locales
```

La désinstallation du premier volet préserve ces deux emplacements. Si le produit doit proposer leur suppression, ajoutez une page ou une section optionnelle clairement nommée ; ne la rendez pas implicite.

Une mise à niveau par simple copie peut laisser un ancien fichier qui n'existe plus dans la nouvelle publication. Pour un petit paquet maîtrisé, listez explicitement ces fichiers avant la copie :

```nsis
Section "Install"
  Delete "$INSTDIR\AncienNom.dll"
  Delete "$INSTDIR\ancien-outil.exe"

  SetOutPath "$INSTDIR"
  File /r "${PUBLISH_DIR}\*"
  ; suite de la section...
SectionEnd
```

Évitez d'effacer récursivement `$INSTDIR` avant une mise à niveau : une mauvaise politique de stockage pourrait alors détruire des données. Lorsque la liste devient longue, générez et versionnez un inventaire du paquet ou concevez une migration dédiée.

## Obtenir un journal exploitable

MakeNSIS sait écrire son propre journal. La syntaxe `/Ochemin` ne contient pas d'espace après `/O` :

```bat
"C:\Program Files (x86)\NSIS\makensis.exe" /NOCD /V4 /Oartifacts\makensis.log /DAPP_DISPLAY_VERSION=2.5.0 /DAPP_FILE_VERSION=2.5.0.0 /DPUBLISH_DIR=publish\win-x64 /DOUTPUT_FILE=artifacts\MyApp-Setup.exe packaging\windows\MyApp.nsi
```

Dans `cmd.exe`, le code suivant affiche le journal et arrête immédiatement la commande en cas d'échec :

```bat
"C:\Program Files (x86)\NSIS\makensis.exe" /NOCD /V4 /Oartifacts\makensis.log /DAPP_DISPLAY_VERSION=2.5.0 /DAPP_FILE_VERSION=2.5.0.0 /DPUBLISH_DIR=publish\win-x64 /DOUTPUT_FILE=artifacts\MyApp-Setup.exe packaging\windows\MyApp.nsi || (type artifacts\makensis.log & exit /b 1)
```

Cela reste une commande de terminal, pas un script PowerShell à maintenir.

## Reproductibilité raisonnable

Ici, « reproductible » signifie qu'une même révision et les mêmes versions d'outils produisent un paquet fonctionnel par la même commande. Cela ne promet pas encore un fichier identique octet par octet.

Pour stabiliser le procédé :

- épinglez le SDK avec `global.json` ;
- commitez le `.nsi` et les fichiers de verrouillage NuGet ;
- documentez la version de NSIS ;
- archivez le commit, les versions des outils et le SHA-256 du Setup ;
- construisez les releases depuis un arbre Git propre.

Vous pouvez calculer le hash avec un outil Windows standard :

```bat
certutil -hashfile artifacts\MyApp-Setup.exe SHA256
```

## Signature Authenticode

Signez d'abord les exécutables publiés qui doivent l'être, construisez le Setup, puis signez le Setup :

```text
dotnet publish → signature de l'application → makensis → signature du Setup → SHA-256
```

Avec le `signtool.exe` du Windows SDK, la forme générale est :

```bat
signtool sign /fd SHA256 /tr https://URL-DU-SERVICE-HORODATAGE /td SHA256 /sha1 EMPREINTE_DU_CERTIFICAT artifacts\MyApp-Setup.exe
```

L'URL et le mode de sélection du certificat dépendent de votre fournisseur et de l'endroit où la clé est conservée. Ne placez ni certificat privé ni secret dans le dépôt. Calculez le hash final après la signature, car elle modifie l'exécutable.

## Matrice de tests à exécuter

| Scénario | Résultat attendu |
|---|---|
| installation interactive | pages, destination, raccourci et lancement corrects |
| installation `/S` | code retour nul et exécutable présent |
| réinstallation identique | aucun doublon |
| mise à niveau N → N+1 | version actualisée et fichiers obsolètes retirés |
| rétrogradation | message puis abandon |
| application ouverte | politique annoncée respectée |
| chemin avec espaces | compilation et installation correctes |
| désinstallation et `/S` | dossier, raccourci et registre supprimés |
| données utilisateur | préservées sans demande explicite |
| compte standard | aucune élévation |

Commencez par les commandes silencieuses dans une VM jetable :

```bat
artifacts\MyApp-Setup.exe /S
if not exist "%LOCALAPPDATA%\Programs\MyApp\MyApp.exe" exit /b 1
"%LOCALAPPDATA%\Programs\MyApp\Uninstall.exe" /S
if exist "%LOCALAPPDATA%\Programs\MyApp\MyApp.exe" exit /b 1
```

Ce test couvre le parcours élémentaire. Les mises à niveau nécessitent de conserver deux Setup différents et de les installer successivement.

## Une seule commande : seulement quand elle devient utile

Le processus repose sur deux commandes explicites : `dotnet publish`, puis `makensis.exe`. Elles sont faciles à examiner et à réparer séparément. Une CI Windows peut les exécuter telles quelles.

Si l'équipe veut ensuite une seule commande, un fichier `package.cmd`, une cible MSBuild ou le système de build déjà utilisé par le dépôt peut les encapsuler. Ce choix vient après la compréhension du processus et ne change pas le rôle de MakeNSIS.

### Checklist de livraison

- Le RID et le mode autonome sont intentionnels.
- Versions d'affichage, de fichier et .NET sont cohérentes.
- Le paquet utilisateur utilise `$LOCALAPPDATA` et `HKCU`.
- Les données durables restent hors de `$INSTDIR`.
- Mise à niveau, rétrogradation et application ouverte ont une règle testée.
- Installation et désinstallation fonctionnent en interactif et en silencieux.
- Le Setup est signé selon la politique du projet, puis haché.

### Références

- [Manuel NSIS](https://nsis.sourceforge.io/Docs/)
- [Options de MakeNSIS](https://nsis.sourceforge.io/Docs/Chapter3.html)
- [WordFunc et VersionCompare](https://nsis.sourceforge.io/Docs/AppendixE.html)
- [Modern UI 2](https://nsis.sourceforge.io/Docs/Modern%20UI%202/Readme.html)

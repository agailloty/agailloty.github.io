---
title: "Distribuer une application .NET avec MakeNSIS — 2. Fiabiliser le packaging"
description: "Mises à niveau, désinstallation sûre, automatisation locale, signature et tests : faire évoluer un script NSIS .NET vers un processus de livraison professionnel."
date: 2026-09-01T00:00:00+02:00
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [dotnet, csharp, avalonia, windows, nsis, packaging]
series: ["MakeNSIS pour .NET"]
---

Le [premier volet](/blog/dotnet-makensis-1/) a présenté le langage NSIS, compilé un premier script avec `makensis`, puis appliqué le procédé à une application .NET réelle. Ce second volet traite les propriétés attendues d'un installateur maintenable : remplacer une version, préserver les données, désinstaller proprement, diagnostiquer un build et signer ce qui sera publié.

## Commencer par le contrat d'installation

Avant d'ajouter des macros, écrivez les décisions du produit :

| Question | Exemple de choix cohérent | Conséquence |
|---|---|---|
| Portée | utilisateur courant | `$LOCALAPPDATA`, `HKCU`, sans élévation |
| Architecture | Windows x64 | publication `win-x64`, un artefact dédié |
| Runtime | autonome | pas de prérequis .NET séparé, taille supérieure |
| Charge utile | tout le dossier publié | `File /r`, inventaire piloté par le SDK |
| Désinstallation | `Uninstall.exe` | entrée dans les applications Windows |
| Interface | Modern UI 2, anglais et français | langue de l'assistant sélectionnable |

Ces choix doivent rester cohérents. Passer seulement `RequestExecutionLevel` à `admin` ne transforme pas proprement un paquet par utilisateur en paquet machine.

Les valeurs de ce tableau correspondent à une application desktop autonome comme TidyMemo, mais le raisonnement s'applique à tout projet .NET. Une application d'entreprise installée pour tous les comptes pourrait faire d'autres choix.

## Versions : séparer le marketing du format Windows

Un installateur manipule généralement une version affichée aux utilisateurs et une ressource de version comprise par Windows. Dans l'exemple TidyMemo, le script reçoit une valeur `MAJOR.MINOR.PATCH` et l'utilise ainsi :

```nsis
VIProductVersion "${APP_VERSION}.0"
VIAddVersionKey /LANG=1033 "FileVersion" "${APP_VERSION}"
```

`VIProductVersion` doit rester numérique. Pour supporter une préversion, séparez deux symboles :

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

Le script PowerShell peut alors valider les deux formats. Évitez de déduire silencieusement une version invalide : une erreur de packaging doit être précoce et lisible.

## Mise à niveau : définir le comportement, pas seulement écraser

Le script actuel réutilise le chemin enregistré et copie les nouveaux fichiers. Cela suffit souvent à une mise à jour simple, mais ne traite pas tous les cas :

- un fichier supprimé de la nouvelle publication peut rester sur disque ;
- une application en cours peut verrouiller des fichiers ;
- une version plus ancienne peut écraser une version plus récente ;
- les raccourcis ou clés renommés peuvent survivre ;
- une interruption peut laisser un état partiel.

Au minimum, stockez la version installée et comparez-la dans `.onInit`, avec `LogicLib.nsh` et `WordFunc.nsh` ou une fonction de comparaison testée. Décidez explicitement si une rétrogradation est refusée, autorisée ou confirmée. Pour supprimer les fichiers obsolètes, privilégiez un inventaire maîtrisé ou l'exécution contrôlée de l'ancien désinstalleur ; effacer récursivement avant la copie peut détruire des données placées par erreur dans le répertoire d'installation.

Détecter le processus en cours nécessite une stratégie propre à l'application : fermeture demandée via Restart Manager/plugin, détection suivie d'un message, ou échec explicite. NSIS ne peut pas rendre atomique une application arbitraire par une seule directive.

## Désinstallation : ne supprimer que ce que l'on possède

Un premier script NSIS se termine souvent par cette instruction, également utilisée dans l'exemple TidyMemo :

```nsis
RMDir /r "$INSTDIR"
```

C'est simple et efficace si `$INSTDIR` ne contient que des fichiers installés. C'est aussi la ligne la plus sensible du script : si l'application écrit des documents ou réglages dans ce répertoire, ils disparaissent.

Une application Windows devrait conserver les données modifiables dans un emplacement distinct, par exemple `$APPDATA\TidyMemo` pour des réglages itinérants ou `$LOCALAPPDATA\TidyMemo` pour des données locales. Le désinstalleur peut alors proposer séparément la suppression des données utilisateur, plutôt que l'imposer.

Renseignez aussi les métadonnées attendues dans la clé de désinstallation :

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

## Un script local réellement réutilisable

Le point d'entrée local devrait être la source de vérité et la CI un simple appelant. Une structure raisonnable :

```text
packaging/windows/
├── TidyMemo.nsi
└── package.ps1
publish/windows-x64/       # généré, non versionné
artifacts/                 # généré, non versionné
```

Le script `package.ps1` doit :

1. valider la version et les prérequis ;
2. résoudre tous les chemins en absolu ;
3. exécuter `dotnet restore` puis `dotnet publish` ;
4. vérifier l'exécutable et les ressources attendues ;
5. appeler `makensis` avec `/V3` ou `/V4` ;
6. contrôler `$LASTEXITCODE` après chaque exécutable natif ;
7. vérifier et hacher l'artefact final ;
8. retourner un code non nul à la première erreur.

Pour diagnostiquer un build sans noyer la console :

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
    throw "Compilation NSIS en échec ($LASTEXITCODE)."
}
```

Attention : `/O` reçoit son nom de fichier sans espace entre l'option et la valeur, conformément à la syntaxe NSIS.

## Reproductibilité : définir le niveau promis

« Reproductible » peut signifier deux choses. Le procédé est reproductible si la même commande reconstruit un paquet fonctionnel depuis une révision donnée. Un binaire bit-for-bit identique est une exigence plus forte : horodatages, SDK, NSIS, compression et signature peuvent changer les octets.

Pour stabiliser le procédé :

- épinglez le SDK avec `global.json` ;
- versionnez le script `.nsi` et `package.ps1` ;
- documentez ou verrouillez la version NSIS ;
- restaurez avec un fichier de verrouillage NuGet si la politique du projet le prévoit ;
- construisez depuis un arbre Git propre pour les releases ;
- archivez le SHA-256, la version des outils et le commit source.

Ne promettez un résultat bit-for-bit qu'après l'avoir mesuré sur des environnements contrôlés.

## Signature : deux artefacts, deux décisions

NSIS ne remplace pas Authenticode. Pour une distribution professionnelle, envisagez de signer :

1. les exécutables et DLL publiés qui doivent porter une signature ;
2. le `Setup.exe` après sa compilation par `makensis`.

Ordre simplifié :

```text
dotnet publish → signature de la charge utile → makensis → signature du Setup.exe → SHA-256
```

La signature change le fichier : le hash final doit donc être calculé après. Conservez les certificats et secrets hors du dépôt. Le choix du fournisseur de certificat, du stockage de clé et du service d'horodatage dépend de l'organisation ; l'article ne prescrit volontairement aucune commande contenant un secret.

Une signature valide améliore l'identité et l'intégrité du paquet, mais ne garantit pas à elle seule l'absence de toute alerte de réputation Windows.

## Matrice de tests minimale

| Scénario | Vérification |
|---|---|
| installation interactive | pages, destination, lancement, raccourcis |
| installation `/S` | code retour et présence de l'exécutable |
| réinstallation identique | comportement déterministe, aucun doublon |
| mise à niveau N → N+1 | nouveaux fichiers, suppression des obsolètes |
| rétrogradation | politique annoncée respectée |
| application ouverte | message ou fermeture contrôlée |
| chemin avec espaces/Unicode | compilation et exécution correctes |
| désinstallation interactive et `/S` | binaires et registre supprimés |
| données utilisateur | préservées ou supprimées avec consentement |
| compte standard | aucune élévation inattendue |

Une VM propre donne un signal bien meilleur qu'une machine de développement sur laquelle runtimes, clés et fichiers existent déjà.

## Annexe : appeler le packaging depuis une CI

Cette étape est facultative et n'ajoute aucune notion NSIS. Une fois `package.ps1` versionné et testé localement, n'importe quelle CI Windows peut installer les outils puis appeler :

```powershell
./packaging/windows/package.ps1 -Version '${{ steps.version.outputs.version }}'
```

Le même point d'entrée est alors utilisé par le développeur et le runner. GitHub Actions reste utile pour isoler le build et publier l'artefact, mais ne contient plus une seconde implémentation du packaging.

## Checklist de livraison

- Le RID et le modèle self-contained/framework-dependent sont intentionnels.
- La version .NET, la version d'affichage et la version fichier Windows sont cohérentes.
- Le paquet par utilisateur utilise `HKCU` et un chemin utilisateur.
- L'application n'écrit pas de données durables sous `$INSTDIR`.
- Installation, mise à niveau et désinstallation sont testées en interactif et en silencieux.
- Les chemins avec espaces fonctionnent.
- L'artefact est signé selon la politique du projet, puis haché.
- La commande locale documentée est celle qu'appelle la CI.

MakeNSIS facilite la distribution parce qu'il rend explicites et versionnables les décisions qui seraient autrement laissées à un ZIP, à une procédure manuelle ou à un workflow opaque. La valeur ne réside pas seulement dans `Setup.exe`, mais dans un contrat de déploiement que l'équipe peut relire, tester et faire évoluer.

### Références

- [Manuel NSIS](https://nsis.sourceforge.io/Docs/)
- [Options du compilateur et des installateurs NSIS](https://nsis.sourceforge.io/Docs/Chapter3.html)
- [Modern UI 2](https://nsis.sourceforge.io/Docs/Modern%20UI%202/Readme.html)
- [Script NSIS TidyMemo](https://github.com/agailloty/TidyMemo/blob/master/packaging/windows/TidyMemo.nsi)
- [Workflow de livraison TidyMemo](https://github.com/agailloty/TidyMemo/blob/master/.github/workflows/dotnet.yml)

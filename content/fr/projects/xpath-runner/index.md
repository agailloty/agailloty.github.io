---
title: "XPath Runner"
description: "Évaluez des expressions XPath sur des fichiers HTML et XML — en ligne de commande ou avec une interface graphique"
date: 2025-04-25T17:25:17+12:00
draft: false
type: "projects"
weight: 2
categories: [Csharp]
link: "https://github.com/agailloty/xpathrunner"
github: "agailloty/xpathrunner"
featured: true
status: "active"
version: "v1.2.0.1"
license: "MPL-2.0"
tech: ["C#", ".NET", "Avalonia UI", "Windows", "Linux", "MacOS"]
features:
  - "Évaluation d'expressions XPath sur des fichiers HTML ou XML locaux"
  - "Support des fichiers distants via URL"
  - "Interface graphique native multiplateforme (Avalonia UI)"
  - "Chargement de fichiers par glisser-déposer"
  - "Prévisualisation des résultats en temps réel"
  - "Outil CLI pour intégration dans des scripts et pipelines"
changelog:
  - date: "2025-02-13"
    version: "v1.2.0.1"
    type: "fix"
    message: "Correction d'avertissements à la compilation — aucun changement fonctionnel"
  - date: "2024-06-01"
    version: "v1.2.0"
    type: "feature"
    message: "Ajout du support des fichiers distants via URL HTTP/HTTPS"
  - date: "2024-02-10"
    version: "v1.1.0"
    type: "feature"
    message: "Introduction de l'interface graphique XPathRunnerUI avec Avalonia"
  - date: "2023-09-15"
    version: "v1.0.0"
    type: "feature"
    message: "Première release publique — outil CLI XPathRunner"
---

XPath Runner regroupe deux outils complémentaires pour travailler avec des expressions XPath sur des documents HTML ou XML.

**XPathRunner** est l'outil en ligne de commande : il évalue une expression XPath directement sur un fichier local ou distant et affiche les résultats dans le terminal. Idéal pour l'automatisation et les scripts.

**XPathRunnerUI** est l'interface graphique construite avec le framework Avalonia. Elle permet de charger un fichier, de saisir une expression XPath et de visualiser les correspondances de façon interactive — sans écrire une seule ligne de code.

## Utilisation CLI

```sh
xpathrunner file --filepath document.html --xpath "//span[@class='price']"
```

## Démarrage rapide

Téléchargez la dernière release depuis la [page des releases GitHub](https://github.com/agailloty/xpathrunner/releases), lancez l'exécutable correspondant à votre système et commencez à évaluer des expressions XPath immédiatement.

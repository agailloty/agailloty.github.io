---
title: "Exif Renamer"
description: "Outil multiplateforme pour renommer des fichiers image à partir de leurs données EXIF"
date: 2025-04-25T17:12:17+12:00
draft: false
type: "projects"
weight: 1
categories: [Csharp]
link: "https://github.com/agailloty/ExifRenamer"
github: "agailloty/ExifRenamer"
featured: true
status: "active"
version: "v2.1.0"
license: "MIT"
tech: ["C#", ".NET 8", "Avalonia UI", "Windows", "Linux", "MacOS"]
features:
  - "Renommage automatique basé sur les métadonnées EXIF (date, appareil, GPS…)"
  - "Mode lot — traitement de centaines de fichiers en un clic"
  - "Prévisualisation du nouveau nom avant renommage"
  - "Interface graphique native multiplateforme (Windows, Linux, MacOS)"
  - "Aucune dépendance cloud — tout se passe localement"
  - "Open source sous licence MIT"
changelog:
  - date: "2025-04-25"
    version: "v2.1.0"
    type: "feature"
    message: "Support natif ARM sur MacOS (Apple Silicon)"
  - date: "2025-02-10"
    version: "v2.0.0"
    type: "breaking"
    message: "Migration vers Avalonia UI 11 — refonte complète de l'interface graphique"
  - date: "2024-11-05"
    version: "v1.4.2"
    type: "fix"
    message: "Correction d'un crash au chargement de fichiers sans données EXIF"
  - date: "2024-09-18"
    version: "v1.4.0"
    type: "feature"
    message: "Ajout d'un aperçu en temps réel du renommage avant application"
  - date: "2024-07-30"
    version: "v1.3.1"
    type: "security"
    message: "Mise à jour des dépendances — correction de vulnérabilités dans les librairies tierces"
---

Exif Renamer est un outil open source conçu pour les photographes et professionnels qui gèrent de grandes collections d'images. Il lit les métadonnées EXIF embarquées dans chaque fichier et génère automatiquement un nom de fichier structuré et lisible.

## Pourquoi Exif Renamer ?

Les appareils photo génèrent des noms comme `IMG_20240315_142301.jpg` qui ne disent rien sur le contenu. Exif Renamer transforme ça en `2024-03-15_Canon_EOS_R5_Paris.jpg` — sans effort, en masse.

## Démarrage rapide

Téléchargez la dernière release depuis GitHub, lancez l'application, sélectionnez un dossier et définissez le pattern de renommage. C'est tout.

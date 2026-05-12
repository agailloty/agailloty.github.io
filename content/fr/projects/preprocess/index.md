---
title: "Preprocess"
description: "Outil CLI multiplateforme rapide pour le prétraitement et l'analyse de données tabulaires"
date: 2025-03-27T00:00:00+00:00
draft: false
type: "projects"
weight: 3
categories: [Go]
link: "https://preprocess-cli.netlify.app/"
github: "agailloty/preprocess"
featured: true
status: "active"
version: "0.1.1"
tech: ["Go", "CLI", "Windows", "Linux", "MacOS"]
features:
  - "Prétraitement rapide de fichiers de données tabulaires (CSV, TSV…)"
  - "Opérations de groupement, d'agrégation et de résumé statistique"
  - "Calcul de différences entre deux versions d'un dataset"
  - "Skim — aperçu instantané de la structure et des types d'un fichier"
  - "Mise à l'échelle et normalisation des colonnes numériques"
  - "Installation en une commande sur Linux, macOS et Windows"
  - "Distribué via GoReleaser pour des binaires natifs optimisés"
changelog:
  - date: "2025-03-27"
    version: "0.1.1"
    type: "fix"
    message: "Mise à jour du numéro de version interne"
  - date: "2024-06-15"
    version: "0.1.0"
    type: "feature"
    message: "Première release publique — commandes skim, group, statistics, scale et diff"
---

**Preprocess** est un outil en ligne de commande écrit en Go, conçu pour accélérer les tâches répétitives de prétraitement de données. Il cible les analystes et développeurs qui travaillent avec des fichiers tabulaires (CSV, TSV) et veulent une alternative rapide et scriptable aux outils graphiques.

## Installation

**Linux / macOS**

```sh
curl -LsSf https://preprocess-cli.netlify.app/install.sh | sh
```

**Windows**

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://preprocess-cli.netlify.app/install.ps1 | iex"
```

Ou téléchargez le binaire directement depuis la [page des releases GitHub](https://github.com/agailloty/preprocess/releases/latest).

## Exemples d'utilisation

```sh
# Aperçu rapide du fichier
preprocess skim data.csv

# Statistiques descriptives
preprocess statistics data.csv

# Groupement par colonne
preprocess group data.csv --by country

# Différences entre deux fichiers
preprocess diff before.csv after.csv
```

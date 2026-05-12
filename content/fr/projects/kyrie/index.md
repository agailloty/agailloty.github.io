---
title: "Kyrie"
description: "Thème Hugo épuré et multilingue pour blog, documentation et portfolio de projets"
date: 2026-05-09T00:00:00+00:00
draft: false
type: "projects"
weight: 4
categories: [Go, web, hugo]
link: "https://github.com/agailloty/kyrie"
github: "agailloty/kyrie"
featured: true
status: "active"
version: "v0.2.0"
license: "MIT"
tech: ["Hugo", "HTML", "CSS", "Go Templates", "i18n"]
features:
  - "7 palettes de couleurs avec mode clair / sombre / auto"
  - "3 tailles d'interface (m, l, xl) configurables en une ligne"
  - "Multilingue natif — fichiers i18n FR et EN inclus"
  - "Blog en disposition cartes ou liste, avec filtre par tag"
  - "Section Docs avec sidebar de navigation et table des matières"
  - "Section Projets avec filtre par catégorie et 5 dispositions de cartes"
  - "Barre de progression de lecture sur les articles"
  - "Shortcodes syntax highlighting via Hugo Chroma"
  - "Page de projet enrichie : badges, stats GitHub live, changelog timeline"
  - "Page À propos configurable avec publications récentes"
changelog:
  - date: "2026-05-09"
    version: "v0.2.0"
    type: "feature"
    message: "Refonte de la page projet — hero grid, badges statut/version, stats GitHub asynchrones, timeline changelog"
  - date: "2026-05-09"
    version: "v0.2.0"
    type: "feature"
    message: "5 dispositions configurables pour la section projets de l'accueil (cards, spotlight, list, shelf, featured-grid)"
  - date: "2026-05-09"
    version: "v0.2.0"
    type: "feature"
    message: "Images à la une sur les cartes, les pages article et les pages projet"
  - date: "2026-05-09"
    version: "v0.2.0"
    type: "feature"
    message: "Page À propos redessinée — profil fusionné, publications configurable en cartes ou liste"
  - date: "2026-05-06"
    version: "v0.1.0"
    type: "feature"
    message: "Première release publique — blog, docs, projets, multilingue FR/EN"
---

**Kyrie** est un thème Hugo conçu pour les développeurs qui veulent un site personnel propre et rapide, sans JavaScript superflu. Il couvre les trois cas d'usage les plus courants : un blog technique, une documentation structurée et un portfolio de projets open source.

## Installation

**Git submodule (recommandé)**

```sh
git submodule add https://github.com/agailloty/kyrie themes/kyrie
```

Puis dans `hugo.toml` :

```toml
theme = "kyrie"
```

**Hugo Modules**

```toml
[module]
[[module.imports]]
  path = "github.com/agailloty/kyrie"
```

```sh
hugo mod get github.com/agailloty/kyrie
hugo mod tidy
```

## Démarrage rapide

Copiez `exampleSite/hugo.toml` comme point de départ et ajustez le `colorScheme`, la `size` et les sections de la page d'accueil.

```toml
[params]
colorScheme = "teal"   # teal | ocean | forest | sunset | lavender | slate | rose
colorMode   = "auto"   # auto | light | dark
size        = "m"      # m | l | xl
```

## Pré-requis

Hugo **0.112.0** ou supérieur (version étendue recommandée).

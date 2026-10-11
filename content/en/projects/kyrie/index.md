---
title: "Kyrie"
description: "Clean multilingual Hugo theme for blogs, documentation and project portfolios"
date: 2026-05-09T00:00:00+00:00
draft: false
type: "projects"
weight: 4
categories: [Go, web, hugo]
link: "https://github.com/agailloty/kyrie"
github: "agailloty/kyrie"
featured: true
status: "active"
version: "v0.3.0"
license: "MIT"
tech: ["Hugo", "HTML", "CSS", "Go Templates", "i18n"]
features:
  - "Bento design system for the homepage (v0.3.0)"
  - "7 color palettes with light / dark / auto mode"
  - "3 UI sizes (m, l, xl) configurable in one line"
  - "Native multilingual — FR and EN i18n files included"
  - "Blog in card or list layout, with tag filtering"
  - "Docs section with navigation sidebar and table of contents"
  - "Projects section with category filter and 5 card layouts"
  - "Reading progress bar on articles"
  - "Syntax highlighting shortcodes via Hugo Chroma"
  - "Rich project page: badges, live GitHub stats, changelog timeline"
  - "Configurable About page with recent publications"
changelog:
  - date: "2026-09-01"
    version: "v0.3.0"
    type: "feature"
    message: "Bento portfolio design system — bento grid homepage"
  - date: "2026-05-09"
    version: "v0.2.0"
    type: "feature"
    message: "Project page redesign — hero grid, status/version badges, async GitHub stats, changelog timeline"
  - date: "2026-05-09"
    version: "v0.2.0"
    type: "feature"
    message: "5 configurable layouts for the homepage projects section (cards, spotlight, list, shelf, featured-grid)"
  - date: "2026-05-09"
    version: "v0.2.0"
    type: "feature"
    message: "Featured images on cards, article pages and project pages"
  - date: "2026-05-09"
    version: "v0.2.0"
    type: "feature"
    message: "Redesigned About page — merged profile, publications as cards or list"
  - date: "2026-05-06"
    version: "v0.1.0"
    type: "feature"
    message: "First public release — blog, docs, projects, FR/EN multilingual"
---

**Kyrie** is a Hugo theme designed for developers who want a clean, fast personal
site without superfluous JavaScript. It covers the three most common use cases: a technical
blog, structured documentation, and an open source project portfolio.

## Installation

**Git submodule (recommended)**

```sh
git submodule add https://github.com/agailloty/kyrie themes/kyrie
```

Then in `hugo.toml`:

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

## Quick start

Copy `exampleSite/hugo.toml` as a starting point and adjust `colorScheme`, `size`
and the homepage sections.

```toml
[params]
colorScheme = "teal"   # teal | ocean | forest | sunset | lavender | slate | rose
colorMode   = "auto"   # auto | light | dark
size        = "m"      # m | l | xl
```

## Requirements

Hugo **0.112.0** or later (extended version recommended).

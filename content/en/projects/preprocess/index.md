---
title: "Preprocess"
description: "Fast cross-platform CLI tool for data analysis preprocessing"
date: 2025-03-27T00:00:00+00:00
draft: false
type: "projects"
weight: 3
categories: [open-source]
link: "https://preprocess-cli.netlify.app/"
github: "agailloty/preprocess"
featured: true
status: "active"
version: "0.1.1"
tech: ["Go", "CLI", "Windows", "Linux", "MacOS"]
features:
  - "Fast preprocessing of tabular data files (CSV, TSV…)"
  - "Group, aggregate and summarize operations"
  - "Diff — compute differences between two versions of a dataset"
  - "Skim — instant overview of a file's structure and column types"
  - "Scale and normalize numeric columns"
  - "One-command install on Linux, macOS and Windows"
  - "Distributed via GoReleaser for optimized native binaries"
changelog:
  - date: "2025-03-27"
    version: "0.1.1"
    type: "fix"
    message: "Update internal version number"
  - date: "2024-06-15"
    version: "0.1.0"
    type: "feature"
    message: "First public release — skim, group, statistics, scale and diff commands"
---

**Preprocess** is a command-line tool written in Go, designed to speed up repetitive data preprocessing tasks. It targets analysts and developers who work with tabular files (CSV, TSV) and want a fast, scriptable alternative to graphical tools.

## Installation

**Linux / macOS**

```sh
curl -LsSf https://preprocess-cli.netlify.app/install.sh | sh
```

**Windows**

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://preprocess-cli.netlify.app/install.ps1 | iex"
```

Or download the binary directly from the [GitHub releases page](https://github.com/agailloty/preprocess/releases/latest).

## Usage Examples

```sh
# Quick file overview
preprocess skim data.csv

# Descriptive statistics
preprocess statistics data.csv

# Group by column
preprocess group data.csv --by country

# Diff between two files
preprocess diff before.csv after.csv
```

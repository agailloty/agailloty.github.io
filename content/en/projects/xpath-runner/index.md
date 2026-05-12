---
title: "XPath Runner"
description: "Evaluate XPath expressions on HTML and XML files — from the command line or with a graphical interface"
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
  - "Evaluate XPath expressions on local HTML or XML files"
  - "Support for remote files via URL"
  - "Native cross-platform graphical interface (Avalonia UI)"
  - "Drag-and-drop file loading"
  - "Real-time result preview"
  - "CLI tool for integration in scripts and pipelines"
changelog:
  - date: "2025-02-13"
    version: "v1.2.0.1"
    type: "fix"
    message: "Fix compile-time warnings — no functional changes"
  - date: "2024-06-01"
    version: "v1.2.0"
    type: "feature"
    message: "Added support for remote files via HTTP/HTTPS URL"
  - date: "2024-02-10"
    version: "v1.1.0"
    type: "feature"
    message: "Introduced XPathRunnerUI graphical interface built with Avalonia"
  - date: "2023-09-15"
    version: "v1.0.0"
    type: "feature"
    message: "First public release — XPathRunner CLI tool"
---

XPath Runner bundles two complementary tools for working with XPath expressions on HTML or XML documents.

**XPathRunner** is the command-line tool: it evaluates an XPath expression directly on a local or remote file and prints the results to the terminal. Perfect for automation and scripting.

**XPathRunnerUI** is the graphical interface built with the Avalonia framework. It lets you load a file, enter an XPath expression, and visualize the matches interactively — without writing a single line of code.

## CLI Usage

```sh
xpathrunner file --filepath document.html --xpath "//span[@class='price']"
```

## Quick Start

Download the latest release from the [GitHub releases page](https://github.com/agailloty/xpathrunner/releases), run the executable for your platform, and start evaluating XPath expressions right away.

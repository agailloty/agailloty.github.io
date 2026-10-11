---
title: "ONNX Studio"
description: "Cross-platform desktop app to load, inspect, run and serve ONNX and scikit-learn models"
date: 2026-10-11T10:00:00Z
draft: false
type: "projects"
weight: 1
categories: [Csharp, MachineLearning]
link: "https://github.com/agailloty/ONNXStudio"
github: "agailloty/ONNXStudio"
featured: true
status: "active"
license: "MIT"
tech: ["C#", ".NET", "Avalonia UI", "ONNX Runtime", "ASP.NET Core", "Python", "scikit-learn"]
features:
  - "Load ONNX models via file picker, drag-and-drop or CLI, with validation and actionable error messages"
  - "Inspect the computation graph: category colors, live search and filter, node details, statistics, initializer list"
  - "Dynamic input forms generated from the model schema (numbers, vectors, images, text)"
  - "Real local inference with ONNX Runtime, input validation, LRU session cache and typed outputs"
  - "Embedded REST API (Kestrel): /models, /schema, /predict, /health with CORS and JSON schema/cURL previews"
  - "API sandbox to send real HTTP requests against the embedded server, with history and re-run"
  - "Open scikit-learn joblib/pickle models, inspect pipelines and parameters, run inference, and convert to ONNX with skl2onnx"
  - "VS Code style workbench with Dark+/Light+ themes and keyboard shortcuts"
---

ONNX Studio is a cross-platform desktop application for data scientists, built as a modular monolith on **Avalonia**, **ONNX Runtime** and **ASP.NET Core**. It loads, inspects, runs and serves ONNX models — and scikit-learn models too.

## Architecture

The solution is a modular monolith: modules communicate through direct .NET method calls wired by dependency injection, in a single process.

- **ONNXStudio.Core** — domain and services, no UI, no HTTP: model loading, registry, inference, LRU session management, form generation, graph analysis, and the Python worker for joblib/pickle models and skl2onnx conversion.
- **ONNXStudio.Api** — ASP.NET Core Minimal APIs exposing the loaded models over REST via an embedded Kestrel server.
- **ONNXStudioUI** — the Avalonia executable hosting the workbench UI and the API.

## Key features

Open a model with Ctrl+O, drag-and-drop, or `ONNXStudioUI --model path.onnx`. The Inspector shows the computation graph with category colors, live search, node details (attributes, inputs/outputs, dependencies) and statistics. Input forms are generated from the model schema, and inference runs through ONNX Runtime with a typed results view.

The embedded REST API exposes `/models`, `/models/{id}/schema` and `/models/{id}/predict`, with a sandbox to test real HTTP requests and copy cURL previews.

Scikit-learn models saved with joblib or pickle open like any other model: inspect pipeline steps, hyper-parameters and learned attributes, run predict/predict_proba/transform, and optionally convert to ONNX with skl2onnx. Python runs in a separate process, and the runtime can be installed standalone by the app, picked from the system, or pointed to manually.

## Build and run

```bash
dotnet build ONNXStudio.slnx
dotnet run --project ONNXStudioUI
```

Pushing a version tag builds tested packages for Windows x64, Linux x64 and macOS Apple Silicon, published as GitHub Releases with installers and SHA-256 checksums.

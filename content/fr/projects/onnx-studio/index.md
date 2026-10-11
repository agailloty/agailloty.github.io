---
title: "ONNX Studio"
description: "Application desktop multiplateforme pour charger, inspecter, exécuter et servir des modèles ONNX et scikit-learn"
date: 2026-10-11T16:00:00Z
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
  - "Chargement de modèles ONNX via sélecteur de fichiers, glisser-déposer ou CLI, avec validation et messages d'erreur explicites"
  - "Inspection du graphe de calcul : couleurs par catégorie, recherche et filtre en direct, détails des nœuds, statistiques, liste des initialiseurs"
  - "Formulaires dynamiques générés depuis le schéma du modèle (nombres, vecteurs, images, texte)"
  - "Inférence locale réelle avec ONNX Runtime, validation des entrées, cache de sessions LRU et sorties typées"
  - "API REST embarquée (Kestrel) : /models, /schema, /predict, /health avec CORS et aperçus JSON schema/cURL"
  - "Bac à sable API pour envoyer de vraies requêtes HTTP contre le serveur embarqué, avec historique et relance"
  - "Ouverture de modèles scikit-learn joblib/pickle, inspection des pipelines et paramètres, inférence, et conversion en ONNX via skl2onnx"
  - "Atelier de style VS Code avec thèmes Dark+/Light+ et raccourcis clavier"
---

ONNX Studio est une application desktop multiplateforme destinée aux data scientists, construite comme un monolithe modulaire sur **Avalonia**, **ONNX Runtime** et **ASP.NET Core**. Elle charge, inspecte, exécute et sert des modèles ONNX — et aussi des modèles scikit-learn.

## Architecture

La solution est un monolithe modulaire : les modules communiquent par appels .NET directs, câblés par injection de dépendances, dans un seul processus.

- **ONNXStudio.Core** — domaine et services, sans UI ni HTTP : chargement des modèles, registre, inférence, gestion des sessions LRU, génération de formulaires, analyse du graphe, et le worker Python pour les modèles joblib/pickle et la conversion skl2onnx.
- **ONNXStudio.Api** — Minimal APIs ASP.NET Core exposant les modèles chargés en REST via un serveur Kestrel embarqué.
- **ONNXStudioUI** — l'exécutable Avalonia qui héberge l'interface et l'API.

## Fonctionnalités principales

Ouvrez un modèle avec Ctrl+O, en glisser-déposer, ou via `ONNXStudioUI --model chemin.onnx`. L'inspecteur affiche le graphe de calcul avec couleurs par catégorie, recherche en direct, détails des nœuds (attributs, entrées/sorties, dépendances) et statistiques. Les formulaires d'entrée sont générés depuis le schéma du modèle, et l'inférence s'exécute avec ONNX Runtime avec des résultats typés.

L'API REST embarquée expose `/models`, `/models/{id}/schema` et `/models/{id}/predict`, avec un bac à sable pour tester de vraies requêtes HTTP et copier des aperçus cURL.

Les modèles scikit-learn sauvegardés en joblib ou pickle s'ouvrent comme n'importe quel modèle : inspection des étapes du pipeline, hyper-paramètres et attributs appris, exécution de predict/predict_proba/transform, et conversion optionnelle en ONNX avec skl2onnx. Python s'exécute dans un processus séparé, et le runtime peut être installé de façon autonome par l'application, choisi parmi ceux du système, ou désigné manuellement.

## Compilation et exécution

```bash
dotnet build ONNXStudio.slnx
dotnet run --project ONNXStudioUI
```

Le push d'un tag de version produit des paquets testés pour Windows x64, Linux x64 et macOS Apple Silicon, publiés en GitHub Releases avec installateurs et sommes de contrôle SHA-256.

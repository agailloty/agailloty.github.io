---
authors: [Axel-Cleris Gailloty]
title: "Les applications basées sur fichier avec .NET 10"
description: Exécuter simplement votre code
date: 2026-02-24T15:15:00Z
categories: [Csharp]
resources:
  - src: featured.png
    name: featured
---

Gardez la puissance du langage (typage fort, performances, écosystème NuGet) sans la lourdeur habituelle.  
{.p-first} <!--more-->


Vous avez déjà eu besoin d'écrire un petit script pour automatiser une tâche, parser un fichier ou tester rapidement une API ? Si vous êtes développeur C#, vous avez probablement ressenti cette friction : créer un projet, un fichier `.csproj`, configurer les dépendances... tout ça pour quelques lignes de code.

Avec .NET 10 et C# 14, Microsoft a enfin répondu à ce besoin avec les **file-based apps** (applications basées sur fichier). Une petite révolution qui change la donne pour les scripts et utilitaires.

## Le problème que ça résout

Traditionnellement, même le plus simple des programmes C# nécessitait :

- Un fichier solution (`.sln`)
- Un fichier projet (`.csproj`)
- La configuration des packages NuGet
- Une structure de dossiers

Pour un "Hello World", c'était déjà beaucoup. Pour un script de 20 lignes, c'était franchement excessif.

Résultat ? Beaucoup de développeurs C# se tournaient vers Python ou PowerShell pour leurs petits scripts, réservant C# aux "vrais" projets.

## La solution : un fichier, zéro configuration

Désormais, vous pouvez écrire votre code C# dans un simple fichier `.cs` et l'exécuter directement :

```bash
dotnet MonScript.cs
```

C'est tout. Pas de projet à créer, pas de configuration à gérer. Le SDK .NET s'occupe de tout en coulisses : il compile votre code dans un dossier temporaire, met en cache le résultat, et les exécutions suivantes sont quasi-instantanées.

## Un exemple concret : un web scraper en 20 lignes

Pour illustrer la puissance de cette approche, prenons un cas pratique. Imaginons que vous vouliez récupérer la liste des articles d'un blog. Voici comment faire avec un fichier unique :

```csharp
#:package HtmlAgilityPack@1.12.4
using HtmlAgilityPack;

HttpClient client = new HttpClient();

string url = "https://gailloty.net/blog/";
string html = await client.GetStringAsync(url);

var doc = new HtmlDocument();
doc.LoadHtml(html);

var articles = doc.DocumentNode
                .SelectNodes("//h2[@class='card__title']/text()");

if (articles != null) {
    articles.Select(x => x.InnerText)
    .ToList().ForEach(Console.WriteLine);
}
```

Enregistrez ce code dans un fichier `Scraper.cs` et exécutez-le :

```bash
dotnet Scraper.cs
```

En quelques secondes, vous obtenez la liste des titres d'articles. Simple, non ?

## Les directives magiques : `#:`

Vous avez remarqué la première ligne ? C'est une **directive** propre aux file-based apps :

```csharp
#:package HtmlAgilityPack@1.12.4
```

Cette syntaxe permet d'ajouter des packages NuGet directement dans votre fichier. Plus besoin de `dotnet add package` ou de modifier un `.csproj`. Vous déclarez vos dépendances, et le SDK s'occupe du reste.

Voici les principales directives disponibles :

| Directive | Description |
|-----------|-------------|
| `#:package Nom@version` | Ajoute un package NuGet |
| `#:sdk Nom@version` | Cible un SDK spécifique (ex: Aspire) |
| `#!/usr/bin/env dotnet` | Shebang pour exécution directe sur Unix |

## Ce qui reste identique

Rassurez-vous, vous êtes toujours en C#. Tout ce que vous connaissez fonctionne :

- **Top-level statements** : pas besoin de classe `Program` ni de méthode `Main`
- **Async/await** : notre exemple utilise `await` sans aucune cérémonie
- **LINQ** : le `.Select()` et `.ToList()` fonctionnent normalement
- **Types** : vous pouvez déclarer des classes, records, interfaces en fin de fichier
- **Toute la BCL** : `HttpClient`, `File`, `Console`... tout est là

## Quand utiliser les file-based apps ?

Cette approche est idéale pour :

- **Scripts d'automatisation** : parsing de fichiers, transformation de données
- **Prototypage rapide** : tester une idée, une API, un algorithme
- **Utilitaires CLI** : petits outils en ligne de commande
- **Apprentissage** : enseigner C# sans noyer les débutants sous la configuration
- **One-shots** : migrations de données, corrections ponctuelles

## Les limites actuelles

Soyons honnêtes, cette fonctionnalité a quelques restrictions :

1. **Un seul fichier** : vous ne pouvez pas (encore) répartir votre code sur plusieurs fichiers. Cette limitation devrait disparaître avec .NET 11.

2. **Support IDE partiel** : l'IntelliSense fonctionne, mais pas aussi bien que dans un projet complet.

3. **Pas pour la production** : pour une vraie application, préférez toujours un projet structuré.

## Migrer vers un projet complet

Votre script a grandi et mérite une vraie structure ? La migration est triviale :

```bash
dotnet project convert MonScript.cs
```

Cette commande génère un projet complet avec toutes vos dépendances préservées. Votre code migre vers `Program.cs`, et vous avez un `.csproj` prêt à l'emploi.

## Conclusion

Les file-based apps représentent un changement de philosophie pour C#. Microsoft reconnaît enfin que tous les programmes n'ont pas besoin d'être des applications d'entreprise. Parfois, on veut juste parser un fichier JSON ou scraper une page web.

Avec cette nouvelle approche, C# devient un choix viable pour les scripts du quotidien. Vous gardez la puissance du langage (typage fort, performances, écosystème NuGet) sans la lourdeur habituelle.

La prochaine fois que vous aurez besoin d'un petit utilitaire, essayez. Créez un fichier `.cs`, écrivez votre code, et lancez `dotnet MonFichier.cs`. Vous pourriez être surpris par la fluidité de l'expérience.

---
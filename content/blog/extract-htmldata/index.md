---
authors: [Axel-Cleris Gailloty]
title: Extraire des données à partir d'un document HTML
description: Manipulation de documents HTML avec HtmlAgilityPack
date: 2023-05-10T00:00:00Z
categories: [Csharp, webscraping]
tags: [dotnet, data]
resources:
  - src: featured.png
    name: featured
    params:
      alt: "C#, web and Google Chrome logos"
      attr: "Web Scraping with Html Agility Pack"
      attrlink: https://www.scrapingbee.com/blog/html-agility-pack/
---
Extraire du contenu dans un fichier HTML en utilisant la librairie .NET HtmlAgilityPack
{.p-first} <!--more-->

# Introduction :
À l'ère numérique d'aujourd'hui, extraire et traiter des données provenant de différentes sources est une tâche courante. Cela est particulièrement vrai lorsqu'il s'agit de données d'emploi, où les informations sont dispersées dans plusieurs documents HTML. Dans cet article de blog, nous allons explorer un extrait de code en C# qui démontre comment extraire efficacement des informations sur les emplois à partir de fichiers HTML en utilisant la bibliothèque HtmlAgilityPack. De plus, nous nous plongerons dans le processus de sérialisation des données extraites en formats CSV et JSON pour une analyse ultérieure et une intégration avec d'autres systèmes.

# Manipulation de documents HTML avec HtmlAgilityPack :
L'extrait de code repose sur la bibliothèque HtmlAgilityPack, un outil puissant pour l'analyse et la manipulation de documents HTML. Examinons les principales méthodes utilisées dans ce code et leurs fonctionnalités.

## Méthode ExtractJobInfo :
La méthode ExtractJobInfo est responsable de l'extraction des informations sur les emplois à partir d'un document HTML. Elle prend un objet HtmlDocument en entrée et renvoie une liste d'objets JobCard. Voici un aperçu de sa fonctionnalité :

```cs {.expand}
private static IList<JobCard> ExtractJobInfo(HtmlDocument doc)
{
    var jobCards = doc.DocumentNode.SelectNodes("//div[starts-with(@class, 'job_seen_beacon')]");
    List<JobCard> jobInfos = new List<JobCard>();
    foreach (var job in jobCards)
        jobInfos.Add(ExtractInfo(job));
    return jobInfos;
}
```

Explication :

La méthode utilise des requêtes XPath pour sélectionner tous les éléments <div> ayant un attribut de classe commençant par "job_seen_beacon".
- Les nœuds de cartes d'emploi sélectionnés sont stockés dans la variable jobCards.
- Ensuite, elle parcourt chaque nœud d'emploi et appelle la méthode ExtractInfo pour extraire le titre, l'entreprise, l'emplacement et la description à partir du nœud d'emploi.
- Les informations extraites sont utilisées pour créer un nouvel objet JobCard, qui est ajouté à la liste jobInfos.
- Enfin, la méthode renvoie la liste des cartes d'emploi.

Voici la définition de l'objet JobCard :

```cs {.expand}
public record JobCard(string Title, 
                      string Company, 
                      string Location, 
                      string Description);
```

## Méthode ExtractInfo :
La méthode ExtractInfo est appelée par la méthode ExtractJobInfo pour extraire des détails spécifiques sur un emploi à partir d'un nœud de carte d'emploi. Elle prend un objet HtmlNode en entrée et renvoie un objet JobCard. Examinons sa mise en œuvre :

```cs {.expand}
private static JobCard ExtractInfo(HtmlNode node)
{
    string title = node.SelectSingleNode(".//span[starts-with(@id, 'jobTitle')]/text()").InnerText.Trim();
    string company = node.SelectSingleNode(".//span[@class='companyName']").InnerText.Trim();
    string location = node.SelectSingleNode(".//div[@class='companyLocation']").InnerText.Trim();
    string description = node.SelectSingleNode(".//div[@class='job-snippet']").InnerText.Trim();
    return new JobCard(title, company, location, description);
}
```

Explication :

- La méthode utilise des requêtes XPath pour extraire des éléments spécifiques à l'intérieur du nœud de la carte d'emploi.
- Elle récupère le titre de l'emploi en sélectionnant l'élément <span> avec un ID commençant par "jobTitle" et en extrayant le texte interne.
- Le nom de l'entreprise est obtenu en sélectionnant l'élément <span> avec un attribut de classe égal à "companyName".
- L'emplacement est extrait en sélectionnant l'élément <div> avec un attribut de classe égal à "companyLocation".
- La description est obtenue en sélectionnant l'élément <div> avec un attribut de classe égal à "job-snippet".
- Enfin, un nouvel objet JobCard est créé en utilisant les informations extraites et renvoyé.

# Sérialisation des données d'emploi en CSV et JSON :
Après avoir extrait les informations sur les emplois des documents HTML, l'extrait de code procède à la sérialisation des données en formats CSV et JSON. Explorons le processus de sérialisation.

## Écriture dans un fichier CSV :
Les informations sur les emplois extraites sont écrites dans un fichier CSV nommé "jobs.csv" en utilisant la méthode File.WriteAllLines. Voici l'extrait de code qui accomplit cela :

```cs {.expand}
File.WriteAllLines("jobs.csv", 
    res.Select(job => $"{job.Title},{job.Company},{job.Location},{job.Description}"));
```

Explication :

- La variable `res` contient une liste aplatie d'objets JobCard obtenue en utilisant SelectMany de LINQ sur la liste allJobs.
- La méthode Select est ensuite utilisée pour transformer chaque objet JobCard en une représentation sous forme de chaîne qui inclut le titre de l'emploi, l'entreprise, l'emplacement et la description.
- Les chaînes résultantes sont écrites dans le fichier "jobs.csv" en utilisant la méthode File.WriteAllLines, où chaque ligne correspond à une entrée d'emploi unique dans le fichier CSV.

## Écriture dans un fichier JSON :
- En plus du fichier CSV, l'extrait de code sérialise également les données sur les emplois dans un fichier JSON nommé "jobs.json". Ce processus est réalisé dans la méthode Persist. Examinons de plus près sa mise en œuvre :

```cs {.expand}
private static void Persist(string filepath, IList<JobCard> data) 
{
    var encoder = JavaScriptEncoder.Create(UnicodeRanges.BasicLatin, UnicodeRanges.Latin1Supplement, UnicodeRanges.GeneralPunctuation);
    var options = new JsonSerializerOptions { WriteIndented = true, Encoder = encoder };
    string jsonString = JsonSerializer.Serialize(data, options);
    File.WriteAllText(filepath, jsonString);
}
```

Explication :

- La méthode Persist reçoit un chemin de fichier et une liste d'objets JobCard en tant que paramètres.
- Elle commence par créer une instance de JavaScriptEncoder, en spécifiant les plages de caractères Unicode à inclure dans le codage. Dans ce cas, les plages de caractères Basic Latin, Latin-1 Supplement et General Punctuation sont incluses.
- Ensuite, un objet `JsonSerializerOptions` est créé, et la propriété `WriteIndented` est définie sur true pour formater la sortie JSON avec une indentation.
- La propriété Encoder de l'objet options est assignée à l'instance d'encodeur créée précédemment.
- La méthode `JsonSerializer.Serialize` est appelée, en passant la liste de données et l'objet options, ce qui renvoie la représentation JSON sous forme de chaîne des données sur les emplois.
- Enfin, la chaîne JSON est écrite dans le chemin de fichier spécifié en utilisant la méthode `File.WriteAllText`.

# Conclusion :
Dans cet article de blog, nous avons exploré un extrait de code en C# qui présente l'utilisation de la bibliothèque HtmlAgilityPack pour la manipulation de documents HTML et a démontré comment extraire efficacement des informations sur les emplois à partir de fichiers HTML. De plus, nous avons examiné le processus de sérialisation des données extraites en formats CSV et JSON pour une analyse ultérieure et une intégration. En utilisant les requêtes XPath, les opérations LINQ et les capacités de sérialisation de System.Text.Json, cet extrait de code fournit un exemple concret d'automatisation de l'extraction et de la transformation de données. Il équipe les développeurs des connaissances et des outils nécessaires pour rationaliser les flux de traitement des données d'emploi et ouvre la voie à l'utilisation des données d'emploi dans une variété d'applications.

# Annexe

J'ai créé un [répertoire Github](https://github.com/agailloty/webscraping-cs) pour ce projet. 

Code en entier

```cs {.expand}
using HtmlAgilityPack;
using System.Text.Json;
using System.Text.Json.Serialization;
using System.Linq;
using System.Text;
using System.Text.Encodings.Web;
using System.Text.Unicode;

public class Program
{
    private static void Main(string[] args)
    {
        List<IList<JobCard>> allJobs = new();

        var files = Directory.GetFiles("files");
        foreach (var file in files)
        {
            var doc = new HtmlDocument();
            doc.Load(file, Encoding.UTF8, false);

            allJobs.Add(ExtractJobInfo(doc));
        }

        var res = allJobs.SelectMany(job => job).ToList();

        File.WriteAllLines("jobs.csv", 
                res.Select(job => $"{job.Title},{job.Company},{job.Location},{job.Description}"));

        Persist("jobs.json", allJobs.SelectMany(job => job).ToList());
    }

    private static IList<JobCard> ExtractJobInfo(HtmlDocument doc)
    {
        var jobCards = doc.DocumentNode
        .SelectNodes("//div[starts-with(@class, 'job_seen_beacon')]");

        List<JobCard> jobInfos = new();

        foreach(var job in jobCards)
            jobInfos.Add(ExtractInfo(job));

        return jobInfos;
    }

    private static JobCard ExtractInfo(HtmlNode node)
    {
        string title = node.SelectSingleNode(".//span[starts-with(@id, 'jobTitle')]/text()").InnerText.Trim();
        string company = node.SelectSingleNode(".//span[@class='companyName']").InnerText.Trim();
        string location = node.SelectSingleNode(".//div[@class='companyLocation']").InnerText.Trim();
        string description = node.SelectSingleNode(".//div[@class='job-snippet']").InnerText.Trim();
        return new JobCard(title, company, location, description);
    }

    private static void Persist(string filepath, IList<JobCard> data) 
    {
        var encoder = JavaScriptEncoder.Create(UnicodeRanges.BasicLatin, UnicodeRanges.Latin1Supplement, UnicodeRanges.GeneralPunctuation);
        var options = new JsonSerializerOptions { WriteIndented = true, Encoder = encoder };
        string jsonString = JsonSerializer.Serialize(data, options);
        File.WriteAllText(filepath, jsonString);
    }
}

public record JobCard(string Title, string Company, string Location, string Description);

```
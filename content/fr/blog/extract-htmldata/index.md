---
title: "Extraire des données à partir d'un document HTML"
description: "Manipulation de documents HTML avec HtmlAgilityPack"
date: 2023-05-10T00:00:00Z
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [dotnet, data, Csharp, webscraping]
---

Extraire du contenu dans un fichier HTML en utilisant la librairie .NET HtmlAgilityPack.

# Introduction

À l'ère numérique d'aujourd'hui, extraire et traiter des données provenant de différentes sources est une tâche courante. Cela est particulièrement vrai lorsqu'il s'agit de données d'emploi, où les informations sont dispersées dans plusieurs documents HTML. Dans cet article de blog, nous allons explorer un extrait de code en C# qui démontre comment extraire efficacement des informations sur les emplois à partir de fichiers HTML en utilisant la bibliothèque HtmlAgilityPack. De plus, nous nous plongerons dans le processus de sérialisation des données extraites en formats CSV et JSON pour une analyse ultérieure et une intégration avec d'autres systèmes.

# Manipulation de documents HTML avec HtmlAgilityPack

L'extrait de code repose sur la bibliothèque HtmlAgilityPack, un outil puissant pour l'analyse et la manipulation de documents HTML.

## Méthode ExtractJobInfo

La méthode `ExtractJobInfo` est responsable de l'extraction des informations sur les emplois à partir d'un document HTML. Elle prend un objet `HtmlDocument` en entrée et renvoie une liste d'objets `JobCard`.

```cs
private static IList<JobCard> ExtractJobInfo(HtmlDocument doc)
{
    var jobCards = doc.DocumentNode.SelectNodes("//div[starts-with(@class, 'job_seen_beacon')]");
    List<JobCard> jobInfos = new List<JobCard>();
    foreach (var job in jobCards)
        jobInfos.Add(ExtractInfo(job));
    return jobInfos;
}
```

Voici la définition de l'objet `JobCard` :

```cs
public record JobCard(string Title, 
                      string Company, 
                      string Location, 
                      string Description);
```

## Méthode ExtractInfo

La méthode `ExtractInfo` est appelée par la méthode `ExtractJobInfo` pour extraire des détails spécifiques sur un emploi à partir d'un nœud de carte d'emploi.

```cs
private static JobCard ExtractInfo(HtmlNode node)
{
    string title = node.SelectSingleNode(".//span[starts-with(@id, 'jobTitle')]/text()").InnerText.Trim();
    string company = node.SelectSingleNode(".//span[@class='companyName']").InnerText.Trim();
    string location = node.SelectSingleNode(".//div[@class='companyLocation']").InnerText.Trim();
    string description = node.SelectSingleNode(".//div[@class='job-snippet']").InnerText.Trim();
    return new JobCard(title, company, location, description);
}
```

# Sérialisation des données en CSV et JSON

## Écriture dans un fichier CSV

```cs
File.WriteAllLines("jobs.csv", 
    res.Select(job => $"{job.Title},{job.Company},{job.Location},{job.Description}"));
```

## Écriture dans un fichier JSON

```cs
private static void Persist(string filepath, IList<JobCard> data) 
{
    var encoder = JavaScriptEncoder.Create(UnicodeRanges.BasicLatin, UnicodeRanges.Latin1Supplement, UnicodeRanges.GeneralPunctuation);
    var options = new JsonSerializerOptions { WriteIndented = true, Encoder = encoder };
    string jsonString = JsonSerializer.Serialize(data, options);
    File.WriteAllText(filepath, jsonString);
}
```

# Conclusion

Dans cet article de blog, nous avons exploré un extrait de code en C# qui présente l'utilisation de la bibliothèque HtmlAgilityPack pour la manipulation de documents HTML et a démontré comment extraire efficacement des informations sur les emplois à partir de fichiers HTML. En utilisant les requêtes XPath, les opérations LINQ et les capacités de sérialisation de `System.Text.Json`, cet extrait de code fournit un exemple concret d'automatisation de l'extraction et de la transformation de données.

# Annexe

J'ai créé un [répertoire Github](https://github.com/agailloty/webscraping-cs) pour ce projet. 

```cs
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

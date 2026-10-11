---
title: "Extracting data from an HTML document"
description: "Manipulating HTML documents with HtmlAgilityPack"
date: 2023-05-10T00:00:00Z
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [dotnet, data, Csharp, webscraping]
---

Extracting content from an HTML file using the .NET library HtmlAgilityPack.

# Introduction

In today's digital era, extracting and processing data from various sources is a common task. This is especially true for job data, where information is scattered across multiple HTML documents. In this blog post, we will explore a C# code snippet that demonstrates how to efficiently extract job information from HTML files using the HtmlAgilityPack library. Additionally, we will dive into the process of serializing the extracted data into CSV and JSON formats for further analysis and integration with other systems.

# Manipulating HTML documents with HtmlAgilityPack

The code snippet relies on the HtmlAgilityPack library, a powerful tool for parsing and manipulating HTML documents.

## The ExtractJobInfo method

The `ExtractJobInfo` method is responsible for extracting job information from an HTML document. It takes an `HtmlDocument` object as input and returns a list of `JobCard` objects.

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

Here is the definition of the `JobCard` object:

```cs
public record JobCard(string Title, 
                      string Company, 
                      string Location, 
                      string Description);
```

## The ExtractInfo method

The `ExtractInfo` method is called by `ExtractJobInfo` to extract specific details about a job from a job card node.

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

# Serializing the data to CSV and JSON

## Writing to a CSV file

```cs
File.WriteAllLines("jobs.csv", 
    res.Select(job => $"{job.Title},{job.Company},{job.Location},{job.Description}"));
```

## Writing to a JSON file

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

In this blog post, we explored a C# code snippet showcasing the use of the HtmlAgilityPack library for manipulating HTML documents, and demonstrated how to efficiently extract job information from HTML files. Using XPath queries, LINQ operations and the serialization capabilities of `System.Text.Json`, this snippet provides a concrete example of automating data extraction and transformation.

# Appendix

I created a [GitHub repository](https://github.com/agailloty/webscraping-cs) for this project.

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

---
authors: [Axel-Cleris Gailloty]
title: Accéder à Postgres dans une application console C#
description: Dans le présent article je vais vous montrer comment exécuter des requêtes SQL basiques du type SELECT sur une table PostgreSQL à partir d'une application console C#.
date: 2024-03-31T00:00:00Z
categories: [Csharp]
tags: [dotnet, postgres, database]
resources:
  - src: featured.jpg
    name: featured
    params:
      alt: Rename files with custom date format
      attr: "Michael's Coding Spot"
      attrlink: https://michaelscodingspot.com/postgres-in-csharp/
---
Dans le présent article je vais vous montrer comment exécuter des requêtes SQL basiques du type SELECT sur une table PostgreSQL à partir d'une application console C#.
{.p-first} <!--more-->

Dans un [précédent article](http://gailloty.net/fr/post/postgres-pgadmin/) j'ai montré comment installer PostgreSQL et PgAdmin à l'aide de Docker pour un usage local. Dans le présent article je vais vous montrer comment exécuter des requêtes SQL basiques du type SELECT sur une table PostgreSQL à partir d'une application console C#.

## Créer un projet console C# 

Pour démontrer cette fonctionnalité, nous allons créer une simple application console .NET. Pour ce faire, naviguer dans votre explorateur de fichier puis saisir la commande suivante : 

```
dotnet new console -o console-postgres
```

Avec cette ligne de commande, le SDK .NET a généré un projet console qui affiche "Hello world" sur la console lorsque nous l'exécutons. Nous voulons afficher quelque chose de plus intéressant sur la console.

# Configuration de la connexion à la base de données
La première étape consiste à configurer la connexion à votre base de données PostgreSQL. Pour cela, vous aurez besoin de l'installation du pilote Npgsql, qui est le fournisseur de données .NET pour PostgreSQL.
Il existe sous forme de paquet Nuget que vous pouvez télécharger en tapant la commande suivante 

```
dotnet add package Npgsql
```

Puis dans le fichier `Program.cs` généré par le SDK .NET nous allons écrire ce code. 

```cs {.expand}
using Npgsql;

namespace PostgresData;
class Program
{
    private const string connectionString = "Host=localhost;Port=5432;Username=postgres;Password=$ecureP@ssword;";
    public static void DisplayTableData(string tableName) 
    {
        using var connection = new NpgsqlConnection(connectionString);
        connection.Open();

        string sqlForTableColumns = $@"SELECT column_name FROM information_schema.columns 
                                WHERE table_name='{tableName}' ORDER BY column_name";
        
        using var command = new NpgsqlCommand(sqlForTableColumns, connection);

        List<string> tableColumns = new();
        using (var reader = command.ExecuteReader()) 
        {
            while (reader.Read())
            {
                tableColumns.Add(reader.GetString(0));
            }
        }
        
        string sqlColumns = string.Join(" , ", tableColumns);

        using var sqlCommand = new NpgsqlCommand($"SELECT {sqlColumns} FROM {tableName}", connection);

        var results = new List<List<string>>();

        using (var reader = sqlCommand.ExecuteReader()) 
        {
            while (reader.Read())
            {
                List<string> rowResult = new();
                for (int i = 0; i < tableColumns.Count; i++) 
                {
                    string currentResult;
                    var currentData = reader.GetValue(i);
                    if (currentData is DBNull) 
                    {
                        currentResult = "null";
                    }
                    else 
                    {
                        currentResult = currentData.ToString();
                    }
                    rowResult.Add(currentResult);
                }
                results.Add(rowResult);
            }
        }

        Console.WriteLine(sqlColumns);

        foreach (var row in results) 
        {
            foreach (var data in row) 
            {
                Console.Write(data);
                Console.Write(", ");
            }
            Console.WriteLine("");
        }

    }
    static void Main()
    {
        DisplayTableData("pg_tables");
    }
}
```

## Explication du code 

Ce code C# se connecte à une base de données PostgreSQL locale, récupère les noms des colonnes et les données d'une table spécifique, puis affiche ces données dans la console.

Voici une explication détaillée de chaque partie du code :

1. **Namespace et Using Directives** :
   ```cs
   using Npgsql;
   ```
   Ce code importe l'espace de noms Npgsql, qui est une bibliothèque .NET pour se connecter à des bases de données PostgreSQL.

2. **Définition de l'espace de noms et de la classe principale** :
   ```cs
   namespace PostgresData;
   class Program
   {
       // Contenu de la classe
   }
   ```
   Le code se trouve dans un espace de noms `PostgresData`, et la classe principale s'appelle `Program`.

3. **Chaîne de connexion à la base de données** :
   ```cs {.expand}
   private const string connectionString = "Host=localhost;Port=5432;Username=postgres;Password=$ecureP@ssword;";
   ```
   Cette constante définit la chaîne de connexion à la base de données PostgreSQL. Elle spécifie le nom d'hôte (`localhost`), le port (`5432`), le nom d'utilisateur (`postgres`) et le mot de passe (`$ecureP@ssword`). Vous devrez ajuster ces valeurs selon votre configuration.

4. **Méthode pour afficher les données de la table** :
   ```cs
   public static void DisplayTableData(string tableName) 
   {
       // Contenu de la méthode
   }
   ```
   Cette méthode prend en paramètre le nom de la table à afficher.

5. **Connexion à la base de données** :
   ```cs
   using var connection = new NpgsqlConnection(connectionString);
   connection.Open();
   ```
   Cette partie utilise la chaîne de connexion pour établir une connexion à la base de données PostgreSQL spécifiée.

6. **Récupération des noms de colonnes de la table spécifiée** :
   ```cs {.expand}
   string sqlForTableColumns = $@"SELECT column_name FROM information_schema.columns 
                               WHERE table_name='{tableName}' ORDER BY column_name";
   ```
   Cette requête SQL récupère les noms des colonnes de la table spécifiée dans le paramètre `tableName`.

7. **Exécution de la requête SQL pour récupérer les noms des colonnes** :
   ```cs
   using var command = new NpgsqlCommand(sqlForTableColumns, connection);
   ```
   Cette commande est exécutée sur la connexion à la base de données pour récupérer les noms des colonnes.

8. **Lecture des noms de colonnes récupérés** :
   ```cs
   using (var reader = command.ExecuteReader()) 
   {
       while (reader.Read())
       {
           tableColumns.Add(reader.GetString(0));
       }
   }
   ```
   Cette partie lit les résultats de la requête SQL et ajoute les noms de colonnes à une liste appelée `tableColumns`.

9. **Construction de la requête SQL pour sélectionner les données de la table** :
   ```cs
   string sqlColumns = string.Join(" , ", tableColumns);
   ```
   Les noms de colonnes récupérés sont joints en une seule chaîne séparée par des virgules pour être utilisés dans la requête SQL de sélection.

10. **Exécution de la requête SQL pour récupérer les données de la table** :
   ```cs {.expand}
   using var sqlCommand = new NpgsqlCommand($"SELECT {sqlColumns} FROM {tableName}", connection);
   ```
   Cette commande exécute une requête SQL pour récupérer les données de la table spécifiée.

11. **Lecture des données récupérées** :
   ```cs
   using (var reader = sqlCommand.ExecuteReader()) 
   {
       while (reader.Read())
       {
           // Contenu de la boucle de lecture
       }
   }
   ```
   Cette boucle lit les données récupérées ligne par ligne et les stocke dans une liste appelée `results`.

12. **Affichage des données dans la console** :
   ```cs
   Console.WriteLine(sqlColumns);

   foreach (var row in results) 
   {
       foreach (var data in row) 
       {
           Console.Write(data);
           Console.Write(", ");
       }
       Console.WriteLine("");
   }
   ```
   Enfin, les données sont affichées dans la console. Les noms de colonnes sont affichés en premier, suivis des données de chaque ligne de la table.

13. **Méthode principale (Main)** :
   ```cs
   static void Main()
   {
       DisplayTableData("pg_tables");
   }
   ```
   Cette méthode principale appelle la méthode `DisplayTableData` en passant le nom de la table `"pg_tables"` comme argument. Cela lance le processus d'affichage des données de cette table dans la console.

   ## Résultat de la requête

   Lorsque nous exécutons ce code alors la console C# s'ouvre et nous affiche les données suivantes qui viennent de la table `pg_tables`.

   ```sh {.expand}
hasindexes , hasrules , hastriggers , rowsecurity , schemaname , tablename , tableowner , tablespace
True, False, False, False, pg_catalog, pg_statistic, postgres, null, 
True, False, False, False, pg_catalog, pg_type, postgres, null,
True, False, False, False, pg_catalog, pg_foreign_table, postgres, null,
True, False, False, False, pg_catalog, pg_authid, postgres, pg_global,
True, False, False, False, pg_catalog, pg_statistic_ext_data, postgres, null,
True, False, False, False, pg_catalog, pg_user_mapping, postgres, null,
True, False, False, False, pg_catalog, pg_subscription, postgres, pg_global,
True, False, False, False, pg_catalog, pg_attribute, postgres, null,
   ```
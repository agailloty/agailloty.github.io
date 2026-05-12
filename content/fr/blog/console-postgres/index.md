---
title: "Accéder à Postgres dans une application console C#"
description: "Dans le présent article je vais vous montrer comment exécuter des requêtes SQL basiques du type SELECT sur une table PostgreSQL à partir d'une application console C#."
date: 2024-03-31T00:00:00Z
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [dotnet, postgres, database, Csharp]
---

Dans le présent article je vais vous montrer comment exécuter des requêtes SQL basiques du type SELECT sur une table PostgreSQL à partir d'une application console C#.

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

```cs
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

2. **Chaîne de connexion à la base de données** :
   ```cs
   private const string connectionString = "Host=localhost;Port=5432;Username=postgres;Password=$ecureP@ssword;";
   ```
   Cette constante définit la chaîne de connexion à la base de données PostgreSQL. Vous devrez ajuster ces valeurs selon votre configuration.

3. **Connexion à la base de données** :
   ```cs
   using var connection = new NpgsqlConnection(connectionString);
   connection.Open();
   ```

4. **Récupération des noms de colonnes** :
   ```cs
   string sqlForTableColumns = $@"SELECT column_name FROM information_schema.columns 
                               WHERE table_name='{tableName}' ORDER BY column_name";
   ```

5. **Affichage des données dans la console** :
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

## Résultat de la requête

Lorsque nous exécutons ce code alors la console C# s'ouvre et nous affiche les données suivantes qui viennent de la table `pg_tables`.

```sh
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

---
title: "EF Core : la compilation des requêtes à la volée, et pourquoi le Native AOT reste hors de portée"
description: "Comment EF Core compile vos requêtes LINQ à l'exécution, pourquoi cette architecture bloque le Native AOT, ce que Microsoft construit pour y remédier, et l'état de l'intégration PostgreSQL."
date: 2026-10-30T09:23:00Z
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [dotnet, efcore, postgresql, aot, performance]
---

En développant un petit serveur MCP en .NET, j'ai dû trancher une question d'architecture : publier l'application en Native AOT ou pas. La réponse courte : non. EF Core, l'ORM que le projet utilise, ne le permet pas de façon fiable aujourd'hui. Cet article explique pourquoi, en détail. On va regarder comment EF Core exécute réellement une requête LINQ, pourquoi ce mécanisme est incompatible avec la compilation ahead-of-time, ce que Microsoft construit pour débloquer la situation, et où en est l'intégration de PostgreSQL dans tout ça.

## Ce qui se passe quand une requête LINQ s'exécute

Une requête EF Core n'est pas du SQL. C'est un arbre d'expression, produit par le compilateur C# à partir de votre LINQ. Quand vous appelez `ToListAsync()`, EF Core doit transformer cet arbre en SQL, puis transformer les lignes retournées en objets .NET. La documentation appelle cette étape la "compilation" de la requête ([Microsoft Learn, Advanced Performance Topics](https://learn.microsoft.com/en-us/ef/core/performance/advanced-performance-topics)).

Cette compilation fait plusieurs choses. Elle extrait les paramètres, construit la représentation intermédiaire de la requête, génère le SQL, et surtout produit ce qu'EF Core appelle un shaper : le code qui matérialise chaque ligne de la base en objet .NET. Ce shaper est du code généré à l'exécution. Le ticket de suivi du projet est explicite : ce processus est coûteux, augmente le temps de démarrage, et repose sur de la génération de code à l'exécution ([dotnet/efcore#25009](https://github.com/dotnet/efcore/issues/25009)).

Ce coût n'est payé qu'une fois. EF Core cache le résultat de la compilation par forme d'arbre. Deux requêtes avec la même structure, même avec des valeurs différentes, réutilisent la même sortie compilée. La documentation donne l'exemple classique : si vous écrivez `p.Title == "post1"` puis `p.Title == "post2"` avec des constantes, chaque requête est compilée séparément et produit du SQL différent. Si vous paramétrez avec une variable, les deux requêtes ont la même forme, et une seule compilation suffit. EF Core expose d'ailleurs une métrique, le Query Cache Hit Rate, qui doit atteindre 100% peu après le démarrage, une fois que la plupart des requêtes ont été exécutées au moins une fois. Sinon, votre application fabrique des formes d'arbre nouvelles en permanence, et vous compilez pour rien. La documentation nuance : quelques requêtes avec des constantes restent parfaitement acceptables, il n'est pas nécessaire de tout paramétrer ([Microsoft Learn, Advanced Performance Topics](https://learn.microsoft.com/en-us/ef/core/performance/advanced-performance-topics)).

Il existe aussi des requêtes compilées manuelles, via `EF.CompileQuery`, qui pré-compilent la requête une fois pour toutes, à la création du délégué. Mais elles restent des délégués générés, avec leurs propres limites : un seul modèle EF, des paramètres scalaires simples ([Microsoft Learn, Advanced Performance Topics](https://learn.microsoft.com/en-us/ef/core/performance/advanced-performance-topics)).

L'architecture est donc : du LINQ, un cache, un pipeline de traduction, et de la génération de code à l'exécution pour la matérialisation.

## Pourquoi cette architecture bloque le Native AOT

Le Native AOT compile toute l'application en code machine avant le déploiement. Les gains : démarrage plus rapide, binaires autonomes plus petits, empreinte mémoire réduite, et la possibilité de tourner sur des environnements sans JIT. Les contreparties sont listées dans la documentation .NET : pas de chargement dynamique, pas de génération de code à l'exécution — `System.Reflection.Emit` est interdit — et les arbres d'expression `System.Linq.Expressions` passent toujours par leur forme interprétée, plus lente que le code compilé à la volée ([Microsoft Learn, Native AOT deployment](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot)).

EF Core utilise exactement ce que le Native AOT interdit, à deux endroits.

Premier endroit : la construction du modèle. Par réflexion, au premier usage du `DbContext`. En publication AOT, EF Core refuse carrément : "Model building is not supported when publishing with NativeAOT. Use a compiled model." — un message signalé dans un ticket de bug aujourd'hui fermé sans suite ([dotnet/efcore#32496](https://github.com/dotnet/efcore/issues/32496)). Le contournement existe : générer un modèle compilé avec `dotnet ef dbcontext optimize`, qui remplace la réflexion par du code statique ([Microsoft Learn, Advanced Performance Topics](https://learn.microsoft.com/en-us/ef/core/performance/advanced-performance-topics)).

Deuxième endroit, plus profond : le pipeline des requêtes. La sortie de la compilation d'une requête est un arbre d'expression, que EF Core compile en un délégué pour produire l'énumérable d'exécution. En AOT, ce chemin est impossible, puisque la génération de code à l'exécution n'existe pas. L'équipe EF le résume ainsi : EF Core repose en interne sur la génération de code pour matérialiser les résultats, et cette génération n'est pas supportée en Native AOT ([Microsoft Learn, NativeAOT Support and Precompiled Queries](https://learn.microsoft.com/en-us/ef/core/performance/nativeaot-and-precompiled-queries)).

Autrement dit : le problème n'est pas un bug ou une dette technique négligée. C'est le fonctionnement même de l'ORM. LINQ permet d'écrire des requêtes arbitraires, composées, dynamiques. EF Core les compile quand il les rencontre. Toute la flexibilité du produit repose sur ce pipeline.

## Ce que Microsoft construit : les requêtes précompilées

La question n'est pas nouvelle. En mai 2021, l'équipe ouvrait l'epic "Don't use reflection at runtime", qui liste tout le travail à faire : modèles compilés avec liaison statique, requêtes précompilées, lambdas de change tracking pré-générés, suppression de `MakeGenericMethod` du chemin chaud ([dotnet/efcore#24903](https://github.com/dotnet/efcore/issues/24903)). Un mois plus tard, un ticket dédié aux requêtes précompilées était ouvert ([dotnet/efcore#25009](https://github.com/dotnet/efcore/issues/25009)).

Le design initial, un source generator Roslyn, a été abandonné : l'approche s'est révélée impraticable, parce qu'elle aurait exigé de charger les assemblys de l'utilisateur et d'exécuter le compilateur EF à l'intérieur du processus du compilateur C#. La solution retenue : un outil en temps de design, hors du processus du compilateur.

Le résultat est arrivé dans EF Core 9, fin 2024, sous forme expérimentale, dans une section "AOT and pre-compiled queries" des notes de version ([Microsoft Learn, What's New in EF Core 9](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-9.0/whatsnew)). Le mécanisme s'appelle la précompilation de requêtes, et il inverse complètement la logique.

Concrètement, à la publication :

- vous activez `<PublishAot>true</PublishAot>` ;
- vous référencez le package `Microsoft.EntityFrameworkCore.Tasks` ;
- l'outillage analyse statiquement votre code source, identifie les requêtes LINQ d'EF, et génère des intercepteurs C# pour chacune.

Chaque intercepteur contient le SQL final de la requête et le code optimisé de matérialisation. Plus rien à compiler au démarrage : le SQL est prêt, le shaper est prêt. Les intercepteurs sont eux-mêmes une fonctionnalité expérimentale du compilateur C#, qui demande un opt-in dédié (`InterceptorsNamespaces`). Vous pouvez aussi générer les intercepteurs manuellement avec `dotnet ef dbcontext optimize --precompile-queries --nativeaot`, ou brancher la génération sur MSBuild au moment du build ou du publish ([Microsoft Learn, NativeAOT Support and Precompiled Queries](https://learn.microsoft.com/en-us/ef/core/performance/nativeaot-and-precompiled-queries), [Microsoft Learn, EF Core MSBuild integration](https://learn.microsoft.com/en-us/ef/core/cli/msbuild)).

La documentation est honnête sur le mode d'emploi : n'importe quel changement dans le source invalide les intercepteurs et impose de régénérer. Ce travail n'a pas sa place dans la boucle de développement quotidienne ; il se branche sur un pipeline CI/CD, au moment du publish ([Microsoft Learn, NativeAOT Support and Precompiled Queries](https://learn.microsoft.com/en-us/ef/core/performance/nativeaot-and-precompiled-queries)).

Un an plus tard, EF Core 10 est sorti (novembre 2025, LTS jusqu'au 10 novembre 2028). Ses notes de version parlent de vector search pour SQL Server, de types JSON, de complex types — mais ne contiennent aucune section Native AOT ([Microsoft Learn, What's New in EF Core 10](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-10.0/whatsnew)). La page dédiée reste sous le même bandeau : fonctionnalité hautement expérimentale, impropre à la production, infrastructure vers la version finale promise "dans une future version" ([Microsoft Learn, NativeAOT Support and Precompiled Queries](https://learn.microsoft.com/en-us/ef/core/performance/nativeaot-and-precompiled-queries)). EF Core 11, en préparation, ajuste déjà l'intégration MSBuild : la propriété `EFOptimizeContext` disparaît ([Microsoft Learn, EF Core MSBuild integration](https://learn.microsoft.com/en-us/ef/core/cli/msbuild)).

## Ce qui marche bien aujourd'hui, et pour qui

Dans son périmètre actuel, le dispositif fonctionne et apporte de vrais gains : démarrage nettement plus rapide (plus la base de requêtes est grande, plus le gain est visible, puisque la compilation LINQ disparaît du démarrage), binaires autonomes, mémoire réduite, exécution sans JIT ([Microsoft Learn, NativeAOT Support and Precompiled Queries](https://learn.microsoft.com/en-us/ef/core/performance/nativeaot-and-precompiled-queries)).

Le scénario cible, aujourd'hui : une application dont toutes les requêtes sont écrites en LINQ statique, dans le code source, sans composition dynamique ; une équipe prête à régénérer intercepteurs et modèle compilé à chaque déploiement, dans la CI ; et un provider EF compatible avec la précompilation.

Deux usages partiels méritent aussi d'être connus :

- La précompilation sans AOT. `dotnet ef dbcontext optimize --precompile-queries` génère le modèle compilé et les intercepteurs pour les requêtes précompilables, sans publier en AOT. Vous gardez les requêtes dynamiques et les autres fonctionnalités, et vous supprimez quand même le coût de compilation au démarrage ([Microsoft Learn, NativeAOT Support and Precompiled Queries](https://learn.microsoft.com/en-us/ef/core/performance/nativeaot-and-precompiled-queries)).
- Npgsql sans EF Core. Le driver ADO.NET Npgsql est, lui, pleinement compatible Native AOT et trimming depuis sa version 8.0. La plupart des fonctionnalités marchent telles quelles ; celles qui sont incompatibles exigent un opt-in explicite et produisent un warning ([Npgsql, Compatibility Notes](https://www.npgsql.org/doc/compatibility.html)). Si votre besoin est un accès PostgreSQL direct, sans ORM, le AOT est déjà une option réaliste — c'est d'ailleurs documenté côté Npgsql : EF Core n'est pas encore compatible Native AOT, et en Native AOT, Npgsql ne peut être utilisé qu'en se passant d'EF Core ([Npgsql EF 8.0 Release Notes](https://www.npgsql.org/efcore/release-notes/8.0.html)).

## Pourquoi ça ne couvre pas encore tout

La limitation centrale tient en une phrase de la documentation : les requêtes dynamiques ne peuvent pas être analysées statiquement ([Microsoft Learn, NativeAOT Support and Precompiled Queries](https://learn.microsoft.com/en-us/ef/core/performance/nativeaot-and-precompiled-queries)). Or la composition dynamique est le cœur de LINQ. Un `IQueryable` construit par étapes, un `Where` ajouté sous condition, un tri choisi à l'exécution : tout cela est du code qu'aucun analyseur statique ne peut prédire. La documentation propose une parade : réécrire chaque variante comme une requête complète et statique. Cela marche pour deux branches. Pour un moteur de filtres arbitraires côté client, cela n'a aucun sens.

La suite annoncée est révélatrice : les requêtes dynamiques seront probablement supportées un jour en Native AOT, mais sans précompilation. Elles ralentiront le démarrage et s'exécuteront moins efficacement qu'en mode normal, parce que EF Core repose en interne sur la génération de code pour la matérialisation, et que cette génération n'existe pas en AOT ([Microsoft Learn, NativeAOT Support and Precompiled Queries](https://learn.microsoft.com/en-us/ef/core/performance/nativeaot-and-precompiled-queries)).

La liste des restrictions connues complète le tableau :

- la syntaxe de requête LINQ (compréhensions, `from x in ...`) n'est pas supportée ;
- les convertisseurs de valeurs avec état capturé ne sont pas supportés ;
- les providers EF peuvent avoir à ajouter leur propre support de la précompilation ; vérifiez la documentation du vôtre ;
- le modèle compilé et les intercepteurs générés peuvent être volumineux et longs à produire ;
- la publication produit encore un grand nombre de warnings trimming et AOT, sans garantie que l'application tourne correctement ([Microsoft Learn, NativeAOT Support and Precompiled Queries](https://learn.microsoft.com/en-us/ef/core/performance/nativeaot-and-precompiled-queries)).

C'est le dernier point qui décide : tant que la publication d'une application EF en AOT génère des warnings sans garantie d'exécution, ce n'est pas une option de production. Le dispositif actuel est une infrastructure, pas une fonctionnalité. L'ultime version, promise sans date, supprimera les warnings et supportera l'ensemble du produit.

## EF Core et PostgreSQL

Parlons de PostgreSQL, parce que c'est là que tout cela devient concret pour le projet que je développe.

Le provider EF Core officiel pour PostgreSQL est Npgsql.EntityFrameworkCore.PostgreSQL. Il se compose du driver ADO.NET Npgsql et d'une couche relationnelle qui traduit les abstractions d'EF Core en SQL PostgreSQL ([npgsql/efcore.pg sur GitHub](https://github.com/npgsql/efcore.pg)). Le projet vise la compatibilité avec toutes les versions de PostgreSQL encore supportées, soit environ cinq ans en arrière ([Npgsql, Compatibility Notes](https://www.npgsql.org/doc/compatibility.html)), et le provider suppose par défaut au moins PostgreSQL 14 ([Npgsql EF 8.0 Release Notes](https://www.npgsql.org/efcore/release-notes/8.0.html)).

Côté AOT, la situation est asymétrique, et c'est important de la comprendre : le driver est AOT-ready, la couche ORM ne l'est pas. En version 10, le driver ajuste même ses règles pour le AOT : les applications Native AOT qui utilisent `NpgsqlSlimDataSourceBuilder` avec des types JSON mutables doivent désormais appeler `EnableJsonTypes()` explicitement ([Npgsql 10.0 Release Notes](https://www.npgsql.org/doc/release-notes/10.0.html)). Le driver est donc suivi de près pour le AOT. Le provider EF, lui, hérite du blocage d'EF Core.

Côté fonctionnalités, la version 10 du provider illustre ce que PostgreSQL apporte de spécifique à EF Core :

- les complex types JSON d'EF Core 10 sont mappés en colonnes `jsonb`, avec support des mises à jour partielles via `jsonb_set` dans `ExecuteUpdate` ;
- les collections scalaires sont mappées en tableaux PostgreSQL natifs (`text[]`), là où les autres bases relationnelles passent par des colonnes JSON — et le SQL généré pour les collections imbriquées dans du JSON s'est nettement amélioré (opérateur de contenu `@>`, exploitable par index) ;
- support des colonnes générées virtuelles de PostgreSQL 18 ;
- support des UUIDv7, y compris la traduction de `Guid.CreateVersion7()` vers la fonction `uuidv7()` native de PostgreSQL 18, si le provider est configuré pour cibler cette version (`SetPostgresVersion(18, 0)`) ;
- support du type `cube` ;
- et un breaking change utile à connaître : la traduction de `Contains` sur les tableaux passe par défaut à `= ANY(...)`, ce qui peut être plus lent qu'un index GIN ; si vous avez un index GIN, il faut le modéliser pour revenir à l'opérateur `@>` ([Npgsql EF 10.0 Release Notes](https://www.npgsql.org/efcore/release-notes/10.0.html)).

Reste le cas des vecteurs, central pour tout projet d'IA. PostgreSQL n'a pas de type vector natif ; il a une extension, pgvector. Côté .NET, le dépôt pgvector-dotnet fournit les packages `Pgvector` et `Pgvector.EntityFrameworkCore` : on enregistre le plugin avec `UseVector()` sur le `NpgsqlDataSourceBuilder`, et on mappe la propriété avec un type de colonne `vector(N)` ; pour que les migrations EF créent l'extension côté base, déclarez-la avec `HasPostgresExtension("vector")` ([roji/pgvector-dotnet](https://github.com/roji/pgvector-dotnet)). Dans mon projet, c'est une colonne `vector(1024)` avec un index HNSW.

La comparaison avec SQL Server est parlante : EF Core 10 apporte un support natif du type `vector` et de la fonction `VECTOR_DISTANCE()` pour Azure SQL et SQL Server 2025 ([Microsoft Learn, What's New in EF Core 10](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-10.0/whatsnew)). Côté PostgreSQL, le même besoin passe par une extension communauté + un package tiers. Ce n'est ni mieux ni moins bien : c'est le modèle PostgreSQL, où les types riches vivent dans des extensions plutôt que dans le noyau.

## Conclusion

Le Native AOT attire, et pour de bonnes raisons : démarrage, taille, mémoire. Mais EF Core est construit sur exactement ce que le Native AOT interdit : de la réflexion pour le modèle, de la génération de code à l'exécution pour les requêtes. Microsoft a un plan, et le plan est sérieux : modèles compilés, requêtes précompilées, intercepteurs générés à la publication. EF Core 9 a posé l'infrastructure. EF Core 10, LTS, ne change rien au statut expérimental. La version finale n'a pas de date.

Pour mon projet, le calcul est simple. Le service est résident : il démarre rarement, et son temps de démarrage n'est pas un argument commercial. Les requêtes de recherche mémoire sont dynamiques par nature, donc hors du périmètre actuel de la précompilation. Et le couple EF Core + Npgsql + pgvector fonctionne très bien en JIT. ReadyToRun suffit largement, et la question sera rouverte quand la page "NativeAOT Support" de la documentation EF perdra son bandeau expérimental.

## Sources

- [Native AOT deployment — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot)
- [Advanced Performance Topics — Microsoft Learn](https://learn.microsoft.com/en-us/ef/core/performance/advanced-performance-topics)
- [NativeAOT Support and Precompiled Queries (Experimental) — Microsoft Learn](https://learn.microsoft.com/en-us/ef/core/performance/nativeaot-and-precompiled-queries)
- [What's New in EF Core 9 — Microsoft Learn](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-9.0/whatsnew)
- [What's New in EF Core 10 — Microsoft Learn](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-10.0/whatsnew)
- [EF Core MSBuild integration — Microsoft Learn](https://learn.microsoft.com/en-us/ef/core/cli/msbuild)
- [Issue #24903 : Don't use reflection at runtime — dotnet/efcore, mai 2021](https://github.com/dotnet/efcore/issues/24903)
- [Issue #25009 : Precompiled queries — dotnet/efcore, juin 2021](https://github.com/dotnet/efcore/issues/25009)
- [Issue #32496 : EF Core ignores compiled model with PublishAot — dotnet/efcore](https://github.com/dotnet/efcore/issues/32496)
- [Npgsql Compatibility Notes — npgsql.org](https://www.npgsql.org/doc/compatibility.html)
- [Npgsql 10.0 Release Notes — npgsql.org](https://www.npgsql.org/doc/release-notes/10.0.html)
- [Npgsql EF Provider 8.0 Release Notes — npgsql.org](https://www.npgsql.org/efcore/release-notes/8.0.html)
- [Npgsql EF Provider 10.0 Release Notes — npgsql.org](https://www.npgsql.org/efcore/release-notes/10.0.html)
- [npgsql/efcore.pg — GitHub](https://github.com/npgsql/efcore.pg)
- [roji/pgvector-dotnet — GitHub](https://github.com/roji/pgvector-dotnet)

---
title: "EF Core: on-the-fly query compilation, and why Native AOT remains out of reach"
description: "How EF Core compiles your LINQ queries at runtime, why this architecture blocks Native AOT, what Microsoft is building to fix it, and the state of the PostgreSQL integration."
date: 2026-10-30T09:23:00Z
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [dotnet, efcore, postgresql, aot, performance]
---

While developing a small .NET MCP server, I had to make an architecture decision: publish the application as Native AOT or not. The short answer: no. EF Core, the ORM the project uses, does not allow it reliably today. This article explains why, in detail. We will look at how EF Core actually executes a LINQ query, why this mechanism is incompatible with ahead-of-time compilation, what Microsoft is building to unblock the situation, and where the PostgreSQL integration stands.

## What happens when a LINQ query executes

An EF Core query is not SQL. It is an expression tree, produced by the C# compiler from your LINQ. When you call `ToListAsync()`, EF Core must transform that tree into SQL, then transform the returned rows into .NET objects. The documentation calls this step query "compilation" ([Microsoft Learn, Advanced Performance Topics](https://learn.microsoft.com/en-us/ef/core/performance/advanced-performance-topics)).

This compilation does several things. It extracts the parameters, builds the intermediate representation of the query, generates the SQL, and above all produces what EF Core calls a shaper: the code that materializes each database row into a .NET object. That shaper is code generated at runtime. The tracking ticket is explicit: this process is costly, increases startup time, and relies on runtime code generation ([dotnet/efcore#25009](https://github.com/dotnet/efcore/issues/25009)).

This cost is paid only once. EF Core caches the compilation result by tree shape. Two queries with the same structure, even with different values, reuse the same compiled output. The documentation gives the classic example: if you write `p.Title == "post1"` then `p.Title == "post2"` with constants, each query is compiled separately and produces different SQL. If you parameterize with a variable, both queries have the same shape, and a single compilation is enough. EF Core also exposes a metric, the Query Cache Hit Rate, which should reach 100% soon after startup, once most queries have executed at least once. If not, your application keeps producing new tree shapes, and you are compiling for nothing. The documentation tempers this: a few queries with constants are perfectly fine, and there is no need to parameterize everything ([Microsoft Learn, Advanced Performance Topics](https://learn.microsoft.com/en-us/ef/core/performance/advanced-performance-topics)).

There are also manually compiled queries, via `EF.CompileQuery`, which pre-compile the query once and for all, when the delegate is created. But they remain generated delegates, with their own limitations: a single EF model, simple scalar parameters ([Microsoft Learn, Advanced Performance Topics](https://learn.microsoft.com/en-us/ef/core/performance/advanced-performance-topics)).

So the architecture is: LINQ, a cache, a translation pipeline, and runtime code generation for materialization.

## Why this architecture blocks Native AOT

Native AOT compiles the entire application to machine code before deployment. The gains: faster startup, smaller self-contained binaries, a reduced memory footprint, and the ability to run in environments without a JIT. The trade-offs are listed in the .NET documentation: no dynamic loading, no runtime code generation — `System.Reflection.Emit` is forbidden — and `System.Linq.Expressions` expression trees always go through their interpreted form, which is slower than code compiled on the fly ([Microsoft Learn, Native AOT deployment](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot)).

EF Core uses exactly what Native AOT forbids, in two places.

First place: model building. By reflection, on first use of the `DbContext`. Under AOT publishing, EF Core flatly refuses: "Model building is not supported when publishing with NativeAOT. Use a compiled model." — a message reported in a bug ticket since closed as not planned ([dotnet/efcore#32496](https://github.com/dotnet/efcore/issues/32496)). The workaround exists: generate a compiled model with `dotnet ef dbcontext optimize`, which replaces reflection with static code ([Microsoft Learn, Advanced Performance Topics](https://learn.microsoft.com/en-us/ef/core/performance/advanced-performance-topics)).

Second place, deeper: the query pipeline. The output of query compilation is an expression tree, which EF Core compiles into a delegate to produce the executable enumerable. Under AOT, this path is impossible, since runtime code generation does not exist. The EF team summarizes it this way: EF Core internally relies on code generation to materialize results, and this generation is not supported under Native AOT ([Microsoft Learn, NativeAOT Support and Precompiled Queries](https://learn.microsoft.com/en-us/ef/core/performance/nativeaot-and-precompiled-queries)).

In other words: the problem is not a bug or neglected technical debt. It is how the ORM works. LINQ lets you write arbitrary, composed, dynamic queries. EF Core compiles them when it meets them. The product's whole flexibility rests on this pipeline.

## What Microsoft is building: precompiled queries

The question is not new. In May 2021, the team opened the epic "Don't use reflection at runtime", which lists all the work to be done: compiled models with static binding, precompiled queries, pre-generated change-tracking lambdas, removing `MakeGenericMethod` from the hot path ([dotnet/efcore#24903](https://github.com/dotnet/efcore/issues/24903)). A month later, a dedicated ticket for precompiled queries was opened ([dotnet/efcore#25009](https://github.com/dotnet/efcore/issues/25009)).

The initial design, a Roslyn source generator, was abandoned: the approach proved impractical, because it would have required loading the user's assemblies and running EF's compiler inside the C# compiler's process. The chosen solution: a design-time tool, outside the compiler process.

The result arrived in EF Core 9, at the end of 2024, in experimental form, in an "AOT and pre-compiled queries" section of the release notes ([Microsoft Learn, What's New in EF Core 9](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-9.0/whatsnew)). The mechanism is called query precompilation, and it completely reverses the logic.

Concretely, at publish time:

- you enable `<PublishAot>true</PublishAot>`;
- you reference the `Microsoft.EntityFrameworkCore.Tasks` package;
- the tooling statically analyzes your source code, identifies the EF LINQ queries, and generates C# interceptors for each one.

Each interceptor contains the query's final SQL and the optimized materialization code. Nothing left to compile at startup: the SQL is ready, the shaper is ready. The interceptors are themselves an experimental C# compiler feature, which requires a dedicated opt-in (`InterceptorsNamespaces`). You can also generate the interceptors manually with `dotnet ef dbcontext optimize --precompile-queries --nativeaot`, or hook generation into MSBuild at build or publish time ([Microsoft Learn, NativeAOT Support and Precompiled Queries](https://learn.microsoft.com/en-us/ef/core/performance/nativeaot-and-precompiled-queries), [Microsoft Learn, EF Core MSBuild integration](https://learn.microsoft.com/en-us/ef/core/cli/msbuild)).

The documentation is honest about the workflow: any change in the source invalidates the interceptors and forces regeneration. This work does not belong in the daily development loop; it hooks into a CI/CD pipeline, at publish time ([Microsoft Learn, NativeAOT Support and Precompiled Queries](https://learn.microsoft.com/en-us/ef/core/performance/nativeaot-and-precompiled-queries)).

A year later, EF Core 10 shipped (November 2025, LTS until November 10, 2028). Its release notes talk about vector search for SQL Server, JSON types, complex types — but contain no Native AOT section ([Microsoft Learn, What's New in EF Core 10](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-10.0/whatsnew)). The dedicated page still carries the same banner: highly experimental feature, unfit for production, infrastructure towards the final version promised "in a future version" ([Microsoft Learn, NativeAOT Support and Precompiled Queries](https://learn.microsoft.com/en-us/ef/core/performance/nativeaot-and-precompiled-queries)). EF Core 11, in preparation, is already adjusting the MSBuild integration: the `EFOptimizeContext` property disappears ([Microsoft Learn, EF Core MSBuild integration](https://learn.microsoft.com/en-us/ef/core/cli/msbuild)).

## What works well today, and for whom

Within its current scope, the mechanism works and brings real gains: significantly faster startup (the bigger the query base, the more visible the gain, since LINQ compilation disappears from startup), self-contained binaries, reduced memory, execution without a JIT ([Microsoft Learn, NativeAOT Support and Precompiled Queries](https://learn.microsoft.com/en-us/ef/core/performance/nativeaot-and-precompiled-queries)).

The target scenario today: an application whose queries are all written as static LINQ, in source code, with no dynamic composition; a team ready to regenerate interceptors and the compiled model on every deployment, in CI; and an EF provider compatible with precompilation.

Two partial usages also deserve to be known:

- Precompilation without AOT. `dotnet ef dbcontext optimize --precompile-queries` generates the compiled model and interceptors for precompilable queries, without publishing as AOT. You keep dynamic queries and the other features, and you still remove the compilation cost from startup ([Microsoft Learn, NativeAOT Support and Precompiled Queries](https://learn.microsoft.com/en-us/ef/core/performance/nativeaot-and-precompiled-queries)).
- Npgsql without EF Core. The Npgsql ADO.NET driver is fully compatible with Native AOT and trimming since version 8.0. Most features work as-is; incompatible ones require an explicit opt-in and produce a warning ([Npgsql, Compatibility Notes](https://www.npgsql.org/doc/compatibility.html)). If your need is direct PostgreSQL access, without an ORM, AOT is already a realistic option — Npgsql's own notes say it: EF Core is not yet Native AOT compatible, and in Native AOT, Npgsql can only be used without EF Core ([Npgsql EF 8.0 Release Notes](https://www.npgsql.org/efcore/release-notes/8.0.html)).

## Why it does not cover everything yet

The central limitation is one sentence in the documentation: dynamic queries cannot be statically analyzed ([Microsoft Learn, NativeAOT Support and Precompiled Queries](https://learn.microsoft.com/en-us/ef/core/performance/nativeaot-and-precompiled-queries)). Yet dynamic composition is the heart of LINQ. An `IQueryable` built step by step, a `Where` added under a condition, a sort chosen at runtime: none of this is code a static analyzer can predict. The documentation offers a workaround: rewrite each variant as a complete, static query. That works for two branches. For an arbitrary client-side filter engine, it makes no sense.

What is announced next is revealing: dynamic queries will likely be supported one day under Native AOT, but without precompilation. They will slow down startup and run less efficiently than in normal mode, because EF Core internally relies on code generation for materialization, and that generation does not exist under AOT ([Microsoft Learn, NativeAOT Support and Precompiled Queries](https://learn.microsoft.com/en-us/ef/core/performance/nativeaot-and-precompiled-queries)).

The list of known restrictions completes the picture:

- LINQ query syntax (comprehensions, `from x in ...`) is not supported;
- value converters with captured state are not supported;
- EF providers may need to add their own precompilation support; check your provider's documentation;
- the generated compiled model and interceptors can be large and slow to produce;
- publishing still produces a large number of trimming and AOT warnings, with no guarantee that the application will run correctly ([Microsoft Learn, NativeAOT Support and Precompiled Queries](https://learn.microsoft.com/en-us/ef/core/performance/nativeaot-and-precompiled-queries)).

The last point is what decides: as long as publishing an EF application as AOT generates warnings with no guarantee of execution, this is not a production option. The current mechanism is infrastructure, not a feature. The final version, promised with no date, will remove the warnings and support the whole product.

## EF Core and PostgreSQL

Let's talk about PostgreSQL, because that is where all of this becomes concrete for the project I am developing.

The official EF Core provider for PostgreSQL is Npgsql.EntityFrameworkCore.PostgreSQL. It is composed of the Npgsql ADO.NET driver and a relational layer that translates EF Core's abstractions into PostgreSQL SQL ([npgsql/efcore.pg on GitHub](https://github.com/npgsql/efcore.pg)). The project targets compatibility with all still-supported PostgreSQL versions, roughly five years back ([Npgsql, Compatibility Notes](https://www.npgsql.org/doc/compatibility.html)), and the provider assumes at least PostgreSQL 14 by default ([Npgsql EF 8.0 Release Notes](https://www.npgsql.org/efcore/release-notes/8.0.html)).

On the AOT side, the situation is asymmetrical, and it is important to understand it: the driver is AOT-ready, the ORM layer is not. In version 10, the driver even adjusted its rules for AOT: Native AOT applications that use `NpgsqlSlimDataSourceBuilder` with mutable JSON types must now call `EnableJsonTypes()` explicitly ([Npgsql 10.0 Release Notes](https://www.npgsql.org/doc/release-notes/10.0.html)). The driver is therefore closely tracked for AOT. The EF provider inherits EF Core's blockage.

On the feature side, version 10 of the provider illustrates what PostgreSQL brings specifically to EF Core:

- EF Core 10's JSON complex types are mapped to `jsonb` columns, with support for partial updates via `jsonb_set` in `ExecuteUpdate`;
- scalar collections are mapped to native PostgreSQL arrays (`text[]`), where other relational databases go through JSON columns — and the SQL generated for nested collections inside JSON has improved markedly (the `@>` containment operator, which can benefit from an index);
- support for PostgreSQL 18 virtual generated columns;
- support for UUIDv7, including translating `Guid.CreateVersion7()` to PostgreSQL 18's native `uuidv7()` function, provided the provider is configured to target that version (`SetPostgresVersion(18, 0)`);
- support for the `cube` type;
- and a useful breaking change to know: the translation of `Contains` on arrays now defaults to `= ANY(...)`, which can be slower than a GIN index; if you have a GIN index, model it to fall back to the `@>` operator ([Npgsql EF 10.0 Release Notes](https://www.npgsql.org/efcore/release-notes/10.0.html)).

That leaves vectors, central to any AI project. PostgreSQL has no native vector type; it has an extension, pgvector. On the .NET side, the pgvector-dotnet repository provides the `Pgvector` and `Pgvector.EntityFrameworkCore` packages: you register the plugin with `UseVector()` on the `NpgsqlDataSourceBuilder`, and map the property with a `vector(N)` column type; if you want EF migrations to create the extension in the database, declare it with `HasPostgresExtension("vector")` ([roji/pgvector-dotnet](https://github.com/roji/pgvector-dotnet)). In my project, that is a `vector(1024)` column with an HNSW index.

The comparison with SQL Server is telling: EF Core 10 brings native support for the `vector` type and the `VECTOR_DISTANCE()` function for Azure SQL and SQL Server 2025 ([Microsoft Learn, What's New in EF Core 10](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-10.0/whatsnew)). On the PostgreSQL side, the same need goes through a community extension plus a third-party package. It is neither better nor worse: it is the PostgreSQL model, where rich types live in extensions rather than in the core.

## Conclusion

Native AOT is attractive, and for good reasons: startup, size, memory. But EF Core is built on exactly what Native AOT forbids: reflection for the model, runtime code generation for queries. Microsoft has a plan, and the plan is serious: compiled models, precompiled queries, interceptors generated at publish time. EF Core 9 laid the infrastructure. EF Core 10, LTS, changes nothing about the experimental status. The final version has no date.

For my project, the calculation is simple. The service is resident: it rarely restarts, and its startup time is not a commercial argument. The memory search queries are dynamic by nature, hence outside the current scope of precompilation. And the EF Core + Npgsql + pgvector trio works very well under JIT. ReadyToRun is more than enough, and the question will be reopened when the "NativeAOT Support" page of the EF documentation loses its experimental banner.

## Sources

- [Native AOT deployment — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot)
- [Advanced Performance Topics — Microsoft Learn](https://learn.microsoft.com/en-us/ef/core/performance/advanced-performance-topics)
- [NativeAOT Support and Precompiled Queries (Experimental) — Microsoft Learn](https://learn.microsoft.com/en-us/ef/core/performance/nativeaot-and-precompiled-queries)
- [What's New in EF Core 9 — Microsoft Learn](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-9.0/whatsnew)
- [What's New in EF Core 10 — Microsoft Learn](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-10.0/whatsnew)
- [EF Core MSBuild integration — Microsoft Learn](https://learn.microsoft.com/en-us/ef/core/cli/msbuild)
- [Issue #24903: Don't use reflection at runtime — dotnet/efcore, May 2021](https://github.com/dotnet/efcore/issues/24903)
- [Issue #25009: Precompiled queries — dotnet/efcore, June 2021](https://github.com/dotnet/efcore/issues/25009)
- [Issue #32496: EF Core ignores compiled model with PublishAot — dotnet/efcore](https://github.com/dotnet/efcore/issues/32496)
- [Npgsql Compatibility Notes — npgsql.org](https://www.npgsql.org/doc/compatibility.html)
- [Npgsql 10.0 Release Notes — npgsql.org](https://www.npgsql.org/doc/release-notes/10.0.html)
- [Npgsql EF Provider 8.0 Release Notes — npgsql.org](https://www.npgsql.org/efcore/release-notes/8.0.html)
- [Npgsql EF Provider 10.0 Release Notes — npgsql.org](https://www.npgsql.org/efcore/release-notes/10.0.html)
- [npgsql/efcore.pg — GitHub](https://github.com/npgsql/efcore.pg)
- [roji/pgvector-dotnet — GitHub](https://github.com/roji/pgvector-dotnet)

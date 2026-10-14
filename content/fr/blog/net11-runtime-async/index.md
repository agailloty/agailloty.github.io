---
title: ".NET 11 a tué la machine à états"
description: "Depuis C# 5, async/await reposait entièrement sur une réécriture du compilateur. .NET 11 change la donne : le runtime exécute désormais l'asynchronie nativement."
date: 2026-10-14T08:00:00Z
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [".NET", "C#", "async", "runtime", "IL"]
---

Quatorze ans d'async/await reposaient sur une illusion. Quand vous écriviez une méthode `async`, elle n'a jamais existé comme méthode : le compilateur C# la découpait en une machine à états, une struct générée dont le rôle était de simuler une coroutine que le runtime ne savait pas exécuter. Avec .NET 11, cette époque s'achève : le runtime lui-même comprend désormais la suspension et la reprise des méthodes et, une fois la feature activée, la machine à états disparaît.

Microsoft l'a présenté comme l'une des avancées majeures de .NET 11 : la feature, appelée runtime async, a été annoncée dès la Preview 1 et détaillée sur le blog .NET. Les bibliothèques du runtime .NET sont elles-mêmes désormais compilées avec `runtime-async=on` et ne contiennent plus aucune machine à états générée par le compilateur. Autant dire que l'équipe .NET y croit.

J'ai voulu voir le changement de mes propres yeux, et vous pouvez le faire aussi. Prenez le programme le plus simple possible, compilez-le avec le SDK .NET 10, puis avec le SDK .NET 11 et la feature `runtime-async` activée, et décompilez les deux DLL avec ildasm. La différence est spectaculaire.

## Le programme de test

Voici notre cobaye. Deux méthodes async, un seul `await` dans chacune. Rien de plus simple :

```csharp
public class Program
{
    public static async Task Main()
    {
        await DoTaskAsynchronously();
    }

    private static async Task DoTaskAsynchronously()
    {
        await Task.Delay(100);
    }
}
```

Sous .NET 10, le csproj est minimal : `net10.0`. Sous .NET 11, on active explicitement la nouvelle feature :

```xml
<TargetFramework>net11.0</TargetFramework>
<Features>runtime-async=on</Features>
```

Puis :

```bash
ildasm /tok /byt async-dotnet.dll /out=dump.il
```

## .NET 10 : la machine à états

Une machine à états, c'est un objet qui mémorise où en est un travail pour pouvoir le poursuivre plus tard. C'est exactement ce que le compilateur génère pour chaque méthode `async` depuis C# 5 (2012) : le CLR de l'époque ne savait pas suspendre puis reprendre une méthode, alors Roslyn transforme votre code en une struct générée, `<Main>d__0`, qui implémente l'interface `IAsyncStateMachine`. Votre méthode n'est plus qu'un stub de lancement de 49 octets : elle instancie la struct, crée un `AsyncTaskMethodBuilder` (le pilote qui produira le `Task` retourné), puis démarre la machine.

Le vrai corps de votre méthode vit dans `MoveNext()`. Concrètement, l'exécution suit quatre étapes :

1. Lancement. Le stub `Main` instancie `<Main>d__0`, initialise son état (`<>1__state = -1`) et appelle `builder.Start()`.
2. Première exécution. `MoveNext` déroule votre code jusqu'au premier `await`. Si la tâche est déjà terminée, l'exécution continue sans interruption.
3. Suspension. Sinon, la machine mémorise son point de reprise (`state = 0`), stocke l'awaiter dans un champ, s'enregistre auprès de la tâche via `AwaitUnsafeOnCompleted`, puis retourne. La méthode "sort" physiquement du call stack.
4. Reprise. Quand la tâche se termine, le runtime rappelle `MoveNext`. Le champ `state` indique de reprendre après l'`await` : l'awaiter valide la fin de la tâche, et `SetResult` conclut le `Task`.

Le pseudo-code C# suivant illustre approximativement ce que le compilateur génère pour aider le runtime. C'est une simplification, mais chaque élément — le champ `state`, l'awaiter mémorisé, le `try/catch` qui route les exceptions vers `SetException` — existe réellement dans l'IL :

```csharp
// Pseudo-code : approximation de ce que Roslyn génère pour aider le runtime
struct <Main>d__0 : IAsyncStateMachine   // struct générée par le compilateur
{
    public int <>1__state;               // où reprendre (-1, 0, -2)
    public AsyncTaskMethodBuilder <>t__builder;
    private TaskAwaiter <>u__1;          // l'awaiter mémorisé entre deux suspensions

    void MoveNext()
    {
        int num = <>1__state;
        try
        {
            TaskAwaiter awaiter;
            if (num != 0)                 // première exécution
            {
                awaiter = DoTaskAsynchronously().GetAwaiter();
                if (!awaiter.IsCompleted)
                {
                    <>1__state = 0;       // mémorise le point de reprise
                    <>u__1 = awaiter;
                    <>t__builder.AwaitUnsafeOnCompleted(ref awaiter, ref this);
                    return;               // suspension : la méthode "sort" ici
                }
            }
            else                          // reprise
            {
                awaiter = <>u__1;
                <>u__1 = default;
                <>1__state = -1;
            }
            awaiter.GetResult();
        }
        catch (Exception ex)
        {
            <>1__state = -2;
            <>t__builder.SetException(ex);
            return;
        }
        <>1__state = -2;
        <>t__builder.SetResult();
    }
}
```

Pour nos deux petites méthodes d'une ligne, le compilateur a produit deux structs, deux `MoveNext` (156 et 154 octets d'IL), deux `SetStateMachine`, six champs, deux blocs `try/catch` et deux stubs de lancement de 49 octets. Pour `Main` seule, l'IL généré totalise 203 octets : 49 pour le stub, 154 pour le `MoveNext` qui contient le vrai corps. Chaque variable qui survit à un `await` devient un champ. Chaque suspension devient un état. C'est le prix historique d'`async`.

Cette conception a remarquablement bien vieilli : elle a fait fonctionner async/await partout, sur des runtimes qui n'en connaissaient rien. Mais elle duplique toute cette logique dans chaque méthode async compilée, et le runtime reste, au fond, spectateur.

## .NET 11 : le CLR devient conscient de l'asynchronie

Ouvrons maintenant le dump .NET 11. Même programme, même compilateur, feature activée. Le contraste : les structs `<Main>d__0` et `<DoTaskAsynchronously>d__1` n'existent plus. Pas d'`IAsyncStateMachine`, pas de builder, pas de champ `<>1__state`. Même la display class `<>c` a disparu : sans lambda, il n'y a plus rien à générer du tout. Il ne reste que `Main`, `DoTaskAsynchronously` et les constructeurs.

Sur la même base que précédemment, `Main` suit désormais trois étapes :

1. Appel direct. `Main` est une vraie méthode de 13 octets d'IL linéaire. Elle appelle `DoTaskAsynchronously()` sans instancier de machine.
2. Suspension exprimée. À la place de la machine à états, un simple appel : `AsyncHelpers.Await(task)`. La méthode ne "sort" plus d'elle-même : elle exprime sa suspension, et c'est le CLR qui sauvegarde l'état de la pile, les variables locales concernées, et qui rendra la main au bon endroit.
3. Reprise native. Le runtime restaure la méthode et poursuit l'exécution. Pas de `try/catch` généré : la propagation des exceptions à travers les points de suspension est gérée par le CLR. Quand la tâche attendue a un résultat, la surcharge générique `Await<T>` le retourne directement.

En pseudo-code, toujours en approximation :

```csharp
// Pseudo-code : ce que Main devient sous .NET 11 — plus de struct générée
static Task Main()
{
    Task task = DoTaskAsynchronously();
    AsyncHelpers.Await(task);   // suspension exprimée, exécution déléguée au CLR
}
```

La comparaison est frappante sur les mêmes critères : 203 octets d'IL générés pour `Main` en .NET 10 (stub de 49 octets plus `MoveNext` de 154 octets) contre 13 octets pour une vraie méthode en .NET 11. Deux structs générées contre zéro. Deux `try/catch` générés contre zéro. Jusqu'au point d'entrée du processus : en .NET 10, le stub `<Main>()` bloquait sur `GetAwaiter().GetResult()` ; en .NET 11, il délègue à `AsyncHelpers.HandleAsyncEntryPoint`. Partout, l'async est repassé du compilateur au runtime.

## Ce que ça change concrètement

Pour le même code source, l'assemblage .NET 11 contient moins de types, moins de méthodes, moins de champs et moins de métadonnées. Le JIT a moins de travail. À l'exécution, le chemin est plus direct : plus de struct machine à états à manipuler, plus de builder.

Le bénéfice le plus intéressant concerne les chaînes d'appels async. Quand une méthode runtime-async en attend une autre, le runtime peut optimiser le relais entre les deux et, dans certains cas, éviter complètement l'allocation de `Task`. Chaque couche intermédiaire d'une chaîne async était hier une machine à états qui composait avec la suivante ; c'est désormais une seule exécution pilotée par le CLR.

Deux précisions honnêtes pour finir. D'abord, la sémantique observable de votre code ne change pas : c'est une évolution de la stratégie d'exécution, pas du langage. Ensuite, la feature est opt-in dans votre csproj via `<Features>runtime-async=on</Features>` — mais répétons-le, les bibliothèques du runtime .NET l'utilisent déjà par défaut. Le jour où elle devient le comportement par défaut du compilateur, vous saurez déjà ce qui se passe sous le capot.

## Conclusion

La machine à états a été l'une des astuces les plus élégantes de l'histoire de C# : faire croire à des coroutines dans un runtime qui n'en voulait pas. .NET 11 la rend obsolète en déplaçant l'asynchronie là où elle aurait toujours dû vivre : dans le runtime. Le compilateur se contente désormais d'exprimer la suspension, `AsyncHelpers.Await`, et le CLR fait le reste.

Les dumps IL complets des deux versions sont en annexe. Ouvrez-les, comparez, et surtout refaites l'expérience chez vous : c'est en décompilant soi-même qu'on mesure vraiment l'ampleur du changement.

## Bibliographie

- [Microsoft, .NET Blog — Performance Improvements in .NET 11 (section runtime async)](https://devblogs.microsoft.com/dotnet/performance-improvements-in-net-11/)
- [Microsoft Learn — What's new in .NET 11 runtime](https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-11/runtime)
- [dotnet/runtime — spécification de design runtime-async](https://github.com/dotnet/runtime/blob/main/docs/design/specs/runtime-async.md)
- [Laurent Kempé — Exploring .NET 11 Preview 1 Runtime Async](https://laurentkempe.com/2026/02/14/exploring-net-11-preview-1-runtime-async-a-dive-into-the-future-of-async-in-net/)

## Annexe

<details>
<summary><strong>IL complet — .NET 10 (SDK 10, machine à états)</strong></summary>

```
// ildasm — async-dotnet.dll (.NET 10, SDK 10)
// Metadata version: v4.0.30319
// =============== CLASS MEMBERS DECLARATION ===================

.class  public auto ansi beforefieldinit Program
  extends [System.Runtime]System.Object
{
  .custom  instance void [System.Runtime]System.Runtime.CompilerServices.NullableContextAttribute::.ctor(uint8)  = ( 01 00 01 00 00 )
  .custom  instance void [System.Runtime]System.Runtime.CompilerServices.NullableAttribute::.ctor(uint8)  = ( 01 00 00 00 00 )
  .class  auto ansi sealed nested private beforefieldinit '<DoTaskAsynchronously>d__1'
  extends [System.Runtime]System.Object
  implements [System.Runtime]System.Runtime.CompilerServices.IAsyncStateMachine
  {
  .custom  instance void [System.Runtime]System.Runtime.CompilerServices.CompilerGeneratedAttribute::.ctor()  = ( 01 00 00 00 )
  .interfaceimpl type [System.Runtime]System.Runtime.CompilerServices.IAsyncStateMachine
  .custom  instance void [System.Runtime]System.Runtime.CompilerServices.NullableAttribute::.ctor(uint8)  = ( 01 00 00 00 00 )
  .field  public int32 '<>1__state'
  .field  public valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder '<>t__builder'
  .field  private valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter '<>u__1'
  .method  public hidebysig specialname rtspecialname
  instance void  .ctor() cil managed
  // SIG: 20 00 01
  {
  // Method begins at RVA 0x20f7
  // Code size  8 (0x8)
  .maxstack  8
  IL_0000:  ldarg.0
  IL_0001:  call  instance void [System.Runtime]System.Object::.ctor()
  IL_0006:  nop
  IL_0007:  ret
  } // end of method '<DoTaskAsynchronously>d__1'::.ctor

  .method  private hidebysig newslot virtual final
  instance void  MoveNext() cil managed
  // SIG: 20 00 01
  {
  .override [System.Runtime]System.Runtime.CompilerServices.IAsyncStateMachine::MoveNext
  // Method begins at RVA 0x2100
  // Code size  156 (0x9c)
  .maxstack  3
  .locals  init (int32 V_0,
  valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter V_1,
  class Program/'<DoTaskAsynchronously>d__1' V_2,
  class [System.Runtime]System.Exception V_3)
  IL_0000:  ldarg.0
  IL_0001:  ldfld  int32 Program/'<DoTaskAsynchronously>d__1'::'<>1__state'
  IL_0006:  stloc.0
  .try
  {
  IL_0007:  ldloc.0
  IL_0008:  brfalse.s  IL_000c

  IL_000a:  br.s  IL_000e

  IL_000c:  br.s  IL_0049

  IL_000e:  nop
  IL_000f:  ldc.i4.s  100
  IL_0011:  call  class [System.Runtime]System.Threading.Tasks.Task [System.Runtime]System.Threading.Tasks.Task::Delay(int32)
  IL_0016:  callvirt  instance valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter [System.Runtime]System.Threading.Tasks.Task::GetAwaiter()
  IL_001b:  stloc.1
  IL_001c:  ldloca.s  V_1
  IL_001e:  call  instance bool [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter::get_IsCompleted()
  IL_0023:  brtrue.s  IL_0065

  IL_0025:  ldarg.0
  IL_0026:  ldc.i4.0
  IL_0027:  dup
  IL_0028:  stloc.0
  IL_0029:  stfld  int32 Program/'<DoTaskAsynchronously>d__1'::'<>1__state'
  IL_002e:  ldarg.0
  IL_002f:  ldloc.1
  IL_0030:  stfld  valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter Program/'<DoTaskAsynchronously>d__1'::'<>u__1'
  IL_0035:  ldarg.0
  IL_0036:  stloc.2
  IL_0037:  ldarg.0
  IL_0038:  ldflda  valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder Program/'<DoTaskAsynchronously>d__1'::'<>t__builder'
  IL_003d:  ldloca.s  V_1
  IL_003f:  ldloca.s  V_2
  IL_0041:  call  instance void [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder::AwaitUnsafeOnCompleted<valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter,class Program/'<DoTaskAsynchronously>d__1'>(!!0&,
  !!1&)
  IL_0046:  nop
  IL_0047:  leave.s  IL_009b

  IL_0049:  ldarg.0
  IL_004a:  ldfld  valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter Program/'<DoTaskAsynchronously>d__1'::'<>u__1'
  IL_004f:  stloc.1
  IL_0050:  ldarg.0
  IL_0051:  ldflda  valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter Program/'<DoTaskAsynchronously>d__1'::'<>u__1'
  IL_0056:  initobj  [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter
  IL_005c:  ldarg.0
  IL_005d:  ldc.i4.m1
  IL_005e:  dup
  IL_005f:  stloc.0
  IL_0060:  stfld  int32 Program/'<DoTaskAsynchronously>d__1'::'<>1__state'
  IL_0065:  ldloca.s  V_1
  IL_0067:  call  instance void [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter::GetResult()
  IL_006c:  nop
  IL_006d:  leave.s  IL_0087

  }  // end .try
  catch [System.Runtime]System.Exception
  {
  IL_006f:  stloc.3
  IL_0070:  ldarg.0
  IL_0071:  ldc.i4.s  -2
  IL_0073:  stfld  int32 Program/'<DoTaskAsynchronously>d__1'::'<>1__state'
  IL_0078:  ldarg.0
  IL_0079:  ldflda  valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder Program/'<DoTaskAsynchronously>d__1'::'<>t__builder'
  IL_007e:  ldloc.3
  IL_007f:  call  instance void [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder::SetException(class [System.Runtime]System.Exception)
  IL_0084:  nop
  IL_0085:  leave.s  IL_009b

  }  // end handler
  IL_0087:  ldarg.0
  IL_0088:  ldc.i4.s  -2
  IL_008a:  stfld  int32 Program/'<DoTaskAsynchronously>d__1'::'<>1__state'
  IL_008f:  ldarg.0
  IL_0090:  ldflda  valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder Program/'<DoTaskAsynchronously>d__1'::'<>t__builder'
  IL_0095:  call  instance void [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder::SetResult()
  IL_009a:  nop
  IL_009b:  ret
  } // end of method '<DoTaskAsynchronously>d__1'::MoveNext

  .method  private hidebysig newslot virtual final
  instance void  SetStateMachine(class [System.Runtime]System.Runtime.CompilerServices.IAsyncStateMachine stateMachine) cil managed
  // SIG: 20 01 01 12 59
  {
  .custom  instance void [System.Runtime]System.Diagnostics.DebuggerHiddenAttribute::.ctor()  = ( 01 00 00 00 )
  .override [System.Runtime]System.Runtime.CompilerServices.IAsyncStateMachine::SetStateMachine
  // Method begins at RVA 0x21b8
  // Code size  1 (0x1)
  .maxstack  8
  IL_0000:  ret
  } // end of method '<DoTaskAsynchronously>d__1'::SetStateMachine

  } // end of class '<DoTaskAsynchronously>d__1'

  .class  auto ansi sealed nested private beforefieldinit '<Main>d__0'
  extends [System.Runtime]System.Object
  implements [System.Runtime]System.Runtime.CompilerServices.IAsyncStateMachine
  {
  .custom  instance void [System.Runtime]System.Runtime.CompilerServices.CompilerGeneratedAttribute::.ctor()  = ( 01 00 00 00 )
  .interfaceimpl type [System.Runtime]System.Runtime.CompilerServices.IAsyncStateMachine
  .custom  instance void [System.Runtime]System.Runtime.CompilerServices.NullableAttribute::.ctor(uint8)  = ( 01 00 00 00 00 )
  .field  public int32 '<>1__state'
  .field  public valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder '<>t__builder'
  .field  private valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter '<>u__1'
  .method  public hidebysig specialname rtspecialname
  instance void  .ctor() cil managed
  // SIG: 20 00 01
  {
  // Method begins at RVA 0x21ba
  // Code size  8 (0x8)
  .maxstack  8
  IL_0000:  ldarg.0
  IL_0001:  call  instance void [System.Runtime]System.Object::.ctor()
  IL_0006:  nop
  IL_0007:  ret
  } // end of method '<Main>d__0'::.ctor

  .method  private hidebysig newslot virtual final
  instance void  MoveNext() cil managed
  // SIG: 20 00 01
  {
  .override [System.Runtime]System.Runtime.CompilerServices.IAsyncStateMachine::MoveNext
  // Method begins at RVA 0x21c4
  // Code size  154 (0x9a)
  .maxstack  3
  .locals  init (int32 V_0,
  valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter V_1,
  class Program/'<Main>d__0' V_2,
  class [System.Runtime]System.Exception V_3)
  IL_0000:  ldarg.0
  IL_0001:  ldfld  int32 Program/'<Main>d__0'::'<>1__state'
  IL_0006:  stloc.0
  .try
  {
  IL_0007:  ldloc.0
  IL_0008:  brfalse.s  IL_000c

  IL_000a:  br.s  IL_000e

  IL_000c:  br.s  IL_0047

  IL_000e:  nop
  IL_000f:  call  class [System.Runtime]System.Threading.Tasks.Task Program::DoTaskAsynchronously()
  IL_0014:  callvirt  instance valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter [System.Runtime]System.Threading.Tasks.Task::GetAwaiter()
  IL_0019:  stloc.1
  IL_001a:  ldloca.s  V_1
  IL_001c:  call  instance bool [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter::get_IsCompleted()
  IL_0021:  brtrue.s  IL_0063

  IL_0023:  ldarg.0
  IL_0024:  ldc.i4.0
  IL_0025:  dup
  IL_0026:  stloc.0
  IL_0027:  stfld  int32 Program/'<Main>d__0'::'<>1__state'
  IL_002c:  ldarg.0
  IL_002d:  ldloc.1
  IL_002e:  stfld  valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter Program/'<Main>d__0'::'<>u__1'
  IL_0033:  ldarg.0
  IL_0034:  stloc.2
  IL_0035:  ldarg.0
  IL_0036:  ldflda  valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder Program/'<Main>d__0'::'<>t__builder'
  IL_003b:  ldloca.s  V_1
  IL_003d:  ldloca.s  V_2
  IL_003f:  call  instance void [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder::AwaitUnsafeOnCompleted<valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter,class Program/'<Main>d__0'>(!!0&,
  !!1&)
  IL_0044:  nop
  IL_0045:  leave.s  IL_0099

  IL_0047:  ldarg.0
  IL_0048:  ldfld  valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter Program/'<Main>d__0'::'<>u__1'
  IL_004d:  stloc.1
  IL_004e:  ldarg.0
  IL_004f:  ldflda  valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter Program/'<Main>d__0'::'<>u__1'
  IL_0054:  initobj  [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter
  IL_005a:  ldarg.0
  IL_005b:  ldc.i4.m1
  IL_005c:  dup
  IL_005d:  stloc.0
  IL_005e:  stfld  int32 Program/'<Main>d__0'::'<>1__state'
  IL_0063:  ldloca.s  V_1
  IL_0065:  call  instance void [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter::GetResult()
  IL_006a:  nop
  IL_006b:  leave.s  IL_0085

  }  // end .try
  catch [System.Runtime]System.Exception
  {
  IL_006d:  stloc.3
  IL_006e:  ldarg.0
  IL_006f:  ldc.i4.s  -2
  IL_0071:  stfld  int32 Program/'<Main>d__0'::'<>1__state'
  IL_0076:  ldarg.0
  IL_0077:  ldflda  valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder Program/'<Main>d__0'::'<>t__builder'
  IL_007c:  ldloc.3
  IL_007d:  call  instance void [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder::SetException(class [System.Runtime]System.Exception)
  IL_0082:  nop
  IL_0083:  leave.s  IL_0099

  }  // end handler
  IL_0085:  ldarg.0
  IL_0086:  ldc.i4.s  -2
  IL_0088:  stfld  int32 Program/'<Main>d__0'::'<>1__state'
  IL_008d:  ldarg.0
  IL_008e:  ldflda  valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder Program/'<Main>d__0'::'<>t__builder'
  IL_0093:  call  instance void [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder::SetResult()
  IL_0098:  nop
  IL_0099:  ret
  } // end of method '<Main>d__0'::MoveNext

  .method  private hidebysig newslot virtual final
  instance void  SetStateMachine(class [System.Runtime]System.Runtime.CompilerServices.IAsyncStateMachine stateMachine) cil managed
  // SIG: 20 01 01 12 59
  {
  .custom  instance void [System.Runtime]System.Diagnostics.DebuggerHiddenAttribute::.ctor()  = ( 01 00 00 00 )
  .override [System.Runtime]System.Runtime.CompilerServices.IAsyncStateMachine::SetStateMachine
  // Method begins at RVA 0x227c
  // Code size  1 (0x1)
  .maxstack  8
  IL_0000:  ret
  } // end of method '<Main>d__0'::SetStateMachine

  } // end of class '<Main>d__0'

  .method  public hidebysig static
  class [System.Runtime]System.Threading.Tasks.Task
  Main() cil managed
  // SIG: 00 00 12 4D
  {
  .custom  instance void [System.Runtime]System.Runtime.CompilerServices.AsyncStateMachineAttribute::.ctor(class [System.Runtime]System.Type)  = ( 01 00 12 50 72 6F 67 72 61 6D 2B 3C 4D 61 69 6E  // ...Program+<Main
  3E 64 5F 5F 30 00 00 )  // >d__0..
  .custom  instance void [System.Runtime]System.Diagnostics.DebuggerStepThroughAttribute::.ctor()  = ( 01 00 00 00 )
  // Method begins at RVA 0x2050
  // Code size  49 (0x31)
  .maxstack  2
  .locals  init (class Program/'<Main>d__0' V_0)
  IL_0000:  newobj  instance void Program/'<Main>d__0'::.ctor()
  IL_0005:  stloc.0
  IL_0006:  ldloc.0
  IL_0007:  call  valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder::Create()
  IL_000c:  stfld  valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder Program/'<Main>d__0'::'<>t__builder'
  IL_0011:  ldloc.0
  IL_0012:  ldc.i4.m1
  IL_0013:  stfld  int32 Program/'<Main>d__0'::'<>1__state'
  IL_0018:  ldloc.0
  IL_0019:  ldflda  valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder Program/'<Main>d__0'::'<>t__builder'
  IL_001e:  ldloca.s  V_0
  IL_0020:  call  instance void [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder::Start<class Program/'<Main>d__0'>(!!0&)
  IL_0025:  ldloc.0
  IL_0026:  ldflda  valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder Program/'<Main>d__0'::'<>t__builder'
  IL_002b:  call  instance class [System.Runtime]System.Threading.Tasks.Task [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder::get_Task()
  IL_0030:  ret
  } // end of method Program::Main

  .method  private hidebysig static
  class [System.Runtime]System.Threading.Tasks.Task
  DoTaskAsynchronously() cil managed
  // SIG: 00 00 12 4D
  {
  .custom  instance void [System.Runtime]System.Runtime.CompilerServices.AsyncStateMachineAttribute::.ctor(class [System.Runtime]System.Type)  = ( 01 00 22 50 72 6F 67 72 61 6D 2B 3C 44 6F 54 61  // .."Program+<DoTa
  73 6B 41 73 79 6E 63 68 72 6F 6E 6F 75 73 6C 79  // skAsynchronously
  3E 64 5F 5F 31 00 00 )  // >d__1..
  .custom  instance void [System.Runtime]System.Diagnostics.DebuggerStepThroughAttribute::.ctor()  = ( 01 00 00 00 )
  // Method begins at RVA 0x2090
  // Code size  49 (0x31)
  .maxstack  2
  .locals  init (class Program/'<DoTaskAsynchronously>d__1' V_0)
  IL_0000:  newobj  instance void Program/'<DoTaskAsynchronously>d__1'::.ctor()
  IL_0005:  stloc.0
  IL_0006:  ldloc.0
  IL_0007:  call  valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder::Create()
  IL_000c:  stfld  valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder Program/'<DoTaskAsynchronously>d__1'::'<>t__builder'
  IL_0011:  ldloc.0
  IL_0012:  ldc.i4.m1
  IL_0013:  stfld  int32 Program/'<DoTaskAsynchronously>d__1'::'<>1__state'
  IL_0018:  ldloc.0
  IL_0019:  ldflda  valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder Program/'<DoTaskAsynchronously>d__1'::'<>t__builder'
  IL_001e:  ldloca.s  V_0
  IL_0020:  call  instance void [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder::Start<class Program/'<DoTaskAsynchronously>d__1'>(!!0&)
  IL_0025:  ldloc.0
  IL_0026:  ldflda  valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder Program/'<DoTaskAsynchronously>d__1'::'<>t__builder'
  IL_002b:  call  instance class [System.Runtime]System.Threading.Tasks.Task [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder::get_Task()
  IL_0030:  ret
  } // end of method Program::DoTaskAsynchronously

  .method  public hidebysig specialname rtspecialname
  instance void  .ctor() cil managed
  // SIG: 20 00 01
  {
  // Method begins at RVA 0x20cd
  // Code size  8 (0x8)
  .maxstack  8
  IL_0000:  ldarg.0
  IL_0001:  call  instance void [System.Runtime]System.Object::.ctor()
  IL_0006:  nop
  IL_0007:  ret
  } // end of method Program::.ctor

  .method  private hidebysig specialname static
  void  '<Main>'() cil managed
  // SIG: 00 00 01
  {
  .entrypoint
  .custom  instance void [System.Runtime]System.Diagnostics.DebuggerStepThroughAttribute::.ctor()  = ( 01 00 00 00 )
  // Method begins at RVA 0x20d8
  // Code size  19 (0x13)
  .maxstack  1
  .locals  init (valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter V_0)
  IL_0000:  call  class [System.Runtime]System.Threading.Tasks.Task Program::Main()
  IL_0005:  callvirt  instance valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter [System.Runtime]System.Threading.Tasks.Task::GetAwaiter()
  IL_000a:  stloc.0
  IL_000b:  ldloca.s  V_0
  IL_000d:  call  instance void [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter::GetResult()
  IL_0012:  ret
  } // end of method Program::'<Main>'

} // end of class Program
```

</details>

<details>
<summary><strong>IL complet — .NET 11 (SDK 11, runtime-async=on)</strong></summary>

```
// ildasm — async-dotnet.dll (.NET 11, SDK 11, runtime-async=on)
// Metadata version: v4.0.30319
// =============== CLASS MEMBERS DECLARATION ===================

.class  public auto ansi beforefieldinit Program
  extends [System.Runtime]System.Object
{
  .custom  instance void [System.Runtime]System.Runtime.CompilerServices.NullableContextAttribute::.ctor(uint8)  = ( 01 00 01 00 00 )
  .custom  instance void [System.Runtime]System.Runtime.CompilerServices.NullableAttribute::.ctor(uint8)  = ( 01 00 00 00 00 )
  .method  public hidebysig static
  class [System.Runtime]System.Threading.Tasks.Task
  Main() cil managed
  // SIG: 00 00 12 41
  {
  // Method begins at RVA 0x2050
  // Code size  13 (0xd)
  .maxstack  8
  IL_0000:  nop
  IL_0001:  call  class [System.Runtime]System.Threading.Tasks.Task Program::DoTaskAsynchronously()
  IL_0006:  call  void [System.Runtime]System.Runtime.CompilerServices.AsyncHelpers::Await(class [System.Runtime]System.Threading.Tasks.Task)
  IL_000b:  nop
  IL_000c:  ret
  } // end of method Program::Main

  .method  private hidebysig static
  class [System.Runtime]System.Threading.Tasks.Task
  DoTaskAsynchronously() cil managed
  // SIG: 00 00 12 41
  {
  // Method begins at RVA 0x205e
  // Code size  15 (0xf)
  .maxstack  8
  IL_0000:  nop
  IL_0001:  ldc.i4.s  100
  IL_0003:  call  class [System.Runtime]System.Threading.Tasks.Task [System.Runtime]System.Threading.Tasks.Task::Delay(int32)
  IL_0008:  call  void [System.Runtime]System.Runtime.CompilerServices.AsyncHelpers::Await(class [System.Runtime]System.Threading.Tasks.Task)
  IL_000d:  nop
  IL_000e:  ret
  } // end of method Program::DoTaskAsynchronously

  .method  public hidebysig specialname rtspecialname
  instance void  .ctor() cil managed
  // SIG: 20 00 01
  {
  // Method begins at RVA 0x206e
  // Code size  8 (0x8)
  .maxstack  8
  IL_0000:  ldarg.0
  IL_0001:  call  instance void [System.Runtime]System.Object::.ctor()
  IL_0006:  nop
  IL_0007:  ret
  } // end of method Program::.ctor

  .method  private hidebysig specialname static
  void  '<Main>'() cil managed
  // SIG: 00 00 01
  {
  .entrypoint
  .custom  instance void [System.Runtime]System.Diagnostics.DebuggerStepThroughAttribute::.ctor()  = ( 01 00 00 00 )
  // Method begins at RVA 0x2077
  // Code size  11 (0xb)
  .maxstack  8
  IL_0000:  call  class [System.Runtime]System.Threading.Tasks.Task Program::Main()
  IL_0005:  call  void [System.Runtime]System.Runtime.CompilerServices.AsyncHelpers::HandleAsyncEntryPoint(class [System.Runtime]System.Threading.Tasks.Task)
  IL_000a:  ret
  } // end of method Program::'<Main>'

} // end of class Program
```

</details>

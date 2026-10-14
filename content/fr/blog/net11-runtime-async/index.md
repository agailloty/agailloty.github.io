---
title: ".NET 11 a tué la machine à états"
description: "Depuis C# 5, async/await reposait entièrement sur une réécriture du compilateur. .NET 11 change la donne : le runtime exécute désormais l'asynchronie nativement."
date: 2026-10-14T08:00:00Z
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [".NET", "C#", "async", "runtime", "IL"]
---

Quinze ans d'async/await reposaient sur une illusion. Quand vous écriviez une méthode `async`, elle n'a jamais existé comme méthode : le compilateur C# la découpait en une machine à états, un struct généré implémentant `IAsyncStateMachine`, dont le rôle était de simuler une coroutine que le runtime ne savait pas exécuter. Avec .NET 11, cette époque s'achève. Le runtime lui-même comprend désormais la suspension et la reprise, et la machine à états disparaît.

C'est le genre de changement qu'on ne voit pas dans les notes de release les plus visibles, mais qui redéfinit la mécanique interne de l'écosystème. J'ai voulu le vérifier par moi-même, et vous pouvez le faire aussi. Prenez le programme le plus simple possible, compilez-le avec le SDK .NET 10, puis avec le SDK .NET 11 et la feature `runtime-async` activée, et décompilez les deux DLL avec ildasm. La différence est spectaculaire.

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
        await Task.Run(() => 5 + 1);
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

## Le contrat historique : la machine à états

Commençons par .NET 10, c'est-à-dire le comportement de C# 5 à C# 13, inchangé depuis 2012.

Dans le dump, la méthode `Main` que vous avez écrite n'exécute rien d'utile. Elle ne contient que 49 octets d'IL, et son unique travail est d'instancier le vrai cœur du programme : un struct nommé `<Main>d__0`. Même chose pour `DoTaskAsynchronously`, relayée à `<DoTaskAsynchronously>d__1`.

Chaque struct générée implémente `IAsyncStateMachine` et porte trois champs de plomberie : `<>1__state` (le point de reprise, un entier), `<>t__builder` (un `AsyncTaskMethodBuilder` qui pilote la complétion et le `Task` retourné) et `<>u__1` (l'awaiter sauvegardé entre deux suspensions).

Toute la logique vit dans `MoveNext()`. La première exécution déroule la méthode jusqu'au premier `await`. Si l'opération n'est pas terminée, la machine sauvegarde son état, appelle `AwaitUnsafeOnCompleted` pour s'enregistrer auprès de la tâche, puis retourne. Plus tard, le runtime rappelle `MoveNext()`, le champ `<>1__state` indique où reprendre, et l'exécution continue après l'`await`. Un bloc `try/catch` encadre le tout pour router les exceptions vers `SetException`, et `SetResult` conclut.

Traduit en C#, `MoveNext` de `Main` ressemble à ceci :

```csharp
int num = <>1__state;
try
{
    TaskAwaiter awaiter;
    if (num != 0)
    {
        awaiter = DoTaskAsynchronously().GetAwaiter();
        if (!awaiter.IsCompleted)
        {
            <>1__state = 0;
            <>u__1 = awaiter;
            <>t__builder.AwaitUnsafeOnCompleted(ref awaiter, ref this);
            return; // suspension : la méthode "sort" ici
        }
    }
    else
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
```

Pour nos deux petites méthodes d'une ligne, le compilateur a produit deux structs, deux `MoveNext`, deux `SetStateMachine`, six champs, deux blocs `try/catch` et deux stubs de lancement. Au total, 339 octets d'IL pour `Main` seule. C'est le prix historique d'`async` : chaque variable qui survit à un `await` devient un champ, chaque suspension devient un état, chaque méthode une classe déguisée.

Il faut le dire : cette conception a remarquablement bien vieilli. Elle a fait fonctionner async/await sur des runtimes qui n'en connaissaient rien. Mais elle duplique la logique dans chaque méthode async compilée, et le runtime reste, au fond, ignorant de ce qui se passe.

## La rupture : le CLR devient conscient de l'asynchronie

Ouvrons maintenant le dump .NET 11. Les structs `<Main>d__0` et `<DoTaskAsynchronously>d__1` n'existent plus. Pas de `IAsyncStateMachine`, pas de builder, pas de champ `<>1__state`, pas d'`AsyncStateMachineAttribute`. Il ne reste que la display class `<>c`, identique dans les deux versions, qui cache le délégué de la lambda.

`Main` tient désormais en 13 octets d'IL :

```il
.method public hidebysig static class [System.Runtime]System.Threading.Tasks.Task Main() cil managed
{
    // Code size 13 (0xd)
    IL_0000:  nop
    IL_0001:  call       class [System.Runtime]System.Threading.Tasks.Task Program::DoTaskAsynchronously()
    IL_0006:  call       void [System.Runtime]System.Runtime.CompilerServices.AsyncHelpers::Await(class [System.Runtime]System.Threading.Tasks.Task)
    IL_000b:  nop
    IL_000c:  ret
}
```

L'IL est linéaire. Plus de switch sur un état, plus de `try/catch` généré, plus de builder. L'`await` devient un appel de méthode ordinaire : `AsyncHelpers.Await`. C'est le point de bascule conceptuel. En .NET 10, la suspension était un `return` depuis `MoveNext` après un appel à `AwaitUnsafeOnCompleted`. En .NET 11, la suspension est encapsulée dans `AsyncHelpers.Await`, et c'est le CLR qui sauvegarde et restaure l'état de la méthode, y compris les variables locales qui vivent à travers le point d'attente. Seules les variables réellement concernées sont préservées, et le runtime s'en charge directement plutôt que de les hoister en champs.

Même le point d'entrée du processus change de visage. En .NET 10, le stub `<Main>()` bloquait sur `GetAwaiter().GetResult()`. En .NET 11, il délègue à `AsyncHelpers.HandleAsyncEntryPoint`. Jusqu'à l'amorçage du programme, l'async est repassé du compilateur au runtime.

## Ce que ça change concrètement

Pour le même code source, l'assemblage .NET 11 contient moins de types, moins de méthodes, moins de champs et moins de métadonnées. Le JIT a moins de travail. À l'exécution, le chemin est plus direct : plus de struct machine à états à boxer quand elle traverse `AwaitUnsafeOnCompleted`, plus de builder à instancier.

Le bénéfice le plus intéressant concerne les chaînes d'appels async. Quand une méthode runtime-async en attend une autre, le runtime peut optimiser le relais entre les deux, et dans certains cas éviter complètement l'allocation de `Task`. Chaque couche intermédiaire d'une chaîne async était hier une machine à états qui composait avec la suivante ; c'est désormais une seule exécution pilotée par le CLR.

Deux précisions honnêtes pour finir. D'abord, la sémantique observable du code ne change pas : c'est une évolution de la stratégie d'exécution, pas du langage. Ensuite, la feature est opt-in via `<Features>runtime-async=on</Features>`, ce qui est cohérent avec un premier aperçu .NET 11. Elle mérite d'être explorée maintenant pour être comprise le jour où elle devient le comportement par défaut.

## Conclusion

La machine à états a été l'une des astuces les plus élégantes de l'histoire de C# : faire croire à des coroutines dans un runtime qui n'en voulait pas. .NET 11 la rend obsolète en déplaçant l'asynchronie là où elle aurait toujours dû vivre : dans le runtime. Le compilateur se contente désormais d'exprimer la suspension, `AsyncHelpers.Await`, et le CLR fait le reste.

Les dumps IL complets des deux versions sont en annexe. Ouvrez-les, comparez, et surtout refaites l'expérience chez vous : c'est en décompilant soi-même qu'on mesure vraiment l'ampleur du changement.

## Annexe

<details>
<summary><strong>IL complet — .NET 10 (SDK 10, machine à états)</strong></summary>

```il
//  Microsoft (R) .NET Framework IL Disassembler.  Version 4.8.3928.0
// Metadata version: v4.0.30319
.assembly extern System.Runtime
{
  .publickeytoken = (B0 3F 5F 7F 11 D5 0A 3A )
  .ver 10:0:0:0
}
.assembly 'async-dotnet'
{
  .hash algorithm 0x00008004
  .ver 1:0:0:0
}
.module 'async-dotnet.dll'
.corflags 0x00000001    //  ILONLY

.class public auto ansi beforefieldinit Program
       extends [System.Runtime]System.Object
{
  .class auto ansi serializable sealed nested private beforefieldinit '<>c'
         extends [System.Runtime]System.Object
  {
    .field public static initonly class Program/'<>c' '<>9'
    .field public static class [System.Runtime]System.Func`1<int32> '<>9__1_0'
    .method private hidebysig specialname rtspecialname static void  .cctor() cil managed
    {
      .maxstack  8
      IL_0000:  newobj     instance void Program/'<>c'::.ctor()
      IL_0005:  stsfld     class Program/'<>c' Program/'<>c'::'<>9'
      IL_000a:  ret
    }
    .method public hidebysig specialname rtspecialname instance void  .ctor() cil managed
    {
      .maxstack  8
      IL_0000:  ldarg.0
      IL_0001:  call       instance void [System.Runtime]System.Object::.ctor()
      IL_0006:  nop
      IL_0007:  ret
    }
    .method assembly hidebysig instance int32  '<DoTaskAsynchronously>b__1_0'() cil managed
    {
      .maxstack  8
      IL_0000:  ldc.i4.6
      IL_0001:  ret
    }
  } // end of class '<>c'

  .class auto ansi sealed nested private beforefieldinit '<DoTaskAsynchronously>d__1'
         extends [System.Runtime]System.Object
         implements [System.Runtime]System.Runtime.CompilerServices.IAsyncStateMachine
  {
    .field public int32 '<>1__state'
    .field public valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder '<>t__builder'
    .field private valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter`1<int32> '<>u__1'
    .method public hidebysig specialname rtspecialname instance void  .ctor() cil managed
    {
      .maxstack  8
      IL_0000:  ldarg.0
      IL_0001:  call       instance void [System.Runtime]System.Object::.ctor()
      IL_0006:  nop
      IL_0007:  ret
    }
    .method private hidebysig newslot virtual final instance void  MoveNext() cil managed
    {
      .override [System.Runtime]System.Runtime.CompilerServices.IAsyncStateMachine::MoveNext
      // Code size 185 (0xb9)
      .maxstack  3
      .locals init (int32 V_0,
               valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter`1<int32> V_1,
               class Program/'<DoTaskAsynchronously>d__1' V_2,
               class [System.Runtime]System.Exception V_3)
      IL_0000:  ldarg.0
      IL_0001:  ldfld      int32 Program/'<DoTaskAsynchronously>d__1'::'<>1__state'
      IL_0006:  stloc.0
      .try
      {
        IL_0007:  ldloc.0
        IL_0008:  brfalse.s  IL_000c
        IL_000a:  br.s       IL_000e
        IL_000c:  br.s       IL_0066
        IL_000e:  nop
        IL_000f:  ldsfld     class [System.Runtime]System.Func`1<int32> Program/'<>c'::'<>9__1_0'
        IL_0014:  dup
        IL_0015:  brtrue.s   IL_002e
        IL_0017:  pop
        IL_0018:  ldsfld     class Program/'<>c' Program/'<>c'::'<>9'
        IL_001d:  ldftn      instance int32 Program/'<>c'::'<DoTaskAsynchronously>b__1_0'()
        IL_0023:  newobj     instance void class [System.Runtime]System.Func`1<int32>::.ctor(object, native int)
        IL_0028:  dup
        IL_0029:  stsfld     class [System.Runtime]System.Func`1<int32> Program/'<>c'::'<>9__1_0'
        IL_002e:  call       class [System.Runtime]System.Threading.Tasks.Task`1<!0> [System.Runtime]System.Threading.Tasks.Task::Run<int32>(class [System.Runtime]System.Func`1<!0>)
        IL_0033:  callvirt   instance valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter`1<!0> class [System.Runtime]System.Threading.Tasks.Task`1<int32>::GetAwaiter()
        IL_0038:  stloc.1
        IL_0039:  ldloca.s   V_1
        IL_003b:  call       instance bool valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter`1<int32>::get_IsCompleted()
        IL_0040:  brtrue.s   IL_0082
        IL_0042:  ldarg.0
        IL_0043:  ldc.i4.0
        IL_0044:  dup
        IL_0045:  stloc.0
        IL_0046:  stfld      int32 Program/'<DoTaskAsynchronously>d__1'::'<>1__state'
        IL_004b:  ldarg.0
        IL_004c:  ldloc.1
        IL_004d:  stfld      valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter`1<int32> Program/'<DoTaskAsynchronously>d__1'::'<>u__1'
        IL_0052:  ldarg.0
        IL_0053:  stloc.2
        IL_0054:  ldarg.0
        IL_0055:  ldflda     valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder Program/'<DoTaskAsynchronously>d__1'::'<>t__builder'
        IL_005a:  ldloca.s   V_1
        IL_005c:  ldloca.s   V_2
        IL_005e:  call       instance void [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder::AwaitUnsafeOnCompleted<valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter`1<int32>,class Program/'<DoTaskAsynchronously>d__1'>(!!0&, !!1&)
        IL_0063:  nop
        IL_0064:  leave.s    IL_00b8
        IL_0066:  ldarg.0
        IL_0067:  ldfld      valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter`1<int32> Program/'<DoTaskAsynchronously>d__1'::'<>u__1'
        IL_006c:  stloc.1
        IL_006d:  ldarg.0
        IL_006e:  ldflda     valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter`1<int32> Program/'<DoTaskAsynchronously>d__1'::'<>u__1'
        IL_0073:  initobj    valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter`1<int32>
        IL_0079:  ldarg.0
        IL_007a:  ldc.i4.m1
        IL_007b:  dup
        IL_007c:  stloc.0
        IL_007d:  stfld      int32 Program/'<DoTaskAsynchronously>d__1'::'<>1__state'
        IL_0082:  ldloca.s   V_1
        IL_0084:  call       instance !0 valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter`1<int32>::GetResult()
        IL_0089:  pop
        IL_008a:  leave.s    IL_00a4
      }  // end .try
      catch [System.Runtime]System.Exception
      {
        IL_008c:  stloc.3
        IL_008d:  ldarg.0
        IL_008e:  ldc.i4.s   -2
        IL_0090:  stfld      int32 Program/'<DoTaskAsynchronously>d__1'::'<>1__state'
        IL_0095:  ldarg.0
        IL_0096:  ldflda     valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder Program/'<DoTaskAsynchronously>d__1'::'<>t__builder'
        IL_009b:  ldloc.3
        IL_009c:  call       instance void [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder::SetException(class [System.Runtime]System.Exception)
        IL_00a1:  nop
        IL_00a2:  leave.s    IL_00b8
      }  // end handler
      IL_00a4:  ldarg.0
      IL_00a5:  ldc.i4.s   -2
      IL_00a7:  stfld      int32 Program/'<DoTaskAsynchronously>d__1'::'<>1__state'
      IL_00ac:  ldarg.0
      IL_00ad:  ldflda     valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder Program/'<DoTaskAsynchronously>d__1'::'<>t__builder'
      IL_00b2:  call       instance void [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder::SetResult()
      IL_00b7:  nop
      IL_00b8:  ret
    } // end of method '<DoTaskAsynchronously>d__1'::MoveNext

    .method private hidebysig newslot virtual final instance void  SetStateMachine(class [System.Runtime]System.Runtime.CompilerServices.IAsyncStateMachine stateMachine) cil managed
    {
      .override [System.Runtime]System.Runtime.CompilerServices.IAsyncStateMachine::SetStateMachine
      .maxstack  8
      IL_0000:  ret
    }
  } // end of class '<DoTaskAsynchronously>d__1'

  .class auto ansi sealed nested private beforefieldinit '<Main>d__0'
         extends [System.Runtime]System.Object
         implements [System.Runtime]System.Runtime.CompilerServices.IAsyncStateMachine
  {
    .field public int32 '<>1__state'
    .field public valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder '<>t__builder'
    .field private valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter '<>u__1'
    .method private hidebysig newslot virtual final instance void  MoveNext() cil managed
    {
      .override [System.Runtime]System.Runtime.CompilerServices.IAsyncStateMachine::MoveNext
      // Code size 154 (0x9a)
      .maxstack  3
      .locals init (int32 V_0,
               valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter V_1,
               class Program/'<Main>d__0' V_2,
               class [System.Runtime]System.Exception V_3)
      IL_0000:  ldarg.0
      IL_0001:  ldfld      int32 Program/'<Main>d__0'::'<>1__state'
      IL_0006:  stloc.0
      .try
      {
        IL_0007:  ldloc.0
        IL_0008:  brfalse.s  IL_000c
        IL_000a:  br.s       IL_000e
        IL_000c:  br.s       IL_0047
        IL_000e:  nop
        IL_000f:  call       class [System.Runtime]System.Threading.Tasks.Task Program::DoTaskAsynchronously()
        IL_0014:  callvirt   instance valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter [System.Runtime]System.Threading.Tasks.Task::GetAwaiter()
        IL_0019:  stloc.1
        IL_001a:  ldloca.s   V_1
        IL_001c:  call       instance bool [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter::get_IsCompleted()
        IL_0021:  brtrue.s   IL_0063
        IL_0023:  ldarg.0
        IL_0024:  ldc.i4.0
        IL_0025:  dup
        IL_0026:  stloc.0
        IL_0027:  stfld      int32 Program/'<Main>d__0'::'<>1__state'
        IL_002c:  ldarg.0
        IL_002d:  ldloc.1
        IL_002e:  stfld      valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter Program/'<Main>d__0'::'<>u__1'
        IL_0033:  ldarg.0
        IL_0034:  stloc.2
        IL_0035:  ldarg.0
        IL_0036:  ldflda     valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder Program/'<Main>d__0'::'<>t__builder'
        IL_003b:  ldloca.s   V_1
        IL_003d:  ldloca.s   V_2
        IL_003f:  call       instance void [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder::AwaitUnsafeOnCompleted<valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter,class Program/'<Main>d__0'>(!!0&, !!1&)
        IL_0044:  nop
        IL_0045:  leave.s    IL_0099
        IL_0047:  ldarg.0
        IL_0048:  ldfld      valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter Program/'<Main>d__0'::'<>u__1'
        IL_004d:  stloc.1
        IL_004e:  ldarg.0
        IL_004f:  ldflda     valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter Program/'<Main>d__0'::'<>u__1'
        IL_0054:  initobj    [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter
        IL_005a:  ldarg.0
        IL_005b:  ldc.i4.m1
        IL_005c:  dup
        IL_005d:  stloc.0
        IL_005e:  stfld      int32 Program/'<Main>d__0'::'<>1__state'
        IL_0063:  ldloca.s   V_1
        IL_0065:  call       instance void [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter::GetResult()
        IL_006a:  nop
        IL_006b:  leave.s    IL_0085
      }  // end .try
      catch [System.Runtime]System.Exception
      {
        IL_006d:  stloc.3
        IL_006e:  ldarg.0
        IL_006f:  ldc.i4.s   -2
        IL_0071:  stfld      int32 Program/'<Main>d__0'::'<>1__state'
        IL_0076:  ldarg.0
        IL_0077:  ldflda     valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder Program/'<Main>d__0'::'<>t__builder'
        IL_007c:  ldloc.3
        IL_007d:  call       instance void [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder::SetException(class [System.Runtime]System.Exception)
        IL_0082:  nop
        IL_0083:  leave.s    IL_0099
      }  // end handler
      IL_0085:  ldarg.0
      IL_0086:  ldc.i4.s   -2
      IL_0088:  stfld      int32 Program/'<Main>d__0'::'<>1__state'
      IL_008d:  ldarg.0
      IL_008e:  ldflda     valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder Program/'<Main>d__0'::'<>t__builder'
        IL_0093:  call       instance void [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder::SetResult()
      IL_0098:  nop
      IL_0099:  ret
    } // end of method '<Main>d__0'::MoveNext

    .method private hidebysig newslot virtual final instance void  SetStateMachine(class [System.Runtime]System.Runtime.CompilerServices.IAsyncStateMachine stateMachine) cil managed
    {
      .override [System.Runtime]System.Runtime.CompilerServices.IAsyncStateMachine::SetStateMachine
      .maxstack  8
      IL_0000:  ret
    }
  } // end of class '<Main>d__0'

  .method public hidebysig static class [System.Runtime]System.Threading.Tasks.Task Main() cil managed
  {
    .custom instance void [System.Runtime]System.Runtime.CompilerServices.AsyncStateMachineAttribute::.ctor(class [System.Runtime]System.Type) = ( 01 00 12 50 72 6F 67 72 61 6D 2B 3C 4D 61 69 6E 3E 64 5F 5F 30 00 00 )
    .custom instance void [System.Runtime]System.Diagnostics.DebuggerStepThroughAttribute::.ctor() = ( 01 00 00 00 )
    // Code size 49 (0x31)
    .maxstack  2
    .locals init (class Program/'<Main>d__0' V_0)
    IL_0000:  newobj     instance void Program/'<Main>d__0'::.ctor()
    IL_0005:  stloc.0
    IL_0006:  ldloc.0
    IL_0007:  call       valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder::Create()
    IL_000c:  stfld      valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder Program/'<Main>d__0'::'<>t__builder'
    IL_0011:  ldloc.0
    IL_0012:  ldc.i4.m1
    IL_0013:  stfld      int32 Program/'<Main>d__0'::'<>1__state'
    IL_0018:  ldloc.0
    IL_0019:  ldflda     valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder Program/'<Main>d__0'::'<>t__builder'
    IL_001e:  ldloca.s   V_0
    IL_0020:  call       instance void [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder::Start<class Program/'<Main>d__0'>(!!0&)
    IL_0025:  ldloc.0
    IL_0026:  ldflda     valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder Program/'<Main>d__0'::'<>t__builder'
    IL_002b:  call       instance class [System.Runtime]System.Threading.Tasks.Task [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder::get_Task()
    IL_0030:  ret
  } // end of method Program::Main

  .method private hidebysig static class [System.Runtime]System.Threading.Tasks.Task DoTaskAsynchronously() cil managed
  {
    .custom instance void [System.Runtime]System.Runtime.CompilerServices.AsyncStateMachineAttribute::.ctor(class [System.Runtime]System.Type) = ( 01 00 22 50 72 6F 67 72 61 6D 2B 3C 44 6F 54 61 73 6B 41 73 79 6E 63 68 72 6F 6E 6F 75 73 6C 79 3E 64 5F 5F 31 00 00 )
    .custom instance void [System.Runtime]System.Diagnostics.DebuggerStepThroughAttribute::.ctor() = ( 01 00 00 00 )
    // Code size 49 (0x31)
    .maxstack  2
    .locals init (class Program/'<DoTaskAsynchronously>d__1' V_0)
    IL_0000:  newobj     instance void Program/'<DoTaskAsynchronously>d__1'::.ctor()
    IL_0005:  stloc.0
    IL_0006:  ldloc.0
    IL_0007:  call       valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder::Create()
    IL_000c:  stfld      valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder Program/'<DoTaskAsynchronously>d__1'::'<>t__builder'
    IL_0011:  ldloc.0
    IL_0012:  ldc.i4.m1
    IL_0013:  stfld      int32 Program/'<DoTaskAsynchronously>d__1'::'<>1__state'
    IL_0018:  ldloc.0
    IL_0019:  ldflda     valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder Program/'<DoTaskAsynchronously>d__1'::'<>t__builder'
    IL_001e:  ldloca.s   V_0
    IL_0020:  call       instance void [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder::Start<class Program/'<DoTaskAsynchronously>d__1'>(!!0&)
    IL_0025:  ldloc.0
    IL_0026:  ldflda     valuetype [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder Program/'<DoTaskAsynchronously>d__1'::'<>t__builder'
    IL_002b:  call       instance class [System.Runtime]System.Threading.Tasks.Task [System.Runtime]System.Runtime.CompilerServices.AsyncTaskMethodBuilder::get_Task()
    IL_0030:  ret
  } // end of method Program::DoTaskAsynchronously

  .method private hidebysig specialname static void '<Main>'() cil managed
  {
    .entrypoint
    // Code size 19 (0x13)
    .maxstack  1
    .locals init (valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter V_0)
    IL_0000:  call       class [System.Runtime]System.Threading.Tasks.Task Program::Main()
    IL_0005:  callvirt   instance valuetype [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter [System.Runtime]System.Threading.Tasks.Task::GetAwaiter()
    IL_000a:  stloc.0
    IL_000b:  ldloca.s   V_0
    IL_000d:  call       instance void [System.Runtime]System.Runtime.CompilerServices.TaskAwaiter::GetResult()
    IL_0012:  ret
  } // end of method Program::'<Main>'
} // end of class Program
```

</details>

<details>
<summary><strong>IL complet — .NET 11 (SDK 11, runtime-async=on)</strong></summary>

```il
//  Microsoft (R) .NET Framework IL Disassembler.  Version 4.8.3928.0
// Metadata version: v4.0.30319
.assembly extern System.Runtime
{
  .publickeytoken = (B0 3F 5F 7F 11 D5 0A 3A )
  .ver 11:0:0:0
}
.assembly 'async-dotnet'
{
  .hash algorithm 0x00008004
  .ver 1:0:0:0
}
.module 'async-dotnet.dll'
.corflags 0x00000001    //  ILONLY

.class public auto ansi beforefieldinit Program
       extends [System.Runtime]System.Object
{
  .class auto ansi serializable sealed nested private beforefieldinit '<>c'
         extends [System.Runtime]System.Object
  {
    .field public static initonly class Program/'<>c' '<>9'
    .field public static class [System.Runtime]System.Func`1<int32> '<>9__1_0'
    .method private hidebysig specialname rtspecialname static void  .cctor() cil managed
    {
      .maxstack  8
      IL_0000:  newobj     instance void Program/'<>c'::.ctor()
      IL_0005:  stsfld     class Program/'<>c' Program/'<>c'::'<>9'
      IL_000a:  ret
    }
    .method public hidebysig specialname rtspecialname instance void  .ctor() cil managed
    {
      .maxstack  8
      IL_0000:  ldarg.0
      IL_0001:  call       instance void [System.Runtime]System.Object::.ctor()
      IL_0006:  nop
      IL_0007:  ret
    }
    .method assembly hidebysig instance int32  '<DoTaskAsynchronously>b__1_0'() cil managed
    {
      .maxstack  8
      IL_0000:  ldc.i4.6
      IL_0001:  ret
    }
  } // end of class '<>c'

  .method public hidebysig static class [System.Runtime]System.Threading.Tasks.Task Main() cil managed
  {
    // Code size 13 (0xd)
    .maxstack  8
    IL_0000:  nop
    IL_0001:  call       class [System.Runtime]System.Threading.Tasks.Task Program::DoTaskAsynchronously()
    IL_0006:  call       void [System.Runtime]System.Runtime.CompilerServices.AsyncHelpers::Await(class [System.Runtime]System.Threading.Tasks.Task)
    IL_000b:  nop
    IL_000c:  ret
  } // end of method Program::Main

  .method private hidebysig static class [System.Runtime]System.Threading.Tasks.Task DoTaskAsynchronously() cil managed
  {
    // Code size 44 (0x2c)
    .maxstack  8
    IL_0000:  nop
    IL_0001:  ldsfld     class [System.Runtime]System.Func`1<int32> Program/'<>c'::'<>9__1_0'
    IL_0006:  dup
    IL_0007:  brtrue.s   IL_0020
    IL_0009:  pop
    IL_000a:  ldsfld     class Program/'<>c' Program/'<>c'::'<>9'
    IL_000f:  ldftn      instance int32 Program/'<>c'::'<DoTaskAsynchronously>b__1_0'()
    IL_0015:  newobj     instance void class [System.Runtime]System.Func`1<int32>::.ctor(object, native int)
    IL_001a:  dup
    IL_001b:  stsfld     class [System.Runtime]System.Func`1<int32> Program/'<>c'::'<>9__1_0'
    IL_0020:  call       class [System.Runtime]System.Threading.Tasks.Task`1<!!0> [System.Runtime]System.Threading.Tasks.Task::Run<int32>(class [System.Runtime]System.Func`1<!!0>)
    IL_0025:  call       !!0 [System.Runtime]System.Runtime.CompilerServices.AsyncHelpers::Await<int32>(class [System.Runtime]System.Threading.Tasks.Task`1<!!0>)
    IL_002a:  pop
    IL_002b:  ret
  } // end of method Program::DoTaskAsynchronously

  .method private hidebysig specialname static void '<Main>'() cil managed
  {
    .entrypoint
    // Code size 11 (0xb)
    .maxstack  8
    IL_0000:  call       class [System.Runtime]System.Threading.Tasks.Task Program::Main()
    IL_0005:  call       void [System.Runtime]System.Runtime.CompilerServices.AsyncHelpers::HandleAsyncEntryPoint(class [System.Runtime]System.Threading.Tasks.Task)
    IL_000a:  ret
  } // end of method Program::'<Main>'
} // end of class Program
```

</details>

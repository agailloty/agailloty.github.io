---
title: ".NET 11 killed the state machine"
description: "Since C# 5, async/await has rested entirely on a compiler rewrite. .NET 11 changes the game: the runtime now executes asynchrony natively."
date: 2026-10-14T08:00:00Z
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [".NET", "C#", "async", "runtime", "IL"]
---

Fifteen years of async/await rested on an illusion. When you wrote an `async` method, it never existed as a method: the C# compiler chopped it into a state machine, a generated struct implementing `IAsyncStateMachine`, whose job was to simulate a coroutine that the runtime could not execute. With .NET 11, that era ends. The runtime itself now understands suspension and resumption, and the state machine disappears.

This is the kind of change you don't see in the most visible release notes, yet it redefines the internal mechanics of the whole ecosystem. I wanted to verify it myself, and you can too. Take the simplest possible program, compile it with the .NET 10 SDK, then with the .NET 11 SDK and the `runtime-async` feature enabled, and disassemble both DLLs with ildasm. The difference is spectacular.

## The test program

Here is our guinea pig. Two async methods, a single `await` in each. Nothing simpler:

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

Under .NET 10, the csproj is minimal: `net10.0`. Under .NET 11, we explicitly opt into the new feature:

```xml
<TargetFramework>net11.0</TargetFramework>
<Features>runtime-async=on</Features>
```

Then:

```bash
ildasm /tok /byt async-dotnet.dll /out=dump.il
```

## The historical contract: the state machine

Let's start with .NET 10, that is, the behavior of C# 5 through C# 13, unchanged since 2012.

In the dump, the `Main` method you wrote executes nothing useful. It holds only 49 bytes of IL, and its sole job is to instantiate the real heart of the program: a struct named `<Main>d__0`. Same for `DoTaskAsynchronously`, relayed to `<DoTaskAsynchronously>d__1`.

Each generated struct implements `IAsyncStateMachine` and carries three plumbing fields: `<>1__state` (the resumption point, an integer), `<>t__builder` (an `AsyncTaskMethodBuilder` that drives completion and the returned `Task`), and `<>u__1` (the awaiter saved between suspensions).

All the logic lives in `MoveNext()`. The first run unwinds the method up to the first `await`. If the operation isn't complete, the machine saves its state, calls `AwaitUnsafeOnCompleted` to register with the task, and returns. Later, the runtime calls `MoveNext()` again, `<>1__state` tells it where to resume, and execution continues after the `await`. A `try/catch` block wraps everything to route exceptions to `SetException`, and `SetResult` concludes.

Translated back to C#, `Main`'s `MoveNext` looks like this:

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
            return; // suspension: the method "exits" here
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

For our two one-line methods, the compiler produced two structs, two `MoveNext` implementations, two `SetStateMachine` methods, six fields, two `try/catch` blocks, and two launcher stubs. In total, 339 bytes of IL for `Main` alone. That is the historical price of `async`: every variable that survives an `await` becomes a field, every suspension becomes a state, every method a class in disguise.

To be fair: this design aged remarkably well. It made async/await work on runtimes that knew nothing about it. But it duplicates logic in every compiled async method, and the runtime remains, deep down, oblivious to what is going on.

## The break: the CLR becomes async-aware

Now open the .NET 11 dump. The `<Main>d__0` and `<DoTaskAsynchronously>d__1` structs no longer exist. No `IAsyncStateMachine`, no builder, no `<>1__state` field, no `AsyncStateMachineAttribute`. Only the `<>c` display class remains, identical in both versions, caching the lambda's delegate.

`Main` now fits in 13 bytes of IL:

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

The IL is linear. No switch on a state, no generated `try/catch`, no builder. The `await` becomes an ordinary method call: `AsyncHelpers.Await`. That is the conceptual shift. In .NET 10, suspension was a `return` from `MoveNext` after calling `AwaitUnsafeOnCompleted`. In .NET 11, suspension is encapsulated inside `AsyncHelpers.Await`, and it is the CLR that saves and restores the method's state, including the locals that live across the await point. Only the variables actually concerned are preserved, handled directly by the runtime instead of being hoisted into fields.

Even the process entry point changes face. In .NET 10, the `<Main>()` stub blocked on `GetAwaiter().GetResult()`. In .NET 11, it delegates to `AsyncHelpers.HandleAsyncEntryPoint`. All the way down to program bootstrap, async has moved from the compiler back to the runtime.

## What it changes in practice

For the same source code, the .NET 11 assembly contains fewer types, fewer methods, fewer fields, and fewer metadata entries. The JIT has less work. At runtime, the path is more direct: no state machine struct to box when it crosses `AwaitUnsafeOnCompleted`, no builder to instantiate.

The most interesting benefit concerns async call chains. When one runtime-async method awaits another, the runtime can optimize the handoff between the two, and in some cases avoid allocating a `Task` entirely. Every intermediate layer of an async chain used to be a state machine composing with the next one; it is now a single CLR-driven execution.

Two honest caveats to finish. First, the observable semantics of your code do not change: this is an execution strategy evolution, not a language change. Second, the feature is opt-in via `<Features>runtime-async=on</Features>`, consistent with a first .NET 11 preview. It is worth exploring now so you understand it on the day it becomes the default.

## Conclusion

The state machine was one of the most elegant tricks in C#'s history: convincing developers they had coroutines in a runtime that wanted none of it. .NET 11 makes it obsolete by moving asynchrony where it always belonged: into the runtime. The compiler now merely expresses suspension, `AsyncHelpers.Await`, and the CLR does the rest.

The complete IL dumps of both versions are in the appendix. Open them, compare, and above all run the experiment yourself: disassembling it yourself is how you truly measure the scale of the change.

## Appendix

<details>
<summary><strong>Full IL — .NET 10 (SDK 10, state machine)</strong></summary>

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
<summary><strong>Full IL — .NET 11 (SDK 11, runtime-async=on)</strong></summary>

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

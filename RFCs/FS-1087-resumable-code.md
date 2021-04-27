# F# RFC FS-1087 - Resumable code and Resumable state machines

The design suggestion [Native support for task { ... } ](https://github.com/fsharp/fslang-suggestions/issues/581) has been marked "approved in principle".
This RFC covers the detailed proposal for the resumable state machine support needed for this and other features.

- [x] Approved in principle
- [x] [Suggestion](https://github.com/fsharp/fslang-suggestions/issues/581)
- [x] [RFC Discussion](https://github.com/fsharp/fslang-design/issues/455)
- [x] [Prototype](https://github.com/dotnet/fsharp/pull/6811)

# Summary

We add a general capability to specify and emit statically compositional resumable
code hosted in state machine objects recognized by the F# compiler. This allows some F#
computation expressions to be implemented via resumable code specified as part of the computation expression implementation.

# Motivation

`task { ... }` and other computation expressions need low-allocation implementations. Some other examples are asynchronous
sequences, faster list/array comprehensions and faster `option` and `result` computation expressions.

# Design Philosophy and Principles

The design philosophy is as follows:

1. No new syntax is added to the F# language. 

2. The primary aim is for _statically composable_ resumable code. That is, chunks of resumable code specified in FSharp.Core and other libraries that can be merged
   to form an overall resumable state machine in user code.

3. The F# metadata format is unchanged. Like `nameof` and other F# features,
   resumable state machines are encoded within existing TypedTree constructs using a combination of known compiler intrinsics
   and TypedTree expresions.

4. We treat this as a compiler feature. The actual feature is barely surfaced
   as a language feature, but is rather a set of idioms known to the F# compiler, together used to build efficient computation
   expression implementations.

5. The feature is activated in compiled code.  An alternative implementation of the primitives can be
   given for reflective execution, e.g. for interpretation of quotation code.

6. The feature is not fully checked during type checking, but rather checks are made as code is emitted. This means
   mis-implemented resumable code may be detected late in the compilation process, potentially when compiling user code. (NOTE: we will review this)

7. The feature is designed for use only by highly skilled F# developers to implement low-allocation computation
   expression builders.

8. Semantically, there's nothing you can do with resumable state machines that you can't already do with existing
   workflows. It is better to think of them as a performance feature, a compiler optimization partly implemented
   in workflow library code.

Points 1-3 guide many of the decisions below.

# Detailed design

### Specifying a resumable state machine (reference types)

Resumable state machines of reference type are specified using the ``ResumableCode`` attribute on a single method of a host object:
```fsharp
    { new SomeStateMachineType() with 
        [<ResumableCode>]
        member __.Step ()  = 
           if __useResumableCode then
               <resumable code>
           else
               <normal-code>
    }
```
Notes

* `__useResumableCode` and an object expression with a single `ResumableCode` method are well-known to the F# compiler.  

* A struct state machine may also be used to host the resumable code, see below. 

* Here `SomeStateMachineType` can be any user-defined reference type, and the object expression must contain a single `Step` method with the `ResumableCode` attribute.

* The `if __useResumableCode then` is needed because the compilation of resumable code is only activated when code is compiled.

* The `<dynamic-implementation>` is used for reflective execution, e.g. quotation interpretation.
  In prototyping it can simply raise an exception. It should be semantically identical to the other branch.
  
* Uses of `ResumableCode` are not allowed except in the exact patterns described in this RFC.

### Specifying resumable code

Resumable code is a new low-level primitive form of compositional re-entrant code suitable only for writing
high-performance compiled implementations of computation expressions.

Resumable code is specified by using compositions of calls to functions or methods whose parameters and/or return
are annotated with the `ResumableCode` attribute. These methods are usually part of a computation expression
builder or its implementation.

To describe the allowed compositions of resumable code, consider the following indicative set of functions or methods,
which is sufficient to covers the range of compositions allowed.  Note the names `Leaf`, `Composition` and `Consume` 
are indicative, and each method can take other non-`ResumableCode` parameters.
 
```fsharp
    [<return: ResumableCode>]
    let inline Leaf(x: int) = ...
 
    [<return: ResumableCode>]
    let inline Composition([<ResumableCode>] __expand_code1, [<ResumableCode>] __expand_code2) = ...
     
    // note no 'ResumableCode' on return, this consumes resumable code.
    let inline Consume([<ResumableCode>] __expand_code1)  = ...
```
     
Anything taking a `ResumableCode` parameter must be 'inline'.  This is because resumable code is always statically composed
at compilation-time.  All `ResumableCode` parameters should have name `__expand_*` and be of function or delegate type (this can include a function returning a delegate).

A method with `return: ResumableCode` must be non-abstract and must return a _resumable expression_. A resumable expression is:

1. A delegate expression with resumable body

   ```fsharp
   Delegate(fun sm -> <resumable-expr>)
   ```

2. A function-expression with resumable body

   ```fsharp
   (fun sm -> <resumable-expr>)
   ```

3. A call to a function or method returning `ResumableCode`.  

   ```fsharp
   Leaf(x)
   Composition(<resumable-expr>, <resumable-expr>)
   ```

   Any `ResumableCode` parameters must be passed a resumable statements for the composition to qualify as a resumable statement.
   A warning is emitted if the expression passed to a `ResumableCode` parameter is not determined to be a valid resumable statement.
   In this case, the expression is compiled as a normal expression where its inlined-expansion assumes `__useResumableCode` is false.

4. A optional resumable expression:

   ```fsharp
   if __useResumableCode then <resumable-expr> else <expr>
   ```

   The alternative implementation is used if resumable code is not being generated, e.g. if the overall construct doesn't form resumable code.

5. A resumption point:

   ```fsharp
   match __resumableEntry() with
   | Some contId -> <resumable-expr>
   | None -> <resumable-expr>
   ```

   If such an expression is executed, the first `Some` branch is taken. However a resumption point is also defined which,
   if a resumption is performed using `__resumeAt`, executes the `None` branch.
  
   The `Some` branch usually suspends execution by saving `contID` into the state machine
   for later use with a `__resumeAt` execution at the entry to the method. For example:
  
   ```fsharp
   let inline returnFrom (task: Task<'T>) =
       let mutable awaiter = task.GetAwaiter()
       match __resumableEntry() with 
       | Some contID ->
           sm.ResumptionPoint <- contID
           sm.MethodBuilder.AwaitUnsafeOnCompleted(&awaiter, &sm)
           false
       | None ->
           sm.Result <- awaiter.GetResult()
           true
   ```
  
   Note that, a resumption expression can return a result - in the above the resumption expression indicates whether the
   task ran to completion or not.

   At runtime, `__resumeAt contId` will jump directly to 'None' branch of the corresponding `match __resumableEntry ... `.
   All `__stack_*` locals in scope will be zero-initialized on resumption.

6. A `let` binding of a stack-bound variable initialized to zero on resumption.

   ```fsharp
   let __stack_var = ... in <resumable-expr> 
   ```

   Within resumable code, the name `__stack_*` indicates that the variable is always stack-bound and given the default value on resumption.
   
   Note that, because the code is resumable, the  `<resumable-expr>` may contain zero or more resumption points.  Again this
   means it is not guaranteed that the first `<resumable-expr>` will be executed before the
   second - a `__resumeAt` call can jump straight into the second code when the method is executed to resume previous execution.
   As a result, the variable should always be consumed prior to any resumption points and re-assigned if used after any resumption points. For example:

   ```fsharp
   let inline combine ([<ResumableCode>] __expand_task1: (unit -> bool), __expand_task2: (unit -> bool)) : [<ResumableCode>] (unit -> bool)  =
       (fun () -> 
           let __stack_step = __expand_task1()
           if __stack_step then 
               __expand_task2()
           else
              false)
   ```

7. A resumable try/with expression:

   ```fsharp
   try <resumable-expr> with <expr>
   ```

   Because the body of the try/with is resumable, the `<resumable-expr>` may contain zero or more resumption points.  The execution
   of the code may thus branch (via `__resumeAt`) into the middle of the `try` expression.
   
   > Note that the rules of .NET IL prohibit jumping directly into the code block of a try/with.  Instead the F# compiler
   > arranges that a jump is performed to the `try` and a subsequent jump is performed after the `try`.
   
   The `with` block is not resumable code and can't contain a resumption point.

8. A resumable try/finally expression:

   ```fsharp
   try <resumable-expr> finally <expr>
   ```

   Similar rules apply as for `try-with`. Because the body of the try/finally  is resumable, the `<resumable-expr>` may contain zero or more resumption points.  The execution
   of the code may thus branch (via `__resumeAt`) into the middle of the `try` expression.
   
   > Note that the rules of .NET IL prohibit jumping directly into the code block of a try/with.  Instead the F# compiler
   > arranges that a jump is performed to the `try` and a subsequent jump is performed after the `try` is entered.
   
9. A resumable while-loop

   ```fsharp
   while <expr> do <resumable-expr>
   ```

   Note that, because the code is resumable, the `<resumable-expr>` may contain zero or more resumption points.   The execution
   of the code may thus branch (via `__resumeAt`) into the middle of the `while` expression.

   The guard expression is not resumable code and can't contain a resumption point.  Asynchronous while loops that
   contain asynchronous while conditions must be handled by placing the resumable code for the guard expression
   separately, see examples.

10. A sequential execution of resumable code

    ```fsharp
    <resumable-stmt>; <resumable-stmt>
    ```

    Note that, because the code is resumable, each `<resumable-stmt>` may contain zero or more resumption points.
    This means it is **not** guaranteed that the first `<resumable-stmt>` will be executed before the
    second - a `__resumeAt` call can jump straight into the second code.

11. A call/invoke of a `ResumableCode` delegate/function parameter, e.g.

    ```fsharp
    __expand_code arg
    __expand_code.Invoke(&sm)
    (__expand_code arg).Invoke(&sm)
    ```

    NOTE: Using delegates to form compositional code fragments is particularly useful because a delegate may take a byref parameter, normally
    the address of the enclosing value type state machine.

12. If no previous case applies, a resumable `match` expression:

    ```fsharp
    match <expr> with
    | ...  -> <resumable-expr>
    | ...  -> <resumable-expr>
    ```

   Note that, because the code is resumable, each `<resumable-stmt>` may contain zero or more resumption points.  The execution
   of the code may thus "begin" (via `__resumeAt`) in the middle of the code on each branch.

13. Any other F# expression excluding `for` loops and `let rec` bindings. 

    ```fsharp
    <expr>
    ```
        
    In this case, the expression may not contain resumable code constructs, and is simply executed.

### Hosting resumable code in an object

A resumable state machine expression is:

```fsharp
{ new StateMachine() with 
     [<ResumableCode>]
     member x.M() = 
       <resumable-stmt>
}
```
or similarly with
```fsharp
__resumableStateMachineStruct(....)
```
as described further below.  

### The semantics of resumable code

The execution of resumable code is best understood in terms of the direct translation of the constructs into a .NET method.
For example, `__resumeAt` corresponds either to a `goto` (for a known label) or a switch table (for a computed label at the
start of a method).

If a `ResumableCode` expression is determined to be valid resumable code, then the semantics of the
method or function hosting the resumable code is detemined by the following:

1. All implementations are inlined under the static assumption `__useResumableCode` is true.

2. All resumption points `match __resumableEntry() with Some contId -> <stmt1> | None -> <stmt2>` are removed by the static allocation of a unique integer within the resumable code for `contID` and using `<stmt1>` as the primary implementation.  `stmt2` is placed as the target for `contID` in a single implied jump table for the overall resumable code.

3. Any `__stack_*` variables are represented as locals of the method. These are zero-initialized each time the method is invoked.

4. Any non `__stack_*` variables are represented as locals of the host object. (Note, if the variables are not used in or after continuation branches then they may be represented as locals of the method).

5. Any uses of `__resumeAt <expr>` are represented as an invocation of the implied jump table.

   - If `<expr>` is a statically-determined code label (e.g. a `contID`) then this is effectively a `goto` statement
     to the `None` branch of the resumption point corresponding to the `contID`. 

   - If `<expr>` is not a statically-determined code label then the `__resumeAt` must be the first statement within the method.
     If at runtime the `<expr>` doesn't correspond to a valid resumption point within the method then execution continues subsequent to the `__resumeAt`.
   
### Static checking of resumable code

Many static checks are performed for the construction of resumable code as outlined above. However, there may still be cases where the
application of the semantics of resumable code fails.  The static checking of resumable code is primarily designed to ensure compositions
of resumable code are checked to form resumable code, and a warning is emitted if this is not statically determined.

Resumable code may **not** contain `let rec` bindings.  These must be lifted out or a warning will be emitted.


### Specifying resumable state machine structs

A struct may be used to host a resumable state machine using the following formulation:

```fsharp
if __useResumableCode then
    __resumableStateMachineStruct<StructStateMachine<'T>, _>
        (MoveNextMethod(fun sm -> <resumable-code>))
        (SetMachineStateMethod(fun sm state -> ...))
        (AfterMethod(fun sm -> ...))
else
    ...
```
Notes:

1. A "template" struct type must be given at a type parameter to `__resumableStateMachineStruct`, in this example it is `StructStateMachine`, a user-defined type normally in the same file.  This must have exactly one interface, and be marked `NoComparison` and `NoEquality`, and if it has methods they must all be non-virtual and marked `inline`.

2. The template struct type must implement one interface, the [`IAsyncMachine`](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.iasyncstatemachine) interface. 

3. The three delegate parameters specify the implementations of the `MoveNext`, `SetMachineState` methods, plus an `After` code
   block that is run on the state machine immediately after creation.  Delegates are used as they can receive the address of the
   state machine.

4. For each use of this construct, the template struct type is copied to to a new (internal) struct type, the state variables
   from the resumable code are added, and the `IAsyncMachine` interface is filled in using the supplied methods.

NOTE: By way of explanation, reference-typed resumable state machines are expressed using object expressions, which can
have additional state variables.  However F# object-expressions may not be of struct type, so it is always necessary
to fabricate an entirely new struct type for each state machine use. There is no existing construct in F#
for the anonymous specification of struct types whose methods can capture a closure of variables. The above
intrinsic effectively adds a limited version of a capability to use an existing struct type as a template for the
anonymous specification of an implicit, closure-capturing struct type.  The anonymous struct type must be immediately
eliminated (i.e. used) in the `AfterMethod`.  

## Feature: Respecting Zero methods in computation expressions 

Prior to this RFC, the `do! expr` construct in final position of a computation branch
in an F# computation expressions was processed as follows:

1. If a `Return` method is present, process as if `builder.Bind(expr, fun () -> builder.Return ())`

2. Otherwise, process as `builder.Bind(expr, fun () -> builder.Zero ())`

The new rule has an extra condition as follows:

1. If a `Return` method is present and there is no `Zero` method present with `DefaultValue` attribute, process as if `builder.Bind(expr, fun () -> builder.Return ())`

2. Otherwise, process as `builder.Bind(expr, fun () -> builder.Zero ())`

For example,
```fsharp
    task {
        if true then 
            do! Task.Delay(100)
        return 4
    }
```

is de-sugared to

```fsharp
task.Combine(
    (if true then task.Bind(Task.Delay(100), fun () -> task.Zero()) else task.Zero()),
    task.Delay(fun () -> task.Return(4)))`.
```

rather than

```fsharp
task.Combine(
    (if true then task.Bind(Task.Delay(100), fun () -> task.Return()) else task.Zero()),
    task.Delay(fun () -> task.Return(4)))`.
```

Motivation: This corrects a minor problem with F# computation expressions.
This allows builders ensure `Return` is not implicitly required by `do!` expressions in final position by adding the `DefaultValue`
attribute to the `Zero` method on the builder type.  This brings the treatment of `do!` in line with the treatment of
the implicit result on an implicit `else` branch in `if .. then` constructs, and allows `Return` to have a more
restrictive signature than `Zero`.  In particular in the task builder, we have

```fsharp
type TaskBuilder =
    member inline Return: x: 'T -> TaskCode<'T, 'T>

    [<DefaultValue>]
    member inline Zero: unit -> TaskCode<'TOverall, unit>
```

Here implicit `Zero` values (implied by `do!` and `if .. then ..`) can now occur anywhere in a computation expression, regardless
of the overall type of the task being returned by  the CE.  In contrast, `Return` values (in an explicit `return`) must
match the type returned by the overall task.


## Library additions

```fsharp
namespace FSharp.Core.CompilerServices

type MoveNextMethod<'Template> = delegate of byref<'Template> -> unit

type SetMachineStateMethod<'Template> = delegate of byref<'Template> * IAsyncStateMachine -> unit

type AfterMethod<'Template, 'Result> = delegate of byref<'Template> -> 'Result

/// Contains compiler intrinsics related to the definition of state machines.
module StateMachineHelpers = 

    val __useResumableCode<'T> : bool 

    val __resumableStateMachineStruct<'Template, 'Result> : moveNext: MoveNextMethod<'Template> -> _setMachineState: SetMachineStateMethod<'Template> -> after: AfterMethod<'Template, 'Result> -> 'Result

    val __resumableEntry: unit -> int option

    val __resumeAt : pc: int -> 'T   
```

### Library additions (for future List builders and high-performance list functions)

In the future we will use resumable code to add more efficient list and array builders.
It is not yet decided if such a builder should be in FSharp.Core.
To support the definition of such a list builder outside FSharp.Core, we expose a library intrinsic to allow the tail-mutation of
lists. Like other constructs in FSharp.Core this is a low-level primitive not for use in user-code and carries a warning.

```fsharp
namespace Microsoft.FSharp.Core.CompilerServices

    [<RequireQualifiedAccess>]
    module RuntimeHelpers = 
        ...
    
        [<Experimental("Experimental library feature, requires '--langversion:preview'")>]
        [<CompilerMessage("This function is for use by compiled F# code and should not be used directly", 1204, IsHidden=true)>]
        val inline FreshConsNoTail: head: 'T -> 'T list
        
        [<Experimental("Experimental library feature, requires '--langversion:preview'")>]
        [<MethodImpl(MethodImplOptions.NoInlining)>]
        [<CompilerMessage("This function is for use by compiled F# code and should not be used directly", 1204, IsHidden=true)>]
        val SetFreshConsTail: cons: 'T list -> tail: 'T list -> unit
```

This function can also be used for higher-performance list function implementations external to FSharp.Core, though must be used with care.

# Performance

[Recent perf status of implementation](https://github.com/dotnet/fsharp/blob/feature/tasks/BenchmarkDotNet.Artifacts/results/TaskPerf.Benchmarks-report-github.md)

# Examples

## Example: option { ... }

See [option.fs](https://github.com/dotnet/fsharp/blob/feature/tasks/tests/fsharp/perf/tasks/FS/option.fs)

The use is as for a typical `option { ... }` computation expression builder:

```fsharp
let testOption i = 
    option {
        let! x1 = (if i % 5 <> 2 then Some i else None)
        let! x2 = (if i % 3 <> 1 then Some i else None)
        return x1 + x2
    } 
```

## Example: sync { ... }

As a micro example of defining a `sync { ... }` builder for entirely synchronous computation with no special semantics.

See [sync.fs](https://github.com/dotnet/fsharp/blob/feature/tasks/tests/fsharp/perf/tasks/FS/sync.fs)

Examples of use:
```fsharp
let t1 y = 
    sync {
       printfn "in t1"
       let x = 4 + 5 + y
       return x
    }

let t2 y = 
    sync {
       printfn "in t2"
       let! x = t1 y
       return x + y
    }

printfn "t2 6 = %d" (t2 6)
```
Code performance is approximately the same as normal F# code except for one allocation for each execution of each `sync { .. }` as we allocate the "SyncMachine".  In later work we may be able to remove this.

## Example: task { ... }

See [tasks.fs](https://github.com/dotnet/fsharp/blob/feature/tasks/src/fsharp/FSharp.Core/tasks.fs).  

## Example: taskSeq { ... }

See [taskSeq.fs](https://github.com/dotnet/fsharp/blob/feature/tasks/tests/fsharp/perf/tasks/FS/taskSeq.fs).

This is for state machine compilation of computation expressions that generate `IAsyncEnumerable<'T>` values. This is a headline C# 8.0 feature and a very large feature for C#.  It appears to mostly drop out as library code once general-purpose state machine support is available.

## Example seq2 { ... }

See [seq2.fs](https://github.com/dotnet/fsharp/blob/feature/tasks/tests/fsharp/perf/tasks/FS/seq2.fs)

This is a resumable machine emitting to a mutable context held in a struct state machine. The state holds the current
value of the iteration.

This is akin to `seq { ... }` expressions, for which we have a baked-in state machine compilation in the F# compiler today. 

## Example: low-allocation list and array builders

See [list.fs](https://github.com/dotnet/fsharp/blob/feature/tasks/tests/fsharp/perf/tasks/FS/list.fs)

These are synchronous machines emitting to a mutable context held in a struct state machine.

The sample defines  `list { .. }`, `array { .. }` and `rsarray { .. }` for collections, where the computations generate directly into a `ResizeArray` (`System.Collections.Generic.List<'T>`).
The overall result is a `list { ... }` builder that runs up to 4x faster than the built-in `[ .. ]` for generated lists of
computationally varying shape (i.e. `[ .. ]` that use conditionals, `yield` and so on).

F#'s existing `[ .. ]` and `[| ... |]` and `seq { .. } |> Seq.toResizeArray` all use an intermediate `IEnumerable` which is then iterated to populate a `ResizeArray` and then converted to the final immutable collection. In contrast, generating directly into a `ResizeArray` is potentially more efficient (and for `list { ... }` further perf improvements are possible if we put this in `FSharp.Core` and use the mutate-tail-cons-cell trick to generate the list directly). This technique has been known for a while and can give faster collection generation but it has not been possible to get good code generation for the expressions in many cases. Note that, these aren't really "state machines" because there are no resumption points - there is just an implicit collection we are yielding into in otherwise synchronous code.

# Drawbacks

Complexity

# Alternatives

1. Don't do it.
2. Don't generalise it (just do it for tasks)

# Compatibility

This is a backward compatible addition.

# Technical note: From computation expressions to state machines

There is a strong relationship between F# computation expressions and state machines.  F# computation expressions compose functional fragments using `Bind`, `ReturnFrom` and other constructs.  With some transformation (e.g. inlining the `Bind` method and others) these quickly reduce to code similar to the inefficient implementation above. For example:

```fsharp
task {
    printfn "intro"
    let! x = task1()
    printfn "hello"
    let! y = task2()
    printfn "world"
    return x + y
}
```

becomes 

```fsharp
task.Run ( 
    task.Delay (fun () -> 
        printfn "intro"
        task.Bind(task1(), fun x -> 
            printfn "hello"
            task.Bind(task2(), fun y -> 
                printfn "world"
                task.Return (x+y)))))
```

Now the meaning of the above code depends on the definition of the builder `task`.  If we assume `Bind` is inlined to `GetAwaiter` and `AwaitOnCompleted` and `Run` ultimately accepts a `MoveNext` function then it is something like:

```fsharp
let rec state0() = 
    printfn "intro"
    let t = task1()
    let awaiter = t.GetAwaiter()
    if t.IsCompleted then 
        state1 (awaiter.GetResult())
    else
        awaiter.AwaitOnCompleted(fun () -> state1 (awaiter.GetResult()))

and state1(x) = 
    printfn "hello"
    let t = task2()
    let awaiter = t.GetAwaiter()
    if t.IsCompleted then 
        state2 (x, awaiter.GetResult())
    else
        awaiter.AwaitOnCompleted(fun () -> state2 (x, awaiter.GetResult()))

and state2(x, y) = 
        printfn "world"
        DONE (x + y)

task.Run ( fun () -> state0()))
```

However today there is no way to get the F# compiler to convert this functional code to more efficient resumable code.   Two main things need to be done. First the "state variables" are all lifted to be mutables, and the code is combined into a single method. The first step is this:

```fsharp
let mutable awaiter1 = Unchecked.defaultof<_>
let mutable xv = Unchecked.defaultof<_>
let mutable awaiter2 = Unchecked.defaultof<_>
let mutable yv = Unchecked.defaultof<_>
let rec state0() = 
    printfn "intro"
    awaiter1 <- task1().GetAwaiter()
    if awaiter1.IsCompleted then 
        state1 ()
    else
        awaiter1.AwaitOnCompleted(fun () -> state1 ())

and state1() = 
    xvar <- awaiter1.GetResult()
    printfn "hello"
    awaiter2 <- task2().GetAwaiter()
    if awaiter2.IsCompleted then 
        state2 ()
    else
        awaiter2.AwaitOnCompleted(fun () -> state2 ())

and state2() = 
    yvar <- awaiter2.GetResult()
    printfn "world"
    Task.FromResult(xvar + yvar)

task.Run ( task.Delay (fun () -> state0()))
```

then:

```fsharp
let mutable awaiter1 = Unchecked.defaultof<_>
let mutable xv = Unchecked.defaultof<_>
let mutable awaiter2 = Unchecked.defaultof<_>
let mutable yv = Unchecked.defaultof<_>
let mutable pc = 0
let next() =
    match pc with  
    | 0 -> 
        printfn "intro"
        awaiter1 <- task1().GetAwaiter()
        if awaiter1.IsCompleted then 
            pc <- 1
            return CONTINUE
        else
            pc <- 1
            awaiter1.AwaitOnCompleted(this)
            return AWAIT
    | 1 -> 
        xvar <- awaiter1.GetResult()
        awaiter2 <- task2().GetAwaiter()
        if awaiter2.IsCompleted then 
            pc <- 2
            return CONTINUE
        else
            pc <- 2
            awaiter2.AwaitOnCompleted(this)
            return AWAIT
    | 2 -> 
        printfn "world"
        return DONE (xvar + yvar)

task.Run (... return a task that repeatedly calls next() until AWAIT or DONE ...)
```

This is a sketch to demonstrate the progression from monadic computation expression code to compiled state machine code with integer state representations.

Note:

* the above kind of transformation is **not** valid for all computation expressions - it depends on the implementation details of the computation expression.  It also depends on doing this transformation for a finite number of related binds (i.e. doing it for all of a single `task { ... }` or other CE expression), which allows the use of a compact sequence of integers to represent the different states.   

* The transformation where values passed between states become "state machine variables" is also not always valid - this can extend the lifetime of values in subtle ways, though the state machine generation can also generally zero out the variables at appropriate points.

* If the computation expression contains conditional control flow (`if ... then... else` and `match` and `while` and `for`) then the AWAIT points can occur in the middle of generated code, and thus the initial `match` in the integer-based version can branch into the middle of control-flow.
 
* If the computation expression contains exception handling then these can sometimes be carefully implemented using the stack-based control-flow constructs in .NET IL and regular F# code.  However this must be done with care.

The heart of a typical state machine is a `MoveNext` or `Step` function that takes an integer `program counter` (`pc`) and jumps to a target:

```fsharp
    member __.MoveNext() = 
        match pc with
           | 1 -> goto L1 
           | 2 -> goto L2 
           | _ -> goto L0
        L0: ...
            ... this code can return, first setting "pc <- L1"...

        L1: ...
            ... this code can return, e.g. first setting "pc <- L2"...

        L2: ...

```

This is roughly what compiled `seq { ... }` code looks like in F# today and what compiled async/await code looks like in C#, at a very high level. Note that, you can't write this kind of code directly in F# - there is no `goto` and especially not a `goto` that can jump directly into other code, resuming from the last step of the state machine.  

# Unresolved questions

None


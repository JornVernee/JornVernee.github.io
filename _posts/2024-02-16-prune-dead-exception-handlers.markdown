---
layout: post
title:  "Pruning dead exception handlers"
date:   2024-02-16 14:14:42 +0100
categories: hotspot jit
---

In JDK 22 we resolved a performance issue that had been plaguing the FFM API for a while.
We were seeing excess allocations when using try-with-resources with freshly created `Arena`s,
which were caused by dead code in exception handlers (`catch` blocks) not being removed by the C2 JIT compiler.

We solved this for JDK 22: [8267532: C2: Profile and prune untaken exception handlers](https://github.com/openjdk/jdk/pull/16416)

The effect of this optimization is more broadly applicable to Java code using try-with-resources,
or just any untaken `catch` block. So I thought it would be interesting to discuss the issue,
and how it was solved, in this post.

By the way, 'pruning' in the title refers to the pruning you would do while gardening, where part of a
plant, such as a branch, is cut off. This is similar to how we 'cut off' dead branches in
code while JIT compiling.

## Why are my allocations escaping?

The use pattern where we ran into this issue is one that is fairly common when using the FFM API:

```java
try (Arena arena = Arena.ofConfined()) {
    MemorySegment segment = arena.allocateFrom("Hello!");
    func(segment);
}
```

This code is creating a new arena, allocating a some data in that arena, and then passing the
pointer to that data to a native function (`func`). Assume that the implementation
of `func` just forwards the call using a native method handle produced by `java.lang.foreign.Linker::downcallHandle`.

There are several allocations in this code: the `Arena` and the `MemorySegment` are the 
most prominent ones. However, since the implementation of `func` will only pass a primitive
address to the native function, none of the allocated objects escape to wider Java code,
and in theory C2 should be able to scalar replace these objects, avoiding their allocation altogether.

But, when verifying this using the techniques described in my other post: 
['Tracking down escaping objects'](https://jornvernee.github.io/hotspot/jit/2023/08/18/debugging-jit.html#5-tracking-down-escaping-objects)
it turned out that there were several escaping allocations. In JDK 21, this code has the
following escaping allocations:

```text
JavaObject(38) allocation in: MemorySessionImpl::createConfined @ bci:0 (line 145)
  -> Field(63)
  -> JavaObject(40)
  -> LocalVar(117)
  -> LocalVar(157)
  Reason: Escapes as argument to call to: jdk.internal.foreign.MemorySessionImpl$1::close void ( jdk/internal/foreign/MemorySessionImpl$1 (java/lang/AutoCloseable,java/lang/foreign/Arena,java/lang/foreign/SegmentAllocator):NotNull * ) TestArena::payload @ bci:36 (line 25)


JavaObject(39) allocation in: ConfinedSession::<init> @ bci:2 (line 55)
  -> Field(45)
  -> JavaObject(38)
  -> Field(63)
  -> JavaObject(40)
  -> LocalVar(117)
  -> LocalVar(157)
  Reason: Escapes as argument to call to: jdk.internal.foreign.MemorySessionImpl$1::close void ( jdk/internal/foreign/MemorySessionImpl$1 (java/lang/AutoCloseable,java/lang/foreign/Arena,java/lang/foreign/SegmentAllocator):NotNull * ) TestArena::payload @ bci:36 (line 25)


JavaObject(40) allocation in: MemorySessionImpl::asArena @ bci:0 (line 80)
  -> LocalVar(117)
  -> LocalVar(157)
  Reason: Escapes as argument to call to: jdk.internal.foreign.MemorySessionImpl$1::close void ( jdk/internal/foreign/MemorySessionImpl$1 (java/lang/AutoCloseable,java/lang/foreign/Arena,java/lang/foreign/SegmentAllocator):NotNull * ) TestArena::payload @ bci:36 (line 25)


JavaObject(41) allocation in: NativeMemorySegmentImpl::makeNativeSegment @ bci:112 (line 136)
  -> Field(67)
  -> JavaObject(39)
  -> Field(45)
  -> JavaObject(38)
  -> Field(63)
  -> JavaObject(40)
  -> LocalVar(117)
  -> LocalVar(157)
  Reason: Escapes as argument to call to: jdk.internal.foreign.MemorySessionImpl$1::close void ( jdk/internal/foreign/MemorySessionImpl$1 (java/lang/AutoCloseable,java/lang/foreign/Arena,java/lang/foreign/SegmentAllocator):NotNull * ) TestArena::payload @ bci:36 (line 25)
```

If you look at the escape routes of these objects, you'll notice that `JavaObject(38)` escapes
because `JavaObject(40)` escapes, `JavaObject(39)` escapes because `JavaObject(38)` escapes,
and `JavaObject(41)` escapes because `JavaObject(39)` escapes. Or, in other words: the entire
_graph_ of objects escapes together with `JavaObject(40)`, which escapes as an argument to
and out-of-line call to `jdk.internal.foreign.MemorySessionImpl$1::close`.

## Why is this call not being inlined?

So, why is there an out-of-line call here that is making our object graph escape? Looking at
the [inlining trace](https://jornvernee.github.io/hotspot/jit/2023/08/18/debugging-jit.html#3-printing-inlining-traces),
we find that... the call is being inlined?

```text
TestArena::payload (53 bytes)
@ 22   jdk.internal.foreign.MemorySessionImpl$1::close (8 bytes)   inline (hot)
  @ 4   jdk.internal.foreign.MemorySessionImpl::close (12 bytes)   inline (hot)
    @ 1   jdk.internal.foreign.ConfinedSession::justClose (52 bytes)   inline (hot)
     ...
```

No wait... it's not?

```text
@ 36   java.lang.foreign.Arena::close (0 bytes)   virtual call
@ 36   jdk.internal.foreign.MemorySessionImpl$1::close (8 bytes)   low call site frequency
```

Ah right! There are 2 calls to `close`: one for the path without an exception, and one for
the path with an exception (in the 'catch block'). This is the bytecode `javac` generates 
for the Java code:

```text
Code:
 0: invokestatic  #12                 // InterfaceMethod java/lang/foreign/Arena.ofConfined:()Ljava/lang/foreign/Arena;
 3: astore_0
 4: aload_0
 5: ldc           #18                 // String Hello!
 7: invokeinterface #20,  2           // InterfaceMethod java/lang/foreign/Arena.allocateUtf8String:(Ljava/lang/String;)Ljava/lang/foreign/MemorySegment;
12: astore_1
13: aload_1
14: invokestatic  #24                 // Method func:(Ljava/lang/foreign/MemorySegment;)V
17: aload_0
18: ifnull        52
21: aload_0
22: invokeinterface #28,  1           // InterfaceMethod java/lang/foreign/Arena.close:()V
27: goto          52
30: astore_1
31: aload_0
32: ifnull        50
35: aload_0
36: invokeinterface #28,  1           // InterfaceMethod java/lang/foreign/Arena.close:()V
41: goto          50
44: astore_2
45: aload_1
46: aload_2
47: invokevirtual #33                 // Method java/lang/Throwable.addSuppressed:(Ljava/lang/Throwable;)V
50: aload_1
51: athrow
52: return
Exception table:
   from    to  target type
       4    17    30   Class java/lang/Throwable
      35    41    44   Class java/lang/Throwable
```

We see two calls to `close`: one at bytecode index (bci) `22` and one at `36`. This is a
consequence of how `finally` blocks are translated by `javac`. The code in a `finally`
block is essentially copy pasted along the non-exception path and the exception path (in the exception handler).
If we look at the exception table, we see that bci 36 is inside an exception handler. This
all checks out so far.

So, if we look back at our inlining trace, we see that the call to `close` along the normal
exception-less path (`@ 22`) is being inlined as expected, but the call to `close` in the exception
handler (`@ 36`) is not being inlined due to 'low call site frequency'. If we reference this back
to the escaping allocations, we see that it is this call in the exception handler at bci 36
into which the object graph escapes:

```text
Reason: Escapes as argument to call to: jdk.internal.foreign.MemorySessionImpl$1::close ... TestArena::payload @ bci:36 (line 25)
```

C2 will not inline calls that are hardly ever reached (low frequency), since these calls
are likely not on the hot path, and therefore there would be less benefit from inlining.
In our specific case, the call site of `close` in the exception handler is _never_ reached
however, because an exception is never thrown. But, this 'dead' code is still interfering
with other optimizations, such as scalar replacement. The dead code is not outright removed
because there is still a chance that an exception might occur some time in the future, so
the code C2 generates has to account for that possibility.

However, C2 also has a way to deal with code that is never reached _in practice_: uncommon traps.
An uncommon traps replaces a piece of code that is very unlikely to be needed (based on 
profiling information), with a _trap_ that deoptimizes the code and continues running in
the interpreter. C2 essentially bets that this code is not needed, so it's not
worth optimizing for. The fact that this is possible is one of the strengths of a mixed
mode VM that can both interpret and JIT compile code: the JIT can speculate and go back to
the interpreter in the worst case. Uncommon traps are for instance used to prune unlikely
branches of `if`/`else` statements.

An uncommon trap can also re-inflate objects that have been scalar replaced, so they do not
interfere with the scalar replacement optimization, even though an uncommon trap may 'use'
an allocated object. So, in theory, it should be possible for C2 to replace the exception
handler of our try-with-resources block with an uncommon trap (since we never enter the 
exception handler). This should then allow our object graph to be scalar replaced.

## Why is there no uncommon trap?

For 'regular' branches, such as the branches of an `if` statement, the VM profiles these
branches by counting how many times a branch is taken. The JIT then uses this count, together
with the invocation count of the enclosing method, to determine how frequent this branch is
taken. If a branch is heuristically deemed to be 'rarely' taken, that branch is replaced with
an uncommon trap.

We could say that exception handlers are a type of branch, where the branch can be entered
from any point in the code that the exception handler covers. So, we should be able to profile
those as well, right? Well, most profiling works based on the byte code that's being executed.
For example, for `if` statements, `goto` bytecodes are used. When a `goto` bytecode is executed,
we also do some branch profiling. This is both simple and fast. However, since exception handlers
can start with any bytecode, we can not apply the same strategy: we can't tell when executing
e.g. an `iconst_0` bytecode whether it's the first bytecode of an exception handler _just
by looking at the bytecode_. Perhaps we could keep a table of all the first bcis of the exception
handlers in the method we are executing, and if the bci we execute is one of them, do the profiling.
But, this would slow down the execution of _all_ bytecodes, for something that is supposed to
be 'exceptional', i.e. happen rarely. This doesn't sound like a good deal, and I suspect one
of the main reasons why profiling of exception handlers wasn't done sooner.

But, when an exception is thrown, we go through a runtime call (an out-of-line call to some
C++ code) to look up the exception handler. We can just put the profiling code in that
runtime call, under the assumption that we only need to look up an exception handler when
we are actually going to execute it. That is also what we ended up doing: whenever we look
up an exception handler, we mark that handler as 'entered', and that is our profiling.
When C2 parses an exception handler, it can now check if that exception handler was ever
entered, and if not, insert an uncommon trap instead of the exception handler. Besides
that, we also need to mark an exception handler as entered when we deoptimize through this
uncommon trap, so that if an exception is thrown after all, we don't try to insert another
uncommon trap the next time that the code is compiled (since exceptions are evidently a
possibility after all).

## Success!

We side step the issue of an out-of-line call into which our objects escape, by replacing
the branch that the call is in with an uncommon trap. As a result, most of our objects no
longer escape:

```text
JavaObject(31) allocation in: SegmentFactories::allocateSegment @ bci:100 (line 158)
  Reason: MergedWithObject[other=JavaObject(1) [ [ ]]    128  ConP  === 0  [[ 216 210 209 213 643 212 59 208 214 1178 167 211 151 503 165 508 166 138 ]]  #null]
```

Only the `MemorySegment` escapes, for another reason (which I have a potential
fix in mind for as well).

For use cases like our little code sample, this cuts the amounts of bytes allocated in half,
and potentially makes the execution twice as fast. See for instance the numbers from the
benchmark included in the linked pull request:

Before:

```text
Benchmark                                                      Mode  Cnt      Score     Error   Units
ResourceScopeCloseMin.confined_close                           avgt   30     10.458 ±   0.070   ns/op
ResourceScopeCloseMin.confined_close:gc.alloc.rate.norm        avgt   30    104.000 ±   0.001    B/op
```

After:

```text
Benchmark                                                      Mode  Cnt      Score     Error   Units
ResourceScopeCloseMin.confined_close                           avgt   30      4.563 ±   0.043   ns/op
ResourceScopeCloseMin.confined_close:gc.alloc.rate.norm        avgt   30     56.000 ±   0.001    B/op
```

The great thing is that this doesn't just help the FFM API, but potentially helps any code
that uses exception handlers, such as try-with-resources, or `catch`. So, if you have any 
use-cases on the hot paths of your code that have untaken exception handlers, keep an eye
out for any performance improvements coming in JDK 22!

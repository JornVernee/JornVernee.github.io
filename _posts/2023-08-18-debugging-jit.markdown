---
layout: post
title:  "Debugging Java JIT compilation"
date:   2022-05-01 14:14:42 +0100
categories: jit debugging
---

Typically when you compile a Java program, it is first compiled into bytecode by a Java compiler. However, this bytecode
is not optimized yet. In HotSpot (OpenJDK's JVM) this instead happens at runtime, and is done by a JIT, a Just-In-Time
compiler. This way of doing things allows the JIT to take maximum advantage of the conditions that the code is running
in, such as hardware, and even to a certain degree the specific data that is fed into the program.

Understanding what the JIT compilers do (yes, there are multiple), is important when trying to understand the performance
characteristics of a Java program. In this blog post I will discuss several techniques that I use to do this. I will
cover getting basic traces for a particular compilation, as well as actual debugging using a debugger tool.

## Setting the stage

First, let's set up a small test project that we can easily modify to test different snippets of Java code. Our goal
is for to be able to trigger a JIT compilation for a particular piece of Java code, so that we can use some of
HotSpot's options for debugging the compilation.

I define a 'payload' method, which will hold the code that we wish to JIT compile. Then, I trigger the JIT compilation
of this payload simply by invoking the method a bunch of times. Here is the code:

```java
public class TestJIT {
    public static void main(String[] args) {
        for (int i = 0; i < 20_000; i++) {
            payload();
        }
    }

    public static void payload() {
        // test code here
    }
}
```

Tier 4 JIT compilation, done by the C2 JIT compiler, which is the highest/most optimized tier, happens after 10K
invocations on x86. The number can be found in the `./src/hotspot/cpu/x86/c2_globals.hpp` file as `CompileThreshold`
[1](https://github.com/openjdk/jdk/blob/752121114f424d8e673ee8b7bb85f7705a82b9cc/src/hotspot/cpu/x86/c1_globals_x86.hpp#L41).
I'm invoking the payload 20K times here to be on the safe side, since the invocation counter is not necessarily 100% accurate.

I just compile the program with `javac`. Then, when running it, there are a few important flags to pass. First:
`-XX:CompileCommand=dontinline,TestJIT::payload`. This flag disables inlining of the `payload`. Doing this is important
in order to get a standalone compilation of the `payload` method, which is required to be able to inspect the compilation
of that particular method in isolation from the loop in `main`.

The other important flag we need to pass is `-Xbatch`. JIT compilation is by default done on a background thread. This
means that your code can just keep running while the compilation happens, but in our case it also means that the code
might finish running before the compilation is done, meaning we would not be able to debug it. `-Xbatch` makes it so the
thread that requests a compilation is stopped while the compilation is happening. FWIW, `-Xbatch` is an alias for
`-XX:-BackgroundCompilation`, i.e. turning off background compilation [2](https://github.com/openjdk/jdk/blob/752121114f424d8e673ee8b7bb85f7705a82b9cc/src/hotspot/share/runtime/globals.hpp#L272-L274)

If you run the test program with these 2 flags (and any other needed flags), the output is not very interesting yet:

```sh
> java -cp classes -XX:CompileCommand=dontinline,TestJIT::payload -Xbatch TestJIT
CompileCommand: dontinline TestJIT.payload bool dontinline = true
```

Next, let's see how we can start using this test setup to get interesting information out the compilation of the payload
method.

## Getting the disassembly of a compiled method



## Thanks for reading

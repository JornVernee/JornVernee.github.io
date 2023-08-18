---
layout: post
title:  "Debugging Java JIT compilation"
date:   2023-08-18 14:14:42 +0100
categories: jit debugging
---

Typically when you compile a Java program, it is first compiled into bytecode by a Java compiler. However, this bytecode
is not optimized yet. In HotSpot (OpenJDK's JVM) this instead happens at runtime, and is done by a JIT, a Just-In-Time
compiler. This way of doing things allows the JIT to take maximum advantage of the conditions that the code is running
in, such as hardware, and even to a certain degree the specific data that is fed into the program.

Understanding what the JIT compilers do is important when trying to understand the performance characteristics of a Java
program. In this blog post I will discuss several techniques that I use to do this. I will cover getting basic traces for
a particular compilation, as well as actual debugging using a debugger tool.

Note that I will be focussed on the HotSpot JVM particulars, and will _not_ be explaining how to read assembly code, or
how certain optimizations work, other than a brief description. If you wish to know more about a certain topic, you'll 
have to research it yourself. But, maybe this blog post can give a few ideas on where to start.

1. [Setting the stage](#1-setting-the-stage)
2. [Getting the assembly of a compiled method](#2-getting-the-assembly-of-a-compiled-method)
3. [Getting inlining traces](#3-getting-inlining-traces)

## 1. Setting the stage

First, let's set up a small test project that we can easily modify to test different snippets of Java code. Our goal
is for to be able to trigger a JIT compilation for a particular piece of Java code, so that we can use some of
HotSpot's options for debugging the compilation.

I define a 'payload' method, which will hold the code that we wish to JIT compile. Then, we trigger the JIT compilation
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

```powershell
> java -cp classes -XX:CompileCommand=dontinline,TestJIT::payload -Xbatch TestJIT
CompileCommand: dontinline TestJIT.payload bool dontinline = true
```

Next, let's see how we can start using this test setup to get interesting information out the compilation of the payload
method.

## 2. Getting the assembly of a compiled method

The JIT compilers output machine code, which is, let's say, a hard to read format. Luckily this machine code can be
dis-assembled into a more readable format where each CPU instruction is represented with a mnemonic. To do that, a dis-
assembler plugin for HotSpot is needed called `hsdis`. You can find instructions on how to build hdis through the [OpenJDK
build instructions](https://github.com/openjdk/jdk/blob/master/src/utils/hsdis/README.md), or through my [earlier post on
the topic]({% post_url 2022-04-30-hsdis %}). I recommend using the capstone-based hsdis, as it's easiest to build,
and also what I'm using. I've put the hsdis library file on the `PATH`, where it is findable by HotSpot.

Now we just run the program again with a couple of additional VM flags to print out the assembly of the `payload` method.
I'm using `-XX:CompileCommand=print,TestJIT::payload` to print out the assembly, and I'm also adding `-XX:-TieredCompilation`
which disables compilation by the HotSpot C1 compiler. This latter flag is used to reduce the amount of output. In this
case I'm only interested in the assembly generated by C2 JIT, which is the more optimized version. Conversely, if you're
only interested in the assembly generated by C1, you can use `-XX:TieredStopAtLevel=3` which disables tier 4 compilation,
i.e. C2. Lastly, I prefer the intel assembly syntax, so I also pass `-XX:PrintAssemblyOptions=intel`. This last flag is
diagnostic, so we also have to pass `-XX:+UnlockDiagnosticVMOptions` _before_ the `PrintAssemblyOptions` flag. Putting
it all together, we run the program as follows:

```powershell
java `
  -cp classes `
  -Xbatch `
  -XX:-TieredCompilation `
  -XX:CompileCommand=dontinline,TestJIT::payload `
  -XX:CompileCommand=print,TestJIT::payload `
  -XX:+UnlockDiagnosticVMOptions `
  -XX:PrintAssemblyOptions=intel `
  TestJIT
```

Note that I'm using powershell. The ticks are the powershell equivalent of `\` in a unix shell. I've put this command
in a script file to make it easier to edit and re-run.

The output generated is architecture specific. I'm using an x64 machine, which gives me x64 assembly. If you are using an
AArch64 machine, such as Apple's M1 chips, the output will be different. The output I get for my test program above as follows:

```text
CompileCommand: dontinline TestJIT.payload bool dontinline = true
CompileCommand: print TestJIT.payload bool print = true

============================= C2-compiled nmethod ==============================
----------------------------------- Assembly -----------------------------------

Compiled method (c2)      54   13             TestJIT::payload (1 bytes)
 total in heap  [0x000001c6bacb0a90,0x000001c6bacb0ca0] = 528
 relocation     [0x000001c6bacb0bd8,0x000001c6bacb0be8] = 16
 main code      [0x000001c6bacb0c00,0x000001c6bacb0c50] = 80
 stub code      [0x000001c6bacb0c50,0x000001c6bacb0c68] = 24
 oops           [0x000001c6bacb0c68,0x000001c6bacb0c70] = 8
 scopes data    [0x000001c6bacb0c70,0x000001c6bacb0c78] = 8
 scopes pcs     [0x000001c6bacb0c78,0x000001c6bacb0c98] = 32
 dependencies   [0x000001c6bacb0c98,0x000001c6bacb0ca0] = 8

[Disassembly]
--------------------------------------------------------------------------------
[Constant Pool (empty)]

--------------------------------------------------------------------------------

[Verified Entry Point]
  # {method} {0x000001c6d88002e8} 'payload' '()V' in 'TestJIT'
  #           [sp+0x20]  (sp of caller)
  0x000001c6bacb0c00:   sub             rsp, 0x18
  0x000001c6bacb0c07:   mov             qword ptr [rsp + 0x10], rbp
  0x000001c6bacb0c0c:   cmp             dword ptr [r15 + 0x20], 0
  0x000001c6bacb0c14:   jne             0x1c6bacb0c43
  0x000001c6bacb0c1a:   add             rsp, 0x10
  0x000001c6bacb0c1e:   pop             rbp
  0x000001c6bacb0c1f:   cmp             rsp, qword ptr [r15 + 0x378]
                                                            ;   {poll_return}
  0x000001c6bacb0c26:   ja              0x1c6bacb0c2d
  0x000001c6bacb0c2c:   ret
  0x000001c6bacb0c2d:   movabs          r10, 0x1c6bacb0c1f  ;   {internal_word}
  0x000001c6bacb0c37:   mov             qword ptr [r15 + 0x390], r10
  0x000001c6bacb0c3e:   jmp             0x1c6bac7ad80       ;   {runtime_call SafepointBlob}
  0x000001c6bacb0c43:   call            0x1c6bac586e0       ;   {runtime_call StubRoutines (2)}
  0x000001c6bacb0c48:   jmp             0x1c6bacb0c1a
  0x000001c6bacb0c4d:   hlt
  0x000001c6bacb0c4e:   hlt
  0x000001c6bacb0c4f:   hlt
[Exception Handler]
  0x000001c6bacb0c50:   jmp             0x1c6baca3f00       ;   {no_reloc}
[Deopt Handler Code]
  0x000001c6bacb0c55:   call            0x1c6bacb0c5a
  0x000001c6bacb0c5a:   sub             qword ptr [rsp], 5
  0x000001c6bacb0c5f:   jmp             0x1c6bac7a020       ;   {runtime_call DeoptimizationBlob}
  0x000001c6bacb0c64:   hlt
  0x000001c6bacb0c65:   hlt
  0x000001c6bacb0c66:   hlt
  0x000001c6bacb0c67:   hlt
--------------------------------------------------------------------------------
[/Disassembly]
```

The thing to focus on is the `[Disassembly]` section of the output. Even though our `payload` method is empty, there is
still quite a bit of code generated by the JIT. We are of course running in a virtual machine, and there is some additional
code needed to make it work. If we're just interested in the assembly generated for the contents of the `payload`
method, then all of the above is irrelevant. After all, the `payload` method is empty, so the above code is purely auxiliary
code. However, I will walk through it once so that we know which parts we can typically ignore:

Setting up the stack frame. Allocates a bit of memory on the thread's stack, and saves the contents of the `rbp` register
to the stack.

```text
0x000001c6bacb0c00:   sub             rsp, 0x18
0x000001c6bacb0c07:   mov             qword ptr [rsp + 0x10], rbp
```

NMethod entry barrier. 'nmethod' is the name for a compiled Java method in HotSpot. The nmethod entry barrier is need to
make some GCs work. I won't get into that right now:

```text
0x000001c6bacb0c0c:   cmp             dword ptr [r15 + 0x20], 0
0x000001c6bacb0c14:   jne             0x1c6bacb0c43
```

Cleaning up the frame:

```text
0x000001c6bacb0c1a:   add             rsp, 0x10
0x000001c6bacb0c1e:   pop             rbp
```

Safepoint poll. The VM needs threads to occasionally poll for safe points. At a safe point, the JVM state for the current
thread is fully known and recoverable. This is a point in the code where the VM might want to inspect the thread to do
various VM operations.

```text
0x000001c6bacb0c1f:   cmp             rsp, qword ptr [r15 + 0x378]
                                                            ;   {poll_return}
0x000001c6bacb0c26:   ja              0x1c6bacb0c2d
```

Return instruction:

```text
0x000001c6bacb0c2c:   ret
```

I'm going to ignore the rest for now. The important part is that the code for the contents of the `payload` method will
be primarily found in this block, between the nmethod entry barrier and the cleanup of the frame. A good strategy when trying
to find the generated code for a snippet is to look for the nmethod entry barrier and work forwards from there, or to look
for the return instruction and work backwards from there.

Let's modify our `payload` method and see what happens to the generated assembly. I'm going to change my payload method to
add two numbers together and return the result:

```java
public static int payload(int a, int b) {
    return a + b;
}
```

I just call it with some dummy arguments in the `main` method. The values don't really matter since we have disabled inlining
of the `payload` method, so the JIT compiler will not be able to 'see' the actual value of the arguments, and has to assume
that they could be anything. For similar reasons, it is safe to discard the return value in the `main` method:

```java
for (int i = 0; i < 20_000; i++) {
    payload(1, 2);
}
```

Recompiling with `javac`, and re-running the program with the flags above, gets me this bit of assembly:

```text
  # {method} {0x0000016af6c002f0} 'payload' '(II)I' in 'TestJIT'
  # parm0:    rdx       = int
  # parm1:    r8        = int
  #           [sp+0x20]  (sp of caller)
  0x0000016ad9230c00:   sub             rsp, 0x18
  0x0000016ad9230c07:   mov             qword ptr [rsp + 0x10], rbp
  0x0000016ad9230c0c:   cmp             dword ptr [r15 + 0x20], 0
  0x0000016ad9230c14:   jne             0x16ad9230c47
  0x0000016ad9230c1a:   lea             eax, [rdx + r8]
  0x0000016ad9230c1e:   add             rsp, 0x10
  0x0000016ad9230c22:   pop             rbp
  0x0000016ad9230c23:   cmp             rsp, qword ptr [r15 + 0x378]
                                                            ;   {poll_return}
  0x0000016ad9230c2a:   ja              0x16ad9230c31
  0x0000016ad9230c30:   ret
```

I've just copied the relevant bits here, starting from the method entry to the return instruction. Note the first few lines,
which tells us in which registers the arguments are being passed. The HotSpot JITs use a custom calling convention for
Java methods. So, this can be different from the calling convention used by e.g. C:

```text
  # parm0:    rdx       = int
  # parm1:    r8        = int
```

The assembly for the code `return a + b;` can be found between the nmethod entry barrier and the frame cleanup:

```text
  0x0000016ad9230c1a:   lea             eax, [rdx + r8]
```

A clever way of adding two values together in a single instruction, and storing the result in the `eax` register, which
is the register in which integer values are returned in the Java compiled calling convention.

And that's really all there is to it! Now you should have to basic skills needed to start analysing the machine code
generated by the JIT compilers for a particular snippet of Java code.

A thing to note here is that this assembly was generated with a release build, i.e. the ones that you can download from
[jdk.java.net](https://jdk.java.net/). There are also 'fastdebug' and 'slowdebug' versions of HotSpot. Using a debug build
can result in more detailed information being printed when printing assembly. If you can get your hands on a fastdebug
build, I recommend using at least that when printing assembly. The most straightforward way of getting a fastdebug build
is building the JDK yourself. This is relatively easy, if you follow the steps in the [build guide](https://github.com/openjdk/jdk/blob/master/doc/building.md).
For a fastdebug build, just make sure to configure the build with `--with-debug-level=fastdebug`.

## 3. Getting inlining traces

The next useful bit of information we can get from a JIT compiler is an inlining trace. This trace indicates whether 
methods being called by the compiled Java code were inlined. Inlining is an important optimization that allows other
optimizations to take place, so understanding which methods are inlined can be important when trying to understand the
performance of a snippet of Java code. While this information is technically present in the assembly as well, the inlining
trace gives a much nicer high-level overview.

To start, let's modify our payload method to call another method:

```java
public static int yetAnotherMethod() {
    return 42;
}

public static int otherMethod() {
    return yetAnotherMethod();
}

public static int payload() {
    return otherMethod() + 1;
}
```

We can generate an inlining trace for this compilation using another `CompileCommand` option. Instead of using 
`-XX:CompileCommand=print,TestJIT::payload` to print the assembly, I'm going to change the `print` command in that flag
to `PrintInlining`, which will give us an inlining trace: `-XX:CompileCommand=PrintInlining,TestJIT::payload`. The output
I get is simply:

```text
                            @ 0   TestJIT::otherMethod (4 bytes)   inline (hot)
                              @ 0   TestJIT::yetAnotherMethod (3 bytes)   inline (hot)
```

The format of an inlining trace is not really documented, unfortunately. The best source of information to understanding
it is the source code found in [`CompileTask::print_inlining_inner`](https://github.com/openjdk/jdk/blob/bcba5e97857fd57ea4571341ad40194bb823cd0b/src/hotspot/share/compiler/compileTask.cpp#L412). Each line in the inlining trace will indicate a method that was either inlined successfully,
or a method which failed to be inlined. Currently this is not indicated very well (something I'm hoping to change in the future),
and we have to interpret the message at the end of the line to decide whether inlining succeeded or failed. In this case,
the message is `inline (hot)`, from which we can ascertain that inlining succeeded. The line also lists the name of the
method which was inlined, and the bytecode location at which the method call is located in the caller method. In this case,
the call to `otherMethod` in `payload` is located at bci (byte code index) `0`, and the `yetAnotherMethod` method call in
`otherMethod` is also located at bci `0`. The indentation of the lines indicates the 'level' of inlining that occurred.
Since the `yetAnotherMethod` method is inlined transitively through `otherMethod`, its line indented by 2 extra spaces.

Let's see what happens if we disable inlining of `otherMethod` using the `-XX:CompileCommand=dontinline,TestJIT::otherMethod`
flag. Now the inlining trace looks like this:

```text
                            @ 0   TestJIT::otherMethod (4 bytes)   disallowed by CompileCommand
```

That should cover the basics of inlining traces.

## 4. A closer look at compile commands

## Thanks for reading

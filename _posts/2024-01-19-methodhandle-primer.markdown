---
layout: post
title:  "MethodHandle primer"
date:   2024-01-19 14:14:42 +0100
categories: methodhandles
---

The method handles API in the `java.lang.invoke` package is a powerful reflection and code
generation API that has extensive JIT support as well. In this blog post I will give an 
overview of this API. I will go over some of the most commonly used parts of the API,
but this is not a comprehensive guide. It's meant to give you a starting point from which
to start learning more on your own.

1. [What is a `MethodHandle`?](#what-is-a-methodhandle)
    1. [Access checks](#access-checks)
    2. [Exception handling](#exception-handling)
    3. [Signature polymorphism](#signature-polymorphism)
    4. [The `invokeExact` method](#the-invokeexact-method)
    5. [The `invokeWithArguments` method](#the-invokewitharguments-method)
2. [Method handle inlining](#method-handle-inlining)
3. [Method handle combinators](#method-handle-combinators)
    1. [`MethodHandles::insertArguments`](#methodhandlesinsertarguments)
    2. [`MethodHandles::filterArguments`](#methodhandlesfilterarguments)
    3. [`MethodHandles::collectArguments`](#methodhandlescollectarguments)
    4. [`MethodHandles::permuteArguments`](#methodhandlespermutearguments)
4. [Appendix A: `MutableCallSite`](#appendix-a-mutablecallsite)
5. [Appendix B: `VarHandle`](#appendix-b-varhandle)

## What is a `MethodHandle`?

A `java.lang.invoke.MethodHandle` is an object that wraps a fixed Java _target method_.
It is, as the name says, a 'handle' for a Java method. The target method can be invoked
through the method handle object.

One of the simplest ways to create a `MethodHandle` for a particular Java method, is to perform
a method handle lookup. First, we have to create a `MethodHandles.Lookup` object. This can
be done using the [`MethodHandles::lookup`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/invoke/MethodHandles.html#lookup())
factory method. Then, we can use one of the `findXXX` methods in `Lookup` to look up an existing
Java method, as demonstrated by the following test program:

```java
import java.lang.invoke.*;

public class Main {
    public static void main(String[] args) throws Throwable {
        MethodHandles.Lookup lookup = MethodHandles.lookup();
        MethodType typeOfTarget = MethodType.methodType(void.class);
        MethodHandle targetMh = lookup.findStatic(Main.class, "target", typeOfTarget);

        targetMh.invoke(); // prints 'invoking target'
    }

    public static void target() {
        System.out.println("invoking target");
    }
}
```

Here we have declared a static `target` method, which we can then look up with `findStatic`.
We pass the class holding the `target` method, its name, and its type represented as a
`java.lang.invoke.MethodType`, to `findStatic`. This gives us back a `MethodHandle` for the
`target` method, which we then invoke by calling `invoke`. Note also that the `invoke` method
throws a `Throwable`, so we handle that by declaring it in the `throws` clause of the `main`
method.

The other `findXXX` methods in `Lookup` can be used to look up other kinds of methods.
The `findVirtual` method corresponds to the `invokevirtual` bytecode, and can be used to
look up instance (i.e. non-static) methods, also known as _virtual methods_.
`findConstructor` can be used to look up constructors. There's also `find(Static)Getter`
and `find(Static)Setter` for reading and writing to fields. Note that these do _not_ look up
a `getX` or `setX` method, but rather look up a notional method that gets or sets a field
directly. It is like a method to get or set the field is generated on the fly. These
correspond the `putfield`, `getfield`, `putstatic`, and `getstatic` bytecodes. These are
just a few examples.

You might have realized that a `MethodHandle` is very similar in concept to a 
`java.lang.reflect.Method`. However, the API is more minimal, as a method handle doesn't
reflect the access modifiers of the target method, or other things such as annotations.
A method handle is really only 2 things: it has a type, and you can invoke it. The rest
of the methods in `MethodHandle` are so-called `combinators`, which allow a client to
create other method handles based on this one. We will discuss combinators in a later chapter.

Like `j.l.r.Method`, a method handle can be used, in the simplest of use cases, to
reflectively invoke a Java target method. There are however also some key difference between
the 2 APIs.

### Access checks

The `invoke` method of a `j.l.r.Method` is `@CallerSensitive`, and will perform
access checks when invoking it. A `MethodHandle` on the other hand, doesn't do any access
checks when it is invoked. Instead, the access checks are preformed when looking up the 
method handle. The access context is described by the lookup object instead. You'll note
that the `MethodHandles::lookup` method is caller sensitive instead. Wherever you call `lookup`,
that's the access context you capture. This allows a client to do a single access check when
the method handle is created, and after that get better performance by avoiding the access
checks when invoking the method handle.

💡This also means that, if you have the method handle object, you have the capability to
invoke it. Thus, we say that a method handle is a 'capability'. Through sharing the method 
handle object, this capability can be shared with other code that might not normally have
access to the target method, in order to allow that code to invoke the target method. This
is one of the reasons why you might want to use method handles.

### Exception handling

The `invoke` method of a `j.l.r.Method` wraps exceptions thrown by the underling Java method
into an `InvocationTargetException`, while the `invoke` method of a `MethodHandle` will
directly propagate the exception without wrapping. That is also why `invoke` throws a `Throwable`.
This accounts for any possible throwable that needs to be propagated from the target method.

The fact that `invoke` throws a `Throwable` might seem cumbersome, since we need to 
deal with this `Throwable` somehow. However, if the target method does not declare any 
checked exceptions, we can assume that `invoke` will also never throw a checked exception
in practice. So, in most cases we can just wrap the `invoke` call in a `try`/`catch` block
that throws an unchecked exception in the catch block, such as:

```java
MethodHandle mh = ...
try {
    mh.invoke();
} catch (Throwable t) {
    throw new RuntimeException("Should not happen", t);
}
```

💡The fact that `invoke` doesn't wrap thrown exceptions also means that a method
handle can be used to cleanly implement a method without having to worry about 
un-wrapping thrown exceptions in order to propagate them. This is another reason why you
might want to use method handles.

### Signature polymorphism

The `invoke` method of `j.l.r.Method` is declared as follows (some details omitted):

```java
@CallerSensitive
public Object invoke(Object obj, Object... args)
    throws IllegalAccessException, InvocationTargetException
```

The first parameter represents the receiver instance of a virtual method call. It is ignored
for static methods.

At first glance, the `invoke` method of `MethodHandle` may look very similar:

```java
public final native @PolymorphicSignature Object invoke(Object... args) throws Throwable;
```

However, the return type and parameter type of this declaration are completely fictional.
As the `@PolymorphicSignature` annotation implies, this method is [_signature polymorphic_](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/invoke/MethodHandle.html#sigpoly).

This method is so special, its even special-cased in the Java language specification.
A signature polymorphic method is a method whose type is determined not by the method 
declaration, but by the method _call site_ (the place in the code where the method is invoked).
In other words, whatever argument types the code
calling the `invoke` method passes to the call, those are the parameter types that the method will have.
Since we can call `invoke` many times, passing different numbers and types of arguments,
this also means that the `invoke` method essentially has many different types. In fact,
the `invoke` method can take on any type that is declarable by a normal Java method declaration.
The signature is thus: _polymorphic_. The same applies to the return type. Whatever type we
cast the return value of the `invoke` method to, that is the return type the method will have.

Calling the `j.l.r.Method::invoke` method requires creating a new `Object[]`, storing all
the argument values into it, requiring us to box primitive values. The `Object[]` is then
passed to the call, and the returned `Object` value has to be unboxed again if the value
is a primitive. Signature polymorphism on the other hand, allows us to avoid all this overhead.
Instead, all argument values are passed to, and returned from a sig-poly method _as is_.

This behavior is reified in the class file as well. The method type attached to the class file
reference pointing to the `MethodHandle::invoke` method, is the
exact (static) types of the argument values and return value, as parameter and return types.
For example, if I have a piece of code like this:

```java
MethodHandle mh = ...
String x = ...
int y = ...
long result = (long) mh.invoke(x, y);
```

Then the `invokevirtual` instruction that `javac` generates for this call to `invoke`, will
point to a method type descriptor like `(Ljava/lang/String;I)J`. This is different than the
method type descriptor of the declaration of the `invoke` method, which would be `([Ljava/lang/Object;)Ljava/lang/Object;`.

You can read more about this in the JLS, for instance
in section [15.12.3](https://docs.oracle.com/javase/specs/jls/se21/html/jls-15.html#jls-15.12.3).

To make sig-poly methods work, the VM will generate some code every time that a sig-poly
method is linked, and this will happen once for each type that a sig-poly method
takes on. This linking process involves spinning both machine code, and usually Java bytecode,
to link the method handle call to the actual target method using the right invocation semantics,
(depending on the target method e.g. static vs. virtual). Another way to think about it is
that, a sig-poly method can have many different implementations. Though, these implementations
are actually just small trampolines that forward the call to the actual target method
that is described by the `MethodHandle` instance.

💡The fact that sig-poly methods accept arguments as-is, without needing to box them up into
an `Object[]`, is another good reason to use method handles. If you want a type that represents
a piece of invocable code, but you aren't sure what the exact types or number of arguments
will be, then method handles are a good option. They provide a generic invocation API,
without the inefficiency associated with boxing up arguments and return values.

### The `invokeExact` method

Because a call to a sig-poly method passes the arguments as is, the bytecode generated for
the call to `invoke` will not, for instance, automatically convert an argument of type `int`
to `long` if that is the type of the parameter of the target method. However, the implementation
of the `invoke` method takes care of
that for us automatically. The type of the `invoke` method used at a call site does not
have to match the type of the method handle. The implementation of the `invoke` method
will convert all the arguments and the return value to the type that the target method expects.

The `invokeExact` method on the other hand, does not do any automatic argument conversions.
Instead, the implementation of `invokeExact` will check that the type used at the call site
matches the type of the method handle instance exactly. If there is a mismatch, a `WrongMethodTypeException`
is thrown. When I say that the types need to match exactly, I really do mean _exactly_.
If a method handle instance has a parameter of type `Object`, and I pass a value whose static
type is `String`, I will get a `WrongMethodTypeException`. To get the right type
at the call site, I would need to cast the argument value to object.

```java
static void foo(Object o) {}

public static void main(String[] ___) throws Throwable {
    MethodHandle fooMh = MethodHandles.lookup().findStatic(Main.class, "foo",
            MethodType.methodType(void.class, Object.class));
    
    fooMh.invokeExact("Hello, world!");
}
```

This code throws:

```text
java.lang.invoke.WrongMethodTypeException: handle's method type (Object)void but found (String)void
```

In order to make the call succeed, I have to cast the argument to `Object`, so that the 
call site has the exact same type as the method handle instance:

```java
fooMh.invokeExact((Object) "Hello, world!");
```

The same goes for return values. If the return type of a method handle instance is not `void`,
then, when using `invokeExact`, we _have_ to cast the return value to the right type, which
may require assigning the value even if it's not used:

```java
static int foo() { return 42; }

public static void main(String[] ___) throws Throwable {
    MethodHandle fooMh = MethodHandles.lookup().findStatic(Main.class, "foo",
            MethodType.methodType(int.class));
    
    int x = (int) fooMh.invokeExact(); // result is not used
}
```

So, why would we use `invokeExact` instead of `invoke`? Well, it turns out that automatically
converting arguments to the right type is costly.

The implementation of `invoke` will call
the `MethodHandle::asType` method. `asType` is our first example of a method handle _combinator_.
The `asType` method accepts a `MethodType` and returns a new method handle with the given type,
that does all the needed type adaptations, and then forwards the converted arguments to the original
method handle. We can also call `asType` manually if we want:

```java
static void foo(Object o) {}

public static void main(String[] ___) throws Throwable {
    MethodHandle fooMh = MethodHandles.lookup().findStatic(Main.class, "foo",
            MethodType.methodType(void.class, Object.class));

    // from (Object) -> void to (String) -> void
    fooMh = fooMh.asType(MethodType.methodType(void.class, String.class));

    fooMh.invokeExact("Hello, world!"); // this now works
}
```

To achieve this, the implementation of `asType` will generate a synthetic class + method that 
implements all the parameter and return type conversions. The result is then wrapped in a
new method handle, which is returned. It is as if the `asType` implementation generated
a little wrapper method like this:

```java
static void foo(String str) {
    foo((Object) str);
}
```

If we call `invoke` on a method handle, the implementation will call `asType` to convert the
type of the method handle to the type that is requested by the call site. It is
probably easy to see how generating a new class is costly when we just want to invoke
a method. It's not all bad though: each method handle instance has a 1 element cache, and
the result of `asType` is stored in that cache. If the next call to `asType` requests the
same method type again, the cached method handle is just returned. In practice this means
that calling `invoke` is only costly if the same method handle instance is invoked multiple times
with different call site types, because this causes repeated cache misses, and recreation of
method handles with the requested type.

While very convenient, the automatic type conversions of `invoke` can also be a
performance pitfall. However, we can be 100% sure to avoid this pitfall by
using `invokeExact`, as it will never perform any type adaptations. So, as a performance
rule of thumb: avoid _inexact_ call. Prefer `invokeExact` over `invoke`.

### The `invokeWithArguments` method

Since `j.l.r.Method::invoke` accepts a vararg array of `Object`, we can not only pass in 
a list of arguments, which `javac` will automatically put into an `Object[]` for us, but
we can also manually create an `Object[]` holding the argument values, that we than pass
to `j.l.r.Method::invoke` directly:

```java
Method m = ...
m.invoke(null, 1, 2, 3); // 1.) Ok
Object[] args = { 1, 2, 3 };
m.invoke(null, args); // 2.) also OK
```

This technique doesn't work for sig-poly methods though. Because the type of a sig-poly method
is derived from the call site, if we pass an `Object[]` when invoking a method handle, the
type of the call site will just have `Object[]` as the parameter type. This
array will not be automatically expanded into a list of arguments, since, again, arguments are
passed to a sig-poly method _as is_.

So, if we wish to pass an `Object[]` or a `List` of argument values to a method handle,
we need to use `invokeWithArguments`. This method will expand the array or list into
a series of scalar argument values that are then passed to the method handle.

```java
static void foo(int x, int y, int z) {}

public static void main(String[] __) throws Throwable {
    MethodHandle fooMh = MethodHandles.lookup().findStatic(Main.class, "foo",
            MethodType.methodType(void.class, int.class, int.class, int.class));
    
    fooMh.invoke(1, 2, 3); // OK
    Object[] args = { 1, 2, 3 };
    // fooMh.invoke(args); // BAD
    // WrongMethodTypeException: cannot convert MethodHandle(int,int,int)void to (Object[])void
    fooMh.invokeWithArguments(args); // Ok
}
```

Keep in mind though, that `invokeWithArguments` also does type adaptations internally, and similar
performance caveats apply as to `MethodHandle::invoke`.

## Method handle inlining

I will now discuss inlining optimizations that the C2 JIT compiler applies to method handle 
calls. This is a tricky topic, but important to get at least some idea of, in order to be
able to use method handles effectively. There can be large performance discrepancies between
different uses of method handles, and most of the time either inexact calls (see `invokeExact` section above)
or lack of method handle inlining are to blame.

For regular Java method calls, we know which target method the call points at:

```java
public static void foo() {}

public static main(String[] args) {
    foo();
}
```

In the above, the reference to `foo` is embedded directly into the class file. This means
that the JIT compiler knows exactly which method is being invoked, and can inline that method,
which enables other optimizations to take place.

For method handles however, the information describing the target method is embedded in the 
method handle instance. So, when a method handle is invoked, we go through a trampoline that
reads the target method from the method handle instance, and then forwards the call to that target.
This indirection would normally prevent inlining the target method. After all, the receiver of a call
to a method handle might be any arbitrary method handle instance, which can point at any arbitrary
target method.

However, if the method handle is a constant, the JIT compiler can determine the target
method at JIT-compile time, by inspecting the constant method handle instance, and treat a
call to it as if it were a normal call to the target method, thus re-enabling inlining.

The easiest way to determine whether a value such as a `MethodHandle` is a constant, is to 
simply determine whether the value can be different for different invocations of the same code.

```java
static final MethodHandle MH_FOO;

static {
    try {
        MH_FOO = MethodHandles.lookup().findStatic(Main.class, "foo",
                MethodType.methodType(void.class));
    } catch (ReflectiveOperationException e) {
        throw new ExceptionInInitializerError(e);
    }
}

static void foo() {}

static void m() throws Throwable {
    MH_FOO.invokeExact();
}
```

Here, the receiver of the `invokeExact` call is loaded directly from the `static final` field `MH_FOO`. `static final`
fields can not be changed, not even through reflection. So, the JIT can constant fold the load
from the `MH_FOO` field, making the receiver of `invokeExact` a constant. The JIT can then inspect
the receiver's target method and treat the call to `invokeExact` as a if it were a call to
the target method `foo`.

Now for a more complex example:

```java
...

static void m1() throws Throwable {
    m2(MH_FOO);
}

static void m2(MethodHandle mh) throws Throwable {
    mh.invokeExact();
}
```

Here the receiver instance on which we call `invokeExact` is not a constant inside the `m2` method.
After all, the argument passed to `m2` can be an arbitrary method handle instance. If `m2`
were JIT compiled, no inlining of the `invokeExact` call could take place.

However, in the method `m1`, we pass the constant `MH_FOO` as an argument to `m2`. So,
if the method `m1` was JIT compiled, and in that compilation `m2` was inlined, the receiver
of `invokeExact` _would_ be a constant, and inlining of the `invokeExact` call could take place.

Finally, lets look at instance fields:

```java
class Widget {
    final MethodHandle mh_foo;

    ...

    void invoke() throws Throwable {
        mh_foo.invokeExact();
    }
}
```

Here, `mh_foo` is a field of an instance of `Widget`. If `invoke` was JIT compiled, we would
get a load from the `this` instance to load the value of `mh_foo`. Since the `this` instance
can be any instance of `Widget`, this load can not be constant folded. So, the receiver of
`invokeExact` would also not be a constant, and we get no inlining.

```java
static final Widget W = new Widget(MH_FOO);

static void m() throws Throwable {
    W.invoke();
}
```

In the method `m`, the receiver of `invoke` is a constant, since it's loaded from the `static final` field `W`.
So, if `m` is JIT compiled, and `invoke` is inlined, the `this` instance would be a constant.
However, in that case, the load of the `mh_foo` field of `Widget` can still not be constant
folded, since `final` fields are not 'trusted'. They can for instance still be modified through
reflection. As a result, the JIT may not constant fold the load of the `mh_foo` field, the
receiver of `invokeExact` will not be constant, and the call can not be inlined.

There are some exceptions to this rule though. Classes in certain packages in `java.base`
have trusted `final` fields. Additionally, fields of a record are trusted as well. The JIT
compiler may constant fold these fields if the enclosing instance is constant as well.
So if we for instance turn `Widget` into a record:

```java
record Widget(MethodHandle mh_foo) {
    void invoke() throws Throwable {
        mh_foo.invokeExact();
    }
}
```

And if we call `invoke` on a constant `Widget` instance, the JIT would be allowed to fold
the load of the `mh_foo` field, and inlining of the `invokeExact` call could take place.

## Method handle combinators

Method handle combinators are a large component of the method handle API. Most of these are
found in the `MethodHandles` class as static methods, but some are also found as instance
methods in the `MethodHandle` class itself.

Method handle combinators are a code generation API. Each method handle instance holds a small
program called a `LambdaForm`, which describes what the method handle does. In most cases,
this `LambdaForm` is rendered as bytecode that is then executed directly. The method handle
combinator API can be used to create these little programs, wrapped up in `MethodHandle` instances.

As mentioned before, `MethodHandle::asType` is such a combinator. Let's go over some other
noteworthy examples.

### The `MethodHandles::insertArguments` combinator

`MethodHandles::insertArguments` is probably one of the simplest combinators that you typically
use. It allows a client to create a new method handle that passes one or more fixed argument values
to a given method handle:

```java
static void foo(int x, int y, int z) {
    System.out.println(STR."x=\{x} y=\{y} z=\{z}");
}

public static void main(String[] __) throws Throwable {
    MethodHandle fooMh = MethodHandles.lookup().findStatic(Main.class, "foo",
        MethodType.methodType(void.class, int.class, int.class, int.class));
    
    // at parameter index 1, insert argument value '2'
    MethodHandle mh = MethodHandles.insertArguments(fooMh, 1, 2);

    mh.invokeExact(1, 3); // prints 'x=1 y=2 z=3'
}
```

Here, we insert the fixed argument value `2` at index `1` in the parameter list of the 
_downstream method handle_ `fooMh`. The result
is a new method handle that takes just 2 `int` arguments. This is equivalent to writing a
wrapper method for `foo` like so:

```java
static void fooPrime(int x, int z) {
    foo(x, 2, z);
}
```

### The `MethodHandles::filterArguments` combinator

Moving on, `MethodHandles::filterArguments` creates a method handle that pre-processes one
or more of its arguments by calling a filter method, before passing on the result to a target
method handle:

```java
static void foo(int x, int y, int z) {
    System.out.println(STR."x=\{x} y=\{y} z=\{z}");
}

static int bar(int x) {
    return x + 1;
}

public static void main(String[] __) throws Throwable {
    MethodHandle fooMh = MethodHandles.lookup().findStatic(Main.class, "foo",
        MethodType.methodType(void.class, int.class, int.class, int.class));
    MethodHandle barMh = MethodHandles.lookup().findStatic(Main.class, "bar",
        MethodType.methodType(int.class, int.class));
    
    // applies filter 'barMh' to argument at index 1, and passes the result to 'fooMh'
    MethodHandle mh = MethodHandles.filterArguments(fooMh, 1, barMh);

    mh.invokeExact(1, 2, 3); // prints 'x=1 y=3 y=3'
}
```

This is equivalent to the following java code:

```java
static void fooPrime(int x, int y, int z) {
    foo(x, bar(y), z);
}
```

Of note is that the filters used with `filterArguments` must have exactly 1 parameter, and
return a value of the type that the downstream method handle accept at the filtered index.

### The `MethodHandles::collectArguments` combinator

`MethodHandles::collectArguments` has less restrictions, as it allows multiple argument values
to be condensed into one value passed to the downstream method handle:

```java
static void foo(int x, int y, int z) {
    System.out.println(STR."x=\{x} y=\{y} z=\{z}");
}

static int bar(int x, int y) {
    return x + y;
}

public static void main(String[] __) throws Throwable {
    MethodHandle fooMh = MethodHandles.lookup().findStatic(Main.class, "foo",
        MethodType.methodType(void.class, int.class, int.class, int.class));
    MethodHandle barMh = MethodHandles.lookup().findStatic(Main.class, "bar",
        MethodType.methodType(int.class, int.class, int.class));
    
    // applies filter 'barMh' to argument at index 1, and passes the result to 'fooMh'
    MethodHandle mh = MethodHandles.collectArguments(fooMh, 1, barMh);

    mh.invokeExact(1, 2, 2, 3); // prints 'x=1 y=4 y=3'
}
```

This is equivalent to the following java code:

```java
static void fooPrime(int a0, int a1, int a2, int a3) {
    foo(a0, bar(a1, a2), a3);
}
```

Alternatively, the collector may also accept zero arguments, in which case it could generate
an argument value on the fly, like a supplier:

```java
static void foo(int x, int y, int z) {
    System.out.println(STR."x=\{x} y=\{y} z=\{z}");
}

static int bar() {
    return ThreadLocalRandom.current().nextInt();
}

public static void main(String[] __) throws Throwable {
    MethodHandle fooMh = MethodHandles.lookup().findStatic(Main.class, "foo",
        MethodType.methodType(void.class, int.class, int.class, int.class));
    MethodHandle barMh = MethodHandles.lookup().findStatic(Main.class, "bar",
        MethodType.methodType(int.class));
    
    // applies filter 'barMh' to argument at index 1, and passes the result to 'fooMh'
    MethodHandle mh = MethodHandles.collectArguments(fooMh, 1, barMh);

    mh.invokeExact(1, 3); // prints 'x=1 y=<random numer> y=3'
}
```

This is still pretty straightforward, but `collectArguments` can also be used together with a
collector that returns `void`. If we do that, the collector essentially acts as a side effect
that is executed before the downstream method handle, rather than computing an argument of
the downstream handle based on zero or more inputs.

In the case of a `void`-returning collector, the index we specify to `collectArguments`
indicates the place in the parameter list of the downstream handle where we want to insert
the parameter list of the collector. Instead of this index pointing at a particular parameter
of the downstream handle, it can be thought of as pointing at the space between parameters.
Where, e.g. index `0` would indicate that the parameters of the collector should appear
before the first parameter of the downstream handle. For instance:

```java
static void foo(String s, Object o, int i) {
}

static void bar(long l, double d) {
}

public static void main(String[] __) throws Throwable {
    MethodHandle fooMh = MethodHandles.lookup().findStatic(Main.class, "foo",
        MethodType.methodType(void.class, String.class, Object.class, int.class));
    MethodHandle barMh = MethodHandles.lookup().findStatic(Main.class, "bar",
        MethodType.methodType(void.class, long.class, double.class));

    MethodHandle mh = MethodHandles.collectArguments(fooMh, 0, barMh);
    System.out.println(mh.type()); // prints '(long,double,String,Object,int)void'
}
```

The type of `mh` here is `(long,double,String,Object,int)void`.

And of course, we could even have a collector that takes no arguments and returns nothing.
`collectArguments` is probably the most flexible combinator out there.

### The `MethodHandles::permuteArguments` combinator

`MethodHandles::permuteArguments` can be used to change the order of the parameters of a 
method handle, or to duplicate certain argument values and "broadcast" them to multiple
parameters of the downstream method handle:

```java
static void foo(int a0, int a1, int a2, int a3) {
    System.out.println(STR."a0=\{a0} a1=\{a1} a2=\{a2} a3=\{a3}");
}

public static void main(String[] __) throws Throwable {
    MethodHandle fooMh = MethodHandles.lookup().findStatic(Main.class, "foo",
        MethodType.methodType(void.class, int.class, int.class, int.class, int.class));

    MethodType newType = MethodType.methodType(void.class, int.class, int.class, int.class);
    int[] reorder = { 1, 0, 2, 2 };
    MethodHandle mh = MethodHandles.permuteArguments(fooMh, newType, reorder);

    mh.invokeExact(1, 2, 3); // prints 'a0=2 a1=1 a2=3 a3=3'
}
```

This is equivalent to the following java code:

```java
static void fooPrime(int a0, int a1, int a2) {
    foo(a1, a0, a2, a2);
}
```

`permuteArguments` accepts a method type describing the type of the notional wrapper method,
and a 'reorder array', which describes how each parameter in the new type is wired to a 
parameter of the target method handle (`fooMh`).

According to the reorder array: the first parameter of `fooMh` should receive the argument
at index `1`, the second parameter of `fooMh` should receive the argument at index `0`, and
the 3rd and 4th parameters of `fooMh` should both receive the argument at index 2.

The argument duplication capability of `permuteArguments` essentially allows you to use an
argument value multiple times in a downstream method handle. It can even be used to drop
argument values that are not needed by the downstream method handle, by specifying a new 
method type that has more arguments than the downstream handle, and a reorder array that 
doesn't use every parameter index:

```java
static void foo(int a0, int a1) {
    System.out.println(STR."a0=\{a0} a1=\{a1}");
}

public static void main(String[] __) throws Throwable {
    MethodHandle fooMh = MethodHandles.lookup().findStatic(Main.class, "foo",
        MethodType.methodType(void.class, int.class, int.class));

    MethodType newType = MethodType.methodType(void.class, int.class, int.class, int.class);
    int[] reorder = { 0, 1 };
    MethodHandle mh = MethodHandles.permuteArguments(fooMh, newType, reorder);

    mh.invokeExact(1, 2, 3); // prints 'a0=1 a1=2'
}
```

However, the `MethodHandles::dropArguments` combinator might be more convenient for that
use case.

### Other combinators

There are many more combinators found in the `MethodHandles` and `MethodHandle` classes,
but I won't go over all of them here. I've gone over some of the commonly used combinators,
to give the basic gist of how combinators work and can be reasoned about, but ultimately,
the best way to learn all the combinators is to experiment with them yourself.

💡As we've seen, method handle combinators make it very easy to generate small snippets
of code. The combinators are essentially an API to programmatically write Java code.
Combinators are yet another good reason why you'd want to use method handles.

## Appendix A: `MutableCallSite`

`MutableCallSite` is another noteworthy class in the `java.lang.invoke` package. It is essentially
just a holder for a `target` method handle. However, the special thing is that, even though
the `target` field is mutable, the JIT may constant fold loads from this field (as long
as the enclosing `MutableCallSite` instance is constant as well).

This means that you can essentially have a 'mostly-constant' method handle, that can still
take advantage of method handle inlining optimizations, but which can also be be swapped out
for another method handle. To make this work, the JIT will record a 'dependency' on the
state of the mutable call site in compiled code, and when the call site's target
is changed, the compiled code will be thrown away. This is a powerful tool, but should be
used sparingly, since the code will go back to running in the interpreter, and will have
to be compiled again by the JIT.

## Appendix B: `VarHandle`

Var handles can be thought of as bundles of method handles that relate specifically to
memory access. Instead of a single `invoke` method they have various `get*`/`set*` methods
that implement memory access with different memory ordering semantics.

The same performance caveats apply to var handles as to method handles. The receiver `VarHandle`
instance of a `get*`/`set*` call needs to be a constant, and the call needs to be exact.
Note though, that while method handles have a dedicated `invokeExact` method, var handles don't.
The exact invocation behavior of var handles has to be turned on explicitly by calling
`VarHandle::withInvokeExactBehavior`. This method returns a new var handle who's `get*`/`set*`
will check that the call site type matches the type of the corresponding var handle's access mode.

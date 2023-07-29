---
layout: post
title:  "Debugging native library linkage errors"
date:   2021-09-13 16:00:42 +0200
categories: java panama-ffi panama jni native
---

When using native libraries in Java, you'll sooner or later run into linkage errors. In this blog I will go over the most common cases in which linkage errors can occur.

1. [Linkage error when loading a library](#1-linkage-error-when-trying-to-load-a-library) \
    1a. [The incorrect library name is being used](#1a-the-incorrect-library-name-is-being-used) \
    1b. [The `java.library.path` system property is not set correctly](#1b-the-javalibrarypath-system-property-is-not-set-correctly)
2. [Linkage error when looking up a symbol in the library](#2-linkage-error-when-looking-up-a-symbol) \
    2a. [The library does not contain the function](#2a-the-library-does-not-contain-the-function) \
    2b. [The function is not exported](#2b-the-function-is-not-exported) \
    2c. [The wrong library was loaded](#2c-the-wrong-library-was-loaded)

### 1. Linkage error when trying to load a library

The first kind of linkage error we will look at typically manifests itself as an [`UnsatisfiedLinkError`](https://docs.oracle.com/en/java/javase/18/docs/api/java.base/java/lang/UnsatisfiedLinkError.html). You might commonly encounter this error when calling [`System.loadLibrary(String)`](https://docs.oracle.com/en/java/javase/18/docs/api/java.base/java/lang/System.html#loadLibrary(java.lang.String)), such as:

```java
public class Main {
    public static void main(String[] args) {
        System.loadLibrary("foo");
    }
}
```

The error message has the form:

```text
Exception in thread "main" java.lang.UnsatisfiedLinkError: no foo in java.library.path: <list of paths>
        at java.base/java.lang.ClassLoader.loadLibrary(ClassLoader.java:2447)
        at java.base/java.lang.Runtime.loadLibrary0(Runtime.java:809)
        at java.base/java.lang.System.loadLibrary(System.java:1893)
        at Main.main(Main.java:4)
```

This error typically means one of two things:

1. An incorrect library name is being used.
2. The `java.library.path` system property is not set correctly.

#### 1a. The incorrect library name is being used

When calling `System.loadLibrary(String)` the name of the library that is passed as an argument will be mapped into a file name in a platform-specific manner, and then loaded.

Some common examples are:

- On Windows, the library name will get the `.dll` suffix. e.g. `foo -> foo.dll`.
- On Linux, the library name will get the `lib` prefix and the `.so` suffix. e.g. `foo -> libfoo.so`.
- On Mac, the library name will get the `lib` prefix and the `.dylib` suffix. e.g. `foo -> libfoo.dylib`.

So, for instance to load a library file called `libfoo.so` on Linux, you would have to use

```java
System.loadLibrary("foo");
```

To see how library names are mapped on other platforms, or to see how a particular library name is mapped to a file name, you can call [`System.mapLibraryName(String)`](https://docs.oracle.com/en/java/javase/20/docs/api/java.base/java/lang/System.html#mapLibraryName(java.lang.String)).

#### 1b. The `java.library.path` system property is not set correctly

The library path that was used to look up the library is printed in the exception message (`<list of paths>` above). It is a list of directories that was used to find the native library. One of those directories should contain the library you're trying to load.

If the library you're trying to load is located in another directory, you can set the `java.library.path` system property to the directory that contains your library using the `-Djava.library.path=/path/to/lib` VM argument, where multiple directories can be added separated by `;` on Windows, and `:` on other platforms, e.g. `-Djava.library.path=/path1:/path2:/path3`.

If the error keeps occurring, check the exception message. If it does not contain the library path you expected, you might be passing the command line option as a program argument instead of a VM argument by accident (VM arguments should be passed _before_ the main class), or it could be a problem with missing quotes in the shell you're using (e.g. `powershell` requires passing system properties in quotes, such as `'-Dmy.prop=val'` to avoid being picked up as other syntax).

### 2. Linkage error when looking up a symbol

The second kind of linkage error you can commonly run into is thrown when a function in a library can not be found.

For instance, if we extend the previous example program as follows:

```java
public class Main {
    public static void main(String[] args) {
        System.loadLibrary("foo");
        bar();
    }

    static native void bar();
}
```

Even if the call to `System.loadLibrary` succeeds and the library is loaded, calling the function `bar` might fail:

```text
Exception in thread "main" java.lang.UnsatisfiedLinkError: 'void Main.bar()'
        at Main.bar(Native Method)
        at Main.main(Main.java:6)
```

This can have one of 3 causes:

1. The library that is being used does not contain the function symbol.
2. The function is not exported.
3. The wrong library is being loaded.

#### 2a. The library does not contain the function

The JVM will derive the name of the function symbol it looks for from the package, class, and method name. The format it will have is roughly like `Java_my_package_MyClass_myMethod`. Another thing to note is that underscores in any of the Java names will be translated into the symbol name as `_1`.

For JNI a good way to make sure the name of the function in the native library is correct, is to regenerate the header file for the class with the method that fails to link (using `javac -h <path>`) and check to make sure that the function name in the header file matches the function name of the implementation (in the `.c/.cpp` file).

Since the function name is derived from the package, class, and method name, moving the class to a different package, or renaming the package, class, or method, are all things that can make the name that the JVM expects go out of sync with the name of the function in the library.

---

Another way to check which symbol the JVM is looking for, is with the `-Xlog:library=info` VM option (added in JDK 15). This will print out messages about which symbols the JVM tries to look up. Such as this:

```text
[0.622s][info][library] Failed to find Java_my_package_MyClass_myMethod in library with handle 0x00007ff8a24f0000
```

These log messages can also be used to check the name of the function that the JVM is looking for.

#### 2b. The function is not exported

A second reason why a function might not be found inside the library, even if the function name is correct, is because the function is not exported from the library. For instance on Windows, using the MSVC compiler, it is important to declare functions that need to be accessible from outside of the library with [`__declspec(dllexport)`](https://docs.microsoft.com/en-us/cpp/cpp/dllexport-dllimport?view=msvc-170).

For JNI the `JNIEXPORT` macro can be used for this, which will do the right thing. This macro will automatically be included in the function declaration generated by `javac -h`.

Checking that the function you're trying to call is exported becomes more important if you're working with the Foreign Linker API, since the function in the library is likely not declared with `JNIEXPORT`.

#### Listing the functions in a native library

There are several tools that can be used to check which functions are contained in a native library, which can be useful to debug one of the above problems.

- On Windows, the `dumpbin` tool that comes with visual studio can be used with the `/EXPORTS <library file>` to print out exported symbols. The [Dependencies tool](https://github.com/lucasg/Dependencies), which also has a GUI, can also be used. 
- On Linux the `nm` tool can be used to print out the symbols in a library.
- On Mac, I believe `otool` can be used (but I have no experience using this).

#### 2c. The wrong library was loaded

Finally, if a symbol can not be found inside a library, but the symbol name is correct, and it is exported from the library, it might be because the wrong library is being loaded.

This can happen for instance because there are 2 versions of the same library found on the `java.library.path`, and the wrong one is being picked up. Or because the library that you're trying to load has the same name as one of the libraries bundled with the JDK (such as `attach`).

Again the `-Xlog:library=info` VM flag comes to our rescue, because it also prints out information about which libraries are being loaded, such as:

```text
[0.066s][info][library] Loaded library C:\Program Files\Java\jdk-18\bin\jimage.dll, handle 0x00007ff8bfc00000
```

If the library you're trying to use is loaded, but the path in the loading message is not what you expected, it's likely that the wrong library file is being loaded.

In the case that the same library is found multiple times on the `java.library.path` one solution is to remove the directories with the incorrect library files from the `java.library.path`, or to remove those library files.

If this is not practical because something else needs those library paths or files (such as the JVM itself, in the case it is a library bundled with the JVM), it might be possible to rename the library you're trying to use instead, so that the name no longer conflicts with others.

If both changing the library path, or changing the name of the library are not an option, then you could finally manually construct the absolute path of the library you're trying to use, and use [`System.load(String)`](https://docs.oracle.com/en/java/javase/18/docs/api/java.base/java/lang/System.html#loadLibrary(java.lang.String)) to load the library directly.

### Thanks for reading

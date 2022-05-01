---
layout: post
title:  "Using defineHiddenClassWithClassData"
date:   2022-05-01 14:14:42 +0100
categories: jli methodhandles asm
---

If you're somewhat familiar with `sun.misc.Unsafe`, you are probably familiar with the `Unsafe.defineAnonymousClass` API. Before it was removed in JDK 16, this API allowed defining classes with live objects 'patched' into the constant pool, by passing them as an `Object[]` argument when defining the class. This is a powerful idiom that can be used to generate classes that references pre-resolved constant data that is not necessarily representable using other constant pool types.

The functionality of this method was replaced with [`MethodHandles.Lookup::defineHiddenClassWithClassData`](https://docs.oracle.com/en/java/javase/18/docs/api/java.base/java/lang/invoke/MethodHandles.Lookup.html#defineHiddenClassWithClassData(byte[],java.lang.Object,boolean,java.lang.invoke.MethodHandles.Lookup.ClassOption...)) in JDK 16. The javadoc for that method says:

> A framework can ... load the class data as dynamically-computed constant(s) via a bootstrap method

But, if you're unfamiliar with dynamic constants, it might be unclear how to do this with, say, ASM.

So, here is an example which uses the new class data API using ASM (tested with ASM 9.3 and JDK 18).

```java
package main;

import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.ConstantDynamic;
import org.objectweb.asm.Handle;
import org.objectweb.asm.MethodVisitor;
import org.objectweb.asm.Type;

import java.lang.constant.ConstantDescs;
import java.lang.invoke.MethodHandle;
import java.lang.invoke.MethodHandles;
import java.lang.invoke.MethodType;
import java.util.List;

import static org.objectweb.asm.Opcodes.*;

public class Main {

    // an ASM 'Handle' describing the MethodHandles::classDataAt method
    private static final Handle H_CLASS_DATA_AT = new Handle(
            H_INVOKESTATIC,
            Type.getInternalName(MethodHandles.class),
            "classDataAt",
            MethodType.methodType(Object.class, MethodHandles.Lookup.class, String.class, Class.class, int.class)
                    .descriptorString(),
            false
        );

    public static void main(String[] args) throws Throwable {
        // generate a class
        ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_MAXS);
        cw.visit(V18, ACC_PUBLIC, "main/Widget", null, Type.getInternalName(Object.class), null);

        // generate a `getClassData` method which will be used to demonstrate the 
        // retrieval of class data.
        MethodVisitor mv = cw.visitMethod(ACC_PUBLIC | ACC_STATIC, "getClassData",
                MethodType.methodType(Object.class).descriptorString(), null, null);
        mv.visitCode();
        mv.visitLdcInsn(
            new ConstantDynamic(
                ConstantDescs.DEFAULT_NAME, // mandated name
                String.class.descriptorString(), // type of the constant
                H_CLASS_DATA_AT, // bootstrap method reference
                0 // bootstrap method arg. The index of the class data object we want to load.
            )
        );
        mv.visitInsn(ARETURN); // return the loaded result
        mv.visitMaxs(-1, -1);
        mv.visitEnd();

        byte[] bytes = cw.toByteArray();
        List<?> data = List.of("Class data"); // index 0 = "Class data". That's what we load above
        // define the class, with our class data
        MethodHandles.Lookup lookup = MethodHandles.lookup().defineHiddenClassWithClassData(bytes, data, true);
        // lookup the `getClassData` method we generated above
        MethodHandle MH_getClassData = lookup.findStatic(lookup.lookupClass(), "getClassData",
                                                         MethodType.methodType(Object.class));
        // invoke it to get our class data
        System.out.println(MH_getClassData.invoke()); // prints out "Class data"
    }
}
```

The class data is loaded through and `ldc` instruction (`mv.visitLdcInsn`), to which we pass a `ConstantDynamic` that represents our dynamic constant. This dynamic constant is set up to call the [`MethodHandles::classDataAt`](https://docs.oracle.com/en/java/javase/18/docs/api/java.base/java/lang/invoke/MethodHandles.html#classDataAt(java.lang.invoke.MethodHandles.Lookup,java.lang.String,java.lang.Class,int)) method (`H_CLASS_DATA_AT`) as a bootstrap method, with a single bootstrap method argument, `0`.

The first time this `ldc` instruction runs, the VM will call this bootstrap method, which will retrieve the object at index 0 of the list of class data which we use when we define the class below that (see the `defineHiddenClassWithClassData` line). The first three arguments to the bootstrap method, the method handle lookup, name, and type, are provided by the VM, and we provide the index of the object we want to load from the class data list as a constant argument. After resolution, the loaded object will be stored in the constant pool slot associated with the dynamic constant, so the next time it is loaded the resolution step will be skipped, and the stored object is loaded directly instead.

This powerful idiom can be used to put live objects into the constant pool of a generated class. These objects will then be treated by the JIT as constants. This approach can be used for instance to generate code that calls a pre-resolved method handle. Because the method handle will be a constant in the constant pool, the call can be fully optimized.

## Thanks for reading
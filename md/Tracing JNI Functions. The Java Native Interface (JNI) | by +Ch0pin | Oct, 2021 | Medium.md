> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [valsamaras.medium.com](https://valsamaras.medium.com/tracing-jni-functions-75b04bee7c58)

> From the viewpoint of a JVM, there are two types of code: Java and Native with the latest referring t......

Tracing JNI Functions
=====================

[![](https://miro.medium.com/fit/c/35/35/0*4uechiEB5YonWvdZ)](/?source=post_page-----75b04bee7c58--------------------------------)[

+Ch0pin

](/?source=post_page-----75b04bee7c58--------------------------------)[

1 day ago·7 min read

](/tracing-jni-functions-75b04bee7c58?source=post_page-----75b04bee7c58--------------------------------)[](https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fbookmark%2Fp%2F75b04bee7c58&operation=register&redirect=https%3A%2F%2Fvalsamaras.medium.com%2Ftracing-jni-functions-75b04bee7c58&source=post_actions_header--------------------------bookmark_preview-----------)

The Java Native Interface (JNI)
===============================

From the viewpoint of a JVM, there are two types of code: **Java** and **Native** with the latest referring to code written in C /C++ or even Assembly. The **J**ava **N**ative **I**nterface (JNI) provides a mechanism (bridge) by which a program written in Java can call routines or make use of services written in native code.

Despite the fact that this bridge refers to a more general JVM concept, we are going to narrow down our scope and focus on the Android OS where it is usually used for extra performance**,** run of computationally intensive applications (such as games or physics simulations) or use of C/C++ libraries.

As the native code introduces an extra level of complexity, when it comes to reverse engineering, the JNI is heavily used by malware developers in order to evade or delay the detection process.

The JNI In practice
===================

Learning, how to program using the JNI, is out of scope of this post since there are excellent resources (see References) that someone may refer in order to do that. Some basic concepts though are necessary in order to set up a sufficient learning path.

The image bellow, taken from [Oracle’s documentation](https://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/design.html), grasps the structure of this framework:

![](https://miro.medium.com/freeze/max/38/0*AKKNEYE5UAtN3o-I.gif?q=20)![](https://miro.medium.com/max/829/0*AKKNEYE5UAtN3o-I.gif)

More specifically, imagine a set of [interface functions](https://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/functions.html) arranged by their pointer in a respective array. The JNI interface pointer, points to another pointer which subsequently points to this array.

Reverse Engineering JNI functions
=================================

From a Reverse Engineering point of view, in order to investigate a native function we usually follow the steps bellow:

*   Locate and extract the library file (1)
*   Trace its usage in the Java code (2)
*   Trace the Java calls to the native code (3)
*   Locate the native functions in the library file (4)

And finally review the code. In regards to (1), **when dealing with an apk file**, use the apktool to decode the application’s resources:

```
apktool d application.apk 

```

![](https://miro.medium.com/max/38/1*AVBHeniG7UddTYPkY_ChEA.png?q=20)![](https://miro.medium.com/max/593/1*AVBHeniG7UddTYPkY_ChEA.png)

cd to the **lib/<arch specific>/** directory and locate the library that you want to reverse. When dealing with an already installed application, either pull the apk file using **adb** or pull the library file from the dedicated directory:

```
legacyNativeLibraryDir=/data/app/~~2znHOCr5mIJyG74aM9WELw==/com.foo.bar-CwB-UExo3en4VD-NZ7k0XQ==/lib

```

**(2)** Assuming that the name of the library is **libnativelib.so** search for `System.loadLibrary("nativelib")`or `System.load("full/path/to/lib.so")`

(the second is less common and used in cases the library is loaded from a custom directory).

**(3,4)** According what has been said so far, the **libnativelib.so** is loaded by the class MyClass using the loadLibrary and declares **3** native calls to foo, bar and foobar functions:

```
public class MyClass {
 static {
     System.loadLibrary(“nativelib”);
 }
...
...
public static native void foo(String str);
public static native void bar(Context context, String str, boolean z, int i);
public static native String foobar(Context context);

```

Tracing the native implementation
=================================

There are two ways to “link” the Java with the Native code:

*   Dynamic Linking using JNI Native Method Name Resolving
*   Static Linking using the RegisterNatives API call

While the first is pretty easy to locate (search for `Java_` in Ghidra’s Symbol Tree — see image bellow), when the RegisterNatives is used, things can be more complex especially in case that the .so has been hardened to reverse (e.g. obfuscated)

![](https://miro.medium.com/max/38/1*I-HZRHe_0Vx1E-rm2GMOww.png?q=20)![](https://miro.medium.com/max/875/1*I-HZRHe_0Vx1E-rm2GMOww.png)Dynamic linking![](https://miro.medium.com/max/38/1*4K3brsLKnSFMu0gaeJ6w1Q.png?q=20)![](https://miro.medium.com/max/875/1*4K3brsLKnSFMu0gaeJ6w1Q.png)Using Register Natives![](https://miro.medium.com/max/38/1*d2Ua0eiz3EehHrsMr-Bf6w.png?q=20)![](https://miro.medium.com/max/875/1*d2Ua0eiz3EehHrsMr-Bf6w.png)Junk / Obfuscation

Setting the correct data type
=============================

For Ghidra to identify the correct JNI Function names when opening a library, the easiest way is to import the proper data types, which for JNI can be found [here](https://github.com/Ayrx/JNIAnalyzer/blob/master/JNIAnalyzer/data/jni_all.gdt). We can import this data type using the Data Type Manager -> Open File Archive option.

![](https://miro.medium.com/max/38/1*eJ9G2PoBSNA_Tausxg8JQQ.png?q=20)![](https://miro.medium.com/max/875/1*eJ9G2PoBSNA_Tausxg8JQQ.png)Import the gdt file

We can then edit the function signature and apply the correct type:

![](https://miro.medium.com/max/38/1*eg8KLMd2uh5EgDjlD-k6_g.png?q=20)![](https://miro.medium.com/max/875/1*eg8KLMd2uh5EgDjlD-k6_g.png)Retype the JNIEnv (first parameter)

Ghidra will resolve the correct names automatically:

![](https://miro.medium.com/max/38/1*XV2qzU_dhGNwa6FW_IyNBA.png?q=20)![](https://miro.medium.com/max/875/1*XV2qzU_dhGNwa6FW_IyNBA.png)

**For the new comers I strongly suggest Maddie Stone’s excellent** [**Android 101 Reverse Engineering**](https://www.ragingrock.com/AndroidAppRE/reversing_native_libs.html) **relevant page to get a good introduction on the above concepts.**

RegisterNatives I
=================

The **JNI_OnLoad** function is the starting point when static linking is used and we normally expect to see the java code declared name as well as the functions signature as string in the ELF file.

For example, assuming the native function bellow:

```
public static native void foo(String str);

```

We should expect to find the following signature

```
(Ljava/lang/String)V;

```

When this is not the case the ELF file is probably using an anti-analysis technique.

RegisterNatives II
==================

Reaching close to the actual goal of this article, we are going to use [https://github.com/chame1eon/jnitrace](https://github.com/chame1eon/jnitrace) and show how to handle an elf file which is using anti-analysis techniques in order to hide the actual native implementation of the java declared native function.

The usage of jnitrace is pretty straight forward, simply type:

`jnitrace -l libnative-lib.so com.example.myapplication`

Where **libnative-lib.so** is the elf file that we want to trace and **com.example.myapplication** is the package name of the Android application which contains it. While the output might be quite long, lets focus on the **RegisterNatives** trace in order to find the actual native functions offsets:

![](https://miro.medium.com/max/38/1*F0BWgCsnF4eqpeDmYrLHIg.png?q=20)![](https://miro.medium.com/max/875/1*F0BWgCsnF4eqpeDmYrLHIg.png)

The figure above includes everything that we need:

*   **jclass:** The java class where the library is loaded
*   **yymsd, elvke, kvmx, eaji, gl, waz** are the names of the functions with their corresponding signatures, e.g:

```
yymsd(Landroid/content/Context;Ljava/lang/String;ZI)V

```

will be:

```
public static native void yymsd(Context context, String str, boolean z, int i);

```

*   The elf (.so) file is loaded at **0x6dbaf81000**

To trace down the implementation of each function we have **to subtract the memory offset of the elf file (elfOffset) from the memory offset of each function (funcOffset)** indicated by jnitrace:

```
actualOffset = elfOffset — funcOffset

```

For example, the **yymsd** memory offset is **0x6dbaf93410,** thus the actual offset will be:

```
6dbaf93410–6dbaf81000 = 12410

```

Opening the file with Ghidra and setting the Image Base to 0:

![](https://miro.medium.com/max/38/1*Rqx1fuSRzNQN2BF0PkOQCA.png?q=20)![](https://miro.medium.com/max/875/1*Rqx1fuSRzNQN2BF0PkOQCA.png)

Then simply type ‘G’ for go to navigate to the offset 0x12410:

![](https://miro.medium.com/max/38/1*6TxP5c48y0PXPIzdEpfaqQ.png?q=20)![](https://miro.medium.com/max/850/1*6TxP5c48y0PXPIzdEpfaqQ.png)![](https://miro.medium.com/max/38/1*rshEvNhIk8E1oAyQ27nstQ.png?q=20)![](https://miro.medium.com/max/875/1*rshEvNhIk8E1oAyQ27nstQ.png)Tracing the implementation of the **yymsd function**

RegisterNatives III
===================

Besides the custom functions, it is possible to trace down each JNI call individually. Using the same figure from above, the offset where the RegisterNatives is called will be (**0x168cc**) :

![](https://miro.medium.com/max/38/1*F0BWgCsnF4eqpeDmYrLHIg.png?q=20)![](https://miro.medium.com/max/875/1*F0BWgCsnF4eqpeDmYrLHIg.png)

Using the same process as above:

![](https://miro.medium.com/max/38/1*6YJR3fQ_8GiA_WJu8bDFkA.png?q=20)![](https://miro.medium.com/max/875/1*6YJR3fQ_8GiA_WJu8bDFkA.png)RegisterNatives call![](https://miro.medium.com/max/38/1*2IWwTbB3eeX5U34tLEVzAQ.png?q=20)![](https://miro.medium.com/max/875/1*2IWwTbB3eeX5U34tLEVzAQ.png)RegisterNatives (Retyped)

The DAT_000xxxx
===============

On the example above despite the fact that we successfully located the actual offset of the native function dealing with the memory references is very hard, as someone has to understand the full login behind the anti-analysis technique in order to get usable values. Lets see the example bellow:

![](https://miro.medium.com/max/38/1*0sbxlSkjep1zbADLqJHi0A.png?q=20)![](https://miro.medium.com/max/875/1*0sbxlSkjep1zbADLqJHi0A.png)

the **signature** and **name** parameters which are subsequently given to the RegisterNatives, are referring to the offsets 0x3df40 and 0x3dfa8 which again don’t have any value to us since we are dealing with a hardened binary:

![](https://miro.medium.com/max/38/1*s_12yPATu_8KgPi9o5N6rw.png?q=20)![](https://miro.medium.com/max/678/1*s_12yPATu_8KgPi9o5N6rw.png)

[MEDUSA](https://github.com/Ch0pin/medusa), has a cool feature (called memory operations or memops) which allows us to get the value of a memory offset during runtime, by providing the actual offset:

![](https://miro.medium.com/max/38/1*5Qq0hSUeA4B49gePvehakQ.png?q=20)![](https://miro.medium.com/max/354/1*5Qq0hSUeA4B49gePvehakQ.png)

By the time that app is running and the library is loaded we have the following options:

![](https://miro.medium.com/max/38/1*dCso0bAVFZaFILw8aiBy_Q.png?q=20)![](https://miro.medium.com/max/494/1*dCso0bAVFZaFILw8aiBy_Q.png)

The r@offset option will allow us to get to our goal. Following up on the example above lets read what is at 0x3df40:

![](https://miro.medium.com/max/38/1*Rd9Mll4mUkGUcWouqx56Ew.png?q=20)![](https://miro.medium.com/max/678/1*Rd9Mll4mUkGUcWouqx56Ew.png)Memory contents of address space 0x6dd3f40000 → 6dd3f7f000

Which is as expected, the signatures and function names of the RegisterNatives. Simply type “dump” to save all the contents of the particular memory space or just keep exploring ….

References
==========

[

Android App Reverse Engineering 101
-----------------------------------

### Learn to reverse engineer Android applications! View the Project on GitHub maddiestone/AndroidAppRE Table of Contents…

www.ragingrock.com

](https://www.ragingrock.com/AndroidAppRE/reversing_native_libs.html)[

Introduction to Java Native Interface: Establishing a bridge between Java and C/C++
-----------------------------------------------------------------------------------

### JNI (Java Native Interface) is a foreign function interface that allows code running on JVM to call (or be called by)…

medium.com

](https://medium.com/swlh/introduction-to-java-native-interface-establishing-a-bridge-between-java-and-c-c-1cc16d95426a)[

JNI APIs and Developer Guides
-----------------------------

### Java Native Interface (JNI) is a standard programming interface for writing Java native methods and embedding the Java…

docs.oracle.com

](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/)[

Understanding the Java Native Interface (JNI)
---------------------------------------------

### This description of the Java Native Interface (JNI) provides background information to help you diagnose problems with…

www.ibm.com

](https://www.ibm.com/docs/en/sdk-java-technology/7?topic=components-java-native-interface-jni)
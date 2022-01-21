> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.trailofbits.com](https://blog.trailofbits.com/2022/01/19/c-your-data-structures-with-rellic-headergen/)

> By Francesco Bertolaccini Have you ever wondered how a compiler sees your data structures? Compiler E......

**By Francesco Bertolaccini**

Have you ever wondered how a compiler sees your data structures? Compiler Explorer may help you understand the relation between the source code and machine code, but it doesn’t provide as much support when it comes to the layout of your data. You might have heard about padding, alignment, and “plain old data types.” Perhaps you’ve even dabbled in emulating inheritance in C by embedding one structure in another. But could you guess the exact memory layout of all these types, without looking at the ABI reference for your platform or the source for your standard library?

```
struct A { int x; };
struct B { double y; };
struct C : A, B { char z; };

```

With the requisite ABI knowledge, reasoning about C structs is relatively simple. However, more complex C++ types are an altogether different story, especially when templates and inheritance come into play. Ideally, we’d be able to convert all these complicated types into simple C structs so that we could more easily reason about their in-memory layouts. This is the exact purpose of `rellic-headergen`, a tool I developed during my internship at Trail of Bits. In this blog post, I will explain why and how it works.

rellic-headergen
----------------

The purpose of `rellic-headergen` is to produce C type definitions that are equivalent to those contained in an LLVM bitcode file, which have not necessarily been generated from C source code. This facilitates the process of analyzing programs that contain complex data layouts. The following image provides an example of rellic-headergen’s capabilities.

[![](https://i0.wp.com/blog.trailofbits.com/wp-content/uploads/2022/01/image-1.png?resize=690%2C674&ssl=1)](https://i0.wp.com/blog.trailofbits.com/wp-content/uploads/2022/01/image-1.png?ssl=1)

The left-side window shows our source code. We execute the first command in the bottom window to compile the code to LLVM bitcode, and we use the second command to run it through `rellic-headergen`. The right-side window shows `rellic-headergen`’s output, which is valid C code that matches the layout of the input C++ code.

The utility works on the assumption that the program being analyzed can be compiled to LLVM bitcode with full debug information. The utility begins building a list of all the types for which debug information is available, beginning with function (“subprogram”) definitions.

Now, the utility needs to decide the order in which the types will be defined, but this is no simple task given the requirements of the C language: the language requires explicit forward declarations when referencing types that have not yet been defined, and structs cannot contain, for example, a field whose type has only been forward declared.

One approach to solving this problem would be to preventively forward declare all present types. However, this is not sufficient. For example, a struct cannot contain a field whose type has not been fully defined, though it can contain a field whose type is a pointer to a forward declared type.

Thus, the utility forms a directed acyclic graph (DAG) from the type definitions, on which it can find a topological sort.

[![](https://i0.wp.com/blog.trailofbits.com/wp-content/uploads/2022/01/rellic-headergen-1.png?resize=690%2C492&ssl=1)](https://i0.wp.com/blog.trailofbits.com/wp-content/uploads/2022/01/rellic-headergen-1.png?ssl=1)

Once the utility finds a topological sort, it can inspect the types in this order with the confidence that the type of any of the fields has been fully defined.

Struct shenanigans
------------------

The DWARF metadata provides a few pieces of information we can use to recover C structure definitions for the types it describes:

*   The size of the type
*   The type of each field
*   The offset of each field
*   Whether the type was originally a struct or a union

`rellic-headergen`’s reconstruction algorithm starts by sorting the fields in order of increasing offset, then defines a new struct in which to add each field. The metadata provides no information on whether the original definition was declared as packed or not, so `rellic-headergen` first tries to generate the layout directly, as specified by the metadata. If the resulting layout doesn’t match the one given as input, the utility starts from scratch and generates a packed layout instead, inserting padding manually as needed.

Now, we could use any number of sophisticated heuristics to decide the offset of each field from the start of the struct, but things can get quite hairy, especially in the case of bit fields. A better approach is to get this information from something that already has the logic worked out: a compiler.

Fortunately, `rellic-headergen` already uses Clang to generate the definitions. Unfortunately, querying Clang itself about the fields’ offsets is not quite so simple, as Clang allows the retrieval of layout information only for complete definitions. To get around this particular quirk of the API, the utility generates temporary struct definitions that contain all the fields up to the one it is currently processing.

Structs and inheritance
-----------------------

As I was working on more involved use cases, I stumbled upon some instances in which the ABI works in ways that are not immediately obvious. For example, handling C++ inheritance takes some care, as the naive approach is not always correct. Converting

```
struct A { int x; };
struct B : A { int y; };

```

into

```
struct A { int x; };
struct B {
    struct A base;
    int y;
};

```

seems like a good idea and works in practice, but this method doesn’t scale very well. For example, the following snippet cannot be converted in this way:

```
struct A { int x; char y; };
struct B : A { char z; };

```

The reason is that on a machine in which `int` is 4 `char`s wide, struct `A` typically contains 3 additional `char`s of padding after `y`. Thus, embedding struct `A` directly into `B` would put `z` at offset 8. In order to minimize the amount of padding in structs, compilers opt to place the fields of the derived type directly inside the base struct instead.

Furthermore, empty structs are technically not valid in C. They can be used via GCC and Clang extensions, and they are valid in C++, but they present an issue: an empty struct’s `sizeof` is never 0. Instead, it is typically 1. Among other reasons, this is so that in a code snippet like the following, every field is guaranteed to have separate addresses:

```
struct A {};
struct B {
    struct A a;
    int b;
};

```

The example above works perfectly fine, but there are places in which treating empty structs the naive way doesn’t work. Consider the following:

```
struct A {};
struct B : A {
    int x;
};

```

This example produces the following DWARF metadata:

```
!2 = !{}
!10 = distinct !DICompositeType(
    tag: DW_TAG_structure_type, name: "A", size: 8, elements: !2)
!11 = distinct !DICompositeType(
    tag: DW_TAG_structure_type, name: "B", size: 32, elements: !12)
!12 = !{!13, !14}
!13 = !DIDerivedType(tag: DW_TAG_inheritance, baseType: !10)
!14 = !DIDerivedType(tag: DW_TAG_member, name: "x", baseType: !15, size: 32)
!15 = !DIBasicType(name: "int", size: 32, encoding: DW_ATE_signed)

```

If we followed the same logic for `DW_TAG_inheritance` as we did for `DW_TAG_member`, we’d end up with this conversion:

```
struct A {};
struct B {
    struct A a;
    int b;
};

```

This is not equivalent to the original definition! Field `b` would end up at an offset different from 0, as fields cannot have size 0. Getting all of these C++ details working was challenging but worthwhile. Now we can use `rellic-headergen` to convert arbitrary C++ types into plain old C types. Many reverse engineering tools embed some form of basic C parsing support in order for a user to provide “type libraries,” which describe the types used by machine code. These basic parsers typically don’t have any C++ support, and so `rellic-headergen` bridges this gap.

What’s next for rellic-headergen?
---------------------------------

There are opportunities to further improve `rellic-headergen`. One of the objectives of the utility is to be able to recover field access patterns from code that has been optimized. Consider the following program:

```
struct A {
    char a, b, c, d;
};

char test(struct A x) { return x.c; }

```

This program produces the following bitcode:

```
define dso_local signext i8 @test(i32 %x.coerce) local_unnamed_addr #0 {
entry:
    %x.sroa.1.0.extract.shift = lshr i32 %x.coerce, 16
    %x.sroa.1.0.extract.trunc = trunc i32 %x.sroa.1.0.extract.shift to i8
    ret i8 %x.sroa.1.0.extract.trunc
}


```

In this bitcode, the original information about the structure of `x` has been lost. Essentially, if Clang/LLVM performs optimizations before emitting bitcode or lifting bitcode from compiled machine code, this could cause the resulting bitcode to be too low level, creating a mismatch between the type information found in the debug metadata and the information in the bitcode itself. In this case, `rellic-headergen` cannot resolve this mismatch on its own. Improving the utility to be able to resolve these issues in the future would be beneficial; knowing the exact layout of structs can be useful when trying to match bit shifts and masks to field accesses in order to produce decompiled code that is as close to the original as possible.

Also, languages that employ different DWARF features are not handled as well by `rellic-headergen`. Rust, for example, uses an ad hoc representation for discriminated unions, which is difficult for the utility to handle. There is an opportunity to one day add functionality to the utility to handle DWARF features such as these.

Finally, another future `rellic-headergen` feature worth exploring is the possibility to change the output language: sometimes you _do_ want to keep that inheritance information as C++, after all!

Closing thoughts
----------------

Although `rellic-headergen` currently has a very narrow scope, it is already incredibly robust when working with C and C++ codebases, as it is able to extract type information for `rellic` itself, which includes LLVM and Clang. It already provides useful insights when navigating binaries that have been built with debugging information, but expanding its set of features to be able to extract information from more varied codebases will make it even more useful when dealing with bigger projects.

Working on `rellic-headergen` was very fun, interesting, and instructive. I am grateful to Trail of Bits for the opportunity to work on such an innovative project with talented people. This was a great learning experience, and I would like to thank my mentor Peter Goodman for giving me guidance with almost free reign over the project, and Marek Surovič for his patience in sharing his experience in `rellic` with me.
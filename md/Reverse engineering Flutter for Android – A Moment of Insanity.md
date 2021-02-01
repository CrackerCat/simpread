> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [rloura.wordpress.com](https://rloura.wordpress.com/2020/12/04/reversing-flutter-for-android-wip/)

_Disclaimer_: the contents of this article are the result of countless hours of personal investigation combined with exhaustive trial and error. I have never contacted Flutter or Dart development team members, nor have I ever used Flutter or Dart as development tools. As such, although I’ve tried my best, I cannot absolutely guarantee that all the information below is 100% accurate.

_Notice_: as usual, technical expressions in _italic_ are used for terms that are yet to be defined, whereas **bold** is used when defining them.

Introduction
------------

[Flutter](https://flutter.dev/) is a recent UI toolkit developed by Google from the ground up, and designed to streamline the creation of beautiful, natively compiled cross-platform apps. To do so, Flutter builds on top of the open-source [Dart VM](https://github.com/dart-lang/sdk), and development is done using the [Dart](https://dart.dev/) programming language. When an Android app is built using Flutter, two special libraries are generated for each supported architecture, and stored in the standard `lib/` directory inside the [APK](https://en.wikipedia.org/wiki/Android_application_package).

```
flutter-app.apk
├── ...
└── lib/    ├── arm64-v8a
    │   ├── libapp.so
    │   └── libflutter.so
    ├── armeabi-v7a
    │   ├── libapp.so
    │   └── libflutter.so
    └── x86_64
        ├── libapp.so
        └── libflutter.so
├──
└── lib/

```

From a reverse engineering perspective, `libapp.so` is the only interesting file, as it contains all the compiled Dart code written during development, whereas `libflutter.so` simply packs the runtime engine. The rest of this article will be dedicated to understanding the layout of this `libapp.so`, and the _snapshot_s contained therein.

Key concepts
------------

#### Dart fundamentals

Before we start, let us define some concepts. Every piece of Dart code runs within an **isolate**, a structure composed of, among other things, a chunk of memory called its **heap**. Each isolate heap manages (allocates, garbage collects, …) all the necessary objects for the code to run, and may not reference objects in the heap of another isolate, with one exception: the _VM isolate_. The **VM isolate** is a special isolate, sometimes called the **pseudo-isolate**, whose heap may be referenced by objects residing in a different isolate. The reason why this doesn’t break basic separation principles is that the VM isolate only manages immutable objects, such as the _null_ object or the _empty array_. Making it accessible to other isolates is thus a clear advantage, without any drawbacks. Even though Dart may generally be running multiple isolates concurrently, this is not exploited by Flutter, which only uses one isolate, apart from the ever-present VM isolate.

Flutter for Android can be used in one of two ways: [AOT](https://en.wikipedia.org/wiki/Ahead-of-time_compilation) or [JIT](https://en.wikipedia.org/wiki/Just-in-time_compilation). The latter is used in debug releases, and its on-the-fly compilation is the basis of the [hot reload](https://flutter.dev/docs/development/tools/hot-reload) functionality. The former is used in production releases. It is unlikely that a JIT-compiled app will be encountered in the wild, and AOT will thus be the focus of this article.

Dart code in a JIT-compiled Android app is run directly from source code, whereas Dart code in an AOT-compiled app runs from a _snapshot_. A **snapshot** is the serialized state of the Dart VM at some point of its execution, including all natively compiled code. In Flutter, the isolate snapshot corresponds to the state of the Dart VM right before `main` is called. This snapshot format is the main focus of this article, as it contains the isolate’s heap and code.

#### Integer encoding

There is one extra piece of information necessary to understanding the format. Some integers in Dart are encoded using a slightly modified variant of a variable-length encoding known as [LEB128](https://en.wikipedia.org/wiki/LEB128). Decoding an **unsigned** integer is easy: read bytes up to and including a byte that has its most significant bit, here called the **marker** bit, set (equivalently, whose value is greater than `0x7f`). Now, reverse the order of the bytes. For each byte read, discard its marker bit by simply deleting it. You are left with the value of the integer.

As an example, consider this sequence of bytes: `0x7900 7487 4e46 8301 826d 9900`, and suppose we know there is an encoded unsigned at offset 4. To decode it, first read bytes until one of them has a value greater than `0x7f`, which yields (starting at offset 4): 0x 4e 46 83. Now, reverse the order of the bytes, resulting in: `0x83 46 4e`, or, in binary: `0b10000011 01000110 01001110`. Finally, delete the marker of each byte, so that you’re left with: `0b0000011 1000110 1001110`, which is 58109 in decimal.

To read a **signed** integer, do the exact same, but, in the very last step, when reading the actual value, interpret the most significant bit as a sign bit. The reason why this is a variant of LEB128 is that “true” LEB128 has every marker bit set except in the last byte, and not the other way around.

#### Layout

The `libapp.so` file of a Flutter production release contains two **snapshots**: one for the ubiquitous VM isolate, and another for the isolate with actual substance. Each of these is split into a data section and an instructions section. The data section contains the isolate’s heap, whereas the instructions section contains the natively compiled code. Since the compiled code needs to be loaded into executable memory, it is placed in the `.text` section of the ELF file. The rest of the isolate’s heap, containing the VM object descriptions, is placed in the non-executable `.rodata` section.

```
libapp.so
├── ...│
├── .text
│   └── _kDartVmSnapshotInstructions
├── .rodata│   └── _kDartVmSnapshotData
├── .text.2
│   └── _kDartIsolateSnapshotInstructions
└── .rodata.2    └── _kDartIsolateSnapshotData
├── ...
├── .rodata
└── .rodata.2

```

Snapshot format
---------------

#### Header

With a solid overview of the ELF file contents, it is now time to dig deeper and understand the structure of a Dart snapshot. The very first bytes of a Dart snapshot are a header, whose format is rather simple, and is probably best described using a classic table. The exact meaning of each entry is described below, along with several important definitions.

<table><thead><tr><th data-align="left"><strong>Offset</strong></th><th data-align="left"><strong>Size or type</strong></th><th data-align="left">Name</th><th data-align="left">Description</th></tr></thead><tbody><tr><td data-align="left">0</td><td data-align="left">4</td><td data-align="left">Magic number</td><td data-align="left">The snapshot’s magic number.<br>This is the constant value <code>0xf5f5dcdc</code>.</td></tr><tr><td data-align="left">4</td><td data-align="left">8</td><td data-align="left">Size</td><td data-align="left">The size of the snapshot, in bytes.</td></tr><tr><td data-align="left">12</td><td data-align="left">8</td><td data-align="left">Kind</td><td data-align="left">The <em>kind</em> of the VM snapshot.</td></tr><tr><td data-align="left">20</td><td data-align="left">32</td><td data-align="left">Version</td><td data-align="left">The <em>version</em> of the compiler used.</td></tr><tr><td data-align="left">52</td><td data-align="left">string</td><td data-align="left">Features</td><td data-align="left">A null-terminated string of features.</td></tr><tr><td data-align="left"></td><td data-align="left">unsigned</td><td data-align="left">Base objects</td><td data-align="left">The number of <em>base objects</em>.</td></tr><tr><td data-align="left"></td><td data-align="left">unsigned</td><td data-align="left">Objects</td><td data-align="left">The total number of objects.</td></tr><tr><td data-align="left"></td><td data-align="left">unsigned</td><td data-align="left">Clusters</td><td data-align="left">The number of <em>clusters</em>.</td></tr><tr><td data-align="left"></td><td data-align="left">unsigned</td><td data-align="left">Field table length</td><td data-align="left">The length of the <em>field table</em>.</td></tr></tbody></table>The Dart VM snapshot header

*   **Magic number**: the constant value `0xf5f5dcdc`, used to easily identify the beginning of a Dart snapshot.
*   **Size**: the size of the snapshot within the ELF file, in bytes (not including the 4 bytes of the previous field).
*   **Kind**: a number identifying the _kind_ of the snapshot. Flutter only uses the AOT or JIT **kind**s, but other kinds exist. They are: _full_, _message_, _none_ and _invalid_ (see the `Kind` enumeration in [`snapshot.h`](https://github.com/dart-lang/sdk/blob/7c148d029de32590a8d0d332bf807d25929f080e/runtime/vm/snapshot.h)). As mentioned previously, production releases always have the AOT kind (constant value 2), whereas debug releases use the JIT kind (constant value 1).
*   **Version**: the 32-byte ASCII encoding of the hex-encoded MD5 hash of the concatenation of source code files responsible for describing the format (convoluted, I know). The specific set of files, as well as the formula for computing the version, can be found in `[make_version.py](https://github.com/dart-lang/sdk/blob/7c148d029de32590a8d0d332bf807d25929f080e/tools/make_version.py)`. Having a version computed this way makes it easy to detect compatibility issues, and ensures that, if any file that directly handles the format changes, the version will automatically be updated. As a major drawback, given the version, finding the Dart release(s) corresponding to it is non-trivial. For reference, all Dart 2.10 releases (the latest production release at the time of writing) have the version `3865 6534 6566 3761 3637 6466 3938 3435 6662 6133 3331 3733 3431 3938 6139 3533`, or, after ASCII decoding, `8ee4ef7a67df9845fba331734198a953`.
*   **Features**: a null-terminated ASCII string of space-separated features. These are used to identify whether the snapshot is of a production or debug version (although Flutter uses JIT for debug, Dart in general supports debugging AOT-compiled apps), as well as the target architecture and other features. A typical such string is shown in the example below.
*   **Base objects**: some objects, such as the self-explanatory _null_ and _empty array_ objects, are always implicitly assumed to exist, and thus not written into the ELF file. These objects are defined in [`clustered_snapshot.cc`](https://github.com/dart-lang/sdk/blob/7c148d029de32590a8d0d332bf807d25929f080e/runtime/vm/clustered_snapshot.cc), under `VMSerializationRoots::AddBaseObjects` (look for `AddBaseObjects`, it’s the first hit). For the `_kDartVmSnapshotData` section, this header field defines the number of such objects. For the `_kDartIsolateSnapshotData`, this header field corresponds to the total number of objects (see next field) defined in `_kDartVmSnapshotData`, and made accessible to the isolate (see previous section).
*   **Objects**: the total number of objects (base objects and otherwise) used by the snapshot.
*   **Clusters**: the number of different Dart types present in the snapshot (see the Clusters subsection below for more details).
*   **Field table length**: not used in this article.

As an example, consider this header of a Dart VM snapshot extracted from a simple Flutter app.

```
f5f5 dcdc 0e79 0b00 0000 0000 0200 0000
0000 0000 3865 6534 6566 3761 3637 6466
3938 3435 6662 6133 3331 3733 3431 3938
6139 3533 7072 6f64 7563 7420 6e6f 2d64
7761 7266 5f73 7461 636b 5f74 7261 6365
735f 6d6f 6465 206e 6f2d 6361 7573 616c
5f61 7379 6e63 5f73 7461 636b 7320 6c61
7a79 5f61 7379 6e63 5f73 7461 636b 7320
6e6f 2d6c 617a 795f 6469 7370 6174 6368
6572 7320 7573 655f 6261 7265 5f69 6e73
7472 7563 7469 6f6e 7320 6465 6475 705f
696e 7374 7275 6374 696f 6e73 206e 6f2d
2261 7373 6572 7473 2220 7836 342d 7379
7376 206e 6f2d 6e75 6c6c 2d73 6166 6574
7900 7487 4e46 8301 826d 99

```

*   Magic number: `0xf5f5dcdc`.
*   Size: 751890 bytes (little-endian integer, not encoded).
*   Kind: 2 (AOT).
*   Version: 8ee4ef7a67df9845fba331734198a953.
*   Features: product no-dwarf_stack_traces_mode no-causal_async_stacks lazy_async_stacks no-lazy_dispatchers use_bare_instructions dedup_instructions no-“asserts” x64-sysv no-null-safety.
*   Base objects: 1012.
*   Objects: 58190.
*   Clusters: 257.
*   Field table length: 3309.

#### Object serialization

Every Dart object of a given type can be decomposed into a fixed set of fields. Each field either has a definite value, or a _reference_ to another Dart object. For instance, integers in Dart have up to 64 bits and the type `Mint`. Every object of type `Mint` can be described with exactly two fields: a flag indicating whether or not it is a _canonical_ object, and the actual value of the `Mint`. It may be helpful to visualize a `Mint` in terms of a high-level implementation in a generic high-level OOP language.

```
class Mint {
    bool isCanonical;
    long value;
}

```

The serialized version of a `Mint` can be obtained by joining the encoding of the boolean (0x00 for false, and 0x01 for true) with the signed encoding of the `value`. The serialization of the non-_canonical_ `Mint` with value 7 would thus be: 0x 00 87.

A slightly more complicated type is the `Array` type. This type can be described with four fields: the length `l` of the array, a flag indicating whether or not it is a _canonical_ object, a _reference_ to an object of type `TypeArguments` that determines the types of the objects the array holds, and `l` _reference_s, one for each one of the array element objects. Once more, visualizing the `Array` type in a high-level OOP language would yield something like the following.

```
class Array {
    int length;
    bool isCanonical;
    TypeArguments typeArguments; // A reference to an object of type TypeArguments, to be exact
    Object[length] data; // An array of references to objects, to be exact
}

```

The `length` is serialized as an unsigned, and the `isCanonical` field is serialized as a boolean, just like before. Each **reference** is also just an unsigned, and its value is the _reference ID_ of the target object. During the serialization process, each serialized object is given a _reference ID_. The **reference ID** of the _n_-th serialized object is simply _n_.

In the above example, the field `typeArguments` could have the value 1, meaning that the first object to be (de)serialized is the `TypeArguments` object the array refers to. Similarly, the `data` array could hold, say, two references, 2 and 3, to two `Mint`s, with values 7 and 12, for instance. The serialization of such a non-_canonical_ `Array` would be : 0x 82 00 81 82 83.

#### Clusters

A **cluster** is the set of objects in a snapshot that share a common Dart type. During serialization, clusters are serialized sequentially, and each one of them takes care of serializing every object in it. There are over 150 different types in Dart, each associated with a number called its **class ID**, and each one with its own cluster serialization procedure. These class IDs are defined in the `ClassId` enumeration, over at `[class_id.h](https://github.com/dart-lang/sdk/blob/7c148d029de32590a8d0d332bf807d25929f080e/runtime/vm/class_id.h)`. The serialization process of all clusters as a whole is done in two separate stages (three, to be precise, but the _trace_ stage plays no role in the deserialization process, and can be safely ignored here): the _alloc_ stage, and the _fill_ stage.

The **alloc** stage consists of iterating over all clusters, and attributing reference IDs to every object they contain, as well as serializing basic information about each cluster. Most often, this basic information is simply the class ID of the cluster followed by the number of objects the cluster contains, or similar. In some cases, such as in the `Mint` case, it may actually include the serialized objects already. The concrete definition of the alloc stage for each cluster can be found in `[clustered_snapshot.cc](https://github.com/dart-lang/sdk/blob/7c148d029de32590a8d0d332bf807d25929f080e/runtime/vm/clustered_snapshot.cc)` (each `SerializationCluster` has a different alloc stage for each of the 150+ types).

Once the alloc stage is finished, and some cluster information is already serialized, the **fill** stage kicks in. Once more, the process consists of iterating over all clusters, and this time serializing every object in it (with some exceptions, such as the `Mint` case, that were already taken care of during the alloc stage), just as shown in the previous section.

If one were to serialize a snapshot containing only the `Array` with two `Mint`s, it would look like the following after the header.

```
.... b582 0087 008c ce81 82
.... 8200 8182 83

```

*   `....`: type arguments alloc stage (here ignored).
*   `b582 0087 008c`: mint alloc stage (class ID: `b5`, count: `82`, followed by the serialization of the non-_canonical_ `Mint`s 7 and 12).
*   `ce81 82`: array alloc stage (class ID: `ce`, count: `81`, length(s): `82` – only one `Array` with length 2).
*   `....`: type arguments fill stage (here ignored).
*   `Mint` fill stage is empty (both `Mint`s have already been serialized during the `alloc` stage).
*   `8200 8182 83`: array fill stage (serialization of the `Array` – length: `82`, canonical: `false`, type argument reference ID: `81`, Mint #1 reference ID: `82`, Mint #2 reference ID: `83`).

The above example is a simplified version of a possible snapshot, with only 3 clusters and 4 objects (the `TypeArguments`, the two `Mint`s and the `Array`). In an actual snapshot, such a serialization would be missing references, for the `TypeArguments` also needs to reference some extra objects of its own. Nonetheless, the process would be the same, just longer and too cumbersome to read.

#### Code objects

One of the many Dart types is the `Code` type. Objects of this type are of particular significance, as they pinpoint the address of the natively compiled methods created during development. Although they have a fairly complex structure, suffice to say they contain an offset into the `_kDartIsolateSnapshotInstructions` section of the ELF where the function may be found. The code of this function is virtualized code, but although not immediate, it is not too difficult to reverse engineer.

The Dart VM keeps track of resources such as strings in dedicated registers that remain unchanged during execution. Enumerating each of these resources and registers for every supported architecture is currently out of scope of this article. As such, the focus here will instead be on the general technique one can use to reverse virtualized snippets. Consider the following snippet, taken from a real Flutter application targeting ARM64-v8:

```
002238d4 60  2b  40  91    add        x0, x27, #0xa, LSL #12
002238d8 00  0c  44  f9    ldr        x0, [x0, #0x818]
002238dc ef  03  1d  aa    mov        x15, x29
002238e0 fd  79  c1  a8    ldp        x29, x30, [x15], #0x10
002238e4 c0  03  5f  d6    ret

```

Ghidra decompiles this to

```
return *(undefined8 *)(unaff_x27 + 0xa818);

```

where `unaff_x27` stands for the contents of register `x27`. In ARM64-v8, `x27` is one of the immutable registers mentioned earlier. It holds a pointer to an array of pointers to resources. Using [Frida](https://frida.re/), and the [Medusa](https://github.com/Ch0pin/medusa) wrapper, one can hook the function this snippet lives in, and read the contents of the register. In a given run of the app, `x27` held the value `0x74f0138f40`. One can thus read the entry at offset `0xa818` in this array, to obtain a new address: `0x74fdea4c41`. Finally, a read at address `0x74fdea4c41` yields:

```
74fdea4c41 38 02 51 00 67 1c f2 1f 8.Q.g...
74fdea4c49 0a 00 00 00 00 00 00 00 ........
74fdea4c51 54 69 74 6c 65 00 00 00 Title...
74fdea4c59 00 00 00 00 00 00 00 00 ........

```

which one recognizes to be the serialized version of the string “Title”. The same technique can be applied to other resources.

Doldrums
--------

Doldrums is a snapshot parser for Dart version 2.10, and can be found over at [https://github.com/rscloura/Doldrums](https://github.com/rscloura/Doldrums). When used, it recovers all Dart classes and methods, and indicates the offset into the `libapp.so` file where the code for each method may be found. The native code can then be reversed using the techniques shown in the last section.

Credits
-------

Although much of this work is the result of my own research, I probably wouldn’t have managed without the following:

*   [Reverse engineering Flutter apps (Part 1)](https://blog.tst.sh/reverse-engineering-flutter-apps-part-1/): an article by Andre Lipke that served as a bootstrap to this project.
*   [Darter](https://github.com/mildsunrise/darter): a Flutter parser by Alba Mendez for Dart version 2.5 releases, that I wish I had found earlier.
*   [Introduction to Dart VM](https://mrale.ph/dartvm/): an unfortunately incomplete article containing very detailed information about the Dart VM.
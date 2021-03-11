> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [tinyhack.com](https://tinyhack.com/2021/03/07/reversing-a-flutter-app-by-recompiling-flutter-engine/)

It is not easy to reverse engineer a release version of a flutter app because the tooling is not available and the flutter engine itself changes rapidly. As of now, if you are lucky, you can dump the classes and method names of a flutter app using [darter](https://github.com/mildsunrise/darter) or [Doldrums](https://github.com/rscloura/Doldrums) if the app was built with a specific version of Flutter SDK.

If you are extremely lucky, which is what happened to me the first time I needed to test a Flutter App: you don’t even need to reverse engineer the app. If the app is very simple and uses a simple HTTPS connection, you can test all the functionalities using intercepting proxies such as Burp or Zed Attack Proxy. The app that I just tested uses an extra layer of encryption on top of HTTPS, and that’s the reason that I need to do actual reverse engineering.

In this post, I will only give examples for the Android platform, but everything written here is generic and applicable to other platforms. The TLDR is: instead of updating or creating a snapshot parser, we just recompile the flutter engine and replace it in the app that we targeted.

Flutter compiled app
--------------------

Currently several articles and repositories that I found regarding Flutter reverse engineering are:

*   [Reverse engineering Flutter for Android](https://rloura.wordpress.com/2020/12/04/reversing-flutter-for-android-wip/) (explains the basic of snapshot format, introduces, Doldrums, as of this writing only supports snapshot version 8ee4ef7a67df9845fba331734198a953)
*   [Reverse engineering Flutter apps (Part 1)](https://blog.tst.sh/reverse-engineering-flutter-apps-part-1/) a very good article explaining Dart internals, unfortunately, no code is provided and part 2 is not yet available as of this writing
*   [darter: Dart snapshot parser](https://github.com/mildsunrise/darter) a tool to dump snapshot version c8562f0ee0ebc38ba217c7955956d1cb

The main code consists of two libraries `libflutter.so` (the flutter engine) and `libapp.so` (your code). You may wonder: what actually happens if you try to open a `libapp.so` (Dart code that is AOT compiled) using a standard disassembler. It’s just native code, right? If you use IDA, initially, you will only see this bunch of bytes.

[![](https://tinyhack.com/wp-content/uploads/2021/03/ida64_9LzjRpWnmW-1024x694.png)](https://tinyhack.com/wp-content/uploads/2021/03/ida64_9LzjRpWnmW.png)

If you use other tools, such as binary ninja which will try to do some linear sweep, you can see a lot of methods. All of them are unnamed, and there are no string references that you can find. There is also no reference to external functions (either libc or other libraries), and there is no syscall that directly calls the kernel (like Go)..

With a tool like Darter dan Doldrums, you can dump the class names and method names, and you can find the address of where the function is implemented. Here is an example of a dump using Doldrums. This helps tremendously in reversing the app. You can also use Frida to hook at these addresses to dump memory or method parameters.

[![](https://tinyhack.com/wp-content/uploads/2021/03/image-1024x601.png)](https://tinyhack.com/wp-content/uploads/2021/03/image.png)

The snapshot format problem
---------------------------

The reason that a specific tool can only dump a specific version of the snapshot is: the snapshot format is not stable, and it is designed to be run by a specific version of the runtime. Unlike some other formats where you can skip unknown or unsupported format, the snapshot format is very unforgiving. If you can’t parse a part, you can parse the next part.

Basically, the snapshot format is something like this: `<tag> <data bytes> <tag> <data bytes>` … There is no explicit length given for each chunk, and there is no particular format for the header of the tag (so you can’t just do a pattern match and expect to know the start of a chunk). Everything is just numbers. There is no documentation of this snapshot, except for the source code itself.

In fact, there is not even a version number of this format. The format is identified by a snapshot version string. The version string is generated from _hashing the source code of snapshot-related files_. It is assumed that if the files are changed, then the format is changed. This is true in most cases, but not always (e.g: if you edit a comment, the snapshot version string will change).

My first thought was just to modify Doldrums or Darter to the version that I needed by looking at the diff of Dart sources code. But it turns out that it is not easy: enums are sometimes inserted in the middle (meaning that I need to shift all constants by a number). And dart also uses extensive bit manipulation using C++ template. For example, when I look at Doldums code, I saw something like this:

```
def decodeTypeBits(value):
       return value & 0x7f


```

I thought I can quickly check this constant in the code (whether it has changed or not in the new version), the type turns out to be not a simple integer.

```
class ObjectPool : public Object {
 using TypeBits = compiler::ObjectPoolBuilderEntry::TypeBits;
}
struct ObjectPoolBuilderEntry {
  using TypeBits = BitField<uint8_t, EntryType, 0, 7>;
}



```

You can see that this `Bitfield` is implemented as [BitField template class](https://dart.googlesource.com/sdk//+/2beb05b829f8bcf188ff05d9c8c8423dde191604/runtime/vm/bitfield.h). This particular bit is easy to read, but if you see `kNextBit`, you need to look back at all previous bit definitions. I know it’s not that hard to follow for seasoned C++ developers, but still: to track these changes between versions, you need to do a lot of manual checks.

My conclusion was: I don’t want to maintain the Python code, the next time the app is updated for retesting, they could have used a newer version of Flutter SDK, with another snapshot version. And for the specific work that I am doing: I need to test two apps with two different Flutter versions: one for something that is already released in the app store and some other app that is going to be released.

Rebuilding Flutter Engine
-------------------------

The flutter engine (`libflutter.so`) is a separate library from `libapp.so` (the main app logic code), on iOS, this is a separate framework. The idea is very simple:

*   Download the engine version that we want
*   Modify it to print Class names, Methods, etc **instead of writing our own snapshot parser**
*   Replace the original `libflutter.so` library with our patched version
*   Profit

The first step is already difficult: how can we find the corresponding snapshot version? This [table from darter](https://github.com/mildsunrise/darter/blob/master/info/versions.md) helps, but is not updated with the latest version. For other versions, we need to hunt and test if it has matching snapshot numbers. The instruction for [recompiling the Flutter engine is here](https://github.com/flutter/flutter/wiki/Compiling-the-engine), but there are some hiccups in the compilation and we need to modify the python script for the snapshot version. And also: the Dart internal itself is not that easy to work with.

Most older versions that I tested can’t be compiled correctly. You need to edit the DEPS file to get it to compile. In my case: the diff is small but I need to scour the web to find this. Somehow the specific commit was not available and I need to use a different version. Note: don’t apply this patch blindly, basically check these two things:

*   If a commit is not available, find nearest one from the release date
*   If something refers to a `_internal` you probably should remove the `_internal` part.

```
diff --git a/DEPS b/DEPS
index e173af55a..54ee961ec 100644
--- a/DEPS
+++ b/DEPS
@@ -196,7 +196,7 @@ deps = {
    Var('dart_git') + '/dartdoc.git@b039e21a7226b61ca2de7bd6c7a07fc77d4f64a9',

   'src/third_party/dart/third_party/pkg/ffi':
-   Var('dart_git') + '/ffi.git@454ab0f9ea6bd06942a983238d8a6818b1357edb',
+   Var('dart_git') + '/ffi.git@5a3b3f64b30c3eaf293a06ddd967f86fd60cb0f6',

   'src/third_party/dart/third_party/pkg/fixnum':
    Var('dart_git') + '/fixnum.git@16d3890c6dc82ca629659da1934e412292508bba',
@@ -468,7 +468,7 @@ deps = {
   'src/third_party/android_tools/sdk/licenses': {
      'packages': [
        {
-        'package': 'flutter_internal/android/sdk/licenses',
+        'package': 'flutter/android/sdk/licenses',
         'version': 'latest',
        }
      ],


```

Now we can start editing the snapshot files to learn about how it works. But as mentioned early: if we modify the snapshot file: the snapshot hash will change, so we need to fix that by returning a static version number in `third_party/dart/tools/make_version.py`. If you touch any of these files in `VM_SNAPSHOT_FILES`, change the line `snapshot_hash = MakeSnapshotHashString()` with a static string to your specific version.

[![](https://tinyhack.com/wp-content/uploads/2021/03/image-1-1024x601.png)](https://tinyhack.com/wp-content/uploads/2021/03/image-1.png)

What happens if we don’t patch the version? the app won’t start. So after patching (just start by printing a hello world) using `OS::PrintErr("Hello World")` and recompiling the code, we can test to replace the .so file, and run it.

I made a lot of experiments (such as trying to `FORCE_INCLUDE_DISASSEMBLER`), so I don’t have a clean modification to share but I can provide some hints of things to modify:

*   in `runtime/vm/clustered_snapshot.cc` we can modify `Deserializer::ReadProgramSnapshot(ObjectStore* object_store)` to print the class table `isolate->class_table()->Print()`
*   in `runtime/vm/class_table.cc` we can modify `void ClassTable::Print()` to print more informations

For example, to print function names:

```
 const Array& funcs = Array::Handle(cls.functions());  
 for (intptr_t j = 0; j < funcs.Length(); j++) {
      Function& func = Function::Handle();
      func = cls.FunctionFromIndex(j);
      OS::PrintErr("Function: %s", func.ToCString());
}


```

Sidenote: SSL certificates
--------------------------

Another problem with Flutter app is: it [won’t trust a user installed root cert](https://github.com/flutter/flutter/issues/41781). This a problem for pentesting, and someone made a note on how to patch the binary (either directly or using Frida) to workaround this problem. Quoting TLDR of this [blog post](https://blog.nviso.eu/2019/08/13/intercepting-traffic-from-android-flutter-applications/):

*   Flutter uses Dart, which doesn’t use the system CA store
*   Dart uses a list of CA’s that’s compiled into the application
*   Dart is not proxy aware on Android, so use ProxyDroid with iptables
*   Hook the [session_verify_cert_chain](https://github.com/google/boringssl/blob/master/ssl/ssl_x509.cc#L362) function in x509.cc to disable chain validation

By recompiling the Flutter engine, this can be done easily. We just modify the source code as-is (`third_party/boringssl/src/ssl/handshake.cc`), without needing to find assembly byte patterns in the compiled code.

Obfuscating Flutter
-------------------

It is possible to obfuscate Flutter/Dart apps using the [instructions provided here.](https://flutter.dev/docs/deployment/obfuscate) This will make reversing to be a bit harder. Note that only the names are obfuscated, there is no advanced control flow obfuscation performed.

Conclusion
----------

I am lazy, and recompiling the flutter engine is the shortcut that I take instead of writing a proper snapshot parser. Of course, others have similar ideas of hacking the runtime engine when reversing other technologies, for example, to reverse engineer an obfuscated PHP script, you can hook `eval` using a PHP module.
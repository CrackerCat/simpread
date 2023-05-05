> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [infosecwriteups.com](https://infosecwriteups.com/flutter-hackers-uncovering-the-devs-myopia-part-2-598a44942b5e)

> Deep dive in reverse engineering Flutter APK Release Mode with Frida

Deep dive in reverse engineering Flutter APK Release Mode with Frida
--------------------------------------------------------------------

![](https://miro.medium.com/v2/resize:fit:875/1*L4EToKunVM3YI6ScSfAVWg.png)

> Not all Flutter Applications are hard to be statically analyzed and to spot a vulnerability itself, we may also began to include the used package as an in-scope variable of the black-box testing demand.

![](https://miro.medium.com/v2/resize:fit:875/1*g9fI2kArRaM6CqFje_ieOA.png)

In the [previous post](https://medium.com/bugbountywriteup/flutter-hackers-uncovering-the-devs-myopia-part-1-6c316be56b13), I’ve introduced how the relations between Dart and Flutter are correlated each other and also one of the mistake on how the Developer should’ve spotted on their application building phase. If you haven’t read about it, I’d recommend you to read it first especially if you’ve never heard about this application framework.

There’s a company which I consider as a startup and invited me to do some black-box testing on their recent web server and APK on their private program. I was not sure about it since once I heard that **Flutter** is their choice on how their APK was built on top of it. Yet, there’s always a **first-time** for everything so I took a chance on scrutinizing the application.

Within approximately 4–5 days, One of the vulnerability that I’ve discovered is pretty unique but the cause is not from their design, in fact it was from the **public package** which they’ve been used. This vulnerability allows me to go inside a developer mode and shows a numerous PII inside of that APK. I also pointed out that it’s not a best practice at all to store a hardcoded “super-user” credentials even if it’s encrypted and it’d be the best if they’ve managed all the users not by a role-based to be distinguished like that, perhaps a certain unique token and a secure authorization would be the best for them to use and compared inside their server.

In this writing, I want to share how I discover that vulnerability and show the methodology which I used to assist me on uncovering the developer’s security myopia by recreating the scenario itself into a CTF challenge in ITS (ARA CTF 2023).

![](https://miro.medium.com/v2/resize:fit:875/1*7NItTxIVEGvJpNtWaNhLhA.png)

Suppose we’re an unauthorized users who don’t have a capability to access those private assets from certain user, or even a privileged ones. We need to know how the application works first, by taking the advantage of a lack of secure perimeter options when the developer tries to build the release-built of the APK itself.

```
flutter build apk --release

```

This command line does tell the framework to build the APK in `release` mode instead of `debug` mode. But what are the advantages that we, as a penetration tester to help us understanding what’s the application doing? They might’ve built the APK pretty secure, but not enough for a reverse engineer to tackle.

In the [Gradle](https://docs.gradle.org/current/userguide/what_is_gradle.html) `(gradle.properties)`, they might be forget to add this line due to their preferences or just don’t really care about exposing the function naming.

```
extra-gen-snapshot-options=--obfuscate

```

They could’ve also added the flags when the application is in building phase by:

```
flutter build apk --release --obfuscate

```

Why do we care on obfuscating the application? If we’re taking a perspective on the developer side, the application size can be reduced by the obfuscation. But they also need to enable a `--split-debug-info` so that they won’t even had a misunderstanding on analyzing the APK, simply debug the stack traces from the function symbols.

This might be the first `blunder` for the developer not viewing in a broad scope on tackling the reverse engineer thus making their life easier to understand what’s going on inside the application.

In this scenario, I’ll make the Flutter Release APK without the `obfuscate` flags which has the objective to retrieve a secret string which has been stored inside the APK (**hardcoded**) with a known [symmetric block encryption](https://www.cryptomathic.com/news-events/blog/symmetric-key-encryption-why-where-and-how-its-used-in-banking#:~:text=Symmetric%20encryption%20is%20a%20type,used%20in%20the%20decryption%20process.).

![](https://miro.medium.com/v2/resize:fit:863/0*VgTlJWWEb2XMPQ8X)Image taken from : [https://medium.com/@.Qubit/symmetric-key-algorithm-in-cryptography-3d839bba8613](https://medium.com/@.Qubit/symmetric-key-algorithm-in-cryptography-3d839bba8613)![](https://miro.medium.com/v2/resize:fit:875/1*v2s1JGuy4SNLzRt0vqQNsA.png)

We know that **Flutter** Snapshots (in the supported libraries) are in a different syntax formatting and it makes a reverse engineer life harder. But since they didn’t enable the build gradle properties of obfuscation, it may make things easier.

We’ll have a sneak peek first on the flutter application. We can install this APK on a real Android device or in a preferred Android emulator. There are various emulators which are free to use:

*   [**NoxPlayer**](https://www.bignox.com/)
*   [**LDPlayer**](https://ldplayer.net/)
*   [**Genymotion**](https://www.genymotion.com/) **(**with [**ARM Translation Packages**](https://github.com/m9rco/Genymotion_ARM_Translation)**)**
*   [**BlueStacks**](https://www.bluestacks.com/)
*   **AVD Manager from** [**Android Studio**](https://developer.android.com/studio)
*   [**QEMU**](https://www.qemu.org/)**-Based Android Emulator**

Once done downloading, we may proceed to install the APK using **adb** command.

```
adb install -g <path-to-the-apk.apk>

```

![](https://miro.medium.com/v2/resize:fit:875/1*FlzLWPuexRG4YiP4W0fkVw.png)

The design of the APK itself is pretty simple since it only prompts us an interface that accepts certain secret strings which will be validated after the user enters the correct string.

Now, first thing that we can do is to analyze the snapshot shared objects (**libapp.so**) using [IDA decompiler](https://hex-rays.com/ida-free/). We need to decompile the APK first using `apktool` . Next, we can check the **lib/** folder if there are two shared objects which consist of **libflutter.so** and **libapp.so**.

```
apktool d <your-path-to-flutter.apk>

```

![](https://miro.medium.com/v2/resize:fit:875/1*PjjKrKbQwbr2tRtCHr2dSA.png)

We’ll use the base **code offset** for **_kDartIsolateSnapshotInstructions** (0x1707f0). This will be useful once we’ve extracted each of the function & widgets offsets from the APK itself using the tool called [**reFlutter**](https://github.com/ptswarm/reFlutter) that will assist us for reversing this APK.

In order to use **reFlutter**, install it first as a **Python package** and simply locate the path to the APK. There’ll be two options which will be the choice of how we want the **reFlutter** to behave. This tool will patch the shared object libraries (ELF) of our APK and eventually may bind to port 8083 on all interfaces (including the invisible proxy mode). What we are seeking is the **dump.dart** file which will be produced after the patched APK is running and there’ll be the code offsets that we may leverage to use.

```
 if ver>27:
        replaceFileText('src/third_party/dart/runtime/vm/class_table.cc','::Print() {','::Print()  { OS::PrintErr("reFlutter");\n char pushArr[60000]="";\n')
        replaceFileText('src/flutter/BUILD.gn','  if (is_android) {\n    public_deps +=\n        [ "//flutter/shell/platform/android:flutter_shell_native_unittests" ]\n  }','')
        replaceFileText('src/third_party/dart/runtime/vm/class_table.cc','OS::PrintErr("%" Pd ": %s\\n", i, name.ToCString());','\n     auto& funcs = Array::Handle(cls.functions());    if (funcs.Length()>1000) {    continue;    } char classText[100000]="";    String& supname = String::Handle();     name = cls.Name(); strcat(classText,cls.ToCString());     Class& supcls = Class::Handle();    supcls = cls.SuperClass();     if (!supcls.IsNull()) {   supname = supcls.Name();    strcat(classText," extends ");   strcat(classText,supname.ToCString());  }    const auto& interfaces = Array::Handle(cls.interfaces()); auto& interface = Instance::Handle();    for (intptr_t in = 0;in < interfaces.Length(); in++) { interface^=interfaces.At(in); if(in==0){strcat(classText," implements ");}    if(in>0){strcat(classText," , ");}  strcat(classText,interface.ToCString()); }    strcat(classText," {\\n"); const auto& fields = Array::Handle(cls.fields());      auto& field = Field::Handle(); auto& fieldType = AbstractType::Handle();    String& fieldTypeName = String::Handle(); String& finame = String::Handle();    Instance& instance2 = Instance::Handle();    for (intptr_t f = 0; f < fields.Length(); f++)    {    field ^= fields.At(f); finame = field.name(); fieldType = field.type(); fieldTypeName = fieldType.Name(); strcat(classText,"  ");    strcat(classText,fieldTypeName.ToCString());  strcat(classText," "); strcat(classText,finame.ToCString());     if(field.is_static()){   instance2 ^= field.StaticValue();   strcat(classText," = ");     strcat(classText,instance2.ToCString());   strcat(classText," ;\\n");  }    else {   strcat(classText," = ");   strcat(classText," nonstatic;\\n");  } }     for (intptr_t c = 0; c < funcs.Length(); c++) {      auto& func = Function::Handle();    func = cls.FunctionFromIndex(c);     String& signature = String::Handle();    signature = func.InternalSignature();auto& codee = Code::Handle(func.CurrentCode());      if(!func.IsLocalFunction()) {    strcat(classText," \\n  "); strcat(classText,func.ToCString()); strcat(classText," ");    strcat(classText,signature.ToCString());    strcat(classText," { \\n\\n              ");   char append[70];   sprintf(append," Code Offset: _kDartIsolateSnapshotInstructions + 0x%016" PRIxPTR "\\n",static_cast<uintptr_t>(codee.MonomorphicUncheckedEntryPoint()));   strcat(classText,append);    strcat(classText,"       \\n       }\\n");    } else {    auto& parf = Function::Handle(); parf=func.parent_function();    String& signParent = String::Handle();       signParent = parf.InternalSignature();     strcat(classText," \\n  ");     strcat(classText,parf.ToCString()); strcat(classText," "); strcat(classText,signParent.ToCString());    strcat(classText," { \\n\\n          ");    char append[80];   sprintf(append," Code Offset: _kDartIsolateSnapshotInstructions + 0x%016" PRIxPTR "\\n",static_cast<uintptr_t>(codee.MonomorphicUncheckedEntryPoint()));   strcat(classText,append);    strcat(classText,"       \\n       }\\n");  } }       strcat(classText," \\n      }\\n\\n");      const Library& libr = Library::Handle(cls.library());if (!libr.IsNull()) {  auto& owner_class = Class::Handle(); owner_class = libr.toplevel_class();   auto& funcsTopLevel = Array::Handle(owner_class.functions());   char pushTmp[1000];   String& owner_name = String::Handle();   owner_name = libr.url();   sprintf(pushTmp,"\'%s\',",owner_name.ToCString());  if (funcsTopLevel.Length()>0&&strstr(pushArr, pushTmp) == NULL) {  strcat(pushArr,pushTmp);   strcat(classText,"Library:"); strcat(classText,pushTmp); strcat(classText," {\\n");         for (intptr_t c = 0; c < funcsTopLevel.Length(); c++) {      auto& func = Function::Handle();    func = owner_class.FunctionFromIndex(c);     String& signature = String::Handle();       signature = func.InternalSignature();   auto& codee = Code::Handle(func.CurrentCode());    if(!func.IsLocalFunction()) {    strcat(classText," \\n  "); strcat(classText,func.ToCString()); strcat(classText," ");    strcat(classText,signature.ToCString());    strcat(classText," { \\n\\n              ");   char append[70];   sprintf(append," Code Offset: _kDartIsolateSnapshotInstructions + 0x%016" PRIxPTR "\\n",static_cast<uintptr_t>(codee.MonomorphicUncheckedEntryPoint()));   strcat(classText,append);    strcat(classText,"       \\n       }\\n");    } else {    auto& parf = Function::Handle(); parf=func.parent_function();    String& signParent = String::Handle();       signParent = parf.InternalSignature();     strcat(classText," \\n  ");     strcat(classText,parf.ToCString()); strcat(classText," "); strcat(classText,signParent.ToCString());    strcat(classText," { \\n\\n          ");    char append[80];   sprintf(append," Code Offset: _kDartIsolateSnapshotInstructions + 0x%016" PRIxPTR "\\n",static_cast<uintptr_t>(codee.MonomorphicUncheckedEntryPoint()));   strcat(classText,append);    strcat(classText,"       \\n       }\\n");  }   }             strcat(classText," \\n      }\\n\\n");}}   struct stat entry_info;   int exists = 0;   if (stat("/data/data/", &entry_info)==0 && S_ISDIR(entry_info.st_mode)){    exists=1;   }         if(exists==1){    pid_t pid = getpid();    char path[64] = { 0 };    sprintf(path, "/proc/%d/cmdline", pid);        FILE *cmdline = fopen(path, "r");    if (cmdline) {          char chm[264] = { 0 };  char pat[264] = { 0 };        char application_id[64] = { 0 };          fread(application_id, sizeof(application_id), 1, cmdline);  sprintf(pat, "/data/data/%s/dump.dart", application_id);          do { FILE *f = fopen(pat, "a+");   fprintf(f, "%s",classText);   fflush(f);   fclose(f);   sprintf(chm,"/data/data/%s",application_id);  chmod(chm, S_IRWXU|S_IRWXG|S_IRWXO);  chmod(pat, S_IRWXU|S_IRWXG|S_IRWXO);   } while (0);        fclose(cmdline);    }   }                                       if(exists==0){           char pat[264] = { 0 };  sprintf(pat, "%s/Documents/dump.dart", getenv("HOME"));   OS::PrintErr("reFlutter dump file: %s",pat);     do { FILE *f = fopen(pat, "a+");   fprintf(f, "%s",classText);   fflush(f);   fclose(f);      } while (0);            }')
        #replaceFileText('src/third_party/dart/runtime/vm/class_table.cc','OS::PrintErr("%" Pd ": %s\\n", i, name.ToCString());','auto& funcs = Array::Handle(cls.functions());    if (funcs.Length()>1000) {    continue;    } char classText[65000]="";  String& supname = String::Handle();    name = cls.Name(); strcat(classText," "); strcat(classText,cls.ToCString());    Class& supcls = Class::Handle();    supcls = cls.SuperClass();    if (!supcls.IsNull()) {   supname = supcls.Name();   strcat(classText," extends ");   strcat(classText,supname.ToCString());  } const auto& interfaces = Array::Handle(cls.interfaces()); auto& interface = Instance::Handle(); for (intptr_t in = 0;in < interfaces.Length(); in++) { interface^=interfaces.At(in); if(in==0){strcat(classText," implements ");}    if(in>0){strcat(classText," , ");}  strcat(classText,interface.ToCString()); } strcat(classText," {\\n"); const auto& fields = Array::Handle(cls.fields());    auto& field = Field::Handle(); auto& fieldType = AbstractType::Handle();    String& fieldTypeName = String::Handle(); String& finame = String::Handle(); Instance& instance2 = Instance::Handle();  for (intptr_t f = 0; f < fields.Length(); f++) {    field ^= fields.At(f); finame = field.name(); fieldType = field.type(); fieldTypeName = fieldType.Name(); strcat(classText," "); strcat(classText,fieldTypeName.ToCString());  strcat(classText," "); strcat(classText,finame.ToCString());   if(field.is_static()){   instance2 ^= field.StaticValue();   strcat(classText," = ");   strcat(classText,instance2.ToCString());   strcat(classText," ;\\n");  } else {   strcat(classText," = ");   strcat(classText," nonstatic;\\n");  } }    for (intptr_t c = 0; c < funcs.Length(); c++) {      auto& func = Function::Handle();    func = cls.FunctionFromIndex(c);    String& signature = String::Handle();    signature = func.InternalSignature(); if(!func.IsLocalFunction()) { strcat(classText," \\n"); strcat(classText,func.ToCString()); strcat(classText," ");    strcat(classText,signature.ToCString()); strcat(classText," { \\n\\n                  }\\n"); } else { auto& parf = Function::Handle(); parf=func.parent_function(); String& signParent = String::Handle();    signParent = parf.InternalSignature(); strcat(classText," \\n"); strcat(classText,parf.ToCString()); strcat(classText," "); strcat(classText,signParent.ToCString()); strcat(classText," { \\n\\n                  }\\n"); } } OS::PrintErr("reflutter:\\n %s \\n      }\\n",classText);')
    else:
        replaceFileText('src/third_party/dart/runtime/vm/class_table.cc','::Print() {','::Print()  { OS::PrintErr("reFlutter");\n char pushArr[60000]="";\n')
        replaceFileText('src/third_party/dart/runtime/vm/class_table.cc','OS::PrintErr("%" Pd ": %s\\n", i, name.ToCString());','\n      auto& funcs = Array::Handle(cls.functions());    if (funcs.Length()>1000) {    continue;    } char classText[100000]="";    String& supname = String::Handle();     name = cls.Name(); strcat(classText,cls.ToCString());     Class& supcls = Class::Handle();    supcls = cls.SuperClass();     if (!supcls.IsNull()) {   supname = supcls.Name();    strcat(classText," extends ");   strcat(classText,supname.ToCString());  }    const auto& interfaces = Array::Handle(cls.interfaces()); auto& interface = Instance::Handle();    for (intptr_t in = 0;in < interfaces.Length(); in++) { interface^=interfaces.At(in); if(in==0){strcat(classText," implements ");}    if(in>0){strcat(classText," , ");}  strcat(classText,interface.ToCString()); }    strcat(classText," {\\n"); const auto& fields = Array::Handle(cls.fields());      auto& field = Field::Handle(); auto& fieldType = AbstractType::Handle();    String& fieldTypeName = String::Handle(); String& finame = String::Handle();    Instance& instance2 = Instance::Handle();    for (intptr_t f = 0; f < fields.Length(); f++)    {    field ^= fields.At(f); finame = field.name(); fieldType = field.type(); fieldTypeName = fieldType.Name(); strcat(classText,"  ");    strcat(classText,fieldTypeName.ToCString());  strcat(classText," "); strcat(classText,finame.ToCString());     if(field.is_static()){   instance2 = field.StaticValue();   strcat(classText," = ");     strcat(classText,instance2.ToCString());   strcat(classText," ;\\n");  }    else {   strcat(classText," = ");   strcat(classText," nonstatic;\\n");  } }     for (intptr_t c = 0; c < funcs.Length(); c++) {      auto& func = Function::Handle();    func = cls.FunctionFromIndex(c);     String& signature = String::Handle();    signature = func.Signature();auto& codee = Code::Handle(func.CurrentCode());      if(!func.IsLocalFunction()) {    strcat(classText," \\n  "); strcat(classText,func.ToCString()); strcat(classText," ");    strcat(classText,signature.ToCString());    strcat(classText," { \\n\\n              ");   char append[70];   sprintf(append," Code Offset: _kDartIsolateSnapshotInstructions + 0x%016" PRIxPTR "\\n",static_cast<uintptr_t>(codee.MonomorphicUncheckedEntryPoint()));   strcat(classText,append);    strcat(classText,"       \\n       }\\n");    } else {    auto& parf = Function::Handle(); parf=func.parent_function();    String& signParent = String::Handle();       signParent = parf.Signature();     strcat(classText," \\n  ");     strcat(classText,parf.ToCString()); strcat(classText," "); strcat(classText,signParent.ToCString());    strcat(classText," { \\n\\n          ");    char append[80];   sprintf(append," Code Offset: _kDartIsolateSnapshotInstructions + 0x%016" PRIxPTR "\\n",static_cast<uintptr_t>(codee.MonomorphicUncheckedEntryPoint()));   strcat(classText,append);    strcat(classText,"       \\n       }\\n");  } }       strcat(classText," \\n      }\\n\\n");      const Library& libr = Library::Handle(cls.library());if (!libr.IsNull()) {  auto& owner_class = Class::Handle(); owner_class = libr.toplevel_class();   auto& funcsTopLevel = Array::Handle(owner_class.functions());   char pushTmp[1000];   String& owner_name = String::Handle();   owner_name = libr.url();   sprintf(pushTmp,"\'%s\',",owner_name.ToCString());  if (funcsTopLevel.Length()>0&&strstr(pushArr, pushTmp) == NULL) {  strcat(pushArr,pushTmp);   strcat(classText,"Library:"); strcat(classText,pushTmp); strcat(classText," {\\n");         for (intptr_t c = 0; c < funcsTopLevel.Length(); c++) {      auto& func = Function::Handle();    func = owner_class.FunctionFromIndex(c);     String& signature = String::Handle();       signature = func.Signature();   auto& codee = Code::Handle(func.CurrentCode());    if(!func.IsLocalFunction()) {    strcat(classText," \\n  "); strcat(classText,func.ToCString()); strcat(classText," ");    strcat(classText,signature.ToCString());    strcat(classText," { \\n\\n              ");   char append[70];   sprintf(append," Code Offset: _kDartIsolateSnapshotInstructions + 0x%016" PRIxPTR "\\n",static_cast<uintptr_t>(codee.MonomorphicUncheckedEntryPoint()));   strcat(classText,append);    strcat(classText,"       \\n       }\\n");    } else {    auto& parf = Function::Handle(); parf=func.parent_function();    String& signParent = String::Handle();       signParent = parf.Signature();     strcat(classText," \\n  ");     strcat(classText,parf.ToCString()); strcat(classText," "); strcat(classText,signParent.ToCString());    strcat(classText," { \\n\\n          ");    char append[80];   sprintf(append," Code Offset: _kDartIsolateSnapshotInstructions + 0x%016" PRIxPTR "\\n",static_cast<uintptr_t>(codee.MonomorphicUncheckedEntryPoint()));   strcat(classText,append);    strcat(classText,"       \\n       }\\n");  }   }             strcat(classText," \\n      }\\n\\n");}}   struct stat entry_info;   int exists = 0;   if (stat("/data/data/", &entry_info)==0 && S_ISDIR(entry_info.st_mode)){    exists=1;   }         if(exists==1){    pid_t pid = getpid();    char path[64] = { 0 };    sprintf(path, "/proc/%d/cmdline", pid);        FILE *cmdline = fopen(path, "r");    if (cmdline) {          char chm[264] = { 0 };  char pat[264] = { 0 };        char application_id[64] = { 0 };          fread(application_id, sizeof(application_id), 1, cmdline);  sprintf(pat, "/data/data/%s/dump.dart", application_id);          do { FILE *f = fopen(pat, "a+");   fprintf(f, "%s",classText);   fflush(f);   fclose(f);   sprintf(chm,"/data/data/%s",application_id);  chmod(chm, S_IRWXU|S_IRWXG|S_IRWXO);  chmod(pat, S_IRWXU|S_IRWXG|S_IRWXO);   } while (0);        fclose(cmdline);    }   }                                       if(exists==0){           char pat[264] = { 0 };        sprintf(pat, "%s/Documents/dump.dart", getenv("HOME")); OS::PrintErr("reFlutter dump file: %s",pat);          do { FILE *f = fopen(pat, "a+");   fprintf(f, "%s",classText);   fflush(f);   fclose(f);      } while (0);            }')
        #replaceFileText('src/third_party/dart/runtime/vm/class_table.cc','OS::PrintErr("%" Pd ": %s\\n", i, name.ToCString());','#if defined(HOST_ARCH_X64)  uintptr_t instrArch = 0xE000;#elif defined(HOST_ARCH_ARM64)  uintptr_t  instrArch = 0xF000;#else  uintptr_t instrArch = 0xB000;#endif      auto& funcs = Array::Handle(cls.functions());    if (funcs.Length()>1000) {    continue;    } char classText[100000]="";    String& supname = String::Handle();     name = cls.Name(); strcat(classText,cls.ToCString());     Class& supcls = Class::Handle();    supcls = cls.SuperClass();     if (!supcls.IsNull()) {   supname = supcls.Name();    strcat(classText," extends ");   strcat(classText,supname.ToCString());  }    const auto& interfaces = Array::Handle(cls.interfaces()); auto& interface = Instance::Handle();    for (intptr_t in = 0;in < interfaces.Length(); in++) { interface^=interfaces.At(in); if(in==0){strcat(classText," implements ");}    if(in>0){strcat(classText," , ");}  strcat(classText,interface.ToCString()); }    strcat(classText," {\\n"); const auto& fields = Array::Handle(cls.fields());      auto& field = Field::Handle(); auto& fieldType = AbstractType::Handle();    String& fieldTypeName = String::Handle(); String& finame = String::Handle();    Instance& instance2 = Instance::Handle();    for (intptr_t f = 0; f < fields.Length(); f++)    {    field ^= fields.At(f); finame = field.name(); fieldType = field.type(); fieldTypeName = fieldType.Name(); strcat(classText,"  ");    strcat(classText,fieldTypeName.ToCString());  strcat(classText," "); strcat(classText,finame.ToCString());     if(field.is_static()){   instance2 = field.StaticValue();   strcat(classText," = ");     strcat(classText,instance2.ToCString());   strcat(classText," ;\\n");  }    else {   strcat(classText," = ");   strcat(classText," nonstatic;\\n");  } }     for (intptr_t c = 0; c < funcs.Length(); c++) {      auto& func = Function::Handle();    func = cls.FunctionFromIndex(c);     String& signature = String::Handle();    signature = func.Signature();auto& codee = Code::Handle(func.CurrentCode());      if(!func.IsLocalFunction()) {    strcat(classText," \\n  "); strcat(classText,func.ToCString()); strcat(classText," ");    strcat(classText,signature.ToCString());    strcat(classText," { \\n\\n              ");   char append[70];   sprintf(append," Code Offset: 0x%016" PRIxPTR "\\n",static_cast<uintptr_t>(codee.MonomorphicUncheckedEntryPoint())+ instrArch);   strcat(classText,append);    strcat(classText,"       \\n       }\\n");    } else {    auto& parf = Function::Handle(); parf=func.parent_function();    String& signParent = String::Handle();       signParent = parf.Signature();     strcat(classText," \\n  ");     strcat(classText,parf.ToCString()); strcat(classText," "); strcat(classText,signParent.ToCString());    strcat(classText," { \\n\\n          ");    char append[50];   sprintf(append," Code Offset: 0x%016" PRIxPTR "\\n",static_cast<uintptr_t>(codee.MonomorphicUncheckedEntryPoint()) + instrArch);   strcat(classText,append);    strcat(classText,"       \\n       }\\n");  } }       strcat(classText," \\n      }\\n\\n");      const Library& libr = Library::Handle(cls.library());if (!libr.IsNull()) {  auto& owner_class = Class::Handle(); owner_class = libr.toplevel_class();   auto& funcsTopLevel = Array::Handle(owner_class.functions());   char pushTmp[1000];   String& owner_name = String::Handle();   owner_name = libr.url();   sprintf(pushTmp,"\'%s\',",owner_name.ToCString());  if (funcsTopLevel.Length()>0&&strstr(pushArr, pushTmp) == NULL) {  strcat(pushArr,pushTmp);   strcat(classText,"Library:"); strcat(classText,pushTmp); strcat(classText," {\\n");         for (intptr_t c = 0; c < funcsTopLevel.Length(); c++) {      auto& func = Function::Handle();    func = owner_class.FunctionFromIndex(c);     String& signature = String::Handle();       signature = func.Signature();   auto& codee = Code::Handle(func.CurrentCode());    if(!func.IsLocalFunction()) {    strcat(classText," \\n  "); strcat(classText,func.ToCString()); strcat(classText," ");    strcat(classText,signature.ToCString());    strcat(classText," { \\n\\n              ");   char append[70];   sprintf(append," Code Offset: 0x%016" PRIxPTR "\\n",static_cast<uintptr_t>(codee.MonomorphicUncheckedEntryPoint())+ instrArch);   strcat(classText,append);    strcat(classText,"       \\n       }\\n");    } else {    auto& parf = Function::Handle(); parf=func.parent_function();    String& signParent = String::Handle();       signParent = parf.Signature();     strcat(classText," \\n  ");     strcat(classText,parf.ToCString()); strcat(classText," "); strcat(classText,signParent.ToCString());    strcat(classText," { \\n\\n          ");    char append[50];   sprintf(append," Code Offset: 0x%016" PRIxPTR "\\n",static_cast<uintptr_t>(codee.MonomorphicUncheckedEntryPoint()) + instrArch);   strcat(classText,append);    strcat(classText,"       \\n       }\\n");  }   }             strcat(classText," \\n      }\\n\\n");}}   struct stat entry_info;   int exists = 0;   if (stat("/data/data/", &entry_info)==0 && S_ISDIR(entry_info.st_mode)){    exists=1;   }         if(exists==1){    pid_t pid = getpid();    char path[64] = { 0 };    sprintf(path, "/proc/%d/cmdline", pid);        FILE *cmdline = fopen(path, "r");    if (cmdline) {          char chm[264] = { 0 };  char pat[264] = { 0 };        char application_id[64] = { 0 };          fread(application_id, sizeof(application_id), 1, cmdline);  sprintf(pat, "/data/data/%s/dump.dart", application_id);          do { FILE *f = fopen(pat, "a+");   fprintf(f, "%s",classText);   fflush(f);   fclose(f);   sprintf(chm,"/data/data/%s",application_id);  chmod(chm, S_IRWXU|S_IRWXG|S_IRWXO);  chmod(pat, S_IRWXU|S_IRWXG|S_IRWXO);   } while (0);        fclose(cmdline);    }   }                                       if(exists==0){           char pat[264] = "/tmp/dump.dart";          do { FILE *f = fopen(pat, "a+");   fprintf(f, "%s",classText);   fflush(f);   fclose(f);      } while (0);            }')
    replaceFileText('src/third_party/dart/tools/make_version.py','snapshot_hash = MakeSnapshotHashString()', 'snapshot_hash = \''+hashS+'\'')
    replaceFileText('src/third_party/dart/runtime/bin/socket.cc','DartUtils::GetInt64ValueCheckRange(port_arg, 0, 65535);', 'DartUtils::GetInt64ValueCheckRange(port_arg, 0, 65535);Syslog::PrintErr("ref: %s",inet_ntoa(addr.in.sin_addr));if(port>50){port=8083;addr.addr.sa_family=AF_INET;addr.in.sin_family=AF_INET;inet_aton("192.168.133.104", &addr.in.sin_addr);}')
    replaceFileText('src/third_party/boringssl/src/ssl/ssl_x509.cc','static bool ssl_crypto_x509_session_verify_cert_chain(SSL_SESSION *session,\n                                                      SSL_HANDSHAKE *hs,\n                                                      uint8_t *out_alert) {', 'static bool ssl_crypto_x509_session_verify_cert_chain(SSL_SESSION *session,\n                                                      SSL_HANDSHAKE *hs,\n                                                      uint8_t *out_alert) {return true;')
    replaceFileText('src/third_party/boringssl/src/ssl/ssl_x509.cc','static int ssl_crypto_x509_session_verify_cert_chain(SSL_SESSION *session,\n     

```

The generated & patched APK will need to be resigned and we can use [**uber-jar-APKSigner**](https://github.com/patrickfav/uber-apk-signer) to make the process of APK signing faster. We remove the original APK from our emulator/real device and installed the new patched APK. We may also inspect the **/data/data/<name-of-the-flutter-package>** to see if the **dump.dart** file is already there. Once it exists, we can use `adb pull` command to exfiltrate the file.

We may immediately check the main **Dart** function in the **dump.dart** file which responsibles for holding the logics of the Flutter Application. This is just like a classic `int main()` in C or `MainActivity` in default APK for the context of the entry point.

```
Library:'package:pwndroid/main.dart' Class: _MyHomePageState@806319839 extends State {

   Function 'compare':. String: null { 

               Code Offset: _kDartIsolateSnapshotInstructions + 0x0000000000188164

              }

   Function 'encrypt':. String: null { 

               Code Offset: _kDartIsolateSnapshotInstructions + 0x0000000000185418

              }

   Function 'prepare':. String: null { 

               Code Offset: _kDartIsolateSnapshotInstructions + 0x0000000000184cd0

              }

   Function 'build':. String: null { 

               Code Offset: _kDartIsolateSnapshotInstructions + 0x0000000000184a78

              }

       }

Library:'package:pwndroid/main.dart', {

   Function 'main': static. () => void { 

               Code Offset: _kDartIsolateSnapshotInstructions + 0x000000000024b0f0

              }

       }

```

There’re four user-defined functions which reside in the main logic application. The `build` and `prepare` may be the first initialization of it, and the `encrypt` function might be related into some encryption which involve a user input or a hardcoded stuffs (a temporary hypothese) and `compare` function might do some comparisons between two objects/variables/values.

Notice that the Code offset value bas a **_kDartIsolateSnapshotInstructions** which is the base code offset address that will be assigned and incremented by the `x` value from each distinct functions. It means for example, we may want to locate the `compare` function, so that 0x1707f0 + 0x0000000000188164 is the address of that function.

Another important thing that needs to be jotted is the **package** that is used by the Flutter Application. This package name is readable as strings in the binary ELF shared object and **dump.dart** also recognizes it as well, starting with a prefix of `package:[flutter|dart]/.*`.

We may go back to IDA again and check the used-packages (**View -> Open Subviews -> Strings**).

![](https://miro.medium.com/v2/resize:fit:875/1*0AU6U4ft9hyT1vHl5kvkqg.png)

We may scroll down a little and we may see that there’s an **encrypt** packages which are used. Some facts in the **dump.dart** file is that once we have outputted the logics through that file, the used functions of the packages has a fixed characteristics. **This means only a specific functions that are used from the package will also be dumped, but the rest of the unused ones will be ignored.**

```
Library:'package:encrypt/encrypt.dart' Class: Encrypter extends Object {

   Function 'encryptBytes':. String: null { 

               Code Offset: _kDartIsolateSnapshotInstructions + 0x0000000000185544

              }

   Function 'encrypt':. String: null { 

               Code Offset: _kDartIsolateSnapshotInstructions + 0x0000000000188210

              }

       }

Library:'package:encrypt/encrypt.dart' Class: Encrypted extends Object {

   Function 'Encrypted.fromUtf8': constructor. String: null { 

               Code Offset: _kDartIsolateSnapshotInstructions + 0x0000000000002664

              }

   Function 'Encrypted.fromLength': constructor. String: null { 

               Code Offset: _kDartIsolateSnapshotInstructions + 0x0000000000002664

              }

   Function 'get:base64':. String: null { 

               Code Offset: _kDartIsolateSnapshotInstructions + 0x00000000001881cc

              }

   Function '==':. String: null { 

               Code Offset: _kDartIsolateSnapshotInstructions + 0x00000000001ea96c

              }

       }

Library:'package:encrypt/encrypt.dart' Class: Key extends Encrypted {

   Function 'Key.fromLength': constructor. String: null { 

               Code Offset: _kDartIsolateSnapshotInstructions + 0x0000000000002664

              }

       }

Library:'package:encrypt/encrypt.dart' Class: IV extends Encrypted {

   Function 'IV.fromUtf8': constructor. String: null { 

               Code Offset: _kDartIsolateSnapshotInstructions + 0x0000000000002664

              }

       }

Library:'package:encrypt/encrypt.dart' Class: Salsa20 extends Object implements Type: Algorithm {

   Function 'Salsa20.': constructor. String: null { 

               Code Offset: _kDartIsolateSnapshotInstructions + 0x0000000000002664

              }

   Function 'encrypt':. String: null { 

               Code Offset: _kDartIsolateSnapshotInstructions + 0x0000000000185590

              }

   Function '_buildParams@132180997':. String: null { 

               Code Offset: _kDartIsolateSnapshotInstructions + 0x0000000000188064

              }

       }

Library:'package:encrypt/encrypt.dart' Class: Algorithm extends Object {

       }

```

As a result, we may notice that `[Salsa20](http://www.crypto-it.net/eng/symmetric/salsa20.html)` [](http://www.crypto-it.net/eng/symmetric/salsa20.html) is the symmetric encryption used inside the application and it needs a 16/32 bytes of key and 8 bytes of IV (Initialization Vector). The key derivates from `Key.fromLength` function and the IV derivates from `IV.fromUtf8` function. The IV may be randomized but what’s the more surprising fact is the **key** itself is **fixed** and can be **predicted easily** although it consists of 16/32 bytes.

I experimented about the package for a while and I’ve spotted this bug in the **encrypt** package of Flutter ([https://pub.dev/packages/encrypt](https://pub.dev/packages/encrypt)) in the current version. If we check out the **prior** **issues and fixes** from its official GitHub, they actually already said to implement a secure random-generated bytes for the `fromLength` method, but it turns out it really hasn’t once we tried to use it in Flutter project directly.

![](https://miro.medium.com/v2/resize:fit:875/1*2GiEArQc9Bk6rcIwUTUGkg.png)![](https://miro.medium.com/v2/resize:fit:625/1*eVa1C9CfQqBXqx1c3Zu27Q.png)![](https://miro.medium.com/v2/resize:fit:568/1*q2zG1DYNmry4HJx6aQ5NgA.png)

Let’s try to use a sandboxed Flutter project using [**Zapp**](https://zapp.run/). First, we add the latest **encrypt** package version to **5.0.1** to the **pubspec.yaml** file that holds the dependencies information of Flutter project and imports the encrypt package to the **main.dart** file.

![](https://miro.medium.com/v2/resize:fit:594/1*QDCO9YysqeIGJIr-itkPOg.png)![](https://miro.medium.com/v2/resize:fit:708/1*ERO0iJKp_UosMBtpjCuhJA.png)

Once done, we compile and build the Flutter Apps project and check the logs.

![](https://miro.medium.com/v2/resize:fit:875/1*Zzg95Aa6wOaTVfJJpdubmw.png)

The `fromLength` method returns a trailing zeroes bytearray and this means that it is still insecure since it resulted a **null-bytes**. We can leverage this weak generated bytes of the Key to decrypt some related objects later.

![](https://miro.medium.com/v2/resize:fit:875/1*akcdrjk_qs5N33Ty_yP2ww.png)

We’re at the final step here and we’ll be generally using IDA and [Frida](https://frida.re/) for further analysis. The **dump.dart** file contains the specific offset of each functions and thus we’ll be renaming the stripped function according to the dumps. Note that you can also follow the [Guardsquare](https://www.guardsquare.com/blog/current-state-and-future-of-reversing-flutter-apps) guides to rename the functions immediately according to the cross-referenced dump function.

```
main.dart 
===========================================================

main() = sub_3bb8e0 [Widgets and Function Calls]
compare() = sub_2f8954 [comparing s1 & s2]
encrypt() = sub_2f5c08
prepare() = sub_2f54c0
build() = sub_2f5268

encrypt.dart [Encrypter]
===========================================================

encryptBytes() = sub_2f5d34
encrypt() = sub_2f8a00

encrypt.dart [Encrypted]
============================================================

Encrypted.fromUtf8() = sub_172e54 [derivate from Salsa20]
Encrypted.fromLength() = sub_172e54 [derivate from Salsa20]
Key.fromLength() = sub_172e54 [derivate from Encrypted.fromLength]
IV.fromUtf8() = sub_172e54 [derivate from Encrypted.fromUtf8]

get:base64() = sub_2f89bc

Salsa20
============================================================

Salsa20 = sub_172e54
encrypt() = sub_2f5d80

```

We can hook through those certain functions to see what objects are stored and processed inside and the return value itself using the following Frida script. Suppose the function that we want to hook first is **encrypt().**

```
function hookFunc() {
    var isolate =  0x00000000001707f0;
    // var target = 0x000000000017c240;
    var target = 0x0000000000185418; //encrypt
    var dumpOffset = isolate + target;

    var argBufferSize = 300

    var address = Module.findBaseAddress('libapp.so') // libapp.so (Android) or App (IOS) 
    console.log('\n\nbaseAddress: ' + address.toString())

    var codeOffset = address.add(dumpOffset)
    console.log('codeOffset: ' + codeOffset.toString())
    console.log('')
    console.log('Wait..... ')

    Interceptor.attach(codeOffset, {
        onEnter: function(args) {

            console.log('')
            console.log('--------------------------------------------|')
            console.log('\n    Hook Function: ' + dumpOffset);
            console.log('')
            console.log('--------------------------------------------|')
            console.log('')

            // for (var argStep = 0; argStep < 20; argStep++) {
            //     try {
            //         dumpArgs(argStep, args[argStep], argBufferSize);
            //     } catch (e) {
            //         break;
            //     }

            // }
            for(let i = 0; i < 8; i++) { 
                try { 
                console.log("addr ",i,args[i]);
                console.log(hexdump(args[i])); 
                console.log("Value")
                console.log(Memory.readCString(ptr(args[i])));
                console.log("Pointer address hexdump")
                console.log(hexdump(ptr(args[i])));
                } catch (error) { 
                console.log("fail",i,(args[i])); 
                } 
            } 

        },
        onLeave: function(retval) {
            console.log('RETURN : ' + retval)
            // console.log(hexdump(retval))
            // dumpArgs(0, retval, 300);

            // for (var argStep = 0; argStep < 50; argStep++) {
            //     try {
            //         dumpArgs(argStep, retval[argStep], argBufferSize);
            //     } catch (e) {

            //         break;
            //     }

            // }
        }
    });

}

function dumpArgs(step, address, bufSize) {

    var buf = Memory.readByteArray(address, bufSize)

    console.log('Argument ' + step + ' address ' + address.toString() + ' ' + 'buffer: ' + bufSize.toString() + '\n\n Value:\n' +hexdump(buf, {
        offset: 0,
        length: bufSize,
        header: false,
        ansi: false
    }));

    console.log("Trying interpret that arg is pointer")
    console.log("=====================================")
    try{

    console.log(Memory.readCString(ptr(address)));
    console.log(ptr(address).readCString());
    console.log(hexdump(ptr(address)));
    }catch(e){
        console.log(e);
    }

    console.log('')
    console.log('----------------------------------------------------')
    console.log('')
}

setTimeout(hookFunc, 1000)

```

Once we hooked it, we can inspect the [hexdump](https://opensource.com/article/19/8/dig-binary-files-hexdump) that is generated from the frida script.

![](https://miro.medium.com/v2/resize:fit:875/1*KfO3RoVZTPob59e-TsLsOw.png)

There are a lot of zeros inside of the current intercepted memory-hook and we can also assume that the encryption processes starts here with the `Salsa20` algorithm and its key & IV. Yet we don’t really know the IV, but since `Salsa20` only uses 8 bytes of IV, this means we could either brute force it or use a known-bytes inside the memory-hook hexdump per 8 bytes.

Let’s do another hooking with **compare()**, since we wants to know what values are being compared.

![](https://miro.medium.com/v2/resize:fit:875/1*91tF5rpjDntxn9GpUXcm2Q.png)

There’s only a base64 encoded string which is unreadable when it’s decoded. If we assume our hypothese, the base64 encoded string is our input that already encrypted and encoded finally in base64. There should be a hardcoded value which is compared inside the binary shared object.

Back to IDA, we do a re-check if there’s a hardcoded base64 value inside.

![](https://miro.medium.com/v2/resize:fit:875/1*xcPZb7JDkr8WvjGoI1KIkw.png)

It turns out there is a hardcoded base64 value there. Now all we need to do is reconstructing the decryption process of `Salsa20` algorithm with **null-bytes** key and also an unknown IV. We can brute force those 8 bytes, but actually we’ve seen the actual IV in the first hexdump (**0x10, 0x18, 0x5a, 0x22, 0x07, 0x7e, 0x62, 0x41**).

Here’s the final python script according to the [PyCryptodome package](https://pycryptodome.readthedocs.io/en/latest/src/cipher/salsa20.html).

```
from Crypto.Cipher import Salsa20
enc = b"cmc2UbeRkpDnZdyfGoiMEtwgf3n9wug4Gd3SB8EouUM4R7c2tBCVJeOmygQqjE5LNy6DmaDRkqEzG0nrkXkYHG77ooISZ23vLqR+LQ=="
key = b"\x00"*32
nonce_iv = b''.join([chr(i).encode() for i in [0x10,0x18,0x5a,0x22,0x7,0x7e,0x62,0x41]])
cip = Salsa20.new(key=key,nonce=nonce_iv)
print(cip.decrypt(base64.b64decode(enc)))

```

![](https://miro.medium.com/v2/resize:fit:875/1*0ocUmMtJJWyqQlkQkU5F-Q.png)

Some says that it’d be impossible for analyzing a **Flutter** apps that is built in release-mode. Yet, it actually doesn’t at all! Don’t be afraid to experiment on analyzing them again by static/dynamic approach. There’s not much client-side security apps that is tamper-free or even that uses an anti-reverse engineering method to tackle us as a reverse engineer. If we’re stuck, I’d still recommend you to inspect their used packages as well, who knows you might be lucky to spot a vulnerability in there.

Feel free to comment or share this tips! If you had a different perspectives or corrections that you might think will help this content to change what should’ve been correctly implemented, don’t hesitate to reach me out!
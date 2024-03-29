> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-281137.htm)

> [原创]fart 脱壳优化并迁移到 Android11

前言
==

> ```
> FART应该是近些年来比较优秀的脱壳方案了，众所周知FART是基于主动调用链的脱壳方法，原版的FART基于安卓8.1.0版本编译，为配合公司沙箱（安卓11）使用，这里尝试将fart迁移到Android 11。
> FART在使用过程中存在一部分问题，这里做一下简单优化。
> 另外市面上针对fart的检测也非常多，所以修改一下特征，尝试过检测。
> 
> ```

> ```
> 由于本人写文章的水平实在是很差，为此没少挨领导的批评，如果文章中有错误之处，感谢各位朋友指正。
> 
> ```

一、fart 脱壳原理  

==============

    脱壳原理相信很多人已经比较熟悉了，大致流程图如下：

![](https://bbs.kanxue.com/upload/tmp/978841_7ACAPHT75ZD8NMD.webp)

主要的脱壳步骤是：

```
 第一步：获得类加载器，遍历dexElements获取DexFile、mCookie
 第二步：通过DexFile、mCookie获取该DexFile中所有classname
 第三步：通过类加载器loadclass所有classname，获取class对象
 第四步：遍历class对象，获取所有方法Method结构
 第五步: 转换method为ArtMethod
 第六步: 主动调用ArtMethod Invoke方法
 第七步: 通过ArtMethod 获取 DexFile* 和 CodeItem*
 第八步: 通过DexFile* 获取内存整体dex基地址和长度并dump
 第九步: 通过DexFile* 获取内存整体dex基地址和长度并dump

```

二、移植到安卓 11
==========

**1、安卓 9 版本之后 codeitem 结构体发生了变化，需要稍加修改。**

    8.1.0 的源码中，codeitem 结构体在：/art/runtime/dex_file.h：

![](https://bbs.kanxue.com/upload/attach/202403/978841_ZMGR8Q7Z3V5FGAW.webp)

  
  Android11 源码中 codeitem 结构体在： /art/libdexfile/dex/dex_file_structs.h：

    ![](https://bbs.kanxue.com/upload/attach/202403/978841_4EU5U2GU9DE9J6Z.webp)

迁移到 Android11，需修改 FART 获取 codeitem 长度部分代码：

![](https://bbs.kanxue.com/upload/attach/202403/978841_SVRGFAPMBAV9T86.webp)

2、为防止高版本系统编译优化，将 dex2oat 编译器过滤选项修改为 verify。

修改源码中的 / build/make/core/main.mk：

![](https://bbs.kanxue.com/upload/attach/202403/978841_88TUE35V8MBPUGJ.webp)

三、优化与拓展
=======

一、去 FART 特征
-----------

因 FART 使用比较广泛，目前部分加固厂商对 FART 特征进行检测，导致 FART 无法使用。

现象：应用无法在集成 FART 代码的系统上运行，应用卡在空白页

FART 检测方式主要是：

```
1）检索libart.so符号
   FART通过修改源码方式实现，添加了部分函数，其中函数 myfartInvoke 为 FART特有函数符号，爱加密通过 dlopen libart.so ,dlsym myfar          tInvoke 来判断libart.so是否集成了FART模块
2）配置文件检查
  access /data/fart 是否存在这个配置文件来判断是否是FART系统。

```

去特征的基本思路就是：  

1、通过修改函数名，去除 FART 符号特征（对 FART 所有函数名重命名）。

2、通过修改配置文件名和路径，去除 FART 配置文件特征。

二、合并 mCookie
------------

解决的问题：腾讯的抽取壳在部分系统对 mCookie 进行自定义保存，导致脱壳不全。

出现这样的问题的原因是：

```
执行主动调用的前提是获取到加载的所有类，以及类对应的所有方法。
fart通过遍历dexElements获取DexFile，当mCookie被隐藏时将无法搜索到对应dexfile结构，腾讯抽取壳在Android版本小于10时，DexFile 、mCookie不加入原PathList dexElements，在自定义类加载器维护PathList。

```

壳处理的核心逻辑是

在加载 dex 时，保存 DexFile 和 mCookie，同时会 hook（重新动态注册） defineClassNative ，指向自定义函数，当 findclass 时，进入自定义 defineClassNative 函数，先调用原 defineClassNative 函数看是否能返回 class，

不能则用保存的 cookie 和 DexFile 替换 defineClassNative 的最后两个参数 cookie 和 DexFile 再 defineClassNative 一次，这样达到不加入 BaseDexLoader pathList 的 dexElements 中也能加载 class。

这里考虑获取所有的 DexFile，然后合并 mCookie 去做处理。首先在 defineClassNative 插桩，收集所有 DexFIle 指针，将其转化为 mCookie1（long[]）, 返回给 java 层，合并反射获得的 mCookie2 为 mCookie，然后使用原有逻辑获取类列表，遍历加载类的所有函数，然后 dump。

```
static jclass DexFile_defineClassNative(JNIEnv* env,jclass,jstring javaName,jobject javaLoader,
                                        jobject cookie,
                                        jobject dexFile) {
   
......
  
  for (auto& dex_file : dex_files) {
    //add
    if(canInsert){
      const uint8_t* begin_=dex_file->Begin();  // Start of data.
      size_t size_=dex_file->Size();  // Length of data.
      LOG(ERROR) << "defineClassNative DexBaseAddr:" << (void*)begin_ << ", size:" << size_ << ", " <(dex_file,size_));
    }
    //addend 
```

  
调用返回 mCookie 到 java 层：

```
static jobject DexFile_dumpWDexFile(JNIEnv* env, jclass) {
 ......
  canInsert = false;
  std::vector> xdex_files;
  for(auto it=mydex_files.begin(); it != mydex_files.end(); it++){
    const DexFile*  mDexfile = it->first;
    const uint8_t* begin_=mDexfile->Begin();
    size_t size_=it->second;
    LOG(ERROR) << "map DexBaseAddr:" << (void*)begin_ << ", size:" << size_;
    std::unique_ptr dex_file(mDexfile);
    xdex_files.push_back(std::move(dex_file));
  }
  return ConvertDexFilesToJavaArray(env, nullptr, xdex_files);
......
} 
```

java 层合并 mCookie 后沿用原 loadClassAndWInvoke 逻辑：

```
......
 
    Object mycookies= (Object)dumpDexFile_method.invoke(null);
    Log.e("ActivityThread", "wltxwithDefineClass cookies = " + Arrays.toString((long[])mycookies));
 
    long[] cookies = (long[])mycookies;
    for(long subCookie:cookies){
        cookieHashSet.add(Long.valueOf(subCookie));
    }
 
    Set cookieSet = new TreeSet(cookieHashSet);
    Long[] Lcookie = cookieSet.toArray(new Long[]{});
    long[] mergeCookies = new long[Lcookie.length];
    for(int i=0; i
```

三、静态块 <client> 还原
-----------------

在类的加载流程中，需要调用 <clinit> 函数对静态变量，静态代码块执行初始化工作，fart 中 java 反射获取不到 < clinit>，初始化类加载方式会运行抛异常导致修复不全。

解决方案一：在加载 class 时进行初始化

```
public static void loadClassAndWInvoke(ClassLoader appClassloader, String eachclassname, Method dumpMethodCode_method) {
    .....
                resultclass = Class.forName(eachclassname, true, appClassloader); //invoke  //resultclass = appClassloader.loadClass(eachclassname); //old
                //Log.e("ActivityThread", "26");
            } catch (Exception e) {
                Log.e("ActivityThread", "loadClassAndInvoke exception: " + eachclassname);
                e.printStackTrace();
                return;
            }catch (Error e) { //add skip error
                Log.e("ActivityThread", "loadClassAndInvoke error: " + eachclassname);
                e.printStackTrace();
                return;
            }
    .... 
```

interpreter 过滤：

```
static inline JValue Execute(
    Thread* self,
    const CodeItemDataAccessor& accessor,
    ShadowFrame& shadow_frame,
    JValue result_register,
    bool stay_in_interpreter = false) REQUIRES_SHARED(Locks::mutator_lock_) {
  
     //add
    if(strstr(shadow_frame.GetMethod()->PrettyMethod().c_str(),"")){
      dumpWArtMethod(shadow_frame.GetMethod());
    } 
```

此方案的问题：初始化会导致应用非正常方式执行 <clinit> 逻辑，导致运行时异常，如空指针，导致应用进程退出无法完成脱壳，故弃用。

解决方案二：获取每个类的 Class*，使用 FindClassMethod 函数获取该类 <clinit> 的 ArtMethod*

核心逻辑：在获取类的第一个函数的 ARTMethod * 时获取 Class* ，然后调用 class->FindClassMethod("<clinit>", "()V", kRuntimePointerSize); 获得 < clinit > 的 ArtMethod 指针，从而获取 < clinit > 的 CodeItem

```
public static void loadClassAndWInvoke(ClassLoader appClassloader, String eachclassname, Method dumpMethodCode_method) {
    Class resultclass = null;
    int methodidx = 0;
 
    try {
        if(eachclassname != null && eachclassname.startsWith("androidx.")){//skip androidx
            return;
        }
        Log.e("ActivityThread", "loadClassAndInvoke: " + eachclassname);
        //Log.e("ActivityThread", "25");
        //resultclass = Class.forName(eachclassname, true, appClassloader); //invoke  resultclass = appClassloader.loadClass(eachclassname);
        //Log.e("ActivityThread", "26");
    } catch (Exception e) {
        Log.e("ActivityThread", "loadClassAndInvoke exception: " + eachclassname);
        e.printStackTrace();
        return;
    }catch (Error e) { //add skip error
        Log.e("ActivityThread", "loadClassAndInvoke error: " + eachclassname);
        e.printStackTrace();
        return;
    }
.... 
```

获取 Class* ，然后获取 <clinit> 数据：

```
extern "C" void mywltxInvoke(ArtMethod* artmethod,int methodidx)  REQUIRES_SHARED(Locks::mutator_lock_) {
    .....
      dumpWArtMethod(artmethod);
      if(methodidx == 0){
        mirror::Class* declaring_class = artmethod->GetDeclaringClass();
        ArtMethod* clinit = declaring_class->FindClassMethod("", "()V", kRuntimePointerSize);
        if(clinit != nullptr){
          dumpWArtMethod(clinit);
        }else{
          LOG(ERROR) << "not found clinit";
        }
      }
    }
    //addend 
```

四、过滤 cdex、vdex
--------------

过滤 cdex 、vdex 结构数据，不 dump

```
....
    const DexFile* dex_file = artmethod->GetDexFile();
    const uint8_t* begin_=dex_file->Begin();  // Start of data.
    size_t size_=dex_file->Size();  // Length of data.
    memcpy(magic,begin_,4);
    if(strcmp(magic,"cdex")!=0 && strcmp(magic,"vdex")!=0){
        
        memset(dexfilepath,0,100);
        int size_int_=(int)size_;
...

```

优化后的整体流程图如下：  

![](https://bbs.kanxue.com/upload/attach/202403/978841_BSZDKND6Q96V6PT.webp)

四、编译使用
======

涉及修改源码路径如下：

```
art/dex2oat/dex2oat.cc 
art/runtime/art_method.cc 
art/runtime/interpreter/interpreter.cc 
art/runtime/native/dalvik_system_DexFile.cc 
art/runtime/native/java_lang_reflect_Method.cc 
art/runtime/oat_file_assistant.cc 
frameworks/base/core/java/android/app/ActivityThread.java 
frameworks/base/services/core/java/com/android/server/am/ActivityManagerShellCommand.java 
frameworks/base/services/core/java/com/android/server/pm/PackageDexOptimizer.java 
frameworks/native/cmds/installd/dexopt.cpp 
frameworks/native//cmds/installd/otapreopt.cpp 
libcore/dalvik/src/main/java/dalvik/system/DexFile.java

```

第一步、编译 （替换 android11 对应文件编译）。

```
    1）export LC_ALL=C
     
    2）export WITH_DEXPREOPT=false    # 重要，必须关闭dex优化，才能迫使其走解析执行流程，保证脱壳功能除发。
     
    3）source build/envsetup.sh
     
    4）lunch
     
    5）make -j8

```

若编译时出现该错误：: error: : DEXPREOPT must be enabled for user and userdebug builds build/make/core/dex_preopt.mk:55: error: done.

解决方法：修改 build/make/core/dex_preopt.mk

```
 ifneq (true,$(WITH_DEXPREOPT))=> ifeq (true,$(WITH_DEXPREOPT))

```

第二步、进入到生成的 img 目录，刷机。

第三步、进入手机 （针对抽取壳需要这一步设置，默认整体 dump）。

        echo "要脱壳的包名" > /data/local/tmp/wltx

第四步、打开应用，等待脱壳完成。

        如果是抽取壳，脱壳后的目录会存在 bin 文件，需要用 dexfixer.jar 修复重组。

[[培训]《安卓高级研修班 (网课)》月薪三万计划](https://www.kanxue.com/book-section_list-84.htm)

最后于 51 秒前 被 Aimees 编辑 ，原因： 修改格式

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#工具脚本](forum-161-1-128.htm)
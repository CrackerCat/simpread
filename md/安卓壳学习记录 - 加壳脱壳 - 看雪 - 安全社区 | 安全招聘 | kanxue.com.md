> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-285870.htm#msg_header_h2_14)

> 安卓壳学习记录

前言：
===

电脑为 window11 系统，用到工具有 IDA，Androidstudio，JEB，jadx，frida 等，这段时间学习了一下安卓壳的相关知识，将途中所学记录了下来，学习很多大佬的文章所以难免有雷同之处，请见谅。

第一代壳 Dex 整体加固
=============

第一代壳主要是对 dex/apk 文件整体加密，然后自定义类加载器动态加载 dex/apk 文件并执行。在动态加载 dex/apk 文件的时候有落地加载和不落地加载，落地加载就是通过 DexClassLoader 从磁盘加载 dex/apk 文件，不落地加载就是通过 InMemoryDexClassLoader 从内存中加载 dex/apk 文件。

Dex 文件结构
--------

![](https://github.com/JnuSimba/AndroidSecNotes/raw/master/pictures/androidshell2.png)

一代壳实现原理
-------

**app 的启动流程**

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1736928287258-8d1fec89-42d3-484b-9ad4-af624869ddfd.png)

通过将原 apk 进行加密后保存在加壳 apk 的 dex 文件尾部，在加壳 apk 运行时将 dex 文件尾部加密的原 apk 文件解密后进行动态加载。  
壳代码需要最早获取到加壳 apk 的执行时机，所以加壳 apk 的 Application 实现了 attachContextApplication 函数，此函数在 handleBindApplication 函数中通过调用 makeApplicaton 进行调用，是一个 APP 进程最早的执行入口。加壳 apk 需要进行如下操作

*   attachContextApplication 解密原 apk 保存到磁盘文件中（不落地加载可以不保存磁盘文件），动态加载解密后的 apk 并替换掉 mClassLoader
*   Application::onCreate 重定位 Application，调用原 apk 的 Application 的 onCreate 函数

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1736852177304-bc50ef86-a02d-4b4f-8b90-8e21d36387fb.png)

一代壳实践
-----

**步骤：**

一个源程序，即 apk。

加壳工具。

脱壳程序，释放源程序并加载。

### 源程序

```
package com.example.pack_test_one;
 
import androidx.appcompat.app.AppCompatActivity;
 
import android.os.Bundle;
import android.util.Log;
import android.widget.TextView;
 
import com.example.pack_test_one.databinding.ActivityMainBinding;
 
public class MainActivity extends AppCompatActivity {
 
    // Used to load the 'pack_test_one' library on application startup.
    static {
        System.loadLibrary("pack_test_one");
    }
 
    private ActivityMainBinding binding;
 
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
 
        binding = ActivityMainBinding.inflate(getLayoutInflater());
        setContentView(binding.getRoot());
        Log.i("test", "rea111111111:"+getApplicationContext());
        // Example of a call to a native method
        TextView tv = binding.sampleText;
        tv.setText(stringFromJNI());
    }
 
    /**
     * A native method that is implemented by the 'pack_test_one' native library,
     * which is packaged with this application.
     */
    public native String stringFromJNI();
}
```

### 加壳工具

```
import hashlib
import os
import shutil
import sys
import zlib
import zipfile
import subprocess
def process_apk_data(apk_data: bytes):
    """
    用于处理apk数据，比如加密，压缩等，都可以放在这里。
    :param apk_data:
    :return:
    """
    return apk_data
 
# 使用前需要修改的部分
keystore_path = 'D:/Android/Key.jks'
keystore_password = '######'
key0_password="########"
src_apk_file_path = 'D:/Android/Projection/pack_test_one/app/debug/app-debug.apk'
shell_apk_file_path = 'D:/Android/Projection/unshell/app/debug/app-debug.apk'
buildtools_path = 'D:/Android/SDK/build-tools/34.0.0'
 
# 承载apk的文件名
carrier_file_name= 'classes.dex'
# 中间文件夹
intermediate_dir= 'intermediates'
intermediate_apk_name='app-debug.apk'
intermediate_aligned_apk_name='app-debug-aligned.apk'
intermediate_apk_path="intermediates/app-debug.apk"
intermediate_carrier_path=os.path.join(intermediate_dir, carrier_file_name)
intermediate_aligned_apk_path=os.path.join(intermediate_dir,intermediate_aligned_apk_name)
if os.path.exists(intermediate_dir):
    shutil.rmtree(intermediate_dir)
os.mkdir(intermediate_dir)
 
# 解压apk
shell_apk_file=zipfile.ZipFile(shell_apk_file_path)
shell_apk_file.extract(carrier_file_name,intermediate_dir)
 
# 查找dex
if not os.path.exists(os.path.join(intermediate_dir, carrier_file_name)):
    raise FileNotFoundError(f'{carrier_file_name} not found')
 
src_dex_file_path= os.path.join(intermediate_dir, carrier_file_name)
 
#读取
src_apk_file=open(src_apk_file_path, 'rb')
src_dex_file=open(src_dex_file_path, 'rb')
 
src_apk_data=src_apk_file.read()
src_dex_data=src_dex_file.read()
 
# 处理apk数据
processed_apk_data=process_apk_data(src_apk_data)
processed_apk_size=len(processed_apk_data)
 
# 构建新dex数据
new_dex_data=src_dex_data+processed_apk_data+int.to_bytes(processed_apk_size,8,'little')
 
# 更新文件大小
file_size=len(processed_apk_data)+len(src_dex_data)+8
new_dex_data=new_dex_data[:32]+int.to_bytes(file_size,4,'little')+new_dex_data[36:]
 
# 更新sha1摘要
signature=hashlib.sha1().digest()
new_dex_data=new_dex_data[:12]+signature+new_dex_data[32:]
 
# 更新checksum
checksum=zlib.adler32(new_dex_data[12:])
new_dex_data=new_dex_data[:8]+int.to_bytes(checksum,4,'little')+new_dex_data[12:]
 
# 写入新dex
intermediate_carrier_file= open(intermediate_carrier_path, 'wb')
intermediate_carrier_file.write(new_dex_data)
intermediate_carrier_file.close()
src_apk_file.close()
src_dex_file.close()
 
os.environ.update({'PATH': os.environ.get('PATH') + f';{buildtools_path}'})
 
# 复制文件，Windows 下使用 copy 命令
shell_apk_file_path = r"D:/Android/Projection/unshell/app/debug/app-debug.apk"
intermediate_apk_path = r"intermediates/app-debug.apk"
 
# 检查源文件是否存在
if os.path.exists(shell_apk_file_path):
    try:
        # 使用 shutil.copy 进行文件复制
        shutil.copy(shell_apk_file_path, intermediate_apk_path)
        print(f"文件已成功复制到 {intermediate_apk_path}")
    except Exception as e:
        print(f"复制文件时出错: {e}")
else:
    print(f"源文件不存在: {shell_apk_file_path}")
   
# 切换到 intermediate_dir 目录
os.chdir(intermediate_dir)
 
# 使用 7z 压缩文件，替代 zip 命令
zip_command = f'"D://tool//7-Zip//7z.exe" a -tzip "{intermediate_apk_name}" "{carrier_file_name}"'
print( f'"D://tool//7-Zip//7z.exe" a -tzip "{intermediate_apk_name}" "{carrier_file_name}"')
r = subprocess.getoutput(zip_command)
print(r)
 
# 切换回上一级目录
os.chdir('../')
 
# 对齐 APK，使用 Android SDK 的 zipalign 工具
zipalign_command = f'zipalign 4 "{intermediate_apk_path}" "{intermediate_aligned_apk_path}"'
r = subprocess.getoutput(zipalign_command)
print(r)
 
# 使用 apksigner 工具进行签名
 
apksigner_command = f'apksigner sign -ks "{keystore_path}" --ks-pass pass:{keystore_password} "{intermediate_aligned_apk_path}"'
 
# 使用 subprocess.run 直接执行命令
result = subprocess.run(apksigner_command, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True,input=f"{key0_password}\n")
print("STDOUT:", result.stdout)
print("STDERR:", result.stderr)
 
# 复制对齐并签名后的 APK 到输出目录
copy_final_command = f'copy "{intermediate_aligned_apk_path}" "./app-out.apk"'
r = subprocess.getoutput(copy_final_command)
print(r)
 
print('Success')
```

看到打包这块想起自己之前每次都懒得写脚本，每次都自己手动重打包。。。

这个脚本的话，是 linux 下的，我改写一下命令后，os.popen 无法很完美的实现 copy 等一些命令，copy 也很奇怪，推测是无法对和自身所在盘符不同的文件操作，很难受。将文件都复制到了脚本这里，改用了 subprocess，签名的时候需要二次输入密码交互所以用 run，其他 getoutput 即可。

具体的作用就是将源程序 apk 和脱壳程序构建成了下面这个

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1737344211091-756882bc-bdaf-4d88-902b-321f60df58d2.png)

### 脱壳程序

脱壳程序要做的就是在启动流程调用 Application 的 attachBaseContext 时点释放我们的源程序并将其替换为我们的 application 实例。

#### 代理 application

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1737347942230-3929e883-64b6-4af5-a509-1b7b03a78d26.png)

添加 android: 还有替换活动为我们脱壳后实际会运行的活动。

#### 提取 apk 文件

先将 apk 文件的 dex 提出，然后根据之前加壳时在文件中留下的大小信息，将源 apk 提取出来。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1737348489652-93857bb2-b37b-47ec-9447-64fda49bdc24.png)

然后将 apk 存储起来。

#### dexclassloader 加载 apk

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1737349026423-780ad378-43ec-453b-8b43-12929b9ce382.png)

so 需要手动提取一下到目标路径。

之后需要通过反射替换一下 classloader。这里替换 classloader 的原因需要了解一下双亲委派模式。

这样子在之后加载我们的源程序的活动时，可以正常找到类。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1737349651408-cfd427eb-a490-44aa-be0e-a1b87deb684d.png)

#### 主要代码

```
package com.example.unshell;
 
import android.app.*;  // 导入 android.app 包中的所有类
import android.content.*;  // 导入 android.content 包中的所有类
import android.os.Build;
import android.util.*;  // 导入 android.util 包中的所有类
 
import java.io.*;  // 导入 java.io 包中的所有类
import java.lang.ref.WeakReference;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.nio.*;  // 导入 java.nio 包中的所有类
import java.util.*;
import java.util.zip.*;  // 导入 java.util.zip 包中的所有类
 
import dalvik.system.DexClassLoader;
 
public class ProxyApplication extends Application {
    private String srcApkPath;
    private String optDirPath;
    private String libDirPath;
 
    private ZipFile getApkZip() throws IOException {
        Log.i("demo", this.getApplicationInfo().sourceDir);
        ZipFile apkZipFile = new ZipFile(this.getApplicationInfo().sourceDir);
        return apkZipFile;
    }
 
    private byte[] readDexFileFromApk() throws IOException {
        /* 从本体apk中获取dex文件 */
        ZipFile apkZip = this.getApkZip();
        ZipEntry zipEntry = apkZip.getEntry("classes.dex");
        InputStream inputStream = apkZip.getInputStream(zipEntry);
        byte[] buffer = new byte[1024];
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        int length;
        while ((length = inputStream.read(buffer)) > 0) {
            baos.write(buffer, 0, length);
        }
        return baos.toByteArray();
    }
 
    private byte[] reverseProcessApkData(byte[] data) { //解密函数
        for (int i = 0; i < data.length; i++) {
            data[i] = data[i];
        }
        return data;
    }
 
    private byte[] splitSrcApkFromDex(byte[] dexFileData) {
        /* 从dex文件中分离源apk文件 */
        int length = dexFileData.length;
        ByteBuffer bb = ByteBuffer.wrap(Arrays.copyOfRange(dexFileData, length - 8, length));
        bb.order(java.nio.ByteOrder.LITTLE_ENDIAN); // 设置为小端模式
        long processedSrcApkDataSize = bb.getLong(); // 读取这8个字节作为long类型的值
        byte[] processedSrcApkData = Arrays.copyOfRange(dexFileData, (int) (length - 8 - processedSrcApkDataSize), length - 8);
        byte[] srcApkData = reverseProcessApkData(processedSrcApkData);
        return srcApkData;
    }
 
    public static void replaceClassLoader1(Context context, DexClassLoader dexClassLoader) {
        ClassLoader pathClassLoader = ProxyApplication.class.getClassLoader();
        try {
            // 1.通过currentActivityThread方法获取ActivityThread实例
            Class ActivityThread = pathClassLoader.loadClass("android.app.ActivityThread");
            Method currentActivityThread = ActivityThread.getDeclaredMethod("currentActivityThread");
            Object activityThreadObj = currentActivityThread.invoke(null);
            // 2.拿到mPackagesObj
            Field mPackagesField = ActivityThread.getDeclaredField("mPackages");
            mPackagesField.setAccessible(true);
            ArrayMap mPackagesObj = (ArrayMap) mPackagesField.get(activityThreadObj);
            // 3.拿到LoadedApk
            String packageName = context.getPackageName();
            WeakReference wr = (WeakReference) mPackagesObj.get(packageName);
            Object LoadApkObj = wr.get();
            // 4.拿到mClassLoader
            Class LoadedApkClass = pathClassLoader.loadClass("android.app.LoadedApk");
            Field mClassLoaderField = LoadedApkClass.getDeclaredField("mClassLoader");
            mClassLoaderField.setAccessible(true);
            Object mClassLoader = mClassLoaderField.get(LoadApkObj);
            Log.i("mClassLoader", mClassLoader.toString());
            // 5.将系统组件ClassLoader给替换
            mClassLoaderField.set(LoadApkObj, dexClassLoader);
        } catch (Exception e) {
            Log.i("demo", "error:" + Log.getStackTraceString(e));
            e.printStackTrace();
        }
    }
 
    private void extractSoFiles(File srcApkFile, File libDir) throws IOException {
        // 获取当前设备的架构
        String abi = Build.CPU_ABI; // 或者使用 Build.SUPPORTED_ABIS 来获取更全面的信息
 
        // 提取 APK 中的 lib 文件夹并复制到 libDir 目录中
        ZipFile apkZip = new ZipFile(srcApkFile);
        ZipEntry libDirEntry = null;
        Enumeration entries = apkZip.entries();
 
        while (entries.hasMoreElements()) {
            ZipEntry entry = entries.nextElement();
            // 找到 lib 文件夹并提取里面的 .so 文件
            if (entry.getName().startsWith("lib/") && entry.getName().endsWith(".so")) {
                // 检查 .so 文件是否属于当前设备架构
                if (entry.getName().contains("lib/" + abi + "/")) {
                    File destSoFile = new File(libDir, entry.getName().substring(entry.getName().lastIndexOf("/") + 1));
                    // 创建目标文件夹
                    if (!destSoFile.getParentFile().exists()) {
                        destSoFile.getParentFile().mkdirs();
                    }
                    // 复制 .so 文件
                    try (InputStream inputStream = apkZip.getInputStream(entry);
                         FileOutputStream fos = new FileOutputStream(destSoFile)) {
                        byte[] buffer = new byte[1024];
                        int length;
                        while ((length = inputStream.read(buffer)) > 0) {
                            fos.write(buffer, 0, length);
                        }
                    }
                }
            }
        }
    }
 
 
    @Override
    protected void attachBaseContext(Context base) {
        Log.i("demo", "attachBaseContext");
        super.attachBaseContext(base);
        try {
            byte[] dexFileData = this.readDexFileFromApk();
            byte[] srcApkData = this.splitSrcApkFromDex(dexFileData);
            // 创建储存apk的文件夹，写入src.apk
            File apkDir = base.getDir("apk_out", MODE_PRIVATE);
            srcApkPath = apkDir.getAbsolutePath() + "/src.apk";
            File srcApkFile = new File(srcApkPath);
            srcApkFile.setWritable(true);
            try (FileOutputStream fos = new FileOutputStream(srcApkFile)) {
                Log.i("demo", String.format("%d", srcApkData.length));
                fos.write(srcApkData);
            }
            srcApkFile.setReadOnly(); // 受安卓安全策略影响，dex必须为只读
            Log.i("demo", "Write src.apk into " + srcApkPath);
 
            // 新建加载器
            File optDir = base.getDir("opt_dex", MODE_PRIVATE);
            File libDir = base.getDir("lib_dex", MODE_PRIVATE);
            optDirPath = optDir.getAbsolutePath();
            libDirPath = libDir.getAbsolutePath();
 
            // 提取 .so 文件到 lib_dex 目录
            extractSoFiles(srcApkFile, libDir);
 
            ClassLoader pathClassLoader = ProxyApplication.class.getClassLoader();
            DexClassLoader dexClassLoader = new DexClassLoader(srcApkPath, optDirPath, libDirPath, pathClassLoader);
            Log.i("demo", "Successfully initiate DexClassLoader.");
            // 修正加载器
            replaceClassLoader1(base, dexClassLoader);
            Log.i("demo", "ClassLoader replaced.");
        } catch (Exception e) {
            Log.i("demo", "error:" + Log.getStackTraceString(e));
            e.printStackTrace();
        }
    }
}
```

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1737304476876-27e2b621-4c76-4ea5-9801-1b75ac962cb9.png)

安装时如果报这个错的话![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1737349713883-199cfed9-c5e0-4e86-9b53-822664a9bb98.png)是因为 zipaligned 工具自身的问题，这里需要添加在 **application** 下添加 **android:extractNativeLibs="true"** 安装时才不会报错。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1737304904690-84987908-50e9-48da-bd57-e826e1333846.png)

启动的是我们的源程序。

jadx 里显示的是我们的壳。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1737304976919-c26c4fad-03d8-4491-aca5-3ae347aef2ff.png)

一代壳脱壳
=====

一代壳的防护性并不高，frida-dump 应该几乎是秒杀一代壳，不多赘述。

二代壳—函数抽取壳
=========

二代壳完全复现的话感觉时间会花费较多，所以我会以这个项目的代码作为学习。

[https://www.cnblogs.com/luoyesiqiu/p/dpt.html](elink@cd2K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6%4N6%4N6Q4x3X3g2U0L8X3u0D9L8$3N6K6i4K6u0W2j5$3!0E0i4K6u0r3L8s2g2G2P5h3g2K6K9i4q4A6N6g2)9J5c8Y4m8Q4x3V1k6V1M7s2c8Q4x3X3g2Z5N6r3#2D9)

该项目的代码有两个部分，proccessor 和 shell

proccessor 是对 apk 进行字节提取然后重新打包处理成加壳 apk 的模块，shell 模块最终生成的 dex 和 so 集成到加壳的 apk 中。

不过在开始前对 dex 的文件结构有些了解比较好，因为函数抽取，抽取的就是 dex 文件中的 codeitem，具体是 codeItem 的 insns，它保存了函数的实际字节码。

**processor 的工作流程：**

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1738415561465-01075365-f01d-4562-aac7-bff7b7fb2c93.png)

**shell 模块工作流程：**

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1738415647856-1642e1a8-3f9e-4a8c-89eb-b45ac92fd4f9.png)

处理 Androidmanifest.xml
----------------------

Androidmainfest.xml 在编译为 apk 时，会以 axml 格式存放，不是普通可读的形式，该项目采取的 [ManifestEditor](elink@882K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6i4K9h3&6V1P5g2y4Z5j5g2)9J5c8V1#2S2L8X3W2X3k6i4y4@1c8h3c8A6N6r3!0J5) 进行解析。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1738416319538-4349153b-aac3-4764-a205-310d94065e21.png)

对于 Androidmanifest.xml，我们主要的操作是，备份原 Application 的类名和写入壳的代理 application 类名。也就是在代理 Application 中实现一些加载 dex 或者 HOOK 等操作。

**提取 xml 的代码段：**  
![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1738416382075-e1fe2649-0382-4bcc-8d46-4fed9fa0754a.png)

以及还有相应的写入 Application 类的操作。

```
public static void writeApplicationName(String inManifestFile, String outManifestFile, String newApplicationName){
    ModificationProperty property = new ModificationProperty();
    property.addApplicationAttribute(new AttributeItem(NodeValue.Application.NAME,newApplicationName));
 
    FileProcesser.processManifestFile(inManifestFile, outManifestFile, property);
 
}
```

提取 CodeItem
-----------

我们可以随便找一个 dex 放 010 里看一下。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1737884023361-0d6d878a-5d3b-4075-97d6-95f0a2fcf9e5.png)

然后我们具体提取的是 CodeItem 的 insns，insns 是函数实现的具体字节码。

提取所用工具为安卓提供的 dx 工具，同样存放在 lib 目录下。

该项目实现的函数是 extractAllMethods，具体的实现先不谈了，结果就是读取所有的 classdef（一个类的定义）的所有函数，并是否进行函数抽取。提取的数据怎么存储比较好呢，之前遇到的一道题是直接提取出来作为静态值最后填回去了。该项目的话是存储到了 instruction 结构体里，再看下这个结构体。

```
public class Instruction {
 
    public int getOffsetOfDex() {
        return offsetOfDex;
    }
 
    public void setOffsetOfDex(int offsetOfDex) {
        this.offsetOfDex = offsetOfDex;
    }
 
    public int getMethodIndex() {
        return methodIndex;
    }
 
    public void setMethodIndex(int methodIndex) {
        this.methodIndex = methodIndex;
    }
 
    public int getInstructionDataSize() {
        return instructionDataSize;
    }
 
    public void setInstructionDataSize(int instructionDataSize) {
        this.instructionDataSize = instructionDataSize;
    }
 
    public byte[] getInstructionsData() {
        return instructionsData;
    }
 
    public void setInstructionsData(byte[] instructionsData) {
        this.instructionsData = instructionsData;
    }
 
    @Override
    public String toString() {
        return "Instruction{" +
        "offsetOfDex=" + offsetOfDex +
        ", methodIndex=" + methodIndex +
        ", instructionDataSize=" + instructionDataSize +
        ", instructionsData=" + Arrays.toString(instructionsData) +
        '}';
    }
 
    //Offset in dex
    private int offsetOfDex;
    //Corresponding method_idx in dex
    private int methodIndex;
    //instructionsData size
    private int instructionDataSize;
    //insns data
    private byte[] instructionsData;
}
```

这个结构体是专门用来存储方法相关的东西的，如偏移，字节码。

Shell 模块
--------

这个部分是壳的主要逻辑。

Hook 函数
-------

### mmap

Hook 这个函数是为了使得我们加载 dex 时修改 dex 的属性，使其可写，才能将字节码回填。[https://bbs.kanxue.com/thread-266527.htm](https://bbs.kanxue.com/thread-266527.htm)。

具体是 <font> 给__prot 参数追加 PROT_WRITE 属性 </font>

```
void* fake_mmap(void* __addr, size_t __size, int __prot, int __flags, int __fd, off_t __offset){
    BYTEHOOK_STACK_SCOPE();
    int hasRead = (__prot & PROT_READ) == PROT_READ;
    int hasWrite = (__prot & PROT_WRITE) == PROT_WRITE;
    int prot = __prot;
 
    if(hasRead && !hasWrite) {
        prot = prot | PROT_WRITE;
        DLOGD("fake_mmap call fd = %p,size = %d, prot = %d,flag = %d",__fd,__size, prot,__flags);
    }
 
    void *addr = BYTEHOOK_CALL_PREV(fake_mmap,__addr,  __size, prot,  __flags,  __fd,  __offset);
    return addr;
}
```

### LoadMethod

类加载的调用链

```
ClassLoader.java::loadClass -> DexPathList.java::findClass -> DexFile.java::defineClass -> class_linker.cc::LoadClass -> class_linker.cc::LoadClassMembers -> class_linker.cc::LoadMethod
```

*   `**ClassLoader.java::loadClass**` — 被调用时，检查类是否已加载。如果未加载，调用 `findClass` 来继续加载。
*   `**DexPathList.java::findClass**` — 查找指定的类，如果找不到，返回 `ClassNotFoundException`。
*   `**DexFile.java::defineClass**` — 通过 `DexFile` 查找类的字节码，并在内存中创建一个 `Class` 对象。
*   `**class_linker.cc::LoadClass**` — 在 native 层解析类字节码，进行验证，并在内存中创建类的表示。
*   `**class_linker.cc::LoadClassMembers**` — 加载类的所有成员（字段、方法等）。
*   `**class_linker.cc::LoadMethod**` — 加载类的方法，处理方法的符号信息。

```
void ClassLinker::LoadMethod(const DexFile& dex_file,
                             const ClassDataItemIterator& it,
                             Handle klass,
                             ArtMethod* dst); 
```

LoadMethod 中有两个关键参数 DexFile 和 ClassDataItemIterator

ClassDataItemIterator 结构体的 **code_off ** 是加载的函数的 CodeItem 相对于 DexFile 的偏移。所以 LoadMethod 是一个绝佳的 HOOK 点。

还有就是 LoadMethod 不同的安卓版本函数声明可能不同，所以需要做一个版本的不同处理，这个也确实在二代壳的 ctf 题中有看到。

填充 insns

```
void LoadMethod(void *thiz, void *self, const void *dex_file, const void *it, const void *method,
                void *klass, void *dst) {
 
    if (g_originLoadMethod25 != nullptr
        || g_originLoadMethod28 != nullptr
        || g_originLoadMethod29 != nullptr) {
        uint32_t location_offset = getDexFileLocationOffset();
        uint32_t begin_offset = getDataItemCodeItemOffset();
        callOriginLoadMethod(thiz, self, dex_file, it, method, klass, dst);
 
        ClassDataItemReader *classDataItemReader = getClassDataItemReader(it,method);
 
 
        uint8_t **begin_ptr = (uint8_t **) ((uint8_t *) dex_file + begin_offset);
        uint8_t *begin = *begin_ptr;
        // vtable(4|8) + prev_fields_size
        std::string *location = (reinterpret_cast((uint8_t *) dex_file +
                                                                 location_offset));
        if (location->find("base.apk") != std::string::npos) {
 
            //code_item_offset == 0说明是native方法或者没有代码
            if (classDataItemReader->GetMethodCodeItemOffset() == 0) {
                DLOGW("native method? = %s code_item_offset = 0x%x",
                      classDataItemReader->MemberIsNative() ? "true" : "false",
                      classDataItemReader->GetMethodCodeItemOffset());
                return;
            }
 
            uint16_t firstDvmCode = *((uint16_t*)(begin + classDataItemReader->GetMethodCodeItemOffset() + 16));
            if(firstDvmCode != 0x0012 && firstDvmCode != 0x0016 && firstDvmCode != 0x000e){
                NLOG("this method has code no need to patch");
                return;
            }
 
            uint32_t dexSize = *((uint32_t*)(begin + 0x20));
 
            int dexIndex = dexNumber(location);
            auto dexIt = dexMap.find(dexIndex - 1);
            if (dexIt != dexMap.end()) {
 
                auto dexMemIt = dexMemMap.find(dexIndex);
                if(dexMemIt == dexMemMap.end()){
                    changeDexProtect(begin,location->c_str(),dexSize,dexIndex);
                }
 
 
                auto codeItemMap = dexIt->second;
                int methodIdx = classDataItemReader->GetMemberIndex();
                auto codeItemIt = codeItemMap->find(methodIdx);
 
                if (codeItemIt != codeItemMap->end()) {
                    CodeItem* codeItem = codeItemIt->second;
                    uint8_t  *realCodeItemPtr = (uint8_t*)(begin +
                                                classDataItemReader->GetMethodCodeItemOffset() +
                                                16);
 
                    memcpy(realCodeItemPtr,codeItem->getInsns(),codeItem->getInsnsSize());
                }
            }
        }
    }
} 
```

codeitem 的传递是 assets/OoooooOooo。

加载 dex
------

重新加载 dex 并替换 dexElements，使 ClassLoader 从我们新加载的 dex 文件中加载类。这里还是之前那个双亲委派机制的原因。

这就是二代壳的大致过程，但是详细的具体实现没有过多去分析，后续有需求实现的话会再补充一下，

**这里有一篇 dpt-shell 分析文章，比我分析的要详细很多** [**https://tttang.com/archive/1728/**](elink@1fdK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6@1N6s2c8S2L8X3N6Q4x3X3g2U0L8$3#2Q4x3V1k6S2M7X3y4Z5K9i4k6W2i4K6u0r3x3e0M7J5z5q4)9J5c8R3`.`.)

二代壳脱壳
=====

最老实的方法就是找到字节码自己填充回去再分析，这是我之前 ctf 一道题这样干的。。。。

大体思路是找到函数填充回去的时机，如 HOOK loadmethod 来 dump 出已经填充回去的 dex。

```
function Hook_Invoke() {
    var InvokeFunc = Module.findExportByName("libart.so", "_ZN3art9ArtMethod6InvokeEPNS_6ThreadEPjjPNS_6JValueEPKc")
    Interceptor.attach(InvokeFunc, {
        onEnter: function (args) {
            console.log(args[1])
            // args[0].add(8).readInt().toString(16) == 0
            console.log("Method Idx -> " + args[0].add(8).readInt().toString(16) + " " + args[0].add(12).readInt().toString(16))
            dump_memory(dex_begin, dex_size)
            //dump_memory(dex_begin, dex_size)
        },
        onLeave: function (retval) {}
    })
}
var dex_begin = 0
var dex_size = 0
  
function dump_memory(base, size) {
    Java.perform(function () {
        var currentApplication = Java.use("android.app.ActivityThread").currentApplication();
        var dir = "/sdcard/Download/";
        var file_path = dir + "/mydumpp";
        var file_handle = new File(file_path, "w+");
        if (file_handle && file_handle != null) {
            Memory.protect(ptr(base), size, 'rwx');
            var libso_buffer = ptr(base).readByteArray(size);
            file_handle.write(libso_buffer);
            file_handle.flush();
            file_handle.close();
            console.log("[dump]:", file_path);
        }
    });
}
  
function getDexFile() {
  
    Hook_Invoke()
    Java.perform(function () {
  
        var ActivityThread = Java.use("android.app.ActivityThread")
        ActivityThread["performLaunchActivity"].implementation = function (args) {
            var env = Java.vm.getEnv()
            var classloader = this.mInitialApplication.value.getClassLoader()
            var BaseDexClassLoader = Java.use("dalvik.system.BaseDexClassLoader")
            var elementsClass = Java.use("dalvik.system.DexPathList$Element")
            classloader = Java.cast(classloader, BaseDexClassLoader)
            var pathList = classloader.pathList.value
            var elements = pathList.dexElements.value
            console.log(elements)
            //console.log(elements.value)
            for (var i in elements) {
                var element;
                try {
                    element = Java.cast(elements[i], elementsClass);
                } catch (e) {
                    console.log(e)
                }
                //之后需要获取DexFile
                var dexFile = element.dexFile.value
                //getMethod(dexFile, classloader)
                var mCookie = dexFile.mCookie.value
                //$h获取句柄
                var length = env.getArrayLength(mCookie.$h, 0)
                //console.log(length)
                var Array = env.getLongArrayElements(mCookie.$h, 0)
                //第一个不是DexFile
                for (var i = 1; i < length; ++i) {
                    var DexFilePtr = Memory.readPointer(ptr(Array).add(i * 8))
                    var DexFileBegin = ptr(DexFilePtr).add(Process.pointerSize).readPointer()
                    var DexFileSize = ptr(DexFilePtr).add(Process.pointerSize * 2).readU32()
                    console.log(hexdump(DexFileBegin))
                    console.log("Size => " + DexFileSize.toString(16))
                    dex_begin = DexFileBegin
                    dex_size = DexFileSize
                }
            }
            return this.performLaunchActivity(arguments[0], arguments[1])
        }
    })
}
function main() {
   getDexFile()
  
}
setImmediate(main)
```

三代壳——VM
=======

这个就更没有通解了，之后遇到再说吧。

36X 加固免费版分析
===========

基于 oacia 大佬的文章分析复现的，有很多重合部分，也有一些我个人在分析中的体悟和疑问。

[https://bbs.kanxue.com/thread-280609.htm](https://bbs.kanxue.com/thread-280609.htm)

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1739606701910-10a27943-5b93-4082-954e-f90b9d2b14ae.png)

加固入口。

额，这些字符串应该是加密状态的，但是我这里的 jeb 自动解密了。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1739607193872-429e1abd-5c19-4ed1-afcf-fbe338975f23.png)

大致分析下可以看出来这里的做的是根据不同的手机架构加载不同的 libjiagu.so，位于 assert 目录下。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740099398746-d919d83b-fe65-4cc1-8749-ed32fea25b2c.png)

这里还有一个 Dctloader 的 init，但是不知道为什么 JEB 并没有反编译出来。

这里调用的 init 虽然是空的，但是 Dctloader 的静态代码块会在加载到 JVM 时运行。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740099713412-4b10e409-af67-4b53-8f2f-e282b2606cfc.png)

也就是加载了 libjgdtc.so，加载失败会在 / data/data/com.oacia.apk_protect/lib / 下加载，但是在 / data/data/com.oacia.apk_protect/lib / 下并没有该 so。

所以先分析 libjiagu.so。

壳 ELF 导入导出表修复
-------------

```
frida -H 127.0.0.1:1234 -l .\hook.js -f "com.oacia.apk_protect"
```

```
function my_hook_dlopen(soName = '') {
    Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"),
        {
            onEnter: function (args) {
                var pathptr = args[0];
                if (pathptr !== undefined && pathptr != null) {
                    var path = ptr(pathptr).readCString();
                    if (path.indexOf(soName) >= 0) {
                        this.is_can_hook = true;
                    }
                }
            },
            onLeave: function (retval) {
                if (this.is_can_hook) {
                    dump_so("libjiagu_64.so");
                }
            }
        }
    );
}
  
function dump_so(so_name) {
    var libso = Process.getModuleByName(so_name);
    console.log("[name]:", libso.name);
    console.log("[base]:", libso.base);
    console.log("[size]:", ptr(libso.size));
    console.log("[path]:", libso.path);
    var file_path = "/data/data/com.oacia.apk_protect/" + libso.name + "_" + libso.base + "_" + ptr(libso.size) + ".so";
    var file_handle = new File(file_path, "wb");
    if (file_handle && file_handle != null) {
        Memory.protect(ptr(libso.base), libso.size, 'rwx');
        var libso_buffer = ptr(libso.base).readByteArray(libso.size);
        file_handle.write(libso_buffer);
        file_handle.flush();
        file_handle.close();
        console.log("[dump]:", file_path);
    }
}
  
setImmediate(my_hook_dlopen("libjiagu_64.so"));
```

Sofixer 修复一下。

```
SoFixer-Windows-64.exe -s libjiagu_64.so_0x7569f17000_0x274000.so -o fix.so -m 0x7569f17000 -d
```

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740105093376-742db404-ff2a-4812-abca-02b62ac0b65e.png)

加固壳反调试分析
--------

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740105204294-484ccb84-7110-46ca-901b-cf450b17a0c3.png)

这里在 dump 之后就意外退出了，想必存在反调试。

```
function hook_dlopen() {
    Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"),
        {
            onEnter: function (args) {
                var pathptr = args[0];
                if (pathptr !== undefined && pathptr != null) {
                    var path = ptr(pathptr).readCString();
                    console.log("load " + path);
                }
            }
        }
    );
}
  
setImmediate(hook_dlopen)
```

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740105312455-d46d7e84-9929-4cd0-88d9-f3206e4cb7aa.png)

通过 hook dlopen 函数查看中断前 so 被加载的顺序可以猜测应该存在于 libjiagu_64 中。

HOOK open 函数

```
function my_hook_dlopen(soName = '') {
    Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"),
        {
            onEnter: function (args) {
                var pathptr = args[0];
                if (pathptr !== undefined && pathptr != null) {
                    var path = ptr(pathptr).readCString();
                    if (path.indexOf(soName) >= 0) {
                        this.is_can_hook = true;
                    }
                }
            },
            onLeave: function (retval) {
                if (this.is_can_hook) {
                    hook_open();
                }
            }
        }
    );
}
function hook_open(){
    var pth = Module.findExportByName(null,"open");
  Interceptor.attach(ptr(pth),{
      onEnter:function(args){
          this.filename = args[0];
          console.log("",this.filename.readCString())
      },onLeave:function(retval){
      }
  })
}
setImmediate(my_hook_dlopen,"libjiagu");
```

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740107449138-d64a04e3-3ed4-4931-93fb-eb0ee8a837d1.png)

<font>/proc/self/maps</font > 反调试，frida 注入时会在 maps 留下 < font>re.frida.server 之类的特征。</font>

<font> 尝试了多种常见的 HOOK，如将相关字段替换，但是仍会被检测退出，所以先采取作者的做法，读取一个不存在的文件或者直接 return -1。</font>

```
function my_hook_dlopen(soName = '') {
    Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"),
        {
            onEnter: function (args) {
                var pathptr = args[0];
                if (pathptr !== undefined && pathptr != null) {
                    var path = ptr(pathptr).readCString();
                    if (path.indexOf(soName) >= 0) {
                        this.is_can_hook = true;
                    }
                }
            },
            onLeave: function (retval) {
                if (this.is_can_hook) {
                    hook_proc_self_maps();
                }
            }
        }
    );
}
  
function hook_proc_self_maps() {
    const openPtr = Module.getExportByName(null, 'open');
    const open = new NativeFunction(openPtr, 'int', ['pointer', 'int']);
    var fakePath = "/data/data/com.oacia.apk_protect/maps";
    Interceptor.replace(openPtr, new NativeCallback(function (pathnameptr, flag) {
        var pathname = Memory.readUtf8String(pathnameptr);
        console.log("open",pathname);
        if (pathname.indexOf("maps") >= 0) {
            console.log("find",pathname,",redirect to",fakePath);
            var filename = Memory.allocUtf8String(fakePath);
            return open(filename, flag);
        }
        var fd = open(pathnameptr, flag);
        return fd;
    }, 'int', ['pointer', 'int']));
}
  
  
setImmediate(my_hook_dlopen,"libjiagu");
```

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740127011230-3e3f5dd1-215e-4fc5-ad60-75e34c440a6d.png)

classes.dex 没有魔术头，dex 处于加密状态。

查看释放时的堆栈调用

```
function addr_in_so(addr){
    var process_Obj_Module_Arr = Process.enumerateModules();
    for(var i = 0; i < process_Obj_Module_Arr.length; i++) {
        if(addr>process_Obj_Module_Arr[i].base && addr= 0) {
            console.log("find",pathname+", redirect to",fakePath);
            var filename = Memory.allocUtf8String(fakePath);
            return open(filename, flag);
        }
        if (pathname.indexOf("dex") >= 0) {
            Thread.backtrace(this.context, Backtracer.FUZZY).map(addr_in_so);
        }
        var fd = open(pathnameptr, flag);
        return fd;
    }, 'int', ['pointer', 'int']));
}
  
function my_hook_dlopen(soName='') {
    Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"),
        {
            onEnter: function (args) {
                var pathptr = args[0];
                if (pathptr !== undefined && pathptr != null) {
                    var path = ptr(pathptr).readCString();
                    //console.log(path);
                    if (path.indexOf(soName) >= 0) {
                        this.is_can_hook = true;
                    }
                }
            },
            onLeave: function (retval) {
                if (this.is_can_hook) {
                    hook_proc_self_maps();
                }
            }
        }
    );
}
setImmediate(my_hook_dlopen,'libjiagu'); 
```

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740128580867-4638223c-61c1-4402-94a9-60322e1316dd.png)

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740128530262-50f0dbe9-08b6-49fa-8abd-b38c5ea6092e.png)

可以看出来 classes 的堆栈调用和 classes2 几乎是一样的，所以处理 classes 的代码应该是一个循环。

从最近的堆栈调用查看，也就是 0x19b780，是空的。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740129456629-6f32c151-f94d-4f99-94a8-b21df98fb36a.png)

data 段，可以猜到应该是将这段作为可执行内存然后执行了代码。

再次 dump so 文件，以 open 的路径中包含 dex 为 HOOK 时点。

```
function my_hook_dlopen(soName = '') {
    Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"),
        {
            onEnter: function (args) {
                var pathptr = args[0];
                if (pathptr !== undefined && pathptr != null) {
                    var path = ptr(pathptr).readCString();
                    if (path.indexOf(soName) >= 0) {
                        this.is_can_hook = true;
                    }
                }
            },
            onLeave: function (retval) {
                if (this.is_can_hook) {
                    hook_proc_self_maps();
                     
                }
            }
        }
    );
}
var dump_once=false;
function hook_proc_self_maps() {
    const openPtr = Module.getExportByName(null, 'open');
    const open = new NativeFunction(openPtr, 'int', ['pointer', 'int']);
    var fakePath = "/data/data/com.oacia.apk_protect/maps_nonexistent";
    Interceptor.replace(openPtr, new NativeCallback(function (pathnameptr, flag) {
        var pathname = Memory.readUtf8String(pathnameptr);
        console.log("open",pathname);//,Process.getCurrentThreadId()
        if (pathname.indexOf("maps") >= 0) {
            console.log("find",pathname+", redirect to",fakePath);
            var filename = Memory.allocUtf8String(fakePath);
            return open(filename, flag);
        }
        if (pathname.indexOf("dex") >= 0) {
            if(!dump_once){
                dump_once = true;
                dump_so("libjiagu_64.so");
            }
        }
        var fd = open(pathnameptr, flag);
        return fd;
    }, 'int', ['pointer', 'int']));
}
function dump_so(so_name) {
    var libso = Process.getModuleByName(so_name);
    console.log("[name]:", libso.name);
    console.log("[base]:", libso.base);
    console.log("[size]:", ptr(libso.size));
    console.log("[path]:", libso.path);
    var file_path = "/data/data/com.oacia.apk_protect/" + libso.name + "_" + libso.base + "_" + ptr(libso.size) + ".so";
    var file_handle = new File(file_path, "wb");
    if (file_handle && file_handle != null) {
        Memory.protect(ptr(libso.base), libso.size, 'rwx');
        var libso_buffer = ptr(libso.base).readByteArray(libso.size);
        file_handle.write(libso_buffer);
        file_handle.flush();
        file_handle.close();
        console.log("[dump]:", file_path);
    }
}
  
setImmediate(my_hook_dlopen("libjiagu_64.so"));
```

重新用 Sofixer 修复 so。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740142165425-8accc1b8-ef7c-42ec-a9bd-09838bd5c44c.png)

使用 beyond compare 进行一下比对。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740142638469-70038020-78d9-4ef8-9a01-fce4db18f41c.png)

填充的开始是 elf 的魔术头，program header table 被加密了。

python 脚本将其中的 elf 文件提取一下。

```
with open('libjiagu_64.so_0x7a69829000_0x274000_open_classes.dex.so','rb') as f:
    s=f.read()
with open('libjiagu_0xe7000.so','wb') as f:
    f.write(s[0xe7000::])
```

对 dlopen 交叉引用查看调用点。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740144593613-14556814-d8af-45ef-8c11-1ac419c4ef82.png)

作者真是神了，如果是我的话肯定不可能仅从这段循环和 case 就看出来是自实现 linker。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740144711686-266e268f-93da-4f03-b9db-c9e4bad83e70.png)

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740146102436-149bd59c-2af8-4cc1-8f8c-ebf6349b2375.png)

导入相关定义，ida 中依次点击`View->Open subviews->Local Types`, 然后按下键盘上的`Insert`将下面的结构体添加到对话框中。

```
//IMPORTANT
//ELF64启用该宏
#define __LP64__  1
//ELF32启用该宏
//#define __work_around_b_24465209__  1
  
/*
//https://android.googlesource.com/platform/bionic/+/master/linker/Android.bp
架构为32位 定义__work_around_b_24465209__宏
arch: {
        arm: {cflags: ["-D__work_around_b_24465209__"],},
        x86: {cflags: ["-D__work_around_b_24465209__"],},
    }
*/
  
//android-platform\bionic\libc\include\link.h
#if defined(__LP64__)
#define ElfW(type) Elf64_ ## type
#else
#define ElfW(type) Elf32_ ## type
#endif
  
//android-platform\bionic\linker\linker_common_types.h
// Android uses RELA for LP64.
#if defined(__LP64__)
#define USE_RELA 1
#endif
  
//android-platform\bionic\libc\kernel\uapi\asm-generic\int-ll64.h
//__signed__-->signed
typedef signed char __s8;
typedef unsigned char __u8;
typedef signed short __s16;
typedef unsigned short __u16;
typedef signed int __s32;
typedef unsigned int __u32;
typedef signed long long __s64;
typedef unsigned long long __u64;
  
//A12-src\msm-google\include\uapi\linux\elf.h
/* 32-bit ELF base types. */
typedef __u32   Elf32_Addr;
typedef __u16   Elf32_Half;
typedef __u32   Elf32_Off;
typedef __s32   Elf32_Sword;
typedef __u32   Elf32_Word;
  
/* 64-bit ELF base types. */
typedef __u64   Elf64_Addr;
typedef __u16   Elf64_Half;
typedef __s16   Elf64_SHalf;
typedef __u64   Elf64_Off;
typedef __s32   Elf64_Sword;
typedef __u32   Elf64_Word;
typedef __u64   Elf64_Xword;
typedef __s64   Elf64_Sxword;
  
typedef struct dynamic{
  Elf32_Sword d_tag;
  union{
    Elf32_Sword d_val;
    Elf32_Addr  d_ptr;
  } d_un;
} Elf32_Dyn;
  
typedef struct {
  Elf64_Sxword d_tag;       /* entry tag value */
  union {
    Elf64_Xword d_val;
    Elf64_Addr d_ptr;
  } d_un;
} Elf64_Dyn;
  
typedef struct elf32_rel {
  Elf32_Addr    r_offset;
  Elf32_Word    r_info;
} Elf32_Rel;
  
typedef struct elf64_rel {
  Elf64_Addr r_offset;  /* Location at which to apply the action */
  Elf64_Xword r_info;   /* index and type of relocation */
} Elf64_Rel;
  
typedef struct elf32_rela{
  Elf32_Addr    r_offset;
  Elf32_Word    r_info;
  Elf32_Sword   r_addend;
} Elf32_Rela;
  
typedef struct elf64_rela {
  Elf64_Addr r_offset;  /* Location at which to apply the action */
  Elf64_Xword r_info;   /* index and type of relocation */
  Elf64_Sxword r_addend;    /* Constant addend used to compute value */
} Elf64_Rela;
  
typedef struct elf32_sym{
  Elf32_Word    st_name;
  Elf32_Addr    st_value;
  Elf32_Word    st_size;
  unsigned char st_info;
  unsigned char st_other;
  Elf32_Half    st_shndx;
} Elf32_Sym;
  
typedef struct elf64_sym {
  Elf64_Word st_name;       /* Symbol name, index in string tbl */
  unsigned char st_info;    /* Type and binding attributes */
  unsigned char st_other;   /* No defined meaning, 0 */
  Elf64_Half st_shndx;      /* Associated section index */
  Elf64_Addr st_value;      /* Value of the symbol */
  Elf64_Xword st_size;      /* Associated symbol size */
} Elf64_Sym;
  
  
#define EI_NIDENT   16
  
typedef struct elf32_hdr{
  unsigned char e_ident[EI_NIDENT];
  Elf32_Half    e_type;
  Elf32_Half    e_machine;
  Elf32_Word    e_version;
  Elf32_Addr    e_entry;  /* Entry point */
  Elf32_Off e_phoff;
  Elf32_Off e_shoff;
  Elf32_Word    e_flags;
  Elf32_Half    e_ehsize;
  Elf32_Half    e_phentsize;
  Elf32_Half    e_phnum;
  Elf32_Half    e_shentsize;
  Elf32_Half    e_shnum;
  Elf32_Half    e_shstrndx;
} Elf32_Ehdr;
  
typedef struct elf64_hdr {
  unsigned char e_ident[EI_NIDENT]; /* ELF "magic number" */
  Elf64_Half e_type;
  Elf64_Half e_machine;
  Elf64_Word e_version;
  Elf64_Addr e_entry;       /* Entry point virtual address */
  Elf64_Off e_phoff;        /* Program header table file offset */
  Elf64_Off e_shoff;        /* Section header table file offset */
  Elf64_Word e_flags;
  Elf64_Half e_ehsize;
  Elf64_Half e_phentsize;
  Elf64_Half e_phnum;
  Elf64_Half e_shentsize;
  Elf64_Half e_shnum;
  Elf64_Half e_shstrndx;
} Elf64_Ehdr;
  
/* These constants define the permissions on sections in the program
   header, p_flags. */
#define PF_R        0x4
#define PF_W        0x2
#define PF_X        0x1
  
typedef struct elf32_phdr{
  Elf32_Word    p_type;
  Elf32_Off p_offset;
  Elf32_Addr    p_vaddr;
  Elf32_Addr    p_paddr;
  Elf32_Word    p_filesz;
  Elf32_Word    p_memsz;
  Elf32_Word    p_flags;
  Elf32_Word    p_align;
} Elf32_Phdr;
  
typedef struct elf64_phdr {
  Elf64_Word p_type;
  Elf64_Word p_flags;
  Elf64_Off p_offset;       /* Segment file offset */
  Elf64_Addr p_vaddr;       /* Segment virtual address */
  Elf64_Addr p_paddr;       /* Segment physical address */
  Elf64_Xword p_filesz;     /* Segment size in file */
  Elf64_Xword p_memsz;      /* Segment size in memory */
  Elf64_Xword p_align;      /* Segment alignment, file & memory */
} Elf64_Phdr;
  
typedef struct elf32_shdr {
  Elf32_Word    sh_name;
  Elf32_Word    sh_type;
  Elf32_Word    sh_flags;
  Elf32_Addr    sh_addr;
  Elf32_Off sh_offset;
  Elf32_Word    sh_size;
  Elf32_Word    sh_link;
  Elf32_Word    sh_info;
  Elf32_Word    sh_addralign;
  Elf32_Word    sh_entsize;
} Elf32_Shdr;
  
typedef struct elf64_shdr {
  Elf64_Word sh_name;       /* Section name, index in string tbl */
  Elf64_Word sh_type;       /* Type of section */
  Elf64_Xword sh_flags;     /* Miscellaneous section attributes */
  Elf64_Addr sh_addr;       /* Section virtual addr at execution */
  Elf64_Off sh_offset;      /* Section file offset */
  Elf64_Xword sh_size;      /* Size of section in bytes */
  Elf64_Word sh_link;       /* Index of another section */
  Elf64_Word sh_info;       /* Additional section information */
  Elf64_Xword sh_addralign; /* Section alignment */
  Elf64_Xword sh_entsize;   /* Entry size if section holds table */
} Elf64_Shdr;
  
  
//android-platform\bionic\linker\linker_soinfo.h
typedef void (*linker_dtor_function_t)();
typedef void (*linker_ctor_function_t)(int, char**, char**);
  
#if defined(__work_around_b_24465209__)
#define SOINFO_NAME_LEN 128
#endif
  
struct soinfo {
#if defined(__work_around_b_24465209__)
  char old_name_[SOINFO_NAME_LEN];
#endif
  const ElfW(Phdr)* phdr;
  size_t phnum;
#if defined(__work_around_b_24465209__)
  ElfW(Addr) unused0; // DO NOT USE, maintained for compatibility.
#endif
  ElfW(Addr) base;
  size_t size;
  
#if defined(__work_around_b_24465209__)
  uint32_t unused1;  // DO NOT USE, maintained for compatibility.
#endif
  
  ElfW(Dyn)* dynamic;
  
#if defined(__work_around_b_24465209__)
  uint32_t unused2; // DO NOT USE, maintained for compatibility
  uint32_t unused3; // DO NOT USE, maintained for compatibility
#endif
  
  soinfo* next;
  uint32_t flags_;
  
  const char* strtab_;
  ElfW(Sym)* symtab_;
  
  size_t nbucket_;
  size_t nchain_;
  uint32_t* bucket_;
  uint32_t* chain_;
  
#if !defined(__LP64__)
  ElfW(Addr)** unused4; // DO NOT USE, maintained for compatibility
#endif
  
#if defined(USE_RELA)
  ElfW(Rela)* plt_rela_;
  size_t plt_rela_count_;
  
  ElfW(Rela)* rela_;
  size_t rela_count_;
#else
  ElfW(Rel)* plt_rel_;
  size_t plt_rel_count_;
  
  ElfW(Rel)* rel_;
  size_t rel_count_;
#endif
  
  linker_ctor_function_t* preinit_array_;
  size_t preinit_array_count_;
  
  linker_ctor_function_t* init_array_;
  size_t init_array_count_;
  linker_dtor_function_t* fini_array_;
  size_t fini_array_count_;
  
  linker_ctor_function_t init_func_;
  linker_dtor_function_t fini_func_;
  
/*
#if defined(__arm__)
  // ARM EABI section used for stack unwinding.
  uint32_t* ARM_exidx;
  size_t ARM_exidx_count;
#endif
  size_t ref_count_;
//怎么找不到link_map这个类型的声明...
  link_map link_map_head;
  
  bool constructors_called;
  
  // When you read a virtual address from the ELF file, add this
  // value to get the corresponding address in the process' address space.
  ElfW(Addr) load_bias;
  
#if !defined(__LP64__)
  bool has_text_relocations;
#endif
  bool has_DT_SYMBOLIC;
*/
};
```

a1 即是 soinfo 结构体指针，但是还是存在一些未知偏移，所以有理由怀疑这个结构体可能经过魔改。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740146251456-00f7faee-cb30-4ebe-bbe5-4cc08095213a.png)

从 <font>sub_3C94</font > 被调用的地方开始分析，也就是 < font>sub_49F0</font>.

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740186189689-7504b853-8264-420d-b4e6-d9903093b180.png)

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740186209809-38f38264-c81a-4f57-b305-56def1ed5a19.png)

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740186219948-b2f0a393-1727-4235-a14b-b62b42e362a5.png)

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740186277815-82d63242-4e1a-4594-800f-fd16ea5f59ed.png)

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740186370689-9aad33f7-5f49-4113-b951-3ba8c3b85186.png)

0x38 是循环的步长，v6 为结束的地址，而 0x38 刚好是程序头表的大小，所以 v5 应该是 ELF64_Phdr * 类型。

所以我们只要将 a1 参数输出就可以得到解密后的 phdr 表。

```
function my_hook_dlopen(soName = '') {
    Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"),
        {
            onEnter: function (args) {
                var pathptr = args[0];
                if (pathptr !== undefined && pathptr != null) {
                    var path = ptr(pathptr).readCString();
                    if (path.indexOf(soName) >= 0) {
                        this.is_can_hook = true;
                         
                    }
                }
            },
            onLeave: function (retval) {
                if (this.is_can_hook) {
                    hook_proc_self_maps();
                    hook_native();//在so加载后再进行hook
                }
            }
        }
    );
}
function hook_native(){
    var module=Process.findModuleByName("libjiagu_64.so");
    Interceptor.attach(module.base.add(0x5E6C),{
        onEnter: function(args){
            console.log(hexdump(args[0],{
                offset:0,
                length:0x38*0x6+0x20,//这里加0x20应该是为了保险多dump了一点，实际长度应该只有0x38*6
                header: true,
                ansi:true
 
            }));
            console.log(args[1])
            console.log(args[2])
            console.log(`base = ${module.base}`)
        }
 
    }
)
}
var dump_once=false;
function hook_proc_self_maps() {
    const openPtr = Module.getExportByName(null, 'open');
    const open = new NativeFunction(openPtr, 'int', ['pointer', 'int']);
    var fakePath = "/data/data/com.oacia.apk_protect/maps_nonexistent";
    Interceptor.replace(openPtr, new NativeCallback(function (pathnameptr, flag) {
        var pathname = Memory.readUtf8String(pathnameptr);
        console.log("open",pathname);//,Process.getCurrentThreadId()
        if (pathname.indexOf("maps") >= 0) {
            console.log("find",pathname+", redirect to",fakePath);
            var filename = Memory.allocUtf8String(fakePath);
            return open(filename, flag);
        }
        if (pathname.indexOf("dex") >= 0) {
            if(!dump_once){
                dump_once = true;             
            }
        }
        var fd = open(pathnameptr, flag);
        return fd;
    }, 'int', ['pointer', 'int']));
}
setImmediate(my_hook_dlopen("libjiagu_64.so"));
```

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740276563641-a7030446-515d-4ee6-a4cd-058569733809.png)

cyberchef 处理一下 dump 的数据。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740277489296-753c9bb5-05fc-47fc-8d85-c48428385d86.png)

010 中按 insert 切换为覆盖模式，然后 ctrl+shift+v 覆盖数据。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740277512548-cf68dbfd-26f8-4882-89cb-063f8a196dff.png)

但是这里 phdr 在 soinfo 结构体里是偏移 0x0 的地方，但是 sub_5E6C 传入的参数一却是 a1+232 偏移。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740295515477-0bfbb61a-0630-484a-a0ae-4901251330a8.png)

重新对 soinfo 结构体进行编辑

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740296170385-230729aa-2c9b-4108-871f-5034b8a3f1f1.png)

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740296177823-de938724-43d6-4080-b464-115037472ad2.png)

现在的传参就很正确，这里也佐证了 soinfo 被魔改的猜测。

使用作者的 ida 插件来跟踪一下函数的调用 [stalker_trace_so](elink@70bK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6G2j5h3y4A6j5g2)9J5c8Y4y4@1j5h3I4C8k6i4u0Q4y4h3k6@1M7X3q4U0k6g2)9#2k6Y4y4G2)

获得函数调用链：![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740297412054-b2a200d2-a3ee-4780-a46e-d03c8f2393de.png)

ELF 解密流程
--------

从之前 dlopen 被调用的地方 <font>sub_3C94</font > 开始分析。<font>sub_4B54->sub_49F0->sub_3C94</font>

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740297462428-7b792be4-840e-4f53-a9a0-33c7e8146bab.png)

sub_4B54 有两个调用点，看函数调用链 sub_8000 应该是调用它的函数。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740297519552-e5c3b96e-25b1-4408-94db-aee6d39f82bb.png)

sub_8000 只有 data 段存放着函数指针，静态分析下没有调用，但是看到很有意思的东西。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740358365402-d3be62a0-e0cf-4f82-babf-8a39acf311b8.png)

如果这块存的都是函数指针的话，那上面附近的大概率也是，并且都被 sub_96E0 调用，恐怕有蹊跷。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740358435850-741aea48-9396-41d1-9372-721df3431645.png)

跟进 sub_6DBC

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740358464441-2531edbc-077d-4678-9a0c-41294a8fa967.png)

接着跟到 sub_6ADC

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740358498290-ad6d7e13-8060-4bc0-a075-82766ebea99d.png)

有一个很扎眼的异或，这里的 v8 就是参数一 + v3，v3 的值和传入的 a2 参数有关，a2 是个常数，可以试着去计算 v3 的值，不过这里直接大胆假设好了，这里是循环异或解密数据。

```
import idaapi
import idc
for addr in range(0x2E125,0x2E125+0x2D):
    a=idc.get_wide_byte(addr)
    idc.patch_byte(addr,a^0xA5)
print("done")
```

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740358638737-fc25be3b-c1a6-40fa-bca0-136cbd31c4ce.png)

确实解密出来一个正常的函数名，但是就算全部解密出来也不知道如果跟进下一步，还是跟回作者的思路，去分析函数调用链好了。

到 sub_5F20 时，这里有很明显的 RC4 算法特征

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740360291000-8cfe87ee-7de9-4801-958c-bc8d66ee97e1.png)

但是从函数调用链上来说的话，上几个函数都没有调用该函数的代码，那它是怎么调用的？

```
function get_call_ptr(){
    var module=Process.findModuleByName("libjiagu_64.so");
    Interceptor.attach(module.base.add(0x5F20), {
        onEnter: function(args) {
            // 获取当前调用栈的信息
            var backtrace = Thread.backtrace(this.context, Backtracer.ACCURATE);
            console.log('Backtrace:');
            backtrace.forEach(function(addr) {
                console.log(hexdump(addr, { length: 16, header: false, ansi: true }));
            });
     
            // 获取调用者的地址（即上层调用栈）
            var callerAddress = backtrace[1]; // 通常 backtrace[0] 是当前函数的地址，backtrace[1] 是调用者
            console.log('[+] sub_5F20 Caller address: ' + (callerAddress-module.base).toString(16)+'\n');
        },
        onLeave: function(retval) {
            // 在函数执行完后可以做进一步的处理
        }
    }
        );
     
}
```

将该函数在安置在该位置进行 hook

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740367307197-bcfeee86-6caa-4a6d-9685-819d7313d3df.png)

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740367319172-5abbf11f-4ee3-4ba0-b7b0-203cd896f2a1.png)

调用地址在 17710

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740375826731-906d1f8d-907d-4bed-bdf4-23037262ab0f.png)

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740367349482-76f9444f-6d24-4a05-a7c4-5b8d8c5a7a3b.png)

是通过函数指针调用的。最早的调用函数是 sub_C918，后面有 call_SYSV 的记录，这里先大概知道是使用的动态调用的方式。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740367460127-c773538b-1076-45d9-9ab9-156ac9ed8a94.png)

我们再回过头来看 rc4_init，HOOK 一下参数。

```
function hook_5f20_guess_rc4(){//像是RC4的样子,hook看看
    var module = Process.findModuleByName("libjiagu_64.so");
    Interceptor.attach(module.base.add(0x5f20), {
        // fd, buff, len
        onEnter: function (args) {
            console.log(hexdump(args[0], {
              offset: 0,// 相对偏移
              length: 0x10,//dump 的大小
              header: true,
              ansi: true
            }));
            console.log(args[1])
            console.log(hexdump(args[2], {
              offset: 0,// 相对偏移
              length: 256,//dump 的大小
              header: true,
              ansi: true
            }));
        },
        onLeave: function (ret) {
  
        }
    });
}
```

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740376992241-31a22e8d-4edf-4456-b4db-a5256cb1b838.png)

密钥：vUV4#.#SVt

之后查看函数调用链的下一个函数 sub_6044![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740378920971-119328b3-0324-4cbe-be54-821815986133.png)

很明显的 RC4 加密过程。

再 HOOK 一下 sub_6044 函数

```
function hook_rc4_enc(){
    var module = Process.findModuleByName("libjiagu_64.so");
    Interceptor.attach(module.base.add(0x6044), {
        // fd, buff, len
        onEnter: function (args) {
            rc4_enc_text_addr = args[0];
            rc4_enc_size = args[1];
            console.log(hexdump(args[0], {
              offset: 0,// 相对偏移
              length: 0x30,//dump 的大小
              header: true,
              ansi: true
            }));
            console.log(args[1])
  
        },
        onLeave: function (ret) {
            console.log(hexdump(rc4_enc_text_addr, {
              offset: 0,// 相对偏移
              length: 0x30,//dump 的大小
              header: true,
              ansi: true
            }));
        }
    });
}
```

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740382342517-8177213f-0dca-498b-bba0-e1b240a6836a.png)

是之前在 sub_8000 看到过的 0xb8010，从该函数的参数使用上来说，这个值是处理的数据的大小。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740382467674-41b70d2a-729b-4fd9-b554-7e43ea5a7007.png)

再看一下 v5[0] 的值

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740382488558-5dd1bccb-139c-455d-8a69-db56f0467df0.png)

和解密的数据是一致的，这段数据就是 RC4 加密过的。

接着看函数调用链，接下来是 uncompress，解压缩函数。

函数定义：

```
int uncompress(
    unsigned char *dest,        // 解压后的数据存放位置
    uLongf *destLen,            // 解压后的数据长度（输入输出参数）
    const unsigned char *source, // 压缩数据源
    uLong sourceLen             // 压缩数据源的长度
);
```

```
function hook_uncompress_res(){
  var module = Process.findModuleByName("libjiagu_64.so");
  Interceptor.attach(Module.findExportByName(null, "uncompress"), {
  onEnter: function (args) {
  console.log("hook uncompress")
  console.log(hexdump(args[2], {
  offset: 0,// 相对偏移
  length: 0x30,//dump 的大小
  header: true,
  ansi: true
}));
console.log(args[3])
dump_memory(args[2],args[3],`uncompress_${args[2]}_${args[3]}`)
},
onLeave: function (ret) {

}
});
}
```

解压缩的数据就是 RC4 解密后的数据![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740384909688-ecb2d91d-11d0-4de0-a072-2089f7d1ab80.png)

不包括前四个字节，根据加壳的一些做法可以猜测是大小。

有 ELF 的数据及解密方式，直接手动解密

```
import zlib
import struct
def RC4(data, key):
    S = list(range(256))
    j = 0
    out = []
  
    # KSA Phase
    for i in range(256):
        j = (j + S[i] + key[i % len(key)]) % 256
        S[i], S[j] = S[j], S[i]
  
    # PRGA Phase
    i = j = 0
    for ch in data:
        i = (i + 1) % 256
        j = (j + S[i]) % 256
        S[i], S[j] = S[j], S[i]
        out.append(ch ^ S[(S[i] + S[j]) % 256])
  
    return out
  
def RC4decrypt(ciphertext, key):
    return RC4(ciphertext, key)
  
  
wrap_elf_start = 0x2e270
wrap_elf_size = 0xb8010
key = b"vUV4#\x91#SVt"
with open('D:/Android-reverse-tool/so_fixer/fix.so','rb') as f:
    wrap_elf = f.read()
  
  
# 对密文进行解密
dec_compress_elf = RC4decrypt(wrap_elf[wrap_elf_start:wrap_elf_start+wrap_elf_size], key)
dec_elf = zlib.decompress(bytes(dec_compress_elf[4::]))
with open('wrap_elf','wb') as f:
    f.write(dec_elf)
```

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740385457837-d1f6e022-3639-4f17-a5ff-77151732dce8.png)

可以找到 elf 标志位，但是前面有一堆 D3 数据。

将 elf 这两部分切割

```
with open('wrap_elf','rb') as f:
    wrap_elf=f.read()
    ELF_magic=bytes([0x7F,0x45,0x4C,0x46])
    for i in range(len(wrap_elf)-len(ELF_magic)+1):
        if(wrap_elf[i:i+len(ELF_magic)]==ELF_magic):
            print(hex(i))
            with open('wrap_elf_part1','wb') as f:
                f.write(wrap_elf[0:i])
            with open('wrap_elf_part2','wb') as f:
                f.write(wrap_elf[i::])
            break
```

之后再接着函数调用链往下看 elf 的解密，sub_5B08

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740386140471-f32b03b8-762c-4e38-8a2a-302e88ba28d3.png)

又是 0x38，program_header_table 的大小，可以猜测又是通过解析 elf 结构来进行一些关键操作。

HOOK 下进行异或的 v5

```
function hook_5B08(){
    var module = Process.findModuleByName("libjiagu_64.so");
    Interceptor.attach(module.base.add(0x5B08), {
        onEnter: function (args) {
            console.log("[+] All arguments:");
            // 遍历并打印前 3 个参数
            for (var i = 0; i < 3; i++) {
                console.log("[+] Argument " + i + ": " + args[i]);
            }
 
            // 获取指向 v4 的指针，假设 args[1] 是一个指针
            
            var v5=ptr(args[1]).add(16).readPointer();
            console.log("[+] Address:",v5);
            console.log("[+] v5:\n" + hexdump(v5, {
                offset: 0, // 相对偏移
                length: 0x30, // dump 的大小
                header: true,
                ansi: true
            }));
        }
    });
}
```

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740391078077-59f72b3f-4b1f-43e9-b166-d00642c17099.png)

和 010 中看到的不明数据吻合

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740397099341-2cf0e8e1-2b55-45e7-a938-fa30fc10a6b0.png)

这个也可以找到他解密完的位置去进行 dump，但是我觉得分析一下解密流程也算学习的一部分了。

首先只是从表面来看的话，似乎设计到加密的部分只有异或，包括 vdupq_n_s8 和 veorq_s8 似乎也只是实现另类的异或。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740403763676-43aae586-a4a7-4766-a704-e444d787ec28.png)

先从第一个循环来看，v5 异或的值，重点应该聚焦于异或的数据，也就是 v18

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740403877283-88ae8ff8-74fd-4f9e-901e-738f24df7631.png)

v18 是 v10+v12，而 v12 在进入该循环时被赋值为 0，也就是说 v18 就是 v10，而 v10——》result——》strtol，

说真的这个 strtol 的使用在这里真的很迷，搞不太懂这里的逻辑，所以我选择了用 frida 的 stalker 来直接跟踪一下返回值。

```
function trace_entry(baseAddr, tatgetAddr){
    // 使用 Interceptor.attach 来 hook 目标函数
    Interceptor.attach(tatgetAddr, {
         
        // 在函数调用进入时触发的钩子
        onEnter: function(args){
            console.log("enter tatgetAddr====================================================================");
            // 获取当前线程的线程 ID
            this.pid = Process.getCurrentThreadId();
           
            // 使用 Stalker API 跟踪当前线程的执行
            Stalker.follow(this.pid, {
                events: {
                    // 设置哪些事件需要被追踪，这里暂时不追踪调用、返回、执行等事件
                    call: false, // 不追踪函数调用
                    ret: false, // 不追踪函数返回
                    exec: false, // 不追踪每一条执行的指令
                    block: false, // 不追踪执行的内存块
                    compile: false // 不追踪已编译的代码
                },
 
                // 当事件被接收到时执行的钩子，这里未使用
                onReceive: function(events){
                    // 目前没有处理接收到的事件
                },
                transform: function(iterator) {
                    var instruction = iterator.next(); // 获取下一条指令的迭代器
                    const startAddress = instruction.address; // 获取当前指令的地址
                    var isModule=startAddress.compare(baseAddr.add(0x5B08))>=0&&startAddress.compare(baseAddr.add(0x5B9C))<0;                                                 
                    do {
                        if(isModule) {
                            console.log(instruction.address.sub(baseAddr) + "\t:\t" + instruction.toString());
                            //var is_x0=instruction.toString().includes('x0');
                            var now_address=instruction.address.sub(baseAddr);
                            // 将一个回调注入到当前的指令（putCallout）中
                            if(now_address==0x5B74){
                            iterator.putCallout((context) => {
                                // 从 x0 寄存器读取指向的字符串
                                var x0_ptr = ptr(context["x0"]);
                                // 打印字符串，假设这是从内存读取的返回值 
                                if(x0_ptr!=null)                                                        
                                console.log("[+] "+now_address+": x0: \n"+context['x0']+hexdump(x0_ptr,{
                                    offset: 0, // 相对偏移
                                    length: 0x30, // dump 的大小
                                    header: true,
                                    ansi: true
                            }));
                                
                            });
                        }
                        }
                        // 保持继续迭代
                        iterator.keep();
                    } while ((instruction = iterator.next()) !== null); // 继续迭代，直到没有更多指令
                },
                onCallSummary:function(summary){
                    // 目前未使用该回调
                }
            });
        },
        onLeave: function(retval){
            // 停止跟踪当前线程的执行
            Stalker.unfollow(this.pid);
            // 打印返回值
            console.log("retval:" + retval);
            console.log("leave tatgetAddr====================================================================");
        }
    });
}
```

该函数的定义来说返回的应该是个整数，所以就是 b40000704205f130，尝试几次没有变化，似乎是固定的值。![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740453437965-62f528eb-e1f5-4776-8a1b-ad310b3e5b61.png)

但是不管是从参数还是返回值来说，都相当奇怪，不排除可能自己对该函数进行了 hook 吧。

这里有一个既视感很强的操作

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740461687380-c845e05f-5fcc-4399-a6eb-adc5cbbd5c33.png)

prctl 的第二个参数正好跳过了前面赋值给其他变量的五个字节，同时四字节作为第三个参数，而参数 1 result 在前面被赋值给了 v10，v10 又赋值给 v18，v18 正好是要进行异或的数据。

所以这里的 prtcl 像是 memcpy 的操作。

改一下前面的 frida-stlaker 中输出 x0 的地址然后 HOOK，不出所料，该函数操作后 result 指向的地址存储了 010 中看到的五字节之后的数据![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740462138949-7dfb53f9-2ba0-4ec4-af4a-b532727f7d14.png)

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740462239716-0c659fc4-b28a-4982-bb94-66ae6559ae30.png)

那这样看来前面的 strtol 实际的函数可能是类似 malloc 的函数分配了空间。

所以这段代码实际就是以第一字节为异或值，2 到 5 字节为大小，取之后的数据异或。同样的，下面也都是类似的操作。

为了验证下函数确实被替换了的猜想，我决定用 frida-stalker 对函数进行追踪。

```
function hook_import_function(){
    var module = Process.getModuleByName('libjiagu_64.so');
    var strtol_ptr=module.base.add(0x2DE00);
    console.log("strtol_ptr address: " + strtol_ptr);
    var strtol_address = Memory.readPointer(strtol_ptr);
    console.log("strtol address: " + strtol_address);
    Interceptor.attach(strtol_address, {
         
        // 在函数调用进入时触发的钩子
        onEnter: function(args){
            console.log("enter strtol_address====================================================================");
            // 获取当前线程的线程 ID
            this.pid = Process.getCurrentThreadId();
           
            // 使用 Stalker API 跟踪当前线程的执行
            Stalker.follow(this.pid, {
                events: {
                    // 设置哪些事件需要被追踪，这里暂时不追踪调用、返回、执行等事件
                    call: false, // 不追踪函数调用
                    ret: false, // 不追踪函数返回
                    exec: false, // 不追踪每一条执行的指令
                    block: false, // 不追踪执行的内存块
                    compile: false // 不追踪已编译的代码
                },
 
                // 当事件被接收到时执行的钩子，这里未使用
                onReceive: function(events){
                    // 目前没有处理接收到的事件
                },
 
                // 转换每条指令，执行特定的逻辑
                transform: function(iterator) {
                    var instruction = iterator.next(); // 获取下一条指令的迭代器
                    const startAddress = instruction.address; // 获取当前指令的地址   ******************************这里有问题
                    var isModule=startAddress.compare(strtol_address)>=0&&startAddress.compare(strtol_address)<=5;                                                 
                    
                    // 用 do...while 循环遍历当前指令和后续指令，直到没有更多指令
                    do {
                        if(isModule) {
                            console.log(instruction.address.sub(module.base) + "\t:\t" + instruction.toString());
                        }
                        // 保持继续迭代
                        iterator.keep();
                    } while ((instruction = iterator.next()) !== null); // 继续迭代，直到没有更多指令
                },
 
                // 不使用的回调函数，假设用于汇总
                onCallSummary:function(summary){
                    // 目前未使用该回调
                }
            });
        },
         
        // 在目标函数返回时触发的钩子
        onLeave: function(retval){
            // 停止跟踪当前线程的执行
            Stalker.unfollow(this.pid);
            // 打印返回值
            console.log("retval:" + retval);
            console.log("leave strtol_address====================================================================");
        }
    });
}
```

代码处理的不是很好，因为函数被多次调用了，所以输出的数据很多，而且限制的范围没有起作用。

如无意外的话，这些应该是 strtol 的汇编内容，返回值和之前 hook 到的是相近的。

```
enter strtol_address====================================================================
0x334028fb0     :       adrp x8, #0x7aaf525000
0x334028fb4     :       mov x20, x1
0x334028fb8     :       add x8, x8, #0x50
0x334028fbc     :       mov x21, x0
0x334028fc0     :       ldar x8, [x8]
0x334028fc4     :       cbnz x8, #0x7aaef3d008
0x334028fc8     :       mov x0, x21
0x334028fcc     :       mov x1, x20
0x334028fd0     :       bl #0x7aaef428e0
0x33402e8e0     :       sub sp, sp, #0x20
0x33402e8e4     :       stp x29, x30, [sp, #0x10]
0x33402e8e8     :       add x29, sp, #0x10
0x33402e8ec     :       umulh x8, x1, x0
0x33402e8f0     :       cmp xzr, x8
0x33402e8f4     :       b.ne #0x7aaef42924
0x33402e8f8     :       mul x1, x1, x0
0x33402e8fc     :       adrp x0, #0x7aaefd7000
0x33402e900     :       add x0, x0, #0x440
0x33402e904     :       mov w2, wzr
0x33402e908     :       mov w3, #0x10
0x33402e90c     :       mov w4, #1
0x33402e910     :       bl #0x7aaef429a0
0x33402e9a0     :       sub sp, sp, #0x70
0x33402e9a4     :       stp x29, x30, [sp, #0x10]
0x33402e9a8     :       add x29, sp, #0x10
0x33402e9ac     :       stp x28, x27, [sp, #0x20]
0x33402e9b0     :       stp x26, x25, [sp, #0x30]
0x33402e9b4     :       stp x24, x23, [sp, #0x40]
0x33402e9b8     :       stp x22, x21, [sp, #0x50]
0x33402e9bc     :       stp x20, x19, [sp, #0x60]
0x33402e9c0     :       mrs x28, tpidr_el0
0x33402e9c4     :       ldr x8, [x28, #0x30]
0x33402e9c8     :       mov w24, w4
0x33402e9cc     :       mov x22, x3
0x33402e9d0     :       mov w21, w2
0x33402e9d4     :       mov x19, x1
0x33402e9d8     :       mov x20, x0
0x33402e9dc     :       cmp x8, #1
0x33402e9e0     :       b.ls #0x7aaef42cd0
0x33402e9e4     :       ldr w23, [x20, #0x80]
0x33402e9e8     :       mov w8, #1
0x33402e9ec     :       movk w8, #0x100, lsl #16
0x33402e9f0     :       cmp x22, x8
0x33402e9f4     :       b.hs #0x7aaef42cf4
0x33402e9f8     :       cmp x22, #0x10
0x33402e9fc     :       mov w8, #0x10
0x33402ea00     :       csel x26, x22, x8, hi
0x33402ea04     :       tbz w24, #0, #0x7aaef42a10
0x33402ea08     :       mov w25, #1
0x33402ea0c     :       b #0x7aaef42a24
0x33402ea24     :       add x8, x19, #0xf
0x33402ea28     :       lsr x9, x19, #0x28
0x33402ea2c     :       and x8, x8, #0xfffffffffffffff0
0x33402ea30     :       add x27, x26, x8
0x33402ea34     :       cbnz x9, #0x7aaef42d04
0x33402ea38     :       mov w8, #0x10
0x33402ea3c     :       str xzr, [sp, #8]
0x33402ea40     :       movk w8, #1, lsl #16
0x33402ea44     :       cmp x27, x8
0x33402ea48     :       b.hi #0x7aaef42d68
0x33402ea4c     :       cmp x27, #0x21
0x33402ea50     :       b.hs #0x7aaef42a84
0x33402ea84     :       sub x8, x27, #0x11
0x33402ea88     :       sub x9, x27, #0x10
0x33402ea8c     :       cmp x9, #0x40
0x33402ea90     :       b.hi #0x7aaef42ab4
0x33402eab4     :       clz x9, x8
0x33402eab8     :       mov w10, #0x39
0x33402eabc     :       sub x9, x10, x9
0x33402eac0     :       adrp x10, #0x7aaef20000
0x33402eac4     :       add x10, x10, #0x5ec
0x33402eac8     :       lsr x8, x8, x9
0x33402eacc     :       add x8, x8, x9, lsl #6
0x33402ead0     :       add x8, x8, x10
0x33402ead4     :       ldurb w22, [x8, #-0x40]
0x33402ead8     :       ldr x8, [x28, #0x30]
0x33402eadc     :       and x28, x8, #0xfffffffffffffffe
0x33402eae0     :       add x0, x28, #0xfcc
0x33402eae4     :       bl #0x7aaef40a50
0x33402ca50     :       mov w8, wzr
0x33402ca54     :       mov w9, #1
0x33402ca58     :       casa w8, w9, [x0]
0x33402ca5c     :       cmp w8, #0
0x33402ca60     :       cset w0, eq
0x33402ca64     :       ret
0x33402eae8     :       tbnz w0, #0, #0x7aaef42a6c
0x33402ea6c     :       str xzr, [x28, #0xfd0]
0x33402ea70     :       mov w8, #0x78
0x33402ea74     :       madd x24, x22, x8, x28
0x33402ea78     :       ldr w8, [x24]
0x33402ea7c     :       cbnz w8, #0x7aaef42b3c
0x33402eb3c     :       mov w9, #0x78
0x33402eb40     :       sub w8, w8, #1
0x33402eb44     :       madd x9, x22, x9, x28
0x33402eb48     :       str w8, [x24]
0x33402eb4c     :       add x10, x9, w8, uxtw #2
0x33402eb50     :       ldr x9, [x9, #8]
0x33402eb54     :       ldr w8, [x10, #0x10]
0x33402eb58     :       ldr x10, [x28, #0xf88]
0x33402eb5c     :       add x10, x10, x9
0x33402eb60     :       str x10, [x28, #0xf88]
0x33402eb64     :       ldr x10, [x28, #0xf90]
0x33402eb68     :       sub x9, x10, x9
0x33402eb6c     :       mov w10, #0xc0
0x33402eb70     :       str x9, [x28, #0xf90]
0x33402eb74     :       ldr x9, [x28, #0xfa0]
0x33402eb78     :       nop
0x33402eb7c     :       madd x9, x22, x10, x9
0x33402eb80     :       ldr x9, [x9, #0x60]
0x33402eb84     :       adds x24, x9, x8, lsl #4
0x33402eb88     :       b.eq #0x7aaef42c50
0x33402eb8c     :       add x0, x28, #0xfcc
0x33402eb90     :       bl #0x7aaef40ae0
0x33402cae0     :       mov w8, #-1
0x33402cae4     :       ldaddl w8, w8, [x0]
0x33402cae8     :       cmp w8, #1
0x33402caec     :       b.ne #0x7aaef40af4
0x33402caf0     :       ret
0x33402eb94     :       cbz x22, #0x7aaef42d68
0x33402eb98     :       add x8, x26, x24
0x33402eb9c     :       neg x9, x26
0x33402eba0     :       add x8, x8, #0xf
0x33402eba4     :       and x26, x8, x9
0x33402eba8     :       tbnz w23, #7, #0x7aaef42ca0
0x33402ebac     :       orr x23, x24, #0x200000000000000
0x33402ebb0     :       orr x28, x26, #0x200000000000000
0x33402ebb4     :       cbnz w25, #0x7aaef42d38
0x33402ed38     :       adrp x8, #0x7aaef20000
0x33402ed3c     :       add x8, x8, #0x56c
0x33402ed40     :       cmp w25, #1
0x33402ed44     :       mov w9, #-0x55
0x33402ed48     :       csel w1, wzr, w9, eq
0x33402ed4c     :       mov x0, x23
0x33402ed50     :       add x8, x8, x22, lsl #2
0x33402ed54     :       ldur w2, [x8, #-4]
0x33402ed58     :       bl #0x7aaefca330
0x3340b6330     :       adrp x16, #0x7aaefcf000
0x3340b6334     :       ldr x17, [x16, #0xd08]
0x3340b6338     :       add x16, x16, #0xd08
0x3340b633c     :       br x17
0x334039fe0     :       dup v0.16b, w1
0x334039fe4     :       add x4, x0, x2
0x334039fe8     :       cmp x2, #0x60
0x334039fec     :       b.hi #0x7aaef4e064
0x33403a064     :       and w1, w1, #0xff
0x33403a068     :       and x3, x0, #0xfffffffffffffff0
0x33403a06c     :       str q0, [x0]
0x33403a070     :       cmp x2, #0x100
0x33403a074     :       ccmp w1, #0, #0, hs
0x33403a078     :       b.eq #0x7aaef4e0a8
0x33403a0a8     :       mrs x5, dczid_el0
0x33403a0ac     :       tbnz w5, #4, #0x7aaef4e07c
0x33403a0b0     :       and w5, w5, #0xf
0x33403a0b4     :       cmp w5, #4
0x33403a0b8     :       b.ne #0x7aaef4e108
0x33403a0bc     :       str q0, [x3, #0x10]
0x33403a0c0     :       stp q0, q0, [x3, #0x20]
0x33403a0c4     :       and x3, x3, #0xffffffffffffffc0
0x33403a0c8     :       stp q0, q0, [x3, #0x40]
0x33403a0cc     :       stp q0, q0, [x3, #0x60]
0x33403a0d0     :       sub x2, x4, x3
0x33403a0d4     :       sub x2, x2, #0x100
0x33403a0d8     :       add x3, x3, #0x80
0x33403a0dc     :       nop
0x33403a0e0     :       dc zva, x3
0x33403a0e4     :       add x3, x3, #0x40
0x33403a0e8     :       subs x2, x2, #0x40
0x33403a0ec     :       b.hi #0x7aaef4e0e0
0x33403a0e0     :       dc zva, x3
0x33403a0e4     :       add x3, x3, #0x40
0x33403a0e8     :       subs x2, x2, #0x40
0x33403a0ec     :       b.hi #0x7aaef4e0e0
0x33403a0f0     :       stp q0, q0, [x3]
0x33403a0f4     :       stp q0, q0, [x3, #0x20]
0x33403a0f8     :       stp q0, q0, [x4, #-0x40]
0x33403a0fc     :       stp q0, q0, [x4, #-0x20]
0x33403a100     :       ret
0x33402ed5c     :       b #0x7aaef42bb8
0x33402ebb8     :       mov w8, wzr
0x33402ebbc     :       mov x25, x24
0x33402ebc0     :       mov x27, x26
0x33402ebc4     :       mov x24, x23
0x33402ebc8     :       add x9, x25, #0x10
0x33402ebcc     :       subs x10, x26, x9
0x33402ebd0     :       b.ne #0x7aaef42d10
0x33402ebd4     :       mov x9, xzr
0x33402ebd8     :       ldr w10, [sp, #8]
0x33402ebdc     :       lsl w11, w21, #0xa
0x33402ebe0     :       add w12, w26, w19
0x33402ebe4     :       cmp w8, #0
0x33402ebe8     :       and x11, x11, #0xc00
0x33402ebec     :       ldr w8, [x20]
0x33402ebf0     :       sub w10, w10, w12
0x33402ebf4     :       bfxil x11, x22, #0, #8
0x33402ebf8     :       csel w10, w10, w19, ne
0x33402ebfc     :       orr x9, x11, x9
0x33402ec00     :       lsl w10, w10, #0xc
0x33402ec04     :       crc32cx w8, w8, x28
0x33402ec08     :       orr x9, x9, x10
0x33402ec0c     :       orr x9, x9, #0x100
0x33402ec10     :       crc32cx w8, w8, x9
0x33402ec14     :       eor w8, w8, w8, lsr #16
0x33402ec18     :       bfi x9, x8, #0x30, #0x10
0x33402ec1c     :       adrp x8, #0x7aaefcf000
0x33402ec20     :       ldr x8, [x8, #0x9e8]
0x33402ec24     :       stur x9, [x28, #-0x10]
0x33402ec28     :       cbnz x8, #0x7aaef42d28
0x33402ec2c     :       mov x0, x27
0x33402ec30     :       ldp x20, x19, [sp, #0x60]
0x33402ec34     :       ldp x22, x21, [sp, #0x50]
0x33402ec38     :       ldp x24, x23, [sp, #0x40]
0x33402ec3c     :       ldp x26, x25, [sp, #0x30]
0x33402ec40     :       ldp x28, x27, [sp, #0x20]
0x33402ec44     :       ldp x29, x30, [sp, #0x10]
0x33402ec48     :       add sp, sp, #0x70
0x33402ec4c     :       ret
0x33402e914     :       cbz x0, #0x7aaef42940
0x33402e918     :       ldp x29, x30, [sp, #0x10]
0x33402e91c     :       add sp, sp, #0x20
0x33402e920     :       ret
0x334028fd4     :       cmp x0, #0
0x334028fd8     :       mov x19, x0
0x334028fdc     :       cset w22, eq
0x334028fe0     :       cbz x0, #0x7aaef3d028
0x334028fe4     :       adrp x8, #0x7aaf525000
0x334028fe8     :       ldrb w8, [x8, #0x4f]
0x334028fec     :       cmp w22, #0
0x334028ff0     :       ldp x22, x21, [sp, #0x10]
0x334028ff4     :       orr x8, x19, x8, lsl #56
0x334028ff8     :       ldp x20, x19, [sp, #0x20]
0x334028ffc     :       csel x0, xzr, x8, ne
0x334029000     :       ldp x29, x30, [sp], #0x30
0x334029004     :       ret
retval:0xb400007972ace1b0
```

其中没有明显的字符串处理，数值转换等操作，喂给 gpt 反编译后简要分析大致执行一系列内存操作和计算，然后返回一个分配的空间或地址。

同理对 prctl 进行操作分析。

```
var hasHooked = false;
var hshooked=true;
function hook_import_function(){
    var module = Process.getModuleByName('libjiagu_64.so');
    var strtol_ptr=module.base.add(0x2DD50);
    console.log("strtol_ptr address: " + strtol_ptr);
    var strtol_address = Memory.readPointer(strtol_ptr);
    console.log("strtol address: " + strtol_address);
    Interceptor.attach(strtol_address, {
         
        // 在函数调用进入时触发的钩子
        onEnter: function(args){
           if(!hasHooked){
            console.log("enter strtol_address====================================================================");
            // 获取当前线程的线程 ID
            this.pid = Process.getCurrentThreadId();
            hasHooked=true;
            hshooked=false;
            // 使用 Stalker API 跟踪当前线程的执行
            Stalker.follow(this.pid, {
                events: {
                    // 设置哪些事件需要被追踪，这里暂时不追踪调用、返回、执行等事件
                    call: false, // 不追踪函数调用
                    ret: false, // 不追踪函数返回
                    exec: false, // 不追踪每一条执行的指令
                    block: false, // 不追踪执行的内存块
                    compile: false // 不追踪已编译的代码
                },
 
                // 当事件被接收到时执行的钩子，这里未使用
                onReceive: function(events){
                    // 目前没有处理接收到的事件
                },
 
                // 转换每条指令，执行特定的逻辑
                transform: function(iterator) {
                    var instruction = iterator.next(); // 获取下一条指令的迭代器
                    const startAddress = instruction.address; // 获取当前指令的地址   ******************************这里有问题
                    var isModule=startAddress.compare(strtol_address)>=0;                                                 
                    
                    // 用 do...while 循环遍历当前指令和后续指令，直到没有更多指令
                    do {
                            if(isModule)
                            console.log(instruction.address.sub(module.base) + "\t:\t" + instruction.toString());
                         
                        // 保持继续迭代
                        iterator.keep();
                    } while ((instruction = iterator.next()) !== null); // 继续迭代，直到没有更多指令
                },
 
                // 不使用的回调函数，假设用于汇总
                onCallSummary:function(summary){
                    // 目前未使用该回调
                }
            });
        }
        },
         
        // 在目标函数返回时触发的钩子
        onLeave: function(retval){
            // 停止跟踪当前线程的执行
            if(!hshooked)
            {Stalker.unfollow(this.pid);
            // 打印返回值
            console.log("retval:" + retval+hexdump(retval,{
                offset:0,
                length:0x100,//这里加0x20应该是为了保险多dump了一点，实际长度应该只有0x38*6
                header: true,
                ansi:true
            }));
            console.log("leave strtol_address====================================================================");
            hshooked=true;
            }
        }
    });
}
```

返回的只有一个 return 的操作，不是很能理解。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740468335966-b0cf5fde-5b57-42b6-b64a-67fefc505c78.png)

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740468554212-f762c180-dbdb-485a-ad45-75ee20b71d88.png)

hexdump 下目标地址的数据，很惊奇，居然有 ELF 的 magic 标志。

我对 stalker 的使用只是初尝，想必有一些错误，想要再去深究这里的 prctl 的话想必要花费不少时间，暂且将中心放回到 elf 解密上吧。

根据 a1 各个偏移的使用可以大致构建出结构体，这里直接使用作者的了。

```
struct deal_extra
{
  char blank[72];
  int phnum;
  int *extra_part1;
  int phdr_size;
  char blank2[36];
  int *extra_part2;
  int *extra_part3;
  int *extra_part4;
  int *main_elf;
};
```

同理根据函数调用链 <font>sub_49F0->sub_5478(&v16, a1, v4)->sub_5B08(a1, a2, a3)</font > 将之前函数的参数也重新定义。

将 sub_49F0 中的 v16 定义为 deal_extra，v7 定义为合适的结构体

```
struct deal_extra_B
{
  char blank[232];
  int *extra_part1;
  char blank1[8];
  int phnum;
  int *extra_part4;
  char blank2[24];
  int *extra_part2;
  char blank3[8];
  int *extra_part3;
};
```

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740469437880-1f837e4f-add5-41b0-a21e-5794417974df.png)

而之后的 v7 分别被传入了 sub_3C94 和 sub_4918 中

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740470546555-6bf96315-f8f9-49f2-b298-e9463e3dff7b.png)

而 sub_3C94 正是之前分析到的处理动态链接库的相关函数

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740470782642-5895584a-ca66-4232-8ccd-ab9c7ea3de62.png)

而这里被处理的 extra_part4 也就对应着. dynamic 段。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740471976144-85fea201-3d7f-4980-b885-3408a72d5adc.png)

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740471999225-3cae7b72-c700-418f-ba51-a081f3827b0e.png)

然后在 sub_4918 中，extra_part2 和 extra3 被传入 sub_4000 中。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740473189145-727f6872-1079-4ebb-af99-d887bb0c790d.png)

sub_4000 的 case 用来处理重定位

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740474396533-d482b8a8-1795-4d8c-af45-64c26bb9f66b.png)

hook 该函数查看一下两次调用时 case 的值。

```
function hook_4000(){
  var module = Process.findModuleByName("libjiagu_64.so");

  Interceptor.attach(module.base.add(0x4000), {
  onEnter: function (args) {
  console.log("[+] All arguments:");
  // 遍历并打印前 3 个参数
  for (var i = 0; i < 3; i++) {
  console.log("[+] Argument " + i + ": " + args[i]);
}
var a2_ptr=ptr(args[1].add(0x10));
console.log("switch with:",(a2_ptr.sub(8)).readInt().toString(16),'\n'); 
}
});
}
```

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740474435225-eb4281d6-965d-4eb1-ac7e-e424f7ed7f87.png)

分别是 403 和 402，也就是`<font>.rela.plt</font>`<font>(402 重定位) 和 </font>`<font>.rela.dyn</font>`<font>(403 重定位)</font>

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740474590188-178e86e3-b83b-404c-a782-b5d2d744e3d1.png)

<font> 我对安卓源码并不了解，但是作者在这里能够很清晰的理清思路，如果有空余时间的话阅读下大致代码应该也会在以后对逆向有帮助。</font>

<font> 最后的 extra_part1 被传入 sub_5E6C 中。</font>

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740474622975-34dee2f8-18bb-46cf-85b9-c9192f8739e1.png)

也就是前面分析了很久的解密 <font>program header table</font > 的函数。

所以四个数据组分别是：

*   `<font>数据组1</font>`<font> 表示 </font>`<font>program header table</font>`
*   `<font>数据组2</font>`<font> 表示 </font>`<font>.rela.plt</font>`
*   `<font>数据组3</font>`<font> 表示 </font>`<font>.rela.dyn</font>`
*   `<font>数据组4</font>`<font> 表示 </font>`<font>.dynamic</font>`

这里直接用作者的脚本来分离文件了。

```
import copy
import zlib
 
# RC4加密/解密算法实现
def RC4(data, key):
    S = list(range(256))  # 初始化S数组，包含0到255的整数
    j = 0
    out = []  # 用于存储加密或解密后的输出数据
 
    # KSA（Key Scheduling Algorithm）阶段
    for i in range(256):
        j = (j + S[i] + key[i % len(key)]) % 256  # 根据密钥对S数组进行初始化
        S[i], S[j] = S[j], S[i]  # 交换S[i]和S[j]
 
    # PRGA（Pseudo-Random Generation Algorithm）阶段
    i = j = 0
    for ch in data:
        i = (i + 1) % 256  # 更新i
        j = (j + S[i]) % 256  # 更新j
        S[i], S[j] = S[j], S[i]  # 交换S[i]和S[j]
        out.append(ch ^ S[(S[i] + S[j]) % 256])  # 对数据进行加密或解密操作（异或）
 
    return out
 
# RC4解密函数，解密过程与加密过程相同
def RC4decrypt(ciphertext, key):
    return RC4(ciphertext, key)
 
# 定义文件的起始偏移和大小
wrap_elf_start = 0x2e270
wrap_elf_size = 0xb8010
 
# 定义密钥
key = b"vUV4#\x91#SVt"
 
# 读取ELF文件
with open('com.oacia.apk_protect/assets/libjiagu_a64.so', 'rb') as f:
    wrap_elf = f.read()  # 读取文件内容到wrap_elf
 
# 对ELF文件中的密文部分进行RC4解密
dec_compress_elf = RC4decrypt(wrap_elf[wrap_elf_start:wrap_elf_start + wrap_elf_size], key)
 
# 对解密后的数据进行zlib解压
dec_elf = zlib.decompress(bytes(dec_compress_elf[4::]))  # 跳过前4个字节，可能是zlib标头
 
# 保存解压后的ELF文件
with open('wrap_elf', 'wb') as f:
    f.write(dec_elf)
 
# 定义一个数据段类part，用于保存ELF文件中的不同部分
class part:
    def __init__(self):
        self.name = ""  # 部分的名称
        self.value = b''  # 部分的字节数据
        self.offset = 0  # 偏移量
        self.size = 0  # 部分的大小
 
# 初始化索引和额外部分的列表
index = 1
extra_part = [part() for _ in range(7)]  # 创建7个part对象
 
# 定义ELF文件中的段名称
seg = ["phdr", ".rela.plt", ".rela.dyn", ".dynamic"]
 
# 取解密后的第一个字节作为XOR操作的密钥
v_xor = dec_elf[0]
 
# 遍历并提取ELF文件中的各个段
for i in range(4):
    # 读取每个段的大小（4个字节）
    size = int.from_bytes(dec_elf[index:index + 4], 'little')  # 小端格式读取大小
    index += 4
    extra_part[i + 1].name = seg[i]  # 设置段的名称
    # 对该段的数据进行XOR解密，使用v_xor对字节进行处理
    extra_part[i + 1].value = bytes(map(lambda x: x ^ v_xor, dec_elf[index:index + size]))
    extra_part[i + 1].size = size  # 设置段的大小
    index += size  # 跳过当前段的数据，指向下一个段
 
# 遍历extra_part中的每个部分，保存到文件
for p in extra_part:
    if p.value != b'':  # 如果该部分有数据（非空）
        # 根据段的名称和大小生成文件名
        filename = f"libjiagu.so_{hex(p.size)}_{p.name}"
        print(f"[{p.name}] get {filename}, size: {hex(p.size)}")
        # 将该段的内容保存为文件
        with open(filename, 'wb') as f:
            f.write(p.value)
```

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740476264433-a39975f0-32d4-4ee6-a0d3-c30b9c49286a.png)

这个我的起始偏移是 2e270，原作者偏移是 1e270，可能我的哪步有差错吧，按自己实际的来即可。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740476348147-0ea28d77-a175-439b-9b9a-df4e61d0409b.png)

主 ELF 导入导出表修复
-------------

在这之前可以了解一下自定义 linker 加固 so：[自实现 linker 加固 so](elink@b2dK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6%4N6%4N6Q4x3X3g2U0L8X3u0D9L8$3N6K6i4K6u0W2j5$3!0E0i4K6u0r3M7X3g2$3k6i4u0U0j5#2)9J5c8Y4m8Q4x3V1j5I4y4K6p5H3x3o6x3H3z5q4)9J5k6h3S2@1L8h3I4Q4x3U0y4Q4x3U0g2q4z5q4)9J5y4e0R3%4i4K6t1#2b7f1q4Q4x3U0g2q4y4g2)9J5y4f1q4q4i4K6t1#2z5f1q4Q4x3U0g2q4y4q4)9J5y4f1t1&6i4K6t1#2z5o6W2Q4x3U0g2q4y4W2)9J5y4e0V1$3i4K6t1#2z5o6N6Q4x3U0g2q4y4q4)9J5y4f1u0n7i4K6t1#2b7U0k6Q4x3U0g2q4y4W2)9J5y4f1p5H3i4K6t1#2b7V1y4Q4x3U0g2q4y4g2)9J5y4f1u0o6i4K6t1#2z5p5j5`.)

我们需要修复的就是之前 010 中看到加密数据后紧挨的 ELF，从文章中了解到这种做法一般会将原来`<font>phdr</font>`<font>,</font>`<font>.rela.plt</font>`<font>,</font>`<font>.rela.dyn</font>`<font>,</font>`<font>.dynamic</font>`<font> 段用无关字节覆盖，所以我们可以直接用 010 将数据覆盖回去。</font>

<font> 这里不删除段的原因是因为删除段的话会影响偏移，可能会导致各种奇怪问题。</font>

<font> 点击 struct_program_header_table 后直接 ctrl+shift+v 将 phdr 覆盖进去。</font>

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740482922433-0ed75384-2ca0-4899-924b-d4dd1263c7f2.png)

跳转到. dynamic 段的位置 0x16E3C0

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740483060660-c53b8b4a-93d6-45ec-adf4-5f1aef87e8db.png)

覆盖模式下 ctrl+shift+v 覆盖

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740483122260-14867f7d-a0e2-4724-ba84-3f97851922f1.png)

然后就是找到. rela.plt 和. rela.dyn

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740484065509-db233f2a-439f-49db-8585-9c1bd21aa3bf.png)

这里的 17192021 就是有关. rela.plt 和. rela.dyn 的偏移及大小。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740484131658-2388099c-38fb-4f7f-8c34-702311602f38.png)

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740484269761-08263d21-eeb0-413e-92d2-cd2b6b3885ed.png)

相应的覆盖回去即可。

可以正常分析

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740484670390-249faf7d-6504-44cb-a8a5-9f4bcf54c271.png)

这里将基地址设置为壳 ELF 的偏移 0xe7000，因为后面还有用 stalker_trace_so 去 HOOK。

加固壳反调试深入分析
----------

在继续往下深入 dex 的解密之前，还是先把前面遗留的问题先解决吧，之前过 frida 反调试的方式实际不能说是优雅，而且也会中断，分析反调试，实际非常需要。

这里也是靠着巨人的肩膀可以知道可行的方法，在此之前作者曾尝试过：

<font>DBus</font>: 某些程序会通过 DBus 向外部进程发送特定的信号，如果调试器正在运行，DBus 的通信机制可能会被修改或拦截。

<font>TracerPid:</font>`TracerPid` 是 `/proc/self/status` 文件中的一个字段，用于表示当前进程是否有调试器（如 GDB）附加。其值表示附加到当前进程的调试器的进程 ID（PID）。如果该值不为 0，表示当前进程被调试。

<font>readlink</font> :`readlink` 是一个用来读取符号链接的系统调用。某些程序通过 `readlink` 检查 `/proc/self/exe` 或其他符号链接文件的路径，来判断是否在调试环境中运行。例如，如果 `/proc/self/exe` 的路径指向调试器相关的文件（如 `gdb` 或 `ld-linux.so`），程序就能检测到调试器 。

<font>strstr</font> : 字符串搜索函数 , 调用 `strstr` 查找路径中是否包含 “gdb” 或 “frida" 等字符串。

而获得突破性进展的是 <font>pthread_create</font>，考虑 < font>phread_create</font > 的一个原因是如果把检测放在主线程的话，可能会造成阻塞卡顿现象，对于用户体验是不妥的，创建一个线程进行检测实际有考量。注入对该函数的 hook 代码

```
function check_pthread_create() {
    // 获取模块中名为 'pthread_create' 的导出函数地址
    var pthread_create_addr = Module.findExportByName(null, 'pthread_create');
  
    // 创建一个新的 NativeFunction 对象，用于调用 'pthread_create'，参数类型为 'pointer', 'pointer', 'pointer', 'pointer'，返回类型为 'int'
    var pthread_create = new NativeFunction(pthread_create_addr, "int", ["pointer", "pointer", "pointer", "pointer"]);
 
    // 替换原有的 'pthread_create' 函数，使用自定义的回调函数
    Interceptor.replace(pthread_create_addr, new NativeCallback(function (parg0, parg1, parg2, parg3) {
        // 获取新创建的线程的函数地址（parg2 指向的是线程的启动函数）
        var so_name = Process.findModuleByAddress(parg2).name;  // 获取共享库名称
        var so_path = Process.findModuleByAddress(parg2).path;  // 获取共享库的路径
        var so_base = Module.getBaseAddress(so_name);  // 获取共享库的基址
        var offset = parg2 - so_base;  // 计算线程函数相对于共享库基址的偏移量
        var PC = 0;  // 初始化程序计数器
 
        // 检查线程函数是否来自与“jiagu”相关的库（例如加固的库）
        if ((so_name.indexOf("jiagu") > -1)) {
            console.log("======")  // 输出日志标识
            console.log("find thread func offset", so_name, offset.toString(16));  // 输出线程函数的偏移地址（16进制）
 
            // 使用 Backtracer 来获取线程的调用栈
            Thread.backtrace(this.context, Backtracer.ACCURATE).map(addr_in_so);
  
            // 定义一个检查列表，包含了可能需要跳过的线程函数的偏移地址
            var check_list = []  // 这里你可以添加具体的偏移地址
            if (check_list.indexOf(offset) !== -1) {  // 如果偏移地址在检查列表中
                console.log("check bypass")  // 输出日志，表示需要跳过
            } else {
                // 否则，继续执行原有的 'pthread_create' 函数，创建线程
                PC = pthread_create(parg0, parg1, parg2, parg3);
            }
        } else {
            // 如果线程不是来自加固库，直接执行原始的 'pthread_create' 调用
            PC = pthread_create(parg0, parg1, parg2, parg3);
        }
 
        // 返回线程创建函数的结果（原始的返回值）
        return PC;
    }, "int", ["pointer", "pointer", "pointer", "pointer"]))  // 自定义回调函数的参数类型和返回类型
}
```

输出所有在 libjiagu 中调用的线程函数，而被调用地址却是一致的 0x17710

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740568824556-de1646a8-ab37-4150-9327-246dd3baa3fe.png)

跳转过去，等等，这里似乎来过？我的 ida 居然有记录，是之前分析调用 RC4 函数的调用者时来过的，![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740569413666-984da773-41bd-46d8-83a5-fa4dbc13cc1d.png)

没想到兜兜转转又回到这里了，我们在前面已经分析过这里的调用方式了，是 ffi_call 进行的动态调用。[libffi](elink@ef6K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6S2L8r3g2^5j5$3!0V1k6e0u0Q4x3X3g2Y4K9i4c8T1L8$3!0C8i4K6u0W2K9h3!0Q4x3V1k6A6L8%4y4Q4x3X3c8V1k6i4k6W2L8r3!0H3L8h3g2F1N6q4)9J5k6r3N6#2K9h3c8W2L8r3W2F1k6i4y4Q4x3V1k6A6L8%4y4Q4x3X3c8*7j5g2)9J5k6s2c8S2L8W2)9J5c8X3I4A6j5X3k6X3K9g2)9J5k6r3c8G2L8X3N6Q4x3X3c8@1j5h3W2Q4x3X3c8@1K9h3q4G2i4K6u0V1P5h3!0F1k6#2)9J5k6r3S2W2i4K6u0V1k6r3W2F1k6#2)9J5k6s2W2A6i4K6u0V1j5#2)9J5k6r3S2S2L8W2)9J5k6s2y4Z5N6b7`.`.)

GPT 的解释：

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740569993003-f9bfb9bc-07bb-44fb-8da8-21ffe9df7e01.png)

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740569658032-0e64e093-581f-47be-a6b5-01e097797c44.png)

可以 hook 一下看看有没有敏感字符串。

```
function anti_frida_check(){
    var module = Process.findModuleByName("libjiagu_64.so");
    Interceptor.attach(module.base.add(0x1770C), {
        onEnter: function (args) {
            try{
                console.log(this.context.x0.readCString())
            }
            catch (e){
                 //不是string时通过try处理掉
            }
        },
        onLeave: function (ret) {
        }
    });
  
}
```

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740570201146-77eb4f1e-e9b8-4ceb-8cad-5fe82467dbbf.png)

实际其中有很多字符串，可以想到这里到底调用了多少函数，筛选敏感字符串。

```
function anti_frida_check(){
    var module = Process.findModuleByName("libjiagu_64.so");
    Interceptor.attach(module.base.add(0x1770C), {
        onEnter: function (args) {
            try{
                var s = this.context.x0.readCString();
                if (s.indexOf('frida')!==-1 ||
                    s.indexOf('gum-js-loop')!==-1 ||
                    s.indexOf('gmain')!==-1 ||
                    s.indexOf('linjector')!==-1 ||
                    s.indexOf('/proc/')!==-1){
                    console.log(s)
                }
            }
            catch (e){
  
            }
        },
        onLeave: function (ret) {
        }
    });
  
}
```

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740570527242-c7524de2-fa01-41e9-be23-bc885e1472d8.png)

第一次 hook 的时候没有下面这一段，发现手机端开的是 huluda，应该是把 so 的特征去除了，但是还是被检测中断了。

更详细的打出了地址

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740574484494-a9386bbc-1b82-4278-b0e7-433252deba9d.png)

这个位置已经是主 ELF 的部分了

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740574538208-b3b741c1-bb7a-4d79-9580-bd0ff35b5168.png)

而 0x31444ec70 似乎已经跳到其他 so 去了。

而且还有一件很有意思的事就是在 sub_217444 中调用的函数有 openat 的系统调用。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740617973838-af8c8d94-7555-4fee-b981-f0818b51322b.png)

并且周围还有很多其他的关于系统调用的片段

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740624504544-f0402b58-5bc4-492c-956b-3533c57f4e89.png)

查看下系统调用 [https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md](elink@a13K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6U0K9s2u0G2L8h3W2#2L8g2)9J5k6h3N6G2L8$3N6D9k6i4y4G2N6i4u0U0k6g2)9J5k6h3y4G2L8g2)9J5c8X3y4Z5M7X3!0E0K9i4g2E0L8%4y4Q4x3V1k6V1L8$3y4K6i4K6u0r3i4K6u0n7i4K6u0r3L8h3q4K6N6r3g2J5i4K6u0r3j5$3!0F1M7%4c8S2L8Y4c8K6i4K6u0r3M7%4W2K6j5$3q4D9L8s2y4Q4x3X3g2E0k6l9`.`.)

56，63，129，167 对应的分别是 openat，read，kill，<font>prctl</font>

然后又看到一篇文章，打算学习一下 HOOK 这种系统调用。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740618070219-90955341-3730-4d2d-9b15-5e9f02000701.png)

找了一个 github 项目：[https://github.com/LLeavesG/Frida-Sigaction-Seccomp?tab=readme-ov-file](elink@cb6K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6x3e0r3g2S2N6X3g2K6c8#2)9J5c8V1k6J5K9h3c8S2i4K6u0V1f1$3W2Y4j5h3y4@1K9h3!0F1i4K6u0V1f1$3g2U0j5$3!0E0M7q4)9K6c8Y4c8S2j5W2)9K6c8s2u0W2j5h3c8E0k6g2)9J5k6r3!0$3i4K6u0V1k6X3W2D9k6b7`.`.)

注入后在 adb shell 中 logcat | grep Openat 筛选出日志。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740624138084-3fde1317-3de2-4209-9491-e27e6aab970d.png)

还是能看到一些东西的，稍微修改下脚本我们再 HOOK 下 read 系统调用，哦，原来是这样，它通过 svc 调用 read 来读取了 maps 中的 frida 的特征数据，那么读取了之后一定就是比较数据了，在主 ELF 中有发现 frida 的字符串，但是实际上只是过掉反调试的话，只要将这些特征数据替换掉就好了。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740626869903-8914cc39-5aff-415c-a730-83a4ef047cae.png)

这里也说明了为什么我之前简单的 hook openat 失败的原因，因为他直接使用 svc 来调用的，而不是通过 libc.so 封装的函数。

这里先用作者的方法过掉检测

将所有有关检测的字符串都替换为无意义的字符。

```
function anti_frida_check(){
    var module = Process.findModuleByName("libjiagu_64.so");
    Interceptor.attach(module.base.add(0x1770C), {
        onEnter: function (args) {
            try{
                var s = this.context.x6.readCString();
                if (s.indexOf('frida')!==-1 ||
                    s.indexOf('gum-js-loop')!==-1 ||
                    s.indexOf('gmain')!==-1 ||
                    s.indexOf('linjector')!==-1 ||
                    s.indexOf('/proc/')!==-1){
                    //console.log(s)
                    Memory.protect(this.context.x0, Process.pointerSize, 'rwx');
                    var replace_str=""
                    for(var i=0;i
```

然后输出次数，可以看到，每隔几秒便会输出大约 2000 个字符串，有趣，应该是创建了一个线程周期性检测 <font>/memfd:frida-agent-64.so</font > 字符串。

所以打算接续前面 hook pthread 看能不能将关键线程取消创建，直到停止前有关的几个线程函数地址。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740635870085-80236656-b5e1-4a27-96ba-f42a4360a828.png)

但是我将这些地址挨个跳过之后还是会中断，猜测是多个反调试的原因，所以只是取消该线程创建无法成功或者没有取消掉检测线程。

正好 frida 的反调试也没有系统学习过，趁这个机会正好一并解决。

我们先使用 huluda 尽量隐藏特征，然后慢慢来，我之前在 hook 某个函数时有看到过 cmdline 的字样，并且在阅读其他文章时也发现有该反调试，所以我先试着去搜索，然后发现可以直接在主 ELF 中找到。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740799106091-d5571daa-c835-4845-b8dd-909cd7d9d312.png)

读取了 cmdline 然后传入了 sub_14495c，然后看下 sub_14495c

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740799161093-48ea595c-778c-4075-9b3e-fea3de82b49c.png)

open 还有 gets，感觉不太需要看了，猜一下大概也猜到了，了解了一下 cmdline，他这里查看父进程是否包含 zygote 来检测是否是调试器，那这个点应该是检测 gdb 等用的。

我们再看看之前多次出现的 status，这个反调试是通过检测其中的 TracePid 来实现的，我们搜索一下该字符串

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740800658524-d4cc0918-fb19-439f-9ea7-27350feecf14.png)

查看了下没有找到很关键的代码，那看来真正关键的 status 字符串使用时应该是加密的状态, 同时也进行了多个函数的 hook，但是并没有相关字符，很明显大概调用时用的还是 SVC，但是只有调用打出没有调用地址，但是为什么 strstr 这种函数也无法 HOOK 到呢？libc.so 明明早就加载了。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740808900251-1fa3ca42-9801-4d55-a5d5-04d78eca7b74.png)

但是我很确定这里 strstr 是绝对被使用了，而主 ELF 中这些函数都是使用跳转表来调用的，所以我本打算通过 hook 该地址来查看寄存器的值，但是提示没有足够空间。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740808992336-076ae8a3-86e8-4906-acab-e6bd9768356f.png)

选择查看 strtol 的调用，因为这个函数可能被用于 status 的反调试中，结果发现一个有趣的函数 sub_14453C，这里有一些检测

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740814098302-36b0757e-382f-4210-ade8-09343afe6399.png)

很有意思，然后查看了这个函数被调用的位置

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740813726877-741e41b9-845c-492b-9269-1822f93ca63e.png)

是个 pthread_create, 非常符合预期, 但是我 hook 这个函数却发现他似乎没有被调用，那么有两种可能，一种是我们在进入这里之前就已经被其他反调试干掉，另一种就是这里是迷惑我们的压根就没有调用，但是我觉得第二种不太可能。HOOK open 等函数都无法找到除了 maps 和反调试相关的字样的话，那可能是在那些反调试之前，我们已经被干掉了？我们有两个可以 hook 到的反调试，一个是 phread，一个是 maps，那么先把这两个拦截，再继续 hook 吧

我们从头开始，我们先 hook read 函数，如果他打算使用 status 或者 maps 的话，那么大概率会使用 read 来读取内容。

很明显读取了 maps![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740824330789-2f9f9db2-5bfd-438d-9fe4-6ec68ac61669.png)

然后我不太清楚为什么他会去读取 cat 的 status。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740824366278-cc7f5fc5-1e91-42db-bef3-2b520a8ac050.png)

无所谓，既然他会读取 maps，那我们先过掉这个，那怎么做呢？用 frida 去 hookread 来替换其中有关的数据吗？太麻烦了，直接用 huluda 得了。

从这里往后的话我尝试很多，比如从函数打印链的最后向前追溯，又或者 frida 盲勾重要的系统函数来查看参数，但是这样子无异于大海捞针，最后还是选择了老老实实用 IDA 附加调试。

**下面的是在慢慢淘宝，并无突出收获**：

我将这一片都识别为了代码，事实是有相当多可疑有趣的函数。例如这个：

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740644876714-74d5fb33-e925-40b6-a9d4-d0ca7684906d.png)

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740643603610-f9b59fd7-b0c5-4d18-a825-134f7a0ea011.png)

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740704665552-e55422aa-c7c9-4e12-ba79-defb08a815b9.png)

这个壳中大量用到了这种跳转段

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740643461351-d69a9307-447b-44ea-8aaa-5d8a7a0088ea.png)

跳转过去后是一些无法反编译的数据，在动调的时候发现是系统函数的地址。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740643527918-00bd7bea-2df2-45fb-bdfd-297b00c3c608.png)

### IDA 动调过程

**手机终端**：

启动 IDAserver

```
./ida64 -p 1234
```

调试状态启动 app，要事先对 app 的 xml 文件进行修改添加调试属性。

```
am start -D -n com.oacia.apk_protect/.MainActivity
```

**电脑端**：

转发端口

```
adb forward tcp:1234 tcp:1234
```

此时 IDA 进行附加并记下进程的 PID

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1741086726820-edc9d322-dbb9-4149-86aa-6a6c2c6a410c.png)

jdb 通知程序继续运行

```
adb forward tcp:8700 jdwp:pid
jdb -connect com.sun.jdi.SocketAttach:hostname=127.0.0.1,port=8700
```

根据我多次的调试，唔，准确来说是两三天的重复调试中，稍微找到些技巧。

首先是之前提到过的动态调用 ffi，在调试中我发现不管是系统函数还是 so 中的函数都有走这里调用，大胆猜测的话，壳几乎所有的函数都是走的这里，此时的 X24 就是要调用的函数地址。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1741086989127-db8f4991-2b93-4013-93cf-a4dbc53c340d.png)

所以在这里下断点

之后在 module list 中找到 libc.so，双击然后再找到一些关键的系统函数进行断点，例如 open，strstr，strtol 等。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1741088059182-05e148e1-b994-49e9-97d2-c724bacf6e06.png)

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1741088070296-b16676eb-4577-44f6-a56c-b764f6dfbf6c.png)

也是动调的时候发现很多系统函数的调用很奇怪。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740921170483-d14e7f5c-661b-4379-bc02-a45b249b0c47.png)

函数的参数对应不上，这里 strtol 跳转后的地方是 calloc 函数。而 open 函数跳过去后是 memset。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1741053575802-77ab831f-feb6-46e1-be42-66a938f29177.png?x-oss-process=image%2Fformat%2Cwebp)、

这张图更为直观，似乎 IDA 分析的跳转段指向的系统函数与实际的错了一个位。

并且在调试的过程中发现他结束程序用的是 kill 系统调用，但是我对 kill 被调用的点下断点时 F9 无法断住并且退出。

所以没有办法直接在 kill 的调用处下断点来快速分析反调试。

#### 第一个反调试：TracerPid：14091

这里要注意寄存器的情况

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1741088125213-50f7850f-99c6-4911-9963-9c8d462e32e9.png)

前面的 B4 似乎是壳程序特意加的，以此来干扰分析，我们只要取后面的有效地址来跳转分析即可。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740989711796-86274b5d-57c1-4fb6-a400-1efa3bf1f314.png)

F2 进行编辑，然后 F2 保存，修改为 0 过掉。这里有个很奇怪的点就是我用 frida HOOK 输出 strstr 的第一个参数发现并没有 TracerPid，所以只能通过 hook fgets 来修改 TracerPid 的值。

#### 第二个反调试：fake-libs

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1741088228120-a8bd4cfc-de55-42ed-a453-7ca9c27336b9.png)

这个反调试很奇怪，因为是在 strstr 的参数中发现的，这里的 fake-libs 就是他查找的字符串，没有找到哪篇文章有提及和这个字符串有关的反调试，但是在我将其全部替换为 0 后再继续跟进就可以进入主 ELF，而之前都会在那之前就中断。

然后程序对 maps 进行了很长一段的循环查找 dev object 等字符串，然后接着调试。

甚至调到了主 ELF 解压的这里了，之后后面又被 kill 了

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1741079579707-36db0fa5-a302-480b-b35d-b5ff38760211.png)

调试到 sub_142EA0 函数之后开始有一些 java 类型的 memcpy 和 memcmp，不太清楚在干什么，在解析什么吗

不过跳到主 ELF 后就没再遇到壳 ELF 那种 VM 式的调用方式了。

这里又被反调试了，根据分析的话大致猜测是通过 raise 来中断的, 因为对 kill 的断点我没有断住。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1741135526878-5e526f3b-263c-406c-8b1f-dc3f75196005.png)

所有 raise 被用到的地方是发送中断指令。

因为跟着动调需要花费不少时间，所以先分析到这里了。360 免费壳的分析花费时间有点超过预期了，只能先搁置，之后有时间会再次进行更细致的分析的。

主 dex 解密流程分析
------------

这里的话作者因为有未加固的 apk 所以对比了大小猜测是在 classes.dex 中，而根据我之前学习的一二代加壳的话，很多时候加密也喜欢放在加固后的 dex 的尾部，并附带上大小。

搜索前几个字节直接定位到尾部。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740486443085-639bc8b0-4782-4264-877d-8464fbe38d93.png)

接下来再次用 stalker_trace_so 跟踪函数调用链，在主 ELF 中使用插件生成 js 脚本，然后将壳 elf 的函数地址及名称 copy 过来并合并，再对输出信息的代码进行微调即可。

```
var func_addr_main = [0x2e130, 0x2e150, 0x2e160, 0x2e170, 0x2e180, 0x2e190, 0x2e1a0, 0x2e1b0, 0x2e1c0, 0x2e1d0, 0x2e1e0, 0x2e1f0, 0x2e200, 0x2e210, 0x2e220, 0x2e230, 0x2e240, 0x2e250, 0x2e260, 0x2e270, 0x2e280, 0x2e290, 0x2e2a0, 0x2e2b0, 0x2e2c0, 0x2e2d0, 0x2e2e0, 0x2e2f0, 0x2e300, 0x2e310, 0x2e320, 0x2e330, 0x2e340, 0x2e350, 0x2e360, 0x2e370, 0x2e380, 0x2e390, 0x2e3a0, 0x2e3b0, 0x2e3c0, 0x2e3d0, 0x2e3e0, 0x2e3f0, 0x2e400, 0x2e410, 0x2e420, 0x2e430, 0x2e440, 0x2e450, 0x2e460, 0x2e470, 0x2e480, 0x2e490, 0x2e4a0, 0x2e4b0, 0x2e4c0, 0x2e4d0, 0x2e4e0, 0x2e4f0, 0x2e500, 0x2e510, 0x2e520, 0x2e530, 0x2e540, 0x2e550, 0x2e560, 0x2e570, 0x2e580, 0x2e590, 0x2e5a0, 0x2e5b0, 0x2e5c0, 0x2e5d0, 0x2e5e0, 0x2e5f0, 0x2e600, 0x2e610, 0x2e620, 0x2e630, 0x2e640, 0x2e650, 0x2e660, 0x2e670, 0x2e680, 0x2e690, 0x2e6a0, 0x2e6b0, 0x2e6c0, 0x2e6d0, 0x2e6e0, 0x2e6f0, 0x2e700, 0x2e710, 0x2e720, 0x2e730, 0x2e740, 0x2e750, 0x2e760, 0x2e770, 0x2e780, 0x2e790, 0x2e7a0, 0x2e7b0, 0x2e7c0, 0x2e7d0, 0x2e7e0, 0x2e7f0, 0x2e800, 0x2e810, 0x2e820, 0x2e830, 0x2e840, 0x2e850, 0x2e860, 0x2e870, 0x2e880, 0x2e890, 0x2e8a0, 0x2e8b0, 0x2e8c0, 0x2e8d0, 0x2e8e0, 0x2e8f0, 0x2e900, 0x2e910, 0x2e920, 0x2e930, 0x2e940, 0x2e950, 0x2e960, 0x2e970, 0x2e980, 0x2e990, 0x2e9a0, 0x2e9b0, 0x2e9c0, 0x2e9d0, 0x2e9e0, 0x2e9f0, 0x2ea00, 0x2ea10, 0x2ea20, 0x2ea30, 0x2ea40, 0x2ea50, 0x2ea60, 0x2ea70, 0x2ea80, 0x2ea90, 0x2eaa0, 0x2eab0, 0x2eac0, 0x2ead0, 0x2eae0, 0x2eaf0, 0x2eb00, 0x2eb10, 0x2eb20, 0x2eb30, 0x2eb40, 0x2eb50, 0x2eb60, 0x2eb70, 0x2eb80, 0x2eb90, 0x2eba0, 0x2ebb0, 0x2ebc0, 0x2ebd0, 0x2ebe0, 0x2ebf0, 0x2ec00, 0x2ec10, 0x2ec20, 0x2ec30, 0x2ec40, 0x2ec50, 0x2ec60, 0x2ec70, 0x2ec80, 0x2ec90, 0x2eca0, 0x2ecb0, 0x2ecc0, 0x2ecd0, 0x2ece0, 0x2ecf0, 0x2ed00, 0x2ed10, 0x2ed20, 0x2ed30, 0x2ed40, 0x2ed50, 0x2ed60, 0x2ed70, 0x2ed80, 0x2ed90, 0x2eda0, 0x2edb0, 0x2edc0, 0x2edd0, 0x2ede0, 0x2edf0, 0x2ee00, 0x2ee10, 0x2ee20, 0x2ee30, 0x2ee40, 0x2ee50, 0x2ee60, 0x2ee70, 0x2ee80, 0x2ee90, 0x2eea0, 0x2eeb0, 0x2eec0, 0x2eed0, 0x2eee0, 0x2eef0, 0x2ef00, 0x2ef10, 0x2ef20, 0x2ef30, 0x2ef40, 0x2ef50, 0x2ef60, 0x2ef70, 0x2ef80, 0x2ef90, 0x2efa0, 0x2efb0, 0x2efc0, 0x2efd0, 0x2efe0, 0x2eff0, 0x2f000, 0x2f010, 0x2f020, 0x2f030, 0x2f03c, 0x2f0b4, 0x2f0c4, 0x2f15c, 0x2f1fc, 0x2f4ac, 0x2f4d8, 0x2f528, 0x2f5c8, 0x2f6fc, 0x2f728, 0x2f750, 0x2f830, 0x2f8f0, 0x2fa80, 0x2fad8, 0x2faf4, 0x2fb10, 0x2fb2c, 0x2fb84, 0x2fba0, 0x308e8, 0x308fc, 0x30b68, 0x30be0, 0x30ca0, 0x30e90, 0x310cc, 0x312d8, 0x3159c, 0x316b8, 0x31748, 0x31ae4, 0x31c8c, 0x31d24, 0x31fb4, 0x32270, 0x3259c, 0x32634, 0x34c1c, 0x34d78, 0x34d9c, 0x34da8, 0x34e58, 0x351ac, 0x351b4, 0x35210, 0x35268, 0x35300, 0x35394, 0x35aa4, 0x35c18, 0x35f8c, 0x36134, 0x364c4, 0x36694, 0x36738, 0x3673c, 0x370d8, 0x37834, 0x38130, 0x381c8, 0x38260, 0x383a4, 0x3856c, 0x38604, 0x3869c, 0x38768, 0x38884, 0x38a18, 0x38b14, 0x390c8, 0x39194, 0x392d4, 0x3936c, 0x393c8, 0x39404, 0x39440, 0x3947c, 0x395a0, 0x39664, 0x39674, 0x39678, 0x39c30, 0x3ab78, 0x3abe0, 0x3ac40, 0x3acd8, 0x3b180, 0x3b2d0, 0x3b2e8, 0x3bae0, 0x3bb54, 0x3bb5c, 0x3bc7c, 0x3bd14, 0x3c324, 0x3c6b8, 0x3c720, 0x3c970, 0x3cbc0, 0x3cce8, 0x3d510, 0x3d710, 0x3d900, 0x3dc68, 0x3df7c, 0x3dfa0, 0x3e3b8, 0x3e650, 0x3e854, 0x3ed2c, 0x3edc4, 0x3f888, 0x3f920, 0x3fed8, 0x3ff84, 0x4043c, 0x40dcc, 0x40ea8, 0x411e4, 0x411ec, 0x418dc, 0x41a2c, 0x41d44, 0x41d64, 0x41d70, 0x41db8, 0x41e60, 0x41e88, 0x41f5c, 0x42468, 0x42758, 0x42a04, 0x43568, 0x43c10, 0x43d6c, 0x43eb0, 0x449fc, 0x44b18, 0x44b80, 0x44f4c, 0x4508c, 0x455d4, 0x45c6c, 0x462f8, 0x46470, 0x46478, 0x46480, 0x4648c, 0x464a4, 0x464ac, 0x464d8, 0x46518, 0x46560, 0x4684c, 0x469fc, 0x46b60, 0x46fe4, 0x489f8, 0x48a34, 0x48a6c, 0x48b00, 0x48b18, 0x4a7a4, 0x4a958, 0x4ad58, 0x4ae20, 0x4b088, 0x4b518, 0x4bda8, 0x4c2b8, 0x4fd68, 0x4fdb0, 0x4fff8, 0x50164, 0x50408, 0x50630, 0x50820, 0x50978, 0x525a8, 0x525c4, 0x525ec, 0x52604, 0x52628, 0x52664, 0x52678, 0x52698, 0x526a0, 0x526b4, 0x526d0, 0x526fc, 0x5270c, 0x52728, 0x52770, 0x52824, 0x52828, 0x5282c, 0x528f0, 0x529d8, 0x52adc, 0x52bac, 0x52bb8, 0x52ce4, 0x52cfc, 0x52d18, 0x52d40, 0x52d4c, 0x52d50, 0x52d8c, 0x52dac, 0x52df4, 0x52e0c, 0x52e40, 0x52e7c, 0x534d4, 0x534e4, 0x534fc, 0x5353c, 0x5355c, 0x53588, 0x535a8, 0x535d4, 0x535e4, 0x53600, 0x5361c, 0x53664, 0x53670, 0x53684, 0x536b4, 0x536d0, 0x5379c, 0x537ac, 0x537e0, 0x537e8, 0x53808, 0x53810, 0x5382c, 0x53870, 0x53894, 0x538ac, 0x538dc, 0x53920, 0x5395c, 0x539a4, 0x539b4, 0x539d0, 0x53a00, 0x53a0c, 0x53a50, 0x53a74, 0x53b40, 0x541cc, 0x541d8, 0x54c98, 0x54cb0, 0x54d80, 0x54e14, 0x54e2c, 0x54e4c, 0x54e64, 0x54e80, 0x54ea0, 0x54eb8, 0x54f30, 0x54f50, 0x54f70, 0x54f90, 0x54fb0, 0x54fd0, 0x54ff0, 0x55064, 0x5507c, 0x5509c, 0x5512c, 0x5513c, 0x551e4, 0x55238, 0x552a8, 0x552f0, 0x5531c, 0x55384, 0x553cc, 0x554ec, 0x554f0, 0x55544, 0x56114, 0x56288, 0x562ec, 0x5630c, 0x56310, 0x56328, 0x56344, 0x5638c, 0x56394, 0x5639c, 0x563a4, 0x56460, 0x56480, 0x56484, 0x5649c, 0x564a4, 0x564ac, 0x564c8, 0x56538, 0x565d8, 0x565e4, 0x5670c, 0x56b7c, 0x56ea8, 0x56f34, 0x570ec, 0x570f8, 0x57510, 0x57654, 0x57784, 0x579ac, 0x57ae0, 0x57c30, 0x57e08, 0x57f2c, 0x58888, 0x58920, 0x5927c, 0x59a3c, 0x5a1fc, 0x5a2c0, 0x5a5f0, 0x5a924, 0x5b7e8, 0x5b83c, 0x5b85c, 0x5b894, 0x5b8bc, 0x5b92c, 0x5b954, 0x5b9b8, 0x5b9c0, 0x5b9cc, 0x5ba10, 0x5baa4, 0x5babc, 0x5bea0, 0x5bfe0, 0x5c008, 0x5c088, 0x5c16c, 0x5c184, 0x5c568, 0x5c848, 0x5c964, 0x5c9fc, 0x5ca38, 0x5cb48, 0x5cc54, 0x5cd24, 0x5cd3c, 0x5cebc, 0x5d0cc, 0x5d23c, 0x5d360, 0x5d4b8, 0x5d508, 0x5d53c, 0x5d774, 0x5d95c, 0x5dafc, 0x5def0, 0x5df4c, 0x5e00c, 0x5e018, 0x5e024, 0x5e030, 0x5e03c, 0x5e040, 0x5e078, 0x5e0a4, 0x5e148, 0x5e154, 0x5e16c, 0x5e184, 0x5e19c, 0x5e1b4, 0x5e1cc, 0x5e1e4, 0x5e1fc, 0x5e214, 0x5e228, 0x5e268, 0x5e280, 0x5e298, 0x5e2b0, 0x5e2c8, 0x5e2e0, 0x5e2f8, 0x5e310, 0x63a44, 0x63a50, 0x63a88, 0x63aac, 0x63acc, 0x63af0, 0x63b14, 0x63b34, 0x63b68, 0x63b78, 0x63bb0, 0x63bc0, 0x63bcc, 0x63bf4, 0x63c1c, 0x63c28, 0x63c60, 0x63c90, 0x63d28, 0x63db8, 0x63e70, 0x63e7c, 0x63e88, 0x63ec4, 0x63eec, 0x6401c, 0x64028, 0x64054, 0x64090, 0x6409c, 0x640bc, 0x64104, 0x64164, 0x64198, 0x641bc, 0x641dc, 0x64204, 0x64240, 0x64250, 0x64278, 0x642a8, 0x642e0, 0x64304, 0x64324, 0x6436c, 0x643cc, 0x64408, 0x64430, 0x6446c, 0x644a8, 0x644b4, 0x644ec, 0x644fc, 0x6457c, 0x645c4, 0x645d0, 0x64638, 0x6465c, 0x6467c, 0x646f4, 0x64740, 0x647a8, 0x647d8, 0x64808, 0x64814, 0x64820, 0x6482c, 0x64854, 0x64868, 0x648d8, 0x64908, 0x64920, 0x64998, 0x649d0, 0x649e0, 0x64a00, 0x64a0c, 0x64a58, 0x64a98, 0x64ae0, 0x64b40, 0x64b64, 0x64b84, 0x64b90, 0x64c44, 0x64c78, 0x64d10, 0x64d1c, 0x64d28, 0x64d4c, 0x64d68, 0x64d6c, 0x64d90, 0x64da0, 0x64f9c, 0x6518c, 0x65198, 0x651a8, 0x651f4, 0x65200, 0x6524c, 0x6525c, 0x652a8, 0x652b4, 0x654b0, 0x6575c, 0x657c8, 0x65848, 0x6584c, 0x65898, 0x65960, 0x65978, 0x65984, 0x65990, 0x659dc, 0x659e8, 0x65a30, 0x65b2c, 0x65d80, 0x65dcc, 0x65df4, 0x65e04, 0x65e50, 0x65e5c, 0x65e68, 0x65f64, 0x6617c, 0x661e8, 0x66200, 0x66214, 0x66234, 0x6626c, 0x6627c, 0x662c8, 0x66390, 0x663bc, 0x663e4, 0x663f8, 0x66418, 0x66440, 0x66458, 0x66474, 0x66570, 0x667e8, 0x66810, 0x66820, 0x6686c, 0x6687c, 0x668a0, 0x668b0, 0x668f8, 0x669f4, 0x66a04, 0x66a08, 0x66a54, 0x66a60, 0x66c5c, 0x66dc4, 0x66e34, 0x66ea8, 0x66eb8, 0x66edc, 0x66eec, 0x66f3c, 0x66fac, 0x67000, 0x67004, 0x67008, 0x67030, 0x670fc, 0x67114, 0x67128, 0x67148, 0x67158, 0x6717c, 0x6718c, 0x671b0, 0x671bc, 0x67208, 0x67214, 0x67238, 0x67258, 0x67280, 0x6734c, 0x67364, 0x67460, 0x67470, 0x67494, 0x674a4, 0x674c8, 0x674d0, 0x674dc, 0x675dc, 0x67718, 0x67728, 0x6774c, 0x6775c, 0x67780, 0x67854, 0x678ac, 0x678d4, 0x678ec, 0x678f8, 0x67904, 0x6794c, 0x67958, 0x67a54, 0x67c90, 0x67c9c, 0x67cc0, 0x67cd0, 0x67ecc, 0x67ed8, 0x67fd4, 0x68228, 0x68234, 0x68258, 0x68268, 0x68274, 0x68280, 0x682a4, 0x682ac, 0x682b8, 0x68304, 0x683d4, 0x68448, 0x68458, 0x6847c, 0x68480, 0x68484, 0x684a8, 0x686a4, 0x687d4, 0x68840, 0x6886c, 0x68894, 0x688a8, 0x688c8, 0x688d8, 0x688fc, 0x6890c, 0x68918, 0x68964, 0x68a2c, 0x68a44, 0x68a58, 0x68a78, 0x68a88, 0x68aac, 0x68ab4, 0x68ad4, 0x68ad8, 0x68b24, 0x68bec, 0x68c28, 0x68c3c, 0x68c5c, 0x68c84, 0x68c88, 0x68c8c, 0x68cb0, 0x68cfc, 0x68d64, 0x68d90, 0x68db8, 0x68dcc, 0x68dec, 0x68e3c, 0x68e60, 0x68e70, 0x68e7c, 0x68e88, 0x68ed4, 0x68edc, 0x68ef4, 0x68f18, 0x68f28, 0x68f4c, 0x68f50, 0x68f54, 0x68f78, 0x68f9c, 0x69008, 0x6906c, 0x69094, 0x690ac, 0x690b8, 0x690c4, 0x690e8, 0x691b4, 0x691cc, 0x691e0, 0x69200, 0x69250, 0x6925c, 0x692ac, 0x692b8, 0x692e4, 0x692ec, 0x692f0, 0x69340, 0x69580, 0x6958c, 0x69808, 0x69814, 0x69930, 0x69940, 0x6994c, 0x69a6c, 0x69b94, 0x69c64, 0x69ca0, 0x69cb4, 0x69cd8, 0x69ce8, 0x69d0c, 0x69d38, 0x69d44, 0x69f84, 0x6a0dc, 0x6a0ec, 0x6a0fc, 0x6a338, 0x6a488, 0x6a4f4, 0x6a570, 0x6a598, 0x6a5a8, 0x6a6c8, 0x6a968, 0x6a9d4, 0x6a9fc, 0x6aa54, 0x6aa64, 0x6aca0, 0x6af10, 0x6af5c, 0x6afc4, 0x6afec, 0x6b10c, 0x6b24c, 0x6b2b8, 0x6b2e4, 0x6b30c, 0x6b320, 0x6b340, 0x6b368, 0x6b378, 0x6b3e4, 0x6b40c, 0x6b44c, 0x6b450, 0x6b474, 0x6b480, 0x6b48c, 0x6b49c, 0x6b4a8, 0x6b574, 0x6b59c, 0x6b5c4, 0x6b5e8, 0x6b5f8, 0x6b61c, 0x6b62c, 0x6b648, 0x6b658, 0x6b6c0, 0x6b6e8, 0x6b7b4, 0x6b7dc, 0x6b7f8, 0x6b84c, 0x6b858, 0x6b860, 0x6b880, 0x6b890, 0x6b8dc, 0x6b8e4, 0x6b904, 0x6b92c, 0x6b93c, 0x6b960, 0x6b97c, 0x6b9a0, 0x6b9a4, 0x6b9a8, 0x6b9cc, 0x6b9e8, 0x6ba2c, 0x6ba34, 0x6ba58, 0x6ba78, 0x6ba90, 0x6bb0c, 0x6bb34, 0x6bb44, 0x6bb68, 0x6bb80, 0x6bbbc, 0x6bbd0, 0x6bbf0, 0x6bbf4, 0x6bbf8, 0x6bc1c, 0x6bc20, 0x6bc5c, 0x6bc98, 0x6bcac, 0x6bccc, 0x6bce4, 0x6bd00, 0x6bd10, 0x6bd34, 0x6bd40, 0x6bd5c, 0x6be2c, 0x6be84, 0x6beac, 0x6bee0, 0x6c014, 0x6c03c, 0x6c058, 0x6c074, 0x6c0dc, 0x6c0f4, 0x6c0fc, 0x6c11c, 0x6c12c, 0x6c150, 0x6c160, 0x6c1a0, 0x6c1c4, 0x6c1cc, 0x6c218, 0x6c280, 0x6c298, 0x6c2ac, 0x6c2cc, 0x6c2d0, 0x6c2d4, 0x6c2f8, 0x6c308, 0x6c354, 0x6c370, 0x6c398, 0x6c3bc, 0x6c4a8, 0x6c4dc, 0x6c5b0, 0x6c608, 0x6c630, 0x6c634, 0x6c664, 0x6c680, 0x6c68c, 0x6c6ac, 0x6c720, 0x6c730, 0x6c754, 0x6c764, 0x6c784, 0x6c790, 0x6c7a8, 0x6c7c0, 0x6c7e8, 0x6c804, 0x6c840, 0x6c868, 0x6c8ac, 0x6c8cc, 0x6c8f0, 0x6c9bc, 0x6c9d4, 0x6c9e8, 0x6ca08, 0x6ca0c, 0x6ca10, 0x6ca34, 0x6ca44, 0x6ca68, 0x6ca78, 0x6cab8, 0x6cadc, 0x6caf4, 0x6cafc, 0x6cb1c, 0x6cb54, 0x6cb64, 0x6cba8, 0x6cdc0, 0x6cdcc, 0x6ce18, 0x6ce44, 0x6ce74, 0x6cea0, 0x6cec4, 0x6cef0, 0x6cf00, 0x6cf44, 0x6cf60, 0x6cf68, 0x6cf6c, 0x6cf84, 0x6cf9c, 0x6cfa4, 0x6cfb4, 0x6cfc8, 0x6cfd4, 0x6cfe8, 0x6d00c, 0x6d040, 0x6d058, 0x6d088, 0x6d09c, 0x6d0a8, 0x6d0b4, 0x6d0d0, 0x6d0dc, 0x6d0f0, 0x6d110, 0x6d138, 0x6d1f0, 0x6d238, 0x6d244, 0x6d250, 0x6d26c, 0x6d278, 0x6d4b8, 0x6d4c4, 0x6d4e8, 0x6d55c, 0x6d58c, 0x6d5b8, 0x6d5dc, 0x6d610, 0x6d614, 0x6d628, 0x6d640, 0x6d64c, 0x6d670, 0x6d67c, 0x6d698, 0x6d760, 0x6d778, 0x6d780, 0x6d7a0, 0x6d7b8, 0x6d7d4, 0x6d81c, 0x6d898, 0x6dfd0, 0x6dff4, 0x6e010, 0x6e048, 0x6e058, 0x6e0c0, 0x6e0e8, 0x6e104, 0x6e128, 0x6e194, 0x6e1d0, 0x6e1e4, 0x6e204, 0x6e220, 0x6e240, 0x6e250, 0x6e294, 0x6e3f0, 0x6e458, 0x6e470, 0x6e484, 0x6e4a4, 0x6e4cc, 0x6e4f8, 0x6e644, 0x6e6b0, 0x6e6ec, 0x6e700, 0x6e720, 0x6e748, 0x6e788, 0x6e7b0, 0x6e7d8, 0x6e7e8, 0x6e7f4, 0x6e828, 0x6e834, 0x6e84c, 0x6e888, 0x6e89c, 0x6e8bc, 0x6e8d4, 0x6e8f0, 0x6e900, 0x6e918, 0x6e97c, 0x6e9a4, 0x6e9bc, 0x6e9f4, 0x6ea20, 0x6ea94, 0x6eabc, 0x6eae8, 0x6ef04, 0x6ef90, 0x6f888, 0x6f8c8, 0x6f9fc, 0x6fa1c, 0x6fa44, 0x6fa6c, 0x6fab0, 0x6fac0, 0x6fae8, 0x6fb0c, 0x6fbd8, 0x6fbf0, 0x6fc04, 0x6fc24, 0x6fc28, 0x6fc2c, 0x6fc50, 0x6fc80, 0x6fcc0, 0x6fce4, 0x6fcf8, 0x6fd1c, 0x6fd2c, 0x6fd50, 0x6fd6c, 0x6fddc, 0x6fe34, 0x6fe5c, 0x6fe6c, 0x6fe90, 0x6feac, 0x6ff44, 0x6ff50, 0x7004c, 0x702a0, 0x702ec, 0x702f8, 0x70414, 0x70548, 0x70560, 0x70578, 0x7058c, 0x705ac, 0x705b0, 0x705b4, 0x705d8, 0x70608, 0x7062c, 0x70690, 0x706b8, 0x706c0, 0x706d8, 0x70704, 0x70728, 0x7077c, 0x7084c, 0x708bc, 0x708d8, 0x708f8, 0x70900, 0x7090c, 0x7093c, 0x70978, 0x7098c, 0x709ac, 0x709bc, 0x709e0, 0x709f0, 0x70a2c, 0x70be4, 0x70c20, 0x70c34, 0x70c54, 0x70c6c, 0x70ccc, 0x70d3c, 0x70d6c, 0x70d98, 0x70dbc, 0x70de8, 0x70df8, 0x70e38, 0x70fc4, 0x7101c, 0x7102c, 0x71050, 0x71060, 0x710a0, 0x71168, 0x71180, 0x71188, 0x711a8, 0x711c0, 0x711dc, 0x711f8, 0x71228, 0x71380, 0x713bc, 0x713d4, 0x713e8, 0x71408, 0x71430, 0x71448, 0x71464, 0x71480, 0x714e8, 0x71564, 0x71568, 0x7156c, 0x71590, 0x71594, 0x71598, 0x715bc, 0x715d8, 0x71640, 0x71658, 0x7166c, 0x7168c, 0x71690, 0x71694, 0x716b8, 0x716bc, 0x716c0, 0x71700, 0x7170c, 0x7172c, 0x71784, 0x717f0, 0x71858, 0x71894, 0x718a8, 0x718c8, 0x718cc, 0x718d0, 0x7191c, 0x71934, 0x71a9c, 0x71ac0, 0x71adc, 0x71ae4, 0x71af0, 0x71b08, 0x71b30, 0x71b58, 0x71bbc, 0x71c78, 0x71d44, 0x71d6c, 0x71d94, 0x71dac, 0x71f0c, 0x71f48, 0x71f70, 0x71fb0, 0x71fdc, 0x72004, 0x72018, 0x72038, 0x72048, 0x72094, 0x720b0, 0x720bc, 0x720dc, 0x7210c, 0x72138, 0x7215c, 0x72164, 0x72168, 0x7218c, 0x721a4, 0x721c0, 0x721d8, 0x72364, 0x723cc, 0x723e4, 0x723ec, 0x7240c, 0x72410, 0x72414, 0x72478, 0x724a0, 0x724bc, 0x72520, 0x72714, 0x72730, 0x72754, 0x727bc, 0x727c0, 0x727e4, 0x727fc, 0x72818, 0x72838, 0x72868, 0x72894, 0x728b8, 0x728cc, 0x728f4, 0x72908, 0x7294c, 0x729b4, 0x729cc, 0x729e0, 0x72a00, 0x72a28, 0x72a40, 0x72a78, 0x72ae0, 0x72b5c, 0x72b60, 0x72b64, 0x72b88, 0x72b98, 0x72bbc, 0x72bdc, 0x72c4c, 0x72ca0, 0x72cb8, 0x72cd8, 0x72cf4, 0x72d2c, 0x72d7c, 0x72d88, 0x72da0, 0x72dc8, 0x72de4, 0x72dfc, 0x72f14, 0x72f38, 0x72f50, 0x72f7c, 0x72fa4, 0x72fac, 0x73018, 0x73040, 0x73068, 0x7308c, 0x730a4, 0x730dc, 0x73104, 0x73114, 0x73138, 0x73200, 0x73228, 0x7325c, 0x73424, 0x73460, 0x7349c, 0x734b0, 0x734d0, 0x734d4, 0x734d8, 0x734fc, 0x73500, 0x73504, 0x73528, 0x73540, 0x7357c, 0x73590, 0x735b0, 0x735c0, 0x735e4, 0x735f4, 0x73618, 0x73630, 0x73664, 0x73698, 0x736d8, 0x736f8, 0x73724, 0x7375c, 0x737a0, 0x737c0, 0x737ec, 0x737fc, 0x73818, 0x73860, 0x738bc, 0x73904, 0x73960, 0x739a8, 0x739f0, 0x73a4c, 0x73a68, 0x73a94, 0x73bb0, 0x73cd8, 0x73ce4, 0x73cfc, 0x73d18, 0x73d40, 0x73de4, 0x73e2c, 0x73e88, 0x73e94, 0x73eac, 0x73ed8, 0x73f10, 0x74014, 0x74124, 0x74130, 0x74148, 0x74190, 0x741ec, 0x74200, 0x74220, 0x7422c, 0x74244, 0x7428c, 0x742e8, 0x74348, 0x74390, 0x743ec, 0x74418, 0x74450, 0x74524, 0x74604, 0x7464c, 0x746a8, 0x746f8, 0x74758, 0x7483c, 0x7492c, 0x74974, 0x749d0, 0x749e8, 0x74a2c, 0x74a88, 0x74a94, 0x74ab0, 0x74ae4, 0x74b24, 0x74b3c, 0x74b60, 0x74b6c, 0x74b88, 0x74b94, 0x74bb0, 0x74bc8, 0x74bec, 0x74c40, 0x74d74, 0x74eb4, 0x74efc, 0x74f54, 0x74f9c, 0x74ff4, 0x75104, 0x75220, 0x7522c, 0x75244, 0x7525c, 0x75280, 0x753b0, 0x754ec, 0x75534, 0x75590, 0x7559c, 0x755b4, 0x756bc, 0x757d0, 0x758b4, 0x759a4, 0x75aa8, 0x75bb8, 0x75c00, 0x75c5c, 0x75d8c, 0x75ec8, 0x75f10, 0x75f6c, 0x75fb4, 0x76010, 0x7601c, 0x76034, 0x7607c, 0x760d8, 0x760f4, 0x76138, 0x76270, 0x763b4, 0x763c0, 0x763d8, 0x76420, 0x7647c, 0x76494, 0x764b8, 0x7659c, 0x7668c, 0x76698, 0x766b0, 0x76794, 0x76884, 0x76928, 0x76964, 0x769b0, 0x76ac8, 0x76bec, 0x76c34, 0x76c90, 0x76c9c, 0x76cb4, 0x76cf0, 0x76d38, 0x76d94, 0x76da0, 0x76db8, 0x76e00, 0x76e5c, 0x76e80, 0x76eb0, 0x76ef8, 0x76f54, 0x76f74, 0x76fa0, 0x76fe8, 0x77044, 0x77050, 0x77068, 0x77088, 0x770b4, 0x770d8, 0x77120, 0x7717c, 0x77188, 0x771a0, 0x771c0, 0x771ec, 0x77234, 0x77290, 0x772d8, 0x77334, 0x77340, 0x77358, 0x77364, 0x7737c, 0x77388, 0x773a0, 0x773e8, 0x77444, 0x77450, 0x77468, 0x77488, 0x774b4, 0x774c0, 0x774d8, 0x774e4, 0x774fc, 0x77544, 0x775a0, 0x775c0, 0x775ec, 0x7760c, 0x77638, 0x77680, 0x776dc, 0x77724, 0x77780, 0x7778c, 0x777a4, 0x777ec, 0x77848, 0x77868, 0x77894, 0x778a0, 0x778b8, 0x77900, 0x7795c, 0x77968, 0x77980, 0x779c8, 0x77a24, 0x77a48, 0x77a78, 0x77a84, 0x77a9c, 0x77ae4, 0x77b40, 0x77b88, 0x77be4, 0x77bf0, 0x77c08, 0x77c14, 0x77c2c, 0x77c38, 0x77c50, 0x77c78, 0x77cac, 0x77cf4, 0x77d48, 0x77d90, 0x77de4, 0x77df0, 0x77e08, 0x77e50, 0x77eac, 0x77ef4, 0x77f48, 0x77f90, 0x77fec, 0x78034, 0x78088, 0x78134, 0x78140, 0x78158, 0x781ac, 0x7820c, 0x782a8, 0x782f0, 0x78344, 0x78398, 0x783f8, 0x7841c, 0x7844c, 0x78458, 0x78470, 0x784b8, 0x78514, 0x78548, 0x78588, 0x785c0, 0x78604, 0x7863c, 0x78680, 0x786bc, 0x78704, 0x7874c, 0x787a8, 0x787ec, 0x7883c, 0x78858, 0x78938, 0x78a54, 0x78b7c, 0x78b88, 0x78ba0, 0x78bc0, 0x78bf4, 0x78c14, 0x78c34, 0x78ca8, 0x78d08, 0x78d24, 0x78d44, 0x78d64, 0x78d84, 0x78da4, 0x78dc4, 0x78de4, 0x78e04, 0x78e24, 0x79108, 0x79128, 0x79148, 0x79168, 0x79188, 0x791a8, 0x791c8, 0x79228, 0x792c4, 0x792fc, 0x7931c, 0x7933c, 0x7935c, 0x7937c, 0x7939c, 0x793bc, 0x793dc, 0x793fc, 0x7941c, 0x7943c, 0x7945c, 0x7947c, 0x7949c, 0x794bc, 0x794dc, 0x794fc, 0x7951c, 0x7953c, 0x7955c, 0x7957c, 0x7959c, 0x795bc, 0x795dc, 0x795fc, 0x7961c, 0x7963c, 0x7965c, 0x7967c, 0x7969c, 0x796bc, 0x796dc, 0x796fc, 0x7971c, 0x7973c, 0x7975c, 0x7977c, 0x7979c, 0x797bc, 0x797dc, 0x797fc, 0x7981c, 0x7983c, 0x7985c, 0x7987c, 0x7989c, 0x798bc, 0x798dc, 0x798fc, 0x7991c, 0x7993c, 0x7995c, 0x7997c, 0x7999c, 0x799bc, 0x79d7c, 0x7c5e4, 0x7c624, 0x7c664, 0x7c6a4, 0x7c6e4, 0x7c8e0, 0x7cadc, 0x7cce4, 0x7cee8, 0x7ee60, 0x7eeec, 0x7ef38, 0x7efa8, 0x7efdc, 0x7effc, 0x7f01c, 0x7f03c, 0x7f0d4, 0x7f0f4, 0x7f114, 0x7f134, 0x7f1cc, 0x7f200, 0x7f234, 0x7f278, 0x7f298, 0x7f2b8, 0x7f2d8, 0x7fdd4, 0x809dc, 0x809e0, 0x80a00, 0x80a7c, 0x80b14, 0x80b30, 0x80bc8, 0x80bfc, 0x80c3c, 0x80ca4, 0x80e68, 0x80e88, 0x80e98, 0x80f30, 0x80f54, 0x80f98, 0x80fb8, 0x80fd8, 0x80ff8, 0x81018, 0x81038, 0x81058, 0x81078, 0x81098, 0x810b8, 0x810d8, 0x810f8, 0x81118, 0x81138, 0x81158, 0x81178, 0x81198, 0x811b8, 0x811d8, 0x811f8, 0x81218, 0x81238, 0x81258, 0x81278, 0x81298, 0x812b8, 0x812d8, 0x812f8, 0x81318, 0x81338, 0x81358, 0x81378, 0x81398, 0x813b8, 0x813d8, 0x813f8, 0x81418, 0x81438, 0x81458, 0x81478, 0x81510, 0x815a8, 0x81640, 0x816d8, 0x81770, 0x81808, 0x818a0, 0x81a48, 0x81aa8, 0x81af8, 0x81b20, 0x81ba8, 0x81bf4, 0x81ca0, 0x81d54, 0x81ea0, 0x81f60, 0x81fe8, 0x8209c, 0x82124, 0x82168, 0x821b4, 0x821d4, 0x821f4, 0x82214, 0x82234, 0x82254, 0x82274, 0x82294, 0x822b4, 0x82380, 0x824b0, 0x826a0, 0x826c4, 0x82714, 0x8273c, 0x82798, 0x827d8, 0x82824, 0x8284c, 0x8288c, 0x828b4, 0x828f0, 0x828f8, 0x82abc, 0x82c34, 0x83d88, 0x83fc0, 0x846a4, 0x848f0, 0x8492c, 0x8498c, 0x84b2c, 0x84c20, 0x84d74, 0x85170, 0x852d0, 0x85430, 0x8590c, 0x85918, 0x85cf0, 0x85dc8, 0x85fb0, 0x86084, 0x860d0, 0x861d4, 0x86288, 0x862e0, 0x863bc, 0x863c0, 0x863c4, 0x863e4, 0x864dc, 0x8658c, 0x865a0, 0x86684, 0x86908, 0x86b1c, 0x87478, 0x8798c, 0x879f8, 0x87b34, 0x87c74, 0x87e9c, 0x8889c, 0x88a30, 0x88d18, 0x88f10, 0x89268, 0x8938c, 0x895fc, 0x89668, 0x8974c, 0x89760, 0x89974, 0x89bb8, 0x89c30, 0x89fcc, 0x8a07c, 0x8a134, 0x8abe8, 0x8b1a4, 0x8b4f4, 0x8b508, 0x8b660, 0x8b968, 0x8baa4, 0x8bd30, 0x8bd78, 0x8bdc0, 0x8be08, 0x8be50, 0x8be98, 0x8c124, 0x8c14c, 0x8c174, 0x8c65c, 0x8c690, 0x8c774, 0x8c92c, 0x8cfbc, 0x8d2c8, 0x8d618, 0x8d648, 0x8d730, 0x8db58, 0x8dfe8, 0x8e2bc, 0x8e878, 0x8ec68, 0x8f390, 0x8f410, 0x8f4b4, 0x8f4f8, 0x8fd14, 0x90174, 0x902f4, 0x90488, 0x91488, 0x92314, 0x92754, 0x92bec, 0x92d20, 0x92e78, 0x92fd0, 0x93128, 0x93280, 0x933d8, 0x93530, 0x93688, 0x937e0, 0x93938, 0x93a90, 0x93be8, 0x93ce8, 0x93e74, 0x93f14, 0x944f4, 0x94f70, 0x95514, 0x95d74, 0x95e08, 0x96040, 0x96994, 0x96b3c, 0x96ec0, 0x9712c, 0x976b0, 0x97a9c, 0x985f8, 0x98b5c, 0x98c80, 0x99060, 0x9906c, 0x99070, 0x990a4, 0x990c4, 0x9919c, 0x99218, 0x99290, 0x99298, 0x992a0, 0x992a8, 0x992b0, 0x992b8, 0x992c0, 0x99348, 0x99384, 0x993c8, 0x99424, 0x99460, 0x99650, 0x996ec, 0x9978c, 0x99824, 0x998b8, 0x99944, 0x99980, 0x9a438, 0x9a5c0, 0x9a658, 0x9a6b0, 0x9b944, 0x9b98c, 0x9babc, 0x9bbf4, 0x9bc98, 0x9bcec, 0x9bd88, 0x9c094, 0x9c2e4, 0x9c3e4, 0x9c4dc, 0x9c558, 0x9c7e0, 0x9c7fc, 0x9c890, 0x9c988, 0x9ca64, 0x9cbd4, 0x9cbf4, 0x9cc38, 0x9cd90, 0x9cd98, 0x9cf14, 0x9cf4c, 0x9cf9c, 0x9cfd4, 0x9d078, 0x9d400, 0x9db74, 0x9de48, 0x9e11c, 0x9e520, 0x9f3f4, 0x9f488, 0x9f58c, 0x9f598, 0x9f5a8, 0x9f5b8, 0x9f5dc, 0x9f600, 0x9f658, 0x9f688, 0x9f774, 0x9f7f8, 0x9f830, 0x9f874, 0x9f898, 0x9f8bc, 0x9fdf4, 0x9fe48, 0x9fe50, 0x9fe7c, 0x9ffdc, 0xa016c, 0xa0184, 0xa025c, 0xa035c, 0xa04a8, 0xa0858, 0xa0a14, 0xa0aac, 0xa0df8, 0xa0fd4, 0xa1144, 0xa12a8, 0xa140c, 0xa1570, 0xa16d4, 0xa18f0, 0xa1b0c, 0xa1ba4, 0xa1cc0, 0xa1d24, 0xa1f1c, 0xa1f44, 0xa1f84, 0xa1f94, 0xa212c, 0xa22e0, 0xa23c8, 0xa23d8, 0xa2654, 0xa2658, 0xa265c, 0xa2660, 0xa28b0, 0xa2918, 0xa293c, 0xa2a64, 0xa2b10, 0xa2c18, 0xa2cb0, 0xa2d80, 0xa2e28, 0xa2fa4, 0xa2fa8, 0xa2fdc, 0xa2ffc, 0xa3020, 0xa3058, 0xa307c, 0xa30b4, 0xa30d8, 0xa3110, 0xa3134, 0xa316c, 0xa3190, 0xa31c8, 0xa31ec, 0xa3224, 0xa3248, 0xa3280, 0xa32a4, 0xa32b0, 0xa32b8, 0xa3304, 0xa330c, 0xa3358, 0xa335c, 0xa3390, 0xa33b0, 0xa3480, 0xa3598, 0xa3634, 0xa36bc, 0xa374c, 0xa37ec, 0xa38cc, 0xa39ac, 0xa3ab0, 0xa3b7c, 0xa3bb0, 0xa3d00, 0xa3e24, 0xa3f4c, 0xa4028, 0xa4074, 0xa4ad4, 0xa4bf8, 0xa4d04, 0xa5024, 0xa5620, 0xa5e34, 0xa6070, 0xa68c4, 0xa693c, 0xa6a00, 0xa6a48, 0xa6ae8, 0xa6b88, 0xa6c28, 0xa6c4c, 0xa6cc0, 0xa6d8c, 0xa6d90, 0xa6d94, 0xa6da8, 0xa6db8, 0xa7244, 0xa78d0, 0xa86ac, 0xa88b0, 0xa88d8, 0xa8b64, 0xa8bb0, 0xa8c9c, 0xa8e18, 0xa8ea8, 0xab6f0, 0xabc54, 0xabcd4, 0xabd54, 0xabda0, 0xabe3c, 0xabe78, 0xabf10, 0xabf4c, 0xabfe8, 0xac034, 0xac07c, 0xac274, 0xac2ac, 0xac2e4, 0xac31c, 0xac3fc, 0xac4cc, 0xac5bc, 0xac7ac, 0xac84c, 0xaca2c, 0xacbf4, 0xaccec, 0xacddc, 0xad114, 0xad2d0, 0xad514, 0xad758, 0xad948, 0xadb30, 0xade44, 0xae008, 0xae2e8, 0xae3d8, 0xae4d0, 0xae70c, 0xaecac, 0xaee6c, 0xaeffc, 0xaf010, 0xaf160, 0xaf164, 0xaf168, 0xaf2b8, 0xaf2bc, 0xaf430, 0xaf434, 0xaf598, 0xaf59c, 0xaf640, 0xaf644, 0xafb24, 0xafc9c, 0xafcc0, 0xafce0, 0xafe58, 0xafe94, 0xaff08, 0xb009c, 0xb00b8, 0xb00bc, 0xb00c0, 0xb0144, 0xb0424, 0xb0478, 0xb0488, 0xb0498, 0xb04d4, 0xb04e4, 0xb04f4, 0xb066c, 0xb0898, 0xb0940, 0xb0960, 0xb096c, 0xb0984, 0xb0994, 0xb0a7c, 0xb0d5c, 0xb0e44, 0xb0f48, 0xb0f50, 0xb0f54, 0xb0f5c, 0xb0fb0, 0xb1070, 0xb12bc, 0xb12c8, 0xb12d0, 0xb12d8, 0xb1300, 0xb130c, 0xb13e8, 0xb1408, 0xb1420, 0xb1708, 0xb1774, 0xb185c, 0xb1b3c, 0xb1c24, 0xb1d28, 0xb1d30, 0xb1d3c, 0xb1d40, 0xb1d48, 0xb1d9c, 0xb1e5c, 0xb255c, 0xb2568, 0xb2570, 0xb2578, 0xb2688, 0xb2794, 0xb27b4, 0xb27c0, 0xb27d8, 0xb27e8, 0xb28d0, 0xb2bb0, 0xb2c98, 0xb2d9c, 0xb2da4, 0xb2da8, 0xb2db0, 0xb2e04, 0xb2ec4, 0xb3110, 0xb311c, 0xb3124, 0xb312c, 0xb314c, 0xb3158, 0xb3170, 0xb33e0, 0xb344c, 0xb3534, 0xb3814, 0xb38fc, 0xb3a00, 0xb3a08, 0xb3a0c, 0xb3a14, 0xb3a68, 0xb3b0c, 0xb4568, 0xb4574, 0xb457c, 0xb4584, 0xb4640, 0xb4760, 0xb47ec, 0xb4844, 0xb4870, 0xb4c9c, 0xb56e0, 0xb57ac, 0xb5920, 0xb5cd4, 0xb5d44, 0xb5d74, 0xb6238, 0xb62cc, 0xb6360, 0xb63e4, 0xb656c, 0xb6750, 0xb6894, 0xb6a08, 0xb6b48, 0xb6ca4, 0xb7088, 0xb7110, 0xb7508, 0xb7a48, 0xb7d64, 0xb80a0, 0xb8358, 0xb8500, 0xb8608, 0xb867c, 0xb868c, 0xb86dc, 0xb870c, 0xb87c4, 0xb87ec, 0xb8984, 0xb8dd8, 0xb8f98, 0xb9330, 0xb9418, 0xb9760, 0xb9c34, 0xb9c88, 0xb9e00, 0xb9f58, 0xba110, 0xba114, 0xba1a0, 0xba2ec, 0xba6a4, 0xba71c, 0xba81c, 0xba908, 0xba9ec, 0xbaa70, 0xbaae8, 0xbab64, 0xbab74, 0xbab78, 0xbab7c, 0xbac78, 0xbac88, 0xbac8c, 0xbac90, 0xbad84, 0xbae74, 0xbaef0, 0xbb034, 0xbb108, 0xbb160, 0xbb378, 0xbb380, 0xbb3cc, 0xbb4c4, 0xbb5e0, 0xbb718, 0xbb830, 0xbb938, 0xbb984, 0xbba94, 0xbbba0, 0xbbcac, 0xbbddc, 0xbc150, 0xbc204, 0xbc228, 0xbc25c, 0xbc260, 0xbc27c, 0xbc2c0, 0xbc434, 0xbc438, 0xbc450, 0xbd6b8, 0xbf2e4, 0xbf48c, 0xbfd4c, 0xbff40, 0xc0de0, 0xc0f58, 0xc1150, 0xc1168, 0xc1300, 0xc1308, 0xc147c, 0xc1728, 0xc18a4, 0xc198c, 0xc1ad4, 0xc1aec, 0xc1c24, 0xc1c84, 0xc1e60, 0xc1f58, 0xc2074, 0xc2190, 0xc2248, 0xc22c0, 0xc2484, 0xc24a8, 0xc24b0, 0xc24b8, 0xc24ec, 0xc2558, 0xc2650, 0xc2670, 0xc26a0, 0xc2f9c, 0xc2fbc, 0xc2fd4, 0xc37c4, 0xc37f4, 0xc3cfc, 0xc3da8, 0xc3dc8, 0xc3dd4, 0xc3e90, 0xc3f48, 0xc3f54, 0xc3f5c, 0xc3ff4, 0xc4088, 0xc40fc, 0xc416c, 0xc4220, 0xc4228, 0xc4230, 0xc4244, 0xc424c, 0xc42e4, 0xc4378, 0xc4410, 0xc44a4, 0xc4518, 0xc4588, 0xc45fc, 0xc466c, 0xc4704, 0xc4798, 0xc480c, 0xc487c, 0xc48dc, 0xc4950, 0xc49c0, 0xc4a34, 0xc4aa4, 0xc4adc, 0xc4b10, 0xc4b84, 0xc4bf4, 0xc4c5c, 0xc4cc0, 0xc4cc4, 0xc4cd4, 0xc4db4, 0xc4e90, 0xc4eb0, 0xc4f0c, 0xc50d4, 0xc5108, 0xc5170, 0xc51f0, 0xc5210, 0xc7394, 0xc74c4, 0xc74dc, 0xc74e4, 0xc74f8, 0xc7528, 0xc7540, 0xc75b4, 0xc7624, 0xc765c, 0xc7690, 0xc76c8, 0xc76fc, 0xc7714, 0xc772c, 0xc7778, 0xc7790, 0xc8394, 0xc8c80, 0xc8e2c, 0xc8e4c, 0xc8e6c, 0xc8e8c, 0xc8e90, 0xc8f58, 0xc9108, 0xc9120, 0xc91dc, 0xc9294, 0xc92b4, 0xc93ac, 0xc94a0, 0xc94f0, 0xc953c, 0xc958c, 0xc95d8, 0xc96b8, 0xc9794, 0xc9868, 0xc9938, 0xc9944, 0xc995c, 0xc9974, 0xc998c, 0xc9b58, 0xc9fe4, 0xca010, 0xca8f0, 0xca9bc, 0xcace8, 0xcb6b4, 0xcb6d4, 0xcb6f4, 0xcb7c8, 0xcb898, 0xcb918, 0xcb994, 0xcb9a8, 0xcb9b8, 0xcb9d0, 0xcb9f0, 0xcba64, 0xcbad4, 0xcbd70, 0xcbd90, 0xcbd98, 0xcbda8, 0xcbdc8, 0xcbde0, 0xcbe04, 0xcbe78, 0xcbf20, 0xcbf58, 0xcbf8c, 0xcc070, 0xcc080, 0xcc09c, 0xcc11c, 0xcc198, 0xcc218, 0xcc294, 0xcc2f0, 0xcc348, 0xcc368, 0xcc384, 0xcc440, 0xcc4f8, 0xcc524, 0xcc54c, 0xcc6f4, 0xcc714, 0xcc758, 0xcc798, 0xcc7dc, 0xcc81c, 0xcc8e8, 0xcc960, 0xccb00, 0xccf0c, 0xccf74, 0xcd040, 0xcd100, 0xcd1dc, 0xce3a8, 0xce500, 0xce6f8, 0xce78c, 0xce86c, 0xceb50, 0xced04, 0xcede4, 0xcee54, 0xceec4, 0xcf080, 0xcf0e8, 0xcf180, 0xcf270, 0xcf448, 0xcf470, 0xcf498, 0xcf4c0, 0xcf528, 0xcf540, 0xcf578, 0xcf678, 0xcf680, 0xcf694, 0xcf698, 0xcf6a8, 0xcf6d0, 0xcf70c, 0xcf710, 0xcf800, 0xcf810, 0xcf814, 0xcf818, 0xcf82c, 0xcf874, 0xcf898, 0xcf8a4, 0xcf974, 0xcf9cc, 0xcf9d8, 0xcfaa8, 0xcfb58, 0xcfbac, 0xcfbc4, 0xcfc94, 0xcfcb0, 0xcfdac, 0xcfde0, 0xcfde4, 0xcfe08, 0xcfe94, 0xcfea8, 0xcfeac, 0xcfeb0, 0xcfeb4, 0xcff00, 0xcff14, 0xcff5c, 0xd0068, 0xd006c, 0xd0084, 0xd0088, 0xd01b4, 0xd01b8, 0xd01f8, 0xd01fc, 0xd0340, 0xd0348, 0xd0350, 0xd03e8, 0xd04c4, 0xd04c8, 0xd0510, 0xd0518, 0xd0580, 0xd0664, 0xd0680, 0xd070c, 0xd0710, 0xd082c, 0xd0890, 0xd0898, 0xd08c8, 0xd08cc, 0xd0960, 0xd09a0, 0xd09d4, 0xd0a08, 0xd0abc, 0xd0b38, 0xd0bac, 0xd0bc4, 0xd0bdc, 0xd0bfc, 0xd0c14, 0xd0c30, 0xd0c40, 0xd0c50, 0xd0c54, 0xd0cb0, 0xd0ccc, 0xd0ce4, 0xd0d50, 0xd0d64, 0xd0d74, 0xd0d78, 0xd0d7c, 0xd0d90, 0xd0da0, 0xd0da4, 0xd0db0, 0xd0db4, 0xd0db8, 0xd0dbc, 0xd0dd0, 0xd0dd4, 0xd0df8, 0xd0e04, 0xd0e18, 0xd0e3c, 0xd0e48, 0xd16cc, 0xd16f8, 0xd1a68, 0xd1ae4, 0xd1c38, 0xd1c4c, 0xd1c50, 0xd1c54, 0xd1c60, 0xd1c64, 0xd1c70, 0xd1c74, 0xd1c98, 0xd1c9c, 0xd1ca0, 0xd1cc4, 0xd1ce8, 0xd1d0c, 0xd1d30, 0xd1d54, 0xd1d78, 0xd1d9c, 0xd1dc0, 0xd1de4, 0xd1e08, 0xd1e14, 0xd1e1c, 0xd1e24, 0xd1e30, 0xd1ef4, 0xd1f5c, 0xd1fd4, 0xd2120, 0xd2140, 0xd22e4, 0xd2410, 0xd275c, 0xd289c, 0xd2940, 0xd2b30, 0xd2be4, 0xd2c84, 0xd2cd8, 0xd2d34, 0xd2d3c, 0xd2d90, 0xd2dec, 0xd2df4, 0xd2e50, 0xd2eac, 0xd2f08, 0xd2f64, 0xd2fc0, 0xd301c, 0xd3078, 0xd307c, 0xd3080, 0xd3094, 0xd3098, 0xd30bc, 0xd30c8, 0xd30dc, 0xd30e0, 0xd3104, 0xd3110, 0xd3124, 0xd3128, 0xd313c, 0xd3140, 0xd317c, 0xd31f8, 0xd320c, 0xd3278, 0xd328c, 0xd32bc, 0xd331c, 0xd3398, 0xd33ac, 0xd3418, 0xd342c, 0xd345c, 0xd34bc, 0xd3538, 0xd35a8, 0xd35bc, 0xd3628, 0xd363c, 0xd3654, 0xd3658, 0xd3730, 0xd37a0, 0xd37fc, 0xd3848, 0xd385c, 0xd3944, 0xd398c, 0xd3a1c, 0xd3b44, 0xd3b60, 0xd3b88, 0xd3bb0, 0xd3f90, 0xd4014, 0xd43c4, 0xd440c, 0xd44e0, 0xd4614, 0xd46c8, 0xd55ac, 0xd5614, 0xd5628, 0xd5634, 0xd5650, 0xd56b0, 0xdd580, 0xdd624, 0xe2634, 0xe27d8, 0xe27dc, 0xe2800, 0xe3790, 0xe3794, 0xe37b4, 0xe465c, 0xe4660, 0xe4670, 0xe4de8, 0xe4dec, 0xe4df0, 0xe5090, 0xe5178, 0xe517c, 0xe5198, 0xe73a8, 0xe73ac, 0xe73bc, 0xe73cc, 0xe73dc, 0xe73f4, 0xe7410, 0xe7434, 0xe8da4, 0xe8f68, 0xe8f9c, 0xeea90, 0xeea94, 0xeeaa4, 0xeeae0, 0xeec08, 0xeece8, 0xeecec, 0xeed08, 0xf0744, 0xf07f8, 0xf1228, 0xf122c, 0xf1240, 0xf174c, 0xf180c, 0xf1ad0, 0xf1b18, 0xf2650, 0xf26a8, 0xf26b8, 0xf26c8, 0xf26f0, 0xf2718, 0xf2740, 0xf2a7c, 0xf2aa4, 0xf32e4, 0xf32e8, 0xf32f4, 0xf3458, 0xf3470, 0xf490c, 0xf4910, 0xf4930, 0xfc6e4, 0xfc84c, 0xfc94c, 0xfcfe4, 0xfd094, 0xfd788, 0xfd78c, 0xfd7a4, 0xfd87c, 0xfd9c4, 0xfdac0, 0xfdae8, 0xfdc10, 0xfddb0, 0xfdeb4, 0xfdec0, 0xfe01c, 0xfe090, 0xfe0f8, 0xfe114, 0xfe118, 0xfe13c, 0xfe150, 0xfe190, 0xfe230, 0xfe244, 0xfe290, 0xfe2e4, 0xfe308, 0xfe30c, 0xfe330, 0xfe33c, 0xfe340, 0xfe344, 0xfe388, 0xfe38c, 0xfe394, 0xfe4bc, 0xfe4c0, 0xfe4c4, 0xfe4c8, 0xfe4d0, 0xfe4d4, 0xfe51c, 0xfe520, 0xfe528, 0xfe55c, 0xfe580, 0xfe59c, 0xfe5d4, 0xfe614, 0xfe628, 0xfe63c, 0xfe67c, 0x1014b0, 0x1020d0, 0x10226c, 0x102384, 0x1029ec, 0x102a00, 0x102cc8, 0x102ccc, 0x102cd0, 0x102d20, 0x102d50, 0x102d58, 0x102dbc, 0x102de8, 0x102e28, 0x103074, 0x103110, 0x10373c, 0x103750, 0x1037a0, 0x1037a4, 0x103830, 0x1038d4, 0x103af8, 0x103c10, 0x103d78, 0x103e04, 0x103e90, 0x103ec0, 0x103efc, 0x103f34, 0x103f64, 0x103f70, 0x103f80, 0x103fb8, 0x1041c4, 0x1041c8, 0x104254, 0x1042bc, 0x104310, 0x10431c, 0x104350, 0x104364, 0x1043a8, 0x1043e8, 0x1043f4, 0x104460, 0x104468, 0x104470, 0x1044fc, 0x104508, 0x104514, 0x10454c, 0x104584, 0x1045ac, 0x104bc0, 0x104cfc, 0x104db0, 0x104ee0, 0x104ee4, 0x104ef4, 0x104efc, 0x104f04, 0x104f0c, 0x104f20, 0x10502c, 0x105174, 0x10523c, 0x105248, 0x105268, 0x105274, 0x105298, 0x1052a8, 0x1052b0, 0x1052b8, 0x1052c0, 0x10538c, 0x1053b8, 0x105660, 0x105668, 0x105670, 0x105678, 0x105680, 0x105684, 0x105688, 0x10569c, 0x1056e0, 0x105730, 0x105754, 0x105790, 0x105814, 0x105838, 0x105844, 0x105850, 0x1058ac, 0x105908, 0x105a3c, 0x105a44, 0x105a4c, 0x105a60, 0x105ae8, 0x105b24, 0x105cd8, 0x1062e4, 0x1062f4, 0x1064b0, 0x106544, 0x106764, 0x10692c, 0x106dc0, 0x107384, 0x10778c, 0x108318, 0x108574, 0x108628, 0x108814, 0x1088bc, 0x108ac8, 0x108cc0, 0x10959c, 0x1099f8, 0x109a00, 0x109db4, 0x10a3fc, 0x10a404, 0x10a734, 0x10a954, 0x10b098, 0x10b09c, 0x10b0a0, 0x10b274, 0x10b28c, 0x10b2b0, 0x10b2ec, 0x10b330, 0x10b764, 0x10b788, 0x10b7c4, 0x10bb10, 0x10bb7c, 0x10bc7c, 0x10bddc, 0x10c2c8, 0x10c6cc, 0x10c6d4, 0x10c6d8, 0x10c6dc, 0x10c6e0, 0x10c6e4, 0x10c6e8, 0x10c6ec, 0x10c6f0, 0x10c6f4, 0x10d360, 0x10d508, 0x10d50c, 0x10d510, 0x10d514, 0x10d518, 0x10d51c, 0x10d520, 0x10d524, 0x10d528, 0x10d8e0, 0x10dd94, 0x10df68, 0x10e13c, 0x10e8c8, 0x111584, 0x111f4c, 0x1120f4, 0x1122c8, 0x114f60, 0x11537c, 0x115550, 0x117fd8, 0x1180f4, 0x118424, 0x1188e8, 0x118ba0, 0x118c10, 0x118cbc, 0x118cc0, 0x118cf4, 0x118d14, 0x118e18, 0x118ec0, 0x118ec4, 0x118ef8, 0x118f18, 0x118fe8, 0x1191a0, 0x119394, 0x119650, 0x1196c4, 0x119738, 0x119960, 0x119ac0, 0x119d54, 0x119d90, 0x119f90, 0x11a0b4, 0x11a238, 0x11a440, 0x11ab20, 0x11ab28, 0x11ac24, 0x11ad9c, 0x11b45c, 0x11b554, 0x11b5f0, 0x11b718, 0x11bc98, 0x11c038, 0x11c8c0, 0x11ced0, 0x11d678, 0x11dbc4, 0x11dcc0, 0x11ddd4, 0x11df7c, 0x11e10c, 0x11ead8, 0x11f91c, 0x11f9dc, 0x11fce8, 0x11fd94, 0x1201c0, 0x1201ec, 0x1205ac, 0x1205c8, 0x1206b8, 0x120908, 0x12095c, 0x120960, 0x120964, 0x120a08, 0x120a78, 0x120af0, 0x120b54, 0x121050, 0x121068, 0x1210cc, 0x121198, 0x1212a0, 0x121310, 0x121370, 0x1213d8, 0x1214c4, 0x121558, 0x1217a8, 0x12195c, 0x121a70, 0x121bc8, 0x121d14, 0x121d84, 0x121ee8, 0x122010, 0x1220b4, 0x12216c, 0x1221dc, 0x12224c, 0x12240c, 0x122674, 0x122cbc, 0x122d88, 0x1235d0, 0x1237d0, 0x12386c, 0x123d24, 0x123d60, 0x123e04, 0x123f0c, 0x1244e0, 0x1244f4, 0x124570, 0x1248f4, 0x124900, 0x124904, 0x124938, 0x124958, 0x12498c, 0x124a5c, 0x124b74, 0x124d54, 0x124e40, 0x124f2c, 0x1250dc, 0x1253b8, 0x126070, 0x1260ec, 0x126138, 0x1262d0, 0x12636c, 0x1264b8, 0x126ef4, 0x127010, 0x1270cc, 0x12721c, 0x12735c, 0x1274d8, 0x1277c0, 0x127824, 0x12791c, 0x127aa4, 0x127c48, 0x127cbc, 0x127d30, 0x1285ac, 0x128768, 0x128a70, 0x129574, 0x1298c8, 0x129ac0, 0x12bee8, 0x12c3b8, 0x12d2ac, 0x12e87c, 0x12ea24, 0x12ebd8, 0x12f198, 0x12f1a4, 0x12f1ac, 0x12f1f0, 0x12f230, 0x12f274, 0x12f2b4, 0x12f2d4, 0x12f3ac, 0x12f504, 0x130188, 0x130444, 0x130510, 0x1305e0, 0x130600, 0x130618, 0x130620, 0x130640, 0x130660, 0x130680, 0x130698, 0x1308bc, 0x130adc, 0x130b20, 0x130c3c, 0x130db8, 0x130ffc, 0x131cb0, 0x131f34, 0x1321f0, 0x132294, 0x132350, 0x13244c, 0x1325d0, 0x1326c4, 0x132720, 0x132728, 0x13272c, 0x132910, 0x132a58, 0x132bec, 0x132c14, 0x132c54, 0x132d44, 0x132e58, 0x132e94, 0x132eec, 0x132ff8, 0x133000, 0x133058, 0x133060, 0x133074, 0x13307c, 0x133084, 0x13308c, 0x1330b4, 0x1330bc, 0x1330c4, 0x13313c, 0x133634, 0x133a50, 0x133fe8, 0x13420c, 0x1342d8, 0x134324, 0x1343e4, 0x1344d0, 0x1345c4, 0x1345c8, 0x134750, 0x134844, 0x134940, 0x134a3c, 0x134a60, 0x134b30, 0x134b58, 0x134b98, 0x134bb4, 0x134c74, 0x134d28, 0x134d88, 0x134de8, 0x134e48, 0x134f38, 0x134fc8, 0x1350c4, 0x135224, 0x135234, 0x135648, 0x1356e4, 0x135844, 0x135978, 0x135ef0, 0x135f74, 0x135f80, 0x135fc0, 0x136030, 0x13603c, 0x136068, 0x136158, 0x13615c, 0x136180, 0x18afc0, 0x18afc8, 0x18afd0, 0x18afd8, 0x18afe0, 0x18afe8, 0x18aff0, 0x18aff8, 0x18b000, 0x18b008, 0x18b010, 0x18b018, 0x18b020, 0x18b028, 0x18b030, 0x18b038, 0x18b040, 0x18b048, 0x18b050, 0x18b058, 0x18b060, 0x18b068, 0x18b070, 0x18b078, 0x18b080, 0x18b088, 0x18b090, 0x18b098, 0x18b0a0, 0x18b0a8, 0x18b0b0, 0x18b0b8, 0x18b0c0, 0x18b0c8, 0x18b0d0, 0x18b0d8, 0x18b0e0, 0x18b0e8, 0x18b0f0, 0x18b0f8, 0x18b100, 0x18b108, 0x18b110, 0x18b118, 0x18b120, 0x18b128, 0x18b130, 0x18b138, 0x18b140, 0x18b148, 0x18b150, 0x18b158, 0x18b160, 0x18b168, 0x18b170, 0x18b178, 0x18b180, 0x18b188, 0x18b190, 0x18b198, 0x18b1a0, 0x18b1a8, 0x18b1b0, 0x18b1b8, 0x18b1c0, 0x18b1c8, 0x18b1d0, 0x18b1d8, 0x18b1e0, 0x18b1e8, 0x18b1f0, 0x18b1f8, 0x18b200, 0x18b208, 0x18b210, 0x18b218, 0x18b220, 0x18b228, 0x18b230, 0x18b238, 0x18b240, 0x18b248, 0x18b250, 0x18b258, 0x18b260, 0x18b268, 0x18b270, 0x18b278, 0x18b280, 0x18b288, 0x18b290, 0x18b298, 0x18b2a0, 0x18b2a8, 0x18b2b0, 0x18b2b8, 0x18b2d0, 0x18b2d8, 0x18b2e0, 0x18b2e8, 0x18b2f8, 0x18b300, 0x18b308, 0x18b310, 0x18b318, 0x18b320, 0x18b328, 0x18b330, 0x18b338, 0x18b340, 0x18b348, 0x18b350, 0x18b358, 0x18b360, 0x18b368, 0x18b370, 0x18b378, 0x18b380, 0x18b388, 0x18b390, 0x18b398, 0x18b3a0, 0x18b3a8, 0x18b3b0, 0x18b3b8, 0x18b3c0, 0x18b3c8, 0x18b3d0, 0x18b3d8, 0x18b3e0, 0x18b3e8, 0x18b3f0, 0x18b3f8, 0x18b400, 0x18b408, 0x18b410, 0x18b418, 0x18b420, 0x18b428, 0x18b430, 0x18b438, 0x18b440, 0x18b448, 0x18b450, 0x18b458, 0x18b460, 0x18b468, 0x18b470, 0x18b478, 0x18b480, 0x18b488, 0x18b490, 0x18b498, 0x18b4a0, 0x18b4a8, 0x18b4b0, 0x18b4b8, 0x18b4c0, 0x18b4c8, 0x18b4d0, 0x18b4d8, 0x18b4e0, 0x18b4e8, 0x18b4f0, 0x18b4f8, 0x18b500, 0x18b508, 0x18b510, 0x18b518];
var func_name_main = ["sub_115130", "opendir", "j__ZNSt16invalid_argumentD2Ev", "execve", "fdopen", "j__ZNSt3__120__throw_system_errorEiPKc", "j__ZNSt3__112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEE6appendIPcEENS_9enable_ifIXsrNS_21__is_forward_iteratorIT_EE5valueERS5_E4typeESA_SA_", "memcmp", "listen", "ioctl", "__errno", "flock", "remove", "j__Unwind_GetDataRelBase", "deflateEnd", "abort", "j__Unwind_Find_FDE", "realloc", "connect", "write", "j___cxa_get_globals_fast", "dup2", "pthread_self", "_exit", "j___cxa_throw", "srandom", "sscanf", "j__ZNSt3__115system_categoryEv", "fsync", "gettimeofday", "j_interpreter_wrap_int64_t", "srand", "getpagesize", "inflate", "pipe", "pthread_cond_broadcast", "epoll_create", "access", "j___cxa_end_catch", "snprintf", "j___deregister_frame_info", "j__Unwind_SetGR", "j__ZNSt3__112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEEC1ERKS5_", "signal", "lseek", "j__ZNKSt3__112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEE7compareEmmPKcm", "getpid", "j_strcmp", "readdir", "j__ZNSt3__112future_errorC2ENS_10error_codeE", "creat", "time", "j___self_kill", "inflateInit2_", "memset", "unlink", "strlcpy", "j__Znwm", "j_log_internal_impl", "dladdr", "fmodf", "malloc", "j__Unwind_GetLanguageSpecificData", "getcwd", "fork", "fmod", "random", "dlopen", "memmove", "gethostbyname", "strcat", "pthread_join", "rand", "fflush", "strncmp", "pthread_exit", "recv", "isxdigit", "strcpy", "j__Unwind_DeleteException", "strrchr", "fstat", "__cxa_atexit", "pthread_key_delete", "lstat", "j__ZNSt3__16vectorIiNS_9allocatorIiEEE21__push_back_slow_pathIRKiEEvOT_", "closedir", "pthread_create", "fclose", "setenv", "j_DobbyHook", "strdup", "inet_addr", "sleep", "open", "j__ZNSt11logic_errorC2ERKNSt3__112basic_stringIcNS0_11char_traitsIcEENS0_9allocatorIcEEEE", "pow", "strlcat", "strstr", "__assert2", "epoll_wait", "vfprintf", "kill", "deflateInit_", "munmap", "pthread_cond_wait", "pthread_setspecific", "deflate", "strchr", "chmod", "j___deregister_frame_info_bases", "pthread_mutex_destroy", "gettid", "bind", "getppid", "adler32", "popen", "j__ZNSt3__112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEEC1ERKS5_mmRKS4_", "pthread_cond_destroy", "fnmatch", "dl_iterate_phdr", "j__ZNSt9exceptionD2Ev", "waitpid", "fwrite", "j___cxa_rethrow_primary_exception", "j__ZNSt9type_infoD2Ev", "j__ZSt15get_new_handlerv", "llabs", "deflateBound", "readlink", "strncpy", "__cxa_finalize", "j___register_frame_info", "j___read_self", "exit", "raise", "dlclose", "vprintf", "j___cxa_uncaught_exception", "send", "atoll", "fgets", "lseek64", "j___cxa_get_globals", "inflateEnd", "pthread_mutex_init", "__android_log_vprint", "j___register_frame_info_table", "strtoul", "socket", "close", "j__ZNSt3__119__thread_local_dataEv", "epoll_ctl", "sysconf", "j___cxa_demangle", "j__Unwind_RaiseException", "calloc", "j__Unwind_SetIP", "execl", "getsockopt", "j__ZdaPv", "j__ZNSt11logic_errorC2EPKc", "read", "j___cxa_allocate_exception", "j___register_frame_info_bases", "j__Unwind_GetRegionStart", "fputc", "prctl", "j__ZdlPv", "j__Unwind_GetTextRelBase", "j__ZSt9terminatev", "j__Znam", "j__ZSt17rethrow_exceptionSt13exception_ptr", "strtoull", "j__Unwind_GetCFA", "setsockopt", "mkdir", "j___cxa_begin_catch", "pthread_mutex_unlock", "fcntl", "j__ZNSt3__16vectorIiNS_9allocatorIiEEE21__push_back_slow_pathIiEEvOT_", "j___cxa_free_exception", "vsnprintf", "setpgid", "j___cxa_decrement_exception_refcount", "ftruncate", "pthread_mutex_lock", "tolower", "j___dynamic_cast", "nanosleep", "syscall", "mprotect", "isupper", "j__ZNSt3__112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEED1Ev", "free", "toupper", "j__ZNSt9exceptionD2Ev_0", "dlsym", "pclose", "atoi", "stat", "accept", "j___cxa_rethrow", "j___cxa_increment_exception_refcount", "memchr", "j___cxa_current_primary_exception", "vsyslog", "inet_ntoa", "strerror", "fopen", "j__Unwind_Resume", "j__ZNSt15underflow_errorD2Ev", "isspace", "j___self_lseek", "pthread_detach", "strtok", "j_CodePatch", "pwrite", "fileno", "atol", "j__ZSt13get_terminatev", "j__ZNSt13runtime_errorC2ERKNSt3__112basic_stringIcNS0_11char_traitsIcEENS0_9allocatorIcEEEE", "pthread_getspecific", "sprintf", "getenv", "pread", "memcpy", "j__ZSt14get_unexpectedv", "pthread_once", "j__Unwind_GetIP", "strlen", "pthread_key_create", "strtol", "j___register_frame_info_table_bases", "__stack_chk_fail", "mmap", "ptrace", "strcasecmp", "getuid", "start", "sub_11603C", "sub_1160B4", "sub_1160C4", "sub_11615C", "sub_1161FC", "sub_1164AC", "sub_1164D8", "sub_116528", "sub_1165C8", "sub_1166FC", "sub_116728", "sub_116750", "sub_116830", "sub_1168F0", "sub_116A80", "sub_116AD8", "sub_116AF4", "sub_116B10", "sub_116B2C", "sub_116B84", "sub_116BA0", "sub_1178E8", "sub_1178FC", "sub_117B68", "sub_117BE0", "sub_117CA0", "sub_117E90", "sub_1180CC", "sub_1182D8", "sub_11859C", "sub_1186B8", "sub_118748", "sub_118AE4", "sub_118C8C", "sub_118D24", "sub_118FB4", "sub_119270", "sub_11959C", "sub_119634", "sub_11BC1C", "JNI_OnLoad", "sub_11BD9C", "sub_11BDA8", "sub_11BE58", "sub_11C1AC", "sub_11C1B4", "sub_11C210", "sub_11C268", "sub_11C300", "sub_11C394", "sub_11CAA4", "sub_11CC18", "sub_11CF8C", "sub_11D134", "sub_11D4C4", "sub_11D694", "nullsub_8", "sub_11D73C", "sub_11E0D8", "sub_11E834", "sub_11F130", "sub_11F1C8", "sub_11F260", "sub_11F3A4", "sub_11F56C", "sub_11F604", "sub_11F69C", "sub_11F768", "sub_11F884", "sub_11FA18", "sub_11FB14", "sub_1200C8", "sub_120194", "sub_1202D4", "sub_12036C", "sub_1203C8", "sub_120404", "sub_120440", "sub_12047C", "sub_1205A0", "sub_120664", "nullsub_9", "sub_120678", "sub_120C30", "sub_121B78", "sub_121BE0", "sub_121C40", "sub_121CD8", "sub_122180", "sub_1222D0", "sub_1222E8", "sub_122AE0", "sub_122B54", "sub_122B5C", "sub_122C7C", "sub_122D14", "sub_123324", "sub_1236B8", "sub_123720", "sub_123970", "sub_123BC0", "sub_123CE8", "sub_124510", "sub_124710", "sub_124900", "sub_124C68", "sub_124F7C", "sub_124FA0", "sub_1253B8", "sub_125650", "sub_125854", "sub_125D2C", "sub_125DC4", "sub_126888", "sub_126920", "sub_126ED8", "sub_126F84", "sub_12743C", "sub_127DCC", "sub_127EA8", "sub_1281E4", "sub_1281EC", "sub_1288DC", "sub_128A2C", "sub_128D44", "sub_128D64", "sub_128D70", "sub_128DB8", "sub_128E60", "sub_128E88", "sub_128F5C", "sub_129468", "sub_129758", "sub_129A04", "sub_12A568", "sub_12AC10", "sub_12AD6C", "sub_12AEB0", "sub_12B9FC", "sub_12BB18", "sub_12BB80", "sub_12BF4C", "sub_12C08C", "sub_12C5D4", "sub_12CC6C", "sub_12D2F8", "sub_12D470", "sub_12D478", "sub_12D480", "sub_12D48C", "sub_12D4A4", "sub_12D4AC", "sub_12D4D8", "sub_12D518", "sub_12D560", "sub_12D84C", "sub_12D9FC", "sub_12DB60", "sub_12DFE4", "sub_12F9F8", "sub_12FA34", "sub_12FA6C", "sub_12FB00", "sub_12FB18", "sub_1317A4", "sub_131958", "sub_131D58", "sub_131E20", "sub_132088", "sub_132518", "sub_132DA8", "sub_1332B8", "sub_136D68", "sub_136DB0", "sub_136FF8", "sub_137164", "sub_137408", "sub_137630", "sub_137820", "sub_137978", "sub_1395A8", "sub_1395C4", "sub_1395EC", "sub_139604", "sub_139628", "sub_139664", "sub_139678", "sub_139698", "sub_1396A0", "sub_1396B4", "sub_1396D0", "sub_1396FC", "sub_13970C", "sub_139728", "sub_139770", "sub_139824", "sub_139828", "sub_13982C", "sub_1398F0", "sub_1399D8", "sub_139ADC", "sub_139BAC", "sub_139BB8", "sub_139CE4", "sub_139CFC", "sub_139D18", "sub_139D40", "sub_139D4C", "sub_139D50", "sub_139D8C", "sub_139DAC", "sub_139DF4", "sub_139E0C", "sub_139E40", "sub_139E7C", "sub_13A4D4", "sub_13A4E4", "sub_13A4FC", "sub_13A53C", "sub_13A55C", "sub_13A588", "sub_13A5A8", "sub_13A5D4", "sub_13A5E4", "sub_13A600", "sub_13A61C", "sub_13A664", "sub_13A670", "sub_13A684", "sub_13A6B4", "sub_13A6D0", "sub_13A79C", "sub_13A7AC", "sub_13A7E0", "sub_13A7E8", "sub_13A808", "sub_13A810", "sub_13A82C", "sub_13A870", "sub_13A894", "sub_13A8AC", "sub_13A8DC", "sub_13A920", "sub_13A95C", "sub_13A9A4", "sub_13A9B4", "sub_13A9D0", "sub_13AA00", "sub_13AA0C", "sub_13AA50", "sub_13AA74", "sub_13AB40", "sub_13B1CC", "sub_13B1D8", "sub_13BC98", "sub_13BCB0", "sub_13BD80", "sub_13BE14", "sub_13BE2C", "sub_13BE4C", "sub_13BE64", "sub_13BE80", "sub_13BEA0", "sub_13BEB8", "sub_13BF30", "sub_13BF50", "sub_13BF70", "sub_13BF90", "sub_13BFB0", "sub_13BFD0", "sub_13BFF0", "sub_13C064", "sub_13C07C", "sub_13C09C", "sub_13C12C", "sub_13C13C", "sub_13C1E4", "sub_13C238", "sub_13C2A8", "sub_13C2F0", "strcmp", "sub_13C384", "sub_13C3CC", "sub_13C4EC", "sub_13C4F0", "sub_13C544", "sub_13D114", "sub_13D288", "sub_13D2EC", "sub_13D30C", "sub_13D310", "sub_13D328", "sub_13D344", "sub_13D38C", "sub_13D394", "sub_13D39C", "sub_13D3A4", "sub_13D460", "sub_13D480", "sub_13D484", "sub_13D49C", "sub_13D4A4", "sub_13D4AC", "sub_13D4C8", "sub_13D538", "sub_13D5D8", "sub_13D5E4", "sub_13D70C", "sub_13DB7C", "sub_13DEA8", "sub_13DF34", "sub_13E0EC", "sub_13E0F8", "sub_13E510", "sub_13E654", "sub_13E784", "sub_13E9AC", "sub_13EAE0", "sub_13EC30", "sub_13EE08", "sub_13EF2C", "sub_13F888", "sub_13F920", "sub_14027C", "sub_140A3C", "sub_1411FC", "sub_1412C0", "sub_1415F0", "sub_141924", "sub_1427E8", "sub_14283C", "sub_14285C", "sub_142894", "sub_1428BC", "sub_14292C", "sub_142954", "sub_1429B8", "sub_1429C0", "sub_1429CC", "sub_142A10", "sub_142AA4", "sub_142ABC", "sub_142EA0", "sub_142FE0", "sub_143008", "sub_143088", "sub_14316C", "sub_143184", "sub_143568", "sub_143848", "sub_143964", "sub_1439FC", "sub_143A38", "sub_143B48", "sub_143C54", "sub_143D24", "sub_143D3C", "sub_143EBC", "sub_1440CC", "sub_14423C", "sub_144360", "sub_1444B8", "sub_144508", "sub_14453C", "sub_144774", "sub_14495C", "sub_144AFC", "sub_144EF0", "sub_144F4C", "sub_14500C", "sub_145018", "sub_145024", "sub_145030", "sub_14503C", "sub_145040", "sub_145078", "sub_1450A4", "sub_145148", "sub_145154", "sub_14516C", "sub_145184", "sub_14519C", "sub_1451B4", "sub_1451CC", "sub_1451E4", "sub_1451FC", "sub_145214", "sub_145228", "sub_145268", "sub_145280", "sub_145298", "sub_1452B0", "sub_1452C8", "sub_1452E0", "sub_1452F8", "sub_145310", "sub_14AA44", "sub_14AA50", "sub_14AA88", "sub_14AAAC", "sub_14AACC", "sub_14AAF0", "sub_14AB14", "sub_14AB34", "sub_14AB68", "sub_14AB78", "sub_14ABB0", "sub_14ABC0", "sub_14ABCC", "sub_14ABF4", "sub_14AC1C", "sub_14AC28", "sub_14AC60", "sub_14AC90", "sub_14AD28", "sub_14ADB8", "sub_14AE70", "sub_14AE7C", "sub_14AE88", "sub_14AEC4", "sub_14AEEC", "sub_14B01C", "sub_14B028", "sub_14B054", "sub_14B090", "sub_14B09C", "sub_14B0BC", "sub_14B104", "sub_14B164", "sub_14B198", "sub_14B1BC", "sub_14B1DC", "sub_14B204", "sub_14B240", "sub_14B250", "sub_14B278", "sub_14B2A8", "sub_14B2E0", "sub_14B304", "sub_14B324", "sub_14B36C", "sub_14B3CC", "sub_14B408", "sub_14B430", "sub_14B46C", "sub_14B4A8", "sub_14B4B4", "sub_14B4EC", "sub_14B4FC", "sub_14B57C", "sub_14B5C4", "sub_14B5D0", "sub_14B638", "sub_14B65C", "sub_14B67C", "sub_14B6F4", "sub_14B740", "sub_14B7A8", "sub_14B7D8", "sub_14B808", "sub_14B814", "sub_14B820", "sub_14B82C", "sub_14B854", "sub_14B868", "sub_14B8D8", "sub_14B908", "sub_14B920", "sub_14B998", "sub_14B9D0", "sub_14B9E0", "sub_14BA00", "sub_14BA0C", "sub_14BA58", "sub_14BA98", "sub_14BAE0", "sub_14BB40", "sub_14BB64", "sub_14BB84", "sub_14BB90", "sub_14BC44", "sub_14BC78", "sub_14BD10", "sub_14BD1C", "sub_14BD28", "sub_14BD4C", "sub_14BD68", "sub_14BD6C", "sub_14BD90", "sub_14BDA0", "sub_14BF9C", "sub_14C18C", "sub_14C198", "sub_14C1A8", "sub_14C1F4", "sub_14C200", "sub_14C24C", "sub_14C25C", "sub_14C2A8", "sub_14C2B4", "sub_14C4B0", "sub_14C75C", "sub_14C7C8", "sub_14C848", "sub_14C84C", "sub_14C898", "sub_14C960", "sub_14C978", "sub_14C984", "sub_14C990", "sub_14C9DC", "sub_14C9E8", "sub_14CA30", "sub_14CB2C", "sub_14CD80", "sub_14CDCC", "sub_14CDF4", "sub_14CE04", "sub_14CE50", "sub_14CE5C", "sub_14CE68", "sub_14CF64", "sub_14D17C", "sub_14D1E8", "sub_14D200", "sub_14D214", "sub_14D234", "sub_14D26C", "sub_14D27C", "sub_14D2C8", "sub_14D390", "sub_14D3BC", "sub_14D3E4", "sub_14D3F8", "sub_14D418", "sub_14D440", "sub_14D458", "sub_14D474", "sub_14D570", "sub_14D7E8", "sub_14D810", "sub_14D820", "sub_14D86C", "sub_14D87C", "sub_14D8A0", "sub_14D8B0", "sub_14D8F8", "sub_14D9F4", "sub_14DA04", "sub_14DA08", "sub_14DA54", "sub_14DA60", "sub_14DC5C", "sub_14DDC4", "sub_14DE34", "sub_14DEA8", "sub_14DEB8", "sub_14DEDC", "sub_14DEEC", "sub_14DF3C", "sub_14DFAC", "sub_14E000", "sub_14E004", "sub_14E008", "sub_14E030", "sub_14E0FC", "sub_14E114", "sub_14E128", "sub_14E148", "sub_14E158", "sub_14E17C", "sub_14E18C", "sub_14E1B0", "sub_14E1BC", "sub_14E208", "sub_14E214", "sub_14E238", "sub_14E258", "sub_14E280", "sub_14E34C", "sub_14E364", "sub_14E460", "sub_14E470", "sub_14E494", "sub_14E4A4", "sub_14E4C8", "sub_14E4D0", "sub_14E4DC", "sub_14E5DC", "sub_14E718", "sub_14E728", "sub_14E74C", "sub_14E75C", "sub_14E780", "sub_14E854", "sub_14E8AC", "sub_14E8D4", "sub_14E8EC", "sub_14E8F8", "sub_14E904", "sub_14E94C", "sub_14E958", "sub_14EA54", "sub_14EC90", "sub_14EC9C", "sub_14ECC0", "sub_14ECD0", "sub_14EECC", "sub_14EED8", "sub_14EFD4", "sub_14F228", "sub_14F234", "sub_14F258", "sub_14F268", "sub_14F274", "sub_14F280", "sub_14F2A4", "sub_14F2AC", "sub_14F2B8", "sub_14F304", "sub_14F3D4", "sub_14F448", "sub_14F458", "sub_14F47C", "sub_14F480", "sub_14F484", "sub_14F4A8", "sub_14F6A4", "sub_14F7D4", "sub_14F840", "sub_14F86C", "sub_14F894", "sub_14F8A8", "sub_14F8C8", "sub_14F8D8", "sub_14F8FC", "sub_14F90C", "sub_14F918", "sub_14F964", "sub_14FA2C", "sub_14FA44", "sub_14FA58", "sub_14FA78", "sub_14FA88", "sub_14FAAC", "sub_14FAB4", "sub_14FAD4", "sub_14FAD8", "sub_14FB24", "sub_14FBEC", "sub_14FC28", "sub_14FC3C", "sub_14FC5C", "sub_14FC84", "sub_14FC88", "sub_14FC8C", "sub_14FCB0", "sub_14FCFC", "sub_14FD64", "sub_14FD90", "sub_14FDB8", "sub_14FDCC", "sub_14FDEC", "sub_14FE3C", "sub_14FE60", "sub_14FE70", "sub_14FE7C", "sub_14FE88", "sub_14FED4", "sub_14FEDC", "sub_14FEF4", "sub_14FF18", "sub_14FF28", "sub_14FF4C", "sub_14FF50", "sub_14FF54", "sub_14FF78", "sub_14FF9C", "sub_150008", "sub_15006C", "sub_150094", "sub_1500AC", "sub_1500B8", "sub_1500C4", "sub_1500E8", "sub_1501B4", "sub_1501CC", "sub_1501E0", "sub_150200", "sub_150250", "sub_15025C", "sub_1502AC", "sub_1502B8", "sub_1502E4", "sub_1502EC", "sub_1502F0", "sub_150340", "sub_150580", "sub_15058C", "sub_150808", "sub_150814", "sub_150930", "sub_150940", "sub_15094C", "sub_150A6C", "sub_150B94", "sub_150C64", "sub_150CA0", "sub_150CB4", "sub_150CD8", "sub_150CE8", "sub_150D0C", "sub_150D38", "sub_150D44", "sub_150F84", "sub_1510DC", "sub_1510EC", "sub_1510FC", "sub_151338", "sub_151488", "sub_1514F4", "sub_151570", "sub_151598", "sub_1515A8", "sub_1516C8", "sub_151968", "sub_1519D4", "sub_1519FC", "sub_151A54", "sub_151A64", "sub_151CA0", "sub_151F10", "sub_151F5C", "sub_151FC4", "sub_151FEC", "sub_15210C", "sub_15224C", "sub_1522B8", "sub_1522E4", "sub_15230C", "sub_152320", "sub_152340", "sub_152368", "sub_152378", "sub_1523E4", "sub_15240C", "sub_15244C", "sub_152450", "sub_152474", "sub_152480", "sub_15248C", "sub_15249C", "sub_1524A8", "sub_152574", "sub_15259C", "sub_1525C4", "sub_1525E8", "sub_1525F8", "sub_15261C", "sub_15262C", "sub_152648", "sub_152658", "sub_1526C0", "sub_1526E8", "sub_1527B4", "sub_1527DC", "sub_1527F8", "sub_15284C", "sub_152858", "sub_152860", "sub_152880", "sub_152890", "sub_1528DC", "sub_1528E4", "sub_152904", "sub_15292C", "sub_15293C", "sub_152960", "sub_15297C", "sub_1529A0", "sub_1529A4", "sub_1529A8", "sub_1529CC", "sub_1529E8", "sub_152A2C", "sub_152A34", "sub_152A58", "sub_152A78", "sub_152A90", "sub_152B0C", "sub_152B34", "sub_152B44", "sub_152B68", "sub_152B80", "sub_152BBC", "sub_152BD0", "sub_152BF0", "sub_152BF4", "sub_152BF8", "sub_152C1C", "sub_152C20", "sub_152C5C", "sub_152C98", "sub_152CAC", "sub_152CCC", "sub_152CE4", "sub_152D00", "sub_152D10", "sub_152D34", "sub_152D40", "sub_152D5C", "sub_152E2C", "sub_152E84", "sub_152EAC", "sub_152EE0", "sub_153014", "sub_15303C", "sub_153058", "sub_153074", "sub_1530DC", "sub_1530F4", "sub_1530FC", "sub_15311C", "sub_15312C", "sub_153150", "sub_153160", "sub_1531A0", "sub_1531C4", "sub_1531CC", "sub_153218", "sub_153280", "sub_153298", "sub_1532AC", "sub_1532CC", "sub_1532D0", "sub_1532D4", "sub_1532F8", "sub_153308", "sub_153354", "sub_153370", "sub_153398", "sub_1533BC", "sub_1534A8", "sub_1534DC", "sub_1535B0", "sub_153608", "sub_153630", "sub_153634", "sub_153664", "sub_153680", "sub_15368C", "sub_1536AC", "sub_153720", "sub_153730", "sub_153754", "sub_153764", "sub_153784", "sub_153790", "sub_1537A8", "sub_1537C0", "sub_1537E8", "sub_153804", "sub_153840", "sub_153868", "sub_1538AC", "sub_1538CC", "sub_1538F0", "sub_1539BC", "sub_1539D4", "sub_1539E8", "sub_153A08", "sub_153A0C", "sub_153A10", "sub_153A34", "sub_153A44", "sub_153A68", "sub_153A78", "sub_153AB8", "sub_153ADC", "sub_153AF4", "sub_153AFC", "sub_153B1C", "sub_153B54", "sub_153B64", "sub_153BA8", "sub_153DC0", "sub_153DCC", "sub_153E18", "sub_153E44", "sub_153E74", "sub_153EA0", "sub_153EC4", "sub_153EF0", "sub_153F00", "sub_153F44", "sub_153F60", "sub_153F68", "sub_153F6C", "sub_153F84", "sub_153F9C", "sub_153FA4", "sub_153FB4", "sub_153FC8", "sub_153FD4", "sub_153FE8", "sub_15400C", "sub_154040", "sub_154058", "sub_154088", "sub_15409C", "sub_1540A8", "sub_1540B4", "sub_1540D0", "sub_1540DC", "sub_1540F0", "sub_154110", "sub_154138", "sub_1541F0", "sub_154238", "sub_154244", "sub_154250", "sub_15426C", "sub_154278", "sub_1544B8", "sub_1544C4", "sub_1544E8", "sub_15455C", "sub_15458C", "sub_1545B8", "sub_1545DC", "sub_154610", "sub_154614", "sub_154628", "sub_154640", "sub_15464C", "sub_154670", "sub_15467C", "sub_154698", "sub_154760", "sub_154778", "sub_154780", "sub_1547A0", "sub_1547B8", "sub_1547D4", "sub_15481C", "sub_154898", "sub_154FD0", "sub_154FF4", "sub_155010", "sub_155048", "sub_155058", "sub_1550C0", "sub_1550E8", "sub_155104", "sub_155128", "sub_155194", "sub_1551D0", "sub_1551E4", "sub_155204", "sub_155220", "sub_155240", "sub_155250", "sub_155294", "sub_1553F0", "sub_155458", "sub_155470", "sub_155484", "sub_1554A4", "sub_1554CC", "sub_1554F8", "sub_155644", "sub_1556B0", "sub_1556EC", "sub_155700", "sub_155720", "sub_155748", "sub_155788", "sub_1557B0", "sub_1557D8", "sub_1557E8", "sub_1557F4", "sub_155828", "sub_155834", "sub_15584C", "sub_155888", "sub_15589C", "sub_1558BC", "sub_1558D4", "sub_1558F0", "sub_155900", "sub_155918", "sub_15597C", "sub_1559A4", "sub_1559BC", "sub_1559F4", "sub_155A20", "sub_155A94", "sub_155ABC", "sub_155AE8", "sub_155F04", "sub_155F90", "sub_156888", "sub_1568C8", "sub_1569FC", "sub_156A1C", "sub_156A44", "sub_156A6C", "sub_156AB0", "sub_156AC0", "sub_156AE8", "sub_156B0C", "sub_156BD8", "sub_156BF0", "sub_156C04", "sub_156C24", "sub_156C28", "sub_156C2C", "sub_156C50", "sub_156C80", "sub_156CC0", "sub_156CE4", "sub_156CF8", "sub_156D1C", "sub_156D2C", "sub_156D50", "sub_156D6C", "sub_156DDC", "sub_156E34", "sub_156E5C", "sub_156E6C", "sub_156E90", "sub_156EAC", "sub_156F44", "sub_156F50", "sub_15704C", "sub_1572A0", "sub_1572EC", "sub_1572F8", "sub_157414", "sub_157548", "sub_157560", "sub_157578", "sub_15758C", "sub_1575AC", "sub_1575B0", "sub_1575B4", "sub_1575D8", "sub_157608", "sub_15762C", "sub_157690", "sub_1576B8", "sub_1576C0", "sub_1576D8", "sub_157704", "sub_157728", "sub_15777C", "sub_15784C", "sub_1578BC", "sub_1578D8", "sub_1578F8", "sub_157900", "sub_15790C", "sub_15793C", "sub_157978", "sub_15798C", "sub_1579AC", "sub_1579BC", "sub_1579E0", "sub_1579F0", "sub_157A2C", "sub_157BE4", "sub_157C20", "sub_157C34", "sub_157C54", "sub_157C6C", "sub_157CCC", "sub_157D3C", "sub_157D6C", "sub_157D98", "sub_157DBC", "sub_157DE8", "sub_157DF8", "sub_157E38", "sub_157FC4", "sub_15801C", "sub_15802C", "sub_158050", "sub_158060", "sub_1580A0", "sub_158168", "sub_158180", "sub_158188", "sub_1581A8", "sub_1581C0", "sub_1581DC", "sub_1581F8", "sub_158228", "sub_158380", "sub_1583BC", "sub_1583D4", "sub_1583E8", "sub_158408", "sub_158430", "sub_158448", "sub_158464", "sub_158480", "sub_1584E8", "sub_158564", "sub_158568", "sub_15856C", "sub_158590", "sub_158594", "sub_158598", "sub_1585BC", "sub_1585D8", "sub_158640", "sub_158658", "sub_15866C", "sub_15868C", "sub_158690", "sub_158694", "sub_1586B8", "sub_1586BC", "sub_1586C0", "sub_158700", "sub_15870C", "sub_15872C", "sub_158784", "sub_1587F0", "sub_158858", "sub_158894", "sub_1588A8", "sub_1588C8", "sub_1588CC", "sub_1588D0", "sub_15891C", "sub_158934", "sub_158A9C", "sub_158AC0", "sub_158ADC", "sub_158AE4", "sub_158AF0", "sub_158B08", "sub_158B30", "sub_158B58", "sub_158BBC", "sub_158C78", "sub_158D44", "sub_158D6C", "sub_158D94", "sub_158DAC", "sub_158F0C", "sub_158F48", "sub_158F70", "sub_158FB0", "sub_158FDC", "sub_159004", "sub_159018", "sub_159038", "sub_159048", "sub_159094", "sub_1590B0", "sub_1590BC", "sub_1590DC", "sub_15910C", "sub_159138", "sub_15915C", "sub_159164", "sub_159168", "sub_15918C", "sub_1591A4", "sub_1591C0", "sub_1591D8", "sub_159364", "sub_1593CC", "sub_1593E4", "sub_1593EC", "sub_15940C", "sub_159410", "sub_159414", "sub_159478", "sub_1594A0", "sub_1594BC", "sub_159520", "sub_159714", "sub_159730", "sub_159754", "sub_1597BC", "sub_1597C0", "sub_1597E4", "sub_1597FC", "sub_159818", "sub_159838", "sub_159868", "sub_159894", "sub_1598B8", "sub_1598CC", "sub_1598F4", "sub_159908", "sub_15994C", "sub_1599B4", "sub_1599CC", "sub_1599E0", "sub_159A00", "sub_159A28", "sub_159A40", "sub_159A78", "sub_159AE0", "sub_159B5C", "sub_159B60", "sub_159B64", "sub_159B88", "sub_159B98", "sub_159BBC", "sub_159BDC", "sub_159C4C", "sub_159CA0", "sub_159CB8", "sub_159CD8", "sub_159CF4", "sub_159D2C", "sub_159D7C", "sub_159D88", "sub_159DA0", "sub_159DC8", "sub_159DE4", "sub_159DFC", "sub_159F14", "sub_159F38", "sub_159F50", "sub_159F7C", "sub_159FA4", "sub_159FAC", "sub_15A018", "sub_15A040", "sub_15A068", "sub_15A08C", "sub_15A0A4", "sub_15A0DC", "sub_15A104", "sub_15A114", "sub_15A138", "sub_15A200", "sub_15A228", "sub_15A25C", "sub_15A424", "sub_15A460", "sub_15A49C", "sub_15A4B0", "sub_15A4D0", "sub_15A4D4", "sub_15A4D8", "sub_15A4FC", "sub_15A500", "sub_15A504", "sub_15A528", "sub_15A540", "sub_15A57C", "sub_15A590", "sub_15A5B0", "sub_15A5C0", "sub_15A5E4", "sub_15A5F4", "sub_15A618", "sub_15A630", "sub_15A664", "sub_15A698", "sub_15A6D8", "sub_15A6F8", "sub_15A724", "sub_15A75C", "sub_15A7A0", "sub_15A7C0", "sub_15A7EC", "sub_15A7FC", "sub_15A818", "sub_15A860", "sub_15A8BC", "sub_15A904", "sub_15A960", "sub_15A9A8", "sub_15A9F0", "sub_15AA4C", "sub_15AA68", "sub_15AA94", "sub_15ABB0", "sub_15ACD8", "sub_15ACE4", "sub_15ACFC", "sub_15AD18", "sub_15AD40", "sub_15ADE4", "sub_15AE2C", "sub_15AE88", "sub_15AE94", "sub_15AEAC", "sub_15AED8", "sub_15AF10", "sub_15B014", "sub_15B124", "sub_15B130", "sub_15B148", "sub_15B190", "sub_15B1EC", "sub_15B200", "sub_15B220", "sub_15B22C", "sub_15B244", "sub_15B28C", "sub_15B2E8", "sub_15B348", "sub_15B390", "sub_15B3EC", "sub_15B418", "sub_15B450", "sub_15B524", "sub_15B604", "sub_15B64C", "sub_15B6A8", "sub_15B6F8", "sub_15B758", "sub_15B83C", "sub_15B92C", "sub_15B974", "sub_15B9D0", "sub_15B9E8", "sub_15BA2C", "sub_15BA88", "sub_15BA94", "sub_15BAB0", "sub_15BAE4", "sub_15BB24", "sub_15BB3C", "sub_15BB60", "sub_15BB6C", "sub_15BB88", "sub_15BB94", "sub_15BBB0", "sub_15BBC8", "sub_15BBEC", "sub_15BC40", "sub_15BD74", "sub_15BEB4", "sub_15BEFC", "sub_15BF54", "sub_15BF9C", "sub_15BFF4", "sub_15C104", "sub_15C220", "sub_15C22C", "sub_15C244", "sub_15C25C", "sub_15C280", "sub_15C3B0", "sub_15C4EC", "sub_15C534", "sub_15C590", "sub_15C59C", "sub_15C5B4", "sub_15C6BC", "sub_15C7D0", "sub_15C8B4", "sub_15C9A4", "sub_15CAA8", "sub_15CBB8", "sub_15CC00", "sub_15CC5C", "sub_15CD8C", "sub_15CEC8", "sub_15CF10", "sub_15CF6C", "sub_15CFB4", "sub_15D010", "sub_15D01C", "sub_15D034", "sub_15D07C", "sub_15D0D8", "sub_15D0F4", "sub_15D138", "sub_15D270", "sub_15D3B4", "sub_15D3C0", "sub_15D3D8", "sub_15D420", "sub_15D47C", "sub_15D494", "sub_15D4B8", "sub_15D59C", "sub_15D68C", "sub_15D698", "sub_15D6B0", "sub_15D794", "sub_15D884", "sub_15D928", "sub_15D964", "sub_15D9B0", "sub_15DAC8", "sub_15DBEC", "sub_15DC34", "sub_15DC90", "sub_15DC9C", "sub_15DCB4", "sub_15DCF0", "sub_15DD38", "sub_15DD94", "sub_15DDA0", "sub_15DDB8", "sub_15DE00", "sub_15DE5C", "sub_15DE80", "sub_15DEB0", "sub_15DEF8", "sub_15DF54", "sub_15DF74", "sub_15DFA0", "sub_15DFE8", "sub_15E044", "sub_15E050", "sub_15E068", "sub_15E088", "sub_15E0B4", "sub_15E0D8", "sub_15E120", "sub_15E17C", "sub_15E188", "sub_15E1A0", "sub_15E1C0", "sub_15E1EC", "sub_15E234", "sub_15E290", "sub_15E2D8", "sub_15E334", "sub_15E340", "sub_15E358", "sub_15E364", "sub_15E37C", "sub_15E388", "sub_15E3A0", "sub_15E3E8", "sub_15E444", "sub_15E450", "sub_15E468", "sub_15E488", "sub_15E4B4", "sub_15E4C0", "sub_15E4D8", "sub_15E4E4", "sub_15E4FC", "sub_15E544", "sub_15E5A0", "sub_15E5C0", "sub_15E5EC", "sub_15E60C", "sub_15E638", "sub_15E680", "sub_15E6DC", "sub_15E724", "sub_15E780", "sub_15E78C", "sub_15E7A4", "sub_15E7EC", "sub_15E848", "sub_15E868", "sub_15E894", "sub_15E8A0", "sub_15E8B8", "sub_15E900", "sub_15E95C", "sub_15E968", "sub_15E980", "sub_15E9C8", "sub_15EA24", "sub_15EA48", "sub_15EA78", "sub_15EA84", "sub_15EA9C", "sub_15EAE4", "sub_15EB40", "sub_15EB88", "sub_15EBE4", "sub_15EBF0", "sub_15EC08", "sub_15EC14", "sub_15EC2C", "sub_15EC38", "sub_15EC50", "sub_15EC78", "sub_15ECAC", "sub_15ECF4", "sub_15ED48", "sub_15ED90", "sub_15EDE4", "sub_15EDF0", "sub_15EE08", "sub_15EE50", "sub_15EEAC", "sub_15EEF4", "sub_15EF48", "sub_15EF90", "sub_15EFEC", "sub_15F034", "sub_15F088", "sub_15F134", "sub_15F140", "sub_15F158", "sub_15F1AC", "sub_15F20C", "sub_15F2A8", "sub_15F2F0", "sub_15F344", "sub_15F398", "sub_15F3F8", "sub_15F41C", "sub_15F44C", "sub_15F458", "sub_15F470", "sub_15F4B8", "sub_15F514", "sub_15F548", "sub_15F588", "sub_15F5C0", "sub_15F604", "sub_15F63C", "sub_15F680", "sub_15F6BC", "sub_15F704", "sub_15F74C", "sub_15F7A8", "sub_15F7EC", "sub_15F83C", "sub_15F858", "sub_15F938", "sub_15FA54", "sub_15FB7C", "sub_15FB88", "sub_15FBA0", "sub_15FBC0", "sub_15FBF4", "sub_15FC14", "sub_15FC34", "sub_15FCA8", "sub_15FD08", "sub_15FD24", "sub_15FD44", "sub_15FD64", "sub_15FD84", "sub_15FDA4", "sub_15FDC4", "sub_15FDE4", "sub_15FE04", "sub_15FE24", "sub_160108", "sub_160128", "sub_160148", "sub_160168", "sub_160188", "sub_1601A8", "sub_1601C8", "sub_160228", "sub_1602C4", "sub_1602FC", "sub_16031C", "sub_16033C", "sub_16035C", "sub_16037C", "sub_16039C", "sub_1603BC", "sub_1603DC", "sub_1603FC", "sub_16041C", "sub_16043C", "sub_16045C", "sub_16047C", "sub_16049C", "sub_1604BC", "sub_1604DC", "sub_1604FC", "sub_16051C", "sub_16053C", "sub_16055C", "sub_16057C", "sub_16059C", "sub_1605BC", "sub_1605DC", "sub_1605FC", "sub_16061C", "sub_16063C", "sub_16065C", "sub_16067C", "sub_16069C", "sub_1606BC", "sub_1606DC", "sub_1606FC", "sub_16071C", "sub_16073C", "sub_16075C", "sub_16077C", "sub_16079C", "sub_1607BC", "sub_1607DC", "sub_1607FC", "sub_16081C", "sub_16083C", "sub_16085C", "sub_16087C", "sub_16089C", "sub_1608BC", "sub_1608DC", "sub_1608FC", "sub_16091C", "sub_16093C", "sub_16095C", "sub_16097C", "sub_16099C", "sub_1609BC", "sub_160D7C", "sub_1635E4", "sub_163624", "sub_163664", "sub_1636A4", "sub_1636E4", "sub_1638E0", "sub_163ADC", "sub_163CE4", "sub_163EE8", "sub_165E60", "sub_165EEC", "sub_165F38", "sub_165FA8", "sub_165FDC", "sub_165FFC", "sub_16601C", "sub_16603C", "sub_1660D4", "sub_1660F4", "sub_166114", "sub_166134", "sub_1661CC", "sub_166200", "sub_166234", "sub_166278", "sub_166298", "sub_1662B8", "sub_1662D8", "sub_166DD4", "sub_1679DC", "sub_1679E0", "sub_167A00", "sub_167A7C", "sub_167B14", "sub_167B30", "sub_167BC8", "sub_167BFC", "sub_167C3C", "sub_167CA4", "sub_167E68", "sub_167E88", "sub_167E98", "sub_167F30", "sub_167F54", "sub_167F98", "sub_167FB8", "sub_167FD8", "sub_167FF8", "sub_168018", "sub_168038", "sub_168058", "sub_168078", "sub_168098", "sub_1680B8", "sub_1680D8", "sub_1680F8", "sub_168118", "sub_168138", "sub_168158", "sub_168178", "sub_168198", "sub_1681B8", "sub_1681D8", "sub_1681F8", "sub_168218", "sub_168238", "sub_168258", "sub_168278", "sub_168298", "sub_1682B8", "sub_1682D8", "sub_1682F8", "sub_168318", "sub_168338", "sub_168358", "sub_168378", "sub_168398", "sub_1683B8", "sub_1683D8", "sub_1683F8", "sub_168418", "sub_168438", "sub_168458", "sub_168478", "sub_168510", "sub_1685A8", "sub_168640", "sub_1686D8", "sub_168770", "sub_168808", "sub_1688A0", "sub_168A48", "sub_168AA8", "sub_168AF8", "sub_168B20", "sub_168BA8", "sub_168BF4", "sub_168CA0", "sub_168D54", "sub_168EA0", "sub_168F60", "sub_168FE8", "sub_16909C", "sub_169124", "sub_169168", "sub_1691B4", "sub_1691D4", "sub_1691F4", "sub_169214", "sub_169234", "sub_169254", "sub_169274", "sub_169294", "sub_1692B4", "sub_169380", "sub_1694B0", "sub_1696A0", "sub_1696C4", "sub_169714", "sub_16973C", "sub_169798", "sub_1697D8", "sub_169824", "sub_16984C", "sub_16988C", "sub_1698B4", "sub_1698F0", "sub_1698F8", "sub_169ABC", "sub_169C34", "sub_16AD88", "sub_16AFC0", "sub_16B6A4", "sub_16B8F0", "sub_16B92C", "sub_16B98C", "sub_16BB2C", "sub_16BC20", "sub_16BD74", "sub_16C170", "sub_16C2D0", "sub_16C430", "sub_16C90C", "sub_16C918", "sub_16CCF0", "sub_16CDC8", "sub_16CFB0", "sub_16D084", "sub_16D0D0", "sub_16D1D4", "sub_16D288", "sub_16D2E0", "nullsub_10", "j_j__ZdlPv_4", "sub_16D3C4", "sub_16D3E4", "sub_16D4DC", "sub_16D58C", "sub_16D5A0", "sub_16D684", "sub_16D908", "sub_16DB1C", "sub_16E478", "sub_16E98C", "sub_16E9F8", "sub_16EB34", "sub_16EC74", "sub_16EE9C", "sub_16F89C", "sub_16FA30", "sub_16FD18", "sub_16FF10", "sub_170268", "sub_17038C", "sub_1705FC", "sub_170668", "sub_17074C", "sub_170760", "sub_170974", "sub_170BB8", "sub_170C30", "sub_170FCC", "sub_17107C", "sub_171134", "sub_171BE8", "sub_1721A4", "_ZNSt3__112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEED1Ev", "sub_172508", "sub_172660", "sub_172968", "sub_172AA4", "sub_172D30", "sub_172D78", "sub_172DC0", "sub_172E08", "sub_172E50", "sub_172E98", "sub_173124", "sub_17314C", "sub_173174", "sub_17365C", "sub_173690", "sub_173774", "sub_17392C", "sub_173FBC", "sub_1742C8", "_ZNSt3__112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEEC1ERKS5_", "sub_174648", "sub_174730", "sub_174B58", "sub_174FE8", "sub_1752BC", "sub_175878", "sub_175C68", "_ZNKSt3__112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEE7compareEmmPKcm", "sub_176410", "_ZNSt3__112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEEC1ERKS5_mmRKS4_", "sub_1764F8", "sub_176D14", "sub_177174", "sub_1772F4", "sub_177488", "sub_178488", "sub_179314", "sub_179754", "sub_179BEC", "sub_179D20", "sub_179E78", "sub_179FD0", "sub_17A128", "sub_17A280", "sub_17A3D8", "sub_17A530", "sub_17A688", "sub_17A7E0", "sub_17A938", "sub_17AA90", "sub_17ABE8", "sub_17ACE8", "_ZNSt3__16vectorINS_12basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEEENS4_IS6_EEE26__swap_out_circular_bufferERNS_14__split_bufferIS6_RS7_EE", "sub_17AF14", "sub_17B4F4", "sub_17BF70", "sub_17C514", "sub_17CD74", "sub_17CE08", "sub_17D040", "sub_17D994", "sub_17DB3C", "sub_17DEC0", "sub_17E12C", "sub_17E6B0", "sub_17EA9C", "sub_17F5F8", "sub_17FB5C", "sub_17FC80", "sub_180060", "sub_18006C", "sub_180070", "sub_1800A4", "sub_1800C4", "sub_18019C", "sub_180218", "sub_180290", "sub_180298", "sub_1802A0", "sub_1802A8", "sub_1802B0", "sub_1802B8", "sub_1802C0", "sub_180348", "sub_180384", "sub_1803C8", "sub_180424", "sub_180460", "sub_180650", "sub_1806EC", "sub_18078C", "sub_180824", "sub_1808B8", "sub_180944", "sub_180980", "sub_181438", "sub_1815C0", "sub_181658", "sub_1816B0", "sub_182944", "sub_18298C", "sub_182ABC", "sub_182BF4", "sub_182C98", "sub_182CEC", "sub_182D88", "sub_183094", "sub_1832E4", "sub_1833E4", "sub_1834DC", "sub_183558", "sub_1837E0", "sub_1837FC", "sub_183890", "sub_183988", "sub_183A64", "sub_183BD4", "sub_183BF4", "sub_183C38", "sub_183D90", "sub_183D98", "sub_183F14", "sub_183F4C", "sub_183F9C", "sub_183FD4", "sub_184078", "sub_184400", "sub_184B74", "sub_184E48", "sub_18511C", "sub_185520", "sub_1863F4", "sub_186488", "sub_18658C", "sub_186598", "sub_1865A8", "sub_1865B8", "sub_1865DC", "sub_186600", "sub_186658", "sub_186688", "sub_186774", "sub_1867F8", "sub_186830", "sub_186874", "sub_186898", "sub_1868BC", "sub_186DF4", "sub_186E48", "sub_186E50", "sub_186E7C", "sub_186FDC", "sub_18716C", "sub_187184", "sub_18725C", "sub_18735C", "sub_1874A8", "sub_187858", "sub_187A14", "sub_187AAC", "sub_187DF8", "sub_187FD4", "sub_188144", "sub_1882A8", "sub_18840C", "sub_188570", "sub_1886D4", "sub_1888F0", "sub_188B0C", "sub_188BA4", "sub_188CC0", "sub_188D24", "sub_188F1C", "sub_188F44", "sub_188F84", "sub_188F94", "sub_18912C", "sub_1892E0", "sub_1893C8", "sub_1893D8", "sub_189654", "sub_189658", "sub_18965C", "sub_189660", "sub_1898B0", "sub_189918", "sub_18993C", "sub_189A64", "sub_189B10", "sub_189C18", "sub_189CB0", "sub_189D80", "sub_189E28", "sub_189FA4", "sub_189FA8", "sub_189FDC", "sub_189FFC", "sub_18A020", "sub_18A058", "sub_18A07C", "sub_18A0B4", "sub_18A0D8", "sub_18A110", "sub_18A134", "sub_18A16C", "sub_18A190", "sub_18A1C8", "sub_18A1EC", "sub_18A224", "sub_18A248", "sub_18A280", "sub_18A2A4", "sub_18A2B0", "sub_18A2B8", "sub_18A304", "sub_18A30C", "sub_18A358", "sub_18A35C", "sub_18A390", "sub_18A3B0", "sub_18A480", "sub_18A598", "sub_18A634", "sub_18A6BC", "sub_18A74C", "sub_18A7EC", "sub_18A8CC", "sub_18A9AC", "sub_18AAB0", "sub_18AB7C", "sub_18ABB0", "sub_18AD00", "sub_18AE24", "sub_18AF4C", "sub_18B028", "sub_18B074", "sub_18BAD4", "sub_18BBF8", "_ZNSt3__112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEE6appendIPcEENS_9enable_ifIXsrNS_21__is_forward_iteratorIT_EE5valueERS5_E4typeESA_SA_", "sub_18C024", "sub_18C620", "sub_18CE34", "sub_18D070", "sub_18D8C4", "sub_18D93C", "sub_18DA00", "interpreter_wrap_int64_t", "interpreter_wrap_float", "interpreter_wrap_double", "sub_18DC28", "sub_18DC4C", "sub_18DCC0", "nullsub_11", "j_j__ZdlPv_5", "sub_18DD94", "sub_18DDA8", "sub_18DDB8", "sub_18E244", "sub_18E8D0", "sub_18F6AC", "sub_18F8B0", "sub_18F8D8", "sub_18FB64", "sub_18FBB0", "sub_18FC9C", "sub_18FE18", "sub_18FEA8", "sub_1926F0", "sub_192C54", "sub_192CD4", "sub_192D54", "sub_192DA0", "sub_192E3C", "sub_192E78", "sub_192F10", "sub_192F4C", "sub_192FE8", "sub_193034", "sub_19307C", "sub_193274", "sub_1932AC", "sub_1932E4", "sub_19331C", "sub_1933FC", "sub_1934CC", "sub_1935BC", "sub_1937AC", "sub_19384C", "sub_193A2C", "sub_193BF4", "sub_193CEC", "sub_193DDC", "sub_194114", "sub_1942D0", "sub_194514", "sub_194758", "sub_194948", "sub_194B30", "sub_194E44", "sub_195008", "sub_1952E8", "sub_1953D8", "sub_1954D0", "sub_19570C", "sub_195CAC", "sub_195E6C", "sub_195FFC", "sub_196010", "nullsub_12", "j_j__ZdlPv_6", "sub_196168", "j_j__ZdlPv_7", "sub_1962BC", "j_j__ZdlPv_8", "sub_196434", "j_j__ZdlPv_9", "sub_19659C", "j_j__ZdlPv_10", "sub_196644", "sub_196B24", "sub_196C9C", "sub_196CC0", "sub_196CE0", "sub_196E58", "sub_196E94", "sub_196F08", "sub_19709C", "nullsub_13", "j_j__ZdlPv_11", "sub_1970C0", "sub_197144", "sub_197424", "sub_197478", "sub_197488", "sub_197498", "sub_1974D4", "sub_1974E4", "sub_1974F4", "sub_19766C", "sub_197898", "sub_197940", "sub_197960", "sub_19796C", "sub_197984", "sub_197994", "sub_197A7C", "sub_197D5C", "sub_197E44", "sub_197F48", "nullsub_14", "sub_197F54", "sub_197F5C", "sub_197FB0", "sub_198070", "sub_1982BC", "sub_1982C8", "sub_1982D0", "sub_1982D8", "sub_198300", "sub_19830C", "sub_1983E8", "sub_198408", "sub_198420", "sub_198708", "sub_198774", "sub_19885C", "sub_198B3C", "sub_198C24", "sub_198D28", "sub_198D30", "nullsub_15", "sub_198D40", "sub_198D48", "sub_198D9C", "sub_198E5C", "sub_19955C", "sub_199568", "sub_199570", "sub_199578", "sub_199688", "sub_199794", "sub_1997B4", "sub_1997C0", "sub_1997D8", "sub_1997E8", "sub_1998D0", "sub_199BB0", "sub_199C98", "sub_199D9C", "nullsub_16", "sub_199DA8", "sub_199DB0", "sub_199E04", "sub_199EC4", "sub_19A110", "sub_19A11C", "sub_19A124", "sub_19A12C", "sub_19A14C", "sub_19A158", "sub_19A170", "sub_19A3E0", "sub_19A44C", "sub_19A534", "sub_19A814", "sub_19A8FC", "sub_19AA00", "nullsub_17", "sub_19AA0C", "sub_19AA14", "sub_19AA68", "sub_19AB0C", "sub_19B568", "sub_19B574", "sub_19B57C", "sub_19B584", "sub_19B640", "sub_19B760", "sub_19B7EC", "sub_19B844", "sub_19B870", "sub_19BC9C", "sub_19C6E0", "sub_19C7AC", "sub_19C920", "sub_19CCD4", "sub_19CD44", "sub_19CD74", "sub_19D238", "sub_19D2CC", "sub_19D360", "sub_19D3E4", "sub_19D56C", "sub_19D750", "sub_19D894", "sub_19DA08", "sub_19DB48", "sub_19DCA4", "sub_19E088", "sub_19E110", "sub_19E508", "sub_19EA48", "sub_19ED64", "sub_19F0A0", "sub_19F358", "sub_19F500", "sub_19F608", "sub_19F67C", "sub_19F68C", "sub_19F6DC", "sub_19F70C", "sub_19F7C4", "sub_19F7EC", "sub_19F984", "sub_19FDD8", "sub_19FF98", "sub_1A0330", "sub_1A0418", "sub_1A0760", "sub_1A0C34", "sub_1A0C88", "sub_1A0E00", "sub_1A0F58", "nullsub_7", "sub_1A1114", "sub_1A11A0", "sub_1A12EC", "sub_1A16A4", "sub_1A171C", "sub_1A181C", "sub_1A1908", "sub_1A19EC", "sub_1A1A70", "sub_1A1AE8", "sub_1A1B64", "nullsub_18", "j_j__ZdlPv_12", "sub_1A1B7C", "sub_1A1C78", "nullsub_19", "j_j__ZdlPv_13", "sub_1A1C90", "sub_1A1D84", "sub_1A1E74", "sub_1A1EF0", "sub_1A2034", "sub_1A2108", "sub_1A2160", "sub_1A2378", "sub_1A2380", "sub_1A23CC", "sub_1A24C4", "sub_1A25E0", "sub_1A2718", "sub_1A2830", "sub_1A2938", "sub_1A2984", "sub_1A2A94", "sub_1A2BA0", "sub_1A2CAC", "sub_1A2DDC", "sub_1A3150", "sub_1A3204", "sub_1A3228", "sub_1A325C", "sub_1A3260", "sub_1A327C", "sub_1A32C0", "sub_1A3434", "sub_1A3438", "sub_1A3450", "sub_1A46B8", "sub_1A62E4", "sub_1A648C", "sub_1A6D4C", "sub_1A6F40", "sub_1A7DE0", "sub_1A7F58", "sub_1A8150", "sub_1A8168", "sub_1A8300", "sub_1A8308", "sub_1A847C", "sub_1A8728", "sub_1A88A4", "sub_1A898C", "sub_1A8AD4", "sub_1A8AEC", "sub_1A8C24", "sub_1A8C84", "sub_1A8E60", "sub_1A8F58", "sub_1A9074", "sub_1A9190", "sub_1A9248", "sub_1A92C0", "sub_1A9484", "sub_1A94A8", "sub_1A94B0", "sub_1A94B8", "sub_1A94EC", "sub_1A9558", "sub_1A9650", "sub_1A9670", "sub_1A96A0", "sub_1A9F9C", "sub_1A9FBC", "sub_1A9FD4", "sub_1AA7C4", "sub_1AA7F4", "sub_1AACFC", "sub_1AADA8", "sub_1AADC8", "sub_1AADD4", "sub_1AAE90", "sub_1AAF48", "sub_1AAF54", "sub_1AAF5C", "sub_1AAFF4", "sub_1AB088", "sub_1AB0FC", "sub_1AB16C", "sub_1AB220", "sub_1AB228", "sub_1AB230", "sub_1AB244", "sub_1AB24C", "sub_1AB2E4", "sub_1AB378", "sub_1AB410", "sub_1AB4A4", "sub_1AB518", "sub_1AB588", "sub_1AB5FC", "sub_1AB66C", "sub_1AB704", "sub_1AB798", "sub_1AB80C", "sub_1AB87C", "sub_1AB8DC", "sub_1AB950", "sub_1AB9C0", "sub_1ABA34", "sub_1ABAA4", "sub_1ABADC", "sub_1ABB10", "sub_1ABB84", "sub_1ABBF4", "sub_1ABC5C", "sub_1ABCC0", "sub_1ABCC4", "sub_1ABCD4", "sub_1ABDB4", "sub_1ABE90", "sub_1ABEB0", "sub_1ABF0C", "sub_1AC0D4", "sub_1AC108", "sub_1AC170", "sub_1AC1F0", "sub_1AC210", "sub_1AE394", "sub_1AE4C4", "sub_1AE4DC", "sub_1AE4E4", "sub_1AE4F8", "sub_1AE528", "sub_1AE540", "sub_1AE5B4", "sub_1AE624", "sub_1AE65C", "sub_1AE690", "sub_1AE6C8", "sub_1AE6FC", "sub_1AE714", "sub_1AE72C", "sub_1AE778", "sub_1AE790", "sub_1AF394", "sub_1AFC80", "sub_1AFE2C", "sub_1AFE4C", "sub_1AFE6C", "sub_1AFE8C", "sub_1AFE90", "sub_1AFF58", "sub_1B0108", "sub_1B0120", "sub_1B01DC", "sub_1B0294", "sub_1B02B4", "sub_1B03AC", "sub_1B04A0", "sub_1B04F0", "sub_1B053C", "sub_1B058C", "sub_1B05D8", "sub_1B06B8", "sub_1B0794", "sub_1B0868", "sub_1B0938", "sub_1B0944", "sub_1B095C", "sub_1B0974", "sub_1B098C", "sub_1B0B58", "sub_1B0FE4", "sub_1B1010", "sub_1B18F0", "sub_1B19BC", "sub_1B1CE8", "sub_1B26B4", "sub_1B26D4", "sub_1B26F4", "sub_1B27C8", "sub_1B2898", "sub_1B2918", "sub_1B2994", "sub_1B29A8", "sub_1B29B8", "sub_1B29D0", "sub_1B29F0", "sub_1B2A64", "sub_1B2AD4", "sub_1B2D70", "sub_1B2D90", "sub_1B2D98", "sub_1B2DA8", "sub_1B2DC8", "sub_1B2DE0", "sub_1B2E04", "sub_1B2E78", "sub_1B2F20", "sub_1B2F58", "sub_1B2F8C", "sub_1B3070", "sub_1B3080", "sub_1B309C", "sub_1B311C", "sub_1B3198", "sub_1B3218", "sub_1B3294", "sub_1B32F0", "sub_1B3348", "sub_1B3368", "sub_1B3384", "sub_1B3440", "sub_1B34F8", "sub_1B3524", "sub_1B354C", "sub_1B36F4", "sub_1B3714", "sub_1B3758", "sub_1B3798", "sub_1B37DC", "sub_1B381C", "sub_1B38E8", "sub_1B3960", "sub_1B3B00", "sub_1B3F0C", "sub_1B3F74", "sub_1B4040", "sub_1B4100", "sub_1B41DC", "sub_1B53A8", "sub_1B5500", "sub_1B56F8", "sub_1B578C", "sub_1B586C", "sub_1B5B50", "sub_1B5D04", "sub_1B5DE4", "sub_1B5E54", "sub_1B5EC4", "sub_1B6080", "sub_1B60E8", "sub_1B6180", "sub_1B6270", "sub_1B6448", "sub_1B6470", "sub_1B6498", "sub_1B64C0", "sub_1B6528", "sub_1B6540", "sub_1B6578", "sub_1B6678", "sub_1B6680", "sub_1B6694", "sub_1B6698", "sub_1B66A8", "sub_1B66D0", "sub_1B670C", "_ZNSt3__111__call_onceERVmPvPFvS2_E", "sub_1B6800", "sub_1B6810", "j_j__ZdlPv_2", "sub_1B6818", "sub_1B682C", "sub_1B6874", "sub_1B6898", "sub_1B68A4", "_ZNSt3__116generic_categoryEv", "sub_1B69CC", "sub_1B69D8", "sub_1B6AA8", "_ZNSt3__115system_categoryEv", "sub_1B6BAC", "sub_1B6BC4", "sub_1B6C94", "sub_1B6CB0", "sub_1B6DAC", "j_j__ZNSt15underflow_errorD2Ev", "sub_1B6DE4", "_ZNSt3__120__throw_system_errorEiPKc", "sub_1B6E94", "nullsub_1", "j_j__ZdlPv", "j_j__ZdlPv_0", "sub_1B6EB4", "sub_1B6F00", "__cxa_allocate_exception", "sub_1B6F5C", "sub_1B7068", "__cxa_free_exception", "sub_1B7084", "__cxa_free_dependent_exception_0", "sub_1B71B4", "__cxa_allocate_dependent_exception", "__cxa_free_dependent_exception", "__cxa_throw", "sub_1B7340", "__cxa_get_exception_ptr", "__cxa_begin_catch", "__cxa_end_catch", "sub_1B74C4", "__cxa_decrement_exception_refcount", "sub_1B7510", "__cxa_current_exception_type", "__cxa_rethrow", "__cxa_increment_exception_refcount", "__cxa_current_primary_exception", "sub_1B770C", "__cxa_rethrow_primary_exception", "sub_1B782C", "sub_1B7890", "__cxa_uncaught_exception", "sub_1B78C8", "__cxa_get_globals", "__cxa_get_globals_fast", "sub_1B79A0", "sub_1B79D4", "sub_1B7A08", "sub_1B7ABC", "sub_1B7B38", "_ZSt14get_unexpectedv", "sub_1B7BC4", "_ZSt10unexpectedv", "_ZSt13get_terminatev", "sub_1B7C14", "sub_1B7C30", "sub_1B7C40", "sub_1B7C50", "_ZSt9terminatev", "_ZSt15set_new_handlerPFvvE", "_ZSt15get_new_handlerv", "_Znwm", "_ZnwmRKSt9nothrow_t", "sub_1B7D64", "sub_1B7D74", "_Znam", "_ZnamRKSt9nothrow_t", "sub_1B7D90", "sub_1B7DA0", "_ZdlPv", "_ZdlPvRKSt9nothrow_t", "_ZdaPv", "_ZdaPvRKSt9nothrow_t", "_ZNSt9bad_allocC2Ev", "_ZNSt9bad_allocD2Ev", "_ZNSt9bad_allocD0Ev", "_ZNKSt9bad_alloc4whatEv", "_ZNSt20bad_array_new_lengthC2Ev", "_ZNSt20bad_array_new_lengthD0Ev", "_ZNKSt20bad_array_new_length4whatEv", "sub_1B7E48", "sub_1B86CC", "sub_1B86F8", "sub_1B8A68", "sub_1B8AE4", "sub_1B8C38", "_ZNSt9exceptionD2Ev", "_ZNSt9exceptionD0Ev", "_ZNKSt9exception4whatEv", "_ZNSt13bad_exceptionD0Ev", "_ZNKSt13bad_exception4whatEv", "j_j__ZNSt9type_infoD2Ev", "sub_1B8C74", "nullsub_2", "nullsub_3", "sub_1B8CA0", "sub_1B8CC4", "sub_1B8CE8", "sub_1B8D0C", "sub_1B8D30", "sub_1B8D54", "sub_1B8D78", "sub_1B8D9C", "sub_1B8DC0", "sub_1B8DE4", "sub_1B8E08", "sub_1B8E14", "sub_1B8E1C", "sub_1B8E24", "sub_1B8E30", "sub_1B8EF4", "sub_1B8F5C", "sub_1B8FD4", "sub_1B9120", "sub_1B9140", "__dynamic_cast", "sub_1B9410", "sub_1B975C", "sub_1B989C", "sub_1B9940", "sub_1B9B30", "sub_1B9BE4", "_ZNSt16invalid_argumentD2Ev", "_ZNSt11logic_errorD0Ev", "_ZNKSt11logic_error4whatEv", "_ZNSt15underflow_errorD2Ev", "_ZNSt13runtime_errorD0Ev", "_ZNKSt13runtime_error4whatEv", "_ZNSt12domain_errorD0Ev", "_ZNSt16invalid_argumentD0Ev", "_ZNSt12length_errorD0Ev", "_ZNSt12out_of_rangeD0Ev", "_ZNSt11range_errorD0Ev", "_ZNSt14overflow_errorD0Ev", "_ZNSt15underflow_errorD0Ev", "_ZNSt9type_infoD2Ev", "_ZNSt9type_infoD0Ev", "_ZNSt8bad_castC2Ev", "_ZNSt8bad_castD2Ev", "_ZNSt8bad_castD0Ev", "_ZNKSt8bad_cast4whatEv", "_ZNSt10bad_typeidC2Ev", "_ZNSt10bad_typeidD2Ev", "_ZNSt10bad_typeidD0Ev", "_ZNKSt10bad_typeid4whatEv", "sub_1BA110", "sub_1BA124", "sub_1BA128", "sub_1BA13C", "_ZNSt3__125notify_all_at_thread_exitERNS_18condition_variableENS_11unique_lockINS_5mutexEEE", "_ZNSt11logic_errorC2ERKNSt3__112basic_stringIcNS0_11char_traitsIcEENS0_9allocatorIcEEEE", "sub_1BA1F8", "_ZNSt11logic_errorC2EPKc", "sub_1BA278", "_ZNSt11logic_errorC2ERKS_", "_ZNSt11logic_erroraSERKS_", "_ZNSt13runtime_errorC2ERKNSt3__112basic_stringIcNS0_11char_traitsIcEENS0_9allocatorIcEEEE", "sub_1BA398", "_ZNSt13runtime_errorC2EPKc", "sub_1BA418", "_ZNSt13runtime_errorC2ERKS_", "_ZNSt13runtime_erroraSERKS_", "_ZNSt3__111this_thread9sleep_forERKNS_6chrono8durationIxNS_5ratioILl1ELl1000000000EEEEE", "_ZNSt3__119__thread_local_dataEv", "sub_1BA5A8", "sub_1BA5BC", "sub_1BA628", "sub_1BA63C", "sub_1BA654", "sub_1BA658", "sub_1BA730", "sub_1BA7A0", "sub_1BA7FC", "sub_1BA848", "sub_1BA85C", "sub_1BA944", "sub_1BA98C", "sub_1BAA1C", "sub_1BAB44", "_ZSt14set_unexpectedPFvvE", "_ZSt13set_terminatePFvvE", "__cxa_demangle", "sub_1BAF90", "sub_1BB014", "sub_1BB3C4", "sub_1BB40C", "sub_1BB4E0", "sub_1BB614", "sub_1BB6C8", "sub_1BC5AC", "sub_1BC614", "sub_1BC628", "sub_1BC634", "sub_1BC650", "sub_1BC6B0", "sub_1C4580", "sub_1C4624", "sub_1C9634", "sub_1C97D8", "sub_1C97DC", "sub_1C9800", "sub_1CA790", "sub_1CA794", "sub_1CA7B4", "sub_1CB65C", "sub_1CB660", "sub_1CB670", "sub_1CBDE8", "sub_1CBDEC", "sub_1CBDF0", "sub_1CC090", "sub_1CC178", "sub_1CC17C", "sub_1CC198", "sub_1CE3A8", "sub_1CE3AC", "sub_1CE3BC", "sub_1CE3CC", "sub_1CE3DC", "sub_1CE3F4", "sub_1CE410", "sub_1CE434", "sub_1CFDA4", "sub_1CFF68", "sub_1CFF9C", "sub_1D5A90", "sub_1D5A94", "sub_1D5AA4", "sub_1D5AE0", "sub_1D5C08", "sub_1D5CE8", "sub_1D5CEC", "sub_1D5D08", "sub_1D7744", "sub_1D77F8", "sub_1D8228", "sub_1D822C", "sub_1D8240", "sub_1D874C", "sub_1D880C", "sub_1D8AD0", "sub_1D8B18", "sub_1D9650", "sub_1D96A8", "sub_1D96B8", "sub_1D96C8", "sub_1D96F0", "sub_1D9718", "sub_1D9740", "sub_1D9A7C", "sub_1D9AA4", "sub_1DA2E4", "sub_1DA2E8", "sub_1DA2F4", "sub_1DA458", "sub_1DA470", "sub_1DB90C", "sub_1DB910", "sub_1DB930", "sub_1E36E4", "sub_1E384C", "sub_1E394C", "sub_1E3FE4", "sub_1E4094", "sub_1E4788", "sub_1E478C", "sub_1E47A4", "sub_1E487C", "sub_1E49C4", "sub_1E4AC0", "sub_1E4AE8", "sub_1E4C10", "sub_1E4DB0", "sub_1E4EB4", "sub_1E4EC0", "_ZNSt3__115future_categoryEv", "_ZNSt3__112future_errorC2ENS_10error_codeE", "sub_1E50F8", "_ZNSt3__112future_errorD2Ev", "_ZNSt3__112future_errorD0Ev", "sub_1E513C", "sub_1E5150", "sub_1E5190", "sub_1E5230", "sub_1E5244", "sub_1E5290", "sub_1E52E4", "_ZNSt3__112bad_weak_ptrD2Ev", "_ZNSt3__112bad_weak_ptrD0Ev", "_ZNKSt3__112bad_weak_ptr4whatEv", "nullsub_6", "j_j__ZdlPv_1", "sub_1E5344", "j_j__ZdlPv_3", "sub_1E538C", "_ZNSt3__112__get_sp_mutEPKv", "_ZNSt3__117declare_reachableEPv", "_ZNSt3__119declare_no_pointersEPcm", "_ZNSt3__121undeclare_no_pointersEPcm", "_ZNSt3__118get_pointer_safetyEv", "_ZNSt3__121__undeclare_reachableEPv", "_ZNSt3__15alignEmmRPvRm", "_ZSt18uncaught_exceptionv", "sub_1E5520", "_ZNSt16nested_exceptionC2Ev", "_ZSt17current_exceptionv", "_ZNSt16nested_exceptionD2Ev", "_ZNSt16nested_exceptionD0Ev", "_ZNKSt16nested_exception14rethrow_nestedEv", "sub_1E5614", "_ZSt17rethrow_exceptionSt13exception_ptr", "sub_1E563C", "sub_1E567C", "sub_1E84B0", "sub_1E90D0", "sub_1E926C", "sub_1E9384", "sub_1E99EC", "sub_1E9A00", "j_j__ZdlPv_14", "nullsub_20", "sub_1E9CD0", "sub_1E9D20", "sub_1E9D50", "sub_1E9D58", "sub_1E9DBC", "sub_1E9DE8", "sub_1E9E28", "sub_1EA074", "sub_1EA110", "sub_1EA73C", "sub_1EA750", "j_j__ZdlPv_15", "sub_1EA7A4", "sub_1EA830", "sub_1EA8D4", "sub_1EAAF8", "DobbyHook", "sub_1EAD78", "sub_1EAE04", "sub_1EAE90", "sub_1EAEC0", "sub_1EAEFC", "sub_1EAF34", "log_set_level", "log_switch_to_syslog", "log_switch_to_file", "log_internal_impl", "nullsub_4", "sub_1EB1C8", "sub_1EB254", "sub_1EB2BC", "sub_1EB310", "sub_1EB31C", "sub_1EB350", "sub_1EB364", "sub_1EB3A8", "sub_1EB3E8", "sub_1EB3F4", "sub_1EB460", "sub_1EB468", "sub_1EB470", "sub_1EB4FC", "sub_1EB508", "sub_1EB514", "sub_1EB54C", "sub_1EB584", "sub_1EB5AC", "sub_1EBBC0", "sub_1EBCFC", "sub_1EBDB0", "sub_1EBEE0", "sub_1EBEE4", "sub_1EBEF4", "sub_1EBEFC", "sub_1EBF04", "sub_1EBF0C", "sub_1EBF20", "sub_1EC02C", "CodePatch", "sub_1EC23C", "sub_1EC248", "sub_1EC268", "sub_1EC274", "sub_1EC298", "sub_1EC2A8", "sub_1EC2B0", "sub_1EC2B8", "sub_1EC2C0", "sub_1EC38C", "sub_1EC3B8", "sub_1EC660", "sub_1EC668", "sub_1EC670", "sub_1EC678", "j_free", "nullsub_21", "sub_1EC688", "sub_1EC69C", "sub_1EC6E0", "sub_1EC730", "sub_1EC754", "sub_1EC790", "sub_1EC814", "sub_1EC838", "sub_1EC844", "sub_1EC850", "sub_1EC8AC", "sub_1EC908", "sub_1ECA3C", "sub_1ECA44", "sub_1ECA4C", "sub_1ECA60", "sub_1ECAE8", "sub_1ECB24", "sub_1ECCD8", "sub_1ED2E4", "sub_1ED2F4", "_ZNSt3__112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEE6__initEPKcmm", "sub_1ED544", "sub_1ED764", "sub_1ED92C", "sub_1EDDC0", "sub_1EE384", "sub_1EE78C", "sub_1EF318", "sub_1EF574", "sub_1EF628", "sub_1EF814", "sub_1EF8BC", "sub_1EFAC8", "sub_1EFCC0", "sub_1F059C", "sub_1F09F8", "sub_1F0A00", "sub_1F0DB4", "sub_1F13FC", "sub_1F1404", "sub_1F1734", "sub_1F1954", "nullsub_23", "j_j__ZdlPv_24", "sub_1F20A0", "sub_1F2274", "sub_1F228C", "sub_1F22B0", "sub_1F22EC", "sub_1F2330", "sub_1F2764", "sub_1F2788", "sub_1F27C4", "sub_1F2B10", "sub_1F2B7C", "sub_1F2C7C", "sub_1F2DDC", "sub_1F32C8", "sub_1F36CC", "j_j__ZdlPv_16", "j_j__ZdlPv_17", "j_j__ZdlPv_18", "j_j__ZdlPv_19", "j_j__ZdlPv_20", "j_j__ZdlPv_21", "j_j__ZdlPv_22", "j_j__ZdlPv_23", "sub_1F36F4", "sub_1F4360", "nullsub_24", "nullsub_25", "nullsub_26", "nullsub_27", "nullsub_28", "nullsub_29", "nullsub_30", "nullsub_31", "sub_1F4528", "sub_1F48E0", "sub_1F4D94", "sub_1F4F68", "sub_1F513C", "sub_1F58C8", "sub_1F8584", "sub_1F8F4C", "sub_1F90F4", "sub_1F92C8", "sub_1FBF60", "sub_1FC37C", "sub_1FC550", "sub_1FEFD8", "sub_1FF0F4", "sub_1FF424", "sub_1FF8E8", "sub_1FFBA0", "sub_1FFC10", "sub_1FFCBC", "sub_1FFCC0", "sub_1FFCF4", "sub_1FFD14", "sub_1FFE18", "sub_1FFEC0", "sub_1FFEC4", "sub_1FFEF8", "sub_1FFF18", "sub_1FFFE8", "sub_2001A0", "sub_200394", "sub_200650", "sub_2006C4", "sub_200738", "sub_200960", "sub_200AC0", "sub_200D54", "sub_200D90", "sub_200F90", "sub_2010B4", "sub_201238", "sub_201440", "sub_201B20", "sub_201B28", "sub_201C24", "sub_201D9C", "sub_20245C", "sub_202554", "sub_2025F0", "sub_202718", "sub_202C98", "sub_203038", "sub_2038C0", "sub_203ED0", "sub_204678", "sub_204BC4", "sub_204CC0", "sub_204DD4", "sub_204F7C", "sub_20510C", "sub_205AD8", "sub_20691C", "sub_2069DC", "sub_206CE8", "sub_206D94", "sub_2071C0", "sub_2071EC", "sub_2075AC", "sub_2075C8", "sub_2076B8", "sub_207908", "nullsub_22", "sub_207960", "sub_207964", "sub_207A08", "sub_207A78", "sub_207AF0", "sub_207B54", "sub_208050", "sub_208068", "sub_2080CC", "sub_208198", "sub_2082A0", "sub_208310", "sub_208370", "sub_2083D8", "sub_2084C4", "sub_208558", "sub_2087A8", "sub_20895C", "sub_208A70", "sub_208BC8", "sub_208D14", "sub_208D84", "sub_208EE8", "sub_209010", "sub_2090B4", "sub_20916C", "sub_2091DC", "sub_20924C", "sub_20940C", "sub_209674", "sub_209CBC", "sub_209D88", "sub_20A5D0", "sub_20A7D0", "sub_20A86C", "sub_20AD24", "sub_20AD60", "sub_20AE04", "sub_20AF0C", "sub_20B4E0", "sub_20B4F4", "sub_20B570", "sub_20B8F4", "sub_20B900", "sub_20B904", "sub_20B938", "sub_20B958", "sub_20B98C", "sub_20BA5C", "sub_20BB74", "_ZNSt3__16vectorIiNS_9allocatorIiEEE21__push_back_slow_pathIiEEvOT_", "_ZNSt3__16vectorIiNS_9allocatorIiEEE21__push_back_slow_pathIRKiEEvOT_", "sub_20BF2C", "sub_20C0DC", "sub_20C3B8", "sub_20D070", "sub_20D0EC", "sub_20D138", "sub_20D2D0", "sub_20D36C", "sub_20D4B8", "sub_20DEF4", "sub_20E010", "sub_20E0CC", "sub_20E21C", "sub_20E35C", "sub_20E4D8", "sub_20E7C0", "sub_20E824", "sub_20E91C", "sub_20EAA4", "sub_20EC48", "sub_20ECBC", "sub_20ED30", "sub_20F5AC", "sub_20F768", "sub_20FA70", "sub_210574", "sub_2108C8", "sub_210AC0", "sub_212EE8", "sub_2133B8", "sub_2142AC", "sub_21587C", "sub_215A24", "sub_215BD8", "sub_216198", "sub_2161A4", "sub_2161AC", "sub_2161F0", "sub_216230", "sub_216274", "sub_2162B4", "sub_2162D4", "sub_2163AC", "sub_216504", "sub_217188", "sub_217444", "sub_217510", "sub_2175E0", "sub_217600", "j_j___read_self", "__read_self", "__self_kill", "__self_prctl", "__self_lseek", "sub_217698", "sub_2178BC", "sub_217ADC", "sub_217B20", "sub_217C3C", "sub_217DB8", "sub_217FFC", "sub_218CB0", "sub_218F34", "sub_2191F0", "sub_219294", "sub_219350", "sub_21944C", "sub_2195D0", "sub_2196C4", "sub_219720", "sub_219728", "sub_21972C", "sub_219910", "sub_219A58", "sub_219BEC", "sub_219C14", "sub_219C54", "sub_219D44", "sub_219E58", "_Unwind_GetGR", "sub_219EEC", "_Unwind_GetCFA", "_Unwind_SetGR", "_Unwind_GetIP", "_Unwind_GetIPInfo", "_Unwind_SetIP", "_Unwind_GetLanguageSpecificData", "_Unwind_GetRegionStart", "_Unwind_FindEnclosingFunction", "_Unwind_GetDataRelBase", "_Unwind_GetTextRelBase", "sub_21A0C4", "sub_21A13C", "sub_21A634", "sub_21AA50", "sub_21AFE8", "sub_21B20C", "sub_21B2D8", "sub_21B324", "sub_21B3E4", "__frame_state_for", "nullsub_5", "_Unwind_RaiseException", "_Unwind_ForcedUnwind", "_Unwind_Resume", "_Unwind_Resume_or_Rethrow", "_Unwind_DeleteException", "_Unwind_Backtrace", "sub_21BB30", "sub_21BB58", "sub_21BB98", "sub_21BBB4", "sub_21BC74", "sub_21BD28", "sub_21BD88", "sub_21BDE8", "sub_21BE48", "sub_21BF38", "sub_21BFC8", "sub_21C0C4", "sub_21C224", "sub_21C234", "sub_21C648", "sub_21C6E4", "sub_21C844", "sub_21C978", "__register_frame_info_bases", "__register_frame_info", "__register_frame", "__register_frame_info_table_bases", "__register_frame_info_table", "__register_frame_table", "__deregister_frame_info_bases", "__deregister_frame_info", "__deregister_frame", "_Unwind_Find_FDE", "__imp_opendir", "__imp_execve", "__imp_fdopen", "__imp_memcmp", "__imp_listen", "__imp_ioctl", "__imp___errno", "__imp_flock", "__imp_remove", "__imp_deflateEnd", "__imp_abort", "__imp_realloc", "__imp_connect", "__imp_write", "__imp_dup2", "__imp_pthread_self", "__imp__exit", "__imp_srandom", "__imp_sscanf", "__imp_fsync", "__imp_gettimeofday", "__imp_srand", "__imp_getpagesize", "__imp_inflate", "__imp_pipe", "__imp_pthread_cond_broadcast", "__imp_epoll_create", "__imp_access", "__imp_snprintf", "__imp_signal", "__imp_lseek", "__imp_getpid", "__imp_readdir", "__imp_creat", "__imp_time", "__imp_inflateInit2_", "__imp_memset", "__imp_unlink", "__imp_strlcpy", "__imp_dladdr", "__imp_fmodf", "__imp_malloc", "__imp_getcwd", "__imp_fork", "__imp_fmod", "__imp_random", "__imp_dlopen", "__imp_memmove", "__imp_gethostbyname", "__imp_strcat", "__imp_pthread_join", "__imp_rand", "__imp_fflush", "__imp_strncmp", "__imp_pthread_exit", "__imp_recv", "__imp_isxdigit", "__imp_strcpy", "__imp_strrchr", "__imp_fstat", "__imp___cxa_atexit", "__imp_pthread_key_delete", "__imp_lstat", "__imp_closedir", "__imp_pthread_create", "__imp_fclose", "__imp_setenv", "__imp_strdup", "__imp_inet_addr", "__imp_sleep", "__imp_open", "__imp_pow", "__imp_strlcat", "__imp_strstr", "__imp___assert2", "__imp_epoll_wait", "__imp_vfprintf", "__imp_kill", "__imp_deflateInit_", "__imp_munmap", "__imp_pthread_cond_wait", "__imp_pthread_setspecific", "__imp_deflate", "__imp_strchr", "__imp_chmod", "__imp_pthread_mutex_destroy", "__imp_gettid", "__imp_bind", "__imp_getppid", "__imp_adler32", "__imp_popen", "__imp_pthread_cond_destroy", "__imp_fnmatch", "__imp_dl_iterate_phdr", "__imp_waitpid", "__imp_fwrite", "__imp_llabs", "__imp_deflateBound", "__imp_readlink", "__imp_strncpy", "__imp___cxa_finalize", "__imp_exit", "__imp_raise", "__imp_dlclose", "__imp_vprintf", "__imp_send", "__imp_atoll", "__imp_fgets", "__imp_lseek64", "__imp_inflateEnd", "__imp_pthread_mutex_init", "__imp___android_log_vprint", "__imp_strtoul", "__imp_socket", "__imp_close", "__imp_epoll_ctl", "__imp_sysconf", "__imp_calloc", "__imp_execl", "__imp_getsockopt", "__imp_read", "__imp_fputc", "__imp_prctl", "__imp_strtoull", "__imp_setsockopt", "__imp_mkdir", "__imp_pthread_mutex_unlock", "__imp_fcntl", "__imp_vsnprintf", "__imp_setpgid", "__imp_ftruncate", "__imp_pthread_mutex_lock", "__imp_tolower", "__imp_nanosleep", "__imp_syscall", "__imp_mprotect", "__imp_isupper", "__imp_free", "__imp_toupper", "__imp_dlsym", "__imp_pclose", "__imp_atoi", "__imp_stat", "__imp_accept", "__imp_memchr", "__imp_vsyslog", "__imp_inet_ntoa", "__imp_strerror", "__imp_fopen", "__imp_isspace", "__imp_pthread_detach", "__imp_strtok", "__imp_pwrite", "__imp_fileno", "__imp_atol", "__imp_pthread_getspecific", "__imp_sprintf", "__imp_getenv", "__imp_pread", "__imp_memcpy", "__imp_pthread_once", "__imp_strlen", "__imp_pthread_key_create", "__imp_strtol", "__imp___stack_chk_fail", "__imp_mmap", "__imp_ptrace", "__imp_strcasecmp", "__imp_getuid"];
var func_addr_pack = [0x2940, 0x2960, 0x2970, 0x2980, 0x2990, 0x29a0, 0x29b0, 0x29c0, 0x29d0, 0x29e0, 0x29f0, 0x2a00, 0x2a10, 0x2a20, 0x2a30, 0x2a40, 0x2a50, 0x2a60, 0x2a70, 0x2a80, 0x2a90, 0x2aa0, 0x2ab0, 0x2ac0, 0x2ad0, 0x2ae0, 0x2af0, 0x2b00, 0x2b10, 0x2b20, 0x2b30, 0x2b40, 0x2b50, 0x2b60, 0x2b70, 0x2b80, 0x2b90, 0x2ba0, 0x2bb0, 0x2bc0, 0x2bd0, 0x2be0, 0x2bf0, 0x2c00, 0x2c10, 0x2c20, 0x2c30, 0x2c40, 0x2c50, 0x2c60, 0x2c70, 0x2c80, 0x2c90, 0x2ca0, 0x2cb0, 0x2cc0, 0x2cd0, 0x2ce0, 0x2cf0, 0x2d00, 0x2d10, 0x2d20, 0x2d30, 0x2d40, 0x2d50, 0x2d60, 0x2d70, 0x2d80, 0x2d90, 0x2da0, 0x2db0, 0x2dc0, 0x2dd0, 0x2de0, 0x2df0, 0x2e00, 0x2e10, 0x2e20, 0x2e2c, 0x2ed0, 0x2ee8, 0x2eec, 0x2f0c, 0x2f1c, 0x2f40, 0x2f54, 0x2f68, 0x2f6c, 0x2f8c, 0x2f98, 0x2fbc, 0x2fd0, 0x2fdc, 0x2fe4, 0x3034, 0x303c, 0x3060, 0x3080, 0x3090, 0x30a4, 0x30a8, 0x30ac, 0x30b0, 0x30e0, 0x30e4, 0x3190, 0x31ac, 0x31d0, 0x31ec, 0x3224, 0x3240, 0x3300, 0x3574, 0x35ac, 0x3604, 0x36c4, 0x36f8, 0x3768, 0x379c, 0x39dc, 0x3b1c, 0x3b58, 0x3bf4, 0x3c94, 0x4000, 0x41b4, 0x4918, 0x49f0, 0x4b54, 0x4c70, 0x4cd4, 0x4e84, 0x509c, 0x52bc, 0x5344, 0x5378, 0x53bc, 0x5400, 0x5444, 0x5478, 0x5650, 0x580c, 0x59bc, 0x5b08, 0x5df0, 0x5e6c, 0x5f20, 0x6044, 0x60e0, 0x6128, 0x6174, 0x61a8, 0x6208, 0x6268, 0x62a0, 0x633c, 0x6398, 0x63f8, 0x6424, 0x6430, 0x6484, 0x6544, 0x6590, 0x6698, 0x66e8, 0x67e0, 0x6814, 0x684c, 0x6884, 0x68c4, 0x68f8, 0x6930, 0x6968, 0x69d0, 0x6a04, 0x6a3c, 0x6a74, 0x6adc, 0x6b78, 0x6c14, 0x6c70, 0x6cd0, 0x6d08, 0x6d5c, 0x6dbc, 0x6ec0, 0x72d4, 0x7348, 0x73b4, 0x7470, 0x74bc, 0x75c8, 0x7724, 0x788c, 0x78c4, 0x78fc, 0x7b04, 0x7b4c, 0x7f44, 0x7fa0, 0x8000, 0x8130, 0x8224, 0x825c, 0x8430, 0x8ab0, 0x8afc, 0x8b50, 0x8c74, 0x8ed4, 0x900c, 0x9218, 0x92b0, 0x94bc, 0x96ac, 0x96e0, 0x9844, 0x98a0, 0x9900, 0x9940, 0x9944, 0x9964, 0x9988, 0x999c, 0x99cc, 0x99f4, 0x9a24, 0x9a4c, 0x9a50, 0x9a54, 0x9a58, 0x9a5c, 0x9a60, 0x9a64, 0x9a68, 0x9a6c, 0x9a70, 0x9a90, 0x9ac4, 0x9ba4, 0x9bd0, 0x9c20, 0x9c64, 0x9cc0, 0x9d04, 0x9d60, 0x9dd8, 0x9e58, 0x9ffc, 0xa070, 0xa3ec, 0xa52c, 0xb970, 0xc918, 0xe3e0, 0x10964, 0x11220, 0x13908, 0x154dc, 0x1561c, 0x1573c, 0x1585c, 0x15938, 0x159d4, 0x15a70, 0x162f8, 0x163dc, 0x1647c, 0x16598, 0x16654, 0x166c4, 0x1674c, 0x167bc, 0x16b08, 0x16bd8, 0x16e64, 0x16ef8, 0x1745c, 0x17664, 0x17680, 0x176a4, 0x176ac, 0x17748, 0x177d4, 0x177d8, 0x17854, 0x274000, 0x274008, 0x274010, 0x274018, 0x274020, 0x274028, 0x274030, 0x274038, 0x274040, 0x274048, 0x274050, 0x274058, 0x274060, 0x274068, 0x274070, 0x274078, 0x274080, 0x274088, 0x274090, 0x274098, 0x2740a0, 0x2740a8, 0x2740b0, 0x2740b8, 0x2740c0, 0x2740c8, 0x2740d0, 0x2740d8, 0x2740e0, 0x2740e8, 0x2740f0, 0x2740f8, 0x274100, 0x274108, 0x274110, 0x274118, 0x274120, 0x274128, 0x274130, 0x274138, 0x274140, 0x274148, 0x274150, 0x274158, 0x274160, 0x274168, 0x274170, 0x274178, 0x274180, 0x274188, 0x274190, 0x274198, 0x2741a0, 0x2741a8, 0x2741b0, 0x2741b8, 0x2741c0, 0x2741c8, 0x2741d0, 0x2741d8, 0x2741e0, 0x2741e8, 0x2741f0, 0x2741f8, 0x274200, 0x274208, 0x274210];
var func_name_pack = ["sub_2940", "strcpy", "uncompress", "fmodf", "j_ffi_call_SYSV", "pthread_create", "strerror", "snprintf", "syscall", "munmap", "__stack_chk_guard", "__errno", "getenv", "_Znwm", "getpid", "fgets", "prctl", "j_interpreter_wrap_int64_t", "j_ffi_prep_cif", "memcpy", "puts", "__cxa_finalize", "feof", "malloc", "j_ffi_prep_cif_var", "vsnprintf", "select", "readdir", "isspace", "inotify_init", "dladdr", "lseek", "_ZdlPv", "j___clear_cache", "mmap", "pthread_detach", "abort", "__stack_chk_fail", "strtol", "calloc", "kill", "j___aarch64_sync_cache_range", "dl_iterate_phdr", "fmod", "strstr", "__cxa_pure_virtual", "j_ffi_prep_closure_loc", "read", "strncmp", "dlopen", "strncpy", "j_ffi_prep_cif_machdep", "setenv", "strtok", "sscanf", "isalpha", "sigaction", "dlsym", "strdup", "j_ffi_call", "fopen", "memset", "_ZdaPv", "fclose", "time", "opendir", "strcmp", "inotify_add_watch", "_Znam", "atoi", "strlen", "j__Z10__arm_a_20v", "open", "mprotect", "closedir", "close", "raise", "start", "sub_2E2C", "sub_2ED0", "sub_2EE8", "sub_2EEC", "sub_2F0C", "sub_2F1C", "sub_2F40", "sub_2F54", "sub_2F68", "sub_2F6C", "sub_2F8C", "sub_2F98", "sub_2FBC", "sub_2FD0", "sub_2FDC", "sub_2FE4", "sub_3034", "sub_303C", "sub_3060", "sub_3080", "sub_3090", "sub_30A4", "sub_30A8", "sub_30AC", "sub_30B0", "j_abort", "sub_30E4", "sub_3190", "sub_31AC", "sub_31D0", "sub_31EC", "sub_3224", "sub_3240", "sub_3300", "sub_3574", "sub_35AC", "sub_3604", "sub_36C4", "sub_36F8", "sub_3768", "sub_379C", "sub_39DC", "sub_3B1C", "sub_3B58", "sub_3BF4", "sub_3C94", "sub_4000", "sub_41B4", "sub_4918", "sub_49F0", "sub_4B54", "sub_4C70", "sub_4CD4", "sub_4E84", "sub_509C", "sub_52BC", "sub_5344", "sub_5378", "sub_53BC", "sub_5400", "sub_5444", "sub_5478", "sub_5650", "sub_580C", "sub_59BC", "sub_5B08", "sub_5DF0", "sub_5E6C", "sub_5F20", "sub_6044", "sub_60E0", "sub_6128", "sub_6174", "sub_61A8", "sub_6208", "sub_6268", "sub_62A0", "sub_633C", "sub_6398", "sub_63F8", "sub_6424", "_ZN9__arm_c_19__arm_c_0Ev", "sub_6484", "sub_6544", "sub_6590", "sub_6698", "sub_66E8", "sub_67E0", "sub_6814", "sub_684C", "sub_6884", "sub_68C4", "sub_68F8", "sub_6930", "sub_6968", "sub_69D0", "sub_6A04", "sub_6A3C", "sub_6A74", "sub_6ADC", "sub_6B78", "sub_6C14", "sub_6C70", "sub_6CD0", "_ZN10DynCryptor9__arm_c_0Ev", "sub_6D5C", "sub_6DBC", "sub_6EC0", "sub_72D4", "sub_7348", "_Z10__arm_a_21v", "_Z10__arm_a_20v", "sub_74BC", "sub_75C8", "sub_7724", "sub_788C", "sub_78C4", "sub_78FC", "sub_7B04", "sub_7B4C", "sub_7F44", "_Z9__arm_a_2PcmS_Rii", "sub_8000", "sub_8130", "sub_8224", "sub_825C", "sub_8430", "sub_8AB0", "JNI_OnLoad", "sub_8B50", "sub_8C74", "sub_8ED4", "sub_900C", "sub_9218", "sub_92B0", "sub_94BC", "sub_96AC", "sub_96E0", "_Z9__arm_a_1P7_JavaVMP7_JNIEnvPvRi", "sub_98A0", "sub_9900", "nullsub_2", "sub_9944", "sub_9964", "sub_9988", "sub_999C", "sub_99CC", "sub_99F4", "sub_9A24", "nullsub_3", "nullsub_4", "nullsub_5", "nullsub_6", "j_lseek", "j_lseek_0", "j_lseek_1", "j_lseek_2", "j_lseek_3", "sub_9A70", "sub_9A90", "sub_9AC4", "sub_9BA4", "sub_9BD0", "sub_9C20", "sub_9C64", "sub_9CC0", "sub_9D04", "sub_9D60", "sub_9DD8", "sub_9E58", "sub_9FFC", "sub_A070", "sub_A3EC", "sub_A52C", "sub_B970", "sub_C918", "sub_E3E0", "sub_10964", "sub_11220", "sub_13908", "interpreter_wrap_int64_t", "interpreter_wrap_float", "interpreter_wrap_double", "interpreter_wrap_int64_t_bridge", "interpreter_wrap_float_bridge", "interpreter_wrap_double_bridge", "sub_15A70", "sub_162F8", "sub_163DC", "sub_1647C", "sub_16598", "sub_16654", "sub_166C4", "sub_1674C", "sub_167BC", "ffi_prep_cif_machdep", "ffi_call", "ffi_prep_closure_loc", "sub_16EF8", "sub_1745C", "ffi_prep_cif", "ffi_prep_cif_var", "ffi_prep_closure", "ffi_call_SYSV", "ffi_closure_SYSV", "__clear_cache", "__aarch64_sync_cache_range", "_Z9__arm_a_0v", "__imp_strcpy", "__imp_uncompress", "__imp_fmodf", "__imp_pthread_create", "__imp_strerror", "__imp_snprintf", "__imp_syscall", "__imp_munmap", "__imp___stack_chk_guard", "__imp___errno", "__imp_getenv", "__imp__Znwm", "__imp_getpid", "__imp_fgets", "__imp_prctl", "__imp_memcpy", "__imp_puts", "__imp___cxa_finalize", "__imp_feof", "__imp_malloc", "__imp_vsnprintf", "__imp_select", "__imp_readdir", "__imp_isspace", "__imp_inotify_init", "__imp_dladdr", "__imp_lseek", "__imp__ZdlPv", "__imp_mmap", "__imp_pthread_detach", "__imp_abort", "__imp___stack_chk_fail", "__imp_strtol", "__imp_calloc", "__imp_kill", "__imp_dl_iterate_phdr", "__imp_fmod", "__imp_strstr", "__imp___cxa_pure_virtual", "__imp_read", "__imp_strncmp", "__imp_dlopen", "__imp_strncpy", "__imp_setenv", "__imp_strtok", "__imp_sscanf", "__imp_isalpha", "__imp_sigaction", "__imp_dlsym", "__imp_strdup", "__imp_fopen", "__imp_memset", "__imp__ZdaPv", "__imp_fclose", "__imp_time", "__imp_opendir", "__imp_strcmp", "__imp_inotify_add_watch", "__imp__Znam", "__imp_atoi", "__imp_strlen", "__imp_open", "__imp_mprotect", "__imp_closedir", "__imp_close", "__imp_raise", "free"];
for (let i = 0; i < func_addr_main.length; i++) {
    func_addr_main[i] += 0xe7000;
}
var func_addr=func_addr_pack.concat(func_addr_main);
var func_name=func_name_pack.concat(func_name_main);
var so_name = "libjiagu_64.so";
/*
    @param print_stack: Whether printing stack info, default is false.
*/
var print_stack = false;
 
/*
    @param print_stack_mode
    - FUZZY: print as much stack info as possible
    - ACCURATE: print stack info as accurately as possible
    - MANUAL: if printing the stack info in an error and causes exit, use this option to manually print the address
*/
var print_stack_mode = "FUZZY";
 
function addr_in_so(addr){
    var process_Obj_Module_Arr = Process.enumerateModules();
    for(var i = 0; i < process_Obj_Module_Arr.length; i++) {
        if(addr>process_Obj_Module_Arr[i].base && addr= 0) {
            console.log("find",pathname+", redirect to",fakePath);
            var filename = Memory.allocUtf8String(fakePath);
            return open(filename, flag);
        }
        if (pathname.indexOf("dex") >= 0) {
            if(!dump_once){
                dump_once = true;             
            }
        }
        var fd = open(pathnameptr, flag);
        return fd;
    }, 'int', ['pointer', 'int']));
}
function hook_dlopen() {
    Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"),
        {
            onEnter: function (args) {
                var pathptr = args[0];
                if (pathptr !== undefined && pathptr != null) {
                    var path = ptr(pathptr).readCString();
                    //console.log(path);
                    if (path.indexOf(so_name) >= 0) {
                        this.is_can_hook = true;
                    }
                }
            },
            onLeave: function (retval) {
                if (this.is_can_hook) {
                    // note: you can do any thing before or after stalker trace so.      
                                                
                    trace_so();               
                }
            }
        }
    );
}
 
function trace_so(){
    var times = 1;
    var module = Process.getModuleByName(so_name);
    var pid = Process.getCurrentThreadId();
    console.log("start Stalker!");
    Stalker.exclude({
        "base": Process.getModuleByName("libc.so").base,
        "size": Process.getModuleByName("libc.so").size
    })
    Stalker.follow(pid,{
        events:{
            call:false,
            ret:false,
            exec:false,
            block:false,
            compile:false
        },
        onReceive:function(events){
        },
        transform: function (iterator) {
            var instruction = iterator.next();
            do{
                if (func_addr.indexOf(instruction.address - module.base) !== -1) {
                    if(func_addr.indexOf(instruction.address-module.base) {
                                console.log("backtrace:\n"+Thread.backtrace(context, Backtracer.FUZZY).map(DebugSymbol.fromAddress).join('\n'));
                                console.log('---------------------')
                            });
                        }
                        else if (print_stack_mode === "ACCURATE") {
                            iterator.putCallout((context) => {
                                console.log("backtrace:\n"+Thread.backtrace(context, Backtracer.ACCURATE).map(DebugSymbol.fromAddress).join('\n'));
                                console.log('---------------------')
                            })
                        }
 
                        else if (print_stack_mode === "MANUAL") {
                            iterator.putCallout((context) => {
                                console.log("backtrace:")
                                Thread.backtrace(context, Backtracer.FUZZY).map(addr_in_so);
                                console.log('---------------------')
                            })
                        }
                    }
                }
                iterator.keep();
            } while ((instruction = iterator.next()) !== null);
        },
 
        onCallSummary:function(summary){
 
        }
    });
    console.log("Stalker end!");
}
setImmediate(hook_dlopen,0); 
```

这里我本来是打算加上 hook_self_map 的代码以便跑更多的函数，但是因为 hook 了 open 使得 stalker 可能的对 maps 的访问也失败了所以报错，于是先暂时仅 trace_so()，这里思考了一下有没有什么办法可以让 hook_self_map() 更加成功地仅过掉壳的检测，但是没有找到具体的检测点到底在哪里，可以猜到的是应该检测的是如 frida 字符串等特征，这样的话，那我编译一份 frida 将这些特征去掉从源头上根除不就好了吗？我觉得相比去阅读代码找检查点，隐藏自己可能更为容易些，不过这些是后话了，现在还是跟着作者的思路走吧。

分析新的函数调用链发现三个 zlib 解压缩的函数

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740554550846-226ab938-3ccd-42ae-9001-d23d5e7967c8.png)

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740662597821-7169674b-16e1-4f42-aca7-334f90f2f227.png)

inflateInit2_有两个调用处。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740659863129-7c7a57ab-7489-4421-9db1-3d790008149f.png)

看函数调用链的话应该是第二个调用的，转到该函数。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740660811656-62981ee4-42ae-42b8-b3af-6776ed5a10fa.png)

导入头文件

```
#  define z_const const
typedef unsigned char  Byte;  /* 8 bits */
typedef unsigned int   uInt;  /* 16 bits or more */
typedef unsigned long  uLong; /* 32 bits or more */
typedef struct z_stream_s {
    z_const Bytef *next_in;     /* next input byte */
    uInt     avail_in;  /* number of bytes available at next_in */
    uLong    total_in;  /* total number of input bytes read so far */
  
    Byte    *next_out; /* next output byte will go here */
    uInt     avail_out; /* remaining free space at next_out */
    uLong    total_out; /* total number of bytes output so far */
} z_stream;
```

重定义 s 的类型为 z_stream

*   `<font>s.next_in</font>`<font>: 压缩数据 </font>
*   `<font>s.avail_in</font>`<font>: 压缩数据的长度 </font>
*   `<font>s.next_out</font>`<font>: 解压后的数据 </font>
*   `<font>s.avail_out</font>`<font>: 解压后数据的长度 </font>

接下来去 HOOK inflate 函数

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740662328769-7142572e-03d0-4c3a-b5af-ee4fa89260e4.png)

接下来要 HOOK 一下 inflate 函数看看解压缩后的数据。

这里仿照作者的做法，获取外部函数的调用次数来判断目前是否已经加载主 elf，这里作者是通过 zlib_count 变量来统计 inflate 解压函数的调用次数，壳会调用一次，主 ELF 会调用一次，所以第二次调用时就一定在主 ELF 了，这里我实在太懒了，有作者现成的脚本，所以我只阅读了下然后 cv。

```
function dump_memory(start, size, filename) {
    // 定义文件路径，将文件保存到指定路径中
    var file_path = "/data/data/com.oacia.apk_protect/" + filename;
     
    // 创建一个文件对象，以二进制写模式打开文件
    var file_handle = new File(file_path, "wb");
 
    // 如果文件句柄有效，且文件成功打开
    if (file_handle && file_handle != null) {
        // 从内存中读取指定大小的数据
        var libso_buffer = start.readByteArray(size.toUInt32());
         
        // 将数据写入到文件
        file_handle.write(libso_buffer);
         
        // 刷新缓存，确保数据已完全写入文件
        file_handle.flush();
         
        // 关闭文件
        file_handle.close();
         
        // 打印转储文件的路径
        console.log("[dump]:", file_path);
    }
}
 
// 这是用于 hook `inflate` 函数结果的函数
function hook_zlib_result() {
    // 查找模块 libjiagu_64.so
    var module = Process.findModuleByName("libjiagu_64.so");
 
    // 拦截目标函数地址并进行 hook
    Interceptor.attach(module.base.add(0x1B63F0), {
        // 拦截函数进入时执行的代码
        onEnter: function (args) {
            console.log("inflate result");
             
            // 打印解压输入数据块
            console.log(hexdump(next_in, {
              offset: 0,   // 相对偏移
              length: 0x50, // dump 的大小
              header: true,
              ansi: true
            }));
             
            // 打印解压输出数据块
            console.log(hexdump(next_out, {
              offset: 0,   // 相对偏移
              length: 0x50, // dump 的大小
              header: true,
              ansi: true
            }));
             
            // 将解压后的数据转储到文件
            dump_memory(next_out, avail_out, "dex001");
        },
 
        // 拦截函数离开时执行的代码
        onLeave: function (ret) {
        }
    });
}
 
// 用于控制 zlib hook 的计数器
var zlib_count = 0;
 
// 定义输入输出相关指针
var next_in, avail_in, next_out, avail_out;
 
// 这是 hook `inflate` 函数的主要函数
function hook_zlib() {
    // 拦截所有调用 `inflate` 函数的地方
    Interceptor.attach(Module.findExportByName(null, "inflate"), {
        // 拦截函数进入时执行的代码
        onEnter: function (args) {
            zlib_count += 1;
 
            // 如果已经执行过一次 `inflate`，则 hook `inflate` 函数结果
            if (zlib_count > 1) {
                hook_zlib_result();
            }
 
            // 读取 `next_in` 指针：输入数据的指针
            next_in = ptr(args[0].add(0x0).readS64());
            // 读取 `avail_in`：输入数据的剩余字节数
            avail_in = ptr(args[0].add(0x8).readS64());
            // 读取 `next_out` 指针：输出数据的指针
            next_out = ptr(args[0].add(0x18).readS64());
            // 读取 `avail_out`：输出数据的剩余字节数
            avail_out = ptr(args[0].add(0x20).readS64());
 
            // 打印输入数据块的十六进制内容
            console.log(hexdump(next_in, {
              offset: 0,   // 相对偏移
              length: 0x50, // dump 的大小
              header: true,
              ansi: true
            }));
            // 打印 `inflate` 函数的第二个参数
            console.log(args[1]);
        },
        // 拦截函数离开时执行的代码
        onLeave: function (ret) {
        }
    });
}
```

同样是放到 dlopen 的 onLeave 即可，这里要先把过反调试的函数注释掉，因为他之后还有很多次调用，如果被反调试的话刚好显示的第二次解压出的 dex。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740714947065-661bbd3e-882e-4eb1-a674-6018e7d26f1c.png)

这个 dex 和壳 dex 是同一个文件，壳的 dex 结尾是有加密 dex 的，也就是说已经获取加密主 dex 的数据，那么解密就不远了。

再查看解压缩的函数的调用者，根据函数调用链推测是 sub_1A0C88。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740718628991-bb95b5fa-8be3-46e6-b198-8251263e10a9.png)

然后再顺着调用链往前追索，期间可以看到很多有趣的字样，

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740718824676-be15a26b-4b68-4a43-a5b7-998cace2b285.png)

直到 sub_1332B8, 这个函数大的可怕，光是变量的定义就有五百多行。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740718922284-943a8c9a-ad38-4723-a411-e605c8550fc2.png)![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740719250061-ad21db68-7313-470b-9ee6-31e25c79a657.png)

看一下是哪里调用的这里，根据之前 hook open 时的堆栈调用是 19b780

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740128580867-4638223c-61c1-4402-94a9-60322e1316dd.png?x-oss-process=image%2Fformat%2Cwebp)

所以我们看下这个函数，同时这里如果查看壳 elf 的同这个位置的话，会发现这里的跳转表在主 ELF 中已经是解析后的状态了。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740723895665-5952819a-389a-4d92-be86-de1b27ca3d91.png)

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1740723900442-623cd54f-2588-4441-9987-d62f7a9377f5.png)

打开文件并写入，再往上追溯调用，是 sub_1332B8 调用的该函数，而 sub_1332B8 代码中获取过壳 dex 的数据。

这里很可能是创建 classes.dex 的位置，对这里进行 HOOK。

同时这里因为 apk 很容易崩溃然后需要重启手机所以写了个 bat 脚本方便一下。

```
adb forward tcp:1234 tcp:1234
echo su > temp.txt
echo cd /data/local/tmp >> temp.txt
echo ./Frida -l 0.0.0.0:1234 >> temp.txt
adb shell < temp.txt
del temp.txt
```

HOOK sub_19B760 并同时打印函数调用

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1741093909174-24ea760b-1f6c-4cd7-821f-7fe6ad62451f.png)

可以看的出来，classes.dex 是第一个调用处，而 classes2 和 3 是第二个调用处。

同时在 sub_128D44 后程序会终止。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1741094201451-789bb00c-a488-4ba0-9b2b-461e2e11322d.png)

而 sub_128D44 传入的是主 dex。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1741094296922-7285be1a-c687-415e-896b-fa66218b40ee.png)

但其在 trace_so 的打印中又并没有，这里我对 stalker 了解并不深，只是仿照作者的方法，在调用到该函数时再调用一次 trace_so(), 好像不太需要考虑 hook 时机问题，直接放在 hook dlopen 那个函数就可以，但是这里我在原本的插件生成的 trace 脚本中不知道为什么没有反应，所以将 trace_so 函数移到了我的脚本中使用，结果倒是也和作者一样

```
function hook_1346B0(){
    var module = Process.findModuleByName("libjiagu_64.so");
    var time=0;
    Interceptor.attach(module.base.add(0x1346B0), {
        onEnter: function (args) {
            hook_dlopenn.detach();
            var backtrace = Thread.backtrace(this.context, Backtracer.ACCURATE);
            time+=1;
            trace_so();
            // 遍历并打印前 3 个参数
            console.log("[+]test++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++")
                //console.log("[+] call time"+time+"in address"+backtrace[1]+"\n");     
        }
    });
}
```

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1741146199659-5bf2562d-1af8-45ae-91a4-ec0aacffd928.png)

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1741146453366-0068f686-9261-4e01-b77d-c9e1b17e0f51.png)

但是壳的系统函数好像不太能相信吧，按照前面的错位来说的话，这里的最后调用的系统函数应该是 close？不过其他普通函数肯定是没有什么问题的。而上一个 feof 则是 __cxa_finalize 。

> `__cxa_finalize` 是一个用于在程序终止时调用 C++ 静态和全局对象析构函数的函数。它通常由编译器和 C++ 运行时库自动调用，以确保在程序结束时正确清理资源。

倒是也符合结束的情况，但是吧，这也很奇怪，因为程序开始的函数调用链和从 sub_128D44 这里开始打印的话，Pack 的函数调用是一样的。所以其实整个分析过程还是有很多疑惑的点存在。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1741147379334-76fdacbf-6308-4cf3-aa5d-bbd75cd08e80.png)

这里没有分析出来什么，还是返回之前创建 classes.dex 的地方，![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1741151316449-8ceb3ec8-f5be-4586-b711-5c902011a1c6.png)

这里如果创建了的话，那解密应该也不会太远，向下接着分析。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1741151490970-ac0049ed-7af9-4399-84c6-0d7087287a6c.png)

这里搭配上 anti_frida_check() 函数 HOOK sub_18FEA8.

sub_18FEA8 的参数 v203 和 clasee23 的参数 v163 是同一个，这个函数的参数分析不一定正确，于是将参数依次打出尝试，发现第二个参数即是 dex，第三个参数猜测是大小。

```
function hook_18FEA8(){
   var module = Process.findModuleByName("libjiagu_64.so");
    var time=0;
    Interceptor.attach(module.base.add(0x18FEA8), {
        onEnter: function (args) {
            //hook_dlopenn.detach();
            var backtrace = Thread.backtrace(this.context, Backtracer.ACCURATE);
            time+=1;
            //trace_so();
            // 遍历并打印前 3 个参数
            console.log("[+]test++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++")
            console.log(hexdump(args[1]));
            dump_memory(ptr(args[1]),args[2],"classes"+time+".dex");
 
        }
    });
}
```

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1741153526076-fd828fe9-d728-4e41-a1bd-a28fa0d414a2.png)

很明显是脱完后的了。

![](https://cdn.nlark.com/yuque/0/2025/png/39289695/1741153786224-077655d4-7771-40bc-8253-f05860fea8f3.png)

onCreate 变成了 native 定义，这些以后有时间再回来接着分析吧。

结尾
--

唉，其实这个壳的分析还是有点不甘心的，因为还有好多东西没搞明白，也还有好多东西还可以深挖，如果能看源码学习下就好了，有点异想天开了。虽然反调试动调那块写的比较少，但基本是我那两三天动调的精华，之后有时间还会再补反调试和 VMP 相关的知识吧，如果有人能将这个壳反调试的过程告诉我就好了。。。其实也找过相关的文章，但是大多是其他时间段的，反调试的机制好像不完全一样，比如那个 fake-libs，没太去分析他在检测什么，很奇怪的一个字符串。后续希望还有高手发布该壳更详细的分析文章吧。途中的一些文件都打包放到网盘内了。  
链接: https://pan.baidu.com/s/1bCQQNtCIpxnGbYPuhs2U1Q?pwd=rea1 提取码: rea1

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-173.htm)
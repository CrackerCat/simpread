> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-278932.htm)

> [原创] 某手新版本 sig3 参数算法还原

[原创] 某手新版本 sig3 参数算法还原

4 小时前 177

### [原创] 某手新版本 sig3 参数算法还原

4 小时前 177

某手新版本 sig3 参数算法还原
=================

免责声明：本文章仅供纯技术讨论和研究，本文提供的可操作性不得用于任何商业用途和违法违规场景。本人对任何原因在使用本人中提供的代码和策略时可能对用户自己或他人造成的任何形式的损失和伤害不承担责任。
-------------------------------------------------------------------------------------------------

### 从星球学到很多，分享给更多人

### Frida Native 层主动调用

```
export function callDoCommandNative(){ // jni方法复习
    Java.perform(function() {
        var base_addr = Module.findBaseAddress("libkwsgmain.so") || ptr(0x0);
        var real_addr = base_addr.add(0x41680)
        var docommand = new NativeFunction(real_addr, "pointer", ["pointer", "pointer", "int", "pointer"]);
 
        var JNIEnv = Java.vm.getEnv();
        var Intger = Java.use("java.lang.Integer");
        var jstring = Java.use("java.lang.String");
        var Boolean = Java.use("java.lang.Boolean");
        var cla = JNIEnv.findClass("java/lang/Object");
 
        // args1:,d7b7d042-d4f2-4012-be60-d97ff2429c17,,,com.yxcorp.gifshow.App@27101bb,,'} data: None
        var currentApplication = Java.use("android.app.ActivityThread").currentApplication();
        var context = currentApplication.getApplicationContext();
        log("context"+context)
 
        var input_1 = JNIEnv.newStringUtf('d7b7d042-d4f2-4012-be60-d97ff2429c17');
        var argList_0 = JNIEnv.newObjectArray(7, cla, ptr(0x0));
        JNIEnv.setObjectArrayElement(argList_0, 0, ptr(0x0));
        JNIEnv.setObjectArrayElement(argList_0, 1, input_1);
        JNIEnv.setObjectArrayElement(argList_0, 2, ptr(0x0));
        JNIEnv.setObjectArrayElement(argList_0, 3, ptr(0x0));
        JNIEnv.setObjectArrayElement(argList_0, 4, context.$h);
        JNIEnv.setObjectArrayElement(argList_0, 5, ptr(0x0));
        JNIEnv.setObjectArrayElement(argList_0, 6, ptr(0x0));
 
        var point_0 = docommand(JNIEnv, ptr(0x0), 10412, argList_0); // 返回的是指针，通过cast转成java对象，从而读出来
        console.log("point_0: " + point_0);
        var s_0 = Java.cast(point_0, Java.use("java.lang.Object"));
        console.log("result: " + s_0);
 
        var argList = JNIEnv.newObjectArray(8, cla, ptr(0x0));
        var argList_1 = JNIEnv.newObjectArray(1, cla, ptr(0x0));
        var input0 = JNIEnv.newStringUtf('/rest/n/feed/selectionbb9caf23ee1fda57a6c167198aba919f');
        var input1 = JNIEnv.newStringUtf('d7b7d042-d4f2-4012-be60-d97ff2429c17');
        var input2 = Boolean.$new(false);
        var input2_2 = Boolean.$new(false);
        var input3 = Intger.$new(-1);
        var input5 = JNIEnv.newStringUtf("010a11c6-f2cb-4016-887d-0d958aef1534");
         
        JNIEnv.setObjectArrayElement(argList_1, 0, input0);
        JNIEnv.setObjectArrayElement(argList, 0, argList_1);
        JNIEnv.setObjectArrayElement(argList, 1, input1);
        JNIEnv.setObjectArrayElement(argList, 2, input3.$h);
        JNIEnv.setObjectArrayElement(argList, 3, input2.$h);
        JNIEnv.setObjectArrayElement(argList, 4, ptr(0x0));
        JNIEnv.setObjectArrayElement(argList, 5, ptr(0x0));
        JNIEnv.setObjectArrayElement(argList, 6, input2_2.$h);
        JNIEnv.setObjectArrayElement(argList, 7, input5);
        var point = docommand(JNIEnv, ptr(0x0), 10418, argList); // 返回的是指针，通过cast转成java对象，从而读出来
        var s = Java.cast(point, Java.use("java.lang.Object")); // $className : java.lang.String
        console.log("result: " + s);
        console.log("result: " + Java.vm.tryGetEnv().getStringUtfChars(point).readCString());
    })
}
     
export function jniOnload(){
    var android_dlopen_ext = Module.findExportByName(null, "android_dlopen_ext");
    if (android_dlopen_ext != null) {
        Interceptor.attach(android_dlopen_ext, {
            onEnter: function (args) {
                this.hook = false;
                var soName = args[0].readCString() || '';
                if (soName.indexOf("libkwsgmain.so") !== -1) {
                    this.hook = true;
                }
            },
 
            onLeave: function (retval) {
                if (this.hook) {
                    var jniOnload = Module.findExportByName("libkwsgmain.so", "JNI_OnLoad") || ptr(0x0);
                    Interceptor.attach(jniOnload, {
                        onEnter: function (args) {
                            console.log("Enter Mtguard JNI OnLoad");
                        },
                        onLeave: function (retval) {
                            console.log("After Mtguard JNI OnLoad");
                            callDoCommandNative();
                            // hook_ks();
                        }
                    });
                }
            }
        });
    }
}

```

这里踩了一个坑，传入的对象数组里有个元素是 context 类型，打印出来是 com.yxcorp.gifshow.App@b97f2c，所以想当然的就按这个类直接 new 了个对象传了进去，很快出现了报错：  
![](https://bbs.kanxue.com/upload/attach/202309/915510_G9P6CJFMMZ54HTR.png)  
这个报错让我百思不得其解，压根就没往 context 这块想，以为哪怕填个空指针都没问题的。

最后，用笨法 frida hooknative 层地址，打印定位，找到了如此地方：

![](https://bbs.kanxue.com/upload/attach/202309/915510_WX93CGM9NSRP4DM.png)  
对应汇编

![](https://bbs.kanxue.com/upload/attach/202309/915510_TQS3DXANUC2CXV9.png)

此处判断 X23 的值是否为 0，正常打印出来是`/data/app/com.smile.gifmaker-q14Fo0PSb77vTIOM1-iEqQ==/base.apk`，调用了`getPackageCodePath` 方法。

![](https://bbs.kanxue.com/upload/attach/202309/915510_76TA7JRUJFJ7Q4P.png)

这是一个 context 方法，那必须传入有效的 context：

```
var currentApplication = Java.use("android.app.ActivityThread").currentApplication();
var context = currentApplication.getApplicationContext();

```

解决方法就是如此简单，基础，却让我绕了一大圈弯路，不得不感叹，基础真重要啊！

### IDA 静态分析

*   花指令
    *   这块看龙哥的分析就好了，非常清晰

### Unidbg 模拟执行

*   搭架子 & 补环境

```
package com.smile.gifmaker3;
 
import com.github.unidbg.*;
import com.github.unidbg.Module;
import com.github.unidbg.arm.backend.Backend;
import com.github.unidbg.arm.backend.CodeHook;
import com.github.unidbg.arm.backend.UnHook;
import com.github.unidbg.arm.backend.UnicornBackend;
import com.github.unidbg.arm.context.Arm32RegisterContext;
import com.github.unidbg.arm.context.Arm64RegisterContext;
import com.github.unidbg.file.FileResult;
import com.github.unidbg.file.IOResolver;
import com.github.unidbg.file.linux.AndroidFileIO;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.api.AssetManager;
import com.github.unidbg.linux.android.dvm.array.ArrayObject;
import com.github.unidbg.linux.android.dvm.wrapper.DvmBoolean;
import com.github.unidbg.linux.android.dvm.wrapper.DvmInteger;
import com.github.unidbg.memory.Memory;
import com.github.unidbg.pointer.UnidbgPointer;
import com.github.unidbg.spi.SyscallHandler;
import com.github.unidbg.utils.Inspector;
import com.github.unidbg.virtualmodule.android.AndroidModule;
import com.github.unidbg.virtualmodule.android.JniGraphics;
import com.sun.jna.Pointer;
import king.trace.GlobalData;
import king.trace.KingTrace;
import unicorn.Unicorn;
import unicorn.UnicornConst;
 
import java.io.File;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.PrintStream;
import java.nio.ByteBuffer;
import java.nio.ByteOrder;
import java.util.ArrayList;
import java.util.List;
 
public class kswgmain11420 extends AbstractJni implements IOResolver {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
 
    kswgmain11420() throws FileNotFoundException {
        // 创建模拟器实例，要模拟32位或者64位，在这里区分
        EmulatorBuilder builder = AndroidEmulatorBuilder.for64Bit().setProcessName("com.smile.gifmaker");
        emulator = builder.build();
        emulator.getSyscallHandler().setEnableThreadDispatcher(true);
        // 模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
 
        // 创建Android虚拟机
        // vm = emulator.createDalvikVM();
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\smile\\gifmaker3\\1142064wei.apk"));
        // 设置是否打印Jni调用细节
        vm.setVerbose(true);
        new JniGraphics(emulator, vm).register(memory);
        new AndroidModule(emulator, vm).register(memory);
        vm.setJni(this);
        SyscallHandler handler = emulator.getSyscallHandler();
        handler.addIOResolver(this);
 
        // 加载libttEncrypt.so到unicorn虚拟内存，加载成功以后会默认调用init_array等函数
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\smile\\gifmaker3\\libkwsgmain.so"), true);
        // 加载好的libttEncrypt.so对应为一个模块
        module = dm.getModule();
 
        // trace code
//        String traceFile = "unidbg-android\\src\\test\\java\\com\\smile\\gifmaker3\\sig3_jniOnload.trc";
//        GlobalData.ignoreModuleList.add("libc.so");
//        GlobalData.ignoreModuleList.add("libhookzz.so");
//        GlobalData.ignoreModuleList.add("libc++_shared.so");
//        emulator.traceCode(module.base, module.base+module.size).setRedirect(new PrintStream(new FileOutputStream(traceFile), true));
 
        dm.callJNI_OnLoad(emulator);
    }
 
    public static void main(String[] args) throws FileNotFoundException {
        kswgmain11420 kk = new kswgmain11420();
        kk.init_native();
        kk.get_NS_sig3();
    }
 
    public void init_native() throws FileNotFoundException {
        // trace code
//        String traceFile = "unidbg-android\\src\\test\\java\\com\\smile\\gifmaker3\\sig3_init_native.trc";
//        GlobalData.ignoreModuleList.add("libc.so");
//        GlobalData.ignoreModuleList.add("libhookzz.so");
//        GlobalData.ignoreModuleList.add("libc++_shared.so");
//        emulator.traceCode(module.base, module.base+module.size).setRedirect(new PrintStream(new FileOutputStream(traceFile), true));
 
        List list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        DvmObject thiz = vm.resolveClass("com/kuaishou/android/security/internal/dispatch/JNICLibrary").newObject(null);
        list.add(vm.addLocalObject(thiz)); // 第二个参数，实例方法是jobject，静态方法是jclass，直接填0，一般用不到。
        DvmObject context = vm.resolveClass("com/yxcorp/gifshow/App").newObject(null); // context
        vm.addLocalObject(context);
        list.add(10412); //参数1
        StringObject appkey = new StringObject(vm,"d7b7d042-d4f2-4012-be60-d97ff2429c17"); // SO文件有校验
        vm.addLocalObject(appkey);
        DvmInteger intergetobj = DvmInteger.valueOf(vm, 0);
        vm.addLocalObject(intergetobj);
        list.add(vm.addLocalObject(new ArrayObject(intergetobj,appkey,intergetobj,intergetobj,context,intergetobj,intergetobj)));
        // 直接通过地址调用
        Number numbers = module.callFunction(emulator, 0x41680, list.toArray());
        System.out.println("numbers:"+numbers);
        DvmObject object = vm.getObject(numbers.intValue());
        String result = (String) object.getValue();
        System.out.println("result:"+result);
    }
 
    @Override
    public DvmObject dvmObject, String signature, VaList vaList) {
        switch (signature) {
            case "com/yxcorp/gifshow/App->getPackageCodePath()Ljava/lang/String;": {
                return new StringObject(vm, "/data/app/com.smile.gifmaker-q14Fo0PSb77vTIOM1-iEqQ==/base.apk");
            }
            case "com/yxcorp/gifshow/App->getAssets()Landroid/content/res/AssetManager;": {
//                return new Long(vm, "3817726272");
                return new AssetManager(vm, signature);
            }
            case "com/yxcorp/gifshow/App->getPackageName()Ljava/lang/String;": {
                return new StringObject(vm, "com.smile.gifmaker");
            }
            case "com/yxcorp/gifshow/App->getPackageManager()Landroid/content/pm/PackageManager;": {
                DvmClass clazz = vm.resolveClass("android/content/pm/PackageManager");
                return clazz.newObject(signature);
            }
        }
        return super.callObjectMethodV(vm, dvmObject, signature, vaList);
    }
 
    @Override
    public boolean callBooleanMethodV(BaseVM vm, DvmObject dvmObject, String signature, VaList vaList) {
        switch (signature) {
            case "java/lang/Boolean->booleanValue()Z":
                DvmBoolean dvmBoolean = (DvmBoolean) dvmObject;
                return dvmBoolean.getValue();
        }
        return super.callBooleanMethodV(vm, dvmObject, signature, vaList);
    }
 
    public String get_NS_sig3() throws FileNotFoundException {
        // trace code
//        String traceFile = "unidbg-android\\src\\test\\java\\com\\smile\\gifmaker3\\sig3_new.trc";
//        GlobalData.ignoreModuleList.add("libc.so");
//        GlobalData.ignoreModuleList.add("libhookzz.so");
//        GlobalData.ignoreModuleList.add("libc++_shared.so");
//        emulator.traceCode(module.base, module.base+module.size).setRedirect(new PrintStream(new FileOutputStream(traceFile), true));
 
        System.out.println("_NS_sig3 start");
        List list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        DvmObject thiz = vm.resolveClass("com/kuaishou/android/security/internal/dispatch/JNICLibrary").newObject(null);
        list.add(vm.addLocalObject(thiz)); // 第二个参数，实例方法是jobject，静态方法是jclass，直接填0，一般用不到。
        DvmObject context = vm.resolveClass("com/yxcorp/gifshow/App").newObject(null); // context
        vm.addLocalObject(context);
        list.add(10418); //参数1
        StringObject urlObj = new StringObject(vm, "/rest/app/eshop/ks/live/item/byGuest6bcab0543b7433b6d0771892528ef686");
        vm.addLocalObject(urlObj);
        ArrayObject arrayObject = new ArrayObject(urlObj);
        StringObject appkey = new StringObject(vm,"d7b7d042-d4f2-4012-be60-d97ff2429c17");
        vm.addLocalObject(appkey);
        DvmInteger intergetobj = DvmInteger.valueOf(vm, -1);
        vm.addLocalObject(intergetobj);
        DvmBoolean boolobj = DvmBoolean.valueOf(vm, false);
        vm.addLocalObject(boolobj);
        StringObject appkey2 = new StringObject(vm,"7e46b28a-8c93-4940-8238-4c60e64e3c81");
        vm.addLocalObject(appkey2);
        list.add(vm.addLocalObject(new ArrayObject(arrayObject,appkey,intergetobj,boolobj,context,null,boolobj,appkey2)));
        // 直接通过地址调用
        Number numbers = module.callFunction(emulator, 0x41680, list.toArray());
        System.out.println("numbers:"+numbers);
        DvmObject object = vm.getObject(numbers.intValue());
        String result = (String) object.getValue();
        System.out.println("result:"+result);
        return result;
    }
 
    @Override
    public FileResult resolve(Emulator emulator, String pathname, int oflags) {
        System.out.println("fuck:"+pathname);
        return null;
    }
 
    public String readStdString(Pointer strptr){
        Boolean isTiny = (strptr.getByte(0) & 1) == 0;
        if(isTiny){
            return strptr.getString(1);
        }
        return strptr.getPointer(emulator.getPointerSize()* 2L).getString(0);
    }
 
    @Override
    public DvmObject callStaticObjectMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
        switch (signature) {
            case "com/kuaishou/android/security/internal/common/ExceptionProxy->getProcessName(Landroid/content/Context;)Ljava/lang/String;":
                return new StringObject(vm, "com.smile.gifmaker");
            case "com/meituan/android/common/mtguard/NBridge->getSecName()Ljava/lang/String;":
                return new StringObject(vm, "ppd_com.sankuai.meituan.xbt");
            case "com/meituan/android/common/mtguard/NBridge->getAppContext()Landroid/content/Context;":
                return vm.resolveClass("android/content/Context").newObject(null);
            case "com/meituan/android/common/mtguard/NBridge->getMtgVN()Ljava/lang/String;":
                return new StringObject(vm, "4.4.7.3");
            case "com/meituan/android/common/mtguard/NBridge->getDfpId()Ljava/lang/String;":
                return new StringObject(vm, "");
        }
        return super.callStaticObjectMethodV(vm, dvmClass, signature,vaList);
    }
} 
```

这里也有一个小 tips，让 unidbg 加载指定 SO 文件，如果这个 SO 有依赖其他的 SO 库，那么很有可能会加载失败。这个情况不同于直接加载 apk 文件里的 so 文件，那种情况下 unidbg 会帮我们自动去寻找需要的 SO 文件。这个情况下，我们只要把需要的 SO 文件提取出来，放在同一目录下即可。

*   trace code
    
    用去花后的 so 文件，trace 关键函数，很快的。
    

### SHA256 还原

结合 ida 伪代码和 sha256 的伪代码，还原

*   思路：先看 iv 和 table，然后看明文编排，最后比对具体运算 ———— 龙哥语录
*   sha256 算法基础

### 确定调用栈

*   [确定调用堆栈]
*   找到 sig3 最先出现的地方
*   `sub_2636c` 编排地址：

![](https://bbs.kanxue.com/upload/attach/202309/915510_N4BU2Q52TG6SAPF.png)

*   `sub_2BD20`
*   `sub_25938`
*   `sub_120C4`

至此，明文输入字符串的加密结果拿到

### 拼接过程

*   固定字符串
*   随机字符串
    *   猜测是随机，进一步验证，追踪，在 JNI_OnLoad 里:

![](https://bbs.kanxue.com/upload/attach/202309/915510_NRECNQNKZV3KWK5.png)

*   明文字符加密结果
*   时间戳
*   固定字符串

### END

  

[[CTF 入门培训] 顶尖高校博士及硕士团队亲授《30 小时教你玩转 CTF》，视频 + 靶场 + 题目！助力进入 CTF 世界](http://www.kanxue.com/book-brief-170.htm#h3a6WRhDT9Q_3D)

[#逆向分析](forum-161-1-118.htm)

上传的附件：

*   [libkwsgmain.so](javascript:void(0)) （454.63kb，9 次下载）
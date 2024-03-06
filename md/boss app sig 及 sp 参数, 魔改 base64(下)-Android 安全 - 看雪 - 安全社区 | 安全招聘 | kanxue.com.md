> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-280770.htm)

> boss app sig 及 sp 参数, 魔改 base64(下)

boss app sig 及 sp 参数, 魔改 base64(下)

1 天前 357

### boss app sig 及 sp 参数, 魔改 base64(下)

 [![](http://passport.kanxue.com/upload/avatar/110/983110.png?1702097191)](user-home-983110.htm) [杨如画](user-home-983110.htm) ![](https://bbs.kanxue.com/view/img/rank/0.png)  ![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 1 天前  357

boss app sig 及 sp 参数, 魔改 base64(下)
==================================

本章所有样本均上传 123 云盘, 需要复刻的自行下载.

[https://www.123pan.com/s/4O7Zjv-UYFBd.html](https://www.123pan.com/s/4O7Zjv-UYFBd.html)

[https://blog.csdn.net/xmx_000/article/details/135188031](https://blog.csdn.net/xmx_000/article/details/135188031)

上篇 boss 分析 sig 的地址在上面了, 把这个 sp 分析完后再把响应解密分析完就可以对 boss 的招聘数据进行采集了, 因为 web 端的__zp_token 算法老是变化, 并且环境检测点几万个, 相比之下 app 端的风控就好很多了. 上篇文章把 sig 参数分析好了, 就是请求的路径拼接查询参数再加个盐值, MD5 后就是结果了.

抓包就不抓了, 上篇抓过了, 上篇分析到 sp 的加密地址是 0x209a4,, 函数符号被抹去了, 只有地址, 我们接着分析.

ida 中看下这个函数

![](https://bbs.kanxue.com/upload/attach/202403/983110_68YHQYAVUQ3Q744.png)

400 多行, 复制到 clion 中折叠看一下

![](https://bbs.kanxue.com/upload/attach/202403/983110_9N4FVTFCRTUMWKR.png)

这个函数写得非常复杂, 一直在比较 v10 的值来判断函数执行流程, 所以按顺序看肯定是行不通的, 看下汇编的图形视图, 里面有着非常多的控制流, 也就是我们常说的流程平坦化, 属于混淆的一种

![](https://bbs.kanxue.com/upload/attach/202403/983110_Z8MFNUFUG4D7ZJR.png)

直接分析算法难度很大, 我也尝试过, 里面一个有用的符号都没有, 怀疑是自写算法, 并且我用 frida 主动调用的时候, 多次输入不同明文, 这个密文的前半部分有规律, 应该不是标准的对称或者非对称算法, 感兴趣的可以试试, 这里我先用 unidbg 试着能不能拿到结果. 如果 unidbg 可以跑通, 对算法还原也会有极大帮助.

unidbg 搭架子
----------

```
package com.boss;
 
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.array.ArrayObject;
import com.github.unidbg.linux.android.dvm.array.ByteArray;
import com.github.unidbg.memory.Memory;
 
import java.io.File;
import java.util.ArrayList;
import java.util.List;
 
public class demo4 extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
 
    demo4(){
        // 创建模拟器实例,进程名建议依照实际进程名填写，可以规避针对进程名的校验
        emulator = AndroidEmulatorBuilder.for64Bit().setProcessName("com.hpbr.bosszhipin").build();
        // 获取模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        // 创建Android虚拟机,传入APK，Unidbg可以替我们做部分签名校验的工作
        vm = emulator.createDalvikVM(new File("unidbg-android/apks/boss/boss11.240.apk"));
        // 设置JNI
        vm.setJni(this);
        // 打印日志
        vm.setVerbose(true);
        // 加载目标SO
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android/apks/boss/libyzwg.so"), true);
        //获取本SO模块的句柄,后续需要用它
        module = dm.getModule();
        // 调用JNI OnLoad
        dm.callJNI_OnLoad(emulator);
    };
    public void callByAddress(){
        // args list
        List list = new ArrayList<>(4);
        // jnienv
        list.add(vm.getJNIEnv());
        // jclazz
        list.add(0);
        // bArr
        String str = "batch_method_feed=%5B%22method%3DzpCommon.adActivity.getV2%26dataType%3D0%26expectId%3D856223940%26dataSwitch%3D1%22%2C+%22method%3Dzpgeek.app.f1.newgeek.jobcard%26encryptExpectId%3Da31512b979676b2d33F82d--GVZQ%26expectId%3D856223940%22%2C+%22method%3Dzpgeek.app.geek.trait.tip%26encryptExpectId%3Da31512b979676b2d33F82d--GVZQ%26expectId%3D856223940%22%2C+%22method%3Dzpgeek.cvapp.applystatus.change.tip%22%2C+%22method%3Dzpinterview.geek.interview.f1.complainTip%22%2C+%22method%3Dzpgeek.cvapp.geek.remind.warnexp%26entrance%3D1%26itemType%3D1%22%2C+%22method%3Dzpgeek.app.f1.banner.query%26encryptExpectId%3Da31512b979676b2d33F82d--GVZQ%26expectId%3D856223940%26filterParams%3D%257B%2522cityCode%2522%253A%2522101190200%2522%252C%2522switchCity%2522%253A%25220%2522%257D%26gpsCityCode%3D0%26jobType%3D0%26mixExpectType%3D0%26sortType%3D1%22%2C+%22method%3Dzpinterview.geek.interview.f1%22%2C+%22method%3Dzpgeek.app.f1.recommend.filter%26commute%3D%26distance%3D0%26encryptExpectId%3Da31512b979676b2d33F82d--GVZQ%26expectPosition%3D%26filterFlag%3D0%26filterParams%3D%257B%2522cityCode%2522%253A%2522101190200%2522%252C%2522switchCity%2522%253A%25220%2522%257D%26filterValue%3D%26jobType%3D0%26mixExpectType%3D0%26partTimeDirection%3D%26positionCode%3D%26sortType%3D1%22%2C+%22method%3Dzpgeek.app.bluecollar.topic.banner%26encryptExpectId%3Da31512b979676b2d33F82d--GVZQ%22%2C+%22method%3Dzpgeek.cvapp.geek.homeexpectaddress.query%26cityCode%3D101190200%22%2C+%22method%3Dzpgeek.app.f1.interview.recjob.tip%26encryptExpectId%3Da31512b979676b2d33F82d--GVZQ%26expectId%3D856223940%22%2C+%22method%3Dzpgeek.app.geek.recommend.joblist%26encryptExpectId%3Da31512b979676b2d33F82d--GVZQ%26sortType%3D1%26expectPosition%3D100101%26pageSize%3D15%26expectId%3D856223940%26page%3D1%26filterParams%3D%257B%2522cityCode%2522%253A%2522101190200%2522%252C%2522switchCity%2522%253A%25220%2522%257D%22%2C+%22method%3Dzpgeek.app.studyabroad.article.headlines%22%2C+%22method%3Dzpgeek.cvapp.geek.resume.queryquality%22%5D&client_info=%7B%22version%22%3A%2210%22%2C%22os%22%3A%22Android%22%2C%22start_time%22%3A%221709449575574%22%2C%22resume_time%22%3A%221709449575574%22%2C%22channel%22%3A%2228%22%2C%22model%22%3A%22google%7C%7CPixel+4+XL%22%2C%22dzt%22%3A0%2C%22loc_per%22%3A0%2C%22uniqid%22%3A%2241f52d41-8934-4a41-a389-d8395ab6bb0e%22%2C%22oaid%22%3A%22NA%22%2C%22did%22%3A%22DUzpQpzBYtoakGWwhYSfr2VDxKhBVPnGWdbfRFV6cFFwekJZdG9ha0dXd2hZU2ZyMlZEeEtoQlZQbkdXZGJmc2h1%22%2C%22is_bg_req%22%3A0%2C%22network%22%3A%22wifi%22%2C%22operator%22%3A%22UNKNOWN%22%2C%22abi%22%3A1%7D&curidentity=0&req_time=1709529344001&uniqid=41f52d41-8934-4a41-a389-d8395ab6bb0e&v=11.240";
        byte[] byteArray = str.getBytes();
        list.add(vm.addLocalObject(new ByteArray(vm,byteArray)));
        // str
        list.add(vm.addLocalObject(new StringObject(vm, "null")));
        Number number = module.callFunction(emulator, 0x209a4, list.toArray());
        String result = vm.getObject(number.intValue()).getValue().toString();
        System.out.println("============result:"+result);
    };
    public static void main(String[] args) {
        demo4 boss = new demo4();
//        boss.callByAddress();
    }
} 
```

这里先把骨架搭好, 先别调用函数, 让 unidbg 先把 callJNI_OnLoad 跑起来, 没问题了再调用目标函数. 后续我没有进行算法还原, 因为并不涉及业务, 没必要到纯算的地步. 所以我把补环境尽量写详细一点.

执行一下, 不出意外的报错了

![](https://bbs.kanxue.com/upload/attach/202403/983110_SFEGJURRSH4QY7H.png)

报错意思是 com/twl/signer/YZWG 类下的 gContext 方法要获取一个 context 对象, 但是 AbstractJni 中没有这个函数签名, 所以需要我们自己补上. 在 Android 开发中，`Context`是一个表示当前应用程序环境的类。它提供了对应用程序的全局信息的访问，例如应用程序的资源、应用程序级别的操作（如启动活动、发送广播等）以及访问应用程序的各种系统服务。如果不知道怎么补, 可以进去 AbstractJni 中搜索一下看看有没有类似的

![](https://bbs.kanxue.com/upload/attach/202403/983110_ESU9NFPQJ9C9RWP.png)

很幸运, 在 callStaticObjectMethodV 有一个 android/content/Context, 并且这个的实现有点复杂, 套了几层娃, 这个解释起来比较费劲, 记住下次遇到这种报错这样补就好了.

```
@Override
    public DvmObject getStaticObjectField(BaseVM vm, DvmClass dvmClass, String signature) {
            switch (signature){
                case "com/twl/signer/YZWG->gContext:Landroid/content/Context;":{
                    return vm.resolveClass("android/app/Application", vm.resolveClass("android/content/ContextWrapper", vm.resolveClass("android/content/Context"))).newObject(signature);
                }
            }
        return super.getStaticObjectField(vm, dvmClass, signature);
    }

```

补上后再次运行

![](https://bbs.kanxue.com/upload/attach/202403/983110_9QSVEGK2N4MSHWT.png)

报错 getPackageManager 方法要返回一个 PackageManager 对象,`PackageManager`是 Android 中的一个类，它提供了对应用程序包的信息的访问，包括安装、卸载和查询已安装的应用程序包等功能。通过`PackageManager`，可以获取应用程序包的各种信息，如应用程序的名称、图标、版本号、权限等。和上面类似, 先看看 AbstractJni 中有没有类似的.

![](https://bbs.kanxue.com/upload/attach/202403/983110_TWRUB9F4ENM57XZ.png)

```
@Override
    public DvmObject dvmObject, String signature, VarArg varArg) {
        switch (signature){
            case "android/app/Application->getPackageManager()Landroid/content/pm/PackageManager;":{
                return vm.resolveClass("android/content/pm/PackageManager").newObject(null);
            }
        }
        return super.callObjectMethod(vm, dvmObject, signature, varArg);
    }

```

补上再次运行

![](https://bbs.kanxue.com/upload/attach/202403/983110_3S25TSWQNE28JHU.png)

这个比较简单

```
case "android/content/pm/PackageManager->getPackagesForUid(I)[Ljava/lang/String;":{
                return new ArrayObject(new StringObject(vm, vm.getPackageName()));
            }

```

![](https://bbs.kanxue.com/upload/attach/202403/983110_THSZ894K6KMABGV.png)

`hashCode` 是 Java 中字符串对象的哈希码方法,`hashCode`方法用于计算对象的哈希码。这个哈希码的主要用途是在哈希表等数据结构中，帮助快速查找对象。

```
@Override
    public int callIntMethod(BaseVM vm, DvmObject dvmObject, String signature, VarArg varArg) {
        switch (signature){
            case "java/lang/String->hashCode()I":{
                String s = dvmObject.getValue().toString();
                int hash = s.hashCode();
                return hash;
            }
        }
        return super.callIntMethod(vm, dvmObject, signature, varArg);
    }

```

补上后没再报错了, 并且输出了 JNI_OnLoad 注册的地址

![](https://bbs.kanxue.com/upload/attach/202403/983110_NTKK7DD4MKS6V7Z.png)

```
package com.boss;
 
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.array.ArrayObject;
import com.github.unidbg.linux.android.dvm.array.ByteArray;
import com.github.unidbg.memory.Memory;
 
import java.io.File;
import java.util.ArrayList;
import java.util.List;
 
public class demo4 extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
 
    demo4(){
        // 创建模拟器实例,进程名建议依照实际进程名填写，可以规避针对进程名的校验
        emulator = AndroidEmulatorBuilder.for64Bit().setProcessName("com.hpbr.bosszhipin").build();
        // 获取模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        // 创建Android虚拟机,传入APK，Unidbg可以替我们做部分签名校验的工作
        vm = emulator.createDalvikVM(new File("unidbg-android/apks/boss/boss11.240.apk"));
        // 设置JNI
        vm.setJni(this);
        // 打印日志
        vm.setVerbose(true);
        // 加载目标SO
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android/apks/boss/libyzwg.so"), true);
        //获取本SO模块的句柄,后续需要用它
        module = dm.getModule();
        // 调用JNI OnLoad
        dm.callJNI_OnLoad(emulator);
    };
    public void callByAddress(){
        // args list
        List list = new ArrayList<>(4);
        // jnienv
        list.add(vm.getJNIEnv());
        // jclazz
        list.add(0);
        // bArr
        String str = "batch_method_feed=%5B%22method%3DzpCommon.adActivity.getV2%26dataType%3D0%26expectId%3D856223940%26dataSwitch%3D1%22%2C+%22method%3Dzpgeek.app.f1.newgeek.jobcard%26encryptExpectId%3Da31512b979676b2d33F82d--GVZQ%26expectId%3D856223940%22%2C+%22method%3Dzpgeek.app.geek.trait.tip%26encryptExpectId%3Da31512b979676b2d33F82d--GVZQ%26expectId%3D856223940%22%2C+%22method%3Dzpgeek.cvapp.applystatus.change.tip%22%2C+%22method%3Dzpinterview.geek.interview.f1.complainTip%22%2C+%22method%3Dzpgeek.cvapp.geek.remind.warnexp%26entrance%3D1%26itemType%3D1%22%2C+%22method%3Dzpgeek.app.f1.banner.query%26encryptExpectId%3Da31512b979676b2d33F82d--GVZQ%26expectId%3D856223940%26filterParams%3D%257B%2522cityCode%2522%253A%2522101190200%2522%252C%2522switchCity%2522%253A%25220%2522%257D%26gpsCityCode%3D0%26jobType%3D0%26mixExpectType%3D0%26sortType%3D1%22%2C+%22method%3Dzpinterview.geek.interview.f1%22%2C+%22method%3Dzpgeek.app.f1.recommend.filter%26commute%3D%26distance%3D0%26encryptExpectId%3Da31512b979676b2d33F82d--GVZQ%26expectPosition%3D%26filterFlag%3D0%26filterParams%3D%257B%2522cityCode%2522%253A%2522101190200%2522%252C%2522switchCity%2522%253A%25220%2522%257D%26filterValue%3D%26jobType%3D0%26mixExpectType%3D0%26partTimeDirection%3D%26positionCode%3D%26sortType%3D1%22%2C+%22method%3Dzpgeek.app.bluecollar.topic.banner%26encryptExpectId%3Da31512b979676b2d33F82d--GVZQ%22%2C+%22method%3Dzpgeek.cvapp.geek.homeexpectaddress.query%26cityCode%3D101190200%22%2C+%22method%3Dzpgeek.app.f1.interview.recjob.tip%26encryptExpectId%3Da31512b979676b2d33F82d--GVZQ%26expectId%3D856223940%22%2C+%22method%3Dzpgeek.app.geek.recommend.joblist%26encryptExpectId%3Da31512b979676b2d33F82d--GVZQ%26sortType%3D1%26expectPosition%3D100101%26pageSize%3D15%26expectId%3D856223940%26page%3D1%26filterParams%3D%257B%2522cityCode%2522%253A%2522101190200%2522%252C%2522switchCity%2522%253A%25220%2522%257D%22%2C+%22method%3Dzpgeek.app.studyabroad.article.headlines%22%2C+%22method%3Dzpgeek.cvapp.geek.resume.queryquality%22%5D&client_info=%7B%22version%22%3A%2210%22%2C%22os%22%3A%22Android%22%2C%22start_time%22%3A%221709449575574%22%2C%22resume_time%22%3A%221709449575574%22%2C%22channel%22%3A%2228%22%2C%22model%22%3A%22google%7C%7CPixel+4+XL%22%2C%22dzt%22%3A0%2C%22loc_per%22%3A0%2C%22uniqid%22%3A%2241f52d41-8934-4a41-a389-d8395ab6bb0e%22%2C%22oaid%22%3A%22NA%22%2C%22did%22%3A%22DUzpQpzBYtoakGWwhYSfr2VDxKhBVPnGWdbfRFV6cFFwekJZdG9ha0dXd2hZU2ZyMlZEeEtoQlZQbkdXZGJmc2h1%22%2C%22is_bg_req%22%3A0%2C%22network%22%3A%22wifi%22%2C%22operator%22%3A%22UNKNOWN%22%2C%22abi%22%3A1%7D&curidentity=0&req_time=1709450176299&uniqid=41f52d41-8934-4a41-a389-d8395ab6bb0e&v=11.240";
        byte[] byteArray = str.getBytes();
        list.add(vm.addLocalObject(new ByteArray(vm,byteArray)));
        // str
        list.add(vm.addLocalObject(new StringObject(vm, "null")));
        Number number = module.callFunction(emulator, 0x209a4, list.toArray());
        String result = vm.getObject(number.intValue()).getValue().toString();
        System.out.println("============result:"+result);
    };
    public static void main(String[] args) {
        demo4 boss = new demo4();
        boss.callByAddress();
    }
    @Override
    public DvmObject getStaticObjectField(BaseVM vm, DvmClass dvmClass, String signature) {
            switch (signature){
                case "com/twl/signer/YZWG->gContext:Landroid/content/Context;":{
                    return vm.resolveClass("android/app/Application", vm.resolveClass("android/content/ContextWrapper", vm.resolveClass("android/content/Context"))).newObject(signature);
                }
            }
        return super.getStaticObjectField(vm, dvmClass, signature);
    }
    @Override
    public DvmObject dvmObject, String signature, VarArg varArg) {
        switch (signature){
            case "android/app/Application->getPackageManager()Landroid/content/pm/PackageManager;":{
                return vm.resolveClass("android/content/pm/PackageManager").newObject(null);
            }
            case "android/content/pm/PackageManager->getPackagesForUid(I)[Ljava/lang/String;":{
                return new ArrayObject(new StringObject(vm, vm.getPackageName()));
            }
        }
        return super.callObjectMethod(vm, dvmObject, signature, varArg);
    }
    @Override
    public int callIntMethod(BaseVM vm, DvmObject dvmObject, String signature, VarArg varArg) {
        switch (signature){
            case "java/lang/String->hashCode()I":{
                String s = dvmObject.getValue().toString();
                int hash = s.hashCode();
                return hash;
            }
        }
        return super.callIntMethod(vm, dvmObject, signature, varArg);
    }
} 
```

接下来直接通过地址调用函数

![](https://bbs.kanxue.com/upload/attach/202403/983110_CYUUB97AYY4R3UJ.png)

有结果但是和 frida 主动调用的差别很大, frida 主动调用的和请求中的是一样的

![](https://bbs.kanxue.com/upload/attach/202403/983110_N68BPV2HVAE7GYM.png)

这里我认为是遇到了暗桩, 程序走了异常的分支, 我看了下 jni 日志, 并没有发现什么异常的地方, 然后又去 ida 中找关键函数, 看看有没有哪个函数可能是检测了环境让程序跑偏了. 一圈找下来没有什么发现, 里面一个符号都没有, 全是一堆运算和流程平坦化.

![](https://bbs.kanxue.com/upload/attach/202403/983110_QHCYWU5UG8RK5VU.png)

并且里面有个关键函数的汇编视图长这样, 注意看左下角, 显示方法太大, 正常的汇编最大节点数是 1000, 这里不改最大值汇编视图都看不了, 翻了 40 倍后才看到, 这个函数是目前我见过最大的, 并且 ida 转后 ida 很卡, 放大都放大不了, 果断放弃还原算法.

但是不知道啥问题补的环境跑不出来正确的值, 这就比较难办了.

但是还好 unidbg 有 console debugger 的优点, 可以非常方便的动态调试, 至少目标函数可以跑起来.

![](https://bbs.kanxue.com/upload/attach/202403/983110_F4P887MR3AF3QQK.png)

用 frida-trace 跑了一下, 关键函数就这么几个, 6fc50 就是上面那个大函数, 中间还有 1000 多行属于 6fc50 的我删了, 看下整体的调用流程, 结合这 ida 中的静态代码来分析, 由于最后返回的是字符串, 那么肯定有 NewStringUTF 这个 cstring 转 jstring 的过程, 复制到 clion 中搜索一下

![](https://bbs.kanxue.com/upload/attach/202403/983110_3KUCZYA85SF82X3.png)

只有一个, 并且 sub_2E680,sub_2E91C,sub_29D6C,sub_29E90,sub_1CEB8 和上面的都对应上了, 最终的结果来自 v23

因为现在跑出来的值不一样, 正经的方法就是 unidbg 中下断看参数, 和 frida 主动调用 hook 的参数对比一下, 就知道问题出在哪了, 所以从后往前推.

先 hook sub_1CEB8

```
debugger.addBreakPoint(module.base+0x1CEB8);  //魔改base64

```

![](https://bbs.kanxue.com/upload/attach/202403/983110_8YCTDS2KGREJ4T9.png)

sub_1CEB8 执行前断下后这个结果已经生成了, 并且仔细对比下上面的结果 sub_1CEB8 执行后是 unidbg 跑的结果, base64 的码表改了, 码表从 A-Za-z0-9+/= 替换成了 A-Za-z0-9-_~

这里方便起我先把入参改成短的字符串

直接把调用的函数的入参改了就可以了

```
String str = "hello";

```

所以接着往上看 sub_29E90

```
debugger.addBreakPoint(module.base+0x29E90);

```

![](https://bbs.kanxue.com/upload/attach/202403/983110_2KJ8V53CJYFGMSE.png)

看下 x0 寄存器的值

![](https://bbs.kanxue.com/upload/attach/202403/983110_69EBTTGGVE9UMDJ.png)

是一个 buffer, 正好对应上了上一个刚刚申请的内存

![](https://bbs.kanxue.com/upload/attach/202403/983110_Y5JCGUEA32UHSSR.png)

这里应该没什么问题, 看看 x1 寄存器

![](https://bbs.kanxue.com/upload/attach/202403/983110_5GDR3QK9NTHR4Y2.png)

长度和上面的 0x1e 对上了, 修改下代码, 看看函数执行完 x1 执行的地址内存变成了什么

```
debugger.addBreakPoint(module.base+0x29E90,new BreakPointCallback() {
        RegisterContext context = emulator.getContext();
        @Override
        public boolean onHit(Emulator emulator, long address) {
            emulator.attach().addBreakPoint(context.getLRPointer().peer, new BreakPointCallback() {
                //onleave
                @Override
                public boolean onHit(Emulator emulator, long address) {
                    return false;  //false断住,true不断
                }
            });
            return false;
        }
    });

```

再次运行

![](https://bbs.kanxue.com/upload/attach/202403/983110_7PCFBDQ4TTQCB74.png)

断在 29e90 处, 记住此时 x0 执行的内存地址, 待会函数执行完看这块内存就行了 0x407130c0, 如果再看 x0,x0 指向的就不是刚刚那块内存了

按 c 继续执行

![](https://bbs.kanxue.com/upload/attach/202403/983110_VPFG8ESFBTB2W6Q.png)

![](https://bbs.kanxue.com/upload/attach/202403/983110_3FCRVDYZ3RYJS3T.png)

有结果, 说明就是在这个函数生成的

接下来看下在 29e90 处断下时入参 2 是什么, 如下图, 参数 3 是 0x1e, 正好对应参数 2 的长度

![](https://bbs.kanxue.com/upload/attach/202403/983110_EPDC3AWQ33BKYZ6.png)

接着来看看 frida 主动调用时的入参 2, 对比下看看是不是这个入参的问题

```
//主动调用代码
function call(){
    Java.perform(function (){
        let a = Java.use("com.twl.signer.a");
        var str = 'hello'
        var str2 = null
        var res = a["d"](str, str2)
        console.log(res)
    })
}

```

hook29e90 的代码

```
var soAddr = Module.findBaseAddress("libyzwg.so");
    var funcAddr = soAddr.add(0x29E90)  //32位的话记得+1
Interceptor.attach(funcAddr,{
            onEnter: function(args){
                console.log('onEnter arg[0]: ',hexdump(args[0]))
                console.log('onEnter arg[1]: ',hexdump(args[1]))
                console.log('onEnter arg[2]: ',args[2])
                this.arg0 = args[0]
            },
            onLeave: function(retval){
                console.log('onLeave arg[0]: ',hexdump(this.arg0))
            }
        });

```

启动 frida-server 后先 hook29e90 再主动调用

![](https://bbs.kanxue.com/upload/attach/202403/983110_XM3XRU8W76S8TXB.png)

这是正确结果

![](https://bbs.kanxue.com/upload/attach/202403/983110_DJ79WYUB672DUCN.png)

hook29e90 的时候结果也得到了, 看看入参 2 和入参 3

![](https://bbs.kanxue.com/upload/attach/202403/983110_MAKJZGB5DBRW6MM.png)

这里 frida 是从 0 开始算的, 所以就是 arg[1] 和 arg[2], 可以看到 arg2 没问题, arg[1] 也就是入参 2 的值和 unidbg 中的不一样, 这里后面还有数据应该是内存清零没做好, 不过没关系, 他还是入参 0x1e 个.

这里在 undibg 中改下入参 2 的值, 看看结果是否和正常结果对的上, 改成和 frida 一样的入参

```
@Override
        public boolean onHit(Emulator emulator, long address) {
            String hexString = "cf0a7f3424d12534b08b70deb1c2e41cf585bd4d1b0086f2d1978fe38e37"; // 十六进制字符串
            int length = hexString.length()/2;
            MemoryBlock fakeInputBlock = emulator.getMemory().malloc(length, true);
            byte[] byteArray = DatatypeConverter.parseHexBinary(hexString);
            fakeInputBlock.getPointer().write(byteArray);
            // 修改X1为指向新字符串的新指针
            emulator.getBackend().reg_write(Arm64Const.UC_ARM64_REG_X1,fakeInputBlock.getPointer().peer);
            return true;
        }
//导入这两个包
//import javax.xml.bind.DatatypeConverter;
//import com.github.unidbg.memory.MemoryBlock;

```

![](https://bbs.kanxue.com/upload/attach/202403/983110_38KGN7SCGR9Y7Y2.png)

结果和 frida 主动调用的一样, 说明就是入参 2 的问题, 说明 29e90 没问题, 入参 2 来自上面 2E91C 函数, 先看看 unidbg 中的入参和结果, 这里先把 29e90 的断点注释掉, 这里我加快点速度了, 和上面分析 29e90 一样的分析 2E91C

```
debugger.addBreakPoint(module.base+0x2E91C,new BreakPointCallback() {
        RegisterContext context = emulator.getContext();
        @Override
        public boolean onHit(Emulator emulator, long address) {
            //onleave
            emulator.attach().addBreakPoint(context.getLRPointer().peer, new BreakPointCallback() {
                @Override
                public boolean onHit(Emulator emulator, long address) {
                    return false;
                }
            });
            return false;
        }
    });

```

入参 1

![](https://bbs.kanxue.com/upload/attach/202403/983110_ETJQTTU6EHM4RWF.png)

入参 2 和 3 是同一个

![](https://bbs.kanxue.com/upload/attach/202403/983110_BQAVVGFWNWZMSJK.png)

入参 4 是 23 的长度 0x1e

![](https://bbs.kanxue.com/upload/attach/202403/983110_GBV3Q7SYZMTCB8F.png)

接下来看看 frida hook 的结果是什么, 尽量调用一次就关掉 app 重开, 他的内存清零没做好, 有的时候入参是错误的, 会影响你的判断

```
var soAddr = Module.findBaseAddress("libyzwg.so");
var funcAddr = soAddr.add(0x2E91C)  //32位的话记得+1
Interceptor.attach(funcAddr,{
            onEnter: function(args){
                console.log('onEnter arg[0]: ',hexdump(args[0]),{length:256})
                console.log('onEnter arg[1]: ',hexdump(args[1]))
                console.log('onEnter arg[3]: ',args[3])
                this.arg0 = args[0]
                this.arg1 = args[1]
            },
            onLeave: function(retval){
                console.log('onLeave arg[1]: ',hexdump(this.arg1))
            }
        });

```

![](https://bbs.kanxue.com/upload/attach/202403/983110_32V5TGF2KNK8A7G.png)

可以看到入参 2 和 4 是一样的, 看看入参 1

![](https://bbs.kanxue.com/upload/attach/202403/983110_DE26S5G3RC99U2M.png)

和 unidbg 中的差别很大, 应该是这里的问题

![](https://bbs.kanxue.com/upload/attach/202403/983110_UDZCMQDXXYKQHPX.png)

并且结束后的结果就是 29e90 的入参 2, 尝试改一下 unidbg 中的入参 1, 改法和上面一样, 改成 frida 中的入参, 修改 hook2E91C 的代码

```
debugger.addBreakPoint(module.base+0x2E91C,new BreakPointCallback() {
        RegisterContext context = emulator.getContext();
        @Override
        public boolean onHit(Emulator emulator, long address) {
            String hexString = "bd3203c76c0e77cccb4097ab17cd6b7f25ee6f1a44188911bcc026fdac3a7e285fa3f547da2efbd42caadd8873d0064624d1a53bd3c1a430d229af8779017a22ad9d4c27db5bf143ea3d5efeefc87c345ade7560ae23820a9f481ce26e61161e8ed5cffcc6c3945d714e91385c394d58e68aca9b78654be0b98bf8ffb10c4f4180158c21555657a89690bab8691fc298b009a93f317437f6bfc949a054138fbb0d1405b5be861da73699e4720bc4201012f2d9422a64d693f463e10f51ed7b6a7d6de5195053b7e9f3b6d759b4f7b208d83e33b32b62048d7095a19ee876e7ce84a22f9cfa3c6702a683f9dc9a2d4581f0854ae30068ecdf073592661bc552eb"; // 十六进制字符串
            int length = hexString.length()/2;
            MemoryBlock fakeInputBlock = emulator.getMemory().malloc(length, true);
            byte[] byteArray = DatatypeConverter.parseHexBinary(hexString);
            fakeInputBlock.getPointer().write(byteArray);
            // 修改X1为指向新字符串的新指针
            emulator.getBackend().reg_write(Arm64Const.UC_ARM64_REG_X0,fakeInputBlock.getPointer().peer);
            //onleave
            emulator.attach().addBreakPoint(context.getLRPointer().peer, new BreakPointCallback() {
                @Override
                public boolean onHit(Emulator emulator, long address) {
                    return true;
                }
            });
            return true;
        }
    });

```

运行后结果正常, 说明就是入参 1 的问题, 所有接下来按照上面的流程接着分析上一个函数 2E680

先跑 unidbg 中的, 把之前的 hook 注释掉, 接着 hook 2e680

```
debugger.addBreakPoint(module.base+0x2E680,new BreakPointCallback() {
        RegisterContext context = emulator.getContext();
        @Override
        public boolean onHit(Emulator emulator, long address) {
            emulator.attach().addBreakPoint(context.getLRPointer().peer, new BreakPointCallback() {
                // onleave
                @Override
                public boolean onHit(Emulator emulator, long address) {
                    return false;
                }
            });
            return false;
        }
    });

```

![](https://bbs.kanxue.com/upload/attach/202403/983110_26N75JANRCPJAN5.png)

![](https://bbs.kanxue.com/upload/attach/202403/983110_CKYKCSQFHRAV6EH.png)

第一个入参看不出来什么

![](https://bbs.kanxue.com/upload/attach/202403/983110_SKMN295P86Z7MHQ.png)

第二个入参如上, 第三个参数 0x24 就是参数 2 的长度, 接下来对比下 frida 中的结果, 还是和上面那样, 重新开关一下 app 会更好

```
// 2E680
var soAddr = Module.findBaseAddress("libyzwg.so");
var funcAddr = soAddr.add(0x2E680)  //32位的话记得+1
Interceptor.attach(funcAddr,{
            onEnter: function(args){
                console.log('onEnter arg[0]: ',hexdump(args[0]))
                console.log('onEnter arg[1]: ',hexdump(args[1]))
                console.log('onEnter arg[2]: ',args[2])
                this.arg0 = args[0]
                this.arg1 = args[1]
            },
            onLeave: function(retval){
                console.log('onLeave arg[0]: ',hexdump(this.arg0))
                console.log('onLeave arg[1]: ',hexdump(this.arg1))
            }
        });

```

![](https://bbs.kanxue.com/upload/attach/202403/983110_76FXZHVN5QVED9K.png)

参数 1 看不出来什么, 应该就是内存没清零造成的, 可能是一个 buffer,unidbg 中的也是

![](https://bbs.kanxue.com/upload/attach/202403/983110_WBNWQZABWX4NABF.png)

参数 2 和 3 就不一样了, 对比下长度, unidbg 中的 0x24,frida 中的 0x20,unidbg 中的多了一个 null, 并且执行完后正好也是下一个函数的入参

![](https://bbs.kanxue.com/upload/attach/202403/983110_FR8Y7DPHXQJHFJJ.png)

问题很可能出在这个长度不一样的问题上, 我突然想起来调用的时候传过一个字符串 null

![](https://bbs.kanxue.com/upload/attach/202403/983110_9XH4RS7RA34VXET.png)

看看 frida 中的主动调用

![](https://bbs.kanxue.com/upload/attach/202403/983110_W6ZFZ6CS5482JX6.png)

frida 中传的是 null, 一个空值, 并不是字符串. 问题就出在这里

```
public void callByAddress(){
        // args list
        List list = new ArrayList<>(4);
        // jnienv
        list.add(vm.getJNIEnv());
        // jclazz
        list.add(0);
        // bArr
        String str = "hello";
        byte[] byteArray = str.getBytes();
        list.add(vm.addLocalObject(new ByteArray(vm,byteArray)));
        // str
        list.add(vm.addLocalObject(null));
        //list.add(vm.addLocalObject(new StringObject(vm, "")));
        Number number = module.callFunction(emulator, 0x209a4, list.toArray());
        String result = vm.getObject(number.intValue()).getValue().toString();
        System.out.println("============result:"+result);
    }; 
```

修改下 unidbg 中的最后一个入参, 直接传 null 或者空字符串, 运行后结果都是正确的.

所以问题解决了, 问题就出在 frida 的时候, 直接复制的 frida 片段, 他命名的是 str, 我就以为这个是个字符串, 没注意他其实是空.

也就是说, 从这里到补环境一大片分析过程对我们的结果都无关紧要, 只是最开始的时候传参出了点问题.

但我觉得中间这一段排查问题的过程是很重要的, 或许中间这段才是整片文章最重要的, 因为用 unidbg 去跑 so 大部分时候补完环境结果都是和正确结果不太吻合的, 或者说是如果 so 里有暗桩, 直接生成的结果要么直接无法通过请求, 要么就是会被标记, 请求量一上来就给你封掉, 这些都是在使用 unidbg 中需要考虑的问题, 而上面的分析流程或许会在你使用 unidbg 的过程中提供些一些思路.

至于算法还原, 全程分析下来我并不是按照上面的流程做的, 我一直觉得会是有暗桩而导致走了异常分支, 但是我每一个函数点进去看的时候, 全部都是一堆流程平坦化, 没有出现一个有用的符号, 视乎整个大函数都是在做运算, 并且里面并没有发现什么算法的特征, 也有可能是算法的魔改程度比较高, 从 web 端__zp_token 的生成来看, boss 是重金请了安全专家来做加密这块的, 所以不排除有这种可能. 并且也没有业务需要到纯算的地步, 也没有必要去分析了.

### 响应解密

在前面 callJNI_OnLoad 的时候看到了有几个解密函数

![](https://bbs.kanxue.com/upload/attach/202403/983110_FPMAGY85JV65V5P.png)

第 3 个是解密密码的, 前两个是解密数据的, 分别 frida hook 这两个看看就知道是哪个了, 这里比较基础就不说了, 直接帖 unidbg 的调用代码了

```
public void decrypt() throws UnsupportedEncodingException {
        // args list
        List list = new ArrayList<>(7);
        // jnienv
        list.add(vm.getJNIEnv());
        // jclazz
        list.add(0);
        // bArr
        String hexString = "cf0a7f3424d12534b08b70deccf0e41c8effbd4d1d4886f271cc91ad8137010acf7cf029f23d572d6764802d1173857a0bbee5d4932c6d64e8f0a267648f1dc8e87e250cf5b4ef17ae59028f85c455e56f57f25390b5104b9f17e48e48a63679fd5e80a012b9d5ee065c29a98e09f75c9c726c1b27ba2aec4bb578850a087fd938101c2e59676e57edea8db56b4114272dc4162c7c4f319faf84dea69005a75a2850bbdc9b0417a1f1793a0cb1bf08e05e35aa8ccfc2caeea2354ccb04da54f656fa169353b46fa56f89bb458c54afe490ff1b2038f055391f2d716cf672cdc8f013c8da440dc9f3fe15a4ed2814d1e385b5fce9b77adb91ce7b256a0633fbdcf386078e22464b112ae7b5af7ef9e09579591c842b7f15fd78caf96185ae9def1d288481658ac5692863b9383814dcf31da1c7ae463cbd82997be665c90f462a1bf47d5da25a09782a797519074f06eb311c458ce4d993d2ddac2c934ab43d030720210817346e520ce10408f1fe0ff13d5facca2f246961ed27273aca6c345e431605e0aa916782c884f2f37be079a6462353e6a591fa04e59b8903ddf67794b403ab4ea3d0600059d892f2dec67b4835a7958e1c1a30fc74b90d087ff5d7103f40c77b4b774867468217c2887ae998a53edb8953dc697b5a71d8793003cf5cc806b4af658c2173c76eb79e2db96d4b2fee6889b6e4c266f601905a580c7741306f6c74571c37fcc30fe44f436e9d3603ffefa868375231ba9a5fa8bcdab411c6217ff139eef1224f802b44c613820b22d193271f4eac2951bc69d30a4fb1cb51bcbd11e4b1bf2617734c2da2fd81f500ea8f20d916ceefa6f0641becb9a483347ed1559037610e3f2cef2438a61375fbedff228f652de96fe474406b5f65a5de5e9b603cc569af1c62632c175d53f3f72316a0fed15d1be30cd1c3828239e5849b4dd0c4732e31e6f1a06d8b7f221f0a78a89bd96b61f32a8ca6f1fa96f43b5959eba9d1c114af85f698936f21a972865620694550b9189c9d296ade55706dd5469c18c357d5ba979aedd1fe72a9e9853db7e093fb7cc2e5e7cd8104e44937477bc0d24fdf24e02352ff09c654b36692543efd97d5b5dabb3a1b150306a81a111fd942d947531d71c971d259f77d5bed2de503744955a2f834b452f717d5e7c5581087b79d71fa33e7299a77c996b60351d5c94dd333fc7d0dd2c88b3672f534129a5c80d4c6e594095d8be51c164f489ff4f63e6b923d3d1896b9353013c8b3aa069df9193417043b26fe1dc3628f449a221e061eba36535a7dc9f02bcc38b64153b7d85523796f769a8b50e2e7ec8f69e45dd8098dbb1b71ea09016d746e43b83691795b55ca845e873d206642647b3003d3acc37deb18ca68721e97264e347ebc1fd146fef9cb0056a2002bfe8d21d50c5c9cc197187ca696e0a6cf45b93c37ebbdfde5b7b6bae633c7e0154e93eb81d5d5a507a30d2137895f8ea2936a1c8dc54c848577f1bb56be3a6c756d75bdfc5d2950dc5f8beb89542ac5a4ff3dd84f6c7970cd205a98977bd580c7daf32a3516fa9c112a71bb7163503359fa04814c1ddbc4b716535e7c226affc38c81e01a0314dbf5244b8122c9eb381527ca7bc3682e828c5c0b904c732670dd25347e871338069aebdda83fb9d3e36d7417e1a67c5b47d841f2798f73ca0cab58b5e7a3d3e88ed9d63e2d4aa585e016190723cb658423d10898b161a524132b098369dadb55ae2cf4d20d17efd4f2547685c99f564386570e91489a94b117889924473567ce958010bea8fc6ef3f7001ceca2c44089b0bffa9f1f3ecb7a884db76e9769fd7d512db73fe3ca65992f63238393e310561fd9accb286559ace9689cd7081d8b5417b1204b046ec17bdb3637072fcb9efe62cfbcc2ed4b77229d7220b3fff6fc69d3ff5d5b2ac06bdf57cebe72d385af6e1f22cca16aa4876d4d899005f172476507082d5b6e9b2e3e731f782d3bec5a8301813181a237046a9019288f86819603376698e233e32952fedaaced3c3edc2fde2203c6dbb1a5388b89ee3adfc9d35c74044f080e933e3e7757da7ac2337cdf61569bc5f4ca3ae2fe10914a55de077db24c10363f5e4ab9b3cb9b772d096524e1c9f14cea108b19b45a641ba928182ad8dcf811dbe1e22f3cab734946cca933cfb6d0387e856fbc8618f2a22627255e93c776361840736c5f1fae40f7169ad4e72463b5f1120bcbfec6f7145da862c2a9a5ca7c70ed47c5c01e1d956b8ee752966e7e40c310da1dc271e82b305acc4ba4f12d848a86e56b65fcdfb6e1481db3797df7e998d4925f3f837a348ddf08380b2c1e8cad6cff716ced9e621a696a1106f19bd07cc28bd9620401dbaad22e8f672caaae99c35b47cebe0d2e199f4c90f71865b29ee588cbadc519a4b184c0503065656f0d9db0479e4d1b1917e06cc393e4679612848a6342647d00004248dba8aafd7462bc1ba4a932299133d5105de31489515ab761fbdc308040eac2b4f3f9c51d704edaeb018fea6867ac953cf3e2eb62426ff3eb28b8abc46bc235dddeaafcf384d9475f95b973dec731a262ff382ba8ddd16102838c283eafabf0f180dbb27419fad86e501a8fc399c6eecfb8588e142b0643ef153b38f6da92bac800a2fe10031f82840d86a493a402460eec9d7aed608851c928f5020d6cce94a29a7b4dc888fb77688bfff7811ee8ce583788ac1642406a64f6b1ea32efd3e7e454796832300efb8a48ca4867caa99e7e293c10b530391f8ea1e8e874d291f9d19eb5c23bd58ad2730b8ac3ccb64ad817b089e9148f87dc69cc77e0f0e2bbdae26ca810cfb0d779f4c1f2cad8c37c5b5cd70db87b6ea4f8db5925b1446ced1a243676044ce8afcf079d21df1ff0d996a2ff508d9c522f6e87452937b7418e00adcc95e86ef60ed7f3d9ac46c2eed7f3585edb216b9d85c6395a6dd311757332727ce4ebde177922d8db192115cd10490ffdcc04660111233b04e9948a6d7699c3ed18e124d370482558c5a9fcdd96fcc6ae01ccdf5a38c76fd9340a66b446e3f577f2fa01471011fa18c7e00653a8714359ff1a5e68234a076405fbf502cd2e7d00592662f29fa0c922be2bc68bdb743eaa42525429dd9aca1d38dab800b11602b285c2f1f78099ca89044cccb04ebd17ed053a753b50f15f922c82a52ac5fda5803fd0f589ca8d1cc88536954a2e14e9e17b4934ec0c1bccf61117ddfe5a7eb74a269cc21ea2595180f0b8e9bc915412e1b4176df9c338c64d9198c4019ae3ef3777e637a11e9853aa955bbeb42cc5c6829025f611c01017d17d868cb291e06ed12de27b9f6048e0d48630b262e116ca9577de585778dfdb7996f78a73d6eb00bb9f477bc362c8ab7d8cada716ae3e54c834fcb61b861d0a47a03d120dd5d4da999adb8138c08ac4a282149ca3956d0f8d5a86fcaed5a0e0fc52052969e38fb18a730c2c26ccefb3589b3a75f6a0806a45e87574dcb48f6cae89ae11ef65ab2a2f59a864470e3f673601f01fb564de6b264f249d11a57dff05f59190bb2d25cb0f527f8d99c02d250e0b85c8de7bdea263a034f77827a55be4b1f71113413d1411af2bc0c005fe95692b20f2ea5a380a66d030edba6712683094e2e88beb55649af669c402a6e837a54286d63d4780ebb043cd3ddd87cd9a55c61f067a863231f842fbb16e687ca2f52272a1c6ef74366ceef8b68d16e734ede36a144b9aa65e318b651916ee2965ba525907d819181485122a94213363139a919e12e1dfeee85943fc2c775b3cf949bfac85d981c5f77a28f3aee666bc82b08774dedb010349d014e71ff57360fe5d4063c2f21ebe3af72cb73f195620e319875924c4655f65b1f59031887f32f890e498c27692ae873ed605d61b3b3f6aee3222aff4f302d26ee4b2ae55fd7236b24c45cf275219b26e7cb0bdf17666850750246a4ed47dae92756b1f61b123b121abd226c5e607cfd6617d33fbfa948f54bb7566ac79aeaa536b7388920fb9961c732db9b210ad6310137e4d72cc1d02498030d97cea458f2823e2b825ea59f70bf36d35033bfa4ba8c5d7869a708a7cefe00cc334b4ad5c6acf72177bfa38225c77a741baf82b3483ad9bd5954989f24a3c6a26eaa83223fc864b719b043ac45d1d5349d57039ecd6305562e6b3d274043050c68724d25114366d80b4d724ce02ae033c3284939588d00c449890c2f183bc188c07f204d23a1484dbbe3f068861841afba642f76375213d6f588a0acbf308715e32d27c3ea3a03477709305d6a477b7fde82b82485929a172c40dd6262ae5ee5e0c6f45e894cc125bb8b395af97b14420898045c2dd39c5406725f70d88bef58015f51ca5b3d90bac15dcdc53163790e7e20ac666f76eb0dda39443a1f475b05fbbface3b400c5d97dc35b6daa1543afd71829172aa3c0a03653aaa63326e015923dd375e9aee8f10c6da3a00c6bf3b14c8c9c47f40d3a90f557d5a2c10d2ae301c32f0a6b2bad55849a6a525eb8451f0f7825e8566f29edda9f87ed7562d230fd0f2dc6e7fe4dc612e9c07c9e62522652accc82960377c8459675ba9452bf2b784807fadd5da027b20d919e4ac76b310b346f5b71b69bca32e940e5c6356fe56bfd1d48f0c84c756d994166c1c670a68b06a90be7dff1bf8f7fff585873e634b79c1e547b3295b3bcf31345a7399b3c6e395a6bd7aaaae83afd5a075e54515ea56571f381bcbf2a542dcf300cefb646f529df7da148c1a07921b525fd1741e071c19a0abf88a980a7f6173096d6196c364ab033f1ccc1db2fb9eeffc8f3f5d586d5d3e86bc9fa43c7216e7252dc18d1aaff8978162a44a72adfbe987073cf5b6090ff4805877383a0a13ef1ff16d137250cebb24f06af65266c90f51fc6271d6c88538876581db5b6c8a8dca1547ff07c8e866abbfd616698c785fc8f8dda449786e4f470ccbcf5f0853d5853585f5d867910713f56f9cd041cc78664073b6a5171898a4a52dd1c5e7da367ddcf7fb1fecc971d826c90d1ba786175b5b4fc235529385442faeb5d826047a320ce3d8154d5fbb949a8af2336621c8ffaa601676f9ce27bb966d67f57dad7d764350b6db97a8e8f151b8e2cddfed00f6f0238f893d149f3bb194c81dc2f094552644a52509be795b5d35a160b5c7d796cd035c6a14a2645137eff6130c6c5c8c004345261b2fcd53aa6bd7015fa7ffddef1e0a586faa25fdd8660b78489bcbe62ae181cbccc44a6cf1e425df26850cf28a1e7e01b10c61f92afa8594f9549dd15a56bf481f9f11a1ffb2333549d3c4cd7097695f8c65f038b6c30da1c9ead76770f8941012c3f7bdf308b7b291f5a86f31b780ec5b4b285acb89ef404cd95d97c19f3d73eb05a879a74293f204d59b49c5fc3053370cd4e3e42cc5611ae584b67630894788b88eaa8f5b1151a1e4106866895475e54f2112797f20ee308b2bb2d34f2a942be2430127cc82475e736c0dfc54ecd5eb0d6ae9078a5d2807f70864c626028ffbdb45fb0579930110203646fe05bbd01a929f983b426abdbe59a3e399b10c16779d4a4a2ebdc8adb4a09a5a437e143fa27bf64c3d580c02448db40011b777915613875e132bdae513703b141b40dad33fa0e3a7319255b21f4cce29e8942215d6e6ebe5c0aadd126c8d412962792864279872191da039a355e2319390e2172e50285468d112fae585162c56b31c56828037f544a425146f274a417b6d245799ece37208d17ba8357cc6352c35da420e3ac0c97b83a6bf3e9fb1e463d3a2633753ff5c4cdf6b43fa3d176f7ddb6284209ceb682ed51ad6aa4a0e78fe98a40a7765505595121c2b8b9b5b20c7d6003a180098ca87ef92a083a83f405d927cde17bd5e2e145ee7961d75a6d5867c9b35f38e9129d4944d073755e72d472ae4f49c80b20d3d7056081f3317ad39917fa738252d0a701cc0a5951a217c30f2ec2c50f869ccf668ad016615cf0494c6eadb7d90392888ae9b62446bdb56b28b736f58ecc52955601588e4fa42a894a97af0d2f3f1f9ee832b7820ef02930e6b8505eb84ea160f22594f0438bb5cedf0a1f3944d8c63a01a9f8866024e9a0b9ce2b01ae0edfc3ec8d650d279cc484d6123f9489f69307316517064d0ef67c31de407bbe7df6a0e5e66f7cd7cb0d4a6edd3f50d93963f312c18b916c642c30d318e262335beca0f317c8075f53c4e6bbdce90dc99aae6304318f1eb667689c020d8ac0fd56b9d6cb56422e0b13835643b90a1bd592c9543400e3ad8662268ceda72a6e30ca54dcbfa5025c0f6b5c07e8ae2946c199b95ca1e69f03dff38f81e8ade2d2333edfa7de7b85032cca59de42e793d561306d5e2e36750150e626c1216b939557ee6aa8560c2dafcc3632c76e559e9da9b481a90334e81c1c501e06aa4e91c03fd82dd97e8fd8ff494a6cafca6ce2a58640640e930951f9b5a2bb3fd60a7e9eac52a2c4d368d43445d29aef3bdeb693366006ebe00d19336161f88a8358162a8479d8fe150cd4ca1b59fadcd071cce93e82bad71bde6144ddd930b755f474011a7bcfbc1df58fb8b52684a1b41b66111d348c33e19058c09936bc0baaaeabf4cf219e609ebb70492dd451f1ddd4377b8b770a59036bc42da3273654b806abceb666157575f2e145ee1f8026de29eb984658fca27c14b769bc84fbd4a5a8f99b6e13a63d36870550e2a0caac25b7290c7d034312f9ff6a7ea98af1d974b6f75389532d610c2d5c2f1454aee223780c653a00ecba8d81c1d6d1c1cc34b75f340a7a987374dec7a46f23f2e2cf70bc817d26d044169f07d876cbf425726f8e0303fdf2dd447a558ca902889c50186b1c3e3f032e6127227a836b7af027e79bf45632bf0d93f98918850df89dee2c6f89320510388f2fe154528cb23bbcc117e71db90b6af87a71c527afa21f096d262e403a9063a2fa808be1c66f0f290a521f05697b691eeca26ef839f29f56d06bd4494bd7c1f98659d0668eb4c4a85428fa3ea2d159dba24032eb3b8efded07c22c53ef7606313ad31bbdb402903313fbf3c6dcce534b460fa24c9465a5e19045edd18225d6831ab0891b072b883246860da9e193c3ea108e65a04a39677a3823547bcdd257673fd9d86b2460ddbbee4930644428c65035dbc88a68a0a4a199c393e684702e7faeca39b3276bb7c950589a2e13c39d7bb9e2f350eccdddf9cf95bdb7ec3cdfb37a93cbad40ca4a25d01dc33636fd6a435030d08b575de2dc3edbbeeb493989bd07fed80a799c949add02c0925885becc9b76386f50167a25e6ba0f6fb34557728eb48617fdf8724b4ef0aad4a760159e9e119ca2908aebba3560c107151995ae896a2c46ac59635da7381eb51eb8975fe52eaf1909b92692cedcde2778bba14d741f9ec5546e784a6bf9f39fa8e7b65f9a8291027e38903bb5edfc914dc804f55397d21cc1d7158f31d66071429b80fff88150affb055bddc07d028ac8cf8877cdb8892e786d6667e9523067db55f50052f1ce0305ede1adf5bbebde8e8272fd57d1d9578e527735726310883764a9cb9b059a0caf48734584271aa6203cab8ab138fb0f56abe7a8258f534b3761ace5c632bc0f29c93796f91644968a510796d0d326f7ac91059f35fd0b213305f143555fa5d9e26ce4a80e1bb49c5235a4ecce1796c65dbd6d51207d70ba6a396210039e10dd4590f11e6a24c98e39f930ef83fae67649ce2d71f38d09a91007900a98f10047ca7c1a9c67979ebff85691b893745d0c91fa3b30658a19804a7f0bbdab942c844df1ccb017b72ce491e312b7d98ed16a98303fbc1d8caafed403e08894433bd787828c65056b708e3ad7b03a8209858d50199a27ea7f85e69429d7f3f126d7cfcf3d01bca7146a831a3e1bc717634c02711edeb2c6d21e3a04206ff3c6ceee113df821cc82f6fa4d27b60b330e1ae0f2cd499c45fa7ef4bf27de1b164d1f14a1d0b55740ba0fd5472354f8a23c9c9d7d6dd95ed510b559b6c15e61947957aa5e6783a6bfa314f3ab2cadccdc13302e79365356dafefa1fc1e6a1a6902eee81e179bebc94cc7ec2520a1901367513181dfa47d73c10266a5680e91c3e6855266c84acdc2a26e81a4aa5f5edac7861c9be5159fb5c05a6be04a3ef998cb49f20c90510bbc01129a3503b6843ee59d1010eea9a5af01cfb88374576eb9884acefeabe3033608328361d866d8ea5edd751f5eab22dd10040754587de8a2422c244a1f323002ba91ce4c9930cdf67c00d260e87aa3d8b981f3fab206d243afbf4c9bce893eb5e2b3f7eba58169815f6adc35c9d06e82bdaeb3107447899caa463517ee2b7db2bc189bbf7d68fc767821d38d286d851a3a81b62f87a1b619217f55e30439f7ebce74c9959c096fc7ef7ba33943cbff0df688a30981f4196caa7cec6510e92e9b2fd266c7f55d019b97958695d2be810a3aa620e0e5e95fe3d14d4b2cddb7d6b84174e1dcd97b3be4961ada8fdaf90b7d23f4e74b95fe2ac1db20b8e98666f45df9aa6525cd4dc436415dc0e3582740f04d0e2272eb511cde714641c4d54d80865907a90cf36e91e23d9eddd056ce7bb61ff35e7104012f061a6b5956ff7a9572d18c6db563452aeab97302979c0f96aa4de42cb9e9484104d2f6176d6de39d5b98dd2067224cf32c197568c2c810b1781ec62906bbc344eeffc67278a87367827fb6265aac37883122879ca01a134416e9e24cba9f6585100da3e359d7e5633fdf4697f91f4b07be204c920c38652a3dc669c389a396fbb628e7fd89a4afd85dd4152d0d1b8e53beeb68a67e8a3b002992f68484d3bc583c5a506254a6765b2bff004f059371075031166f7bcffb2ce8f41acf97068aa6c771acf653729422df7d75fa90ca7fc2a73ede462320aec3b364a9b04f4a7d3b12032c990361dc2a90829a383fb57e597b441615415d103383bff870102be51413cedf7a920dfb25f30b9ef8705c53c1e26eb7a631cb9f1eb0862f5b75f30a26305534d26b48b05c8577ecd3a66beebbb8f6f19a45835ece7ccb88f3279e20d0ce5f4c042a6e127d28eaa3a43d9cf2db53cdf3d5181ab6cce7064a4f11fc25d79631defed781ee0274b2d27ee031fef04915d1f384a4ba77e1d107b23fc3eae66b8d1daa4cf15dd4439856e9d2569e5b65f3efa05d82877427ac877858eafcadeb4e5e342e095d4a08b8db0c777547b699d9c9dd1c7c5039aaa0fa74a47b9672043ae90c8a36fef4443150e8338aa33e2eed22c9cad473eed336803811eff5de096990598fbd3f46c3acf400a7b2295904edf35c752ef75fd10b883b63cb5883862959de6802cfb6b477a99e2a0c65a65b3d430ce74a654bbbfb5f597e57922be40f909356ac2f54c9023f5422625cabd189e1ca4789d80892071abdc10b812824d71023baa2f7d32515b136fb587f5a3bf0c13c62d19f544f0532d20884c2158e6c8aaed87b7a9074e35ecb4063c5930c3709cc4c53ac109c7c7c82744639135523377b05d356dbc84bd278beed64a7fe0b07778fec57466192ec9f479cebe6a63ef977b1ee2db0977669139659139707e5d9b2142f585607baae84afad3460b3dc31112c80ce3e7f6bc9b0d71089a1605d1cb0e7ec8334ae26a0b0be669e69cfbee8f9dac7a1ea703b6a64291e9f2d688ff4810589ac37ca215193c79fb5b67364e8336a36b85a891043e7d04e417e8bad57873fce04c744c80282bf1a92136fdaed734bbf618b55ca323fc0c2335cbf09691793a2047beb5a5a62421c3b8d4fc1dfca01019ecf60d3b1d42697b49eeddcab843bbd422a5ddf84cbcb0d6d781106ab79522bbdf279f531f750df77fabfffa8c070f02fbacf52c04e53745d38d689bda78d2a48f4e27a1015153729d89e3dc35f8cfc41f81c8744e7e024bf40bf55e50577ce6cb68d38e9e02aa74ef3908232e7b803c44bed69fe44827c84edc9eb24013a2be671bbb72a7acb346cdbd5df9d44d0369f93b4c04451dfbba084569b195a4d6dd813519a9c9bf4076dc61d540e674aa1e960772574c5631dc7f3719c49a12fc13054f5b71364fd021c3f5e6e02a91fdbbde355075584f066078fbf5606163fe8b562f5cbcbfcf21c2f5b4d63f41488544fd09c1f89f028f3577688806ed1e4937590d2bda7795351849c64846cf500ca369387dbf805b203821d37fe693cf9c980a1bdc040d6fd007809e8fee33005afb736df56f1015a7a2e350be75556cf571b30cf91bd36e6f0ce3cf05a4d4f1f0bef7fd78150b906599c3e602274b8d3d82b09874293c75f1372523d5b35f9c5117f76da52bd6c58760c50532c054ebc95e779e6fbcf876497aa9aae80b8ba64763f31e4be34c2dad4bfc52ddbbd9952cd0e31be13e32bd074a0d3d77835119efa61d76c330ad3317c1b3e0b3b16b7a69e620234fffae3c69d5879d3a8108645065f5fe3ad5fd807466a2ac85381062d1a6d289f37597a6e77a3be7e55172bbc0ebb34d46ff25276215138c5a96524a882723ea9090214838562ed76adb77b51617cf93b8227c4dded0c4d7f84f2a25a449094557c67e2ba0b44db7ac7d08ff356b4de50c792e83b34f9122793553b12eca9bf7a7f13460bd5cf792938c94dfe1cd0387a8897e3692f24aaa60d0c99862a191e1c234144dd8ab05f03beed460caf8593449bb537595d02e024daa39c239e438df30774d37cdc0bf56a986aaa47aa58de832de577c8fbf21069db8e74ca428d9f514c21282f1160f79598981ab52eb679efaabba2d570d5d82fa9aedb78c091102dba20681d2391f7f55feaad098614ece00eb446f0e7301d0159c0dc63fec5867aee1e0993d8a42a3ac4eb367a3c82f4c0d486be9047c11a222ebbda2fab3c66f25a8ac2928620fc174ffdd2dba1afd1507fd7474551f38c6678b8c8d736e9352e10c4ee1b577c0d8a3e10b3e33db68d7153963ebba3eed9be7d0903081a125b3536d457ca94388fb64a82f8c93481df42a7522d7605d6d9598ae8de7eab89945e01f15c7dc3cf1c30125394e90589d11729234764f35b3a9824da53e86a16b1a341a204d798b5e8af88425602fd56acce3f6b005b49d100e64f404fac840856845dc487894feaa3690dd7fccd8606a1249d6551f7f2d894fcb291bfa6c0502f7c321bbc72485b8b4931c42b0c4840ce0d619c2ac0ebc7830593e21fe08de254c76a5c76df1b1109e6697a24d1bd21610ffb287c241004907a11ab562ab7dbae4c4e83ffa6ee0b89802237590edc966f0ac7c453c447a0dfb55f6f605a5ee9df051ac0466c8d820a01addc3a025544696408ee2ebf5aa18771796ef9fc125584023bc8359fba1e5d28e5cc295ca80cdb14395422f8297b757dd68883560b2a9f3a28456d1228aeacd03648dc9e9d42905b19410cbcc0f2064d2ae528ce5c9a97032f6e41dfd7397ffabca3db292441784f5c89e4c5343e8a4b4ac76d63bd65b489e689d5c206bd772b569326b6d3f2a41e5f817128d9e1e196fe0d4d5af1254423470ad92f3a3b7a5c855415f97d4daf7ed395a9f176f7ea2ff2d4e26a36be59fbdc64160b286e16d9425aa5cbcae9f8a2d5bea788f68f5cfaaf891f1e226a5dcad48ac66650efcf358490815a31e332e01b9335508a5540a5a5ddbdb18bfc49a46daa2ec5e58e3ab3ac29a672270d321c78036c86030293cdffbd5f7a85435977a0898e32a8a3f296a095d5e4b6f2388d0f0a14bfab15fc425afd0c084364c58dd3053a9ec27800dcf83f6e503b950eab93a655754d657f9b01f547a03b0ef6aa9f61694958eb4a22e90049c03fbac6671162029b73b0c8b7ac658bc4c10c9fef7542db6cd1d8ddf02b8736d48aaee8796e294f8c4020d4106192f6fb4b4bfa34cdb62f559e0a5a098ade8f493e991d1afbd9829e3df7f4a3aa9e8924f8a9cafebe9221aa78be7c26337d11f46b155fde26d0cf60f89243b5cf1a75c8bb468d9e5732d3e10cfeb4be07706ddadcaf80f481b926ec1f5715c54df1ac0f1ff9db9e9ab910b7ecd4344af59925ee25093483ad8af5d0c492e2d4f161ad325ea03fadbe41076fbf35728719399def20974bf8cb1121b29c8db7ac956da1d97b01228d6719612fbadd5cc361ed88a0148d29052d29a255a1649a6c8eaf5efb49399553dbb045828e159543c4238b34a4c6c580b31f6a3cd28f29fee23556629150257bd9f197c75eda0fd652c362042cf640e3eab60a8f528a76dd2c12c4a29b7bc92833468c15527cabee2d86154808467963720738412e641e619f0467c7e3dffd26401520277fbf837cf6eff4060d1120892ed65dbbef4a80cb9eddd785035b74650015bea31a4e67a03ba460e522569b12c33669679f0ecd9ea7c009ef1e25f9b664ec8dcb6f6182ea668019e661577e2e929902d0dd3d89228d331358a9576e64a84343462c5016d9bde9ba17e4d50c3b0de6f92c646291303fb81fa85f83b49049801e1bfb8e0cddd519a44818a6b8b1b931b684a03672d45248206bd8caf35146b5b7ce43c82776bc84b9477d63ffb3f5d89932079541eb4f815f7972d3f7f83fcde35c0d00c31855b39c7975880ba237ab99973d9b5ae2b572de187846f6e3cde137cd4499cb53cdc9e23e727feb99e9e10c7c690e060a47e8d45505b1530462a979ce4074e21528c487f2fcd678669e6af0061bb35d98de904f66c2b4c5d4f8c570dee93a7d1433fdbbd5a6917df5994913fd72a6d4bf6fef644f8765f03e4aeb7197ac85a115c95e63ce654896c2b51d51182c9605c7ce459d4e51b47bdccdbcb53a1b654ae55728ced5cab24655134d593b2cbe972b29b68ab650bcc90b04d7609fca238118c5fe50ea024e873951ee8c218c18658677f2b0580e6a43541c371e608b05a0c75d16c49db99905e472e5cf5edd8f6f5c9858361e69c1a354228e6ef4f6135740087ea2b08345d30ca493960de2aa9e93381ef7aa643c193557c24b8e71cd41278abd09c058512aa4433d05dcb99bab7185f91bd0a34d4127538f3d0bf463929a0cfc1ebb5d2180e90071d8aded22990653414c29a5bdd119fbe955c3f8da5f4828fc735cfbbb9d8fcb2c171d1e61a723d394e80d70d855254f4fb65fcc127790a5c4d30f493ea073f46c7cdaf0d8ec3ffd769cb2054d824b1c13133ca722c0c819cad1dfa7ae258cc05541dd618fe5cd6182d035d23cfc7e91dc2241ddcb22d44c52226f5e3353566ba9763bf46e4ec44fe9becd0c8f4afa6e2ed738e1aab8f03c5831e4642fd007e78b812dbd055e4d9ef645e2dc931ba005c7b37ccfebec145c2f77f4aee62ca19f1a86e5d16a85f17822ea6205c93f2f1f6a2c406090cd93b96246d6431dbd27208116d5442a6cfeb49032eb875d43ca5d9bc780d55585563d052bb9952b9395a4e1ba2fe82f4ccec2d762f39f8c96335a9597357d5bf1c4fc09879862878722fd699de58f65578318663044dee9385ef52affa8fd1d59b83a9a1c962cd137268e75ab5217a2a3c122f8d8e7bf7b79bca1adcf9235f8140f0928136307afad086768c39793022e547d555484293a4db6415aaad1ccf8860a8c0ccca2a23ce6405fd11c8409746c1db84c015b009394ec4a22d8c9b8fa9c06a80b35cce75fb73507a553d4653457d6c2f9e5a1b5efdccb1a95690532a9fd430bf3d4d2dfefb149678acbf808f6f88198568b4a9a83dc0e39167ebe9beb00bbe3feffc9d49bff54365ea09ab825fbbe050c004ac8eb0e853e5f86b069c7456d65e1efd24348dd3090266857d33f8bd8325797c1f28e81162876ac38170b710b37f7af7c9d5d7a75b2d1eca99a4bbfb93ae6faea6eaccc69accdb7e2ca63714add0b89a7e178737baa71903c763967b206fb7c3ba4717e2e8514b6679380315b540a455c7c5d52d98c02acd9f813927fff27643e5eb7e29a23b348899443c0a3f50c5d0473bdbb51ffad4c0078c5727e0e45f0d7319e1135196957a69738e2471108305d189f24801181808c80c6039fa85a1d91a7d79cf4a7b7ba223a3e90b0d6cda9647b311d347c5c4d5bfa27a8550ce92ab2a4e92f97f0a80b36aefb42ac205fa621869da3b36b7734f1e639ece98973526bbe54c0cd03311c5b538b0fd940f29fcd5c3ecc42b6023f67fa65fba9ed7d1e04557fc9cf2386952acbacb4b646cba555c162ec082ba2aed0fcfa9cb5602e878c34fb130133c9157fa25f07d6f289a28bb539fc75085213ef1d2934c0bbac5f1cb8e2d94b7dbcfbf6459653d8a478800bf60edbde0fcf9df803643cfd8dab74892fe093c529f32681289f6dc98baad518ac488b3b31ca234bec2fd7f98922f3aa5982a6ae5c702659aeba6cd2de2fb9efc23946cae03a78d63c49eec8096ff8260c7a42b0420af51434aeab2afca08e2c416a5ccecab5905a13a2f0ab25fae377d4876a892f639b8b5ec2ba07f26774ebf13fb6e7864795e901396784e208d81886356ef1d0fd563b7d99f81150841e59cc0b780a8b6fdf1441f15637772cf3a5113e53f8bba93ef8c159276d985e318aab057fd6933d4da8b09e9e258b446ab5ec23a94aa556a75ecd9c9be4a1a6c55097070aaf2d31486b511cf6ee574a74a972990f5de642bd21934472b1d437bba15bf498b09e00f9fbd2c5b5aeedd7c902712269a0db1960abbae41678f45887f50b8ab099267fdc171b9ea6a8ca249af819fad31b01b9a2e88223ca3e9b189cef194647fbfe2141e49fb1789f39e6f43ee27d0d4decb1f1567a99b6d242e414fce5dc7ecdb894098adf91aa78e134d551a27458d9b5cb651e1a42d798420097849cec400cccc25f60a5d432af29b9f4d042e1b0d535fb56c1582fda094a218ad04635b4bdf1550229b4968baed79a2039ef9e87fc1212e7db34625234c7afed0a349ec1272e8f9d72925e519ae99253c7d07ef48a42635192bbc18b156e9927d9ebba4793151742d0289f19873b6079f6ace4b40644bf4c45a2699b4624f9000125bd0d2a7dd4bd8038bb2d0668a9b30bd81afdf17630a48bc9a3d8b1dcaf6ff376d511c9256b55e8f958c0b8c199a4b35ff284e2cf2a0a3f0cee862fc37df2fa5c40302196939c526b4198ccc7f7691a1c15fd07ccd5dd68ba0c735e3cc6e8a9f1af1d03dc48222c789d563a37fbd8fd39834d1c257cb76ffad3a901a1443c941a47fb74993d0031c6671284f39e910559628dcb22074bddc3db51b82175851a9efa12758d697790649376a7d216ab99d5e882b8831b3dded8d427c17b84dbc7d3ccb4427899f6be9127a93f4d79dfbdcdd180c97822391edc78f0070c50d754b0716fdd21c19045c71864da86eedf6170c4e43061b13214c20ff98656e5415a5c451a1e6a71af34d6d67e8a4d9148bf78e5957fc9b0a0b6c7c13c351b1f29ab849b450ff8c867d000e53c49bd419cfc5f6567749d149a4c77d8c2bcf46f0ab0cb13e186d302e836474aa2abdbf0963a2395b9c7c11acef54f510b61e7fc0b07d78eaf3bc1bc5da9dd4a66690ad02dbc863ca3d498db432b704fbf1e7f6af3b8d17b1ddd84822050f5075370928248788aa41a6028cc2d97c24d34d7694d74d376bad5b6c3a39a78d0cdcc43b05a14af0332d87c5d5792839750f9f56efa21160d1fba8fa8e3f0af2df4157b30cee109522671d9d81e4b594d970c8792c2df3b36a195c118bb664319d3cb7822a970ee68a95921d71458c6bf74fda1682274a6d26f3a3bbabe421f8ce5a2450b7acc2706d0555fc09ea0d5ef505cd81991aac0d33e2d16f4b22d43082e95db86cbdd6b401f10763c3acb480dcb6e45f358136e55843919d8fe988f4ee3a5d7554c7f33047309fbba696ea223142dc3949712e43d9b3fe2dd567ac4047b861734a646eabdb3bd7206d5dcf4d5cf871de3db16250c1f1585310e9f0028268534d04b48b92222cab015d20e67f4c9c1cc503fe7cc6e0c5ce4d0437bda885b0087345165f2630263e6dbb78c7dd6b17794beda3b2884fb2d89fd94f4ce0f4b3ec5fe56781633e132fc135c7fd56e4bc1c1988e02123dd0f8c9dae5412d053c8ca0de3ce328cb5aee752bb1ac975546d9b30ad9073c956c61aca99561b5818834aeb2f2f0acd2f5d88243197a036270efa089f2418998e2fab57b0399a34ae8ec54b376e2f318618f441c80844a1b95a1ab5f253cecf2550bff6b58a4666f8c305791fd37e28819985977c027158d47e5c5f687c43a4e481d54e971752ea538d95ee08ba12ef5d6cbe661fece67b539c565a37ab861c83b1a139fc5fd5fe4598a53eb10a9398b572399fc8c8345e95a7196dd91e16229581f520f1c8d0c9e7a6c7de506963a7745f75ca855ae2731934337a58c6a0d813bab7c59e779d5e8fb921ee30cca0788eeee8d64d969c9f5c8ed06f7a21809c7a4cdfce6395fd66e93d8cef4f7cb9080d54b6ecb069332579bdf0b5b6d88cfe842fa5045dd7bc734b0b44e23f76012bd8874972ec3588221eebceffd07d2ef3acfb920bb0bab8be96f0969f3f6d791599b3d8bc6695398f94a0819711174c55bcba3e04a5ad56ec72850ce9736e00da3671a83479fc086c3ebeb01949100727a4b0ea8a16bf68c22344407da944b4875ae0c205ff34198802faab626326e2fe38eb89c483242891c270e89d868201f6cf0e97723e89044be5e98f5e1ac7df8a910c2a2a7516541590b6f9b172d76af5d200afbdb36276db2bbd288268dc07554da93ee72173b158a63b4fadfd443d66bfa4e2c5f57dab4a1461af51aa9590e34588a82d7db0c84f860ec0204095de59bbe7fd3f2407dcb1c6e14c0e8eb08714eae9644e31c51dc0bed3472ccafb89449c51460dde81b774cfea78bd20a0d398da4589f8055ba798210daaccf2b65fcb82a69c64362dbb90f27e71d5811171f4131a176384a00566654bfbb71d9d569aeb84650d887943edacc28fbd135d13af1344ce1ab2f7682847f10994f0edee19d0a0358b1b5f1c606f0841a177489b386e77be18aee8cb9d598a5b8177a09ea34bc8c06f3914f2604fd006a36e42ce495e2a71dc5993b574c90ab3885d055e671d95fa9064c34b04da1bdf8f478a5b7ec1be3f67818b6d49cdb9a09df87a2b772a16198a3b4956ca75ed4a2e9766b6164e05e0f58ee44f7aff4cc89a0e2a7dc326d91169c3bfe4eade42bf3f27d46dde5d3886291b60778e4141ff2533a491aab897ad1290faf24af4e4d60813d50c68151b158c1daf2570ce9b8d4a0836eb2879fb31feb2b5b451ef83131086bc29bf2637fecdb17d6b81abf5451636abed54e3cc22654caab841bf6fffd39a1ace16aecceba1437fd98d5c4438590da5108b2829dcb4269b94fde32782a68fb1e6769ddd7b3ad4bf4a6fa6bf7a07ba5bb2d649edc80cbd655433fd4b5633ba7eab3bb4e001a035c2c7809da413c767e60d6ec82ebf3faf365fa801c25ff09bb020d887972ead46b1f84bee6685e907309215fb2ce758d6d145e6b0cb50e0c51be53d4f24a4b07fdca53418a309a7ecbcaee8ec20671d04e241336752adcab43a3581e9f48b6710e3c8d6225aef48bae08049fa0385b1a2b4d051238ed14b77c41cf535ebc53bffe70a7f5372a90a8024e678cd3de42dd4817b9401e69d3474b9b7bd0b0bf90ccd41d2d4fc2e71c096c663d678e004cf05b130521bf808c7501c79ca2082cb377d133c83b3d141cace9db2fbe900fbb363ea463bbd957b8558808f684b1d171fb3f7f4e51fcfa4e9b5062ef03c6326b53db627764a534c344d339c4e91d7c5f08d552c2744a09283b7f8b8c0f7b9b79ac029a57a1a0d517817fbd9f7120434cad9d3a768a079f77ae0a42d35f318b9b936d538d67081ac165da34ede4b5ceb30b4f346cff9885baaf16b324dbf108d73848777790a9cb12c8689cafbe9008623ac16f81266f0f5021c4ea7a8805b61399365b1c12fe524408fc7a4cf593aca99481769310003f2f428087128c0be55891e99af2558b3141acab1401e790913f1b071e0689e3967e7810af1f5a9ce88ed64f1c38ea3832a3cd60d9ca9eb11ad5438942f8e5381e8b24e204483f3c773a7d81870b3e077be926cd1cf30f0bcbc6cfd0eab1018d67cfb739fc05db86be187b256a982d54ddfbf267ef887b45b830a50765ae4219b15c03b67eefec8c790979d6f2359273eb3b966ed6617e18e148cce7b6591fd6799844aad7e8471edbc5c44ee8869883f32e20121cc241947618f54511fdf7714c5f4e18aeb183e14436a85ff3e7535cf04df354c57604e06ad894f5b9b637ec984335fb6254a7c968a22606fe1809d398a8f7f58b5f77660419071f5b67a861e24022487f8379d208ef25f464294118f77b30";
        byte[] byteArray = DatatypeConverter.parseHexBinary(hexString);
        list.add(vm.addLocalObject(new ByteArray(vm,byteArray)));
        // str
        list.add(vm.addLocalObject(null));
        // int1
        list.add(0);
        // int2
        list.add(1);
        // int3
        list.add(2);
        Number number = module.callFunction(emulator, 0x24dc8, list.toArray());
        ByteArray res = vm.getObject(number.intValue());
        String str = new String(res.getValue());
        System.out.println("============result:"+str);
    }; 
```

### 完整 unidbg 代码

```
package com.boss;
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.array.ArrayObject;
import com.github.unidbg.linux.android.dvm.array.ByteArray;
import com.github.unidbg.memory.Memory;
import javax.xml.bind.DatatypeConverter;
import java.io.File;
import java.io.UnsupportedEncodingException;
import java.util.ArrayList;
import java.util.List;
 
public class demo4 extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
 
    demo4(){
        // 创建模拟器实例,进程名建议依照实际进程名填写，可以规避针对进程名的校验
        emulator = AndroidEmulatorBuilder.for64Bit().setProcessName("com.hpbr.bosszhipin").build();
        // 获取模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        // 创建Android虚拟机,传入APK，Unidbg可以替我们做部分签名校验的工作
        vm = emulator.createDalvikVM(new File("unidbg-android/apks/boss/boss11.240.apk"));
        // 设置JNI
        vm.setJni(this);
        // 打印日志  要jni日志就把这个放开
//        vm.setVerbose(true);
        // 加载目标SO
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android/apks/boss/libyzwg.so"), true);
        //获取本SO模块的句柄,后续需要用它
        module = dm.getModule();
        // 调用JNI OnLoad
        dm.callJNI_OnLoad(emulator);
    };
    public void callByAddress(){
        // args list
        List list = new ArrayList<>(4);
        // jnienv
        list.add(vm.getJNIEnv());
        // jclazz
        list.add(0);
        // bArr
        String str = "hello";
        byte[] byteArray = str.getBytes();
        list.add(vm.addLocalObject(new ByteArray(vm,byteArray)));
        // str
        list.add(vm.addLocalObject(null));
        //list.add(vm.addLocalObject(new StringObject(vm, "")));
        Number number = module.callFunction(emulator, 0x209a4, list.toArray());
        String result = vm.getObject(number.intValue()).getValue().toString();
        System.out.println("============result:"+result);
    };
    public void decrypt() throws UnsupportedEncodingException {
        // args list
        List list = new ArrayList<>(7);
        // jnienv
        list.add(vm.getJNIEnv());
        // jclazz
        list.add(0);
        // bArr
        String hexString = "cf0a7f3424d12534b08b70deccf0e41c8effbd4d1d4886f271cc91ad8137010acf7cf029f23d572d6764802d1173857a0bbee5d4932c6d64e8f0a267648f1dc8e87e250cf5b4ef17ae59028f85c455e56f57f25390b5104b9f17e48e48a63679fd5e80a012b9d5ee065c29a98e09f75c9c726c1b27ba2aec4bb578850a087fd938101c2e59676e57edea8db56b4114272dc4162c7c4f319faf84dea69005a75a2850bbdc9b0417a1f1793a0cb1bf08e05e35aa8ccfc2caeea2354ccb04da54f656fa169353b46fa56f89bb458c54afe490ff1b2038f055391f2d716cf672cdc8f013c8da440dc9f3fe15a4ed2814d1e385b5fce9b77adb91ce7b256a0633fbdcf386078e22464b112ae7b5af7ef9e09579591c842b7f15fd78caf96185ae9def1d288481658ac5692863b9383814dcf31da1c7ae463cbd82997be665c90f462a1bf47d5da25a09782a797519074f06eb311c458ce4d993d2ddac2c934ab43d030720210817346e520ce10408f1fe0ff13d5facca2f246961ed27273aca6c345e431605e0aa916782c884f2f37be079a6462353e6a591fa04e59b8903ddf67794b403ab4ea3d0600059d892f2dec67b4835a7958e1c1a30fc74b90d087ff5d7103f40c77b4b774867468217c2887ae998a53edb8953dc697b5a71d8793003cf5cc806b4af658c2173c76eb79e2db96d4b2fee6889b6e4c266f601905a580c7741306f6c74571c37fcc30fe44f436e9d3603ffefa868375231ba9a5fa8bcdab411c6217ff139eef1224f802b44c613820b22d193271f4eac2951bc69d30a4fb1cb51bcbd11e4b1bf2617734c2da2fd81f500ea8f20d916ceefa6f0641becb9a483347ed1559037610e3f2cef2438a61375fbedff228f652de96fe474406b5f65a5de5e9b603cc569af1c62632c175d53f3f72316a0fed15d1be30cd1c3828239e5849b4dd0c4732e31e6f1a06d8b7f221f0a78a89bd96b61f32a8ca6f1fa96f43b5959eba9d1c114af85f698936f21a972865620694550b9189c9d296ade55706dd5469c18c357d5ba979aedd1fe72a9e9853db7e093fb7cc2e5e7cd8104e44937477bc0d24fdf24e02352ff09c654b36692543efd97d5b5dabb3a1b150306a81a111fd942d947531d71c971d259f77d5bed2de503744955a2f834b452f717d5e7c5581087b79d71fa33e7299a77c996b60351d5c94dd333fc7d0dd2c88b3672f534129a5c80d4c6e594095d8be51c164f489ff4f63e6b923d3d1896b9353013c8b3aa069df9193417043b26fe1dc3628f449a221e061eba36535a7dc9f02bcc38b64153b7d85523796f769a8b50e2e7ec8f69e45dd8098dbb1b71ea09016d746e43b83691795b55ca845e873d206642647b3003d3acc37deb18ca68721e97264e347ebc1fd146fef9cb0056a2002bfe8d21d50c5c9cc197187ca696e0a6cf45b93c37ebbdfde5b7b6bae633c7e0154e93eb81d5d5a507a30d2137895f8ea2936a1c8dc54c848577f1bb56be3a6c756d75bdfc5d2950dc5f8beb89542ac5a4ff3dd84f6c7970cd205a98977bd580c7daf32a3516fa9c112a71bb7163503359fa04814c1ddbc4b716535e7c226affc38c81e01a0314dbf5244b8122c9eb381527ca7bc3682e828c5c0b904c732670dd25347e871338069aebdda83fb9d3e36d7417e1a67c5b47d841f2798f73ca0cab58b5e7a3d3e88ed9d63e2d4aa585e016190723cb658423d10898b161a524132b098369dadb55ae2cf4d20d17efd4f2547685c99f564386570e91489a94b117889924473567ce958010bea8fc6ef3f7001ceca2c44089b0bffa9f1f3ecb7a884db76e9769fd7d512db73fe3ca65992f63238393e310561fd9accb286559ace9689cd7081d8b5417b1204b046ec17bdb3637072fcb9efe62cfbcc2ed4b77229d7220b3fff6fc69d3ff5d5b2ac06bdf57cebe72d385af6e1f22cca16aa4876d4d899005f172476507082d5b6e9b2e3e731f782d3bec5a8301813181a237046a9019288f86819603376698e233e32952fedaaced3c3edc2fde2203c6dbb1a5388b89ee3adfc9d35c74044f080e933e3e7757da7ac2337cdf61569bc5f4ca3ae2fe10914a55de077db24c10363f5e4ab9b3cb9b772d096524e1c9f14cea108b19b45a641ba928182ad8dcf811dbe1e22f3cab734946cca933cfb6d0387e856fbc8618f2a22627255e93c776361840736c5f1fae40f7169ad4e72463b5f1120bcbfec6f7145da862c2a9a5ca7c70ed47c5c01e1d956b8ee752966e7e40c310da1dc271e82b305acc4ba4f12d848a86e56b65fcdfb6e1481db3797df7e998d4925f3f837a348ddf08380b2c1e8cad6cff716ced9e621a696a1106f19bd07cc28bd9620401dbaad22e8f672caaae99c35b47cebe0d2e199f4c90f71865b29ee588cbadc519a4b184c0503065656f0d9db0479e4d1b1917e06cc393e4679612848a6342647d00004248dba8aafd7462bc1ba4a932299133d5105de31489515ab761fbdc308040eac2b4f3f9c51d704edaeb018fea6867ac953cf3e2eb62426ff3eb28b8abc46bc235dddeaafcf384d9475f95b973dec731a262ff382ba8ddd16102838c283eafabf0f180dbb27419fad86e501a8fc399c6eecfb8588e142b0643ef153b38f6da92bac800a2fe10031f82840d86a493a402460eec9d7aed608851c928f5020d6cce94a29a7b4dc888fb77688bfff7811ee8ce583788ac1642406a64f6b1ea32efd3e7e454796832300efb8a48ca4867caa99e7e293c10b530391f8ea1e8e874d291f9d19eb5c23bd58ad2730b8ac3ccb64ad817b089e9148f87dc69cc77e0f0e2bbdae26ca810cfb0d779f4c1f2cad8c37c5b5cd70db87b6ea4f8db5925b1446ced1a243676044ce8afcf079d21df1ff0d996a2ff508d9c522f6e87452937b7418e00adcc95e86ef60ed7f3d9ac46c2eed7f3585edb216b9d85c6395a6dd311757332727ce4ebde177922d8db192115cd10490ffdcc04660111233b04e9948a6d7699c3ed18e124d370482558c5a9fcdd96fcc6ae01ccdf5a38c76fd9340a66b446e3f577f2fa01471011fa18c7e00653a8714359ff1a5e68234a076405fbf502cd2e7d00592662f29fa0c922be2bc68bdb743eaa42525429dd9aca1d38dab800b11602b285c2f1f78099ca89044cccb04ebd17ed053a753b50f15f922c82a52ac5fda5803fd0f589ca8d1cc88536954a2e14e9e17b4934ec0c1bccf61117ddfe5a7eb74a269cc21ea2595180f0b8e9bc915412e1b4176df9c338c64d9198c4019ae3ef3777e637a11e9853aa955bbeb42cc5c6829025f611c01017d17d868cb291e06ed12de27b9f6048e0d48630b262e116ca9577de585778dfdb7996f78a73d6eb00bb9f477bc362c8ab7d8cada716ae3e54c834fcb61b861d0a47a03d120dd5d4da999adb8138c08ac4a282149ca3956d0f8d5a86fcaed5a0e0fc52052969e38fb18a730c2c26ccefb3589b3a75f6a0806a45e87574dcb48f6cae89ae11ef65ab2a2f59a864470e3f673601f01fb564de6b264f249d11a57dff05f59190bb2d25cb0f527f8d99c02d250e0b85c8de7bdea263a034f77827a55be4b1f71113413d1411af2bc0c005fe95692b20f2ea5a380a66d030edba6712683094e2e88beb55649af669c402a6e837a54286d63d4780ebb043cd3ddd87cd9a55c61f067a863231f842fbb16e687ca2f52272a1c6ef74366ceef8b68d16e734ede36a144b9aa65e318b651916ee2965ba525907d819181485122a94213363139a919e12e1dfeee85943fc2c775b3cf949bfac85d981c5f77a28f3aee666bc82b08774dedb010349d014e71ff57360fe5d4063c2f21ebe3af72cb73f195620e319875924c4655f65b1f59031887f32f890e498c27692ae873ed605d61b3b3f6aee3222aff4f302d26ee4b2ae55fd7236b24c45cf275219b26e7cb0bdf17666850750246a4ed47dae92756b1f61b123b121abd226c5e607cfd6617d33fbfa948f54bb7566ac79aeaa536b7388920fb9961c732db9b210ad6310137e4d72cc1d02498030d97cea458f2823e2b825ea59f70bf36d35033bfa4ba8c5d7869a708a7cefe00cc334b4ad5c6acf72177bfa38225c77a741baf82b3483ad9bd5954989f24a3c6a26eaa83223fc864b719b043ac45d1d5349d57039ecd6305562e6b3d274043050c68724d25114366d80b4d724ce02ae033c3284939588d00c449890c2f183bc188c07f204d23a1484dbbe3f068861841afba642f76375213d6f588a0acbf308715e32d27c3ea3a03477709305d6a477b7fde82b82485929a172c40dd6262ae5ee5e0c6f45e894cc125bb8b395af97b14420898045c2dd39c5406725f70d88bef58015f51ca5b3d90bac15dcdc53163790e7e20ac666f76eb0dda39443a1f475b05fbbface3b400c5d97dc35b6daa1543afd71829172aa3c0a03653aaa63326e015923dd375e9aee8f10c6da3a00c6bf3b14c8c9c47f40d3a90f557d5a2c10d2ae301c32f0a6b2bad55849a6a525eb8451f0f7825e8566f29edda9f87ed7562d230fd0f2dc6e7fe4dc612e9c07c9e62522652accc82960377c8459675ba9452bf2b784807fadd5da027b20d919e4ac76b310b346f5b71b69bca32e940e5c6356fe56bfd1d48f0c84c756d994166c1c670a68b06a90be7dff1bf8f7fff585873e634b79c1e547b3295b3bcf31345a7399b3c6e395a6bd7aaaae83afd5a075e54515ea56571f381bcbf2a542dcf300cefb646f529df7da148c1a07921b525fd1741e071c19a0abf88a980a7f6173096d6196c364ab033f1ccc1db2fb9eeffc8f3f5d586d5d3e86bc9fa43c7216e7252dc18d1aaff8978162a44a72adfbe987073cf5b6090ff4805877383a0a13ef1ff16d137250cebb24f06af65266c90f51fc6271d6c88538876581db5b6c8a8dca1547ff07c8e866abbfd616698c785fc8f8dda449786e4f470ccbcf5f0853d5853585f5d867910713f56f9cd041cc78664073b6a5171898a4a52dd1c5e7da367ddcf7fb1fecc971d826c90d1ba786175b5b4fc235529385442faeb5d826047a320ce3d8154d5fbb949a8af2336621c8ffaa601676f9ce27bb966d67f57dad7d764350b6db97a8e8f151b8e2cddfed00f6f0238f893d149f3bb194c81dc2f094552644a52509be795b5d35a160b5c7d796cd035c6a14a2645137eff6130c6c5c8c004345261b2fcd53aa6bd7015fa7ffddef1e0a586faa25fdd8660b78489bcbe62ae181cbccc44a6cf1e425df26850cf28a1e7e01b10c61f92afa8594f9549dd15a56bf481f9f11a1ffb2333549d3c4cd7097695f8c65f038b6c30da1c9ead76770f8941012c3f7bdf308b7b291f5a86f31b780ec5b4b285acb89ef404cd95d97c19f3d73eb05a879a74293f204d59b49c5fc3053370cd4e3e42cc5611ae584b67630894788b88eaa8f5b1151a1e4106866895475e54f2112797f20ee308b2bb2d34f2a942be2430127cc82475e736c0dfc54ecd5eb0d6ae9078a5d2807f70864c626028ffbdb45fb0579930110203646fe05bbd01a929f983b426abdbe59a3e399b10c16779d4a4a2ebdc8adb4a09a5a437e143fa27bf64c3d580c02448db40011b777915613875e132bdae513703b141b40dad33fa0e3a7319255b21f4cce29e8942215d6e6ebe5c0aadd126c8d412962792864279872191da039a355e2319390e2172e50285468d112fae585162c56b31c56828037f544a425146f274a417b6d245799ece37208d17ba8357cc6352c35da420e3ac0c97b83a6bf3e9fb1e463d3a2633753ff5c4cdf6b43fa3d176f7ddb6284209ceb682ed51ad6aa4a0e78fe98a40a7765505595121c2b8b9b5b20c7d6003a180098ca87ef92a083a83f405d927cde17bd5e2e145ee7961d75a6d5867c9b35f38e9129d4944d073755e72d472ae4f49c80b20d3d7056081f3317ad39917fa738252d0a701cc0a5951a217c30f2ec2c50f869ccf668ad016615cf0494c6eadb7d90392888ae9b62446bdb56b28b736f58ecc52955601588e4fa42a894a97af0d2f3f1f9ee832b7820ef02930e6b8505eb84ea160f22594f0438bb5cedf0a1f3944d8c63a01a9f8866024e9a0b9ce2b01ae0edfc3ec8d650d279cc484d6123f9489f69307316517064d0ef67c31de407bbe7df6a0e5e66f7cd7cb0d4a6edd3f50d93963f312c18b916c642c30d318e262335beca0f317c8075f53c4e6bbdce90dc99aae6304318f1eb667689c020d8ac0fd56b9d6cb56422e0b13835643b90a1bd592c9543400e3ad8662268ceda72a6e30ca54dcbfa5025c0f6b5c07e8ae2946c199b95ca1e69f03dff38f81e8ade2d2333edfa7de7b85032cca59de42e793d561306d5e2e36750150e626c1216b939557ee6aa8560c2dafcc3632c76e559e9da9b481a90334e81c1c501e06aa4e91c03fd82dd97e8fd8ff494a6cafca6ce2a58640640e930951f9b5a2bb3fd60a7e9eac52a2c4d368d43445d29aef3bdeb693366006ebe00d19336161f88a8358162a8479d8fe150cd4ca1b59fadcd071cce93e82bad71bde6144ddd930b755f474011a7bcfbc1df58fb8b52684a1b41b66111d348c33e19058c09936bc0baaaeabf4cf219e609ebb70492dd451f1ddd4377b8b770a59036bc42da3273654b806abceb666157575f2e145ee1f8026de29eb984658fca27c14b769bc84fbd4a5a8f99b6e13a63d36870550e2a0caac25b7290c7d034312f9ff6a7ea98af1d974b6f75389532d610c2d5c2f1454aee223780c653a00ecba8d81c1d6d1c1cc34b75f340a7a987374dec7a46f23f2e2cf70bc817d26d044169f07d876cbf425726f8e0303fdf2dd447a558ca902889c50186b1c3e3f032e6127227a836b7af027e79bf45632bf0d93f98918850df89dee2c6f89320510388f2fe154528cb23bbcc117e71db90b6af87a71c527afa21f096d262e403a9063a2fa808be1c66f0f290a521f05697b691eeca26ef839f29f56d06bd4494bd7c1f98659d0668eb4c4a85428fa3ea2d159dba24032eb3b8efded07c22c53ef7606313ad31bbdb402903313fbf3c6dcce534b460fa24c9465a5e19045edd18225d6831ab0891b072b883246860da9e193c3ea108e65a04a39677a3823547bcdd257673fd9d86b2460ddbbee4930644428c65035dbc88a68a0a4a199c393e684702e7faeca39b3276bb7c950589a2e13c39d7bb9e2f350eccdddf9cf95bdb7ec3cdfb37a93cbad40ca4a25d01dc33636fd6a435030d08b575de2dc3edbbeeb493989bd07fed80a799c949add02c0925885becc9b76386f50167a25e6ba0f6fb34557728eb48617fdf8724b4ef0aad4a760159e9e119ca2908aebba3560c107151995ae896a2c46ac59635da7381eb51eb8975fe52eaf1909b92692cedcde2778bba14d741f9ec5546e784a6bf9f39fa8e7b65f9a8291027e38903bb5edfc914dc804f55397d21cc1d7158f31d66071429b80fff88150affb055bddc07d028ac8cf8877cdb8892e786d6667e9523067db55f50052f1ce0305ede1adf5bbebde8e8272fd57d1d9578e527735726310883764a9cb9b059a0caf48734584271aa6203cab8ab138fb0f56abe7a8258f534b3761ace5c632bc0f29c93796f91644968a510796d0d326f7ac91059f35fd0b213305f143555fa5d9e26ce4a80e1bb49c5235a4ecce1796c65dbd6d51207d70ba6a396210039e10dd4590f11e6a24c98e39f930ef83fae67649ce2d71f38d09a91007900a98f10047ca7c1a9c67979ebff85691b893745d0c91fa3b30658a19804a7f0bbdab942c844df1ccb017b72ce491e312b7d98ed16a98303fbc1d8caafed403e08894433bd787828c65056b708e3ad7b03a8209858d50199a27ea7f85e69429d7f3f126d7cfcf3d01bca7146a831a3e1bc717634c02711edeb2c6d21e3a04206ff3c6ceee113df821cc82f6fa4d27b60b330e1ae0f2cd499c45fa7ef4bf27de1b164d1f14a1d0b55740ba0fd5472354f8a23c9c9d7d6dd95ed510b559b6c15e61947957aa5e6783a6bfa314f3ab2cadccdc13302e79365356dafefa1fc1e6a1a6902eee81e179bebc94cc7ec2520a1901367513181dfa47d73c10266a5680e91c3e6855266c84acdc2a26e81a4aa5f5edac7861c9be5159fb5c05a6be04a3ef998cb49f20c90510bbc01129a3503b6843ee59d1010eea9a5af01cfb88374576eb9884acefeabe3033608328361d866d8ea5edd751f5eab22dd10040754587de8a2422c244a1f323002ba91ce4c9930cdf67c00d260e87aa3d8b981f3fab206d243afbf4c9bce893eb5e2b3f7eba58169815f6adc35c9d06e82bdaeb3107447899caa463517ee2b7db2bc189bbf7d68fc767821d38d286d851a3a81b62f87a1b619217f55e30439f7ebce74c9959c096fc7ef7ba33943cbff0df688a30981f4196caa7cec6510e92e9b2fd266c7f55d019b97958695d2be810a3aa620e0e5e95fe3d14d4b2cddb7d6b84174e1dcd97b3be4961ada8fdaf90b7d23f4e74b95fe2ac1db20b8e98666f45df9aa6525cd4dc436415dc0e3582740f04d0e2272eb511cde714641c4d54d80865907a90cf36e91e23d9eddd056ce7bb61ff35e7104012f061a6b5956ff7a9572d18c6db563452aeab97302979c0f96aa4de42cb9e9484104d2f6176d6de39d5b98dd2067224cf32c197568c2c810b1781ec62906bbc344eeffc67278a87367827fb6265aac37883122879ca01a134416e9e24cba9f6585100da3e359d7e5633fdf4697f91f4b07be204c920c38652a3dc669c389a396fbb628e7fd89a4afd85dd4152d0d1b8e53beeb68a67e8a3b002992f68484d3bc583c5a506254a6765b2bff004f059371075031166f7bcffb2ce8f41acf97068aa6c771acf653729422df7d75fa90ca7fc2a73ede462320aec3b364a9b04f4a7d3b12032c990361dc2a90829a383fb57e597b441615415d103383bff870102be51413cedf7a920dfb25f30b9ef8705c53c1e26eb7a631cb9f1eb0862f5b75f30a26305534d26b48b05c8577ecd3a66beebbb8f6f19a45835ece7ccb88f3279e20d0ce5f4c042a6e127d28eaa3a43d9cf2db53cdf3d5181ab6cce7064a4f11fc25d79631defed781ee0274b2d27ee031fef04915d1f384a4ba77e1d107b23fc3eae66b8d1daa4cf15dd4439856e9d2569e5b65f3efa05d82877427ac877858eafcadeb4e5e342e095d4a08b8db0c777547b699d9c9dd1c7c5039aaa0fa74a47b9672043ae90c8a36fef4443150e8338aa33e2eed22c9cad473eed336803811eff5de096990598fbd3f46c3acf400a7b2295904edf35c752ef75fd10b883b63cb5883862959de6802cfb6b477a99e2a0c65a65b3d430ce74a654bbbfb5f597e57922be40f909356ac2f54c9023f5422625cabd189e1ca4789d80892071abdc10b812824d71023baa2f7d32515b136fb587f5a3bf0c13c62d19f544f0532d20884c2158e6c8aaed87b7a9074e35ecb4063c5930c3709cc4c53ac109c7c7c82744639135523377b05d356dbc84bd278beed64a7fe0b07778fec57466192ec9f479cebe6a63ef977b1ee2db0977669139659139707e5d9b2142f585607baae84afad3460b3dc31112c80ce3e7f6bc9b0d71089a1605d1cb0e7ec8334ae26a0b0be669e69cfbee8f9dac7a1ea703b6a64291e9f2d688ff4810589ac37ca215193c79fb5b67364e8336a36b85a891043e7d04e417e8bad57873fce04c744c80282bf1a92136fdaed734bbf618b55ca323fc0c2335cbf09691793a2047beb5a5a62421c3b8d4fc1dfca01019ecf60d3b1d42697b49eeddcab843bbd422a5ddf84cbcb0d6d781106ab79522bbdf279f531f750df77fabfffa8c070f02fbacf52c04e53745d38d689bda78d2a48f4e27a1015153729d89e3dc35f8cfc41f81c8744e7e024bf40bf55e50577ce6cb68d38e9e02aa74ef3908232e7b803c44bed69fe44827c84edc9eb24013a2be671bbb72a7acb346cdbd5df9d44d0369f93b4c04451dfbba084569b195a4d6dd813519a9c9bf4076dc61d540e674aa1e960772574c5631dc7f3719c49a12fc13054f5b71364fd021c3f5e6e02a91fdbbde355075584f066078fbf5606163fe8b562f5cbcbfcf21c2f5b4d63f41488544fd09c1f89f028f3577688806ed1e4937590d2bda7795351849c64846cf500ca369387dbf805b203821d37fe693cf9c980a1bdc040d6fd007809e8fee33005afb736df56f1015a7a2e350be75556cf571b30cf91bd36e6f0ce3cf05a4d4f1f0bef7fd78150b906599c3e602274b8d3d82b09874293c75f1372523d5b35f9c5117f76da52bd6c58760c50532c054ebc95e779e6fbcf876497aa9aae80b8ba64763f31e4be34c2dad4bfc52ddbbd9952cd0e31be13e32bd074a0d3d77835119efa61d76c330ad3317c1b3e0b3b16b7a69e620234fffae3c69d5879d3a8108645065f5fe3ad5fd807466a2ac85381062d1a6d289f37597a6e77a3be7e55172bbc0ebb34d46ff25276215138c5a96524a882723ea9090214838562ed76adb77b51617cf93b8227c4dded0c4d7f84f2a25a449094557c67e2ba0b44db7ac7d08ff356b4de50c792e83b34f9122793553b12eca9bf7a7f13460bd5cf792938c94dfe1cd0387a8897e3692f24aaa60d0c99862a191e1c234144dd8ab05f03beed460caf8593449bb537595d02e024daa39c239e438df30774d37cdc0bf56a986aaa47aa58de832de577c8fbf21069db8e74ca428d9f514c21282f1160f79598981ab52eb679efaabba2d570d5d82fa9aedb78c091102dba20681d2391f7f55feaad098614ece00eb446f0e7301d0159c0dc63fec5867aee1e0993d8a42a3ac4eb367a3c82f4c0d486be9047c11a222ebbda2fab3c66f25a8ac2928620fc174ffdd2dba1afd1507fd7474551f38c6678b8c8d736e9352e10c4ee1b577c0d8a3e10b3e33db68d7153963ebba3eed9be7d0903081a125b3536d457ca94388fb64a82f8c93481df42a7522d7605d6d9598ae8de7eab89945e01f15c7dc3cf1c30125394e90589d11729234764f35b3a9824da53e86a16b1a341a204d798b5e8af88425602fd56acce3f6b005b49d100e64f404fac840856845dc487894feaa3690dd7fccd8606a1249d6551f7f2d894fcb291bfa6c0502f7c321bbc72485b8b4931c42b0c4840ce0d619c2ac0ebc7830593e21fe08de254c76a5c76df1b1109e6697a24d1bd21610ffb287c241004907a11ab562ab7dbae4c4e83ffa6ee0b89802237590edc966f0ac7c453c447a0dfb55f6f605a5ee9df051ac0466c8d820a01addc3a025544696408ee2ebf5aa18771796ef9fc125584023bc8359fba1e5d28e5cc295ca80cdb14395422f8297b757dd68883560b2a9f3a28456d1228aeacd03648dc9e9d42905b19410cbcc0f2064d2ae528ce5c9a97032f6e41dfd7397ffabca3db292441784f5c89e4c5343e8a4b4ac76d63bd65b489e689d5c206bd772b569326b6d3f2a41e5f817128d9e1e196fe0d4d5af1254423470ad92f3a3b7a5c855415f97d4daf7ed395a9f176f7ea2ff2d4e26a36be59fbdc64160b286e16d9425aa5cbcae9f8a2d5bea788f68f5cfaaf891f1e226a5dcad48ac66650efcf358490815a31e332e01b9335508a5540a5a5ddbdb18bfc49a46daa2ec5e58e3ab3ac29a672270d321c78036c86030293cdffbd5f7a85435977a0898e32a8a3f296a095d5e4b6f2388d0f0a14bfab15fc425afd0c084364c58dd3053a9ec27800dcf83f6e503b950eab93a655754d657f9b01f547a03b0ef6aa9f61694958eb4a22e90049c03fbac6671162029b73b0c8b7ac658bc4c10c9fef7542db6cd1d8ddf02b8736d48aaee8796e294f8c4020d4106192f6fb4b4bfa34cdb62f559e0a5a098ade8f493e991d1afbd9829e3df7f4a3aa9e8924f8a9cafebe9221aa78be7c26337d11f46b155fde26d0cf60f89243b5cf1a75c8bb468d9e5732d3e10cfeb4be07706ddadcaf80f481b926ec1f5715c54df1ac0f1ff9db9e9ab910b7ecd4344af59925ee25093483ad8af5d0c492e2d4f161ad325ea03fadbe41076fbf35728719399def20974bf8cb1121b29c8db7ac956da1d97b01228d6719612fbadd5cc361ed88a0148d29052d29a255a1649a6c8eaf5efb49399553dbb045828e159543c4238b34a4c6c580b31f6a3cd28f29fee23556629150257bd9f197c75eda0fd652c362042cf640e3eab60a8f528a76dd2c12c4a29b7bc92833468c15527cabee2d86154808467963720738412e641e619f0467c7e3dffd26401520277fbf837cf6eff4060d1120892ed65dbbef4a80cb9eddd785035b74650015bea31a4e67a03ba460e522569b12c33669679f0ecd9ea7c009ef1e25f9b664ec8dcb6f6182ea668019e661577e2e929902d0dd3d89228d331358a9576e64a84343462c5016d9bde9ba17e4d50c3b0de6f92c646291303fb81fa85f83b49049801e1bfb8e0cddd519a44818a6b8b1b931b684a03672d45248206bd8caf35146b5b7ce43c82776bc84b9477d63ffb3f5d89932079541eb4f815f7972d3f7f83fcde35c0d00c31855b39c7975880ba237ab99973d9b5ae2b572de187846f6e3cde137cd4499cb53cdc9e23e727feb99e9e10c7c690e060a47e8d45505b1530462a979ce4074e21528c487f2fcd678669e6af0061bb35d98de904f66c2b4c5d4f8c570dee93a7d1433fdbbd5a6917df5994913fd72a6d4bf6fef644f8765f03e4aeb7197ac85a115c95e63ce654896c2b51d51182c9605c7ce459d4e51b47bdccdbcb53a1b654ae55728ced5cab24655134d593b2cbe972b29b68ab650bcc90b04d7609fca238118c5fe50ea024e873951ee8c218c18658677f2b0580e6a43541c371e608b05a0c75d16c49db99905e472e5cf5edd8f6f5c9858361e69c1a354228e6ef4f6135740087ea2b08345d30ca493960de2aa9e93381ef7aa643c193557c24b8e71cd41278abd09c058512aa4433d05dcb99bab7185f91bd0a34d4127538f3d0bf463929a0cfc1ebb5d2180e90071d8aded22990653414c29a5bdd119fbe955c3f8da5f4828fc735cfbbb9d8fcb2c171d1e61a723d394e80d70d855254f4fb65fcc127790a5c4d30f493ea073f46c7cdaf0d8ec3ffd769cb2054d824b1c13133ca722c0c819cad1dfa7ae258cc05541dd618fe5cd6182d035d23cfc7e91dc2241ddcb22d44c52226f5e3353566ba9763bf46e4ec44fe9becd0c8f4afa6e2ed738e1aab8f03c5831e4642fd007e78b812dbd055e4d9ef645e2dc931ba005c7b37ccfebec145c2f77f4aee62ca19f1a86e5d16a85f17822ea6205c93f2f1f6a2c406090cd93b96246d6431dbd27208116d5442a6cfeb49032eb875d43ca5d9bc780d55585563d052bb9952b9395a4e1ba2fe82f4ccec2d762f39f8c96335a9597357d5bf1c4fc09879862878722fd699de58f65578318663044dee9385ef52affa8fd1d59b83a9a1c962cd137268e75ab5217a2a3c122f8d8e7bf7b79bca1adcf9235f8140f0928136307afad086768c39793022e547d555484293a4db6415aaad1ccf8860a8c0ccca2a23ce6405fd11c8409746c1db84c015b009394ec4a22d8c9b8fa9c06a80b35cce75fb73507a553d4653457d6c2f9e5a1b5efdccb1a95690532a9fd430bf3d4d2dfefb149678acbf808f6f88198568b4a9a83dc0e39167ebe9beb00bbe3feffc9d49bff54365ea09ab825fbbe050c004ac8eb0e853e5f86b069c7456d65e1efd24348dd3090266857d33f8bd8325797c1f28e81162876ac38170b710b37f7af7c9d5d7a75b2d1eca99a4bbfb93ae6faea6eaccc69accdb7e2ca63714add0b89a7e178737baa71903c763967b206fb7c3ba4717e2e8514b6679380315b540a455c7c5d52d98c02acd9f813927fff27643e5eb7e29a23b348899443c0a3f50c5d0473bdbb51ffad4c0078c5727e0e45f0d7319e1135196957a69738e2471108305d189f24801181808c80c6039fa85a1d91a7d79cf4a7b7ba223a3e90b0d6cda9647b311d347c5c4d5bfa27a8550ce92ab2a4e92f97f0a80b36aefb42ac205fa621869da3b36b7734f1e639ece98973526bbe54c0cd03311c5b538b0fd940f29fcd5c3ecc42b6023f67fa65fba9ed7d1e04557fc9cf2386952acbacb4b646cba555c162ec082ba2aed0fcfa9cb5602e878c34fb130133c9157fa25f07d6f289a28bb539fc75085213ef1d2934c0bbac5f1cb8e2d94b7dbcfbf6459653d8a478800bf60edbde0fcf9df803643cfd8dab74892fe093c529f32681289f6dc98baad518ac488b3b31ca234bec2fd7f98922f3aa5982a6ae5c702659aeba6cd2de2fb9efc23946cae03a78d63c49eec8096ff8260c7a42b0420af51434aeab2afca08e2c416a5ccecab5905a13a2f0ab25fae377d4876a892f639b8b5ec2ba07f26774ebf13fb6e7864795e901396784e208d81886356ef1d0fd563b7d99f81150841e59cc0b780a8b6fdf1441f15637772cf3a5113e53f8bba93ef8c159276d985e318aab057fd6933d4da8b09e9e258b446ab5ec23a94aa556a75ecd9c9be4a1a6c55097070aaf2d31486b511cf6ee574a74a972990f5de642bd21934472b1d437bba15bf498b09e00f9fbd2c5b5aeedd7c902712269a0db1960abbae41678f45887f50b8ab099267fdc171b9ea6a8ca249af819fad31b01b9a2e88223ca3e9b189cef194647fbfe2141e49fb1789f39e6f43ee27d0d4decb1f1567a99b6d242e414fce5dc7ecdb894098adf91aa78e134d551a27458d9b5cb651e1a42d798420097849cec400cccc25f60a5d432af29b9f4d042e1b0d535fb56c1582fda094a218ad04635b4bdf1550229b4968baed79a2039ef9e87fc1212e7db34625234c7afed0a349ec1272e8f9d72925e519ae99253c7d07ef48a42635192bbc18b156e9927d9ebba4793151742d0289f19873b6079f6ace4b40644bf4c45a2699b4624f9000125bd0d2a7dd4bd8038bb2d0668a9b30bd81afdf17630a48bc9a3d8b1dcaf6ff376d511c9256b55e8f958c0b8c199a4b35ff284e2cf2a0a3f0cee862fc37df2fa5c40302196939c526b4198ccc7f7691a1c15fd07ccd5dd68ba0c735e3cc6e8a9f1af1d03dc48222c789d563a37fbd8fd39834d1c257cb76ffad3a901a1443c941a47fb74993d0031c6671284f39e910559628dcb22074bddc3db51b82175851a9efa12758d697790649376a7d216ab99d5e882b8831b3dded8d427c17b84dbc7d3ccb4427899f6be9127a93f4d79dfbdcdd180c97822391edc78f0070c50d754b0716fdd21c19045c71864da86eedf6170c4e43061b13214c20ff98656e5415a5c451a1e6a71af34d6d67e8a4d9148bf78e5957fc9b0a0b6c7c13c351b1f29ab849b450ff8c867d000e53c49bd419cfc5f6567749d149a4c77d8c2bcf46f0ab0cb13e186d302e836474aa2abdbf0963a2395b9c7c11acef54f510b61e7fc0b07d78eaf3bc1bc5da9dd4a66690ad02dbc863ca3d498db432b704fbf1e7f6af3b8d17b1ddd84822050f5075370928248788aa41a6028cc2d97c24d34d7694d74d376bad5b6c3a39a78d0cdcc43b05a14af0332d87c5d5792839750f9f56efa21160d1fba8fa8e3f0af2df4157b30cee109522671d9d81e4b594d970c8792c2df3b36a195c118bb664319d3cb7822a970ee68a95921d71458c6bf74fda1682274a6d26f3a3bbabe421f8ce5a2450b7acc2706d0555fc09ea0d5ef505cd81991aac0d33e2d16f4b22d43082e95db86cbdd6b401f10763c3acb480dcb6e45f358136e55843919d8fe988f4ee3a5d7554c7f33047309fbba696ea223142dc3949712e43d9b3fe2dd567ac4047b861734a646eabdb3bd7206d5dcf4d5cf871de3db16250c1f1585310e9f0028268534d04b48b92222cab015d20e67f4c9c1cc503fe7cc6e0c5ce4d0437bda885b0087345165f2630263e6dbb78c7dd6b17794beda3b2884fb2d89fd94f4ce0f4b3ec5fe56781633e132fc135c7fd56e4bc1c1988e02123dd0f8c9dae5412d053c8ca0de3ce328cb5aee752bb1ac975546d9b30ad9073c956c61aca99561b5818834aeb2f2f0acd2f5d88243197a036270efa089f2418998e2fab57b0399a34ae8ec54b376e2f318618f441c80844a1b95a1ab5f253cecf2550bff6b58a4666f8c305791fd37e28819985977c027158d47e5c5f687c43a4e481d54e971752ea538d95ee08ba12ef5d6cbe661fece67b539c565a37ab861c83b1a139fc5fd5fe4598a53eb10a9398b572399fc8c8345e95a7196dd91e16229581f520f1c8d0c9e7a6c7de506963a7745f75ca855ae2731934337a58c6a0d813bab7c59e779d5e8fb921ee30cca0788eeee8d64d969c9f5c8ed06f7a21809c7a4cdfce6395fd66e93d8cef4f7cb9080d54b6ecb069332579bdf0b5b6d88cfe842fa5045dd7bc734b0b44e23f76012bd8874972ec3588221eebceffd07d2ef3acfb920bb0bab8be96f0969f3f6d791599b3d8bc6695398f94a0819711174c55bcba3e04a5ad56ec72850ce9736e00da3671a83479fc086c3ebeb01949100727a4b0ea8a16bf68c22344407da944b4875ae0c205ff34198802faab626326e2fe38eb89c483242891c270e89d868201f6cf0e97723e89044be5e98f5e1ac7df8a910c2a2a7516541590b6f9b172d76af5d200afbdb36276db2bbd288268dc07554da93ee72173b158a63b4fadfd443d66bfa4e2c5f57dab4a1461af51aa9590e34588a82d7db0c84f860ec0204095de59bbe7fd3f2407dcb1c6e14c0e8eb08714eae9644e31c51dc0bed3472ccafb89449c51460dde81b774cfea78bd20a0d398da4589f8055ba798210daaccf2b65fcb82a69c64362dbb90f27e71d5811171f4131a176384a00566654bfbb71d9d569aeb84650d887943edacc28fbd135d13af1344ce1ab2f7682847f10994f0edee19d0a0358b1b5f1c606f0841a177489b386e77be18aee8cb9d598a5b8177a09ea34bc8c06f3914f2604fd006a36e42ce495e2a71dc5993b574c90ab3885d055e671d95fa9064c34b04da1bdf8f478a5b7ec1be3f67818b6d49cdb9a09df87a2b772a16198a3b4956ca75ed4a2e9766b6164e05e0f58ee44f7aff4cc89a0e2a7dc326d91169c3bfe4eade42bf3f27d46dde5d3886291b60778e4141ff2533a491aab897ad1290faf24af4e4d60813d50c68151b158c1daf2570ce9b8d4a0836eb2879fb31feb2b5b451ef83131086bc29bf2637fecdb17d6b81abf5451636abed54e3cc22654caab841bf6fffd39a1ace16aecceba1437fd98d5c4438590da5108b2829dcb4269b94fde32782a68fb1e6769ddd7b3ad4bf4a6fa6bf7a07ba5bb2d649edc80cbd655433fd4b5633ba7eab3bb4e001a035c2c7809da413c767e60d6ec82ebf3faf365fa801c25ff09bb020d887972ead46b1f84bee6685e907309215fb2ce758d6d145e6b0cb50e0c51be53d4f24a4b07fdca53418a309a7ecbcaee8ec20671d04e241336752adcab43a3581e9f48b6710e3c8d6225aef48bae08049fa0385b1a2b4d051238ed14b77c41cf535ebc53bffe70a7f5372a90a8024e678cd3de42dd4817b9401e69d3474b9b7bd0b0bf90ccd41d2d4fc2e71c096c663d678e004cf05b130521bf808c7501c79ca2082cb377d133c83b3d141cace9db2fbe900fbb363ea463bbd957b8558808f684b1d171fb3f7f4e51fcfa4e9b5062ef03c6326b53db627764a534c344d339c4e91d7c5f08d552c2744a09283b7f8b8c0f7b9b79ac029a57a1a0d517817fbd9f7120434cad9d3a768a079f77ae0a42d35f318b9b936d538d67081ac165da34ede4b5ceb30b4f346cff9885baaf16b324dbf108d73848777790a9cb12c8689cafbe9008623ac16f81266f0f5021c4ea7a8805b61399365b1c12fe524408fc7a4cf593aca99481769310003f2f428087128c0be55891e99af2558b3141acab1401e790913f1b071e0689e3967e7810af1f5a9ce88ed64f1c38ea3832a3cd60d9ca9eb11ad5438942f8e5381e8b24e204483f3c773a7d81870b3e077be926cd1cf30f0bcbc6cfd0eab1018d67cfb739fc05db86be187b256a982d54ddfbf267ef887b45b830a50765ae4219b15c03b67eefec8c790979d6f2359273eb3b966ed6617e18e148cce7b6591fd6799844aad7e8471edbc5c44ee8869883f32e20121cc241947618f54511fdf7714c5f4e18aeb183e14436a85ff3e7535cf04df354c57604e06ad894f5b9b637ec984335fb6254a7c968a22606fe1809d398a8f7f58b5f77660419071f5b67a861e24022487f8379d208ef25f464294118f77b30";
        byte[] byteArray = DatatypeConverter.parseHexBinary(hexString);
        list.add(vm.addLocalObject(new ByteArray(vm,byteArray)));
        // str
        list.add(vm.addLocalObject(null));
        // int1
        list.add(0);
        // int2
        list.add(1);
        // int3
        list.add(2);
        Number number = module.callFunction(emulator, 0x24dc8, list.toArray());
        ByteArray res = vm.getObject(number.intValue());
        String str = new String(res.getValue());
        System.out.println("============result:"+str);
    };
    public static void main(String[] args) throws UnsupportedEncodingException {
        demo4 boss = new demo4();
        boss.callByAddress();
        boss.decrypt();
    }
    @Override
    public DvmObject getStaticObjectField(BaseVM vm, DvmClass dvmClass, String signature) {
            switch (signature){
                case "com/twl/signer/YZWG->gContext:Landroid/content/Context;":{
                    return vm.resolveClass("android/app/Application", vm.resolveClass("android/content/ContextWrapper", vm.resolveClass("android/content/Context"))).newObject(signature);
                }
            }
        return super.getStaticObjectField(vm, dvmClass, signature);
    }
    @Override
    public DvmObject dvmObject, String signature, VarArg varArg) {
        switch (signature){
            case "android/app/Application->getPackageManager()Landroid/content/pm/PackageManager;":{
                return vm.resolveClass("android/content/pm/PackageManager").newObject(null);
            }
            case "android/content/pm/PackageManager->getPackagesForUid(I)[Ljava/lang/String;":{
                return new ArrayObject(new StringObject(vm, vm.getPackageName()));
            }
        }
        return super.callObjectMethod(vm, dvmObject, signature, varArg);
    }
    @Override
    public int callIntMethod(BaseVM vm, DvmObject dvmObject, String signature, VarArg varArg) {
        switch (signature){
            case "java/lang/String->hashCode()I":{
                String s = dvmObject.getValue().toString();
                int hash = s.hashCode();
                return hash;
            }
        }
        return super.callIntMethod(vm, dvmObject, signature, varArg);
    }
} 
```

### 打 jar 包

打 jar 包给 python 调用, 这里简单说一下吧, 网上也有教程

![](https://bbs.kanxue.com/upload/attach/202403/983110_6HASWUEUMEY6V9G.png)

![](https://bbs.kanxue.com/upload/attach/202403/983110_9M77FGBCX2SMQ6D.png)

![](https://bbs.kanxue.com/upload/attach/202403/983110_DYFEAR8KGU8H5Q7.png)

![](https://bbs.kanxue.com/upload/attach/202403/983110_GZGNMUJGYX53HZZ.png)

接下来一路确定下来

![](https://bbs.kanxue.com/upload/attach/202403/983110_K339ZYNERPWCCZ9.png)

![](https://bbs.kanxue.com/upload/attach/202403/983110_UDD2F2JK2X4DP6V.png)

点构建就可以了, 对了, 还要处理一下传参, 再给 python 用 subprocess 调用 cmd 命令就可以了.

这里 windows 下输出老是中文乱码, 昨天整了一个下午, 改这改那都不行, 换成 ubantu 系统一下就好了

![](https://bbs.kanxue.com/upload/attach/202403/983110_DNJVMDHEKPATC28.png)

python 代码就不放了, 相信看到这里你应该也可以弄出来, 请勿对 boss 发送大量请求, 否则后果自负!

**本文章未经许可禁止转载，禁止任何修改后二次传播，擅自使用本文讲解的技术而导致的任何意外，作者均不负责，若有侵权，请联系作者立即删除！**

总结
--

1 由于本节涉及知识点重多, 有很多讲解不到位的地方还请在评论区指出!

2 本章所涉及的材料都上传在网盘了, 刚兴趣的自行还原验证, 相信对你的安卓逆向水平一定会有提升!

3js 逆向转安卓逆向, 如有讲解错误的还请多多包涵!

4 技术交流 + v lyaoyao__i(两个杠)

最后
--

微信公众号: 爬虫爬呀爬

![](https://bbs.kanxue.com/upload/attach/202403/983110_BNXTGBQMDBYCS3D.png)

知识星球

![](https://bbs.kanxue.com/upload/attach/202403/983110_64TFDTYGRD4DJ4S.png)

如果你觉得这篇文章对你有帮助, 不妨请作者喝一杯咖啡吧!

![](https://bbs.kanxue.com/upload/attach/202403/983110_2WWTQ4E64MZQY5V.png)

  

[[CTF 入门培训] 顶尖高校博士及硕士团队亲授《30 小时教你玩转 CTF》，视频 + 靶场 + 题目！助力进入 CTF 世界](http://www.kanxue.com/book-brief-170.htm#h3a6WRhDT9Q_3D)

[#逆向分析](forum-161-1-118.htm)
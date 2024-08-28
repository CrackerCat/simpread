> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-283142.htm)

> [原创] 某大厂 vmp 算法全参 trace 分析

某大厂 vmp 算法全参 trace 分析
=====================

好久没写新的文章了, 因为最近都在弄大厂的样本, 所以时间会比较久, 而且这种类型也比较敏感, 提醒, 仅学习交流, 请勿非法使用.

**写在最前面, 侵权系删!!!** yruhua0

哪个厂暂时不说, 看完你应该知道, 关键位置会暴露 app 的用 xx 代替. 本章将会从抓包到整个算法全流程讲解, 篇幅很长, 如有错别字还请理解.

前置准备
----

### 工具

ida, 010editer, frida, unidbg, jadx,CyberChef,charles(抓包看你们自己, 我喜欢用 vpn 转发, 花瓶 + socksdroid)

### 基础知识

ida,010,frida,jadx 的基本使用, unidbg 的进阶使用, 还有就是离不开的算法分析 (逆向必不可少的), 包括

热门算法

hash: md5,sha1,sha256,sha512, 以及 hamc 组合, 要求熟悉算法细节, 能根据算法特征直接定位明文, 还原出正确的密文, 以及各种魔改方向

分组加密: 主要是 aes 的熟练程度, 秘钥扩展, 十轮运算怎么算的, 以及查表和白盒相关, 然后就是加密模式, ecb,cbc, 填充等等.

冷门算法

冷门算法有很多, 主要是大厂在用

比如 crc32,rc4,sm3 的算法细节, 不要求很熟悉, 但要求能做到拿到算法源码仔细阅读后能达到和热门算法一样的熟练程度

抓包
--

![](https://bbs.kanxue.com/upload/tmp/983110_UYQF254B4HJK3DP.png)

抓包没什么特别的, 过掉 sslping 就可以了, 头部有一个 jdgs, 格式如下

![](https://bbs.kanxue.com/upload/tmp/983110_AACF8DCSJAVVZ5X.png)

抓包对比后 b1 是不变的, 不同版本的不一样, 可以认为是 key,b2 这个没什么, 后面会有介绍, b3:2.1, 版本号, b4, 疑似一段 base64 数据, b5 和 b6, 一直变化, 长度 40,b7, 时间戳. 所以要分析的就是 b4,b5,b6. 抓包的是 13.2.x 版本的, 最新的, 算法分析用 13.x.0, 大概一两个月前发布的, 最终算法通用, 只不过有一些秘钥要从 apk 里拿, 这里不同版本不一样.

定位加密位置
------

肯定是 so 的啦, 哪还有 java 层的算法给你逆?

headers 或者 params 这种键值对的类型很喜欢用 hashmap 添加

```
Java.perform(function (){
    var hashMap = Java.use("java.util.HashMap");
    hashMap.put.implementation = function (a, b) {
        if(a!=null && a.equals("jdgs")){
        console.log(Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Throwable").$new()))
        console.log("hashMap.put: ", a, b);
    }
    return this.put(a, b);
}
})
```

![](https://bbs.kanxue.com/upload/tmp/983110_AWJ49AGGUEQG7JS.png)

有结果, 但不是最终位置, 最终位置在 com.xx.security.xxguard.core.Bridge 这个类下, 我相信你来看大厂的应该不至于堆栈跟踪找不到. 下图是最终结果.

![](https://bbs.kanxue.com/upload/tmp/983110_HXSGEKXCK4YDKTG.png)

右键复制 frida 片段再 hook

![](https://bbs.kanxue.com/upload/tmp/983110_KEH2ZNJXUDC7QS9.png)

入参两个一个数字, 一个 obj 数组

```
[B@99ac1f5,coral|-|coral|-|-|-|-|-|-|-|-|-|-|-|-|-|-§§0|1§§-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-§§-|53684973568§§-|-|1.0|-§§1|1|-|-|-|-|-|-|-|-|-
§§google/coral/coral:10/QD1A.190821.007/5831595:user/release-keys/|-|-|1724741737147|-§§-|-|-|-|-|-|-,eidA9d988120a5s4CYER7EnMT++7gzxHQ7PuFwmBE33cauicjiUlMN8hEQz9Sqr+c2MTtGVN+7IqnBeQMy3v7E2GYbpB21iuG+SC35rf+q406ZICqmRr,1.0,83
```

这个数组传了 5 个参数, 后面 4 个都是字符串, 第一个参数 [B 是字节数组的意思, 要打印可见的数组也很简单, 取第一个元素, 由于是对象数组, 所以要先转型成 [B, 再输出 utf8 格式, 代码如下

```
Java.perform(function (){
    let Bridge = Java.use("com.xx.security.xxguard.core.Bridge");
    Bridge["main"].implementation = function (i2, objArr) {
    console.log(`Bridge.main is called: i2=${i2}, objArr=${objArr}`);
    for (var i = 0; i < objArr.length; i++) {
        if(i===0){
            if(objArr[0]!=null){
                var JavaByte = Java.use("[B");
                var buffer = Java.cast(objArr[0],JavaByte);
                var res = Java.array('byte', buffer);
                //object 先转[B
                var ByteString = Java.use("com.android.okhttp.okio.ByteString");
                console.log('0=',ByteString.of(res).utf8())
                continue
            }
        }
        console.log(`${i}=${objArr[i]}`)
    }
    let result = this["main"](i2, objArr);
    console.log(`Bridge.main result=${result.toString()}`);
    return result;
};
})
```

![](https://bbs.kanxue.com/upload/tmp/983110_TBFSE52BT7CEGV7.png)

所以第一个参数就是请求类型拼接路径和 params 和 data 组成的字符串, 接下来写主动调用, 作用是有一个按钮, 可以随时调用函数, 后面 hook 的时候可以由自己来构造参数, 同时也可观察相同入参, 结果是否变化.

```
function call(){
    Java.perform(function (){
        let Bridge = Java.use("com.xx.security.xxguard.core.Bridge");
        var str0 = 'POST /client.action avifSupport=1&bef=1&build=99208&client=android&clientVersion=13.1.0&ef=1&eid=eidAcc518121b5sdnj7F7Uw4TcaishY7tVQB254%2Bx37JnZ6PeiUW7ppOj%2BnldGjNFrcI%2FmI54gGvpGfOVVYAnIVEgBM8ofUy0hwGx%2B7L5g9B47fttQxV&ep=%7B%22hdid%22%3A%22JM9F1ywUPwflvMIpYPok0tt5k9kW4ArJEU3lfLhxBqw%3D%22%2C%22ts%22%3A1724385686962%2C%22ridx%22%3A-1%2C%22cipher%22%3A%7B%22area%22%3A%22CV83Cv81DJY3DP8m%22%2C%22d_model%22%3A%22UQv4ZWm0WOm%3D%22%2C%22wifiBssid%22%3A%22dW5hbw93bq%3D%3D%22%2C%22osVersion%22%3A%22CJK%3D%22%2C%22d_brand%22%3A%22H29lZ2nv%22%2C%22screen%22%3A%22Ctu4DMenDNGm%22%2C%22uuid%22%3A%22YzG0CtOmDNczDzK5CWVtEG%3D%3D%22%2C%22aid%22%3A%22YzG0CtOmDNczDzK5CWVtEG%3D%3D%22%2C%22openudid%22%3A%22YzG0CtOmDNczDzK5CWVtEG%3D%3D%22%7D%2C%22ciphertype%22%3A5%2C%22version%22%3A%221.2.0%22%2C%22appname%22%3A%22com.jingdong.app.mall%22%7D&ext=%7B%22prstate%22%3A%220%22%2C%22pvcStu%22%3A%221%22%2C%22cfgExt%22%3A%22%7B%5C%22privacyOffline%5C%22%3A%5C%220%5C%22%7D%22%7D&f9c28ecee0666c4febaa160e4e56038d034ec409809b23f32e2f7c22616f4a89=&functionId=uniformRecommend71&harmonyOs=0&lang=zh_CN&networkType=wifi&partner=xiaomi001&recommendSource=9&sdkVersion=29&sign=806911e114d055771f14c8bbc3f89f33&st=1724387382329&sv=101&uemps=2-2-2&x-api-eid-token=jdd01AWSOWPBNUEQE6QYLS626725F4R55GJLSPBYQWRYLDRSLNV27TXG5NK4TEDJR4XQ5TVIO3OBB6KFD3BTLMLNRB2FJLWMZKRU2ZRKWMRA01234567'
        var StringClass = Java.use('java.lang.String');
        var byteArray = StringClass.$new(str0).getBytes();
        var objArr = Java.array('Ljava.lang.Object;',[byteArray,'coral|-|coral|-|-|-|-|-|-|-|-|-|-|-|-|-|-§§0|1§§-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-§§-|53684973568§§-|-|1.0|-§§1|1|-|-|-|-|-|-|-|-|-§§google/coral/coral:10/QD1A.190821.007/5831595:user/release-keys/|-|-|1724385488732|-§§-|-|-|-|-|-|-','eidAcc518121b5sdnj7F7Uw4TcaishY7tVQB254+x37JnZ6PeiUW7ppOj+nldGjNFrcI/mI54gGvpGfOVVYAnIVEgBM8ofUy0hwGx+7L5g9B47fttQxV','1.0','83'])
        let result = Bridge["main"](101, objArr);
        console.log("res:",result)
    })
}
```

这些都是基本操作, 如果不熟可以 google 一下

多次调用结果如下

![](https://bbs.kanxue.com/upload/tmp/983110_BDMCSMRRMSR8WH5.png)

发现 b5 的结果是不变的, b4 和 b6 一直在变, 因为时间戳也在变, 也就是说入参固定的情况下 b5 的值应该要是固定的才对, 接下来用 unidbg 模拟执行, 用 frida 来还原算法工作量会是 unidbg 的好几个数量级, 前面的都是一些基础操作, 后面的都是需要注意的一些细节

unidbg 模拟执行
-----------

推荐用最新版的 unidbg

![](https://bbs.kanxue.com/upload/tmp/983110_NPFZQD5NMKC9CGF.png)

否则有可能报错和我的不一样, 确保下载下来执行里面的 demo 能跑结果

### 把骨架先搭好

#### 写在最前面

疑似 unidbg bug, 由于下面的 patch 的方式不同, so 中走向不同的异常分支, 这里有一个 unidbg 的异常出现时机不一定, 所以我把它写在最前面, 可能是 unidbg 的 bug, 我已经发给凯神了, 在等他的回复, 这里只需要把 unidbg-android/src/main/java/com/github/unidbg/linux/ARM64SyscallHandler.java 下的 mmap 函数的 <<MMAP2_SHIFT 注释掉, 只要这里注释下, 下面的就不会有问题. 这个 apk 的初始化异常分支很多 (不懂初始化可以往下看). **注意, 一定要改**

![](https://bbs.kanxue.com/upload/tmp/983110_2JY9UY7QNNPZQBG.png)

![](https://bbs.kanxue.com/upload/tmp/983110_SP9V3M2EKE8Z8CE.png)

异常的字眼 java.io.IOException: Negative seek offset

#### 搭骨架

改好后再进行下面的, 搭骨架

一定要按顺序, 先把 jniOnload 部分执行起来, 不然你直接去调用函数到时候报很多错会弄的很乱

```
package com.xx;
 
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.memory.Memory;
import com.github.unidbg.virtualmodule.android.AndroidModule;
 
import java.io.File;
 
public class jdgs2 extends AbstractJni{
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
    jdgs2(){
        // 创建模拟器实例
        emulator = AndroidEmulatorBuilder.for64Bit().setProcessName("包名自己填").setRootDir(new File("target/rootfs")).build();
        // 获取模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        // 创建Android虚拟机,传入APK，Unidbg可以替我们做部分签名校验的工作
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/java/com/xx/files//13.x.0.apk"));
        // 设置JNI
        vm.setJni(this);
        // 打印日志
        vm.setVerbose(true);
        // 加载目标SO
        DalvikModule dm = vm.loadLibrary("jdg", true);
        //获取本SO模块的句柄,后续需要用它
        module = dm.getModule();
        // 调用JNI OnLoad
        dm.callJNI_OnLoad(emulator);
    };
    public static void main(String[] args) {
        jdgs2 demo = new jdgs2();
    }
}
```

![](https://bbs.kanxue.com/upload/tmp/983110_EXBNJYFG72QNVQ4.png)

执行下有个 info, 缺少 libandroid.so, 这个 so 依赖很多 so, 不好加载, unidbg 实现了它的部分函数, 可以注册虚拟模块.

```
jdgs2(){
        // 创建模拟器实例
        emulator = AndroidEmulatorBuilder.for64Bit().setProcessName("com.xxx.app.mall").setRootDir(new File("target/rootfs")).build();
        // 获取模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        // 创建Android虚拟机,传入APK，Unidbg可以替我们做部分签名校验的工作
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/java/com/xx/files/xx/xx13.x.0.apk"));
        // 注册虚拟Android模块
        new AndroidModule(emulator, vm).register(memory);
        // 设置JNI
        vm.setJni(this);
        // 打印日志
        vm.setVerbose(true);
        // 加载目标SO
        DalvikModule dm = vm.loadLibrary("jdg", true);
        //获取本SO模块的句柄,后续需要用它
        module = dm.getModule();
        // 调用JNI OnLoad
        dm.callJNI_OnLoad(emulator);
    };
```

执行后成功跑起来了, 恭喜你, 完成了第一步, 同时可以看到函数在 so 里的偏移是 0x29ce4

![](https://bbs.kanxue.com/upload/tmp/983110_FMYB26GRQE4U7KC.png)

第二步, 开始调用函数

```
public void callByAddress(){
        // args list
        List list = new ArrayList<>(4);
        // jnienv
        list.add(vm.getJNIEnv());
        // jclazz
        list.add(0);
        list.add(101);
        byte[] bytes = "POST /client.action avifSupport=1&bef=1&build=99208&client=android&clientVersion=13.1.0&ef=1&eid=eidAcc518121b5sdnj7F7Uw4TcaishY7tVQB254%2Bx37JnZ6PeiUW7ppOj%2BnldGjNFrcI%2FmI54gGvpGfOVVYAnIVEgBM8ofUy0hwGx%2B7L5g9B47fttQxV&ep=%7B%22hdid%22%3A%22JM9F1ywUPwflvMIpYPok0tt5k9kW4ArJEU3lfLhxBqw%3D%22%2C%22ts%22%3A1724385686962%2C%22ridx%22%3A-1%2C%22cipher%22%3A%7B%22area%22%3A%22CV83Cv81DJY3DP8m%22%2C%22d_model%22%3A%22UQv4ZWm0WOm%3D%22%2C%22wifiBssid%22%3A%22dW5hbw93bq%3D%3D%22%2C%22osVersion%22%3A%22CJK%3D%22%2C%22d_brand%22%3A%22H29lZ2nv%22%2C%22screen%22%3A%22Ctu4DMenDNGm%22%2C%22uuid%22%3A%22YzG0CtOmDNczDzK5CWVtEG%3D%3D%22%2C%22aid%22%3A%22YzG0CtOmDNczDzK5CWVtEG%3D%3D%22%2C%22openudid%22%3A%22YzG0CtOmDNczDzK5CWVtEG%3D%3D%22%7D%2C%22ciphertype%22%3A5%2C%22version%22%3A%221.2.0%22%2C%22appname%22%3A%22com.jingdong.app.mall%22%7D&ext=%7B%22prstate%22%3A%220%22%2C%22pvcStu%22%3A%221%22%2C%22cfgExt%22%3A%22%7B%5C%22privacyOffline%5C%22%3A%5C%220%5C%22%7D%22%7D&f9c28ecee0666c4febaa160e4e56038d034ec409809b23f32e2f7c22616f4a89=&functionId=uniformRecommend71&harmonyOs=0&lang=zh_CN&networkType=wifi&partner=xiaomi001&recommendSource=9&sdkVersion=29&sign=806911e114d055771f14c8bbc3f89f33&st=1724387382329&sv=101&uemps=2-2-2&x-api-eid-token=jdd01AWSOWPBNUEQE6QYLS626725F4R55GJLSPBYQWRYLDRSLNV27TXG5NK4TEDJR4XQ5TVIO3OBB6KFD3BTLMLNRB2FJLWMZKRU2ZRKWMRA01234567".getBytes();
        ByteArray arr = new ByteArray(vm, bytes);
        vm.addLocalObject(arr);
        StringObject str_1 = new StringObject(vm, "coral|-|coral|-|-|-|-|-|-|-|-|-|-|-|-|-|-§§0|1§§-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-§§-|53684973568§§-|-|1.0|-§§1|1|-|-|-|-|-|-|-|-|-§§google/coral/coral:10/QD1A.190821.007/5831595:user/release-keys/|-|-|1724385488732|-§§-|-|-|-|-|-|-");
        vm.addLocalObject(str_1);
        StringObject str_2 = new StringObject(vm, "eidAcc518121b5sdnj7F7Uw4TcaishY7tVQB254+x37JnZ6PeiUW7ppOj+nldGjNFrcI/mI54gGvpGfOVVYAnIVEgBM8ofUy0hwGx+7L5g9B47fttQxV");
        vm.addLocalObject(str_2);
        StringObject str_3 = new StringObject(vm, "1.0");
        vm.addLocalObject(str_3);
        StringObject str_4 = new StringObject(vm, "83");
        vm.addLocalObject(str_4);
        ArrayObject arrayObject = new ArrayObject(arr,str_1,str_2,str_3,str_4);
        list.add(vm.addLocalObject(arrayObject));
        Number number = module.callFunction(emulator, 0x29CE4, list.toArray());
        ArrayObject resultArr = vm.getObject(number.intValue());
        System.out.println("result:"+resultArr);
    }; 
```

我喜欢用地址来调, 当然也可通过 api 调, 看个人习惯

![](https://bbs.kanxue.com/upload/tmp/983110_HMA6XTBTAGZ2AUN.png)

直接调用没有补环境的部分, 而是直接报了一个日志, 这个红色的是 so 输出的, 不是 unidbg 输出的, Error pp not init, 意思也很明显, 未初始化.

为什么要初始化? 其实这个主要是用来 anti unidbg 的, 如果你直接调就出结果了那不是一下就被破解了吗? 大厂普遍都有, 中小厂也并不少见.

所以该怎么办?

正确的方法就是 frida hook 这个方法的初始化过程, apk 怎么做的, 我们就在 unidbg 里照做.

### hook init

只有一个函数, 难道初始化和调用共用同一个函数吗? 没错, 现在大厂基本都是这样, 比如 dy, 阿里, mt 这些都是, 函数怎么知道自己是要调用出结果还是初始化呢? 很简单, 传不同的参数呗, 比如这个样本, 第一个参数一个数字, 第二个参数一个 obj 数组, 完全可以通过传不同数字来决定, 事实上, 大厂就是这么干的.

接下来用 frida 来 hook 这个初始化过程

```
Java.perform(function (){
    let Bridge = Java.use("com.xx.security.xxguard.core.Bridge");
    Bridge["main"].implementation = function (i2, objArr) {
    console.log(`Bridge.main is called: i2=${i2}, objArr=${objArr}`);
    let result = this["main"](i2, objArr);
    console.log(`Bridge.main result=${result.toString()}`);
    return result;
};
})
```

以 spawn 启动即可, 如下图, 最先执行两个 103, 返回两个 0 后执行 101 就有结果了, 猜测需要先执行 103, 返回两个 0 再调用应该能出正确结果

![](https://bbs.kanxue.com/upload/tmp/983110_7ZTBNBZNQ7WBWJX.png)

怎么验证?

```
Java.perform(function (){
    let Bridge = Java.use("com.xx.security.xxguard.core.Bridge");
    Bridge["main"].implementation = function (i2, objArr) {
    console.log(`Bridge.main is called: i2=${i2}, objArr=${objArr}`);
    let result = this["main"](i2, objArr);
    console.log(`Bridge.main result=${result.toString()}`);
    if(i2===103){
        console.log('===================')
        call()
    }
    return result;
};
})
function call(){
    Java.perform(function (){
        let Bridge = Java.use("com.xx.security.xxguard.core.Bridge");
        var str0 = 'POST /client.action avifSupport=1&bef=1&build=99208&client=android&clientVersion=13.1.0&ef=1&eid=eidAcc518121b5sdnj7F7Uw4TcaishY7tVQB254%2Bx37JnZ6PeiUW7ppOj%2BnldGjNFrcI%2FmI54gGvpGfOVVYAnIVEgBM8ofUy0hwGx%2B7L5g9B47fttQxV&ep=%7B%22hdid%22%3A%22JM9F1ywUPwflvMIpYPok0tt5k9kW4ArJEU3lfLhxBqw%3D%22%2C%22ts%22%3A1724385686962%2C%22ridx%22%3A-1%2C%22cipher%22%3A%7B%22area%22%3A%22CV83Cv81DJY3DP8m%22%2C%22d_model%22%3A%22UQv4ZWm0WOm%3D%22%2C%22wifiBssid%22%3A%22dW5hbw93bq%3D%3D%22%2C%22osVersion%22%3A%22CJK%3D%22%2C%22d_brand%22%3A%22H29lZ2nv%22%2C%22screen%22%3A%22Ctu4DMenDNGm%22%2C%22uuid%22%3A%22YzG0CtOmDNczDzK5CWVtEG%3D%3D%22%2C%22aid%22%3A%22YzG0CtOmDNczDzK5CWVtEG%3D%3D%22%2C%22openudid%22%3A%22YzG0CtOmDNczDzK5CWVtEG%3D%3D%22%7D%2C%22ciphertype%22%3A5%2C%22version%22%3A%221.2.0%22%2C%22appname%22%3A%22com.jingdong.app.mall%22%7D&ext=%7B%22prstate%22%3A%220%22%2C%22pvcStu%22%3A%221%22%2C%22cfgExt%22%3A%22%7B%5C%22privacyOffline%5C%22%3A%5C%220%5C%22%7D%22%7D&f9c28ecee0666c4febaa160e4e56038d034ec409809b23f32e2f7c22616f4a89=&functionId=uniformRecommend71&harmonyOs=0&lang=zh_CN&networkType=wifi&partner=xiaomi001&recommendSource=9&sdkVersion=29&sign=806911e114d055771f14c8bbc3f89f33&st=1724387382329&sv=101&uemps=2-2-2&x-api-eid-token=jdd01AWSOWPBNUEQE6QYLS626725F4R55GJLSPBYQWRYLDRSLNV27TXG5NK4TEDJR4XQ5TVIO3OBB6KFD3BTLMLNRB2FJLWMZKRU2ZRKWMRA01234567'
        var StringClass = Java.use('java.lang.String');
        var byteArray = StringClass.$new(str0).getBytes();
        var objArr = Java.array('Ljava.lang.Object;',[byteArray,'coral|-|coral|-|-|-|-|-|-|-|-|-|-|-|-|-|-§§0|1§§-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-§§-|53684973568§§-|-|1.0|-§§1|1|-|-|-|-|-|-|-|-|-§§google/coral/coral:10/QD1A.190821.007/5831595:user/release-keys/|-|-|1724385488732|-§§-|-|-|-|-|-|-','eidAcc518121b5sdnj7F7Uw4TcaishY7tVQB254+x37JnZ6PeiUW7ppOj+nldGjNFrcI/mI54gGvpGfOVVYAnIVEgBM8ofUy0hwGx+7L5g9B47fttQxV','1.0','83'])
        let result = Bridge["main"](101, objArr);
        console.log("res:",result)
    })
}
```

很简单

```
if(i2===103){
        console.log('===================')
        call()
    }
```

每执行一次 103 后都主动调用一次, 看看是否出结果

![](https://bbs.kanxue.com/upload/tmp/983110_CZ8WM825BPM8822.png)

如上, 调一次 103 后直接 call 返回一个数字, 第二次调 103 后 call 就是结果, 注意, 这两个调 103 的过程是 app 自己完成的, 我这样写最简洁, 又可以判断出到底哪些过程是初始化.

所以需要先 call 两次 103, 第一次传 0, 第二次传 1

### unidbg init

```
public void callInit(String arg){
        // args list
        List list = new ArrayList<>(4);
        // jnienv
        list.add(vm.getJNIEnv());
        // jclazz
        list.add(0);
        list.add(103);
        StringObject arg0 = new StringObject(vm, arg);
        vm.addLocalObject(arg0);
        ArrayObject arrayObject = new ArrayObject(arg0);
        list.add(vm.addLocalObject(arrayObject));
        Number number = module.callFunction(emulator, 0x29CE4, list.toArray());
        ArrayObject arrayObject2 = vm.getObject(number.intValue());
        System.out.println(arrayObject2.getValue()[0].getValue());
 
    }
public static void main(String[] args) {
        jdgs2 demo = new jdgs2();
        demo.callInit("0");
        demo.callInit("1");
//        demo.callByAddress();
    } 
```

代码如上, 调用后结果如下, 但是第二次 call 的时候返回的是一个负数, 正常应该是 0, 和 app 不同千万不要着急调用函数, 要先排除出错误

![](https://bbs.kanxue.com/upload/tmp/983110_E4WKDPUYEMKY6K9.png)

很多同学到这里不知道怎么办了. 别着急, 如果我告诉你上面的 jni 日志中有一个地方有问题, 它是专门用来反制 unidbg 的, 仔细从上往下观察 3 遍, 如果你能找出来说明你 unidbg 基础很扎实, 如果你发现了, 也可以打在评论区哦, 这个我们后面揭晓, 我当时也没看出来, 因为这个坑有点费眼神, 不过没关系, 我们一步一步排查. 肯定要去 so 里排查, 所以先把 so 反编译一下

### 反编译 so + 去花

只有 64 位的, 用 ida64 打开, 当然 9.0 的不分 32 和 64 了, 我用的是 8.3

![](https://bbs.kanxue.com/upload/tmp/983110_HEPVBFPPNWPGRZH.png)

打开先看 jniOnload, 有花指令, 再正常不过了, 大厂的都有花

![](https://bbs.kanxue.com/upload/tmp/983110_BT2YU96YDS2SDNJ.png)

上面那块有没有很熟悉?

某手的 kwsgmain 的花

```
.text:000000000004ABB4 E0 07 BE A9                   STP             X0, X1, [SP,#-32]!
.text:000000000004ABB8 E2 7B 01 A9                   STP             X2, X30, [SP,#16]
.text:000000000004ABBC 01 01 00 10                   ADR             X1, dword_4ABDC
.text:000000000004ABC0 21 10 00 F1                   SUBS            X1, X1, #4
.text:000000000004ABC4 E0 03 01 AA                   MOV             X0, X1
.text:000000000004ABC8 00 D0 00 B1                   ADDS            X0, X0, #0x34 ; '4'
.text:000000000004ABCC E0 0F 00 F9                   STR             X0, [SP,#24]
.text:000000000004ABD0 E2 27 41 A9                   LDP             X2, X9, [SP,#16]
.text:000000000004ABD4 E0 07 C2 A8                   LDP             X0, X1, [SP],#0x20
.text:000000000004ABD8 20 01 1F D6                   BR              X9
```

不能说一模一样, 十有八九了吧.

还是匹配开头部分的 E0 07 BE A9 E2 7B 01 A9

```
import keystone
from keystone import *
import ida_bytes
import idaapi
import idc
import flare_emu
 
# 来自叶谷雨的代码 O(∩_∩)O
def binSearch(start, end, pattern):
    matches = []
    addr = start
    if end == 0:
        end = idc.BADADDR
    if end != idc.BADADDR:
        end = end + 1
    while True:
        addr = ida_bytes.bin_search(addr, end, bytes.fromhex(pattern), None, idaapi.BIN_SEARCH_FORWARD,
                                    idaapi.BIN_SEARCH_NOCASE)
        if addr == idc.BADADDR:
            break
        else:
            matches.append(addr)
            addr = addr + 1
    return matches
 
 
myEH = flare_emu.EmuHelper()
def getJumpAddress(addr):
    myEH.emulateRange(startAddr=addr, endAddr=addr + 36)
    return myEH.getRegVal("X9")
 
 
def makeInsn(addr):
    if idc.create_insn(addr) == 0:
        idc.del_items(addr, idc.DELIT_EXPAND)
        idc.create_insn(addr)
    idc.auto_wait()
 
 
def generate(code, addr):
    ks = Ks(keystone.KS_ARCH_ARM64, keystone.KS_MODE_LITTLE_ENDIAN)
    encoding, _ = ks.asm(code, addr)
    return encoding
 
 
matches = binSearch(0, 0, "E0 07 BE A9 E2 7B 01 A9")
for addr in matches:
    print("try:" + hex(addr))
    makeInsn(addr)
    targetAddr = getJumpAddress(addr)
    code = f"B {hex(targetAddr)}"
    bCode = generate(code, addr)
    nopCode = generate("nop", 0)
    ida_bytes.patch_bytes(addr, bytes(bCode))
    ida_bytes.patch_bytes(addr + 4, bytes(nopCode) * 9)
 
print('finish')
```

还是之前的代码, 由龙哥编写, 如果不用 ida 脚本, 直接 patch 也是可以的, 只不过麻烦些, 具体的可以看龙哥这篇文章 [https://www.yuque.com/lilac-2hqvv/zfho3g/issny5?#yGLEd](https://www.yuque.com/lilac-2hqvv/zfho3g/issny5?#yGLEd)

patch 回填后重新打开, 全部正常了

![](https://bbs.kanxue.com/upload/tmp/983110_ERZEWKR7EJ23HE5.png)

### 排查异常分支

![](https://bbs.kanxue.com/upload/tmp/983110_Q247A3YN84CT3KT.png)

结果异常明显是走了异常分支, 这个时候从出现异常的地方往上回溯是比较高效的方法, 结尾返回了一个 - 3301, 就以这个为突破口, 首先找到目标函数地址 0x29ce4, 进来先转换 jniEnv, 看到清晰些

![](https://bbs.kanxue.com/upload/tmp/983110_6Q8XWM9KZW7JE3H.png)

流程图长这样, 这只是 ollvm 过了, vmp 还在后头

![](https://bbs.kanxue.com/upload/tmp/983110_C68D4J9SBXD8752.png)

前面是一些变量定义, 然后就是一个 switch case 分发

![](https://bbs.kanxue.com/upload/tmp/983110_SM2TJWSPBN39XGK.png)

转汇编视图可以看到上面的 case e 对应 101 分支, 101 我们还没执行, 我们要找的是 103 分支

```
case 'g':
        dword_B0154 = 0;
        dword_B0150 = 0;
        v15 = (*env)->GetObjectArrayElement(env, a4, 0LL);
        v16 = (*env)->ExceptionCheck(env);
        dword_B0154 = 0;
        dword_B0150 = 0;
        if ( v16 == 1 )
        {
          (*env)->ExceptionClear(env);
          v17 = 0LL;
          v6 = 0LL;
          dword_B0154 = 0;
          dword_B0150 = 0;
          if ( !v15 )
            goto LABEL_220;
          goto LABEL_68;
        }
        dword_B0154 = 0;
        dword_B0150 = 0;
        if ( !v15 )
        {
          v28 = -3104;
LABEL_36:
          v6 = sub_2D43C(env, v28);
LABEL_162:
          dword_B0154 = 0;
          dword_B0150 = 0;
LABEL_220:
          dword_B0154 = 0;
          dword_B0150 = 0;
          break;
        }
        sub_32E70(env, v15);
        if ( (v101 & 1) != 0 )
          v23 = v103;
        else
          v23 = v102;
        v24 = atoi(v23);
        dword_B0154 = 0;
        dword_B0150 = 0;
        if ( v24 != 1 )
        {
          if ( v24 )
          {
            v17 = 0LL;
LABEL_65:
            dword_B0154 = 0;
            dword_B0150 = 0;
            if ( (v101 & 1) != 0 )
              j__free(v103);
            dword_B0154 = 0;
            dword_B0150 = 0;
LABEL_68:
            (*env)->DeleteLocalRef(env, v15);
            v6 = v17;
            goto LABEL_220;
          }
          dword_B0154 = 0;
          dword_B0150 = 0;
          do
            v25 = __ldaxr(&dword_AC900);
          while ( __stlxr(v25 | 1, &dword_AC900) );
          v26 = env;
          v27 = 0;
LABEL_64:
          v17 = sub_2D43C(v26, v27);
          goto LABEL_65;
        }
        dword_B0154 = 0;
        dword_B0150 = 0;
        v32 = sub_43954(env);
        dword_B0154 = 0;
        dword_B0150 = 0;
        if ( v32 )
        {
          v27 = -3301;
LABEL_63:
          v26 = env;
          goto LABEL_64;
        }
        sub_358E0(env);
        if ( (s & 1) != 0 )
        {
          v37 = *(&s + 1);
          j__free(v105);
          dword_B0154 = 0;
          dword_B0150 = 0;
          if ( !v37 )
          {
LABEL_51:
            memset(&s, 0, 0x90uLL);
            sub_2F9DC(env);
            sub_1E228(&s, &v87);
            if ( (v87 & 1) != 0 )
              j__free(ptr);
            sub_30494(env);
            sub_1E228(&v108[1], &v87);
            if ( (v87 & 1) != 0 )
              j__free(ptr);
            sub_3180C(env);
            sub_1E228(&v110, &v87);
            if ( (v87 & 1) != 0 )
              j__free(ptr);
            sub_30F9C(env);
            sub_1E228(&v105 + 8, &v87);
            if ( (v87 & 1) != 0 )
              j__free(ptr);
            v36 = sub_30CD0(env);
            sub_6487C(v36);
            if ( (v107 & 1) != 0 )
            {
              *v108[0] = 0;
              *(&v107 + 1) = 0LL;
              if ( (v107 & 1) != 0 )
              {
                j__free(v108[0]);
                *&v107 = 0LL;
              }
            }
            else
            {
              LOWORD(v107) = 0;
            }
            v108[0] = ptr;
            v107 = v87;
            sub_316D4(env);
            sub_1E228(v113, &v87);
            if ( (v87 & 1) != 0 )
              j__free(ptr);
            sub_30238(env);
            sub_31298(env);
            sub_3159C(env);
            dword_B0154 = 0;
            dword_B0150 = 0;
            if ( (s & 1) != 0 )
              v42 = *(&s + 1);
            else
              v42 = s >> 1;
            if ( !v42 )
              goto LABEL_121;
            dword_B0154 = 0;
            dword_B0150 = 0;
            if ( !((v108[1] & 1) != 0 ? v109 : LOBYTE(v108[1]) >> 1) )
              goto LABEL_121;
            dword_B0154 = 0;
            dword_B0150 = 0;
            if ( !((v110 & 1) != 0 ? v111 : v110 >> 1) )
              goto LABEL_121;
            dword_B0154 = 0;
            dword_B0150 = 0;
            if ( !((BYTE8(v105) & 1) != 0 ? v106[0] : (BYTE8(v105) >> 1)) )
              goto LABEL_121;
            dword_B0154 = 0;
            dword_B0150 = 0;
            if ( !((v107 & 1) != 0 ? *(&v107 + 1) : v107 >> 1) )
              goto LABEL_121;
            dword_B0154 = 0;
            dword_B0150 = 0;
            if ( !((v113[0] & 1) != 0 ? v114 : v113[0] >> 1) )
              goto LABEL_121;
            dword_B0154 = 0;
            dword_B0150 = 0;
            v48 = (v99[0] & 1) != 0 ? v99[1] : (LOBYTE(v99[0]) >> 1);
            if ( !v48
              || ((dword_B0154 = 0, dword_B0150 = 0, (endptr[0] & 1) != 0)
                ? (v49 = endptr[1])
                : (v49 = (LOBYTE(endptr[0]) >> 1)),
                  !v49 || ((dword_B0154 = 0, dword_B0150 = 0, (v94 & 1) != 0) ? (v50 = v95) : (v50 = v94 >> 1), !v50)) )
            {
LABEL_121:
              dword_B0154 = 0;
              dword_B0150 = 0;
              v17 = sub_2D43C(env, 0xFFFFF3E0);
LABEL_122:
              dword_B0154 = 0;
              dword_B0150 = 0;
              if ( (v94 & 1) != 0 )
                j__free(v96);
              if ( (endptr[0] & 1) != 0 )
                j__free(v98);
              if ( (v99[0] & 1) != 0 )
                j__free(v100);
              if ( (v113[0] & 1) != 0 )
                j__free(v115);
              if ( (v110 & 1) != 0 )
                j__free(v112);
              if ( (v108[1] & 1) != 0 )
                j__free(*(&v109 + 1));
              if ( (v107 & 1) != 0 )
                j__free(v108[0]);
              if ( (BYTE8(v105) & 1) != 0 )
                j__free(v106[1]);
              if ( (s & 1) != 0 )
                j__free(v105);
              goto LABEL_65;
            }
            dword_B0154 = 0;
            dword_B0150 = 0;
            v92 = 0LL;
            v90 = 0LL;
            v51 = sub_324B8(v99, endptr, &v94, &v91, &v90, &v93, &v92);
            dword_B0154 = 0;
            dword_B0150 = 0;
            if ( (v51 & 1) != 0 )
            {
              v52 = v93;
              v53 = v91;
              dword_B0154 = 0;
              dword_B0150 = 0;
              if ( v93 && v92 && v91 && v90 )
              {
                ptr = v91;
                v89 = v90;
                *&v87 = v93;
                *(&v87 + 1) = v92;
                v54 = sub_1F688(env, &s, &v87);
                dword_B0154 = 0;
                dword_B0150 = 0;
                if ( v54 )
                {
                  v17 = sub_2D43C(env, v54);
                }
                else
                {
                  if ( (byte_ACAAA & 1) == 0 )
                  {
                    sub_19DDC(asc_A04F5, &unk_A0E38, 0xAu, 2u);
                    byte_A04F7 = 0;
                  }
                  byte_ACAAA = 1;
                  sub_1E0E4(v85, asc_A04F5);
                  sub_1DE30(v85, v86);
                  if ( (v85[0] & 1) != 0 )
                    j__free(v85[2]);
                  dword_B0154 = 0;
                  dword_B0150 = 0;
                  if ( (v86[0] & 1) != 0 )
                    v78 = v86[1];
                  else
                    v78 = LOBYTE(v86[0]) >> 1;
                  if ( v78 )
                  {
                    v79 = sub_23250(&v108[1], v86);
                    dword_B0154 = 0;
                    dword_B0150 = 0;
                    if ( (v79 & 1) != 0 )
                    {
                      sub_3E634(env);
                      do
                        v80 = __ldaxr(&dword_AC900);
                      while ( __stlxr(v80 | 2, &dword_AC900) );
                      v81 = 0;
                    }
                    else
                    {
                      v81 = -3106;
                    }
                  }
                  else
                  {
                    v81 = -3105;
                  }
                  v17 = sub_2D43C(env, v81);
                  dword_B0154 = 0;
                  dword_B0150 = 0;
                  if ( (v86[0] & 1) != 0 )
                    j__free(v86[2]);
                }
                dword_B0154 = 0;
                dword_B0150 = 0;
                free(v53);
                free(v52);
                goto LABEL_241;
              }
              v76 = -3101;
            }
            else
            {
              v76 = -3102;
            }
            v17 = sub_2D43C(env, v76);
LABEL_241:
            dword_B0154 = 0;
            dword_B0150 = 0;
            goto LABEL_122;
          }
        }
        else
        {
          dword_B0154 = 0;
          dword_B0150 = 0;
          if ( s < 2u )
            goto LABEL_51;
        }
        v27 = -3302;
        goto LABEL_63;
```

搜索发现 - 3301 经过 v32 真假判断后被赋值了, 所以这个 v32 的来源很重要, 来自 sub_43954 这个的返回值

![](https://bbs.kanxue.com/upload/tmp/983110_CWXY2ZAGEMCPWKP.png)

如下图, 返回值来自 env findclass, 第二个参数就是要找的类, v32 的结果为 1, 说明 v2 为 1, 也就是说这个类找到了, 但是由于字符串加密了, stru_A81C6 点过去看不出来是什么, 最简单的办法 unidbg 在这行汇编下个断看下

![](https://bbs.kanxue.com/upload/tmp/983110_Q6H25XGSSW84QNH.png)

鼠标放在 findcalss 按 tab 转汇编视图

![](https://bbs.kanxue.com/upload/tmp/983110_4X8XU3U5AVFAEZA.png)

你想要知道哪个是参数你得懂汇编的意思, 比如 00 01 3F D6 BLR X8, 你只需记住 B Branch, 跳转, 剩下的都是它的变体, 只是略有区别, 所以这行汇编就是要跳到 findclass 函数执行了, 参数呢? 根据 arm64 调用约定, 参数 1X7 寄存器中 ，剩下的参数从右往左一次入栈，被调用者实现栈平衡，返回值存放在 X0 中。所以 x1 就是要找的类, 下断看下 43A14

```
public void HookByConsoleDebugger() {
        Debugger debugger = emulator.attach();
        debugger.addBreakPoint(module.base+0x43A14);
    }
```

![](https://bbs.kanxue.com/upload/tmp/983110_NZ2P7J9YNQ5WDBN.png)

看到这个 java/lang/string 应该明白了吧 java 压根没有这个类啊, 正确的应该是大写的 S String, 所以这是安全人员专门给 unidbg 挖的坑, 就是让你跑不起来. 还有种方法看这个加密的字符串就是用 dump 下来的 so, 内存中的是解密的.

![](https://bbs.kanxue.com/upload/tmp/983110_ES4WRMUUSU72DY9.png)

但是跑 unidbg 最好不要用 dump 后的

然后知道问题所在就很好办了

第一种 ida patch so, 但是不推荐, 最好不要直接改 so

第二种 hook 它的返回值, 让它返回 0

```
public void patch(){
        UnidbgPointer pointer = UnidbgPointer.pointer(emulator,module.base + 0x2a148);
        byte[] code = new byte[]{(byte) 0x40, (byte) 0x05,0x00, (byte) 0xB5};// 400500B5  find class找不到 直接走向错误分支
        pointer.write(code);
    }
```

第三种 unidbg 也想到了这个, 可以在 vm 注册后 加一句 vm.addNotFoundClass("java/lang/string");

因为 unidbg 判断不了一个类是否真的存在, 所以默认只要你找这个类, 就是存在的, 安全人员可以设一个隐蔽的坑, 修改一个不存在的类, 真机找不到返回的是 0, 但是 unidbg 返回的是 1, 而且看 jni 日志很难分辨出来.

### 检测 emulator

排查掉这个分支后运行下

![](https://bbs.kanxue.com/upload/tmp/983110_EMKA345U32RCHT5.png)

接下来就是补环境的过程了, 不过上面有 189 条警告, 最好处理下再补环境, 以免走入新的异常

点 ARM64SyscallHandler 跳过去看看, 上面的 NR 都是 49

![](https://bbs.kanxue.com/upload/tmp/983110_55XJBMS89NEE26C.png)

![](https://bbs.kanxue.com/upload/tmp/983110_QGYMX4AQBUNY5A7.png)

![](https://bbs.kanxue.com/upload/tmp/983110_ZC35RECZ7QTT3TS.png)

unidbg 没有实现这个 case 的 syscall, 所以弹警告了, 这个 49 是什么指令? 可以在 [https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md#arm64-64_bit](https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md#arm64-64_bit) 查看

![](https://bbs.kanxue.com/upload/tmp/983110_XUE2BR4UHAA5P6R.png)

chdir 也就是常说的 cd app 初始化的时候切换目录干啥? 看看它要切到哪些目录去, LR 指向的是 0x23ab8, 它的上一行就是 cd, 到 ida 中看看

![](https://bbs.kanxue.com/upload/tmp/983110_3JU2THJEDRHRWXB.png)

23aB4 下断

![](https://bbs.kanxue.com/upload/tmp/983110_9VH7E23NBE6PCJ9.png)

![](https://bbs.kanxue.com/upload/tmp/983110_5R5NFP8CRDPHY6N.png)

![](https://bbs.kanxue.com/upload/tmp/983110_7SYJ6MCN7W44E6X.png)

明白了吧, 检测模拟器

![](https://bbs.kanxue.com/upload/tmp/983110_XZ6FHZ5BF3PDJM5.png)

在 dump 下的 so 里可以看到这些路径, 都是模拟器相关的, unidbg 没有这个 case, 所以不用管, 直接补环境就好了

### 补环境

```
@Override
    public DvmObject callStaticObjectMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
        switch (signature) {
            case "com/xx/security/xxguard/core/Bridge->getAppContext()Landroid/content/Context;":
                return vm.resolveClass("android/content/Context").newObject(null);
            case "com/xx/security/xxguard/core/Bridge->getAppKey()Ljava/lang/String;":{
                return new StringObject(vm,"6bbc8976-d0fe-44ef-9776-ab9da0c127e8");
            }
            case "com/xx/security/xxguard/core/Bridge->getJDGVN()Ljava/lang/String;":{
                return new StringObject(vm,"3.2.8.4");
            }
            case "com/xx/security/xxguard/core/Bridge->getPicName()Ljava/lang/String;":{
                return new StringObject(vm,"6bbc8976-d0fe-44ef-9776-ab9da0c127e8.jdg.jpg");
            }
            case "com/xx/security/xxguard/core/Bridge->getSecName()Ljava/lang/String;":{
                return new StringObject(vm,"6bbc8976-d0fe-44ef-9776-ab9da0c127e8.jdg.xbt");
            }
        }
        return super.callStaticObjectMethodV(vm, dvmClass, signature, vaList);
    }
    @Override
    public DvmObject dvmObject, String signature, VaList vaList) {
        switch (signature){
            case "android/content/Context->getPackageCodePath()Ljava/lang/String;":{
                return new StringObject(vm, "/data/app/com.xxx.app.mall-NMPcDqBzdYwQrAm-23cOyQ==/base.apk");
            }
        }
        return super.callObjectMethodV(vm, dvmObject, signature, vaList);
    }
```

都是些基础操作

![](https://bbs.kanxue.com/upload/tmp/983110_XFZ9DK787QDY78T.png)

补完返回另一个负值 - 3102, 还是上面那样的排查, 很容易就定位到它读 apk 资源没读到, jni 中也有体现

![](https://bbs.kanxue.com/upload/tmp/983110_7YS9E47DGSWK96E.png)

把下面框中的补上就可以看到它读了哪些

![](https://bbs.kanxue.com/upload/tmp/983110_KYENRTRECK8855W.png)

![](https://bbs.kanxue.com/upload/tmp/983110_EN5S47UUBTUVVWQ.png)

Open File:/data/app/com, 读了 apk 就把本地的 apk 给它

```
@Override
public FileResult resolve(Emulator emulator, String pathname, int oflags) {
    System.out.println("Open File:" + pathname);
    switch (pathname) {
        case "/data/app/com.xxx.app.mall-NMPcDqBzdYwQrAm-23cOyQ==/base.apk": {
            return FileResult.success(new SimpleFileIO(oflags, new File("unidbg-android/src/test/java/com/xx/files/xx/xx13.1.0.apk"), pathname));
        }
    }
    return null;
}
```

然后就正常了

![](https://bbs.kanxue.com/upload/tmp/983110_8GDCXE2PSTQNDN9.png)

### 调用目标函数

![](https://bbs.kanxue.com/upload/tmp/983110_NP6HU4F6EAWXVFK.png)

直接出结果, 但是是一直变的, 猜测随机数, 时间戳等等, 测试只要改 unidbg-android/src/main/java/com/github/unidbg/linux/ARM64SyscallHandler.java 下的 gettimeofday64 的 currentTimeMillis() 改成固定 13 位就可以了, 基本都是改这里, 32 的就改 gettimeofday

![](https://bbs.kanxue.com/upload/tmp/983110_7HRJD396RETSBQR.png)

```
{"b1":"6bbc8976-d0fe-44ef-9776-ab9da0c127e8","b2":"3.2.8.4_0","b3":"2.1","b4":"Y+0jkt2FIdCIV7z0mmcidio1XB/pDmDb6Wq9lXKhNiJvGwwHAPoKQJ8EJhQEF0Ssf1M9komvy2ufFocI2HvAcoq93b3n1kyR1xQyPFFteqltXADYy/z8oUryNr7y/8Tx8voe13qVNa+1HeTf4MzL+AP3GJTh79lGgyShIR1m8VGkmoAcK8j9Bf5z77hsTkAnMBwYjq9jq5clngeRfv2ZRyBs16/ckyGrj4FVKqSi7YS3OCK/ieXbw8N8wh6a7lAsc1UbobM04P+JBRq6ScQ8FI2IBYc+2u4MoV2UeYA5Vl7f7M3Q27uHGM61zN20+ynScTKp28nrMyioBR9CAW5EVTFFXaM8Buvx6RwpkZf2dmT1Q2mFvShO4r8sBk0l84AaNkLKXd4ihQbanZe9zePfFUjJTG8/YKzLK6wTt53k","b5":"8d56fc55ced7dfeae5da682b850782d67736ae3b","b7":"1714398197968","b6":"96d7b46ed8e5c2814672b92d1190330201bd6c8c"}
```

结果是固定下来了, 但是有个问题 frida call 的 b5 是固定的 761b422627f7872479184ba1d5036b3780ea06ce, 和这里不一样, 说明还存在暗桩, 这个时候别着急去分析 b4 和 b6, 如果 b5 走了异常分支, b4 和 b6 同样有可能走向异常, 所以正确的做法就是分析 b5 的算法, 并找出这个暗桩

### 排除暗桩

我比较喜欢从后往前分析, 这个方法适用大部分人.

先用 ida 的 findcrypt 找找可能有哪些算法

![](https://bbs.kanxue.com/upload/tmp/983110_7U9PTF2NBZ35WFU.png)

单看 b5 长度 40 位来说, 如果没有自定义算法的话好像没有匹配上的, 40 位首选 sha1, 说到 sha1, 应该马上想到前 4 个魔数和 md5 一样, 第 5 个魔数 0xC3D2E1F0 是 sha1 独有的, 利用 unidbg 天然的可下断, 内存检索, 高效的 trace 是做算法还原的首选.

#### trace

打算做两组 trace, 一组从从 init 开始, 另一组从 call fun 开始, 下面这组代码是通用的

```
public void trace(){
    String traceFile = "unidbg-android/src/test/java/com/xx/biji/trace5.txt";
    PrintStream traceStream = null;
    try{
        traceStream = new PrintStream(new FileOutputStream(traceFile), true);
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    }
    emulator.traceCode(module.base,module.base+module.size).setRedirect(traceStream);
}
```

init 开始 trace 的 3430 万行, 耗时 8min,call fun 开始的 23 万行, 耗时 5s, 多复制几份出来 (直接复制文件), 全部扔到 010 里, 多弄几个文件的目的是方便做定位, 一份搜了之后可以搜另一份, 结合着定位更快更准. 做 init trace 的原因是有些数据来自原始的初始化过程, 这个留着备用, 主要是看 call fun 那份.

上面说的 sha1, 先看看有没有 0xC3D2E1F0 这个魔数

![](https://bbs.kanxue.com/upload/tmp/983110_ZN9BUVGATWEDJNY.png)

稳妥点搜一下 k 值 0x5A827999, 一共 4 个 80 轮中每 20 轮用一个

![](https://bbs.kanxue.com/upload/tmp/983110_H43V4RT8PG9782R.png)

有的, 说明 sha1 肯定参与了运算, 而且看左边的热点图, 应该是进行了两次 sha1, 正好 b6 也是 40 位, 说不好 b6 也使用了 sha1.

```
def sha1(data):
    def left_rotate(n, b):
        return ((n << b) | (n >> (32 - b))) & 0xFFFFFFFF
    # 初始化哈希值
    h0 = 0x67452301
    h1 = 0xEFCDAB89
    h2 = 0x98BADCFE
    h3 = 0x10325476
    h4 = 0xC3D2E1F0
 
    # 预处理
    original_byte_len = len(data)
    original_bit_len = original_byte_len * 8
    data += b'\x80'
 
    while (len(data) % 64) != 56:
        data += b'\x00'
 
    data += original_bit_len.to_bytes(8, 'big')  # 附加消息长度 大端序
 
    # 处理每个512-bit块
    for i in range(0, len(data), 64):
        w = [0] * 80
        chunk = data[i:i + 64]
        # 将块划分为16个32-bit字
        for j in range(16):
            w[j] = int.from_bytes(chunk[4 * j:4 * j + 4], 'big')
 
        # 扩展到80个字
        for j in range(16, 80):
            w[j] = left_rotate(w[j - 3] ^ w[j - 8] ^ w[j - 14] ^ w[j - 16], 1)
 
        # 初始化hash值
        a = h0
        b = h1
        c = h2
        d = h3
        e = h4
 
        # 主循环
        for j in range(80):
            if 0 <= j <= 19:
                f = (b & c) | (~b & d)
                k = 0x5A827999
            elif 20 <= j <= 39:
                f = b ^ c ^ d
                k = 0x6ED9EBA1
            elif 40 <= j <= 59:
                f = (b & c) | (b & d) | (c & d)
                k = 0x8F1BBCDC
            elif 60 <= j <= 79:
                f = b ^ c ^ d
                k = 0xCA62C1D6
 
            temp = (left_rotate(a, 5) + f + e + k + w[j]) & 0xFFFFFFFF
            e = d
            d = c
            c = left_rotate(b, 30)
            b = a
            a = temp
 
        # 增加到当前的hash值
        h0 = (h0 + a) & 0xFFFFFFFF
        h1 = (h1 + b) & 0xFFFFFFFF
        h2 = (h2 + c) & 0xFFFFFFFF
        h3 = (h3 + d) & 0xFFFFFFFF
        h4 = (h4 + e) & 0xFFFFFFFF
 
    # 生成最终的哈希值
    return ''.join(f'{x:08x}' for x in [h0, h1, h2, h3, h4])
 
 
message = "yangruhua"
hash_value = sha1(message.encode())
print(hash_value)
```

上面是一份标准的 sha1 python 版

下面我演示一下怎么实现开头说的 "要求熟悉算法细节, 能根据算法特征直接定位明文, 还原出正确的密文", 此为扩展篇, 不看这里的不影响后面的. 我得先解释一下上面的代码, 不然下面你可能听不懂.

第一步: 明文预处理

```
# 预处理
    original_byte_len = len(data)
    original_bit_len = original_byte_len * 8
    data += b'\x80'
 
    while (len(data) % 64) != 56:
        data += b'\x00'
 
    data += original_bit_len.to_bytes(8, 'big')  # 附加消息长度 大端序
```

明文 (下面说的都是 16 进制) 不管长度多少, 首先填充 0x80, 接着填充 00, 然后 sha1 的分组长度是 512 位, 64 字节, while (len(data) % 64) != 56: 为什么这里要判断 %64 后要是 56 的倍数? 因为结尾要填充 8 字节的附加消息长度, 注意这里的是大端序, 也就是正常的字节序, 如果是 md5 则是小端序. md5,sha1,sha256 都是 512 分组, sha512 是 1024 分组.

第二步: 扩展 64 块

```
# 扩展到80个字
for j in range(16, 80):
    w[j] = left_rotate(w[j - 3] ^ w[j - 8] ^ w[j - 14] ^ w[j - 16], 1)
```

前面说的 512 分组, 64 字节, 4 字节一块, 一共 16 块, sha1 80 轮运算需要 80 个块, 所以这里有 64 个循环, 同时观察到这里有一个循环左移, 你知道为什么这个算法叫 sha1 吗? 如果你了解 sha0 算法的话, 这个 sha1 就是在 sha0 的基础上多了一个循环左移. NSA 1993 年发布, 两年后 SHA-0 被撤回，因为发现了该算法存在安全漏洞，可能使得生成哈希碰撞, SHA-1 对 SHA-0 进行了一个小的修改，主要是通过增加循环左移操作来增强算法的安全性，从而使得寻找哈希碰撞变得更困难. 这块也很容易理解.

第三步: 核心 80 轮

```
for j in range(80):
    if 0 <= j <= 19:
        f = (b & c) | (~b & d)
        k = 0x5A827999
    elif 20 <= j <= 39:
        f = b ^ c ^ d
        k = 0x6ED9EBA1
    elif 40 <= j <= 59:
        f = (b & c) | (b & d) | (c & d)
        k = 0x8F1BBCDC
    elif 60 <= j <= 79:
        f = b ^ c ^ d
        k = 0xCA62C1D6
 
    temp = (left_rotate(a, 5) + f + e + k + w[j]) & 0xFFFFFFFF
    e = d
    d = c
    c = left_rotate(b, 30)
    b = a
    a = temp
 
# 增加到当前的hash值
h0 = (h0 + a) & 0xFFFFFFFF
h1 = (h1 + b) & 0xFFFFFFFF
h2 = (h2 + c) & 0xFFFFFFFF
h3 = (h3 + d) & 0xFFFFFFFF
h4 = (h4 + e) & 0xFFFFFFFF
```

核心的 80 轮运算, 一共 4 个 k, 每 20 轮用一个, 如果实现开头说的 "要求熟悉算法细节, 能根据算法特征直接定位明文, 还原出正确的密文", 只需要看明文可能在哪些地方参与了运算, 这里说的主要是和特殊值的运算, 比如 k, 体现在这一行 temp = (left_rotate(a, 5) + f + e + k + w[j]) & 0xFFFFFFFF, 看上去是与 k 相加的是明文, 但不能想的那么简单如果 so 里是 w[j]+left_rotate(a, 5)+f+e+k, 这样的话与 k 相加的就不是明文了, 所以需要 so 里的代码辅助分析, 事实上很多算法可操作性很大, 明明是一个标准的, 就是让你找不到 key. 比如后面说的 aes.

我们来 010 里搜一下 0x5A827999, 原始明文与他进行计算

![](https://bbs.kanxue.com/upload/tmp/983110_YKC4H6QFMMVUPQM.png)

先看第一轮 15a64,ida 中看一眼

![](https://bbs.kanxue.com/upload/tmp/983110_5XMAK846NWN5E4P.png)

```
v15 = v10 + v666 + (v7 | v5) + 0x5A827999; 

v10 = v6 + (v4 >> 27);  v666 = a1[4];
替代一下 
v15 = v6 + (v4 >> 27) + a1[4] + (v7 | v5) + 0x5A827999;   // v6 = bswap32(*a2);
left_rotate(a, 5) + f + e + k + w[j]  // f = (b & c) | (~b & d)  e=最开始的魔数[4]
```

到这里你能理清这些对应关系吗?

v4 对应 a,a1[4] 对应 e,(v7 | v5) 对应 f,0x5A827999 对应 k,v6 对应 w[j], 这些都是伪代码, 才会看起来很奇怪, 看多了就觉得正常了, 或者从汇编层面直接看寄存器也是一样的.

所以 a1(参数 0) 就是 context(拿 c 语言中的形容, 我也不知道叫啥), 可以理解为新的魔数. a2 就是参数 2(明文),v15 = v6 + (v4 >> 27) + a1[4] + (v7 | v5) + 0x5A827999; 所以这里与 k 相加的肯定不是明文, 可以在 unidbg 中下断看下, 外层函数的地址是 0x15974

![](https://bbs.kanxue.com/upload/tmp/983110_QW3BQDVWVHMZDVC.png)

验证确实参数 0 是魔数, 注意是小端序

![](https://bbs.kanxue.com/upload/tmp/983110_RGCEVN4SZARTVG3.png)

参数 1 确实是明文

先验证下是否是标准 sha1, 结果就是下一次的魔数

```
明文
0000: 50 4F 53 54 20 2F 63 6C 69 65 6E 74 2E 61 63 74    POST /client.act
0010: 69 6F 6E 20 61 76 69 66 53 75 70 70 6F 72 74 3D    ion avifSupport=
0020: 31 26 62 65 66 3D 31 26 62 75 69 6C 64 3D 39 39    1&bef=1&build=99
0030: 32 30 38 26 63 6C 69 65 6E 74 3D 61 6E 64 72 6F    208&client=andro
unidbg中的结果
0000: A0 93 31 E3 9B 7E ED A8 FE BA CA 89 5E 36 46 82    ..1..~......^6F.
0010: 79 EB B5 05
```

![](https://bbs.kanxue.com/upload/tmp/983110_6UT22FE442VKSJY.png)

不要直接把明文拿到 cyberchef 中 sha1, 那里面默认是有填充的, 这里只是 sha1 的一个分组, 为什么明文理论上可以无限长, 就是这里每一分组的结果都作为下一分组的魔数, 验证的话也很简单, 只要把填充部分注释掉

![](https://bbs.kanxue.com/upload/tmp/983110_KC39ZH55NG246CV.png)

明文输入 POST /client.action avifSupport=1&bef=1&build=99208&client=andro, 结果 e33193a0a8ed7e9b89cabafe8246365e05b5eb79, 字节序反转下就是了, 所以没有魔改, 是标准的.

![](https://bbs.kanxue.com/upload/tmp/983110_T9NBA4ZHGHXSCUF.png)

对比下可知, 与 k 值相加的确实不是明文., 如果一开始直接看这个函数的两个入参也能猜出来, 为什么要介绍这些细节呢? 这里我只是想说明熟悉算法的作用性很大, 后面不会再那么介绍了.

如果这个函数走了很多遍, 一遍遍断下来看内存不是很麻烦.

是的, unidbg 提供了更简单的方法

![](https://bbs.kanxue.com/upload/tmp/983110_NKT4HYUSD42ENSY.png)

这里有一个上面说的花, 所以其实 15974 和 159ac 是同一块地址, 你 hook 哪个都是一样的效果

```
Debugger debugger = emulator.attach();
debugger.addBreakPoint(module.base + 0x159ac, new BreakPointCallback() {
RegisterContext context = emulator.getContext();  // sha1
int num = 0;
@Override
public boolean onHit(Emulator emulator, long address) {
    num+=1;
    UnidbgPointer src = context.getPointerArg(0);
    Inspector.inspect(src.getByteArray(0,0x20),"0x159ac onEnter arg0 "+num);
    UnidbgPointer src2 = context.getPointerArg(1);
    Inspector.inspect(src2.getByteArray(0,0x40),"0x159ac onEnter arg1 "+num);
    debugger.addBreakPoint(context.getLRPointer().peer, new BreakPointCallback() {
    @Override
    public boolean onHit(Emulator emulator, long address) {
        Inspector.inspect(src.getByteArray(0,0x20),"0x159ac onLeave arg0 "+num);
        return true;
    }
});
    return true;
}
```

使用 Inspector 的 inspect 的可以直接监控一块内存的数据, 后面的参数是一个 tag, 利用 context 对象可以直接获取到当前位置的寄存器

![](https://bbs.kanxue.com/upload/tmp/983110_JDS934E6TJBZTQ9.png)

最后一组就是 b6 96d7b46ed8e5c2814672b92d1190330201bd6c8c, 内存中的是小端序

```
6E B4 D7 96 81 C2 E5 D8 2D B9 72 46 02 33 90 11
8C 6C BD 01
```

往上搜 b5

![](https://bbs.kanxue.com/upload/tmp/983110_4H7QVGBPYGARE8X.png)

b5 由 uri 拼接 0x20 字节神秘数据后 sha1 得到

![](https://bbs.kanxue.com/upload/tmp/983110_5EXPX9EUGM8SPFX.png)

```
{"b1":"6bbc8976-d0fe-44ef-9776-ab9da0c127e8","b2":"3.2.8.4_0","b3":"2.1","b4":"Y+0jkt2FIdCIV7z0mmcidio1XB/pDmDb6Wq9lXKhNiJvGwwHAPoKQJ8EJhQEF0Ssf1M9komvy2ufFocI2HvAcoq93b3n1kyR1xQyPFFteqltXADYy/z8oUryNr7y/8Tx8voe13qVNa+1HeTf4MzL+AP3GJTh79lGgyShIR1m8VGkmoAcK8j9Bf5z77hsTkAnMBwYjq9jq5clngeRfv2ZRyBs16/ckyGrj4FVKqSi7YS3OCK/ieXbw8N8wh6a7lAsc1UbobM04P+JBRq6ScQ8FI2IBYc+2u4MoV2UeYA5Vl7f7M3Q27uHGM61zN20+ynScTKp28nrMyioBR9CAW5EVTFFXaM8Buvx6RwpkZf2dmT1Q2mFvShO4r8sBk0l84AaNkLKXd4ihQbanZe9zePfFUjJTG8/YKzLK6wTt53k","b5":"8d56fc55ced7dfeae5da682b850782d67736ae3b","b7":"1714398197968"}
```

b6 由除去 b6 的数据拼接 0x20 字节的神秘数据 sha1 得到

由此可以猜测 b5 和 b6 应该是同一套算法, 只是最开始的明文不一样. 事实也确实如此, 所以后面不再介绍 b6

但注意这个是走向异常分支的结果, 所以只能说有一定参考价值. frida call 和 hook 区别就在于最后面 0x20 字节

正确的应该是

```
2D 75 86 0E 3C 71 84 27 7E 2D B6 B2 A5 AA C7 45
71 74 2E BB 94 89 6C A1 ED 0A 0A 5A 15 1C 84 79
```

所以需要向上层寻找异常点, 最开始的做法是跟踪这 0x20 字节的生成, 就是利用 emulator.traceWrite 和下断跟踪, 最终发现这个是在 142F4 函数生成的, 一共 3 个参数, 第三个不变如下

```
0000: 6D AA 32 E2 4B 3A 1E FD 2E D2 20 DF 9F 3E DD 76    m.2.K:.... ..>.v
0010: 55 71 80 22 1E 4B 9E DF 30 99 BE 00 AF A7 63 76    Uq.".K..0.....cv
0020: 6D 08 DC DB 73 43 42 04 43 DA FC 04 EC 7D 9F 72    m...sCB.C....}.r
0030: 2D C6 23 04 5E 85 61 00 1D 5F 9D 04 F1 22 02 76    -.#.^.a.._...".v
0040: 15 67 B0 7B 4B E2 D1 7B 56 BD 4C 7F A7 9F 4E 09    .g.{K..{V.L...N.
0050: 14 3B 6B 44 5F D9 BA 3F 09 64 F6 40 AE FB B8 49    .;kD_..?.d.@...I
0060: 2F DF 64 08 70 06 DE 37 79 62 28 77 D7 99 90 3E    /.d.p..7yb(w...>
0070: 9D D1 8A 28 ED D7 54 1F 94 B5 7C 68 43 2C EC 56    ...(..T...|hC,.V
0080: 2C CB FB 66 C1 1C AF 79 55 A9 D3 11 16 85 3F 47    ,..f...yU.....?G
0090: 8C 8C 6C 08 4D 90 C3 71 18 39 10 60 0E BC 2F 27    ..l.M..q.9.`../'
00A0: 40 27 09 2B 0D B7 CA 5A 15 8E DA 3A 1B 32 F5 1D    @'.+...Z...:.2..
第二个参数用来存结果的,最终结果就是上图中的
0010: 5C 97 10 A8 75 A9 55 E2 55 E6 36 9A 46 87 39 66    \...u.U.U.6.F.9f
0020: F3 5F F1 C2 6E 5B D0 51 1D 1B 3E 03 74 E5 36 E7    ._..n[.Q..>.t.6.
第一个参数的两次入参
0000: F6 4F F2 16 6B E6 A2 1D CE 17 ED 14 EE 7B 3B 93    .O..k........{;.
0010: 30 31 30 32 30 33 30 34 30 35 30 36 30 37 30 38    0102030405060708
 
0000: 4C 87 00 B8 65 B9 45 F2 45 F6 26 8A 56 97 29 76    L...e.E.E.&.V.)v
0010: 5C 97 10 A8 75 A9 55 E2 55 E6 36 9A 46 87 39 66    \...u.U.U.6.F.9f
```

ida 中已经识别出了 aes 字眼

![](https://bbs.kanxue.com/upload/tmp/983110_F2A7J739CDTB8M5.png)

入参 3 16*11 怀疑是扩展的秘钥, 可以用第一组秘钥扩展试试, 以下代码也是龙哥提供的

```
Sbox = (
    0x63, 0x7C, 0x77, 0x7B, 0xF2, 0x6B, 0x6F, 0xC5, 0x30, 0x01, 0x67, 0x2B, 0xFE, 0xD7, 0xAB, 0x76,
    0xCA, 0x82, 0xC9, 0x7D, 0xFA, 0x59, 0x47, 0xF0, 0xAD, 0xD4, 0xA2, 0xAF, 0x9C, 0xA4, 0x72, 0xC0,
    0xB7, 0xFD, 0x93, 0x26, 0x36, 0x3F, 0xF7, 0xCC, 0x34, 0xA5, 0xE5, 0xF1, 0x71, 0xD8, 0x31, 0x15,
    0x04, 0xC7, 0x23, 0xC3, 0x18, 0x96, 0x05, 0x9A, 0x07, 0x12, 0x80, 0xE2, 0xEB, 0x27, 0xB2, 0x75,
    0x09, 0x83, 0x2C, 0x1A, 0x1B, 0x6E, 0x5A, 0xA0, 0x52, 0x3B, 0xD6, 0xB3, 0x29, 0xE3, 0x2F, 0x84,
    0x53, 0xD1, 0x00, 0xED, 0x20, 0xFC, 0xB1, 0x5B, 0x6A, 0xCB, 0xBE, 0x39, 0x4A, 0x4C, 0x58, 0xCF,
    0xD0, 0xEF, 0xAA, 0xFB, 0x43, 0x4D, 0x33, 0x85, 0x45, 0xF9, 0x02, 0x7F, 0x50, 0x3C, 0x9F, 0xA8,
    0x51, 0xA3, 0x40, 0x8F, 0x92, 0x9D, 0x38, 0xF5, 0xBC, 0xB6, 0xDA, 0x21, 0x10, 0xFF, 0xF3, 0xD2,
    0xCD, 0x0C, 0x13, 0xEC, 0x5F, 0x97, 0x44, 0x17, 0xC4, 0xA7, 0x7E, 0x3D, 0x64, 0x5D, 0x19, 0x73,
    0x60, 0x81, 0x4F, 0xDC, 0x22, 0x2A, 0x90, 0x88, 0x46, 0xEE, 0xB8, 0x14, 0xDE, 0x5E, 0x0B, 0xDB,
    0xE0, 0x32, 0x3A, 0x0A, 0x49, 0x06, 0x24, 0x5C, 0xC2, 0xD3, 0xAC, 0x62, 0x91, 0x95, 0xE4, 0x79,
    0xE7, 0xC8, 0x37, 0x6D, 0x8D, 0xD5, 0x4E, 0xA9, 0x6C, 0x56, 0xF4, 0xEA, 0x65, 0x7A, 0xAE, 0x08,
    0xBA, 0x78, 0x25, 0x2E, 0x1C, 0xA6, 0xB4, 0xC6, 0xE8, 0xDD, 0x74, 0x1F, 0x4B, 0xBD, 0x8B, 0x8A,
    0x70, 0x3E, 0xB5, 0x66, 0x48, 0x03, 0xF6, 0x0E, 0x61, 0x35, 0x57, 0xB9, 0x86, 0xC1, 0x1D, 0x9E,
    0xE1, 0xF8, 0x98, 0x11, 0x69, 0xD9, 0x8E, 0x94, 0x9B, 0x1E, 0x87, 0xE9, 0xCE, 0x55, 0x28, 0xDF,
    0x8C, 0xA1, 0x89, 0x0D, 0xBF, 0xE6, 0x42, 0x68, 0x41, 0x99, 0x2D, 0x0F, 0xB0, 0x54, 0xBB, 0x16,
)

Rcon = (0x00, 0x01, 0x02, 0x04, 0x08, 0x10, 0x20, 0x40, 0x80, 0x1B, 0x36)


def text2matrix(text):
    matrix = []
    for i in range(16):
        byte = (text >> (8 * (15 - i))) & 0xFF
        if i % 4 == 0:
            matrix.append([byte])
        else:
            matrix[i // 4].append(byte)
    return matrix


def shiftRound(array, num):
    '''

    :param array: 需要循环左移的数组
    :param num: 循环左移的位数
    :return: 使用Python切片，返回循环左移num个单位的array
    '''
    return array[num:] + array[:num]


def g(array, index):
    '''
    g 函数
    :param array: 待处理的四字节数组
    :index:从1-10，每次使用Rcon中不同的数
    '''
    # 首先循环左移1位
    array = shiftRound(array, 1)
    # 字节替换
    array = [Sbox[i] for i in array]
    # 首字节和rcon中对应元素异或
    array = [(Rcon[index] ^ array[0])] + array[1:]
    return array


def xorTwoArray(array1, array2):
    '''
    返回两个数组逐元素异或的新数组
    :param array1: 一个array
    :param array2: 另一个array
    :return:
    '''
    assert len(array1) == len(array2)
    return [array1[i] ^ array2[i] for i in range(len(array1))]


def showRoundKeys(kList):
    for i in range(len(kList)):
        print("K%02d:" %i +"".join("%02x" % k for k in kList[i]))


def keyExpand(key):
    master_key = text2matrix(key)
    round_keys = [[0] * 4 for i in range(44)]
    # 规则一(图中红色部分)
    for i in range(4):
        round_keys[i] = master_key[i]
    for i in range(4, 4 * 11):
        # 规则二(图中红色部分)
        if i % 4 == 0:
            round_keys[i] = xorTwoArray(g(round_keys[i - 1], i // 4), round_keys[i - 4])
        # 规则三(图中橙色部分)
        else:
            round_keys[i] = xorTwoArray(round_keys[i - 1], round_keys[i - 4])
    # 将轮密钥从44*4转成11*16,方便后面在明文的运算里使用
    kList = [[] for i in range(11)]
    for i in range(len(round_keys)):
        kList[i//4] += round_keys[i]

    showRoundKeys(kList)
    return kList
key = 0x6DAA32E24B3A1EFD2ED220DF9F3EDD76
kList = keyExpand(key)
```

![](https://bbs.kanxue.com/upload/tmp/983110_4T78NADDDUY7ZGX.png)

扩展完竟然对不上, 莫非是魔改了秘钥扩展? 当时我也不知道怎么办, 研究了下两组参数

```
第一个参数的两次入参
0000: F6 4F F2 16 6B E6 A2 1D CE 17 ED 14 EE 7B 3B 93    .O..k........{;.
0010: 30 31 30 32 30 33 30 34 30 35 30 36 30 37 30 38    0102030405060708
 
0000: 4C 87 00 B8 65 B9 45 F2 45 F6 26 8A 56 97 29 76    L...e.E.E.&.V.)v
0010: 5C 97 10 A8 75 A9 55 E2 55 E6 36 9A 46 87 39 66    \...u.U.U.6.F.9f
```

第二组有没有感觉很奇怪, 为什么后面的数字前后都是一样的. 往加密模式和填充方式那块想, 如果是 cbc 然后填充都是 10 是不是有可能这种情况, 而且 5C 97 10 A8 75 A9 55 E2 55 E6 36 9A 46 87 39 66 这个就是第一组的密文, 填充 16 个 0x10 再异或第一组的密文是不是有可能是 4C 87 00 B8 65 B9 45 F2 45 F6 26 8A 56 97 29 76, 验证下是正确的, 那按理 30 31 30 32 30 33 30 34 30 35 30 36 30 37 30 38 就是最开始的 iv,F6 4F F2 16 6B E6 A2 1D CE 17 ED 14 EE 7B 3B 93 明文和 iv 异或的结果, 所以初始明文是 c67ec2245bd59229fe22dd22de4c0bab

还差一个 key, 既然他可能魔改秘钥扩展那直接用扩展的 11 轮秘钥加密, 结果不正确, 然后就想着 dfa 出来 key, 但是这个查表用的很独特

![](https://bbs.kanxue.com/upload/tmp/983110_C6QKQP8QD4PAERZ.png)

这是标准的 10 轮, 每一轮的某个方法都是这 16 个字节一起处理的, 这里的每 4 字节一组, 这 4 组都是分开处理的, 不在连续的地址上, 感觉根本 dfa 不了, 想了挺久的, 最后发现这个 6D AA 32 E2 4B 3A 1E FD 2E D2 20 DF 9F 3E DD 76 秘钥按小端序来扩展竟然对得上, 所以真正的秘钥是 0xE232AA6DFD1E3A4BDF20D22E76DD3E9F, 拿到 cyberchef 里加密下

![](https://bbs.kanxue.com/upload/tmp/983110_CAJDQS7RA4RHUBT.png)

还真是这个结果, 但是我用 frida hook 这个 aes 的地址发现根本就不走这个位置, 白忙活了, 这还不是最气人的, 最后把异常分支打通, 然后跟踪这组数据是在 vmp 里面. 这; 流程图看上去好像很小, 但其实分支很多, 也不是 ollvm, 待会会说怎么区别 ollvm 和 vmp, 气人的是我以为这会是 ollvm 的 aes, 一直在里面找 aes 的特征, 结果这是个 sha256![](https://bbs.kanxue.com/upload/tmp/983110_7UZ4RFPFGJTNERG.png)

![](https://bbs.kanxue.com/upload/tmp/983110_5RF2CW5K3S74U76.png)

#### 回溯排查异常

前面说了 frida 不走这, 所以这个分支也是错误的, 要往上回溯

在刚开始 aes 断下的地方按 bt back trace

![](https://bbs.kanxue.com/upload/tmp/983110_4MU76647TQM2XAF.png)

frida 去 hook 直到相邻两个地址一个走, 一共不走为止, 问题就出在这发现 215c0 往下都是走的, 22b00 往上都不走, 说明出在 215c0 的内部, 它不应该走 22b00 分支

![](https://bbs.kanxue.com/upload/tmp/983110_53RS8VG9ABAW4U3.png)

用流程视图看更直观, 往上找, 最终发现是在 20144 的返回值 hook 的和 unidbg 中的不一样

![](https://bbs.kanxue.com/upload/tmp/983110_26E44BGWCQX9WH9.png)

```
var soAddr = Module.findBaseAddress("libjdg.so");
var funcAddr = soAddr.add(0x20144)  //32位的话记得+1
 
Interceptor.attach(funcAddr,{
            onEnter: function(args){
                console.log('onEnter arg[]: ',args[0])
                this.arg0 = args[0]
            },
            onLeave: function(retval){
                console.log('onLeave result: ',retval)
 
            }
        });
```

```
onLeave result:  0x0
onEnter arg[]:  0x46ddc60f
onLeave result:  0x0
onEnter arg[]:  0x36fbbfa1
onLeave result:  0x0
onEnter arg[]:  0x665c9199
onLeave result:  0x0
onEnter arg[]:  0xa203ef4
onLeave result:  0x0
onEnter arg[]:  0xfb95b21
onLeave result:  0x0
```

真机返回的全 0,unidbg 部分会返回 1, 这个地方引用很多, 就改 aes 引用的一处会走向另一个分支, 改成全 0 后结果后 b5 的结果和真机一致, b4 不一样有可能就是真机的随机和 unidbg 的随机数实现不同, 肯定是做不到随机相同的, 而且不同手机也无法做到.

排除的过程比较耗时, hook 上 b5 就正常了

```
public void patch2() {
        Debugger debugger = emulator.attach();
        debugger.addBreakPoint(module.base + 0x20144, new BreakPointCallback() {
            RegisterContext context = emulator.getContext();
            @Override
            public boolean onHit(Emulator emulator, long address) {
                debugger.addBreakPoint(context.getLRPointer().peer, new BreakPointCallback() {
                    @Override
                    public boolean onHit(Emulator emulator, long address) {
                        emulator.getBackend().reg_write(Unicorn.UC_ARM64_REG_X0,0);
                        return true;
                    }
                });
                return true;
            }
        });
    }
```

这个异常怎么引起的不是很明白, 因为有时候返回 0 有时候返回 1, 当时还认为是不是低版本不走 vmp, 高版本走 vmp, 因为 unidbg 是安卓 6, 真机是安卓 10, 没有安卓 6 的手机所以用的模拟器, 模拟器的安卓 6 call 的和真机一样, 怀疑是一个很隐秘的 anti unidbg, 大概是一些系统库和真机不同导致的. 也可能是安全人员留下的备用方案, vmp 走不通的话走这条, 当时测的搜索接口这个也是能用的. 不过最好和真机保持一致. 这种备用方案很正常, 比如某音 quic 协议都有 https 的降级通道.

patch 后正常了, 和真机一致

![](https://bbs.kanxue.com/upload/tmp/983110_DGRT6Z69MHYX9QY.png)

重新 trace 一份, 从 call fun 开始, 这次 240 万行, 耗时 25s, 不 patch 是 20 多万行, 5s, 对比可知 patch 后的算法明显复杂的多. 某音的从 call fun 开始大概 1300 万行 vmp, 对比某音还是弱了很多.

### 分析 b5

```
要分析的数据
0010: 2D 75 86 0E 3C 71 84 27 7E 2D B6 B2 A5 AA C7 45    -u..
```

b6 同 b5, 这里不介绍了, 上面分析到只是最后 0x20 字节不一样, 前面说了这是一个 vmp 版的 sha256, 接下来按照我最开始的思路来续写.

上面说了这 0x20 字节不走 aes, 不确定它是什么算法的情况下, 直接 trace write + 下断跟踪 (循环几次就能找到最开始赋值的位置)

最终赋值的位置在 24918

![](https://bbs.kanxue.com/upload/tmp/983110_FPFSVYCKJFKV38Y.png)

![](https://bbs.kanxue.com/upload/tmp/983110_74443Z766669GPP.png)

![](https://bbs.kanxue.com/upload/tmp/983110_GCAP8NZ6T2VRVUR.png)

![](https://bbs.kanxue.com/upload/tmp/983110_EN35SZBSYXTG5G6.png)

可以发现都是一些左移右移, 模运算, 取反, 异或, 写虚拟机需要把这些基础的运算实现.

#### vmp 对比 ollvm

ollvm 与 vmp 有什么区别?

![](https://bbs.kanxue.com/upload/tmp/983110_VS7TDKEJZ4Y8CWA.png)

这是最开始的 call 的目标函数 0x29CE4

![](https://bbs.kanxue.com/upload/tmp/983110_E5DHQAHXXXJMTXD.png)

截了部分, 对比可知, ollvm 只是混淆的厉害, 但基础的逻辑还是可以看到的, 比如函数调用, jni,vmp 是只有基本的运算指令.

这是某音的 metasec_ml 的 vmp

![](https://bbs.kanxue.com/upload/tmp/983110_BGSSV9EUV7WC227.png)

![](https://bbs.kanxue.com/upload/tmp/983110_Q5YQWC8BJCBUPN4.png)

伪代码也是只有基础运算, Helios 和 Ladon Argus Medusa 这 4 个都在 vmp 里, Gorgon 不在, Gorgon 简单的 trace write 就可以解决, 新版的还多了一个参数 soter,7 神了

![](https://bbs.kanxue.com/upload/tmp/983110_MPQQ3MU6EWR7MAP.png)

新版 Ladon 和 Argus 很短, 即将弃用, ladon 和 Helios 同一个算法, 所以没必要留两个, Argus 收集的指纹远没有 Medusa 多, 弃用也正常, 至于最新的 soter, 还没来得及研究, 后面会试试

![](https://bbs.kanxue.com/upload/tmp/983110_XNK84UF87D4FAKS.png)

这是我朋友修的 metasec_ml, 对比上面我修的好看多了! 不过最终 trace 都一样.

#### 如何应对 vmp

两种方法

第一种, 还原虚拟机指令, 这块难度很大, 综合性很强, 需要有多年的逆向 vmp 经验, 最好是有 win vmp 的经验, 因为安卓的 vmp 很多是借鉴的 pc 的. 这块我也正在学.

第二种, 就是这篇文章的主题, trace 还原, 主要体现是在 010 里搜关键常量, 密文, 从后往前推, 根据对应关系直接还原算法, 这提供了一个弯道超车的机会, 但是技巧性很高, 而且很费精力, 想象一下, 一整天就盯着一堆汇编, 十六进制看, 是啥感受, 而且复杂点的样本要看几个星期.

![](https://bbs.kanxue.com/upload/tmp/983110_4MEFJZEGNTFGCW2.png)

#### trace 还原 b5

010 里搜 0x40024918, 最后赋值的位置

```
"eor w9, w10, w9" w10=0x61 w9=0x4c => w9=0x2d
"eor w9, w10, w9" w10=0xe5 w9=0x90 => w9=0x75
"eor w9, w10, w9" w10=0xfc w9=0x7a => w9=0x86
"eor w9, w10, w9" w10=0x18 w9=0x16 => w9=0xe
"eor w9, w10, w9" w10=0x27 w9=0x1b => w9=0x3c
"eor w9, w10, w9" w10=0xb8 w9=0xc9 => w9=0x71
"eor w9, w10, w9" w10=0x32 w9=0xb6 => w9=0x84
"eor w9, w10, w9" w10=0x4e w9=0x69 => w9=0x27
"eor w9, w10, w9" w10=0x4 w9=0x7a => w9=0x7e
"eor w9, w10, w9" w10=0x48 w9=0x65 => w9=0x2d
"eor w9, w10, w9" w10=0x32 w9=0x84 => w9=0xb6
"eor w9, w10, w9" w10=0xc1 w9=0x73 => w9=0xb2
"eor w9, w10, w9" w10=0x27 w9=0x82 => w9=0xa5
"eor w9, w10, w9" w10=0x69 w9=0xc3 => w9=0xaa
"eor w9, w10, w9" w10=0xc3 w9=0x4 => w9=0xc7
"eor w9, w10, w9" w10=0x54 w9=0x11 => w9=0x45
"eor w9, w10, w9" w10=0xe5 w9=0x94 => w9=0x71
"eor w9, w10, w9" w10=0xdf w9=0xab => w9=0x74
"eor w9, w10, w9" w10=0xc w9=0x22 => w9=0x2e
"eor w9, w10, w9" w10=0x60 w9=0xdb => w9=0xbb
"eor w9, w10, w9" w10=0xf8 w9=0x6c => w9=0x94
"eor w9, w10, w9" w10=0x64 w9=0xed => w9=0x89
"eor w9, w10, w9" w10=0xf1 w9=0x9d => w9=0x6c
"eor w9, w10, w9" w10=0x21 w9=0x80 => w9=0xa1
"eor w9, w10, w9" w10=0x1c w9=0xf1 => w9=0xed
"eor w9, w10, w9" w10=0x78 w9=0x72 => w9=0xa
"eor w9, w10, w9" w10=0xca w9=0xc0 => w9=0xa
"eor w9, w10, w9" w10=0x29 w9=0x73 => w9=0x5a
"eor w9, w10, w9" w10=0xfe w9=0xeb => w9=0x15
"eor w9, w10, w9" w10=0x5b w9=0x47 => w9=0x1c
"eor w9, w10, w9" w10=0xce w9=0x4a => w9=0x84   
"eor w9, w10, w9" w10=0x w9=0x => w9=0x79    // 最后一组不给,因为含秘钥,测试过不同版本秘钥一致
```

![](https://bbs.kanxue.com/upload/tmp/983110_UXSX6XJZ7N4J9GG.png)

往下滑 发现 b7 的结果也与这些数据异或, 说明 w9 很有可能是固定的, 暂时理解固定的, 只是这里是固定的, 某音的就不会这样, 他是这样操作的至少嵌套几层才会有秘钥的出现. 所以重点关注 w10 的来源, 直接搜 "eor w9, w10, w9" w10=0x61 w9=0x4c => w9=0x2d 这里的 0x61, 往上找最有可能的 (需要经验), 因为会有很多

![](https://bbs.kanxue.com/upload/tmp/983110_5J3BWJPFJ6QTKUS.png)

下面的全是经验了, 有经验快些, 最好找经过运算得到 0x61 的, 而不是找 ldr(load 加载) str(stor 存) 这种, 因为 c 语言传参取参需要通过地址来实现, 所以会有很多这种指令, 而且很多时候这些不是最开始运算的地方, 所以一句话, 还是经验.

![](https://bbs.kanxue.com/upload/tmp/983110_RCD4HX43ZZ8VDXP.png)

最后发现其实是 0x161,w9 也是秘钥, 这次是加运算, 需要多个文件联合定位, 核心的就是这两组秘钥, 仅学习交流, 所以秘钥不公开. 这组秘钥不同版本也是定值.

所以需要再次寻找 w10 e9 5b f8 0f, 搜 0xe95bf80f

![](https://bbs.kanxue.com/upload/tmp/983110_NQKBWFQCDN792SH.png)

还是上面说的找运算得到结果的, 别找赋值的, 第一个就是了 w10=0x6a09e667 w9=0x7f5211a8 => w9=0xe95bf80f,0x6a09e667 什么常量? sha256, 前面说的熟悉算法到这就有用了

```
import struct
 
# 常量
K = [
    0x428a2f98, 0x71374491, 0xb5c0fbcf, 0xe9b5dba5, 0x3956c25b, 0x59f111f1, 0x923f82a4, 0xab1c5ed5,
    0xd807aa98, 0x12835b01, 0x243185be, 0x550c7dc3, 0x72be5d74, 0x80deb1fe, 0x9bdc06a7, 0xc19bf174,
    0xe49b69c1, 0xefbe4786, 0x0fc19dc6, 0x240ca1cc, 0x2de92c6f, 0x4a7484aa, 0x5cb0a9dc, 0x76f988da,
    0x983e5152, 0xa831c66d, 0xb00327c8, 0xbf597fc7, 0xc6e00bf3, 0xd5a79147, 0x06ca6351, 0x14292967,
    0x27b70a85, 0x2e1b2138, 0x4d2c6dfc, 0x53380d13, 0x650a7354, 0x766a0abb, 0x81c2c92e, 0x92722c85,
    0xa2bfe8a1, 0xa81a664b, 0xc24b8b70, 0xc76c51a3, 0xd192e819, 0xd6990624, 0xf40e3585, 0x106aa070,
    0x19a4c116, 0x1e376c08, 0x2748774c, 0x34b0bcb5, 0x391c0cb3, 0x4ed8aa4a, 0x5b9cca4f, 0x682e6ff3,
    0x748f82ee, 0x78a5636f, 0x84c87814, 0x8cc70208, 0x90befffa, 0xa4506ceb, 0xbef9a3f7, 0xc67178f2
]
 
# 初始哈希值
H = [
    0x6a09e667, 0xbb67ae85, 0x3c6ef372, 0xa54ff53a,
    0x510e527f, 0x9b05688c, 0x1f83d9ab, 0x5be0cd19
]
 
def right_rotate(value, bits):
    return ((value >> bits) | (value << (32 - bits))) & 0xffffffff
 
def sha256(data):
    # 步骤 1: 填充消息
    original_byte_len = len(data)
    original_bit_len = original_byte_len * 8
    data += b'\x80'
    data += b'\x00' * ((56 - (original_byte_len + 1) % 64) % 64)
    data += struct.pack('>Q', original_bit_len)
 
    # 步骤 2: 解析消息为512-bit块
    blocks = []
    for i in range(0, len(data), 64):
        blocks.append(data[i:i + 64])
 
    # 步骤 3: 初始化工作变量
    hash_pieces = H[:]
 
    # 步骤 4: 处理每一个块
    for block in blocks:
        W = list(struct.unpack('>16L', block)) + [0] * 48
 
        for i in range(16, 64):
            s0 = right_rotate(W[i-15], 7) ^ right_rotate(W[i-15], 18) ^ (W[i-15] >> 3)
            s1 = right_rotate(W[i-2], 17) ^ right_rotate(W[i-2], 19) ^ (W[i-2] >> 10)
            W[i] = (W[i-16] + s0 + W[i-7] + s1) & 0xffffffff
 
        a, b, c, d, e, f, g, h = hash_pieces
 
        for i in range(64):
            S1 = right_rotate(e, 6) ^ right_rotate(e, 11) ^ right_rotate(e, 25)
            ch = (e & f) ^ (~e & g)
            temp1 = (h + S1 + ch + K[i] + W[i]) & 0xffffffff
            S0 = right_rotate(a, 2) ^ right_rotate(a, 13) ^ right_rotate(a, 22)
            maj = (a & b) ^ (a & c) ^ (b & c)
            temp2 = (S0 + maj) & 0xffffffff
 
            h = g
            g = f
            f = e
            e = (d + temp1) & 0xffffffff
            d = c
            c = b
            b = a
            a = (temp1 + temp2) & 0xffffffff
 
        hash_pieces = [(x + y) & 0xffffffff for x, y in zip(hash_pieces, [a, b, c, d, e, f, g, h])]
 
    # 步骤 5: 拼接哈希值
    return ''.join(f'{piece:08x}' for piece in hash_pieces)
 
 
hash_value = sha256('yangruhua'.encode())
print(f'SHA-256: {hash_value}')
```

这是一份 python 版的 sha256, 写的很清晰, 按照我上面说的方法很容易就能找到明文, 找到明文后往上回溯就脱离 vmp 了, 是一个 md5, 剩下的都不难了.

#### 如何反制 trace vmp

这点我觉得某音做的很好

第一: 首先不会把秘钥那么早就暴露出来, 中间会有很多复杂的运算流程才会到最终秘钥出场, 这里至少花几天.

第二: 算法不要只有一个分支, 多弄几组, 根据特殊值走不同的算法, 这样逆向人员很难找规律, 比如 medusa 的 16 组算法

第三: 不让逆向人员 trace 出来, 这就是反制 unidbg 的内容了

### 分析 b4

如果你前面都完整看明白了, b4 就不可能拦住你, 又不在 vmp 里.

需要注意的是会调用外部的一个 compress 压缩算法, 根据压缩特征知道是 zlib

```
gzip 1f 8b 08   H4sI // 格式 16进制 base64
zlib 78 9c      eJw
```

![](https://bbs.kanxue.com/upload/tmp/983110_HBM49N8DAJNW2EC.png)

b4 可以直接解密出明文, vmp 里需要两组秘钥, 除此以外还需要解密 apk 里的资源获取一个 cJson, 这里同样有两组秘钥 a6, 和 a11, 如果不走 vmp 算法, a6 用 a7 替换. b4 用秘钥 a11,b5 b6 用秘钥 a6

```
{
  "a1":0,
  "a10":400,
  "a2":"com.xxx.app.mall",
  "a11":"966479bf34cb1eaxxxxxxxxbfce0421893f29a24", // 解密b4所需要的RC4的key的一部分
  "a3":"E0D1A70367Cxxxxxxxx4678DFD05F84F",  // 解密b4会有这个结果出来
  "a4":"99208",
  "a5":"13.x.0",
  "a6":"iy0yIVKJzyT7f5cAwMXGVpQQ1azrxxxxxxxxxxxh4rlgfCgP4T5RqgN5DZ57yu8Y",
  "a7":"eWnsSes/ypEXkWOvqYWbWYruuFz2xxxxxxxxxxx6UNnXqO/CbpA8VT37cF4ap9pg",
  "a8":1717412177879
}
```

```
最新版本的a6和a11,版本不方便透露,你看看时间关系就知道什么版本了
a11 = '250130793xxxxxxxx8d84931fdfa4163d22cf3b4'
a6 = 'MiOdPNIQ22pOxxxxxxxxxxMx6FqVzvcjWsINvJyZKahqMRH3f1M2PVCfBNv9urfO'
```

感兴趣的可以尝试一下, 适合做大厂的第一个入门案例, 后续会更新 al 系, 某团, 需要些时间, 可以期待一下

写在最后
----

如果你只做自己能力范围之内的事情, 就永远没法进步. 或者你能把一件事做到极致.

可是

![](https://bbs.kanxue.com/upload/tmp/983110_XG6UQA2EY2BZGBJ.png)

![](https://bbs.kanxue.com/upload/tmp/983110_28JS6E86NYG3PXC.png)

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-174.htm)

[#逆向分析](forum-161-1-118.htm) [#协议分析](forum-161-1-120.htm) [#HOOK 注入](forum-161-1-125.htm)
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-273274.htm)

> [原创]Burp Suite Pro 破解

前言
--

感谢 scz、surferxyz、h3110w0r1d-y 等各位老师傅在前面开路，这篇文章只是跟着前面的老师傅脚步走，用了自己的方式（ClassFileTransformer）开刀。

 

基于 burpsuite_pro_v2022.3.9.jar

使用工具
----

*   [cfr](https://github.com/leibnitz27/cfr) - 反编译工具
    
*   [jadx](https://github.com/skylot/jadx) - 反编译 gui 工具, 可以查找引用
    
*   [btrace](https://github.com/btraceio/btrace) - 运行状态跟踪工具
    
*   [arthas](https://arthas.aliyun.com/doc/watch.html) - 用起来比 btrace 更方便的跟踪工具
    
*   IDEA
    

正文
--

这里直接用 h3110w0r1d-y 老师傅的 jar 开看

```
文件名称: E:\xxx\burp_latest\BurpLoaderKeygen.jar
文件大小: 18.1 KB (18,580 字节)
修改时间: 2021年07月23日，10:20:48
MD5: B7E345B5594331516F49A709C04A3E41
SHA1: B9F4E9CA60E7493E459E21019441D115D2F43D98
SHA256: 63BE39F1AEEBEA9B86477ACEBBEFEE469A7562216BA8B253A4BB87B0A7A68566
CRC32: BCF7BD4F
计算时间: 0.00s
```

启动方式：`java -javaagent:BurpLoaderKeygen.jar -noverify -jar burpsuite_pro_v2022.3.9.jar`

### 启动解析

1.  -noverify 这个是跳过 jvm 对 class 文件的验证，后面详细说为什么要跳过
2.  -javaagent 这个是 class 加载之前对 class 进行修改

### 原理分析

关于 javaagent 的详细介绍, 可以看知道创宇的 paper [认识 JavaAgent](https://paper.seebug.org/1099/)

#### 破解原理

上 jadx, 先看看老师傅们都搞了啥.

 

![](https://bbs.pediy.com/upload/attach/202206/956011_C9K7UVPVE7AFEAF.png)

 

这里很明显, 找到 Class 里面包含`751a8be34c1a9ed9633d04be3ba075a7`的 Class 文件进行修改, 变量 p 以及 p2 是负责定位要修改的位置, 找到位置后进行 patch.

 

先照搬进 IDEA 看看到底干了啥

 

![](https://bbs.pediy.com/upload/attach/202206/956011_F3D6FYMCUT3T4BX.png)

 

![](https://bbs.pediy.com/upload/attach/202206/956011_H9UYW9MNHZMG2AC.png)

 

这里可以看到, 修改的地方其实很相近, 那就 dump 整个 class 下来看看情况.

 

dump 的办法

 

![](https://bbs.pediy.com/upload/attach/202206/956011_DP4G2D3NJ53NPNT.png)

 

然后调用下

 

![](https://bbs.pediy.com/upload/attach/202206/956011_Y3HFNY8UVT5X8X4.png)

 

得到两个 class 文件

 

![](https://bbs.pediy.com/upload/attach/202206/956011_KVCEE7KDPUAJK6W.png)

 

还记得之前的启动参数么,-noverify, 由于这里简单粗暴的修改了原 class 的 byte, 导致 jvm 的字节码校验失败, 根据这个道理, 很容易就能找到师傅们干了什么, 直接上图把.

 

用 cfr 反编译后, 直接找 empty

 

![](https://bbs.pediy.com/upload/attach/202206/956011_X7KJDSG8ETYFUPC.png)

 

两个 if 被改成 empty 了, 但是到这里我们还是不知道老师傅这么干的道理, 继续往下走把.

#### 查找调用链

根据之前 cfr 反编译出来的源码, 能看到两个超级混淆的办法, void a 以及 bool b, 知道类名跟方法名了, 直接写个 btrace, 把调用链弄出来.

```
import org.openjdk.btrace.core.annotations.*;
import org.openjdk.btrace.core.BTraceUtils;
 
 
@BTrace(unsafe = true)
public class nls {
 
    @OnMethod(clazz="burp.nls", method="a")
    public static void void_a(Object[] var1, Object var2) {
        BTraceUtils.println("calleded void a");
        BTraceUtils.println("1st arg");
        BTraceUtils.println("arg length: " + var1.length);
        BTraceUtils.printArray(var1);
        byte[] my_var1 = (byte [])var1[0];
        BTraceUtils.println(new String(my_var1));
 
        BTraceUtils.println("2nd arg");
        String var2_str = String.valueOf(var2);
        BTraceUtils.println(var2_str);
        BTraceUtils.println("-----------");
    }
 
 
    @OnMethod(clazz="burp.nls", method="b")
    public static void bool_b(Object[] var1, Object var2) {
        BTraceUtils.println("called boolean b");
        BTraceUtils.println("1st arg");
        BTraceUtils.println("arg length: " + var1.length);
        BTraceUtils.printArray(var1);
 
        BTraceUtils.println("2nd arg");
        String var2_str = String.valueOf(oos);
        BTraceUtils.print(var2_str);
        BTraceUtils.jstack();
        BTraceUtils.println("-----------");
    }
}
```

过滤后, 得到以下输出

```
burp.nls.b(Unknown Source)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
java.base/java.lang.reflect.Method.invoke(Method.java:566)
burp.pev.a(Unknown Source)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
java.base/java.lang.reflect.Method.invoke(Method.java:566)
burp.dcw.b(Unknown Source)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
java.base/java.lang.reflect.Method.invoke(Method.java:566)
burp.ib2.b(Unknown Source)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
java.base/java.lang.reflect.Method.invoke(Method.java:566)
burp.gw4.a(Unknown Source)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
java.base/java.lang.reflect.Method.invoke(Method.java:566)
burp.hcp.b(Unknown Source)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
java.base/java.lang.reflect.Method.invoke(Method.java:566)
burp.ciq.b(Unknown Source)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
java.base/java.lang.reflect.Method.invoke(Method.java:566)
burp.a_q.a(Unknown Source)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
java.base/java.lang.reflect.Method.invoke(Method.java:566)
burp.f09.a(Unknown Source)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
java.base/java.lang.reflect.Method.invoke(Method.java:566)
burp.bh5.a(Unknown Source)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
java.base/java.lang.reflect.Method.invoke(Method.java:566)
burp.fqs.a(Unknown Source)
burp.e20.b(Unknown Source)
burp.etx.b(Unknown Source)
burp.etx.g(Unknown Source)
burp.etx.(Unknown Source)
burp.epi.d(Unknown Source)
burp.StartBurp.main(Unknown Source) 
```

这里能看出, 很多类都是动态加载进去的, 想知道怎么调用还是得看看上一个调用类干了什么.

 

还记得之前的 javaagent 工程里面写了 writeModifyClass 方法不, 把他放到最外层, 然后修改下即可 dump 整个 burp 加载的 class.

 

![](https://bbs.pediy.com/upload/attach/202206/956011_Z2WAP79S4YGSCM2.png)

 

分析后发现...pev.class 其实就是把 nls.class 从 string 数组变成 byte 数组然后解密加载

 

![](https://bbs.pediy.com/upload/attach/202206/956011_JRE3V2BQDH4UWZP.png)

 

无实质性进展.. 尝试硬啃混淆过的验证类

#### 代码分析

首先, nls 里面有两个核心方法, 一个 void a, 另外一个 boolean b

 

![](https://bbs.pediy.com/upload/attach/202206/956011_BA5FW7YS86B6PR5.png)

 

这里看到 b 方法, 直接调用了 a 方法, 其中 var_9 是 new 的 object array, 内容是 var3_3 的 byte array, 整个传入 a 方法第一个参数就是 object [][], 第二个参数就是固定的一串字符串

 

再查找整个 nls.a 调用, 发现整个文件就两次调用, 第二次调用也是差不多

 

![](https://bbs.pediy.com/upload/attach/202206/956011_RCX6BPC7NG7EWAD.png)

 

那就可以认定, nls.a 的第二个参数是固定的字符串, 第一个参数是 object [][] 类型, 具体传入了什么上 arthas 看看传入

#### 获取验证函数的调用信息

由于是 Windows, 写个 bat 放到 arthas 的目录方便 arthas attach 到 burp 的进程

 

这里有个小技巧, 如果 burp 调用太快手速不够, 可以进入主界面后, 用 help->License->Update License 来触发验证类的调用

```
@echo off
 
for /f "tokens=1" %%a in ('jps ^| findstr /rc:"burpsuite_pro_"') do @set BURP_PID=%%a
 
as.bat %BURP_PID% --ignore-tools
```

用老师傅的 jar 先启动, 然后运行 bat 就行

 

这里监控用`watch 类名 方法名 -b -s -x 4`

 

先输入个错误的 license key, 看看 nls 类里面方法 a 跟 b 什么情况

 

![](https://bbs.pediy.com/upload/attach/202206/956011_PE683SUF6E5VPSV.png)

 

这里输入的是 abcd, 所以对着的是 97,98,99,100, 那就知道了其实 nls.a 传入的是字符串的 byte array 再放进 Object[]

 

再尝试_正确_的 License Key, 根据之前代码进行分析得到下面的结论

*   b 方法进入时收到的参数是 Object[][], 第一个数组是 int 的数组负责控制走哪个流程, 第二个是字符串获取到的 Byte 数组; 退出时第一个数组变成了 Object 数组套着的验证信息, 第二数组没变
*   a 方法进入时收到的参数是 Object[][], 第一个参数是字符串获取到的 Byte 数组, 第二个参数是固定的字符串; 退时, 第一个参数变成验证信息, 第二个参数没变

结合注册流程, 可以得知

*   用户输入的 License Key, 先进入 nls.b, 然后由 nls.b 调用 nls.a 进行解密, 解密成功则修改传入的第一个变量作为字符串数组给 nls.b 进行下一步操作
*   nls.a 是解密函数
*   nls.a 会被调用两次, 两次修改的变量长度不一致, 对应 License Key 解密跟 Activation Respond 解密

有了以上信息, 尝试构造 nls.a 的调用

#### 分析解密函数

之前有 dump 过破解跟原版的验证 class, 使用破解后的 class 用 jar 打包

 

`jar -cvf burp.jar burp/nls.class`

 

作为依赖塞进 IDEA 的工程, 这里注意 Run 的配置要加上 jvm 参数 - noverify

 

![](https://bbs.pediy.com/upload/attach/202206/956011_YTCRFKAFJ4448Q6.png)

 

这里能成功解密了, 现在就是这串 License Key 是怎么生成的问题了

 

继续用 jadx 查看 Loader 的注册机部分

 

![](https://bbs.pediy.com/upload/attach/202206/956011_BTBQETERW46RXV8.png)

 

这里很明显了, DES 加密, key 在最顶上是祖传的`burpr0x!`

 

大体流程就是字符串数组然后转成 byte array 用 0 作为分隔, 再进行 DES 加密, 最后 base64

 

那么我们就可以根据这个流程反向写出解密函数

#### 写自己的解密函数

这里没什么好说的, 直接跟着原理, 先 base64 解码, 然后 des 解密成 byte array, 转化成字符串后再用 0 分割成字符串数组

 

这里注意, 解密出来的字符串数组长度会比原生的 nls.a 解密出来的字符串数组长, 需要根据字符串数组长度删减返回的数组长度

 

其中长度是 7 的是 License Key 解密, 长度 10 的是 Activation Respond 解密

 

![](https://bbs.pediy.com/upload/attach/202206/956011_6UNDBCR7CWNFBD7.png)

 

到这里就差不多了, 可以开始写自己的 javaagent 了

### 编写 Javaagent

#### 获取 java 版本的 asm 代码

要把上面的 java 代码覆盖到原来的文件, 我用的 org.ow2.asm 这个库, 当然你用其他的库也可以, 例如 bytebuddy.

 

首先把 java 方法变成 asm 库的代码, 这里我直接用 asm 库里面的 asmifier(org.ow2.asm:asm-util:9.3)

1.  下载 [asm 本体](https://repo1.maven.org/maven2/org/ow2/asm/asm/9.3/)以及 [asm.util](https://repo1.maven.org/maven2/org/ow2/asm/asm-util/9.3/) 两个 jar
2.  把之前写的 Main 编译成 class, 什么方式都行
3.  把 class 丢到下载的 jar 同目录
4.  `java -cp asm-9.3.jar;asm-util-9.3.jar org.objectweb.asm.util.ASMifier Main.class`
5.  得到 java 版的 asm 代码

#### 写 ClassFileTransformer

1.  在 manifest 定义 Premain-Class 以及 Agent-Class
    
    ![](https://bbs.pediy.com/upload/attach/202206/956011_AAU58FXFSH87UUC.png)
    
2.  实现接口 ClassFileTransformer
    
    ![](https://bbs.pediy.com/upload/attach/202206/956011_KMXQRTEKVSMUZGF.png)
    
3.  覆写激活类的方法
    
    ![](https://bbs.pediy.com/upload/attach/202206/956011_VJX36QUGB3EPYRT.png)
    

代码部分我就不详说了, 有兴趣的可以自己围观

展示
--

测试版本 burpsuite_pro_v2022.5.1.jar

 

![](https://bbs.pediy.com/upload/attach/202206/956011_75Z5QKEXDNUZJEK.png)

结束
--

自己还是太菜不会用动态 debug, 只能用这样的笨方法来研究, 代码附上, 成品附上, 欢迎各位大佬指导.

 

在这里感谢各位老师傅的路子, 让刀 burp 不用找具体激活类具体是哪个, 方便我这种后来破解的小白.

 

截至写完本文, 2022.5.1 还能这么刀

[【看雪培训】《Adroid 高级研修班》2022 年春季班招生中！](https://bbs.pediy.com/thread-271992.htm)

[#调试逆向](forum-4-1-1.htm)

上传的附件：

*   [BurpLoader-1.0-SNAPSHOT.jar](javascript:void(0)) （125.38kb，9 次下载）
*   [BurpLoader-src.zip](javascript:void(0)) （64.30kb，11 次下载）
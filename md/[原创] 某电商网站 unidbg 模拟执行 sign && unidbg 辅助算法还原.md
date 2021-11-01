> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270070.htm)

> [原创] 某电商网站 unidbg 模拟执行 sign && unidbg 辅助算法还原

某电商网站 unidbg 模拟执行`sign` && unidbg 辅助算法还原
========================================

### [](#一、前言)一、前言

免责声明：本文仅作为学习逆向技术分析、研究所用，一切由于其他非法用途而引起的法律纠纷与本作者无关。

 

本篇文章旨在分享如何用 unidbg 模拟执行 so 以及如何用 unidbg 辅助算法还原（`v10.2.0`）

*   unidbg 模拟执行，
*   unidbg 辅助分析 so 算法

二、定位受害者：`sign`签名

 

![](https://bbs.pediy.com/upload/attach/202111/907779_HSWGDMUHNT7TARS.png)

 

定位到目标，在下方加载了`libjdbitmapkit.so`。那我们 frida 跑一下看看入参和返回值，frida 开起来，打开 app 搜索，app 直接闪退了，那应该是检测 frida 的端口，做下 frida 端口转发，重新 hook，成功获取到入参和返回值

 

![](https://bbs.pediy.com/upload/attach/202111/907779_2SP55SVG234F6FQ.png)

 

分析下入参：

*   arg1: Context
*   arg2: 接口中 functionId
*   arg3: 查询参数
*   arg4: uid
*   arg5-arg6：apk 版本
*   返回值：时间戳 + sign + 算法版本

hook 完后，我们接下来进行模拟执行！

### [](#二、unidbg模拟执行)二、unidbg 模拟执行

1.  先把 unidbg 的架子搭起来

```
public class JdSign extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
    public PKCS7 pkcs7;
 
 
    JdSign() {
        // 创建模拟器实例,进程名建议依照实际进程名填写，可以规避针对进程名的校验
        emulator = AndroidEmulatorBuilder.for32Bit().build();
        // 获取模拟器的内存操作接口
        final Memory memory = emulator.getMemory();
        // 设置系统类库解析
        memory.setLibraryResolver(new AndroidResolver(23));
        // 创建Android虚拟机,传入APK，Unidbg可以替我们做部分签名校验的工作
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\jd\\1.apk"));
        vm.setJni(this); // 设置JNI
        vm.setVerbose(true); // 打印日志
        // 加载目标SO
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\jd\\libjdbitmapkit.so"), true); // 加载so到虚拟内存
        //获取本SO模块的句柄,后续需要用它
        module = dm.getModule();
        dm.callJNI_OnLoad(emulator); // 调用JNI OnLoad
    }
 
    public static void main(String[] args) {
        JdSign jdSign = new JdSign();
    }
}

```

开始运行，果不其然报错了

 

![](https://bbs.pediy.com/upload/attach/202111/907779_PUXBWD5SK23859E.png)

 

那就在这个类下补一下这个

```
@Override
public DvmObject getStaticObjectField(BaseVM vm, DvmClass dvmClass, String signature) {
    switch (signature) {
        case "com/jingdong/common/utils/BitmapkitUtils->a:Landroid/app/Application;": {
            return vm.resolveClass("android/app/Activity", vm.resolveClass("android/content/ContextWrapper", vm.resolveClass("android/content/Context"))).newObject(signature);
        }
    }
    return super.getStaticObjectField(vm, dvmClass, signature);
}

```

以此类推，报什么类型的错误就对应补响应的返回就可以了

 

有个坑强调下：

 

在补`com/jingdong/common/utils/BitmapkitZip->unZip(Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;)[B`时我们可以看到调用 unZip 方法时传入的参数正是 apk，然后进行解压，所以要把 apk 传进去，不然后续就会一直异常

 

![](https://bbs.pediy.com/upload/attach/202111/907779_7F57FK2MAJJABFG.png)

1.  调用函数
    
    ```
    public void callSign(){
            List list = new ArrayList<>(10);
            list.add(vm.getJNIEnv());
            list.add(0);
            DvmObject context = vm.resolveClass("android/content/Context").newObject(null);
            list.add(vm.addLocalObject(context));
            StringObject str = new StringObject(vm,"search");
            StringObject str2 = new StringObject(vm,"{\"addrFilter\":\"1\",\"addressId\":\"0\",\"articleEssay\":\"1\",\"deviceidTail\":\"49\",\"exposedCount\":\"40\",\"frontExpids\":\"F_0_0\",\"gcAreaId\":\"12,904,3024,0\",\"gcLat\":\"0.0\",\"gcLng\":\"0.0\",\"imagesize\":{\"gridImg\":\"532x532\",\"listImg\":\"341x341\",\"longImg\":\"532x681\"},\"insertArticle\":\"1\",\"insertScene\":\"1\",\"insertedCount\":\"2\",\"isCorrect\":\"1\",\"keyword\":\"第一行代码\",\"newMiddleTag\":\"1\",\"newVersion\":\"3\",\"oneBoxMod\":\"1\",\"orignalSearch\":\"1\",\"orignalSelect\":\"1\",\"page\":\"4\",\"pageEntrance\":\"1\",\"pagesize\":\"10\",\"pvid\":\"8cb41ee45f4145d8b5b97d4b9a81ae78\",\"searchVersionCode\":\"9418\",\"secondInsedCount\":\"0\",\"showShopTab\":\"yes\",\"showStoreTab\":\"1\",\"stock\":\"1\",\"ver\":\"108\"}");
            StringObject str3 = new StringObject(vm,"4b9655ae0e4a755a");
            StringObject str4 = new StringObject(vm,"android");
            StringObject str5 = new StringObject(vm,"10.2.0");
            list.add(vm.addLocalObject(str));
            list.add(vm.addLocalObject(str2));
            list.add(vm.addLocalObject(str3));
            list.add(vm.addLocalObject(str4));
            list.add(vm.addLocalObject(str5));
            Number number = module.callFunction(emulator,0x28b4+1,list.toArray())[0];
            String result = vm.getObject(number.intValue()).getValue().toString();
            System.out.println("result: "+ result);
        } 
    ```
    
    此时运行还会报一些简单的环境错误，对应的补一下就可以了，此处就不累述了。
    
    小 tips：**要是对 unidbg 模拟执行和补环境不是特别了解，可以加龙哥的 unidbg 星球详细学习一下，这男人真就不眠不休的更**。。。
    
    ![](https://bbs.pediy.com/upload/attach/202111/907779_WHUFTGN9FGWNENW.png)
    
    现在可以正确执行出来`sign`的结果了，unidbg 调用成功，接下来分析算法
    

### [](#三、sign算法还原)三、sign 算法还原

本案例采用 IDA + unidbg 来还原算法

1.  打开 IDA 搜索下`getSignFromJni`这个函数是否为静态注册，运气比较好，正好是静态注册，如果是动态注册，就直接使用 yang 神的脚本 hook 下 [registerNatives](https://github.com/lasting-yang/frida_hook_libart), 找下函数的偏移地址
    
    ![](https://bbs.pediy.com/upload/attach/202111/907779_VHV5CM7ZN739GE2.png)
    

1.  点进去看看这个函数，并修正下 JNIEnv，以及参数
    
    ![](https://bbs.pediy.com/upload/attach/202111/907779_XEFP84NE63MEDS6.png)
    
    大致看下这个函数
    
    ![](https://bbs.pediy.com/upload/attach/202111/907779_DHUTCF7FYCRAGDV.png)
    
    多次调用函数 sub_1261C, 将 java 层传过来的字符串拼接起来，比如 function=search&body = 省略
    
2.  在 so 层生成了一个时间戳，并拼接上去 st=13 位时间戳
    
    ![](https://bbs.pediy.com/upload/attach/202111/907779_XNWYEJXGHN7SDEB.png)
    
    原来是拼接了时间戳，我说呢，每次 unidbg 调用的 sign 都不一样，从 unidbg 调用结果可以看出，还生成了 st 时间戳传到 java 层，这很好理解，把时间戳也放进去加密了，总要让服务器知道是哪个时间戳吧。
    
    这个 sign 的值又是 32 位 16 进制，我很难不想到 md5 加密
    
3.  接着向下看
    
    ![](https://bbs.pediy.com/upload/attach/202111/907779_HPBQDYDUB4DGGGK.png)
    
    这里的 v48 就是`&sign=`这个字符串，然后又通过 sub_1261c 把 v47 拼接上去了，那 v47 应该就是 sign 的值了，往上看看前面几个主要函数，分析算法时一定要抓主干分析。
    
4.  接下来重点分析`sub_18B8`和`sub_227C`函数
    
    掏出龙哥写的 IDA 插件：[findhash](https://github.com/Pr0214/findhash) 跑一下，骚等一会应该就出结果了
    
    ![](https://bbs.pediy.com/upload/attach/202111/907779_VNBXHEXBPV4GHKC.png)
    
    此时有两种思路
    
    *   直接查看这几个特征，观察是谁调用了它
    *   frida 写个主动调用，然后用生成的 frida 脚本，hook 下看下调用栈
    
    我直接使用第一种简单看了下第一个函数，点进去看谁调用了它
    
    ![](https://bbs.pediy.com/upload/attach/202111/907779_RCWBRSMG3J4EBBF.png)
    
    大大的写着 sub_227C 调用了它，这个函数不就是刚才主流程中的函数吗，这个时候再往下看看这个函数，密密麻麻的算法，像极了 md5 运算
    
    插播一下，md5 的大致流程 (抄来的)
    
    > 哈希加密一般分为 Init、Update、Final 三步
    > 
    > 简单来说：
    > 
    > 1.  Init 是一个初始化函数，初始化核心变量
    > 2.  Update 是主计算过程
    > 3.  Final 整理和填写输出结果
    
    那这个函数不就是 md5 的计算过程吗，而且`sub_227C`多处调用了这个函数，入参肯定有参与计算的明文
    
5.  console debugger 可以派上用场了
    
    可以先逆推，我先看下面的函数结果，再看参数是怎么生成的
    
    *   先 hook 下 sub_227C 以及刚才的 update 函数
    
    ```
    public void HookByConsoleDebugger() {
            Debugger debugger = emulator.attach();
            debugger.addBreakPoint(module.base + 0x227c+1); // hook sub_227C
        }
    
    ```
    
    运行后就停在了`sub_227C`处，看一下参数的三个参数 (在 r0-r2)
    
    ![](https://bbs.pediy.com/upload/attach/202111/907779_CFQTR67KUB83J48.png)
    
    r0 和 r2 是个地址，r1 应该是个明文长度吧，打印试试
    
    ![](https://bbs.pediy.com/upload/attach/202111/907779_6PQVVAUPEQUDFJT.png)
    
    这 r1 果然是 r0 的长度，r0 应该就是明文了
    
    ![](https://bbs.pediy.com/upload/attach/202111/907779_NC794SSPJGET9F6.png)
    
    这 r2 看不出来啥，看着像个 buffer，可能用来存放结果（我们可以在函数结尾打印下这个地址看看）
    
    直接在面板输入`b0x244a`在 sub_227C 函数结尾下个断点看看，
    
    ![](https://bbs.pediy.com/upload/attach/202111/907779_QS7ZZ5W5TEEKDPZ.png)
    
    按 c 是到下一个断点，这个时候停了下来，打印刚才 r2 的地址康康
    
    ![](https://bbs.pediy.com/upload/attach/202111/907779_9MXEGC96TNEEYMD.png)
    
    胸弟，你果然不简单啊，原来你就是最后的 sign 值了，那看来分析是没错了。
    
    *   hook 下 md5update 函数看看加密的明文参数是什么
        
        在 0x1988 处下个断点，
        
        ![](https://bbs.pediy.com/upload/attach/202111/907779_TJQ8XRPK38PW3T8.png)
        
        我们看下参数
        
        ![](https://bbs.pediy.com/upload/attach/202111/907779_XPW7YE377MUFNX8.png)
        
        mr1 好像就是刚才`sub_227C`的一部分啊
        
        继续按 c 到下一个断点，还是断在 md5update 这个函数，这是因为 update 截取一部分明文 update 进去
        
        ![](https://bbs.pediy.com/upload/attach/202111/907779_5DDBJ6HWDMWE645.png)
        
        这样看下来就比较明显了，每次传 64 字节 update 进去运算，那我们直接把`sub_227C`函数传入的明文在[逆向之友](https://gchq.github.io/CyberChef/)上验证一下
        
        ![](https://bbs.pediy.com/upload/attach/202111/907779_UYTUPF3AD8FSRCM.png)
        
        有点意思，那我们只要知道入参是什么那不就结束了吗，看这个入参像个 base64，还有上面一个函数没看呢
        
    *   hook 下`sub_18B8`
        
        先打开这个函数整体看一下，
        
        ![](https://bbs.pediy.com/upload/attach/202111/907779_WVF8XPYCAVVX4E3.png)
        
        多处用到了 _aAbcdefghijklmn_，双击看一下是什么，发现就是 "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/", 这像极了 base64 的码表
        
        下个断点验证下入参和结果：
        
        ![](https://bbs.pediy.com/upload/attach/202111/907779_ZQEFC4WQR63P7NY.png)
        
        这 r0 看起来像个 buffer，等下函数结尾验证下这个地址，r1 一串很长的 16 进制是不是 base64 的明文呢？r2 应该是 r1 的长度了，在函数结尾下个断点`b0x1976`，验证下
        
        ![](https://bbs.pediy.com/upload/attach/202111/907779_EABRDKSPNGUH7EC.png)
        
        还真是 base64 的结果了，在线网站验证下，ok 没啥问题。那现在流程就很清楚了啊，就只要知道传入的参数 r1 这一串是怎么生成的就 ok 了
        
6.  梳理下 sign 加密流程
    
    *   sub_127E4 函数传入了 java 层的字符串，拼接成一个字符串
    *   so 生成一个时间戳拼接到明文上
    *   sub_126AC 函数将明文做异或操作，变成一串像乱码一样的字符
    *   sub_18B8 函数进行 base64 编码
    *   base64 编码的结果进行 md5 加密
7.  分析 sub_126AC 对明文做了什么操作
    
    *   入参分析
        
        ![](https://bbs.pediy.com/upload/attach/202111/907779_JJAW8JPVRPS2VKK.png)
        
        参数 1 就是 JNIEnv，,v65 为加密明文字节码, v33 为加密长度, v26 以及 v27 为随机数，v25、v26 经过 sub_12640 函数转成整数，控制算法的版本
        
    *   sub_126AC 函数下面的 switch 就是控制走哪个版本的算法，我分析了其中一个 v=111 走 sub_10DE4 生成了 base64 的入参
        
    *   ![](https://bbs.pediy.com/upload/attach/202111/907779_RSBEHARUJREZVDC.png)
    
    hook 下看下入参
    
    参数 1：固定的字符串
    
    ![](https://bbs.pediy.com/upload/attach/202111/907779_JAXBTJ7738D9E3B.png)
    
    参数 2：拼接后的明文
    
    ![](https://bbs.pediy.com/upload/attach/202111/907779_BHWQ6EJ92YVMR8Y.png)
    
    拼接的格式 functionId=...&body = 查询的参数 & uuid=...&client=android&clientVersion=10.2.0&st=13 位时间戳 & sv=111(算法版本，可以请求时固定)
    
8.  sub_12ECC 对明文做了异或处理
    
    ![](https://bbs.pediy.com/upload/attach/202111/907779_NGZEU32JCDW7THT.png)
    
    进入 sub_12ECC, 分析下入参：
    
    五个参数，此处讲一下 r0-r3 存放前 4 个参数，第 5 个参数在 sp 中，可小端序查看一下
    
    参数 1：
    
    ![](https://bbs.pediy.com/upload/attach/202111/907779_JCGD7E53E3QWEJZ.png)
    
    参数 2：固定的字符串  
    ![](https://bbs.pediy.com/upload/attach/202111/907779_H99MGQ7UDE36N2A.png)
    
    参数 3：1
    
    参数 4：明文
    
    参数 5：明文长度
    
    msp 查看 sp 寄存器，小端序查看为 0x02e8, 然后打印 r3
    
    ![](https://bbs.pediy.com/upload/attach/202111/907779_TWMGUMFCWC9NTF6.png)
    
    经分析，主要的异或在下方的循环中
    
    ```
    if ( a3 )                                   // pass
       {
         v15 = 0;
         do
         {
           v16 = &v21[7] + (v15 & 15);             // v21这个数组偏移0x20(&v21[7])
           v17 = v15++ & 7;
           v18 = *(v16 - 20); 取数组往前偏移20字节
           result = ((v18 ^ *plainText ^ *(a2 + v17)) + v18);
           LOBYTE(v18) = v18 ^ result;             // 这里逐位异或,LOBYTE取低位的一个字节
           *plainText++ = v18;
           *(plainText - 1) = v18 ^ *(a2 + v17);
         }
         while ( v15 != plainTextlen );
       }
    
    ```
    

打一个断点在循环的开始 b0x12F68, 查看下循环

 

![](https://bbs.pediy.com/upload/attach/202111/907779_73PBXQJBA8SUJSG.png)

 

可以配合汇编一起看下

1.  and r2, r3, #0xf
    
    将 r3 & 0xf 并存到 r2 寄存器中，r3 观察循环可知就是 v15，
    
2.  add r0, sp, #0x20
    
    可以按 s 单步调试，查看下 sp 寄存器
    
    ![](https://bbs.pediy.com/upload/attach/202111/907779_H4BSVJ2RKY85EJP.png)
    
    这几个是循环中一直用到的分析下
    
    ```
    v21[2] = 0x68449237;
    v21[3] = 0x7FCC3DA5;
    v21[4] = 0x88D90FBB;
    v21[5] = 0x5AE99AEE;
    
    ```
    
    其实就是 v21 的小端序，写死就可以。其它的可以对照汇编指令修改一下，就可以把这个循环改写出来了
    
3.  测试下 sign，没问题
    
    ![](https://bbs.pediy.com/upload/attach/202111/907779_RB3M2Y48NUZB3B6.png)
    

### [](#四、表单body加密)四、表单 body 加密

本来想写到这里就结束了，后来发现我换个关键词去搜索，就歇菜了，那肯定是还有其他参数做了关键词和页数校验

 

hook 发现，果然这里还有个 base64，而且是个自写的仿 base64

```
function hook_base64() {
    // hook cipher中的body加密
    Java.perform(function () {
        var cls = Java.use("com.jd.phc.d")
        var jstring = Java.use("java.lang.String")
        cls.x.implementation = function (bArr) {
            var result = this.x(bArr)
            console.log("base64 arg1: ",jstring.$new(bArr))
            console.log("base64 result:",result)
            return result
        }
    })
}

```

逻辑比较简单，重写下 java 就可以了

### [](#五、写在最后)五、写在最后

这篇整体来说难度不大，后面可能会遇到算法去符号，md5 魔改、aes 等，这就需要对算法的原理非常熟悉了。

 

最后还是推荐加入龙哥的星球学习 unidbg 教程，或者肉师傅之前的 so 系列课系统的学习密码学实现

 

![](https://bbs.pediy.com/upload/attach/202111/907779_B46MP788FXUBFEE.jpg)

[[注意] 欢迎加入看雪团队！base 上海，招聘安全工程师、逆向工程师多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)

[#调试逆向](forum-4-1-1.htm) [#加密算法](forum-4-1-5.htm)
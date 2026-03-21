> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-290441.htm)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

本文为过银行类 app  某加密检测流程以及详细分析 ，属于自嗨 ，但是是我也做了很认真的分析 ， 分析的时候遇到了很多原理上的盲区，还希望大佬指点 纯为新手逆向流程，希望能给和我一样入门 so 层的新手朋友提供一些下手的思路  ，如有误导，还请各位大佬指正，谢谢！    本文所有包名进行打码

1.  如何拿到的脱壳的防护 so 文件 libexec.so ，怎么修复       这里篇幅过长  所以大家可以参考东方玻璃大佬这个帖子 [[原创] 某加固新版 frida 检测绕过 - trace 一把嗦 - Android 安全 - 看雪安全社区｜专业技术交流与安全研究论坛](https://bbs.kanxue.com/thread-289545.htm)
    
2.  hook clone 偏移的原理             看雪有不少优质帖，大家可以去了解了解，这里本人底层知识不够扎实 ，怕误导大家
    
3.  追踪工具的使用              可以来私信作者，我不记得我怎么下载的了，嘿嘿 
    

 某加密 libexec.so 的进程终止并非来自 SO 加载阶段（init_array/JNI_OnLoad）的子线程检测，而是 Java 层早期调用 JNI 动态注册的 native 方法（sub_68060），该方法通过全局检测结构体（off_E3290）调用核心检测函数 sub_55eb8，检测异常后触发 sub_66998 杀进程；最终通过 Hook sub_55eb8 并强制返回 0 实现绕过。

![](https://bbs.kanxue.com/upload/tmp/1068038_GWY6UXFGPG2MU44.webp)

frida 注入应用被杀进程 ， 发现 dlopen 了两个 so ，这是某加密的 so 检测文件 经验来说 某加密的 so 检测主要在 libexec.so 

下面分析是哪个层面被杀的 这里的层面指的是杀死进程的时机

先明确 SO 加载核心时机：`call_constructors`（SO 加载时最先执行，触发. init/.init_array 段）→ `JNI_OnLoad` → SO 业务函数 / 子线程阶段。

验证逻辑：

1.  **能跑出 libexec.so 的 JNI 函数 → 说明`call_constructors`（含. init/.init_array）和`JNI_OnLoad`阶段未直接杀进程（这两个阶段杀进程，JNI 函数根本跑不出来）；**
2.  **因此锁定后续阶段，Hook clone 函数排查：是否是 SO 通过 call_constructors 初始化后，创建子线程执行检测并杀进程。**
    

这里如果想了解详细机制 ，大家可以去自行找帖子，b 站勇敢的小佳有一个视频比较好的介绍了这个流程 ，这里先用脚本注入验证

```
const target_so = "libexec.so";
var flag = false;

console.log("[+] 监控启动！精准阻断加固，保留业务逻辑...");

// Hook android_dlopen_ext 监控SO加载
Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"), {
    onEnter: function (args) {
        this.path = args[0].readCString();
        console.log("加载SO：" + this.path);
        if (this.path.indexOf(target_so) !== -1) {
            console.log("检测到目标SO加载，准备处理");
            flag = true;
        }
    },
    onLeave: function (retval) {
        if (flag) {
            flag = false;
            console.log("目标SO已加载，开始精准处理加固函数");

            // 获取模块基址
            const moduleBase = Module.findBaseAddress(target_so);
            if (!moduleBase) {
                console.error("[-] 找不到模块基址！");
                return;
            }
            console.log("[+] 模块基址：" + moduleBase);

            // 计算sub_66998的绝对地址
            const sub_66998_Addr = moduleBase.add("");
            console.log("[+] jni_onload地址：" + sub_66998_Addr);
            Interceptor.attach(sub_66998_Addr, {
                onEnter: function (args) {
                    console.log("进来");
                },
                onLeave: function (retval) {
                    console.log("出去")
                }

            })
        }
    }
});

// 兜底：拦截栈检查失败函数，避免崩溃
Interceptor.attach(Module.findExportByName(null, "__stack_chk_fail"), {
    onEnter: function () {
        console.log("[+] 阻断栈检查失败函数__stack_chk_fail！");
        this.returnValue = ptr(0);
        this.context.pc = this.context.lr;
    }
});

```

![](https://bbs.kanxue.com/upload/attach/202603/1068038_66D82MGNUMGN69Z.webp)

发现成功跑出 libexec.so 的 jni 函数，应该不是在 iniarray 或者 jni_onload 里面做的进程终止 

hook clone 函数，查看 so 文件地址上有没有创建子线程的痕迹 

 clone 函数是 Android 系统中创建线程的底层核心函数，监控它就能精准捕捉到 libexec.so 创建检测子线程的源头行为       详细参考前置说明

```
function hook_cer_test_and_offset() {
    // 只监控目标防护SO，过滤无关线程
    // 【原有】基础防护SO + 首轮可疑SO | 【新增】最新日志里的所有出现的SO（全匹配日志原名）
    const TARGET_SO = [
        // 原有监控SO（保留不动，无任何修改）
     


    ];
    var clone = Module.findExportByName('libc.so', 'clone');
    if (!clone) {
        console.log("[-] 未找到clone函数");
        return;
    }
    Interceptor.attach(clone, {
        onEnter: function (args) {
            // 容错：args[3]为0则跳过（无指定栈地址，非独立线程）
            if (args[3].isNull() || args[3] == 0) return;
            try {
                // arm64 Android 8.x 通用栈偏移96，若解析失败可调试±16/±32
                var addr = args[3].add(96).readPointer();
                // 容错：解析出的地址无效则跳过
                if (addr.isNull() || addr == 0) return;
                var mod = Process.findModuleByAddress(addr);
                // 只打印目标防护SO的线程信息

                var so_base = mod.base;
                var offset = addr - so_base;
                // 格式化输出：SO名 + 虚拟地址 + 十进制偏移 + 十六进制偏移（大写，适配IDA）
                console.log("【检测线程定位】");
                console.log("SO名：", mod.name);
                console.log("入口地址：", addr);
                console.log("十进制偏移：", offset);
                console.log("十六进制偏移：", "0x" + offset.toString(16).toUpperCase());
                console.log("=====================================");

            } catch (e) {
                // 捕获解析异常，避免脚本崩溃
                // console.log("[-] 解析失败：", e.message);
            }
        },
        onLeave: function (retval) { }
    });
    console.log("[+] clone函数Hook完成，仅监控目标防护SO（含最新新增3个可疑SO + 日志所有出现SO）");
}

// 配合你原有的patch+libnllvm Hook，一起运行
setImmediate(() => {
    hook_cer_test_and_offset();
    // 你的原hook_dlopen_and_patch()  // 长沙银行补丁逻辑
    // 你的原hook_dlopen_libnllvm()  // 原有libnllvm监控逻辑
});

```

![](https://bbs.kanxue.com/upload/tmp/1068038_HPSN3J4Q6G5J3YW.webp)

检测到 so 层创造了两个线程 ，这里打印的偏移是 clone 函数执行的入口函数 ， 大家可以理解为产生了一个工人 ，这里偏移就是他的任务      我们跳转到 ida 的入口函数看看 ，看看是不是有什么检测

先看看 0x42FE4

![](https://bbs.kanxue.com/upload/tmp/1068038_8U2MGH7QQ9K6EMK.webp)

![](https://bbs.kanxue.com/upload/tmp/1068038_D8ZCJEQJ36MWU8V.webp)

 可以发现这里调用了 TPIDR_EL0 后续又做了校验 ，有反调试嫌疑 ，继续跟进。发现末尾做了标志位校验，如果监测到异常，直接跳入死亡函数分支，如果没有，函数 ret0     sub_dd770 是一个跳板函数 ， 跳入一个偏移 ，偏移是自杀函数

![](https://bbs.kanxue.com/upload/tmp/1068038_XMCBHCMDS6QHT9E.webp)

![](https://bbs.kanxue.com/upload/tmp/1068038_35775T3NJEC5WUV.webp)

那么另外一个偏移呢 0x42F90  ，我们同样跳转看看

![](https://bbs.kanxue.com/upload/attach/202603/1068038_UVSWEQ97MJFN943.webp)

这里其实我已经确定是一个检测函数了 ，为什么呢，要归于我之前的判断 ，首先，开头 if（*off_E3290）很重要，这个极有可能是一个检测防护结构体的地址，是一个二级指针 ， 为什么这样说呢 ，我们对刚刚分析的偏移 0x42FE4 分析看看 ，我们一直对这个函数链交叉引用

![](https://bbs.kanxue.com/upload/tmp/1068038_HJ8DZ4R9HPVDFV6.webp)

![](https://bbs.kanxue.com/upload/tmp/1068038_SCVN5E8XNUEHMHE.webp)

![](https://bbs.kanxue.com/upload/tmp/1068038_65VB6PQWSF8XVMW.webp)

![](https://bbs.kanxue.com/upload/tmp/1068038_VQRXC69PCUKSNHU.webp)

最后追踪到了 init_array 段，我们先梳理一下，我们刚刚分析的链条 sub_432A4 -> off_E5BB0 偏移 -> 431A0  ->  检测子线程入口 42FE4 

开头的 sub_432A4 在 init_array 段和其他函数一起被早期执行 这里一般有大量的初始化和检测，是绕过的重点区域

![](https://bbs.kanxue.com/upload/tmp/1068038_W9RNSPT9NRVT3FQ.webp)

我们再聚焦于 sub_432A4 干了什么，就可以知道，我为什么说偏移 off_E3290 是一个防护结构体了 ，后面也可以验证我的猜想 

![](https://bbs.kanxue.com/upload/tmp/1068038_9TG44CBMQXQX8UF.webp)

这里先是 LDR 读取 off_E3290 的所在内存页偏移 （就是拿到 off_E3290）的地址到 x8

然后读取 off_E5BB0 的地址保存到 x9

最后把 x9 的地址放入 x8 的 0x40 偏移处

这里就类似于 *off_E3290 -> 0x40 = int * off_E5BB0 就是把这个偏移放入 off_E3290 结构体

所以我们现在来看看到底放了什么东西吧

先看 off_E5BB0

![](https://bbs.kanxue.com/upload/tmp/1068038_AFKMBKPZ9KFWYZK.webp)

 这不正是我们向上追溯的 431A0 吗，也就是子线程入口函数的调用上层，所以说这里很明确了，这个 so 会把偏移函数放到函数表 off_E5BB0, 再最终放入 off_E3290, 肯定不止一个函数表 off_E5BB0 

, 肯定会有其他的函数表被放入检测结构体，这里后面也有验证 但是需要注意的是，也有可能其实更大可能是一个全局结构体，只是放入了很多检测函数表

![](https://bbs.kanxue.com/upload/tmp/1068038_8PBG6A5RWUNT585.webp)

大量函数调用 off_E3290  这里肯定是有很多检测函数调用 off_E3290 结构体 ，为什么要这样多此一举呢，因为可以妨碍我们追踪调用堆栈，我们定位死亡函数，向上交叉引用是引用不到真正调者的，因为可能是通过这个调用 off_E3290 结构体进行调用，但是注意，这里这个结构体不仅仅有函数，还有检测状态数等

我们具体聚焦刚刚分析的第二个子线程入口函数偏移 0x42F90，这里就很明确了

![](https://bbs.kanxue.com/upload/tmp/1068038_6YQPMN9JN2H3JUM.webp)

 先判断检测体是不是为空， 执行计数操作最后进入函数 sub_43364 

  ![](https://bbs.kanxue.com/upload/tmp/1068038_YCWWFP9224ED6A6.webp)

![](https://bbs.kanxue.com/upload/tmp/1068038_N4WD3KETB9HHDXJ.webp)

这里先拿全局结构体 off_E3290 + 18 和 + 88 ，调用函数 sub_75F40 做校验，检查源码发现就是一个逐字节对比函数，检查是否是相同字符 ，源码在上图 ，  只要不匹配进入检测 if 分支 ，再调用 sub_75F40  把一段字符串放入缓冲区 v71 再调用 sub_75DC4 进行校验，看缓冲区包不包含 ":"，如果失败 ，跳转 LABEL_52 

![](https://bbs.kanxue.com/upload/tmp/1068038_T9CC6QY2H6MUE2S.webp)

![](https://bbs.kanxue.com/upload/tmp/1068038_CWB2S2MJDN435BF.webp)'

最后跳转死亡函数，和刚刚一样的 stack_chk_fail 

检测远不止这点，这里不逐个分析了，所以这里这个子线程的入口函数 0x42F90 也是检测函数，所以两者都为检测线程，直接试试把两个入口 ret0 

```
const LIB_NAME = "libexec.so";
const TARGET_OFFSETS = [0x42F90 ， 0x42FE4]; // 需要Hook的函数偏移列表

// 核心Hook逻辑：拦截SO加载 + 批量Hook目标函数
function hookTargetSOAndForceReturn() {
    // 1. 查找并拦截android_dlopen_ext（SO加载入口）
    const dlopen = Module.findExportByName(null, "android_dlopen_ext");
    if (!dlopen) {
        console.log("[-] 未找到 android_dlopen_ext 函数");
        return;
    }

    Interceptor.attach(dlopen, {
        onEnter(args) {
            // 标记是否为目标SO
            this.isTarget = args[0] ? args[0].readCString()?.includes(LIB_NAME) : false;
        },
        onLeave(retval) {
            // 仅当目标SO加载成功时执行Hook
            if (this.isTarget && !retval.isNull()) {
                const soBase = Module.findBaseAddress(LIB_NAME);
                if (!soBase) {
                    console.log(`[-] 未找到 ${LIB_NAME} 基址`);
                    return;
                }
                console.log(`[+] ${LIB_NAME} 加载成功，基址: ${soBase}`);

                // 2. 批量Hook目标函数，强制返回0（64位用0n）
                TARGET_OFFSETS.forEach(offset => {
                    const funcAddr = soBase.add(offset);
                    Interceptor.replace(funcAddr, new NativeCallback(() => {
                        console.log(`[+] 强制拦截 0x${funcAddr.toString(16)}，直接返回0`);
                        return 0n; // 64位函数返回long类型，用0n更稳妥
                    }, 'long', [])); // 匹配目标函数返回值类型（__int64对应long）
                });
            }
        }
    });
}

// 立即执行
setImmediate(hookTargetSOAndForceReturn);

```

但是我注入脚本之后，发现进程还是死了

![](https://bbs.kanxue.com/upload/attach/202603/1068038_YC5X37VA75P226Z.webp)

我们使用 ida 的外部插件 stalker trace so 来追踪一下函数的执行流，定位死亡函数 

我们注入生成后的 trace 流脚本 ，同时注入我们的线程杀函数脚本，对比两者函数执行流有无区别

这里是同时注入的      frida frida -U -f 包名 -l 杀偏移脚本   -l  trace 脚本

用了杀函数脚本

![](https://bbs.kanxue.com/upload/attach/202603/1068038_NKYDGHB96DPC3FZ.webp)

没用杀函数脚本

  用了杀函数脚本

![](https://bbs.kanxue.com/upload/attach/202603/1068038_M3YDKGK99BYA42E.webp)

可以看到，没有任何区别，这就奇怪了啊，为什么呢我们杀线程没有影响一点最后的终止函数线程流，这说明并不那两个线程最终触发的函数死亡

我们进一步分析最后死亡的函数 sub_6AAC8

![](https://bbs.kanxue.com/upload/tmp/1068038_9EZT863DVC2KTNN.webp)

可以看到直接发送死亡信号 ，杀了进程，这肯定是死亡流末尾了，也就是已经在上层触发了死亡分支，我们要向上交叉引用

![](https://bbs.kanxue.com/upload/tmp/1068038_6532NATBMWYKSPW.webp)

也就是执行流堆栈的 66998 ， 我们直接查看代码

66998 函数是一个烈性的检测杀进程，包含大量自杀逻辑，没有任何业务分支

![](https://bbs.kanxue.com/upload/tmp/1068038_ZNDURTSMRUH3KHJ.webp)

 拿到自身 pid 为后续 kill 进程做准备

![](https://bbs.kanxue.com/upload/tmp/1068038_8ZK973M32FRZRSX.webp)

轮询发送死亡信号，杀进程

![](https://bbs.kanxue.com/upload/tmp/1068038_EHKG6NMG9TQQ7QH.webp)

末尾调用死亡函数 6AAC8 ，没有任何活分支 ，这里只展现冰山一角，我们继续向上追溯，这里肯定远远不够

![](https://bbs.kanxue.com/upload/tmp/1068038_VVKSTWWV8BFF274.webp)

![](https://bbs.kanxue.com/upload/tmp/1068038_KEU42P3S753BRUU.webp)

交叉引用发现静态调用没有符合 66998 的上层，无论是 sub57594，还是，sub_55EB8，这里为什么不说 sub_75DC4 呢，因为刚刚已经分析了，这是一个校验函数，同时没有发现 66998 被导入到检测函数表的行为，因为如果有，可能是调用结构体引用，隐式调用死亡函数 66998 ，但是这里都没有，逻辑已经完全断了

我开始思考是不是线程没杀顺利，因为我们是在 dlopen onleave 时期进行的函数 ret0 ，但是如果是 init_array 段的话，是在 dlopen 进入时期，调用 call_constructor 这个更早的时期执行的，这个

关于这里执行机制的讲解，不赘述了，大家可以去看看 b 站视频，有关于这个时期的讲解，也就是说，我们 hookcall_constructor，可以在 init_array 执行之前阻断子线程的加载，但是前提是确实是 init_array 执行的子线程，所以我们 hook 一下调用 clone 函数的调用者，看看所在偏移，定位具体函数，明确产生时机

```
function hook_cer_test_and_offset() {
    const TARGET_SO = [
       
    ];

    // 核心优化：从调用栈中提取目标SO的调用者
    function extractTargetCallerFromStack() {
        try {
            const stack = Thread.backtrace(this.context, Backtracer.ACCURATE);
            // 遍历调用栈，找第一个属于目标SO的帧
            for (const frame of stack) {
                const mod = Process.findModuleByAddress(frame);
                if (mod && TARGET_SO.includes(mod.name)) {
                    const caller_offset = frame - mod.base;
                    return {
                        name: mod.name,
                        base: mod.base,
                        addr: frame,
                        offset: caller_offset
                    };
                }
            }
        } catch (e) { }
        return null;
    }

    var clone = Module.findExportByName('libc.so', 'clone');
    if (!clone) {
        console.log("[-] 未找到clone函数");
        return;
    }

    Interceptor.attach(clone, {
        onEnter: function (args) {
            // 1. 从调用栈提取目标SO的调用者（核心优化）
            const targetCaller = extractTargetCallerFromStack.call(this);
            if (targetCaller) {
                console.log("\n========== 【目标SO调用clone定位】 ==========");
                console.log(`调用clone的目标SO: ${targetCaller.name}`);
                console.log(`目标SO基址: ${targetCaller.base}`);
                console.log(`调用者函数地址: ${targetCaller.addr}`);
                console.log(`调用者函数偏移: 0x${targetCaller.offset.toString(16).toUpperCase()}`);
                console.log("=============================================");
            }

            // 2. 原有逻辑：解析clone创建的线程入口（保留并优化）
            if (args[3].isNull() || args[3] == 0) return;

            try {
                const stackOffsets = [96, 80, 112, 64];
                let addr = null;
                for (const offset of stackOffsets) {
                    addr = args[3].add(offset).readPointer();
                    if (!addr.isNull() && addr != 0) break;
                }
                if (addr.isNull() || addr == 0) return;

                var mod = Process.findModuleByAddress(addr);
                if (!mod || !TARGET_SO.includes(mod.name)) return;

                var so_base = mod.base;
                var offset = addr - so_base;

                console.log("\n========== 【检测线程入口定位】 ==========");
                console.log(`SO名：${mod.name}`);
                console.log(`线程入口地址：${addr}`);
                console.log(`线程入口十进制偏移：${offset}`);
                console.log(`线程入口十六进制偏移：0x${offset.toString(16).toUpperCase()}`);
                console.log("===========================================");

            } catch (e) { }
        },
        onLeave: function (retval) { }
    });

    console.log(`[+] clone函数Hook完成，监控${TARGET_SO.length}个目标防护SO`);
}

setImmediate(() => {
    hook_cer_test_and_offset();
});

```

![](https://bbs.kanxue.com/upload/tmp/1068038_N2NMRS9Y3QJU6YC.webp)

定位核心偏移 ，也就是红圈处, 这里就是调用 clone 函数的上层

![](https://bbs.kanxue.com/upload/tmp/1068038_ZSRR3Y2GZW5PMA8.webp)

该偏移所属函数正是刚刚分析的子线程入口函数上层 sub_431A0 , 它正是刚刚说的，被放入函数表的函数，![](https://bbs.kanxue.com/upload/tmp/1068038_ERDKG8MB9Q2JN9X.webp)

所以流程可能为，init_array 段 初始化函数表，把函数表放入全局结构体 off_E3290 ， init_array 后续函数调用 off_E3290 中的 sub_431A0  产生子线程 ， off_E3290 的被引用次数非常大，一个个定位来验证猜想很花时间，所以这里决定把注入时机换成 call_constructor，看看是否能成功   

call_constructor 是一个比 dlopen onleave 阶段还早的时期，详细知识参考前置说明

```
const LIB_DEX_HELPER_NAME = "libexec.so";
const TARGET_FUNC_OFFSET = 0x55eb8;
const TARGET_FUNC_OFFSET2 = 0x353BC;

// 所有需要 Hook 的函数偏移列表
const TARGET_OFFSETS = [
    0x42F90,
    0x42FE4
];

// 目标防护SO列表（保留你的原有配置）
const TARGET_SO = [
    "libDexHelper.so",
    "libnllvm1717052779924.so",
    "lib52pojie.so",
    "libCmbShield.so",
    "libcmbwbsk_crypto_tool.so",
    "libzhim_crypto_tool.so",
    "libexec.so",
    "libexecmain.so",
    "libzhkv.so",
    "libbangcle_risk.so",
    "libbangcle_risk00.so",
    "RiskStub00.odex",
    ""
];

// 全局变量：标记SO是否已Hook，避免重复操作
let isSOHooked = false;
// 全局变量：存储libexec.so基址
let libexecBase = null;

/**
 * 核心函数：替换目标地址为返回0
 * @param {NativePointer} address - 要替换的函数地址
 * @param {string} retType - 返回值类型（int/long/void）
 */
function replace0(address, retType = 'int') {
    try {
        Interceptor.replace(address, new NativeCallback(function () {
            console.log(`[+] 强制拦截 ${address}，直接返回0`);
            return retType === 'long' ? 0n : 0;
        }, retType, []));
        console.log(`[+] 成功替换函数: ${address}`);
    } catch (e) {
        console.log(`[-] 替换函数失败 ${address}: ${e.message}`);
    }
}

/**
 * 你的原有逻辑：监控目标SO的线程创建
 */
function hook_cer_test_and_offset() {
    var pthread_create = Module.findExportByName('libc.so', 'pthread_create');
    if (!pthread_create) {
        console.log("[-] 未找到pthread_create函数");
        return;
    }

    Interceptor.attach(pthread_create, {
        onEnter: function (args) {
            try {
                if (args[2].isNull() || args[2] == 0) return;
                var entry_addr = args[2];
                var mod = Process.findModuleByAddress(entry_addr);

                if (mod && TARGET_SO.includes(mod.name)) {
                    var so_base = mod.base;
                    var offset = entry_addr - so_base;
                    console.log("【检测线程定位】");
                    console.log("SO名：", mod.name);
                    console.log("入口地址：", entry_addr);
                    console.log("十进制偏移：", offset);
                    console.log("十六进制偏移：", "0x" + offset.toString(16).toUpperCase());
                    console.log("=====================================");
                }
            } catch (e) { }
        },
        onLeave: function (retval) {
            if (retval != 0) {
                console.log(`[-] 线程创建失败，错误码：${retval}`);
            }
        }
    });
    console.log("[+] pthread_create函数Hook完成，仅监控目标防护SO");
}

/**
 * 核心修改：通过call_constructors提前Hook目标函数
 */
function hook_linker_call_constructors() {
    // 1. 获取linker64基址（兼容32位linker）
    let linker_base = Module.getBaseAddress("linker64") || Module.getBaseAddress("linker");
    if (!linker_base) {
        console.error("[-] 未找到 linker 模块！");
        // 降级方案：使用原有dlopen拦截
        hook_dlopen_and_force_return();
        return;
    }

    // 你的call_constructors偏移（根据实际设备调整，这里保留你的0x20AF8）
    let call_constructors_addr = linker_base.add(0x20AF8);
    console.log(`[+] linker基址: ${linker_base}`);
    console.log(`[+] call_constructors地址: ${call_constructors_addr}`);

    // 2. Hook call_constructors（SO构造函数执行前的关键节点）
    Interceptor.attach(call_constructors_addr, {
        onEnter: function (args) {
            console.log("\n[+] ===== call_constructors 被调用（SO构造函数即将执行）=====");

            // 提前查找libexec.so基址（此时SO已加载，init_array还未执行）
            libexecBase = Module.findBaseAddress(LIB_DEX_HELPER_NAME);
            if (libexecBase && !isSOHooked) {
                console.log(`[+] 提前找到libexec.so基址: ${libexecBase}`);
                // 批量替换目标函数（核心：此时替换，init_array执行前生效）
                TARGET_OFFSETS.forEach(offset => {
                    const funcAddr = libexecBase.add(offset);
                    console.log(`[+] 准备提前Hook函数: 0x${offset.toString(16)} -> ${funcAddr}`);
                    // 根据函数返回类型调整（64位建议用long）
                    replace0(funcAddr, 'long');
                });
                isSOHooked = true; // 标记已Hook，避免重复操作
            }

            // 打印调用来源和调用栈（辅助调试）
            let caller_addr = this.context.pc;
            let caller_module = Process.findModuleByAddress(caller_addr);
            if (caller_module) {
                console.log(`[+] 调用来源模块: ${caller_module.name} (基址: ${caller_module.base})`);
            }
            console.log("[+] 调用栈：");
            console.log(Thread.backtrace(this.context, Backtracer.ACCURATE)
                .map(DebugSymbol.fromAddress).join("\n"));
        },
        onLeave: function (retval) {
            console.log("[+] ===== call_constructors 执行完成 =====\n");
        }
    });
    console.log("[+] call_constructors Hook 挂载成功！");
}

/**
 * 原有dlopen拦截逻辑（降级方案）
 */
function hook_dlopen_and_force_return() {
    const android_dlopen_ext = Module.findExportByName(null, "android_dlopen_ext");
    if (!android_dlopen_ext) {
        console.log("[-] 未找到 android_dlopen_ext 函数");
        return;
    }

    Interceptor.attach(android_dlopen_ext, {
        onEnter(args) {
            this.isTargetSO = false;
            const pathPtr = args[0];
            if (pathPtr) {
                const path = pathPtr.readCString();
                if (path && path.includes(LIB_DEX_HELPER_NAME)) {
                    this.isTargetSO = true;
                    console.log(`[+] 检测到加载目标SO: ${path}`);
                }
            }
        },
        onLeave(retval) {
            if (this.isTargetSO && !retval.isNull() && !isSOHooked) {
                console.log(`[+] ${LIB_DEX_HELPER_NAME} 加载成功`);
                libexecBase = Module.findBaseAddress(LIB_DEX_HELPER_NAME);
                if (!libexecBase) {
                    console.log(`[-] 未找到 ${LIB_DEX_HELPER_NAME} 基址`);
                    return;
                }
                console.log(`[+] ${LIB_DEX_HELPER_NAME} 基址: ${libexecBase}`);
                // 批量Hook（降级方案，仅当call_constructors Hook失败时执行）
                TARGET_OFFSETS.forEach(offset => {
                    const funcAddr = libexecBase.add(offset);
                    console.log(`[+] 准备 Hook 函数: 0x${offset.toString(16)} -> ${funcAddr}`);
                    replace0(funcAddr, 'long');
                });
                isSOHooked = true;
            }
        }
    });
    console.log("[+] android_dlopen_ext Hook 挂载成功（降级方案）");
}

// 主执行逻辑：先启动call_constructors Hook + 线程监控
setImmediate(() => {
    hook_linker_call_constructors(); // 核心：提前Hook
    hook_cer_test_and_offset();      // 保留：线程监控
});

```

![](https://bbs.kanxue.com/upload/attach/202603/1068038_9KTFN35X98EV6K7.webp)

依旧死亡，说明子线程打不打，都是无法影响后续的死亡，说明还有其他核心的检测点 ，但是刚刚核心逻辑已经断了 ，因为 66998 静态引用并没有 trace 脚本日志打印的上层合适的函数，所以我决定直接 ret 66998 死亡函数为 0，看看是否能过检测 

![](https://bbs.kanxue.com/upload/tmp/1068038_W6PRB7QC665JBNB.webp)

调用这个之后，又会转而调用 __stack_chk_fail 函数，无论如何都是死，我决定放弃聚焦下层，聚焦上层看看，我们交叉引用 66998，发现在早期执行流中，66254 有调用嫌疑 ![](https://bbs.kanxue.com/upload/tmp/1068038_RKWEZHQ4KJGQBK4.webp)

![](https://bbs.kanxue.com/upload/tmp/1068038_EZ9TTBTS53YW8SB.webp)

但是这可是第 20 层啊，66998 在 100 多层去了，中间跨度极大，但是我还是抱着试试看的心态，先 hook 这个函数，看看能不能进来出去，看看是不是在这里面死亡

![](https://bbs.kanxue.com/upload/tmp/1068038_4GW76XUUGUXBHPK.webp)

进来出去都成功    果然不是这里死亡的 ，并且向上追溯发现，这是 jnionload 的后续调用的业务方法，只是在里面检测分支调用 sub_66998，可是这里的 sub_66998 没有触发，因为我用的是魔改 frida，当时这层检验可能通过了，但是后续并没有通过。那么还有什么原因呢。

第一，我此时杀了子线程，我 hook 了 libexec 的 so 里面的所有子线程的入口函数，并且入口函数后续其实并没有调用 sub_66998 的分支。 第二，66998 交叉引用并没有直接被放入函数表，也就是没有被结构体引用调用的可能性 ，第三 66998 的交叉引用调用者只有一个 66254 在函数执行流里面，我 hook66254，并没有在里面死，我把三条可能的链路全走不通

子线程 -> 66998 -> 6AAC8 -> 死亡 （被推倒）

主线程 -> jni_onload -> 66254 -> 66998 -> 6AAC8 ->  死亡 （被推倒）

66254 -> 结构体函数表 -> 引用 66998  -> 66AC8 -> 死亡（被推倒）

66254 -> 结构体函数表 -> 引用 66998 父亲函数 （调用者） -> 66998  -> 66AC8 -> 死亡（被推倒）

没有线索就创造线索 ，我们 hook 66998，看看调用者是谁

![](https://bbs.kanxue.com/upload/tmp/1068038_RFGJZ596JE7F3H8.webp)

发现了一个极为重要的线索  可以看到，这里调用者是 0x68480，我们跳转看看属于哪个函数

![](https://bbs.kanxue.com/upload/attach/202603/1068038_39R5ZACKHGUCBZ7.webp)

居然是这个，之前执行流完全没出现！！

这里显式调用了 66998 死亡函数 ![](https://bbs.kanxue.com/upload/attach/202603/1068038_Q4HZGR5H776JCYP.webp)

这里验证了我的猜想，要想调用 66998，一定是显式调用，不可能是结构体偏移调用，因为它交叉引用没有去任何一个函数表，如果是通过上级跳转然后注册上级函数的方式放入函数表，上级也会在执行流引用，这里上级引用就一个 66254 ，已经判断了没在这个函数里面死亡 ，执行流完全没有 680C0 的痕迹 ， 子线程入口函数偏移和这里完全不相关   这里我基本可以判断了，是结合 jni 函数进行调用 ，也就是在 66259 注册一个 native 函数 ，然后被 java 通过 jni 调用，这样，就和刚刚的所有流完全脱了关系，我们找到代码来验证猜想

![](https://bbs.kanxue.com/upload/attach/202603/1068038_7E9PN7Z2G7EM6ZN.webp)

![](https://bbs.kanxue.com/upload/attach/202603/1068038_KKSMZESU3V3C35E.webp)

我们看看 deepseek 的分析，大家遇到这种混淆重的代码，结合结合 ai，哈哈

![](https://bbs.kanxue.com/upload/attach/202603/1068038_BPWPDSRGSD4FAQ3.webp)

所以说，这里是 jni 调用的 native 检测方法 ，在 java 极早期被调用，所以，现在我们要聚焦这里，我们要分析 68060 的源码

这里全局使用了大量大量的 off_E3290 函数，全部是隐式调用，调用这些结构体函数，发现异常，直接调用 66998 杀进程 

![](https://bbs.kanxue.com/upload/attach/202603/1068038_A8Q9Q36Y8HCJKSJ.webp)

![](https://bbs.kanxue.com/upload/attach/202603/1068038_ZHVN6EMW23Y2WZR.webp)

![](https://bbs.kanxue.com/upload/attach/202603/1068038_KA5HK3H2FF8EQU8.webp)

注册 v9 为 off_E3290 , 这里直接调用 v9 也就是结构体的偏移处函数，让 v12 接收，后续使用 

我们直接对 sub_68060ret0 试试

![](https://bbs.kanxue.com/upload/attach/202603/1068038_34BAXVRU8FEURNS.webp)

虽然还是死了，但是执行流明显改变了！！160 的死亡位变成了 60，说明是极早期调用 68060  通过不断调用检测函数 ，最后调用 66998 

![](https://bbs.kanxue.com/upload/attach/202603/1068038_K3FZG7APTT2K37N.webp)

那么我就试试直接把这个方法 ret 0 

但是这里对 sub_68060 ret0 之后居然还是调用了 66998，我认为这里的 66998 不是 sub_68060 调用的，我们说过，66254 通过动态注册 68060 让 jni 调用，那么我直接 ret0 可能触发了检测，我们看看打印看看本次 sub_66998 函数的调用链

![](https://bbs.kanxue.com/upload/attach/202603/1068038_PDHA65RKKJSH4SV.webp)

和之前不同    跳转看看属于哪个函数

![](https://bbs.kanxue.com/upload/attach/202603/1068038_HDZCZGKBY5JAT8Z.webp)

属于 69880 ，交叉引用发现被 66254 也就是刚刚分析的函数调用！！猜想正确！是 66254 注册时发现了异常，调用了 69880，从而再次调用 66998 死亡函数 ，导致进程死亡 

![](https://bbs.kanxue.com/upload/attach/202603/1068038_DNZJTK777VK8GDG.webp)

我想对这两个函数直接 ret0，发现崩溃了，java 报错

![](https://bbs.kanxue.com/upload/attach/202603/1068038_KKWJKPKM9YRKM39.webp)

看到了 classList 这里极有可能在脱壳，sub_69880 或者 sub_68060 包含业务脱壳分支 我把它业务分支一起杀了，这里我们还是动态入手

我们回退到分析 68060 ， 因为这里执行流很长，执行到了 160 层才调用的 66998 ，我们看看他的上几层函数有没有可能被这个 jni 函数调用，我猜测有可能是因为脱壳完最后进行了一层 jni 调用检测，所以最后几层函数应该全是 68060 通过结构体调用的 

![](https://bbs.kanxue.com/upload/attach/202603/1068038_BE9EJ2EDPE4JHZ7.webp)

由于 sub_68060 含有 非常多非常多的结构体调用，静态分析十分艰难，无法定位具体函数，我决定结合动态分析 

刚刚分析到了，66998 是由动态注册的 native 函数 68060 早期调用的 ，那么一定死亡末尾有触发的函数，我们可以找找  ，这里 75DC4 我们可以结合之前的分析，是一个字符串比较函数，我认为这里 hook 一定会有大量调用，但是我相信 68060 也会调用这个函数，可能是间接，但是会做校验，我们试试

![](https://bbs.kanxue.com/upload/attach/202603/1068038_VXM6RFP7YZJ6VTT.webp)

有大量调用 75DC4 的痕迹 ，我们跳转到最后一个，果然是 68060 调用的（第四个偏移） 看到是通过间接调用，我盲猜一手 68060 -> 检测函数 (56A04) 所属函数 -> 75DC4 -> 检测异常 68060 接收，调用 66998 杀进程 ，我们验证一下我们的猜想 ，跳转 56A04 所在函数

![](https://bbs.kanxue.com/upload/attach/202603/1068038_JVGFT56DBU6EBRF.webp)

是一个叫 55eb8 的超长检测函数，有 800 行 ，包含大量检测逻辑，看返回值，最后返回是一个 int64，我们交叉引用看看

![](https://bbs.kanxue.com/upload/attach/202603/1068038_A2ZDE59XY5GXC6A.webp)

第一层

![](https://bbs.kanxue.com/upload/attach/202603/1068038_NAB9R4QU3FVEFVM.webp)

第二层

![](https://bbs.kanxue.com/upload/attach/202603/1068038_FNSAMTXDK44EB9U.webp)

第三层！！函数表看到没有朋友们 ，第四层一定是注册到结构体！！

![](https://bbs.kanxue.com/upload/attach/202603/1068038_ESTBSX2AXY27EP5.webp)

猜想完全正确！！大家看到没有 E3290 

那么，看返回值，应该是如果 55eb8 检验到了异常，就直接异常（1 或 0） 我们直接 ret0 试试

![](https://bbs.kanxue.com/upload/attach/202603/1068038_26FYYGGEBRKHFJ5.webp)

成功过了 ， frida 没被杀，成功 ，lib_exec 被彻底解决 

我们还是要验证一下刚刚的猜想  刚刚是 ret0 ，成功了，我们 ret1 试试

![](https://bbs.kanxue.com/upload/attach/202603/1068038_AKA2N2Q3QSWSA75.webp)

可以看到，死亡了，我们猜想完全正确 ，这样一来，过这层 so 的检测就已经结束了

所以检测杀进程流程为 jni_onload -<sub_66254 -> 动态注册 jni native 方法 -> java 早期调用 -> sub_68060 -> 55eb8 检测到异常 -> sub_66998 杀进程

谢谢能看到这里的朋友，这个过程可以说是山路十八弯，我们经历了很多，最后定位到了核心检测函数，这里看似很通顺，是作者经历大量试错才有的文章，但是写到这里，是真的很开心，当所有证据链齐全，能验证自己猜想的感觉，是真的很舒服，大家一定不要怕试错，试的多，就经验多

如果对大家有帮助，麻烦点个关注，后续分享更多学习内容！

[[培训]Windows 内核深度攻防：从 Hook 技术到 Rootkit 实战！](https://www.kanxue.com/book-section_list-220.htm)

最后于 38 秒前 被 reserve_zhou 编辑 ，原因：
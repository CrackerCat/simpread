> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-291158.htm)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

近年来商业移动端 RASP 越来越卷。Garuda 系列从去年下半年开始在反作弊和金融风控圈子里频繁出现, 把代码混淆、完整性校验、环境检测、反 Hook、网络保护、远程策略这一堆东西全栈打包, 被很多团队当成新一代客户端防护的参考样本。

前阵子接到一个 GarudaDefender 的分析需求, 我用 Hermes Agent 当主驱动, 花了两天时间, 把它的告警链路从 Java 层 SystemAlert 弹窗一路追到 native 层 hash_crc 校验, 绕过了它的全部 Hook Detected / Malicious framework detected / Root Detected 检测。能拿下这种规模 (21MB 混淆 so、函数上万、双 SO 协作) 的商业 RASP 当然有意义。然而, 和我在此之后注意到的事情比起来, 这次绕过就不算什么了。

虽然不是重点, 作为背景还是先简单说一下。

GarudaDefender 的核心检测逻辑都在 `libkikypspro.so` 里, 通过 `libcrackme.so` 的自定义 dlsym 跳板间接调用。Java 层 `SystemAlert.onCreate` 是 native 方法, 真实业务在 `libkikypspro.so + 0x37A754`, 后续经过 `sub_68BD10` 调度, 最终由 `sub_64B840` 构造并显示 AlertDialog。

所有弹窗的源头, 是 `sub_AB31A0` 里的一处 hash_crc 一致性校验:

```
0xAB3C7C  BL hash_crc
0xAB3C80  CMP X23, X0       ; X23 = hash_A, X0 = hash_B
0xAB3C88  B.EQ loc_AB336C   ; 相等则继续,不等则进入告警链


```

把 `X23` 强制设为 `X0` 让校验恒等, 再把 14 个环境检测函数的返回值统一替换为 `0x64`(在该样本族中是 "通过" 状态码), 整个 RASP 就被静默了——主页面 root 检测正常, 弹窗不再出现, Frida 注入完全无感。

整个绕过过程并不简单。21MB 混淆 so、函数上万、间接调用一抓一大把, 反编译经常因为混淆给出错的伪代码, 光定位 SystemAlert 弹窗的真实入口按传统方法就可能耗掉一两天。但说到底, 这仍然是任何有经验的逆向工程师都能做到的事情, 谈不上有多新鲜。

让我决定把这次分析认真写成长文的, 是另一件事。

时间紧, 这次没按老办法走。我把 Hermes Agent 接进来当主驱动, 自己只负责出假设、审证据、定方向, 剩下查 xref、跑 decompile、写 Frida 脚本、对栈、改返回值这些事全部丢给 Agent 去做。

事情的转折发生在分析的第三天下午。

那天我在追 `sub_61482C` 的上游 caller, 原本计划手动反编译十几个候选函数, 根据形态判断哪个是真正的告警处理器。Hermes 跑完一轮工具调用之后, 在日志里多出了一行我没下达过的命令:

```
[skill_manage] action=create
  path = ~/.hermes/skills/reversing/locate-rasp-alert-entry/SKILL.md


```

——难道 Hermes 自己创建了一个逆向方法论的 Skill? 我打开那个文件:

```
---
name: locate-rasp-alert-entry
title: Locate Native Unified Alert Entry in Commercial RASP
description: Identify the native function that bridges all detection results into a Java alert dispatch
tags: [rasp, reverse-engineering, native, jni, alert-dispatch]
---


- 函数体 > 0x100,调用方数 >= 10
- 入口处对全局 mutex 调用 lock helper
- 参数形态为 (string title, string message, int type)
- 内部 BL 一个 JNI varargs 包装函数(从 vtable+0x1F0 取函数指针)


1. xref_query 定位 mutex 系列符号
2. func_query 按 size > 0x100 且 caller 数 >= 10 过滤候选
3. 对剩余候选 decompile,匹配上述参数形态
4. Frida hook 候选地址,验证是否打印告警标题/正文


```

我反复确认了对话历史——之前我从来没让 Hermes 总结方法论, 更没让它创建 Skill。它是自己做的。

后面几天我顺着这套机制观察下去。Hermes 一共自主沉淀了 7 个跟 GarudaDefender 系列分析直接相关的 Skill:

```
locate-rasp-alert-entry         定位 native 统一告警入口
identify-hashcrc-checkpoint     识别完整性校验比较点
enumerate-detection-task-table  枚举环境检测任务表
trace-thread-template-handoff   线程模板到真实任务的虚表交接
bypass-with-stable-status-code  通过状态码批量绕过
cross-validate-static-dynamic   静态 callsite 与动态 LR 交叉验证
hook-string-decryptor-slot      .data 段字符串解密函数指针 hook


```

每一个 Skill 都不是 "`sub_61482C` 在 0x61482C"这种一次性偏移结论, 而是" 如何在新版本中重新定位 `sub_61482C` 等价物 " 的可执行方法论。这意味着 GarudaDefender 下个版本 v4.5.x、v5.x 出来, 符号重新混淆, 本文里所有具体地址都会作废——但只要加载这 7 个 Skill,Hermes 会按方法论在新偏移上自动重新定位、自动生成新版本的 Frida 脚本。

得出这个结论后, 我非常震惊。但反复检查之后还是确认了: 分析开始之前, 这些 Skill 一个都不存在,`~/.hermes/skills/reversing/` 目录甚至是空的; 一切都是 Hermes 在分析过程中的自主行为。

也就是说, 没有人为提示, 没有预置的方法论模板, 没有外部审计——Hermes 在做完一次 RASP 逆向分析之后, 自己把这次分析的全套方法论提取了出来, 变成了下次面对同家族样本时可以直接复用的能力。

兴奋之余, 我开始想: 这一切是怎么发生的?

翻 Hermes 源码,`run_agent.py` 里有两个计数器:

```
self._turns_since_memory = 0    
self._iters_since_skill = 0     


```

每当工具调用次数超过阈值, Hermes 会 Fork 一个 Review Agent 重新审视整段对话, 把值得沉淀的经验写成 Memory 或 Skill。这套机制最初设计是给 Agent 积累用户偏好和工作流程用的——并不是为了逆向工程方法论沉淀。然而, 用户偏好是经验, 工作流程是经验, 逆向方法论也是经验。由于我反复多次按相似模式定位 `sub_61482C`、`sub_AB31A0`、`sub_81DDF0` 这一类函数, 机缘巧合之下, Hermes 忽然意识到 "这套查 mutex + 筛 caller + 形态匹配的流程值得沉淀", 于是自己调用 `skill_manage` 把它写成了 Skill。

读过玄武实验室那篇 Hermes Agent 自主防御网络攻击的文章的人应该会觉得眼熟——同一套 Review 机制, 同一个 `skill_manage` 工具。区别只在于: 他们触发的是 "安全免疫"——在反复遭受攻击后, Hermes 自己建立起了对网络攻击的免疫系统; 而我触发的是 "逆向方法论沉淀"——在反复执行同类逆向操作后, Hermes 自己把这些操作的共性抽取成了可复用的 Skill。

换个说法, Hermes 的这套 Review 机制本质上不只是一套安全免疫系统, 而是一套通用的**经验适应性沉淀系统**: 安全是经验, 逆向方法论是经验, 任何在对话里反复出现的高价值模式都可能被它识别、提取、固化。这一次, 我恰好把它用在了商业 RASP 对抗这个场景里, 于是看到了它在另一个方向上的涌现行为。

所以本文 1-19 章是这次跟 GarudaDefender 对抗的具体过程记录, 但真正值得关注的是第 20 章——那里会详细展开 Hermes 自主沉淀出来的 7 个 Skill 是什么样的、怎么在新变体上跑的、以及为什么这套 "Agent 自主方法论沉淀" 的范式可能改变整个商业 RASP 对抗的工作流。

我以为我在分析 GarudaDefender, 但其实是这次分析, 让我看到了 Agent 时代逆向工程的新形态。一个具体的绕过结果只对一个版本有效, 但一套被 Agent 自己内化的方法论, 只要 RASP 家族的行为模式没变, 就能持续生效。和这比起来, 绕过本身当然就不算什么了。

*   硬件 / 系统环境：Rock5B + redroid12 arm64
*   Frida：16.5.1
*   IDA：9.3 (IDA MCP 插件)
*   Hermes Agent
*   jadx 分析 java 代码
*   目标包名：`com.kikyps.crackme`
*   主要目标 so：
    *   `libcrackme.so`
    *   `libkikypspro.so`

本文章的分析过程使用 Hermes Agent 辅助记录分析日志、整理 IDA 静态分析结果，并配合 Frida 脚本进行动态验证。实验前需要先确认本地已经可以正常调用 `hermes`：

```
hermes version


```

本文章环境中使用的版本信息为：

```
Hermes Agent v0.11.0
Python: 3.11.15
OpenAI SDK: 2.32.0


```

可以使用下面命令检查 Hermes Agent 运行环境：

```
hermes doctor


```

如果是首次使用，先完成 Hermes Agent 初始化和认证配置：

```
hermes setup
hermes login


```

动态分析时，通过 Hermes Agent 执行 Frida 命令，由 Frida 连接目标设备并注入脚本。本文实际运行脚本的方式为：

```
frida -H 127.0.0.1:8888 -f com.kikyps.crackme -l ok.js


```

其中 `ok.js` 是本文最终使用的 Frida 脚本，负责等待 `libkikypspro.so` 加载、安装检测绕过点，并 hook `sub_61482C` 打印告警文案和 native 调用栈。

![](https://bbs.kanxue.com/upload/attach/202605/662006_NA7XNAXTVGZA3GW.png)

这些可见文本后来分别在 Frida 中通过 `TextView.setText` 捕获到：

```
Garuda Defender
Hook Detected!
Detected that the system has been hooked, or is running in an abnormal environment.
Oke
Oke (10) / Oke (9) / Oke (8)


```

其中 `Oke` 是检测弹窗确认按钮的初始文本，后续倒计时文本 `Oke (10)`、`Oke (9)` 等来自 Java 层 `AppTerminationTimer.onTick`。

最开始通过 hook Android 弹窗相关 API 和 `TextView.setText` 观察到弹窗文案：

```
Garuda Defender
Hook Detected!
Detected that the system has been hooked, or is running in an abnormal environment.
Oke


```

对应 Java 栈显示弹窗来自：

```
com.kikyps.kikypspro.SystemAlert.onCreate(Native Method)


```

堆栈片段：

```
at android.widget.TextView.setText(Native Method)
at com.kikyps.kikypspro.SystemAlert.onCreate(Native Method)
at android.app.Activity.performCreate(Activity.java:8050)
at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1329)
at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:3608)


```

`Dialog.show` 也显示同样的来源：

```
at android.app.Dialog.show(Native Method)
at com.kikyps.kikypspro.SystemAlert.onCreate(Native Method)
at android.app.Activity.performCreate(Activity.java:8050)


```

反编译 Java 类：

```
package com.kikyps.kikypspro;

import android.app.Activity;
import android.os.Bundle;

public class SystemAlert extends Activity {
    public static volatile boolean instance;

    @Override
    protected native void onCreate(Bundle bundle);

    @Override
    protected native void onDestroy();

    @Override
    protected native void onPause();
}


```

结论：`SystemAlert` 是承载弹窗的 Activity，但生命周期方法是 native 实现，真正逻辑不在 dex 里。

同时还观察到按钮倒计时来自：

```
com.kikyps.kikypspro.AppTerminationTimer.onTick


```

对应日志：

```
[TextView.setText] Oke (10)
at com.kikyps.kikypspro.AppTerminationTimer.onTick(Unknown Source:23)
at android.os.CountDownTimer$1.handleMessage(CountDownTimer.java:145)


```

因此弹窗显示本体和倒计时更新不是同一处逻辑：弹窗构造在 `SystemAlert.onCreate(Native Method)`，倒计时更新在 Java 层 `AppTerminationTimer.onTick`。

通过 hook `libart.so` 的 `RegisterNatives`，确认 `SystemAlert` 的 native 方法注册在 `libcrackme.so`：

```
[RegisterNatives] class = com.kikyps.kikypspro.SystemAlert, count = 3
  0: onCreate(Landroid/os/Bundle;)V -> libcrackme.so!0x1b15c
  1: onDestroy()V -> libcrackme.so!0x1b364
  2: onPause()V -> libcrackme.so!0x1b55c


```

同时还观察到：

```
[RegisterNatives] class = com.kikyps.kikypspro.KikyPS, count = 4
  0: myStr(Ljava/lang/String;)Ljava/lang/String; -> libcrackme.so!0x194c8
  1: fieldStr(Ljava/lang/String;)Ljava/lang/String; -> libcrackme.so!0x1b754
  2: isRoot()I -> libcrackme.so!0x1ad7c
  3: deviceInfo()Ljava/lang/String; -> libcrackme.so!0x1af6c


```

因此 `SystemAlert.onCreate` 的 JNI 注册入口是：

```
libcrackme.so + 0x1B15C


```

IDA 中查看 `libcrackme.so + 0x1B15C` 附近代码后，发现它不是最终业务函数，而是一个 wrapper。它会：

1.  初始化 / 缓存一个 handle。
2.  解密或构造符号名。
3.  调用 `sub_26F78` 解析真实函数地址。
4.  缓存函数指针。
5.  调用真实函数。

这类结构的特点是：`RegisterNatives` 注册到的地址不是最终业务逻辑，而是一个延迟解析跳板。第一次执行时解析真实函数地址，后续执行时直接调用缓存的函数指针。

动态 hook `libcrackme.so + 0x26F78` 后确认它是一个自定义符号解析器，行为类似 `dlsym`。关键日志：

```
[sub_26F78 resolver] symbol = _ZTSN5boost9garbage21Env08out_of_rangeE
[sub_26F78 resolver] leave retval = libkikypspro.so!0x37a754


```

对应关系：

```
libcrackme.so + 0x1B15C
  -> resolver
  -> libkikypspro.so + 0x37A754


```

在运行时日志中，`libcrackme.so + 0x1B15C` 的 wrapper 进入后，紧接着出现 resolver 解析：

```
[SystemAlert.onCreate wrapper] enter
  arg0 = JNIEnv*
  arg1 = SystemAlert jobject
  arg2 = Bundle

[sub_26F78 resolver] symbol = _ZTSN5boost9garbage21Env08out_of_rangeE
[sub_26F78 resolver] leave retval = libkikypspro.so!0x37a754


```

这说明 `libcrackme.so + 0x1B15C` 不是直接构造弹窗，而是把调用转发到 `libkikypspro.so + 0x37A754`。

因此，本文将 `SystemAlert.onCreate` 后续实际业务入口定位到：

```
libkikypspro.so + 0x37A754


```

另外还确认：

```
SystemAlert.onPause
  libcrackme.so + 0x1B55C
  -> libkikypspro.so + 0x37CF20

SystemAlert.onDestroy
  libcrackme.so + 0x1B364
  -> libkikypspro.so + 0x37C180


```

IDA 中查看 `libkikypspro.so + 0x37A754`，结合 JNI vtable 偏移可判断：

```
JNIEnv* = a1
jobject SystemAlert = a2
Bundle = a3


```

该函数里可见类似流程：

```
FindClass
GetMethodID
sub_37C0E8(...)
sub_68BD10(a1, a2)
DeleteLocalRef


```

动态 hook 后确认调用链：

```
[real SystemAlert.onCreate] enter
[sub_37C0E8 maybe super.onCreate] enter
[sub_68BD10 maybe build dialog] enter
[sub_64B840 candidate Hook Detected dialog] enter


```

`sub_37C0E8` 更像生命周期转发 / 调用父类方法；`sub_68BD10` 是弹窗构造的上层调度函数。

`sub_37C0E8` 在 `onCreate`、`onPause`、`onDestroy` 中都被调用过，且参数中包含从 `FindClass/GetMethodID` 得到的类和方法，因此它更接近 “调用 Activity 生命周期父类 / 目标方法” 的辅助函数，而不是弹窗本体。

IDA 中查看 `sub_68BD10`，结合 JNI vtable 偏移、动态日志和调用形态，可以把其中一批间接调用归类为 JNI 调用及字符串解密 / 初始化逻辑，包括：

```
GetObjectClass
GetMethodID
GetStaticFieldID
SetStaticBooleanField
NewStringUTF
GetStringUTFChars
CallObjectMethod
DeleteLocalRef


```

该函数最后会走到一个分支，调用：

```
libkikypspro.so + 0x64B840


```

在本轮运行时验证中，`sub_64B840` 的返回地址 `lr` 固定落在：

```
libkikypspro.so + 0x68CA48


```

结合静态反汇编可知，这是 `sub_68BD10` 内部调用 `sub_64B840` 之后的下一条指令。运行时记录：

```
[sub_64B840] enter
  lr = libkikypspro.so!0x68ca48


```

动态 hook 进一步确认：

```
[sub_68BD10 maybe build dialog] enter

[sub_64B840 candidate Hook Detected dialog] enter
  arg0 = JNIEnv*
  arg1 = SystemAlert jobject
  arg2 = ...
  arg3 = ...

[TextView.setText] ...
[Dialog.show] android.app.AlertDialog

[sub_64B840 candidate Hook Detected dialog] leave
[sub_68BD10 maybe build dialog] leave


```

结合静态 callsite、`TextView.setText` 和 `Dialog.show` 日志，可以确认：

```
libkikypspro.so + 0x64B840


```

是当前告警路径中实际构造并触发显示 `AlertDialog` 的核心函数。

`sub_64B840` 的前两个参数已通过 JNI 调用形态确认：

```
arg0 = JNIEnv*
arg1 = SystemAlert jobject


```

第 3、4 个参数是小整数，不是指针：

```
arg2 = 0xd5 / 0xd9 / 0xd1
arg3 = 0xe1 / 0xe5 / 0xe9


```

后续通过绑定 `TextView.setText` 和 `Dialog.show`，确认这两个参数会影响弹窗文案类型。

为了避免仅凭日志顺序猜测，后续使用脚本同时 hook：

```
libkikypspro.so + 0x64B840
TextView.setText
Dialog.show
JNIEnv->NewStringUTF


```

脚本逻辑是在 `sub_64B840` 进入时记录当前线程的 `arg2/arg3`，在同一线程内捕获 `TextView.setText` 和 `Dialog.show`，从而把参数与最终显示文案绑定。

确认结果如下。

```
arg2 = 0xd5 dec=213
arg3 = 0xe1 dec=225


```

同一次函数调用内捕获到：

```
Garuda Defender
Malicious framework detected
Illegal action detected!
Oke


```

映射：

```
0xD5 / 0xE1 -> Malicious framework detected / Illegal action detected!


```

```
arg2 = 0xd9 dec=217
arg3 = 0xe5 dec=229


```

同一次函数调用内捕获到：

```
Garuda Defender
Hook Detected!
Detected that the system has been hooked, or is running in an abnormal environment.
Oke


```

映射：

```
0xD9 / 0xE5 -> Hook Detected


```

```
arg2 = 0xd1 dec=209
arg3 = 0xe9 dec=233


```

同一次函数调用内捕获到：

```
Garuda Defender
Hook Detected!
Detected that the system has been hooked, or is running in an abnormal environment.
Oke


```

映射：

```
0xD1 / 0xE9 -> Hook Detected


```

因此目前可靠结论是：

```
0xD5 / 0xE1 = Malicious framework / Illegal action 弹窗
0xD9 / 0xE5 = Hook Detected 弹窗
0xD1 / 0xE9 = Hook Detected 弹窗


```

`0xD9/0xE5` 与 `0xD1/0xE9` 显示相同文案，可能表示不同检测来源共用同一套 Hook Detected 弹窗内容。这个 “检测来源不同” 的解释只是推测，当前日志只确认它们显示相同文案。

截图 `upload/attach/202605/662006_NA7XNAXTVGZA3GW.png` 对应的是 Hook Detected 类型弹窗，即运行时映射中的：

```
0xD9 / 0xE5 -> Hook Detected


```

或：

```
0xD1 / 0xE9 -> Hook Detected


```

仅凭截图本身不能区分是哪一组参数触发；需要依赖同一次 `sub_64B840` 调用内捕获到的参数和 `TextView.setText` 文案绑定日志。

完整链路如下：

```
Java Activity:
com.kikyps.kikypspro.SystemAlert.onCreate(Bundle)

RegisterNatives:
libcrackme.so + 0x1B15C

wrapper / resolver:
libcrackme.so + 0x26F78
  symbol: _ZTSN5boost9garbage21Env08out_of_rangeE

real native onCreate:
libkikypspro.so + 0x37A754

生命周期/父类调用辅助:
libkikypspro.so + 0x37C0E8

弹窗调度:
libkikypspro.so + 0x68BD10

实际构造并 show AlertDialog:
libkikypspro.so + 0x64B840


```

最终确认的关键函数：

```
libkikypspro.so + 0x64B840


```

它会设置弹窗标题、正文、按钮文案，并调用：

```
android.app.Dialog.show


```

最终弹出：

```
android.app.AlertDialog


```

动态 hook `JNIEnv CallVoidMethodV -> startActivity(Landroid/content/Intent;)V` 时，捕获到的关键 native 栈：

```
libkikypspro.so!0x38e96c
libkikypspro.so!0x68b1bc
libkikypspro.so!0x6148dc
libkikypspro.so!0x4ccf10
libkikypspro.so!0x3626b8


```

静态分析后，这 5 个地址分别落在：

```
0x38e96c -> sub_38E8F4
0x68b1bc -> sub_689370
0x6148dc -> sub_61482C
0x4ccf10 -> sub_4CCE88
0x3626b8 -> sub_36267C


```

`sub_38E8F4` 是 JNI 可变参数调用包装器。它从 `a1` 的虚表偏移 `0x1F0` 取函数指针，拷贝 `va_list`，然后执行：

```
vfunc(a1, a2, a3, va_list)


```

结合动态日志里 `GetMethodID startActivity(Landroid/content/Intent;)V` 和 `CallVoidMethodV`，这里基本可以归为 `JNIEnv->CallVoidMethodV` 这一层。`0x38e96c` 本身已经在 `BLR X8` 之后，是调用返回后的栈保护检查位置，不是业务触发点。

`sub_689370` 是这条链里最关键的 native 分发函数。`0x68b1bc` 附近逻辑如下：

```
0x68b1a8  X0 = X19
0x68b1ac  X1 = X22
0x68b1b0  X2 = X27
0x68b1b4  X3 = X26
0x68b1b8  BL sub_38E8F4
0x68b1bc  LDR X8, [X19]
0x68b1c4  LDR X8, [X8,#0x720]
0x68b1c8  BLR X8


```

也就是说 `0x68b1b8` 负责调用 JNI 包装器，触发 `startActivity`；`0x68b1bc` 是调用之后的对象状态 / 清理回调。结合动态日志中 `sub_64B840` 的 LR 为 `0x68ca48`，本文将 `sub_689370` 描述为 “弹窗 / startActivity 分发器”。

`sub_61482C` 是上层桥接 / 缓存函数。它持有全局互斥锁，使用 `qword_152BB00/qword_152BB08/qword_152BAF8` 维护状态；当状态满足时会调用：

```
sub_689370(v9, v11, qword_152BAF8, a1, a2)


```

`0x6148dc` 位于 `sub_689370` 返回之后，随后通过虚表偏移 `0xB8` 调用对象清理 / 释放逻辑。因此它能出现在栈上，但不是直接构造弹窗的位置。

`sub_4CCE88` 是 `basic_string::reserve` 风格的字符串扩容函数，负责检查容量、分配新 buffer、复制旧数据，失败路径会抛出：

```
basic_string::reserve max_size() exceeded


```

`0x4ccf10` 位于分配完成后的旧字符串状态读取位置。它出现在栈上，说明弹窗链路中正在构造字符串，例如类名、方法名、Intent 相关字符串或参数文本。

`sub_36267C` 是字符串赋值辅助函数。流程是：

```
strlen(src)
sub_4CCE88(dst, len)
memmove(dst, src, len)
dst[len] = 0
更新字符串长度


```

`0x3626b8` 位于调用 `sub_4CCE88` 后的分支位置。它同样属于字符串构造辅助层，不是弹窗触发点。

本轮结论：当前这条 `SystemAlert` 弹窗链路中，真正值得继续下钻的是：

```
sub_61482C -> sub_689370 -> sub_38E8F4 -> JNIEnv CallVoidMethodV(startActivity)


```

其中 `sub_689370` 是核心分发函数，`sub_61482C` 是上层状态 / 缓存桥接，`sub_38E8F4` 是 JNI 调用包装器；`sub_4CCE88` 和 `sub_36267C` 只是字符串基础设施。

动态日志里 `sub_64B840` 的 LR 是：

```
libkikypspro.so!0x68ca48


```

静态确认 `0x68ca48` 不属于 `sub_689370`，而是属于 `sub_68BD10`。`sub_68BD10` 起始地址：

```
sub_68BD10 @ 0x68BD10


```

`sub_64B840` 的直接调用点：

```
0x68ca34  MOV X0, X19
0x68ca38  MOV X1, X21
0x68ca3c  MOV X2, X26
0x68ca40  MOV X3, X27
0x68ca44  BL  sub_64B840
0x68ca48  LDR X8, [X19]


```

因此动态日志中的：

```
[sub_64B840 show alert dialog] enter
  arg2 = 0xd5 / 0xd9 / 0xd1
  arg3 = 0xe1 / 0xe5 / 0xe9
  lr   = libkikypspro.so!0x68ca48


```

可以和静态代码完全对上：

```
sub_68BD10(...):
    sub_64B840(x19, x21, x26, x27)


```

`sub_64B840` 返回后，`sub_68BD10` 会连续通过虚表偏移 `0xB8` 释放或清理若干 JNI / 对象引用：

```
0x68ca48  cleanup(x25)
0x68ca5c  cleanup(x26)
0x68ca70  cleanup(x27)
0x68ca84  cleanup(x24)
0x68ca98  cleanup(stack_var_48)
0x68caac  cleanup(x22)
0x68cac0  cleanup(x20)


```

这也解释了为什么 hook `sub_64B840` 的 LR 总是落在 `0x68ca48`：这是 `BL sub_64B840` 后的下一条指令。

动态日志中 `sub_68BD10 alert dispatcher` 的 LR 是：

```
libkikypspro.so!0x37bfa8


```

静态确认 `0x37bfa8` 属于导出 / 解析出来的混淆函数：

```
_ZTSN5boost9garbage21Env08out_of_rangeE


```

关键调用点：

```
0x37bf84  MOV X0, X19
0x37bf88  MOV X1, X20
0x37bf8c  MOV X2, X22
0x37bf90  MOV X3, X23
0x37bf94  MOV X4, X21
0x37bf98  BL  sub_37C0E8

0x37bf9c  MOV X0, X19
0x37bfa0  MOV X1, X20
0x37bfa4  BL  sub_68BD10
0x37bfa8  LDR X8, [X19]


```

所以这里的真实链路是：

```
_ZTSN5boost9garbage21Env08out_of_rangeE
  -> sub_37C0E8
  -> sub_68BD10
  -> sub_64B840


```

`sub_37C0E8` 更像生命周期 / 父类调用辅助，`sub_68BD10` 才是弹窗调度器，`sub_64B840` 是当前告警路径中实际构造并展示 AlertDialog 的函数。

目前静态和动态结合后，可以把弹窗相关逻辑拆成两段：

```
段 A：构造/展示弹窗内容

_ZTSN5boost9garbage21Env08out_of_rangeE
  -> sub_68BD10
  -> sub_64B840
  -> AlertDialog / Dialog.show


```

```
段 B：启动 SystemAlert Activity

sub_61482C
  -> sub_689370
  -> sub_38E8F4
  -> JNIEnv CallVoidMethodV(startActivity)
  -> com.kikyps.kikypspro.SystemAlert


```

这两段不是互相替代关系，而是同一弹窗流程的前后两部分：

1.  `startActivity` 把 `SystemAlert` 拉起来。
2.  `SystemAlert.onCreate` 的 native 注册函数进入 `_ZTSN5boost9garbage21Env08out_of_rangeE`。
3.  该函数继续调用 `sub_68BD10`。
4.  `sub_68BD10` 准备参数并调用 `sub_64B840`。
5.  `sub_64B840` 最终构造并显示 `AlertDialog`。

因此后续如果要定位 “是哪一个检测结果导致弹窗”，更应该继续追 `sub_61482C` 的调用者，特别是动态栈中两条分支：

```
0x36e4f0 -> 0xace1d0 -> 0xacab7c -> 0x4c874c -> 0x1095010
0x36b3b8 -> 0x6d237c -> 0x945054 -> 0x56bbbc -> 0xe0ec78


```

`sub_68BD10/sub_64B840` 已经偏向 “展示结果”，而不是 “产生检测结果”。

继续从 `sub_61482C` 的动态栈往上追，确认两条分支都不是简单的普通 `BL` 链，而是经过任务对象、队列和虚表回调进入检测处理函数。

动态栈：

```
0x36e4f0
0xace1d0
0xacab7c
0x4c874c
0x1095010


```

`0x36e4f0` 位于 `sub_36CE08`，但它不是调用点，而是调用 `sub_61482C` 后的清理逻辑。真正调用点是：

```
0x36e4e4  LDR W2, [SP,#var_188+4]
0x36e4e8  MOV X0, X19
0x36e4ec  BL  sub_61482C
0x36e4f0  cleanup...


```

因此 `sub_36CE08` 会把：

```
X0 = X19
X1 = 调用前已准备好的上下文
W2 = 栈上计算出的检测/告警 code


```

传给 `sub_61482C`。这里的 `W2` 后续会进入 `sub_61482C(a1, a2, a3)` 的第三个参数，并参与后面的 `sub_689370` / `SystemAlert` 启动流程。

`0xace1d0` 位于 `sub_ACE184`。这个函数很小，是一个虚表回调转发器：

```
v5 = *(a1 + 0x20)
call (*(v5)->vtable + 0x30)(v5, &a2, &a3, &a4, &a5)


```

也就是说 `sub_36CE08` 很可能是通过对象虚表 `+0x30` 间接调进来的。

`0xacab7c` 位于 `sub_AC7A90`。关键位置：

```
0xacab54  load qword_152DDE8 / s2
0xacab6c  MOV X0, X21
0xacab70  MOV X3, XZR
0xacab74  MOV X4, XZR
0xacab78  BL  sub_ACE184
0xacab7c  after call


```

这层负责准备参数并调用 `sub_ACE184`，属于上游检测调度 / 任务逻辑。它传入的 `X2` 来自全局字符串对象 `qword_152DDE8 / s2`。

`0x4c874c` 位于 `sub_4C871C`，这个函数更像任务入口：

```
v0 = sub_360B7C()
sub_1171030(*(v0+0x20), *(v0+0x18))
call (***(v0+8))(*(v0+8))


```

它从 `sub_360B7C()` 拿到一个全局 / 单例对象，然后调用该对象的虚函数，最终进入 `sub_AC7A90` 这一类任务执行逻辑。

`0x1095010` 位于 `sub_1094FB4`，它像线程任务 / 异步任务的收尾调度函数：

```
(***v1)(*v1)
v3 = *(v1 + 0xC0)
if (v3)
    (*(v3->vtable + 0x30))(v3, &arg)


```

它还维护原子计数、互斥锁、任务释放等状态。这个位置更偏调度框架，不像具体检测点。

分支 A 归纳：

```
sub_1094FB4
  -> sub_4C871C
  -> sub_AC7A90
  -> sub_ACE184
  -> vtable + 0x30
  -> sub_36CE08
  -> sub_61482C
  -> sub_689370
  -> startActivity(SystemAlert)


```

动态栈：

```
0x36b3b8
0x6d237c
0x945054
0x56bbbc
0xe0ec78


```

`0x36b3b8` 位于 `sub_369CA4`，同样不是调用点，而是 `sub_61482C` 返回后的清理位置。真实调用点：

```
0x36b3ac  LDR W2, [SP,#var_1C8+4]
0x36b3b0  MOV X0, X19
0x36b3b4  BL  sub_61482C
0x36b3b8  cleanup...


```

`sub_369CA4` 和 `sub_36CE08` 结构高度相似：都是大量混淆计算后，把 `X19` 和栈上计算出的 `W2` 提交给 `sub_61482C`。

`0x6d237c` 位于 `sub_6CEE14`。关键位置：

```
0x6d2368  X0 = *(X19 + 0x60)
0x6d2370  LDR X8, [X0]
0x6d2374  LDR X8, [X8,#0x30]
0x6d2378  BLR X8
0x6d237c  after call


```

这里也是虚表 `+0x30` 间接调用。结合动态栈，`sub_369CA4` 很可能就是这个 `BLR X8` 调到的处理函数。

`0x945054` 位于 `sub_94500C`，主要是状态置位和条件变量广播：

```
*(a1 + 0x28) = 1
pthread_cond_broadcast(a1 + 0x80)
遍历等待队列并 broadcast
call (*(a1)->vtable + 0x18)(a1, a2)


```

它更像异步任务 / 事件完成通知层。

`0x56bbbc` 位于 `sub_56BA1C`。它处理链表 / 队列节点：

```
从 a3 链表取节点
node[3] = 125
node[4] = &off_14C40B8
node[5] = 3
sub_56B4D4(*(a1+0x30), &list)


```

这层像任务队列搬运 / 封装层，会把待执行节点取出后交给下游。

`0xe0ec78` 位于 `sub_E0EBE8`。它是带锁的执行器：

```
if (*(a1+0x80))
    sub_5655EC(*(a1+0x30), a5, 0)
else {
    ok = sub_E0F420(a2, a3, a4, a5)
    sub_143AFF0(1, *(a1+0x30)+0xD0)
    if (ok)
        (*(a1->vtable + 0x28))(a1)
}


```

分支 B 归纳：

```
sub_E0EBE8
  -> sub_56BA1C
  -> sub_94500C
  -> sub_6CEE14
  -> vtable + 0x30
  -> sub_369CA4
  -> sub_61482C
  -> sub_689370
  -> startActivity(SystemAlert)


```

两条分支最终都收敛到：

```
sub_61482C


```

但上游意义不同：

```
sub_36CE08 / sub_369CA4


```

更像 “检测结果处理函数”，它们把检测上下文和 code 提交给统一告警桥。

```
sub_AC7A90 / sub_6CEE14 / sub_56BA1C / sub_E0EBE8


```

更像任务调度、队列执行、虚表派发框架。

如果要继续找 “具体检测了什么”，下一步应该在 `sub_36CE08` 和 `sub_369CA4` 内部继续看它们在调用 `sub_61482C` 前构造了哪些字符串 / 对象，尤其是提交给 `sub_61482C` 的 `X1` 和 `W2` 的来源。

`sub_AB31A0` 是一个很大的混淆函数：

```
sub_AB31A0 @ 0xAB31A0
size      = 0x4A3C
blocks    = 770
complexity= 376


```

它不像直接弹窗函数，也不像单个检测函数；更像 “任务 / 规则描述表初始化 + 字符串匹配 + 对象绑定” 的中间层。

入口处先检查 `a1` 的状态字段：

```
0xab31d8  LDRB W10, [X0,#0x80]
0xab31e0  CBZ  W10, loc_AB51EC

0xab31e4  LDRB W8, [X0,#0x88]
0xab31e8  CBZ  W8, loc_AB31F8
0xab31ec  LDP  W9, W8, [X0,#0x8C]
0xab31f0  CMP  W8, W9
0xab31f4  B.GE loc_AB5AD8


```

这说明 `a1` 是一个带状态、计数或容量字段的对象，`0x80/0x88/0x8C` 附近像任务表或条目表状态。

随后调用：

```
0xab3204  BL sub_AB2800


```

`sub_AB2800` 会从全局 `obj__2` 构造一个临时列表 / 容器：

```
sub_4BC64C(p_ptr, &obj__2)
sub_C7F844(...)


```

所以 `sub_AB31A0` 的第一步是拿到一批待处理条目。

之后它遍历这个临时表，每个条目大小是 `0x18`：

```
0xab3210  X27 = p_ptr
0xab3228  end = p_ptr + count * 0x18
...
0xab3284  X27 += 0x18


```

对每个条目，它会取字符串并计算长度：

```
0xab3290  LDRB W8, [X27]
0xab3294  LDR  X9, [X27,#0x10]
0xab329c  CSEL X0, X9, X27, EQ   ; short/long string data
0xab32a4  BL .strlen


```

然后用 `sub_DB877C / sub_DB8768 / sub_D9B120` 做字符串 / 对象封装：

```
sub_DB877C(...)
sub_DB8768(...)
sub_D9B120(...)


```

接着它遍历另一个全局对象表：

```
obj__3       @ 0x152DD50
qword_152DD58


```

全局表的每个外层条目大小也是 `0x18`：

```
0xab3314  LDR X8, [obj__3 + 8]      ; qword_152DD58/count
0xab331c  LDR X24, [obj__3]         ; obj__3/base
0xab332c  end = base + count * 0x18


```

匹配到外层条目后，又遍历内层表，内层条目大小是 `0x60`：

```
0xab3350  X25 = [var_170]
0xab3358  end = X25 + count * 0x60
0xab3370  X25 += 0x60


```

核心匹配逻辑是比较字符串长度和内容：

```
0xab33b0  CMP X13, X11       ; length compare
0xab33b4  B.NE next
0xab33e8  BL .memcmp         ; content compare
0xab33ec  CBNZ W0, next


```

匹配成功后进入对象绑定 / 构造：

```
0xab340c  X0 = a1 + 8
0xab3410  BL sub_ABCFD8

0xab3420  X0 = matched_outer_obj + 0x30
0xab342c  X2 = matched_inner_obj
0xab3430  BL sub_ABC9D0


```

`sub_ABCFD8` 的作用是按条件创建 / 获取一个对象节点，必要时分配 `0x50` 字节对象，并初始化其中的链表字段：

```
if (*a2)
    v = sub_ABC700()
else
    v = sub_ABC554()

if (need_alloc) {
    v = malloc(0x50)
    sub_379C00(v + 0x18, a3)
    初始化 v+0x30/v+0x38/v+0x40/v+0x48 链表字段
    sub_93019C()
    ++*a4
}


```

`sub_AB31A0` 后面有大量混淆分支，但结尾可以看到它会清理临时字符串表，并返回一个整型结果：

```
0xab7714  if temp_count != 0:
             遍历 p_ptr，每个 0x18 条目释放 string buffer
0xab7760  free(p_ptr)
0xab776c  W0 = saved_result
0xab77a0  RET


```

另外，异常 / 失败路径里会调用：

```
sub_4BC0DC
sub_4C5E60
sub_4BCCA4
sub_360B7C


```

这些更像容器断言、异常处理或全局状态恢复，不是正常主路径。

当前判断：

```
sub_AB31A0 = 规则/任务描述表匹配器 + 对象绑定器


```

它做的事情大致是：

1.  从 `obj__2` 构造一批输入字符串条目。
2.  遍历 `obj__3/qword_152DD58` 描述表。
3.  用长度和 `memcmp` 匹配字符串。
4.  匹配成功后调用 `sub_ABCFD8` / `sub_ABC9D0` 创建或绑定任务对象。
5.  清理临时字符串表并返回状态码。

它不是 `SystemAlert` 的直接触发函数，但可能参与 “检测任务 / 规则对象” 的初始化。如果某条检测分支最终通过虚表调到 `sub_36CE08` 或 `sub_369CA4`，`sub_AB31A0` 这类函数可能就是更早期负责把字符串规则和回调对象装配起来的地方。

`sub_ABC9D0` 是 `sub_AB31A0` 匹配成功后调用的对象绑定函数之一：

```
sub_AB31A0:
  0xab3420  X0 = matched_outer_obj + 0x30
  0xab3428  X1 = local container slot
  0xab342c  X2 = matched_inner_obj
  0xab3430  BL sub_ABC9D0


```

它不是检测函数，也不直接触发弹窗；它是一个 “按字符串 key 查找或插入节点” 的容器函数。

函数原型按寄存器实际用途更适合理解为：

```
sub_ABC9D0(out_pair, container, key_holder, key_string)


```

其中：

```
X8 = 返回结构地址 out_pair
X0 = container
X1 = key_holder / 查找状态
X2 = key_string


```

入口会先把容器指针调整到 `container + 8`：

```
0xabc9f8  MOV X20, X0
0xabc9fc  ADD X0, X0, #8
0xabca04  LDR X1, [X1]


```

如果 `*X1 != 0`，走 `sub_ABCD08`：

```
0xabca10  CBZ X1, loc_ABCA30
0xabca28  BL sub_ABCD08


```

如果 `*X1 == 0`，走 `sub_ABCB5C`：

```
0xabca30  ...
0xabca44  BL sub_ABCB5C


```

这两个函数都是树 / 有序容器查找逻辑，内部用字符串长度和 `memcmp` 比较 key：

```
sub_ABCB5C:
  compare string length
  memcmp(s1, s2, n)
  根据比较结果向左/右子节点走

sub_ABCD08:
  类似逻辑，但从已知节点附近继续查找/定位插入点


```

`sub_ABC9D0` 调完查找函数后，返回：

```
X0 = 找到的节点或插入位置
X1 = 是否需要新建节点


```

对应代码：

```
0xabca48  MOV X22, X0
0xabca4c  MOV X21, X1
0xabca50  TST W21, #0xFF
0xabca54  B.EQ loc_ABCB04


```

如果 `X21 == 0`，说明已经找到现有节点，直接返回：

```
0xabcb04  STR X22, [X19]
0xabcb08  STRB W21, [X19,#8]


```

如果 `X21 != 0`，说明没有找到，需要插入新节点。新节点大小是 `0x38`：

```
0xabca58  MOV W0, #0x38
0xabca5c  BL sub_14388B0


```

随后复制 key 字符串到新节点 `+0x18`：

```
0xabca64  ADD X0, X0, #0x18
0xabca68  MOV X1, X23
0xabca6c  BL sub_379C00


```

`sub_379C00` 是 string copy helper，会把 `X23` 指向的 short/long string 拷贝到目标字符串对象。

新节点结构大致可以按这个理解：

```
node + 0x00  parent/color/flag 混合字段
node + 0x08  left child
node + 0x10  right child
node + 0x18  key string
node + 0x30  extra/value/list ptr = 0


```

插入时会根据查找返回的方向，把新节点挂到父节点左 / 右：

```
0xabca90  STR X22, [X8,#8]     ; left child
...
0xabcaac  STR X22, [X8,#0x10]  ; right child
...
0xabcac0  root 初始化路径


```

随后初始化新节点指针关系，并调用平衡 / 修复函数：

```
0xabcadc  LDR X9, [X22]
0xabcae4  STP XZR, XZR, [X22,#8]
0xabcaf0  STR X8, [X22]
0xabcaf4  BL sub_93019C


```

最后增加容器节点计数：

```
0xabcaf8  LDR X8, [X20]
0xabcafc  ADD X8, X8, #1
0xabcb00  STR X8, [X20]


```

最终返回结构：

```
out_pair[0] = node
out_pair[8] = inserted_flag


```

对应：

```
0xabcb04  STR  X22, [X19]
0xabcb08  STRB W21, [X19,#8]


```

当前判断：

```
sub_ABC9D0 = std::map/set 风格的 find-or-emplace(key) 函数


```

和 `sub_ABCFD8` 的关系：

```
sub_ABCFD8:
  分配 0x50 字节节点
  节点里还初始化了额外链表字段

sub_ABC9D0:
  分配 0x38 字节节点
  主要保存 key string
  更像内层 key 表或轻量映射表


```

结合 `sub_AB31A0` 看，`sub_ABC9D0` 的作用是把匹配到的内层描述对象挂进某个容器，形成后续任务 / 规则对象的索引关系。它本身不做检测，只负责容器插入和对象绑定。

`sub_AB9EA8` 不是弹窗触发函数，也不是对象调度函数。它更像是一个高度优化的 64-bit 非加密 hash 函数，输入是：

```
uint64_t sub_AB9EA8(void *data, uint64_t len);


```

IDA 反编译原型显示为：

```
__int64 __fastcall sub_AB9EA8(int8x16_t *a1, unsigned __int64 n0x61)


```

其中 `a1` 是待 hash 的 buffer，`n0x61` 是长度。

函数内部按长度分多条路径处理：

```
len == 0          -> 返回固定常量 0x2D06800538D394C2
len < 4           -> 使用首字节/中间字节/尾字节和长度混合
4 <= len < 9      -> 读取首尾 4 字节
9 <= len <= 16    -> 读取首尾 8 字节
17 <= len <= 0x80 -> 使用 NEON 16 字节块混合
0x81..0xF0        -> 更多 NEON 块处理
len > 0xF0        -> 大 buffer 向量化循环，len >= 0x401 时按 0x400 chunk 处理


```

大长度路径会使用若干常量表和 NEON 寄存器做并行混合，例如 `xmmword_103900`、`xmmword_103C60`、`unk_170A40` 等。

最终收尾 avalanche 有典型 hash 混合形态：

```
h ^= h >> 37;
h *= 0x165667919E3779F9;
h ^= h >> 32;


```

小长度路径还会使用类似下面的乘法常量：

```
0x9FB21C651E98DF25
0xC2B2AE3D27D4EB4F
0x6782737BEA4239B9
0xAF56BC3B0996523A


```

整体风格类似 wyhash / XXH3 这一类非加密高速 hash，但目前不能直接断言就是某个公开算法的原版实现。

`sub_AB31A0` 中有一处很关键的调用：

```
0xAB3C70  LDR    X8, [X25,#0x10]
0xAB3C74  LDRSW  X1, [X25,#0x50]
0xAB3C78  ADD    X0, X0, X8
0xAB3C7C  BL     sub_AB9EA8
0xAB3C80  CMP    X23, X0


```

结合后续反编译代码，这里的 `sub_AB9EA8` 更准确地说是 `hash_crc` 校验函数。它会对两份来源的数据片段分别计算 hash/crc，然后比较结果：

```
if (p_ptr_2 == SHIDWORD(ptr_57)) {
    v49 = sub_DB8760(v341);
    p_ptr_2 = hash_crc(v49 + *((uint64_t *)ptr_18 + 1),
                       *((int *)ptr_18 + 20));

    ptr_57 = 0;
    sub_ABCFD8(&ptr_7, v328, &ptr_57, &ptr_16);

    ptr_57 = 0;
    sub_ABC9D0(&ptr_7, ptr_7 + 48, &ptr_57, ptr_18 + 24);

    *(uint64_t *)(ptr_7 + 48) = p_ptr_2;
}

v50 = sub_DB8774(v341);
p_ptr_1 = hash_crc(v50 + *((uint64_t *)ptr_18 + 2),
                   *((int *)ptr_18 + 20));

if (p_ptr_2 != p_ptr_1) {
    break;
}


```

这说明 `sub_AB31A0` 的核心逻辑不是简单缓存，而是做一致性校验：

```
来源 A: sub_DB8760(v341) + ptr_18[1] offset
来源 B: sub_DB8774(v341) + ptr_18[2] offset
长度:   *((int *)ptr_18 + 20)

hash_A = hash_crc(source_A + offset_A, len)
hash_B = hash_crc(source_B + offset_B, len)

hash_A != hash_B -> break，进入后续失败处理路径


```

其中 `sub_ABCFD8` / `sub_ABC9D0` 负责找到或创建对应节点，并把第一次计算出来的 `hash_A` 保存到 `node+0x30 / ptr_7+48`。

因此更准确的判断是：`sub_AB31A0` 内部存在一组基于 `hash_crc` 的完整性 / 一致性校验。如果两份数据算出的 hash 不一致，就会跳出当前循环，后续很可能进入异常处理或告警路径。结合前面已经确认的 `SystemAlert` 弹窗链，这个校验失败点很可能是弹窗触发条件之一。

另一条链里，`sub_AB2250` 会先计算 hash，再通过 `sub_ABC9D0` 找到或创建节点，最后把 hash 存到节点偏移 `+0x30`：

```
0xAB2574  BL     sub_AB9EA8
0xAB2578  MOV    X23, X0
...
0xAB25B8  BL     sub_ABC9D0
0xAB25C4  STR    X23, [X8,#0x30]


```

结合前面对 `sub_ABC9D0` 的分析，可以推测结构关系大概是：

```
node = find_or_emplace(container, key);
node->hash = sub_AB9EA8(data, len);


```

`sub_AB9EA8` 负责提供 “内容指纹 / CRC-like 校验值”。它本身不直接弹窗，但 `sub_AB31A0` 会用它比较两份数据是否一致。

和前面的函数串起来看：

```
sub_AB2250
  -> sub_AB9EA8       // 计算 data hash
  -> sub_ABC9D0       // 按 key 查找/创建节点
  -> node+0x30 = hash // 保存当前 hash

sub_AB31A0
  -> sub_DB8760       // 取来源 A base
  -> sub_AB9EA8       // 计算来源 A hash_crc
  -> sub_ABCFD8
  -> sub_ABC9D0       // 找到/创建节点并保存 hash
  -> sub_DB8774       // 取来源 B base
  -> sub_AB9EA8       // 计算来源 B hash_crc
  -> compare hash_A/hash_B
  -> 不一致时 break，进入失败处理路径


```

所以 `sub_AB9EA8` 可以命名为：

```
calc_content_hash64
hash_buffer64
rule_payload_hash64


```

如果后面要动态验证，可以 hook 这个函数打印：

```
data pointer
length
return hash
调用方 LR / backtrace


```

重点看它是否只在规则表初始化和规则刷新路径里出现。如果是，就能进一步确认它是规则对象的内容 hash，而不是检测逻辑本体。

`sub_AB31A0` 中这段反编译：

```
if (p_ptr_2 != p_ptr_1)
    break;


```

对应的关键汇编是：

```
0xAB3C68  BL      sub_DB8774
0xAB3C70  LDR     X8, [X25,#0x10]
0xAB3C74  LDRSW   X1, [X25,#0x50]
0xAB3C78  ADD     X0, X0, X8
0xAB3C7C  BL      hash_crc
0xAB3C80  CMP     X23, X0
0xAB3C84  MOV     X8, X23
0xAB3C88  B.EQ    loc_AB336C
0xAB3C8C  LDR     X8, [SP,#var_1B8]
0xAB3C90  LDR     X8, [X8,#0x70]
0xAB3C94  CBZ     X8, loc_AB3D9C


```

含义：

```
hash_A == hash_B -> 跳回 loc_AB336C，继续下一项循环
hash_A != hash_B -> 顺序落到 0xAB3C8C，进入失败/变更处理分支


```

不相等以后并不是立刻弹窗，而是先检查 `a1 + 0x70`：

```
callback_obj = *(a1 + 0x70);
if (!callback_obj)
    goto loc_AB3D9C;


```

如果 `callback_obj` 存在，后面会做第二层去重 / 缓存判断：

```
0xAB3CB0  BL      sub_ABD170
0xAB3CB4  CMP     X0, X24
0xAB3CB8  B.EQ    loc_AB3D00
...
0xAB3CF4  LDR     X8, [X8,#0x30]
0xAB3CF8  CMP     X8, X27
0xAB3CFC  B.EQ    loc_AB3D9C


```

也就是：

```
如果这次 hash_B 已经记录过 -> 跳到 loc_AB3D9C，不触发回调
如果没记录过，或者记录值不同 -> 进入 loc_AB3D00 更新记录


```

真正的回调调用点在：

```
0xAB3D74  LDR     X0, [X9,#0x70]
0xAB3D7C  CBZ     X0, loc_AB7804
0xAB3D80  LDR     X8, [X0]
0xAB3D84  LDR     X8, [X8,#0x30]
0xAB3D88  SUB     X1, X29, #-var_48
0xAB3D8C  ADD     X2, SP, #var_138
0xAB3D90  ADD     X3, SP, #var_148
0xAB3D94  ADD     X4, SP, #var_158
0xAB3D98  BLR     X8


```

伪代码可以写成：

```
if (hash_A != hash_B) {
    callback_obj = ctx->field_70;
    if (callback_obj) {
        old = find_saved_hash(...);

        if (!old || old->hash != hash_B) {
            node = find_or_create(...);
            node->hash = hash_B;

            callback_obj->vtable[0x30 / 8](
                callback_obj,
                &key_or_name,
                &desc_string,
                &hash_A_low32,
                &hash_B_low32
            );
        }
    }
}


```

所以这条分支的意义是：

```
hash_crc 校验失败
  -> 检查是否配置了回调对象 a1+0x70
  -> 检查该失败项是否已经处理过
  -> 没处理过则更新 hash 记录
  -> 调用 callback_obj 的 vtable+0x30


```

结合之前动态栈里多次出现的 `vtable+0x30 -> sub_36CE08/sub_369CA4 -> sub_61482C -> startActivity`，这里很可能就是把 “校验失败事件” 派发给上层处理器的地方。弹窗不是在 `CMP/B.NE` 当场发生，而是在这个虚函数回调后面的处理链里发生。

动态 hook `0xAB3C80` 得到：

```
[sub_AB31A0 hash_crc compare]
  hash_A/x23 = 0x608a672c064ca5cf
  hash_B/x0  = 0x73a08eafdb92622d
  equal      = false


```

说明 `sub_AB31A0` 内部确实出现了两份数据 hash_crc 不一致。

随后 hook `0xAB3D98 BLR X8` 得到：

```
[sub_AB31A0 callback BLR X8]
  x8  = libkikypspro.so!0xabc344
  rva = 0xabc344


```

动态验证进一步确认：`0xAB3D98` 的 `X8` 没有直接指向 `sub_36CE08` 或 `sub_369CA4`，而是先指向 `sub_ABC344`。

`sub_ABC344` 是一个桥接包装函数：

```
void sub_ABC344(ctx, key, desc, hash_A_low32, hash_B_low32)
{
    obj = *(ctx + 0x30);
    if (obj) {
        obj->vtable[0x30 / 8](
            obj,
            &key,
            &desc,
            &hash_A_low32,
            &hash_B_low32
        );
    }
}


```

关键汇编：

```
0xABC374  LDR     X0, [X0,#0x30]
0xABC380  CBZ     X0, loc_ABC3C0
0xABC384  LDR     X8, [X0]
0xABC398  LDR     X8, [X8,#0x30]
0xABC39C  BLR     X8


```

也就是说完整分发关系是：

```
sub_AB31A0
  -> hash_crc 比较失败
  -> 0xAB3D98 BLR X8
  -> sub_ABC344
  -> obj = *(ctx + 0x30)
  -> obj->vtable+0x30
  -> sub_36B564/sub_36CE08/sub_369CA4 等具体告警处理器


```

动态日志显示后面进入了 `sub_61482C`，并且参数就是弹窗文案：

```
[sub_61482C] enter
  a1 = "Hook Detected!"
  a2 = "Detected that the system has been hooked, or is running in an abnormal environment."
  a3 = 3
  lr = libkikypspro.so!0x36cc5c


```

随后进入 JNI `startActivity`：

```
sub_61482C
  -> sub_689370
  -> sub_38E8F4 CallVoidMethodV
  -> startActivity(Landroid/content/Intent;)V


```

因此现在可以确认：

```
sub_AB31A0 的 hash_crc 校验失败
  -> 经 sub_ABC344 桥接
  -> 进入告警处理器
  -> sub_61482C 接收并转发 Hook Detected / Malicious framework detected 文案
  -> startActivity(SystemAlert)


```

这里的弹窗不是 `sub_AB31A0` 直接创建的，而是 `sub_AB31A0` 产生校验失败事件，交给后面的虚函数回调链处理。

`sub_61482C` 可以命名为：

```
native_start_system_alert
dispatch_alert_to_java
show_system_alert_activity


```

它不是最早的检测函数，也不是 `SystemAlert.onCreate`。它是 native 层的告警分发 / 启动函数，负责把上游检测器传来的告警标题和正文继续传给 `sub_689370`，最终通过 JNI 调用 `Context.startActivity(Intent)` 拉起 `com.kikyps.kikypspro.SystemAlert`。

函数原型可整理为：

```
void sub_61482C(char *title, char *message, int level_or_type);


```

入口处保存参数：

```
0x61484C  MOV X19, X1      ; message
0x614854  MOV X20, X0      ; title
0x614864  MOV W21, W2      ; level/type


```

随后它会加锁并做状态检查：

```
0x614858  ADRL X1, mutex_
0x61486C  BL   sub_4C56F0
0x61488C  BL   sub_614484
0x614890  TBNZ W0,#0, loc_6148FC


```

如果全局告警上下文没有初始化，还会创建一批对象：

```
0x614898  CBZ  qword_152BB00, loc_61493C
...
0x614950  BL   malloc_like(0xA8)
0x614990  BL   sub_E0E6F4(..., sub_616590)


```

核心调用在 `0x6148D8`：

```
0x61489C  BL   sub_10923E0
0x6148B8  BL   sub_614758
0x6148C4  MOV  X0, X21
0x6148C8  MOV  X1, X22
0x6148CC  MOV  X2, X23
0x6148D0  MOV  X3, X20      ; title
0x6148D4  MOV  X4, X19      ; message
0x6148D8  BL   sub_689370


```

所以 `sub_61482C` 本身没有直接操作 Java `Intent`，而是把 `title/message` 传给下一层 `sub_689370`。

hook `sub_61482C` 后，运行时参数直接显示告警文案：

```
[sub_61482C] enter
  a1 = "Hook Detected!"
  a2 = "Detected that the system has been hooked, or is running in an abnormal environment."
  a3 = 3


```

另一组：

```
[sub_61482C] enter
  a1 = "Malicious framework detected"
  a2 = "Illegal action detected!"
  a3 = 3


```

这说明：

```
a1 = alert title
a2 = alert message
a3 = alert type / severity / code


```

同时动态栈显示它的上游来自不同告警处理器：

```
sub_AB31A0 hash_crc failed
  -> sub_ABC344
  -> sub_36B564 / sub_36CE08 / sub_369CA4
  -> sub_61482C


```

例如：

```
sub_61482C
  lr = libkikypspro.so!0x36cc5c


```

`0x36cc5c` 位于 `sub_36B564` 内部。

```
sub_61482C
  lr = libkikypspro.so!0x36e4f0


```

`0x36e4f0` 位于 `sub_36CE08` 内部。

```
sub_61482C
  lr = libkikypspro.so!0x36b3b8


```

`0x36b3b8` 位于 `sub_369CA4` 内部。

`sub_61482C` 调用 `sub_689370`：

```
sub_61482C
  -> sub_689370


```

动态日志里 `sub_689370` 收到的 `arg3/arg4` 正是 `sub_61482C` 的标题和正文：

```
[sub_689370] enter
  arg3 = "Hook Detected!"
  arg4 = "Detected that the system has been hooked, or is running in an abnormal environment."


```

随后：

```
sub_689370
  -> sub_38E8F4
  -> JNIEnv CallVoidMethodV
  -> startActivity(Landroid/content/Intent;)V
  -> com.kikyps.kikypspro.SystemAlert


```

结合动态参数和后续 `startActivity` 调用，可以把这条链路整理为：

```
sub_61482C 是当前样本中 native 层的统一告警入口。
上游检测器负责判断异常类型。
sub_61482C 负责接收 title/message/code。
sub_689370 负责把这些内容包装成 Java Intent 并调用 startActivity。


```

`sub_61482C` 的定位不是单独靠反编译猜出来的，而是通过动态栈、静态调用点和运行时参数三部分交叉验证出来的。

第一步，先从最终弹窗位置反推 native 调用栈。

hook JNI `CallVoidMethodV`，只在方法名为 `startActivity(Landroid/content/Intent;)V` 时打印 native backtrace，得到：

```
[sub_38E8F4 CallVoidMethodV wrapper] startActivity
  lr = libkikypspro.so!0x68b1bc

native backtrace:
  #0 libkikypspro.so!0x68b1bc
  #1 libkikypspro.so!0x6148dc
  #2 libkikypspro.so!0x36cc5c / 0x36e4f0 / 0x36b3b8
  ...


```

其中 `0x6148dc` 位于 `sub_61482C` 内部。这个地址非常关键，因为它不是函数入口，而是 `sub_61482C` 调用下一层函数返回后的地址。

第二步，静态确认 `0x6148dc` 的前一条调用。

反汇编 `sub_61482C` 可见：

```
0x6148C4  MOV X0, X21
0x6148C8  MOV X1, X22
0x6148CC  MOV X2, X23
0x6148D0  MOV X3, X20
0x6148D4  MOV X4, X19
0x6148D8  BL  sub_689370
0x6148DC  ...


```

因此动态栈里的 `0x6148dc` 可以准确解释为：

```
sub_61482C 调用了 sub_689370，当前栈回溯点落在调用返回地址 0x6148dc。


```

第三步，确认 `sub_61482C` 的参数含义。

入口处参数保存逻辑：

```
0x61484C  MOV X19, X1
0x614854  MOV X20, X0
0x614864  MOV W21, W2


```

调用 `sub_689370` 时：

```
0x6148D0  MOV X3, X20
0x6148D4  MOV X4, X19


```

说明：

```
sub_61482C arg0 -> sub_689370 arg3
sub_61482C arg1 -> sub_689370 arg4
sub_61482C arg2 -> 本地保存为 W21，作为告警类型/等级参与后续逻辑


```

动态 hook `sub_61482C` 后，运行时参数直接显示：

```
[sub_61482C] enter
  a1 = "Hook Detected!"
  a2 = "Detected that the system has been hooked, or is running in an abnormal environment."
  a3 = 0x3


```

另一条：

```
[sub_61482C] enter
  a1 = "Malicious framework detected"
  a2 = "Illegal action detected!"
  a3 = 0x3


```

所以 `sub_61482C` 的参数含义可以确定为：

```
void sub_61482C(char *title, char *message, int type);


```

第四步，确认下游确实启动了 Java 弹窗。

`sub_689370` 进入时收到同样的字符串：

```
[sub_689370] enter
  arg3 = "Hook Detected!"
  arg4 = "Detected that the system has been hooked, or is running in an abnormal environment."


```

随后 `sub_689370` 内部调用：

```
0x68B1B8  BL sub_38E8F4
0x68B1BC  ...


```

动态 hook `sub_38E8F4` 映射到 JNI 方法：

```
[sub_38E8F4 CallVoidMethodV wrapper] startActivity
  method = startActivity(Landroid/content/Intent;)V
  lr = libkikypspro.so!0x68b1bc


```

Java 层 hook 也确认最终 Intent 指向：

```
Intent {
  cmp=com.kikyps.crackme/com.kikyps.kikypspro.SystemAlert
}


```

所以 `sub_61482C -> sub_689370 -> sub_38E8F4 -> startActivity(SystemAlert)` 这条下游链闭合。

第五步，确认上游来源。

`sub_61482C` 有多条上游告警处理器调用路径：

```
sub_36B564 + 0x16F8  -> sub_61482C
sub_36CE08 + 0x16E8  -> sub_61482C
sub_369CA4 + 0x1714  -> sub_61482C


```

动态日志中分别表现为：

```
lr = libkikypspro.so!0x36cc5c
lr = libkikypspro.so!0x36e4f0
lr = libkikypspro.so!0x36b3b8


```

其中 `sub_AB31A0` 的 hash_crc 失败路径已经验证到：

```
sub_AB31A0 hash_crc compare
  equal = false

sub_AB31A0 callback BLR X8
  x8 = sub_ABC344

sub_ABC344
  -> obj->vtable+0x30
  -> 告警处理器
  -> sub_61482C


```

因此完整闭环为：

```
sub_AB31A0 做 hash_crc 一致性校验
  -> hash_A != hash_B
  -> sub_ABC344 桥接回调
  -> sub_36B564 / sub_36CE08 / sub_369CA4 告警处理器
  -> sub_61482C(title, message, type)
  -> sub_689370(env, context, global, title, message)
  -> sub_38E8F4
  -> JNIEnv->CallVoidMethodV(startActivity)
  -> SystemAlert


```

所以 `sub_61482C` 的最终定位是：

```
native 层统一告警分发入口。
它接收已经确定的告警标题、正文和类型，负责把告警事件送入 JNI 启动 Activity 的下游链路。


```

`sub_689370` 是 native 到 Java 的告警启动桥接函数。它的作用是：

```
接收 JNIEnv / Context / 全局 Java 对象 / title / message
  -> 查找 Java 类和方法
  -> 构造 Intent / 设置参数
  -> 调用 Context.startActivity(Intent)


```

可以命名为：

```
build_and_start_system_alert_intent
jni_start_system_alert
start_alert_activity_jni


```

`sub_61482C` 调用它的位置：

```
0x6148C4  MOV X0, X21
0x6148C8  MOV X1, X22
0x6148CC  MOV X2, X23
0x6148D0  MOV X3, X20      ; title
0x6148D4  MOV X4, X19      ; message
0x6148D8  BL  sub_689370


```

动态日志对应：

```
[sub_689370] enter
  arg0 = JNIEnv*
  arg1 = jobject / Context
  arg2 = global helper/class/method holder
  arg3 = "Hook Detected!"
  arg4 = "Detected that the system has been hooked, or is running in an abnormal environment."


```

所以参数可以整理为：

```
void sub_689370(JNIEnv *env, jobject context, void *global, char *title, char *message);


```

函数内部大量通过 `JNIEnv` 函数表调用 JNI API。

开头检查参数后，先通过 `env->GetObjectClass(context)` 类似的调用获取对象类：

```
0x6893C0  LDR X8, [X0]
0x6893CC  LDR X8, [X8,#0xF8]
0x6893D0  BLR X8


```

随后多次检查 JNI 异常：

```
0x6893E4  LDR X8, [X8,#0x720]
0x6893E8  BLR X8


```

释放 local ref 的清理逻辑集中在：

```
0x689444  LDR X1, [SP,#var_68]
0x689448  LDR X8, [X8,#0xB8]
0x68944C  BLR X8


```

`0x108` 偏移用于 `GetMethodID` 一类方法查找：

```
0x6896B0  LDR X8, [X19]
0x6896C4  LDR X8, [X8,#0x108]
0x6896C8  BLR X8


```

最终关键调用：

```
0x68B1A8  MOV X0, X19
0x68B1AC  MOV X1, X22
0x68B1B0  MOV X2, X27
0x68B1B4  MOV X3, X26
0x68B1B8  BL  sub_38E8F4
0x68B1BC  ...


```

`sub_38E8F4` 是 JNI varargs 包装：

```
v3 = *(env->functions + 0x1F0);
v3(env, obj, method, va_list);


```

动态 hook 已经把它映射成：

```
CallVoidMethodV startActivity(Landroid/content/Intent;)V


```

动态栈里：

```
[sub_38E8F4 CallVoidMethodV wrapper] startActivity
  lr = libkikypspro.so!0x68b1bc


```

`0x68B1BC` 正好是：

```
0x68B1B8  BL sub_38E8F4
0x68B1BC  ...


```

也就是 `sub_689370` 调用了 `CallVoidMethodV(startActivity)`。

同时 `sub_689370` 的参数保留了上游告警文本：

```
[sub_689370] enter
  arg3 = "Hook Detected!"
  arg4 = "Detected that the system has been hooked, or is running in an abnormal environment."


```

所以完整关系是：

```
sub_61482C(title, message, type)
  -> sub_689370(env, context, global, title, message)
  -> JNI GetObjectClass / FindClass / GetMethodID
  -> 构造 Intent / 设置 SystemAlert 参数
  -> sub_38E8F4
  -> JNIEnv->CallVoidMethodV(context, startActivity, intent)


```

`sub_689370` 不负责检测，也不负责生成告警文案。告警文案在进入它之前已经确定。

它负责的是：

```
native 告警事件 -> Java Activity 启动


```

因此它是弹窗链路里最关键的 JNI 桥接层。

`ok.js` 当前承担两个目标：

```
1. 等 libcrackme.so 自实现 dlopen 加载 libkikypspro.so 后，再对 libkikypspro.so 安装 hook。
2. 通过绕过关键检测结果，让程序继续运行，同时 hook sub_61482C 打印告警文案和调用堆栈。


```

这份脚本的核心不是一开始就直接 hook `libkikypspro.so`。原因是 `libkikypspro.so` 并不是普通系统 loader 直接加载的，而是由 `libcrackme.so` 内部的 `sub_26538` 自实现 dlopen-like wrapper 加载。

因此脚本先 hook 系统 `android_dlopen_ext`，等 `libcrackme.so` 加载完成：

```
function dlopen() {
    Interceptor.attach(Module.getExportByName(null, "android_dlopen_ext"), {
        onEnter: function (args) {
            this.path = readStr(args[0]);
        },
        onLeave: function (retval) {
            if (this.path.indexOf("libcrackme.so") >= 0) {
                console.log("[dlopen] loaded " + this.path);
                hookSub26538();
            }
        }
    });
}


```

然后再 hook `libcrackme.so + 0x26538`：

```
var target = crackme.base.add(0x26538);
Interceptor.attach(target, {
    onEnter: function (args) {
        this.path = readStr(args[0]);
        this.flags = args[1].toUInt32();
    },
    onLeave: function (retval) {
        if (this.path.indexOf("libkikypspro.so") >= 0 && !retval.isNull()) {
            var pro = Process.findModuleByName("libkikypspro.so");
            ...
        }
    }
});


```

这一步的意义是：

```
sub_26538("/.../libkikypspro.so", 2) 返回非空
  -> libkikypspro.so 已经映射进进程
  -> Frida 可以通过 Process.findModuleByName 找到 base
  -> 后续所有 libkikypspro.so + offset hook 才是有效地址


```

动态日志已经验证：

```
[sub_26538]
  path  = /data/app/.../lib/arm64/libkikypspro.so
  flags = 0x2
  ret   = 0xb400fee882a4f850

[libkikypspro.so]
  base = 0x...
  path = /data/app/.../lib/arm64/libkikypspro.so


```

也就是说，`sub_26538` 是 `libcrackme.so` 到 `libkikypspro.so` 的加载交接点。

如果脚本过早 hook `libkikypspro.so + 0x61482C`，模块还不存在，地址无法计算；如果等 `sub_26538` 返回后再 hook，就能稳定拿到：

```
var base = pro.base;
hook_message(base);
hook_crc(base);
sub_6CEE14(base);
hook_AEB528(base);
hook_AC7A90(base);


```

脚本里真正参与当前绕过的点主要有四组。

第一组，`hook_crc(base)`：

```
function hook_crc(base) {
    Interceptor.attach(base.add(0xAB3C80), {
        onEnter: function (args) {
            this.context.x23 = this.context.x0;
        }
    })
}


```

`0xAB3C80` 是 `sub_AB31A0` 中比较两份 hash_crc 的位置：

```
0xAB3C7C  BL  hash_crc
0xAB3C80  CMP X23, X0
0xAB3C88  B.EQ loc_AB336C


```

寄存器含义：

```
X23 = hash_A
X0  = hash_B


```

脚本在比较前执行：

```
this.context.x23 = this.context.x0;


```

等价于强制：

```
hash_A = hash_B


```

这样 `CMP X23, X0` 一定相等，随后走：

```
B.EQ loc_AB336C


```

也就是跳回正常循环，不进入：

```
0xAB3C8C hash mismatch handler
  -> sub_ABC344
  -> 告警处理器
  -> sub_61482C


```

这就是绕过 `sub_AB31A0` hash_crc 校验触发弹窗的关键点。

第二组，`sub_6CEE14(base)` 下的返回值替换：

```
hook_6D3C44(base)  -> retval.replace(0xF5)
hooK_74DE1C(base) -> retval.replace(0xF5)
hook_7503C0(base) -> retval.replace(0xF5)
hook_756F0C(base) -> retval.replace(0xF4)
hook_75B1D0(base) -> retval.replace(0xF4)


```

这些 hook 对应的是 `sub_6CEE14 / sub_369CA4` 这一类检测分支的下游判断函数。动态栈里这一支会进入：

```
sub_369CA4
  -> sub_61482C("Malicious framework detected", "Illegal action detected!", 3)


```

脚本把这些检测函数返回值改成期望的正常值，避免进入恶意框架告警路径。

第三组，`hook_AEB528(base)`：

```
hook_AF0098(base) -> retval.replace(0xC2)


```

这属于另一条检测分支的返回值修正。它不是弹窗启动函数本身，而是让上游检测状态保持在 “通过 / 正常” 的值。

第四组，`hook_AC7A90(base)`：

```
hook_11442C0(base) -> retval.replace(0xF4)


```

这一组对应动态栈里的另一条告警来源：

```
sub_AC7A90
  -> sub_ACE184
  -> sub_36CE08
  -> sub_61482C("Hook Detected", ...)


```

通过改写返回值，脚本阻断这条检测分支继续产生告警事件。

脚本中的：

```
function hook_message(base) {
    Interceptor.attach(base.add(0x61482C), {
        onEnter: function (args) {
            console.log("\n[sub_61482C]");
            console.log("message = " + args[1].readCString());

            console.log(
                Thread.backtrace(this.context, Backtracer.FUZZY)
                    .map(DebugSymbol.fromAddress)
                    .join("\n")
            );
        }
    });
}


```

`sub_61482C` 已经静态和动态确认是：

```
void sub_61482C(char *title, char *message, int type);


```

所以 hook 它可以直接看到：

```
message = "Detected that the system has been hooked, or is running in an abnormal environment."
message = "Illegal action detected!"


```

同时 backtrace 能看到是谁调用了 `sub_61482C`：

```
sub_36CE08 / sub_369CA4 / sub_36B564
  -> sub_61482C


```

这就是 “通过堆栈看谁调用了 `0x61482C`” 的方法。

对应的动态堆栈截图：

![](https://bbs.kanxue.com/upload/attach/202605/662006_33U3SD9RY5W6UW9.png)

`sub_61482C` 本身不直接调用 Java。它会把告警文案传给 `sub_689370`：

```
0x6148D0  MOV X3, X20      ; title
0x6148D4  MOV X4, X19      ; message
0x6148D8  BL  sub_689370
0x6148DC  ...


```

动态栈里 `0x6148dc` 正是 `sub_61482C` 调用 `sub_689370` 后的返回地址：

```
sub_38E8F4 CallVoidMethodV startActivity
  #0 libkikypspro.so!0x68b1bc
  #1 libkikypspro.so!0x6148dc


```

`sub_689370` 是 JNI 桥接函数，会构造 / 查找 Java 对象和方法：

```
GetObjectClass(context)
FindClass("android/content/Intent")
GetMethodID(...)
CallVoidMethodV(startActivity)


```

最终调用：

```
Context.startActivity(Intent)


```

Java 层 hook 进一步确认 Intent 指向：

```
cmp=com.kikyps.crackme/com.kikyps.kikypspro.SystemAlert


```

实际弹窗效果如下，标题和正文与 `sub_61482C` 中打印到的 `title/message` 一致：

![](https://bbs.kanxue.com/upload/attach/202605/662006_NA7XNAXTVGZA3GW.png)

因此完整闭环是：

```
检测函数返回异常 / hash_crc 校验失败
  -> 告警处理器 sub_36B564 / sub_36CE08 / sub_369CA4
  -> sub_61482C(title, message, type)
  -> sub_689370(env, context, global, title, message)
  -> sub_38E8F4
  -> JNIEnv->CallVoidMethodV(startActivity)
  -> Context.startActivity(Intent)
  -> com.kikyps.kikypspro.SystemAlert


```

脚本中的绕过点则对应闭环的上游：

```
hook_crc(base)
  -> 阻断 sub_AB31A0 hash_crc mismatch

sub_6CEE14(base)
hook_AEB528(base)
hook_AC7A90(base)
  -> 修正其它检测分支返回值

hook_message(base)
  -> 观察是否仍然进入 sub_61482C，以及是谁调用它

hook_str(base)
  -> hook 字符串解密/还原函数，观察检测规则和特征字符串


```

当前 `ok.js` 已经去掉了前期用于线程、SVC 和大函数跟踪的历史调试代码，只保留和本文闭环直接相关的逻辑。

脚本入口是：

```
dlopen()


```

`dlopen()` 等 `libcrackme.so` 加载完成后，调用：

```
hookSub26538()


```

`hookSub26538()` 再等待 `libcrackme.so + 0x26538` 加载 `libkikypspro.so` 成功。`libkikypspro.so` 可枚举后，安装下面这些 hook：

```
hook_message(base)  // 打印 sub_61482C 告警文案和调用栈
hook_crc(base)      // 绕过 sub_AB31A0 hash_crc mismatch
sub_6CEE14(base)    // 修正 sub_6CEE14 相关检测返回值
hook_AEB528(base)   // 修正 sub_AEB528 相关检测返回值
hook_AC7A90(base)   // 修正 sub_AC7A90 相关检测返回值
hook_str(base)      // 打印字符串解密/还原结果


```

因此当前脚本可以分为三类：

```
加载时机控制：dlopen(), hookSub26538()
绕过逻辑：hook_crc(), sub_6CEE14(), hook_AEB528(), hook_AC7A90()
证据采集：hook_message(), hook_str()


```

`ok.js` 里还加入了 `hook_str(base)`，用于跟踪运行时字符串解密 / 还原结果：

```
function hook_str(base){
    var ptr_str = ptr(base.add(0x15264F0)).readPointer();
    Interceptor.attach(ptr_str,{
        onEnter:function(args){
            this.a = args[0];
            this.lr = this.context.lr.sub(base);
        },
        onLeave:function(retval){
            console.log("hook_str:", this.a.readCString(), "lr:", this.lr)
        }
    })
}


```

这里 `base + 0x15264F0` 保存的是字符串处理函数指针。hook 这个函数后，可以直接看到运行时被还原出来的检测规则和特征字符串。

典型日志：

```
hook_str: exact renef payload path
hook_str: rustfrida agent memfd
hook_str: rustfrida named VMA artifact
hook_str: rustfrida trace VMA artifact
hook_str: rustfrida memfd artifact
hook_str: rustfrida anonymous exec artifact
hook_str: hidden executable mapping absent from dl_iterate_phdr
hook_str: qbdi trace bundle artifact
hook_str: qbdi dynamic trace artifact
hook_str: /proc/self/smaps
hook_str: wxshadow_probe
hook_str: /proc/self/maps
hook_str: /data/local/tmp/libagent.so


```

这批字符串非常有价值，因为它直接暴露了 Garuda Defender 的检测关注点：

```
RustFrida agent / memfd / VMA artifact
QBDI trace artifact
/proc/self/maps 和 /proc/self/smaps 内存映射扫描
隐藏 executable mapping
/data/local/tmp/libagent.so 这类常见注入路径


```

也就是说，动态 hook 字符串解密函数可以快速枚举检测规则，比单纯静态搜索字符串更直接。很多字符串只有运行时解密后才会出现，直接 hook 解密出口能帮助定位后续检测函数和绕过点。

前面的弹窗链路主要解决了 “告警如何从 native 进入 Java 并展示出来” 的问题。后续继续分析主页面的 root 检测时，入口不是从 `SystemAlert` 开始，而是从 native 线程创建开始追。

#### 18.8.1 从 pthread_create 定位线程模板

先 hook `libc.so` 的 `pthread_create`，观察 `libkikypspro.so` 创建的检测线程。运行时多次捕获到同一组信息：

```
[+] 检测到线程创建！
    来源模块: libkikypspro.so
    调用地址: 0x11aaf10 (偏移)
    执行函数偏移地址: 0x11aafbc


```

静态查看 `0x11AAF0C` 附近：

```
0x11AAF00  ADR X2, 0x11AAFBC      ; start_routine
0x11AAF04  MOV X1, XZR            ; attr
0x11AAF08  ADD X0, X3, #0x28      ; pthread_t*
0x11AAF0C  BL  pthread_create
0x11AAF10  MOV W20, W0


```

也就是说，`pthread_create` 传入的线程入口是 `libkikypspro.so + 0x11AAFBC`。

但是 `0x11AAFBC` 并不是具体检测函数，而是一个通用线程模板函数。它会从线程任务对象的 vtable 中取出真实执行函数：

```
0x11AAFE8  LDR X0, [SP,#pointer]
0x11AAFEC  LDR X8, [X0]
0x11AAFF0  LDR X8, [X8,#0x10]
0x11AAFF4  BLR X8


```

对 `0x11AAFF4` 前的 `X8` 做动态打印并按 RVA 去重，得到多个真实任务入口：

```
x8 = 0x4bc99c
x8 = 0x4c5f4c
x8 = 0x1095af8
x8 = 0x11712a4
x8 = 0xc3b764
x8 = 0x5578fc
x8 = 0x5a39dc
x8 = 0xba4920
x8 = 0x911970
x8 = 0x9125c0
x8 = 0xb8f6a4


```

其中和 root 检测相关的关键入口是：

```
x8 = libkikypspro.so + 0x911970


```

静态查看 `0x911970`：

```
0x911970  STP X29, X30, [SP,#-0x10]!
0x911978  BL  sub_81DDF0
0x911980  B   sub_81A128


```

因此这条环境检测线程链路可以先闭合到：

```
pthread_create
  -> 0x11AAFBC 线程模板入口
  -> vtable + 0x10
  -> X8 = 0x911970
  -> sub_911970
  -> sub_81DDF0


```

这里需要注意，`0x9125C0` 这类地址不一定落在 IDA 识别的函数入口上。它位于 `sub_9125BC + 4`，说明样本中有些任务入口可能会指向函数中间位置。因此这类链路应以动态记录到的 `X8` 为准，不能只依赖 IDA 自动识别的函数起始地址。

#### 18.8.2 sub_81DDF0 注册 14 个环境检测任务

继续查看 `sub_81DDF0`，可以看到它内部初始化了一张检测任务表。每个检测任务由两部分组成：检测对象 / 上下文生成函数，以及真正的检测执行函数。

关键伪代码结构如下：

```
v166[0]  = sub_821570(sub_8214B0());
v166[1]  = sub_82162C;

v166[2]  = sub_826BA4(sub_826AE4());
v166[3]  = sub_826C60;

v166[4]  = sub_827C3C(sub_827B78());
v166[5]  = sub_827CF8;

v166[6]  = sub_828C44(sub_828BB8());
v166[7]  = sub_828D00;

v166[8]  = sub_829C44(sub_829BC0());
v166[9]  = sub_829D00;

v166[10] = sub_82AC4C(sub_82ABC8());
v166[11] = sub_82AD08;

v166[12] = sub_82BCDC(sub_82BC18());
v166[13] = sub_82BD98;

v166[14] = sub_82FD1C(sub_82FC58());
v166[15] = sub_82FDD8;

v166[16] = sub_8328D4(sub_832814());
v166[17] = sub_832990;

v166[18] = sub_8362B4(sub_8361F4());
v166[19] = sub_836370;

v166[20] = sub_83B6B8(sub_83B600());
v166[21] = sub_83B774;

v166[22] = sub_847B6C(sub_847AA8());
v166[23] = sub_847C28;

v166[24] = sub_849C18(sub_849B58());
v166[25] = sub_849CD4;

v166[26] = sub_84F954(sub_84F8C8());
v166[27] = sub_84FA10;

sub_867F80(v166, 14);


```

`sub_867F80(v166, 14)` 的第二个参数明确说明这里注册了 14 个检测任务。对应的检测执行函数为：

```
0x82162C
0x826C60
0x827CF8
0x828D00
0x829D00
0x82AD08
0x82BD98
0x82FDD8
0x832990
0x836370
0x83B774
0x847C28
0x849CD4
0x84FA10


```

这里暂时称为 “环境检测任务”，而不是全部归类为 root 检测。因为这 14 个任务可能同时覆盖 root、Hook、注入、虚拟环境、异常文件路径、运行时特征等多种检测面。

#### 18.8.3 14 个检测任务的返回值验证

对这 14 个检测函数逐个 hook 返回值，当前环境下得到：

```
0x82162C retval: 0x64
0x826C60 retval: 0x697
0x827CF8 retval: 0x64
0x828D00 retval: 0x64
0x829D00 retval: 0x64
0x82AD08 retval: 0x64
0x82BD98 retval: 0x413
0x82FDD8 retval: 0x64
0x832990 retval: 0x64
0x836370 retval: 0x64
0x83B774 retval: 0x25c
0x847C28 retval: 0x64
0x849CD4 retval: 0x64
0x84FA10 retval: 0x64


```

可以看到，大多数检测任务返回 `0x64`，而当前环境中有三个检测项返回了非 `0x64`：

```
0x826C60 -> 0x697
0x82BD98 -> 0x413
0x83B774 -> 0x25c


```

动态验证时，如果不修改这些返回值，App 主页面会显示 root 检测异常；如果将异常返回值替换为 `0x64`，主页面显示恢复正常。因此可以判断，在当前样本中 `0x64` 是可验证的通过状态码，非 `0x64` 返回值会被上层聚合逻辑视为检测命中或环境异常。

用于验证的 hook 逻辑可以简化为：

```
const envChecks = [
    0x82162C, 0x826C60, 0x827CF8, 0x828D00,
    0x829D00, 0x82AD08, 0x82BD98, 0x82FDD8,
    0x832990, 0x836370, 0x83B774, 0x847C28,
    0x849CD4, 0x84FA10
];

envChecks.forEach(function (off) {
    Interceptor.attach(base.add(off), {
        onLeave: function (retval) {
            console.log("0x" + off.toString(16).toUpperCase() + " retval:", retval);
            if (!retval.equals(ptr(0x64))) {
                retval.replace(0x64);
            }
        }
    });
});


```

这一步说明 `0x826C60 / 0x82BD98 / 0x83B774` 是当前环境中实际命中的检测项。但这还不能直接说明它们全部都是 root 检测，需要继续看每个函数内部使用的检测字符串和行为。

#### 18.8.4 0x826C60 root/root-hide 框架检测闭环

为了确认 `0x826C60` 的具体语义，在进入该函数后开启字符串解密 hook：

```
Interceptor.attach(base.add(0x826C60), {
    onEnter: function () {
        hook_str(base);
    },
    onLeave: function (retval) {
        console.log("0x826C60 retval:", retval);
        retval.replace(0x64);
    }
});


```

运行时捕获到 `0x826C60` 相关字符串：

```
hook_str: neozygisk lr: 0x9b7778
hook_str: /data/adb/modules/ lr: 0x9b78f8
hook_str: /data/adb/modules/ lr: 0x9b7a78
hook_str: /data/adb/ lr: 0x9b7bb8
hook_str: /data/adb/ lr: 0x9b7cf8
hook_str: zygisk lr: 0x9b7e2c
hook_str: zygisk lr: 0x9b7f60
hook_str: magisk lr: 0x9b8094
hook_str: magisk lr: 0x9b81c8
hook_str: kernelsu lr: 0x9b8304
hook_str: kernelsu lr: 0x9b8440
hook_str: apatch lr: 0x9b8574
hook_str: apatch lr: 0x9b86a8
hook_str: shamiko lr: 0x9b8764
hook_str: shamiko lr: 0x9b8820
hook_str: riru lr: 0x9b895c
hook_str: riru lr: 0x9b8a98


```

这些字符串均指向 Magisk、Zygisk、KernelSU、APatch、Shamiko、Riru 等 root 或 root-hide 框架，以及 `/data/adb/`、`/data/adb/modules/` 这类典型 root 模块路径。

结合返回值验证：

```
0x826C60 原始返回值 = 0x697
替换为 0x64 后主页面 root 检测恢复正常


```

因此可以确认，`0x826C60` 是 14 个环境检测任务中的 root/root-hide 框架特征检测函数之一。它不是单纯的字符串初始化函数，而是会影响最终 root 检测 UI 状态的关键检测项。

当前这条 root 检测链路可以整理为：

```
pthread_create
  -> 0x11AAFBC 线程模板入口
  -> vtable + 0x10
  -> X8 = 0x911970
  -> sub_911970
  -> sub_81DDF0
  -> 注册 14 个环境检测任务
  -> 0x826C60 root/root-hide 框架检测
  -> 解密/使用 root 框架特征字符串
  -> 返回 0x697
  -> 主页面显示 root 检测异常

绕过验证：
  0x826C60 retval.replace(0x64)
  -> 上层判断通过
  -> 主页面恢复正常


```

`0x82BD98` 和 `0x83B774` 当前也返回异常值，并且修改为 `0x64` 会影响检测结果，但它们的具体检测对象还需要继续通过字符串 hook、文件访问 hook 或返回值传播分析确认。

`images` 目录下的图片可以作为文章配图，用来支撑脚本绕过点和闭环分析。

`ok.png` 是绕过成功后的运行截图。可以看到脚本持续打印检测函数返回值，同时 App 正常进入 Garuda Defender 主界面：

![](https://bbs.kanxue.com/upload/attach/202605/662006_ADJYPN6YC3VZPG9.png)

`ok2.png` 是进一步绕过 root 检测、Frida 检测等环境检测后的主页面截图，用来支撑前面的 root/root-hide 检测返回值修正和其它环境检测绕过点最终生效：

![](https://bbs.kanxue.com/upload/attach/202605/662006_DZ8MVW4THQD5PFY.png)

`001.png` 是未绕过时的弹窗截图，标题为 `Hook Detected!`，正文与 native 层 `sub_61482C` 打印到的文案一致：

![](https://bbs.kanxue.com/upload/attach/202605/662006_NA7XNAXTVGZA3GW.png)

`stack.png` 是 `0x61482C / sub_61482C` 函数的堆栈截图。它支撑了两个结论：hook `sub_61482C` 后可以直接看到告警文案，并且可以通过 native backtrace 反推出上游调用链：

![](https://bbs.kanxue.com/upload/attach/202605/662006_33U3SD9RY5W6UW9.png)

`str.png` 是调用字符串解密 / 还原函数的代码片段截图，用来辅助说明这类检测特征字符串会经过运行时字符串处理路径：

![](https://bbs.kanxue.com/upload/attach/202605/662006_DFYGCJ8SKTYATT5.png)

`thread_template.png` 是 `pthread_create` 创建线程后进入的第一层线程模板函数截图。该模板函数内部通过 `LDR X8, [X8,#0x10]` / `BLR X8` 从任务对象中取出真实任务入口：

![](https://bbs.kanxue.com/upload/attach/202605/662006_M4XKPJJKJKVSU6Z.png)

`0x911970.png` 是线程模板函数动态跳转到的关键任务入口截图。`0x911970` 会继续调用 `sub_81DDF0`，进入环境检测任务调度逻辑：

![](https://bbs.kanxue.com/upload/attach/202605/662006_Z7J4RGZRYT7SCN6.png)

`sub_81DDF0.png` 是 `sub_81DDF0` 中初始化 14 个环境检测任务的代码截图。这里能看到检测函数被成对写入任务表，随后通过 `sub_867F80(v166, 14)` 注册：

![](https://bbs.kanxue.com/upload/attach/202605/662006_5449HTSR7R5H963.png)

`apatch.png` 是进入 `0x826C60` 后 hook 字符串解密函数得到的日志截图，可以看到 `apatch`、`magisk`、`zygisk`、`kernelsu`、`shamiko`、`riru` 和 `/data/adb/` 相关 root/root-hide 框架特征：

![](https://bbs.kanxue.com/upload/attach/202605/662006_6K52SQ9W79MJNBR.png)

`root.png` 是未绕过 root 检测时 App 主页面显示 root 异常的截图：

![](https://bbs.kanxue.com/upload/attach/202605/662006_89JV7D3NJAVA6DT.png)

`sub_6CEE14_01.png` 是 `sub_6CEE14` 这一组检测点的第一张分析图：

![](https://bbs.kanxue.com/upload/attach/202605/662006_YK3CNV2DW3YNMC7.png)

`sub_6CEE14_02.png` 是 `sub_6CEE14` 这一组检测点的第二张分析图：

![](https://bbs.kanxue.com/upload/attach/202605/662006_V7RS69A4VM2TTUK.png)

`sub_6CEE14_03.png` 是 `sub_6CEE14` 这一组检测点的第三张分析图：

![](https://bbs.kanxue.com/upload/attach/202605/662006_YQT5HV8PXNBP9T9.png)

`sub_6CEE14_04.png` 是 `sub_6CEE14` 这一组检测点的第四张分析图：

![](https://bbs.kanxue.com/upload/attach/202605/662006_DFJ5KF7ZN49H7F8.png)

`sub_6CEE14_05.png` 是 `sub_6CEE14` 这一组检测点的第五张分析图：

![](https://bbs.kanxue.com/upload/attach/202605/662006_VG9M4ZU8NY9XYAM.png)

这些图对应脚本里的 `sub_6CEE14(base)` 返回值修正逻辑：

```
hook_6D3C44(base)
hooK_74DE1C(base)
hook_7503C0(base)
hook_756F0C(base)
hook_75B1D0(base)


```

`sub_AEB528.png` 是 `sub_AEB528` 相关检测点图片，对应脚本里的 `hook_AEB528(base)` / `hook_AF0098(base)` 返回值修正：

![](https://bbs.kanxue.com/upload/attach/202605/662006_UVJ4N32BVQVFZCG.png)

`sub_AC7A90.png` 是 `sub_AC7A90` 相关检测点图片，对应脚本里的 `hook_AC7A90(base)` / `hook_11442C0(base)` 返回值修正。该分支和动态栈中的 `sub_AC7A90 -> sub_ACE184 -> sub_36CE08 -> sub_61482C` 路径对应：

![](https://bbs.kanxue.com/upload/attach/202605/662006_A5MCWGKX8FWM2PK.png)

本次分析的核心不是单独定位某一个可 hook 的函数，而是把 Garuda Defender native 层的告警展示链路和环境检测链路分别闭环。

告警展示链路可以分成两段。第一段是检测结果触发后的 native 告警入口：上游检测器命中后，通过 `sub_61482C` 接收告警标题、内容和类型，再经过 `sub_689370`、`sub_38E8F4` 调用 JNI `startActivity`，最终启动 `com.kikyps.kikypspro.SystemAlert`。第二段是 `SystemAlert` 启动后的弹窗构造流程：`RegisterNatives` 注册到 `libcrackme.so` 的 wrapper，wrapper 再通过自定义 resolver 解析到 `libkikypspro.so + 0x37A754`，后续进入 `sub_68BD10`，并由 `sub_64B840` 构造和显示 `AlertDialog`。

因此，`sub_61482C` 更适合看作当前样本 native 层的统一告警入口，`sub_689370/sub_38E8F4` 负责把告警结果送回 Java 层，`sub_68BD10/sub_64B840` 则负责 `SystemAlert` 页面内的弹窗内容构造和展示。这几个点的定位都不是只靠 IDA 伪代码判断，而是结合了运行时参数、JNI 调用日志、返回地址 `lr`、native backtrace 和静态 callsite 交叉验证。

环境检测链路则从 `pthread_create` 开始。`libkikypspro.so + 0x11AAFBC` 是线程模板入口，它通过 `LDR X8, [X8,#0x10]` / `BLR X8` 从任务对象中取出真实执行函数。动态记录 `X8` 后，可以定位到 `0x911970`，再进入 `sub_81DDF0`。`sub_81DDF0` 初始化并注册了 14 个环境检测任务，这些任务不应简单全部归为 root 检测，而应理解为覆盖 root、root-hide、Hook、注入、运行时特征和异常路径等多个检测面的任务集合。

在这 14 个检测任务中，`0x826C60` 已经可以通过动态证据闭环到 root/root-hide 框架检测。它在当前环境中返回 `0x697`，同时字符串解密 hook 捕获到 `magisk`、`zygisk`、`kernelsu`、`apatch`、`shamiko`、`riru`、`/data/adb/` 等特征；将该返回值修正为 `0x64` 后，主页面 root 检测状态恢复正常。`0x82BD98` 和 `0x83B774` 同样在当前环境中返回非 `0x64`，但具体检测对象还需要继续结合字符串、文件访问、maps/smaps 扫描或返回值传播路径确认。

从绕过验证角度看，`0x64` 在当前样本和当前环境中是可验证的通过状态码。将命中的异常检测函数返回值修正为 `0x64` 后，App 可以进入正常主界面，并且 root、Frida 等环境检测结果被压制。这个结论的边界也需要明确：它说明当前版本、当前运行环境下的检测聚合逻辑接受 `0x64` 作为通过值，但不代表所有版本、所有检测函数都必然使用完全相同的返回语义。

整体来看，Garuda Defender 的 native 防护不是单点判断，而是由多层结构组成：字符串运行时还原、线程任务调度、检测任务表、虚表回调、native 告警入口、JNI Activity 启动、Java AlertDialog 展示。分析这类样本时，单纯静态看伪代码很容易被混淆符号、间接调用和 IDA 反编译误差误导；更可靠的方法是把动态 hook、返回地址、参数日志和静态 callsite 一起使用，逐层确认 "谁创建任务、谁执行检测、谁聚合结果、谁触发告警、谁最终展示 UI"。

前面 1-19 章的分析过程对一个 21MB 的混淆 so（`libkikypspro.so`）覆盖得已经比较细，但代价也很明确：手工定位 wrapper、跟踪 resolver、追 `sub_61482C` 的上游 callers、识别 `pthread_create` 的线程模板、抓 `sub_81DDF0` 注册的 14 个检测任务，整个过程几乎全是人工。

问题在于 GarudaDefender 这类商业 RASP 不会停留在一个版本。每次版本更新，符号被重新混淆，函数被重新分布，关键偏移（`0x37A754`、`0x68BD10`、`0x64B840`、`0x61482C`、`0x689370`、`0x11AAFBC`、`0x911970`、`0x826C60`）几乎全部变化。如果每个变体都从零重做一遍 1-19 章的工作，分析成本会随着版本数线性增长。

但本文 1-19 章实际上已经显露了一个事实：**变化的是偏移，不变的是行为模式**。

```
不变的特征
  RegisterNatives 在 libcrackme.so 中只是 wrapper，
  真正业务在 libkikypspro.so

  自定义 resolver（类 dlsym）通过 hash 字符串解析符号，
  形态固定为 "wrapper -> resolver -> real func"

  native 告警入口具有的汇编/行为特征：
    入口处 pthread_mutex 加锁 + 状态字段检查
    至少 10+ 个调用方（TBNZ/CBZ 后跳入）
    传入参数中 a1/a2 是字符串、a3 是小整数 type code

  pthread_create 创建的检测线程使用统一模板，
  通过 vtable+0x10 间接调用真实任务

  检测任务表通过连续 STP 写入函数指针对，
  最后由 register(table, count) 一次注册

  字符串解密函数指针存放在 .data 段固定槽位，
  调用形态为 "ptr_str(args[0]) -> 解密后的 C 字符串"

  检测函数的"通过状态码"在该样本族中保持稳定（0x64）


```

这些行为特征不会随版本改变。把这些特征写成可执行的方法论，就是 Hermes Agent Skill 的意义。

如果把第 1-19 章压缩成一份普通笔记，最自然的写法是：

```
v4.4.0:
  SystemAlert.onCreate native = libcrackme.so + 0x1B15C
  resolver                    = libcrackme.so + 0x26F78
  real onCreate               = libkikypspro.so + 0x37A754
  alert dispatcher            = libkikypspro.so + 0x68BD10
  show dialog                 = libkikypspro.so + 0x64B840
  unified alert entry         = libkikypspro.so + 0x61482C
  hash_crc compare CMP        = libkikypspro.so + 0xAB3C80
  thread template             = libkikypspro.so + 0x11AAFBC
  env check registrar         = libkikypspro.so + 0x81DDF0
  root detector               = libkikypspro.so + 0x826C60
  bypass status code          = 0x64


```

这种笔记对于本版本可以直接 hook，但对下一个版本完全没有指导作用——所有偏移都会变。

把同样的信息写成 Skill 后，描述方式会发生根本变化：

```
[Skill 节选: 定位 native 统一告警入口]

特征：
  函数体足够大（前文 sub_61482C 体量明显高于普通 helper），
  入口处对全局 mutex 调用 lock helper，
  在多个上游告警处理器中以 BL 形式被调用（>10 个 caller）。

定位步骤：
  1. IDA MCP: xrefs_to(pthread_mutex 系列符号或对应 lock helper)
  2. func_query: 过滤 "size > 0x100 且 caller 数 >= 10" 的候选
  3. 对每个候选 decompile，确认形态为
       sub_X(a1: string, a2: string, a3: int)
       内部 BL 一个 JNI 包装函数
  4. 动态验证：hook 候选地址，观察是否打印告警标题/正文

特征签名：
  入口处的 ARM64 序列大致为
    SUB SP, SP, #...
    STP X28..X19, [SP,#...]
    MOV X19/X20 = X1/X0
    MOV W21 = W2
    ADRL X1, <mutex>
    BL <lock helper>
  上述序列可作为版本无关签名搜索锚点。


```

注意这里没有任何具体地址。Hermes Agent 拿到这段 Skill，对任何变体都可以按步骤执行：搜 mutex 系列符号 → 拿 xrefs → 过滤函数体大小和 caller 数 → 反编译形态匹配 → Frida 验证。最后输出的是这个版本里 `sub_61482C` 等价物的实际偏移。

把 1-19 章的方法论拆成 Skill 时，建议按以下结构组织：

```
1. 架构假设
   - libcrackme.so + libkikypspro.so 双 SO 模型
   - libcrackme.so 提供 RegisterNatives wrapper + 自定义 dlsym
   - libkikypspro.so 承载所有真实检测/告警/字符串还原

2. 入口定位方法
   - hook RegisterNatives 拿 SystemAlert/KikyPS 类的 native 注册表
   - hook libcrackme.so 内部 dlopen wrapper（本文档中是 sub_26538）
     等待 libkikypspro.so 加载完成

3. 告警链路定位方法（对应本文 5/9/10/16/17 章）
   - 找 native 统一告警入口（特征：mutex + 大函数 + 多 caller +
     a1/a2 字符串 a3 整型）
   - 它的下游是 JNI varargs 包装器（特征：从 vtable 偏移 0x1F0 取函数指针）
   - 再下游是 CallVoidMethodV(startActivity)
   - SystemAlert.onCreate 的真实入口由 RegisterNatives + resolver 解出

4. 完整性校验定位方法（对应本文 13/15 章）
   - 找含 hash_crc 比较的函数（特征：BL <hash> 后紧跟 CMP X?, X0 + B.EQ）
   - 该 hash 函数的固定常量包括 0x9FB21C651E98DF25 / 0x165667919E3779F9 等
   - 比较失败的分支会经一个桥接函数（特征：从 ctx+0x30 取对象，
     然后 vtable+0x30 间接调用），再进入告警处理器

5. 环境检测链路定位方法（对应本文 18.8 章）
   - hook libc pthread_create
   - 过滤 start_routine 落在 libkikypspro.so 范围内的调用
   - 该 start_routine 是线程模板，特征：从对象 vtable+0x10 间接调用
   - 在 BLR X8 前打印 X8 并按 RVA 去重，得到所有任务真实入口
   - 其中负责注册检测任务表的函数特征：
       连续 N 次 "STP <ctx_func>, <check_func>, [stack/obj+offset]"
       最后 BL <register>(table, count)
     count 即检测项数（本样本族目前为 14）

6. 状态码协议
   - 在该样本族中，检测函数返回 0x64 = 通过
   - 非 0x64（如 0x697 / 0x413 / 0x25c）= 命中
   - 验证方式：把命中的检测函数 retval 替换为 0x64，
     观察主页面 root 检测和告警弹窗是否消失

7. Frida 脚本生成模板
   - dlopen 等待 libcrackme.so 加载
   - hook libcrackme.so 自实现 dlopen，等 libkikypspro.so 加载
   - 在告警入口下 hook（打印 title/message + backtrace）
   - 在 hash_crc CMP 处把 X23 改为 X0（或等价的强制相等）
   - 在每个检测任务入口下 hook，retval != 0x64 时 retval.replace(0x64)

8. 字符串解密 hook 方法
   - .data 段固定槽位保存解密函数指针
   - 在该样本中是 base + 0x15264F0
   - 在变体中需要重新定位：搜索"指针被 LDR 后立刻 BLR 且 args[0] 在 hook 后
     可读出可见英文 ASCII"的函数
   - hook 出口可枚举所有运行时还原的检测特征字符串

9. 已知陷阱
   - 任务真实入口 X8 不一定落在 IDA 自动识别的函数起始地址上
     （样本中存在 "X8 = sub_X + 4" 的情况，需以动态记录为准）
   - SystemAlert 弹窗是分两段产生的：先 startActivity 再 onCreate
     两段不能互相替代，绕过任意一段都不够
   - hash_crc 失败不立即弹窗，先经桥接函数 + 二次去重判断，
     不能只 hook 比较点本身

10. 版本差异速查表
    - v4.4.0 偏移见本文档；新版本由 Agent 按上述方法重新定位后追加


```

每一节都是 "该如何重新做这件事"，不是 "这次的答案是什么"。

假设拿到 GarudaDefender 的下一个变体（暂称 v4.5.x），加载这套 Skill 后 Hermes Agent 的执行序列大致如下：

```
Phase 1 — 加载与初始化
  terminal: adb install <new.apk>
  terminal: unzip -l <apk> | grep "lib/arm64"
  terminal: frida -U -f <pkg> --pause  (拿 base 用)

Phase 2 — 双 SO 模型确认
  IDA MCP: open libcrackme.so / libkikypspro.so
  IDA MCP: list_imports，确认 libkikypspro.so 不被系统直接 dlopen
  Frida:   hook RegisterNatives，拿到 SystemAlert.onCreate 注册地址
           （Skill 不假设它仍是 0x1B15C，由这一步实测得出）
  Frida:   hook android_dlopen_ext + libcrackme.so 内部 dlopen wrapper，
           记录 libkikypspro.so 真实 base

Phase 3 — 告警入口定位
  IDA MCP: xref_query(pthread_mutex_lock 等价 helper)
  IDA MCP: func_query(min_size=0x100, caller_count>=10)
  IDA MCP: 对每个候选 decompile，按 Skill 中的形态匹配
           （a1/a2 字符串、a3 小整型、内部 BL JNI 包装）
  Frida:   hook 候选地址，观察是否打印 "Hook Detected!" 一类字符串
  → 输出 v4.5.x 中 sub_61482C 等价物的偏移

Phase 4 — 完整性校验定位
  IDA MCP: insn_query(mnem="BL", 后续 CMP X?, X0 + B.EQ 模式)
  IDA MCP: func 内部包含常量 0x9FB21C651E98DF25 → 命中 hash 函数
  IDA MCP: xrefs_to(hash 函数) → 找到比较点
  Frida:   在比较点 hook，确认 X23/X0 是两份 hash 值
  → 输出 v4.5.x 中 0xAB3C80 等价物的偏移

Phase 5 — 检测任务表枚举
  Frida:   hook libc pthread_create，过滤 start_routine ∈ libkikypspro.so
  Frida:   在 start_routine 内部 BLR X8 前打印 X8，按 RVA 去重
  → 拿到所有任务真实入口
  IDA MCP: 对每个任务入口看其上游，找出包含 "连续 STP 写入函数指针对 +
           BL register(table, count)" 模式的函数
  → 输出 v4.5.x 中 sub_81DDF0 等价物 + 检测项数（可能仍是 14，
     也可能因版本差异变为 12/16/...）

Phase 6 — 状态码与绕过验证
  Frida:   对每个检测任务 hook onLeave，记录 retval 分布
  Frida:   命中（非 0x64）的项替换为 0x64，看主页面状态变化
  → 输出 v4.5.x 实际命中的检测项清单

Phase 7 — 脚本生成与归档
  write_file: 按 Skill 中的脚本模板，把 v4.5.x 偏移填入，生成 ok.js
  terminal:   frida -H ... -f <pkg> -l ok.js，验证主页面恢复
  skill_manage: 在 Skill 的"版本差异速查表"中追加 v4.5.x 一行
                  （仅追加偏移，方法论部分不需要改）


```

整个流程不需要逆向人员逐条下指令。Skill 加载后，Agent 按方法论自己跑完。

第 19 章总结里提到："分析这类样本时，单纯静态看伪代码很容易被混淆符号、间接调用和 IDA 反编译误差误导；更可靠的方法是把动态 hook、返回地址、参数日志和静态 callsite 一起使用。"

这句话的重要程度高于任何具体偏移。具体偏移只对一个版本有效，"动态 hook + 返回地址 + 参数日志 + 静态 callsite 交叉验证" 这套方法论对整个 RASP 家族都有效。Skill 系统的价值就是让这种方法论能被 Agent 直接执行，而不是停留在文章结论里。

每次分析新变体时遇到的新陷阱（比如 18.8.1 节里 "任务真实入口可能落在函数中间"），都可以追加到 Skill 的 "已知陷阱" 小节。Skill 因此随每次分析变得更完善，下一次分析的起点更高。

把第 1-19 章的工作过程做一次方法论编码，对该 RASP 家族的所有未来版本都是一次投入永久收益的事。这也是把这次分析记录留下来的额外价值——它不只是一次绕过的过程描述，更是一份可以转化为 Agent 可执行能力的素材。

#### 20.5.1 版本漂移下 Skill 的实际表现

商业 RASP 在版本迭代时通常会改三种东西：符号被重新混淆（同一个 mutex 加锁 helper 在 v4.4.0 是 `sub_4C56F0`，在 v4.5.x 可能完全是另一个偏移和符号）、函数布局被编译器内联策略重排（`sub_61482C` 等价物可能被拆成两段，也可能被合并进上层调用者）、检测项数量微调（14 个环境检测任务可能变成 12 个或 16 个）。但这三种变化都没有动到行为模式本身：全局 mutex 加锁 helper 仍然存在，且仍然被告警入口调用；告警入口仍然接收 `(string title, string message, int type)` 三个参数；检测任务表仍然通过 " 连续 STP 写入函数指针对 + `BL register(table, count)`" 模式注册；通过状态码可能从 `0x64` 换成 `0x65`，但用 "批量统计返回值找众数" 的方法依然能直接定位它。

`locate-rasp-alert-entry` 这一类 Skill 编码的就是上面这些行为模式，而不是任何一个具体偏移。所以版本变化并不会让 Skill 失效，只会让 "在新偏移上重新落地" 这一步多跑一次。从 Agent 视角看，跨版本分析的工作量从 "重新理解整个样本" 压缩成了 "按已知方法论重新对地址表"。这是 Skill 化最直接的收益。

如果 Garuda 系列在某个未来版本彻底重构防护架构（比如换掉自定义 dlsym 跳板、改用线程池而不是线程模板），那么对应的 Skill 就需要更新。但这种架构级变化在商业 RASP 里发生频率很低，通常间隔多个版本才出现一次。在两次架构级变化之间，Skill 都能持续生效。

#### 20.5.2 Skill 之间的协同

7 个 Skill 不是孤立模块，而是相互调用形成一张方法论图谱：

```
locate-rasp-alert-entry
  └─ 内部依赖 cross-validate-static-dynamic 验证候选

enumerate-detection-task-table
  └─ 内部依赖 trace-thread-template-handoff 拿到任务真实入口

bypass-with-stable-status-code
  └─ 内部依赖 enumerate-detection-task-table 输出的检测函数列表

identify-hashcrc-checkpoint
  └─ 内部依赖 hook-string-decryptor-slot 还原比较点附近的特征字符串


```

这种结构意味着 Agent 在执行一个 Skill 时，会按依赖关系自动调起其它 Skill。研究者只需要给一个高层目标（例如 "绕过这个新版本的 Hook Detected 弹窗"），Agent 会自己把它拆解成 "先 enumerate task table → 再 locate alert entry → 再 bypass via status code" 的执行序列，并在每一步用 cross-validate 做交叉验证。

这种协同也带来一个边际效益：单独添加一个新 Skill，往往不只是增加了它自己的能力，还会被现有 Skill 调用，从而提升整个图谱的覆盖度。比如新增一个 "识别字符串解密函数指针在 .data 段的位置" 的辅助 Skill，会立刻让所有依赖字符串特征的上游 Skill 在新版本上更稳定。

#### 20.5.3 第 N 次分析的成本曲线

传统逆向工作流下，分析第二个、第三个 RASP 变体的成本和分析第一个差不多——经验只在研究者脑子里，无法直接复用。Skill 化之后，成本曲线会发生明显变化：

```
第 1 次分析 GarudaDefender v4.4.0
  → 从零提取 7 个 Skill，研究者主导
  → 耗时基线 100%

第 2 次分析另一个商业 RASP（比如 Doraemon 系列）
  → 7 个 Skill 中可复用 5 个（商业 RASP 共性）
  → 新增 2-3 个该 RASP 特有的 Skill
  → 耗时降到 50-60%

第 3 次回到 Garuda v5.0
  → 加载已有 7 个 Skill，Agent 自动跑出新版本偏移
  → 研究者只需审核 + 处理 1-2 个新陷阱
  → 耗时降到 20-30%

第 N 次（同家族第 N 个版本）
  → 几乎全自动，研究者只参与异常分支
  → 耗时趋近于"运行一次完整 Frida 验证"的固定成本


```

也就是说，Skill 不只是一次性沉淀，而是会指数级压缩未来同类工作的成本。同一套 Skill 复用得越多，它覆盖的边界条件越完整，下一次分析的起点就越高。这跟传统逆向 "每个版本都要重头来一遍" 的成本结构完全不同。

#### 20.5.4 Skill 的边界：哪些环节适合固化，哪些不适合

并不是所有方法论环节都适合写成 Skill。可以固化的是机械性可验证的部分——"找含 mutex 加锁的函数"、"在 BL 后下断点"、"批量 hook 检测函数返回值"——这些都有明确的输入输出和验证方法。

但有些环节高度依赖研究者的直觉判断：在十几个候选函数里挑哪个先反编译、字符串解密 hook 输出的几百条字符串里哪些是真特征哪些是干扰、动态栈里哪条分支值得先深入下钻——这些判断很难写成 Skill，也不应该强行写。Skill 系统设计良好的边界，是把可固化的机械环节交给 Agent，把判断环节留给人。

本文 1-19 章的过程里，Hermes 自主沉淀的 7 个 Skill 全部落在前者范围内；后者那部分，我自己保留了完全控制权。这种分工不是为了 "AI 不抢人的工作"，而是因为两类工作的最优执行者本来就不一样：机械验证 Agent 比人快几个数量级且不会犯困，而模式判断在当前阶段仍然是人类研究者的优势项。Skill 化做对的标志，不是 "什么都能让 Agent 跑"，而是 "该让 Agent 跑的全跑了，剩下的研究者真的需要思考"。

[[培训]《冰与火的战歌：Windows 内核攻防实战》！从零到实战，融合 AI 与 Windows 内核攻防全技术栈，打造具备自动化能力的内核开发高手。](https://www.kanxue.com/book-section_list-227.htm)

最后于 57 分钟前 被秋落编辑 ，原因：

上传的附件：

*   ok.js （11.85kb，3 次下载）
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269003.htm)

> vx 逆向分析随笔

图片太多了，就直接用的 github 博客的图片链接……

vx 逆向分析随笔
=========

@app_version = 7.0.20

 

@app_date = 2020.11.19

 

本来是想分析 vx8 的，但是电脑性能不够…… 反编译工具直接卡死

vx Log 日志
---------

当进行分析时，发现反复调用了`ae.i`,`ae.d`方法

 

![](https://forgo7ten.github.io/2021081401/image-20210814171654155.png)

 

来到`com.tencent.mm.sdk.platformtools.ae;`查看

 

![](https://forgo7ten.github.io/2021081401/image-20210814171854620.png)

 

同时找到了`setConsoleLogOpen`方法，猜测这是对控制台 log 全局控制的方法

 

![](https://forgo7ten.github.io/2021081401/image-20210814172452010.png)

 

对该方法向上追溯，得到

```
com.tencent.mm.sdk.platformtools.ae$a;setConsoleLogOpen
com.tencent.mm.sdk.platformtools.ae;setConsoleLogOpen
com.tencent.mm.xlog.app.XLogSetup;keep_setupXLog
com.tencent.mm.xlog.app.XLogSetup;realSetupXlog
com.tencent.mm.plugin.account.ui.SimpleLoginUI;onCreate/com.tencent.mm.ui.LauncherUI;onResume

```

其中设置 log 的是 keep_setupXLog 的 p5 参数

```
public static void keep_setupXLog(boolean p0,String p1,String p2,Integer p3,Boolean p4,Boolean p5,String p6){
   // ...
   XLogSetup.cachePath = p1;    // 缓存目录:/data/user/0/com.tencent.mm/files/xlog
   XLogSetup.logPath = p2;      // log目录:/storage/emulated/0/Android/data/com.tencent.mm/MicroMsg/xlog
   XLogSetup.toolsLevel = p3;   // 工具版本:null
   XLogSetup.appendIsSync = p4;
   XLogSetup.isLogcatOpen = p5; // 是否开启日志:false
   XLogSetup.nameprefix = p6;   // 前缀:MM
   if (!p0) {  
      // ...
   }else if(XLogSetup.setup){      
      // ...
   }else { 
      // ...
      ae.setConsoleLogOpen(XLogSetup.isLogcatOpen.booleanValue());
      // ...
   }
   return;
}

```

于是开启 log 只需要给`p5`赋值为`true`

 

frida 脚本

```
Java.perform(function() {
    /*
    * 开启vx全局日志
    */
    var clazz = Java.use('com.tencent.mm.xlog.app.XLogSetup');
    clazz.keep_setupXLog.overload('boolean', 'java.lang.String', 'java.lang.String', 'java.lang.Integer', 'java.lang.Boolean', 'java.lang.Boolean', 'java.lang.String').implementation = function(p0,p1,p2,p3,p4,p5,p6) {
 
        //console.log("arguments",arguments[5]);
        arguments[5] = Java.use("java.lang.Boolean").TRUE.value;
        console.log("已开启vx Log打印");
        return clazz.keep_setupXLog.apply(this, arguments);
    }
});

```

vx 发送消息
-------

使用`DDMS`的`Start Method Profiling`功能，同时点击【发送】键

 

然后停止记录，搜索`onClick`关键字，定位到类`com.tencent.mm.pluginsdk.ui.chat.ChatFooter$7`

 

其中`sstr`为输入的字符串内容

 

![](https://forgo7ten.github.io/2021081401/image-20210815161220040.png)

 

尝试 hook `ChatFooter.a`方法

```
Java.perform(function() {
    var clazz = Java.use('com.tencent.mm.pluginsdk.ui.chat.ChatFooter');
    clazz.a.overload('com.tencent.mm.pluginsdk.ui.chat.ChatFooter', 'java.lang.String').implementation = function() {
        console.log("Send Message",arguments[1]);
        return clazz.a.apply(this, arguments);
    }
});

```

可以得到输出

```
Send Message 1
Send Message 2
Send Message [微笑]
Send Message

```

vx 聊天窗口 / 修改内容
--------------

### 定位

使用`DDMS`的`Dump View Hierarchy for UI Automator`来定位控件 ID

 

![](https://forgo7ten.github.io/2021081401/image-20210815003423908.png)

 

查看顶层 activity

```
adb shell dumpsys activity top

```

再找控件 ID，最终定位到`com.tencent.mm.ui.chatting.view.MMChattingListView`

 

![](https://forgo7ten.github.io/2021081401/image-20210815003515022.png)

### 分析

#### MMChattingListView

这是一个`ListView`

 

发现了 List 的`Adapter`，

 

![](https://forgo7ten.github.io/2021081401/image-20210815004245534.png)

 

交叉引用查看是哪里调用了这个方法

 

跟到`com.tencent.mm.ui.chatting.ChattingUIFragment`的`fTJ`方法

#### ChattingUIFragment

![](https://forgo7ten.github.io/2021081401/image-20210815004604975.png)

 

发现这个`Fragment`对 Adapter 进行了设置，同时存放在了`this.Lrn`字段

 

其中`this.Lrr`类型为`MMChattingListView`，`this.Lro`的类型为`ListView`

 

也可以得到 Adapter 的类型为`com.tencent.mm.ui.chatting.a.a`，同时继承了`BaseAdapter`

#### Adapter

根据其重写的`getCount`方法，得到数据源是`this.LtF`字段

 

![](https://forgo7ten.github.io/2021081401/image-20210815010612094.png)

 

而`this.LtF`字段的定义是

```
public SparseArray LtF;

```

使用 frida hook 一下，查看一下 SparseArray 每个元素的类型

 

随便 hook 了一个 Fragment 的`zt`方法（因为看 Log 日志每次操作都会执行）

 

获得 Fragment 的`Lrn`字段也就是 Adapter

 

再直接调用`getCount()`和`getItem()`获取到数据源的单个数据

```
Java.perform(function() {
 
    var clazz = Java.use('com.tencent.mm.ui.chatting.ChattingUIFragment');
    clazz.zt.implementation = function() {
        console.log("this.Lrn.value",this.Lrn.value);
        var adapter = Java.cast(this.Lrn.value,Java.use("com.tencent.mm.ui.chatting.a.a"))
        console.log("adapter",adapter);
        console.log("adapter.getCount",adapter.getCount());
        console.log("LtF",adapter.LtF.value);
        var adapter_size = adapter.getCount();
        var msg = adapter.getItem(0);
        console.log("msg",msg);
        return clazz.zt.apply(this, arguments);
    }
});

```

此时手机界面为

 

![](https://forgo7ten.github.io/2021081401/img.png)

 

获得输出为

```
this.Lrn.value com.tencent.mm.ui.chatting.a.a@f945978
adapter com.tencent.mm.ui.chatting.a.a@f945978
adapter.getCount 5
LtF {0=com.tencent.mm.storage.bz@cc74ce8, 1=com.tencent.mm.storage.bz@dba9f01, 2=com.tencent.mm.storage.bz@ae31a6, 3=com.tencent.mm.storage.bz@9afe7e7, 4=com.tencent.mm.storage.bz@c384f94}
msg com.tencent.mm.storage.bz@cc74ce8

```

可以知道每个数据的类型是`com.tencent.mm.storage.bz`

#### 每条消息 storage.bz

该类的继承关系为

```
com.tencent.mm.storage.bz
com.tencent.mm.ah.aa
com.tencent.mm.g.c.ej
com.tencent.mm.sdk.e.c

```

查看个方法，发现里面有如下字段

 

![](https://forgo7ten.github.io/2021081401/image-20210815143427853.png)

 

这些字段是`ej`类的

 

![](https://forgo7ten.github.io/2021081401/image-20210815144136894.png)

 

同时`bz`还有 5 个子类

 

`bz$a`

 

![](https://forgo7ten.github.io/2021081401/image-20210815152148504.png)

 

`bz$b`，大概是有关位置的

 

![](https://forgo7ten.github.io/2021081401/image-20210815152234412.png)

 

`bz$c`

 

![](https://forgo7ten.github.io/2021081401/image-20210815152302867.png)

 

`bz$d`，可以看到`nickname`, `cityCode`, `CountryCode`字段

 

![](https://forgo7ten.github.io/2021081401/image-20210815152614851.png)

### 脚本实现

最终选择 hook 的是 Adapter 的`notifyDataSetChanged`方法，这样可以对最后的消息进行查看以及修改

 

修改一下消息内容

```
Java.perform(function() {
    var clazz = Java.use('com.tencent.mm.ui.chatting.a.a');
    clazz.notifyDataSetChanged.implementation = function() {
        console.log("notifyDataSetChanged");
        var data = this.LtF.value;
        var data_size = data.size();
        console.log("data_size",data_size);
        for(var i=0;i
```

运行脚本之前的界面

 

![](https://forgo7ten.github.io/2021081401/img-1629011094588.png)

 

可以得到以下输出

```
notifyDataSetChanged
data_size 7
Message0 1
Message1 2
Message2 3
Message3 4
Message4 5
Message5 哈哈哈哈哈哈哈哈哈哈哈哈
Message6 6

```

同时界面被修改为了

 

![](https://forgo7ten.github.io/2021081401/img-1629011192790.png)

 

当然只能修改本地数据（虽然没什么卵用）

 

同时 json 输出该消息的数据结构为

```
// com.tencent.mm.storage.bz
{
    "KzF": false,
    "KzG": false,
    "Zq": false,
    "__hadSetcreateTime": false,
    "__hadSettype": false,
    "dvP": "",
    "eBD": false,
    "eBO": false,
    "eBf": false,
    "eEH": 0,
    "eEI": "",
    "eEL": false,
    "eEM": false,
    "eEN": false,
    "eEO": false,
    "eEP": false,
    "eEQ": false,
    "eER": false,
    "eES": false,
    "eEf": false,
    "eEg": false,
    "eEh": false,
    "eEi": false,
    "eEj": false,
    "eEq": false,
    "ewj": true,
    "ewq": false,
    "exe": true,
    "fdE": false,
    "fdF": false,
    "fdI": "",
    "fdJ": 0,
    "fdK": 0,
    "fdL": 0,
    "fdM": 1,
    "fdN": 0,
    "fdO": 0,
    "fdP": "",
    "fdQ": "",
    "fdR": "",
    "fdS": 0,
    "fdT": [],
    "fdU": "",
    "fdV": "",
    "fdW": 0,
    "fdX": 0,
    "field_bizChatId": -1,
    "field_bizClientMsgId": "",
    "field_content": "",            // 聊天内容
    "field_createTime": 1629009756000,      // 聊天时间戳
    "field_flag": 0,
    "field_isSend": 0,
    "field_isShowTimer": 0,
    "field_lvbuffer": [0, 0, 125],
    "field_msgId": 00,                      // 消息id
    "field_msgSeq": 000000000,
    "field_msgSvrId": 4746617000000000000,
    "field_status": 0,
    "field_talker": "wxid_00000000000000",  // 对话的微信用户ida
    "field_talkerId": 00,
    "field_type": 1,
    "systemRowid": -1
}

```

其中接收到的消息比发送的消息要多了几个字段（普通文本消息）

### 类总结

因此可以分析到聊天窗口 UI 以及数据的类为

<table><thead><tr><th>描述</th><th>类名</th></tr></thead><tbody><tr><td><code>Fragment</code></td><td><code>com.tencent.mm.ui.chatting.ChattingUIFragment</code></td></tr><tr><td><code>ListView</code></td><td><code>com.tencent.mm.ui.chatting.view.MMChattingListView</code></td></tr><tr><td><code>Adapter</code></td><td><code>com.tencent.mm.ui.chatting.a.a</code></td></tr><tr><td>单条消息类</td><td><code>com.tencent.mm.storage.bz</code></td></tr><tr><td></td><td></td></tr><tr><td></td></tr></tbody></table>

vx 投骰子
------

### 分析

先用 DDMS 的记录功能找到点击事件为`com.tencent.mm.emoji.panel.a.q$1.onClick()`

 

![](https://forgo7ten.github.io/2021081401/image-20210816123125656.png)

 

随后进入了`glG.a`方法，也就是`com.tencent.mm.emoji.panel.a.d.a()`方法

 

![](https://forgo7ten.github.io/2021081401/image-20210816123331547.png)

 

frida 调试`arg14.type`一直等于 0

 

刚开始没咋看代码，然后去了`v0_2.A(v6)`方法，然后找了三四层。:confused: 后来发现里面的有的方法不止是投骰子的时候才被调用才发觉找错了。

 

然后 frida 查看了一下`v1.getGroup()`，发现我添加的自定义表情是`81`，而骰子和石头剪刀布都是`18`；同时`EmojiGroupInfo.Qbd`也为`18`

 

然后分析`.getProvider().p(v1)`方法，交叉引用有多个，分别是`com.tencent.mm.ca.a.p()`和`com.tencent.mm.plugin.emoji.e.f.p()`

 

frida 打印调用信息发现`com.tencent.mm.ca.a.p()`先被调用

 

![](https://forgo7ten.github.io/2021081401/image-20210816124809022.png)

 

frida 调试发现进入了`getEmojiMgr().p(arg9)`方法，也就是`com.tencent.mm.plugin.emoji.e.f.p()`

 

![](https://forgo7ten.github.io/2021081401/image-20210816130822813.png)

 

Cursor 是数据库的游标，通过`aec`方法来查询相应内容

 

![](https://forgo7ten.github.io/2021081401/image-20210816132210776.png)

 

然后`v0.moveToPosition(v1)`让游标 v0 移动到 v1 位置

 

`arg6.convertFrom(v0)`这个方法，位于`EmojiInfo`的父类`com.tencent.mm.g.c.bj`，查询该行相应的值然后给字段赋值

 

![](https://forgo7ten.github.io/2021081401/image-20210816132511952.png)

 

之后发送的 emoji 表情就是这个`arg6`了

 

所以看`int v1 = bu.jE(v0.getCount() - 1, 0);`

 

frida 得到，石头剪刀布`getCount()`为 3；而投骰子`getCount()`为 6

 

查看`com.tencent.mm.sdk.platformtools.bu.jE()`方法

 

![](https://forgo7ten.github.io/2021081401/image-20210816132833909.png)

> public int nextInt(int n)
> 
> 该方法的作用是生成一个随机的 int 值, 该值介于 [0,n) 的区间, 也就是 0 到 n 之间的随机 int 值, 包含 0 而不包含 n。

 

`arg6`是恒为 0 的。也就是说返回的 v0 是介于`[0,arg5]`的。

 

经 Frida 调试，骰子点数 1-6 对应着 0-5；剪刀石头布分别对应着 0 1 2

 

若想修改相应点数，直接 hook 该函数返回值就好

 

相应堆栈信息（**不太懂为什么有的方法出现了两次，并且有个`(Native Method)`**有大佬可以解释一下嘛: yum:）

```
com.tencent.mm.sdk.platformtools.bu.jE(Native Method)
com.tencent.mm.plugin.emoji.e.f.p(SourceFile:91)
com.tencent.mm.plugin.emoji.e.f.p(Native Method)
com.tencent.mm.ca.a.p(SourceFile:240)
com.tencent.mm.ca.a.p(Native Method)
com.tencent.mm.emoji.panel.a.d.a(SourceFile:66)
com.tencent.mm.emoji.panel.a.d.a(Native Method)
com.tencent.mm.emoji.panel.a.q$1.onClick(SourceFile:41)
android.view.View.performClick(View.java:6294)
android.view.View$PerformClick.run(View.java:24770)
android.os.Handler.handleCallback(Handler.java:790)
android.os.Handler.dispatchMessage(Handler.java:99)
android.os.Looper.loop(Looper.java:164)
android.app.ActivityThread.main(ActivityThread.java:6494)
java.lang.reflect.Method.invoke(Native Method)
com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:438)
com.android.internal.os.ZygoteInit.main(ZygoteInit.java:807)

```

### 脚本实现

```
Java.perform(function () {
 
    var clazz = Java.use('com.tencent.mm.sdk.platformtools.bu');
    clazz.jE.implementation = function () {
        for(var i=0;i
```

vx 防撤回
------

### 分析

防撤回难点是监听消息啥时候发送过来，真没啥思路。刚开始从聊天页面的 ListView 的`notifyDataSetChanged`开始回溯，回溯了好久找不到。

 

就搜索 Log 日志`revoke`

 

![](https://forgo7ten.github.io/2021081401/image-20210816160615130.png)

 

然后去类中搜索字符串定位到`com.tencent.mm.plugin.msgquote.PluginMsgQuote.handleRevokeMsgBySvrId`

 

打印一下堆栈信息

```
com.tencent.mm.plugin.msgquote.PluginMsgQuote.handleRevokeMsgBySvrId(Native Method)
com.tencent.mm.model.f.a(SourceFile:2190)
com.tencent.mm.model.ch.b(SourceFile:258)
com.tencent.mm.s.b.b(SourceFile:40)
com.tencent.mm.plugin.messenger.foundation.c.processAddMsg(SourceFile:165)
com.tencent.mm.plugin.messenger.foundation.c.a(SourceFile:1059)
com.tencent.mm.plugin.messenger.foundation.f.a(SourceFile:118)
com.tencent.mm.plugin.zero.c.a(SourceFile:57)
com.tencent.mm.modelmulti.q$a$1.onTimerExpired(SourceFile:830)
com.tencent.mm.sdk.platformtools.aw.handleMessage(SourceFile:86)
com.tencent.mm.sdk.platformtools.aq$2.handleMessage(SourceFile:362)
android.os.Handler.dispatchMessage(Handler.java:106)
com.tencent.mm.sdk.platformtools.aq$2.dispatchMessage(SourceFile:350)
android.os.Looper.loop(Looper.java:164)
android.os.HandlerThread.run(HandlerThread.java:65)

```

之后写出如下 frida 脚本

```
Java.perform(function () {
    var clazz = Java.use('com.tencent.mm.plugin.msgquote.PluginMsgQuote');
    clazz.handleRevokeMsgBySvrId.implementation = function () {
        var log_str = "arguments:"
        for (var i = 0; i < arguments.length; i++) {
            log_str += " [" + i + "]" + arguments[i]
        }
        console.log("8 handleRevokeMsgBySvrId", log_str);
 
        return clazz.handleRevokeMsgBySvrId.apply(this, arguments);
    }
});
 
Java.perform(function () {
    var clazz = Java.use('com.tencent.mm.model.f');
    clazz.a.implementation = function () {
        var log_str = "arguments:"
        for (var i = 0; i < arguments.length; i++) {
            log_str += " [" + i + "]" + arguments[i]
        }
 
        console.log("7 f.a()", log_str);
 
        return clazz.a.apply(this, arguments);
    }
});
 
Java.perform(function () {
    var clazz = Java.use('com.tencent.mm.model.ch');
    clazz.b.overload('com.tencent.mm.ak.e$a').implementation = function () {
        var log_str = "arguments:"
        for (var i = 0; i < arguments.length; i++) {
            log_str += " [" + i + "]" + arguments[i]
        }
 
        console.log("6 ch.b()", log_str);
 
        return clazz.b.overload('com.tencent.mm.ak.e$a').apply(this, arguments);
    }
});
 
 
Java.perform(function () {
    var clazz = Java.use('com.tencent.mm.s.b');
    clazz.b.overload('com.tencent.mm.ak.e$a').implementation = function () {
        var log_str = "arguments:"
        for (var i = 0; i < arguments.length; i++) {
            log_str += " [" + i + "]" + arguments[i]
        }
        console.log("5 b.b()", log_str);
 
        return clazz.b.overload('com.tencent.mm.ak.e$a').apply(this, arguments);
    }
});
 
 
Java.perform(function () {
    var clazz = Java.use('com.tencent.mm.plugin.messenger.foundation.c');
    clazz.processAddMsg.implementation = function () {
        var log_str = "arguments:"
        for (var i = 0; i < arguments.length; i++) {
            log_str += " [" + i + "]" + arguments[i]
        }
 
        console.log("4 c.processAddMsg()", log_str);
 
        return clazz.processAddMsg.apply(this, arguments);
    }
});
 
Java.perform(function () {
    var clazz = Java.use('com.tencent.mm.plugin.messenger.foundation.c');
    clazz.a.implementation = function () {
        var log_str = "arguments:"
        for (var i = 0; i < arguments.length; i++) {
            log_str += " [" + i + "]" + arguments[i]
        }
 
        console.log("3 c.a()", log_str);
 
        return clazz.a.apply(this, arguments);
    }
});
 
Java.perform(function () {
    var clazz = Java.use('com.tencent.mm.plugin.messenger.foundation.f');
    clazz.a.implementation = function () {
        var log_str = "arguments:"
        for (var i = 0; i < arguments.length; i++) {
            log_str += " [" + i + "]" + arguments[i]
        }
 
        console.log("2 f.a()", log_str);
 
        return clazz.a.apply(this, arguments);
    }
});
 
Java.perform(function () {
    var clazz = Java.use('com.tencent.mm.plugin.zero.c');
    clazz.a.implementation = function () {
        var log_str = "arguments:"
        for (var i = 0; i < arguments.length; i++) {
            log_str += " [" + i + "]" + arguments[i]
        }
 
        console.log("1 c.a()", log_str);
 
        return clazz.a.apply(this, arguments);
    }
});

```

对方发送一条消息

```
1 c.a() arguments: [0]com.tencent.mm.protocal.protobuf.aaf@84db3f8 [1]false
2 f.a() arguments: [0]com.tencent.mm.protocal.protobuf.aaf@84db3f8 [1][object Object] [2]false
3 c.a() arguments: [0]com.tencent.mm.protocal.protobuf.aaf@84db3f8 [1][object Object] [2]false [3][object Object]
4 c.processAddMsg() arguments: [0]AddMsgInfo(263938257), get[false], fault[false], up[false] fixTime[0] [1][object Object]

```

对方撤回该消息

```
1 c.a() arguments: [0]com.tencent.mm.protocal.protobuf.aaf@d4208e1 [1]false
2 f.a() arguments: [0]com.tencent.mm.protocal.protobuf.aaf@d4208e1 [1][object Object] [2]false
3 c.a() arguments: [0]com.tencent.mm.protocal.protobuf.aaf@d4208e1 [1][object Object] [2]false [3][object Object]
4 c.processAddMsg() arguments: [0]AddMsgInfo(47744518), get[false], fault[false], up[false] fixTime[0] [1][object Object]
5 b.b() arguments: [0]AddMsgInfo(47744518), get[false], fault[false], up[false] fixTime[0]
6 ch.b() arguments: [0]AddMsgInfo(47744518), get[false], fault[false], up[false] fixTime[0]
7 f.a() arguments: [0]revokemsg [1][object Object] [2]AddMsgInfo(47744518), get[false], fault[false], up[false] fixTime[0]
8 handleRevokeMsgBySvrId arguments: [0]796494781522958800

```

#### 4 c.processAddMsg()

发现是从`processAddMsg()`方法开始不一样的，也就是最先在这里判断是否为撤回消息的

 

而判断是否进入第五层的消息是靠`v2!=null`来判断的，而 v2 来自`v3.vkP`，v3 来自参数一`arg10.gpg`

 

![](https://forgo7ten.github.io/2021081401/image-20210816201709925.png)

 

经调试，普通文本消息的`v3.vkP`为`1`，而撤回消息的`v3.vkP`为`10002`

 

![](https://forgo7ten.github.io/2021081401/image-20210816210123382.png)

 

而这恰好有个`10002`，不过并没有详细分析这里

#### 5 b.b()

进入调用的`b(arg10)`方法，也就是`com.tencent.mm.s.b.b()`没发现啥东西

 

![](https://forgo7ten.github.io/2021081401/image-20210816210228078.png)

 

**因为正常消息不会执行这个方法，当直接将该方法替换为空的时候，已经可以实现防撤回**:joy: 暴力 yyds，但是没有撤回提示，还得继续分析

 

![](https://forgo7ten.github.io/2021081401/Image-20210816211759.jpg)

#### 6 ch.b()

直接进入下一层`com.tencent.mm.model.ch.b`方法，在一开始又对`.vkP`做了一次判断

 

![](https://forgo7ten.github.io/2021081401/image-20210816210702683.png)

 

至于`10001`暂不知作用。对于`10001`只有那一点处理，之后就`return null`了；而 default 分支最后也是`return null`，可以得知`.vkp`的含义是`msgType`

 

可以知道这一个函数都是对撤回消息也就是`msgType==10002`进行处理的

 

然后该方法下面都是对 v5 的判断逻辑，而`v5=v0.GYI.IYz`

 

将`v0.GYI`json 化输出，得到

```
// com.tencent.mm.protocal.protobuf.cv
{
    "CreateTime": 1629120000,
    "GYG": {
        "IYA": true,
        "IYz": "wxid_00000000000000",
        "includeUnKnownField": false
    },
    "GYH": {
        "IYA": true,
        "IYz": "wxid_00000000000000",
        "includeUnKnownField": false
    },
    "GYI": {
        "IYA": true,
        "IYz": "\u003csysmsg type\u003d\"revokemsg\"\u003e\n\t\u003crevokemsg\u003e\n\t\t\u003csession\u003ewxid_00000000000000\u003c/session\u003e\n\t\t\u003cmsgid\u003e1070000000\u003c/msgid\u003e\n\t\t\u003cnewmsgid\u003e600000000000000\u003c/newmsgid\u003e\n\t\t\u003creplacemsg\u003e\u003c![CDATA[\"Forgo7ten.\" 撤回了一条消息]]\u003e\u003c/replacemsg\u003e\n\t\u003c/revokemsg\u003e\n\u003c/sysmsg\u003e\n",
        "includeUnKnownField": false
    },
    "GYJ": 1,
    "GYK": {
        "hasBuffer": false,
        "hasILen": true,
        "iLen": 0,
        "includeUnKnownField": false
    },
    "GYN": 656800000,
    "nNf": 0,
    "vkP": 10002,
    "ysP": 1073900000,
    "ysR": 2498250000000000000,
    "data": [0,0,0,0],      // 这些消息的字节数据
    "includeUnKnownField": false
}

```

而 v5 也就是

```
 wxid_##############
                10739#####
                7829338############ 

```

尝试一下修改信息

```
function modify_revoke_str(old_revoke_str) {
    var message = "尝试撤回了一条消息";
 
    var revoke_msg = Java.use("java.lang.String").$new(message);
    var sub_str = Java.use("java.lang.String").$new("]]>")
    var index = old_revoke_str.indexOf(sub_str);
    var new_revoke_str = old_revoke_str.substring(0, index - 7) + revoke_msg + old_revoke_str.substring(index);
    return Java.use("java.lang.String").$new(new_revoke_str);
}

```

```
Java.perform(function () {
    Java.openClassFile("/data/local/tmp/r0gson.dex").load();
    const gson = Java.use('com.r0ysue.gson.Gson');
 
    var clazz = Java.use('com.tencent.mm.model.ch');
    clazz.b.overload('com.tencent.mm.ak.e$a').implementation = function () {
        var log_str = "arguments:"
        for (var i = 0; i < arguments.length; i++) {
            log_str += " [" + i + "]" + arguments[i]
        }
        console.log("====================");
        var v0 = Java.cast(arguments[0].gpg.value, Java.use("com.tencent.mm.protocal.protobuf.cv"));
        console.log(gson.$new().toJson(v0));
 
        function modify_revoke_str(old_revoke_str) {
            var message = "尝试撤回了一条消息";
 
            var revoke_msg = Java.use("java.lang.String").$new(message);
            var sub_str = Java.use("java.lang.String").$new("]]>")
            var index = old_revoke_str.indexOf(sub_str);
            var new_revoke_str = old_revoke_str.substring(0, index - 7) + revoke_msg + old_revoke_str.substring(index);
            return Java.use("java.lang.String").$new(new_revoke_str);
        }
        var revoke_xml_obj = Java.cast(v0.GYI.value, Java.use("com.tencent.mm.protocal.protobuf.deo"));
 
        revoke_xml_obj.IYz.value = modify_revoke_str(revoke_xml_obj.IYz.value);
        console.log(revoke_xml_obj.IYz.value);
        console.log("====================");
        console.log("6 ch.b()", log_str);
 
        return clazz.b.overload('com.tencent.mm.ak.e$a').apply(this, arguments);
    }
});

```

虽然消息还是被撤回了，但也说明这种修改提示信息的方式是可行的

 

![](https://forgo7ten.github.io/2021081401/img-1629127911174.png)

 

通过 hook`notifyDataSetChanged`，json 打印最后一条消息发现

 

撤回消息与撤回提示，仅有`field_content"`（文本内容）、`"field_type"`、`"exe"`字段不一样

 

其中正常消息的`field_type=1,exe=true`；撤回提示`field_type=10000,exe=false`

 

被撤回消息和撤回消息的提示的`field_msgSvrId`字段是一样的，同时对应着撤回提示 xml 的`<newmsgid>`字段

 

尝试修改撤回提示的`<newmsgid>`字段，发现没有什么用。直接就变成不撤回了。

##### 返回该函数的分析

大概是对 xml 进行分析处理的一些东西

 

![](https://forgo7ten.github.io/2021081401/image-20210816235316259.png)

 

之后就去了`com.tencent.mm.model.f.a()`方法

```
public final com.tencent.mm.ak.e.b a(String arg24, Map arg25, com.tencent.mm.ak.e.a arg26);

```

*   参数一为字符串 String，获取过程为上图中蓝框
    
    *   即获取的是 xml 中`sysmsg`的`type`属性
    *   撤回消息就为`"revokemsg"`
*   参数二是一个 XML 内容转成的 Map 数组
    
    ```
    {
        ".sysmsg.revokemsg.newmsgid": "4308###############",
        ".sysmsg.revokemsg": "\n\t",
        ".sysmsg.revokemsg.session": "wxid_##############",
        ".sysmsg": "\n",
        ".sysmsg.$type": "revokemsg",
        ".sysmsg.revokemsg.replacemsg": "\"Forgo7ten.\" 撤回了一条消息",
        ".sysmsg.revokemsg.msgid": "##########"
    }
    
    ```
    
*   参数三保存着原始的所有数据
    
    ```
    // com.tencent.mm.ak.e$a
    {
        "gpg": {
            "CreateTime": 1629120000,
            "GYG": {
                "IYA": true,
                "IYz": "wxid_00000000000000",
                "includeUnKnownField": false
            },
            "GYH": {
                "IYA": true,
                "IYz": "wxid_00000000000000",
                "includeUnKnownField": false
            },
            "GYI": {
                "IYA": true,
                "IYz": "\u003csysmsg type\u003d\"revokemsg\"\u003e\n\t\u003crevokemsg\u003e\n\t\t\u003csession\u003ewxid_00000000000000\u003c/session\u003e\n\t\t\u003cmsgid\u003e1070000000\u003c/msgid\u003e\n\t\t\u003cnewmsgid\u003e600000000000000\u003c/newmsgid\u003e\n\t\t\u003creplacemsg\u003e\u003c![CDATA[\"Forgo7ten.\" 撤回了一条消息]]\u003e\u003c/replacemsg\u003e\n\t\u003c/revokemsg\u003e\n\u003c/sysmsg\u003e\n",
                "includeUnKnownField": false
            },
            "GYJ": 1,
            "GYK": {
                "hasBuffer": false,
                "hasILen": true,
                "iLen": 0,
                "includeUnKnownField": false
            },
            "GYN": 656800000,
            "nNf": 0,
            "vkP": 10002,
            "ysP": 1073900000,
            "ysR": 2498250000000000000,
            "data": [0, 0, 0, 0], // 这些消息的字节数据
            "includeUnKnownField": false
        },
        "hQb": false,
        "hQc": false,
        "hQd": false,
        "hQe": 0,
        "hQf": false,
        "what": 0
    }
    
    ```
    

#### 7 f.a()

这个方法里面是分块对参数一也就是`.sysmsg.$type`的值进行判断

 

![](https://forgo7ten.github.io/2021081401/image-20210817135838492.png)

 

之后发现将`.a()`方法置为空则消息不会撤回，继续跟入

 

![](https://forgo7ten.github.io/2021081401/image-20210817140049459.png)

 

在里面发现了数据库操作方法 update

 

![](https://forgo7ten.github.io/2021081401/image-20210817140125502.png)

 

之后嵌套了三个 update，最后来到了`updateWithOnConflict`方法

 

![](https://forgo7ten.github.io/2021081401/image-20210817142313024.png)

 

附上堆栈信息

```
=============================updateWithOnConflict Stack strat=======================
com.tencent.wcdb.database.SQLiteDatabase.updateWithOnConflict(Native Method)
com.tencent.wcdb.database.SQLiteDatabase.update(SourceFile:1726)
com.tencent.mm.storagebase.f.update(SourceFile:895)
com.tencent.mm.storagebase.h.update(SourceFile:601)
com.tencent.mm.storage.ca.a(SourceFile:2426)
com.tencent.mm.storage.ca.a(Native Method)
com.tencent.mm.model.f.a(SourceFile:2187)
com.tencent.mm.model.f.a(Native Method)
com.tencent.mm.model.ch.b(SourceFile:258)
com.tencent.mm.model.ch.b(Native Method)
com.tencent.mm.s.b.b(SourceFile:40)
com.tencent.mm.s.b.b(Native Method)
com.tencent.mm.plugin.messenger.foundation.c.processAddMsg(SourceFile:165)
com.tencent.mm.plugin.messenger.foundation.c.processAddMsg(Native Method)
com.tencent.mm.plugin.messenger.foundation.c.a(SourceFile:1059)
com.tencent.mm.plugin.messenger.foundation.c.a(Native Method)
com.tencent.mm.plugin.messenger.foundation.f.a(SourceFile:118)
com.tencent.mm.plugin.messenger.foundation.f.a(Native Method)
com.tencent.mm.plugin.zero.c.a(SourceFile:57)
com.tencent.mm.plugin.zero.c.a(Native Method)
com.tencent.mm.modelmulti.q$a$1.onTimerExpired(SourceFile:830)
com.tencent.mm.sdk.platformtools.aw.handleMessage(SourceFile:86)
com.tencent.mm.sdk.platformtools.aq$2.handleMessage(SourceFile:362)
android.os.Handler.dispatchMessage(Handler.java:106)
com.tencent.mm.sdk.platformtools.aq$2.dispatchMessage(SourceFile:350)
android.os.Looper.loop(Looper.java:164)
android.os.HandlerThread.run(HandlerThread.java:65)
=============================updateWithOnConflict Stack end=======================

```

经过一系列分析之后……（其实上面肯定都有分析的，只不过是很杂乱无从记录，这块就简写了

 

自己撤回消息提示的`msgType=10002`，别人撤回消息提示的`msgType=10000`

```
Java.perform(function () {
    var clazz = Java.use('com.tencent.wcdb.database.SQLiteDatabase');
    clazz.updateWithOnConflict.implementation = function () {
        // printStack("updateWithOnConflict");
        // var log_str = "arguments:"
        // for (var i = 0; i < arguments.length; i++) {
        //     log_str += " [" + i + "]" + arguments[i]
        // }
        // console.log("updateWithOnConflict", log_str);
        return clazz.updateWithOnConflict.apply(this, arguments);
    }
});

```

`updateWithOnConflict()`方法将参数进行 sql 的拼接，然后使用`new SQLiteStatement()`创建`SQLiteStatement`对象，再用`executeUpdateDelete()`来执行

 

Frida hook 一下`executeUpdateDelete()`方法

```
Java.perform(function () {
    Java.openClassFile("/data/local/tmp/r0gson.dex").load();
    const gson = Java.use('com.r0ysue.gson.Gson');
 
    var clazz = Java.use('com.tencent.wcdb.database.SQLiteStatement');
    clazz.executeUpdateDelete.overload().implementation = function () {
 
        // console.log("====================");
        // console.log("====================");
        console.log("7.1.1.1 executeUpdateDelete");
        console.log("msql:", this.mSql.value, "Args:", this.mBindArgs.value);
 
        return clazz.executeUpdateDelete.overload().apply(this, arguments);
    }
});

```

当消息被撤回时，得到了以下输出

```
7 f.a() arguments: [0]revokemsg [1][object Object] [2]AddMsgInfo(260954771), get[false], fault[false], up[false] fixTime[0]
7.1 ca.a() arguments: [0]234 [1]com.tencent.mm.storage.bz@8b10ad0
7.1.1 updateWithOnConflict arguments:
7.1.1.1 executeUpdateDelete
msql: UPDATE message SET msgId=?,type=?,content=? WHERE msgId=? Args: 234,10000,"Forgo7ten." 撤回了一条消息,234
7.1.1 updateWithOnConflict arguments:
7.1.1.1 executeUpdateDelete
msql: UPDATE rconversation SET msgType=?,flag=?,digestUser=?,digest=?,isSend=?,hasTrunc=?,unReadCount=?,conversationTime=?,content=?,username=?,status=? WHERE username=? Args: 10000,1629182140000,,"Forgo7ten." 撤回了一条消息,0,1,0,1629182140000,"Forgo7ten." 撤回了一条消息,wxid_00000000000000,3,wxid_00000000000000
7.1.1 updateWithOnConflict arguments:
7.1.1.1 executeUpdateDelete
msql: UPDATE rconversation SET UnReadInvite=?,atCount=? WHERE username= ? Args: 0,0,wxid_00000000000000

```

而我的目的主要是想 有撤回消息提示，同时不撤回消息

 

其中的参数`msgId`是本地的消息 id，只有第一条数据库操作指令是`WHERE msgId=?`的

 

所以实际上将被撤回消息 替换成撤回提示的就是这一条。

### 思路

当然 已经找到数据库操作了。。剩下的就想怎么操作就怎么操作了

 

我想的是 将该条撤回提示直接`INSERT`进去，不知道`msgId`会不会重复，重复的话可以挨个调整？（不知道为啥无法使用`select count(*)`）

 

或者让提示在指定位置的就是让他被撤回，然后聊天内容添加到撤回提示后边。原聊天内容可以通过`select content from message where msgId = ?`查询，缺点是只能文字内容；弥补的话只能判断非文本内容不让他撤回了

 

是想用 frida 来写的，毕竟 frida 比较方便。但是构造不出方法参数`Object []`来。

```
Java.array("Object",[]);
 
var args = Java.use("java.util.ArrayList").$new();
args.add(...);

```

这两种都尝试了，但都会报错

 

![](https://forgo7ten.github.io/2021081401/image-20210817185555176.png)

 

mark 一下，有知道的大佬麻烦分享一下呀。

### 脚本实现

这里只用 xposed 实现了，文本被撤回，但是有撤回提示；图片等等直接暴力不执行方法了... 因为撤回提示无法解释图片啥的

 

效果图

 

![](https://forgo7ten.github.io/2021081401/image-20210817235450332.png)

```
package com.forgo7ten.vxhook;
 
import android.util.Log;
 
import de.robv.android.xposed.IXposedHookLoadPackage;
import de.robv.android.xposed.XC_MethodHook;
import de.robv.android.xposed.XposedBridge;
import de.robv.android.xposed.XposedHelpers;
import de.robv.android.xposed.callbacks.XC_LoadPackage;
 
public class VxRevoke implements IXposedHookLoadPackage {
 
    @Override
    public void handleLoadPackage(XC_LoadPackage.LoadPackageParam lpparam) throws Throwable {
        if (!"com.tencent.mm".equals(lpparam.packageName)) {
            return;
        }
        final String className = "com.tencent.wcdb.database.SQLiteStatement";
        final String methodName = "executeUpdateDelete";
        Class SDClazz = XposedHelpers.findClass(className, lpparam.classLoader);
        XposedBridge.hookAllMethods(SDClazz, methodName, new XC_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                super.beforeHookedMethod(param);
                Log.d("VxHook", "executeUpdateDelete");
                Object mDatabase = XposedHelpers.getObjectField(param.thisObject, "mDatabase");
                String mSql = (String) XposedHelpers.getObjectField(param.thisObject, "mSql");
                Object[] mBindArgs = (Object[]) XposedHelpers.getObjectField(param.thisObject, "mBindArgs");
                if ("UPDATE message SET msgId=?,type=?,content=? WHERE msgId=?".equals(mSql) && ((String) mBindArgs[2]).contains("撤回了一条消息")) {
                    Object sm = XposedHelpers.newInstance(lpparam.classLoader.loadClass("com.tencent.wcdb.database.SQLiteStatement"),
                            mDatabase,
                            "select content from message where msgId = ?",
                            new Object[]{mBindArgs[3]});
                    String msg = (String) XposedHelpers.callMethod(sm, "simpleQueryForString");
                    if (!msg.contains("")) {
                        mBindArgs[2] = mBindArgs[2] + ":" + msg;
                        Log.d("VxHook", "对方撤回了：" + msg);
                    } else {
                        param.setResult(1);
                    }
                }
            }
        });
 
    }
} 
```

其实也想将被撤回消息下面的每条消息都`msgId+1`然后将撤回消息`insert`进去。但是`select count(*)`不能使用，这块知识掌握也不太熟练，就没实现……

vx 自动抢红包
--------

### 分析

#### 红包消息监听

自动抢红包肯定是先要监听到消息

 

之前已经分析到数据库，接收到消息肯定要插入或者更新数据库，于是 hook 了`com.tencent.wcdb.database.SQLiteDatabase.insertWithOnConflict`和`updateWithOnConflict`方法。观察接收消息时候的参数

 

当接收红包的时候，`insertWithOnConflict`被触发了 4 次，分别对`WalletLuckyMoney`,`LuckyMoneyDetailOpenRecord`,`message`,`AppMessage`数据库进行了插入

 

其中对`message`数据库操作参数字段

```
{
    "bizClientMsgId": "",
    "talker": "wxid_",
    "flag": 0,
    "bizChatId": -1,
    "msgId": 00,
    "type": 436207665,
    "content": "",
    "msgSvrId": 7222869153754354100,
    "lvbuffer": [123, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 125],
    "createTime": 1629,
    "reserved": "",
    "imgPath": "",
    "talkerId": 33,
    "isSend": 0,
    "msgSeq": 723600000,
    "status": 3
}

```

而`updateWithOnConflict`则被触发了三次，都是对`rconversation`数据库进行的操作

 

同时发现，普通消息在插入的时候，插入到`message`数据库的`"type":1`；图片`"type":3`；语音`"type:34"`；撤回消息上面说过了，为`"type":10000`(“你领取了 xxx 的红包” 也是这种形式)；红包`"type":436207665`；转账`"type":419430449`

 

同时发现了语音文件是藏在`/storage/emulated/0/Android/data/com.tencent.mm/MicroMsg/####/voice2/##/##/msg_###.amr`下，不知道为啥打不开

 

打开红包的时候，聊天界面也就是`message`数据库插入了一条消息`"领取了xxx的红包"`同时向`WalletLuckyMoney`数据库插入了两条消息

 

其中一条消息有`"receiveTime": ,"hbType":0,"receiveAmount":1,"receiveStatus":2,"hbStatus":4,"mNativeUrl":`等参数，而且如果点开了红包没有点击【开】按钮选择关闭的话，仍会插入上述参数，只是`"receiveTime":0,"hbType":0,"receiveAmount":0,"receiveStatus":0,"hbStatus":2,"mNativeUrl":`

#### 红包流程分析

还是先 DDMS 寻找按钮点击事件，发现在聊天页面点击红包消息触发的方法为`com.tencent.mm.ui.chatting.t$e.onClick()`

 

最后在`onDone`方法里调用了`startActivity`

 

![](https://forgo7ten.github.io/2021081401/image-20210818172734198.png)

 

堆栈信息为

```
com.tencent.mm.br.d$9.onDone(Native Method)
com.tencent.mm.br.d.a(SourceFile:909)
com.tencent.mm.br.d.b(SourceFile:267)
com.tencent.mm.br.d.a(SourceFile:148)
com.tencent.mm.br.d.b(SourceFile:129)
com.tencent.mm.ui.chatting.viewitems.g$b.c(SourceFile:728)
com.tencent.mm.ui.chatting.viewitems.d$d.a(SourceFile:582)
com.tencent.mm.ui.chatting.t$e.onClick(SourceFile:804)
com.tencent.mm.ui.chatting.t$e.onClick(Native Method)
android.view.View.performClick(View.java:6294)
android.view.View$PerformClick.run(View.java:24770)
android.os.Handler.handleCallback(Handler.java:790)
android.os.Handler.dispatchMessage(Handler.java:99)
android.os.Looper.loop(Looper.java:164)
android.app.ActivityThread.main(ActivityThread.java:6494)
java.lang.reflect.Method.invoke(Native Method)
com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:438)
com.android.internal.os.ZygoteInit.main(ZygoteInit.java:807)

```

其中`com.tencent.mm.ui.chatting.viewitems.g$b.c`中的判断，这个 url 是收到红包时插入`WalletLuckyMoney`数据库中的那条

 

![](https://forgo7ten.github.io/2021081401/image-20210818221446826.png)

 

在`com.tencent.mm.br.d.b(android.content.Context, java.lang.String, java.lang.String, android.content.Intent) : void`截取一下`intent`的参数

```
{
    "key_msgid": 0,
    "key_material_flag": 0,
    "key_has_story": false,
    "key_way": 1,
    "scene_id": 0,
    "key_cropname": "",
    "key_native_url": "wxpay://c2cbizmessagehandler/hongbao/receivehongbao?msgtype",
    "key_emoji_md5": "",
    "key_receive_envelope_md5": "",
    "key_receive_envelope_url": "",
    "key_detail_envelope_md5": "",
    "key_detail_envelope_url": "",
    "key_username": "wxid_00000000000000"
}

```

而红包界面的【开】按钮是在`LuckyMoneyNotHookReceiveUI$10.onClick()`这里触发的

 

![](https://forgo7ten.github.io/2021081401/image-20210818214917150.png)

 

然后进入了`auQ()`方法

 

监听红包消息，然后让软件执行这些流程。我分析了一些，没有找到最直接的控制红包入账的方法（:joy: 分析躁了，开学补考得开始复习了）

### 思路

这里参考了 [skyun1314/WeChat-plug-in (github.com)](https://github.com/skyun1314/WeChat-plug-in)

 

监听`insertWithOnConflict`方法，当接收到红包消息的时候，先使用`com.tencent.mm.br.d.b(android.content.Context, java.lang.String, java.lang.String, android.content.Intent) : void`这个静态方法初始化`LuckyMoneyNotHookReceiveUI`，然后监听`onCreate()`方法，来执行`this.wrX`按钮的点击事件。

### 脚本实现

这样实现的缺点是会弹窗……，如果想实现不弹窗，估计要继续追着【开】按钮`onClick`方法进去找最主要的那个方法了: astonished::sleepy:

#### 效果图

![](https://forgo7ten.github.io/2021081401/auto_luckymoney.gif)

 

输出

```
收到红包
find instance :com.tencent.mm.ui.chatting.e.a@df7c891
content = com.tencent.mm.ui.LauncherUI@f03a980
click result: true
click result: true
click result: true

```

#### 代码实现

```
function main(){
    Java.perform(function () {
        Java.openClassFile("/data/local/tmp/r0gson.dex").load();
        const gson = Java.use('com.r0ysue.gson.Gson');
 
        var clazz = Java.use('com.tencent.wcdb.database.SQLiteDatabase');
        clazz.insertWithOnConflict.implementation = function () {
            // console.log("insertWithOnConflict");
            // console.log("[0]"+arguments[0],"[1]"+arguments[1],"[2]"+gson.$new().toJson(arguments[2]));
 
 
            if (arguments[0] == "message") {
                var values = Java.cast(arguments[2], Java.use("android.content.ContentValues"));
                var msg_type = values.get("type");
                if (msg_type == "436207665") {
                    console.log("收到红包");
                    var content;
                    Java.choose("com.tencent.mm.ui.chatting.e.a", {
                        onMatch: function (x) {
                            console.log("find instance :" + x);
                            content = x.LDE.value.getContext();
                            console.log("content =", content);
                        },
                        onComplete: function () {
                        }
                    });
                    open_luck_money(content, values);
 
                    // var intent = Java.cast(arguments[2],Java.use("android.content.Intent"));
                } else if (msg_type == "419430449") {
                    console.log("收到转账");
                }
 
            }
            return clazz.insertWithOnConflict.apply(this, arguments);
        }
    });
 
    function open_luck_money(content, values) {
        function xml_get_value(content, start, end) {
            var javaString = Java.use("java.lang.String");
            var str_start = javaString.$new(start);
            var str_end = javaString.$new(end);
            var str_content = javaString.$new(content);
            var start_index = str_content.indexOf(str_start);
            var end_index = str_content.indexOf(str_end);
            var result = str_content.substring(start_index+str_start.length(),end_index);
            return result;
 
        }
        var javaString = Java.use("java.lang.String");
        var intent = Java.use("android.content.Intent").$new();
        var v_content = values.get("content");
        intent.putExtra(javaString.$new("key_msgid"), Java.use("java.lang.Long").parseLong(""+values.get("msgId")));
        intent.putExtra(javaString.$new("key_way"), Java.use("java.lang.Integer").parseInt("1"));
        intent.putExtra(javaString.$new("key_has_story"), Java.use("java.lang.Boolean").FALSE.value);
        intent.putExtra(javaString.$new("key_username"), javaString.$new(values.get("talker")));
        intent.putExtra(javaString.$new("key_native_url"), javaString.$new(xml_get_value(v_content,"")));
 
        var luckymoney_str = javaString.$new("luckymoney");
        var luckymoneyUI_str = javaString.$new(".ui.LuckyMoneyNotHookReceiveUI");
        Java.use('com.tencent.mm.br.d').b(content, luckymoney_str, luckymoneyUI_str,intent);
    }
 
 
    Java.perform(function () {
        var clazz = Java.use('com.tencent.mm.plugin.luckymoney.ui.LuckyMoneyNotHookReceiveUI');
        clazz.onSceneEnd.implementation = function () {
            var result = clazz.onSceneEnd.apply(this, arguments);
            console.log("click result:",this.wrX.value.performClick());
            return result;
        }
    });
 
 
}
 
setImmediate(main);

```

参考文章
----

[微信 Log 日志分析——初步探索 (toutiao.com)](https://www.toutiao.com/i6480401747104760333/)

 

[简单分析骰子流程 - 『移动安全区』 - 吾爱破解 - LCG - LSG | 安卓破解 | 病毒分析 | www.52pojie.cn](https://www.52pojie.cn/thread-1097114-1-1.html)

 

[某聊天软件撤回流程的分析与防撤回实现 - 『移动安全区』 - 吾爱破解 - LCG - LSG | 安卓破解 | 病毒分析 | www.52pojie.cn](https://www.52pojie.cn/thread-1097737-1-1.html)

 

[[原创]Xposed 第二课 (微信篇) 聊天界面修改文字 - Android 安全 - 看雪论坛 - 安全社区 | 安全招聘 | bbs.pediy.com](https://bbs.pediy.com/thread-226260.htm)

 

[[原创]Xposed 第三课 (微信篇) 防止好友消息撤回 - Android 安全 - 看雪论坛 - 安全社区 | 安全招聘 | bbs.pediy.com](https://bbs.pediy.com/thread-226674.htm)

 

[[下载] 微信 6.6.1 Xposed 模块 包含 主动发消息 防撤回 抢红包 骰子作弊 模拟位置 步数最高 - Android 安全 - 看雪论坛 - 安全社区 | 安全招聘 | bbs.pediy.com](https://bbs.pediy.com/thread-225878.htm)

 

深表谢意

 

代码啥的格式有些乱，太多了也不好改，麻烦大家手动格式化下。或者移步我的个人博客 [vx 逆向分析随笔 | Forgo7ten'blog](https://forgo7ten.github.io/2021081401/) 见谅啦

[[注意] 招人！base 上海，课程运营、市场多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)

最后于 2 小时前 被 Forgo7ten 编辑 ，原因： 修改好代码格式

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#HOOK 注入](forum-161-1-125.htm)
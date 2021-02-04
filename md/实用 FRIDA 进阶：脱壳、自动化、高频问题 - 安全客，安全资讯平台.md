> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.anquanke.com](https://www.anquanke.com/post/id/197670)

[![](https://p1.ssl.qhimg.com/t0167bf11e82f16975b.png)](https://p1.ssl.qhimg.com/t0167bf11e82f16975b.png)

前面我们聊到了`Frida`在内存漫游、`hook anywhere`、抓包等场景中地用法，今天我们聊`Frida`在脱壳、自动化的用法以及经常被问到的高频问题。

`Frida`系列文章地址：[https://github.com/r0ysue/AndroidSecurityStudy](https://github.com/r0ysue/AndroidSecurityStudy)

1 Frida 用于脱壳
------------

安全工程师在拿到应用评测的任务之后，第一件事情是抓到他的收包发包，第二件事情应该就是拿到它的`apk`，打开看看里面是什么内容，如果不幸它加了壳，可能打开就是这样的场景，见下图，什么内容都看不到，这时候就要首先对它进行脱壳。

[![](https://p0.ssl.qhimg.com/t012ee812f4a691f619.jpg)](https://p0.ssl.qhimg.com/t012ee812f4a691f619.jpg)

壳的种类非常多，根据其种类不同，使用的技术也不同，这里稍微简单分个类：

*   一代整体型壳：采用 Dex 整体加密，动态加载运行的机制；
*   二代函数抽取型壳：粒度更细，将方法单独抽取出来，加密保存，解密执行；
*   三代 VMP、Dex2C 壳：独立虚拟机解释执行、语义等价语法迁移，强度最高。

先说最难的`Dex2C`目前是没有办法还原的，只能跟踪进行分析；`VMP`虚拟机解释执行保护的是映射表，只要心思细、功夫深，是可以将映射表还原的；二代壳函数抽取目前是可以从根本上进行还原的，`dump`出所有的运行时的方法体，填充到`dump`下来的`dex`中去的，这也是 [`fart`](https://bbs.pediy.com/user-632473.htm)的核心原理；最后也就是目前我们推荐的几个内存中搜索和`dump`出`dex`的`Frida`工具，在一些场景中可以满足大家的需求。

### 1.1 文件头搜`dex`

地址是：[https://github.com/r0ysue/frida_dump](https://github.com/r0ysue/frida_dump)

```
# frida -U --no-pause -f com.xxxxxx.xxxxxx  -l dump_dex.js
     ____
    / _  |   Frida 12.8.9 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://www.frida.re/docs/home/
Spawned `com.xxxxx.xxxxx`. Resuming main thread!                
[Google Pixel::com.xxxxx.xxxxx]-> [dlopen:] libart.so
_ZN3art11ClassLinker11DefineClassEPNS_6ThreadEPKcmNS_6HandleINS_6mirror11ClassLoaderEEERKNS_7DexFileERKNS9_8ClassDefE 0x7adcac4f74
[DefineClass:] 0x7adcac4f74
[find dex]: /data/data/com.xxxxx.xxxxx/files/7abfc00000_8341c4.dex
[dump dex]: /data/data/com.xxxxx.xxxxx/files/7abfc00000_8341c4.dex
[find dex]: /data/data/com.xxxxx.xxxxx/files/7ac4096000_6e6c8.dex
[dump dex]: /data/data/com.xxxxx.xxxxx/files/7ac4096000_6e6c8.dex 
[find dex]: /data/data/com.xxxxx.xxxxx/files/7ac37c4028_8781c4.dex
[dump dex]: /data/data/com.xxxxx.xxxxx/files/7ac37c4028_8781c4.dex


```

其核心逻辑原理就是下面一句话`magic.indexOf("dex") == 0`，只要文件头中含有魔数`dex`，就把它`dump`下来。

```
if (dex_maps[base] == undefined) {
    dex_maps[base] = size;
    var magic = ptr(base).readCString();
    if (magic.indexOf("dex") == 0) {
        var process_name = get_self_process_name();
        if (process_name != "-1") {
            var dex_path = "/data/data/" + process_name + "/files/" + base.toString(16) + "_" + size.toString(16) + ".dex";
            console.log("[find dex]:", dex_path);
            var fd = new File(dex_path, "wb");
            if (fd && fd != null) {
                var dex_buffer = ptr(base).readByteArray(size);
                fd.write(dex_buffer);
                fd.flush();
                fd.close();
                console.log("[dump dex]:", dex_path);

            }
        }
    }
}


```

### 1.2 DexClassLoader：objection

安卓只能使用继承自`BaseDexClassLoader`的两种`ClassLoader`，一种是`PathClassLoader`，用于加载系统中已经安装的`apk`；一种就是`DexClassLoader`，加载未安装的`jar`包或`apk`。

可以用`objcetion`直接在堆上暴力搜索所有的`dalvik.system.DexClassLoader`实例，效果见下图：

```
# android heap search instances dalvik.system.DexClassLoader


```

[![](https://p3.ssl.qhimg.com/t0102d6fab0837f948d.png)](https://p3.ssl.qhimg.com/t0102d6fab0837f948d.png)

连热补丁都被搜出来了，在某些一代壳上效果不错。

### 1.3 暴力搜内存：DEXDump

地址：[https://github.com/hluwa/FRIDA-DEXDump](https://github.com/hluwa/FRIDA-DEXDump)

*   对于完整的`dex`，采用暴力搜索`dex035`即可找到。
*   而对于抹头的`dex`，通过匹配一些特征来找到，然后自动修复文件头。

效果非常好：

```
root@roysuekali:~/Desktop/FRIDA-DEXDump
[DEXDump]: found target [7628] com.xxxxx.xxxxx
[DEXDump]: DexSize=0x8341c4, SavePath=./com.xxxxx.xxxxx/0x7abfc00000.dex
[DEXDump]: DexSize=0x8341c4, SavePath=./com.xxxxx.xxxxx/0x7ac0600000.dex
root@roysuekali:~/Desktop/FRIDA-DEXDump
8.3M    com.xxxxx.xxxxx/0x7abfc00000.dex
8.3M    com.xxxxx.xxxxx/0x7ac0600000.dex
root@roysuekali:~/Desktop/FRIDA-DEXDump
com.xxxxx.xxxxx/0x7abfc00000.dex: Dalvik dex file version 035
com.xxxxx.xxxxx/0x7ac0600000.dex: Dalvik dex file version 035


```

打开`dump`下来的`dex`，非常完整，可以用`jadx`直接解析。用`010`打开可以看到完整的文件头——`dexn035`，其实现代码也是简单粗暴，直接搜索：`64 65 78 0a 30 33 35 00`：

```
Memory.scanSync(range.base, range.size, "64 65 78 0a 30 33 35 00").forEach(function (match) {
var range = Process.findRangeByAddress(match.address);

if (range != null && range.size < match.address.toInt32() + 0x24 - range.base.toInt32()) {
    return;
}

var dex_size = match.address.add("0x20").readInt();

if (range != null) {

    if (range.file && range.file.path
        && (range.file.path.startsWith("/data/app/")
            || range.file.path.startsWith("/data/dalvik-cache/")
            || range.file.path.startsWith("/system/"))) {
        return;
    }

    if (match.address.toInt32() + dex_size > range.base.toInt32() + range.size) {
        return;
    }
}


```

还有一部分想要特征匹配的功能还在实现中：

```
if (
    range.size >= 0x60
    && range.base.readCString(4) != "dexn"
    && range.base.add(0x20).readInt() <= range.size 
    
    && range.base.add(0x34).readInt() < range.size
    && range.base.add(0x3C).readInt() == 112 
) {
    result.push({
        "addr": range.base,
        "size": range.base.add(0x20).readInt()
    });
}


```

### 1.4 暴力搜内存：objection

既然直接使用`Frida`的`API`可以暴力搜索内存，那么别忘了我们上面介绍过的`objection`也可以暴力搜内存。

```
# memory search "64 65 78 0a 30 33 35 00"


```

[![](https://p4.ssl.qhimg.com/t01f1d3b104c5e8df8f.jpg)](https://p4.ssl.qhimg.com/t01f1d3b104c5e8df8f.jpg)

搜出来的`offset`是：`0x79efc00000`，大小是`c4 41 83 00`，也就是`0x8341c4`，转化成十进制就是`8602052`，最后`dump`下来的内容与`FRIDA-DEXDump`脱下来的一模一样，拖到`jdax`里可以直接解析。

2 Frida 用于自动化
-------------

在`Frida`出现之前，没有任何一款工具，可以在语言级别支持直接在电脑上调用`app`中的方法。像`Xposed`是纯`Java`，根本就没有电脑上运行的版本；各种`Native`框架也是一样，都是由`C/C++/asm`实现，根本与电脑毫无关系。

而`Frida`主要是一款在电脑上操作的工具，其本身就决定了其 “高并发”、“多联通”、“自动化” 等特性：

*   “高并发”：同时操作多台手机，同时调用多个手机上的多个`app`中的算法；
*   “多联通”：电脑与手机互联互通，手机上处理不了的在电脑上处理、反之亦然；
*   “自动化”：手机电脑互相协同，实现横跨桌面、移动平台协同自动化利器。

### 2.1 连接多台设备

`Frida`用于自动化的场景中，必然是不可能在终端敲`frida-tools`里的那些命令行工具的，有人说可以将这些命令按顺序写成脚本，那为啥不直接写成`python`脚本呢？枉费大胡子叔叔（`Frida`的作者 oleavr 的头像）为我们写好了`Python bindings`，我们只需要直接调用即可享受。

`Python bindings`在安装好`frida-tools`的时候已经默认安装在我们的电脑上了，可以直接使用。

连接多台设备非常简单，如果是`USB`口直接连接的，只要确保`adb`已经连接上，如果是网络调试的，也要用`adb connect`连接上，并且都开启`frida server`，键入`adb devices`或者`frida-ls-devices`命令时多台设备的`id`都会出现，最终可以使用`frida.get_device(id)`的`API`来选择设备，如下图所示。

[![](https://p2.ssl.qhimg.com/t01bf504ce84f284ade.jpg)](https://p2.ssl.qhimg.com/t01bf504ce84f284ade.jpg)

### 2.2 互联互通

互联互通是指把`app`中捕获的内容传输到电脑上，电脑上处理结束后再发回给`app`继续处理。看似很简单的一个功能，目前却仅有`Frida`可以实现。

比如说我们有这样一个`app`，其中最核心的地方在于判断用户是否为`admin`，如果是，则直接返回错误，禁止登陆。如果不是，则把用户和密码上传到服务器上进行验证登录操作，其核心代码逻辑如下：

```
public class MainActivity extends AppCompatActivity {

    EditText username_et;
    EditText password_et;
    TextView message_tv;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        password_et = (EditText) this.findViewById(R.id.editText2);
        username_et = (EditText) this.findViewById(R.id.editText);
        message_tv = ((TextView) findViewById(R.id.textView));

        this.findViewById(R.id.button).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {

                if (username_et.getText().toString().compareTo("admin") == 0) {
                    message_tv.setText("You cannot login as admin");
                    return;
                }
                
                message_tv.setText("Sending to the server :" + Base64.encodeToString((username_et.getText().toString() + ":" + password_et.getText().toString()).getBytes(), Base64.DEFAULT));

            }
        });

    }
}


```

运行起来的效果如下图：

[![](https://p3.ssl.qhimg.com/t0166872c8261546c36.png)](https://p3.ssl.qhimg.com/t0166872c8261546c36.png)

我们的目标就是在电脑上 “得到” 输入框输入的内容，并且修改其输入的内容，并且 “传输” 给安卓机器，使其通过验证。也就是说，我们的目标是哪怕输入`admin`的账户名和密码，也可以绕过本地校验，进行服务器验证登陆的操作。

所以最终我们的`hook`代码的逻辑就是，截取输入，传输给电脑，暂停执行，得到电脑传回的数据之后，继续执行，用`js`来写就这么写：

```
Java.perform(function () {
    var tv_class = Java.use("android.widget.TextView");
    tv_class.setText.overload("java.lang.CharSequence").implementation = function (x) {
        var string_to_send = x.toString();
        var string_to_recv;
        send(string_to_send); 
        recv(function (received_json_object) {
            string_to_recv = received_json_object.my_data
            console.log("string_to_recv: " + string_to_recv);
        }).wait(); 
        return this.setText(string_to_recv);
    }
});


```

在电脑上的处理流程是，将接受到的`JSON`数据解析，提取出其中的密码部分保持不变，然后将用户名替换成`admin`，这样就实现了将`admin`和`password`发送给服务器的结果。我们的代码如下：

```
import time
import frida

def my_message_handler(message, payload):
    print message
    print payload
    if message["type"] == "send":
        print message["payload"]
        data = message["payload"].split(":")[1].strip()
        print 'message:', message
        data = data.decode("base64") 
        user, pw = data.split(":") 
        data = ("admin" + ":" + pw).encode("base64") 
        print "encoded data:", data
        script.post({"my_data": data})  
        print "Modified data sent"

device = frida.get_usb_device()
pid = device.spawn(["com.roysue.demo04"])
device.resume(pid)
time.sleep(1)
session = device.attach(pid)
with open("s4.js") as f:
    script = session.create_script(f.read())
script.on("message", my_message_handler)  
script.load()
raw_input()


```

同样很多手机上无法处理的数据，也可以编码后发送到电脑上进行处理，比如处理`GBK`编码的中文字符集数据，再比如对`dump`下来的内存或`so`进行二次解析还原等，这些在`js`几乎是无法处理的（或难度非常大），但是到了电脑上就易如反掌，用`python`导入几个库就可以。

在一些 (网络) 接口的模糊测试的场景中，一些字典和畸形数据的构造也会在电脑上完成，`app`端最多作为执行端接受和发送这些数据，这时候也需要使用到`Frida`互联互通动态修改的功能。

### 2.3 远程调用（RPC）

在脚本里定义一个导出函数，并用`rpc.exports`的字典进行声明：

```
function callSecretFun() { 
    Java.perform(function () {
        
        
        Java.choose("com.roysue.demo02.MainActivity", {
            onMatch: function (instance) {
                console.log("Found instance: " + instance);
                console.log("Result of secret func: " + instance.secret());
            },
            onComplete: function () { }
        });
    });
}
rpc.exports = {
    callsecretfunction: callSecretFun 
};


```

在电脑上就可以直接在`py`代码里调用这个方法：

```
import time
import frida

def my_message_handler(message, payload):
    print message
    print payload

device = frida.get_usb_device()
pid = device.spawn(["com.roysue.demo02"])
device.resume(pid)
time.sleep(1)
session = device.attach(pid)
with open("s3.js") as f:
    script = session.create_script(f.read())
script.on("message", my_message_handler)
script.load()

command = ""
while 1 == 1:
    command = raw_input("Enter command:n1: Exitn2: Call secret functionnchoice:")
    if command == "1":
        break
    elif command == "2": 
        script.exports.callsecretfunction()


```

最终效果就是按一下`2`，`function callSecretFun()`就会被执行一次，并且结果会显示在电脑上的`py`脚本里，以供后续继续处理，非常方便。

笔者有一位朋友甚至将该接口使用`python`的`flask`框架暴露出去，让网络里的每个人都可以调用该方法，给自己的发包进行签名，可用说是一个需求非常庞大的场景。

3 Frida 更多技巧
------------

最后收集和整理一下大家在学习`Frida`的过程中可能会遇到的几个高频问题，以餮读者。

### 3.1 必须上版本管理

`Frida`从面世到现在已经有四五年了，大概 17~18 年那会儿开始火爆起来，大量的脚本和工具代码都是那段时间写出来的，而`Frida`又升级特别快，新的`Frida`对老的脚本兼容性不是很好，见下图最新的`Frida`运行老的脚本，日志格式已经乱掉了，而老版本（`12.4.8`）就没问题，见图 2-18。如果要运行一些两三年历史的代码，必然需要安装两三年前左右的版本，这样才能跑起来，并且不出错。

[![](https://p4.ssl.qhimg.com/t0192d8454aca537187.png)](https://p4.ssl.qhimg.com/t0192d8454aca537187.png)

版本管理用`pyenv`即可，熟练使用`pyenv`可以基本上满足同时安装几十个`Frida`版本的需求。

### 3.2 反调试基本思路

几个最基本的思路，首先`frida-server`的文件名改掉，类似于`frida-server-12.8.9-android-arm64`这样的文件名，我一般改成`fs1289amd64`，当然读者可以想改成啥就改成啥。

有些反调试还会检查端口，比如`frida-server`的默认端口是`27042`，这个端口一般不会有人用，如果`27042`端口打开并且正在监听，反调试就会工作，可以把端口改成非标准端口，方法下一小节就讲。

最后还有一种通过`Frida`内存特征对`maps`中`elf`文件进行扫描匹配特征的反调试方法，支持`frida-gadget`和`frida-server`，项目地址在[这里](https://github.com/qtfreet00/AntiFrida)。

其核心代码如下：

```
void *check_loop(void *) {
    int fd;
    char path[256];
    char perm[5];
    unsigned long offset;
    unsigned int base;
    long end;
    char buffer[BUFFER_LEN];
    int loop = 0;
    unsigned int length = 11;
    
    unsigned char frida_rpc[] =
            {

                    0xfe, 0xba, 0xfb, 0x4a, 0x9a, 0xca, 0x7f, 0xfb,
                    0xdb, 0xea, 0xfe, 0xdc
            };

    for (unsigned char &m : frida_rpc) {
        unsigned char c = m;
        c = ~c;
        c ^= 0xb1;
        c = (c >> 0x6) | (c << 0x2);
        c ^= 0x4a;
        c = (c >> 0x6) | (c << 0x2);
        m = c;
    }
    
    LOGI("start check frida loop");
    while (loop < 10) {
        fd = wrap_openat(AT_FDCWD, "/proc/self/maps", O_RDONLY, 0);
        if (fd > 0) {
            while ((read_line(fd, buffer, BUFFER_LEN)) > 0) {

                
                if (sscanf(buffer, "%x-%lx %4s %lx %*s %*s %s", &base, &end, perm, &offset, path) !=
                    5) {
                    continue;
                }
                if (perm[0] != 'r') continue;
                if (perm[3] != 'p') continue; 
                if (0 != offset) continue;
                if (strlen(path) == 0) continue;
                if ('[' == path[0]) continue;
                if (end - base <= 1000000) continue;
                if (wrap_endsWith(path, ".oat")) continue;
                if (elf_check_header(base) != 1) continue;
                if (find_mem_string(base, end, frida_rpc, length) == 1) {
                    
                    LOGI("frida found in memory!");
#ifndef DEBUG
                    
                    wrap_kill(wrap_getpid(),SIGKILL);
#endif
                    
                    break;
                }
            }
        } else {
            LOGI("open maps error");
        }
        wrap_close(fd);
        
        loop++;
        sleep(3);
    }
    return nullptr;
}


void anti_frida_loop() {
    pthread_t t;
    
    if (pthread_create(&t, nullptr, check_loop, (void *) nullptr) != 0) {
        exit(-1);
    };
    pthread_detach(t);
}


```

想过这种反调试，得找到反调试在哪个`so`的哪里，`nop`掉创建`check_loop`线程的地方，或者`nop`掉`kill`自己进程的地方，都可以。也可以直接`kill`掉反调试进程，笔者就曾经遇到过这种情况，`frida`命令注入后，`app`调不起来，这时候用`ps -e`命令查看多一个反调试进程，直接`kill`掉那个进程后，`app`就起来了，这个`app`是使用的一个大厂的加固服务，这个进程就是壳的一部分。

### 3.3 非标准端口连接

比如将`frida-server`启动在`6666`端口：

```
# ./fs1287amd64 -l 0.0.0.0:6666


```

使用`frida-tools`工具和`objection`分别连接的方法如下：

```
# frida-ps -H 192.168.1.102:6666
# objection -N -h 192.168.1.102 -p 6666 -g com.android.settings explore


```

效果如图所示：

[![](https://p0.ssl.qhimg.com/t01daa9eb908fc0e555.jpg)](https://p0.ssl.qhimg.com/t01daa9eb908fc0e555.jpg)

图 连接非标准端口

在`python bindings`中连接的话，会稍微复杂一点点，因为`python bindings`只认`adb`，所以要通过`adb`命令将手机的`6666`端口映射到电脑的`27042`端口：

```
$ adb forward tcp:27042 tcp:6666


```

这样`python bindings`也可以正常使用了。

### 3.4 打印`byte[]``[B`

`ByteString.of`是用来把`byte[]`数组转成`hex`字符串的函数, 安卓系统自带`ByteString`，`app`里面没有也没关系，可以去系统里面拿，这里给个小案例：

```
var ByteString = Java.use("com.android.okhttp.okio.ByteString");
var j = Java.use("xxxxxxx.business.comm.j");
j.x.implementation = function() {
    var result = this.x();
    console.log("j.x:", ByteString.of(result).hex());
    return result;
};

j.a.overload('[B').implementation = function(bArr) {
    this.a(bArr);
    console.log("j.a:", ByteString.of(bArr).hex());
};


```

### 3.5 `hook`管理子进程

经常有人会问，像那种`com.xxx.xxx:push`、`com.xxx.xxx:service`、`com.xxx.xxx:notification`、`com.xxx.xxx:search`这样的进程如何`hook`，或者说如何在其创建伊始进行`hook`，因为这样的进程一般都是由主进程`fork()`出来的。

这种的就要用到`Frida`最新的`Child gating`机制，可以参考我的这篇文章[《FRIDA 脚本系列（四）更新篇：几个主要机制的大更新》](https://www.anquanke.com/post/id/177597)，官方的完整代码在[这里](https://github.com/frida/frida-python/blob/master/examples/child_gating.py)。可以在进程创建之初对该进程进行控制和`hook`，已经很多人用了，效果很好，达成目标。

### 3.6 `hook`混淆方法名

有些方法名上了很强的混淆，如何处理？其实很简单，可以看上面`ZenTracer`的源码，`hook`类的所有子类，`hook`类的所有方法，并且`hook`方法的所有重载，我这里也提供了源码和文章，[《FRIDA 脚本系列（二）成长篇：动静态结合逆向 WhatsApp》](https://www.anquanke.com/post/id/169315)。

### 3.7 中文参数问题

`hook`某些方法的时候，发现传进来的参数竟然是中文的，如何打印出来？如果是`utf8`还好，`Frida`的`CLI`也是直接支持`utf8`的，如果是`GBK`字符集的，目前没有找到在`js`里进行打印的方法，可以`send()`到电脑上进行打印。

### 3.8 `hook`主动注册

使用`Frida`来`hook` `JNI`的一些函数，打印出主动调用的执行路径。下面是`hook` `Google play Market`的例子：

```
frida -U --no-pause -f com.android.vending -l hook_RegisterNatives.js
     ____
    / _  |   Frida 12.6.13 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at http://www.frida.re/docs/home/
Spawning `com.android.vending`...                                      
GetFieldID is at  0xf1108e4d _ZN3art3JNI10GetFieldIDEP7_JNIEnvP7_jclassPKcS6_
AllocObject is at  0xf10f1809 _ZN3art3JNI11AllocObjectEP7_JNIEnvP7_jclass
GetMethodID is at  0xf10f3175 _ZN3art3JNI11GetMethodIDEP7_JNIEnvP7_jclassPKcS6_
NewStringUTF is at  0xf111fc71 _ZN3art3JNI12NewStringUTFEP7_JNIEnvPKc
GetObjectClass is at  0xf10f2841 _ZN3art3JNI14GetObjectClassEP7_JNIEnvP8_jobject
RegisterNatives is at  0xf11301fd _ZN3art3JNI15RegisterNativesEP7_JNIEnvP7_jclassPK15JNINativeMethodi
CallObjectMethod is at  0xf10f3745 _ZN3art3JNI16CallObjectMethodEP7_JNIEnvP8_jobjectP10_jmethodIDz
GetStaticFieldID is at  0xf111949d _ZN3art3JNI16GetStaticFieldIDEP7_JNIEnvP7_jclassPKcS6_
GetStaticMethodID is at  0xf110e6d1 _ZN3art3JNI17GetStaticMethodIDEP7_JNIEnvP7_jclassPKcS6_
GetStringUTFChars is at  0xf11203e1 _ZN3art3JNI17GetStringUTFCharsEP7_JNIEnvP8_jstringPh
ReleaseStringUTFChars is at  0xf11207fd _ZN3art3JNI21ReleaseStringUTFCharsEP7_JNIEnvP8_jstringPKc
FindClass is at  0xf10ec7a1 _ZN3art3JNI9FindClassEP7_JNIEnvPKc
Spawned `com.android.vending`. Resuming main thread!                   
[Google Pixel XL::com.android.vending]-> [RegisterNatives] method_count: 0x6
[RegisterNatives] java_class: org.chromium.base.CommandLine name: nativeInit sig: ([Ljava/lang/String;)V fnPtr: 0xd454f349 module_name: libcronet.76.0.3809.21.so module_base: 0xd441f000 offset: 0x130349
[RegisterNatives] java_class: org.chromium.base.CommandLine name: nativeHasSwitch sig: (Ljava/lang/String;)Z fnPtr: 0xd454f369 module_name: libcronet.76.0.3809.21.so module_base: 0xd441f000 offset: 0x130369
[RegisterNatives] java_class: org.chromium.base.CommandLine name: nativeGetSwitchValue sig: (Ljava/lang/String;)Ljava/lang/String; fnPtr: 0xd454f3bd module_name: libcronet.76.0.3809.21.so module_base: 0xd441f000 offset: 0x1303bd
[RegisterNatives] java_class: org.chromium.base.CommandLine name: nativeAppendSwitch sig: (Ljava/lang/String;)V fnPtr: 0xd454f461 module_name: libcronet.76.0.3809.21.so module_base: 0xd441f000 offset: 0x130461
[RegisterNatives] java_class: org.chromium.base.CommandLine name: nativeAppendSwitchWithValue sig: (Ljava/lang/String;Ljava/lang/String;)V fnPtr: 0xd454f499 module_name: libcronet.76.0.3809.21.so module_base: 0xd441f000 offset: 0x130499
[RegisterNatives] java_class: org.chromium.base.CommandLine name: nativeAppendSwitchesAndArguments sig: ([Ljava/lang/String;)V fnPtr: 0xd454f4f1 module_name: libcronet.76.0.3809.21.so module_base: 0xd441f000 offset: 0x1304f1
[RegisterNatives] method_count: 0x3
[RegisterNatives] java_class: org.chromium.base.EarlyTraceEvent name: nativeRecordEarlyEvent sig: (Ljava/lang/String;JJIJ)V fnPtr: 0xd454f94d module_name: libcronet.76.0.3809.21.so module_base: 0xd441f000 offset: 0x13094d
[RegisterNatives] java_class: org.chromium.base.EarlyTraceEvent name: nativeRecordEarlyStartAsyncEvent sig: (Ljava/lang/String;JJ)V fnPtr: 0xd454fa3d module_name: libcronet.76.0.3809.21.so module_base: 0xd441f000 offset: 0x130a3d
[RegisterNatives] java_class: org.chromium.base.EarlyTraceEvent name: nativeRecordEarlyFinishAsyncEvent sig: (Ljava/lang/String;JJ)V fnPtr: 0xd454fae5 module_name: libcronet.76.0.3809.21.so module_base: 0xd441f000 offset: 0x130ae5
[RegisterNatives] method_count: 0x4
[RegisterNatives] java_class: org.chromium.base.FieldTrialList name: nativeFindFullName sig: (Ljava/lang/String;)Ljava/lang/String; fnPtr: 0xd454fb8d module_name: libcronet.76.0.3809.21.so module_base: 0xd441f000 offset: 0x130b8d
[RegisterNatives] java_class: org.chromium.base.FieldTrialList name: nativeTrialExists sig: (Ljava/lang/String;)Z fnPtr: 0xd454fbff module_name: libcronet.76.0.3809.21.so module_base: 0xd441f000 offset: 0x130bff
[RegisterNatives] java_class: org.chromium.base.FieldTrialList name: nativeGetVariationParameter sig: (Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String; fnPtr: 0xd454fc2f module_name: libcronet.76.0.3809.21.so module_base: 0xd441f000 offset: 0x130c2f
[RegisterNatives] java_class: org.chromium.base.FieldTrialList name: nativeLogActiveTrials sig: ()V fnPtr: 0xd454fd1d module_name: libcronet.76.0.3809.21.so module_base: 0xd441f000 offset: 0x130d1d
[RegisterNatives] method_count: 0x2


```

源码地址：[https://github.com/lasting-yang/frida_hook_libart](https://github.com/lasting-yang/frida_hook_libart)

### 3.9 追踪`JNI API`

地址：[https://github.com/chame1eon/jnitrace](https://github.com/chame1eon/jnitrace)

[![](https://p3.ssl.qhimg.com/t0147cc44b8a36f61e4.png)](https://p3.ssl.qhimg.com/t0147cc44b8a36f61e4.png)

### 3.10 延迟`hook`

很多时候在带壳`hook`的时候，善用两个`frida`提供的延时`hook`机制：

*   `frida --no-pause`是进程直接执行，有时候会`hook`不到，如果把`--no-pause`拿掉，进入`CLI`之后延迟几秒再使用`%resume`恢复执行，就会`hook`到；
*   `js`中的`setTimeout(func, delay[, ...parameters])`函数，会延时`delay`毫秒来调用`func`，有时候不加延时会`hook`不到，加个几百到几千毫秒的延时就会`hook`到。
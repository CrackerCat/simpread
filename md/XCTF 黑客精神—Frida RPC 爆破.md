> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1414060-1-1.html) ![](https://avatar.52pojie.cn/data/avatar/001/00/27/01_avatar_middle.jpg)genliese  

1. 背景
-----

此题比较简单，网上也有各种各样的 wp。wp 可以分为两类，一类分析算法，另一类是爆破。此题采用的是异或算法，所以分析算法求 flag 是最快的。而爆破的话，可以重写成等价的代码，如 C++、python，或者直接采用主动调用的方式进行爆破，如 Frida 的主动调用。采用主动调用的好处是不用重写，而我采用 Frida RPC 进行主动调用的目的是想利用 python 丰富的库，方便爆破，缺点相对于直接用 JS 代码进行爆破是太慢了，慢了 200 多倍。

*   用到的工具：
    *   pixel 3 android 9.0（不能用模拟器，因为被 hook 的 libmyjni.so 只有 arm 架构的）
    *   jadx
    *   IDA
    *   Frida12.8
    *   Pycharm
    *   VSCode

2. 分析过程
-------

输入注册码，显示如下：

![](https://genliesephotos.genliese.cn/Markdown/xman_frida_rpc/input_registration_code.png)

在代码中搜索 "您的注册码已保存"，相关代码如下：

```
public class RegActivity extends Activity {
    private Button btn_reg;
    private EditText edit_sn;

    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_reg);
        this.btn_reg = (Button) findViewById(R.id.button1);
        this.edit_sn = (EditText) findViewById(R.id.editText1);
        this.btn_reg.setOnClickListener(new View.OnClickListener() {
            /* class com.gdufs.xman.RegActivity.AnonymousClass1 */

            public void onClick(View v) {
                String sn = RegActivity.this.edit_sn.getText().toString().trim();
                if (sn == null || sn.length() == 0) {
                    Toast.makeText(RegActivity.this, "您的输入为空", 0).show();
                    return;
                }
                ((MyApp) RegActivity.this.getApplication()).saveSN(sn);
                new AlertDialog.Builder(RegActivity.this).setTitle("回复").setMessage("您的注册码已保存").setPositiveButton("好吧", new DialogInterface.OnClickListener() {
                    /* class com.gdufs.xman.RegActivity.AnonymousClass1.AnonymousClass1 */

                    public void onClick(DialogInterface dialog, int which) {
                        Process.killProcess(Process.myPid());
                    }
                }).show();
            }
        });
    }
}

```

然后跳转到`saveSN`函数所在的类`MyApp`，代码如下：

```
public class MyApp extends Application {
    public static int m = 0;

    public native void initSN();

    public native void saveSN(String str);

    public native void work();

    static {
        System.loadLibrary("myjni");
    }

    public void onCreate() {
        initSN();
        Log.d("com.gdufs.xman m=", String.valueOf(m));
        super.onCreate();
    }
}

```

接着分析`libmyjni.so`文件，在`JNI_OnLoad`函数中注册了`initSN`、`saveSN`和`work`函数，代码如下：

![](https://genliesephotos.genliese.cn/Markdown/xman_frida_rpc/initsn_1.png)

![](https://genliesephotos.genliese.cn/Markdown/xman_frida_rpc/initsn_2.png)

### 2.1 分析`initSN`函数

首先分析`initSN`函数，其中`setValue`函数的作用是设置`com/gdufs/xman/MyApp`类的静态字段`m`的值，

![](https://genliesephotos.genliese.cn/Markdown/xman_frida_rpc/setValue.png)

![](https://genliesephotos.genliese.cn/Markdown/xman_frida_rpc/initsn.png)

`initSN`函数只做了一件事，即如果`/sdcard/reg.dat`文件的内容是 "EoPAoY62@ElRD"，`com/gdufs/xman/MyApp`类的静态字段`m`的值则设置为 1，否则设置为 0

通过对`com/gdufs/xman/MyApp`类的静态字段`m`交叉引用发现，如果`m`的值为 1，则显示**已注册**，所以我们怀疑输入的注册码通过一系列的计算后得到的值会保存到`/sdcard/reg.dat`文件中，如果得到的值为 "EoPAoY62@ElRD"，则输入的注册码即为 **flag**

![](https://genliesephotos.genliese.cn/Markdown/xman_frida_rpc/m_cross_reference.png)

### 2.2 分析 `saveSN`函数

`saveSN`也只干了一件事，把输入的注册码经过**异或运算**之后存到了`/sdcard/reg.dat`文件中

![](https://genliesephotos.genliese.cn/Markdown/xman_frida_rpc/savesn_1.png)

![](https://genliesephotos.genliese.cn/Markdown/xman_frida_rpc/savesn_2.png)

我们可以试试，直接把`/sdcard/reg.dat`文件的内容替换成 "EoPAoY62@ElRD"，会是什么效果，如下：

![](https://genliesephotos.genliese.cn/Markdown/xman_frida_rpc/enter_result.png)

则我们的 flag 格式为`xman{注册码}`

3. 编写脚本
-------

通过上面的分析过程可知，我们可以通过 Frida 主动调用的方式爆破出 flag，首先给出直接用 JS 代码进行爆破的脚本，再给出 RPC 爆破的脚本。

### 3.1 JS 代码爆破

```
var fputs_str = null;

function Hook() {
    Java.perform(function () {
        const imports = Module.enumerateImportsSync("libmyjni.so");
        const imports_len = imports.length;
        var fputs_addr = null;
        for (var i = 0; i < imports_len; i++) {
            if (imports[i].name == "fputs") {
                fputs_addr = imports[i].address;
                break;
            }
        }
        if (fputs_addr != null) {
            Interceptor.attach(fputs_addr, {
                onEnter: function (args) {
                    fputs_str = args[0].readCString();
                },
                onLeave: function (retval) {
                }
            })
        }
    })
}

function Invoke(try_str) {
    Java.perform(function () {
        Java.choose("com.gdufs.xman.MyApp", {
            onMatch: function (instance) {
                instance.saveSN(try_str);
            },
            onComplete: function () {
            }
        })
    })
}

function Main() {
    Hook();
    Java.perform(function () {
        const three_character_array = new Array("EoP", "AoY", "62@", "ElR");
        const last_character = "D";
        const my_dict = "abcdefghijklmnopqrstuvwxyz1234567890ABCDEFGHIJKLMNOPQRSTUVWXYZ!@#$%^&*()?_"
        const myapp_class_obj = Java.use("com.gdufs.xman.MyApp");
        const three_character_array_len = three_character_array.length;
        const my_dict_len = my_dict.length
        const myapp = Java.use("com.gdufs.xman.MyApp").$new();
        var flag = "";
        for (var i = 0; i < three_character_array_len; i++) {
            var found = false;
            for (var j = 0; j < my_dict_len; j++) {
                if (found == true) {
                    break;
                }
                for (var k = 0; k < my_dict_len; k++) {
                    if (found == true) {
                        break;
                    }
                    for (var m = 0; m < my_dict_len; m++) {
                        const try_str = my_dict[j] + my_dict[k] + my_dict[m];
                        console.log(`try_str: ${try_str}`);
                        myapp.saveSN(try_str);
                        if (three_character_array[i] == fputs_str) {
                            flag += try_str;
                            console.log(`found: ${try_str}`);
                            found = true;
                            break;
                        }
                    }
                }
            }
        }

        for (var i = 0; i < my_dict_len; i++) {
            const try_str = my_dict[i];
            console.log(`try_str: ${try_str}`);
            Invoke(try_str);
            if (last_character == fputs_str) {
                flag += try_str;
                console.log(`found: ${try_str}`);
                break;
            }
        }
        console.log(`flag: xman{${flag}}`);
    })
}

//不能用setImmediate(Main)，会出现Process crashed: SIGSYS SYS_SECCOMP
//需要延迟一下再爆破
setTimeout(Main, 500);


```

![](https://genliesephotos.genliese.cn/Markdown/xman_frida_rpc/JS_result.png)

花了大概两分钟爆破出了 flag

### 3.2 RPC 爆破

**JavaScript 代码**

```
var myapp = null;

function Hook() {
    Java.perform(function () {
        myapp = Java.use("com.gdufs.xman.MyApp").$new();
        const imports = Module.enumerateImportsSync("libmyjni.so");
        const imports_len = imports.length;
        var fputs_addr = null;
        for (var i = 0; i < imports_len; i++) {
            if (imports[i].name == "fputs") {
                fputs_addr = imports[i].address;
                break;
            }
        }
        if (fputs_addr != null) {
            Interceptor.attach(fputs_addr, {
                onEnter: function (args) {
                    send(args[0].readCString());
                },
                onLeave: function (retval) {
                }
            })
        }
    })
}

function Invoke(try_str) {
    Java.perform(function () {
        myapp.saveSN(try_str);
    })
}

rpc.exports = {
    hook: Hook,
    invoke: Invoke
}


```

**Python 代码**

```
 Flag: xman{{{}}} ".format("".join(flag)))    sys.stdout.flush()    if first_part_times == 4:        break# 清屏print("\033c")try_count = 0for try_str in my_dict:    sys.stdout.write("[{}]Try_str: {}\r".format(try_count, "".join(try_str)))    script.exports.invoke(try_str)    if result == goal_last_character:        flag[12] = try_str        break    sys.stdout.write("
 Flag: xman{{{}}} ".format("".join(flag)))    sys.stdout.flush()# 清屏print("\033c")print("
 Flag: xman{{{}}}".format("".join(flag)))time_end = time.time()cost_time = time_end - time_beginprint(" cost time: " + str(cost_time // 60) + "min")

```

![](https://genliesephotos.genliese.cn/Markdown/xman_frida_rpc/RPC_result.png)

采用 RPC 爆破的方式花了 429 分钟才爆破出了 flag，花的时间是直接使用 JS 代码进行爆破的 214 倍

4. 问题答疑
-------

### 4.1 设备问题

如果出现各种无法解决的问题，尝试换成跟我一样的设备和系统——pixel 3、android 9.0

### 4.2 出现`global reference table overflow`

错误是全局引用表溢出了，具体意思是 Java 对象的全局引用表溢出了，你在循环里 new Java 对象就可能会溢出，因为 frida 对 Java 对象的引用就是用的全局引用，不知道什么时候才释放，反正循环的时候没有释放

### 4.3 爆破卡住了

多次调用`Java.choose`程序就会卡住，好像跟 frida 版本无关，不知道为什么，所以我直接`new`一个对象进行主动调用

### 4.4 为什么 233 字符没有被爆破出来？

排列使用的字典是没有重复字符的，所以 3 个字符中不可能同时出现两个相同的字符，下面的代码就会出现这种问题

```
my_dict = "abcdefghijklmnopqrstuvwxyz1234567890ABCDEFGHIJKLMNOPQRSTUVWXYZ!@#$%^&*()?_"
permutations(dict, 3)

```

解决办法是把字典自加多次，代码如下

```
def GeneratePossibilities(dict, count, repetitive=False):
    """

    :param dict: Dictionary
    :param count: Number of characters per group
    :param repetitive: Whether there are duplicate characters in each group
    :return: return permutations(dict, count)
    """
    if repetitive and (count > 1):
        src_dict = dict
        for i in range(count - 1):
            dict += src_dict
    return permutations(dict, count)

my_dict = "abcdefghijklmnopqrstuvwxyz1234567890ABCDEFGHIJKLMNOPQRSTUVWXYZ!@#$%^&*()?_"
possibilities = GeneratePossibilities(my_dict, 3, True)

```

5. 附件
-----

链接：[https://pan.baidu.com/s/1NRPGU_j6CK7bgB363FN33w](https://pan.baidu.com/s/1NRPGU_j6CK7bgB363FN33w)  
提取码：09wb

![](https://avatar.52pojie.cn/data/avatar/001/68/34/64_avatar_middle.jpg)ghostdough 大佬今天早上手机接到一条短信，说营业执照要过期要审核资料，我妈点进去按照步骤填写被骗了 25 万，现在这个网站还在运行，大佬能不能帮帮忙看一下能搞到后台数据，给警方提供线索早点破案吗，账户查到是黑龙江的但是具体账户里有没有钱也不知道，网站地址是 http://amssuw.bar/index2.asp  
，ip 是 185.251.249.137:80 谢谢大佬，现在这个网站还能访问 ![](https://avatar.52pojie.cn/data/avatar/001/00/27/01_avatar_middle.jpg) genliese

> [tlhelen 发表于 2021-4-19 22:17](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=38129211&ptid=1414060)  
> 麻烦看下图，还是这个问题，怎么回事？

我测试了网站上的 JS 爆破代码，是没有问题的。如果是你自己写的代码的话，你的问题可能是没有把 "EoPAoY62@ElRD" 进行分组，需要分成三个字符一组才能爆破出来，如下[JavaScript] _纯文本查看_ _复制代码_

```
const three_character_array = new Array("EoP", "AoY", "62@", "ElR");
const last_character = "D";

```

你不能一个字符一个字符的进行爆破，原因在于 `saveSN` 函数中的如下部分，字符的位置会决定执行哪个 if-else 分支，如果你单字符进行爆破，那么永远只会走 if 分支  
![](https://genliesephotos.genliese.cn/Markdown/xman_frida_rpc/grouping.png)![](https://avatar.52pojie.cn/data/avatar/000/31/06/44_avatar_middle.jpg)fanvalen 这是不是说明 js 比 py 算的快 ![](https://avatar.52pojie.cn/data/avatar/000/14/50/86_avatar_middle.jpg) tan567421 不错 uo 不错。。学习了![](https://avatar.52pojie.cn/data/avatar/000/53/86/12_avatar_middle.jpg) 马克, 顺便 lznb![](https://avatar.52pojie.cn/data/avatar/001/54/75/64_avatar_middle.jpg)ai474427793 谢谢大佬 学习了！![](https://avatar.52pojie.cn/data/avatar/001/65/72/65_avatar_middle.jpg)PrincessSnow 学习了，学习了 ![](https://avatar.52pojie.cn/data/avatar/001/63/84/75_avatar_middle.jpg) whats_OvO 学习学习![](https://avatar.52pojie.cn/data/avatar/000/48/16/21_avatar_middle.jpg)尘封_ 学习了，谢谢分享 ![](https://avatar.52pojie.cn/data/avatar/001/67/05/16_avatar_middle.jpg) anlovedong 不错，学习了，学习了 ![](https://avatar.52pojie.cn/data/avatar/001/63/91/62_avatar_middle.jpg) helanzhu1 太强了，学习学习！
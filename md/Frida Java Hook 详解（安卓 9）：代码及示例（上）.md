> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/2BdX-rtAu8WZuzY3pK94NQ)

**Frida Java Hook 详解（安卓 9）：代码及示例（上）**

**前言**

1.1 FRIDA SCRIPT 的 "hello world"

1.1.1 "hello world" 脚本代码示例

1.1.2 "hello world" 脚本代码示例详解

1.2 Java 层拦截普通方法

1.2.1 拦截普通方法脚本示例

1.2.2 执行拦截普通方法脚本示例

1.3 Java 层拦截构造函数

1.3.1 拦截构造函数脚本代码示例

1.3.2 拦截构造函数脚本代码示例解详解

1.4 Java 层拦截方法重载

1.4.1 拦截方法重载脚本代码示例

1.5 Java 层拦截构造对象参数

1.5.1 拦截构造对象参数脚本示例

1.6 Java 层修改成员变量的值以及函数的返回值

1.6.1 修改成员变量的值以及函数的返回值脚本代码示例

1.6.2 修改成员变量的值以及函数的返回值之小实战

 结语

### Frida Java Hook 详解（安卓 9）：代码及示例（上）

### 前言

在前面几篇中，我们学会了如何使用`FRIDA官方API`，对了`FRIDA`的`API`有了基本的认识 (详见 https://github.com/r0ysue/AndroidSecurityStudy）。咱们在这篇来深入学习如何`HOOK Java`层函数，应用于与各种不同的`Java`层函数，结合实际`APK`案例使用`FRIDA`框架对其`APP`进行附加、`hook`、以及`FRIDA`脚本的详细编写。

### 1.1 FRIDA SCRIPT 的 "hello world"

在本章节中，依然会大量使用注入模式附加到一个正在运行进程程序，亦或是在`APP`程序启动的时候对其`APP`进程进行劫持，再在目标进程中执行我们的`js`文件代码逻辑。

`FRIDA`脚本就是利用`FRIDA`动态插桩框架，使用`FRIDA`导出的`API`和方法，对内存空间里的对象方法进行监视、修改或者替换的一段代码。`FRIDA`的`API`是使用`JavaScript`实现的，所以我们可以充分利用`JS`的匿名函数的优势、以及大量的`hook`和回调函数的`API`。

那么大家跟我一起来操作吧，先打开在`vscode`中创建一个 js 文件：`helloworld.js`：

### 1.1.1 "hello world" 脚本代码示例

```
setTimeout(function(){
  Java.perform(function(){
      console.log("hello world!");
    });
});

```

### 1.1.2 "hello world" 脚本代码示例详解

这基本上就是一个`FRIDA`版本的`Hello World!`，我们把一个匿名函数作为参数传给了`setTimeout()`函数，然而函数体中的`Java.perform()`这个函数本身又接受了一个匿名函数作为参数，该匿名函数中最终会调用`console.log()`函数来打印一个`Hello world！`字符串。我们需要调用`setTimeout()`方法因为该方法将我们的函数注册到`JavaScript`运行时中去，然后需要调用`Java.perform()`方法将函数注册到`Frida`的`Java`运行时中去，用来执行函数中的操作，当然这里只是打了一条`log`。

然后我们在手机上将`frida-server`运行起来，在电脑上进行操作：

```
roysue@ubuntu:~$ adb shell
sailfish:/ $ su
sailfish:/ $ ./data/local/tmp/frida-server

```

这个时候，我们需要再开启一个终端运行另外一条命令：

`frida -U com.roysue.roysueapplication -l helloworld.js`

这句代码是指通过`USB`连接对`Android`设备中的`com.roysue.roysueapplication`进程对其附加并且注入一个`helloworld.js`脚本。注入完成之后会立刻执行`helloworld.js`脚本所写的代码逻辑！

我们可以看到成功注入了脚本以及附加到自己所编写包名为：`com.roysue.roysueapplication`的`apk`应用程序中，并且打印了一条 `hell world!`。

```
roysue@ubuntu:~$ frida -U -l helloworld.js com.roysue.roysueapplication
     ____
    / _  |   Frida 12.7.24 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://www.frida.re/docs/home/
Attaching...
hello world!

```

执行了该命令之后，`FRIDA`返回一个了`CLI`终端工具与我们交互，在上面可见打印出来了一些信息，显示了我们的 FRIDA 版本信息还有一个反向 R 的图形，往下看，需要退出的时候我们只需要在终端输入`exit`即可完成退出对`APP`的附加，其后我们看到的是`Attaching...`正在对目标进程附加，当附加成功了打印了一句`hello world!`，至此，我们的`helloworld.js`通过`FRIDA`的`-l`命令成功的注入到目标进程并且执行完毕。学会了注入可不要高兴的太早哟~~ 咱们继续深入学习`HOOK Java`代码中的普通函数。

### 1.2 Java 层拦截普通方法

`Java`层的普通方法相当常见，也是我们要学习的基础中的基础，我们先来看几个比较常见的普通的方法，见下图 1-1。

![](https://mmbiz.qpic.cn/mmbiz_png/1qk1GGFlliagVlCYFEoC8NYicn6y7dmozGpFugAhiaI0CYuajz2aErgW3S6nv3KLhyiaQp2S0VXaiaUDZcabkGBUUJw/640?wx_fmt=png)

图 1-1 JADX-GUI 软件打开的反编译代码

通过图 1-1 我们能看三个函数分别是构造函数`a()`、普通函数`a()`和`b()`。诸如这种普通函数会特别多，那我们在本小章节中尝试`hook`普通函数、查看函数中的参数的具体值。

在尝试写`FRIDA HOOK`脚本之前咱们先来看看需要`hook`的代码吧~，`Ordinary_Class`类中有四个函数，都是很普通的函数，`add`函数的功能也很简单，参数`a`+`b`；sub 函数功能是参数`a`-`b`；而`getNumber`只返回`100`；`getString`方法返回了 `getString()`+ 参数的`str`。见下图 1-2。

![](https://mmbiz.qpic.cn/mmbiz_png/1qk1GGFlliagVlCYFEoC8NYicn6y7dmozGXicMLahL5VoFDcX9o10sTTMDfvwbjtMHIdQmpsttjZkK5pVibtaXjKXw/640?wx_fmt=png)

图 1-2 反编译的 Ordinary_Class 类的代码

然后咱们再看`MainActivity`中的编写的代码，通过反编译出来的代码一共有四个按钮（`Button`），当`btn_add`点击时会运行`Ordinary_Class`类中`add`方法，计算`100+200`的结果，通过`String.valueOf`函数把计算结构转字符串然后通过`Toast`弹出信息；点击`btn_sub`按钮的时候触发点击事件会运行`Ordinary_Class`类中`sub`方法，计算`100-100`的结果，通过`String.valueOf`函数把计算结构转字符串然后通过`Toast`弹出信息。在`MainActivity`类中的`onCreate`方法中的四个按钮分别对应情况是`ADD`按钮对应`btn_add`点击事件，`SUB`对应`btn_sub`的点击事件。见下图 1-3。

![](https://mmbiz.qpic.cn/mmbiz_png/1qk1GGFlliagVlCYFEoC8NYicn6y7dmozGPVd6ux0ibtDEb0y2vPx2V6eLKfNJ5FR6ZdZgEeuszicOFueEj6be8JJg/640?wx_fmt=png)

图 1-3 MainActivity 中的编写的代码

按照正常流程当我们点击`ADD`的按钮界面会弹出一条信息显示，其中的值是`300`，因为我们在`ADD`的点击事件中添加了`Toast`，将`ADD`方法运行的结果放在`Toast`参数中，通过它显示了我们的计算结果；而`SUB`函数会显示`0`，见下图 1-4，图 1-5。

![](https://mmbiz.qpic.cn/mmbiz_png/1qk1GGFlliagVlCYFEoC8NYicn6y7dmozGaphGCYjiawcRvqBmjy6Fp9eXbQRNp0pCEnWxnJCdncmydF88sxJ03QA/640?wx_fmt=png)

图 1-4 点击 ADD 按钮时显示的结果

![](https://mmbiz.qpic.cn/mmbiz_png/1qk1GGFlliagVlCYFEoC8NYicn6y7dmozG3p2oEHyUiarengbxO5hkFjPOSBBKomsqB3iaBFDRxMve5PNoLvia1AicEA/640?wx_fmt=png)

图 1-5 点击 SUB 按钮时显示的结果

我们现在知道已经知道它的运行流程以及函数的执行结果和所填写的参数，我们现在来正式编写一个基本的使用`Frida`钩子来拦截图 1-2 中`add`和`sub`函数的调用并且在终端显示每个函数所传入的参数、返回的值，开始写`roysue_0.js`：

### 1.2.1 拦截普通方法脚本示例

```
setTimeout(function(){
    //判断是否加载了Java VM，如果没有加载则不运行下面的代码
    if(Java.available) {
        Java.perform(function(){
            //先打印一句提示开始hook
            console.log("start hook");
            //先通过Java.use函数得到Ordinary_Class类
            var Ordinary_Class = Java.use("com.roysue.roysueapplication.Ordinary_Class");
            //这里我们需要进行一个NULL的判断，通常这样做会排除一些不必要的BUG
            if(Ordinary_Class != undefined) {
                //格式是：类名.函数名.implementation = function (a,b){
                //在这里使用钩子拦截add方法，注意方法名称和参数个数要一致，这里的a和b可以自己任意填写，
                Ordinary_Class.add.implementation = function (a,b){
                    //在这里先得到运行的结果，因为我们要输出这个函数的结果
                    var res = this.add(a,b);
                    //把计算的结果和参数一起输出
                    console.log("执行了add方法 计算result:"+res);
                    console.log("执行了add方法 计算参数a:"+a);
                    console.log("执行了add方法 计算参数b:"+b);
                    //返回结果，因为add函数本身是有返回值的，否则APP运行的时候会报错
                    return res;
                }
                Ordinary_Class.sub.implementation = function (a,b){
                    var res = this.sub(a,b);
                    console.log("执行了sub方法 计算result:"+res);
                    console.log("执行了sub方法 计算参数a:"+a);
                    console.log("执行了sub方法 计算参数b:"+b);
                    return res;
                }
                Ordinary_Class.getString.implementation = function (str){
                    var res = this.getString(str);
                    console.log("result:"+res);
                    return res;
                }
            } else {
                console.log("Ordinary_Class: undefined");
            }
            console.log("hook end");
          });
    }
});

```

### 1.2.2 执行拦截普通方法脚本示例

写完脚本后我们执行：`frida -U com.roysue.roysueapplication -l roysue_0.js`，当我们执行了脚本后会进入`cli`控制台与`frida`交互，可以看到已经对该`app`附加并且成功注入脚本。立刻打印出了`start hook`和`hook end`，正是我们刚刚所写的。再继续点击`app`应用中的`ADD`按钮和`SUB`按钮会在终端立刻输出计算的结果和参数，在这里甚至可以看到清晰，参数、返回值一览无余。

```
roysue@ubuntu:~$ frida -U com.roysue.roysueapplication -l roysue_0.js
     ____
    / _  |   Frida 12.7.24 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://www.frida.re/docs/home/
Attaching...
start hook
hook end
[Google Pixel::com.roysue.roysueapplication]-> 执行了add方法 计算result:300
执行了add方法 计算参数a:100
执行了add方法 计算参数b:200
执行了sub方法 计算result:0
执行了sub方法 计算参数a:100
执行了sub方法 计算参数b:100

```

这样我们就已经成功的打印了来了我们想要知道的值，每个参数的值和返回值的结构。我们就对普通函数的钩子完成了一个基本操作，大家可以自己多多尝试对其他的普通的函数进行`hook`，多多练习，那咱们这一章节就愉快的完成了~~ 继续深入吧。

### 1.3 Java 层拦截构造函数

那咱们这章来玩如何`HOOK`类的构造函数，很多时候在实例化类的瞬间就会把参数传递到内部为成员变量赋值，这样一来就省的类中的成员变量一个个去赋值，在`Android`逆向中，也有很多的类似的场景，利用有参构造函数实例化类并且赋值。我建立了一个`class`类，类名是`User`，其中代码这样写：

```
public class User {
    public int age;
    public String name;
    public String toString() {
        StringBuilder sb = new StringBuilder();
        sb.append("User{);
        sb.append(this.name);
        sb.append('\'');
        sb.append(", age=");
        sb.append(this.age);
        sb.append('}');
        return sb.toString();
    }
    public User(String name2, int age2) {
        this.name = name2;
        this.age = age2;
    }
    public User() {
    }
}

```

我们可以看到`User`类中有 2 个成员变量分别是`age`和`name`，还有`2`个构造方法，分别是无参构造有有参构造。我们现在要做的是在`User`类进行有参实例化时查看所填入的参数分别是什么值。在图 1-3 中，可以看到`btn_init`的点击事件时会对`User`类进行实例化参数分别填写了`roysue`和`30`，然后再继续调用了`toString`方法把它们组合到一起并且通过`Toast`弹出信息显示值。`TEST_INTI对应btn_init`的点击事件，点击时效果见下图 1-6。

![](https://mmbiz.qpic.cn/mmbiz_png/1qk1GGFlliagVlCYFEoC8NYicn6y7dmozGgbbBOb2cJLL69UQIQiaibtoAuoRCogzYymwjaKSqibqkRMnL5ibG1kHEwA/640?wx_fmt=png)

图 1-6 点击 TEST_INIT 时显示的值

### 1.3.1 拦截构造函数脚本代码示例

现在开始编写`frida`钩子来拦截`User`类的构造函数的脚本, 来打印出构造函数的参数，编写`roysue_1.js`。

```
setTimeout(function(){
    if(Java.available) {
        Java.perform(function(){
            console.log("start hook");
            //同样的在这里先获取User类对象
            var User = Java.use("com.roysue.roysueapplication.User");
            if(User != undefined) {
                //注意在使用钩子拦截构造函数时需要使用到 $init 也要注意参数的个数，因为该构造函数是2个所以此处填2个参数
                User.$init.implementation = function (name,age){
                    //这里打印成员变量name和age的在运行中被调用填写的值
                    console.log("name:"+name);
                    console.log("age:"+age);
                    //最终要执行原本的init方法否则运行时会报异常导致原程序无法正常运行。
                    this.$init(name, age);
                }
            } else {
                console.log("User: undefined");
            }
            console.log("hook end");
          });
    }
});

```

### 1.3.2 拦截构造函数脚本代码示例解详解

脚本写好之后打开终端执行`frida -U com.roysue.roysueapplication -l roysue_1.js`，把刚刚写的脚本注入的目标进程，然后我们在`APP`应用中按下`TEST_INIT`按钮，注入的脚本会立即拦截构造函数并且打印 2 个参数的具体的值，见下图 1-7。

![](https://mmbiz.qpic.cn/mmbiz_png/1qk1GGFlliagVlCYFEoC8NYicn6y7dmozG6M6YgFw3wyicId3vxLoJdvawGvtFd2J3xnbXnzPZX7A0eaiaV8IoGbibA/640?wx_fmt=png)

图 1-7 终端显示

打印的值就是图 1-3 中所填的`roysue`和`30`，这就说明我们使用`FRIDA`钩子拦截到了`User`类的有参构造函数并且有效的打印了参数的值。需要注意的是在输入打印参数的值之后一定要记得执行原本的有参构造函数，这样程序才可以正常执行。

### 1.4 Java 层拦截方法重载

在学习`HOOK`之前，咱们先了解一下什么是方法重载，方法重载是指在同一个类内定义了多个相同的方法名称，但是每个方法的参数类型和参数的个数都不同。在调用方法重载的函数编译器会根据所填入的参数的个数以及类型来匹配具体调用对应的函数。总结起来就是方法名称一样但是参数不一样。在逆向`JAVA`代码的时候时常会遇到方法重载函数，见下图 1-8。

![](https://mmbiz.qpic.cn/mmbiz_png/1qk1GGFlliagVlCYFEoC8NYicn6y7dmozG9nExTcnxLQFG3F2s8kCLw9wHialptiaY9ibPZKSZtFAOCbKZiayxrmZcxQ/640?wx_fmt=png)

图 1-8 反编译后重载函数样本代码

在图 1-8 中，我们能看到一共有三个方法重载的函数，有时候实际情况甚至更多。咱们也不要怕，撸起袖子加油干。对于这种重载函数在`FRIDA`中，`js`会写`.overload`来拦截方法重载函数，当我们使用`overload`关键字的时候`FRIDA`会非常智能把当前所有的重载函数的所需要的类型打印出来。在了解了这个之后我们来开始实战使用`FRIDA`钩子拦截我们想拦截的重载函数的脚本吧！还是之前的那个`app`，还是之前的那个类，原汁原味~~，我新增了一些`add`的方法，使`add`方法重载，见下图 1-9。

![](https://mmbiz.qpic.cn/mmbiz_png/1qk1GGFlliagVlCYFEoC8NYicn6y7dmozGaA54GO5sibzu2ib9ReefEPEwk9ds96LjQ5zSgRrIUFAib3BrDf2UEFzpA/640?wx_fmt=png)

图 1-9 反编译后的`Ordinary_Class`中的重载函数样本代码

在图 1-9 中`add`有三个重名的方法名称，但是参数的个数不同，在图 1-3 中的`btn_add`点击事件会执行拥有 2 个参数的方法重载的`add`函数，当我在脚本中写`Ordinary_Class.add..implementation = function (a,b)`，然后继续注入所写好的脚本，`FRIDA`在终端提示了红色的字体，一看吓一跳！但咱们仔细看，它说`add`是一个方法重载的函数，有三个参数不同的`add`函数，让我们写`.overload(xxx)`，以识别 hook 的到底是哪个`add`的方法重载函数。

当我们写了这样的`js`脚本去运行的时候，`frida`提示报错了，因为有三个重载函数，我用红色的框圈出了，可以看到`frida`十分的智能，三个重载的参数类型完全一致的打印出来了，当它打印出来之后我们就可以复制它的这个智能提示的`overloadxxx`重载来修改我们自己的脚本了，进一步完善我们的脚本代码如下。

### 1.4.1 拦截方法重载脚本代码示例

```
function hook_overload() {
    if(Java.available) {
        Java.perform(function () {
            console.log("start hook");
            var Ordinary_Class = Java.use("com.roysue.roysueapplication.Ordinary_Class");
            if(Ordinary_Class != undefined) {
                //要做的仅仅是将frida提示出来的overload复制在此处
                Ordinary_Class.add.overload('int', 'int').implementation = function (a,b){
                    var res = this.add(a,b);
                    console.log("result:"+res);
                    return res;
                }
                Ordinary_Class.add.overload('int', 'int', 'int').implementation = function (a,b,d){
                    var res = this.add(a,b,d);
                    console.log("result:"+res);
                    return res;
                }
                Ordinary_Class.add.overload('int', 'int', 'int', 'int').implementation = function (a,b,d,c){
                    var res = this.add(a,b,d,c);
                    console.log("result:"+res);
                    return res;
                }
            } else {
                console.log("Ordinary_Class: undefined");
            }
            console.log("start end");
        });
    }
}
setImmediate(hook_overload);

```

修改完相应的地方之后我们保存代码时终端会自动再次运行 js 中的代码，不得不说`frida`太强大了~，当`js`再次运行的时候我们在`app`应用中点击图 1-4 中`ADD`按钮时会立刻打印出结果，因为`FRIDA`钩子已经对该类中的所有的`add`函数进行了拦截，执行了自己所写的代码逻辑。点击效果如下图 1-11。

![](https://mmbiz.qpic.cn/mmbiz_png/1qk1GGFlliagVlCYFEoC8NYicn6y7dmozGLGfsQoMkVDhdia7ic6jaUvUjSAe6ppOtic8zxzZBJXMJgiakkWAZqniaWFQ/640?wx_fmt=png)

图 1-11 终端显示效果

在这一章节中我们学会了处理方法重载的函数，我们只要依据 FRIDA 的终端提示，将智能提示出来的代码衔接到自己的代码就能够对方法重载函数进行拦截，执行我们自己想要执行的代码。

### 1.5 Java 层拦截构造对象参数

很多时候，我们不但要`HOOK`使用钩子拦截函数对函数的参数和返回值进行记录，而且还要自己主动调用类中的函数使用。`FRIDA`中有一个`new()`关键字，而这个关键字就是实例化类的重要方法。

在官方`API`中有这样写道：“`Java.use(ClassName)：`动态获取`className`的`JavaScript`包装器，通过对其调用`new()`来调用构造函数，可以从中实例化对象。对实例调用`Dispose()`以显式清理它 (或等待`JavaScript`对象被垃圾收集，或脚本被卸载)。静态和非静态方法都是可用的。”，那我们就知道通过`Java.use`获取的`class`类可以调用`$new()`来调用构造函数，可以从实例化对象。在图 1-9 中有`6`个函数，了解了`API`的调用之后我们来开始入手编写我们的 js 文件。（在这里我觉得大家一定要动手做测试，动手尝试，你会发现其中的妙趣无穷！）。

### 1.5.1 拦截构造对象参数脚本示例

```
function hook_overload_1() {
    if(Java.available) {
        Java.perform(function () {
            console.log("start hook");
            //还是先获取类
            var Ordinary_Class = Java.use("com.roysue.roysueapplication.Ordinary_Class");
            if(Ordinary_Class != undefined) {
                //这里因为add是一个静态方法能够直接调用方法
                var result = Ordinary_Class.add(100,200);
                console.log("result : " +result);
                //调用方法重载无压力
                result = Ordinary_Class.add(100,200,300);
                console.log("result : " +result);
                //调用方法重载
                result = Ordinary_Class.add(100,200,300,400);
                console.log("result : " +result);         
                //调用方法重载       
                result = Ordinary_Class.getNumber();
                console.log("result : " +result);   
                //调用方法重载             
                result = Ordinary_Class.getString("  HOOK");
                console.log("result : " +result);
                //在这里，使用了官方API的$new()方法来实例化类，实例化之后返回一个实例对象，通过这个实例对象来调用类中方法。
                var Ordinary_Class_instance = Ordinary_Class.$new();
                result = Ordinary_Class_instance.getString("Test");
                console.log("instance --> result : " +result);
                result = Ordinary_Class_instance.add(1,2,3,4);
                console.log("instance --> result : " +result);     
            } else {
                console.log("Ordinary_Class: undefined");
            }
            console.log("start end");
        });
    }
}
setImmediate(hook_overload_1);

```

当我们执行了上面写的脚本之后终端会打印调用方法之后的结果，见下图 1-12。

![](https://mmbiz.qpic.cn/mmbiz_png/1qk1GGFlliagVlCYFEoC8NYicn6y7dmozGRU6trlJcJcias7xfWXZ5dTAhVWlcmfaBict9IKQ6j11icE7aPYLRVQ9Gw/640?wx_fmt=png)

图 1-12 终端显示调用函数的结果

因为很多时候类中的方法并不一定是静态的，所以这里提供了`2`种调用方法，第一种调用方式十分的方便，不需要实例化一个对象，再通过对象调本身的方法。但是遇到了没有`static`关键字的函数时只能使用第二种方式来实现方法调用，在这一章节中我们学会了如何自己主动去调用类中的函数了~~ 大家也可以尝试主动调用有参的构造函数玩玩。

### 1.6 Java 层修改成员变量的值以及函数的返回值

我们上章学完了如何自己主动调用`JAVA`层的函数了，经过上章的学习我们的功夫又精进了一些~~，现在我们来深入内部修改类的对象的成员变量和返回值，打入敌人内部，提高自己的内功。现在我们来看下图 1-13。

![](https://mmbiz.qpic.cn/mmbiz_png/1qk1GGFlliagVlCYFEoC8NYicn6y7dmozGxrDSb6ZbAIqM9VF7ABg14syfa4AxkU492FY9ic34lUVqjJZ9PfL0Gicg/640?wx_fmt=png)

图 1-13 User 类

上图中的`User`类是我之前建的一个类，类中写了 2 个公开成员变量分别是`age`是`name`；还有`2`个方法分别是`User`的有参构造函数和一个`toString`函数打印成员变量的函数。我们要做就是在`User`类实例化的时候拦截程序并且修改掉`age`和`name`的值，从而改写成我们自己需要的值再运行程序，那我们接下开始编写`JS`脚本来修改成员变量的值。

这段代码主要有的功能是：通过`User.$new("roysue",29)`拿到`User`类的有参数构造的实例化对象，这个恰好也是使用了上章节中学到的知识自己构建对象，这里我们也学习了如何使用 FRIDA 框架通过有参构造函数实例化对象，实例化之后先是调用了类本身的`toString`方法打印出未修改前的成员变量的值，打印了之后再通过`User_instance.age.value = 0;`来修改对象当前的成员变量的值，可以看到修改`age`修改为`0`，`name`修改为`roysue_123`，然后再次调用`toString`方法查看其成员变量的最新值是否已经被更改。

### 1.6.1 修改成员变量的值以及函数的返回值脚本代码示例

```
function hook_overload_2() {
    if(Java.available) {
        Java.perform(function () {
            console.log("start hook");
            //拿到User类
            var User = Java.use("com.roysue.roysueapplication.User");
            if(User != undefined) {
                //这里利用上章学到知识来自己构建一个User的有参构造的实例化对象
                var User_instance  = User.$new("roysue",29);
                //并且调用了类中的toString()方法
                var str = User_instance.toString();
                //打印成员变量的值
                console.log("str:"+str);
                //这里获取了属性的值以及打印
                console.log("User_instance.name:"+User_instance.name);
                console.log("User_instance.age:"+User_instance.age);
                //这里重新设置了age和name的值
                User_instance.age.value = 0;
                User_instance.name.value = "roysue_123";
                str = User_instance.toString();
                //再次打印成员变量的值
                console.log("str:"+str);
            } else {
                console.log("User: undefined");
            }
            console.log("start end");
        });
    }
}

```

可以看到终端显示了原本有参构造函数的值 roysue 和 30 修改为 roysue_123 和 0 已经成功了，效果见下图 1-14。

![](https://mmbiz.qpic.cn/mmbiz_png/1qk1GGFlliagVlCYFEoC8NYicn6y7dmozGIXepiajG3icMDqpr477AvSriaehic1DpVDKUC9OLHichPuiaG14uCMlWpC6A/640?wx_fmt=png)

图 1-14 终端显示修改效果

通过上面的学习，我们学会了如何修改类的成员变量，上个例子中是使用的有参构造函数给与成员变量赋值，通常在写代码类似这种实体类会定义相关的`get set`方法以及修饰符为私有权限，外部不可调用，这个时候他们可能会通过 set 方法来设置其值和`get`方法获取成员的变量的值，这个时候我们可以通过钩子拦截`set`和`get`方法自己定义值也是可以达到修改和获取的效果。现在学完了如何修改成员变量了，那我们接下来要学习如何修改函数的返回值，假设在逆向的过程中已知检测函数`A`的结果为`B`，正确结果为`C`，那我们可以强行修改函数`A`的返回值，不论在函数中执行了什么与返回结果无关，我们只要修改结果即可。

### 1.6.2 修改成员变量的值以及函数的返回值之小实战

我在`rdinary_Class`类建立了`2`个函数分别是`isCheck`和`isCheckResult`，假设`isCheck`是一个检测方法，经过`add`运行后必然结果`2`，代表被检测到了，在`isCheckResult`方法进行了判断调用`isCheck`函数结果为`2`就是错误的，那这个时候要把`isCheck`函数或者`add`函数的结果强行改成不是`2`之后`isCheckResult`即可打印`Successful`，见下图 1-15。

![](https://mmbiz.qpic.cn/mmbiz_png/1qk1GGFlliagVlCYFEoC8NYicn6y7dmozGXJhDqpSO53StmUTlqE5kccM48TdUG1SMhWQ6kzgUKwBnq5iaUsyPD5Q/640?wx_fmt=png)

图 1-15 isCheck 函数与 isCheckResult 函数

我们现在要做的是使`sCheckResult`函数成功打印出`"Successful"`，而不是`errer`，那我们现在开始来写`js`脚本吧~~

```
function hook_overload_7() {
    if(Java.available) {
        Java.perform(function () {
            console.log("start hook");
            //先获取类
            var Ordinary_Class = Java.use('com.roysue.roysueapplication.Ordinary_Class');
            if(Ordinary_Class != undefined) {
                //先调用一次肯定会输出error
                Ordinary_Class.isCheckResult();
                //在这里重写isCheck方法将返回值强行改为123并且输出了一句Ordinary_Class: isCheck
                Ordinary_Class.isCheck.implementation = function (){
                    console.log("Ordinary_Class: isCheck");
                    return 123;
                }
                //再调用一次isCheckResult()方法
                Ordinary_Class.isCheckResult();
            } else {
                console.log("Ordinary_Class: undefined");
            }
            console.log("hook end");
        });
    }
}

```

上面这段代码的主要功能是：首先通过`Java.use`获取`Ordinary_Class`, 因为`isCheckResult()`是静态方法，可以直接调用，在这里先调用一次，因为这样比较好看效果，第一次调用会在`Android LOG`中打印`errer`，之后紧接着利用`FRIDA`钩子对`isCheck`函数进行拦截，改掉其返回值为`123`，这样每次调用`isCheck`函数时返回值都必定会是`123`，再调用一次`isCheckResult()`方法，`isCheckResult()`方法中判断`isCheck`返回值是否等于`2`，因为我们已经重写了`isCheck`函数，所以不等于`2`，所以程序往下执行，会打印`Successful`字符串到`Android Log`中，实际运行效果见下图 1-16。

![](https://mmbiz.qpic.cn/mmbiz_png/1qk1GGFlliagVlCYFEoC8NYicn6y7dmozGLkWoNCXiaLqRJia380tnMZ5Zdgc7X6yu4rrlVG7ia7ia8rSKHAUUEaV8xg/640?wx_fmt=png)

图 1-16 终端显示以及 Android Log 信息

可以清晰的看到先是打印了`errer`后打印了`Successful`了，这说明我们已经成功过掉`isCheck`的判断了。这是一个小小的综合例子。建议大家多多动手尝试。

### 结语

在这章中我们学习了 HOOK Java 层的一些函数，如拦截普通函数、构造函数、以及修改成员变量以及函数返回值。下一篇中我们来枚举所有的类、所有的方法重载、所有的子类以及 RPC 远程调用 Java 层函数。
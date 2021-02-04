> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/heK_r0zXo_6_RoA37yPtGQ)

### 前言  

在（上）篇及之前的若干篇中，我们学会了如何使用 FRIDA 官方 API，对 FRIDA 的 API 有了基本的认识 (详见`https://github.com/r0ysue/AndroidSecurityStudy`）。学会了拦截构造函数、方法重载、成员变量等等，在这篇来深入学习如何拦截定位类、拦截内部类、类的所有方法等，结合实际 APK 案例使用 FRIDA 框架对其 APP 进行附加、hook、以及 FRIDA 脚本的详细编写。

### 1.1 Java 层拦截内部类函数

之前我们已经学习过了`HOOK`普通函数、方法重载、构造函数，现在来更深入的学习`HOOK`在`Android`逆向中，我们也会经常遇到在`Java`层的内部类。`Java`内部类函数，使得我们更难以分析代码。我们在这章节中对内部类进行一个基本了解和使用`FRIDA`对内部类进行钩子拦截处理。什么是内部类？所谓内部类就是在一个类内部进行其他类结构的嵌套操作，它的优点是内部类与外部类可以方便的访问彼此的私有域（包括私有方法、私有属性），所以`Android`中有很多的地方都会使用到内部类，我们来见一个例子也是最直观的，如下图 4-17。

![](https://mmbiz.qpic.cn/mmbiz_png/1qk1GGFlliagEVCmQQuLWJDTkpaH0atc7Y6hZ3JgeAeBW0fEUTPJcwyOhTqTJCiaI3UQ3ogwkl0mGqjicc4lXp6kQ/640?wx_fmt=png)

图 4-17 User 类中的 clz 类

在图 4-17 中看到`User`类中嵌套了一个`clz`，这样的操作也是屡见不鲜了。在`frida`中，我们可以使用`$`符号对起进行处理。首先打开`jadxgui`软件对代码进行反编译，反编译之后进入`User`类, 下方会有一个`smali`的按钮，点击`smali`则会进入`smali`代码，进入`smali`代码直接按`ctrl+f`局部搜索字符串`clz`，因为`clz`是内部类的名称，那么就会搜到`Lcom/roysue/roysueapplication/User\$clz;`，我们将翻译成`java`代码就是：`com.roysue.roysueapplication.User\$clz`，去掉第一个字符串的`L`和`/`以及`;`就构成了内部类的具体类名了，见下图 4-18。

![](https://mmbiz.qpic.cn/mmbiz_png/1qk1GGFlliagEVCmQQuLWJDTkpaH0atc7EQoQuTuTzgqkHlle6mSiafP4qaAicPuVqKEF5dUmheGoBeT0A1FrIWoQ/640?wx_fmt=png)

图 4-18 smali 代码  

经过上面的分析我们已经得知最重要的部分类的路径：`com.roysue.roysueapplication.User\$clz`，现在来对内部类进行`HOOK`，现在开始编写 js 脚本。

### 1.1.1 拦截内部类函数代码示例

```
function hook_overload_3() {
    if(Java.available) {
        Java.perform(function () {
            console.log("start hook");
            //注意此处类的路径填写更改所分析的路径
            var clz = Java.use('com.roysue.roysueapplication.User$clz');
            if(clz != undefined) {
                //这边也是像正常的函数来hook即可
                clz.toString.implementation = function (){
                    console.log("成功hook clz类");
                    return this.toString();
                }
            } else {
                console.log("clz: undefined");
            }
            console.log("start end");
        });
    }
}

```

执行脚本之后，我们可以看到控制也已经成功附加并且打印了成功`hook clz`类，这样我们也能够对`Java`层的内部类进行处理了。

```
[Google Pixel::com.roysue.roysueapplication]-> 成功hook clz类
成功hook clz类

```

### 1.2 Java 层枚举所有的类并定位类

在前面我们学会了如何在`java`层的各种函数的`HOOK`操作了，现在开始学习枚举所有的类并定位类的骚套路了~，学习之前我们要了解`API`中的`enumerateLoadedClasses`方法，它是属于`Java`对象中的一个方法。能够枚举现在加载的所有类，`enumerateLoadedClasses`存在`2`个回调函数，分别是`onMatch：function(ClassName)：`为每个加载的具有`className`的类调用，每个`ClassName`返回来的都是一个类名；和`onComplete：function()：`在枚举所有类枚举完之后回调一次。

### 1.2.1 枚举所有的类并定位类代码示例

```
setTimeout(function (){
  Java.perform(function (){
    console.log("n[*] enumerating classes...");
    //Java对象的API enumerateLoadedClasses
    Java.enumerateLoadedClasses({
      //该回调函数中的_className参数就是类的名称，每次回调时都会返回一个类的名称
      onMatch: function(_className){
        //在这里将其输出
        console.log("[*] found instance of '"+_className+"'");
        //如果只需要打印出com.roysue包下所有类把这段注释即可，想打印其他的替换掉indexOf中参数即可定位到~
        //if(_className.toString().indexOf("com.roysue")!=-1)
        //{
        //    console.log("[*] found instance of '"+_className+"'");
        //}
      },
      onComplete: function(){
        //会在枚举类结束之后回调一次此函数
        console.log("[*] class enuemration complete");
      }
    });
  });
});

```

当我们执行该脚本时，注入目标进程之后会开始调用`onMatch`函数，每次调用都会打印一次类的名称，当`onMatch`函数回调完成之后会调用一次`onComplete`函数，最后会打印出`class enuemration complete`，见下图。

![](https://mmbiz.qpic.cn/mmbiz_png/1qk1GGFlliagEVCmQQuLWJDTkpaH0atc7q676ZCU82kNFcUwZ7ZxgKCb0oHiaJFhZic71pqAfUjHPL58PUl7UJCicQ/640?wx_fmt=png)

图 4-19 枚举所有类  

### 1.3 Java 层枚举类的所有方法并定位方法

上文已经将类以及实例枚举出来，接下来我们来枚举所有方法，打印指定类或者所有的类的内部方法名称，主要核心功能是通过类的反射方法中的`getDeclaredMethods()`, 该`api`属于`JAVAJDK`中自带的`API`，属于`java.lang.Class`包中定义的函数。该方法获取到类或接口声明的所有方法，包括公共、保护、默认（包）访问和私有方法，但不包括继承的方法。当然也包括它所实现接口的方法。在`Java`中它是这样定义的：`public Method[] getDeclaredMethods()；`其返回值是一个`Method`数组，`Method`实际上就是一个方法名称字符串，当然也是一个对象数组，然后我们将它打印出来。

### 1.3.1 枚举类的所有方法并定位方法代码示例

```
function enumMethods(targetClass)
{
    var hook = Java.use(targetClass);
    var ownMethods = hook.class.getDeclaredMethods();
    hook.$dispose;
    return ownMethods;
}
function hook_overload_5() {
    if(Java.available) {
        Java.perform(function () {
           var a = enumMethods("com.roysue.roysueapplication.User$clz")
           a.forEach(function(s) {
                console.log(s);
           });
        });
    }
}

```

我们先定义了一个`enumMethods`方法，其参数`targetClass`是类的路径名称，用于`Java.use`获取类对象本身，获取类对象之后再通过其`.class.getDeclaredMethods()`方法获取目标类的所有方法名称数组，当调用完了`getDeclaredMethods()`方法之后再调用`$dispose`方法释放目标类对象，返回目标类所有的方法名称、返回类型以及函数的权限，这是实现获取方法名称的核心方法，下面一个方法主要用于注入到目标进程中去执行逻辑代码，在`hook_overload_5`方法中先是使用了`Java.perform`方法，再在内部调用`enumMethods`方法获取目标类的所有方法名称、返回类型以及函数的权限，返回的是一个`Method`数组，通过`forEach`迭代器循环输出数组中的每一个值，因为其本身实际就是一个字符串所以直接输出就可以得到方法名称，脚本执行效果如下图 4-20。

![](https://mmbiz.qpic.cn/mmbiz_png/1qk1GGFlliagEVCmQQuLWJDTkpaH0atc7Nic8xKyKKNaLYFx55ujjnhw6lpQPUa14jpskNRQ37zIYQGo6pKwRB0g/640?wx_fmt=png)

图 4-20 脚本执行后效果

在图 4-17 中`clz`只有一个`toString`方法，我们填入参数为`com.roysue.roysueapplication.User$clz`，就能够定位到该类中所有的方法。

### 1.4 Java 层拦截方法的所有方法重载

我们学会了枚举所有的类以及类的有方法之后，那我们还想知道如何获取所有的方法重载函数，毕竟在`Android`反编译的源码中方法重载不在少数，对此，一次性`hook`所有的方法重载是非常有必要的学习。我们已经知道在`hook`重载方法时需要写`overload('x')`，也就是说我们需要构造一个重载的数组，并把每一个重载都打印出来。

### 1.4.1 拦截方法的所有方法重载代码示例

```
function hook_overload_8() {
    if(Java.available) {
        Java.perform(function () {
            console.log("start hook");
            var targetMethod = 'add';
            var targetClass = 'com.roysue.roysueapplication.Ordinary_Class';
            var targetClassMethod = targetClass + '.' + targetMethod;
            //目标类
            var hook = Java.use(targetClass);
            //重载次数
            var overloadCount = hook[targetMethod].overloads.length;
            //打印日志：追踪的方法有多少个重载
            console.log("Tracing " + targetClassMethod + " [" + overloadCount + " overload(s)]");
            //每个重载都进入一次
            for (var i = 0; i < overloadCount; i++) {
                    //hook每一个重载
                    hook[targetMethod].overloads[i].implementation = function() {
                        console.warn("n*** entered " + targetClassMethod);
                        //可以打印每个重载的调用栈，对调试有巨大的帮助，当然，信息也很多，尽量不要打印，除非分析陷入僵局
                        Java.perform(function() {
                            var bt = Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Exception").$new());
                                console.log("nBacktrace:n" + bt);
                        });   
                        // 打印参数
                        if (arguments.length) console.log();
                        for (var j = 0; j < arguments.length; j++) {
                            console.log("arg[" + j + "]: " + arguments[j]);
                        }
                        //打印返回值
                        var retval = this[targetMethod].apply(this, arguments); // rare crash (Frida bug?)
                        console.log("nretval: " + retval);
                        console.warn("n*** exiting " + targetClassMethod);
                        return retval;
                    }
                }
            console.log("hook end");
        });
    }
}

```

### 1.4.2 拦截方法的所有方法重载代码示例详解

上面这段代码可以打印出`com.roysue.roysueapplication.Ordinary_Class`类中`add`方法重载的个数以及 hook 该类中所有的方法重载函数，现在来剖析上面的代码为什么可以对一个类中的所有的方法重载`HOOK`挂上钩子。首先我们定义了三个变量分别是`targetMethod、targetClass、targetClassMethod`，这三个变量主要于定义方法的名称、类名、以及类名 + 方法名的赋值，首先使用了`Java.use`获取了目标类对象，再获取重载的次数。

这里详细说一下如何获取的：`var method_overload = cls[<func_name>].overloads[index];`这句代码可以看出通过`cls`索引`func_name`到类中的方法，而后面写到`overloads[index]`是指方法重载的第`index`个函数，大致意思就是返回了一个`method`对象的第`index`位置的函数。而在代码中写道：`var overloadCount = hook[targetMethod].overloads.length;`，采取的方法是先获取类中某个函数所有的方法重载个数。继续往下走，开始循环方法重载的函数，刚刚开始循环时`hook[targetMethod].overloads[i].implementation`这句对每一个重载的函数进行`HOOK`。

这里也说一下`Arguments:Arguments`是`js`中的一个对象，`js`内的每个函数都会内置一个`Arguments`对象实例`arguments`，它引用着方法实参，调用其实例对象可以通过`arguments[]`下标的来引用实际元素，`arguments.length`为函数实参个数,`arguments.callee`引用函数自身。这就是为什么在该段代码中并看不到`arguments`的定义却能够直接调用的原因，因为它是内置的一个对象。好了，讲完了`arguments`咱们接着说，打印参数通过`arguments.length`来循环以及`arguments[j]`来获取实际参数的元素。

那现在来看`apply`，`apply`在`js`中是怎么样的存在，`apply`的含义是：应用某一对象的一个方法，用另一个对象替换当前对象，`this[targetMethod].apply(this, arguments);`这句代码简言之就是执行了当前的`overload`方法。执行完当前的`overload`方法并且打印以及返回给真实调用的函数，这样不会使程序错误。那么最终执行效果见下图 4-21：

![](https://mmbiz.qpic.cn/mmbiz_png/1qk1GGFlliagEVCmQQuLWJDTkpaH0atc7xlex0HtTxsjqr6VaQHU4pia3DaOyEUwp1xnK0Gpibgok66hMVmTZlGMw/640?wx_fmt=png)

图 4-21 终端显示  

可以看到成功打印了`add`函数的方法重载的数量以及`hook`打印出来的参数值、返回值！

### 1.5 Java 层拦截类的所有方法

学会了如何`HOOK`所有方法重载函数后，我们可以把之前学习的整合到一起，来`hook`指定类中的所有方法，也包括方法重载的函数。下面`js`中核心代码是利用重载函数的特点来`HOOK`全部的方法，普通的方法也是一个特殊方法重载，只是它只是一个方法而已，直接把它当作方法重载来`HOOK`就好了，打个比方正方形是特殊的长方形，而长方形是不是特殊的正方形。这个正方形是普通函数，而长方形是重载方法这样大家应该很好理解了~ 在上一章节中已经知道了如何`hook`方法重载，只是方法名称和类名是写死的，只需要把成员的`targetClass、targetMethod`定义方法中的参数即可, 在该例子中拿到指定类所有的所有方法名称，更加灵活使用了，代码如下。

### 1.5.1 拦截类的所有方法代码示例

```
function traceClass(targetClass)
{
    //Java.use是新建一个对象哈，大家还记得么？
    var hook = Java.use(targetClass);
    //利用反射的方式，拿到当前类的所有方法
    var methods = hook.class.getDeclaredMethods();
    //建完对象之后记得将对象释放掉哈
    hook.$dispose;
    //将方法名保存到数组中
    var parsedMethods = [];
    methods.forEach(function(method) {
        //通过getName()方法获取函数名称
        parsedMethods.push(method.getName());
    });
    //去掉一些重复的值
    var targets = uniqBy(parsedMethods, JSON.stringify);
    //对数组中所有的方法进行hook
    targets.forEach(function(targetMethod) {
        traceMethod(targetClass + "." + targetMethod);
    });
}
function hook_overload_9() {
    if(Java.available) {
        Java.perform(function () {
            console.log("start hook");
            traceClass("com.roysue.roysueapplication.Ordinary_Class");
            console.log("hook end");
        });
    }
}
s1etImmediate(hook_overload_9);

```

执行脚本效果可以看到，`hook`到了`com.roysue.roysueapplication.Ordinary_Class`类中所有的函数，在执行其被`hook`拦截的方法时候，也打印出了每个方法相应的的参数以及返回值，见下图 4-22。

![](https://mmbiz.qpic.cn/mmbiz_png/1qk1GGFlliagEVCmQQuLWJDTkpaH0atc7iciaiaJstBKbkk70K1aTBGvtKric9gAVWWwQUWYza9RvcV1M9UukFlP5sg/640?wx_fmt=png)

图 4-22 终端运行显示效果

### 1.6 Java 层拦截类的所有子类

这里的核心功能也用到了上一小章节中定义的`traceClass`函数，该函数只需要传入一个`class`路径即可对`class`中的函数完成注入`hook`。那么在本小章节来`hook`掉所有类的子类，使我们的脚本更加的灵活方便。通过之前的学习我们已经知道`enumerateLoadedClasses`这个`api`可以枚举所有的类，用它来获取所有的类然后再调用`traceClass`函数就可以对所有类的子进行全面的`hook`。但是一般不会`hook`所有的函数，因为`AndroidAPI`函数实在太多了，在这里我们需要匹配自己需要`hook`的类即可，代码如下。

```
//枚举所有已经加载的类
Java.enumerateLoadedClasses({
    onMatch: function(aClass) {
        //迭代和判断
        if (aClass.match(pattern)) {
            //做一些更多的判断，适配更多的pattern
            var className = aClass.match(/[L]?(.*);?/)[1].replace(///g, ".");
            //进入到traceClass里去
            traceClass(className);
        }
    },
    onComplete: function() {}
});

```

### 1.7 RPC 远程调用 Java 层函数

在`FRIDA`中，不但提供很完善的`HOOK`机制，并且还提供`rpc`接口。可以导出某一个指定的函数，实现在`python`层对其随意的调用，而且是随时随地想调用就调用，极其方便，因为是在供给外部的`python`，这使得`rpc`提供的接口可以与`python`完成一些很奇妙的操作，这些导出的函数可以是任意的`java`内部的类的方法，调用我们自己想要的对象和特定的方法。那我们开始动手吧，现在我们来通过`RPC`的导出功能将图 4-9 中的`add`方法供给外部调用，开始编写`rpc_demo.py`文件，这次是`python`文件了哦~ 不是`js`文件了

### 1.7.1 rpc 导出 Java 层函数代码示例

```
import codecs
import frida
from time import sleep
# 附加进程名称为：com.roysue.roysueapplication
session = frida.get_remote_device().attach('com.roysue.roysueapplication')
# 这是需要执行的js脚本，rpc需要在js中定义
source = """
    //定义RPC
    rpc.exports = {
        //这里定义了一个给外部调用的方法：sms
        sms: function () {
            var result = "";
            //嵌入HOOK代码
            Java.perform(function () {
                //拿到class类
                var Ordinary_Class = Java.use("com.roysue.roysueapplication.Ordinary_Class");
                //最终rpc的sms方法会返回add(1,3)的结果！
                result = Ordinary_Class.add(1,3);
             });
            return result;
        },
    };
"""
# 创建js脚本
script = session.create_script(source)
script.load()
# 这里可以直接调用java中的函数
rpc = script.exports
# 在这里也就是python下直接通过rpc调用sms()方法
print(rpc.sms())
sleep(1)
session.detach()

```

当我们执行`python rpc_demo.py`时先会创建脚本并且注入到目标进程，在上面的`source`实际上就是 js 逻辑代码了。在`js`代码内我们定义了`rpc`可以给`python`调用的`sms`函数，而`sms`函数内部嵌套调用`Java.perform`再对需要拿到的函数的类进行主动调用，把最终的结果返回作为`sms`的返回值，当我们在`python`层时候可以任意调用`sms`中的原型`add`方法~

### 1.8 综合案例一：在安卓 8.1 上 dump 蓝牙接口和实例

一个比较好的综合案例 ：`dump`蓝牙信息的 “加强版”——`BlueCrawl`。

```
VERSION="1.0.0"
setTimeout(function(){
    Java.perform(function(){
        Java.enumerateLoadedClasses({
                onMatch: function(instance){
                    if (instance.split(".")[1] == "bluetooth"){
                        console.log("[->]t"+lightBlueCursor()+instance+closeCursor());
                    }
                },
                onComplete: function() {}
            });
        Java.choose("android.bluetooth.BluetoothGattServer",{
                onMatch: function (instance){
                    ...
                onComplete: function() { console.log("[*] -----");}
            });
        Java.choose("android.bluetooth.BluetoothGattService",{
                onMatch: function (instance){
                    ...
                onComplete: function() { console.log("[*] -----");}
            });
         Java.choose("android.bluetooth.BluetoothSocket",{
                onMatch: function (instance){
                    ...
                onComplete: function() { console.log("[*] -----");}
            });
          Java.choose("android.bluetooth.BluetoothServerSocket",{
                onMatch: function (instance){
                    ...
                onComplete: function() { console.log("[*] -----");}
            });
          Java.choose("android.bluetooth.BluetoothDevice",{
                onMatch: function (instance){
                    ...
                onComplete: function() { console.log("[*] -----");}
            });
    });
},0);

```

该脚本首先枚举了很多蓝牙相关的类，然后`choose`了很多类，包括蓝牙接口信息以及蓝牙服务接口对象等，还加载了内存中已经分配好的蓝牙设备对象，也就是上文我们已经演示的信息。我们可以用这个脚本来 “查看”`App`加载了哪些蓝牙的接口，`App`是否正在查找蓝牙设备、或者是否窃取蓝牙设备信息等。

在电脑上运行命令：

`$ frida -U -l bluecrawl-1.0.0.js com.android.bluetooth`

执行该脚本时会详细打印所有蓝牙接口信息以及服务接口对象~~

### 1.9 综合案例二：动静态结合逆向 WhatsApp

我们来试下它的几个主要的功能，首先是本地库的导出函数。

```
setTimeout(function() {
    Java.perform(function() {
        trace("exports:*!open*");
        //trace("exports:*!write*");
        //trace("exports:*!malloc*");
        //trace("exports:*!free*");
    });
}, 0);

```

我们`hook`的是`open()`函数，跑起来看下效果：

```
$ frida -U -f com.whatsapp -l raptor_frida_android_trace_fixed.js --no-pause

```

如图所示`*!open*`根据正则匹配到了`openlog`、`open64`等导出函数，并`hook`了所有这些函数，打印出了其参数以及返回值。接下来想要看哪个部分，只要扔到 jadx 里，静态 “分析” 一番，自己随便翻翻，或者根据字符串搜一搜。比如说我们想要看上图中的`com.whatsapp.app.protocol`包里的内容，就可以设置`trace("com.whatsapp.app.protocol")`。可以看到包内的函数、方法、包括重载、参数以及返回值全都打印了出来。这就是`frida`脚本的魅力。当然，脚本终归只是一个工具，你对`Java`、安卓`App`的理解，和你的创意才是至关重要的。接下来可以搭配`Xposed module`看看别人都给`whatsapp`做了哪些模块，`hook`的哪些函数，实现了哪些功能，学习自己写一写。

### 结语

看完了之后对于 FRIDA 的基本篇章差不多学好了，对于 hook 所有函数已经不是啥难事了。更多问题可以在我的 github 上（`https://github.com/r0ysue/AndroidSecurityStudy`）联系我，谢谢大家。
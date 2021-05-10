> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/Gfpcwbpp3wtT3Pw-rwte7g)

前言
--

    还原抽取壳的关键即为需要等待壳将抽空的函数修复回去的时机再 dump 下 dex, 所以 dump 的时机很关键. 最近学习了 Fart 的原理, 但是不太喜欢编译刷机去脱壳, 比较独爱 Frida, 正好寒冰老师在直播课中也讲了 Fridafart 的脚本原理, 本文通过分析 Fridafart 的代码, 然后借鉴来写一个自己的脱壳脚本

FridaFart 学习分析与源码跟踪
-------------------

    先是 hook 了 libart.so 里面的 LoadMethod 方法, 然后加载了 fart.so 里面的方法 (这部分可以暂且不先去关注, 是用来保存 codeitem 的)

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqUXeEIJ92gzGagboBesDE6fOpVJMk0Rpd5NsyI1lmPziaHaIUs6iccBibg/640?wx_fmt=png)

    然后对 LoadMethod 方法中的 dexfile 参数进行解析然后 dump 到文件, 并缓存一份到全局对象中

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqLkoOTz2AhQnLqJzwYtlicI2ueSOfnSaxROWUULQD2GxyBmltzLvOVaA/640?wx_fmt=png)

    这里要注意的是寒冰老师 github 上的脚本适用于安卓 8 以上, 因为安卓 7 版本的 LoadMethod 参数和安卓 8 以上不一样

安卓 8 以上:

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqqEp17icZmbSdbdjy4Ffb2vf3e0Y9wMDrWPfMicPYNIok9VSQV91aicmYw/640?wx_fmt=png)

安卓 7.1:

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRq6XkicZTsK1MEWib6s5aeGDHeotuGXqM359te5m9V8xPp87cY16UxdoMA/640?wx_fmt=png)

    所以这里我们可以通过版本来判断修改参数位置由于它是 ClassLinker 中函数, Frida 第零个参数会指向 this, 所以要从第一个参数开始算.

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRq2R5nzBYWWkdKMa6NK03ZqaXT2H3H8ywkQoSoKcTvwgZ1HR78fU3lCQ/640?wx_fmt=png)

    后面通过 dexfile 的结构体偏移拿到 dex 文件的起始地址和大小, 这里涉及到 dexfile 的内存布局, 这里暂且还没学到, 不太了解, 就先照抄就好了. 下面就是先把第一次经过 LoadMethod 的 dexfile 给 dump 下来, 然后再做一个缓存, 为什么要缓存呢, 后面就马上介绍了.

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRq6x7PjtOKDhDHZXEfibU5A2lxuuiaIutaSiafq6uCuIvjFsZNWULIticwfg/640?wx_fmt=png)

    下面提供了一个 fart 的 api, 注释写了先是利用被动调用 (就是你人工点击触发的函数) 还原的函数进行 dump, 再利用枚举 classloader 传入 dealwithclassloader 这个 api, 最后在对所有缓存的 dex 进行 dump.

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqTJA3aiaeBSaC84nJYlbX1qUQjakuIiaI0icG6K0Zhdr2sib1eFaPJTY8FA/640?wx_fmt=png)

    这就是缓存的作用, dealwithclassloader 的这个 api 就是来枚举一个 classloader 中的所有类, 然后主动调用 loadClass 加载所有类使得其中方法可能会被还原, 然后利用之前 hook 保存的 dex 缓存, 等待 loadClass 加载完所有类, 然后进行 dump.

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqF7m8AGxwlibdxEibeqpMcDibBia9JUkYUDjQib4uERibbRlwVA1icnG40nmpQ/640?wx_fmt=png)

    为什么要选择 loadClass 呢, 一般加载类可以使用 Class.forName() 和 ClassLoader.loadClass(), 一个类的装载过程为三步. 装载 - 链接 - 初始化

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqsou49GPGiahsu1GDDurBKpoq31OzoQeB4icGibX2w1ViaokJMqCHT3OLIw/640?wx_fmt=png)

    两者的区别就是 forName 默认初始化一个类, 导致类的静态代码块执行, loadClass 默认不执行初始化, 有些壳会对抗你主动初始化了一些不会被初始化的类从而检测到你正在脱壳于是退出 app.

可以追踪一下源码, 调用了 loadClass 会发生什么.

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqRiahO545W5bmmum2OedicRo8Qf4tA69sNgatAbMic9J2xw2vxx9BFcsmA/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqw9vjju0gQsox1BsRlZC6qmuDmU5L4NQuOMmibFTN0yJJOTeJjTgtvfA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqQ7IJYyvQyh2bVEOnufQ4ggUpJLMgTs0ib8JtWQ696M5TfNLjgt0Miaxw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqjmibqCzubsLicmvTZI7POdaA9pEgX1QKMV1343Q1QibLXD4MfMVNJXVJQ/640?wx_fmt=png)

进入到 native 层

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqvkgKgf3OibVxeA67PC0zS3fWibRHYyWca06zqDOrBDoyettzic0MMo2nw/640?wx_fmt=png)

下面这个函数代码非常的多, 看的比较眼花撩乱, 做了很多的判断

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqA4SPq07A8JCUXZC39H74TjEXIgmAQYVcC1ZYI2cYheH0nD8E6dKwQw/640?wx_fmt=png)

其中 DefineClass 做了很多 Class 的准备

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqzSJDY2wDxosDXTY6JAr7lMIPUBo2ceCMrMv59NzpSBg958nL4iapP2A/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqhBZEjZK4IjQgBShEpFH7ia6mVJUEaTicLBMPHXkXY4BB1KmUQgFQmGDg/640?wx_fmt=png)

    通过注释可以知道一些 fields 和其他加载都在这个方法里, 重点关注 LoadClass

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqSDF3HukayPUF5uiaBrjRibNs4XL9akd2a0anYtsWXxzQVklCV1dZuYSw/640?wx_fmt=png)

这里可以发现 LoadMethod 和 另一个脱壳点 LinkCode

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqvRlNfbXtQNlA0tyBgtT1Eow7S4T7JazflyZuIibGRnnicHfbYn1juVCw/640?wx_fmt=png)

    Loadmethod 对 CodeItem 进行了第一次的 set, 说明调用 loadClass 加载一个类会对一个类中函数进行初始化. 并且这个函数直接出现了 DexFile 对象和 ArtMethod 对象, 是个非常好的时机, 所以 FridaFart 对这个函数进行了 hook 保存了其中的 DexFile 对象和 ArtMethod 对象 (使用 ArtMethod 对象来 dump codeitem 就是 fart 中的. bin 文件)

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRq891Vicxr16vk2bUKF1sBNiaXI8kdsBqCpGJibDFNnn6rPZLCM2r0YftEw/640?wx_fmt=png)

    接下来我们可以看看 LinkCode, LinkCode 函数对不同函数类型进行了不同的处理，进而完成对 ArtMethod 中相关变量的初始化工作，如针对 native 函数进行 method->UnregisterNative()，针对以 quick 模式或 interpreter 模式执行的函数的不同的初始化工作。虽然这个函数没有出现 dexfile 对象. 但是, ArtMethod 类提供了一个函数：GetDexFile()，该函数也可以获取到当前 ArtMethod 对象所在的 DexFile 对象引用，在获得了当前 DexFile 对象引用后，也依然可以 dump 得到当前内存中的 dex。将 ArtMethod 对象转为 DexFile 对象这个用 Frida 写不太方便, 但是发现 fridafart 的反射版本出现了将 ArtMethod 转为 DexFile 的方法, 我就直接拿来用了, fart.so 做了混淆, 暂时无法学习他的代码.

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqjc8umaia7c2OATlooo82j6UhFZcziaImYPzNnaSYUmYXCjZaucXMlENQ/640?wx_fmt=png)

    所以可以将 LinkCode 也加入我们 Frida 脚本的脱壳点, 经过测试 android7.1 无法使用 fart.so 的这个方法, 因为无法获取到 GetObsoleteDexCache 这个方法. 所以我写的 LinkCode 这个脱壳点只能在安卓 8 或以上使用 (需要将 fart.so push 到手机的 / data/app 目录下)

    完整流程图:

![](https://mmbiz.qpic.cn/mmbiz_jpg/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqnn6XGwwIYopX1Nk96Mg2ibdAOnjTFonMdW7udIvGdiaNGduKqzXbLokA/640?wx_fmt=jpeg)

编写 Frida 脚本
-----------

```
var LinkCode_addr;
var addrGetDexFile;
var funcGetDexFile;
var savepath = "/sdcard";
var dex_maps = {};
var LinkCode_artmethod_maps = {};
function init() {
    console.log("go into init," + "Process.arch:" + Process.arch);
    var module_libext = null;
    if (Process.arch === "arm64") {
        module_libext = Module.load("/data/app/fart64.so");
    } else if (Process.arch === "arm") {
        module_libext = Module.load("/data/app/fart.so");
    }
    if (module_libext != null) {
        addrGetDexFile = module_libext.findExportByName("GetDexFile");
        funcGetDexFile = new NativeFunction(addrGetDexFile, "pointer", ["pointer", "pointer"]);
    }
}
function FindArtAddr() {
    var symbols = Process.getModuleByName("libart.so").enumerateSymbols();
    for (var i = 0; i < symbols.length; i++) {
        var symbol = symbols[i];
        if (symbol.name.indexOf("ClassLinker") >= 0
        && symbol.name.indexOf("LinkCode") >= 0
        ) {
            //console.log(JSON.stringify(symbol))
            LinkCode_addr = symbol.address
        }
        // Android >= 8
        if (symbol.name.indexOf("ArtMethod") >= 0
           && symbol.name.indexOf("GetObsoleteDexCache") >= 0
        ) {
            addrGetObsoleteDexCache = symbol.address;
        }
    }
    hook_LinkCode();
}
function hook_LinkCode(){
    if(LinkCode_addr){
        console.log("LinkCode_addr", LinkCode_addr)
        Interceptor.attach(LinkCode_addr,{
            onEnter:function (args){
                this.artmethodptr = args[1];
            },onLeave: function (retval){
                    this.dexfileptr = funcGetDexFile(ptr(this.artmethodptr), ptr(addrGetObsoleteDexCache));
                    var dexfilebegin = Memory.readPointer(ptr(this.dexfileptr).add(Process.pointerSize * 1));
                    var dexfilesize = Memory.readU32(ptr(this.dexfileptr).add(Process.pointerSize * 2));
                    var dexfile_path = savepath + "/" + "LinkCode_" + dexfilesize + ".dex";
                    var dexfile_handle = null;
                    try {
                        dexfile_handle = new File(dexfile_path, "r");
                        if (dexfile_handle && dexfile_handle != null) {
                            dexfile_handle.close()
                        }
                    } catch (e) {
                        dexfile_handle = new File(dexfile_path, "a+");
                        if (dexfile_handle && dexfile_handle != null) {
                            var dex_buffer = ptr(dexfilebegin).readByteArray(dexfilesize);
                            dexfile_handle.write(dex_buffer);
                            dexfile_handle.flush();
                            dexfile_handle.close();
                            console.log("[First Dumpdex]:", dexfile_path);
                        }
                    }
                    var dexfileobj = new DexFile(dexfilebegin, dexfilesize);
                    if (dex_maps[dexfilebegin] == undefined) {
                        dex_maps[dexfilebegin] = dexfilesize;
                        console.log("got a dex:", dexfilebegin, dexfilesize);
                    }
                if (this.artmethodptr != null) {
                    var artmethodobj = new ArtMethod(dexfileobj, this.artmethodptr);
                    if (LinkCode_artmethod_maps[this.artmethodptr] == undefined) {
                        LinkCode_artmethod_maps[this.artmethodptr] = artmethodobj;
                    }
                }
            }
        })
    }
}
init()
FindArtAddr()

```

    也是一样做一份缓存 等待 loadclass 执行完再 dump.

    然后提一下如何通过 classloader 来拿到所有类的, 通过将 classloader 转为 basedexclassloader 拿到 pathlist 这个域 (需要判断是否继承自他或者他的子类, 不然没有这个域), 然后在拿到其中的 dexElements, 遍历后就可以拿到 dexFile 对象通过其中的 mCookie, 使用 getClassNameList 方法获取到所有类的名字然后去 dump, 下面我加了个过滤条件, 只 dump 包含 app 包名的类, 快速过滤无关代码. 这个 mCookie 是什么? 改写 fdex2 脚本的时候会再提到的.

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqZSSfSkkUfnEOcX54BzzaO9icycTco2fK5rA1JbDVZ7nveUhQFaOGEAA/640?wx_fmt=png)  

    说了这么多, 可以先看看 dump 效果. 测试了上篇文章的 app

    这个 app 貌似对 Loadmethod 做了处理, attach hook 不上这个函数, 用 spwan 模式 hook 就退出

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqG8XEq2XUJ6iaxBCOaPliaG9X296nrGKqw4zB69CURJ7hwKSl1GTs95bg/640?wx_fmt=png)  

所以可以用上刚刚写的 LinkCode 这个脱壳点了. spwan 模式也不崩溃了

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqq6hy02avZAna7DyfpwUDtvnWXAkf75OOKCu6aGBT7h2FiaGmm0g1ynw/640?wx_fmt=png)  

    然后调用 fart(), 我这里没对被动调用的 dex 进行 dump, 只等所有类加载完后再 dumpclass, 因为我没加上 dump codeitem 脚本 所以只对最晚时间的 dex 进行 dump 可以保证 dex 相对完整 不需要用修复脚本进行修复.

效果演示
----

![](https://mmbiz.qpic.cn/mmbiz_gif/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqdyO4UGicokzicVicpHic1wJMj9U2d6Od1b8mjJZRhiao0ibnfKHLCaEicpt2Q/640?wx_fmt=gif)

拖出来看看上一篇文章的那个被抽空的类.

dexfile_dexfile:

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqPbXgcLTUSAw4sfyelsWY1X3q0rf8zibOlY8iaBxq0ZibjIPhu27hE9NwQ/640?wx_fmt=png)

第一次 LinkCode dump 下来的

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqsP1SlCFsr5PZBC1PWFtT9UX3RuzZNmpeLHU251AFWEE6Q9A8PdDmvA/640?wx_fmt=png)

loadclass 后 dump 缓存的 dex

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqFG2vEqp4HWjTf9KsFWsUFe81cVBFubjFeOaUC2Z3D6sKPW4m2SPiaVQ/640?wx_fmt=png)

直接都还原了

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpTQ1CnX6MTYE5WFKMVU2zRqFYxQq54UAuA35icp9ticVRqPQJZQAwmnUE6v0UIf6KNJZzZP7fbXsMqw/640?wx_fmt=png)

    说明可以一定程度上对抗这种不清场型的抽取壳. 还原时机在 loadmethod 和 linkcode 之后的壳应该也不好对抗. 可以使用刷机版 fart 大杀器来对抗. 后续有空还会对 fart 原理进行介绍. 写了好多, 有点累, 下篇文章再水 fdex2 的 Frida 脚本吧

我将脚本上传到了 Github, 后台回复 “脱壳 “, 即可获得脚本地址和抽取壳样本.

•android8 运行 fart() 主动调用 loadclass 可能会崩溃, 这种情况下可以试试 android7, 会稳定一点, 但是 android7 不支持 hookLinkCode 可以手动注释.• 脚本只测试过 androi7.1.2 和 android8.1.0 两个版本 其他版本自行实验修改  
再次感谢寒冰老师~  

参考资料:

https://bbs.pediy.com/thread-254028.htm

https://blog.csdn.net/zhu929033262/article/details/77477402

https://www.cnblogs.com/zabulon/p/5826610.html
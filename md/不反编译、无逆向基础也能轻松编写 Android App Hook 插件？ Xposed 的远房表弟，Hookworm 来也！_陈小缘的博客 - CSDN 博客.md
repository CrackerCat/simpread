> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/u011387817/article/details/113482330)

### 前言

Xposed 的大名相信很多同学都不陌生，它提供了一种能力，可以在不修改原 apk 的情况下，以插件的方式改变目标 App 的某些行为。  
但随着 Android 系统版本的迭代，原来的 Xposed 已经不适合在高版本的系统上运行了，原 Xposed 作者也在 3 年前就停止了更新，取而代之的是 _Magisk_ + _Riru_ + _EdXposed_ 这一套组合。不过，基于此类框架开发 Hook 插件，是需要掌握一定的逆向知识的，比如你在进行 Hook 之前，首先要知道方法签名以及其执行时机，在没有源代码的情况下，这些信息只能从反编译的目标 apk 的 smali(或转 jar) 代码中获取，如果目标 apk 加了壳的话，还要先脱壳……  
而本篇文章将结合一个实践案例，给大家介绍无需反编译，对无逆向基础的同学非常友好的一个库：Hookworm（钩虫）。

### 诞生背景

前段时间在 WanAndroid 每日一问里有个问题：“应用进程中那 4 个 Binder 线程分别是跟谁通讯？”  
一番简单分析无果后，就想着写个 Xposed 插件来 hook Thread 对象的创建，但是看到有同学提醒：“Xposed 插件的`handleLoadPackage`方法是在`handleBindApplication`时才回调的！”。 没办法，只能找其他的方案了。  
忽然想到了 Magisk，可是我又不会做 Magisk 插件……  
第二天看了下自己一直在用的那个【微信指纹支付】的 Magisk 模块源码（其实之前也看过好多遍了，一直没看懂，一头雾水），发现核心代码其实就是 libs 下面的那个 apk 的 dex！  
反编译看了下，大概摸清了思路，但是想到用这种方式（先手动打包成 apk 放在插件项目 libs 下）开发起来太繁琐了，而且维护起来成本又高，还没有一个规范的模板，这样就很难抽出来为大家所用。  
于是心有不甘的我又继续在 github 上面搜 Riru 相关的模块，看到一个叫【QQ Simplify】的项目，是用来阉割 QQ 一些 “花里胡哨” 的功能的，看了下代码，它是在进程 Fork 之后，用反射把 ServiceFetcher 里面的 LayoutInflater 对象换成自己的代理类，这样就可以在布局 inflate 时，选择性地把一些 View 的宽高 set 为 0，达到隐藏的效果。  
结合这两个项目的部分代码以及思路，我封装出来一个入侵程度非常低的 Module，开发新的插件的话，只需要添加这个 Module 的依赖，然后在 module.properties 中配置一下模块属性就行了，非常简单！

### Hello Hookworm

那么接下来就跟大家一起编写一个寄生插件版的 Hello World，以便大家对 Hookworm 有个大致的了解。  
**编写和安装寄生插件，有几个前提：**

1.  目标手机系统版本至少是 6.0（API 23）；
    
2.  目标手机已正确安装 _[Magisk](https://github.com/topjohnwu/Magisk)_ （无从下手的同学可以搜索引擎搜一下: “【目标手机型号】Magisk 安装” 等字眼，相关资料挺多）；
    
3.  目标手机已刷入 _[Riru](https://github.com/RikkaApps/Riru)_ 模块（这个简单，只要成功安装 Magisk 之后，在 Magisk APP 里面就能直接搜索和刷入）；
    
4.  Android Studio 已升级到 4.1 或以上（因为 Hookworm 用到了 Gradle Plugin4.1.1 的一些新功能，需要更高版本的 Android Studio 才支持）；
    
5.  最好会一点 Kotlin 语法（Hookworm 的核心代码基本都是用 Kotlin 编写的，会 Kotlin 的话能帮助你更好地编写寄生插件）；
    

好，那我们开始吧。  
首先创建一个新项目：  
![](https://img-blog.csdnimg.cn/20210201010621766.png)  
新建项目的时候，注意 _Language_ 要选 **Kotlin**，还有 _Minimum SDK_ 选 **API 23**，因为等下依赖的 Module，都是以这个为基准的。  
项目初始化完成后，打开 _File_ -> _Project Structure_ ：  
![](https://img-blog.csdnimg.cn/20210201010647891.png)  
检查下当前的 Gradle Plugin 和 Gradle 版本是不是 **>=4.1.1** 和 **>=6.5**，如果不是请手动更正。

接着来创建一个入口类，名字随便，只要有：

```
public static void main(String processName);

```

这个方法就行。（注意`main`方法的参数`processName`是 String 而不是 String 数组哦）  
看下用 Kotlin 是怎么写的：

```
object HelloHookworm {

    @JvmStatic
    fun main(processName: String) {

    }
}

```

好，现在把 Hookworm Clone 下来，Github 地址在这里： [https://github.com/wuyr/HookwormForAndroid](https://github.com/wuyr/HookwormForAndroid) 。  
然后导入到项目中，并在 app 模块的`build.gradle`中添加 Hookworm 的依赖：

```
dependencies {

    ......

    implementation project(path: ':HookwormForAndroid')

}

```

同步一下，会发现报错了：

```
Please copy "module.properties.sample" and rename to "module.properties" and fill in the module information!

```

这是插件信息还没有完善的原因。  
切换到 Project 视图，把`HookwormForAndroid/module.properties.sample`复制一份，并改名为`module.properties`，然后编辑插件信息：

```
# 模块唯一标识，只能使用字母 + 下划线组合，如：my_module_id
moduleId=hello_hookworm

# 模块名称
moduleName=Hello Hookworm

# 模块作者
moduleAuthor=Demo

# 模块描述
moduleDescription=Hello World for Hookworm

# 版本名
moduleVersion=v1.0.0

# 版本号，只能填数字
moduleVersionCode=1

```

上面这几个属性都很好理解，可以随便填写，只要保证`moduleId`不跟其他已安装的模块有冲突就行了。  
还有几个比较重要的属性：

```
# 主入口类名，例：com.demo.ModuleMain
moduleMainClass=com.demo.hellohookworm.HelloHookworm

# 目标进程名/包名，即要寄生的目标。
targetProcessName=com.tencent.mm

# 自动安装模块
automaticInstallation=1

# 免重启安装
debug=1

```

第一个`moduleMainClass`是入口类的类名，在宿主进程启动时，这里填写的类里面的`public static void main(String processName)`方法就会被调用。我们就把刚刚创建的那个 HelloHookworm 的完整类名填上去；  
`targetProcessName`就是要寄生的目标，emmmm。。。。就选择微信吧，好记一点；  
`automaticInstallation`，编译后自动安装，肯定开着更好啦；  
最后一个`debug`属性，开启后会提供一种最小化安装的能力，也就是免重启安装，可以用来快速测试模块（注意，在正式发布时需要关闭）。

嗯，配置好属性之后，回到刚刚创建的 HelloHookworm，随便写点代码：

```
object HelloHookworm {

    private const val TAG = "HelloHookworm"

    @JvmStatic
    fun main(processName: String) {
        Log.i(TAG, processName)

        //监听宿主应用初始化
        Hookworm.onApplicationInitializedListener = { application ->
            Log.i(TAG, "Application initialized！")
            Toast.makeText(application, "Hello Hookworm！", Toast.LENGTH_SHORT).show()
        }
    }
}

```

可以看到这里第一时间打印了宿主的进程名，然后监听宿主初始化，在初始化完成时打印 log 并且 show 了一个 “Hello Hookworm” 的 Toast。

好啦，现在属性配置好了，代码也写好了，来看下效果吧。  
不过，因为寄生插件的特殊性，它并不是通过常规方式去安装的，所以在打包的时候也会跟平时有点不同：  
![](https://img-blog.csdnimg.cn/20210201010737599.png)  
就像上图一样，寄生插件它是通过 Project 的`assemble`这个 Task 来打包的，只需要双击一下就 ok 啦。  
在打包之前，可以先连上测试手机的 adb，等打包完成后就会自动安装并重启了~  
  
。。。。。。。。。。。。。。。。。。。。。。  
  
如果过程顺利的话，手机自动重启后，点开微信：  
![](https://img-blog.csdnimg.cn/20210201120559454.gif)  
哈哈哈，看到刚刚我们添加的 Toast 了没！！！  
再看下 log：

```
2021-02-01 09:52:26.956 21929-21929/? I/HelloHookworm: com.tencent.mm
2021-02-01 09:52:27.802 21929-21929/? I/HelloHookworm: Application initialized！

```

成功了！！！  
接下来开始 hook 实战。

### Hook 实战

为避免侵犯他人利益，我们决定选一个非商用的开源项目来作为这次的 Hook 目标：[WanAndroid 客户端](https://www.wanandroid.com/blog/show/2779) （选第二个）。  
安装打开看下：  
![](https://img-blog.csdnimg.cn/20210201010833712.jpg)  
首页的结构大致就是 Banner + 文章列表，底部有 4 个导航按钮，用来切换 Fragment。  
emmmm。。。。这个 Banner 图片就有点不好看，先把它换成好看的。  
**想一想，如何在不修改 apk 代码的前提下，替换掉图片呢？**  
要知道，寄生插件的代码是运行在宿主进程中的，也就是只要拿到对应的 Banner 对象，就能给它重新指定图片 Url，或者换成自己的 Adapter。

**那怎么才能拿到这个 Banner 对象？**  
刚刚做 Hello World 的 Demo 时，不是可以监听到 Application 的初始化嘛？  
既然有 Application 对象，那就能通过`registerActivityLifecycleCallbacks`方法，监听到所有 Activity 的生命周期，从而拿到目标 Activity 对象！然后就可以通过`findViewById`获取到想要的 Banner 对象啦！  
监听 Activity 生命周期这一步，Hookworm 也已经做了封装：

```
// Activity完整类名
val mainActivity = "com.demo.MainActivity"

// 监听目标Activity onCreate
Hookworm.registerOnActivityCreated(mainActivity) { activity, savedInstanceState ->
    Log.i(TAG, "Activity ${activity.javaClass.simpleName} created")
}

// 监听目标Activity onResume
Hookworm.registerOnActivityResumed(mainActivity) { activity ->
    Log.i(TAG, "Activity ${activity.javaClass.simpleName} resumed")
}

// 监听目标Activity onDestroy
Hookworm.registerOnActivityDestroyed(mainActivity) { activity ->
    Log.i(TAG, "Activity ${activity.javaClass.simpleName} destroyed")
}

```

不过，【监听 Activity 生命周期 + `findViewById`】这个方法，是明显存在问题的，因为我们不能确定目标 View 的 attach 时机，如果目标 View 在`findViewById`之后才加载，这个方法就失效了。  
**那有没有办法监听到目标 View 加载呢？** 要是可以的话，问题就解决了。  
有，可以用反射把 Activity 对应的 LayoutInflater 替换成自己做过手脚的类。  
这一步 Hookworm 也已经帮我们实现了：

```
// Activity完整类名
val mainActivity = "com.demo.MainActivity"

//劫持Activity布局加载
Hookworm.registerPostInflateListener(mainActivity) { resourceId, resourceName, rootView ->
    // do something…
}

```

Hookworm 的`registerPostInflateListener`方法会在目标 Activity 每次加载布局资源时回调后面的 lambda。  
lambda 的三个参数分别是：

*   _resourceId_: 目标 Activity 正在加载的布局资源 id；
    
*   _resourceName_: 目标 Activity 正在加载的布局资源名称；
    
*   _rootView_: inflate 完成后的 View 对象；
    

我们可以通过这个方法来监听首页 Activity 的布局加载，在它每次 inflate 布局之后去查找 Banner 的实例。  
不过在此之前，需要先知道**对应 Activity 完整类名**和 **Banner 的 id 值或 id 名称**。

获取 Activity 类名有很多种方式，我就比较喜欢用 shell 命令：

```
adb shell
dumpsys activity activities top | grep "Hist #" | awk 'NR==1{print $6}'&&exit

```

打开 WanAndroid 应用首页，然后在终端里执行上面的命令：  
![](https://img-blog.csdnimg.cn/20210201010902710.png)  
类名就出来了：`per.goweii.wanandroid/.module.main.activity.MainActivity`（注意这里多了个`/`符号，等下用到的时候要删掉）。

至于 Banner 的 id 值，是需要反编译才能看到的，既然文章标题说了不用反编译，那就换一种方式——借助 Android SDK 提供的 ***UIAutomatorViewer*** 来获取它的 id 资源名称。

### 布局分析

uiautomatorviewer 工具位于`sdk/tools/bin`目录下（Windows 系统是 bat 文件），把它拖到终端里 enter 就能运行了：  
![](https://img-blog.csdnimg.cn/20210201010929555.png)  
左上角四个功能按钮分别是：打开、**获取屏幕当前页面布局信息**、**获取精简 (去掉多余嵌套) 后的页面布局信息**、保存。

> 它获取布局信息其实是借助 ***`uiautomator`*** 命令来完成的，这个`uiautomator`内部会通过 AccessibilityService 把视图层级信息 dump 出来。

手机再次打开 WanAndroid 应用首页，然后点一下**获取屏幕当前页面布局信息**按钮：  
![](https://img-blog.csdnimg.cn/20210201010959732.png)  
通过右边的布局信息可以知道，这个 Banner 原来是个 ViewPager，id 名称是`bannerViewPager`，还可以看到它的 Item 也只是 ImageView 而已。  
有了 id 名称，就能通过 Hookworm 提供的扩展函数`findViewByIDName`来获取到这个 ViewPager 对象，ViewPager 里刚好有个监听 Adapter 变更的方法：`addOnAdapterChangeListener`，我们可以借助这个方法，在 ViewPager 的 Adapter 变更时将目标 Adapter 替换掉，这样就能确保 Banner 总是能显示修改后的图片了。

### 编写 Hook 代码

像前面创建 _Hello Hookworm_ 那样，先创建一个 _Hookworm For WanAndroid_ 项目，并配置插件信息（目标应用包名记得要改成 WanAndroid 应用的包名）。  
然后编写代码：

```
object Main {

    private fun Any?.logD() = Log.d("Main", toString())

    @JvmStatic
    fun main(processName: String) {
        // 首页Activity类名
        val mainActivity = "per.goweii.wanandroid.module.main.activity.MainActivity"

        // 拦截mainActivity的布局加载
        Hookworm.registerPostInflateListener(mainActivity) { _, resourceName, rootView ->
            rootView?.apply {
                hookBanner(resourceName)
            }
        }
    }

    private fun View.hookBanner(resourceName: String) {
        // 根据id名称: "bannerViewPager" 查找ViewPager
        findViewByIDName<ViewPager>("bannerViewPager")?.let { viewPager ->
            // 打印布局名称和查找到的ViewPager对象实例
            "bannerViewPager所在布局：$resourceName".logD()
            viewPager.logD()
        }
    }
}

```

在找到 ViewPager 之后打印所在布局名称和 ViewPager 的实例，运行看下效果：

```
java.lang.ClassCastException: com.youth.banner.view.BannerViewPager cannot be cast to androidx.viewpager.widget.ViewPager
        at com.wuyr.hookwormforwanandroid.Main$main$1.invoke(Main.kt:24)
        at com.wuyr.hookwormforwanandroid.Main$main$1.invoke(Main.kt:13)
        at com.wuyr.hookworm.extensions.PhoneLayoutInflater.inflate(PhoneLayoutInflater.kt:66)
        at android.view.LayoutInflater.inflate(LayoutInflater.java:532)
        at com.wuyr.hookworm.extensions.PhoneLayoutInflater.inflate(PhoneLayoutInflater.kt:57)
        at com.youth.banner.Banner.initView(Banner.java:102)
        at com.youth.banner.Banner.<init>(Banner.java:96)
        at com.youth.banner.Banner.<init>(Banner.java:84)
        at com.youth.banner.Banner.<init>(Banner.java:80)
        at per.goweii.wanandroid.module.home.fragment.HomeFragment.createHeaderBanner(HomeFragment.java:395)
        at per.goweii.wanandroid.module.home.fragment.HomeFragment.initView(HomeFragment.java:300)
        ......

```

**咦？为什么会报强转失败呢？这个 Banner 不就是 ViewPager 嘛？！**  
其实是因为寄生插件的 DexClassLoader 和宿主的 PathClassLoader 都分别加载了 ViewPager 这个 Class 造成的，就像这样：  
![](https://img-blog.csdnimg.cn/20210201011033628.jpg)  
通俗地说，寄生插件的 DexClassLoader 跟宿主的 PathClassLoader 是属于叔侄关系，并不是直系亲属，不能进行 **Parent-Delegation**，所以才会各自加载一次 ViewPager 类。  
要解决这个问题也很简单，把寄生插件的 DexClassLoader 转接到宿主的 PathClassLoader 下面（~强制户口迁移~），让它们变成直系亲属：  
![](https://img-blog.csdnimg.cn/20210201011058194.jpg)  
这样寄生插件在使用到 ViewPager 时就会优先让宿主的 PathClassLoader 去加载了。

具体要怎么做呢：  
在调用`registerPostInflateListener`之前，加上这句代码：

```
Hookworm.transferClassLoader = true

```

再修改一下 build.gradle，把几个相关的依赖库从 _implementation_ 改成 _compileOnly_（只编译，不打包）：

```
dependencies {
    compileOnly 'androidx.core:core-ktx:1.3.2'
    compileOnly 'androidx.appcompat:appcompat:1.2.0'
    compileOnly 'com.google.android.material:material:1.2.1'
}

```

就行了，编译运行看下 log（如果编译时报错的话，直接删掉对应文件即可，比如：`AAPT: error: style attribute 'attr/colorPrimary xxx' not found.`之类的错误，可以把`themes.xml`删掉，还有 Manifest 里面的`android:theme`也去掉）：

```
D/Main: bannerViewPager所在布局：banner
D/Main: com.youth.banner.view.BannerViewPager{3b21ab4 VFED..... ......I. 0,0-0,0 #7f080071 app:id/bannerViewPager}

```

成功打印了！  
接下来开始替换图片。

### 替换 Banner 图片

就按照之前说的那样做：借助`addOnAdapterChangeListener`方法在 ViewPager 的 Adapter 变更时将目标 Adapter 替换掉，这样就能确保 Banner 总是能显示修改后的图片了。  
看看代码要怎么写：

```
object Main {
    
    ......

    private fun View.hookBanner(resourceName: String) {
        // 根据id名称: "bannerViewPager" 查找ViewPager
        findViewByIDName<ViewPager>("bannerViewPager")?.let { viewPager ->
            // 打印布局名称和查找到的ViewPager对象实例
            "bannerViewPager所在布局：$resourceName".logD()
            viewPager.logD()

            // 监听Adapter变更，在每次Adapter变更时替换掉目标Adapter
            viewPager.addOnAdapterChangeListener(object : ViewPager.OnAdapterChangeListener {

                // 自己的Adapter
                private val adapter = ImageAdapter(context)

                override fun onAdapterChanged(
                    viewPager: ViewPager, oldAdapter: PagerAdapter?, newAdapter: PagerAdapter?
                ) {
                    // 先移除监听避免递归调用
                    viewPager.removeOnAdapterChangeListener(this)
                    viewPager.adapter = adapter
                    viewPager.addOnAdapterChangeListener(this)
                }
            })
        }
    }
}

```

好，现在就差一个加载自己的图片的 Adapter 了。  
不过我们这次并不打算直接依赖图片加载框架，而是先看宿主依赖了哪个，我们直接拿来用。。。

### 借用宿主类库

刚刚通过`Hookworm.transferClassLoader = true`把插件 ClassLoader 转接到了宿主 ClassLoader 下面，这样就已经能直接使用宿主里面的资源了。  
比如加载图片的类库，我们可以先测试下宿主有没有使用一些常见的图片加载框架：

```
object Main {

    private fun Any?.logD() = Log.d("Main", toString())

    @JvmStatic
    fun main(processName: String) {
        Hookworm.transferClassLoader = true

        ......

        Hookworm.onApplicationInitializedListener = {

            fun String.classExists() = runCatching { Class.forName(this) }.isSuccess

            when {
                "com.facebook.drawee.view.SimpleDraweeView".classExists() -> "正在使用Fresco".logD()

                "com.squareup.picasso3.Picasso".classExists() -> "正在使用Picasso".logD()

                "com.bumptech.glide.Glide".classExists() -> "正在使用Glide".logD()

                else -> "没有使用常见的图片加载框架".logD()
            }
        }
    }

    ......

}

```

只要调用 Class.`forName`后不报 _NoClassDefFoundError_ 就说明宿主添加了对应类库的依赖。  
编译运行，会看到打印的是`"正在使用Glide"`这句 log，那现在可以直接在插件里使用 Glide 了。

先给 build.gradle 加上 glide 的依赖（注意使用的是 _compileOnly_ 而非 _implementation_）：

```
compileOnly 'com.github.bumptech.glide:glide:4.11.0'

```

准备几张图片：  
https://c-ssl.duitang.com/uploads/item/201708/13/20170813095305_FSQhj.thumb.700_0.jpeg  
https://c-ssl.duitang.com/uploads/item/201512/05/20151205212633_nFx3d.thumb.700_0.jpeg  
https://c-ssl.duitang.com/uploads/item/201606/12/20160612235102_z3dja.thumb.700_0.jpeg  
https://c-ssl.duitang.com/uploads/item/201707/27/20170727121828_Z5TRA.thumb.700_0.png  
https://c-ssl.duitang.com/uploads/item/201707/27/20170727122213_3HBaN.thumb.700_0.png  
https://c-ssl.duitang.com/uploads/item/201512/04/20151204202153_nEUMt.thumb.700_0.jpeg

扩展一个 PagerAdapter：

```
class ImageAdapter(context: Context) : PagerAdapter() {

    private val imageUrls = arrayOf(
        "https://c-ssl.duitang.com/uploads/item/201708/13/20170813095305_FSQhj.thumb.700_0.jpeg",
        "https://c-ssl.duitang.com/uploads/item/201512/05/20151205212633_nFx3d.thumb.700_0.jpeg",
        "https://c-ssl.duitang.com/uploads/item/201606/12/20160612235102_z3dja.thumb.700_0.jpeg",
        "https://c-ssl.duitang.com/uploads/item/201707/27/20170727121828_Z5TRA.thumb.700_0.png",
        "https://c-ssl.duitang.com/uploads/item/201707/27/20170727122213_3HBaN.thumb.700_0.png",
        "https://c-ssl.duitang.com/uploads/item/201512/04/20151204202153_nEUMt.thumb.700_0.jpeg"
    )

    private val imageViews = ArrayList<ImageView>().apply {
        imageUrls.forEach { url ->
            add(ImageView(context).apply {
                scaleType = ImageView.ScaleType.CENTER_CROP
                Glide.with(context).load(url).into(this)
            })
        }
    }

    override fun instantiateItem(container: ViewGroup, position: Int) =
        imageViews[position].also { container.addView(it) }

    override fun destroyItem(container: ViewGroup, position: Int, `object`: Any) =
        container.removeView(imageViews[position])

    override fun getCount() = imageUrls.size

    override fun isViewFromObject(view: View, `object`: Any) = view == `object`
}

```

编译运行，看下效果：  
![](https://img-blog.csdnimg.cn/20210201011139935.gif)  
哈哈哈哈，成功替换了！  
那接下来试着拦截首页文章列表的 Item 点击吧。

### 拦截点击事件

有同学可能会想：不就是给 Item 重新设置一个 OnClickListener 嘛，这有什么的。  
emmmm，如果只是粗暴地直接给 Item 重新 setOnClickListener，那就不能保留宿主原来的点击逻辑了，这确实没什么可说的，不过我们要的是可以随心所欲地控制每一次的点击事件。  
比如只把文章标题含有 “每日一问” 字眼的交给宿主去处理，其余的 Item 在点击时都弹出一个 “禁止点击” 的 Dialog。  
这就需要先拿到 Item 原来的 OnClickListener，但是 View 的 OnClickListener 都是不公开的，只能用反射来获取，再加上一个列表那么多 Item，难道还要用 List 装起来？这也太麻烦了叭！  
还好 Hookworm 替我们做了这个事情，有个叫`setOnClickProxy`的扩展方法，它会在回调时把旧的（原来的）OnClickListener 实例也传回来，像这样：

```
view.setOnClickProxy { targetView, oldListener -> 
    if (xxx) {
        // Do something...
    } else {
        // 交给宿主原有listener去处理
        oldListener?.onClick(targetView)
    }
}

```

好，那现在来看看首页的文章列表是个什么 View，打开 UIAutomatorViewer，dump 一下视图：  
![](https://img-blog.csdnimg.cn/20210201011317442.png)  
是 RecyclerView，id 名就叫`rv`。  
**在开始 hook 之前，有一个问题需要解决：**  
因为 RecyclerView 的 Item 都是会复用的，每个 Item 复用时，都会经过一次`onBindViewHolder`方法，通常 Item 的点击事件都会在这里去设置。如果插件的点击代理是在`onBindViewHolder`调用前设置的，那就不起作用了（Listener 会被覆盖），要是在 View 显示出来之后才设置，也有可能 Item 会先被点击，那时候走的还是原来的点击逻辑。  
所以必须找到一个时机：在`onBindViewHolder`执行之后，在 Item 显示出来之前。  
有没有想到呢？  
RecyclerView 有个`addOnChildAttachStateChangeListener`方法，可以监听到每个 Item 的 _Attached_ 和 _Detached_！我们可以在 Item Attached 时给它设置点击代理！  
看看代码怎么写：

```
object Main {

    @JvmStatic
    fun main(processName: String) {

        ......        

        Hookworm.registerPostInflateListener(mainActivity) { _, resourceName, rootView ->
            rootView?.apply {
                if (resourceName == "banner") {
                    hookBanner(resourceName)
                }
                hookArticleItem(resourceName)
            }
        }
    }

    private fun View.hookArticleItem(resourceName: String) {
        // 根据id名“rv” 找到首页文章列表RecyclerView实例
        findViewByIDName<RecyclerView>("rv")?.let { recyclerView ->
            // 监听Item的attach状态
            recyclerView.addOnChildAttachStateChangeListener(object :
                RecyclerView.OnChildAttachStateChangeListener {

                private val dialog = AlertDialog.Builder(context).setMessage("禁止点击！").create()

                private val onClickProxy: (view: View, oldListener: View.OnClickListener?) -> Unit =
                    { view, oldListener ->
                        // 查找id名为“tv_title”的TextView
                        view.findViewByIDName<TextView>("tv_title")?.let { titleView ->
                            // 检查是否包含 “每日一问” 字眼
                            if (titleView.text.toString().contains("每日一问")) {
                                // 有则交给宿主处理
                                oldListener?.onClick(view)
                            } else {
                                // 没有就弹出dialog
                                dialog.show()
                            }
                        } ?: oldListener?.onClick(view) // 没找到，交给宿主去处理
                    }

                override fun onChildViewAttachedToWindow(child: View) {
                    // 在Item每次attach之后重新设置点击代理
                    child.setOnClickProxy(onClickProxy)
                }

                override fun onChildViewDetachedFromWindow(child: View) {
                }
            })
        }
    }

    ......

}

```

看看效果怎么样：  
![](https://img-blog.csdnimg.cn/20210201011200857.gif)  
可以了，现在只有点击 “每日一问” 的 Item 才会跳转 Web 页面，点击其他的 Item 都会弹出 “禁止点击” Dialog，跟预期的一样。

### 隐藏多余模块

前面几个小节都只是直接调用目标对象原有的 api 来实现 UI 的修改，看上去好像有点过于简单了，那现在就试着结合反射来把 4 个导航页改成 2 个。  
先把底部的第 3，第 4 个 Tab 移除掉吧，打开 UIAutomatorViewer：  
![](https://img-blog.csdnimg.cn/20210201110159105.png)  
可以看到底部的导航栏是一个 LinearLayout，4 个子 View 都是 RelativeLayout。分别点一下这 4 个子 View，会发现它们的 id 名称都是相同的，都叫`ll_ltab`，这说明了什么？说明它们很有可能都是来自同一个独立的 xml 布局，这样我们要在它 inflate 时移除掉后面两个的话，就必须加一个变量去记录当前 inflate 的数量，有点麻烦，干脆换一个方式吧：  
我们可以监听它父容器的子 View 添加，在它的子 View 数量大于 2 的时候，移除掉后面的子 View：

```
object Main {

    @JvmStatic
    fun main(processName: String) {
        
        ......

        Hookworm.registerPostInflateListener(mainActivity) { _, resourceName, rootView ->
            rootView?.apply {
                ......
                removeTabs(resourceName)
            }
        }
    }

    private fun View.removeTabs(resourceName: String) {
        // 查找ll_bb，监听其子View的添加
        findViewByIDName<ViewGroup>("ll_bb")?.setOnHierarchyChangeListener(
            object : ViewGroup.OnHierarchyChangeListener {
                override fun onChildViewAdded(parent: View, child: View) {
                    // 转成ViewGroup
                    (parent as ViewGroup).run {
                        // 当子View数量大于2时移除最后一个
                        if (childCount > 2) {
                            removeViewAt(2)
                        }
                    }
                }

                override fun onChildViewRemoved(parent: View?, child: View?) {
                }
            })
    }
}

```

看看效果：  
![](https://img-blog.csdnimg.cn/20210201113408912.gif)  
嗯，现在底部的 Tab 是移除了，但实际的页面还没移除，向右滑动还是能看到。  
回到 UIAutomatorViewer 窗口，翻一下右边视图层级信息，会发现这个切换页面的 View 其实也是 ViewPager，id 名称是`vp_tab`，那移除它的页面，我们可以从 Adapter 下手。  
先看一下它设置的 Adapter 里面都有些什么：

```
private fun View.removeTabs(resourceName: String) {

    ......

    // 查找id名为“vp_tab”的ViewPager
    findViewByIDName<ViewPager>("vp_tab")?.let { viewPager ->
        // 监听Adapter变更
        viewPager.addOnAdapterChangeListener { _, _, newAdapter ->
            // 遍历Adapter所有变量
            newAdapter?.javaClass?.declaredFields?.forEach { f ->
                f.isAccessible = true
                // 分别打印变量修饰符、变量类型、变量名、变量值
                ("${Modifier.toString(f.modifiers)} ${f.type.simpleName} ${f.name} = ${f.get(newAdapter)};").logD()
            }
        }
    }
}

```

编译运行，看下 log：

```
D/Main: private Page[] mPages = null;
D/Main: private final LinearLayout mTabContainer = android.widget.LinearLayout{5cfd786 V.E...... ......I. 0,0-0,0 #7f080172 app:id/ll_bb};
D/Main: private final int mTabItemRes = 2131427509;
D/Main: private final ViewPager mViewPager = androidx.viewpager.widget.ViewPager{c8e1874 VFED..... ......I. 0,0-0,0 #7f0802b4 app:id/vp_tab};

```

第一个数组`mPages`，估计就是页面 Item 了，不过现在是 null 可能是打印的时候数据还没有准备好，我们可以套一个 post，等它显示出来的时候再打印：

```
private fun View.removeTabs(resourceName: String) {

    ......

    // 查找id名为“vp_tab”的ViewPager
    findViewByIDName<ViewPager>("vp_tab")?.let { viewPager ->
        // 监听Adapter变更
        viewPager.addOnAdapterChangeListener { _, _, newAdapter ->
            viewPager.post {
                // 遍历Adapter所有变量
                newAdapter?.javaClass?.declaredFields?.forEach { f ->
                    f.isAccessible = true
                    // 分别打印变量修饰符、变量类型、变量名、变量值
                    ("${Modifier.toString(f.modifiers)} ${f.type.simpleName} ${f.name} = ${
                        // 如果变量类型是数组，则直接打印数组内容
                        if (f.type.isArray) (f.get(newAdapter) as Array<*>).contentToString()
                        else f.get(newAdapter)
                    };").logD()
                }
            }
        }
    }
}

```

编译运行，看下 log：

```
D/Main: private Page[] mPages = [per.goweii.basic.core.adapter.TabFragmentPagerAdapter$Page@efc18e5, per.goweii.basic.core.adapter.TabFragmentPagerAdapter$Page@70a8fba, per.goweii.basic.core.adapter.TabFragmentPagerAdapter$Page@b260e6b, per.goweii.basic.core.adapter.TabFragmentPagerAdapter$Page@95aedc8];
D/Main: private final LinearLayout mTabContainer = android.widget.LinearLayout{da5e61 V.E...... ........ 0,1749-1080,1878 #7f080172 app:id/ll_bb};
D/Main: private final int mTabItemRes = 2131427509;
D/Main: private final ViewPager mViewPager = androidx.viewpager.widget.ViewPager{5c1ed86 VFED..... .......D 0,0-1080,1878 #7f0802b4 app:id/vp_tab};

```

可以看到`mPages`里有四个元素，刚好对应了四个导航页，可以断定它储存的就是导航页的 Item 实例了，那现在试试用反射把最后 2 个元素移除掉：

```
private fun View.removeTabs(resourceName: String) {

    ......

    // 查找id名为“vp_tab”的ViewPager
    findViewByIDName<ViewPager>("vp_tab")?.let { viewPager ->
        // 监听Adapter变更
        viewPager.addOnAdapterChangeListener { _, _, newAdapter ->
            viewPager.post {
                newAdapter?.let { adapter ->
                    // 取出Adapter变量mPages
                    adapter::class.get<Array<*>>(adapter, "mPages")?.let { pages ->
                        // 只保留前2个元素
                        val newPages = pages.filterIndexed { index, _ -> index < 2 }.toTypedArray()
                        // 重新赋值
                        adapter::class.set(adapter, "mPages", newPages)
                    }
                    // 通知Adapter数据变更
                    adapter.notifyDataSetChanged()
                }
            }
        }
    }
}

```

运行看看：

```
java.lang.IllegalArgumentException: field per.goweii.basic.core.adapter.TabFragmentPagerAdapter.mPages has type per.goweii.basic.core.adapter.TabFragmentPagerAdapter$Page[], got java.lang.Object[]
        at java.lang.reflect.Field.set(Native Method)
        at com.wuyr.hookworm.utils.ReflectUtilKt.set(ReflectUtil.kt:35)
        at com.wuyr.hookworm.utils.ReflectUtilKt.set(ReflectUtil.kt:207)
        at com.wuyr.hookwormforwanandroid.Main$removeTabs$2$1$1.run(Main.kt:72)
        at android.os.Handler.handleCallback(Handler.java:938)
        at android.os.Handler.dispatchMessage(Handler.java:99)
        at android.os.Looper.loop(Looper.java:223)
        at per.goweii.ponyo.crash.Crash$Companion$initialize$1.run(Crash.kt:23)
        at android.os.Handler.handleCallback(Handler.java:938)
        at android.os.Handler.dispatchMessage(Handler.java:99)
        at android.os.Looper.loop(Looper.java:223)
        at android.app.ActivityThread.main(ActivityThread.java:7656)
        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:592)
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:947)

```

额，报错了，我们给的是 Object 数组，它要的是`per.goweii.basic.core.adapter.TabFragmentPagerAdapter$Page`数组。  
这个也不难解决，直接用反射创建对应类型的数组就行了：

```
private fun View.removeTabs(resourceName: String) {

    ......

    // 查找id名为“vp_tab”的ViewPager
    findViewByIDName<ViewPager>("vp_tab")?.let { viewPager ->
        // 监听Adapter变更
        viewPager.addOnAdapterChangeListener { _, _, newAdapter ->
            viewPager.post {
                newAdapter?.let { adapter ->
                    // 取出Adapter变量mPages
                    adapter::class.get<Array<*>>(adapter, "mPages")?.let { pages ->
                        // 通过反射创建长度为2的数组
                        val newPages = java.lang.reflect.Array.newInstance(
                            Class.forName("per.goweii.basic.core.adapter.TabFragmentPagerAdapter\$Page"),
                            2
                        ) as Array<Any?>
                        // 只取前面2个元素
                        newPages[0] = pages[0]
                        newPages[1] = pages[1]
                        // 重新赋值
                        adapter::class.set(adapter, "mPages", newPages)
                    }
                    // 通知Adapter数据变更
                    adapter.notifyDataSetChanged()
                }
            }
        }
    }
}

```

里面的 class.`get`、class.`set`是 Hookworm 的 ReflectUtil 里面的扩展函数，借助它们可以很方便地使用反射操作。  
好了，运行看下效果：  
![](https://img-blog.csdnimg.cn/20210201144348895.gif)  
怎么滑都滑不到第 3 页，证明后 2 页的 Item 实例已经成功被移除。

那么，本次的 Hook 实战也就告一段落了，**回顾一下：**  
我们替换了目标应用的 Banner 图片、拦截了首页文章列表的 Item 点击、移除了多余的导航页，一共只用了 100 多行的代码哦~

### 能力扩展

整篇文章看下来，貌似 Hookworm 最多也只能通过反射和动态代理之类的方式来进行一些浅层次的 Hook 操作，没办法像 Xposed 那样可以随意拦截任何方法。  
**Hookworm 的能力还可以再强大一点吗？**  
当然可以！Hookworm 内部已经对寄生插件带有 so 的依赖库做了处理，也就是说，你现在可以直接在插件里依赖一些诸如 _epic_ 、 _SandHook_ 、 _YAHFA_ 等 ART Hook 框架，**让你的寄生插件马上拥有像 Xposed 一样的能力！**，如果你会 JS，还可以在插件中直接使用 _frida_！

### 好啦，文章到此结束，有错误的地方请指出，谢谢大家！

> **免责声明：** 文章所介绍知识点仅用于学习研究，利用本文知识进行非法行为造成的一切后果自负！

### Github 地址：[https://github.com/wuyr/HookwormForAndroid](https://github.com/wuyr/HookwormForAndroid) 欢迎 Star
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bugstack.cn](https://bugstack.cn/itstack-demo-agent/2020/05/12/%E5%AD%97%E8%8A%82%E7%A0%81%E7%BC%96%E7%A8%8B-Byte-buddy%E7%AF%87%E4%BA%8C-%E7%9B%91%E6%8E%A7%E6%96%B9%E6%B3%95%E6%89%A7%E8%A1%8C%E8%80%97%E6%97%B6%E5%8A%A8%E6%80%81%E8%8E%B7%E5%8F%96%E5%87%BA%E5%85%A5%E5%8F%82%E7%B1%BB%E5%9E%8B%E5%92%8C%E5%80%BC.html)

> 通过对 Byte-buddy 高级 API 的委托方式的使用，再加上注解 @Origin、@SuperCall 等获取方法在执行过程中的入参信息方法的出参结果，最终学习委托处理的方式对方法进行监控。

作者：小傅哥  
博客：[https://bugstack.cn](https://bugstack.cn/)

> 沉淀、分享、成长，让自己和他人都能有所收获！

一、前言
----

**案例**是剥去外衣包装展示出核心功能的最佳学习方式！

就像是我们研究字节码编程最终是需要应用到实际场景中，例如：实现一款非入侵的全链路最终监控系统，那么这里就会包括一些基本的核心功能点；`方法执行耗时`、`出入参获取`、`异常捕获`、`添加链路ID`等等。而这些一个个的功能点，最快的掌握方式就是去实现他最基本的功能验证，这个阶段基本也是技术选型的阶段，验证各项技术点是否可以满足你后续开发的需求。否则在后续开发中，如果已经走了很远的时候再发现不适合，那么到时候就很麻烦了。

在前面的`ASM`、`Javassist` 章节中也有陆续实现过获取方法的出入参信息，但实现的方式还是偏向于_字节码控制_，尤其`ASM`，更是需要使用到字节码指令将入参信息压栈操作保存到局部变量用于输出，在这个过程中需要深入了解`Java虚拟机规范`，否则很不好完成这一项的开发。但！`ASM`也是性能最牛的。其他的字节码编程框架都是基于它所开发的。**关于这部分系列文章可以访问链接进行专题系列的学习**：[https://bugstack.cn/itstack/itstack-demo-bytecode.html](https://bugstack.cn/itstack/itstack-demo-bytecode.html)

**那么**，本章节我们会使用 `Byte-buddy` 来实现这一功能，在接下来的操作中你会感受到这个字节码框架的魅力，它的 _API_ 更加高级也更符合普遍易接受的操作方式进行处理。

二、开发环境
------

1.  JDK 1.8.0
2.  byte-buddy 1.10.9
3.  byte-buddy-agent 1.10.9
4.  本章涉及源码在：`itstack-demo-bytecode-2-02`，可以关注**公众号**：[`bugstack虫洞栈`](https://bugstack.cn/assets/images/qrcode.png)，回复源码下载获取。`你会获得一个下载链接列表，打开后里面的第17个「因为我有好多开源代码」`，记得给个`Star`！

三、案例目标
------

在这里我们定义一个类并创建出等待被监控的方法，当方法执行时监控方法的各项信息；`执行耗时`、`出入参信息`等。

```
public class BizMethod {

    public String queryUserInfo(String uid, String token) throws InterruptedException {
        Thread.sleep(new Random().nextInt(500));
        return "德莱联盟，王牌工程师。小傅哥(公众号：bugstack虫洞栈)，申请出栈！";
    }

}


```

*   我们这里模拟监控并没有使用 `Javaagent` 去做字节码加载时的增强，主要为了将**最核心**的内容体现出来。后续的章节会陆续讲解各个核心功能的组合使用，做出一套监控系统。

四、技术实现
------

在技术实现的过程中，我会陆续的将需要监控的内容一步步完善。这样将一个总体的内容进行拆解后，方便学习和理解。

### 1. 创建监控主体类

```
@Test
public void test_byteBuddy() throws Exception {
    DynamicType.Unloaded<?> dynamicType = new ByteBuddy()
            .subclass(BizMethod.class)
            .method(ElementMatchers.named("queryUserInfo"))
            .intercept(MethodDelegation.to(MonitorDemo.class))
            .make();

    // 加载类
    Class<?> clazz = dynamicType.load(ApiTest.class.getClassLoader())
            .getLoaded();  

    // 反射调用
    clazz.getMethod("queryUserInfo", String.class, String.class).invoke(clazz.newInstance(), "10001", "Adhl9dkl");
}


```

*   这一部分是 `Byte Buddy` 的模版代码，定义需要被加载的类和方法；_BizMethod.class_、_ElementMatchers.named(“queryUserInfo”)_，这一步也就是让程序可以定位到你的被监控内容。
*   接下来就是最重要的一部分**委托**；`MethodDelegation.to(MonitorDemo.class)`，最终所有的监控操作都会被 `MonitorDemo.class` 类中的方法进行处理。
*   最后就是类的加载和反射调用，这部分主要用于每次的测试验证。_查找方法，传递对象和入参信息_

### 2. 监控方法耗时

如上一步所述这里主要需要使用到，委托类进行控制监控信息。

```
public class MonitorDemo {

    @RuntimeType
    public static Object intercept(@SuperCall Callable<?> callable) throws Exception {
        long start = System.currentTimeMillis();
        try {
            return callable.call();
        } finally {
            System.out.println("方法耗时：" + (System.currentTimeMillis() - start) + "ms");
        }
    }

}


```

*   这里面包括几个核心的知识点；`@RuntimeType`：定义运行时的目标方法。`@SuperCall`：用于调用父类版本的方法。
*   定义好方法后，下面有一个 `callable.call();` 这个方法是调用原方法的内容，返回结果。而前后包装的。
*   最后在`finally`中，打印方法的执行耗时。`System.currentTimeMillis() - start`

**测试结果：**

```
方法耗时：419ms

Process finished with exit code 0


```

### 3. 获取方法信息

获取方法信息的过程其实就是在获取方法的描述内容，也就是你编写的方法拆解为各个内容进行输出。那么为了实现这样的功能我们需要使用到新的注解 `@Origin Method method`

```
@RuntimeType
public static Object intercept(@Origin Method method, @SuperCall Callable<?> callable) throws Exception {
    long start = System.currentTimeMillis();
    Object resObj = null;
    try {
        resObj = callable.call();
        return resObj;
    } finally {
        System.out.println("方法名称：" + method.getName());
        System.out.println("入参个数：" + method.getParameterCount());
        System.out.println("入参类型：" + method.getParameterTypes()[0].getTypeName() + "、" + method.getParameterTypes()[1].getTypeName());
        System.out.println("出参类型：" + method.getReturnType().getName());
        System.out.println("出参结果：" + resObj);
        System.out.println("方法耗时：" + (System.currentTimeMillis() - start) + "ms");
    }
}


```

*   `@Origin`，用于拦截原有方法，这样就可以获取到方法中的相关信息。
*   这一部分的信息相对来说比较全，尤其也获取到了参数的个数和类型，这样就可以在后续的处理参数时进行循环输出。

**测试结果：**

```
方法名称：queryUserInfo
入参个数：2
入参类型：java.lang.String、java.lang.String
出参类型：java.lang.String
出参结果：德莱联盟，王牌工程师。小傅哥(公众号：bugstack虫洞栈)，申请出栈！
方法耗时：490ms

Process finished with exit code 0


```

### 4. 获取入参内容

当我们能获取入参的基本描述以后，再者就是获取入参的内容。在一段方法执行的过程中，如果可以在必要的时候拿到当时入参的信息，那么就可以非常方便的进行排查异常快速定位问题。在这里我们会用到新的注解；`@AllArguments` 、`@Argument(0)`，一个用于获取全部参数，一个获取指定的参数。

```
@RuntimeType
public static Object intercept(@Origin Method method, @AllArguments Object[] args, @Argument(0) Object arg0, @SuperCall Callable<?> callable) throws Exception {
    long start = System.currentTimeMillis();
    Object resObj = null;
    try {
        resObj = callable.call();
        return resObj;
    } finally {
        System.out.println("方法名称：" + method.getName());
        System.out.println("入参个数：" + method.getParameterCount());
        System.out.println("入参类型：" + method.getParameterTypes()[0].getTypeName() + "、" + method.getParameterTypes()[1].getTypeName());
        System.out.println("入参内容：" + arg0 + "、" + args[1]);
        System.out.println("出参类型：" + method.getReturnType().getName());
        System.out.println("出参结果：" + resObj);
        System.out.println("方法耗时：" + (System.currentTimeMillis() - start) + "ms");
    }
}


```

*   与上面的代码块相比，多了参数的获取和打印。主要知道这个方法就可以很方便的获取入参的内容。

**测试结果：**

```
方法名称：queryUserInfo
入参个数：2
入参类型：java.lang.String、java.lang.String
入参内容：10001、Adhl9dkl
出参类型：java.lang.String
出参结果：德莱联盟，王牌工程师。小傅哥(公众号：bugstack虫洞栈)，申请出栈！
方法耗时：405ms

Process finished with exit code 0


```

### 5. 其他注解汇总

除了以上为了获取方法的执行信息使用到的注解外，`Byte Buddy` 还提供了很多其他的注解。如下；

<table><thead><tr><th>注解</th><th>说明</th></tr></thead><tbody><tr><td>@Argument</td><td>绑定单个参数</td></tr><tr><td>@AllArguments</td><td>绑定所有参数的数组</td></tr><tr><td>@This</td><td>当前被拦截的、动态生成的那个对象</td></tr><tr><td>@Super</td><td>当前被拦截的、动态生成的那个对象的父类对象</td></tr><tr><td>@Origin</td><td>可以绑定到以下类型的参数：Method 被调用的原始方法 Constructor 被调用的原始构造器 Class 当前动态创建的类 MethodHandle MethodType String 动态类的 toString() 的返回值 int 动态方法的修饰符</td></tr><tr><td>@DefaultCall</td><td>调用默认方法而非 super 的方法</td></tr><tr><td>@SuperCall</td><td>用于调用父类版本的方法</td></tr><tr><td>@Super</td><td>注入父类型对象，可以是接口，从而调用它的任何方法</td></tr><tr><td>@RuntimeType</td><td>可以用在返回值、参数上，提示 ByteBuddy 禁用严格的类型检查</td></tr><tr><td>@Empty</td><td>注入参数的类型的默认值</td></tr><tr><td>@StubValue</td><td>注入一个存根值。对于返回引用、void 的方法，注入 null；对于返回原始类型的方法，注入 0</td></tr><tr><td>@FieldValue</td><td>注入被拦截对象的一个字段的值</td></tr><tr><td>@Morph</td><td>类似于 @SuperCall，但是允许指定调用参数</td></tr></tbody></table>

### 6. 常用核心 API

1.  `ByteBuddy`
    *   流式 API 方式的入口类
    *   提供 Subclassing/Redefining/Rebasing 方式改写字节码
    *   所有的操作依赖 DynamicType.Builder 进行, 创建不可变的对象
2.  `ElementMatchers(ElementMatcher)`
    *   提供一系列的元素匹配的工具类 (named/any/nameEndsWith 等等)
    *   ElementMatcher(提供对类型、方法、字段、注解进行 matches 的方式, 类似于 Predicate)
    *   Junction 对多个 ElementMatcher 进行了 and/or 操作
3.  `DynamicType`(动态类型, 所有字节码操作的开始, 非常值得关注)
    *   Unloaded(动态创建的字节码还未加载进入到虚拟机, 需要类加载器进行加载)
    *   Loaded(已加载到 jvm 中后, 解析出 Class 表示)
    *   Default(DynamicType 的默认实现, 完成相关实际操作)
4.  `Implementation`(用于提供动态方法的实现)
    *   FixedValue(方法调用返回固定值)
    *   MethodDelegation(方法调用委托, 支持两种方式: Class 的 static 方法调用、object 的 instance method 方法调用)
5.  `Builder`(用于创建 DynamicType, 相关接口以及实现后续待详解)
    *   MethodDefinition
    *   FieldDefinition
    *   AbstractBase

五、总结
----

*   在这一章节的实现过程来看，只要知道相关 API 就可以很方便的解决我们的监控方法信息的诉求，他所处理的方式非常易于使用。而在本章节中也要学会几个关键知识点；委托、方法注解、返回值注解以及入参注解。
*   当我们学会了监控的核心功能，在后续与`Javaagent`结合使用时就可以很容易扩展进去，而不是看到了陌生的代码。对于这一部分非入侵的入侵链路监控，也是目前比较热门的话题和需要探索的解决方案，就像最近阿里云也举办了类似的编程挑战赛。[首届云原生编程挑战赛 1：实现一个分布式统计和过滤的链路追踪](https://tianchi.aliyun.com/competition/entrance/231790/introduction)
*   关于字节码编程专栏已经完成了大部分文章的编写，包括如下文章；(**学习链接**：[`https://bugstack.cn/itstack/itstack-demo-bytecode.html`](https://bugstack.cn/itstack/itstack-demo-bytecode.html))
    
    *   [`字节码编程，Byte-buddy篇一《基于Byte Buddy语法创建的第一个HelloWorld》`](https://bugstack.cn/itstack/itstack-demo-bytecode.html)
    *   [`字节码编程，Javassist篇五《使用Bytecode指令码生成含有自定义注解的类和方法》`](https://bugstack.cn/itstack/itstack-demo-bytecode.html)
    *   [`字节码编程，Javassist篇四《通过字节码插桩监控方法采集运行时入参出参和异常信息》`](https://bugstack.cn/itstack/itstack-demo-bytecode.html)
    *   [`字节码编程，Javassist篇三《使用Javassist在运行时重新加载类「替换原方法输出不一样的结果」》`](https://bugstack.cn/itstack/itstack-demo-bytecode.html)
    *   [`字节码编程，Javassist篇二《定义属性以及创建方法时多种入参和出参类型的使用》`](https://bugstack.cn/itstack/itstack-demo-bytecode.html)
    *   [`字节码编程，Javassist篇一《基于javassist的第一个案例helloworld》`](https://bugstack.cn/itstack/itstack-demo-bytecode.html)
    *   [`ASM字节码编程 | 用字节码增强技术给所有方法加上TryCatch捕获异常并输出`](https://bugstack.cn/itstack/itstack-demo-bytecode.html)
    *   [`ASM字节码编程 | JavaAgent+ASM字节码插桩采集方法名称以及入参和出参结果并记录方法耗时`](https://bugstack.cn/itstack/itstack-demo-bytecode.html)
    *   [`ASM字节码编程 | 如果你只写CRUD，那这种技术你永远碰不到`](https://bugstack.cn/itstack/itstack-demo-bytecode.html)
*   **最佳的学习体验和方式**是，在学习和探索的过程中不断的对知识进行深度的学习，通过一个个实践的方式让知识成结构化和体系的建设。
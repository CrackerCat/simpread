> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-273838.htm)

> [原创]（随笔）有风控 & 无风控 App 对抗深入浅出

[](#前言：)前言：
-----------

随着移动互联网的发展，App 的防护也逐渐升级，从最早的 App 端上防护慢慢的到服务端做策略进行对抗，中间衍生很多 App 防护的方式，比如加固，唯一 ID 服务，数据分析服务等。大厂有能力可以搭建自己的风控策略平台，小厂就只能依赖三方机构，或者 App 端上的检测去实现爬虫的对抗和分析。

 

18 年之前的爬虫成本很低，当时的 App 防护也很少，很多 App 加个壳就觉得高枕无忧，攻击者进行脱壳，加密算法还原以后可以进行高效率点对点之间的数据请求。但是放眼现在的话，这种方式可能比较落后，只能针对一些没有防护的小公司。

 

**这篇文章主要站在防护方和攻击者的角度去思考，讲一下有风控 & 无风控对抗的方式和防护的策略。**

[](#风控概述：)风控概述：
---------------

风控的全称应该是风险控制，为了解决和预防将要发生，或者可能发生的一些危险情况，从而减轻损失。

 

经过不断地衍化，App 风控的方式也有很多种，比如人工进行风控，肉眼过审。到现在很多大厂可以根据一些特定的数据模型

 

从而实现的 AI 自动过审。但是不管如何，都需要经过一些信息的输入，才能得到输出结果。而输入的过程就是我们要讲的内容。

### [](#蜜罐数据：)蜜罐数据：

什么是蜜罐数据？当发现作弊以后返回的数据是非正常的数据，可能存在埋点等信息，比如视频里面在随机帧率里面添加水印，或者返回一些错误数据或者重复的数据，这些数据往往是已经被污染或者肉眼无法识别是否正确，从而欺骗攻击者。

### [](#ip限制：)IP 限制：

这个不多说，当某一个 IP 过量或者过快请求的时候会进行限制，返回错误的数据或者返回蜜罐数据。

> 很多爬虫会买入很多代理，这些代理也都是不安全的，很多大厂也会买入一部分代理，代理毕竟是谁都可以用的，买完以后在服务端直接配置上黑名单即可，很多黑产或者攻击者会采用流量的方式进行请求，将数据转发到路由器，一个类似 “猫池” 的路由器，里面内置很多手机卡，可以通过设置，将数据通过内部随机手机卡进行发送请求，从而实现动态代理。从而规避一些 IP 限制的检测。

### [](#设备指纹：)设备指纹：

设备指纹主要为了解决就是设备的唯一性，防护方通过采集手机的某些字段，从而实现得到设备唯一的标识。谷歌其实一直在规避这种方式，主要就是为了防止大数据杀熟，不过还是有很多种办法可以进行设备唯一性的采集。详细的获取可以参考我之前的文章。

 

https://bbs.pediy.com/thread-273759.htm ，很多策略会想尽办法去尽可能完善设备指纹的唯一性。以此确保策略的稳定性。

 

客户端的采集准确程度，有时候也大幅度的决定了后端策略的风控方向，有时候也不能乱加策略，因为有可能就是误杀一片。

 

导致很多 App 客户进行投诉。设备指纹唯一 ID，相当于用户的 token ，设备指纹的准确性和唯一性，决定安全 SDK 的强度。不过国内貌似安全 SDK 强度还是围绕在几年之前的样子。

 

这个字段有时候也不能作为风控策略的关键因子，更多的还是其他因素，比如用户画像，用户行为，等多维度进行判断。不过设备指纹也是风控里面一个很关键的因素。**其检测方式也只是为了提升攻击者绕过的成本。**

 

不过现在的大厂指纹都有一些问题，就是手机重置以后或者恢复出厂设置以后，大部分大厂的设备指纹都无法做到设备唯一性的确认。

### [](#app环境信息：)App 环境信息：

和设备指纹一样重要，当策略或者防护发现某个 token 或者某个设备唯一标识出现问题的时候，可能第一时间就是去检测这个设备的环境信息，用于是否石锤当前用户是否作弊，因此，很多策略也会将当前用户分为很多种等级，此篇文章分为如下三个设备，用于描述后续下文。

*   正常设备 （字面含义，正常用户使用的设备，服务端信任此设备）
*   灰产设备 （可能存在作弊可能的设备，当手机环境存在 root 等，hook 框架等）
*   黑产设备 （已经被石锤作弊的设备）

正常我们肯定希望，尽可能准确的将设备进行分类，从而规避风险，主要靠的就是设备指纹和环境检测能力。比如常见的大厂石锤当前用户是否作弊方法，动态上报当前手机日志，上报 Xposed hook 方法 item 列表 ，等... ... 技术手段，都可以以此石锤当前用户是否作弊。

 

当然作弊的方式也有很多种，对于不同的业务区别也不一样，比如如果发现 Hook 的方法里面存在 location 相关，即可得知，攻击的标签认定为修改定位等。

 

常用的环境检测主要包含以下几部分

#### Root 检测

检测手机的 root 环境，是否越狱等，当然现在检测的纬度也不只是去检测 su 文件那么简单，比如通过技术手段去扫描 magisk 等关键字，

 

maps 里面，mount 里面，通过检测 magisk 端口等都可以石锤当前用户是否存在 root。包括比如 riru ,edxp ,lsp 等 moudle，特征文件，都可以认定为当前手机存在 Root，客户端只需要做的是尽可能的收集准确，至于具体确认与否，取决于策略，而策略又取决于这个字段是否可行，可信度达到多少以上可用 。

#### isHook 检测

检测主流 hook 框架，比如在 android 上面最主要的两个就是 xposed 和 frida，这两个占 android 市面上 hook 框架占比 99% 以上，检测的方式也一堆，当然这块需要注意，检测到是否存在，和是否被 hook 是两码事。 如果想检测当前环境是否被 Hook 的话需要具体内存扫描。

##### Xposed:

比如 Xposed 的话就是把攻击者 hook 的方法 item 进行上报，判断是否存在当前业务方的相关的类和方法，以此确认攻击者动机和目的。

 

具体上报的方法在指纹那篇帖子里里面有整理。这里就不多说。

##### Frida:

Frida 的话就更简单了，函数执行之前判断当前函数是否被 inlinehook ，有一种比较好的办法，就是先扫描本地文件，比如 libc 这个 so 库。

 

我们先通过解析本地 elf 文件的方式，libc.so 文件得到函数符号，得到这个方法的相对偏移，从而得到这个方法的真实代码指令。

 

然后在解析内存判断和本地文件的指令是否相同即可确认当前方法是否被修改，而不需要做 32 位和 64 位判断。

> 以前的老方法基本都是检测函数头 32 位检测 LDR 64 位检测 BLX 等指令，判断当前方法是否被 Hook。
> 
> 这种老方法的检测方式都有弊端，比如异常 Hook 或者断点 Hook BKPT 等断点指令 hook。都没办法做到检测。
> 
> 直接对比指令是否发生变化是最直接也是最有效的办法

#### 沙箱 & 虚拟环境（关键）

沙箱的话检测方式还是很多的，列出来常用几种方式参考如下

 

1，刚启动的时候判断当前的进程，是否存在其他进程，直接 popen ps 一下即可。

 

2，或者检测当前 Apk 的私有路径是否正确，是否是 / data/data / 包名结构，

 

3，readdir 打开当前用户所有的 dir 目录，判断是否存在多余的文件描述符 fd。是否存在和父进程不同的 pid。

 

4，检测当前 IPC 等类是否被动态代理，很多 VA 等虚拟沙盒原理就是动态代理 IPC 相关，直接获取到 IPC 类实体类以后. getClass().getName()

 

判断是否包含 proxy 等关键字即可。

 

5，.....

 

**注意：此字段一般在环境占比中很重要，当当前 App 环境一旦被石锤沙箱环境以后，直接可认定当前用户为黑设备**

 

####

#### [](#apk签名（关键）)Apk 签名（关键）

APK 签名这个具体也在设备指纹那篇文章有详细介绍，市面上大厂和加固也都用这两种方法去检测。这里就不细说了。

 

一旦检测出来当前 APK 签名不正确的话，可以被认定为重打包，可以直接石锤用户作弊。

#### [](#模拟器（关键）)模拟器（关键）

常用的模拟器也有很多种，不同类型的模拟器检测也不一样，可以针对不同类型的模拟器特征文件去检测，百度一下一堆。

 

这个字字段也是关键字段，正常我们是不希望我们的 App 在模拟器里面进行执行的。所以当发现设备在模拟器内部执行以后可以直接认定为风险设备。

#### 自定义 ROM

现在能自定义的很少，基本都是谷歌系列的手机，因为很多大厂他不会开源当前手机的驱动代码，没有驱动代码也就没办法编译成 rom。

 

很多大厂会扫描系统的一些文件 MD5，主要用于判断 google 系列当前手机是否存在自定义 Rom 。

### [](#查杀分离：)查杀分离：

这个也是一种很重要的策略，主要就是当服务端或者命中风控以后，他不会及时封你的号，或者立刻给你号返回错误数据。

 

而是隔一段时间，可能是几个小时，也有可能是几天，这样做的好处防止你去不断地试探从而找到正确的检测规律。

 

防止攻击者不断试探的方式去获取正确的风控规则。规避风险。

### 用户行为 & 心跳包上报：

#### [](#检测原理：)检测原理：

一般大厂会使用这种方案，在一些 SDK 初始化以后会开启一个 socket，tcp 长连接，覆盖 App 整个生命周期，当用户进行点击的时候

 

对页面某个位置点击的时候会进行上报，后台可以很清楚的看到当前用户的点击路径。如果攻击者直接通过 rpc 或者算法还原接口破解的方式调用接口的话，就可能会存在心跳包遗漏的问题，当长时间无心跳以后，可能会直接认为当前 IP 是风险 IP，从而实现封禁。

 

这个方法也是很好用的办法，可以通过 AI 等进行自动化的行为判断，等 AI 的模型和数据足够完善以后，即可实现，自动化判断自动化 & 非自动化（人手点击）的判断。

 

比如某些的自动点击框架，如果只是为了业务去点击的话，是没有一些多余操作的，而我们们的手在屏幕不断滑动的时候是会产生很多用户路径，我们称之为随机路径。

#### [](#对抗原理：)对抗原理：

自动化点击脚本控制 App + 用户随机路径，自动化脚本控制 App 去实现自动化点击，防止心跳包和用户点击路径的遗漏。

 

需要添加随机路径，防止被 AI 检测出来自动化操作，添加随机路径也很简单，在不影响点击结果的情况下，仿人触摸随机对屏幕滑动。

 

（可以提前录制一些用户的操作流程，将数据保存到 Json 里面，在对屏幕进行 dispatch 事件分发的时候，采用真人点击的 event 即可）

> 细节点：如何记录用户的点击行为？
> 
> 手机屏幕好像是一个分发器，而屏幕的 view 是消费者。他可能选择消费这个事件，也可以选择抛出去，给下级 view 去消费。
> 
> 但是事件只要被消费了就一定会走 view->ontouch(); 方法，所以我们只需要 hook view 的 view->ontouch(); 方法，把参数 1 进行 toString 打印和保存即可。即可得到全部的的点击事件消费对象 objection。

### 异常 & 行为埋点：

#### [](#检测原理：)检测原理：

指的是在某个页面进行埋点，只有触发某条请求或者打开某个页面的时候才会进行埋点上报。

> 举个 case:
> 
> 当攻击者调用登入接口获取 token 的时候，正常肯定需要打开 App 的登入页面，而这个埋点是通过打开页面时候进行上报。
> 
> 如果攻击者只进行了调用登入接口，没有调用埋点接口，可能会导致当前请求缺少前置埋点。可能会导致数据被风控。

 

埋点上报其实在风控里面发挥的作用还是很大的，正常用户从登入到查看个人信息，需要触发 5 个埋点信息。

 

但是攻击者只触发了 1-2 个埋点，则可认定当前用户存在作弊行为，可能存在脱机的嫌疑。会被直接标识成黑设备。

 

在后台看的话就是一些点点，而这些点就是不同的埋点信息，哪个点被点亮，哪个点没有被点亮，和正常的用户做一下对比，很容易就可以确认。

#### [](#对抗原理：)对抗原理：

同上

### [](#总结：)总结：

说了这么多总结一下，用上述的方法可以有效对抗，RPC，或者常规的自动点击，包括一些大批量的数据获取。

 

当账号数量足够多的时候（账号足够成熟，很多新号会有限制），并且满足一下条件的时候：

 

**群控 + 自动点击 + 用户点击随机路径 + 完善的改机软件 + 一机多号（设备够多无视），即可实现风控的对抗。**

 

当然还需要分析，一些账号的临界值，不同的数据可能触发的风控点也不一样。

 

比如有的数据单用户日获取量不超过 20 条，那么你你就不可能获取超过 20 次。群控服务端还需要记录每个用户的日点击数，等信息。

 

**<u> 当然有风控肯定不是无敌的，他能做的就是提高你得逆向成本，小规模的抓取基本无视。当发现数据过大的时候及时发现并上报，这就已经足够了。而这些成本最小的就是客户端 SDK 检测能力的提升，客户端的数据上传的越准确，往往对灰黑产的识别效果也越好。不过现在国内基本检测都差不多。有创新的东西很少了。</u>**

无风控 App:
--------

无风控 App 对抗起来还是很简单的，因为服务端没有限制，所以遇到这种我一般都是不会去分析他的实现的。

 

我们只关注我们需要的数据即可。找到可以触发这次请求的地方，直接控制目标 App 无限触发该行为，比如点击或者刷新等操作。找到数据保存的地方进行上报即可，RPC 等也是很好的办法。因为对抗这种无风控的 App RPC 是效率最高的方法之一。

> 举个 case:
> 
> 猿人学比赛 App
> 
> 他这个就是很典型的无风控 App 例子，因为加密过程很复杂，但是服务端没有请求限制，我们先找到保存这个数据的实体类，直接通过内存漫游的方式（根据一个 Class 获取这个 Class 在内存里面的实体类），找到内存里保存的实体类。五秒刷新一次内存。
> 
> 然后无限触发这个刷新功能，得到实体类对象，每隔 5 秒进行一次内存漫游，把全部的对象信息打印并且累加即可。
> 
> 用这种方式可以 20 行代码即可快速 Pass 掉 1-10 题。

[[注意] 传递专业知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#NDK 分析](forum-161-1-119.htm) [#协议分析](forum-161-1-120.htm) [#混淆加固](forum-161-1-121.htm) [#脱壳反混淆](forum-161-1-122.htm) [#漏洞相关](forum-161-1-123.htm) [#程序开发](forum-161-1-124.htm) [#HOOK 注入](forum-161-1-125.htm) [#系统相关](forum-161-1-126.htm) [#源码框架](forum-161-1-127.htm) [#工具脚本](forum-161-1-128.htm) [#其他](forum-161-1-129.htm)
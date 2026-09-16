> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-292966.htm#msg_header_h2_6)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

由于业务需求，最近需要熟悉一些端技术，把日常的一些分析学习用文章记录一下。

对普通用户来说，手机是用来聊天、购物和支付的终端；对黑产来说，一部手机首先是一台可以被批量控制、重复执行任务并伪装运行环境的计算设备。

正常 Android 会把不同 App 隔离开：每个应用有自己的身份和数据目录，不能随意读取其他应用、修改系统文件或干预别的进程。这样的边界保护用户，也妨碍黑产把手机改造成自动化工具。Root 的价值，就在于突破其中一部分边界，为后续工具提供更强的设备控制基础。

黑产的目标通常不是 “看到 UID 0” 这个技术结果，而是完成某种业务动作：批量养号、自动操作、薅取活动权益、干预广告或游戏逻辑、修改客户端看到的环境，或者长期控制一批设备。

在没有 Root 的手机上，普通 App 被限制在自己的沙箱中。它可以使用系统公开的接口，却很难稳定完成下面这些事情：

<table><thead><tr><th>黑产想获得的能力</th><th>Root 提供的基础</th><th>可能支撑的滥用场景</th></tr></thead><tbody><tr><td>读取或修改受保护的数据</td><td>以更高身份访问文件和进程资源</td><td>批量维护账号环境、改写本地状态</td></tr><tr><td>控制应用和系统进程</td><td>创建特权命令进程、调整进程运行条件</td><td>自动拉起、停止、清理或编排 App</td></tr><tr><td>改变应用看到的文件与属性</td><td>覆盖挂载、属性修改、命名空间调整</td><td>构造不同的设备或系统视图</td></tr><tr><td>在应用进程中运行额外代码</td><td>借助进程注入或扩展框架加载模块</td><td>观察或改变客户端内部逻辑</td></tr><tr><td>扩展内核行为</td><td>加载内核模块或植入内核扩展</td><td>在更底层处理系统调用、权限和访问控制</td></tr></tbody></table>

需要强调的是，**Root 并不会自动完成这些行为。** 它更像取得了总机房的通行证：真正的自动化、注入、数据处理和环境改造，还需要脚本、模块、Hook 框架、设备控制程序等工具配合。TEE、安全芯片、服务端校验和其他硬件边界，也不会因为 Android 取得 Root 就当然失效。

从一项黑产业务目标到设备动作，中间通常是这样一条链：

```
业务目标
  → 需要批量、稳定地控制设备
  → Root 建立高权限执行基础
  → 模块／脚本／注入框架改造运行环境
  → 自动化程序反复完成目标动作
```

这解释了黑产为什么追求 Root，也解释了为什么 “手机已 Root” 和“某个 App 正在作恶”不能直接画等号。Root 是能力底座；谁获得授权、具体执行什么命令、加载了什么扩展，才决定这项能力如何被使用。

打开 Root 管理器，给某个 App 点一下 “允许”，看起来只是打开一个开关。手机内部却需要先完成一套完整改造：Root 工具进入启动链，在 Android 安全环境定型前安放组件；手机开机后，它识别请求来自哪个 App，为获准进程准备身份、能力和安全域，最后再把命令交给用户态 shell 执行。  
https://bbs.kanxue.com/forum-45.htm  
Magisk、KernelSU 和 APatch 都完成了这件事，却把控制点放在不同位置：Magisk 主要改造启动早期和用户态环境；KernelSU 把请求识别与权限准备放进内核；APatch 借助 KernelPatch 定位并修补现成内核，在其中建立可持续扩展的处理能力。

要看懂这些改造，先要认识一部没有 Root 的 Android 手机原本怎样启动和运行 App。

下面这张图把 Android 画成一座手机形状的工厂。上层是用户态，运行系统服务和 App；下层是内核态，负责进程、内存、文件、驱动和底层安全检查。图中的上下楼层是软件分层，不是两块真实硬件。

![图片描述] ![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

_图 1：橙色路径表示开机，蓝色路径表示一次典型的 App 冷启动。_

**手机先完成一次从下到上的启动。**

按下电源后，芯片中的早期启动代码先运行，再逐级进入 Bootloader。Bootloader 检查设备状态，按照启动验证流程加载内核、和早期启动资源，然后把控制权交给 Linux 内核。[](https://bbs.kanxue.com/elink@957K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6K6L8%4g2J5j5$3g2Q4x3X3g2S2L8X3c8J5L8$3W2V1i4K6u0W2j5$3!0E0i4K6u0r3k6r3!0U0M7#2)9J5c8X3y4G2M7X3g2Q4x3V1k6S2M7X3y4Z5K9i4c8W2j5%4c8#2M7X3g2Q4x3V1k6T1L8$3!0@1L8r3!0S2k6r3g2J5)Android Bootloader 说明

内核建立内存管理、进程调度和驱动等基础环境，随后启动第一个用户态进程 `init`。init 的 PID 是 1，它读取 rc 配置，组织挂载、属性、安全环境和服务启动。rc 是开工安排，init 是执行安排的程序，两者并不是一回事。[](https://bbs.kanxue.com/elink@b40K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6S2L8X3c8J5L8$3W2V1i4K6u0W2k6$3!0G2k6$3I4W2M7$3!0#2M7X3y4W2i4K6u0W2j5$3!0E0i4K6u0r3M7r3I4S2N6r3k6G2M7X3#2Q4x3V1k6K6P5i4y4@1k6h3#2Q4x3V1k6U0L8%4u0W2i4K6u0r3i4K6u0n7i4K6u0r3M7X3g2X3M7#2)9J5c8X3S2W2j5h3c8K6i4K6u0r3L8h3q4A6L8W2)9J5c8X3W2F1K9i4c8Q4x3V1k6d9c8f1q4p5e0f1g2Q4x3X3g2E0k6l9%60.%60.)Android init 说明

init 随后启动 Zygote。Zygote 预先加载应用常用的运行环境，需要新 App 进程时，再从已经准备好的环境派生。它还会创建 `system_server`，应用管理、包管理和窗口管理等大量 Android 系统服务运行在这个进程中。系统服务就绪后，桌面应用 Launcher 显示出来，我们才看到图标。[](https://bbs.kanxue.com/elink@003K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6S2L8X3c8J5L8$3W2V1i4K6u0W2k6$3!0G2k6$3I4W2M7$3!0#2M7X3y4W2i4K6u0W2j5$3!0E0i4K6u0r3M7r3I4S2N6r3k6G2M7X3#2Q4x3V1k6X3M7X3q4E0k6i4N6G2M7X3E0K6i4K6u0r3j5X3q4K6k6g2)9J5c8W2)9J5b7W2)9J5c8X3#2S2M7%4c8W2M7W2)9J5c8X3y4G2M7X3g2Q4x3V1k6B7j5i4k6S2i4K6u0r3j5$3!0E0i4K6u0r3j5h3&6V1M7X3!0A6k6q4)9J5c8X3W2F1N6r3g2J5L8X3q4D9i4K6u0r3L8%4y4Q4x3V1k6K9P5h3N6G2N6r3g2u0L8X3W2@1i4K6u0W2K9X3q4$3j5b7%60.%60.)Zygote 初始化源码

```
上电
  → Bootloader 加载内核和启动资源
  → Linux 内核建立底层环境
  → init 组织用户态服务
  → Zygote 与 system_server 就绪
  → Launcher 显示桌面
```

**点击图标，又会发生一次从系统服务到应用的接力。**

假设目标 App 还没有运行，Launcher 先向系统服务提交启动请求。`system_server` 中的相关服务解析目标、检查条件并安排任务；需要新进程时，再让 Zygote 体系准备应用进程。新进程获得属于该 App 的 UID、安全域等运行条件，加载代码和资源，初始化 Application 与 Activity，最后绘制首帧。[](https://bbs.kanxue.com/elink@e34K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6K6L8%4g2J5j5$3g2Q4x3X3g2S2L8X3c8J5L8$3W2V1i4K6u0W2j5$3!0E0i4K6u0r3k6r3!0U0M7#2)9J5c8X3y4G2M7X3g2Q4x3V1k6J5N6h3&6@1K9h3#2W2i4K6u0r3P5Y4W2Y4L8%4c8W2)Zygote 进程说明、[](https://bbs.kanxue.com/elink@23bK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6V1k6i4k6W2L8r3!0H3k6i4u0Q4x3X3g2S2L8X3c8J5L8$3W2V1i4K6u0W2j5$3!0E0i4K6u0r3N6r3!0H3K9h3y4Q4x3V1k6H3k6i4u0X3L8%4u0E0j5h3&6U0k6g2)9J5c8Y4k6A6N6r3q4D9M7#2)9J5c8X3I4S2N6h3&6U0K9q4)9J5k6s2c8A6L8h3f1%60.)Android 应用启动说明

```
点击图标
  → Launcher 提交请求
  → system_server 安排启动
  → Zygote 体系准备应用进程
  → App 初始化并显示界面
```

Launcher 不负责亲自创建目标进程，system_server 也不是每个 App 的父进程。真实设备还可能复用已有进程或使用预创建进程池。这里保留的是理解 Root 最需要的职责关系。

init、Zygote、system_server、Launcher 和 App 全部位于用户态。系统服务拥有较高权限，也仍然是用户态进程；root shell 同样如此。**Root 描述的是进程拥有的身份与能力，用户态和内核态描述的是代码执行位置。** 这是理解后面三种方案的第一个关键区别。

普通 App 不能随意读取其他应用数据、修改系统文件或控制设备。它访问文件、创建进程、操作设备时，需要通过系统调用让内核代为处理，内核再根据调用者的身份和安全规则决定是否允许。

这些规则不是一个开关，而是几套机制共同作用：

<table><thead><tr><th>机制</th><th>通俗理解</th><th>实际作用</th></tr></thead><tbody><tr><td>UID / GID</td><td>工牌与所属工作组</td><td>决定进程以谁的身份访问资源</td></tr><tr><td>capabilities</td><td>被拆开的特殊操作资格</td><td>决定能否执行某些传统特权操作</td></tr><tr><td>SELinux</td><td>车间准入规则</td><td>按进程安全域、对象类型和策略检查访问</td></tr><tr><td>seccomp</td><td>可使用工具的清单</td><td>限制进程能够调用哪些系统调用</td></tr><tr><td>挂载命名空间</td><td>每个工作间看到的物料架</td><td>决定进程能看到哪些挂载和文件视图</td></tr></tbody></table>

因此，取得 UID 0 不代表其他限制自动消失。一个完整 Root 方案至少要做好四件事：

```
把能力装进手机
  → 开机时让能力重新就位
  → 识别并授权某个 App 的请求
  → 为具体进程准备权限并执行命令
```

这四步横跨两个时间阶段。

**安装阶段**决定 Root 组件藏在哪、怎样随手机启动。它可能修改 ramdisk、加载 `.ko` 内核模块，也可能直接修补内核镜像。

**运行阶段**才处理一次具体请求。App 通常创建 su 客户端进程，请求执行 root shell 或一条命令。Root 工具检查调用者，准备目标身份、能力、安全域和执行环境，再进入用户态命令后端。获得权限的是执行链中的相应进程，原 App 的全部线程和其他进程不会一起自动变成 root。

三款工具的真正差别，可以先压缩成一张地图：

<table><thead><tr><th></th><th>能力主要放在哪里</th><th>谁处理授权和权限</th><th>谁执行最后的命令</th></tr></thead><tbody><tr><td>Magisk</td><td>启动早期建立的用户态环境</td><td>用户态特权服务 <code>magiskd</code></td><td>magiskd 派生的用户态进程</td></tr><tr><td>KernelSU</td><td>内建内核代码或 <code>kernelsu.ko</code></td><td>内核中的 KernelSU 逻辑</td><td>用户态 <code>ksud</code> 后端</td></tr><tr><td>APatch</td><td>KernelPatch 的 kpimg 或 LKM</td><td>内核中的 KernelPatch 逻辑</td><td>用户态 <code>apd</code> 后端</td></tr></tbody></table>

如果把手机继续想象成一栋大楼，三种方案就像三种不同的改造队伍：

<table><thead><tr><th>方案</th><th>通俗角色</th><th>它把 “自己人” 放在哪里</th><th>一次 Root 请求怎样完成</th></tr></thead><tbody><tr><td>Magisk</td><td>金牌大管家</td><td>抢先进入开机大堂，在用户态建立特权控制室</td><td>App 找到管家，管家审核后派出一个有权限的执行进程</td></tr><tr><td>KernelSU</td><td>物业总公司的编制内员工</td><td>把权限判断代码放进内核</td><td>App 的请求先由内核验身份、配权限，再交给 ksud 执行</td></tr><tr><td>APatch</td><td>给成品内核做手术的外科医生</td><td>在已经编译好的内核里植入 KernelPatch 能力</td><td>内核中的补丁逻辑验身份、改权限，再交给 apd 执行</td></tr></tbody></table>

这三个角色不是三句宣传语，而是后面技术流程的索引。大管家对应 `magiskinit + magiskd`；编制内员工对应 KernelSU 内核组件；外科手术对应 kallsyms 定位、kpimg 植入与 inline hook。记住角色，再去看函数、镜像和系统调用，会容易很多。

接下来沿着同一条链分别看三种改造方式。

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

_图 2：机器人手机工厂展示 Magisk 改造的主要位置：启动镜像、SELinux、magiskd、文件视图和应用进程。_

这张图先回答 “Magisk 动了哪里”。下面的技术流程图再把这些改动按安装、开机和运行三个阶段连接起来。

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)  
_图 3：Magisk 完整改造流程。上方是镜像修补，中间是 magiskinit 与原生 init 的交接，下方是一次 su 请求。_

先把图读成一个简单故事：大楼开门前，Magisk 让 `magiskinit` 先到前台，布置好通道和规则后，再把位置交还给原生 init。正式营业后，`magiskd` 留在楼里当大管家。某个 App 需要 Root 时，它不会自己冲进内核改身份，而是向管家申请；管家确认授权，再派出一个本来就拥有高权限的 “分身” 替它执行命令。

```
换前台：修补启动镜像，让 magiskinit 抢先运行
  → 做布置：准备挂载、属性和 SELinux 环境
  → 请管家：启动 magiskd 并接入开机阶段
  → 发钥匙：审核 su 请求，派生特权执行进程
  → 换布景：按需加载模块、Magic Mount 与 Zygisk
```

以常见的 ramdisk 路线为例，安装时 `magiskboot` 解析启动镜像，检查是否已经修补，备份原始内容，部署 `magiskinit` 等载荷，再把镜像重新打包。ramdisk 是开机早期使用的一组文件，它可能位于 boot、init_boot 或其他与设备布局相关的位置，不能把所有手机都概括成修改同一个文件。

设备重启后，magiskinit 比大量 Android 服务更早参与启动。它要识别当前设备使用的 rootfs、SAR 和多阶段 init 布局，在挂载关系和安全环境定型前放入自己的内容。

SELinux 策略也是早期准备的一部分。Magisk 的用户态工具修改策略，再交给内核加载执行。补充规则是为了让 Magisk 组件取得所需访问，并不等于关闭整个 SELinux。准备完成后，magiskinit 恢复并执行真正的 init，Android 原有启动链继续向前。

```
安装：解析启动镜像 → 备份 → 部署 magiskinit → 重新打包

开机：magiskinit 进入早期启动
        → 准备挂载与 SELinux 环境
        → 交还原生 init
        → 接上 post-fs-data、service、boot-complete 等阶段
```

后续启动阶段建立 `magiskd` 和模块环境。magiskd 能成为 “已经拥有特权的控制室”，根源就在这条启动链，而不是普通 App 发来一次请求后凭空获得权限。

Magisk 的主要 Root 控制和命令执行逻辑位于用户态，但这不等于安装时绝不修改内核字节。材料所分析版本在部分设备的兼容处理里包含内核镜像 hexpatch。准确说法是：**Magisk 不把主要 Root 逻辑做成常驻内核组件。**

普通 App 需要 Root 时，会创建 su 客户端进程。客户端通过本地 socket 联系 magiskd；magiskd 识别请求 UID，查询授权记录，必要时让管理器显示允许或拒绝。本文分析基线的默认发布构建还启用了客户端可执行文件校验，不能理解成任何进程只要连上 socket 就能发号施令。

获准后，magiskd 派生一个执行进程，为它设置目标身份、环境变量、文件描述符或伪终端、挂载命名空间等条件，然后执行 shell 或指定命令，把输出和退出状态返回客户端。

```
App
  → su 客户端
  → socket 请求 magiskd
  → 识别调用者并检查授权
  → magiskd 派生特权子进程
  → 准备身份、终端、环境和文件视图
  → 执行命令
```

这是一种**特权服务代理执行**模型。原 App 不会整体变成 root，magiskd 与它派出的 shell 也仍在用户态。它们只是以更高身份调用内核原有的进程、凭据和挂载机制。

Magisk 模块可以包含文件、启动脚本、系统属性和 SELinux 规则。Magic Mount 负责让模块文件出现在系统路径中：普通文件替换主要依靠 bind mount；需要新增、删除或重组目录项时，会借助 tmpfs 构造目录视图，再把相应内容挂载进去。

这个视图通常在启动阶段组织好，并不是 App 每次访问路径时才临时替换，也不是重写 Linux 的 VFS。所谓 systemless，强调目标系统分区里的原文件可以保留，不表示手机上完全没有写入：启动镜像、模块和配置依然需要存储。

Zygisk 则把扩展点放到应用进程形成的阶段。它借助 Native Bridge 等用户态路径进入 Zygote，在应用进程 specialize 前后让模块代码参与运行。**代码进入 App 进程与给 App 授予 Root 是两件事**，一次普通 su 请求也不依赖 Zygisk。

DenyList 负责按配置处理指定进程看到的 Magisk tmpfs 和模块挂载。它与 Zygisk 可以配合，却不是同一功能。在本文分析基线中，Zygisk 与 DenyList 都默认关闭，需要分别开启和配置。

Magisk 因此形成了三条相互配合、又彼此独立的能力线：

<table><thead><tr><th>能力线</th><th>解决的问题</th></tr></thead><tbody><tr><td>su + magiskd</td><td>谁可以执行特权命令</td></tr><tr><td>Magic Mount + 模块阶段</td><td>系统文件视图和启动行为怎样扩展</td></tr><tr><td>Zygisk</td><td>模块代码怎样进入 Zygote 与 App 进程</td></tr></tbody></table>

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

_图 4：机器人手机工厂展示 KernelSU 对内核代码、系统调用、进程凭据、SELinux 和 init 启动配置的改造。_

位置图说明 KernelSU 把哪些工作放入内核；下面的技术流程图继续说明代码怎样进入内核、管理器怎样配置授权，以及虚拟 su 怎样转入 ksud。

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

_图 5：KernelSU 完整改造流程。内建与 LKM 是运行形态，三条路线是 LKM 的不同投递方式。_

KernelSU 的故事发生在 “物业总公司” 里。第一步不是先安排一位用户态管家，而是让 KernelSU 代码成为内核的一部分：可以随内核编译进去，也可以把 `kernelsu.ko` 运进去。之后 App 再找 `su`，内核会先认人，再按照 `root_profile` 重写这次执行需要的身份和能力，最后把真正的命令交回用户态 `ksud`。内核负责发证，用户态仍负责办事。

```
进总公司：内建 KernelSU，或在启动／运行期加载 kernelsu.ko
  → 登记规则：管理器写入允许列表和 root_profile
  → 总机转接：内核识别 su 探测与执行请求
  → 重做工牌：准备 UID、GID、能力、安全域和命名空间
  → 回到用户态：ksud 执行 shell、脚本和模块任务
```

KernelSU 有两种运行形态。它可以与内核一起编译，成为内建代码；也可以做成 `kernelsu.ko`，在合适时机加载进正在运行的内核。LKM 方式不要求每位用户都取得厂商源码重新编译，但仍受内核版本、KMI 和模块加载条件限制，并不是一个 `.ko` 文件适配所有手机。内建形态仍然存在，也不能说已经被 LKM 完全取代。

LKM 又有不同的投递方式。它们回答的是 “模块由谁、在什么时候加载”，与“内建还是 LKM” 是两个分类层级。

<table><thead><tr><th>LKM 路线</th><th>改造方式</th><th>开始运行的时机</th><th>持久性</th></tr></thead><tbody><tr><td>boot-patch</td><td>ramdisk 中由 ksuinit 先接替 init，加载模块后执行真正的 init</td><td>早期用户态</td><td>修补镜像保留即可再次加载</td></tr><tr><td>boot-patch-v2</td><td>把模块和引导内容加入内核镜像，在 <code>kernel_init</code> 阶段加载</td><td>用户态 init 之前</td><td>修补镜像保留即可再次加载</td></tr><tr><td>late-load / Magica</td><td>在已经运行的内核中动态加载</td><td>系统启动之后</td><td>正常重启后失效，需要再次加载</td></tr></tbody></table>

ramdisk 路线中的 ksuinit 是用户态程序，真正进入内核执行的是它加载的 kernelsu.ko。现有源码也没有支持 “late-load 必须使用 SELinux Permissive” 这一绝对前提。

KernelSU 管理器先设置允许列表与每个应用的 `root_profile`。管理器身份由内核核对 APK v2 签名信息，并结合应用 UID 进行追踪。root_profile 描述目标 UID、GID、附加组、capabilities、SELinux 安全域、命名空间等条件。它可以精细配置，但不代表默认 profile 天然就是最小权限。

KernelSU 不要求系统里真的放一个 `/system/bin/su` 文件。应用探测 su 路径时，内核中的 sucompat 可以为获准调用者呈现对应结果；应用尝试执行 su 时，内核把目标改道到 `/data/adb/ksud`，按照 root_profile 准备权限，另行处理 seccomp 与挂载命名空间，再通过临时文件描述符执行 ksud。ksud 识别自己正在扮演 su 后端，最终运行用户态 shell 或命令。

```
管理器预先配置 allowlist 与 root_profile
  → App 探测或执行 su
  → 内核 sucompat 识别调用者
  → 准备凭据、seccomp 与命名空间
  → 通过临时执行 fd 转入 ksud
  → ksud 执行 shell／命令
```

常规凭据修改使用 “复制当前凭据—修改副本—提交新凭据” 的内核机制。与 Magisk 相比，差异不是最后的 shell 跑在内核里——shell 仍在用户态——而是**请求识别和权限准备已经进入内核。**

管理器与内核通信还使用匿名控制 fd `[ksu_driver]` 和 IOCTL。这个控制 fd 与执行 ksud 时使用的临时 fd 不是同一个对象。可信管理器的 Zygote 子进程可以自动获得控制 fd；普通获授权 App 不会仅因为在允许列表中就自动得到它。接口存在与某条管理命令获准，也仍是两道判断。

KernelSU 的长期 su 处理会使用未占用系统调用槽、`sys_enter` tracepoint 和按进程标记等机制。为了把 ksud 接入开机阶段，它还临时接管 read、fstat 系统调用表入口：当 init 读取目标 rc 时，为它提供追加了 KernelSU 指令的内容，并同步呈现新的文件大小；完成后再恢复这些临时 hook。

因此，KernelSU 同时存在两套目标不同的接入：一套在运行期识别 su 相关请求，另一套只为 init 启动集成服务。磁盘中的原 rc 不需要被永久覆盖。

追加的指令把 ksud 接到 post-fs-data、services、boot-completed 等阶段。KernelSU 还建立 `ksu` 安全域和相应 SELinux 规则，让相关特权进程拥有需要的访问条件。可选的 SELinux 查询隐藏在分析基线中默认关闭，不属于所有 KernelSU 环境必然启用的基础能力。

这里也有三个名称相近、实际不同的对象：

<table><thead><tr><th>对象</th><th>所在位置</th><th>作用</th></tr></thead><tbody><tr><td><code>kernelsu.ko</code></td><td>内核态</td><td>KernelSU 的 Root 载体</td></tr><tr><td>用户功能模块</td><td>用户态为主</td><td>提供文件、脚本、属性和策略扩展</td></tr><tr><td>metamodule</td><td>用户态挂载配套</td><td>为功能模块提供文件挂载实现</td></tr></tbody></table>

本文分析基线中，用户模块的文件挂载依赖 metamodule 的 `metamount.sh`。没有 metamodule 时模块文件不会自动挂载，但不能由此推出全部生命周期脚本都不会执行。KernelSU 的安全模式也主要用于禁用模块，无法修复已经刷坏的启动镜像。

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

_图 6：机器人手机工厂展示 APatch/KernelPatch 对内核镜像、启动入口、系统调用、进程权限和 SELinux 的改造。_

位置图给出 APatch 改造手机的全景。下面的技术流程图进一步展开它最有辨识度的实现：怎样从现成内核中定位函数、植入 kpimg、完成三级启动，再把请求交给 apd。

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

_图 7：APatch/KernelPatch 完整改造流程，包括离线修补、三级启动、分级鉴权和用户态命令后端。_

APatch 面对的是一台已经造好的机器：手里往往没有完整源码，只有编译后的内核镜像。它先从 kallsyms 中拼回 “函数目录”，确定哪里可以下刀；再把 kpimg 追加进镜像，暂时改写启动入口。手机开机时，载荷像三级火箭一样逐步搬运、建好自己的内存和 hook 环境，同时恢复临时借用的位置。系统起来后，内核里的能力负责权限与控制，用户态 `apd` 负责执行命令和组织模块。

```
找位置：从成品内核中恢复符号与关键函数位置
  → 放载荷：追加 kpimg，备份原入口并写入跳转
  → 三级启动：自定位、建映射、安装长期 hook
  → 建通道：提供 su、SuperCall、SuperCMD 等不同入口
  → 接回 Android：由 apd 执行命令、响应启动阶段并管理扩展
```

APatch 与 KernelPatch 承担不同角色。KernelPatch 提供内核二进制修补、hook 和内核扩展基础；APatch 把这些能力连接到 Android 的授权、命令、启动事件和模块管理。图中重点展示的是 kpimg 镜像植入路线。

现成内核镜像由机器指令和数据构成。同一个函数在不同构建中的位置会变化，工具不能永远使用一个固定偏移。KernelPatch 会识别并重建镜像里的 kallsyms 信息。可以把 kallsyms 理解为内核的 “函数名称—位置索引”；工具恢复这份索引后，才能找到需要接入的函数。

随后，工具把 kpimg 载荷加入镜像，保存必要的原始内容，写入预设的符号偏移等信息，调整启动入口，再重新组织启动镜像。“不需要完整源码” 表示它可以处理已经编译的二进制，并不意味着没有兼容要求；架构、内核版本、配置和符号信息都会影响结果。

把载荷放进文件，还不等于它已经能在内核里运行。KernelPatch 通过三个阶段为它建立环境：

```
第一阶段：MMU 尚未开启
自定位 → 保存并恢复原启动入口 → 把载荷送到临时位置

第二阶段：借 paging_init 建立条件
先恢复 paging_init 原指令并运行原逻辑
→ 分配保留内存 → 建立映射 → 二次搬运载荷

第三阶段：进入正式运行
设置代码、数据与 hook 区权限
→ 解析运行期符号 → 安装 hook → 恢复临时 map 区域
```

三次恢复针对的是不同的临时接入位置，不能都归给一个 `restore_map`。入口恢复也不意味着内核回到了完全没有外来代码的状态：常驻载荷和运行期 hook 仍然存在。

inline hook 会修改目标函数开头的指令，让执行先跳进新增逻辑。被覆盖的原指令仍可能需要运行，所以 KernelPatch 还要建立 trampoline，把保存并重定位后的原逻辑接回去。某些 ARM64 指令依赖当前位置，搬家后必须修正，不能简单复制。

```
原来：调用者 → 原函数入口 → 原函数主体

接入后：调用者 → KernelPatch 新逻辑
                    ├─ 直接处理或改变结果
                    └─ trampoline → 原函数主体
```

除 kpimg 外，当前材料还包含动态加载 KernelPatch LKM 的方式。Jailbreak 是在用户态获得加载条件并装入 `kernelpatch.ko` 的流程，LKM 是被加载的内核载体；它们是同一条路线的两个侧面，不是彼此独立的两代技术。

APatch 管理器预先维护允许状态并同步到内核。应用使用 su 时，内核根据已有配置判断，而不是每次都由 apd 临时弹窗询问。可信管理器身份也由内核核对 APK 证书信息；这不是 APatch 独有的能力，KernelSU 同样在内核中处理管理器验签。

KernelPatch/APatch 有几种不同入口：

<table><thead><tr><th>入口</th><th>它负责什么</th></tr></thead><tbody><tr><td>su 兼容入口</td><td>处理获准应用的 su 执行，准备权限并转入 apd</td></tr><tr><td>SuperCall</td><td>借现有系统调用通道传递管理和扩展命令</td></tr><tr><td>SuperCMD</td><td>借特定程序执行形式承载启动事件等命令</td></tr></tbody></table>

它们不是一次 Root 请求连续经过的三站，也不全是 “提权接口”。控制命令采用分级鉴权：调用者可能凭有效密钥状态、可信管理器 UID 或 su 允许身份成为可信来源；密钥管理与 KPM 管理等命令还会进一步要求 `is_authed`。部分允许列表与配置命令位于这层密钥闸门之前，所以不能说所有 SuperCall 都必须提供 SuperKey。

一次 su 请求的主链反而比较清楚：

```
App 发起 su 请求
  → 内核检查调用身份与允许状态
  → 准备 UID、GID、capabilities、安全上下文和相关限制
  → 把执行转入用户态 apd
  → apd 准备终端与环境并运行 shell／命令
```

常规路径使用凭据准备与提交机制，特殊路径可能采用其他处理方式，不能概括成 “一律直接修改几个内存字段”。KernelPatch 还会在内核的 SELinux 访问判断路径中，按特定上下文或任务标记处理结果；这不等于全局关闭 SELinux。

内核已经有了 Root 能力，APatch 仍需要用户态 apd。它负责读取配置、同步允许列表、响应启动阶段、执行 shell，并组织用户模块。

为了让 apd 在正确时间出现，APatch 会在 init 打开配置时介入 openat 路径，把原配置复制到 `/dev/user_init.rc` 一类临时文件，加入所需事件指令，再让 init 读取临时版本。原系统 rc 不需要永久修改。

init 触发相应阶段后，apd 在 post-fs-data 等时机读取 `/data/adb/ap` 中的配置、运行模块任务，并加载加入规则后的 SELinux 策略。因此 APatch 同时存在内核侧访问处理和用户态策略修改，“SELinux 策略一个字不改” 并不准确。

APatch 的扩展还分成两层：

<table><thead><tr><th>扩展</th><th>运行位置</th><th>作用</th></tr></thead><tbody><tr><td>APM</td><td>用户态为主</td><td>提供文件、脚本、属性和策略模块；文件挂载依赖 metamodule</td></tr><tr><td>KPM</td><td>内核态</td><td>KernelPatch 自己的 ELF 内核扩展，由加载器解析、重定位后运行</td></tr></tbody></table>

APM 的形式接近 Magisk 模块，但不能由 “格式相似” 推出所有模块都直接兼容。KPM 不是普通模块压缩包，也不能简单等同于标准 Linux `.ko`。

动态路线中的 `kernelpatch.ko` 又是第三个对象：它负责把 KernelPatch 本体装进内核；KPM 是在这套基础上继续加载的内核扩展。**APM、KPM 和 kernelpatch.ko 名字都带模块含义，实际位于不同层次。**

三者都能让获准应用执行高权限命令，但不能简单排成 “旧、较新、最新” 三代。它们选择的是不同工程路线：Magisk 尽量在用户态把兼容性和生态做完整；KernelSU 把授权与凭据准备下沉到内核；APatch 进一步解决 “只有成品内核、没有源码时怎样植入内核能力” 的问题。

<table><thead><tr><th>比较维度</th><th>Magisk</th><th>KernelSU</th><th>APatch / KernelPatch</th></tr></thead><tbody><tr><td>主要改造入口</td><td>ramdisk 与启动早期用户态</td><td>内建内核代码，或加载 <code>kernelsu.ko</code></td><td>直接修补内核镜像植入 kpimg，或动态加载 KernelPatch LKM</td></tr><tr><td>Root 决策位置</td><td>用户态 <code>magiskd</code></td><td>内核 KernelSU 逻辑</td><td>内核 KernelPatch/APatch 逻辑</td></tr><tr><td>权限怎样产生</td><td>特权服务派生执行进程</td><td>内核按 <code>root_profile</code> 准备并提交凭据</td><td>内核按允许状态和 profile 准备凭据</td></tr><tr><td>最终命令在哪里运行</td><td>用户态 shell／命令进程</td><td>用户态 <code>ksud</code> 后端及其命令进程</td><td>用户态 <code>apd</code> 后端及其命令进程</td></tr><tr><td>App 如何提出请求</td><td>su 客户端连接本地 socket</td><td>sucompat 识别探测／执行，控制面另用匿名 fd 与 IOCTL</td><td>su 兼容入口；控制和扩展另有 SuperCall、SuperCMD</td></tr><tr><td>文件系统模块</td><td>内置 Magic Mount，生态成熟</td><td>模块生命周期内置，文件挂载依赖 metamodule</td><td>APM 配合用户态模块和 metamodule</td></tr><tr><td>App 进程扩展</td><td>原生提供 Zygisk</td><td>通常依赖额外的 Zygisk 实现或其他框架</td><td>通常依赖额外框架</td></tr><tr><td>内核扩展</td><td>Root 主逻辑不常驻内核</td><td>KernelSU 本体位于内核；用户功能模块不是内核插件</td><td>除 Root 本体外，还提供 KPM 内核扩展能力</td></tr><tr><td>兼容性的主要约束</td><td>设备启动镜像、ramdisk 与 init 布局</td><td>内核版本、KMI、模块加载条件或内建适配</td><td>arm64 内核、符号信息、版本和二进制补丁适配</td></tr><tr><td>故障影响范围</td><td>多数逻辑在用户态，但启动镜像修补仍可能导致无法启动</td><td>内核组件故障可能造成 panic 或启动失败</td><td>入口补丁、映射或 hook 出错可能造成启动失败或 panic</td></tr></tbody></table>

这张表也解释了一个常见误会：**“内核 Root” 不等于命令在内核里运行。** KernelSU 和 APatch 把身份判断、凭据修改或 hook 放在内核，最终的 shell、脚本和大量模块任务依然回到用户态完成。它们改变的是授权链的控制点，而不是把整个 Android 工具生态搬进内核。

<table><thead><tr><th>方案</th><th>相对优势</th><th>主要代价或边界</th><th>更适合的需求与设备条件</th></tr></thead><tbody><tr><td>Magisk</td><td>启动布局兼容经验丰富；模块生态最大；Magic Mount 与 Zygisk 集成完整；授权交互直观</td><td>用户态组件、挂载和进程扩展形成较多可观察面；复杂厂商启动布局仍需专门兼容；不提供通用内核插件平台</td><td>希望优先获得兼容性、成熟模块与 Zygisk 生态；日常系统定制、开发调试和广泛机型支持</td></tr><tr><td>KernelSU</td><td>授权识别和凭据准备位于内核；<code>root_profile</code> 能细分 UID、组、能力、安全域和命名空间；GKI/LKM 路线减少重新编译门槛</td><td>受 KMI、内核配置和模块装载条件约束；内核故障影响更大；文件叠加与进程注入通常需要额外组件</td><td>有合适内核或可加载模块条件；希望按应用精细配置 Root 能力；愿意承担内核适配和维护成本</td></tr><tr><td>APatch</td><td>能在缺少完整源码时直接处理成品内核；KernelPatch 提供 inline hook 与 KPM 扩展平台；内核控制与 Android 用户态管理结合紧密</td><td>对架构、kallsyms、内核版本和具体镜像敏感；二进制补丁与 hook 的调试门槛高；系统升级后通常需要重新验证与修补</td><td>原厂内核满足补丁条件、但缺少合适 KernelSU 适配；需要研究成品内核改造或开发 KPM 内核扩展</td></tr></tbody></table>

这里的 “适合” 描述的是技术匹配，不代表某一种方案在所有设备上都更安全、更稳定或更难发现。内核方案把控制点放得更深，也把错误带进了更敏感的位置；用户态方案暴露面相对更多，却积累了更长时间的兼容经验。实际选择取决于启动镜像结构、内核条件、系统升级方式、所需模块，以及能否承担恢复成本。

三者通常还共享几项现实前提：需要取得启动相关镜像并具备刷写或加载条件；持久安装往往涉及 Bootloader 解锁；模块与管理器本身必须可信；系统 OTA 后可能需要重新处理镜像或验证兼容性。某些动态加载或特殊设备路线存在例外，不能把同一安装步骤套在所有手机上。

如果需求只是一条高权限 shell，三者的差别似乎不大；一旦继续追问，路线就会分开：

```
只需要执行高权限命令
  → 三者都能完成，比较设备兼容性与恢复成本

需要大量现成模块、文件替换或 App 进程扩展
  → Magisk 的原生生态最完整

需要按应用限制 UID／能力／安全域／命名空间
  → KernelSU 的 root_profile 更适合表达精细权限

需要直接研究和扩展成品内核行为
  → APatch／KernelPatch 的二进制补丁与 KPM 更有针对性
```

这不是一张 “谁更强” 的排行榜。它展示的是能力放置位置带来的交换：越靠近用户态，越容易复用 Android 生态；越靠近内核，越容易直接干预权限和系统行为，同时也越依赖内核适配与稳定性。

到这里，可以把三种方案放回同一部手机，沿着 “安装—启动—授权—执行—扩展” 五个动作对齐。

<table><thead><tr><th>动作</th><th>Magisk</th><th>KernelSU</th><th>APatch / KernelPatch</th></tr></thead><tbody><tr><td>安装</td><td>修补 ramdisk，部署 magiskinit</td><td>内建，或通过不同路线投递 kernelsu.ko</td><td>修补内核镜像植入 kpimg，或动态加载 KernelPatch LKM</td></tr><tr><td>启动</td><td>magiskinit 准备环境后交还原生 init</td><td>内核组件初始化，并用 rc 接入 ksud 生命周期</td><td>kpimg 分阶段建立映射和 hook，init 再接上 apd</td></tr><tr><td>授权</td><td>magiskd 查询策略，必要时弹窗</td><td>内核读取 allowlist 与 root_profile</td><td>内核读取预授权状态并按命令分级鉴权</td></tr><tr><td>执行</td><td>magiskd 派生特权进程</td><td>内核准备权限，转入 ksud</td><td>内核准备权限，转入 apd</td></tr><tr><td>扩展</td><td>Magic Mount、脚本、Zygisk</td><td>用户功能模块与 metamodule</td><td>APM、metamodule 与内核 KPM</td></tr></tbody></table>

如果只看一次命令执行，三条链分别是：

```
Magisk
App → su 客户端 → magiskd → 特权子进程 → 命令

KernelSU
App → 内核 sucompat → root_profile → ksud → 命令

APatch
App → 内核允许状态／权限处理 → apd → 命令
```

三种方案最后都有用户态命令进程，也都借助 Linux 已有的进程、凭据、文件和安全机制。真正改变的是控制权放在哪里：

*   Magisk 在用户态建立一个拥有特权的控制室。
*   KernelSU 把识别请求和配置权限的闸门放进内核。
*   KernelPatch/APatch 先给现成内核建立一套可定位、可 hook、可扩展的新基础，再把它接回 Android 用户态。

所以，Root 从来不是管理器中的一个按钮。按钮只是入口，背后是一条从启动镜像延伸到内核、从内核返回用户态命令的完整链路。理解这条链，比记住某个文件名或某个工具界面更接近 Root 的本质。

关于检测，帖子和工具已经不少了，比较稳健的比如基于 sellinux 查询的一些检测，需要 zygote 的一些特性，就不重复赘述了，主要是想记录一下这三类方法的操作原理。

[传递专业知识、拓宽行业人脉——看雪讲师团队等你加入！！](https://bbs.kanxue.com/thread-275828.htm)
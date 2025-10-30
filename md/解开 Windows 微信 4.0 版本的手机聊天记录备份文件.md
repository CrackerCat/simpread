> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2068774-1-1.html)

> [md]# 解开 Windows 微信 4.0 版本的手机聊天记录备份文件 -> 今年年初我发布的 [《解开 Windows 微信备份文件》](https://www.52pojie.cn/thread-2021739......

![](https://avatar.52pojie.cn/data/avatar/002/39/16/21_avatar_middle.jpg)xuxinhang _ 本帖最后由 xuxinhang 于 2025-10-30 00:24 编辑_  

解开 Windows 微信 4.0 版本的手机聊天记录备份文件
-------------------------------

* * *

> 今年年初我发布的[《解开 Windows 微信备份文件》](https://www.52pojie.cn/thread-2021739-1-1.html)介绍了 Windows 微信 3.9 版本中备份文件的解密方法，很受大家的欢迎。但随后不久，微信团队正式推出了 4.0 版本，重新设计了整个 “备份与恢复” 功能，因而老方法也不再适用于新版本生成的备份文件。  
> **本文将详细介绍解密 Windows 微信 4.0 版本备份文件的方法。**

* * *

Windows 微信 4.0 的 “备份与恢复” 功能同样可以将手机微信上的聊天记录存储到电脑。但新版本的 “备份与恢复” 功能是彻底推倒重来的，备份文件的结构也与老版本完全不一样。新版本的 “备份与恢复” 需配合新版本的手机微信使用。本文使用的版本为：  
![](https://attach.52pojie.cn/forum/202510/29/205408i6vs638g6fn5n2vc.png)

**版本 2.png** _(34.27 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxMDQ2N3xkMGVkNWI0MHwxNzYxODI3Mjc3fDIxMzQzMXwyMDY4Nzc0&nothumb=yes)

2025-10-29 20:54 上传

新版 “备份与恢复” 的操作步骤与微信 3.9 版本类似：在电脑微信上进入菜单 -“备份与迁移”-“备份与恢复”-“新建备份”，接着在手机上设定时间范围等即开始备份。备份文件存储于如下路径：  
`C:\Users\[用户名]\Documents\xwechat_files\Backup\[微信号]`

![](https://attach.52pojie.cn/forum/202510/29/205406hlvwhw5d6hnqrrny.png)

**版本 1.png** _(40.33 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxMDQ2NnwxOTc1YjM4MXwxNzYxODI3Mjc3fDIxMzQzMXwyMDY4Nzc0&nothumb=yes)

2025-10-29 20:54 上传

分析备份文件的特征
---------

正常操作进行备份，合理选择备份范围使之仅包含一条文字消息，这样得到的备份目录是最简单的，目录结构如下：

```
wxid_0xt662v10ru629 - 微信号
    │  roam_device_info.dat
    │
    └─73cfbe036d741ddf3 - 设备标识
        │  alt_name.dat
        │  backup.attr ＊
        │
        └─files
            └─39 - 第多少次备份
                │  backup_time.dat ＊
                │  detail.dat
                │  phoneid.dat ＊
                │  phone_history.dat ＊
                │  pkg.attr ＊
                │  pkg_info.dat
                │
                └─98dffe08c400f2b…  按会话分组
                    ├─ChatPackage
                    │      1760266855000-1760266855000 ＊  按时间分组
                    └─Index
                            1760266855000-1760266855000 ＊
                            time.dat ＊
                            wholetime.dat ＊

```

使用十六进制编辑器查看每个文件。树状图中标以星号的九个文件结构类似：文件开头是以 RMFH 为首的 128 字节；近结尾处 RMFT 字样至末尾的长度也为 128 字节；中间是一些看不出意义的字节，可能被加密过。剩下的非 RMFH 格式文件包含些许有实际含义的字符，但没有共通的结构，且文件不大，估计也没什么有价值的信息。

![](https://attach.52pojie.cn/forum/202510/29/205411gi5cz1zeizdyzyso.png)

**电脑 1.png** _(38.16 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxMDQ2OHwwYjFlMzg3NnwxNzYxODI3Mjc3fDIxMzQzMXwyMDY4Nzc0&nothumb=yes)

2025-10-29 20:54 上传

备份目录的结构较为复杂。将这份备份恢复到手机，同时使用 FileActivityWatch 软件监视这一过程中微信进程`Weixin.exe`对备份文件的访问情况。

```
在电脑上打开备份列表后
    \73cfbe03\alt_name.dat
    \73cfbe03\files\39\pkg_info.dat
    \73cfbe03\files\39\detail.dat

在手机上启动恢复过程后
    \73cfbe03\alt_name.dat
    \73cfbe03\files\39\pkg.lock
    \73cfbe03\files\39\pkg_info.dat
    \73cfbe03\files\39\detail.dat
    \73cfbe03\backup.attr
    \73cfbe03\files\39\pkg.attr
    \73cfbe03\files\39\98dffe08c400f\ChatPackage\1760266855000-1760266855000

```

在启动恢复过程前，微信访问的文件都是非 RMFH 文件，而 RMFH 文件在启动恢复之后才被访问。据此推断聊天记录具体内容保存在`ChatPackage`文件夹下的 RMFH 格式文件中。现需找出密钥和加密算法将 RMFH 文件解密。

下面顺着生成 RMFH 文件的路径，探寻文件加密的实现逻辑。

调试电脑微信寻找线索
----------

首先想到的还是从电脑微信介入。启动 x64dbg 并附加到微信进程上，查看所有已经加载的模块。打开 “备份与恢复” 界面后，发现新加载了两个名字很有意义的库，分别是`roma_server.dll` 和 `roma_immigrate.dll` ，从文件名推测后者与 “迁移” 功能相关，故把关注点主要放在`roma_server.dll`上面。

到这些 RMFH 格式文件的结构特征明显，如果程序要实现 RMFH 文件结构的生成，则程序内部必然会存在 RMFH 和 RMFT 这两组字符。使用十六进制编辑器打开 `roma_server.dll`  ，搜索 “RMF” 这组字符，没有结果。这说明`roma_server.dll` 没有生成 RMFH 格式文件的功能，即电脑接收到的就已经就是 RMFH 格式文件了。进一步猜测文件加密操作可能同样不在电脑上进行。

至此，我们需要深入手机微信程序寻找答案。

安卓微信静态分析
--------

借助 Android 设置 -“开发者工具”-“显示应用程序的包名” 功能，得知微信备份界面的类名为 `CreateRoamLitePkgUI`。在 Jadx 中打开 apk 文件，定位到类 `CreateRoamLitePkgUI`，从此处着手层层深入分析逻辑。

```
定位到类 CreateRoamLitePkgUI
        button.setOnClickListener(new ViewOnClickListenerC93344f(this));

ViewOnClickListenerC93344f
        "begin save new package"
        C27130x0.f81265a.m27781h 里面的日志提示 GetAllBackupPackage 
                其中的调用 getAllPackagesAsync 是 JNI原生方法
        使用 countDownLatch.await() 与上面getAllPackagesAsync 的调用等待同步
        "WXGBACKUPPACKAGEPREFIX_" 是类似ID的东西，在电脑上的备份文件中也有相关内容
        sourceDeviceId.setBackupRange 设定备份范围，一种链式调用的编码风格
        下面着重分析一下下面这一行
        ((C99054b3) AbstractC99266l.m79336d(
          AbstractC3350d0.m3824a(createRoamLitePkgUI),
          null,
          null,
          new C93354k(build, createRoamLitePkgUI, null),
          3,
          null))
        .m79161N(
          new C93348h(
            c106441d, 
            createRoamLitePkgUI, 
            ProgressDialogC63600q3.m59059f
          );
        build存储了备份指令的一些信息，非常关键，进入调用它的C93354k
        C93348h可能涉及UI更新的一些功能，而m79161N可能是回调的一种写法，类似JavaScript中的Promise链式调用的风格
        AbstractC99266l.m79336d 进入看看里面的变量和枚举的命名，不是业务代码，而像是线程池之类的东西很抽象。
        我们还是进入C93354k一探究竟

进入C93354k，重点关注 invokeSuspend
        其中的kotlin.coroutines.Continuation指明了这是个协程
        反复出现的c63598q12.m59041h 都是更新UI，与失败退出的处理分支联系在一起
        可以识别出正确处理的分支  if (i16 == 0)
        C27123v.f81245a.m27771e().createPackagesAsync 是 JNI原生方法包装，它的参数：
                backupPackage = this.f265414e 是上面提到的备份范围信息，外面包了一层AbstractC0787c0.m707c
                具体到这段代码 AbstractC0787c0.m707c(backupPackage) 代表的是仅包含一个元素的数组，这唯一一个元素是上面讲到的备份范围信息
                new C27135z(c72310n) C27135z是业务代码作为createPackagesCallback

createPackagesAsync 的形参
        通过 CreatePackagesCallbackBridge 实现回调
        真正的JNI方法 jniCreatePackagesAsync
        jniCreatePackagesAsync(
           this.nativeHandler,    
           ZidlUtil.mmpbListSerializeToBasic(arrayList),   
           createPackagesCallbackBridge)
        Native Handler 是什么
        ZidlUtil.mmpbListSerializeToBasic 是MicroMsgProtobuf的意思吗
                返回二维字节数组，其中的每个子数组都是原始数组对应元素的序列化表示

```

最后定位到原生方法 `jniCreatePackagesAsync`。在压缩软件中打开 apk 文件，提取位于`lib/arm64-v8a`的全部 so 动态库文件，接着不区分大小写地搜索函数名`CreatePackagesAsync`。搜索结果指向了 `libaff_biz.so` 这个动态链接库。使用 IDA 打开，继续深入分析备份文件的生成逻辑。

![](https://attach.52pojie.cn/forum/202510/29/205402e3vtcmz6c8zt8v8x.png)

**安静 2.png** _(73.18 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxMDQ2NHxjYzlmMzMwN3wxNzYxODI3Mjc3fDIxMzQzMXwyMDY4Nzc0&nothumb=yes)

2025-10-29 20:54 上传

等待 IDA 分析完成（转为 idle 小绿灯）。函数导出表（“Exports” 视图）中搜不到`CreatePackagesAsync`或类似的函数，说明这个原生方法是动态注册而非静态绑定的。

进入 `JNI_OnLoad` 函数，按 F5 反编译，尝试找出`CreatePackagesAsync`函数的入口。 `JNI_OnLoad` 似乎使用了静态数组结合遍历的写法，各种数据段绕来绕去，完全理不清其中逻辑，暂时放弃从正面逐层深入的做法。

那尝试在 “String” 视图中搜索与加密相关的字符串呢，比如说 "key size"、 "key empty" 什么的。似乎也没有什么发现。线索难道就在这里断了吗？我们需要进一步的思考……

进入 “Imports” 视图搜索导入表，在尝试了 “key”、“size” 等多个字符串后，搜索“enc"（encrypt）出现了一些有意思的结果。

![](https://attach.52pojie.cn/forum/202510/29/205404pessgtkgg4qsz4ks.png)

**安静 3.png** _(47.79 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxMDQ2NXxmZTVlNmI2MXwxNzYxODI3Mjc3fDIxMzQzMXwyMDY4Nzc0&nothumb=yes)

2025-10-29 20:54 上传

这些 “EVP” 开头的函数显然来自 OpenSSL 库，指明 `libaff_biz.so` 调用了 OpenSSL 的加密功能接口。进一步搜索，发现还导入了更多 EVP 开头的函数。查阅 OpenSSL 加密函数的资料 @，基本上这些导入的 EVP 函数涵盖了完整的 OpenSSL 加密过程。另外，注意到 `EVP_aes_128/192/256_gcm` 的导入，联想到使用 EVP 函数前必须导入算法描述字，推测 OpenSSL 库进行的加密只涉及 AES-GCM 算法。

我们不知道这些 GCM 加密是否就是用于生成 RMFH 文件，也不能确定 `libaff_biz.so` 中是否还存在其它加密函数。无论如何，不妨先动态调试一下这部分的加密逻辑。

安卓微信动态调试
--------

Frida 带有 so 动态库注入功能，这次还是用它。使用的 Frida 版本是 16.6.1。（我也试过用 IDA 远程调试器在 so 上打断点，成功过几次，但更多时候还是以微信崩溃告终。因此本文只介绍 Frida 注入 so 的调试方法。）

用一台 root 过的旧手机登录微信。使用真机有一个好处，就是可以原生执行 ARM 指令。（在模拟器中，ARM 会先被转译为 x86 指令，故须提取内部生成的 x86 指令重新做静态分析。）（手机原来的系统基于 Android 6.0，微信中按下 “开始备份” 按钮就失去响应，所以刷入了 Havoc-OS  3.12，基于 Android 10 的三方系统。刷系统也折腾了好久。root 用的是 Magisk。）

### 配置 Frida 系列工具

首先配置 Frida 环境，包括手机上运行的 frida-server 程序和电脑上的 Python 包 frida-tools。这里写得再详细些。

**在手机上运行 frida-server**

1.  从 [Frida 的 Github Release](https://github.com/frida/frida/releases) 下载 ARM64 平台的 frida-server 可执行文件。
2.  [下载 Android Platform Tools](https://developer.android.google.cn/tools/releases/platform-tools?hl=zh-cn) 并解压出 platform-tools 文件夹。配置 PATH 环境变量，使 adb.exe 可运行。
3.  手机里开发者选项打开 ADB 调试，USB 连接到电脑。
4.  电脑 PowerShell 运行 `adb shell` 进入 Android 终端后执行`su`命令，并在手机上 “总是允许”bash 的管理员权限。
5.  使用 `adb push` 将 frida-server 传入手机的 `/data/local/tmp` 目录
6.  执行 `adb shell` - `su`  为 frida-server 添加可执行权限并运行

```
PS > adb push E:\Downloads\frida-server-16.6.1-android-arm64 /data/local/tmp
PS > adb shell

markw:/ $ su
markw:/ # cd /data/local/tmp
markw:/data/local/tmp # chmod +x frida-server-16.6.1-android-arm64   # 指定执行权限
markw:/data/local/tmp # ./frida-server-16.6.1-android-arm64          # 运行 frida server

```

**在电脑上配置 frida-tools**

1.  创建新目录，准备 Python 虚拟环境并激活
2.  安装 frida-tools
3.  执行命令 `frida-ps -U` 确定微信的进程名

```
PS > python -m venv .                              # 在当前目录创建虚拟环境
PS > .\Scripts\activate                            # 激活虚拟环境
(FridaTemp) PS > pip install frida==16.6.1         # 手动指定要安装的版本
(FridaTemp) PS > pip install frida-tools==13.6.0
(FridaTemp) PS > frida-ps -U                       # 列出手机中活动的进程

```

### 选定注入位置

回到 IDA，从`EVP_EncryptInit_ex`函数切入，寻找合适的注入位置。从导入表开始，层层查找交叉引用，定位到四处调用，前两处在一个函数内紧邻，后两处在另一个函数内紧邻。这两个大函数就是目标，分别是`sub_9D5490`和`sub_A0061C`。

![](https://attach.52pojie.cn/forum/202510/29/205414ot38o31bt3blgp3t.png)

**安动 3.png** _(227.21 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjgxMDQ2OXwyOTZkYzlmMHwxNzYxODI3Mjc3fDIxMzQzMXwyMDY4Nzc0&nothumb=yes)

2025-10-29 20:54 上传

先来看`sub_9D5490`。我们在`sub_9D5490`的`EVP_EncryptInit_ex`函数处按 F5 反编译，分析具体逻辑。反编译所得代码简要摘录如下：

```
  v45 = EVP_aes_256_gcm(v43);
  if ( (unsigned int)EVP_EncryptInit_ex(v42, v45, 0LL, 0LL, 0LL) == 1 )
  {
    v46 = (*(_BYTE *)(a1 + 16) & 1) != 0 ? *(_QWORD *)(a1 + 32) : a1 + 17;
    v47 = (v85 & 1) != 0 ? v87 : (char *)&v85 + 1;
    if ( (unsigned int)EVP_EncryptInit_ex(v42, 0LL, 0LL, v46, v47) == 1
      && (unsigned int)EVP_EncryptUpdate(v42, v39, dest, v36, v38) == 1
      && (unsigned int)EVP_EncryptFinal_ex(v42, &v39[SLODWORD(dest[0])], dest) == 1
      && (unsigned int)EVP_CIPHER_CTX_ctrl(v42, 16LL, 16LL, &v110) == 1 )
    {
      EVP_CIPHER_CTX_free(v42);
      v48 = 0;
      // 省略 ... ...
      goto LABEL_93;
    }
  }

```

这是调用 OpenSSL EVP 加密接口的典型套路，结合`EVP_EncryptInit_ex`的函数定义来看，第一次调用指定加密算法，第二次调用才指定密钥 key 和初始化向量 iv。

```
int EVP_EncryptInit_ex(EVP_CIPHER_CTX *ctx, const EVP_CIPHER *cipher,
                       ENGINE *impl, const unsigned char *key,const unsigned char *iv)
int EVP_EncryptUpdate(EVP_CIPHER_CTX *ctx, unsigned char *out, 
                       int *outl,const unsigned char *in, int inl)
int EVP_EncryptFinal_ex(EVP_CIPHER_CTX *ctx, unsigned char *out, int *outl)

```

至于`sub_A0061C`，调用两次 `EVP_EncryptInit_ex` 后，函数就结束了。那么对`EVP_EncryptUpdate`等后续函数的调用应该位于别处。合理猜测`sub_A0061C`相关的加密过程用于加密较长的内容，这才需要把其它 EVP 函数的调用分拆到不同的位置。

```
if ( (unsigned int)EVP_EncryptInit_ex(*(_QWORD *)(a1 + 16), v34, 0LL, 0LL, 0LL) == 1 )
{
    v36 = (*a2 & 1) != 0 ? (unsigned __int8 *)*((_QWORD *)a2 + 2) : a2 + 1;
    v37 = (*(_BYTE *)(a1 + 56) & 1) != 0 ? *(_QWORD *)(a1 + 72) : a1 + 57;
    if ( (unsigned int)EVP_EncryptInit_ex(*(_QWORD *)(a1 + 16), 0LL, 0LL, v36, v37) == 1 )
      *(_BYTE *)(a1 + 24) = 1;
}

```

### 初次注入：尝试是否命中

Frida 的 so 注入功能只能向函数添加钩子，即仅能在进入函数和退出函数时触发钩子`onEnter` 和 `onLeave`。找出 `sub_9D5490` 和 `sub_A0061C` 这两个函数的起始地址：  
　　`.text:00000000009D5490    sub_9D5490`  
　　`.text:0000000000A0061C    sub_A0061C`

编写 Frida 脚本，把两个函数都钩上。步骤大致是，先找到 `libaff_biz.so` 模块基址，再在基址上偏移得到函数地址，最后注册`onEnter`钩子。一并把链接库路径打印出来。

```
/* aff_hook_try.js */

Java.perform(function(){
    var module_object = Process.findModuleByName('libaff_biz.so')
    var module_address_int = parseInt(module_object.base.toString(10))
    console.log(`Found module: ${module_object.name}@${module_object.base.toString()} -> ${module_object.path}`);

    // sub_9D5490
    Interceptor.attach(
        ptr(module_address_int + 0x9D5490),
        {
            onEnter: function () {
                console.log('sub_9D5490:EVP_EncryptInit_ex onEnter')
            }
        }
    )

    // sub_A0061C
    Interceptor.attach(
        ptr(module_address_int + 0xA0061C),
        {
            onEnter: function () {
                console.log('sub_A0061C:EVP_EncryptInit_ex onEnter')
            }
        }
    )
})

```

在电脑上点击 “新建备份”，等手机弹出备份界面后，在电脑上启动 Frida 并执行脚本：

```
(FridaTemp) PS F:\wechat\FridaTemp> frida -U -n 微信
[Redmi 4::微信 ]-> %load  aff_hook_simple.js

```

特别注意看清`libaff_biz.so`的位于哪个目录，`lib`目录下的`libaff_biz.so` 才是 apk 内置的链接库。位于 `tinker` 目录的 `libaff_biz.so` 是微信通过热更新机制下载的其它版本。如果调用的是`tinker`下面的，清空该目录后完全退出微信再重新操作。

继续进行备份，仍仅备份一条聊天消息。开始备份后，可以看到注入的两个钩子都触发了，`sub_9D5490`执行了很多次，而`sub_A0061C`仅执行了九次。

九次？备份目录中 RMFH 文件也恰好是九个！可以肯定`sub_A0061C`就是用来做 RMFH 文件加密的！接下来，只需要从`sub_A0061C`对`EVP_EncryptInit_ex`的调用中提取出密钥和初始向量就可以了。

### 再次注入：提取加密参数

回到 IDA 继续分析`sub_A0061C`。第一次`EVP_EncryptInit_ex`指定加密算法时是从不同密钥长度的 AES-GCM 中选择一种。GCM 算法涉及密钥 key 和初始向量 iv 两个参数输入，以及 TAG 校验输出。

第二次`EVP_EncryptInit_ex`调用才明确了密钥和初始向量，对应 C 代码中的`v36`和`v37`，而它们又来分别自于`sub_A0061C`的两个函数输入参数，第二位的参数`a2`和第一位的参数`a1`。以`v37`为例，先判断`(a1+56)`处字节低位是否为零，为零则使`v37`指向`(a1+57)`开始的字节。

修改 Frida 脚本，把可能有用的字节范围打印出来。这里需要特别注意，`onEnter`在进入函数时触发，而刚进入函数时数据还没有加载，所以应该在`onEnter`中保存函数参数的指针，而另在`onLeave`中读出来。

```
/* aff_hook_simple.js */

Java.perform(function(){
    var module_object = Process.findModuleByName('libaff_biz.so')
    var module_address_int = parseInt(module_object.base.toString(10))
    console.log(`Found module: ${module_object.name}@${module_object.base.toString()} -> ${module_object.path}`);

    // 不再关注sub_9D5490，删掉

    // sub_A0061C
    Interceptor.attach(
        ptr(module_address_int + 0xA0061C),
        {
            onEnter: function (args) {
                console.log('sub_A0061C:EVP_EncryptInit_ex onEnter')
                this.ctxargs = [ args[0].add(0), args[1].add(0) ];
            },
            onLeave: function () {
                console.log('sub_A0061C:EVP_EncryptInit_ex onLeave')
                var args = Array.from(this.ctxargs);
                // 初始向量 vi -> v37 -> a1+56
                console.log(hexdump(args[0].add(56), { offset: 0, length: 32, header: true, ansi: true }))
                // 密钥 key -> v36 -> a2
                console.log(hexdump(args[1].add(0), { offset: 0, length: 32, header: true, ansi: true }));
            }
        }
    )
})

```

加载注入脚本后再次新建备份，Frida 打印出两组看似随机的字节。以零字节为分界，推测密钥长 16 字节，初始向量长 12 字节，符合惯例。

```
sub_A0061C:EVP_EncryptInit_ex onEnter
sub_A0061C:EVP_EncryptInit_ex onLeave
             0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F 
7a0fbf8da8  18 cf 7c cf 07 44 9f 2f 0c 82 59 d8 8d 00 00 00 
7a0fbf8db8  00 00 00 00 00 00 00 00 d0 b2 3b e6 79 00 00 00 
             0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F 
7a0fbf8d08  20 59 d8 8d 4d 81 5b 97 cf 07 44 9f aa dd 5a a1 
7a0fbf8d18  79 00 00 00 00 00 00 00 01 00 00 00 00 00 00 00 
sub_A0061C:EVP_EncryptInit_ex onEnter
sub_A0061C:EVP_EncryptInit_ex onLeave
             0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F 
7a0fbf8da8  18 de 79 95 aa ab aa dd 5a a1 aa 86 90 00 00 00 
7a0fbf8db8  00 00 00 00 00 00 00 00 d0 b2 3b e6 79 00 00 00 
             0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F 
7a0fbf8d08  20 59 d8 8d 4d 81 5b 97 cf 07 44 9f aa dd 5a a1
7a0fbf8d18  79 00 00 00 00 00 00 00 01 00 00 00 00 00 00 00 
[Redmi 4::微信 ]->

```

TAG 校验字节也可经类似步骤提取。提取 TAG 校验的函数为`EVP_CIPHER_CTX_ctrl(ctx, EVP_CTRL_GCM_GET_TAG, gmac_len, gmac)`，其中 `EVP_CTRL_GCM_GET_TAG=16`。没有校验码不影响解密，所以提取 TAG 的具体步骤就不再展开了。如果想进一步了解 EVP 函数的细节，可阅读 OpenSSL 源代码中的`evp.h`头文件。

解密备份文件
------

每个 RMFH 密文的初始向量不同，但各自初始向量在文件中的位置应该是一致的。任取一个初始向量，在这九个 RMFH 文件中搜索，就可以确定初始向量在文件中的偏移。

搞清 RMFH 文件结构后，即编写 Python 脚本实现解密。先尝试解密存储了聊天消息的备份文件（`ChatPackage\1760266855000-1760266855000`）。所得明文包含了有意义的字符，试着按 protobuf 格式解析也是正常的。

```
from Crypto.Cipher import AES  # pip install pycryptodome
import blackboxprotobuf        # pip install bbpb

def aes_gcm_decrypt(key, nonce, ciphertext):
    # 使用密文ciphertext和初始向量nonce解密，不做校验
    cipher = AES.new(key, AES.MODE_GCM, nonce=nonce)
    plaintext = cipher.decrypt(ciphertext)
    return plaintext

def yank_package(filename='phoneid.dat'):
    #以二进制打开RMFT文件，读取密文cipher,初始向量iv,校验tag
    with open(filename, 'rb') as f:
        filebytes = f.read()
        rmfh = filebytes[0:128]
        rmft = filebytes[-128:]
        cipher = filebytes[128:-128]
        iv = rmfh[19:19+12]
        tag = rmft[10:10+16]
    return iv, cipher, tag

if __name__ == "__main__": 
    KEY = bytes.fromhex('59 d8 8d 4d 81 5b 97 cf 07 44 9f aa dd 5a a1 79')

    iv, ciphertext, tag = yank_package('1760266855000-1760266855000')
    plaintext = aes_gcm_decrypt(KEY, iv, ciphertext)

    print(plaintext)
    print(' ')

    pbob, _ = blackboxprotobuf.decode_message(plaintext)
    from pprint import pprint
    pprint(pbob)


```

```
Python 3.13.0 (tags/v3.13.0:60403a5, Oct  7 2024, 09:38:07) [MSC v.1941 64 bit (AMD64)]
Type "help", "copyright", "credits" or "license()" for more information.

================= RESTART: F:\wechat\FridaOnMarkwHavoc\test.py =================

b'\n\xc9\x03\n\xc6\x03\x08\x01\x12\x1825912385999386020@openim\x1a\x13
wxid_0xt662v10ru629"\x98\x01[\xe5\xba\x86\xe7\xa5\x9d]\xe5\x85\xac\xe5
\xbc\x80\xe7\x9b\xb4\xe6\x92\xad19:00\xe5\xbc\x80\xe5\xa7\x8b\xef\xbc
\x9a\n[\xe7\xa4\xbc\xe7\x89\xa9]\xe4\xef\xbc\x9a\xe9\x9b\x86\xe4\xbd
\x93\xe5\xa4\xa7\xe8\xb0\x83\xe6\x88\xe7\x9c\x8b\xef\xbc\x9f\x9d32\xe0
\x01<msgsource>\n\t<strid>KHGSp2AgvgQ022919AMg+qT86358_1</strid>\n\t
<signature>N0_V1_0Mxf|vYVp</signature>\n\t<tmp_node>\n\t\t<publisher-id>
</publisher-id>\n\t</tmp_node>\n</msgsource>\n8\xec\xc5\xa2\xca]@\xd5
\xdd\xf5\x96\x03\x12\n\x08\x83\xa2\xca]'

{'1': {'1': {'1': 1,
             '2': '25912385999386020@openim',
             '3': 'wxid_0xt662v10ru629',
             '4': '[庆祝]公开直播19:00开始：\n'
                  '[礼物]今晚主题：集体大调整，下周怎么看？\n',
             '5': 1760277455000,
             '6': '<msgsource>\n'
                  '\t'
                  '<strid>KHGSp2AgvgQ022919AMg+qT86358_1</strid>\n'
                  '\t<signature>N0_V1_0Mxf|vYVp</signature>\n'
                  '\t<tmp_node>\n'
                  '\t\t<publisher-id></publisher-id>\n'
                  '\t</tmp_node>\n'
                  '</msgsource>\n',
             '7': 6743167896357006380,
             '8': 895172629}},
 '2': {'1': 6743164213579636380}}

```

现在，我们得到了消息的原始内容。其它 RMFH 文件可以也同样解密。

———

文章开头举例分析的文件树来自仅含一条文字消息的备份。如果备份包含多个对话、多条消息，以及富媒体或接收的文件，目录结构会更复杂。用 “新建备份” 为同一设备再创建一个备份，新备份包含两个会话，涉及文字、图片、视频。其简略目录树如下所示：

```
wxid_0xt662v10ru629 - 还是同样的微信号
└─73cfbe036d741ddf3 - 还是同样的设备标识
    │  alt_name.dat
    │  backup.attr
    │
    └─files
        ├─43  -  备份序号 (这是文章开头举例的那个简单的备份)
        │  └─ 还是那些文件
        │
        └─44  -  备份序号 (对同一设备执行多次“新建备份”后，序号不同的多份备份会并存)
            │  pkg.attr
            │  pkg_info.dat
            │  tar_index.dat
            │
            ├─43d8dd89afdcd  -  会话 (1/2 共2个会话)
            │  ├─ChatPackage - 聊天消息(按时间段拆分至数个文件)
            │  │      1760284755000-1761564350000
            │  │
            │  ├─Index  - 索引 (不感兴趣)
            │  │      1760284755000-1761564350000
            │  │      time.dat
            │  │      wholetime.dat
            │  │
            │  └─Media  - 媒体文件 (图片、视频、文件等)
            │          1760284755000-1761564350000.tar.enc
            │
            └─e48f8a5ea7c263 - 会话 (2/2)
                ├─ChatPackage  - 聊天消息
                │      1761747730794-1761747730794
                │
                ├─Index
                │
                └─Media  - 媒体文件
                        1761747730794-1761747730794.tar.enc
                        2319603979416722223_m

```

`ChatPackage`存储的是聊天消息，RMFH 解密后可按 protobuf 格式解析，上面已经展示过了。聊天消息引用的媒体文件（包括图片、视频、语音、文件等各种形式）可对应地在`Media`中找到。如果媒体文件体积较小，如普通图片、视频缩略图等，则与`ChatPackage`对应地按时间段归集至不同的`.tar.enc`文件中，解密后即普通的 tar 归档文件，归档文件内含小文件未经加密。如果文件体积较大，则存储至独立文件，如上面的`23196_m`，解密后即原文件本身。不论文件大小，均需据对应的聊天消息确定文件类型或文件名。

∞
-

整个逆向过程遇到的最大困难是原生方法和 so 文件的分析。而解决问题的关键，在于根据导入表函数反推定位加密逻辑的位置。否则，从入口函数一点一点地读代码，一层一层地分析根本不现实。

比较遗憾的是，根据分析 Windows 微信不会解密 RMFH 文件，核查内存转储文件可知内存中也不存在该密钥，换言之目前该 RMFH 密钥仅能从手机端获取。特别提醒，每个微信账号的 RMFH 密钥在较长时间内应该是固定的，故请尽可能安全可靠地保管该密钥。

微信 4.0 版本已经实现了 Windows、macOS、Linux 多桌面端的功能统一，各平台下备份文件的格式是一致的。

参考
--

**使用的软件**

*   IDA Pro 7.7.220118 (SP1) (x86 , x64 , ARM64) - 吾爱破解  
    [https://www.52pojie.cn/thread-1581672-1-1.html](https://www.52pojie.cn/thread-1581672-1-1.html)
*   jadx 1.5.0
*   frida 16.6.1

**参考资料**

｜ 关于 ASE 加密算法和 OpenSSL EVP 接口使用

*   OpenSSL 中文手册之 EVP 库详解_openssl evp-CSDN 博客  
    [https://blog.csdn.net/woaiclh13/article/details/121096898](https://blog.csdn.net/woaiclh13/article/details/121096898)
*   分组密码的模式——ECB、CBC、CFB、OFB、CTR  
    [https://blog.csdn.net/weixin_43946212/article/details/108116251](https://blog.csdn.net/weixin_43946212/article/details/108116251)
*   使用 OpenSSl 库实现 AES-GCM-128 算法（C 语言）_aes-128-gcm-CSDN 博客  
    [https://blog.csdn.net/2201_75357739/article/details/143184907](https://blog.csdn.net/2201_75357739/article/details/143184907)

｜ 关于 Frida 使用

*   基于 frida 的 so 函数 hook 实战_frida so-CSDN 博客  
    [https://blog.csdn.net/hao5335156/article/details/113475875](https://blog.csdn.net/hao5335156/article/details/113475875)  
    [https://web.archive.org/web/20250212231739/https://blog.csdn.net/hao5335156/article/details/113475875](https://web.archive.org/web/20250212231739/https://blog.csdn.net/hao5335156/article/details/113475875)  
    [https://yandexwebcache.net/...](https://yandexwebcache.net/yandbtm?fmode=inject&tm=1761754695&tld=com&lang=zh&la=1757700096&text=https%3A//blog.csdn.net/hao5335156/article/details/113475875&url=https%3A//blog.csdn.net/hao5335156/article/details/113475875&l10n=en&mime=html&sign=2b07217f1a08591c93a9806a5735e0a2&keyno=0)
*   Android 之 Frida 框架完全使用指南_android frida-CSDN 博客  
    [https://blog.csdn.net/qq_38474570/article/details/120876120](https://blog.csdn.net/qq_38474570/article/details/120876120)
*   frida 学习笔记 3--hook so 中的方法  
    [https://www.52pojie.cn/thread-1264025-1-1.html](https://www.52pojie.cn/thread-1264025-1-1.html)
*   Frida 文档 - Javascript API  
    [https://frida.re/docs/javascript-api/#nativepointer](https://frida.re/docs/javascript-api/#nativepointer)![](https://avatar.52pojie.cn/images/noavatar_middle.gif)windowsyyj 公众号看到的，很有用  
经过测试，微信聊天记录备份到 PC 端后，微信号从手机端退出登录，登录另一台手机，可以从 PC 端进行聊天记录恢复。  
如果 RMFH 密钥没有存储到 PC 端上，这是否说明密钥是存储到微信服务器或者可以通过某种算法计算出来？![](https://avatar.52pojie.cn/data/avatar/001/14/40/46_avatar_middle.jpg)jmzqwh 公众号看到的，很有用  
经过测试，微信聊天记录备份到 PC 端后，微信号从手机端退出登录，登录另一台手机，可以从 PC 端进行聊天记录恢复。  
如果 RMFH 密钥没有存储到 PC 端上，这是否说明密钥是存储到微信服务器或者可以通过某种算法计算出来？![](https://avatar.52pojie.cn/data/avatar/000/41/31/24_avatar_middle.jpg)frankssy 强是真的强![](https://avatar.52pojie.cn/images/noavatar_middle.gif)呵 Eric 呵 太牛了大佬 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) pes2077 支持微信 pc 版，越来越好。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)woshixiaocaini1 太牛了大佬 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) mmdg13142 顶级大佬，满眼都是崇拜![](https://avatar.52pojie.cn/data/avatar/000/49/40/39_avatar_middle.jpg)梦入神机 本人老电脑上存有聊天记录，可有方法打开？？？![](https://avatar.52pojie.cn/images/noavatar_middle.gif)LYG001 大佬太强了，牛牛牛 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) zyvaliant 这个太强了，大佬牛 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) Y87699R3578 支持微信 pc 版，越来越好。
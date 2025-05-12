> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2025482-1-1.html)

> [md]# **Pyinstaller Repack 指南 ** 大一牲，想要 Patch 一个抢学校图书馆座位的软件，却困于 Pyinstaller 程序的重打包，以此为契机研究了一下 Repack 的完整流程，避免......

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)hitachimako _ 本帖最后由 hitachimako 于 2025-4-19 03:29 编辑_  

**Pyinstaller Repack 指南**
-------------------------

大一牲，想要 Patch 一个抢学校图书馆座位的软件，却困于 Pyinstaller 程序的重打包，以此为契机研究了一下 Repack 的完整流程，避免接触难懂难修改的字节码，并使用了一个几乎无人知晓的工具`pyinstaller-repacker`（截至这篇文章完稿，仅有 4 个 star）

### **0. 目标**

本指南将用这个简单的例子教你如何：

*   导出程序中所有的`pyc`文件
*   反编译并阅读程序源码
*   直接对源码进行修改，并 Repack 一个修改版

### **1. 准备工具**

*   查看源程序 Python 版本的工具：[pyinstxtractor](https://github.com/extremecoders-re/pyinstxtractor)
*   Repack 工具：[pyinstaller-repacker](https://github.com/pyinstxtractor/pyinstaller-repacker)
*   在线反编译器：[PyLingual](https://pylingual.io/)
*   一个任意版本的 [Python 环境](https://python.org)

### **2. 解包源程序**

1.  查看源程序打包使用的 Python 版本：  
    ![](https://attach.52pojie.cn/forum/202504/19/031555kzbqq1bpoxkswg96.png)
    
    **图片 1.png** _(79.22 KB, 下载次数: 0)_
    
    [下载附件](forum.php?mod=attachment&aid=Mjc3MjMzM3wwNmUyMzZlYnwxNzQ3MDMyNDM2fDIxMzQzMXwyMDI1NDgy&nothumb=yes)
    
    2025-4-19 03:15 上传
    
    如图得知源程序打包使用的 Python 版本为`3.8`，所以我们要安装一个`3.8`版本的 Python 环境（此处不作管理多 Python 版本的教程，请自行管理环境），并在**对应的 Python 版本**中安装所需的依赖： `pip install lxml lief`
    
2.  使用**对应版本的 Python 环境**运行`pyinstaller-repacker`脚本，解包源程序: `python .\pyinst-repacker.py extract [源程序名]`  
    ![](https://attach.52pojie.cn/forum/202504/19/031558gb4fsw4wq0b0slcs.png)
    
    **图片 2.png** _(125.33 KB, 下载次数: 0)_
    
    [下载附件](forum.php?mod=attachment&aid=Mjc3MjMzNHxjMzhjYjk2NnwxNzQ3MDMyNDM2fDIxMzQzMXwyMDI1NDgy&nothumb=yes)
    
    2025-4-19 03:15 上传
    
    运行至`[+] Done!`时，即为完成解包
    

### **3. 反编译源码**

1.  进入目录`[源程序名]-repacker/FILES`，在此处即可找到程序的**入口点** pyc，将其拖入 [PyLingual 反编译器](https://pylingual.io/)进行反编译：  
    ![](https://attach.52pojie.cn/forum/202504/19/031600leiz08gxo667ue3x.png)
    
    **图片 3.png** _(231.29 KB, 下载次数: 0)_
    
    [下载附件](forum.php?mod=attachment&aid=Mjc3MjMzNXxmMzM5MWIxOHwxNzQ3MDMyNDM2fDIxMzQzMXwyMDI1NDgy&nothumb=yes)
    
    2025-4-19 03:16 上传
    
      
    如图，发现入口点启动了`igotolib_editable`包含的 App 窗口，接着分析该文件
2.  进入目录`[源程序名]-repacker/FILES/PYZ-00.pyz`，找到其中的`igotolib_editable.pyc`，拖入反编译器：  
    ![](https://attach.52pojie.cn/forum/202504/19/031602qa8al30c3d29a9vv.png)
    
    **图片 4.png** _(166.42 KB, 下载次数: 0)_
    
    [下载附件](forum.php?mod=attachment&aid=Mjc3MjMzNnw0MmRkNzc4MnwxNzQ3MDMyNDM2fDIxMzQzMXwyMDI1NDgy&nothumb=yes)
    
    2025-4-19 03:16 上传
    
      
    找到购买检测逻辑

### **4. 修改与重新打包**

1.  点击反编译器右上角的蓝色下载按钮，即可将源码下载至本地。
2.  完全删除`status`检测逻辑，并加上一些 Cracked 信息：  
    ![](https://attach.52pojie.cn/forum/202504/19/031604py5nbyxufnrfyza7.png)
    
    **图片 5.png** _(31.87 KB, 下载次数: 0)_
    
    [下载附件](forum.php?mod=attachment&aid=Mjc3MjMzN3xlNjdmNDllYnwxNzQ3MDMyNDM2fDIxMzQzMXwyMDI1NDgy&nothumb=yes)
    
    2025-4-19 03:16 上传
    
    ]
3.  使用**对应版本的 Python 环境**将修改后的源码编译为`pyc`文件：`python -m py_compile [源码]`：  
    ![](https://attach.52pojie.cn/forum/202504/19/031606a2w7cg748cpaucrf.png)
    
    **图片 6.png** _(68.25 KB, 下载次数: 0)_
    
    [下载附件](forum.php?mod=attachment&aid=Mjc3MjMzOHw2ZWE3ZTQyMHwxNzQ3MDMyNDM2fDIxMzQzMXwyMDI1NDgy&nothumb=yes)
    
    2025-4-19 03:16 上传
    
      
    如图，将会生成一个`__pycache__`文件夹，内含的文件即为修改后的`pyc`文件
4.  将修改后的`pyc`文件重命名为被替换的`pyc`文件名称，替换原`pyc`文件后，重新打包为`exe`：`python .\pyinst-repacker.py build .\[源程序名]-repacker\`  
    ![](https://attach.52pojie.cn/forum/202504/19/031609rg09466kkb3v266o.png)
    
    **图片 7.png** _(135.97 KB, 下载次数: 0)_
    
    [下载附件](forum.php?mod=attachment&aid=Mjc3MjMzOXw0ZmY4MTMwNXwxNzQ3MDMyNDM2fDIxMzQzMXwyMDI1NDgy&nothumb=yes)
    
    2025-4-19 03:16 上传
    
      
    等待一段时间后运行至`[+] Done!`时，即为完成打包，在目录`[源程序名]-repacker\`中即可找到 Repack 后的程序。

### **5. 大功告成**

测试程序，运行正常：  
![](https://attach.52pojie.cn/forum/202504/19/031611ulb8bhlal66zyam6.png)

**图片 8.png** _(33.97 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3MjM0MHwyZDk0YTMzYnwxNzQ3MDMyNDM2fDIxMzQzMXwyMDI1NDgy&nothumb=yes)

2025-4-19 03:16 上传

### **6. 说明**

*   使用到的两个工具`pyinstxtractor`与`pyinstaller-repacker`已上传至附件
*   即使使用的 Python 版本不对应，`pyinstaller-repacker`也能够从 pyz 中解压出文件，但是会导致无法反编译，所以请**务必使用与源程序相同的 Python 版本**!!
*   经测试，PyLingual 能够正确反编译 3.13 及以下所有版本的 pyc

![](https://static.52pojie.cn/static/image/filetype/zip.gif)

[tools.zip](forum.php?mod=attachment&aid=Mjc3MjM0MXxhNzJiOWQxNnwxNzQ3MDMyNDM2fDIxMzQzMXwyMDI1NDgy)

2025-4-19 03:20 上传

点击文件名下载附件

下载积分: 吾爱币 -1 CB  

9.93 KB, 下载次数: 106, 下载积分: 吾爱币 -1 CB![](https://avatar.52pojie.cn/data/avatar/001/29/66/42_avatar_middle.jpg)Jx29 解包可以用 pyinstxtractor-ng，而且还有 web 版本 ![](https://avatar.52pojie.cn/data/avatar/000/00/00/01_avatar_middle.jpg) Hmily 刚看文章之前我还在想既然用了 pyinstxtractor 为啥不用 pyinstxtractor-ng 解包然后直接用 Decompyle++ 反编译，直到看完文章试了下 PyLingual ，居然之前遇到 Decompyle++ 无法反编译的都支持了，666！ ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 沙发，学习了，不过我一般是正向使用 pyinstaller，怕破解的话可以用 Themida 加壳 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) xiatao

> [Jx29 发表于 2025-4-19 10:36](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=52892441&ptid=2025482)  
> 解包可以用 pyinstxtractor-ng，而且还有 web 版本

虽然 ng 跨版本解包比较方便，但是好像不支持 repack![](https://avatar.52pojie.cn/images/noavatar_middle.gif)hitachimako 跨版本！！我的 3.13 版本无法反编译 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) waimn 这是好东西，不错，以后有需要解包的回来找 ![](https://avatar.52pojie.cn/data/avatar/000/11/87/93_avatar_middle.jpg) 大一都这么厉害了，赞一个 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) fanqie8 Python 的加密 pye 文件有解密的工具吗？![](https://avatar.52pojie.cn/images/noavatar_middle.gif)燃香小狼

> [gxlly 发表于 2025-4-20 09:31](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=52896779&ptid=2025482)  
> Python 的加密 pye 文件有解密的工具吗？

这个是用 AES128 ECB 模式加密的，要逆向_pyconcrete.pyd 拿到密钥才行 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) gxlly 这么厉害，赞一个！
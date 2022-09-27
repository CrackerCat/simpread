> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-263458.htm)

> [原创]server03 从源码到镜像

说明：不知道能不能发！已经在我公众号里发了，如果违规请版主帮忙删除吧！好像看雪发不了视频？麻烦各位到文中提到的链接或者我公众号里，或者 B 站里我都传了。或者文末的百度云盘里也有一份更加详细的资料。

前言
--

最近跟着国外大佬的视频，从头到尾编译了一遍 `server2003`。特地把整个过程录制了下来，记录了把 `server2003` 源码打包成镜像的完整过程，主要分为：环境搭建，编译，验证三个部分。

 

本文及视频仅用于技术交流，请勿用于非法用途。如有侵权，请联系我删除。

环境搭建
----

有几个关键点需要注意：

1.  为了避免不必要的麻烦，请选择 `32` 位英文版的 `xp` 系统进行安装。我使用中文版 `xp` 编译有问题。
2.  在 `vmware` 中安装系统时，默认只会生成一个 `C` 盘。源码最好解压到 `D` 盘，需要新增一个盘符。
3.  `D` 盘需要 `40 GB` 的磁盘空间。
4.  根据自己主机情况，调整虚拟机中的设置。配置越高，编译时间越短。

以上几点，视频中都有提及。

 

<video src="http://resources.bianchengnan.tech/server2003-from-source-to-image/part1-setup-install-and-config-xp.mp4"></video>

编译
--

环境搭建好之后，就可以编译了。主要步骤如下：

*   解压源码（`nt5src\Source\Win2k3\NT\*`）到 `d:\srv03rtm` 下，为了避免不必要的麻烦，请务必解压到该文件夹下。
    
*   解压完成后，去除 `d:\srv03rtm` 的只读属性，一定要勾选 `将修改应用于此文件夹、子文件夹和文件`。
    
*   解压 `win2003_prepatched_v6b.zip` 到 `d:\srv03rtm` 下，如果操作正确的话会提示是否覆盖现有文件，选择 `Yes To All`。
    
*   手动安装证书文件。在 `d:\srv03rtm\tools` 文件夹下找到 `driver.pfx` 文件，双击安装。一直点击 `Next`，直到完成。如果是第一次安装会有安全警告，请选择 `Yes`。
    
*   跳过脚本中的证书安装操作。编辑 `d:\srv03rtm` 文件夹下的 `prebuild.cmd` 文件，修改 `SKIPCERTINSTALL` 的值为 `1`。
    
*   双击运行 `prebuild.cmd`，耐心等待出现 `Press Any key to continue...`，回车。
    
*   新建 `razzle.cmd` 快捷方式。
    
    设置 `Target` 的值为 `%windir%\system32\cmd.exe /k d:\srv03rtm\tools\razzle.cmd free offline`。
    
    设置 `Start in` 的值为`d:\srv03rtm\`。
    
    ![](https://bbs.pediy.com/upload/attach/202011/873494_VU4ZEDJC4CTM6AW.png)
    
*   双击新建的 `razzle` 快捷方式，执行一段时间后会弹出记事本界面，直接关闭即可。
    
*   `razzle` 执行完成后，**不要关闭！不要关闭！不要关闭！**输入 `build /cZP` 进行编译。我编译了大概 `3` 个小时。
    
    > **敲黑板：**
    > 
    > `razzle.cmd` 会为当前命令行设置一些临时的环境变量，比如，添加 `build` 所在的路径到 `PATH`。
    
*   确认编译结果！编译完成后，不应该有任何错误，只会有一些警告，如果有错误，说明前面某个步骤出错了。
    
    ![](https://bbs.pediy.com/upload/attach/202011/873494_5JQRJD4W7H3JYAT.png)
    

编译成功后就可以开始准备打包了。

*   解压 `missing.7z` 中的文件到 `d:\binaries.x86fre` 下。
    
*   执行 `tools\postbuild.cmd -sku:{srv}`。执行需要一段时间，请耐心等待。
    
*   执行完成后，检查 `d:\binaries.x86fre\build_logs` 下的 `postbuild.err` 中的错误数。不应该有很多，但也不会太少，很可能像下图这样。
    
    ![](https://bbs.pediy.com/upload/attach/202011/873494_ZEZPDC7S25KKSZG.png)
    
*   解压 `2k3missingx86fre NOTFINAL v3.7.7z` 中的文件到 `d:\binaries.x86fre` 下。一定要注意：**不要覆盖任何现有文件**。
    
*   解压完成后，再次执行 `tools\postbuild.cmd -sku:{srv}`。
    
*   执行完成后，再次检查 `d:\binaries.x86fre\build_logs` 下的 `postbuild.err` 文件中的错误数，这次应该只有很少的几个错误，类似下图：
    
    ![](https://bbs.pediy.com/upload/attach/202011/873494_82JGEZYNEFEMAQ9.png)
    
*   执行 `tools\postbuild.cmd -sku:srv` 。执行成功后，会在 `d:\binaries.x86fre` 下生成一个名为 `srv` 的文件夹。
    
    > **敲黑板：**
    > 
    > 这是第三次执行 `tools\postbuild.cmd`，这次的参数是不带大括号的 `-sku:srv`。国外大佬的视频中并没有录制执行第三次的过程，所以有的小伙伴儿会在这里被坑。
    
*   最后，执行 `tools\oscdimg.cmd srv` 即可生成最终的系统镜像文件。
    

如果上面的描述不够明白，没关系，看视频。

 

<video src="http://resources.bianchengnan.tech/server2003-from-source-to-image/part2-build.mp4"></video>

验证
--

拷贝制作好的系统镜像和符号文件（只需要拷贝 `symbols.pri` 文件夹下的符号文件）到主机上。拷贝完成后，使用生成的镜像文件新建虚拟机，具体过程与安装 `XP` 虚拟机类似，安装过程从略。调试环境搭建，及使用 `windbg` 进行内核调试的过程请参考视频。

 

<video src="http://resources.bianchengnan.tech/server2003-from-source-to-image/part3-verify_image_and_symbol.mp4"></video>

相关文件
----

我已经把相关的文件上传到百度云盘了。

 

链接: https://pan.baidu.com/s/1M8vId2uFyxgIUTE2wDdXvQ

 

提取码: 163c

 

这些文件包括：

*   `32` 位英文版 `xp` 系统镜像。
    
*   油管上外国大佬录制的视频及缺失的文件，及相应的链接。
    
*   `razzle.cmd` 快捷方式。
    
*   `server03` 可用的 `lisence`。
    
*   我录制的三段视频。
    
*   没有源码，请自行到网上搜索。
    

[[2022 夏季班]《安卓高级研修班 (网课)》月薪三万班招生中～](https://www.kanxue.com/book-section_list-84.htm)
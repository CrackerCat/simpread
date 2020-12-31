> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2NzUzNzk1Mw==&mid=2247484206&idx=2&sn=af3641c3499e6a42ba638be072ec93ac&chksm=cebb4660f9cccf7614f7684b1944973889ec785921713efaca44561850ab8f17a6715befcbfb&scene=21#wechat_redirect)

**一、演示软硬件环境**  

      PC 配置: Window10 64bit

      手机型号: pixel 2

      手机代号:walleye

**二、配置 adb 和 fasboot**  

      1. 从以下地址下载 windows 系统运行的 android sdk platform-tools 压缩包。

   下载地址:_https://dl.google.com/android/repository/platform-tools-latest-windows.zip_

2. 解压压缩包，将 adb 和 fastboot 所在目录配置到环境变量。

      3. 在 Windows 终端测试 adb 和 fastboot 命令，检查是否配置正常。

**三、下载工厂镜像刷机包**  

      1. 从以下地址下载 pixel 2 手机的工厂刷机包镜像，请选择 Android 10 的下载。下载地址:https://developers.google.cn/android/images#walleye

      2. 解压下载的刷机包镜像文件，如下图所示:

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5433EbW6ic6fzDiceyEicPe0kTjR2LJkluMIpMR05GB5erhyKSLUgFIlJcibLqlfVq2QiajeyL2pdBbTU2Fg/640?wx_fmt=jpeg)

    以上截图文件中, flash-all.bat 就是 pixel 2 手机在 windows 平台的 fastboot 刷机脚本。原脚本内容如下:

```
PATH=%PATH%;"%SYSTEMROOT%\System32"
fastboot flash bootloader bootloader-walleye-mw8998-002.0081.00.img
fastboot reboot-bootloader
ping -n 5 127.0.0.1 >nul
fastboot flash radio radio-walleye-g8998-00020-1912122233.img
fastboot reboot-bootloader
ping -n 5 127.0.0.1 >nul
fastboot -w update image-walleye-qq2a.200501.001.a3.zip
echo Press any key to exit...
pause >nul
exit

```

     为了让刷机脚本更智能一些, 将脚本修改为如下:

```
:: 配置脚本运行环境变量，配置之后就能找到adb和fasboot命令
PATH=%PATH%;"%SYSTEMROOT%\System32"
:: 手机开启调试连接到电脑情况下,该命令进入手机fastboot刷机模式
adb reboot bootloader
ping -n 5 127.0.0.1 >nul
:: 手机解锁
fastboot oem unlock
ping -n 5 127.0.0.1 >nul
:: 刷入bootloader镜像
fastboot flash bootloader bootloader-walleye-mw8998-002.0081.00.img
:: 重启进入到bootloader
fastboot reboot-bootloader
ping -n 5 127.0.0.1 >nul
:: 刷入基带镜像
fastboot flash radio radio-walleye-g8998-00020-1912122233.img
:: 重启进入bootloader
fastboot reboot-bootloader
ping -n 5 127.0.0.1 >nul
:: 刷入手机系统,image-walleye-qq2a.200501.001.a3.zip文件中包含了编译系统产生的各种镜像文件
fastboot -w update image-walleye-qq2a.200501.001.a3.zip
echo Press any key to exit...
pause >nul
exit

```

**四、刷机**  

 1. 手机数据线连接电脑, 打开手机调试模式，在手机设置 "开发者选项" 中开启 "USB 调试" 和 "OEM 解锁" 选项，如下图所示 :

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5433EbW6ic6fzDiceyEicPe0kTjRGnOBGOKdd5rhqiaa52xPqHAh816oTocVSmOp4l5gJxN1eOXcia9vTMAA/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5433EbW6ic6fzDiceyEicPe0kTjRAuuLBY06CKV68xkau65UqaibVrSRdibQ9iafHEY8XE3DZUG5DNVa5bqUQ/640?wx_fmt=jpeg)

2. 终端执行 adb devices 命令，查看手机是否正常连接到电脑。  

3. 手机能通过 adb 命令进行连接之后，点击修改过的工厂镜像中的 "flash-all.bat" 脚本执行刷机。  

4. 脚本执行过程中，如果手机未解锁，在 fastboot 模式下手机会提示 bootloader 解锁，请按照手机界面解锁提示操作。

**五、如何刷入修改编译的系统**

 1. 解压工厂镜像中的手机系统镜像压缩包，如图所示:

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5433EbW6ic6fzDiceyEicPe0kTjR12nNn5FG8pepoRAnZkXCZbU3rlcvESOaL2SJkjwMOEfBUxO6FJMVPw/640?wx_fmt=jpeg)

  2. 进入源码编译输出目录, 比如 pixel 2 手机编译目录 out/target/product/walleye。在该目录中找到和工厂手机镜像中文件同名文件，比如 boot.img、system.img 等。将同名文件拷贝覆盖到工厂手机镜像解压目录中的文件，然后压缩为同名的工厂手机镜像文件。点击 "flash-all.bat" 完成刷机操作。

如果你喜欢该文章，请关注我的公众号:

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5433EbW6ic6fzDiceyEicPe0kTjRnyKCFcMFoicc7APewgUGMuS7BRMiaiaWFrFvjTuUFd4TG2oD2taRVaUBQ/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/LtmuVIq6tF3FUeiaXU7G3N3DpMgKhWnsYJo371deoRZibcibTtz5xL6GphibDxAbBIHyxn5nTk2zHIsFGPxhTdlQ5w/640?wx_fmt=jpeg)
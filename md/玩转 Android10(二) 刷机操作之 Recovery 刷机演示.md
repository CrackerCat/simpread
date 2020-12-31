> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2NzUzNzk1Mw==&mid=2247484206&idx=3&sn=4e0eda4df787e48a1b76a9cd1a8dcf98&chksm=cebb4660f9cccf7627a5ed8c715002b6700abeef32bed33059bac3f64b313bf4330ad4026af3&scene=21#wechat_redirect)

**一、演示环境准备**  

      PC 环境:

                 Windows10 64bit

      手机设备:

                 oneplus 3  

      为了保证刷机成功，请将 oneplus 3 官方系统升级到 Android9 及以上系统。

     提前配置好 adb 和 fastboot 命令环境，确保 Windows 终端通过 adb 命令能找到设备。

  
**二、刷机包准备**  

 下载 oneplus 3 的 twrp  recovery 镜像, 下载地址如下:

```
https://dl.twrp.me/oneplus3/twrp-3.4.0-0-oneplus3.img.html

```

      下载 oneplus 3 的 lineageOS  Android 10  卡刷包镜像, 下载地址如下:  

```
https://download.lineageos.org/oneplus3

```

以下是我下载的刷机包文件, 如下图所示:   

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430HpkFIRvrbTB68PwHwicZh5q6ouJlerNlfcHlR2bTm4UHchAOgSKZtbFyVp7jGQHQrvOtrjQ9HibCA/640?wx_fmt=jpeg)

**三、刷机**

   **1. 手机 OEM 解锁**

         （1）在手机 "开发者选项" 中打开 "OEM 解锁" 开关

         （2）手机连接电脑之后，执行如下命令解锁 bootloader       

```
adb reboot bootloader
fastboot devices
fastboot oem unlock

```

  2. 手机解锁成功之后，安装 twrp recovery 工具

         (1) 手机开机情况下执行 adb reboot bootloader 进入 fastboot 模式或者关机状态下同时长按 “音量加键”+“开机键”。  

        （2）执行如下命令刷入 oneplus3 twrp recovery 镜像。

```
fastboot flash recovery E:\studyspace\Android10\测试刷机包\oneplus3\twrp-3.4.0-0-oneplus3.im

```

          **(3) 执行如下命令重启手机, 同时长按 “音量减键”+“开机键” 进入刚刷入的 recovery。**

```
fastboot reboot

```

        刷成功进入 recovery 之后如图所示:  

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430HpkFIRvrbTB68PwHwicZh5KicDiafSr6ibwLibXxYQZDYn2CfwrCrIylMYXNSZeiaFKn4nLQEhfd32IXg/640?wx_fmt=jpeg)

        ** (4) 手机双清操作**  

              在 recovery 主界面，顺序操作 "Wipe->Advanced Wipe", 再清理界面，选择所有的分区，然后点击 "Swipe to Wipe" 完成手机格式化。

        **(5) 刷机方式 1: 使用 adb sideload 方式刷机**

             在 recovery 主界面，依次操作 "Advanced->ADB Sideload->Swipe to Start Sideload"。然后在终端执行如下命令等待 adb sideload 方式刷机完成, 完成之后开机就可以看到刷的系统了。

```
adb sideload  E:\studyspace\Android10\测试刷机包\oneplus3\lineage-17.1-20201221-nightly-oneplus3-signed.zip

```

     如下图所示:

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430HpkFIRvrbTB68PwHwicZh5Yib0AYI2eLI0n9hIRHM8mxQXu7xOqic3ae5nTko3nczWf5icmJJlJ7tFw/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430HpkFIRvrbTB68PwHwicZh5x6wg7VrIUK0OsWpEA2jf3s2B30ibveshEib1Xjr8jWLJ8nd7mhe9W57Q/640?wx_fmt=jpeg)

**（6) 刷机方式 2: 在 recovery 里面选择刷机包的方式刷机**

            执行如下 adb push 命令将刷机包拷贝到手机外置 sdcard 上面:

```
adb push E:\studyspace\Android10\测试刷机包\oneplus3\lineage-17.1-20201221-nightly-oneplus3-signed.zip  /sdcard/update.zip

```

              在 recovery 主界面，依次操作 "Install"进入刷机包选择页面，在该界面选择"update.zip"文件之后，操作"Swipe to confirm Flash" 完成刷机操作，刷完之后重启手机就可以看到刷的手机系统了。

       **关注下面的公众号，文章更新第一时间抢先看。**

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5433EbW6ic6fzDiceyEicPe0kTjRnyKCFcMFoicc7APewgUGMuS7BRMiaiaWFrFvjTuUFd4TG2oD2taRVaUBQ/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/LtmuVIq6tF3FUeiaXU7G3N3DpMgKhWnsYJo371deoRZibcibTtz5xL6GphibDxAbBIHyxn5nTk2zHIsFGPxhTdlQ5w/640?wx_fmt=jpeg)
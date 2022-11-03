> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [o0xmuhe.github.io](https://o0xmuhe.github.io/2022/09/02/HW-OTA-unpack/)

> 步骤 unzip 解开 OTA 包

[](#步骤 "步骤")步骤
--------------

### [](#unzip解开OTA包 "unzip解开OTA包")unzip 解开 OTA 包

![](https://my-own-image.oss-cn-beijing.aliyuncs.com/note_pic/image-20220825111641389.png)

我们的目标在 update_sd_base.zip 里，其他部分咨询了是一些出厂带的 APP，比如里面就看到了今日头条 抖音啥的。

直接解开 update_sd_base.zip 到下一步

### [](#从UPDATE-APP提取SYSTEM "从UPDATE.APP提取SYSTEM")从 UPDATE.APP 提取 SYSTEM

直接用 [https://github.com/jenkins-84/split_updata.pl/blob/master/splitupdate](https://github.com/jenkins-84/split_updata.pl/blob/master/splitupdate) 来分割就行

![](https://my-own-image.oss-cn-beijing.aliyuncs.com/note_pic/image-20220825111723396.png)

### [](#unpack-erofs "unpack erofs")unpack erofs

#### [](#方法1-simg2img然后挂在erofs-kernel-5-4 "方法1: simg2img然后挂在erofs(kernel 5.4)")方法 1: simg2img 然后挂在 erofs(kernel 5.4)

```
~/android-simg2img/simg2img SYSTEM.img system1.img
sudo mount -t erofs system1.img 1 -oloop
```

尝试读文件的时候发现报错

![](https://my-own-image.oss-cn-beijing.aliyuncs.com/note_pic/image-20220825111833524.png)

dmesg 发现 :

![](https://my-own-image.oss-cn-beijing.aliyuncs.com/note_pic/image-20220825111906816.png)

```
~/extracotr/erofs_tool.py extract --verify-zip system1.img harmony_system
```

[](#参考 "参考")参考
--------------

[https://zhuanlan.zhihu.com/p/60617375](https://zhuanlan.zhihu.com/p/60617375)

[https://github.com/jenkins-84/split_updata.pl](https://github.com/jenkins-84/split_updata.pl)

[https://github.com/srlabs/extractor](https://github.com/srlabs/extractor)
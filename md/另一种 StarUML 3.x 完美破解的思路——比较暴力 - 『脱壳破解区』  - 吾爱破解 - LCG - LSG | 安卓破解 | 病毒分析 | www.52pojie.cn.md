> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/forum.php?mod=viewthread&tid=796683&highlight=starUML)  ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) 吾名空白 _ 本帖最后由 吾名空白 于 2018-9-15 16:29 编辑_  
我不知道这种算不算重复发帖，但是我觉得这是另一种思路，应该不算重复发帖吧![](https://static.52pojie.cn/static/image/smiley/default/45.gif)  

背景
==

在昨天，我发布了一篇又长又臭的 [StarUML 3.x 完美破解方案](https://www.52pojie.cn/thread-796355-1-1.html)

然后今天早上一起床就想到，啊～好麻烦啊！能不能让完美再简单点？

好的，然后我又开始了折腾（真丶生命在于折腾![](https://static.52pojie.cn/static/image/smiley/default/4.gif)）

我的想法
====

这一次，我要先从`about-dialog.js`开始分析

（`about-dialog.js`路径：解压`app.asar`后`app/src/dialogs/about-dialog.js`）

正所谓：你要什么我就给什么才是最方便的🌚

正文
==

下面开始正式分析破解

找出关于中要显示的信息的数据来源
----------------

首先我们要找出要在关于中显示的信息来源

`about-dialog.js`中第 45-71 行：

```
// set license info
if (app.licenseManager.getStatus() === true) {
    var info = app.licenseManager.getLicenseInfo()
    var licenseTypeName = 'Unknown'
    switch (info.licenseType) {
    case 'PS':
      licenseTypeName = 'Personal'
      break
    case 'CO':
      licenseTypeName = 'Commercial'
      break
    case 'ED':
      licenseTypeName = 'Educational'
      break
    case 'CR':
      licenseTypeName = 'Classroom'
      break
    }
    $license.html('Licensed to ' + info.name)
    $licenseType.html(licenseTypeName + ' License')
    $quantity.html(info.quantity + ' User(s)')
    $crackedAuthor.html('Cracked by ' + info.crackedAuthor)
  } else {
    $license.html('UNREGISTERED')
  }
  return dialog
}

```

可以看出，它首先会先调用`license-manager.js`中的`getStatus()`方法判断程序的注册状态。那现在我们去`license-manager.js`看一下`getStatus()`这个方法：

```
/**
 * Get Registration Status
 * home.php?mod=space&uid=155549 {string}
 */
getStatus () {
  return status
}

```

嗯...... 很简单的一个方法，我是试过直接在这里设置`"true"`(注意！这里是返回字符串)，但是没什么用，我就没改这里了。那么这里到底有什么用呢？大家看它返回的那个变量：`status`，这是我们唯一从这里得到的信息，我们全文搜索一下`status`，可以找到一个`setStatus(...)`的方法。我修改了一下让它总是设置为`true`，代码如下：

```
function setStatus (licenseManager, newStat) {
  if (status !== newStat) {
    status = newStat
    licenseManager.emit('statusChanged', 'true') // status修改为'true',注意要带单引号
  }
}

```

好了，现在我们可以进入那个 if 语句了🌚。不难看出，接下来需要的数据都在变量`info`里，从这一句

```
var info = app.licenseManager.getLicenseInfo()

```

可以看出，它调用了`license-manager.js`中的`getLicenseInfo()`方法获取所需数据，我们去看一下`getLicenseInfo()`方法：

```
getLicenseInfo () {
    return licenseInfo
}

```

嗯..... 还是那么简洁，但是从昨天的文章我们已经知道这个`licenseInfo`的数据内容格式，他要的数据也是`licenseInfo`的数据。那么，我们直接模拟`licenseInfo`的数据即可：

```
getLicenseInfo () {
  licenseInfo = {
          name: "Reborn",
          product: "Reborn product",
          licenseType: "PS",
          quantity: "Reborn Quantity",
          timestamp: "1529049036",
          licenseKey: "It's Cracked!!",
          crackedAuthor: "Reborn"
        };
  return licenseInfo
}

```

好了，模拟成功，`about-dialog.js`那边应该能获取到`licenseInfo`的数据了。但是，仅仅是这样还不行哦！还有最重要的一点你们别忘了——我们还没破解！

破解注册
----

破解注册很简单，直接修改`license-manager.js`中的`checkLicenseValidity()`这个方法就好了。

修改后的代码如下：

```
checkLicenseValidity () {
  this.validate().then(() => {
    setStatus(this, true)
  }, () => {
    // 原来的代码，如果失败就会将状态设置成false
//       setStatus(this, false)
//       UnregisteredDialog.showDialog()

    //修改后的代码
    setStatus(this, true)
  })
}

```

注册成功！！
------

完成以上流程后应该就能成功**直接破解**了，**不用输入注册码**，并且这种方法破解后同样能在关于显示你自定义的破解信息！！一样完美～

![](https://attach.52pojie.cn/forum/201809/15/160925uawgxjr0haxtt0zh.png)

**staruml-about.png** _(82.6 KB, 下载次数: 3)_

[下载附件](forum.php?mod=attachment&aid=MTI0MDI3M3wwNTBhNDRlOXwxNjIzNzU3NzAzfDIxMzQzMXw3OTY2ODM%3D&nothumb=yes)

2018-9-15 16:09 上传

  
这种方法和昨天的比起来更简单，但是也更暴力。昨天的比较接近正常的验证流程，这种就有点爆破的味道了。这里给出另一种思路给大家参考，希望对大家有帮助。  
各位要是觉得对大家有帮助的话，给点热心，有免费评分的评分走一走，谢谢各位![](https://static.52pojie.cn/static/image/smiley/laohu/laohu33.gif)。  
![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)乌懂懂 

> [吾名空白 发表于 2018-9-18 21:03](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=21841041&ptid=796683)  
> 有具体情况吗？比如截图之类的？  
> npm 安装成功了吗？我是 npm install asar -g 就安装好可以用了

我早就安装了 npm 了， 你看看，返回错误：  error: unknown option `-v'  
Windows PowerShell  
版权所有 (C) Microsoft Corporation。保留所有权利。  
PS C:\Users\Administrator> npm -v  
5.6.0  
PS C:\Users\Administrator> npm install asar -g  
C:\Users\Administrator\AppData\Roaming\npm\asar -> C:\Users\Administrator\AppData\Roaming\npm\node_modules\asar\bin\asar.js  
+ [asar@0.14.3](mailto:asar@0.14.3)  
added 1 package and updated 2 packages in 18.779s  
PS C:\Users\Administrator> asar -version  
  error: unknown option `-v'  
PS C:\Users\Administrator>  
![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)吾名空白 

> [乌懂懂 发表于 2018-9-20 19:47](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=21866620&ptid=796683)  
> 我早就安装了 npm 了， 你看看，返回错误：  error: unknown option `-v'  
> Windows PowerShell

这个提示的话应该是安装成功了，不然就会报错 "xxx command not found" 之类的。这个我觉得是你后面带的参数错了，是 --version 才对（两条杠）![](https://avatar.52pojie.cn/data/avatar/000/06/82/17_avatar_middle.jpg)heiketian10  不错的东西  思路很好 ![](https://avatar.52pojie.cn/data/avatar/000/88/00/40_avatar_middle.jpg) shenbl201 厉害了~~![](https://static.52pojie.cn/static/image/smiley/laohu/laohu36.gif)![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)FENGMUTIAN  嘿嘿，前来学习 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) suilibin 来个打包好的文件呗 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) may_be_2018 过程很详细，谢谢分享![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)丿颠覆灬虎哥  不错，感谢分享 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) anhua123 感谢热心分享 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) VisualBoy 经过测试，十分好用，谢谢分享![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)游叶子明  有没有打包好的文件?
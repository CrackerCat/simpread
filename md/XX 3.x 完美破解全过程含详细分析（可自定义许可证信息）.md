> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-796355-1-1.html)

> 刚刚版主把我的帖子移到脱壳破解区了，吓得我赶紧又看了一下原创破解区的版规，发现我确实发错板块了，在这里说声抱歉，之前没有好好看版规 @云在天 [md]# 背景终于开始 ... XX 3.x 完美破解全过程......

 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) 吾名空白 _ 本帖最后由 swb8023 于 2019-6-6 18:29 编辑_  
刚刚版主把我的帖子移到脱壳破解区了，吓得我赶紧又看了一下原创破解区的版规，发现我确实发错板块了，在这里说声抱歉，之前没有好好看版规 @云在天  

背景
==

终于开始新的学期了，这学期我有个 UML 建模 的课程，上机作业就是要用到 StarUML 这个软件，但是...... 啊——为什么 StarUML 那么丑啊！！！而且，为什么这么难用啊！！！（马上开始怀疑是旧旧旧...<省略一万个 “旧”>... 版）

**（图片下次上机后附上）**

好吧，颜值帝的我无奈翻了一下官方，发现还机房的还真的是超级旧版，一进去[官网](http://staruml.io/)就能看到我最喜欢的暗黑风格😍。

  
![](https://attach.52pojie.cn/forum/201809/14/194516p5ewehdbqb3tehu9.png)

**staruml-officialwebsite.png** _(225.41 KB, 下载次数: 7)_

[下载附件](forum.php?mod=attachment&aid=MTIzOTYyMHxiNjgyZTcxZnwxNjIzNzU3NzI5fDIxMzQzMXw3OTYzNTU%3D&nothumb=yes)

2018-9-14 19:45 上传

  

果断下载来试用了一下，哇！这个操作比机房的体验好超级多了好吗😂

这软件开源，但是却收费，穷得要死（铁公鸡）的我肯定是不会付费用的了 (其实是想顺便看下这个软件的验证逻辑)，于是研究起了破解之道～

关于 StarUML 的破解原理
================

> 参考：[https://www.jianshu.com/p/b6b1f6ad0bd6](https://www.jianshu.com/p/b6b1f6ad0bd6)

StarUML 是用 nodejs 写的，确切的说是用 [Electron 前端框架](https://github.com/electron/electron) 写的。

新版本中所有的 StarUML 源代码是通过 [asar](https://github.com/electron/asar) 工具打包而成，确切的代码位置在`%LOCALAPPDATA%\Programs\StarUML\resources\app.asar`。

我们可以通过 asar 工具解压修改达到破解目的，关于 asar 工具使用可看本文[附录 1](https://www.52pojie.cn/forum.php?mod=viewthread&tid=796355&page=1#21794892_附录1：关于app.asar的解压与重打包)。

> **提取 app.asar**  
> 下载的 `StarUML.app`，右键显示包内容  
> 进入`Contents/Resources/`  
> 把`app.asar`复制出来

为什么网上那么多破解教程我还要写？
=================

> 参考： [Mac StarUML 3.0 破解](https://www.jianshu.com/p/b6b1f6ad0bd6)

那...... 当然不是为了存档！

相信搜过 StarUML3 破解的朋友都知道，想要破解就修改`app.asar`解压出来的`app/src/engine/license-manager.js`，把其中的`checkLicenseValidity`函数修改成：

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

然后重新打包`app.asar`放回`StarUML.app/Contents/Resources/`，破解成功了。

那么，为什么网上那么多破解教程我还要写一个教程呢？

因为我是个完美主义者，我发现网上的教程都不能达到完美破解的效果——我的注册信息怎么没法显示（自定义）！！！于是我就着手研究起`app.asar`解压出来的代码了🌚

开始分析
====

破解的核心：`license-manager.js`
--------------------------

> 参考： [绕过 StarUML3 正版验证，去除水印](http://dxkite.cn/2018/06/15/bypass-staruml-3/)
> 
> 破解的核心还是`license-manager.js`这个跑不掉，那么我们先从这个文件开始分析。

### 根据注册码验证流程分析

#### 流程图

在这之前，我们先想一下正常的注册码验证流程是什么：

![](https://attach.52pojie.cn/forum/201809/14/195906tjjsxu4494c2pt77.png)

**正常的注册码验证流程. png** _(26.6 KB, 下载次数: 3)_

[下载附件](forum.php?mod=attachment&aid=MTIzOTYyOXwxODRjOGJjMHwxNjIzNzU3NzI5fDIxMzQzMXw3OTYzNTU%3D&nothumb=yes)

2018-9-14 19:59 上传

  

#### 打开 app

可以看到，我们首先要`打开 app`，对应的代码是：

```
htmlReady () {
    this.projectManager.on('projectSaved', (filename, project) => {
      var val = Math.floor(Math.random() * (1.0 / LICENSE_CHECK_PROBABILITY))
      if (val === 0) {
        this.checkLicenseValidity()
      }
    })
}

appReady () {
    this.checkLicenseValidity()
}

```

可以看到，这两个方法中都会用到`this.checkLicenseValidity()`，看英文意思明显是检查许可证是否有效，所以把`checkLicenseValidity`这个方法修改一下就可以破解了（修改方法看[上面](https://www.52pojie.cn/forum.php?mod=viewthread&tid=796355&page=1#21794892_为什么网上那么多破解教程我还要写？))

**但是，这里我们先不修改！！！因为修改后没法弹出无法弹出输入注册码的窗口，就无法完美破解了！！**

#### 输入注册码

接下来我们来看一下**输入注册码**也就是**注册**的逻辑：

```
  register (licenseKey) {
    return new Promise((resolve, reject) => {
      $.post(app.config.validation_url, {licenseKey: licenseKey})
        .done(data => {
          var file = path.join(app.getUserPath(), '/license.key')
          fs.writeFileSync(file, JSON.stringify(data, 2))
          licenseInfo = data
          setStatus(this, true)
          resolve(data)
        })
        .fail(err => {
          setStatus(this, false)
          if (err.status === 499) { /* License key not exists */
            reject('invalid')
          } else {
            reject()
          }
        })
    })
  }

```

解释一下，这个`register`函数需要传入的参数`licenseKey`就是输入的注册码，然后用 POST 请求 balabala，请求成功后返回一个`data`，然后在 StarUML 的配置信息目录新建一个`license.key`的文件，里面包含着所有许可证信息。可以看到，这个许可证信息是**用 JSON 格式储存的**，而请求后获取的`data`就是许可证信息的内容，**那么我们是不是只要模拟已经获取到了许可证信息是不是就 ok 了？**

那么问题又来了，我们怎么知道`data`的内容格式呢？

我们不妨参考一下 StarUML2 的破解：[https://www.jianshu.com/p/0c49ebf342e0](https://www.jianshu.com/p/0c49ebf342e0)

在 StarUML2 的破解中我们可以看到如下代码：

```
return {
    name: "0xcb",
    product: "StarUML",
    licenseType: "vip",
    quantity: "bbs.chinapyg.com",
    licenseKey: "later equals never!"
};

```

我斗胆猜测这就是`data`的内容格式（也就是许可证信息的内容格式）！！！

我们广东人常常会说一句话：讲多无谓，行动最实际。我们下面就来尝试一下！

既然是本地模拟，那当然要把网络请求去掉然后模拟`data`的内容，修改整理后得到以下代码：

```
  register (licenseKey) {
    return new Promise((resolve, reject) => {
          var data = {
            name: "Reborn",
            product: "Reborn product",
            licenseType: "PS",
            quantity: "Reborn Quantity",
            timestamp: "1529049036",
            licenseKey: "It's Cracked!!",
            crackedAuthor: "Reborn"
          }
          var file = path.join(app.getUserPath(), '/license.key')
          fs.writeFileSync(file, JSON.stringify(data, 2))
          licenseInfo = data
          setStatus(this, true)
          resolve(data)
    })
  }

```

到这里许可证信息应该是模拟成功了，看 StarUML 的配置信息目录下应该能看到一个`license.key`的文件。输完注册码后就要验证注册码是否有效，接下来我们看验证许可证信息模块。

> **这里要注意一下：**`licenseType`的设置是固定几个参数的，在这里我设置为`PS`，之前设置`Reborn Personal`导致关于那里显示`Unknown License`。关于这个大家可以去看看`app/src/dialogs/about-dialog.js`。

#### 验证许可证信息

先来看下代码吧：

```
  validate () {
    return new Promise((resolve, reject) => {
      try {
        // Local check
        var file = this.findLicense()
        if (!file) {
          reject('License key not found')
        } else {
          var data = fs.readFileSync(file, 'utf8')
          licenseInfo = JSON.parse(data)
          var base = SK + licenseInfo.name +
            SK + licenseInfo.product + '-' + licenseInfo.licenseType +
            SK + licenseInfo.quantity +
            SK + licenseInfo.timestamp + SK
          var _key = crypto.createHash('sha1').update(base).digest('hex').toUpperCase()
          if (_key !== licenseInfo.licenseKey) {
            reject('Invalid license key')
          } else {
            // Server check
            $.post(app.config.validation_url, {licenseKey: licenseInfo.licenseKey})
              .done(data => {
                resolve(data)
              })
              .fail(err => {
                if (err && err.status === 499) { /* License key not exists */
                  reject(err)
                } else {
                  // If server is not available, assume that license key is valid
                  resolve(licenseInfo)
                }
              })
          }
        }
      } catch (err) {
        reject(err)
      }
    })
  }

```

可以看到验证分两部分：先`本地验证`再进行`网络验证`，网络验证成功后返回许可证信息。

##### 本地验证

我们一步步来，先处理本地验证：

```
// Local check
var file = this.findLicense()
if (!file) {
    reject('License key not found')
} else {
    var data = fs.readFileSync(file, 'utf8')
    licenseInfo = JSON.parse(data)
    var base = SK + licenseInfo.name +
    SK + licenseInfo.product + '-' + licenseInfo.licenseType +
    SK + licenseInfo.quantity +
    SK + licenseInfo.timestamp + SK
    var _key = crypto.createHash('sha1').update(base).digest('hex').toUpperCase()
    if (_key !== licenseInfo.licenseKey) {
        reject('Invalid license key')
    }
    .....
}

```

本地验证真正开始验证的地方是从第 8 行开始的。通过文件`license.key`中的部分许可证信息计算出一个注册码然后和许可证信息中的`licenseKey`进行比较，如果不相等那就返回注册码失效（从这里的`licenseInfo`调用也可以看出许可证信息包含了什么，与上一步的`data`验证）。

既然如此，我们就不让它比较嘛，我们直接返回成功不就得了！

好的，**第 8 行之后的都删掉**。

##### 网络验证

由于返回的逻辑不在本地验证，所以我们还要分析网络验证：

```
// Server check
$.post(app.config.validation_url, {licenseKey: licenseInfo.licenseKey})
    .done(data => {
        resolve(data)
    })
    .fail(err => {
        if (err && err.status === 499) { /* License key not exists */
           reject(err)
        } else {
            // If server is not available, assume that license key is valid
            resolve(licenseInfo)
        }
})

```

又是一个 POST 请求！又是返回`data`！认真看了一下还是与上一步`输入注册码`同样的网络请求！好了，多的我就不说了👀，大家应该知道怎么办，可以同上一步一样模拟`data`的内容，然后`resolve(data)`返回即可。

**但是，这里我选择直接返回`licenseInfo`**，为什么呢？

大家看一下本地验证，`licenseInfo`的内容是从文件`license.key`中转换而来，文件`license.key`的内容是上一步的 POST 请求而来，嗯.... 不用我多说了吧！

整理代码后如下：

```
  validate () {
    return new Promise((resolve, reject) => {
      try {
        // Local check
        var file = this.findLicense()
        if (!file) {
          reject('License key not found')
        } else {
          var data = fs.readFileSync(file, 'utf8')
          licenseInfo = JSON.parse(data)
          resolve(licenseInfo)
        }
      } catch (err) {
        reject(err)
      }
    })
  }

```

#### 注册成功！！

完成以上流程后应该就能成功实现**输入任意注册码**都可以破解了，并且这种方法破解后还能在关于显示你自定义的破解信息！！完美～

![](https://attach.52pojie.cn/forum/201809/14/195237msrpy22pmz2spmxn.png)

**staruml-about.png** _(24.18 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MTIzOTYyM3xkZTA3MTE0NHwxNjIzNzU3NzI5fDIxMzQzMXw3OTYzNTU%3D&nothumb=yes)

2018-9-14 19:52 上传

  

附录 1：关于 app.asar 的解压与重打包
========================

> 参考：[https://www.jianshu.com/p/b6b1f6ad0bd6](https://www.jianshu.com/p/b6b1f6ad0bd6)

安装 asar
-------

```
sudo npm install -g asar

```

解压 app.asar
-----------

```
asar extract app.asar app

```

重新打包 app.asar
-------------

```
asar pack app app.asar

```

附录 2：自定义关于界面
============

大家做完上面的操作肯定会发现，为什么你们的关于没有`Cracked By XXX`的字样，那是因为我修改了`关于`的 GUI，在前面并没有说，在这里我特别分出来写一下，算是个小彩蛋吧！

找到代码所在文件
--------

首先我们要找到`关于`这个界面的代码文件，经过分析和查找，我找到了`about-dialog.js`和`about-dialog.html`

至于怎么找到的，大家看一下`about-dialog.js`的这句代码就知道了：

```
const aboutDialogTemplate = fs.readFileSync(path.join(__dirname, '../static/html-contents/about-dialog.html'), 'utf8')

```

开始分析
----

### about-dialog.html

我们先来看一下 html 吧，在这个文件中大家可以很明显看到这么一段：

```
<div>{{metadata.name}}</div>
<div></div>
<div></div>
<div></div>
<br>
<div>{{metadata.copyright}}</div>
<div><b>Version {{metadata.version}}</b></div>
<div><a href="#">Third party softwares</a></div>

```

对比一下关于的界面，是不是觉得刚好对上了🌚

好的那么接下来我们加一个破解者的信息：

```
......
<div></div>
<div></div>
<br>
......

```

这里第 3 行就是我添加的信息，其中`font-size`是调整字体大小，`class="crackedAuthor"`这个类选择器名`crackedAuthor`要记住，等下要用到。

接下来我们看`about-dialog.js`

### about-dialog.js

在这个 js 文件里面，我们可以看到这几行代码：

```
......
var $license = $dlg.find('.license')
var $licenseType = $dlg.find('.licenseType')
var $quantity = $dlg.find('.quantity')
......
$license.html('Licensed to ' + info.name)
$licenseType.html(licenseTypeName + ' License')
$quantity.html(info.quantity + ' User(s)')
......

```

我们来分析一下这几行代码吧！

`$dlg.find('.xxx')`的意思是搜索所有类选择器名为`.xxx`的元素。  
`$xxx.html('abc')`的意思是设置`xxx`元素的内容为`abc`。

说到这里大家应该知道怎么修改了吧！这是我修改的代码：

```
......
var $license = $dlg.find('.license')
var $licenseType = $dlg.find('.licenseType')
var $quantity = $dlg.find('.quantity')
var $crackedAuthor = $dlg.find('.crackedAuthor')
......
$license.html('Licensed to ' + info.name)
$licenseType.html(licenseTypeName + ' License')
$quantity.html(info.quantity + ' User(s)')
$crackedAuthor.html('Cracked by ' + info.crackedAuthor)
......

```

### 修改成功

保存好代码[重新打包`app.asar`](https://www.52pojie.cn/forum.php?mod=viewthread&tid=796355&page=1#21794892_重新打包app.asar)就可以看到修改后的关于界面啦！

各位要是觉得对大家有帮助的话，给点热心，有免费评分的评分走一走（其实我非常想进入高级会员区学习，所以求点热心哈哈哈），谢谢各位![](https://static.52pojie.cn/static/image/smiley/laohu/laohu33.gif)。![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)吾名空白 _ 本帖最后由 吾名空白 于 2018-9-14 22:02 编辑_  

> [marine153 发表于 2018-9-14 21:56](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=21795927&ptid=796355)  
> 呦呵，这么巧，我也是软件工程的，我们对外都说自己是计算机的，怕别人不知道软件工程具体是干啥的，哈哈 ...

哈哈，对于这个，别人问我我都是直接说读计算机的，追问计算机什么方向之类的我才说软件工程，都懒得一个个解释软件工程是干嘛的了![](https://static.52pojie.cn/static/image/smiley/default/45.gif)![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)吾名空白 _ 本帖最后由 吾名空白 于 2018-9-14 22:03 编辑_  

> [marine153 发表于 2018-9-14 21:56](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=21795927&ptid=796355)  
> 呦呵，这么巧，我也是软件工程的，我们对外都说自己是计算机的，怕别人不知道软件工程具体是干啥的，哈哈 ...

没有，我是大三狗，我们大三才有这门课，这两天放假正好有空研究一下这个东西，反正以后也要用![](https://static.52pojie.cn/static/image/smiley/default/4.gif)![](https://avatar.52pojie.cn/data/avatar/000/15/24/83_avatar_middle.jpg)不苦小和尚 没有成品吗？看的人眼花缭乱的 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) dapeng2113 看着有点乱啊 - - ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) 吾名空白

> [dapeng2113 发表于 2018-9-14 20:27](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=21795065&ptid=796355)  
> 看着有点乱啊 - -

我从源码上分析的![](https://static.52pojie.cn/static/image/smiley/default/2.gif)，可能贴的代码会有点多，其实就是去掉网络验证和修改本地验证逻辑并且猜测他的许可证信息的存储格式 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) dapeng2113

> [吾名空白 发表于 2018-9-14 20:37](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=21795160&ptid=796355)  
> 我从源码上分析的，可能贴的代码会有点多，其实就是去掉网络验证和修改本地验证逻辑并且猜测他 ...

反手一个赞。![](https://static.52pojie.cn/static/image/smiley/mogu/hh.gif)![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)吾名空白

> [不苦小和尚 发表于 2018-9-14 20:20](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=21795000&ptid=796355)  
> 没有成品吗？看的人眼花缭乱的

我是在 Mac 上测试的，没有在 Windows 上测试过，所以没放成品出来，不过我记得 Windows 也是可以这样破解的![](https://static.52pojie.cn/static/image/smiley/default/4.gif) ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) marine153 楼主这么巧啊，我们这学期也学 UML 软件建模，你也是计算机专业的吧![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)吾名空白

> [marine153 发表于 2018-9-14 21:43](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=21795806&ptid=796355)  
> 楼主这么巧啊，我们这学期也学 UML 软件建模，你也是计算机专业的吧

对啊，我也是计算机专业的，我是软件工程的，你咧？![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)marine153

> [吾名空白 发表于 2018-9-14 21:48](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=21795851&ptid=796355)  
> 对啊，我也是计算机专业的，我是软件工程的，你咧？

呦呵，这么巧，我也是软件工程的，我们对外都说自己是计算机的，怕别人不知道软件工程具体是干啥的，哈哈，楼主不会也是大二的吧，我们是大二学这门课的
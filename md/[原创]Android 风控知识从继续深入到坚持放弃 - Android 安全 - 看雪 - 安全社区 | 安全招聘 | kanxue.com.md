> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-279619.htm)

> [原创]Android 风控知识从继续深入到坚持放弃

[原创]Android 风控知识从继续深入到坚持放弃

1 小时前 235

### [原创]Android 风控知识从继续深入到坚持放弃

* * *

继续前几篇的内容，最近研究国内几大电商 App 以及几大风控 sdk 有了一些深入了解，以及有一些有趣的检测方式的了解，本文主要给大家讲解一下我遇到的一些的奇葩检测方式。

有兴趣的朋友也可以了解一下前几篇的内容：  
· 常见参数讲解：[https://bbs.kanxue.com/thread-265169.htm](https://bbs.kanxue.com/thread-265169.htm)  
·Binder 黑科技：[https://bbs.kanxue.com/thread-278364.htm](https://bbs.kanxue.com/thread-278364.htm)  
· 设备指纹讲解：[https://bbs.kanxue.com/thread-275202.htm](https://bbs.kanxue.com/thread-275202.htm)

对了，想起来有些事情跟大家讲一下，本人既不卖课也不卖教程，各位不要再来找我要课了，不打算出书也不打算卖课，都是纯分享给新手解惑！

出处：某宝、某付宝 SDK  
奇葩程度：666 (三星中评)  
![](https://bbs.kanxue.com/upload/attach/202311/862224_3M82DRWQGQ8P39B.webp)

以上代码已被学习用于我公司的检测逻辑（感谢大厂的支招！）

###### 复现代码：

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p></td><td><p><code>/</code><code>/</code> <code>检测手机是否有锁屏密码</code></p><p><code>/</code><code>/</code> <code>Write By: Liankong xhew.new@qq.com</code></p><p><code>public static boolean isHasUnlockPassword(Context context) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>KeyguardManager km </code><code>=</code> <code>(KeyguardManager) context.getSystemService(Context.KEYGUARD_SERVICE);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code> <code>km.isDeviceSecure() &amp;&amp; km.isKeyguardSecure();</code></p><p><code>}</code></p></td></tr></tbody></table>

出处：某多  
奇葩程度：66666（五星好评）  
=== 此处应有图，但是因为不方便贴出来 (手动滑稽) ===

###### 复现代码：

![](https://bbs.kanxue.com/upload/attach/202311/862224_MEBNFE4ZU9M9TVH.webp)

###### 获取远程 binder 接口：

![](https://bbs.kanxue.com/upload/attach/202311/862224_7JUGGPNPMURJ7US.webp)  
因为研究用只做了高版本的支持，低版本应该改一下 getRemoteBinder 的对 IBinder 的获取方式

出处：某多、某东  
奇葩程度：666（一般吧）  
![](https://bbs.kanxue.com/upload/attach/202311/862224_98XH2U7REQAAYKT.webp)  
因为高版本 SELinux 限制了无法读取 / system/build.prop，但是有很多模拟器或者比较弱的黑产还是采用修改 build.prop 重启的方式修改数据。  
![](https://bbs.kanxue.com/upload/attach/202311/862224_UENUUPQQBCT4EPT.webp)  
而且 SELinux 并未先对该文件的 stat/statfs 访问，所以可以通过读取改文件的修改时间判断是否修改（正常通过编译出来的 build.prop 的修改时间要么是 1970 年 1 月要么是 2009 年 1 月 1 日）

###### 复现代码：

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p></td><td><p><code>/</code><code>/</code> <code>检测build.prop是否安全</code></p><p><code>/</code><code>/</code> <code>Write By: Liankong xhsw.new@qq.com</code></p><p><code>public static boolean isBuildPropSecure() {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>File</code> <code>file</code> <code>=</code> <code>new </code><code>File</code><code>(</code><code>"/system/build.prop"</code><code>);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code> <code>file</code><code>.lastModified() </code><code>/</code> <code>1000</code> <code>%</code> <code>1000</code> <code>=</code><code>=</code> <code>0</code><code>;</code></p><p><code>}</code></p></td></tr></tbody></table>

出处：某多、某某（编不下去了）  
奇葩程度：66  
![](https://bbs.kanxue.com/upload/attach/202311/862224_CGWWEPAE52J8Q6V.webp)

###### 复现代码：

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p><p>11</p><p>12</p><p>13</p><p>14</p></td><td><p><code>/</code><code>/</code> <code>获取手机上的应用市场列表</code></p><p><code>/</code><code>/</code> <code>Write By: Liankong xhsw.new@qq.com</code></p><p><code>public static String[] getAppMarketList(Context context) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>PackageManager pm </code><code>=</code> <code>context.getPackageManager();</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>Intent intent </code><code>=</code> <code>new Intent();</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>intent.setAction(</code><code>"android.intent.action.MAIN"</code><code>);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>intent.addCategory(</code><code>"android.intent.category.APP_MARKET"</code><code>);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>List</code><code>&lt;ResolveInfo&gt; </code><code>list</code> <code>=</code> <code>pm.queryIntentActivities(intent);</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>String res[] </code><code>=</code> <code>new String[</code><code>list</code><code>.size()];</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>for</code><code>(</code><code>int</code> <code>i </code><code>=</code> <code>0</code><code>;i &lt; </code><code>list</code><code>.size(); i</code><code>+</code><code>+</code><code>) {</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>res[i] </code><code>=</code> <code>list</code><code>.get(i).activityInfo.packageName;</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>}</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code> <code>res;</code></p><p><code>}</code></p></td></tr></tbody></table>

先讲到这里吧，各位大厂手下留情，只为了让广大新手学习！同时，欢迎各位来交流，QQ：2928455383

[[CTF 入门培训] 顶尖高校博士及硕士团队亲授《30 小时教你玩转 CTF》，视频 + 靶场 + 题目！助力进入 CTF 世界](http://www.kanxue.com/book-brief-170.htm#h3a6WRhDT9Q_3D)

最后于 53 分钟前 被恋空编辑 ，原因： 添加内容

返回
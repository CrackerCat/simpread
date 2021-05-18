> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1441405-1-1.html) ![](https://avatar.52pojie.cn/data/avatar/001/69/25/27_avatar_middle.jpg)Forwindz萌新一枚，这篇算是一个笔记吧

起因是玩某个国外的单机内购游戏，然后受不了肝度打算改存档

刚买的米 10，root 需要先刷开发版再申请解锁，so....  
（模拟器上这破游戏打不开，不然 root 后就很轻松了 _(:з」∠)_）

### 主要思路：

安卓有**备份**与恢复机制，大部分应用允许从备份数据恢复。  
这是无 root 下覆写内部存储数据最轻松的方式。

> 不过应用自己也能禁止或者管理备份文件，具体查 Android Developer Documentation

手机：MIUI 12，安卓版本 11  
开始折腾：

### 解包备份

首先备份，只备份想要的应用。  
然后把备份文件拉到电脑上去  
MIUI 上是存储设备 \ MIUI\backup\AllBackup\  
bak 文件拉出来，拖到 HxD 里看二进制

Android 的备份文件很简单，主要是安卓备份文件头 + tar 压缩文件，当然国产 MIUI 还有自己额外的文件头。

文件头：  
选中的区域就是 Android 备份文件头，  
ANDROID BACKUP 标识，备份版本（5.0），加密模式（none）  
选中区域上方是 MIUI 备份文件头  
选中区域下方就是压缩文件开始部分

去掉 MIUI 文件头（选区上方数据），然后用 abe.jar 解包成 tar 文件[↗github](https://github.com/nelenkov/android-backup-extractor)  
`java -jar abe.jar unpack xxxx.bak xxx.tar`

> abe.jar 实现的是比较通用的安卓备份协议，然而 MIUI 里备份设置超级简单，就是 MIUI 文件头 + Android 备份文件头 + tar 文件内容，无压缩无加密，所以直接在 HxD 里把两个文件头去了直接当 tar 也行...

tar 文件直接 7zip 解压就行，至此，应该能拿到备份的数据了：  
apk 和 cache，Android/data / 包名下的数据，应用私有的数据

### Unity 逆向

然后是逆向 Unity，不过小游戏一般没什么壳和加密，所以直接上  
apk 解压，assets/bin/Data/，资源文件基本都在这儿，一般拿 [Unity Studio↗](https://github.com/RaduMC/UnityStudio)或者 [disUnity↗](https://github.com/ata4/disunity)解压一下就行

> 然而更多情况是遇到新版本和加密，解析不出来（doge  
> disunity 支持的 unity 版本确实过老了，个人觉得 Unity Studio 更好用，而且还有 GUI（短期记忆每天被迫查文档...  
> 虽然我最后还是没能提取资源，有空再研究下（瘫

#### 反编译代码

所以美术资源先放一边，先来代码  
Unity 现在应该都是 il2cpp backend 了，mono 就不提了，所以直接定位路径：  
运行库 so：`lib\arm64-v8a\libil2cpp.so`  
元数据文件：`设备\Android\data\应用包名\il2cpp\Metadata\global-metadata.dat`  
然后把两个文件丢给 [Il2CppDumper↗](https://github.com/Perfare/Il2CppDumper)，dump 一下 class 和方法声明  
`Il2CppDumper.exe ibil2cpp.so global-metadata.dat`  
生成一堆文件，对于我来说里面最有用的是`dump.cs`，所有 C# 类和方法声明，外加每个方法的虚拟地址  
然后开始递归搜索，搜 Diamond 查找相关方法，然后在 ida 中查看相应的`Assembly-CSharp.dll`文件，分析具体代码...

> 实际上根本分析不动 (╯‵□′)╯︵┻━┻，但至少现在有汇编可以看了

#### 存档文件

回头看存档文件，`apps\com.ssicosm.dungeon_princess_2_free\sp\com.ssicosm.dungeon_princess_2_free.v2.playerprefs.xml`  
800KB+，一看就是存档（用 Preference Key-Value 存档是认真的吗...）  
点进去大概是这种画风：

```
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <string >0%2FevWhQCAEK4%2FBE%3D</string>
    <string >DwIABV3MAg%3D%3D</string>
    <string >VW5pdAUCAGnZ1gg%3D</string>
    <string >SW52ZRQCAGnZ1gg%3D</string>
    <string >SW52ZRQCAGnZ1gg%3D</string>

```

%3D，很明显，URL 编码，先过一遍 URL 解码  
`LFZGUwhpJ11cQitXDFRvBTl5FUxZWQhgBFRFU1A%3D`=>`LFZGUwhpJ11cQitXDFRvBTl5FUxZWQhgBFRFU1A=`  

末尾等号，多半是 base64 之类的，然后直接 base64 解码...  

... 获得了一堆乱码

然后只能回头分析代码，看来是简单加密过了（doge

#### 存档解密

众所周知，AES, DES 是超级常用的加密算法，  
然后我顺着 C# 和 Unity 相关的密码学库（Encrypt）找引用... 找了半天没找到，淦（时间成本 +⑨）  
只能从存档相关变量入手，比如`Diamond`这个变量，  
直接在`dump.cs`里搜，找到相关类`Data_Manager_Cntr`，是个数据存储工具人，  
查关键字 save，找到`SaveStageDataAll()`方法，对应虚拟地址`0xA7FDD4`，离文件存储指日可待  
IDA 启动，载入 DummyDll 下的`Assembly-CSharp.dll`分析  

实际上完全没必要看懂代码，直接找`B 0xABCDEF`之类的地址跳转指令和调用函数指令，  
一路跳到真正的加密解密的地方：`ObscuredInt`, `ObscuredByte`, `EncryptIntValue`  
看汇编我怀疑是异或，因为有 EOR 这个指令，异或计算是最高效简便的办法，至少能直接拦住改存档的萌新数个小时（比如我）

实际上看到一半啃不动了，打开 [iLSpy↗](https://github.com/icsharpcode/ILSpy)，看了一下包名，毕竟小型游戏或多或少会用些 SDK 和社区插件，除了我这种动不动就想要造一个轮子的人（专门生产不能用的方轮子）  
然后查到了有个叫 anti-cheat toolkit 的插件，C# 源代码（旧? 新版本）直接到手，完美 _(:з」∠)_，剩下的就很简单了

看了下源码，和 ida 中比对了一下，虽然有的地方不同，但变化不大。  
加密算法非常简单，就是异或！  
作者指定一个 key，（这个 key 可能和设备唯一标示符有关），然后数据转成字节流，按位与 key 进行异或就行了，key 长度不够的话就循环用 key  
对于一个键值对，键名会被初始 key 异或混淆一下，然后键名 + 初始 key 组成一个新的 key 来异或键值对的值，然后再加上 3bytes 的类型信息和 4bytes 的原始数据的 xxhash  
异或用来混淆的话，加密解密计算就很简单了，异或奇数次，加密，异或偶数次，解密（不用遇到数论题真是太好了）

现在问题就是怎么获取 **key**  
如果能挂断点分析的话，应该能很轻松地拿到了  
打开安卓模拟器，游戏启动失败... 不对，可能设备相关，我模拟器拿到的 key 可能不管用

> Android_ID 高版本下，每个应用获得的 ID 与 硬件信息，软件信息，APP 本身签名信息关联，所以很有可能每个应用拿到不一样的 ANDROID_ID。Unity 中用 deviceUniqueIdentifier 获得唯一标示符，参考[↗](https://docs.unity3d.com/ScriptReference/SystemInfo-deviceUniqueIdentifier.html)  

手机没 root，告辞（当然可以考虑强制让系统 dump 内存，可是...C# 的 GC 会把内存搬来搬去）  
还是静态分析吧（误

进入游戏变动游戏数值，比如 Diamond 16->15，然后备份，拉取存档，和原文比较，  
找到变动的值，`S2lhbQUCAI5W1RE=` => `VGlhbQUCAEi9msI=`  
对应的键：`IVFRWwlYAQ==`

好吧，看不出来什么，转成二进制  
`S2lhbQUCAI5W1RE=` =>`0x5469616d 05 02 00 48bd9ac2`  
`VGlhbQUCAEi9msI=`=>`0x4b69616d 05 02 00 8e56d511`  
按照加密算法，`` `0x5469616d``和`0x4b69616d`是加密数据，其他是类型信息`0x05 02 00`和哈希信息`0x48bd9ac2`，`0x8e56d511`  
所以  
`0x5469616d` xor key = 16  
`0x4b69616d` xor key = 15  
解得 key=`0x5b 69 61 6d`

但是这个 key 只对这个变量起作用，并且键值对的键也作为 key 的一部分，所以，我们拿这个 key 去还原对应的键`IVFRWwlYAQ==`  
对应二进制为`0x2151515b095801`，获得 Diam 四个字母，注意到这和 Diamond 相似，且正好匹配 7 个字节，那就可以认为这个键的真实值是 Diamand 字符串了  
得到真正的原始 key="0x45383036663665"，7 个字节的 key，先扔进 xml 解密一下  

注意到解密出 artifac 这个单词（artifact），遗物当然得拼全了，于是更新下我们的 key 到 8 位`0x4538303666366538`

再次解密，这次拿装备名字试试，比如 inven_Br 怎么看都应该是 inven_Bracer，得到 key=`0x453830366636653830366636`  
注意到 key 非常长，但对一些长字符串数据仍解密不成功，考虑到循环，尝试缩短 key，得到 key=`0x653830366636`  
这个就是真正的 key 了，解密后数据就是

```
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <string >1.7000000476837158 @20@2@0</string>
    <string > @15@2@0</string>
    <string >0 @5@2@0</string>
    <string >0.0 @20@2@0</string>

```

总之看起来非常舒服就是了 233

> 另外 anti-cheat toolkit 也对内存数据部分进行了异或混淆，所以就算内存查找也多半搜不到真实数据，需要一些特别的技巧

### 打包备份文件

这部分本来应该很简单的，我们不是有 abe.jar 吗，直接`java -jar abe.jar pack xxx.tar xxx.ab`不就行了....

然后我就凉了（MIUI 直接恢复失败）

首先是 tar 打包的问题，abe.jar 对应的 [github readme](https://github.com/nelenkov/android-backup-extractor) 提示过，先把原备份文件的文件单给拉下来（文件在 tar 包中存储的**顺序**很重要）  
`tar tf 原备份文件.tar | grep -F "com.包名.name" > package.list`  
然后再根据此列表打包，  
`tar cf 打包的压缩名.tar -T package.list`  
package.list 存储了你要压缩的文件，这样能保证严格按照安卓备份的文件顺序打包

注意 linux 和 windows 下打包产生的 tar 可能还是不满足 Android/MIUI 备份文件要求，  
虽然 MIIUI 告诉我恢复成功了，但是 app 数据从原先 8MB 变成了 100KB，很明显没有完全成功啊）

抓取 debug 日志分析，adb.exe 找不到的话用[这个来抓取日志↗](https://blog.csdn.net/aikongmeng/article/details/9764297)

```
05-16 00:57:08.924 W/BpBinder(30448): Slow Binder: BpBinder transact took 3851ms, interface=miui.app.backup.IBackupManager, code=3 oneway=false
05-16 00:57:08.924 I/ACC_EVENT_TRIGGERED( 7597): TYPE: 2048     PACKAGE: com.miui.backup
05-16 00:57:08.924 I/AUTOSTART_DEBUG( 7597): EVENT IS VALID     TYPE: 2048     PACKAGE: com.miui.backup
05-16 00:57:08.925 E/Backup:BackupManager(30448): IOException
05-16 00:57:08.925 E/Backup:BackupManager(30448): java.io.IOException: write failed: EPIPE (Broken pipe)
05-16 00:57:08.925 E/Backup:BackupManager(30448): 

``` ![](https://avatar.52pojie.cn/data/avatar/000/86/57/10_avatar_middle.jpg)CIBao 图片呢? 发布后也不看一下的么 ![](https://avatar.52pojie.cn/data/avatar/001/33/06/05_avatar_middle.jpg) bjxiaoyao 思路清奇，方法得当，感谢分享。![](https://avatar.52pojie.cn/data/avatar/001/65/94/14_avatar_middle.jpg)plauger 嗯，IOS 的方式也类似，修改备份，再同步回去![](https://avatar.52pojie.cn/data/avatar/000/60/43/88_avatar_middle.jpg)雨之幽 看不懂。。。![](https://avatar.52pojie.cn/data/avatar/000/27/77/70_avatar_middle.jpg)丶 man 学习了~ 找时间试试 ![](https://avatar.52pojie.cn/data/avatar/001/46/29/24_avatar_middle.jpg) xdfg 卧潮，难度这么大？![](https://static.52pojie.cn/static/image/smiley/default/3.gif)![](https://avatar.52pojie.cn/data/avatar/001/64/24/40_avatar_middle.jpg)jerry1314 学习一下![](https://static.52pojie.cn/static/image/smiley/default/42.gif) ![](https://avatar.52pojie.cn/data/avatar/000/10/24/33_avatar_middle.jpg) cacuts 太秀了吧你也, 玩个游戏这么拼 ![](https://avatar.52pojie.cn/data/avatar/001/30/05/35_avatar_middle.jpg) HQXZ ![](https://static.52pojie.cn/static/image/smiley/default/47.gif)感谢分享
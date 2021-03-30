> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/luoyesiqiu/p/10791511.html)

以前刚用手机的时候，经常可以在玩机论坛上看到很多发 ROM 包的帖子，譬如什么大深度定制 ROM，什么大深度深度精简纯净版 ROM... 相信很多喜欢搞机的都有见过这类帖子。后来自己不满每次刷机后都要手动设置一大堆东西，遂按照论坛上的教程改了 Defy + 的 cm11 的 ROM，集成了绿色守护，默认允许安装未知来源的应用，默认电池百分号显示等等。时隔 4 年，又玩起了 ROM 包定制，感慨颇多

1. 解包
=====

假设有一个名为 update.zip 的 ROM 包，我们要在 Ubuntu 下对它进行定制。首先把`system.transfer.list`和`system.new.dat.br`（有些旧版的系统的镜像可能是 system.new.dat）从 update.zip 解压出来，转成 system.img（原始镜像格式），修改完后又按步骤打包回原来的格式。本文只写了 system 分区的定制方法，但是对于其他分区也是类似的，都要转成原始镜像格式后才能对它修改。如果使用`file system.img`命令来查看 system.img 文件信息，会得到类似下面的信息：

```
system.img: Linux rev 1.0 ext4 filesystem data, UUID=da594c53-9beb-f85c-85c5-cedf76546f7a, volume name "system" (extents) (large files)


```

1.1 system.new.dat.br 转换为 system.new.dat[#](#11-systemnewdatbr转换为systemnewdat)
------------------------------------------------------------------------------

`brotli -d system.new.dat.br`

> 注：如果镜像就是 system.new.dat 格式，就跳过这步

1.2 system.new.dat 转成 system.img[#](#12-systemnewdat转成systemimg)
----------------------------------------------------------------

```
git clone https://github.com/xpirt/sdat2img
cd sdat2img
python sdat2img.py  ../system.transfer.list ../system.new.dat


```

1.3 挂载 system.img[#](#13-挂载systemimg)
-------------------------------------

```
sudo mkdir -p /mnt/system
sudo mount -o loop system.img /mnt/system


```

1.4 扩容（可选）[#](#14-扩容（可选）)
-------------------------

挂载后可以通过`df -h`来查看挂载点`/mnt/system`剩余空间有多少，如果没有剩余，就要对它进行扩容，下面的例子是给它增加 128M 的空间，扩容之前要先取消挂载

```
dd if=/dev/zero bs=1M count=128 >> system.img
e2fsck -f system.img
resize2fs system.img


```

2. 修改
=====

现在，可以在 / mnt/system 目录下根据自己的需求增删文件了

[![](https://images.cnblogs.com/cnblogs_com/luoyesiqiu/1455621/o_system_files.png)](https://images.cnblogs.com/cnblogs_com/luoyesiqiu/1455621/o_system_files.png)

增删文件需要注意：

1.  对 / mnt/system 进行写操作需要 root 权限
2.  如果需要往 / system/app 目录或者 / system/priv-app 目录下加入自己的 apk, 需要注意除了把 apk 复制进去外，还要把 apk 里面的 so 文件复制进去（如果有的话），复制进去时注意 apk 和 so 文件的路径，可以参考其他系统 App 是怎么存放的
3.  对于非 Apk 文件，复制进去后，还要使用 chmod,chown 等命令给它们合理的权限才能生效

3. 打包
=====

打包其实就是解包的逆过程

3.1 生成 system.img[#](#31-生成systemimg)
-------------------------------------

```
sudo make_ext4fs -T 0 -S file_contexts -l 1024M -a system system_new.img /mnt/system


```

*   -T 代表对镜像中的 unix 文件时间戳进行设置，这里设置为 0，表示 1970-1-1
*   -S 指定 file_contexts
*   -l 表示目标镜像的大小。如果不懂得写多少可以使用`df -h`命令查看挂载点`/mnt/system`的总大小，然后取整数（512M,1024M,2048M...），比如查得挂载点空间大小是 992M, 你就得写 1024M
*   -a 指定目标 img 文件在 Android 中的挂载点
*   system_new.img 表示生成的镜像
*   /mnt/system/ 表示源目录

> 注： file_contexts 可以去[这里](https://github.com/LineageOS/android_system_sepolicy)获取，根据系统版本选择分支 (Android7.0 对应的是 cm14.0 分支，Android7.1 对应的是 cm14.1 分支，Android8.0 对应 lineage-15.0 分支, 以此类推)，下载后也可以根据自己的需求定制 file_contexts

成功后会在当前目录下生成 system_new.img。如果发生错误，根据错误进行调整参数，直到没有错误提示为止。

3.2 卸载 system[#](#32-卸载system)
------------------------------

```
sudo umount /mnt/system


```

3.3 把 system.img 转成 system.new.dat[#](#33-把systemimg转成systemnewdat)
-------------------------------------------------------------------

转换之前可以对之前解压出来的文件进行备份：

```
mv system.transfer.list system.transfer.list.bak
mv system.new.dat system.new.dat.bak


```

开始转换

```
git clone https://github.com/jazchen/rimg2sdat
cd rimg2sdat
python rimg2sdat.py system_new.img


```

成功后会在当前目录下生成 system.transfer.list 和 system.new.dat

3.4 system.new.dat 转成 system.new.dat.br[#](#34-systemnewdat转成systemnewdatbr)
----------------------------------------------------------------------------

```
brotli -0 system.new.dat


```

> 注：如果开始解压出来的镜像就是 system.new.dat 格式，就跳过这步

3.5 更新文件到刷机包[#](#35-更新文件到刷机包)
-----------------------------

```
zip update.zip <system.new.dat.br或者system.new.dat> system.transfer.list


```

4. 扩展知识
=======

在有些刷机包里，它里面包含的 system.img 镜像是`sparse image`格式的，如果用 file 命令查看它的信息，显示如下：

```
system.img: Android sparse image, version: 1.0, Total of 655360 4096-byte output blocks in 6009 input chunks.


```

对于这种格式的镜像，如果想把它挂载和修改，就要转成我们上面提到的 raw image（原始镜像）格式，命令如下：

```
simg2img <sparse_image_files> <raw_image_file>


```

修改完成后，取消挂载，再使用下面的命令将`raw image`转成`sparse image`:

```
img2simg <raw_image_file> <sparse_image_file> [<block_size>]


```

5. 总结
=====

相对于修改 Android 源码的方式，直接修改镜像的方法对 PC 配置要求低很多。如果我们只想增加一些现有的模块和删除不必要的模块，这是很好的方式。而且对于一些手机厂商，他们没有提供 Android 源码，我们就只能用直接修改镜像的方式来定制我们的 ROM。修改 ROM 的方法是灵活的，总结下来就是，看见一个镜像，可以根据后缀名和 file 命令确认它的格式，看情况将它转成原始镜像格式并挂载，就可以修改了，修改后又转回它原来的格式，最后替换刷机包中原有的镜像

[![](https://images.cnblogs.com/cnblogs_com/luoyesiqiu/1570030/o_200606011422luoyesiqiu_qr.jpg)](//images.cnblogs.com/cnblogs_com/luoyesiqiu/1570030/o_200606011422luoyesiqiu_qr.jpg)

**关注微信公众号：luoyesiqiu，浏览更多内容**
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [wrlus.com](https://wrlus.com/harmonyos-next-security/extract-huawei-ota-package/)

> UPDATE.APP 是华为手机升级包的主要部分，解析这个文件可以获得升级包中的文件系统。实际上这对于华为刷机黑灰产来说都已经不是问题，本文只是对过程进行记录，方便后面查阅。

背景
--

UPDATE.APP 是华为手机升级包的主要部分，解析这个文件可以获得升级包中的文件系统。实际上这对于华为刷机黑灰产来说都已经不是问题，本文只是对过程进行记录，方便后面查阅。

获取升级包
-----

目前华为 OTA 请求仍旧采用 http，所以可以使用热点抓包的方式获取到，例如：

```
GET /download/data/pub_13/HWHOTA_hota_900_9/93/v3/OQysmoJHSE-Sk1GV2_iRMA/full/update_full_base.zip HTTP/1.1
Host: update.dbankcdn.com
Connection: Keep-Alive
User-Agent: HwOUCDownloadManager

```

似乎还是永久链接，至少目前看还没有失效。

解析 UPDATE.APP
-------------

解压下载的`update_full_base.zip`，可以得到一个`UPDATE.APP`文件，还有一些其他文件：

```
-rw-r--r-- 1 xiaolu xiaolu    2067505 2009年 1月 1日 build_tools.zip
-rw-r--r-- 1 xiaolu xiaolu         14 2009年 1月 1日 full_mainpkg.tag
drwxr-xr-x 3 xiaolu xiaolu       4096 10月12日 11:46 META-INF
-rw-r--r-- 1 xiaolu xiaolu         20 2009年 1月 1日 OTA_update.tag
-rw-r--r-- 1 xiaolu xiaolu        102 2009年 1月 1日 packageinfo.mbn
-rw-r--r-- 1 xiaolu xiaolu        854 2009年 1月 1日 permission_hash.bin
-rw-r--r-- 1 xiaolu xiaolu     201912 2009年 1月 1日 PTABLE.APP
-rw-r--r-- 1 xiaolu xiaolu     200704 2009年 1月 1日 ptable.img
-rw-r--r-- 1 xiaolu xiaolu          8 2009年 1月 1日 rom_hota_for_charger
-rw-r--r-- 1 xiaolu xiaolu       8192 2009年 1月 1日 sec_thee_header
-rw-r--r-- 1 xiaolu xiaolu       8192 2009年 1月 1日 sec_xloader_header
-rw-r--r-- 1 xiaolu xiaolu         18 2009年 1月 1日 SOFTWARE_VER_LIST.mbn
-rw-r--r-- 1 xiaolu xiaolu         17 2009年 1月 1日 SOFTWARE_VER.mbn
-rw-r--r-- 1 xiaolu xiaolu          5 2009年 1月 1日 switch_os.tag
-rw-r--r-- 1 xiaolu xiaolu 9601890072 2009年 1月 1日 UPDATE.APP
-rw-r--r-- 1 xiaolu xiaolu         24 2009年 1月 1日 update_feature_list.conf
-rw-r--r-- 1 xiaolu xiaolu         53 2009年 1月 1日 update.tag
-rw-r--r-- 1 xiaolu xiaolu         11 2009年 1月 1日 UPT_VER.tag
-rw-r--r-- 1 xiaolu xiaolu         33 2009年 1月 1日 VERSION.mbn

```

UPDATE.APP 可以用 [huawei_UPDATE.APP_unpacktool](https://github.com/xjljian/huawei_UPDATE.APP_unpacktool/tree/Y300-G510-G330) 来解析，不过似乎因为年久失修并没有读取出每个镜像的名称，后续可以尝试修复下：

```
-rw-r--r-- 1 xiaolu xiaolu        256 10月12日 15:20 unknown_file.0
-rw-r--r-- 1 xiaolu xiaolu     585766 10月12日 15:20 unknown_file.1
-rw-r--r-- 1 xiaolu xiaolu     375488 10月12日 15:20 unknown_file.10
-rw-r--r-- 1 xiaolu xiaolu    7397330 10月12日 15:20 unknown_file.11
-rw-r--r-- 1 xiaolu xiaolu     383360 10月12日 15:20 unknown_file.12
-rw-r--r-- 1 xiaolu xiaolu      50152 10月12日 15:20 unknown_file.13
-rw-r--r-- 1 xiaolu xiaolu      69952 10月12日 15:20 unknown_file.14
-rw-r--r-- 1 xiaolu xiaolu    1576960 10月12日 15:20 unknown_file.15
-rw-r--r-- 1 xiaolu xiaolu    1542656 10月12日 15:20 unknown_file.16
-rw-r--r-- 1 xiaolu xiaolu      51855 10月12日 15:20 unknown_file.17
-rw-r--r-- 1 xiaolu xiaolu      75456 10月12日 15:20 unknown_file.18
-rw-r--r-- 1 xiaolu xiaolu     195584 10月12日 15:20 unknown_file.19
-rw-r--r-- 1 xiaolu xiaolu         18 10月12日 15:20 unknown_file.2
-rw-r--r-- 1 xiaolu xiaolu    3331200 10月12日 15:20 unknown_file.20
-rw-r--r-- 1 xiaolu xiaolu     155968 10月12日 15:20 unknown_file.21
-rw-r--r-- 1 xiaolu xiaolu   14680064 10月12日 15:20 unknown_file.22
-rw-r--r-- 1 xiaolu xiaolu    4967424 10月12日 15:20 unknown_file.23
-rw-r--r-- 1 xiaolu xiaolu    1092712 10月12日 15:20 unknown_file.24
-rw-r--r-- 1 xiaolu xiaolu    9926976 10月12日 15:20 unknown_file.25
-rw-r--r-- 1 xiaolu xiaolu   10159808 10月12日 15:20 unknown_file.26
-rw-r--r-- 1 xiaolu xiaolu     167624 10月12日 15:20 unknown_file.27
-rw-r--r-- 1 xiaolu xiaolu   20971520 10月12日 15:20 unknown_file.28
-rw-r--r-- 1 xiaolu xiaolu   10485760 10月12日 15:20 unknown_file.29
-rw-r--r-- 1 xiaolu xiaolu         17 10月12日 15:20 unknown_file.3
-rw-r--r-- 1 xiaolu xiaolu   56623104 10月12日 15:20 unknown_file.30
-rw-r--r-- 1 xiaolu xiaolu   33554432 10月12日 15:20 unknown_file.31
-rw-r--r-- 1 xiaolu xiaolu    2097152 10月12日 15:20 unknown_file.32
-rw-r--r-- 1 xiaolu xiaolu   25165824 10月12日 15:20 unknown_file.33
-rw-r--r-- 1 xiaolu xiaolu   56623104 10月12日 15:20 unknown_file.34
-rw-r--r-- 1 xiaolu xiaolu   33554432 10月12日 15:20 unknown_file.35
-rw-r--r-- 1 xiaolu xiaolu    2097152 10月12日 15:20 unknown_file.36
-rw-r--r-- 1 xiaolu xiaolu   25165824 10月12日 15:20 unknown_file.37
-rw-r--r-- 1 xiaolu xiaolu  123731968 10月12日 15:20 unknown_file.38
-rw-r--r-- 1 xiaolu xiaolu   20971520 10月12日 15:20 unknown_file.39
-rw-r--r-- 1 xiaolu xiaolu        102 10月12日 15:20 unknown_file.4
-rw-r--r-- 1 xiaolu xiaolu   12582912 10月12日 15:20 unknown_file.40
-rw-r--r-- 1 xiaolu xiaolu      30832 10月12日 15:20 unknown_file.41
-rw-r--r-- 1 xiaolu xiaolu      16960 10月12日 15:20 unknown_file.42
-rw-r--r-- 1 xiaolu xiaolu   48234496 10月12日 15:20 unknown_file.43
-rw-r--r-- 1 xiaolu xiaolu    3145728 10月12日 15:20 unknown_file.44
-rw-r--r-- 1 xiaolu xiaolu   16777216 10月12日 15:20 unknown_file.45
-rw-r--r-- 1 xiaolu xiaolu   16777216 10月12日 15:20 unknown_file.46
-rw-r--r-- 1 xiaolu xiaolu 2904555520 10月12日 15:21 unknown_file.47
-rw-r--r-- 1 xiaolu xiaolu  987758592 10月12日 15:21 unknown_file.48
-rw-r--r-- 1 xiaolu xiaolu 4114612224 10月12日 15:21 unknown_file.49
-rw-r--r-- 1 xiaolu xiaolu        104 10月12日 15:20 unknown_file.5
-rw-r--r-- 1 xiaolu xiaolu    4194304 10月12日 15:21 unknown_file.50
-rw-r--r-- 1 xiaolu xiaolu  964689920 10月12日 15:21 unknown_file.51
-rw-r--r-- 1 xiaolu xiaolu   33554432 10月12日 15:21 unknown_file.52
-rw-r--r-- 1 xiaolu xiaolu    6033488 10月12日 15:21 unknown_file.53
-rw-r--r-- 1 xiaolu xiaolu     866165 10月12日 15:21 unknown_file.54
-rw-r--r-- 1 xiaolu xiaolu    4194304 10月12日 15:21 unknown_file.55
-rw-r--r-- 1 xiaolu xiaolu    4194304 10月12日 15:21 unknown_file.56
-rw-r--r-- 1 xiaolu xiaolu     200704 10月12日 15:20 unknown_file.6
-rw-r--r-- 1 xiaolu xiaolu     581184 10月12日 15:20 unknown_file.7
-rw-r--r-- 1 xiaolu xiaolu      69376 10月12日 15:20 unknown_file.8
-rw-r--r-- 1 xiaolu xiaolu    6299648 10月12日 15:20 unknown_file.9

```

提取 EROFS 文件系统
-------------

上述文件可以通过`file`命令查看都是什么文件，现在的话最大的几个文件都是 EROFS 镜像，解析需要依赖于`erofs-utils`：

```
sudo apt-get install erofs-utils

```

提取 EROFS 文件系统的话，还需要使用 [extract.erofs](https://github.com/sekaiacg/erofs-utils)，其实也是利用 erofs-utils 中的命令实现，最后可以得到 fs_options：

```
Filesystem created:        Thu Jun  6 08:00:00 2024
Filesystem UUID:           ce164447-05b7-4590-9aa5-688d1ed7b18f
mkfs.erofs options:        -zlz4hc -T 1717632000 -U ce164447-05b7-4590-9aa5-688d1ed7b18f --mount-point=/unknown_file --fs-config-file=./config/unknown_file_fs_config --file-contexts=./config/unknown_file_file_contexts unknown_file_repack.img ./unknown_file

```

以及文件系统的内容：

```
lrwxrwxrwx  1 xiaolu xiaolu   11  6月 6日 08:00 bin -> /system/bin
drwxr-xr-x  2 xiaolu xiaolu 4096  6月 6日 08:00 chip_prod
lrwxrwxrwx  1 xiaolu xiaolu    7  6月 6日 08:00 chipset -> /vendor
dr-xr-xr-x  2 xiaolu xiaolu 4096  6月 6日 08:00 config
drwxr-xr-x  2 xiaolu xiaolu 4096  6月 6日 08:00 cust
drwxr-x--x  2 xiaolu xiaolu 4096  6月 6日 08:00 data
drwxr-xr-x  2 xiaolu xiaolu 4096  6月 6日 08:00 dev
drwxr-xr-x  2 xiaolu xiaolu 4096  6月 6日 08:00 eng_chipset
drwxr-xr-x  2 xiaolu xiaolu 4096  6月 6日 08:00 eng_system
lrwxrwxrwx  1 xiaolu xiaolu   11  6月 6日 08:00 etc -> /system/etc
lrwxrwxrwx  1 xiaolu xiaolu   16  6月 6日 08:00 init -> /system/bin/init
lrwxrwxrwx  1 xiaolu xiaolu   11  6月 6日 08:00 lib -> /system/lib
drwxr-xr-x  2 xiaolu xiaolu 4096  6月 6日 08:00 log
drwxr-xr-x  2 xiaolu xiaolu 4096  6月 6日 08:00 mnt
drwxr-xr-x  2 xiaolu xiaolu 4096  6月 6日 08:00 module_update
drwxr-xr-x  2 xiaolu xiaolu 4096  6月 6日 08:00 patch_hw
drwxr-xr-x  2 xiaolu xiaolu 4096  6月 6日 08:00 preload
drwxr-xr-x  2 xiaolu xiaolu 4096  6月 6日 08:00 proc
drwxr-xr-x  2 xiaolu xiaolu 4096  6月 6日 08:00 sec_storage
drwxr-xr-x  2 xiaolu xiaolu 4096  6月 6日 08:00 storage
drwxr-xr-x  2 xiaolu xiaolu 4096  6月 6日 08:00 sys
drwxr-xr-x  2 xiaolu xiaolu 4096  6月 6日 08:00 sys_prod
drwxr-xr-x 12 xiaolu xiaolu 4096 10月12日 15:44 system
drwxr-xr-x  2 xiaolu xiaolu 4096  6月 6日 08:00 tmp
drwxr-xr-x  2 xiaolu xiaolu 4096  6月 6日 08:00 updater
drwxr-xr-x  2 xiaolu xiaolu 4096  6月 6日 08:00 vendor
drwxr-xr-x  2 xiaolu xiaolu 4096  6月 6日 08:00 version

```

分析
--

这样就可以进行系统安全分析甚至 TEE 安全分析了，后续有时间会撰写更多文章分享这部分的内容。
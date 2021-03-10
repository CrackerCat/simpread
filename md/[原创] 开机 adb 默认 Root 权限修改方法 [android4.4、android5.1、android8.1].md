> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-260471.htm)

**android 4.4.4_r1** 修改 /system/core/adb/adb.c 处 should_drop_Privileges() 函数  
![](https://bbs.pediy.com/upload/attach/202007/778210_QUA5B5UP65UCJ6S.png)  
**Android 5.1 版本修改方法**

```
1.直接修改函数should_drop_privileges() 函数, 清空这个函数，直接返回 0 即可。返回0 即开启root 权限。
2.修改 alps/external/sepolicy/Android.mk 文件， 将-D target_build_variant=$(TARGET_BUILD_VARIANT) 改成 -D target_build_variant=eng

```

**Android 8.1 版本修改方法**  
1. 路径为 /system/core/adb/daemon/main.cpp 文件中 寻找 should_drop_privileges 函数  
![](https://bbs.pediy.com/upload/attach/202007/778210_5KFJCF92RAESBY3.png)  
2. 将 should_drop_privileges 函数所有返回值都修改为 false 即可。  
注释说明只有调试模式和 root 权限下不会运行 drop_capabilities_bounding_set_if_needed 函数。  
修改后刷机 adb 链接如下图默认 root 权限：  
![](https://bbs.pediy.com/upload/attach/202007/778210_9YNFZ44CXZYNZMQ.png)

[看雪学院推出的专业资质证书《看雪安卓应用安全能力认证 v1.0》（中级和高级）！](https://bbs.pediy.com/thread-265424.htm)

最后于 2020-7-3 12:36 被山竹笠编辑 ，原因：
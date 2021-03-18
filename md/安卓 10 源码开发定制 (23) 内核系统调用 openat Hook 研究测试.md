> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/iPNav3Lc4g1p5hlBt3EkMA)

    以下操作基于安卓 10 系统 lineageOs 17.1 源码研究, 手机型号 oneplus3 镜像研究测试。

一、安卓内核模块开发编译
------------

    安卓系统如何开发内核可加载模块参考以下文章:  
[玩转 Android10 源码开发定制 (11) 内核篇之安卓内核模块开发编译](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483936&idx=1&sn=89cdee39c0bda4cca7ea4d289a03c62f&scene=21#wechat_redirect)

二、内核系统调用 hook 原理
----------------

    内核系统调用 hook 主要是在内核模块加载的时候，通过修改替换内核系统调用表 sys_call_table 的系统调用号地址来实现目的。

三、内核源码默认配置修改支持系统调用挂钩功能
----------------------

*   为了通过 insmod 命令加载内核模块，需要增加 Android 内核可加载内核模块配置的支持。
    
*   为了获取 sys_call_table 导出全局内核符号表的地址需要增加内核符号表导出配置功能。
    
*   为了向内核内存空间读写数据，需要禁用 Android 内核的内存保护。需要关闭 CONFIG_STRICT_MEMORY_RWX 这个选项来编译内核。
    

   如下是所有相关的配置修改:

```
# ///ADD START
# 允许通过insmod命令加载模块
CONFIG_MODULES=y
# 允许通过命令rmmod命令卸载内核模块
CONFIG_MODULE_UNLOAD=y
# 关闭内核内存读写保护
CONFIG_STRICT_MEMORY_RWX=n

# 打开该配置使符号表中包含所有的函数地址
CONFIG_KALLSYMS=y

# 打开该配置方便通过adb shell cat /proc/kallsyms查看内核导出的符号地址
CONFIG_KALLSYMS_ALL=y
# ///ADD END



```

  将以上配置追加到安卓源码内核源文件中，文件路径如下:

> home/qiang/lineageOs/kernel/oneplus/msm8996/arch/arm64/configs/lineageos_oneplus3_defconfig

   配置好之后编译一次源码，产生内核的编译输出文件。后续编写内核模块需要依赖这些文件来进行编译。

四、获取内核符号表地址
-----------

     将以上内核源码配置修改之后，编译安卓系统。lineageOs 源码编译过程中会自动编译内核源码。内核编译输出目录如下:

> /home/qiang/lineageOs/out/target/product/oneplus3/obj/KERNEL_OBJ

   在内核编译输出目录找到 System.map 文件。打开该文件找到 system_call_table 符号表的地址。如下所示:

```
qiang@ubuntu:~/lineageOs/out/target/product/oneplus3/obj/KERNEL_OBJ$
 
qiang@ubuntu:~/lineageOs/out/target/product/oneplus3/obj/KERNEL_OBJ$ pwd
/home/qiang/lineageOs/out/target/product/oneplus3/obj/KERNEL_OBJ

qiang@ubuntu:~/lineageOs/out/target/product/oneplus3/obj/KERNEL_OBJ$ ls -la System.map 
-rw-rw-r-- 1 qiang qiang 4946646 2月   3 22:46 System.map


qiang@ubuntu:~/lineageOs/out/target/product/oneplus3/obj/KERNEL_OBJ$ cat System.map  |grep "sys_call_table"

ffffffc000093000 T compat_sys_call_table
ffffffc001057000 R sys_call_table



```

从以上 System.map 文件可以知道 sys_call_table 符号地址为:0xffffffc001057000

五、挂钩 openat 函数模块开发
------------------

     可加载模块 openat hook 核心关键代码如下:

```
//保存内核系统原有的openat函数地址
asmlinkage int (*real_openat)(int, const char __user*,int,umode_t);
 //保存系统调用表地址
void **sys_call_table;

// 替换内核系统调用openat新的new_openat函数
int new_openat(int dirfd, const char __user* pathname, int flags,umode_t modex)
{
  
  int ret=-1;
  //cred 中保存了当前进程的uid pid等信息
  const struct cred *cred = current_cred();
  kuid_t uid=cred->uid;
  int pid=current->pid;
  int myuid=uid.val;
  if(myuid>10000)
  {
     char bufname[256]={0};
     strncpy_from_user(bufname,pathname,255);
     //打印日志输出测试
     printk("openat pathname:%s  uid:%d pid:%d\n",bufname,myuid,pid);
  }
  ret= real_openat(dirfd, pathname, flags,modex);
  return ret;
}
 
//模块加载的时候调用
static int __init hook_init(void){
    printk("load module success\n");
    //sys_call_table符号地址
    sys_call_table=(void*)0xffffffc001057000;
    printk("sys_call_table %p",(void*)sys_call_table);
    // 获取openat函数的调用地址
    real_openat = (void*)(sys_call_table[__NR_openat]);
    printk("11111111  real_openat %p\n",(void*)real_openat);
    //替换系统调用表openat函数地址
    sys_call_table[__NR_openat]=(unsigned long*)new_openat;
    printk("replace openat finish");
    return 0;
}


```

六、测试模块
------

    1. 编译内核模块，本人编译的模块为 kernel_hook_openat.ko

        安卓系统如何开发内核可加载模块参考以下文章:  
        [玩转 Android10 源码开发定制 (11) 内核篇之安卓内核模块开发编译](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247483936&idx=1&sn=89cdee39c0bda4cca7ea4d289a03c62f&scene=21#wechat_redirect)

    2. 将内核模块 adb push 到手机, 并用 insmod 命令加载模块，如下图所示:  
![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew54326OOEQreRzYqqewuA1FgvZViadywdFXfggw6UMhojjso41iblUQnLVWibczHnSJiaIt3KnPvoUnKkwlA/640?wx_fmt=png)  
    3. 打印内核日志验证，日志如下:

```
4,3927,3837282014,-;load module success
4,3928,3837282027,-;sys_call_table ffffffc00105700011111111  real_openat ffffffc0001db1f8
4,3929,3837282031,-;replace openat finish

4,3974,3850163171,-;openat pathname:/system/lib64/libselinux.so  uid:10129 pid:5822
4,3975,3850163867,-;openat pathname:/system/lib64/libcutils.so  uid:10129 pid:5822
4,3976,3850164538,-;openat pathname:/system/lib64/libprocessgroup.so  uid:10129 pid:5822
4,3977,3850164870,-;openat pathname:/system/lib64/libcrypto.so  uid:10129 pid:5822
4,3978,3850165322,-;openat pathname:/system/lib64/libz.so  uid:10129 pid:5822
4,3979,3850165703,-;openat pathname:/system/lib64/libc.so  uid:10129 pid:5822
4,3980,3850167419,-;openat pathname:/system/lib64/libm.so  uid:10129 pid:5822
4,3981,3850167877,-;openat pathname:/system/lib64/libdl.so  uid:10129 pid:5822
4,3982,3850168240,-;openat pathname:/system/lib64/libc++.so  uid:10129 pid:5822
4,3983,3850168565,-;openat pathname:/system/lib64/libpcre2.so  uid:10129 pid:5822
4,3984,3850168795,-;openat pathname:/system/lib64/libpackagelistparser.so  uid:10129 pid:5822
4,3985,3850169027,-;openat pathname:/system/lib64/libbase.so  uid:10129 pid:5822
4,3986,3850169780,-;openat pathname:/system/lib64/libcgrouprc.so  uid:10129 pid:5822
4,3987,3850179680,-;openat pathname:/sys/kernel/mm/transparent_hugepage/enabled  uid:10129 pid:5822
4,3988,3850180250,-;openat pathname:/dev/__properties__/property_info  uid:10129 pid:5822
4,3989,3850180348,-;openat pathname:/dev/__properties__/properties_serial  uid:10129 pid:5822
4,3990,3850180448,-;openat pathname:/dev/__properties__/u:object_r:default_prop:s0  uid:10129 pid:5822
4,3991,3850180496,-;openat pathname:/dev/__properties__/u:object_r:debug_prop:s0  uid:10129 pid:5822
4,3992,3850180572,-;openat pathname:/dev/__properties__/u:object_r:heapprofd_prop:s0  uid:10129 pid:5822
4,3993,3850180722,-;openat pathname:/system/lib64/libnetd_client.so  uid:10129 pid:5822


```

参考文章:

*   Hooking Android System Calls for Pleasure and Benefit
    
    **https://www.cnblogs.com/lanrenxinxin/p/6289436.html**
    

微信扫描以下二维码关注我公众号 "卓码空间", 更多资讯分享。  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew54326OOEQreRzYqqewuA1FgvZQ4tNq7rJ9wBksn0NA0Q1RpufSj8kk5P2qjWEmvlkEIM2ibC8Zm33pOQ/640?wx_fmt=png)
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-274660.htm)

> [原创] Android Studio Debug dlopen

[原创] Android Studio Debug dlopen

2022-10-9 14:32 4880

### [原创] Android Studio Debug dlopen

 [![](http://passport.kanxue.com/upload/avatar/604/918604.png?1616807717)](user-home-918604.htm) [.KK](user-home-918604.htm) ![](https://bbs.kanxue.com/view/img/rank/7.png)  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 2022-10-9 14:32  4880

Android Studio Debug dlopen
===========================

前言
==

*   目的：debug dlopen，对 ELF 加载及解析流程进行解析
*   对于直接分析 AOSP 源码我感觉脑子里很难形成一个概念，如果能 debug 进行分析的话，我觉得会直观很多，所以这就是此文的目的
*   效果写在前
    
    ![](https://bbs.kanxue.com/upload/attach/202210/918604_SN5S57KB6GZA86P.png)
    

1. 环境准备
-------

*   手机环境为 `eng` 或 `userdebug` ，`user` 版本及 ARM 模拟器我没测试成功过（甚至连 ARM 模拟器都跑不起来！！！
    
    ```
    $ adb shell getprop ro.build.type  # 这个命令可以查看版本
    eng
    
    ```
    
*   其他方式可以跳到文末参考链接，这里就给出稳妥的方案
    
*   本文测试手机环境：
    *   Pixel 1
    *   自编译 AOSP 源码 Android10_r17
*   准备 debug 所需文件
    
    *   `aosp/bionic` C++ 源代码
    *   `aosp/out/target/product/sailfish/symbols/apex/com.android.runtime.debug` 带 debug 信息的 So
        
        ```
        k@k:~/bin/aosp/out/target/product/sailfish/symbols/apex/com.android.runtime.debug/bin$ readelf -S linker64
        There are 30 section headers, starting at offset 0x11c0f00:
         
        节头：
          [号] 名称              类型             地址              偏移量    大小              全体大小          旗标   链接   信息   对齐
          [ 0]                   NULL             0000000000000000  00000000  0000000000000000  0000000000000000           0     0     0
          [ 1] .note.gnu.bu[...] NOTE             0000000000000238  00000238  0000000000000020  0000000000000000   A       0     0     4
          [ 2] .dynsym           DYNSYM           0000000000000258  00000258  0000000000000240  0000000000000018   A       4     1     8
          [ 3] .gnu.hash         GNU_HASH         0000000000000498  00000498  00000000000000c0  0000000000000000   A       2     0     8
          [ 4] .dynstr           STRTAB           0000000000000558  00000558  00000000000002b2  0000000000000000   A       0     0     1
          [ 5] .relr.dyn         LOOS+0xfffff00   0000000000000810  00000810  00000000000001f0  0000000000000008   A       0     0     8
          [ 6] .rodata           PROGBITS         0000000000000a00  00000a00  0000000000017d0d  0000000000000000 AMS       0     0     32
          [ 7] .gcc_except_table PROGBITS         0000000000018710  00018710  00000000000050dc  0000000000000000   A       0     0     4
          [ 8] .eh_frame_hdr     PROGBITS         000000000001d7ec  0001d7ec  0000000000004bc4  0000000000000000   A       0     0     4
          [ 9] .eh_frame         PROGBITS         00000000000223b0  000223b0  0000000000014f24  0000000000000000   A       0     0     8
          [10] .text             PROGBITS         0000000000038000  00038000  00000000000d1528  0000000000000000  AX       0     0     64
          [11] .data             PROGBITS         000000000010a000  0010a000  0000000000000ef8  0000000000000000  WA       0     0     32
          [12] .data.rel.ro      PROGBITS         000000000010b000  0010b000  0000000000006270  0000000000000000  WA       0     0     8
          [13] .init_array       INIT_ARRAY       0000000000111270  00111270  0000000000000060  0000000000000008  WA       0     0     8
          [14] .dynamic          DYNAMIC          00000000001112d0  001112d0  00000000000000e0  0000000000000010  WA       4     0     8
          [15] .got              PROGBITS         00000000001113b0  001113b0  0000000000000820  0000000000000000  WA       0     0     8
          [16] .bss              NOBITS           0000000000112000  00111bd0  0000000000009e70  0000000000000000  WA       0     0     4096
          [17] .debug_str        PROGBITS         0000000000000000  00111bd0  00000000003ae347  0000000000000001  MS       0     0     1
          [18] .debug_loc        PROGBITS         0000000000000000  004bff17  00000000003f6b1c  0000000000000000           0     0     1
          [19] .debug_abbrev     PROGBITS         0000000000000000  008b6a33  000000000003f6ae  0000000000000000           0     0     1
          [20] .debug_info       PROGBITS         0000000000000000  008f60e1  00000000006391f2  0000000000000000           0     0     1
          [21] .debug_ranges     PROGBITS         0000000000000000  00f2f2d3  00000000000ed620  0000000000000000           0     0     1
          [22] .debug_macinfo    PROGBITS         0000000000000000  0101c8f3  0000000000000164  0000000000000000           0     0     1
          [23] .comment          PROGBITS         0000000000000000  0101ca57  000000000000019c  0000000000000001  MS       0     0     1
          [24] .debug_line       PROGBITS         0000000000000000  0101cbf3  000000000011cb4a  0000000000000000           0     0     1
          [25] .debug_aranges    PROGBITS         0000000000000000  0113973d  00000000000013e0  0000000000000000           0     0     1
          [26] .gnu_debuglink    PROGBITS         0000000000000000  0113ab1d  0000000000000020  0000000000000000           0     0     1
          [27] .shstrtab         STRTAB           0000000000000000  011c0dbe  000000000000013c  0000000000000000           0     0     1
          [28] .symtab           SYMTAB           0000000000000000  0113ab40  00000000000484b0  0000000000000018          29   12315     8
          [29] .strtab           STRTAB           0000000000000000  01182ff0  000000000003ddce  0000000000000000           0     0     1
        Key to Flags:
          W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
          L (link order), O (extra OS processing required), G (group), T (TLS),
          C (compressed), x (unknown), o (OS specific), E (exclude),
        k@k:~/bin/aosp/out/target/product/sailfish/symbols/apex/com.android.runtime.debug/bin$ ll -hr|grep linker64
        -rwxrwxr-x 1 k k   18M 10月  8 01:09 linker64*
        
        ```
        
*   这里可以把手机里的 `linker64` 拿出来做一个对比，这里就可以看见手机里的 So 并没有带 `debug_xx` 的信息
    
    ```
    $ adb pull /system/apex/com.android.runtime.debug/bin/linker64
    $ ll -hr linker64
    -rw-r--r-- 1 k k 1.6M 10月  9 13:30 linker64
    $ readelf -S linker64
    There are 23 section headers, starting at offset 0x187dd0:
     
    节头：
      [号] 名称              类型             地址              偏移量    大小              全体大小          旗标   链接   信息   对齐
      [ 0]                   NULL             0000000000000000  00000000  0000000000000000  0000000000000000           0     0     0
      [ 1] .note.gnu.bu[...] NOTE             0000000000000238  00000238  0000000000000020  0000000000000000   A       0     0     4
      [ 2] .dynsym           DYNSYM           0000000000000258  00000258  0000000000000240  0000000000000018   A       4     1     8
      [ 3] .gnu.hash         GNU_HASH         0000000000000498  00000498  00000000000000c0  0000000000000000   A       2     0     8
      [ 4] .dynstr           STRTAB           0000000000000558  00000558  00000000000002b2  0000000000000000   A       0     0     1
      [ 5] .relr.dyn         LOOS+0xfffff00   0000000000000810  00000810  00000000000001f0  0000000000000008   A       0     0     8
      [ 6] .rodata           PROGBITS         0000000000000a00  00000a00  0000000000017d0d  0000000000000000 AMS       0     0     32
      [ 7] .gcc_except_table PROGBITS         0000000000018710  00018710  00000000000050dc  0000000000000000   A       0     0     4
      [ 8] .eh_frame_hdr     PROGBITS         000000000001d7ec  0001d7ec  0000000000004bc4  0000000000000000   A       0     0     4
      [ 9] .eh_frame         PROGBITS         00000000000223b0  000223b0  0000000000014f24  0000000000000000   A       0     0     8
      [10] .text             PROGBITS         0000000000038000  00038000  00000000000d1528  0000000000000000  AX       0     0     64
      [11] .data             PROGBITS         000000000010a000  0010a000  0000000000000ef8  0000000000000000  WA       0     0     32
      [12] .data.rel.ro      PROGBITS         000000000010b000  0010b000  0000000000006270  0000000000000000  WA       0     0     8
      [13] .init_array       INIT_ARRAY       0000000000111270  00111270  0000000000000060  0000000000000008  WA       0     0     8
      [14] .dynamic          DYNAMIC          00000000001112d0  001112d0  00000000000000e0  0000000000000010  WA       4     0     8
      [15] .got              PROGBITS         00000000001113b0  001113b0  0000000000000820  0000000000000000  WA       0     0     8
      [16] .bss              NOBITS           0000000000112000  00112000  0000000000009e70  0000000000000000  WA       0     0     4096
      [17] .comment          PROGBITS         0000000000000000  00112000  000000000000019c  0000000000000001  MS       0     0     1
      [18] .gnu_debuglink    PROGBITS         0000000000000000  0011219c  0000000000000020  0000000000000000           0     0     1
      [19] .symtab           SYMTAB           0000000000000000  001121c0  0000000000037ed8  0000000000000018          20   9522     8
      [20] .strtab           STRTAB           0000000000000000  0014a098  000000000003dc4d  0000000000000000           0     0     1
      [21] .shstrtab         STRTAB           0000000000000000  00187ce5  00000000000000d4  0000000000000000           0     0     1
      [22] .gnu_debuglink    PROGBITS         0000000000000000  00187dbc  0000000000000010  0000000000000000           0     0     4
    Key to Flags:
      W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
      L (link order), O (extra OS processing required), G (group), T (TLS),
      C (compressed), x (unknown), o (OS specific), E (exclude),
      D (mbind), p (processor specific)
    
    ```
    

2. 新建 AS 工程
-----------

*   所需文件
    1.  带 debug 信息的 So
    2.  AOSP 对应 C++ 源代码
*   工程目录
    
    ![](https://bbs.kanxue.com/upload/attach/202210/918604_YSBFYT28THNWBEU.png)
    

3. Debug dlopen Gif
-------------------

![](https://bbs.kanxue.com/upload/attach/202210/918604_2VUD23RDEF4ZVAB.gif)

4. Debug dlopen
---------------

1.  写一个按钮调用 JNI 方法，方便调试
2.  把准备的源代码以及 debug So 文件放进 Android Studio 工程目录
3.  导入 debug 符号，这里 Debugtype 选择 `Native Only` ，Symbol Directories 别选择顶级目录 `com.android.runtime.debug` 会有 bug，某些 So 会无法加载到
    
    ![](https://bbs.kanxue.com/upload/attach/202210/918604_Q8JPPVECWP6R6G9.png)
    
4.  接下来就是进行 debug 调试，这里有一个需要注意的点就是不要直接打断点，需要先暂停，使用 lldb 进行断点
    
    ![](https://bbs.kanxue.com/upload/attach/202210/918604_TBH2JVCXJZ7D6XA.png)
    
5.  `target modules list` 查看已加载 So 列表，因为我需要 debug 的 符号在 linker 里面，所以重点查看 linker 加载的情况，如果没有被加载就无法 debug
    
    ```
    (lldb) target modules list | grep linker64
    [  0] C87DDFF9-3A23-6A5B-654C-9182F7C24A0D 0x0000007dd64d7000 /home/k/Desktop/AndroidStudioProjects/example/com.android.runtime.debug/bin/linker64
    [  1] C87DDFF9-3A23-6A5B-654C-9182F7C24A0D 0x0000007dd64d7000 /home/k/Desktop/AndroidStudioProjects/example/com.android.runtime.debug/bin/linker64
    
    ```
    
6.  断点命令
    
    ```
    br s -n __loader_dlopen # 断点符号
    br s -f dlfcn.cpp -l 138 # 在指定文件数断点：在 dlfcn.cpp 139 行进行drd
    c # continue 在当前进程中继续执行所有线程
    
    ```
    
7.  接下来就是常规的一步一步跟进调试了，dlopen 加载流程分析以及 ELF 解析这个我就不多费口舌了，网上资料一大堆～
    
    ![](https://bbs.kanxue.com/upload/attach/202210/918604_P92TS9SFZ7RV5QC.png)
    

5. 总结
-----

*   能实践成功还得感谢某不知名佬以及参考文章里的佬
*   该方法不仅仅局限于 debug dlopen，也可以 debug native framework 以及 ART 虚拟机，具体方式可以看文末参考文章引用
*   根据这个方式进行 debug AOSP 进行源码学习我觉得比只能看代码分析会更直观一点，本文仅是自己的学习总结，如有问题还请大佬斧正！

参考
--

*   [如何调试 Android Native Framework](https://weishu.me/2017/01/14/how-to-debug-android-native-framework-source/)
*   [如何用 AndroidStudio 动态调试 ART 虚拟机？](https://keeplooking.top/2020/05/16/Android/android-native-debug/)
*   [LLDB 学习与使用笔记・常用工具学习笔记](https://dbgtech.github.io/Tools/lldb-using.html)

  

[反勒索软件开发实战篇](https://www.kanxue.com/book-section_list-46.htm)

[#系统相关](forum-161-1-126.htm) [#其他](forum-161-1-129.htm)
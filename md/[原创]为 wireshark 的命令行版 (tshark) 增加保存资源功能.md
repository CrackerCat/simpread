> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-224502.htm)

> [原创]为 wireshark 的命令行版 (tshark) 增加保存资源功能

为 Wireshark 的命令行版 (tshark) 增加保存资源功能
===================================

目录

*   [为 Wireshark 的命令行版 (tshark) 增加保存资源功能](#为wireshark的命令行版(tshark)增加保存资源功能)
*            Wireshark 简介
*            Wireshark 树形插件模式协议解析
*                    [协议树](#协议树)
*                    [插件模式](#插件模式)
*                    [源码阅读建议](#源码阅读建议)
*            tshark 捕获、解析逻辑
*            [调试、分析过程](#调试、分析过程)
*                    [抓包时调试、分析](#抓包时调试、分析)
*                    [分析数据包时调试](#分析数据包时调试)
*            [保存图片新增代码](#保存图片新增代码)
*                    [addsave-file.h](#addsave-file.h)
*                    [addsave-save.h](#addsave-save.h)
*                    [addsave-save.c](#addsave-save.c)
*                    [tshrk.c 新增代码](#tshrk.c新增代码)
*                            [头文件](#头文件)
*                            [print_columns() 函数中](#print_columns()函数中)
*                            [process_packet() 函数中](#process_packet()函数中)
*            [使用方法](#使用方法)
*                    [编译](#编译)
*                    [使用](#使用)
*            [效果展示](#效果展示)
*                    [从已经捕获的数据包中抓取图片](#从已经捕获的数据包中抓取图片)
*                    [在捕获数据包同时保存图片](#在捕获数据包同时保存图片)
*                    png 图片
*                    gif 图片

> 在科锐第二阶段学到插件设计模式时姚老师指出 wireshark 的协议解析是一个很好的插件模式的例子。于是阅读在其源码的同时为 tshark 增加自动保存 http 资源的功能。现在来分享给大家，源码在附件中。这个思路也可以用来自动过滤、分析数据包。  
> github: https://github.com/KevinsBobo/wireshark-modify

Wireshark 简介
------------

Wireshark 是一个免费开源的网络数据包分析软件。网络数据包分析软件的功能是截取网络数据包，并尽可能显示出最为详细的网络数据包数据。

 

Wrieshark 是协议分析的必备工具，它的强大之处并不是因为它的抓包功能，因为它的抓包是直接利用 WinPcap(Windows) 和 libpcap(Linux) 实现的，它真正强大之处在于协议解析。而它协议解析的设计是一个基于插件模式的树形结构。

Wireshark 树形插件模式协议解析
--------------------

### 协议树

基于 osi 的 7 层协议模型的协议数据是从上到下封装的，即在网络模型的每一层给上一层的数据加个头部数据，而解析时则需要从下到上进行，每一层又有不同的协议（例如传输层分为 TCP 和 UDP），这样就可以以树形结构一层一层的解析。每个解析器解析了自己的头部后，丢给下一个解析器，由于每一层的协议很多，对应也那么多个解析器，解析器也就可以构成一个和协议一样的树形结构，Wireshark 也是这样做的。如果我们想要添加一个自定义的解析器，只需要向这颗树里添加一个节点就搞定了。

 

一颗简单的协议树：

```
         IP
      /      \
   TCP       UDP
  /  \       /  \
HTTP SMTP  TFTP ICQ

```

### 插件模式

wireshark 启动时会加载所有`$installdir/plugins/$buildversion/*.dll`，调用每个 dll 的以下两个导出函数注册插件、协议并记录版本号来实现插件的加载。

```
__declspec(dllexport) void plugin_register(void)
__declspec(dllexport) const gchar version[] = VERSION;
__declspec(dllexport) void plugin_reg_handoff(void)

```

插件在这两个函数里完成`dissector`的注册和初始化工作，其中`plugin_register`注册插件，`plugin_reg_handoff`注册协议。注册协议的`plugin_reg_handoff`函数是关键的地方，这里面要指出该解析器的父结点是谁，还要指出自己区别于父节点下所有兄弟节点协议的特征字，示例如下（感兴趣的朋友可以下载附件里由 **@五行猫** 同学编写的 Wireshark 插件开发笔记）：

```
void proto_reg_handoff_foo(void)
{
    static dissector_handle_t foo_handle;
    foo_handle = create_dissector_handle(dissect_foo, proto_foo);
    dissector_add_uint("udp.port", FOO_PORT, foo_handle);
}

```

### 源码阅读建议

对协议解析感兴趣的朋友直接看源码中`epan`目录的的代码就可以了

tshark 捕获、解析逻辑
--------------

> 其实这自动保存资源可以采用在协议树中插入一个解释器的方法来做，但是为了了解清楚 Wireshark 的协议树调用方法就在分析调用代码了

```
tshark.c
main()
    解析命令行参数
    初始化环境
    -->
    caption()
        初始化信息
        while(...)
            --> capchild\capture_sync.c -> syn_pipe_input_cp(...) // 回调的方式
                从管道(文件)中读取数据头部信息
                如果读到的是一个数据包的头部则开始处理
                --> capture_input_new_packets(capture_session ...)
                    根据传入的capture_session获得一个 capture_file 类型的数据指针
                    如需解码：
                        获得一个 epan_dissect_t 类型的空间
                        并根据 capture_file 结构体初始化数据
                        // edt 中将包含一个具有链表结构的数据包
                        --> process_packet(capture_file *cf, epan_dissect_t *edt ...)
                            解析数据
                            --> epan_dissect_run_with_taps(...)
                                获得 edt 中的 tvbuff_t 类型的数据指针 // 具有链表结构的数据包
                            --> print_packet(...)
                                打印数据包
                                --> print_columns()
                                    以列的形式打印数据
                                    通过 cf 获得数据包的列结构数据
                                    // 编号、时间 IP TCP/UDP 端口 类型 数据类型
                                    // 我是在这里通过字符串比较判断是否为 HTTP 资源
                                    如果通过参数设置需要打印数据包16进制数据包
                                        则继续打印16进制数据
                                        这里打印的16进制数据是数据包的原格式
                                        后面我借鉴了这里面对 tvbuff_t 类型数据的分解的代码
                                        从而得到图片数据

```

[](#调试、分析过程)调试、分析过程
-------------------

### [](#抓包时调试、分析)抓包时调试、分析

> `tshark src tcp 80`

1.  单步找到 `caption() capture_input_new_packets() print_packet() print_columns()`等函数
    
2.  通过 vs2013 下`fprintf()`API 断点并通过栈回溯分析函数调用情况 (因为 tshark 是通过`fprintf`输出数据的)
    
3.  通过 API 断点发现了`print_columns()`的输出`HTTP`状态的位置
    

### 分析数据包时调试

> `tshark -r out.pcap -Y http -x`

1.  在`printf_columns()`中发现了输出 16 进制数据的`print.c -> print_hex_data()`函数
    
2.  通过条件断点发现这里输出的数据是抓取到的整个数据包
    
3.  于是找到了`print_hex_data()`中使用`tvbuff_t`的代码
    

保存图片新增代码
--------

> 注：在 tshark 原代码中新增的代码都以`/* 保存图片新增 start */ ... /* 保存图片新增 stop */`的形式标注 <br> 调试新增代码都以`/* 调试新增 start */ ... /* 调试新增 stop */`的形式标注 <br><br > 为方便管理，新增代码的实现和声明放在了单独的`addsave-....c`和`addsave-....h`文件中，位于 wireshark 源码目录的`addsave`目录中 <br> 所有函数和全局变量均以`addsave-...`开头

### addsave-file.h

> 这里的函数是文件相关的，一个实创建目录，一个是初始化文件，没什么说的

```
#define ADDSAVE_MAXPATHLEN 255
 
void addsave_init_floder(char* szFloder);
 
int addsave_init_file(char* szFile);

```

### addsave-save.h

> 这是保存图片的主要代码

```
/* 照抄 tsark.h 包含的头文件 start */
... // 省略
/* 照抄 tsark.h 包含的头文件 stop */
 
/* 新增的头文件 start */
#include #include /* 新增的头文件 stop */
 
// 图片根目录
#define ADDSAVE_FILE_FLODER_NAME "..\\addsave_save\\"
// 图片名前缀
#define ADDSAVE_FILE_NAME "save_file"
 
// 全局变量 标记数据包是否为图片
extern int addsave_g_is_savefile;
// 全局变量 标记数据包来源IP
extern char addsave_g_src_ip[ ADDSAVE_MAXPATHLEN ];
 
// 图片类型
#define ADDSAVE_PIC_JPG 1
#define ADDSAVE_PIC_PNG 2
#define ADDSAVE_PIC_GIF 3
#define ADDSAVE_FILE_AUDIO 4
 
// 主要功能函数
void addsave_save_pic(epan_dissect_t * edt); 
```

### addsave-save.c

> 实现代码

```
#include int addsave_g_is_savefile = 0;
char addsave_g_src_ip[ ADDSAVE_MAXPATHLEN ];
 
void addsave_save_pic(epan_dissect_t * edt)
{
  if(edt == NULL)
  {
    return ;
  }
 
  char szFilePath[ MAXPATHLEN ] = { 0 };
  char* pszFileType = NULL;
  tvbuff_t * tvb = NULL;
  u_char* pData = NULL;
  unsigned long long *pVerify = NULL;
  FILE* fpPic = NULL;
  time_t save_time = 0;
  time(&save_time);
  static int nTime = 0;
  const guchar *cp = NULL;
  guint         length = 0;
 
  // gboolean      multiple_sources;
  GSList       *src_le = NULL;
  struct data_source *src = NULL;
 
 
  // 将 tvb 指针移动到合适的位置
  for( tvb = edt->tvb ; tvb != NULL; tvb = tvb->next)
  {
    // 可以确定 jgp/gif 图片肯定在最后一个数据包
    // PNG 图片在倒数第六个数据包
    // 所以直接将 jpg/gif 的指针移到最后一个数据包的位置
    if(((addsave_g_is_savefile == ADDSAVE_PIC_JPG
      || addsave_g_is_savefile == ADDSAVE_PIC_GIF
      || addsave_g_is_savefile == ADDSAVE_FILE_AUDIO)
         && tvb->next != NULL))
    {
      continue;
    }
 
    if(tvb->real_data == NULL)
    {
      return ;
    }
 
    // jpg 数据首地址在最后一个数据包地址的前两个字节的位置，png 和 gif 则正常
    pData = (unsigned char*)(tvb->real_data);
    pVerify = (unsigned long long*)tvb->real_data;
    // 再次判断，匹配则跳出循环，按照逻辑只有png才会多次判断
    if((*(unsigned short*)(pData - 2) == 0xD8FF && *pVerify == 0x4649464A1000E0FF)       // jpg
       || (*(unsigned long long*)(pData) == 0x0A1A0A0D474E5089)                          // png
       || (((*(unsigned long long*)(pData)) & 0x0000FFFFFFFFFFFF) == 0x0000613938464947) // gif
       || addsave_g_is_savefile == ADDSAVE_FILE_AUDIO   // audio 类型直接跳出
       )
    {
      break;
    }
  }
 
 
  /* 参考自 print.c -> print_hex_data 函数 */
  // 获取数据长度 和 http 数据包首部指针
  for(src_le = edt->pi.data_src; src_le != NULL;
      src_le = src_le->next)
  {
    if(src_le->next != NULL)
    {
      continue;
    }
 
    src = (struct data_source *)src_le->data;
    tvb = get_data_source_tvb(src);
    length = tvb_captured_length(tvb);
    if(length == 0)
      return ;
    // 获取http数据首部指针
    cp = tvb_get_ptr(tvb , 0 , length);
 
    if(cp == NULL)
    {
      return ;
    }
 
    break;
  }
 
  if(addsave_g_is_savefile == ADDSAVE_PIC_JPG)
  {
    // 偏移指针
    pData -= 2;
    pszFileType = "jpg";
  }
  else if(addsave_g_is_savefile == ADDSAVE_PIC_PNG)
  {
    pszFileType = "png";
  }
  else if(addsave_g_is_savefile == ADDSAVE_PIC_GIF)
  {
    pszFileType = "gif";
  }
  else if(addsave_g_is_savefile == ADDSAVE_FILE_AUDIO)
  {
    pszFileType = "mp3";
  }
 
  if(pszFileType == NULL)
  {
    return ;
  }
 
  sprintf_s(szFilePath , MAXPATHLEN , "%s%s\\%s\\" ,
            ADDSAVE_FILE_FLODER_NAME , addsave_g_src_ip, pszFileType);
  // 创建文件夹
  addsave_init_floder(szFilePath);
 
  sprintf_s(szFilePath ,
            MAXPATHLEN ,
            "%s%s\\%s\\%s_%lld.%s" ,
            ADDSAVE_FILE_FLODER_NAME ,
            addsave_g_src_ip,
            pszFileType ,
            ADDSAVE_FILE_NAME,
            save_time + nTime++,
            pszFileType);
 
  // 创建文件
  if(addsave_init_file(szFilePath))
  {
    return ;
  }
 
  // 打开文件
  fopen_s(&fpPic , szFilePath , "wb");
  if(fpPic == NULL)
  {
    return ;
  }
 
  // 获取文件长度
  u_int pic_length = length - (pData - cp);
 
  // 写文件
  fwrite(pData , pic_length , 1 , fpPic);
 
  // 关闭文件
  fclose(fpPic);
} 
```

### tshrk.c 新增代码

#### 头文件

> 将`addsave-....c`文件加入到 tshark 项目工程中并包含新增代码的头文件

```
/* 保存图片新增 start */
#include #include /* 保存图片新增 stop */ 
```

#### print_columns() 函数中

> 这里主要判别是否为图片，并设置标志位

```
static gboolean
print_columns(capture_file *cf)
{
  ...
  switch (col_item->col_fmt) {
    ...
    case COL_UNRES_NET_SRC:
      column_len = col_len = strlen(col_item->col_data);
      if (column_len < 12)
        column_len = 12;
      line_bufp = get_line_buf(buf_offset + column_len);
      put_spaces_string(line_bufp + buf_offset, col_item->col_data, col_len, column_len);
      /* 保存图片新增 start */
      strcpy_s(addsave_g_src_ip , ADDSAVE_MAXPATHLEN , col_item->col_data);
      /* 保存图片新增 stop */
      break;
    ...
    default:
      column_len = strlen(col_item->col_data);
      line_bufp = get_line_buf(buf_offset + column_len);
      put_string(line_bufp + buf_offset, col_item->col_data, column_len);
      /* 保存图片新增 start */
      if(!strcmp(col_item->col_data , "HTTP/1.1 200 OK  (JPEG JFIF image)"))
      {
        addsave_g_is_savefile = ADDSAVE_PIC_JPG;
      }
      else if(!strcmp(col_item->col_data , "HTTP/1.1 200 OK  (PNG)"))
      {
        addsave_g_is_savefile = ADDSAVE_PIC_PNG;
      }
      else if(!strcmp(col_item->col_data , "HTTP/1.1 200 OK  (GIF89a)")
              || !strcmp(col_item->col_data , "HTTP/1.1 200 OK  (GIF89a) (image/jpeg)"))
      {
        addsave_g_is_savefile = ADDSAVE_PIC_GIF;
      }
      else if(!strcmp(col_item->col_data , "HTTP/1.1 206 Partial Content  (audio/mpeg)")
              || !strcmp(col_item->col_data , "HTTP/1.0 206 Partial Content  (audio/mpeg)"))
      {
        addsave_g_is_savefile = ADDSAVE_FILE_AUDIO;
      }
      else
      {
        // 保险措施
        addsave_g_is_savefile = 0;
      }
      /* 保存图片新增 stop */

```

#### process_packet() 函数中

> 是在获取`edt`中`tvb`指针数据后调用保存图片的函数

```
static gboolean
process_packet(capture_file *cf, epan_dissect_t *edt, gint64 offset, struct wtap_pkthdr *whdr,
               const guchar *pd, guint tap_flags)
{
  ...
  if (passed) {
    frame_data_set_after_dissect(&fdata, &cum_bytes);
 
 
    /* Process this packet. */
    if (print_packet_info) {
      /* We're printing packet information; print the information for
         this packet. */
      print_packet(cf, edt);
 
      /* 保存图片新增 start */
      if(addsave_g_is_savefile)
      {
        addsave_save_pic(edt);
        addsave_g_is_savefile = 0;
      }
      /* 保存图片新增 stop */
  ...
 
  /* 保存图片新增 start */
  // 保险措施，标志位置 0
  addsave_g_is_savefile = 0;
  /* 保存图片新增 start */
 
  return passed;
}

```

使用方法
----

### 编译

1.  将`addsave`文件夹拷贝到`wirshark`源码目录中
    
2.  替换`tshark.c`文件
    
3.  打开`wirshark`vs2013 解决方案
    
4.  将`addsave`中的`addsave-save.c`和`addsave-file.c`添加到`tshark`项目`Soure Files`中
    
5.  重新编译`tshark`
    

### 使用

1.  捕获网卡信息流时保存`tshark src port 80`
    
2.  从`.pcap`文件中保存文件`tshark -r out.pcap -Y http`
    
3.  保存的文件在执行目录的上级目录中的`addsave_save`目录中按照 IP 和文件类型分文件夹保存
    

效果展示
----

### 从已经捕获的数据包中抓取图片

> `tshark -r out.pcap`

 

![](https://bbs.pediy.com/upload/attach/201802/751258_87A7TBU4F3YQWWP.png)

### 在捕获数据包同时保存图片

![](https://bbs.pediy.com/upload/attach/201802/751258_7QHF983HUSNHTZA.png)

### png 图片

![](https://bbs.pediy.com/upload/attach/201802/751258_H95PKD4AH7HJZQF.png)

### gif 图片

![](https://bbs.pediy.com/upload/attach/201802/751258_RD7JMPE4T4MK2SG.png)

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

上传的附件：

*   [Wireshark-modify.zip](javascript:void(0)) （38.81kb，38 次下载）
*   [Wireshark 插件开发笔记 @五行猫. zip](javascript:void(0)) （226.54kb，48 次下载）
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-274198.htm)

> [原创] 基于钉钉探索针对 CEF 框架的一些逆向思路

前言
==

[CEF](https://bitbucket.org/chromiumembedded/cef/src/master/) 是 Chromium Embedded Framework 的简写，这是**一个把 Chromium 嵌入其他应用的开源框架**。  
现在市面上有许多桌面软件都使用了 CEF 框架，比如我们经常使用的钉钉、网易云音乐等等。  
我本意是突破钉钉的一些功能限制，结果发现钉钉使用了 CEF 框架，故开始对 CEF 框架做了一些浮于表面的探索。**由于个人能力有限，如果文章中有什么错误之处，还望大家多多指教**。

初探
==

在开始正式开始之前，有必要先观察一下钉钉的安装目录，看看里面有哪些我们感兴趣的文件。  
我电脑上的钉钉版本是`6.5.30-Release.7289101`  
![](https://bbs.pediy.com/upload/attach/202208/861753_CFJYQG879SPY57T.png)  
通过查看运行中的 DingTalk.exe 进程的映射文件锁定你电脑上目前运行的钉钉的目录 (这个地方会发现有多个同名进程，我们随便选择一个)  
![](https://bbs.pediy.com/upload/attach/202208/861753_XKK9ETNMRCWR74B.png)  
有朋友可能要问为什么要通过这种方式确定目录，这其实是因为**钉钉的安装目录下面一般都会存在两个版本的文件，一个是当前版本另外一个则是上一个版本**。据我观察这两个目录下的文件结构基本一致。  
![](https://bbs.pediy.com/upload/attach/202208/861753_HD3KRYAFCNTV5AK.png)  
我电脑上的钉钉目前就使用的是`current`目录。  
打开`current`目录可以发现许多的资源文件和依赖库文件，其中对于本文来说最重要的文件是`libcef.dll`和`web_content.pak`。`libcef.dll`是 CEF 框架的支持库，`web_content.pak`则是钉钉缓存在本地的`html、js、css`文件。`web_content.pak`本质是一个`zip`压缩文件，我们可以通过解压软件查看里面的内容  
![](https://bbs.pediy.com/upload/attach/202208/861753_QEQBHCNCUUHUSBT.png)  
那么可以知道这个压缩文件是被加密了，解压的时候会让输入密码，后面会提到怎么获取密码。通过观察文件的名字也大致可以猜出这些文件的作用。  
钉钉中使用 CEF 框架的区域主要在聊天框显示区域。  
![](https://bbs.pediy.com/upload/attach/202208/861753_JD6VNJJNEHM6JH6.png)

 

下面主要介绍三个方面的内容

1.  `CEF`框架部分`API`和数据结构的介绍
2.  `web_content.pak`文件解密
3.  在钉钉中开启`CEF`框架内置的调试窗口

另外提一嘴，在钉钉的安装目录下面我们还可以发现有`cef_LICENSE.txt``duilib_license.txt`等`license`声明，通过这些声明我们也可以获得一些信息，比如钉钉还使用了`duilib`界面库。

环境准备
====

既然钉钉使用了`CEF`框架，那么学会简单的使用`CEF`框架，了解相关的`API`会使我们事半功倍。

框架下载
----

根据官方库的指引，我们前往 [https://cef-builds.spotifycdn.com/index.html](https://cef-builds.spotifycdn.com/index.html) 下载框架。  
官方实现了 C 语言版本的 CEF 框架以及 C++ 版本的 CEF 框架，其中 C++ 版本的框架是基于 C 语言版本的二次封装。而我们需要的`libcef.dll`就是 C 版本的框架。  
![](https://bbs.pediy.com/upload/attach/202208/861753_6V5A68WHZZBDG7B.png)

 

在此处下载的文件包含了已经编译好的`libcef.dll`，无需我们从源码编译`libcef`库。  
实质上从源码编译`libcef`库并不容易，因为其中涉及到编译 chromium，我猜这也是为什么官方会提供各种平台各种版本的库的原因吧。  
![](https://bbs.pediy.com/upload/attach/202208/861753_C8W64W3CQ3JTPVV.png)

CEF 版本编号格式
----------

在下载时我们需要先了解 CEF 的版本编号格式  
格式解释如下  
![](https://bbs.pediy.com/upload/attach/202208/861753_KXYC95XX7SYHYU4.png)  
以`cef_binary_104.4.25+gd80d467+chromium-104.0.5112.102_windows32.tar.bz2`为例，其中  
`104.4.25`和`104.0.5112.102`是 CEF 和 Chromium 的版本信息，`gd80d467`是`git commit`的`hash`

 

我们可以先看看钉钉使用的`libcef.dll`是什么版本  
![](https://bbs.pediy.com/upload/attach/202208/861753_KN8KGD7W7NSN5MT.png)  
这里发现一个很坑的点，就是 Windows 的文件属性显示不全，而且还不能拖开，也不能复制。  
不过根据已经显示出来的内容，可以发现钉钉使用的`libcef.dll`明显不是在官方提供的页面下载的。版本约定和官方的太不一样，`git commit`是 8 位的，官方库可是只有 7 位。  
`g2e1fb6b`, 我尝试使用`g2e1fb6`、`2e1fb6b`等`hash`在`commit`列表中搜索也没有发现，只能猜测钉钉使用的`libcef.dll`是自己从源码编译的，而且可能对源码做了一些修改吧。  
同时我使用`91.0.0`在下载界面搜索也没有发现相同的版本。后面的版本信息显示不全，得想个办法解决一下子，争取下载一个最接近的版本。**其实这里有一个大坑，后面会提到。**

获取钉钉 libcef 版本信息
----------------

其实文件属性的信息是存在于 PE 中的资源节中的，使用 Windows 系统提供的 API 或者自己解析都可以拿到相关信息。不过我是本着能不写代码就不写代码的懒人思想的。

 

一般这种库或者框架的动态库中都会提供函数查询版本信息，所以我浏览了一下`libcef`的导出函数  
在`libcef`的导出函数中我发现了`cef_version_info`这个函数，看名字就知道干什么用的了。  
该说不说，官方提供了 C++ 版本的文档，为什么不提供一个`libcef`的`api`文档呢？反正我是没找到。不过虽然没有文档，还是有源码和大量注释的。  
![](https://bbs.pediy.com/upload/attach/202208/861753_96WXXRB2DEBW2S5.png)  
这个函数的定义是这样的

```
int cef_version_info(int entry);
```

我们再结合下面的信息  
从反汇编很明显的看出来这是一个数组下标寻址  
![](https://bbs.pediy.com/upload/attach/202208/861753_XK8J876D83PWAMK.png)  
从源码得知不同的参数获取不同的信息，那么完整的版本信息存在于一个 32 字节的数组中  
![](https://bbs.pediy.com/upload/attach/202208/861753_SMYUDC3DZNC3BBK.png)  
在内存窗口转到数组内存  
![](https://bbs.pediy.com/upload/attach/202208/861753_BQXUUEP7MA8BRV8.png)  
我们缺少的是最后 Chromium 的版本信息，那么就是最后四个 int。那么简单的拼接，得到  
`5B.0.1178.A4` 转成 10 进制 `91.0.4472.164`。

 

搜索发现只有一个版本满足要求，那么就用这个好了，下载 Standard Distribution，这个里面的文件是完整的，包含了框架代码和示例代码。  
![](https://bbs.pediy.com/upload/attach/202208/861753_JQKBKTEAWBVDPBJ.png)

 

后面突然想起使用解析 PE 的格式的一些工具，也能很方便的查看资源信息。  
我用 CFF 试了一下  
![](https://bbs.pediy.com/upload/attach/202208/861753_FATWAU2CVNJMKF6.png)

一些学习资料
------

将下载后的文件解压，使用`cmake`生成`vs`工程。然后使用 vs 编译。  
这个时候编译成功了，当然可能会在编译的时候遇到一些错误或者警告，按照提示解决即可。  
那么环境准备好了，我们需要去学习一些 CEF 框架的基础知识了，直接看示例代码或者直接看框架源码都不是那么容易的，可以先在网上找前辈取点经。

*   掘金小册 -[CEF 桌面软件开发实战](https://juejin.cn/book/7075387142121193502)
    
*   知乎专栏 -[CEF](https://www.zhihu.com/column/c_1333096419650269184)
    

基于钉钉的实战
=======

最终的目标是实现钉钉聊天窗口的防撤回功能，基于这个目标，一步步的解决一些遇到的问题。

定位资源文件
------

CEF 可以从本地或者网络加载资源，一般来说桌面应用程序会将大部分需要用到的文件缓存在本地。  
所以第一步就是需要找到资源文件的位置，这个不同的软件可能使用的资源文件的名称不太一样，存放的位置也不太一样。比如钉钉是放在安装目录下的，但是网易云音乐就没有放在安装目录下。

### 从 CEF 框架 API 入手

在钉钉登录页面附加`DingTalk.exe`

 

![](https://bbs.pediy.com/upload/attach/202208/861753_GHYKRQGA7DXQ8UA.png)  
选择没有命令行参数的附加  
![](https://bbs.pediy.com/upload/attach/202208/861753_QEUFSGGJY8SGCPY.png)

 

选择这两个函数下断点  
`cef_stream_reader_create_for_data`  
`cef_stream_reader_create_for_file`  
这两个函数是 CEF 提供的两个操作文件数据的函数，返回值都是`cef_stream_reader_t`结构体。  
区别在于`cef_stream_reader_create_for_file`的参数是文件路径  
`cef_stream_reader_create_for_data`的参数是内存地址和大小，即内存中的文件数据。  
这两个函数的声明和相关的结构体如下：

```
///
// Structure used to read data from a stream. The functions of this structure
// may be called on any thread.
///
typedef struct _cef_stream_reader_t {
  ///
  // Base structure.
  ///
  cef_base_ref_counted_t base;
 
  ///
  // Read raw binary data.
  ///
  size_t(CEF_CALLBACK* read)(struct _cef_stream_reader_t* self,
                             void* ptr,
                             size_t size,
                             size_t n);
 
  ///
  // Seek to the specified offset position. |whence| may be any one of SEEK_CUR,
  // SEEK_END or SEEK_SET. Returns zero on success and non-zero on failure.
  ///
  int(CEF_CALLBACK* seek)(struct _cef_stream_reader_t* self,
                          int64 offset,
                          int whence);
 
  ///
  // Return the current offset position.
  ///
  int64(CEF_CALLBACK* tell)(struct _cef_stream_reader_t* self);
 
  ///
  // Return non-zero if at end of file.
  ///
  int(CEF_CALLBACK* eof)(struct _cef_stream_reader_t* self);
 
  ///
  // Returns true (1) if this reader performs work like accessing the file
  // system which may block. Used as a hint for determining the thread to access
  // the reader from.
  ///
  int(CEF_CALLBACK* may_block)(struct _cef_stream_reader_t* self);
} cef_stream_reader_t;
 
 
///
// Create a new cef_stream_reader_t object from a file.
///
CEF_EXPORT cef_stream_reader_t* cef_stream_reader_create_for_file(
    const cef_string_t* fileName);
 
///
// Create a new cef_stream_reader_t object from data.
///
CEF_EXPORT cef_stream_reader_t* cef_stream_reader_create_for_data(
    void* data,
    size_t size);
```

断点下好之后，直接登录。  
钉钉中没有使用`cef_stream_reader_create_for_data`函数，使用的是`cef_stream_reader_create_for_file`。  
命中断点，观察参数  
`/local_res/common_res.pak`  
![](https://bbs.pediy.com/upload/attach/202208/861753_RGP953X37BUG3SZ.png)  
`/web_content.pak`  
![](https://bbs.pediy.com/upload/attach/202208/861753_G88QJ9AS8CT8MHJ.png)

 

`/local_res/common_res.pak`文件中的内容  
![](https://bbs.pediy.com/upload/attach/202208/861753_RFN4TDFQM6X9AVB.png)  
`/web_content.pak`文件中的内容  
![](https://bbs.pediy.com/upload/attach/202208/861753_YMYQ7W8Q8BQW8TW.png)  
到这就已经确定了资源文件的路径了。  
不过需要注意的一点是，如果程序使用了`cef_stream_reader_create_for_data`函数，那我们就不能从参数直接得到路径了。这个时候需要配合下面的方法使用。

### 从 Windows API 入手

直接在`kernel32.dll.CreateFileW/A`和`kernel32.dll.ReadFileW/A`下断点，观察函数的参数，如果觉得这样比较废手的话，可以使用行为监控软件比如微软的`ProcessMonitor`，设置好过滤选项之后监控程序的文件操作。

解密资源文件
------

如果资源文件被加密了，怎么解密文件。  
思路其实很简单，程序运行时肯定会在某个时机解密数据，我们在相关 API 处下断点，逆向分析即可得到密码。  
钉钉的资源文件是 zip 压缩加密，得到密码的方式有两个方向。

### 从 CEF 框架 API 入手

`cef_zip_directory` 写数据到 zip 文件  
`cef_zip_reader_create`从 zip 文件读取数据

 

函数声明和相关结构体声明

```
///
// All ref-counted framework structures must include this structure first.
///
typedef struct _cef_base_ref_counted_t {
  ///
  // Size of the data structure.
  ///
  size_t size;
 
  ///
  // Called to increment the reference count for the object. Should be called
  // for every new copy of a pointer to a given object.
  ///
  void(CEF_CALLBACK* add_ref)(struct _cef_base_ref_counted_t* self);
 
  ///
  // Called to decrement the reference count for the object. If the reference
  // count falls to 0 the object should self-delete. Returns true (1) if the
  // resulting reference count is 0.
  ///
  int(CEF_CALLBACK* release)(struct _cef_base_ref_counted_t* self);
 
  ///
  // Returns true (1) if the current reference count is 1.
  ///
  int(CEF_CALLBACK* has_one_ref)(struct _cef_base_ref_counted_t* self);
 
  ///
  // Returns true (1) if the current reference count is at least 1.
  ///
  int(CEF_CALLBACK* has_at_least_one_ref)(struct _cef_base_ref_counted_t* self);
} cef_base_ref_counted_t;
 
 
///
// Structure that supports the reading of zip archives via the zlib unzip API.
// The functions of this structure should only be called on the thread that
// creates the object.
///
typedef struct _cef_zip_reader_t {
  ///
  // Base structure.
  ///
  cef_base_ref_counted_t base;
 
  ///
  // Moves the cursor to the first file in the archive. Returns true (1) if the
  // cursor position was set successfully.
  ///
  int(CEF_CALLBACK* move_to_first_file)(struct _cef_zip_reader_t* self);
 
  ///
  // Moves the cursor to the next file in the archive. Returns true (1) if the
  // cursor position was set successfully.
  ///
  int(CEF_CALLBACK* move_to_next_file)(struct _cef_zip_reader_t* self);
 
  ///
  // Moves the cursor to the specified file in the archive. If |caseSensitive|
  // is true (1) then the search will be case sensitive. Returns true (1) if the
  // cursor position was set successfully.
  ///
  int(CEF_CALLBACK* move_to_file)(struct _cef_zip_reader_t* self,
                                  const cef_string_t* fileName,
                                  int caseSensitive);
 
  ///
  // Closes the archive. This should be called directly to ensure that cleanup
  // occurs on the correct thread.
  ///
  int(CEF_CALLBACK* close)(struct _cef_zip_reader_t* self);
 
  // The below functions act on the file at the current cursor position.
 
  ///
  // Returns the name of the file.
  ///
  // The resulting string must be freed by calling cef_string_userfree_free().
  cef_string_userfree_t(CEF_CALLBACK* get_file_name)(
      struct _cef_zip_reader_t* self);
 
  ///
  // Returns the uncompressed size of the file.
  ///
  int64(CEF_CALLBACK* get_file_size)(struct _cef_zip_reader_t* self);
 
  ///
  // Returns the last modified timestamp for the file.
  ///
  cef_basetime_t(CEF_CALLBACK* get_file_last_modified)(
      struct _cef_zip_reader_t* self);
 
  ///
  // Opens the file for reading of uncompressed data. A read password may
  // optionally be specified.
  ///
  int(CEF_CALLBACK* open_file)(struct _cef_zip_reader_t* self,
                               const cef_string_t* password);
 
  ///
  // Closes the file.
  ///
  int(CEF_CALLBACK* close_file)(struct _cef_zip_reader_t* self);
 
  ///
  // Read uncompressed file contents into the specified buffer. Returns < 0 if
  // an error occurred, 0 if at the end of file, or the number of bytes read.
  ///
  int(CEF_CALLBACK* read_file)(struct _cef_zip_reader_t* self,
                               void* buffer,
                               size_t bufferSize);
 
  ///
  // Returns the current offset in the uncompressed file contents.
  ///
  int64(CEF_CALLBACK* tell)(struct _cef_zip_reader_t* self);
 
  ///
  // Returns true (1) if at end of the file contents.
  ///
  int(CEF_CALLBACK* eof)(struct _cef_zip_reader_t* self);
} cef_zip_reader_t;
 
///
// Writes the contents of |src_dir| into a zip archive at |dest_file|. If
// |include_hidden_files| is true (1) files starting with "." will be included.
// Returns true (1) on success.  Calling this function on the browser process UI
// or IO threads is not allowed.
///
CEF_EXPORT int cef_zip_directory(const cef_string_t* src_dir,
                                 const cef_string_t* dest_file,
                                 int include_hidden_files);
 
 
///
// Create a new cef_zip_reader_t object. The returned object's functions can
// only be called from the thread that created the object.
///
CEF_EXPORT cef_zip_reader_t* cef_zip_reader_create(
    struct _cef_stream_reader_t* stream);
```

需要特别关注的是`cef_zip_reader_t`中的`open_file`成员

```
///
// Opens the file for reading of uncompressed data. A read password may
// optionally be specified.
///
int(CEF_CALLBACK* open_file)(struct _cef_zip_reader_t* self,
                             const cef_string_t* password);
```

参数中带有`password`，那我们在这个函数下断点就可以得到密码了。

 

具体步骤如下  
在钉钉登录页面附加程序，`cef_stream_reader_create_for_file`函数下断点。  
登录钉钉，在函数`cef_stream_reader_create_for_file`参数是`web_content.pak`路径的时候记住返回值，并给`cef_zip_reader_create`下断点，程序继续运行。  
![](https://bbs.pediy.com/upload/attach/202208/861753_HNAJHBYWSATWZQK.png)  
`cef_zip_reader_create`断点名命中，检查参数是否是上面记住的返回值  
![](https://bbs.pediy.com/upload/attach/202208/861753_3C8UUVFB89CXS3Z.png)  
如果没问题断到则先让程序回到返回处，得到`cef_zip_reader_t*`返回值`0x25CF2940`。  
![](https://bbs.pediy.com/upload/attach/202208/861753_X64EAFXQA2GYZXN.png)  
在内存中按地址查看`0x25CF2940`  
![](https://bbs.pediy.com/upload/attach/202208/861753_S4VB5GDQTHWDB8R.png)  
根据 open_file 在结构体中的偏移我们直接就可以找到函数地址，我直接数了一下偏移是`0x30`，下标第 12 项，直接下断点，运行程序等待断点命中。

 

然后断点确实命中了，第二个参数就是密码。这里就不截图了，感兴趣的可以自己去试一下。

### 从 Windows API 入手

如果程序没有使用 CEF 框架提供的函数解密，那么上面说的方法就不行了。这种时候只能使用老办法，在`CreateFileA/W`和`ReadFileA/W`下断点，调试程序。  
用这种方式也能得到密码，好奇的同学可以去试一下，可以在栈中发现密码。

 

最后提一嘴，这个密码钉钉是怎么计算出来的。我只能说这个算法是 MD5，可以利用 IDA 分析安装目录下的`MainFrame.dll`结合算法识别插件。不过我没有逆，有大哥逆过，感谢大哥，手动 at 大哥 [0xC5](https://bbs.pediy.com/user-768770.htm)。

修改 CEF 框架加载的资源
--------------

可以解密资源之后，我们就可以分析 Js 文件了。想让修改生效，有两种方式

1.  直接修改文件，然后重新加密替换原来的资源文件
2.  hook CEF 框架的相关函数在内存中实现修改

直接替换文件非常简单，但是有个问题。这个方式不太稳定，据我观察钉钉会不定期的更新资源文件 (这个更新不是指钉钉的升级)，更新之后还得重新替换。  
第二种方式的话，其实也不难。我们可以 hook `cef_zip_reader_t`结构体中的`read_file`函数，并配合`get_file_name`函数实现在内存中修改。

 

不过内存替换我也没有去尝试，这里只提供一种思路。

```
int CEF_CALLBACK hook_read_file(
    struct _cef_zip_reader_t* self,
    void* buffer,
    size_t bufferSize) {
 
    // 调用原始的read_file
    int result = old_read_file(self, buffer, bufferSize);
 
    // 获取文件名
    cef_string_userfree_t ptr_file_name = get_file_name(self);
 
    // 对比文件名
    if (strcmp(ptr_file_name->str, "xxxx") == 0) {
 
        // 如果文件名满足要求，则可以考虑遍历buffer修改关键点
    }
}
```

开启 DevTools
-----------

改代码不是什么难事，难的是找到关键点。如果能开启 Chromium 本身的动态调试功能，那对于分析人员来说简直是如虎添翼。

 

在 `cef_browser_host_t`结构体中有一个`show_dev_tools`成员，可以用来开启调试窗口。  
`cef_browser_host_t`对象可以通过`cef_browser_t`的`get_host`拿到。

 

`get_host ``show_dev_tools`声明

```
///
// Returns the browser host object. This function can only be called in the
// browser process.
/// 
struct _cef_browser_host_t* CEF_CALLBACK get_host(
      struct _cef_browser_t* self);
 
///
// Open developer tools (DevTools) in its own browser. The DevTools browser
// will remain associated with this browser. If the DevTools browser is
// already open then it will be focused, in which case the |windowInfo|,
// |client| and |settings| parameters will be ignored. If |inspect_element_at|
// is non-NULL then the element at the specified (x,y) location will be
// inspected. The |windowInfo| parameter will be ignored if this browser is
// wrapped in a cef_browser_view_t.
///
void CEF_CALLBACK show_dev_tools(
    struct _cef_browser_host_t* self,
    const struct _cef_window_info_t* windowInfo,
    struct _cef_client_t* client,
    const struct _cef_browser_settings_t* settings,
    const cef_point_t* inspect_element_at);
```

`cef_browser_t`声明，`cef_browser_host_t`声明比较大，就不放上来了，可以自己去看头文件 (include/capi/cef_browser_capi.h)。

```
///
// Structure used to represent a browser window. When used in the browser
// process the functions of this structure may be called on any thread unless
// otherwise indicated in the comments. When used in the render process the
// functions of this structure may only be called on the main thread.
///
typedef struct _cef_browser_t {
  ///
  // Base structure.
  ///
  cef_base_ref_counted_t base;
 
  ///
  // Returns the browser host object. This function can only be called in the
  // browser process.
  ///
  struct _cef_browser_host_t*(CEF_CALLBACK* get_host)(
      struct _cef_browser_t* self);
 
  ///
  // Returns true (1) if the browser can navigate backwards.
  ///
  int(CEF_CALLBACK* can_go_back)(struct _cef_browser_t* self);
 
  ///
  // Navigate backwards.
  ///
  void(CEF_CALLBACK* go_back)(struct _cef_browser_t* self);
 
  ///
  // Returns true (1) if the browser can navigate forwards.
  ///
  int(CEF_CALLBACK* can_go_forward)(struct _cef_browser_t* self);
 
  ///
  // Navigate forwards.
  ///
  void(CEF_CALLBACK* go_forward)(struct _cef_browser_t* self);
 
  ///
  // Returns true (1) if the browser is currently loading.
  ///
  int(CEF_CALLBACK* is_loading)(struct _cef_browser_t* self);
 
  ///
  // Reload the current page.
  ///
  void(CEF_CALLBACK* reload)(struct _cef_browser_t* self);
 
  ///
  // Reload the current page ignoring any cached data.
  ///
  void(CEF_CALLBACK* reload_ignore_cache)(struct _cef_browser_t* self);
 
  ///
  // Stop loading the page.
  ///
  void(CEF_CALLBACK* stop_load)(struct _cef_browser_t* self);
 
  ///
  // Returns the globally unique identifier for this browser. This value is also
  // used as the tabId for extension APIs.
  ///
  int(CEF_CALLBACK* get_identifier)(struct _cef_browser_t* self);
 
  ///
  // Returns true (1) if this object is pointing to the same handle as |that|
  // object.
  ///
  int(CEF_CALLBACK* is_same)(struct _cef_browser_t* self,
                             struct _cef_browser_t* that);
 
  ///
  // Returns true (1) if the window is a popup window.
  ///
  int(CEF_CALLBACK* is_popup)(struct _cef_browser_t* self);
 
  ///
  // Returns true (1) if a document has been loaded in the browser.
  ///
  int(CEF_CALLBACK* has_document)(struct _cef_browser_t* self);
 
  ///
  // Returns the main (top-level) frame for the browser window. In the browser
  // process this will return a valid object until after
  // cef_life_span_handler_t::OnBeforeClose is called. In the renderer process
  // this will return NULL if the main frame is hosted in a different renderer
  // process (e.g. for cross-origin sub-frames).
  ///
  struct _cef_frame_t*(CEF_CALLBACK* get_main_frame)(
      struct _cef_browser_t* self);
 
  ///
  // Returns the focused frame for the browser window.
  ///
  struct _cef_frame_t*(CEF_CALLBACK* get_focused_frame)(
      struct _cef_browser_t* self);
 
  ///
  // Returns the frame with the specified identifier, or NULL if not found.
  ///
  struct _cef_frame_t*(CEF_CALLBACK* get_frame_byident)(
      struct _cef_browser_t* self,
      int64 identifier);
 
  ///
  // Returns the frame with the specified name, or NULL if not found.
  ///
  struct _cef_frame_t*(CEF_CALLBACK* get_frame)(struct _cef_browser_t* self,
                                                const cef_string_t* name);
 
  ///
  // Returns the number of frames that currently exist.
  ///
  size_t(CEF_CALLBACK* get_frame_count)(struct _cef_browser_t* self);
 
  ///
  // Returns the identifiers of all existing frames.
  ///
  void(CEF_CALLBACK* get_frame_identifiers)(struct _cef_browser_t* self,
                                            size_t* identifiersCount,
                                            int64* identifiers);
 
  ///
  // Returns the names of all existing frames.
  ///
  void(CEF_CALLBACK* get_frame_names)(struct _cef_browser_t* self,
                                      cef_string_list_t names);
} cef_browser_t;
```

我们通过注入 DLL，HOOK CEF 的**事件处理回调函数**，使用回调函数的`struct _cef_browser_t* browser`参数，从而调用到`show_dev_tools`。

 

以按键事件为例  
(代码来自[将 js 代码注入到第三方 CEF 应用程序的一点浅见](https://bbs.pediy.com/thread-268570.htm) 的评论区[风铃 i](https://bbs.pediy.com/user-home-752458.htm) 大佬的评论，我做了一些修改)

```
// dllmain.cpp : 定义 DLL 应用程序的入口点。
#include "pch.h"
#include "detours/detours.h"
#include "include/capi/cef_browser_capi.h"
#include "include/internal/cef_types_win.h"
#include "include/capi/cef_client_capi.h"
#include "include/internal/cef_win.h"
#include PVOID g_cef_browser_host_create_browser = nullptr;
PVOID g_cef_get_keyboard_handler = NULL;
PVOID g_cef_on_key_event = NULL;
 
void SetAsPopup(cef_window_info_t* window_info) {
 
    window_info->style =
        WS_OVERLAPPEDWINDOW | WS_CLIPCHILDREN | WS_CLIPSIBLINGS | WS_VISIBLE;
    window_info->parent_window = NULL;
    window_info->x = CW_USEDEFAULT;
    window_info->y = CW_USEDEFAULT;
    window_info->width = CW_USEDEFAULT;
    window_info->height = CW_USEDEFAULT;
}
 
 
int CEF_CALLBACK hook_cef_on_key_event(
    struct _cef_keyboard_handler_t* self,
    struct _cef_browser_t* browser,
    const struct _cef_key_event_t* event,
    cef_event_handle_t os_event) {
 
    OutputDebugStringA("[detours] hook_cef_on_key_event \n");
 
    auto cef_browser_host = browser->get_host(browser);
 
    // 键盘按下且是F12
    if (event->type == KEYEVENT_RAWKEYDOWN && event->windows_key_code == 123) {
 
        cef_window_info_t windowInfo{};
        cef_browser_settings_t settings{};
        cef_point_t point{};
        SetAsPopup(&windowInfo);
        OutputDebugStringA("[detours] show_dev_tools \n");
 
        // 开启调试窗口
        cef_browser_host->show_dev_tools
            (cef_browser_host, &windowInfo, 0, &settings, &point);
    }
 
    return reinterpret_cast (g_cef_on_key_event)(self, browser, event, os_event);
}
 
 
 
 
struct _cef_keyboard_handler_t* CEF_CALLBACK hook_cef_get_keyboard_handler(
    struct _cef_client_t* self) {
    OutputDebugStringA("[detours] hook_cef_get_keyboard_handler \n");
 
    // 调用原始的修改get_keyboard_handler函数
    auto keyboard_handler = reinterpret_cast (g_cef_get_keyboard_handler)(self);
    if (keyboard_handler) {
 
        // 记录原始的按键事件回调函数
        g_cef_on_key_event = keyboard_handler->on_key_event;
 
        // 修改返回值中的按键事件回调函数
        keyboard_handler->on_key_event = hook_cef_on_key_event;
    }
    return keyboard_handler;
}
 
int hook_cef_browser_host_create_browser(
    const cef_window_info_t* windowInfo,
    struct _cef_client_t* client,
    const cef_string_t* url,
    const struct _cef_browser_settings_t* settings,
    struct _cef_dictionary_value_t* extra_info,
    struct _cef_request_context_t* request_context) {
 
    OutputDebugStringA("[detours] hook_cef_browser_host_create_browser \n");
 
    // 记录原始的get_keyboard_handler
    g_cef_get_keyboard_handler = client->get_keyboard_handler;
 
    // 修改get_keyboard_handler
    client->get_keyboard_handler = hook_cef_get_keyboard_handler;
 
 
    return reinterpret_cast (g_cef_browser_host_create_browser)(
        windowInfo, client, url, settings, extra_info, request_context);
}
 
// Hook cef_browser_host_create_browser
BOOL APIENTRY InstallHook()
{
    OutputDebugStringA("[detours] InstallHook \n");
    DetourTransactionBegin();
    DetourUpdateThread(GetCurrentThread());
    g_cef_browser_host_create_browser =
        DetourFindFunction("libcef.dll", "cef_browser_host_create_browser");
    DetourAttach(&g_cef_browser_host_create_browser,
                 hook_cef_browser_host_create_browser);
    LONG ret = DetourTransactionCommit();
    return ret == NO_ERROR;
}
 
 
BOOL APIENTRY DllMain( HMODULE hModule,
                       DWORD  ul_reason_for_call,
                       LPVOID lpReserved
                     )
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
        InstallHook();
        break;
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
} 
```

这个有个需要注意的点，非常重要 (还记得我上面说的大坑嘛)。我使用的**库的版本和钉钉的不一致**，那么上面代码中使用的结构体声明可能在不同版本会有不同。这意味着我们编译出来的 DLL 中结构体的偏移和钉钉中也可能不一致。

 

注意上面的第 43 行代码，调用`show_dev_tools`

```
cef_browser_host->show_dev_tools
            (cef_browser_host, &windowInfo, 0, &settings, &point);
```

在我实际测试中，`show_dev_tools`的偏移和钉钉中就不一致。当时也是找了很久原因，一开始也没往这方面想，还以为是参数没传对，或者有什么对抗存在。最后在调试的时候和官方例子做了对比，才发现调用的函数都不是`show_dev_tools`！

 

所以我最后改了一下 43 行的代码，`show_dev_tools`偏移差了 4 个字节，用`close_dev_tools`刚好对上。

```
reinterpret_castshow_dev_tools)>
    (cef_browser_host->close_dev_tools)
            (cef_browser_host, &windowInfo, 0, &settings, &point); 
```

在聊天框中 F12，最后终于是开启成功。  
![](https://bbs.pediy.com/upload/attach/202208/861753_A4KJFSYPQRDKK7H.png)

 

最后还要说一点就是 DLL 注入的时机，我选择的是程序在登录框界面的时候。这个时候`libcef.dll`已经加载，`cef_browser_host_create_browser`函数也没被调用。

聊天框防撤回功能
--------

刀已经准备好了，可以试试刀锋了。

 

首先考虑消息撤回的时候大概发生了什么。

 

用户 A 点击撤回 -> 触发 Js 点击事件 -> 向服务器发送网络请求 -> 服务器处理请求，向各个客户端发送消息  
用户 B 收到撤回的请求 ->Js 处理请求，最后修改页面元素  
向服务器发送请求这里有两种可能，一种是直接在 Js 中发送请求，另一种是 Js 代码和 C++ 代码通信 C++ 来发这个请求。钉钉使用的是后者，因为在撤回的时候调试窗口的 Network 页面没有发现有网络请求。  
所以防撤回的实现点有很多种，我这里主要尝试在 Js 层做防撤回。

1.  准备两个钉钉号，其中 A 给 B 发消息
2.  B 收到消息之后，给页面元素下一个**子树修改断点**
3.  断点设置好之后，A 撤回消息
4.  断点命中，观察栈锁定关键点

设置好断点  
![](https://bbs.pediy.com/upload/attach/202208/861753_SUNNXHTYUA8BQ22.png)

 

撤回时断点命中，调用链出来了。阅读代码看看什么地方修改比较合适。  
![](https://bbs.pediy.com/upload/attach/202208/861753_659AYXJZZK3X39S.png)  
找了一圈，发现最顶层的调用处做消息过滤比较合适  
![](https://bbs.pediy.com/upload/attach/202208/861753_22FVQUSZR9YF4ZZ.png)  
修改代码如下，成功防撤回  
![](https://bbs.pediy.com/upload/attach/202208/861753_JQV7FDRCWTJEHQ9.png)

 

这里调试的时候还会遇到一个问题 --**Js 文件太大，调试窗口格式化代码的时候卡死了**。  
解决方法很简单，我们把在`web_content.pak`中找到代码文件把该文件先格式化了，不用调试的时候去格式化，这样调试就不会因为格式化的原因卡死了。

总结
==

CEF 框架是一个开源的框架，而且钉钉也没有加入诸如反调试之内的对抗手段，研究起来比较容易，遇到的一些问题基本都解决了。最大的坑就在于库的版本问题，但是通过调试也能发现端倪。  
最后可以思考一些防御的手段，比如：

*   在加载文件的时候校验文件是否被修改，如果被修改则不加载。
*   在 libcef 库的代码中将调试功能相关代码删除，防止开启调试窗口。
*   或者在 Js 代码中加反调试，增加调试难度，等等等......

可以进行的相关研究还有很多，无聊的时候玩玩也挺好，毕竟 CEF 框架的使用还是挺普遍的。

参考资料
====

*   [框架源码](https://bitbucket.org/chromiumembedded/cef/src/master/)
*   [知乎专栏 - CEF](https://www.zhihu.com/column/c_1333096419650269184)
*   [CEF 桌面软件开发实战](https://juejin.cn/book/7075387142121193502)
*   [将 js 代码注入到第三方 CEF 应用程序的一点浅见](https://bbs.pediy.com/thread-268570.htm)

[[2022 夏季班]《安卓高级研修班 (网课)》月薪三万班招生中～](https://www.kanxue.com/book-section_list-84.htm)

最后于 7 小时前 被 Learn Life 编辑 ，原因：

[#调试逆向](forum-4-1-1.htm)
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247485021&idx=2&sn=01a463886f50ac12637efe86f844fe4d&chksm=ce075718f970de0e2edcbd211170b2616731f508d465bee83381f6183c537a9d8b0937008894&scene=178&cur_album_id=1681422395766538242#rd)

**一、screencap 命令介绍**

  

   安卓系统中 **screencap** 命令是一个屏幕截图命令。以下是该命令的帮助说明:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5431dibicduq0vZf3yX7gdCQGZmvhcBZs0Ih867aQm2yD05BiaTW4sg0lSjZSNQibWku01iaibmNia7Vtoa9Hg/640?wx_fmt=png)

    在终端通过 **adb shell** 命令执行屏幕截图，可以将当前的屏幕保存为 **png** 图片。如下所示:  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5431dibicduq0vZf3yX7gdCQGZme2HX6o3XWzyImRaz4fwfwF4XN316FibAmgSv9hxbUWWia1oAuqfEDPiaw/640?wx_fmt=png)

  

  如果命令不带 **-p** 参数，会将屏幕的原始数据直接输出到终端。  

**二、screencap 源码分析**

     

 **1.screencap 源码路径**

    **screencap** 源码路径位于:

```
//源码路径
frameworks/base/cmds/screencap/screencap.cpp
//编译配置文件，Android.bp代替了Android.mk
frameworks/base/cmds/screencap/Android.bp

```

**2.**screencap** 中的 Android.bp 分析**

     该源码使用 **Android.bp** 文件配置来编译的。**Android.bp** 文件信息如下 (顺便学习 **Android.bp** 的语法了):

```
//cc_binary表示需要编译成二进制可执行文件
cc_binary {
     //name表示编译之后的名称
     name: "screencap",
     //srcs表示编译源文件
     srcs: ["screencap.cpp"],
     //shared_libs表示依赖了哪些动态库
     shared_libs: [
         "libcutils",
         "libutils",
         "libbinder",
         "libhwui",
         "libui",
         "libgui",
     ],
     //cflags表示指定编译器的编译选项
     cflags: [
         "-Wall",
         "-Werror",
         "-Wunused",
         "-Wunreachable-code",
     ],
 }

```

**3. screencap.cpp 分析**  

  **screencap** 中的 **main** 方法如下:

```
int main(int argc, char** argv)
{
    //获取系统的显示列表
    std::optional<PhysicalDisplayId> displayId = SurfaceComposerClient::getInternalDisplayId();
    if (!displayId) {
        fprintf(stderr, "Failed to get token for internal display\n");
        return 1;
    }
    const char* pname = argv[0];
    bool png = false;
    int c;
    //参数解析
    while ((c = getopt(argc, argv, "phd:")) != -1) {
        switch (c) {
            case 'p':
                //指定-p表示生成png图片
                png = true;
                break;
            case 'd':
                displayId = atoll(optarg);
                break;
            case '?':
            case 'h':
                usage(pname, *displayId);
                return 1;
        }
    }
    argc -= optind;
    argv += optind;
    int fd = -1;
    const char* fn = NULL;
    if (argc == 0) {
        fd = dup(STDOUT_FILENO);
    } else if (argc == 1) {
        fn = argv[0];
        fd = open(fn, O_WRONLY | O_CREAT | O_TRUNC, 0664);
        if (fd == -1) {
            fprintf(stderr, "Error opening file: %s (%s)\n", fn, strerror(errno));
            return 1;
        }
        const int len = strlen(fn);
        if (len >= 4 && 0 == strcmp(fn+len-4, ".png")) {
            png = true;
        }
    }
    if (fd == -1) {
        usage(pname, *displayId);
        return 1;
    }
    void const* mapbase = MAP_FAILED;
    ssize_t mapsize = -1;
    void* base = NULL;
    uint32_t w, s, h, f;
    size_t size = 0;
    // setThreadPoolMaxThreadCount(0) actually tells the kernel it's
    // not allowed to spawn any additional threads, but we still spawn
    // a binder thread from userspace when we call startThreadPool().
    // See b/36066697 for rationale
    ProcessState::self()->setThreadPoolMaxThreadCount(0);
    ProcessState::self()->startThreadPool();
    ui::Dataspace outDataspace;
    sp<GraphicBuffer> outBuffer;
    //使用ScreenshotClient屏幕截图操作
    status_t result = ScreenshotClient::capture(*displayId, &outDataspace, &outBuffer);
    if (result != NO_ERROR) {
        close(fd);
        return 1;
    }
    result = outBuffer->lock(GraphicBuffer::USAGE_SW_READ_OFTEN, &base);
    if (base == nullptr || result != NO_ERROR) {
        String8 reason;
        if (result != NO_ERROR) {
            reason.appendFormat(" Error Code: %d", result);
        } else {
            reason = "Failed to write to buffer";
        }
        fprintf(stderr, "Failed to take screenshot (%s)\n", reason.c_str());
        close(fd);
        return 1;
    }
    w = outBuffer->getWidth();
    h = outBuffer->getHeight();
    s = outBuffer->getStride();
    f = outBuffer->getPixelFormat();
    size = s * h * bytesPerPixel(f);
    if (png) {
        //创建图片信息对象
        const SkImageInfo info =
            SkImageInfo::Make(w, h, flinger2skia(f), kPremul_SkAlphaType,
                              dataSpaceToColorSpace(outDataspace));
        SkPixmap pixmap(info, base, s * bytesPerPixel(f));
        struct FDWStream final : public SkWStream {
          size_t fBytesWritten = 0;
          int fFd;
          FDWStream(int f) : fFd(f) {}
          size_t bytesWritten() const override { return fBytesWritten; }
          bool write(const void* buffer, size_t size) override {
            fBytesWritten += size;
            return size == 0 || ::write(fFd, buffer, size) > 0;
          }
        } fdStream(fd);
        //屏幕数据进行编码，这个地方传入的kPNG表示png图片格式编码
        (void)SkEncodeImage(&fdStream, pixmap, SkEncodedImageFormat::kPNG, 100);
        if (fn != NULL) {
            //通知图片视频库更新
            notifyMediaScanner(fn);
        }
    } else {
        //如果没指定生成png，后面直接输出到终端了，一大堆乱码
        uint32_t c = dataSpaceToInt(outDataspace);
        write(fd, &w, 4);
        write(fd, &h, 4);
        write(fd, &f, 4);
        write(fd, &c, 4);
        size_t Bpp = bytesPerPixel(f);
        for (size_t y=0 ; y<h ; y++) {
            write(fd, base, w*Bpp);
            base = (void *)((char *)base + s*Bpp);
        }
    }
    close(fd);
    if (mapbase != MAP_FAILED) {
        munmap((void *)mapbase, mapsize);
    }
    return 0;
}

```

    在以上方法中有一个通知媒体库的调用 notifyMediaScanner，实现代码如下:  

```
 static status_t notifyMediaScanner(const char* fileName) {
      String8 cmd("am broadcast -a android.intent.action.MEDIA_SCANNER_SCAN_FILE -d file://");
      cmd.append(fileName);
      cmd.append(" > /dev/null");
      int result = system(cmd.string());
      if (result < 0) {
          fprintf(stderr, "Unable to broadcast intent for media scanner.\n");
          return UNKNOWN_ERROR;
      }
      return NO_ERROR;
  }

```

    该方法通过 **am** 命令发送广播通知媒体扫描实现图片或者视频加入媒体库。 受该命令启发，如果我们通过 **adb push** 图片或者视频到 **sdcard** 之后，如果需要更新图片或者视频到视频库，可以使用 am 命令发送广播通知。对于通过 **adb** 控制手机来说相当有用。

```
adb shell am broadcast -a android.intent.action.MEDIA_SCANNER_SCAN_FILE -d file://xxxx

```

    **screencap** 官方实现的时候只支持了 **png** 格式的图片格式。通过查找 **SkEncodedImageFormat** 类中的定义，原来是可以通过修改支持很多中图片格式的。比如 **jpeg**、**webp** 等图片格式。**SkEncodedImageFormat** 中定义的所有支持的格式如下:

```
enum class SkEncodedImageFormat {
  #ifdef SK_BUILD_FOR_GOOGLE3
      kUnknown,
  #endif
      kBMP,
      kGIF,
      kICO,
      kJPEG,
      kPNG,
      kWBMP,
      kWEBP,
      kPKM,
      kKTX,
      kASTC,
      kDNG,
      kHEIF,
  };

```

**4、总结**

1.   **通过 am 发送广播命令通知媒体库更新**
    
     通过 adb push 图片或者视频之后，如果想让媒体库立即更新。可以通过发送如下命令通知媒体库扫描。如下:
    
    ```
    adb shell am broadcast -a android.intent.action.MEDIA_SCANNER_SCAN_FILE -d file://xxxx
    
    ```
    

  ** 2. 可以通过修改 screencap 源码中的编码类型生成更小的图片**

       比如如果需要通过 **screencap** 屏幕截图实现电脑同屏，那就需要更小的图片加快传输。

**如果你对安卓系统相关的开发学习感兴趣:**

       可加作者的 QQ 群（1017017661), 本群专注安卓系统方面的技术，欢迎加群技术交流。

 **![](https://mmbiz.qpic.cn/mmbiz_gif/rFWVXwibLGty0S5JgMN8PpBib2631p7cDvlvTEaxFBzljBX9qWcVMSOymhkTd6ZmanRibYWsh0HmccjGWkadiaLwAA/640?wx_fmt=gif)** 点击屏末 ****| **********阅****读****原****文********** |** 查看更多文章**

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjguCKrYZQfRXxK6hibNjOh10JibAdHj553dxk3PmoyUibjDCGcNdq3IQBKA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgibOWZXyrOLic5KPJ2y9A1gznt4xUa1H7MEhlgmcQgnE3IJvphZfOezfA/640?wx_fmt=png)  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgxGibv8NMwbmJuQo55Ry33RkQj6WTGwwyXgrcduXPL3xnUWeLUa3cDvA/640?wx_fmt=png)
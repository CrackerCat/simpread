> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/Bv7bksyGpZDlRI3hut0Bjg)

**一、surfaceflinger 启动**

    **surfaceflinger** 作为显示系统非常重要的服务。在 **init** 启动之后 **init** 进程通过解析 **surfaceflinger.rc** 配置文件进行启动。**surfaceflinger.rc** 文件路径如下:

```
frameworks\native\services\surfaceflinger\surfaceflinger.rc

```

    该文件内容如下:  

```
service surfaceflinger /system/bin/surfaceflinger
    class core animation
    user system
    group graphics drmrpc readproc
    # 如果重启会导致zygote重启
    onrestart restart zygote
    writepid /dev/stune/foreground/tasks
    socket pdx/system/vr/display/client     stream 0666 system graphics u:object_r:pdx_display_client_endpoint_socket:s0
    socket pdx/system/vr/display/manager    stream 0666 system graphics u:object_r:pdx_display_manager_endpoint_socket:s0
    socket pdx/system/vr/display/vsync      stream 0666 system graphics u:object_r:pdx_display_vsync_endpoint_socket:s0

```

      以上脚本可以看出如果 **surfaceflinger** 重启会导致 **zygote** 进程也被重启,**zygote** 进程是整个 **Java** 层进程的祖先, 重启之后会导致整个 **Java** 层系统重启。所以该进程非常重要。

**二、surfaceflinger 中的截屏处理**

**1.SurfaceFlinger 类分析**

      **SurfaceFlinger.cpp** 源文件路径如下:

```
frameworks\native\services\surfaceflinger\SurfaceFlinger.cpp

```

     该文件中类 SurfaceFlinger 定义如下:  

```
class SurfaceFlinger : public BnSurfaceComposer,
                       public PriorityDumper,
                       public ClientCache::ErasedRecipient,
                       private IBinder::DeathRecipient,
                       private HWC2::ComposerCallback {
                ...                       
}

```

    **SurfaceFlinger** 类继承了 **BnSurfaceComposer**。**BnSurfaceComposer** 中定义了服务端处理命令的分发函数 **onTransact**。所以 **SurfaceFlinger** 继承之后也会进行相应的重载实现。

   **SurfaceFlinger** 中 **onTransact** 函数实现如下:

```
status_t SurfaceFlinger::onTransact(uint32_t code, const Parcel& data, Parcel* reply,
                                    uint32_t flags) {
    ...
    //此处使用了父类BnSurfaceComposer的分发函数处理
    status_t err = BnSurfaceComposer::onTransact(code, data, reply, flags);
    ...
}

```

    以上代码中逻辑跳转到 **BnSurfaceComposer** 中的 **onTransact** 函数处理。**BnSurfaceComposer** 类中 **onTransact** 的实现如下:

```
status_t BnSurfaceComposer::onTransact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    switch(code) {
        ...
        //处理屏幕截图
        case CAPTURE_SCREEN: {
            CHECK_INTERFACE(ISurfaceComposer, data, reply);
            sp<IBinder> display = data.readStrongBinder();
            ui::Dataspace reqDataspace = static_cast<ui::Dataspace>(data.readInt32());
            ui::PixelFormat reqPixelFormat = static_cast<ui::PixelFormat>(data.readInt32());
            sp<GraphicBuffer> outBuffer;
            Rect sourceCrop(Rect::EMPTY_RECT);
            data.read(sourceCrop);
            uint32_t reqWidth = data.readUint32();
            uint32_t reqHeight = data.readUint32();
            bool useIdentityTransform = static_cast<bool>(data.readInt32());
            int32_t rotation = data.readInt32();
            bool captureSecureLayers = static_cast<bool>(data.readInt32());
            bool capturedSecureLayers = false;
            //调用方法captureScreen进行真正的截图操作
            status_t res = captureScreen(display, &outBuffer, capturedSecureLayers, reqDataspace,
                                         reqPixelFormat, sourceCrop, reqWidth, reqHeight,
                                         useIdentityTransform,
                                         static_cast<ISurfaceComposer::Rotation>(rotation),
                                         captureSecureLayers);
            reply->writeInt32(res);
            if (res == NO_ERROR) {
                reply->write(*outBuffer);
                reply->writeBool(capturedSecureLayers);
            }
            return NO_ERROR;
        }
        ...
  }

```

‍      以上代码中调用了 **captureScreen** 方法进行截屏。该方法定义在类 **ISurfaceComposer** 中。以下是定义的代码:

```
class ISurfaceComposer: public IInterface {
    ...
    virtual status_t captureScreen(const sp<IBinder>& display, sp<GraphicBuffer>* outBuffer,
                                   bool& outCapturedSecureLayers, const ui::Dataspace reqDataspace,
                                   const ui::PixelFormat reqPixelFormat, Rect sourceCrop,
                                   uint32_t reqWidth, uint32_t reqHeight, bool useIdentityTransform,
                                   Rotation rotation = eRotateNone,
                                   bool captureSecureLayers = false) = 0;
    ...
}

```

     **captureScreen** 定义为虚方法，所以子类会实现该接口。在 **SurfaceFlinger** 类中找到了对应实现的逻辑方法，实现逻辑如下:

```
status_t SurfaceFlinger::captureScreen(const sp<IBinder>& displayToken,
                                       sp<GraphicBuffer>* outBuffer, bool& outCapturedSecureLayers,
                                       const Dataspace reqDataspace,
                                       const ui::PixelFormat reqPixelFormat, Rect sourceCrop,
                                       uint32_t reqWidth, uint32_t reqHeight,
                                       bool useIdentityTransform,
                                       ISurfaceComposer::Rotation rotation,
                                       bool captureSecureLayers) {
    ...
    //调用captureScreenCommon
    return captureScreenCommon(renderArea, traverseLayers, outBuffer, reqPixelFormat,
                               useIdentityTransform, outCapturedSecureLayers);
}

```

     以上代码中逻辑转到 **captureScreenCommon** 中。**captureScreenCommon** 实现逻辑如下:

```
status_t SurfaceFlinger::captureScreenCommon(RenderArea& renderArea,
                                             TraverseLayersFunction traverseLayers,
                                             sp<GraphicBuffer>* outBuffer,
                                             const ui::PixelFormat reqPixelFormat,
                                             bool useIdentityTransform,
                                             bool& outCapturedSecureLayers) {
    ...
    //调用重载的captureScreenCommon
    return captureScreenCommon(renderArea, traverseLayers, *outBuffer, useIdentityTransform,
                               outCapturedSecureLayers);
}

```

    以上调用的 captureScreenCommon 对应的实现逻辑如下:

```
status_t SurfaceFlinger::captureScreenCommon(RenderArea& renderArea,
                                             TraverseLayersFunction traverseLayers,
                                             const sp<GraphicBuffer>& buffer,
                                             bool useIdentityTransform,
                                             bool& outCapturedSecureLayers) {
    // This mutex protects syncFd and captureResult for communication of the return values from the
    // main thread back to this Binder thread
    ...
    //调用了captureScreenImplLocked
                result = captureScreenImplLocked(renderArea, traverseLayers, buffer.get(),
                                                 useIdentityTransform, forSystem, &fd,
    ...
}

```

     **captureScreenImplLocked** 的实现逻辑如下:

```
status_t SurfaceFlinger::captureScreenImplLocked(const RenderArea& renderArea,
                                                 TraverseLayersFunction traverseLayers,
                                                 ANativeWindowBuffer* buffer,
                                                 bool useIdentityTransform, bool forSystem,
                                                 int* outSyncFd, bool& outCapturedSecureLayers) {
    ...
    // We allow the system server to take screenshots of secure layers for
    // use in situations like the Screen-rotation animation and place
    // the impetus on WindowManager to not persist them.
    //判断当前屏幕是否允许截屏，此处可以知道我们开发App中防止截屏系统是在这个处理的
    if (outCapturedSecureLayers && !forSystem) {
        ALOGW("FB is protected: PERMISSION_DENIED");
        return PERMISSION_DENIED;
    }
    //调用renderScreenImplLocked方法
    renderScreenImplLocked(renderArea, traverseLayers, buffer, useIdentityTransform, outSyncFd);
    return NO_ERROR;
}

```

    以上方法判断了当前界面是否可以截屏。已经使用了 **renderScreenImplLocked** 继续进行截屏逻辑处理。 **renderScreenImplLocked** 实现逻辑如下:

```
void SurfaceFlinger::renderScreenImplLocked(const RenderArea& renderArea,
                                            TraverseLayersFunction traverseLayers,
                                            ANativeWindowBuffer* buffer, bool useIdentityTransform,
                                            int* outSyncFd) {
    ...
    base::unique_fd bufferFence;
    base::unique_fd drawFence;
    getRenderEngine().useProtectedContext(false);
    //使用渲染引擎绘制当前的界面到buffer
    getRenderEngine().drawLayers(clientCompositionDisplay, clientCompositionLayers, buffer,
                                 /*useFramebufferCache=*/false, std::move(bufferFence), &drawFence);
    *outSyncFd = drawFence.release();
}

```

**三、总结**

*    安卓系统中屏幕截图最终在 **surfaceflinger** 服务进程中处理
    
*   安卓 **App** 开发中的界面防止截屏，最终逻辑判断是在 **surfaceflinger** 中进行判断
    
    以下是 **App** 开发中**防止屏幕截屏**的参考关键代码:
    
    ```
    getActivity().getWindow().addFlags(WindowManager.LayoutParams.FLAG_SECURE);
    
    ```
    

**如果你对安卓相关的开发学习感兴趣:**

       可加作者的 **QQ** 群（**1017017661**), 本群专注安卓方面的技术，欢迎加群技术交流。

 **![](https://mmbiz.qpic.cn/mmbiz_gif/rFWVXwibLGty0S5JgMN8PpBib2631p7cDvlvTEaxFBzljBX9qWcVMSOymhkTd6ZmanRibYWsh0HmccjGWkadiaLwAA/640?wx_fmt=gif)** 点击屏末 ****| **********阅****读****原****文********** |** 查看更多文章**

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjguCKrYZQfRXxK6hibNjOh10JibAdHj553dxk3PmoyUibjDCGcNdq3IQBKA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgibOWZXyrOLic5KPJ2y9A1gznt4xUa1H7MEhlgmcQgnE3IJvphZfOezfA/640?wx_fmt=png)  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgxGibv8NMwbmJuQo55Ry33RkQj6WTGwwyXgrcduXPL3xnUWeLUa3cDvA/640?wx_fmt=png)
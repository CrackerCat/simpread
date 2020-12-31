> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=Mzg2NzUzNzk1Mw==&mid=2247483953&idx=1&sn=77fe73729da780a26f3ae56579617e48&chksm=cebb477ff9ccce690062c28fe2e933f427d4d7b0378446c4230ebe936a2e0b003123cfffac74&scene=21#wechat_redirect)

一、Audio 音频架构简介

二、Android Audio 系统框架

三、Audio 架构以及各层的代码分布图

四、音频框架在 Android 系统中的进一步细化

五、创建声卡和注册声卡

六、Android Audio 系统的结构

七、Audio 音频原理介绍

八、Audio 音频策略制定与策略执行的调用流程

九、Android AudioPolicyService 服务启动过程

十、Android 系统中所有的音频接口设备保存到 AudioFlinger 的成员变量 mAudioHwDevs 中

十一、audio_policy.conf 同时定义了多个 audio 接口

十二、通过 AudioFlinger 的 loadHwModule 加载各 audio 接口对应的库文件实现调用 PlaybackThread 播放线程或 RecordThread 录音线程

十三、AudioFlinger 的 openInput() 方法调用流程分析

十四、AudioFlinger 的 openOutput() 方法的调用流程分析

十五、Audio 系统为了能正常播放音频数据，需要创建抽象的音频输出接口对象，打开音频输出过程

十六、打开音频输入的流程

十七、打开音频输出后，在 AudioFlinger 与 AudioPolicyService 中的表现形式

十八、打开音频输入后，在 AudioFlinger 与 AudioPolicyService 中的表现形式

十九、AudioPolicyService 加载完系统定义的所有音频接口，并生成相应的数据对象

二十、AudioPolicyService 与 AudioTrack 和 AudioFlinger 的关系

二十一、AudioPolicyService 注册名为服务的流程

二十二、AudioTrack 构造过程

二十三、AudioTrack 和 AudioFlinger 的关系

二十四、audio_policy 与 AudioPolicyService、AudioPolicyCompatClient 之间的关系

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhcN5vMvaCvia4HjL6ibP3wm6gSo7Q9Z8ZOcHxTWlqkSOL2Aab2Picweu5bQ/640?wx_fmt=png)

**一、Audio 音频架构简介**

**APP**

整个音频体系的最上层

**Framework**

MediaPlayer 和 MediaRecorder、AudioTrack 和 AudioRecorder，Android 系统为控制音频系统提供了 AudioManager、AudioService 及 AudioSystem 类，这些都是 framework 为便利上层应用开发所设计的

**Libraries**

系统服务 AudioFlinger 和 AudioPolicyService(比如: ServiceManager、LocationManagerService、ActivityManagerService 等等)，音频体系中另一个重要的系统服务是 MediaPlayerService

**HAL**

硬件抽象层是 AudioFlinger 直接访问的对象，这说明了两个问题，一方面 AudioFlinger 并不直接调用底层的驱动程序; 另一方面，AudioFlinger 上层 (包括和它同一层的 MediaPlayerService) 的模块只需要与它进行交互就可以实现音频相关的功能了。因而我们可以认为 AudioFlinger 是 Android 音频系统中真正的“隔离板”，无论下面如何变化，上层的实现都可以保持兼容。音频方面的硬件抽象层主要分为两部分，即 AudioFlinger 和 AudioPolicyService。实际上后者并不是一个真实的设备，只是采用虚拟设备的方式来让厂商可以方便地定制出自己的策略，抽象层的任务是将 AudioFlinger/AudioPolicyService 真正地与硬件设备关联起来

以前 Android 系统中的 Audio 系统依赖于 ALSA-lib，但后期就变为了 tinyalsa，这样的转变不应该对上层造成破坏。因而 Audio HAL 提供了统一的接口来定义它与 AudioFlinger/AudioPolicyService 之间的通信方式，这就是 audio_hw_device、audio_stream_in 及 audio_stream_out 等等存在的目的，这些 Struct 数据类型内部大多只是函数指针的定义，是一些 “壳”。当 AudioFlinger/AudioPolicyService 初始化时，它们会去寻找系统中最匹配的实现(这些实现驻留在以 audio.primary.*,audio.a2dp.* 为名的各种库中) 来填充这些“壳”

**理解 Android 音频系统的时候分为两条线索**

以库为线索，比如: AudioPolicyService 和 AudioFlinger 都是在 libaudioflinger 库中，而 AudioTrack、AudioRecorder 等一系列实现则在 libmedia 库中

以进程为线索，库并不代表一个进程，进程则依赖于库来运行。虽然有的类是在同一个库中实现的，但并不代表它们会在同一个进程中被调用。比如 AudioFlinger 和 AudioPolicyService 都驻留于名为 mediaserver 的系统进程中，而 AudioTrack/AudioRecorder 和 MediaPlayer/MediaRecorder 一样实际上只是应用进程的一部分，它们通过 binder 服务来与其它系统进程通信

**二、Android Audio 系统框架**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhc7rSI08ODIy4bHbiczWAg1Yv68VStf7dibogMf9PnRsCAcbmDVu9YOdGA/640?wx_fmt=png)

**三、Audio 架构以及各层的代码分布图**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhcVO31MUnGP3tWZOWywP48Fq6CBLxE979Z6dZIYzib0zSRtXaekqgDzhA/640?wx_fmt=png)

**四、音频框架在 Android 系统中的进一步细化**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhcpU4PfTkFOv6CBWENq6y7JoHDFZcLOibQiap5q6kDDkfrME6H3m2hvGEg/640?wx_fmt=png)

**五、创建声卡和注册声卡**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhcTAAPTFCKWtI679xtgyjibv7U6LTdsR0KolcdrIyqX2WwgIuYYqulziag/640?wx_fmt=png)

**六、Android Audio 系统的结构**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhcRPbGlN5Rtibibzddw6RiawgI1qoWzru1A2mQeqFcVVasKv3BL5zuxvPIA/640?wx_fmt=png)

**七、Audio 音频原理介绍**

AudioFlinger、AudioPolicyService 和 AudioTrack/AudioRecorder 抛开 MediaPlayer、MediaRecorder 这些与应用开发直接关联的部分，整个音频系统的核心就是由这三者构建而成的。其中前两个都是 System Service，驻留在 mediaserver 进程中，不断地处理 AudioTrack/AudioRecorder 的请求。音频的回放和录制从大的流程上看都是相似的，所以我们侧重于对 AudioTrack 的分析

把所有媒体相关的 native 层服务 (包括 AudioFlinger,MediaPlayerService,CameraService 和 AudioPolicyService) 启动起来, 编译生成的 mediaserver 将被烧录到设备的 / system/bin/mediaserver 路径, 然后由系统启动时的 init 进程启动

**Audio 系统的结构**

libmedia.so 提供 Audio 接口，这些 Audio 接口既向上层开放，也向本地代码开放

libaudiofilnger.so 提供 Audio 接口实现

Audio 硬件抽象层提供到硬件的接口，供 AudioFlinger 调用

Audio 使用 JNI 和 JAVA 对上层提供接口

**media 库中的 Audio 框架部分**

Android 的 Audio 的核心框架在 media 库中提供，其中对上面主要实现 AudioSystem、AudioTrack 和 AudioRecorder 三个类。提供了 IAudioFlinger 类接口，在这个类中，可以获得 IAudioTrack 和 IAudioRecorder 两个接口，分别用于声音的播放和录制。AudioTrack 和 AudioRecorder 分别通过调用 IAudioTrack 和 IAudioRecorder 来实现

**Audio 系统的头文件**

路径为: frameworks/base/include/media/

AudioSystem.h

IAudioFlinger.h

AudioTrack.h

IAudioTrack.h

AudioRecorder.h

IAudioRecorder.h

Ixxx 的接口通过 AudioFlinger 来实现，其他接口通过 JNI 向上层提供接口

**Audio 系统的头文件在 frameworks/base/include/media / 目录中，主要的头文件如下**

AudioSystem.h：media 库的 Audio 部分对上层的总管接口

IAudioFlinger.h：需要下层实现的总管接口

AudioTrack.h：放音部分对上接口

IAudioTrack.h：放音部分需要下层实现的接口

AudioRecorder.h：录音部分对上接口

IAudioRecorder.h：录音部分需要下层实现的接口

IAudioFlinger.h、IAudioTrack.h 和 IAudioRecorder.h 这三个接口通过下层的继承来实现 (即: AudioFlinger)

AudioFlinger.h，AudioTrack.h 和 AudioRecorder.h 是对上层提供的接口，它们既供本地程序调用 (例如: 声音的播放器、录制器等)，也可以通过 JNI 向 Java 层提供接口

AudioTrack 和 AudioRecorder 都具有 start，stop 和 pause 等接口。前者具有 write 接口，用于声音的播放，后者具有 read 接口，用于声音的录制

AudioSystem 用于 Audio 系统的控制工作，主要包含一些 set 和 get 接口，是一个对上层的类

**AudioFlinger 是 Audio 系统的核心，来自 AudioTrack 的数据，最终在这里得到处理并被写入 Audio HAL 层**

MediaPlayer 在 framework 层还是会创建 AudioTrack，把解码后的 PCM 数流传递给 AudioTrack，AudioTrack 再传递给 AudioFlinger 进行混音，然后才传递给硬件播放, 所以是 MediaPlayer 包含了 AudioTrack。使用 AudioTrack 播放音乐

MediaPlayer 提供了更完整的封装和状态控制, 相比 MediaPlayer，AudioTrack 更为精练、高效，实际上 MediaPlayerService 的内部实现就是使用了 AudioTrack 把所有媒体相关的 native 层服务 (包括 AudioFlinger,MediaPlayerService,CameraService 和 AudioPolicyService) 启动起来, 编译生成的 mediaserver 将被烧录到设备的 / system/bin/mediaserver 路径, 然后由系统启动时的 init 进程启动

**两种 Audio Hardware HAL 接口定义**

legacy：hardware/libhardware_legacy/include/hardware_legacy/AudioHardwareInterface.h

非 legacy：hardware/libhardware/include/hardware/audio.h

前者是 2.3 及之前的音频设备接口定义, 后者是 4.0 的接口定义, 为了兼容以前的设计, 4.0 实现一个中间层: hardware/libhardware_legacy/audio/audio_hw_hal.cpp, 结构与其他的 audio_hw.c 大同小异, 差别在于 open 方法事实上 legacy 也要封装成非 legacy 中的 audio.h，确切的说需要一个联系 legacy interface 和 not legacy interface 的中间层，这里的 audio_hw_hal.cpp 就充当这样的一个角色了

hardware/libhardware/modules/audio/

createAudioHardware() 函数

external/tinyalsa/

mixer.c      类 alsa-lib 的 control，作用音频部件开关、音量调节等

pcm.c        类 alsa-lib 的 pcm，作用音频 pcm 数据回放录制

上面的 hardware/libhardware_legacy/audio/、hardware/libhardware/modules/audio/、device/samsung/tuna/audio / 是同层的。之一是 legacy audio，用于兼容 2.2 时代的 alsa_sound；之二是 stub audio 接口；之三是 Samsung Tuna 的音频抽象层实现。调用层次：AudioFlinger -> audio_hw -> tinyalsa

Audio 硬件抽象层的实现在各个系统中可能是不同的，需要使用代码去继承相应的类并实现它们，作为 Android 系统本地框架层和驱动程序接口 AudioFlinger 继承了 libmedia.so(Audio 本地框架类) 里面的接口，上层调用的只是 libmedia.so 部分的接口, 但实际上调用的内容是 libaudioflinger.so, 使用 JNI 和 Java 对上层提供接口，JNI 部分通过调用 libmedia.so 库提供的接口来实现

Audio 硬件抽象层提供到硬件的接口，供 AudioFlinger 调用，Audio 的硬件抽象层实际上是各个平台开发过程中需要主要关注和独立完成的部分，因为 Android 中的 Audio 系统不涉及编解码环节，只负责上层系统和底层 Audio 硬件的交互，所以通常以 PCM 作为输入 / 输出格式

IAudioFlinger 类接口通过该类可以获得 IAudioTrack 和 IAudioRecorder 两个接口，分别用于声音的播放和录制, AudioTrack 和 AudioRecorder 分别通过调用 IAudioTrack 和 IAudioRecorder 来实现

 硬件抽象层主要实现了 AudioStreamInALSA 和 AudioStreamOutALSA 两个类，这两个类又会调用该文件下的 ALSAStreamOps 类的方法。AudioStreamInALSA 是录音部分调用的路径。在 AudioStreamInALSA 的构造函数中会对 alsa 进行一些初始化参数设置，

AudioStreamInALSA 的 read 方法是最主要的方法，audioflinger 层的 read 调用就是对 AudioStreamInALSA 的 read 的调用。由于

录音部分出现单声道和双声道数据传输的问题，修改 read 方法如下，即可实现了录音功能正常， 避免了在编码的时候修改数据时其他编码仍不能工作的弊端

**八、Audio 音频策略制定与策略执行的调用流程**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhczjUwialpplTAZhygU87cmG1qthzbs1xZewibCvbNub4Z6ib3YfVJwln9Q/640?wx_fmt=png)

AudioPolicyService 是策略的制定者, AudioFlinger 则是策略的执行者

AudioTrack 是 AudioFlinger 的客户端，AudioFlinger 是 Android 系统中 Audio 管理的中枢

在 Android 中 AudioPolicyService 最终都会调用到 AudioFlinger 中去，因为 AudioFlinger 实际创建并管理了硬件设备

AudioFlinger 类是代表整个 AudioFlinger 服务的类，其余所有的工作类都是通过内部类的方式在其中定义的

**九、Android AudioPolicyService 服务启动过程**

**AudioPolicyService 完成的工作**

加载 audio_policy.default.so 库得到 audio_policy_module 模块

通过 audio_policy_module 模块打开 audio_policy_device 设备

通过 audio_policy_device 设备创建 audio_policy

**hw_get_module 函数加载硬件抽象层模块的过程**

audio_policy 实现在 audio_policy_hal.cpp 中，audio_policy_service_ops 实现在 AudioPolicyService.cpp 中。create_audio_policy() 函数就是创建并初始化一个 legacy_audio_policy 对象，AudioPolicyCompatClient 是对 audio_policy_service_ops 的封装类，对外提供 audio_policy_service_ops 数据结构中定义的接口

**Android AudioPolicyService 服务启动过程**

引用 AudioPolicyCompatClient 对象，这样音频管理器 AudioPolicyManager 就可以使用 audio_policy_service_ops 中的接口

优先加载 / vendor/etc/audio_policy.conf 配置文件，如果该配置文件不存在，则加载 / system/etc/audio_policy.conf 配置文件，如果该文件还是不存在，则通过函数 defaultAudioPolicyConfig() 来设置默认音频接口

设置各种音频流对应的音量调节点

通过名称打开对应的音频接口硬件抽象库

打开 mAttachedOutputDevices 对应的输出

将输出 IOProfile 封装为 AudioOutputDescriptor 对象

设置当前音频接口的默认输出设备

打开输出，在 AudioFlinger 中创建 PlaybackThread 线程，并返回该线程的 id

设置可以使用的输出设备为 mAttachedOutputDevices

将输出描述符对象 AudioOutputDescriptor 及创建的 PlaybackThread 线程 id 以键值对形式保存

设置默认输出设备

**AudioPolicyManagerBase 对象构造过程中主要完成以下几个步骤**

loadAudioPolicyConfig(AUDIO_POLICY_CONFIG_FILE) 加载 audio_policy.conf 配置文件

initializeVolumeCurves() 初始化各种音频流对应的音量调节点

加载 audio policy 硬件抽象库：mpClientInterface->loadHwModule(mHwModules[i]->mName)

打开 attached_output_devices 输出 mpClientInterface->openOutput();

保存输出设备描述符对象 addOutput(output, outputDesc);

**十、Android 系统中所有的音频接口设备保存到 AudioFlinger 的成员变量 mAudioHwDevs 中**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhcshbmwx7xmjCc1h1hUwCK0Kmm6Xm1pHnttSG7TDqMEeh4wUR1cs827g/640?wx_fmt=png)

**audio_policy.conf 文件**

从 audio_policy.conf 文件中可以发现，系统包含了 primary、a2dp、usb 等音频接口，对应着系统中的 audio.<primary/a2dp/usb>.<device>.so。每个音频接口中又包含了若干个 outputs & inputs，并且每个 output or input 又包含了若干个 devices，且还有采样频率，声道数等信息。这些 devices 信息、采样频率信息 & 声道信息等都会保存在各自 module 的 IOProfile 中。按上文中 audio_policy.conf 配置文件所描述，系统最后会生成 6 个 modules(eg.primary,a2dp,hdmi,r_submix,hs_usb & usb) 以及 7 个 outputs

**十一、audio_policy.conf 同时定义了多个 audio 接口**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhc40TrFicHrtefoy58ibynoZSkiaFmbpa2ORIiacqkEAKdmVOic6qG0kXoLDw/640?wx_fmt=png)

不同的 Android 产品在音频的设计上通常是存在差异的，而这些差异可以通过 Audio 的配置文件 audio_policy.conf 来获得。在 Android 系统中音频配置文件存放路径有两处，存放地址可以从 AudioPolicyManagerBase.cpp 文件中获取

在 AudioPolicyManager.cpp 文件中可以知道系统会首先加载 vendor/etc 目录下的 configure 文件，再加载 system/etc 目录下的 configure 文件。若这两者加载都发生错误的话，系统会加载 default 配置文件，并命名为 primary module

通过 loadGlobalConfig(root)函数来读取这些全局配置信息，通过 loadHwModules()函数来加载系统配置的所有 audio 接口 (加载音频接口)，由于 audio_policy.conf 可以定义多个音频接口，因此该函数循环调用 loadHwModule() 来解析每个音频接口参数信息。Android 定义 HwModule 类来描述每一个 audio 接口参数，定义 IOProfile 类来描述输入输出模式配置

/system/etc/audio_policy.conf

/vendor/etc/audio_policy.conf

**加载 audio_module 模块**

AudioPolicyManager 通过读取 audio_policy.conf 配置文件，可以知道系统当前支持哪些音频接口以及 attached 的输入输出设备、默认输出设备。接下来就需要加载这些音频接口的硬件抽象库

AudioPolicyClientInterface 提供了加载音频接口硬件抽象库的接口函数，AudioPolicyCompatClient 通过代理 audio_policy_service_ops 实现 AudioPolicyClientInterface 接口

AudioPolicyCompatClient 将音频模块加载工作交给 audio_policy_service_ops，AudioPolicyService 又将其转交给 AudioFlinger 

**当 AudioPolicyManagerBase 构造时，它会根据用户提供的 audio_policy.conf 来分析系统中有哪些 audio 接口 (primary,a2dp 以及 usb), 然后通过 AudioFlinger::loadHwModule 加载各 audio 接口对应的库文件，并依次打开其中的 output(openOutput) 和 input(openInput)**

打开音频输出时创建一个 audio_stream_out 通道，并创建 AudioStreamOut 对象以及新建 PlaybackThread 播放线程

打开音频输入时创建一个 audio_stream_in 通道，并创建 AudioStreamIn 对象以及创建 RecordThread 录音线程 

**audio_policy.conf 文件定义了两种音频配置信息** 

当前系统支持的音频输入输出设备及默认输入输出设备，这些信息是通过 global_configuration 配置项来设置，在 global_configuration 中定义了三种音频设备信息

attached_output_devices：已连接的输出设备

default_output_device：默认输出设备

attached_input_devices：已连接的输入设备

系统支持的音频接口信息

audio_policy.conf 定义了系统支持的所有音频接口参数信息，比如 primary、a2dp、usb 等

每种音频接口包含输入输出，每种输入输出又包含多种输入输出配置，每种输入输出配置又支持多种音频设备。AudioPolicyManagerBase 首先加载 / vendor/etc/audio_policy.conf，如果该文件不存在，则加载 / system/etc/audio_policy.conf

audio_policy.conf 同时定义了多个 audio 接口，每一个 audio 接口包含若干 output 和 input，而每个 output 和 input 又同时支持多种输入输出模式，每种输入输出模式又支持若干种设备

**十二、通过 AudioFlinger 的 loadHwModule 加载各 audio 接口对应的库文件实现调用 PlaybackThread 播放线程或 RecordThread 录音线程**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhcWmlcfSJWGpmPJuCUAUMjRrhAwGicricGTRGI1volFXXO4a1DDxfricOcg/640?wx_fmt=png)

**十三、AudioFlinger 的 openInput() 方法调用流程分析**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhcoZCdnl5QIAfz5NvdtyxK7vsq7ImN96OGZ2iaZebuanicnjQpvGX4kSIw/640?wx_fmt=png)

**十四、AudioFlinger 的 openOutput() 方法的调用流程分析**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhcesnZ4uQVsO0ej4qarValMrPpuMmdyY3zNZicLuen3HJJGxdxkslzj3w/640?wx_fmt=png)

**十五、Audio 系统为了能正常播放音频数据，需要创建抽象的音频输出接口对象，打开音频输出过程**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhcxmjk6S33VbNiaciajl7LOLtYI8IIHQPh6vCksENygKmhZvSMD41brIDA/640?wx_fmt=png)

**十六、打开音频输入的流程**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhczaXD7SGCME41axHTcTL2SVwbnL9BGrxgkFPqG2ibLn5QAoqcNDt3V7w/640?wx_fmt=png)

**十七、打开音频输出后，在 AudioFlinger 与 AudioPolicyService 中的表现形式**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhctRJID87XBvOTUFiaT0CfbqR17y10RHpuLs93icGehibibbcZicGxK0hmhEQ/640?wx_fmt=png)

**十八、打开音频输入后，在 AudioFlinger 与 AudioPolicyService 中的表现形式**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhcqyOqFLTpAkOoz4M2zazNLRoBibPljXA6yELiaCj1xlMUficficCnN8xWibQ/640?wx_fmt=png)

**十九、AudioPolicyService 加载完系统定义的所有音频接口，并生成相应的数据对象**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhcWiaeicxpr4gCfYFNePMcKgeoibzLZMUDfqOF4JtLTnbvUB5R0K9oFNkKA/640?wx_fmt=png)

**二十、AudioPolicyService 与 AudioTrack 和 AudioFlinger 的关系**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhcicOSsXybXnAAeD7xaxz0TsTdshLcIjyL1pWOOGp6bSqricdo8P5kNcQg/640?wx_fmt=png)

**二十一、AudioPolicyService 注册名为服务的流程**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhcSuUdv2rXZcfIXzusDTlPwiaEtt3aVkNPzPr8jIdxrLmOibmlSI692KRw/640?wx_fmt=png)

**二十二、AudioTrack 构造过程**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhcqMs9CCOEEVVvPyPI5z4UTN3hLFqQC1q7maRkBmpWO4Vo8MNAsQCR5g/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhc31CYjkKqvhYmAiakMr2Itia3L3Wqap7NsJnQT3tiblZA0SuNDKVVeRAgg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhc0YQdXeIia4dqV0geMTo1ia1UqTRZtMicFynZZJibic5hYPpWphqibfeQbc8g/640?wx_fmt=png)

**二十三、AudioTrack 和 AudioFlinger 的关系**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhc00zVEazuXKRpO0IxKpOow9jbMHouxek4ugVsjr5C7BoIqSBPWHpVtA/640?wx_fmt=png)

**二十四、audio_policy 与 AudioPolicyService、AudioPolicyCompatClient 之间的关系**

![](https://mmbiz.qpic.cn/mmbiz_png/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhcIic0w90icHbV9VDRUODialHhGLzTcZWRDdPDTjgicCXJ3HibKmEUEE8ricZw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_jpg/LtmuVIq6tF2a452k3XDOb8yy5AvAfVhcnZ8Hx6Tkk07AYV3Spx5VuARMqSMRhW8Fic1pNc1uKlV1ib1Znl6BXkibw/640?wx_fmt=jpeg)

欢迎关注公众号
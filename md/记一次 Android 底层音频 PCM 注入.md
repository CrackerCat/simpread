> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2094538-1-1.html)

> [md]# Android 底层音频逆向 ((我好菜我好菜 ## 背景 ### * 契机在开发 , `Alicla`QQ bot 中 , 我希望它拥有对别人打电话的能力 , 翻遍了整个 napcat 代码也未找到 ......

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)mignonRex

Android 底层音频逆向 ((我好菜我好菜
-----------------------

### 背景

#### * 契机

在开发, `Alicla`QQ bot 中, 我希望它拥有对别人打电话的能力, 翻遍了整个 napcat 代码也未找到 **QQ 网络电话** API, 那这就很难受了, 总得有原因吧? 原来 QQ 电话的底层协议较为复杂, napcat 出现应该较久, 彼时无 LLM, 都是指令 Bot, 且拥有此需求的人很少. 那只能往现实中的 运营商电话凑了.

#### * 第二座大山

在研究 LLM 中, 发现 minimax 模型 似乎拥有 根据一小段 `音频` 模拟音色的能力, 这下声音也有了, 就差电话了. 做应用第一件事就是找 API, 但是囿于电诈防范等, 需要申请 / 购买 API 的必须有企业资质, 而且 500 分钟也需要 49 CNY, 宰客这一块, 那没办法, 只能自己手搓了

### 无 Root 尝试

第一次尝试是在自己的 Google pixel 9 上.

先罗列一下自己需要什么.

#### * 要求:

*   可以全天跑在手机不死
*   编译速度快, 适配 arm64, 可定向编译, 不需要在手机上编译 (因为在手机上下个终端真的很 ben)
*   可以劫持通话收音
*   可以劫持通话音频注入
*   可以自己唤起屏幕
*   可以自己调节音量
*   可以自己指定拨通电话
*   可以自己挂掉电话
*   可以自己锁屏

这样局势就明朗了, 我们先在非 root 上试试, 全天不死? /data/local/tmp 是个好去向, 速度快, 适配 arm? 这不就是 golang 的主场吗, 唤起屏幕, 一行命令搞定, 调节音量, 指定拨通电话, 挂掉电话, 锁屏亦是如此, 远程操控? `tailscale`组网是闹着玩的吗

###### 现在只剩最难最关键的, 可以劫持通话收音, 劫持通话音频注入

###### 如何解决?

#### 初代解决方案

go 作为后端在 / data/local/tmp 下, 写一个简单的 android APK (无障碍服务) 提权, `adb install helper.apk` 安装后, 此时能监听到声音, Great!

###### 但是, 现在拥有几个问题, 监听完了, 推送咋办? 莫非我要跟那些劣质机器人一样, 用耳机线短接耳机的麦克风和耳机的扬声器, 给它放声音吗? 这也太傻了. 而且监听音质还一般

于是亦然决然转 root

### Root 尝试

#### * 拦路虎依旧, 戏剧性结尾

我有一张广电的卡, 移动的卡, 以及小米移动的卡, 移动卡是主卡, 因此只能从 广电, 移动找机会. 不过广电连 LTE 都亮不起来, shizuku 与 Pixel IMS 完全救不回来, 这样看小米算最优选, 但是我取卡针找不到了! 只能撕开一个口罩, 拿那个铁丝折两半开.

##### Pixel 软限制

作为 google 的亲儿子, pixel 对这些奇奇怪怪的运营商, 有着天然的抵触, 因此默认掉 `策略` 打不了电话, 那怎么办呢?  

换 APN, 换完看 *#*#4636#*#* 状态, 也没啥问题, 但是打不了, 那就改策略, 去网上找一个 Chinese-Carrier-Pixel-3-xl-LTE-VoLTE-Enabler 模块, 想用 Magisk 刷入, 但是 github zip 压缩压两层 , 那我重新解压再来, 结果我的 Magisk Alpha 的 UI 死了, 打不开, 那只能 adb 转 shell 转 su, 清基带缓存后, magisk --install-module /path/xxx.zip, 刷入新的策略表, 打电话看似完美了, 但是等会换卡的时候, 好, 又死了, 折腾了很久,,, 没想到还是国内搞的 volte 模块管用...

#### * 景阳冈后的天晴雨雪

##### 天晴

那还说啥呢, 直接火力全开, 不需要去无障碍绕过了这样, 直接就是

有 Root 就好用多了, 我的 Id 是 0 , 他们直接放行, 用 d8 编译 dex 由 go 拉起注入如下:

```
package com.mignon.tools;

import android.media.AudioFormat;
import android.media.AudioRecord;
import java.io.OutputStream;

public class AudioCapture {
    public static void main(String[] args) {

        try {
            // 采样率 16000Hz (双向通话原生最佳采样率)
            int sampleRate = 16000;
            int minBufSize = AudioRecord.getMinBufferSize(sampleRate, AudioFormat.CHANNEL_IN_MONO, AudioFormat.ENCODING_PCM_16BIT);
            AudioRecord recorder = new AudioRecord(
                    4,
                    sampleRate,
                    AudioFormat.CHANNEL_IN_MONO,
                    AudioFormat.ENCODING_PCM_16BIT,
                    minBufSize * 4
            );

            if (recorder.getState() == AudioRecord.STATE_INITIALIZED) {
                recorder.startRecording();

                OutputStream out = System.out;
                byte[] buffer = new byte[minBufSize];
                while (true) {
                    int read = recorder.read(buffer, 0, buffer.length);
                    if (read > 0) {
                        out.write(buffer, 0, read);
                        out.flush();
                    } else if (read < 0) {
                        break;
                    }
                }
            } else {
                System.err.println("[Root Capture]  AudioRecord 初始化失败，请确保 Go 代码中使用了 su 提权执行");
            }
        } catch (Exception e) {
            e.printStackTrace(System.err);
        }
    }
}

```

哇塞, 这也太好了, 完全无杂音!

##### 暴风雪伊始: 大风起兮云飞扬

我依旧尝试这种 dex 注入的方案:

```
package com.mignon.tools;

import android.media.AudioAttributes;
import android.media.AudioFormat;
import android.media.AudioTrack;
import java.io.ByteArrayOutputStream;
import java.io.InputStream;

public class AudioPlayer {

    private static AudioTrack createTrack(int sampleRate, int minBuf) {
        return new AudioTrack.Builder()
            .setAudioAttributes(new AudioAttributes.Builder()
                .setUsage(AudioAttributes.USAGE_VOICE_COMMUNICATION)
                .setContentType(AudioAttributes.CONTENT_TYPE_SPEECH)
                .build())
            .setAudioFormat(new AudioFormat.Builder()
                .setEncoding(AudioFormat.ENCODING_PCM_16BIT)
                .setSampleRate(sampleRate)
                .setChannelMask(AudioFormat.CHANNEL_OUT_MONO)
                .build())
            .setBufferSizeInBytes(minBuf * 4) // 缓冲区给足
            .setTransferMode(AudioTrack.MODE_STREAM)
            .build();
    }

    public static void main(String[] args) {
        try {
            InputStream input = System.in;
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            byte[] buffer = new byte[8192];
            int read;
            while ((read = input.read(buffer)) != -1) {
                baos.write(buffer, 0, read);
            }
            byte[] allPcmData = baos.toByteArray();

            if (allPcmData.length == 0) return;

            int sampleRate = 16000;
            int minBuf = AudioTrack.getMinBufferSize(sampleRate, AudioFormat.CHANNEL_OUT_MONO, AudioFormat.ENCODING_PCM_16BIT);
            AudioTrack track = createTrack(sampleRate, minBuf);
            track.play();

            int offset = 0;
            while (offset < allPcmData.length) {
                int chunk = Math.min(minBuf, allPcmData.length - offset);
                chunk = chunk - (chunk % 2);
                if (chunk <= 0) break;

                int res = track.write(allPcmData, offset, chunk);

                if (res < 0) {
                    track.release();
                    Thread.sleep(500);
                    track = createTrack(sampleRate, minBuf);
                    track.play();
                    continue;
                }
                offset += res;
            }

            track.stop();
            track.release();

        } catch (Exception e) {
            e.printStackTrace(System.err);
            System.exit(1);
        }
    }
}

```

**Android 系统服务的运行上下文（Context）和音频路由（Audio Routing）的限制**, `AudioTrack` 的实例化虽然不需要特殊的 `Permission`，但它极度依赖系统的 `AudioService` 状态, **问题所在**：`AudioTrack` 在底层会调用 `IAudioFlinger`。如果你的进程没有被系统识别为拥有音频输出焦点的 “应用”，或者没有被赋予正确的 `SessionId`，音频流可能会被静音或直接丢弃。且还容易崩溃, 那换一种方式.

##### 暴风雪来了

测试了几轮, 好像没啥用, 此时 , 煞笔小米移动, 给我禁了!!!

```
[小米】尊敬的用户您好，为了深入推进防范治理电信网络诈骗工作，根据《中华人民共和国反电信网络诈骗法》及相关法律法规要求，您尾号3410的号码存在异常使用行为，您个人身份信息或号卡可能被不法分子利用，为了保护您个人及广大人民群众利益，将对您的号码限制呼出(仅可接听电话、接收短信)，48小时后将进行保护性停机，停机期间会收取5元/月的保号费。请您本人在收到短信2小时后点击
xxxx进行申诉，并持本人身份证完成二次实人认证(仅支持安卓手机)。如有疑问请致电小米移动客服热线:10046。给您带来的不便，我们深感抱歉，感谢您的配合。

```

我靠 (这就是前章节为什么换卡的原因)

生气了 10 分钟, 那没办法, 1. 换卡, 2. 试试录音机, 万一是一个方式呢?

于是, 写一 NDK:

```
#include <SLES/OpenSLES.h>
#include <SLES/OpenSLES_Android.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#define BUF_SIZE 4096

FILE *fp = NULL;
char buffer[BUF_SIZE];
SLObjectItf engineObject = NULL;
SLObjectItf outputMixObject = NULL;
SLObjectItf bqPlayerObject = NULL;

// 释放资源
void shutdown() {
    if (bqPlayerObject) (*bqPlayerObject)->Destroy(bqPlayerObject);
    if (outputMixObject) (*outputMixObject)->Destroy(outputMixObject);
    if (engineObject) (*engineObject)->Destroy(engineObject);
    if (fp) fclose(fp);
}

void bqPlayerCallback(SLAndroidSimpleBufferQueueItf bq, void *context) {
    if (fp && !feof(fp)) {
        size_t readSize = fread(buffer, 1, BUF_SIZE, fp);
        if (readSize > 0) {
            // 将新数据喂进队列，实现无缝连接
            (*bq)->Enqueue(bq, buffer, readSize);
        }
    } else {
        printf("\n 播放完毕，流结束。\n");
    }
}

int main(int argc, char **argv) {
    const char* file_path = "/data/local/tmp/test.wav";
    if (argc > 1) file_path = argv[1];

    fp = fopen(file_path, "rb");
    if (!fp) {
        printf("错误: 无法打开文件 %s\n", file_path);
        return -1;
    }
    fseek(fp, 44, SEEK_SET);

    SLEngineItf engineEngine;
    slCreateEngine(&engineObject, 0, NULL, 0, NULL, NULL);
    (*engineObject)->Realize(engineObject, SL_BOOLEAN_FALSE);
    (*engineObject)->GetInterface(engineObject, SL_IID_ENGINE, &engineEngine);

    (*engineEngine)->CreateOutputMix(engineEngine, &outputMixObject, 0, NULL, NULL);
    (*outputMixObject)->Realize(outputMixObject, SL_BOOLEAN_FALSE);

    SLDataLocator_AndroidSimpleBufferQueue loc_bufq = {SL_DATALOCATOR_ANDROIDSIMPLEBUFFERQUEUE, 2};
    SLDataFormat_PCM format_pcm = {
        SL_DATAFORMAT_PCM,
        1,
        SL_SAMPLINGRATE_16,          // 16kHz
        SL_PCMSAMPLEFORMAT_FIXED_16, // 16-bit
        SL_PCMSAMPLEFORMAT_FIXED_16,
        SL_SPEAKER_FRONT_CENTER,
        SL_BYTEORDER_LITTLEENDIAN
    };
    SLDataSource audioSrc = {&loc_bufq, &format_pcm};

    SLDataLocator_OutputMix loc_outmix = {SL_DATALOCATOR_OUTPUTMIX, outputMixObject};
    SLDataSink audioSnk = {&loc_outmix, NULL};

    const SLInterfaceID ids[1] = {SL_IID_BUFFERQUEUE};
    const SLboolean req[1] = {SL_BOOLEAN_TRUE};
    (*engineEngine)->CreateAudioPlayer(engineEngine, &bqPlayerObject, &audioSrc, &audioSnk, 1, ids, req);
    (*bqPlayerObject)->Realize(bqPlayerObject, SL_BOOLEAN_FALSE);

    SLPlayItf bqPlayerPlay;
    (*bqPlayerObject)->GetInterface(bqPlayerObject, SL_IID_PLAY, &bqPlayerPlay);
    SLAndroidSimpleBufferQueueItf bqPlayerBufferQueue;
    (*bqPlayerObject)->GetInterface(bqPlayerObject, SL_IID_BUFFERQUEUE, &bqPlayerBufferQueue);

    (*bqPlayerBufferQueue)->RegisterCallback(bqPlayerBufferQueue, bqPlayerCallback, NULL);

    (*bqPlayerPlay)->SetPlayState(bqPlayerPlay, SL_PLAYSTATE_PLAYING);

    size_t firstRead = fread(buffer, 1, BUF_SIZE, fp);
    (*bqPlayerBufferQueue)->Enqueue(bqPlayerBufferQueue, buffer, firstRead);

    printf("Native 平滑注入中 [Callback Mode]...\n");
    printf("按 Ctrl+C 停止，或等待播放完成。\n");

    while (!feof(fp)) {
        usleep(500000);
    }
    sleep(1);
    shutdown();
    return 0;
}

```

试试. OpenSL 麦克风暴力播放, 录音确实有, 但是.... 还没测通话到底如何

##### 风雪中的狼群

我决定换卡测一下, 换上移动的卡, 然后 call, 囿于前车之鉴, 我决定一次电话打很久, 然后发送试试

很好, 泡汤了.

试了试 firda, 肯定能行, 但是, frida 在 android 启动 android 很呆的!

```
(gxty) PS E:\gxty> frida -U -n com.android.phone -l ./src/hook_audio.js
     ____
    / _  |   Frida 16.2.2 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://frida.re/docs/home/

```

那怎么办呢, 难道我要再编译 tinymix, 然后去 / vendor/etc/mixer_paths.xml 看, 拨开关吗? 试试吧

#### 雨后天晴

在 Android 中级 / 高级逆向中，实现 “通话状态下的音频注入” 一直是个难点。尤其是在搭载高通芯片的设备（如 Pixel 3, sdm845）上，由于其采用了极度苛刻的动态音频路由（DPCM）和 ADSP 硬件时钟校验，传统的底层注入方法往往会在重启后失效。

本教程将复盘整个注入过程，详细拆解如何探测高通黑盒的底层参数，并提供一键突破的解决方案。

##### 核心侦察：如何探测到真实的硬件缓冲要求？

我们最终得出 `-p 1024 -n 2` 这个完美伪造参数，并不是凭空猜出来的，而是通过系统的 `/proc` 节点 “偷窥” 到了 ADSP 当前正在使用的真实内存布局。具体侦察分为两步：

##### 第一步：探测设备支持的参数范围 (tinypcminfo)

在尝试注入前，我们先查看了目标虚拟节点（Device 13）的规格：

```
/data/local/tmp/tinypcminfo -D 0 -d 13

```

输出显示：  
`Rate: min=8000Hz max=384000Hz`  
`Period size: min=4 max=61440`  
`Period count: min=2 max=8`  
这告诉我们，这根管子理论上支持 8k 采样率，且 Period Size 在 4 到 61440 之间。但 “理论支持” 和“实际放行”在高通 ADSP 里是两码事，系统默认分配的参数被拒绝了 (`Invalid argument`)。

##### 第二步：偷窥正在运行的真实硬件参数 (cat hw_params)

既然默认参数被拒，我们就去找**当前正在通话的物理设备**抄作业。我们在通话状态下，直接抓取了正在运行 (`RUNNING`) 的原生录音节点 (`pcm2c`) 的硬件参数：

```
cat /proc/asound/card0/pcm2c/sub0/hw_params

```

系统老老实实交出了底牌：

```
access: RW_INTERLEAVED
format: S16_LE
channels: 1
rate: 8000 (8000/1)
period_size: 1024
buffer_size: 2048

```

**破案逻辑：**

1.  采样率 (`rate`) 严格锁死在 **8000**。
2.  每次硬件中断搬运的数据帧数 (`period_size`) 严格锁死在 **1024**。
3.  总环形缓冲区大小 (`buffer_size`) 是 **2048**。
4.  根据 ALSA 公式：`period_count = buffer_size / period_size`，得出周期数为 **2048 / 1024 = 2**。

这就是 `-p 1024 -n 2` 的由来！我们用这个偷窥来的参数去伪装我们的 `tinyplay`，ADSP 校验时发现与它的共享内存映射完全一致，直接放行！

* * *

##### 样品解剖：一键注入脚本逐行解析

下面是我们在实战中跑通的最终脚本，我们来逐行拆解它为什么要这么写：

```
# 1. 开启 AudioService 逻辑后门
service call audio 35 s16 "incall_music_enabled=true"
service call audio 35 s16 "voice_mix_multimedia=1"

```

*   **原理：** `service call audio 35` 是直接通过 Binder IPC 调用 Android Java 层的 `AudioService` (Transaction Code 35 通常对应 `setParameters`)。
*   **作用：** 原生 Android 会拦截外部多媒体声音进入通话上行链路。这两行命令直接在内存中修改了 Audio HAL 的策略标志位，告诉系统：“现在允许将多媒体 (multimedia) 混音 (mix) 到通话 (voice/incall) 中”。如果 Parcel 返回 1，说明底层接收了该指令。

```
# 2. 强行物理并线 (操作 ALSA 混音器)
/data/local/tmp/tinymix set "Incall_Music Audio Mixer MultiMedia1" 1 2>/dev/null
/data/local/tmp/tinymix set "Incall_Music Audio Mixer MultiMedia2" 1 2>/dev/null
/data/local/tmp/tinymix set "Incall_Music Audio Mixer MultiMedia5" 1 2>/dev/null

```

*   **原理：** `tinymix` 是直接操作内核音频驱动 (ASoC) 的工具。Pixel 3 有近 3000 个混音开关。
*   **作用：** 这三行命令强行合上了物理开关，把播放多媒体的 `MultiMedia1`, `2`, `5` 端口的音频流，硬桥接到 `Incall_Music`（通话混音）的路径上。
*   **细节：** 为什么要写三个？还要加 `2>/dev/null`？因为高通路由是动态的，重启后你不知道系统具体把哪个 MultiMedia 节点分配给了当前的音频会话。全开一遍可以暴力刷新路由表（即使某些节点处于关闭状态报错 524，我们用 `2>/dev/null` 屏蔽掉报错即可）。

```
# 3. 精准参数注入 (突破 DSP 硬件校验)
/data/local/tmp/tinyplay /data/local/tmp/test_8k.pcm -D 0 -d 13 -c 1 -r 8000 -b 16 -p 1024 -n 2

```

*   **原理：** `tinyplay` 向指定的声卡和设备节点灌入裸 PCM 数据。
*   **逐项解析：**
    *   `/data/local/tmp/test_8k.pcm`: 你准备好的 8000Hz 单声道音频源文件。
    *   `-D 0 -d 13`: 目标是 Card 0, Device 13（我们探测到的支持通话混音的虚拟前端节点）。
    *   `-c 1`: 通道数，单声道 (Mono)。
    *   `-r 8000`: 采样率，必须严格对齐通话基带的 8000Hz 时钟，否则会 Wait for Clock 假死 (`Played 0 bytes`)。
    *   `-b 16`: 16 位深度。
    *   `-p 1024 -n 2`: **决胜局参数**！这是我们在上面用 `cat hw_params` 偷窥来的。强制设置 Period Size 为 1024，Count 为 2。这绕过了高通 ADSP 极其苛刻的对齐校验，彻底消除了 `Invalid argument`。

* * *

##### 结论

高通音频的 “重启失效” 是因为其极端的 DPCM 动态路由设计。通过 `service call` 开逻辑锁、`tinymix` 接物理线，最后利用 `/proc` 节点探测真实的 `hw_params`，用 `-p 1024 -n 2` 完美通过 ADSP 硬件校验，这就是攻克这座黑盒的核心心法。

在注入时, 遇到几个事情:

1.  编译 tinymix, tinypcminfo, tinyplay 报错
2.  需要转换通道数和采样率

如下:

```
#include <stdio.h>
#include <stdint.h>

int main(int argc, char *argv[]) {
    FILE *fin = fopen("/data/local/tmp/test.wav", "rb");
    FILE *fout = fopen("/data/local/tmp/test_8k.pcm", "wb");
    fseek(fin, 44, SEEK_SET); // 跳过 wav 头
    int16_t sample;
    int count = 0;
    while (fread(&sample, 2, 1, fin)) {
        if (count++ % 2 == 0) { // 每两个采样取一个，实现 16k -> 8k
            fwrite(&sample, 2, 1, fout);
        }
    }
    fclose(fin); fclose(fout);
    return 0;
}

```

最终结果:

```
service call audio 35 s16 "incall_music_enabled=true"
service call audio 35 s16 "voice_mix_multimedia=1"
/data/local/tmp/tinymix set "Incall_Music Audio Mixer MultiMedia1" 1 2>/dev/null
/data/local/tmp/tinymix set "Incall_Music Audio Mixer MultiMedia2" 1 2>/dev/null
/data/local/tmp/tinymix set "Incall_Music Audio Mixer MultiMedia5" 1 2>/dev/null
/data/local/tmp/tinyplay /data/local/tmp/test_8k.pcm -D 0 -d 13 -c 1 -r 8000 -b 16 -p 1024 -n 2

```

golang 的部分代码

```
package unlock

import (
    "log"
    "os/exec"
)

// WakeScreen 仅发送 KEYCODE_WAKEUP (224) 唤醒屏幕
func WakeScreen() error {
    log.Println("[Unlock] 正在执行屏幕唤醒指令...")

    cmd := exec.Command("input", "keyevent", "224")
    if err := cmd.Run(); err != nil {
        log.Printf("[Unlock] 唤醒失败: %v\n", err)
        return err
    }

    log.Println("[Unlock] 唤醒指令下发成功")
    return nil
}

// SleepScreen 发送 KEYCODE_POWER (26) 息屏/锁屏
func SleepScreen() error {
    log.Println("[Unlock] 正在执行息屏/锁屏指令...")

    cmd := exec.Command("input", "keyevent", "26")
    if err := cmd.Run(); err != nil {
        log.Printf("[Unlock] 息屏失败: %v\n", err)
        return err
    }

    log.Println("[Unlock] 息屏指令下发成功")
    return nil
}


```

```
package call

import (
    "fmt"
    "log"
    "os/exec"
    "strings"
)

func MakeCall(phone string) error {
    log.Printf("[Call] 准备拨打目标号码: %s\n", phone)
    cmd := exec.Command("am", "start", "-a", "android.intent.action.CALL", "-d", fmt.Sprintf("tel:%s", phone))
    return cmd.Run()
}

// IsCallActive 精准检测通话状态
// true = 正在拨号或通话中; false = 已经挂断或待机
func IsCallActive() bool {
    cmd := exec.Command("sh", "-c", "dumpsys telephony.registry | grep mCallState")
    out, err := cmd.Output()
    if err != nil {
        return false
    }

    statusStr := string(out)
    // 1 = 响铃, 2 = 通话中
    if strings.Contains(statusStr, "mCallState=1") || strings.Contains(statusStr, "mCallState=2") {
        return true
    }
    return false
}


```

python 的部分代码

```
 等待 12 秒以便电话接通...")    time.sleep(12)    threading.Thread(target=record_test, daemon=True).start()    # 开始执行暴扣推流！    push_audio_test()    while is_call_active:        time.sleep(1)

```

### 结束了. 这个标题用来发牢骚, google, 高通, 你俩有病吧??
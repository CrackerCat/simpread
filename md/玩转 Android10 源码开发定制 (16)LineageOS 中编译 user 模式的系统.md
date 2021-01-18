> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/F5DTy5SICRUZsH6F2ZoRIQ)

  

一、安卓系统编译选项简介
------------

android 编译的时候可以选择编译选项 eng、user 和 userdebug。

### 1.eng 编译选项

**(1).** 系统编译的时候安装标签 LOCAL_MODULE_TAGS 为 user、debug、eng 的模块

**(2).** 设定属性 ro.secure=0，关闭安全检查功能

**(3).** 设定属性 ro.debuggable=1，启用应用调试功能

**(4).** 默认打开 adb 功能，adb 拥有 root 权限

### 2.userdebug 编译选项

**(1).** 系统编译的时候安装标签 LOCAL_MODULE_TAGS 为 user、debug 的模块

**(2).** 设定属性 ro.secure=1，打开安全检查功能

**(3).** 设定属性 ro.debuggable=1，启用应用调试功能

**(4).** 默认打开 adb 功能，adb 拥有 root 权限

**(5).** userdebug 与 “user” 类似, 但具有 root 权限和调试功能；是进行调试时的首选编译类型

### 3.user 编译选项

**(1).** 系统编译的时候安装标签 LOCAL_MODULE_TAGS 为 user 的模块

**(2).** 设定属性 ro.secure=1，打开安全检查功能

**(3).** 设定属性 ro.debuggable=0，关闭应用调试功能

**(4).** 默认关闭 adb 功能，adb 无 root 权限

**(5).** 权限受限, 适用于生产环境

**综上总结:**

eng 就是工程师用的开发测试环境，方便开发调试各种软硬件之间的交互、性能等等。

userdebug 就是 user 版本发布之前的开发调试版本

user 就是生产环境用的版本，平时我们正规渠道买的手机都是 user 版本的。

目前很多 App 检测运行环境是否正常的检测点之一就是检测当前运行的系统属于哪个编译选项。eng、userdebug 都是属于风险设备考虑范畴。

二、lineageOs 中编译 user 选项系统
-------------------------

lineageOs 源码中，官方提供的编译方式命令如下 (oneplus3 说明):

```
source build/envsetup.sh
breakfast oneplus3
brunch oneplus3


```

通过以上命令编译出来的是 userdebug 选项的刷机包。通过 "breakfast/brunch" 关键字搜索源码找到实现的地方，breakfast、brunch 命令实现文件路径如下:

```
vendor/lineage/build/envsetup.sh


```

从以上路径可以看出这个是 lineage 自己增加的编译命令。查看 breakfast 和 brunch 的实现代码:

```
# brunch命令实现,会调用breakfast
function brunch()
{
    breakfast $*
    if [ $? -eq 0 ]; then
        mka bacon
    else
        echo "No such item in brunch menu. Try 'breakfast'"
        return 1
    fi
    return $?
}
# breakfast实现，代码中可以看到可以传入编译的设备代码和编译选项
# 如果没有指定编译选项，默认使用"userdebug"
function breakfast()
{
    target=$1
    local variant=$2

    if [ $# -eq 0 ]; then
        # No arguments, so let's have the full menu
        lunch
    else
        echo "z$target" | grep -q "-"
        if [ $? -eq 0 ]; then
            # A buildtype was specified, assume a full device name
            lunch $target
        else
            # This is probably just the Lineage model name
            if [ -z "$variant" ]; then
                variant="userdebug"
            fi

            lunch lineage_$target-$variant
        fi
    fi
    return $?
}


```

从以上编译命令的实现分析，需要编译 user 版本的系统只需要将编译命令改成如下:

```
source build/envsetup.sh
breakfast oneplus3 user
brunch oneplus3 user


```

三、新增编译 user 选项命令 makeuser
-------------------------

为了方便编译 user 版本的，完全可以按照自己的想法增加属于自己的编译命令。比如增加一个 makeuser 命令，传入设备代码就编译对应手机的 lineageOs 刷机包。实现方式如下: 

(1)、在 vendor/lineage/build/envsetup.sh 文件中添加 makeuser 命令，具体实现如下

```
# ///ADD START
# 只需要传入设备代码号就可以进行user版本系统编译，参考breakfast命令实现
function makeuser()
{
    if [ $# != 1 ]; then
        # No arguments, so let's have the full menu
        printf  "Error:arguments must be one,please input device codename\n"
        return
    fi    
    breakfast $1 user
    if [ $? -eq 0 ]; then
        mka bacon
    else
        echo "No such item in brunch menu. Try 'breakfast'"
        return 1
    fi
    return $?
}
# ///ADD END


```

（2）、使用 source 命令使新增的命令生效

```
source build/envsetup.sh


```

(3)、使用新命令: makeuser 编译

```
makeuser oneplus3


```

如下是我修改之后的执行效果图:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5430FXfZPWYTCmzicFvibB561v5Ogu0J9OlrYw7mEhOsTn18eOUE73VpjnNsoS074Wgiab73b9w8gictJsw/640?wx_fmt=png)

关于 "user" 编译选项下如何配置 adb 的 root 权限，后面文章会分享，可以关注我的公众号不定时更新。

[上一篇] [玩转 Android10 源码开发定制 (15) 实现跳过开机向导、插电源线不休眠等默认配置](http://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247484123&idx=1&sn=d5b5dac3811b250873e2850128486d34&chksm=ce07539ef970da88081d75b49d841bb0bff28d3c7c51d26ee1c91e28c4ce2429176351c38e73&scene=21#wechat_redirect)

**专注安卓系统、安卓 ndk 开发、安卓应用安全和逆向分析相关等 IT 知识分享，系统定制、frida、xposed(sandhook、edxposed) 系统集成、加固、脱壳等等。微信搜索公众号 "QDOIRD88888" 或者扫描以下二维码关注公众号。第一时间接收更新文章。**

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430HpkFIRvrbTB68PwHwicZh5YG5aXIeibCxz29DDYLdQrf3ibjZxrCHST9r0zicRIsBYJ8HasrIwJU55Q/640?wx_fmt=jpeg)

扫一扫关注公众号
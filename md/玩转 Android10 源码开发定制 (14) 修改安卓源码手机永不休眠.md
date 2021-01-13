> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/2_NQF-8fWF9d4xPezF4C8g)

一、设置永不休眠简单分析
------------

安卓手机中 " **设置** " 应用里面可以设置手机屏幕超时时间。如下图所示:![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4kWiazZO7ncV0Z0GKMlMFrDNlZlxkHPR1PSYxAMsAX8ZbLIYux9Dkhb1Q/640?wx_fmt=png) 通过选项可以看到最多能设置 30 分钟, 那如果需要永不休眠，只要把屏幕超时时间设置的足够大就可以达到目的，比如 Int 的最大值。接下来源码中追踪一下 "设置" 应用中如何实现的。源码中 "设置" 应用的源码路径如下:

```
packages/apps/Settings


```

通过对该目录关键字搜索和分析，找到设置屏幕超时的源码文件如下:

```
packages/apps/Settings/src/com/android/settings/display/TimeoutPreferenceController.java


```

"**TimeoutPreferenceController.java**" 中修改屏幕超时的关键函数如下:

```
 @Override
    public boolean onPreferenceChange(Preference preference, Object newValue) {
        try {
            int value = Integer.parseInt((String) newValue);
            //最终是调用这个函数实现的
            Settings.System.putInt(mContext.getContentResolver(), SCREEN_OFF_TIMEOUT, value);
            updateTimeoutPreferenceDescription((TimeoutListPreference) preference, value);
        } catch (NumberFormatException e) {
            Log.e(TAG, "could not persist screen timeout setting", e);
        }
        return true;
    }


```

从代码中可以看到通过以下代码修改屏幕超时时间:

```
Settings.System.putInt(getContentResolver(),Settings.System.SCREEN_OFF_TIMEOUT, 6000);


```

以下将在设置应用中的屏幕超时中添加一个 "永不休眠" 的功能。

二、涉及修改的文件
---------

```
//添加永不休眠以及时间选项
packages/apps/Settings/res/values/arrays.xml

//添加中文展示的永不休眠选项
packages/apps/Settings/res/values/values-zh-rCN/arrays.xml
//Settings.System.putInt方法的实现类，根据传入的特殊值-1修改为一个很大的休眠时间
/frameworks/base/core/java/android/provider/Settings.java


```

三、修改实战
------

### 1. 设置应用中添加永不休眠选项

**packages/apps/Settings/res/values/arrays.xml** 添加如下内容:

```
  <string-array >
      <item>15 seconds</item>
      <item>30 seconds</item>
      <item>1 minute</item>
      <item>2 minutes</item>
      <item>5 minutes</item>
      <item>10 minutes</item>
      <item>30 minutes</item>
      <!-- ///ADD START 此处新增的英文语言下Never展示项-->
      <item>Never</item>
      <!-- ///ADD END -->
      
  </string-array>

  <!-- Do not translate. -->
  <string-array >
      <!-- Do not translate. -->
      <item>15000</item>
      <!-- Do not translate. -->
      <item>30000</item>
      <!-- Do not translate. -->
      <item>60000</item>
      <!-- Do not translate. -->
      <item>120000</item>
      <!-- Do not translate. -->
      <item>300000</item>
      <!-- Do not translate. -->
      <item>600000</item>
      <!-- Do not translate. -->
      <item>1800000</item>
      <!-- ///ADD START 此处新增的休眠时间-1-->
      <item>-1</item>
      <!-- ///ADD END -->
  </string-array>


```

**packages/apps/Settings/res/values/values-zh-rCN/arrays.xml** 添加如下内容:

```
<string-array >
 <item msgid="8386012403457852396">"15 秒"</item>
 <item msgid="4572123773028439079">"30 秒"</item>
 <item msgid="7016081293774377048">"1 分钟"</item>
 <item msgid="838575533670111144">"2 分钟"</item>
 <item msgid="2693197579676214668">"5 分钟"</item>
 <item msgid="1955784331962974678">"10 分钟"</item>
 <item msgid="5578717731965793584">"30 分钟"</item>
 <!-- ///ADD START 此处新增的中文语言展示选择项-->
 <item>永不睡觉</item>
 <!-- ///ADD END -->
</string-array>


```

### 2.Settings.java 文件中修改设置的休眠时间

在该文件中找到 Settings.System.putInt 方法，修改关联的方法代码如下:

```
//putInt最终调用的是putIntForUser
 public static boolean putInt(ContentResolver cr, String name, int value) {
          return putIntForUser(cr, name, value, cr.getUserId());
      }

//putIntForUser中根据传入的name和value特殊值-1进行修改设置
/** @hide */
@UnsupportedAppUsage
public static boolean putIntForUser(ContentResolver cr, String name, int value,int userHandle) {
///ADD START
if(name.equals(SCREEN_OFF_TIMEOUT))    
{
    //-1说明是我们在设置中添加的永不休眠的值
    if(value==-1)
    {
            //
            Log.d("Settings","change screen timeout for:"+Integer.toString(Integer.MAX_VALUE-1000));
            return putStringForUser(cr, name, Integer.toString(Integer.MAX_VALUE-1000), userHandle);
     }
}
              ///ADD END
          return putStringForUser(cr, name, Integer.toString(value), userHandle);
      }



```

四、效果展示
------

修改之后编译刷机，我本机测试了挂了一天都没休眠。展示图片:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4kmW6iaicDf6icrvYXdiaCyKTgfXmj0logBicHJYItcTBgrD43ibEC5dzml3icw/640?wx_fmt=png)

**说明:** ****微信搜索公众号**** **"****QDOIRD88888****"** **或者公众号名称 "** **卓码空间** **"或者扫描下面二维码关注微信公众号, 发消息" 永不睡觉 " 下载修改过的源码文件。**

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432e7u878mQicQ3WCESZVjf4k87a4OO3XJTFQrIohlNmiaflthbzKroEQ0wlKedZYGCfFmwrUu2399rw/640?wx_fmt=png)

**专注安卓系统、安卓 ndk 开发、安卓应用安全和逆向分析相关知识分享，系统定制、frida、xposed(sandhook、edxposed) 系统集成、加固、脱壳等等。微信搜索公众号 "QDOIRD88888" 或者扫描以下二维码关注公众号。第一时间接收更新文章。**![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5432e7u878mQicQ3WCESZVjf4kvEcaTA9Drgiby5WCjNAqPOibiaVZwe3lHvj93WknnXAh27vWQPpUDneew/640?wx_fmt=jpeg)
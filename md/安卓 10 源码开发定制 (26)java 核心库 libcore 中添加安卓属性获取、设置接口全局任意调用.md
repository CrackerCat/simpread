> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/UhyqZRfzqu7H84G-d7exjg)

**一、前言**

    安卓源码中 **java** 核心库 **libcore** 中没有像 **frameworks** 中提供的 **SystemProperties** 那样的接口来获取和设置的属性的接口支持。有时候在 **libcore** 中非常需要读取一些属性来进行逻辑判断来完成一些逻辑工作。比如我在 **File** 类中添加一个开关来判断是否需要日志输出当前的 **App** 打开的文件, 此处我就可以考虑使用自定义的一个属性来获取当前设置的进程 **uid**，如果当前 uid 和属性配置的 **uid** 一致, 就可以进行打印输出。

       由于 **frameworks** 中 **SystemProperties** 有现成的接口实现，所以以下将参考该接口在 **libcore** 中添加安卓整个 **java** 层全局支持的属性 **get/set** 接口类 **XSystemProperties**。

****二、**frameworks** 中 **SystemProperties** 分析****

*   **java 层实现分析**  
    

      **SystemProperties** 源码中文件路径如下:  

> **frameworks\base\core\java\android\os\SystemProperties.java**

     在该类中抽取了几个感兴趣的 **get、set 方法,** 关键代码如下:

```
//...省略
public class SystemProperties {
    //...省略
    //native方法，实现了读取属性
    private static native String native_get(String key, String def);
    //...省略
    //native方法，实现了设置属性
    private static native void native_set(String key, String def);
    //...省略
    //实现了对外调用的读取属性方法
    public static String get(@NonNull String key, @Nullable String def) {
        if (TRACK_KEY_ACCESS) onKeyAccess(key);
        return native_get(key, def);
    }
   //...省略
   //实现了对外调用的设置属性方法
    public static void set(@NonNull String key, @Nullable String val) {
        if (val != null && !val.startsWith("ro.") && val.length() > PROP_VALUE_MAX) {
            throw new IllegalArgumentException("value of system property '" + key
                    + "' is longer than " + PROP_VALUE_MAX + " characters: " + val);
        }
        if (TRACK_KEY_ACCESS) onKeyAccess(key);
        native_set(key, val);
    }
   //...省略
}

```

      以上代码中可以发现 **SystemProperties** 中的 **get**、**set** 最终调用的是 **native** 层的方法来实现的。接下来分析 **native** 层的实现情况。  

*   **native 层实现分析**
    
    在安卓源码中搜索，**SystemProperties** 的 **native** 实现源文件路径如下:
    

```
frameworks\base\core\jni\android_os_SystemProperties.cpp

```

     以下是抽取需要的部分代码示例如下:

```
//...省略
//读取属性逻辑代码
jstring SystemProperties_getSS(JNIEnv *env, jclass clazz, jstring keyJ,
                               jstring defJ)
{
    // Using ConvertKeyAndForward is sub-optimal for copying the key string,
    // but improves reuse and reasoning over code.
    auto handler = [&](const std::string& key, jstring defJ) {
        std::string prop_val = android::base::GetProperty(key, "");
        if (!prop_val.empty()) {
            return env->NewStringUTF(prop_val.c_str());
        };
        if (defJ != nullptr) {
            return defJ;
        }
        // This function is specified to never return null (or have an
        // exception pending).
        return env->NewStringUTF("");
    };
    return ConvertKeyAndForward(env, keyJ, defJ, handler);
}
//...省略
//设置属性逻辑代码
void SystemProperties_set(JNIEnv *env, jobject clazz, jstring keyJ,
                          jstring valJ)
{
    auto handler = [&](const std::string& key, bool) {
        std::string val;
        if (valJ != nullptr) {
            ScopedUtfChars key_utf(env, valJ);
            val = key_utf.c_str();
        }
        return android::base::SetProperty(key, val);
    };
    if (!ConvertKeyAndForward(env, keyJ, true, handler)) {
        // Must have been a failure in SetProperty.
        jniThrowException(env, "java/lang/RuntimeException",
                          "failed to set system property");
    }
}
//...省略

```

    在以上代码分析中，安卓 **10** 系统调用了 **android::base::setProperty** 和 **android:base::getProperty** 方法来实现 **native** 层的调用。在安卓源码中 **android::base::set/getProperty** 方法的实现文件路径如下:  

```
system\core\base\properties.cpp

```

   抽取以上 **set/getProperty** 中的代码实现如下:

```
//set实现
bool SetProperty(const std::string& key, const std::string& value) {
  return (__system_property_set(key.c_str(), value.c_str()) == 0);
}
//get实现
std::string GetProperty(const std::string& key, const std::string& default_value) {
  std::string property_value;
#if defined(__BIONIC__)
  const prop_info* pi = __system_property_find(key.c_str());
  if (pi == nullptr) return default_value;
  __system_property_read_callback(pi,
                                  [](void* cookie, const char*, const char* value, unsigned) {
                                    auto property_value = reinterpret_cast<std::string*>(cookie);
                                    *property_value = value;
                                  },
                                  &property_value);
#else
  auto it = g_properties.find(key);
  if (it == g_properties.end()) return default_value;
  property_value = it->second;
#endif
  // If the property exists but is empty, also return the default value.
  // Since we can't remove system properties, "empty" is traditionally
  // the same as "missing" (this was true for cutils' property_get).
  return property_value.empty() ? default_value : property_value;
}

```

    以上代码中最终调用了安卓属性系统中提供的**__system_property_read_callback** 和**__system_property_set** 进行调用。安卓属性系统中提供了多个_**_system_property_xx** 的接口供外部调用。本文暂不追踪分析属性系统这块相关的细节。以下列举了几个 **get、set** 相关的接口。如下所示 ：

```
//get属性接口
int __system_property_get(const char* name, char* value) {
  return system_properties.Get(name, value);
}
//设置属性接口
int __system_property_set(const char* key, const char* value) {
  if (key == nullptr) return -1;
  if (value == nullptr) value = "";
  //...省略
  }

```

    在 **Android** 系统 **system** 目录中 **libcutils** 中，对外封装了以上**__system_property_get** 和**__system_property_set** 接口。封装源文件如下:

```
system\core\libcutils\properties.cpp

```

       封装的关键核心代码如下:  

```
int property_set(const char *key, const char *value) {
    return __system_property_set(key, value);
}
int property_get(const char *key, char *value, const char *default_value) {
    int len = __system_property_get(key, value);
    if (len > 0) {
        return len;
    }
    if (default_value) {
        len = strnlen(default_value, PROPERTY_VALUE_MAX - 1);
        memcpy(value, default_value, len);
        value[len] = '\0';
    }
    return len;
}

```

      综合以上分析，我们以下将使用封装版的 **property_set**、**property_get** 接口来进行属性的获取和写入。

****三、libcore 中实现 SystemProperties 的功能****

      **libcore** 中实现 **SystemProperties** 的流程和 **libcore** 中添加 **log** 日志接口非常类似。**libcore** 中添加 **log** 的全局接口可以参考文章: [玩转安卓 10 源码开发定制 (19)Java 核心库 libcore 中添加 Log 接口任意调用](http://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247484261&idx=1&sn=87a2a32b6392b854f4fc8369ad3506b5&chksm=ce075220f970db36a799be0e33df2bcc224277d6a47aae7a2bf61befaeb4bd4a7304cb6c0637&scene=21#wechat_redirect)。

1. 添加 java.lang.XSystemProperties 属性接口类

###       (1)、创建 java.lang.XSystemProperties 类

       在 **libcore** 路径位置:

```
libcore/ojluni/src/main/java/java/lang

```

目录下新建 **XSystemProperties** 文件，如下所示:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5430RD0pL3HddSictNVOrTfsNJiab7e1JTmJ34A8Il90381JI0n6DvkszcXZcobbyoKAUkicACo4bTl6BQ/640?wx_fmt=png)

### (2)、参考 android.os.SystemProperties 的实现添加 XSystemProperties 获取 / 设置属性的方法

此处只添加几个常用的方法，添加之后核心代码如下所示:

```
private static native String native_get(String key, String def);
    private static native void native_set(String key, String def);
    public static String get( String key,String defV) {
        return native_get(key,defV);
    }
    public static void set( String key,  String val) {
        native_set(key, val);
    }

```

（3）、将 XSystemProperties.java 文件添加到编译文件链

        在 **libcore/openjdk_java_files.bp** 文件中加入 **XSystemProperties.java** 到编译文件链。如下所示:

```
        "ojluni/src/main/java/java/lang/System.java",
        ///ADD START
        "ojluni/src/main/java/java/lang/XSystemProperties.java",
        ///ADD END
        "ojluni/src/main/java/java/lang/ThreadDeath.java",

```

2.XSystemProperties.java   jni 实现
---------------------------------

### （1）、在 libcore native 层添加 XSystemProperties.c                     

###           XSystemProperties.c 主要是实现 java 层 XSystemProperties 的 native 方法。XSystemProperties.c 创建之后如下:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5430RD0pL3HddSictNVOrTfsNJYKa8OnCnRUOt0feia5Jrp8qiajicnSBfkPrENIicpVWWDM3GgYhxt6FzpQ/640?wx_fmt=png)

     参考 **android_os_SystemProperties.cpp** 中 **native_get/native_set** 实现方式，在 **XSystemPrope****rties.c** 中添加如下实现:

```
static jstring XSystemProperties_getSS(JNIEnv *env, jobject clazz,
                                      jstring keyJ, jstring defJ)
{
    int len;
    const char* key;
    char buf[PROPERTY_VALUE_MAX];
    jstring rvJ = NULL;
    if (keyJ == NULL) {
        JNU_ThrowNullPointerException(env, "key must not be null.");
        goto error;
    }
    key = (*env)->GetStringUTFChars(env,keyJ, NULL);
    len = property_get(key, buf, "");
    if ((len <= 0) && (defJ != NULL)) {
        rvJ = defJ;
    } else if (len >= 0) {
        rvJ = (*env)->NewStringUTF(env,buf);
    } else {
        rvJ = (*env)->NewStringUTF(env,"");
    }
    (*env)->ReleaseStringUTFChars(env,keyJ, key);
error:
    return rvJ;
}
static void XSystemProperties_set(JNIEnv *env, jobject clazz,
                                      jstring keyJ, jstring valJ)
{
    int err;
    const char* key;
    const char* val;
    if (keyJ == NULL) {
        JNU_ThrowNullPointerException(env, "key must not be null.");
        return ;
    }
    key = (*env)->GetStringUTFChars(env,keyJ, NULL);
    if (valJ == NULL) {
        val = "";       /* NULL pointer not allowed here */
    } else {
        val = (*env)->GetStringUTFChars(env,valJ, NULL);
    }
    err = property_set(key, val);
    (*env)->ReleaseStringUTFChars(env,keyJ, key);
    if (valJ != NULL) {
        (*env)->ReleaseStringUTFChars(env,valJ, val);
    }
    if (err < 0) {
        JNU_ThrowNullPointerException(env, "failed to set system property");
    }
}

```

    在 **XSystemProperties.c** 中添加 **register_java_lang_XSystemProperties** 方法来注册 **XSystemProperties** 的 **jni** 的方法。如下所示:

```
static JNINativeMethod gMethods[] = {
  NATIVE_METHOD(XSystemProperties, getSS, "(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;"),
  NATIVE_METHOD(XSystemProperties, set, "(Ljava/lang/String;Ljava/lang/String;)V"),
};
void  register_java_lang_XSystemProperties(JNIEnv *env)
{
       jniRegisterNativeMethods(env, "java/lang/XLog", gMethods, NELEM(gMethods));
}

```

### (2).OnLoad.cpp 中声明和调用 register_java_lang_XSystemProperties

**OnLoad.cpp** 文件路径如下:

```
libcore/ojluni/src/main/native/OnLoad.cpp


```

    在 **OnLoad.cpp** 中添加 **register_java_lang_XSystemProperties** 方法的声明和调用。如下所示:

```
...省略
///ADD START
extern "C" void register_java_lang_XSystemProperties(JNIEnv* env);
///ADD END
...省略
extern "C" JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void*) {
  ...省略
  ///ADD START
  register_java_lang_XSystemProperties(env);
  ///ADD END
  ...省略
}
...省略

```

    (3)、Android.bp 中添加 XSystemProperties.c 文件到编译文件中

      在文件 "**libcore/ojluni/src/main/native/Android.bp**" 中添加 **XSystemProperties.c** 到编译文件中。添加之后如下:

```
filegroup {
    name: "libopenjdk_native_srcs",
    srcs: [
         ...省略
        ///ADD START
        "XSystemProperties.c",
        ///ADD END
        ...省略
    ],
}

```

       以上接口新增完成之后，就可以在安卓系统整个 **java** 层使用接口来设置和获取属性。  

**四、编译系统更新 api 到系统**

    由于向系统加了新的接口，需要执行如下命令更新系统 **api**:  

```
source build/envsetup.sh
breakfast oneplus3
make update-api

```

     更新之后编译刷机，可以在 java 层的地方任意调用接口进行测试验证。

 **![](https://mmbiz.qpic.cn/mmbiz_gif/rFWVXwibLGty0S5JgMN8PpBib2631p7cDvlvTEaxFBzljBX9qWcVMSOymhkTd6ZmanRibYWsh0HmccjGWkadiaLwAA/640?wx_fmt=gif)** 点击屏末 ****| **********阅****读****原****文********** |**** ****进入 "**** **历史消息** ****" 页面查看更多资讯**** 

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjguCKrYZQfRXxK6hibNjOh10JibAdHj553dxk3PmoyUibjDCGcNdq3IQBKA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgibOWZXyrOLic5KPJ2y9A1gznt4xUa1H7MEhlgmcQgnE3IJvphZfOezfA/640?wx_fmt=png)  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgxGibv8NMwbmJuQo55Ry33RkQj6WTGwwyXgrcduXPL3xnUWeLUa3cDvA/640?wx_fmt=png)
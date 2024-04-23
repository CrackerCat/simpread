> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1916553-1-1.html)

> [md]> 炒冷饭，再聊聊大家都知晓的隐藏接口的限制解除。

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)iofomo

> 炒冷饭，再聊聊大家都知晓的隐藏接口的限制解除。

### 说明

由于我们容器产品的特性，需要将应用完整的运行起来，所以必须涉及一些隐藏接口的反射调用，而突破反射限制则成为我们实现的基础。现将我们的解决方案分享给大家，一起学习。

### Android 9.0 → 首次启用

这个大家都知道原理了，简单巴拉巴拉下，从下往上溯源。

**1、找到 API 判断规则豁免点。**

```
// source code: art/runtime/hidden_api.cc
template<typename T>
bool ShouldDenyAccessToMemberImpl(T* member, ApiList api_list, AccessMethod access_method) {

    // ......

  // Check for an exemption first. Exempted APIs are treated as SDK.
  if (member_signature.DoesPrefixMatchAny(runtime->GetHiddenApiExemptions())) {
    // Avoid re-examining the exemption list next time.
    // Note this results in no warning for the member, which seems like what one would expect.
    // Exemptions effectively adds new members to the public API list.
    MaybeUpdateAccessFlags(runtime, member, kAccPublicApi);
    return false;
  }

    // ......

  return deny_access;
}

```

**2、找到成员属性位置。**

```
// source code /art/runtime/runtime.h

class Runtime {
 public:
    // ......

    void SetHiddenApiExemptions(const std::vector<std::string>& exemptions) {
      hidden_api_exemptions_ = exemptions;
    }

    const std::vector<std::string>& GetHiddenApiExemptions() {
      return hidden_api_exemptions_;
    }

    // ......
};

```

**3、找到设置方法**

```
// source code: /art/runtime/native/dalvik_system_VMRuntime.cc

// ......

static void VMRuntime_setHiddenApiAccessLogSamplingRate(JNIEnv*, jclass, jint rate) {
  Runtime::Current()->SetHiddenApiEventLogSampleRate(rate);
}

// ......

static JNINativeMethod gMethods[] = {
    // ......
        NATIVE_METHOD(VMRuntime, setHiddenApiExemptions, "([Ljava/lang/String;)V"),
    // ......
};

void register_dalvik_system_VMRuntime(JNIEnv* env) {
        REGISTER_NATIVE_METHODS("dalvik/system/VMRuntime");
}

```

**4、找到上层调用入口。**

```
// source code /libcore/libart/src/main/java/dalvik/system/VMRuntime.java
package dalvik.system;

public final class VMRuntime {
    /**
     * Sets the list of exemptions from hidden API access enforcement.
     *
     * home.php?mod=space&uid=952169 signaturePrefixes
     *         A list of signature prefixes. Each item in the list is a prefix match on the type
     *         signature of a blacklisted API. All matching APIs are treated as if they were on
     *         the whitelist: access permitted, and no logging..
     *
     * @hide
     */
    @SystemApi(client = MODULE_LIBRARIES)
    @libcore.api.CorePlatformApi(status = libcore.api.CorePlatformApi.Status.STABLE)
    public native void setHiddenApiExemptions(String[] signaturePrefixes);
}

```

**5、形成解决方案。**

```
try {
    Method mm = Class.class.getDeclaredMethod("forName", String.class);
    Class<?> cls = (Class)mm.invoke((Object)null, "dalvik.system.VMRuntime");
    mm = Class.class.getDeclaredMethod("getDeclaredMethod", String.class, Class[].class);
    Method m = (Method)mm.invoke(cls, "getRuntime", null);
    Object vr = m.invoke((Object)null);
    m = (Method)mm.invoke(cls, "setHiddenApiExemptions", new Class[]{String[].class});
    String[] args = new String[]{"L"};
    m.invoke(vr, args);
} catch (Throwable e) {
    e.printStackTrace();
}

```

### Android 11.0 → 限制升级

从此版本开始，系统升级了上层接口的访问限制，直接将`VMRuntime`的类接口限制升级，因此只能通过`native`层进行访问。原理不变，利用系统加载`lib`库时`JNI_OnLoad`通过反射调用`setHiddenApiExemptions`，此时`caller`为`java.lang.System`其`domain`级别为`libcore.api.CorePlatformApi`，就可以访问`hiddenapi`了。

**方式 1：反射调用**

```
static int setApiBlacklistExemptions(JNIEnv* env) {
    jclass jcls = env->FindClass("dalvik/system/VMRuntime");
    if (env->ExceptionCheck()) {
        env->ExceptionDescribe();
        env->ExceptionClear();
        return -1;
    }

    jmethodID jm = env->GetStaticMethodID(jcls, "setHiddenApiExemptions", "([Ljava/lang/String;)V");
    if (env->ExceptionCheck()) {
        env->ExceptionDescribe();
        env->ExceptionClear();
        return -2;
    }

    jclass stringCLass = env->FindClass("java/lang/String");
    jstring fakeStr = env->NewStringUTF("L");
    jobjectArray fakeArray = env->NewObjectArray(1, stringCLass, NULL);
    env->SetObjectArrayElement(fakeArray, 0, fakeStr);
    env->CallStaticVoidMethod(jcls, jm, fakeArray);

    env->DeleteLocalRef(fakeStr);
    env->DeleteLocalRef(fakeArray);
    return 0;
}

jint JNI_OnLoad(JavaVM* vm, void* reserved) {
    //......
    JNIEnv * env = NULL;// got env from JavaVM

    // make sure call here
    setApiBlacklistExemptions(env);

    //......
    return 0;
}

```

**方式 2：直接函数调用。**

> 将系统的`libart.so`导出来，在 IDA 中查看导出的`c`函数名为：`_ZN3artL32VMRuntime_setHiddenApiExemptionsEP7_JNIEnvP7_jclassP13_jobjectArray`

```
void* utils_dlsym_global(const char* libName, const char* funcName) {
    void* funcPtr = NULL;
    void* handle = dlopen(libName, RTLD_LAZY|RTLD_GLOBAL);
    if (__LIKELY(handle)) {
        funcPtr = dlsym(handle, funcName);
    } else {
        LOGE("dlsym: %s, %s, %d, %s", libName, funcName, errno, strerror(errno))
        __ASSERT(0)
    }
    return funcPtr;
}

typedef void *(*setHiddenApiExemptions_Func)(JNIEnv* env, jclass, jobjectArray exemptions);
int fixHiddenApi(JNIEnv* env) {
    setHiddenApiExemptions_Func func = (setHiddenApiExemptions_Func)utils_dlsym_global("libart.so", "_ZN3artL32VMRuntime_setHiddenApiExemptionsEP7_JNIEnvP7_jclassP13_jobjectArray");
    __ASSERT(func)
    if (__UNLIKELY(!func)) return -1;

    jclass stringCLass = env->FindClass("java/lang/String");
    jstring fakeStr = env->NewStringUTF("L");
    jobjectArray fakeArray = env->NewObjectArray(1, stringCLass, NULL);
    env->SetObjectArrayElement(fakeArray, 0, fakeStr);
    func(env, NULL, fakeArray);
    env->DeleteLocalRef(fakeArray);
    if (env->ExceptionCheck()) {
        LOG_JNI_EXCEPTION(env, true)
        return -2;
    }
    return 0;
}


```

### Android 14 & 鸿蒙 4 → 异常补丁

通常情况下以上方法均可以达到隐藏接口的访问解除，但是我们通过兼容性测试，在鸿蒙和小米的最新版本系统，某些时候依然还是会出现一下日志：

```
Accessing hidden method Landroid/app/IUiModeManager$Stub;->asInterface(Landroid/os/IBinder;)Landroid/app/IUiModeManager; (max-target-p, reflection, denied)

```

而实际上其他的隐藏类是可以正常访问的，并且在一段时间内该类也是可以访问的，运行一段时间后就出现此问题。猜测`ROM`定制了一些缓存机制。于是尝试另一种方案：利用`VM`无法识别调用者的方式破坏调用堆栈。这可以通过函数创建的新线程，此时，我们处于一个新的`VM`调用堆栈中，没有任何调用历史记录。

```
#include <future>

static jobject reflect_getDeclaredMethod_internal(jobject clazz, jstring method_name, jobjectArray params) {
    bool attach = false;
    JNIEnv *env = jni_get_env(attach);
    if (!env) return;

    jclass clazz_class = env->GetObjectClass(clazz);
    jmethodID get_declared_method_id = env->GetMethodID(clazz_class, "getDeclaredMethod", "(Ljava/lang/String;[Ljava/lang/Class;)Ljava/lang/reflect/Method;");
    jobject res = env->CallObjectMethod(clazz, get_declared_method_id, method_name, params);
    if (env->ExceptionCheck()) {
        env->ExceptionDescribe();
        env->ExceptionClear();
    }
    jobject global_res = nullptr;
    if (res != nullptr) {
        global_res = env->NewGlobalRef(res);
    }

    jni_env_thread_detach();
    return global_res;
}

jobject reflect_getDeclaredMethod(JNIEnv *env, jclass interface, jobject clazz, jstring method_name, jobjectArray params) {
    jobject global_clazz = env->NewGlobalRef(clazz);
    jstring global_method_name = (jstring) env->NewGlobalRef(method_name);
    int arg_length = env->GetArrayLength(params);
    jobjectArray global_params = nullptr;
    if (params != nullptr) {
        jobject element;
        for (int i = 0; i < arg_length; i++) {
            element = (jobject) env->GetObjectArrayElement(params, i);
            env->SetObjectArrayElement(params, i, env->NewGlobalRef(element));
        }
        global_params = (jobjectArray) env->NewGlobalRef(params);
    }

    auto future = std::async(&reflect_getDeclaredMethod_internal, global_clazz, global_method_name, global_params);
    return future.get();
}

```

和上面一样，我们可以扩展出对应其他常用的函数实现（如`getMethod`，`getDeclaredField`，`getField`等）。只不过我们的容器项目需要兼容较久的版本，因此不能使用高版本的`std::async`特性，为此我们写了一个`pthead`的兼容性版本，可以适配低版本的`ndk`编译。

```
int ThreadAsyncUtils::threadAsync(BaseThreadAsyncArgument& argument) {
    pthread_t thread;
    int ret = pthread_create(&thread, NULL, threadAsyncInternal, &argument);
    if (0 != ret) {
        LOGE("thread async create error: %d, %s", errno, strerror(errno))
        return ret;
    }

    ret = pthread_join(thread, NULL);
    if (0 != ret) {
        LOGE("thread async join error: %d, %s", errno, strerror(errno))
        return ret;
    }
    return 0;
}

static void reflect_getDeclaredMethod_internal(BaseThreadAsyncArgument* _args) {
    ReflectThreadAsyncArgument* args = (ReflectThreadAsyncArgument*)_args;
    jobject clazz = args->jcls_clazz;
    jstring method_name = args->jcls_name;
    jobjectArray params = args->jcls_params;

    bool attach = false;
    JNIEnv *env = jni_get_env(attach);
    if (!env) return;

    jclass clazz_class = env->GetObjectClass(clazz);
    jmethodID get_declared_method_id = env->GetMethodID(clazz_class, "getDeclaredMethod", "(Ljava/lang/String;[Ljava/lang/Class;)Ljava/lang/reflect/Method;");
    jobject res = env->CallObjectMethod(clazz, get_declared_method_id, method_name, params);
    if (env->ExceptionCheck()) {
        LOG_JNI_CLEAR_EXCEPTION(env)
    }
    if (res != nullptr) {
        args->jcls_result = env->NewGlobalRef(res);
    }

    jni_env_thread_detach();
}

jobject ReflectUtils::getDeclaredMethod(JNIEnv *env, jclass interface, jobject clazz, jstring method_name, jobjectArray params) {
    auto global_clazz = env->NewGlobalRef(clazz);
    jstring global_method_name = (jstring) env->NewGlobalRef(method_name);
    int arg_length = env->GetArrayLength(params);
    jobjectArray global_params = nullptr;
    if (params != nullptr) {
        jobject element;
        for (int i = 0; i < arg_length; i++) {
            element = (jobject) env->GetObjectArrayElement(params, i);
            env->SetObjectArrayElement(params, i, env->NewGlobalRef(element));
        }
        global_params = (jobjectArray) env->NewGlobalRef(params);
    }

    ReflectThreadAsyncArgument argument(reflect_getDeclaredMethod_internal);
    argument.setMethod(global_clazz, global_method_name, global_params);
    if (0 == ThreadAsyncUtils::threadAsync(argument)) {
        return argument.jcls_result;
    }
    return NULL;
}

```

此方法作为当获取失败时，再调用此方法补偿，由于方案实现为异步线程转同步，故效率低下，通常只有在我们确定存在但获取失败的时候才会使用。

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)Gxd1703 谢楼主分享优秀资源
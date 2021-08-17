> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268936.htm)

> [原创] 安卓 hiddenapi 访问绝技

[下面摘录自官方](https://developer.android.com/guide/app-compatibility/restrictions-non-sdk-interfaces)

针对非 SDK 接口的限制
-------------

从 Android 9（API 级别 28）开始，Android 平台对应用能使用的非 SDK 接口实施了限制。只要应用引用非 SDK 接口或尝试使用反射或 JNI 来获取其句柄，这些限制就适用。这些限制旨在帮助提升用户体验和开发者体验，为用户降低应用发生崩溃的风险，同时为开发者降低紧急发布的风险。如需详细了解有关此限制的决定，请参阅[通过减少非 SDK 接口的使用来提高稳定性](https://android-developers.googleblog.com/2018/02/improving-stability-by-reducing-usage.html)。

非 SDK API 名单
------------

随着每个 Android 版本的发布，会有更多非 SDK 接口受到限制。 我们知道这些限制会影响您的发布工作流，同时我们希望确保您拥有相关工具来检测非 SDK 接口的使用情况、有机会[向我们提供反馈](https://developer.android.com/guide/app-compatibility/restrictions-non-sdk-interfaces#feature-request)，并且有时间根据相应新政策做出规划和调整。

 

为最大程度地降低非 SDK 使用限制对开发工作流的影响，我们将非 SDK 接口分成了几个名单，这些名单界定了非 SDK 接口使用限制的严格程度（取决于应用的目标 API 级别）。下表介绍了这些名单：

<table><thead><tr><th>名单</th><th>说明</th></tr></thead><tbody><tr><td>屏蔽名单 (<code>blacklist</code>)</td><td>无论应用的<a href="https://developer.android.com/distribute/best-practices/develop/target-sdk">目标 API 级别</a>是什么，您都无法使用的非 SDK 接口。 如果您的应用尝试访问其中任何一个接口，系统就会<a href="https://developer.android.com/guide/app-compatibility/restrictions-non-sdk-interfaces#results-of-keeping-non-sdk">抛出错误</a>。</td></tr><tr><td>有条件屏蔽 (<code>greylist-max-x</code>)</td><td>从 Android 9（API 级别 28）开始，当有应用以该 API 级别为目标平台时，我们会在每个 API 级别分别限制某些非 SDK 接口。这些名单会以应用无法再访问该名单中的非 SDK 接口之前可以作为目标平台的最高 API 级别 (<code>max-target-x</code>) 进行标记。例如，在 Android Pie 中未被屏蔽、但现在已被 Android 10 屏蔽的非 SDK 接口会列入 <code>max-target-p</code> (<code>greylist-max-p</code>) 名单，其中的 “p” 表示 Pie 或 Android 9（API 级别 28）。如果您的应用尝试访问受目标 API 级别限制的接口，系统就会<a href="https://developer.android.com/guide/app-compatibility/restrictions-non-sdk-interfaces#results-of-keeping-non-sdk">将此 API 视为已列入屏蔽名单</a>。</td></tr><tr><td>不支持 (<code>greylist</code>)</td><td>当前不受限制且您的应用可以使用的非 SDK 接口。 但请注意，这些接口<strong>不受支持</strong>，可能会在未发出通知的情况下随时发生更改。预计这些接口在未来的 Android 版本中会被有条件地屏蔽，并列在 <code>max-target-x</code> 名单中。</td></tr><tr><td>SDK (<code>whitelist</code>)</td><td>已在 Android 框架<a href="https://developer.android.com/reference/packages">软件包索引</a>中正式记录、受支持并且可以自由使用的接口。</td></tr></tbody></table>

 

尽管您目前仍可以使用某些非 SDK 接口（取决于应用的目标 API 级别），但只要您使用任何非 SDK 方法或字段，终归存在导致应用出问题的显著风险。如果您的应用依赖于非 SDK 接口，建议您开始计划迁移到 SDK 接口或其他替代方案。如果您无法为应用中的功能找到无需使用非 SDK 接口的替代方案，我们建议您[请求添加新的公共 API](https://developer.android.com/guide/app-compatibility/restrictions-non-sdk-interfaces#feature-request)。

访问受限的非 SDK 接口时可能会出现的预期行为
------------------------

下表说明了当您的应用尝试访问屏蔽名单中的非 SDK 接口时可能会出现的预期行为。

<table><thead><tr><th>访问方式</th><th>结果</th></tr></thead><tbody><tr><td>Dalvik 指令引用某个字段</td><td>抛出 <code>NoSuchFieldError</code></td></tr><tr><td>Dalvik 指令引用某个方法</td><td>抛出 <code>NoSuchMethodError</code></td></tr><tr><td>通过 <code>Class.getDeclaredField()</code> 或 <code>Class.getField()</code> 进行反射</td><td>抛出 <code>NoSuchFieldException</code></td></tr><tr><td>通过 <code>Class.getDeclaredMethod()</code>、<code>Class.getMethod()</code> 进行反射</td><td>抛出 <code>NoSuchMethodException</code></td></tr><tr><td>通过 <code>Class.getDeclaredFields()</code>、<code>Class.getFields()</code> 进行反射</td><td>结果中未获取到非 SDK 成员</td></tr><tr><td>通过 <code>Class.getDeclaredMethods()</code>、<code>Class.getMethods()</code> 进行反射</td><td>结果中未获取到非 SDK 成员</td></tr><tr><td>通过 <code>env-&gt;GetFieldID()</code> 进行 JNI 调用</td><td>返回 <code>NULL</code>，抛出 <code>NoSuchFieldError</code></td></tr><tr><td>通过 <code>env-&gt;GetMethodID()</code> 进行 JNI 调用</td><td>返回 <code>NULL</code>，抛出 <code>NoSuchMethodError</code></td></tr></tbody></table>

从官网的 sdk api 介绍中，我们可获得以下几种信息：
-----------------------------

1. 谷歌建议开发者尽量使用公开的 sdk 接口，并没有禁止非 sdk 接口的使用

 

2. 非 sdk 接口，开发者一般都是通过反射的机制进行使用，api>=28 会进行限制，也就是说访问非 sdk 接口可能会失败

 

3. 非 sdk 接口会进行等级分类，包括 blacklist, greylist-max-x, greylist, whitelist 等，其中 greylist 与 whitelist 不受限制

 

4. 非 sdk 接口的等级可能会随着 targetSdk 以及系统 sdk 版本变化

### [](#写个demo，测试一下访问blacklist-api)写个 demo，测试一下访问 blacklist-api

Blacklist-api 的配置文件一般在 aops 源码的 aosp/frameworks/base/config/hiddenapi-force-blacklist.txt

```
//android-11.0.0_r36
liuzhuangzhuang@virtualbox:~/work/aosp/frameworks/base$
liuzhuangzhuang@virtualbox:~/work/aosp/frameworks/base$ vim config/hiddenapi-force-blacklist.txt
 
Ldalvik/system/VMRuntime;->setHiddenApiExemptions([Ljava/lang/String;)V
Ldalvik/system/VMRuntime;->setTargetSdkVersion(I)V
Ldalvik/system/VMRuntime;->setTargetSdkVersionNative(I)V
Ljava/lang/invoke/MethodHandles$Lookup;->IMPL_LOOKUP:Ljava/lang/invoke/MethodHandles$Lookup;
Ljava/lang/invoke/VarHandle;->acquireFence()V
Ljava/lang/invoke/VarHandle;->compareAndExchange([Ljava/lang/Object;)Ljava/lang/Object;
Ljava/lang/invoke/VarHandle;->compareAndExchangeAcquire([Ljava/lang/Object;)Ljava/lang/Object;
Ljava/lang/invoke/VarHandle;->compareAndExchangeRelease([Ljava/lang/Object;)Ljava/lang/Object;
Ljava/lang/invoke/VarHandle;->compareAndSet([Ljava/lang/Object;)Z
Ljava/lang/invoke/VarHandle;->fullFence()V
Ljava/lang/invoke/VarHandle;->get([Ljava/lang/Object;)Ljava/lang/Object;
Ljava/lang/invoke/VarHandle;->getAcquire([Ljava/lang/Object;)Ljava/lang/Object;
Ljava/lang/invoke/VarHandle;->getAndAdd([Ljava/lang/Object;)Ljava/lang/Object;
Ljava/lang/invoke/VarHandle;->getAndAddAcquire([Ljava/lang/Object;)Ljava/lang/Object;
Ljava/lang/invoke/VarHandle;->getAndAddRelease([Ljava/lang/Object;)Ljava/lang/Object;
Ljava/lang/invoke/VarHandle;->getAndBitwiseAnd([Ljava/lang/Object;)Ljava/lang/Object;
Ljava/lang/invoke/VarHandle;->getAndBitwiseAndAcquire([Ljava/lang/Object;)Ljava/lang/Object;
Ljava/lang/invoke/VarHandle;->getAndBitwiseAndRelease([Ljava/lang/Object;)Ljava/lang/Object;
Ljava/lang/invoke/VarHandle;->getAndBitwiseOr([Ljava/lang/Object;)Ljava/lang/Object;
Ljava/lang/invoke/VarHandle;->getAndBitwiseOrAcquire([Ljava/lang/Object;)Ljava/lang/Object;
Ljava/lang/invoke/VarHandle;->getAndBitwiseOrRelease([Ljava/lang/Object;)Ljava/lang/Object;
Ljava/lang/invoke/VarHandle;->getAndBitwiseXor([Ljava/lang/Object;)Ljava/lang/Object;
Ljava/lang/invoke/VarHandle;->getAndBitwiseXorAcquire([Ljava/lang/Object;)Ljava/lang/Object;
Ljava/lang/invoke/VarHandle;->getAndBitwiseXorRelease([Ljava/lang/Object;)Ljava/lang/Object;
Ljava/lang/invoke/VarHandle;->getAndSet([Ljava/lang/Object;)Ljava/lang/Object;
Ljava/lang/invoke/VarHandle;->getAndSetAcquire([Ljava/lang/Object;)Ljava/lang/Object;
Ljava/lang/invoke/VarHandle;->getAndSetRelease([Ljava/lang/Object;)Ljava/lang/Object;
Ljava/lang/invoke/VarHandle;->getOpaque([Ljava/lang/Object;)Ljava/lang/Object;
Ljava/lang/invoke/VarHandle;->getVolatile([Ljava/lang/Object;)Ljava/lang/Object;
Ljava/lang/invoke/VarHandle;->loadLoadFence()V
Ljava/lang/invoke/VarHandle;->releaseFence()V
Ljava/lang/invoke/VarHandle;->set([Ljava/lang/Object;)V
Ljava/lang/invoke/VarHandle;->setOpaque([Ljava/lang/Object;)V
Ljava/lang/invoke/VarHandle;->setRelease([Ljava/lang/Object;)V
Ljava/lang/invoke/VarHandle;->setVolatile([Ljava/lang/Object;)V
Ljava/lang/invoke/VarHandle;->storeStoreFence()V
Ljava/lang/invoke/VarHandle;->weakCompareAndSet([Ljava/lang/Object;)Z
Ljava/lang/invoke/VarHandle;->weakCompareAndSetAcquire([Ljava/lang/Object;)Z
Ljava/lang/invoke/VarHandle;->weakCompareAndSetPlain([Ljava/lang/Object;)Z
Ljava/lang/invoke/VarHandle;->weakCompareAndSetRelease([Ljava/lang/Object;)Z

```

以访问 Ldalvik/system/VMRuntime;->setTargetSdkVersion(I)V 为例

```
try {
    Class runtimeClass = Class.forName("dalvik.system.VMRuntime");
    Method nativeLoadMethod = runtimeClass.getDeclaredMethod("setTargetSdkVersionNative",
            new Class[] {int.class});
 
    Log.d("whulzz", "setTargetSdkVersionNative success!");
} catch (Exception e) {
    e.printStackTrace();
}

```

然后运行时抛出了异常 NoSuchMethodException

```
08-16 12:57:13.379 21326 21326 W System.err: java.lang.NoSuchMethodException: dalvik.system.VMRuntime.setTargetSdkVersionNative [int]
08-16 12:57:13.380 21326 21326 W System.err:    at java.lang.Class.getMethod(Class.java:2072)
08-16 12:57:13.380 21326 21326 W System.err:    at java.lang.Class.getDeclaredMethod(Class.java:2050)
08-16 12:57:13.380 21326 21326 W System.err:    at com.example.demo.test.ApiFakeTest.fakeHiddenAPiTest(ApiFakeTest.java:17)
08-16 12:57:13.380 21326 21326 W System.err:    at com.example.demo.DemoApplication.onCreate(DemoApplication.java:26)
08-16 12:57:13.380 21326 21326 W System.err:    at android.app.Instrumentation.callApplicationOnCreate(Instrumentation.java:1190)
08-16 12:57:13.380 21326 21326 W System.err:    at android.app.ActivityThread.handleBindApplication(ActivityThread.java:6584)
08-16 12:57:13.380 21326 21326 W System.err:    at android.app.ActivityThread.access$1400(ActivityThread.java:226)
08-16 12:57:13.380 21326 21326 W System.err:    at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1889)
08-16 12:57:13.380 21326 21326 W System.err:    at android.os.Handler.dispatchMessage(Handler.java:107)
08-16 12:57:13.380 21326 21326 W System.err:    at android.os.Looper.loop(Looper.java:225)
08-16 12:57:13.380 21326 21326 W System.err:    at android.app.ActivityThread.main(ActivityThread.java:7564)
08-16 12:57:13.380 21326 21326 W System.err:    at java.lang.reflect.Method.invoke(Native Method)
08-16 12:57:13.380 21326 21326 W System.err:    at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:539)
08-16 12:57:13.381 21326 21326 W System.err:    at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:950)

```

[](#思考：)思考：
===========

1. 能不能继续使用反射访问非 sdk?
--------------------

​ 非 sdk 接口，greylist 以及 whitelist 不受限制，但是 blacklist 以及 greylist-max-x 会进行限制

2. 如何访问 blacklist 以及 greylist-max-x 呢？
--------------------------------------

​ 这个是本文的主题，后面会通过手段达到目的

声明: 跟 freereflection 比较
=======================

<table><thead><tr><th></th><th>本文</th><th>Free reflection</th></tr></thead><tbody><tr><td>setHiddenApiExemptions</td><td>是</td><td>是</td></tr><tr><td>dex 放到 bootclasspath</td><td>否</td><td>是</td></tr><tr><td>元反射</td><td>否</td><td>是</td></tr><tr><td>兼容安卓 11</td><td>是</td><td>不太清楚</td></tr><tr><td>第三方 ROM 兼容性得分</td><td>优秀</td><td>良好</td></tr></tbody></table>

从 android framework 的角度分析非 sdk 接口限制的原理，查找漏洞
===========================================

java.lang.Class;->getDeclaredMethod 函数源码
----------------------------------------

```
public Method getDeclaredMethod(String name, Class... parameterTypes)
    throws NoSuchMethodException, SecurityException {
    return getMethod(name, parameterTypes, false);
}

```

其内部调用了 getMethod 函数

```
private Method getMethod(String name, Class[] parameterTypes, boolean recursivePublicMethods)
        throws NoSuchMethodException {
    if (name == null) {
        throw new NullPointerException("name == null");
    }
    if (parameterTypes == null) {
        parameterTypes = EmptyArray.CLASS;
    }
    for (Class c : parameterTypes) {
        if (c == null) {
            throw new NoSuchMethodException("parameter type is null");
        }
    }
    Method result = recursivePublicMethods ? getPublicMethodRecursive(name, parameterTypes)
                                           : getDeclaredMethodInternal(name, parameterTypes);
    // Fail if we didn't find the method or it was expected to be public.
    if (result == null ||
        (recursivePublicMethods && !Modifier.isPublic(result.getAccessFlags()))) {
        throw new NoSuchMethodException(getName() + "." + name + " "
                + Arrays.toString(parameterTypes));
    }
    return result;
}

```

第三个参数 recursivePublicMethods 为 false，所以内部实际调用的是 getDeclaredMethodInternal

```
@FastNative
private native Method getDeclaredMethodInternal(String name, Class[] args);

```

getDeclaredMethodInternal 为 native 函数，会调用到 c++，源码可在 art/runtime/native/java_lang_Class.cc 中查找

 

大家可以留意一下 art/runtime/native 这个目录，包含了很多 native-jni 的函数实现

```
liuzhuangzhuang.das@virtualbox:~/work/aosp/art/runtime/native$ ls
dalvik_system_BaseDexClassLoader.cc   java_lang_Object.cc                  java_lang_reflect_Proxy.cc  java_util_concurrent_atomic_AtomicLong.cc
dalvik_system_BaseDexClassLoader.h    java_lang_Object.h                   java_lang_reflect_Proxy.h   java_util_concurrent_atomic_AtomicLong.h
dalvik_system_DexFile.cc              java_lang_ref_FinalizerReference.cc  java_lang_ref_Reference.cc  libcore_util_CharsetUtils.cc
dalvik_system_DexFile.h               java_lang_ref_FinalizerReference.h   java_lang_ref_Reference.h   libcore_util_CharsetUtils.h
dalvik_system_VMDebug.cc              java_lang_reflect_Array.cc           java_lang_String.cc         native_util.h
dalvik_system_VMDebug.h               java_lang_reflect_Array.h            java_lang_StringFactory.cc  org_apache_harmony_dalvik_ddmc_DdmServer.cc
dalvik_system_VMRuntime.cc            java_lang_reflect_Constructor.cc     java_lang_StringFactory.h   org_apache_harmony_dalvik_ddmc_DdmServer.h
dalvik_system_VMRuntime.h             java_lang_reflect_Constructor.h      java_lang_String.h          org_apache_harmony_dalvik_ddmc_DdmVmInternal.cc
dalvik_system_VMStack.cc              java_lang_reflect_Executable.cc      java_lang_System.cc         org_apache_harmony_dalvik_ddmc_DdmVmInternal.h
dalvik_system_VMStack.h               java_lang_reflect_Executable.h       java_lang_System.h          scoped_fast_native_object_access.h
dalvik_system_ZygoteHooks.cc          java_lang_reflect_Field.cc           java_lang_Thread.cc         scoped_fast_native_object_access-inl.h
dalvik_system_ZygoteHooks.h           java_lang_reflect_Field.h            java_lang_Thread.h          sun_misc_Unsafe.cc
java_lang_Class.cc                    java_lang_reflect_Method.cc          java_lang_Throwable.cc      sun_misc_Unsafe.h
java_lang_Class.h                     java_lang_reflect_Method.h           java_lang_Throwable.h
java_lang_invoke_MethodHandleImpl.cc  java_lang_reflect_Parameter.cc       java_lang_VMClassLoader.cc
java_lang_invoke_MethodHandleImpl.h   java_lang_reflect_Parameter.h        java_lang_VMClassLoader.h

```

java_lang_Class.cc 中 getDeclaredMethodInternal 源码
-------------------------------------------------

```
static jobject Class_getDeclaredMethodInternal(JNIEnv* env, jobject javaThis,
                                               jstring name, jobjectArray args) {
  ScopedFastNativeObjectAccess soa(env);
  StackHandleScope<1> hs(soa.Self());
  DCHECK_EQ(Runtime::Current()->GetClassLinker()->GetImagePointerSize(), kRuntimePointerSize);
  DCHECK(!Runtime::Current()->IsActiveTransaction());
  ObjPtr klass = DecodeClass(soa, javaThis);/*1.获取class.*/
  if (UNLIKELY(klass->IsObsoleteObject())) {
    ThrowRuntimeException("Obsolete Object!");
    return nullptr;
  }
  Handle result = hs.NewHandle(
      mirror::Class::GetDeclaredMethodInternal(/*2.调用GetDeclaredMethodInternal*/
          soa.Self(),
          klass,
          soa.Decode(name),
          soa.Decode>(args),
          GetHiddenapiAccessContextFunction(soa.Self())));/*3.hiddenapi访问上下文*/
  if (result == nullptr || ShouldDenyAccessToMember(result->GetArtMethod(), soa.Self())) {
    return nullptr;
  }
  return soa.AddLocalReference(result.Get());
} 
```

其内部调用了 mirror::Class::GetDeclaredMethodInternal

```
vim art/runtime//mirror/class.cc
template ObjPtr Class::GetDeclaredMethodInternal(
    Thread* self,
    ObjPtr klass,
    ObjPtr name,
    ObjPtr> args,
    const std::function& fn_get_access_context) {
  // Covariant return types (or smali) permit the class to define
  // multiple methods with the same name and parameter types.
  // Prefer (in decreasing order of importance):
  //  1) non-hidden method over hidden
  //  2) virtual methods over direct
  //  3) non-synthetic methods over synthetic
  // We never return miranda methods that were synthesized by the runtime.
  StackHandleScope<3> hs(self);
  auto h_method_name = hs.NewHandle(name);
  if (UNLIKELY(h_method_name == nullptr)) {
    ThrowNullPointerException("name == null");
    return nullptr;
  }
  auto h_args = hs.NewHandle(args);
  Handle h_klass = hs.NewHandle(klass);
  constexpr hiddenapi::AccessMethod access_method = hiddenapi::AccessMethod::kNone;
  ArtMethod* result = nullptr;
  bool result_hidden = false;
  for (auto& m : h_klass->GetDeclaredVirtualMethods(kPointerSize)) {/*4.遍历virtual method*/
    if (m.IsMiranda()) {
      continue;
    }
    auto* np_method = m.GetInterfaceMethodIfProxy(kPointerSize);
    // May cause thread suspension.
    ObjPtr np_name = np_method->ResolveNameString();
    if (!np_name->Equals(h_method_name.Get()) || !np_method->EqualParameters(h_args)) {/*5.判断方法名与参数类型*/
      if (UNLIKELY(self->IsExceptionPending())) {
        return nullptr;
      }
      continue;
    }
    bool m_hidden = hiddenapi::ShouldDenyAccessToMember(&m, fn_get_access_context, access_method);/*6.调用ShouldDenyAccessToMember*/
    if (!m_hidden && !m.IsSynthetic()) {
      // Non-hidden, virtual, non-synthetic. Best possible result, exit early.
      return Method::CreateFromArtMethod(self, &m);
    } else if (IsMethodPreferredOver(result, result_hidden, &m, m_hidden)) {
      // Remember as potential result.
      result = &m;
      result_hidden = m_hidden;
    }
  }
 
  if ((result != nullptr) && !result_hidden) {
    // We have not found a non-hidden, virtual, non-synthetic method, but
    // if we have found a non-hidden, virtual, synthetic method, we cannot
    // do better than that later.
    DCHECK(!result->IsDirect());
    DCHECK(result->IsSynthetic());
  } else {
    for (auto& m : h_klass->GetDirectMethods(kPointerSize)) {/*7.遍历direct method*/
      auto modifiers = m.GetAccessFlags();
      if ((modifiers & kAccConstructor) != 0) {
        continue;
      }
      auto* np_method = m.GetInterfaceMethodIfProxy(kPointerSize);
      // May cause thread suspension.
      ObjPtr np_name = np_method->ResolveNameString();
      if (np_name == nullptr) {
        self->AssertPendingException();
        return nullptr;
      }
      if (!np_name->Equals(h_method_name.Get()) || !np_method->EqualParameters(h_args)) {/*8.判断方法名与参数类型*/
        if (UNLIKELY(self->IsExceptionPending())) {
          return nullptr;
        }
        continue;
      }
      DCHECK(!m.IsMiranda());  // Direct methods cannot be miranda methods.
      bool m_hidden = hiddenapi::ShouldDenyAccessToMember(&m, fn_get_access_context, access_method);/*9.调用ShouldDenyAccessToMember*/
      if (!m_hidden && !m.IsSynthetic()) {
        // Non-hidden, direct, non-synthetic. Any virtual result could only have been
        // hidden, therefore this is the best possible match. Exit now.
        DCHECK((result == nullptr) || result_hidden);
        return Method::CreateFromArtMethod(self, &m);
      } else if (IsMethodPreferredOver(result, result_hidden, &m, m_hidden)) {
        // Remember as potential result.
        result = &m;
        result_hidden = m_hidden;
      }
    }
  }
 
  return result != nullptr
      ? Method::CreateFromArtMethod(self, result)
      : nullptr;
} 
```

源码中关键的地方做了标注，上述代码中有 9 个关键的地方，大家可自行查看，其中 ShouldDenyAccessToMember 为非 sdk 接口限制的关键函数

接下来看看 ShouldDenyAccessToMember 函数源码
-----------------------------------

```
template inline bool ShouldDenyAccessToMember(T* member,
                                     const std::function& fn_get_access_context,
                                     AccessMethod access_method)
    REQUIRES_SHARED(Locks::mutator_lock_) {
  DCHECK(member != nullptr);
 
  // Get the runtime flags encoded in member's access flags.
  // Note: this works for proxy methods because they inherit access flags from their
  // respective interface methods.
  const uint32_t runtime_flags = GetRuntimeFlags(member);
 
  // Exit early if member is public API. This flag is also set for non-boot class
  // path fields/methods.
  if ((runtime_flags & kAccPublicApi) != 0) {/*1.如果方法是public api，则允许访问*/
    return false;
  }
 
  // Determine which domain the caller and callee belong to.
  // This can be *very* expensive. This is why ShouldDenyAccessToMember
  // should not be called on every individual access.
  const AccessContext caller_context = fn_get_access_context();/*2.获取caller的上下文*/
  const AccessContext callee_context(member->GetDeclaringClass());/*3.获取所调用方法的上下文*/
 
  // Non-boot classpath callers should have exited early.
  DCHECK(!callee_context.IsApplicationDomain());
 
  // Check if the caller is always allowed to access members in the callee context.
  if (caller_context.CanAlwaysAccess(callee_context)) {/*4.caller是否可以不受约束访问callee*/
    return false;
  }
 
  // Check if this is platform accessing core platform. We may warn if `member` is
  // not part of core platform API.
  /*5.根据domain级别区分对待hiddenapi策略*/
  switch (caller_context.GetDomain()) {
    case Domain::kApplication: {
      DCHECK(!callee_context.IsApplicationDomain());
 
      // Exit early if access checks are completely disabled.
      EnforcementPolicy policy = Runtime::Current()->GetHiddenApiEnforcementPolicy();
      if (policy == EnforcementPolicy::kDisabled) {/*5.1.如果policydisable，则返回false，即允许访问*/
        return false;
      }
 
      // If this is a proxy method, look at the interface method instead.
      member = detail::GetInterfaceMemberIfProxy(member);
 
      // Decode hidden API access flags from the dex file.
      // This is an O(N) operation scaling with the number of fields/methods
      // in the class. Only do this on slow path and only do it once.
      ApiList api_list(detail::GetDexFlags(member));
      DCHECK(api_list.IsValid());
 
      // Member is hidden and caller is not exempted. Enter slow path.
      return detail::ShouldDenyAccessToMemberImpl(member, api_list, access_method);
    }
 
    case Domain::kPlatform: {
      DCHECK(callee_context.GetDomain() == Domain::kCorePlatform);
 
      // Member is part of core platform API. Accessing it is allowed.
      if ((runtime_flags & kAccCorePlatformApi) != 0) {
        return false;
      }
 
      // Allow access if access checks are disabled.
      EnforcementPolicy policy = Runtime::Current()->GetCorePlatformApiEnforcementPolicy();
      if (policy == EnforcementPolicy::kDisabled) {
        return false;
      }
 
      // If this is a proxy method, look at the interface method instead.
      member = detail::GetInterfaceMemberIfProxy(member);
 
      // Access checks are not disabled, report the violation.
      // This may also add kAccCorePlatformApi to the access flags of `member`
      // so as to not warn again on next access.
      return detail::HandleCorePlatformApiViolation(member,
                                                    caller_context,
                                                    access_method,
                                                    policy);
    }                                              
 
    case Domain::kCorePlatform: {
      LOG(FATAL) << "CorePlatform domain should be allowed to access all domains";
      UNREACHABLE();
    }
  }
} 
```

### [](#上述代码中有5处标注的关键点，为了方便理解后续的fn_get_access_context，先在这里提一下这玩意。)上述代码中有 5 处标注的关键点，为了方便理解后续的 fn_get_access_context，先在这里提一下这玩意。

fn_get_access_context 的源头是在 java_lang_Class->getMethodIdInternal 传过来的函数指针，即 GetHiddenapiAccessContextFunction

```
//java_lang_Class.cc
    Handle result = hs.NewHandle(
      mirror::Class::GetDeclaredMethodInternal(/*2.调用GetDeclaredMethodInternal*/
          soa.Self(),
          klass,
          soa.Decode(name),
          soa.Decode>(args),
          GetHiddenapiAccessContextFunction(soa.Self())));/*3.hiddenapi访问上下文*/ 
```

```
//java_lang_Class.cc
static std::function GetHiddenapiAccessContextFunction(Thread* self) {
  return [=]() REQUIRES_SHARED(Locks::mutator_lock_) { return GetReflectionCaller(self); };
} 
```

### 1. 如果方法是 public api，则允许访问

```
if ((runtime_flags & kAccPublicApi) != 0) {/*1.如果方法是public api，则允许访问*/
  return false;
}

```

### 2. 获取 caller 的上下文

```
const AccessContext caller_context = fn_get_access_context();/*2.获取caller的上下文*/

```

### 3. 获取所调用方法的上下文

```
const AccessContext callee_context(member->GetDeclaringClass());/*3.获取所调用方法的上下文*/

```

### 4.caller 是否可以不受约束访问 callee

```
// Check if the caller is always allowed to access members in the callee context.
if (caller_context.CanAlwaysAccess(callee_context)) {/*4.caller是否可以不受约束访问callee*/
  return false;
}

```

```
//hidden_api.h 
// Returns true if this domain is always allowed to access the domain of `callee`.
  bool CanAlwaysAccess(const AccessContext& callee) const {
    return IsDomainMoreTrustedThan(domain_, callee.domain_);
  }

```

```
//art/libartbase/base/hiddenapi_domain.h
enum class Domain : char {
  kCorePlatform = 0,
  kPlatform,
  kApplication,
};
 
inline bool IsDomainMoreTrustedThan(Domain domainA, Domain domainB) {
  return static_cast(domainA) <= static_cast(domainB);
} 
```

也就是说，如果 caller 的 domain 值越小，能访问的 hiddenapi 范围越广。比如 corePlatform 能访问所有级别的 api，但是 application 级别不能访问 corePlatform 以及 platform 级别的 api。

 

###

### 5. 第三方 app 肯定会走到这里，根据 domain 级别区分对待 hiddenapi 策略

#### 5.1 如果 policydisable，则返回 false，即允许访问 (重点)

#### 5.2 最终会调用 detail::ShouldDenyAccessToMemberImpl(member, api_list, access_method)

```
template bool ShouldDenyAccessToMemberImpl(T* member, ApiList api_list, AccessMethod access_method) {
  DCHECK(member != nullptr);
  Runtime* runtime = Runtime::Current();
 
  EnforcementPolicy hiddenApiPolicy = runtime->GetHiddenApiEnforcementPolicy();
  DCHECK(hiddenApiPolicy != EnforcementPolicy::kDisabled)
      << "Should never enter this function when access checks are completely disabled";
 
  MemberSignature member_signature(member);
 
  // Check for an exemption first. Exempted APIs are treated as white list.
  if (member_signature.DoesPrefixMatchAny(runtime->GetHiddenApiExemptions())) {
    // Avoid re-examining the exemption list next time.
    // Note this results in no warning for the member, which seems like what one would expect.
    // Exemptions effectively adds new members to the whitelist.
    MaybeUpdateAccessFlags(runtime, member, kAccPublicApi);
    return false;
  }
.....
省略
.... 
```

这里返回了 false(重点), 后边的代码省略，因为当前已经找到了两处返回 false 的地方 (返回 false 表示可以访问)。

第一处返回 false 关键点:
----------------

```
EnforcementPolicy policy = Runtime::Current()->GetHiddenApiEnforcementPolicy();
if (policy == EnforcementPolicy::kDisabled) {/*5.1.如果policydisable，则返回false，即允许访问*/
  return false;
}

```

也就是说 policy 策略关闭时，可以自由访问 hiddenapi，该方案可使用[开源方案](https://github.com/tiann/FreeReflection)，但是需要适配不同的系统版本。

 

开源的方案是通过修改 runtime 内存实现的, 内存 hidden_api_policy_ 的偏移值可因为系统版本，也可因为厂家定制导致不统一，所以该方案兼容性问题较大。

```
hiddenapi::EnforcementPolicy GetHiddenApiEnforcementPolicy() const {
  return hidden_api_policy_;
}

```

第二处返回 false 关键点
---------------

runtime->GetHiddenApiExemptions, 顾名思义：获取豁免的 hiddenapi 签名

```
// Check for an exemption first. Exempted APIs are treated as white list.
if (member_signature.DoesPrefixMatchAny(runtime->GetHiddenApiExemptions())) {
  // Avoid re-examining the exemption list next time.
  // Note this results in no warning for the member, which seems like what one would expect.
  // Exemptions effectively adds new members to the whitelist.
  MaybeUpdateAccessFlags(runtime, member, kAccPublicApi);
  return false;
}

```

### MemberSignature::DoesPrefixMatchAny 函数

```
bool MemberSignature::DoesPrefixMatchAny(const std::vector& exemptions) {
  for (const std::string& exemption : exemptions) {
    if (DoesPrefixMatch(exemption)) {
      return true;
    }
  }
  return false;
} 
```

该函数会遍历 exemptions，只要有一个 exemption，匹配当前访问的 method->signature 前缀，就可返回 false。

```
bool MemberSignature::DoesPrefixMatch(const std::string& prefix) const {
  size_t pos = 0;
  for (const char* part : GetSignatureParts()) {
    size_t count = std::min(prefix.length() - pos, strlen(part));
    if (prefix.compare(pos, count, part, 0, count) == 0) {
      pos += count;
    } else {
      return false;
    }
  }
  // We have a complete match if all parts match (we exit the loop without
  // returning) AND we've matched the whole prefix.
  return pos == prefix.length();
}

```

### 接下来看看 MemberSignature->GetSignatureParts

```
inline std::vector MemberSignature::GetSignatureParts() const {
  if (type_ == kField) {
    return { class_name_.c_str(), "->", member_name_.c_str(), ":", type_signature_.c_str() };
  } else {
    DCHECK_EQ(type_, kMethod);
    return { class_name_.c_str(), "->", member_name_.c_str(), type_signature_.c_str() };
  }
} 
```

无论 MemberSignature 的 type_为 field 还是 method，其返回值都是以 class_name.c_str 为前缀

#### [](#membersignature的构造函数：)MemberSignature 的构造函数：

##### 类型 field

```
MemberSignature::MemberSignature(ArtField* field) {
  class_name_ = field->GetDeclaringClass()->GetDescriptor(&tmp_);
  member_name_ = field->GetName();
  type_signature_ = field->GetTypeDescriptor();
  type_ = kField;
}

```

##### 类型 method

```
MemberSignature::MemberSignature(ArtMethod* method) {
  DCHECK(method == method->GetInterfaceMethodIfProxy(kRuntimePointerSize))
      << "Caller should have replaced proxy method with interface method";
  class_name_ = method->GetDeclaringClass()->GetDescriptor(&tmp_);
  member_name_ = method->GetName();
  type_signature_ = method->GetSignature().ToString();
  type_ = kMethod;
}

```

另外，我们知道一个 class 的签名，形式都是如 Ljava/lang/String; 这种，肯定是以 L 开头的。

### [](#综上：所以exemption只要是l，就可以达到返回false的目的。)综上：所以 exemption 只要是 L，就可以达到返回 false 的目的。

接下来分析 HiddenApiExemptions，从 runtime->GetHiddenApiExemptions 着手
==============================================================

```
//art/runtime/runtime.h
  void SetHiddenApiExemptions(const std::vector& exemptions) {
    hidden_api_exemptions_ = exemptions;
  }
 
  const std::vector& GetHiddenApiExemptions() {
    return hidden_api_exemptions_;
  } 
```

很明显可以通过 SetHiddenApiExemptions 设置 exemptions
--------------------------------------------

### 搜一下 aosp 源码中哪里使用了 SetHiddenApiExemptions

```
liuzhuangzhuang.das@virtualbox:~/work/aosp/art/runtime$ cgrep SetHiddenApiExemptions
./runtime.h:611:  void SetHiddenApiExemptions(const std::vector& exemptions) {
./native/dalvik_system_VMRuntime.cc:97:  Runtime::Current()->SetHiddenApiExemptions(exemptions_vec); 
```

### dalvik_system_VMRuntime.cc：

```
static void VMRuntime_setHiddenApiExemptions(JNIEnv* env,
                                            jclass,
                                            jobjectArray exemptions) {
  std::vector exemptions_vec;
  int exemptions_length = env->GetArrayLength(exemptions);
  for (int i = 0; i < exemptions_length; i++) {
    jstring exemption = reinterpret_cast(env->GetObjectArrayElement(exemptions, i));
    const char* raw_exemption = env->GetStringUTFChars(exemption, nullptr);
    exemptions_vec.push_back(raw_exemption);
    env->ReleaseStringUTFChars(exemption, raw_exemption);
  }
 
  Runtime::Current()->SetHiddenApiExemptions(exemptions_vec);
} 
```

### VMRuntime_setHiddenApiExemptions 是 jni 方法，其 native 函数的声明在 VMRuntime.java 中。

```
/**
 * Provides an interface to VM-global, Dalvik-specific features.
 * An application cannot create its own Runtime instance, and must obtain
 * one from the getRuntime method.
 *
 * @hide
 */
@libcore.api.CorePlatformApi
public final class VMRuntime {
 
    /**
     * Sets the list of exemptions from hidden API access enforcement.
     *
     * @param signaturePrefixes
     *         A list of signature prefixes. Each item in the list is a prefix match on the type
     *         signature of a blacklisted API. All matching APIs are treated as if they were on
     *         the whitelist: access permitted, and no logging..
     */
    @libcore.api.CorePlatformApi
    public native void setHiddenApiExemptions(String[] signaturePrefixes);

```

### class VMRuntime 竟然也被声明为 hide，陷入了僵局？

也就是说，想要调用 setHiddenApiExemptions， 必须 fake 掉 hiddenapi 的限制。

### [](#之前的关键点中，caller是否可以不受约束访问callee，给了我们一些启示)之前的关键点中，caller 是否可以不受约束访问 callee，给了我们一些启示

#### [](#caller的上下文是这样的获得的，如果该上下文domain的值越小，拥有的hiddenapi访问权限越大。)caller 的上下文是这样的获得的，如果该上下文 domain 的值越小，拥有的 hiddenapi 访问权限越大。

```
const AccessContext caller_context = fn_get_access_context();/*2.获取caller的上下文*/

```

所以看一下 java_lang_Class 中的 GetHiddenapiAccessContextFunction

```
//java_lang_Class.cc
static std::function GetHiddenapiAccessContextFunction(Thread* self) {
  return [=]() REQUIRES_SHARED(Locks::mutator_lock_) { return GetReflectionCaller(self); };
} 
```

#### GetReflectionCaller 函数，返回 caller 的 AccessContext.

```
// Walks the stack, finds the caller of this reflective call and returns
// a hiddenapi AccessContext formed from its declaring class.
static hiddenapi::AccessContext GetReflectionCaller(Thread* self)
    REQUIRES_SHARED(Locks::mutator_lock_) {
  // Walk the stack and find the first frame not from java.lang.Class,
  // java.lang.invoke or java.lang.reflect. This is very expensive.
  // Save this till the last.
  struct FirstExternalCallerVisitor : public StackVisitor {
    explicit FirstExternalCallerVisitor(Thread* thread)
        : StackVisitor(thread, nullptr, StackVisitor::StackWalkKind::kIncludeInlinedFrames),
          caller(nullptr) {
    }
 
    bool VisitFrame() override REQUIRES_SHARED(Locks::mutator_lock_) {/*获取caller的关键函数*/
      ArtMethod *m = GetMethod();
      if (m == nullptr) {
        // Attached native thread. Assume this is *not* boot class path.
        caller = nullptr;
        return false;
      } else if (m->IsRuntimeMethod()) {
        // Internal runtime method, continue walking the stack.
        return true;
      }
 
      ObjPtr declaring_class = m->GetDeclaringClass();
      if (declaring_class->IsBootStrapClassLoaded()) {
        if (declaring_class->IsClassClass()) {
          return true;
        }
        // Check classes in the java.lang.invoke package. At the time of writing, the
        // classes of interest are MethodHandles and MethodHandles.Lookup, but this
        // is subject to change so conservatively cover the entire package.
        // NB Static initializers within java.lang.invoke are permitted and do not
        // need further stack inspection.
        ObjPtr lookup_class = GetClassRoot();
        if ((declaring_class == lookup_class || declaring_class->IsInSamePackage(lookup_class))
            && !m->IsClassInitializer()) {
          return true;
        }
        // Check for classes in the java.lang.reflect package, except for java.lang.reflect.Proxy.
        // java.lang.reflect.Proxy does its own hidden api checks (https://r.android.com/915496),
        // and walking over this frame would cause a null pointer dereference
        // (e.g. in 691-hiddenapi-proxy).
        ObjPtr proxy_class = GetClassRoot();
        if (declaring_class->IsInSamePackage(proxy_class) && declaring_class != proxy_class) {
          if (Runtime::Current()->isChangeEnabled(kPreventMetaReflectionBlacklistAccess)) {
            return true;
          }
        }
      }
 
      caller = m;
      return false;
    }
 
    ArtMethod* caller;
  };
 
  FirstExternalCallerVisitor visitor(self);
  visitor.WalkStack();
 
  // Construct AccessContext from the calling class found on the stack.
  // If the calling class cannot be determined, e.g. unattached threads,
  // we conservatively assume the caller is trusted.
  ObjPtr caller = (visitor.caller == nullptr)
      ? nullptr : visitor.caller->GetDeclaringClass();
  return caller.IsNull() ? hiddenapi::AccessContext(/* is_trusted= */ true)
                         : hiddenapi::AccessContext(caller);
} 
```

##### 最后一行 caller.IsNull() 的时候上下文传入了 true

```
//art/runtime/hidden_api.h
// Represents the API domain of a caller/callee.
class AccessContext {
 public:
  // Initialize to either the fully-trusted or fully-untrusted domain.
  explicit AccessContext(bool is_trusted)
      : klass_(nullptr),
        dex_file_(nullptr),
        domain_(ComputeDomain(is_trusted)) {}

```

在此，domain_通过 ComputeDomain 初始化

```
static Domain ComputeDomain(bool is_trusted) {
  return is_trusted ? Domain::kCorePlatform : Domain::kApplication;
}

```

###### [](#当is_trusted为true的时候，确实会获取级别最高的hiddenapi访问权限)当 is_trusted 为 true 的时候，确实会获取级别最高的 hiddenapi 访问权限

##### 再来看看 caller 不为 null 时

```
// Initialize from Class.
explicit AccessContext(ObjPtr klass)
    REQUIRES_SHARED(Locks::mutator_lock_)
    : klass_(klass),
      dex_file_(GetDexFileFromDexCache(klass->GetDexCache())),
      domain_(ComputeDomain(klass, dex_file_)) {} 
```

###### [](#通过kclass拿到dex_file，然后调用computedomain计算该dex_file的domain)通过 kclass 拿到 dex_file，然后调用 computeDomain 计算该 dex_file 的 domain

```
static Domain ComputeDomain(ObjPtr klass, const DexFile* dex_file)
    REQUIRES_SHARED(Locks::mutator_lock_) {
  // Check other aspects of the context.
  Domain domain = ComputeDomain(klass->GetClassLoader(), dex_file);
 
  if (domain == Domain::kApplication &&
      klass->ShouldSkipHiddenApiChecks() &&
      Runtime::Current()->IsJavaDebuggable()) {/*只有debugable的包才会走if中的逻辑*/
    // Class is known, it is marked trusted and we are in debuggable mode.
    domain = ComputeDomain(/* is_trusted= */ true);
  }
 
  return domain;
} 
```

###### 最终 dex_file 的 domain 值是通过 GetHiddenapiDomain() 获取的

```
static Domain ComputeDomain(ObjPtr class_loader, const DexFile* dex_file) {
  if (dex_file == nullptr) {
    return ComputeDomain(/* is_trusted= */ class_loader.IsNull());
  }
 
  return dex_file->GetHiddenapiDomain();
} 
```

```
hiddenapi::Domain GetHiddenapiDomain() const { return hiddenapi_domain_; }
void SetHiddenapiDomain(hiddenapi::Domain value) const { hiddenapi_domain_ = value; }

```

###### [](#接下来看看dex_file的domain值是如何设置的：)接下来看看 dex_file 的 domain 值是如何设置的：

 

直接搜索

```
liuzhuangzhuang.das@virtualbox:~/work/aosp/art$ cgrep SetHiddenapiDomain
./runtime/native/dalvik_system_DexFile.cc:899:    const_cast(dex_file)->SetHiddenapiDomain(hiddenapi::Domain::kCorePlatform);
./runtime/hidden_api.cc:125:    dex_file.SetHiddenapiDomain(dex_domain);=============//真正初始化dex_file的domain值是在这里
./libdexfile/dex/dex_file.h:767:  void SetHiddenapiDomain(hiddenapi::Domain value) const { hiddenapi_domain_ = value; }
./test/674-hiddenapi/hiddenapi.cc:49:    const_cast(dex_file.get())->SetHiddenapiDomain(
./openjdkjvmti/fixed_up_dex_file.cc:151:  new_dex_file->SetHiddenapiDomain(original.GetHiddenapiDomain()); 
```

```
//art/runtime/hidden_api.cc
void InitializeDexFileDomain(const DexFile& dex_file, ObjPtr class_loader) {
  Domain dex_domain = DetermineDomainFromLocation(dex_file.GetLocation(), class_loader);
 
  // Assign the domain unless a more permissive domain has already been assigned.
  // This may happen when DexFile is initialized as trusted.
  if (IsDomainMoreTrustedThan(dex_domain, dex_file.GetHiddenapiDomain())) {
    dex_file.SetHiddenapiDomain(dex_domain);
  }
} 
```

###### [](#determinedomainfromlocation顾名思义：根据dex_file的文件位置，计算出其domain值)DetermineDomainFromLocation 顾名思义：根据 dex_file 的文件位置，计算出其 domain 值

```
static Domain DetermineDomainFromLocation(const std::string& dex_location,
                                          ObjPtr class_loader) {
  // If running with APEX, check `path` against known APEX locations.
  // These checks will be skipped on target buildbots where ANDROID_ART_ROOT
  // is set to "/system".
  if (ArtModuleRootDistinctFromAndroidRoot()) {/*1.只是为了判断相关的dir路径是否存在*/
    if (LocationIsOnArtModule(dex_location.c_str()) ||/*2.dex的路径是否是在artModule*/
        LocationIsOnConscryptModule(dex_location.c_str())) {/*3.dex的路径是否是在ConscryptModule*/
      return Domain::kCorePlatform;
    }
 
    if (LocationIsOnApex(dex_location.c_str())) {/*4.dex的路径是否是在apex目录*/
      return Domain::kPlatform;
    }
  }
 
  if (LocationIsOnSystemFramework(dex_location.c_str())) {/*5.dex的路径是否是在system/framework目录*/
    return Domain::kPlatform;
  }
 
  if (class_loader.IsNull()) {
    LOG(WARNING) << "DexFile " << dex_location
        << " is in boot class path but is not in a known location";
    return Domain::kPlatform;
  }
 
  return Domain::kApplication;
} 
```

代码中一共判断了 5 个文件位置，分别对应不同的 domain

###### 这些位置可以在 init.environ.rc 中查看到:

```
on early-init
    export ANDROID_BOOTLOGO 1
    export ANDROID_ROOT /system
    export ANDROID_ASSETS /system/app
    export ANDROID_DATA /data
    export ANDROID_STORAGE /storage
    export ANDROID_ART_ROOT /apex/com.android.art
    export ANDROID_I18N_ROOT /apex/com.android.i18n
    export ANDROID_TZDATA_ROOT /apex/com.android.tzdata
    export EXTERNAL_STORAGE /sdcard
    export ASEC_MOUNTPOINT /mnt/asec
    export BOOTCLASSPATH /apex/com.android.art/javalib/core-oj.jar:/apex/com.android.art/javalib/core-libart.jar:/apex/com.android.art/javalib/core-icu4j.jar:/apex/com.android.art/javalib/okhttp.jar:/apex/com.android.art/javalib/bouncycastle.jar:/apex/com.android.art/javalib/apache-xml.jar:/system/framework/framework.jar:/system/framework/ext.jar:/system/framework/telephony-common.jar:/system/framework/voip-common.jar:/system/framework/ims-common.jar:/system/framework/framework-atb-backward-compatibility.jar:/apex/com.android.conscrypt/javalib/conscrypt.jar:/apex/com.android.media/javalib/updatable-media.jar:/apex/com.android.mediaprovider/javalib/framework-mediaprovider.jar:/apex/com.android.os.statsd/javalib/framework-statsd.jar:/apex/com.android.permission/javalib/framework-permission.jar:/apex/com.android.sdkext/javalib/framework-sdkextensions.jar:/apex/com.android.wifi/javalib/framework-wifi.jar:/apex/com.android.tethering/javalib/framework-tethering.jar

```

1. 只是为了判断相关 module 的 dir 是否存在，一般都是存在的

 

2.artModule 路径为 / apex/com.android.art(android 11)， 安卓 10 上的代码稍有区别

 

3.conscryptModule 路径为 / apex/com.android.conscrypt

 

4.apex 的路径为 / apex/

 

5.SystemFramework 的路径为 / system/framework

###### [](#如果caller的路径为artmodule或者conscryptmodule即可将domain值置为kcoreplatform，达到目的。)如果 caller 的路径为 artModule 或者 conscryptModule 即可将 domain 值置为 kCorePlatform，达到目的。

 

先随便找个 debug 的 app 看一下运行时 apex 路径都有哪些

```
ginkgo:/data/data/com.example.demo $ cat /proc/3485/maps |grep "/apex/.*.jar"                                                                                 
6fa5766000-6fa57d0000 r--p 00000000 fd:00 119                            /apex/com.android.conscrypt/javalib/conscrypt.jar
6faca42000-6facb6a000 r--p 00000000 fd:00 301                            /apex/com.android.runtime/javalib/apache-xml.jar
6facb6a000-6faccc0000 r--p 00000000 fd:00 302                            /apex/com.android.runtime/javalib/bouncycastle.jar
6faccc0000-6facfe6000 r--p 00004000 fd:00 303                            /apex/com.android.runtime/javalib/core-libart.jar
6facfe6000-6fad49e000 r--p 00000000 fd:00 304                            /apex/com.android.runtime/javalib/core-oj.jar
6fad524000-6fad588000 r--p 00000000 fd:00 305                            /apex/com.android.runtime/javalib/okhttp.jar
7031d66000-7031d75000 r--p 00000000 fd:00 137                            /apex/com.android.media/javalib/updatable-media.jar
70329ff000-7032a00000 r--s 0000e000 fd:00 137                            /apex/com.android.media/javalib/updatable-media.jar
7032a00000-7032a01000 r--s 00069000 fd:00 119                            /apex/com.android.conscrypt/javalib/conscrypt.jar
703313e000-7033140000 r--s 00329000 fd:00 303                            /apex/com.android.runtime/javalib/core-libart.jar
70331bb000-70331bd000 r--s 004be000 fd:00 304                            /apex/com.android.runtime/javalib/core-oj.jar
703344d000-703344e000 r--s 0012b000 fd:00 301                            /apex/com.android.runtime/javalib/apache-xml.jar
70334ff000-7033500000 r--s 00155000 fd:00 302                            /apex/com.android.runtime/javalib/bouncycastle.jar
703355a000-703355b000 r--s 00063000 fd:00 305                            /apex/com.android.runtime/javalib/okhttp.jar

```

### core-oj.jar 正是可以利用的点:

这里有一个 java.lang.System 类，该类我们经常使用其加载 so 库，比如 System.loadLibrary。

 

所以，我们可以通过 System.loadLibrary，然后在 native 层的 JNI_OnLoad 中通过反射调用 setHiddenApiExemptions(此时 caller 为 java.lang.System. 其 domain 级别为 corePlatform)，然后就可以随意访问 hiddenapi 了

总结:
===

1. 系统 framework 代码中可以通过设置 setHiddenApiExemptions，达到随意访问 hiddenapi 的目的
---------------------------------------------------------------------

2. 由于 class VMRuntime 被 hide，可以在 JNI_OnLoad 中操作 VMRuntime，达到调用 setHiddenApiExemptions 的目的
-----------------------------------------------------------------------------------------

[核心代码](https://github.com/whulzz1993/RePublic)：

```
public class ApiFakeTest {
    public static void fakeHiddenAPiTest() {
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.P) {
            return;
        }
        System.loadLibrary("apifake");
 
        try {
            Class runtimeClass = Class.forName("dalvik.system.VMRuntime");
            Method nativeLoadMethod = runtimeClass.getDeclaredMethod("setTargetSdkVersionNative",
                    new Class[] {int.class});
 
            Log.d("whulzz", "setTargetSdkVersionNative success!");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```

```
//
// Created by liuzhuangzhuang on 2021/8/16.
//
#define DEBUG
#include #include #define LOG_TAG "apifake"
#define DEBUG
#ifdef DEBUG
#define ALOGD(fmt, args...)  do {__android_log_print(ANDROID_LOG_DEBUG, LOG_TAG, fmt, ##args);} while(0)
#define ALOGI(fmt, args...)  do {__android_log_print(ANDROID_LOG_INFO, LOG_TAG, fmt, ##args);} while(0)
#else
#define ALOGD(fmt, args...)  do {} while(0)
#define ALOGI(fmt, args...)  do {} while(0)
#endif
#define ALOGE(fmt, args...)  do {__android_log_print(ANDROID_LOG_ERROR, LOG_TAG, fmt, ##args);} while(0)
 
typedef union {
    JNIEnv* env;
    void* venv;
} UnionJNIEnvToVoid;
 
bool setApiBlacklistExemptions(JNIEnv* env) {
    /*=================在这里使用了ZygoteInit,而不是VMRuntime, 效果一样==================*/
    jclass zygoteInitClass = env->FindClass("com/android/internal/os/ZygoteInit");
    if (zygoteInitClass == nullptr) {
        ALOGE("not found class");
        env->ExceptionClear();
        return false;
    }
 
 
    jmethodID setApiBlackListApiMethod =
            env->GetStaticMethodID(zygoteInitClass,
                    "setApiBlacklistExemptions",
                    "([Ljava/lang/String;)V");
    if (setApiBlackListApiMethod == nullptr) {
        env->ExceptionClear();
        setApiBlackListApiMethod =
                env->GetStaticMethodID(zygoteInitClass,
                        "setApiDenylistExemptions",
                        "([Ljava/lang/String;)V");
    }
 
    if (setApiBlackListApiMethod == nullptr) {
        ALOGE("not found method");
        return false;
    }
 
    jclass stringCLass = env->FindClass("java/lang/String");
 
    jstring fakeStr = env->NewStringUTF("L");
 
    jobjectArray fakeArray = env->NewObjectArray(
            1, stringCLass, NULL);
 
    env->SetObjectArrayElement(fakeArray, 0, fakeStr);
 
    env->CallStaticVoidMethod(zygoteInitClass,
            setApiBlackListApiMethod, fakeArray);
 
    env->DeleteLocalRef(fakeStr);
    env->DeleteLocalRef(fakeArray);
    ALOGD("fakeapi success!");
    return true;
}
 
jint JNI_OnLoad(JavaVM* vm, void* reserved) {
    UnionJNIEnvToVoid uenv;
    uenv.venv = NULL;
    jint result = -1;
    JNIEnv* env = NULL;
 
    ALOGD("JNI_OnLoad");
 
    if (vm->GetEnv(&uenv.venv, JNI_VERSION_1_6) != JNI_OK) {
        ALOGE("ERROR: GetEnv failed");
        goto bail;
    }
    env = uenv.env;
 
    if (!setApiBlacklistExemptions(env)) {
        ALOGE("failed");
        goto bail;
    }
    result = JNI_VERSION_1_6;
 
    bail:
    return result;
} 
```

测试通过:
-----

```
08-16 16:00:51.672  3485  3485 D apifake : JNI_OnLoad
08-16 16:00:51.672  3485  3485 D apifake : fakeapi success!
08-16 16:00:51.673  3485  3485 D whulzz  : setTargetSdkVersionNative success!
08-16 16:00:51.678   606  1217 I netd    : tetherGetStats() <1.74ms>

```

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

最后于 12 小时前 被 whulzz 编辑 ，原因： 添加 git 仓库

[#基础理论](forum-161-1-117.htm) [#系统相关](forum-161-1-126.htm) [#源码分析](forum-161-1-127.htm)
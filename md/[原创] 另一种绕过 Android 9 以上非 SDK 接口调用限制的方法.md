> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268523.htm)

> [原创] 另一种绕过 Android 9 以上非 SDK 接口调用限制的方法

原文：[另一种绕过 Android 9 以上非 SDK 接口调用限制的方法](https://windysha.github.io/2021/07/20/%E5%8F%A6%E4%B8%80%E7%A7%8D%E7%BB%95%E8%BF%87%20Android%209%E4%BB%A5%E4%B8%8A%E9%9D%9ESDK%E6%8E%A5%E5%8F%A3%E8%B0%83%E7%94%A8%E9%99%90%E5%88%B6%E7%9A%84%E6%96%B9%E6%B3%95/)

前言
==

从 Android9 开始，Google 开始在 Android 平台限制对非 SDK 接口的调用。只要应用使用非 SDK 接口，或尝试使用反射或 JNI 来调用非 SDK 接口，都会收到某些限制。而且随着 Android 版本的升级，这种限制越来越强，被限制的接口也越来越多。这些限制对于 Android 平台上的一些黑科技来说 (插件化，热修复，App 双开，性能监控，Art Hook 等)，简直就是致命的。所以，各路大神都纷纷寻找绕过这个限制的手段。

 

最近在翻阅 Android ART 虚拟机源码时，发现另外一种简单绕过限制的方法。经测试，能够在 Android 9-12 上稳定运行。

 

下面先介绍主流的绕过限制的原理，再详细介绍这种全新的绕过策略，并在最后给出完整的源码实现。

非 SDK 接口如何被限制
=============

一般调用一个非 SDK 接口，大多数情况下都是在 Java 层通过反射获取 Class 对应的方法 Method，然后通过 Method.invoke 实现方法调用。`Class.getDeclaredMethod`最终都会进入一个 native 方法`getDeclaredMethodInternal`中，这个方法的实现如下：  
（以下源码全部来自 Android 11)

```
// art/runtime/native/java_lang_Class.cc
static jobject Class_getDeclaredMethodInternal(JNIEnv* env, jobject javaThis,
                                               jstring name, jobjectArray args) {
  ScopedFastNativeObjectAccess soa(env);
  StackHandleScope<1> hs(soa.Self());
  ......
  Handle result = hs.NewHandle(
      mirror::Class::GetDeclaredMethodInternal(
          soa.Self(),
          klass,
          soa.Decode(name),
          soa.Decode>(args),
          GetHiddenapiAccessContextFunction(soa.Self())));
  if (result == nullptr || ShouldDenyAccessToMember(result->GetArtMethod(), soa.Self())) {
    return nullptr;
  }
  return soa.AddLocalReference(result.Get());
} 
```

根据函数的名称，显而易见，这里通过`ShouldDenyAccessToMember()`这个函数来进行了调用限制。这个函数返回 false，则返回一个空的 jobject 对象，上层就获取不到此方法对应的 Method。

 

然后，在`mirror::Class::GetDeclaredMethodInternal`函数里，也有几处调用了`ShouldDenyAccessToMember`这个函数的判断，对返回结果进行限制：

```
template ObjPtr Class::GetDeclaredMethodInternal(
    Thread* self, ObjPtr klass, ObjPtr name, ObjPtr> args,
    const std::function& fn_get_access_context) {
    ......
 
  bool m_hidden = hiddenapi::ShouldDenyAccessToMember(&m, fn_get_access_context, access_method);
    if (!m_hidden && !m.IsSynthetic()) {
      // Non-hidden, virtual, non-synthetic. Best possible result, exit early.
      return Method::CreateFromArtMethod(self, &m);
    } else if (IsMethodPreferredOver(result, result_hidden, &m, m_hidden)) {
      // Remember as potential result.
      result = &m;
      result_hidden = m_hidden;
    }
  }
      ......
      DCHECK(!m.IsMiranda());  // Direct methods cannot be miranda methods.
      bool m_hidden = hiddenapi::ShouldDenyAccessToMember(&m, fn_get_access_context, access_method);
      if (!m_hidden && !m.IsSynthetic()) {
        // Non-hidden, direct, non-synthetic. Any virtual result could only have been
        // hidden, therefore this is the best possible match. Exit now.
        DCHECK((result == nullptr) || result_hidden);
        return Method::CreateFromArtMethod(self, &m);
      } else if (IsMethodPreferredOver(result, result_hidden, &m, m_hidden)) {
      }
    }
  }
 
  return result != nullptr ? Method::CreateFromArtMethod(self, result) : nullptr;
} 
```

从这几处的逻辑来看，虚拟机里是通过`hiddenapi::ShouldDenyAccessToMember`这个函数进行访问限制的。  
绕过的方法似乎只能是对这个函数的返回值进行篡改。目前主流的一些绕过方法确实也是这样做的。

主流的绕过方法
=======

目前，开源社区里已有好几个绕过非 SDK 调用限制的方法。主要的思路都是对`ShouldDenyAccessToMember`这个函数进行干涉，想办法修改其返回值为 false。

 

由于`ShouldDenyAccessToMember`函数是一个导出函数，打开 libart.so 可以看到其查找到对应的函数符号。很容易想到一个最简单的思路是，使用 native inline hook 技术，hook 住`ShouldDenyAccessToMember`函数，使其不调用原函数，直接返回 false。这样就简单绕过了调用限制。

 

另外，如果不适用使用 hook 技术，也可以修改这个函数返回值。只需要修改这个函数里调用的`Runtime::Current()->GetHiddenApiEnforcementPolicy()`接口的返回值，使其返回值为`EnforcementPolicy.kDisabled`，这样`ShouldDenyAccessToMember`函数就一定返回 false。  
因为`ShouldDenyAccessToMember`函数里有这样一行代码：

```
// art/runtime/hidden_api.h #ShouldDenyAccessToMember()
EnforcementPolicy policy = Runtime::Current()->GetHiddenApiEnforcementPolicy();
if (policy == EnforcementPolicy::kDisabled) {
  return false;
}

```

不过这种方案比较麻烦的点是，需要使用内存搜索的方式查找`Runtime`类里的`hidden_api_policy_`成员的偏移量，根据偏移量和 current_runtime 的地址计算出`hidden_api_policy_`的地址，再修改这个地址的内存值为`EnforcementPolicy.kDisabled`。

```
// art/runtime/runtime.h
// 这是一个inline函数，因此只能通过内存搜索才能获取到这个成员变量.
hiddenapi::EnforcementPolicy GetHiddenApiEnforcementPolicy() const {
   return hidden_api_policy_;
}
 
  // Whether access checks on hidden API should be performed.
 hiddenapi::EnforcementPolicy hidden_api_policy_;

```

在 Android9 中还可以使用双重反射 (元反射) 的方式来绕过。但在 android11 中元反射的方式已经被 google 屏蔽。  
另外，还有一个取巧的方式，可以绕过 google 的这种屏蔽。将双重反射调用的相关代码单独编译成一个 dex 文件，然后构造一个 DexFile 对象来执行双重反射的方法，并且在加载时，设置加载的 classloader 为空，从而绕过限制。具体代码如下：

```
DexFile dexFile = new DexFile(dexFile);
Class bootstrapClass = dexFile.loadClass(BootstrapClass.class.getCanonicalName(), null);
Method exemptAll = bootstrapClass.getDeclaredMethod("exemptAll");
return  (boolean) exemptAll.invoke(null);

```

这样为什么可行？  
跟踪源码可以知，`dexFile.loadClass()`时，最终会执行到`ClassLinker::RegisterDexFileLocked`中，这个函数调用了`InitializeDexFileDomain`：

```
// art/runtime/class_linker.cc
void ClassLinker::RegisterDexFileLocked(const DexFile& dex_file,
    ObjPtr dex_cache, ObjPtr class_loader) {
    ...
    // Let hiddenapi assign a domain to the newly registered dex file.
    hiddenapi::InitializeDexFileDomain(dex_file, class_loader);
    ...                                     
} 
```

`InitializeDexFileDomain`这个函数传进来的 classloader 为空时，以下的`DetermineDomainFromLocation`则必定返回`Domain::kPlatform`，则这个 dex_file 对应的`hiddenapi_domain_`成员对象的值就是`Domain::kPlatform`。

```
// // art/runtime/hidden_api.cc
  void InitializeDexFileDomain(const DexFile& dex_file, ObjPtr class_loader) {
    Domain dex_domain = DetermineDomainFromLocation(dex_file.GetLocation(), class_loader);
 
    if (IsDomainMoreTrustedThan(dex_domain, dex_file.GetHiddenapiDomain())) {
      dex_file.SetHiddenapiDomain(dex_domain);
    }
  }
 
static Domain DetermineDomainFromLocation(const std::string& dex_location,
                                            ObjPtr class_loader) {
 
    ...
 
    if (LocationIsOnSystemFramework(dex_location.c_str())) {
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

这个值又是怎样影响到`ShouldDenyAccessToMember()`函数的返回结果呢？  
回到`ShouldDenyAccessToMember()`函数，有这样的逻辑：

```
// art/runtime/hidden_api.h
 template inline bool ShouldDenyAccessToMember(T* member,
     const std::function& fn_get_access_context,
     AccessMethod access_method) {
     ...
    // Determine which domain the caller and callee belong to.
    const AccessContext caller_context = fn_get_access_context();
    const AccessContext callee_context(member->GetDeclaringClass());
 
    // Non-boot classpath callers should have exited early.
    DCHECK(!callee_context.IsApplicationDomain());
 
    // Check if the caller is always allowed to access members in the callee context.
    if (caller_context.CanAlwaysAccess(callee_context)) {
      return false;
    }
    ...
 
  }
 
    // Returns true if this domain is always allowed to access the domain of `callee`.
    bool CanAlwaysAccess(const AccessContext& callee) const {
      return IsDomainMoreTrustedThan(domain_, callee.domain_);
    } 
```

其中，caller_context 对应的就是调用方的 context, 其对应的 domain 就是 dex_file 的 domain，上面返回的是`kPlatform`, 而 callee_context 是根据 class 和 dex_file 来计算的，这个值也是`kPlatform`。因此上面代码中的`caller_context.CanAlwaysAccess(callee_context)`就返回了 true，因为`IsDomainMoreTrustedThan`函数仅仅就是比较两个 domain 值的大小：

```
// art/libartbase/base/hiddenapi_domain.h
enum class Domain : char {
    kCorePlatform = 0,
    kPlatform,
    kApplication,
  };
 
  inline bool IsDomainMoreTrustedThan(Domain domainA, Domain domainB) {
    return static_cast(domainA) <= static_cast(domainB);
  } 
```

这样，`ShouldDenyAccessToMember()`也就返回了 false，从而绕过访问限制。  
这种方案的本质是，本来 App 的 DexFile 对应的 domain 应该是 kApplication, 由于加载 class 时传进来了空的 classloader，导致 domain 值变成了 kPlatform，从而绕过了访问限制。

 

另外，google 工程师已经想到了方法来堵住这个方案，修复的代码已提交，不过，可能是由于测试用例没有全部通过，代码目前并没有合入到主分支中。  
相关 Patch 提交是：https://android-review.googlesource.com/c/platform/art/+/1668945

 

这个 Patch 的改法其实很简单，就是在`DetermineDomainFromLocation()`函数中，当 classloader 为空时，返回`Domain::kApplication`, 而不是`Domain::kPlatform`。

 

此外，还有一种比较巧妙的方法绕过方法，借助了 Java 操作内存的 Unsafe 类。  
实现源码为：https://github.com/LSPosed/AndroidHiddenApiBypass

 

这个方案的实现步骤为：

> 1.  根据类中相邻两个方法的差值，计算出 native 层 ArtMethod 数据结构的 size；
> 2.  构造一个跟 java.lang.Class 类成员完全一样的类，计算出 methods 成员在 native 层的偏移；
> 3.  根据这个偏移量，利用 Unsafe 算出类中类中 method 的数量，根据 methods 起始偏移，数量，以及每个 method 的 size, 遍历类中所有的方法；
> 4.  利用 MethodHandleImpl.java 将 ArtMethod(jmethodId) 指针转换为 mirror::Method(对应 Java 层的 java.lang.Method)。

 

下面这个是 java.lang.Class 的镜像类，使用镜像类是为了避免反射获取 java.lang.Class 里面的私有成员对象。

```
static final public class Class {
        private transient ClassLoader classLoader;
        private transient java.lang.Class componentType;
        private transient Object dexCache;
        private transient Object extData;
        private transient Object[] ifTable;
        private transient String name;
        private transient java.lang.Class superClass;
        private transient Object vtable;
        private transient long iFields;
        private transient long methods;
        private transient long sFields;
        private transient int accessFlags;
        private transient int classFlags;
        private transient int classSize;
        private transient int clinitThreadId;
        ...
        ...
  }

```

下面代码是源码中将 MethodHandle 的 artFieldOrMethod 成员 (对应 native 层 ArtMethod 指针) 转换为 mirror::Method(对应 Java 层 java.lang.Method)的流程：

```
// art/runtime/native/java_lang_invoke_MethodHandleImpl.cc
static jobject MethodHandleImpl_getMemberInternal(JNIEnv* env, jobject thiz) {
  ScopedObjectAccess soa(env);
  StackHandleScope<2> hs(soa.Self());
  Handle handle = hs.NewHandle(
      soa.Decode(thiz));
 
  const mirror::MethodHandle::Kind handle_kind = handle->GetHandleKind();
 
  MutableHandle h_object(hs.NewHandle(nullptr));
  if (handle_kind >= mirror::MethodHandle::kFirstAccessorKind) {
    ArtField* const field = handle->GetTargetField();
    h_object.Assign(mirror::Field::CreateFromArtField(
        soa.Self(), field, /* force_resolve= */ false));
  } else {
    ArtMethod* const method = handle->GetTargetMethod();
    if (method->IsConstructor()) {
      h_object.Assign(mirror::Constructor::CreateFromArtMethod(
          soa.Self(), method));
    } else {
      h_object.Assign(mirror::Method::CreateFromArtMethod(
          soa.Self(), method));
    }
  }
 
  return soa.AddLocalReference(h_object.Get());
} 
```

这个函数没有调用`ShouldDenyAccessToMember`函数，因此，此方法可行。

新的绕过方法
======

调用隐藏接口，除了使用上面提到的`Class.getDeclaredMethod`这个方法以外，也可以在 native 层通过`JNIEnv->GetMethodId()`来获取方法对应的`jmethodId`，然后调用`JNIEnv->CallObjectMethod()`来执行对应的 method：

```
jclass context_class = env->FindClass("android/content/Context");
jmethodID get_content_resolver_mid = env->GetMethodID(context_class, "getContentResolver", "()Landroid/content/ContentResolver;");
jobject content_resolver_obj = env->CallObjectMethod(context, get_content_resolver_mid);

```

或许我们可以从 native 层可以找到一些突破口。  
先看看`JNIEnv->GetMethodId()`的源码：

```
// art/runtime/jni/jni_internal.cc
 static jmethodID GetMethodID(JNIEnv* env, jclass java_class, const char* name, const char* sig) {
      ScopedObjectAccess soa(env);
      return FindMethodID(soa, java_class, name, sig, false);
}
 
template static jmethodID FindMethodID(ScopedObjectAccess& soa, jclass jni_class,
                                const char* name, const char* sig, bool is_static)
      REQUIRES_SHARED(Locks::mutator_lock_) {
    return jni::EncodeArtMethod(FindMethodJNI(soa, jni_class, name, sig, is_static));
}
 
ArtMethod* FindMethodJNI(const ScopedObjectAccess& soa,
                           jclass jni_class,
                           const char* name,
                           const char* sig,
                           bool is_static) {
    ObjPtr c = EnsureInitialized(soa.Self(), soa.Decode(jni_class));
    ...
    ArtMethod* method = nullptr;
    auto pointer_size = Runtime::Current()->GetClassLinker()->GetImagePointerSize();
    if (c->IsInterface()) {
      method = c->FindInterfaceMethod(name, sig, pointer_size);
    } else {
      method = c->FindClassMethod(name, sig, pointer_size);
    }
    if (method != nullptr && ShouldDenyAccessToMember(method, soa.Self())) {
      method = nullptr;
    }
    ...
    return method;
  } 
```

在`FindMethodJNI`函数中，又看到熟悉的:`ShouldDenyAccessToMember`函数。也就是说，在这个流程里，也是通过`ShouldDenyAccessToMember`函数来限制 App 获取非 SDK 方法对应的 ArtMethod 对象。  
再看看`c->FindClassMethod(name, sig, pointer_size);`这个调用流程里有没有类似的限制：

```
// art/runtime/mirror/class.cc
 ArtMethod* FindClassMethod(std::string_view name,
                                    std::string_view signature,
                                    PointerSize pointer_size) {
    return FindClassMethodWithSignature(this, name, signature, pointer_size);
 
 
template static inline ArtMethod* FindClassMethodWithSignature(ObjPtr this_klass,
                                                      std::string_view name,
                                                      const SignatureType& signature,
                                                      PointerSize pointer_size)
    REQUIRES_SHARED(Locks::mutator_lock_) {
  for (ArtMethod& method : this_klass->GetDeclaredMethodsSlice(pointer_size)) {
    ArtMethod* np_method = method.GetInterfaceMethodIfProxy(pointer_size);
    if (np_method->GetName() == name && np_method->GetSignature() == signature) {
      return &method;
    }
  }
 
  ObjPtr klass = this_klass->GetSuperClass();
  ArtMethod* uninherited_method = nullptr;
  for (; klass != nullptr; klass = klass->GetSuperClass()) {
    DCHECK(!klass->IsProxyClass());
    for (ArtMethod& method : klass->GetDeclaredMethodsSlice(pointer_size)) {
      if (method.GetName() == name && method.GetSignature() == signature) {
        if (IsInheritedMethod(this_klass, klass, method)) {
          return &method;
        }
        uninherited_method = &method;
        break;
      }
    }
    if (uninherited_method != nullptr) {
      break;
    }
  }
 
  ObjPtr end_klass = klass;
  DCHECK_EQ(uninherited_method != nullptr, end_klass != nullptr);
  klass = this_klass;
  ...
  for (; klass != end_klass; klass = klass->GetSuperClass()) {
    DCHECK(!klass->IsProxyClass());
    for (ArtMethod& method : klass->GetCopiedMethodsSlice(pointer_size)) {
      if (method.GetName() == name && method.GetSignature() == signature) {
        return &method;  // No further check needed, copied methods are inherited by definition.
      }
    }
  }
  return uninherited_method;  // Return the `uninherited_method` if any.
}
} 
```

显然，`FindClassMethod`这个流程里并没有调用`ShouldDenyAccessToMember`函数，因此没有访问限制。  
这个函数的主要流程是从类对象`mirror::Class`以及其父类中遍历所有 ArtMethod 指针，查找与目标 name 和 signture 匹配的 ArtMethod 指针。

柳暗花明
====

既然`mirror::Class::FindClassMethod`这个函数中没有对非 SDK 接口进行限制的逻辑，那我们何不直接调用这个函数呢?  
若要调用一个非公开的 native 函数，需满足以下两个条件：

> 1.  可以获取到这个函数的地址；
> 2.  能够构造出调用函数需要传递的各个参数；

 

先看第一个条件，`FindClassMethod`是一个非 inline 函数，并且在系统的 libart.so 文件符号表中，可以查找到这个函数对应符号为：

> _ZN3art6mirror5Class15FindClassMethodENSt3__117basic_string_viewIcNS2_11char_traitsIcEEEES6_NS_11PointerSizeE

 

因此，可以使用 linux 动态库 dl 接口轻松获取到这个函数的地址，代码如下：

```
const char *func_name = "_ZN3art6mirror5Class15FindClassMethodENSt3__117basic_string_viewIcNS2_11char_traitsIcEEEES6_NS_11PointerSizeE";
void *art_so_handle = dlopen("libart.so", RTLD_NOW);
void* address = dlsym(art_so_handle, func_name);
auto findClassMethod = reinterpret_cast(address); 
```

不过，这里需要注意的是，从 Android7.0 开始，Android 系统限制了 App 使用 dlopen 打开系统动态库。不过，笔者之前开发了一个库可以轻松绕过这种限制。  
源码：[bypass_dlfunctions](https://github.com/WindySha/bypass_dlfunctions)  
实现原理：[另一种绕过 Android 系统库访问限制的方法](https://windysha.github.io/2021/05/26/%E5%8F%A6%E4%B8%80%E7%A7%8D%E7%BB%95%E8%BF%87Android%E7%B3%BB%E7%BB%9F%E5%BA%93%E8%AE%BF%E9%97%AE%E9%99%90%E5%88%B6%E7%9A%84%E6%96%B9%E6%B3%95/)

 

再看第二个条件，这个函数需要传递四个参数，第一个是这个函数所在类`mirror::Class`的指针, 第二个参数是方法的 name，第三个参数是方法的签名，第四个参数是指针的大小。显然，除了第一个对象，其他几个都是已知的。  
难点是第一个参数，回到上面，看看`FindMethodJNI`这个函数是如何获取到`mirror::Class`指针：

```
ObjPtr c = soa.Decode(jni_class); 
```

`Decode`函数源码如下：

```
// art/runtime/scoped_thread_state_change-inl.h
template inline ObjPtr ScopedObjectAccessAlreadyRunnable::Decode(jobject obj) const {
  Locks::mutator_lock_->AssertSharedHeld(Self());
  DCHECK(IsRunnable());  // Don't work with raw objects in non-runnable states.
  return ObjPtr::DownCast(Self()->DecodeJObject(obj));
} 
```

最终调用到了`Thread::DecodeJObject`函数，这个函数的逻辑比较清晰，就是从 jobject 对应的引用类型的 table 中查找对应的`mirror::Object`对象:

```
// art/runtime/thread.cc
ObjPtr Thread::DecodeJObject(jobject obj) const {
  ...
  IndirectRef ref = reinterpret_cast(obj);
  IndirectRefKind kind = IndirectReferenceTable::GetIndirectRefKind(ref);
  ObjPtr result;
  bool expect_null = false;
  if (kind == kLocal) {
    IndirectReferenceTable& locals = tlsPtr_.jni_env->locals_;
    result = locals.Get(ref);
  } else if (kind == kJniTransitionOrInvalid) {
    result = reinterpret_cast*>(obj)->AsMirrorPtr();
    VerifyObject(result);
  } else if (kind == kGlobal) {
    result = tlsPtr_.jni_env->vm_->DecodeGlobal(ref);
  } else {
    result = tlsPtr_.jni_env->vm_->DecodeWeakGlobal(const_cast(this), ref);
    if (Runtime::Current()->IsClearedJniWeakGlobal(result)) {
      expect_null = true;
      result = nullptr;
    }
  }
  ...
  return result;
} 
```

查看 libart.so 的符号表，发现`DecodeJObject`这个函数也是导出函数，对应的符号为：

> _ZNK3art6Thread13DecodeJObjectEP8_jobject

 

这个函数的第一个参数是当前线程 native Thread 的指针。这个指针可以有两种方法获取到，第一种方式是反射获取 Thread.java 的 nativePeer 成员变量，这个值里存的就是 native Thread 的指针，反射过程代码如下在:

```
jclass thread_class = env->FindClass("java/lang/Thread");
jmethodID currentThread_id =
    env->GetStaticMethodID(thread_class, "currentThread", "()Ljava/lang/Thread;");
jobject current_thread = env->CallStaticObjectMethod(thread_class, currentThread_id);
jfieldID nativePeer_id = env->GetFieldID(thread_class, "nativePeer", "J");
jlong native_thread = env->GetLongField(current_thread, nativePeer_id);

```

使用这种方式需要反射获取 Thread.java 中的私有成员变量 nativePeer，目前这个变量被列入了 greylist 中，greylist 中的 api 可以被调用，但在未来更高的 TargetSDK 版本可能会将其列入黑名单中。

```
Accessing hidden field Ljava/lang/Thread;->nativePeer:J (greylist, JNI, allowed)

```

为此，我们可以使用另外一种方法获取 native Thread 指针。

 

在 JNIEnv 结构体中，第一个位置是虚函数表，第二个位置就是 native Thread 指针。每个线程中 JNIEnv 都是已知的。因此，可以通过下面方法简单得到 Thread 指针，和上面的方法得到的结果刚好一致。

```
auto* fakeEnv = reinterpret_cast(jni_env);
void* native_thread = fakeEnv->self_;
 
struct FakeJNIEnv {
    void* vtb_;
    void *const self_;    // Link to Thread::Current().
    void *const vm_;      // The invocation interface JavaVM.
}; 
```

有了 native Thread 指针，调用`Thread::DecodeJObject`函数就能获取到当前 class 对应的 mirror::Class 指针, 再通过这个 mirror::Class 指针，方法的名称以及签名，调用`mirror::Class::FindClassMethod`函数获取到方法对应的 ArtMethod 指针，这个指针就是此方法的 jmethodId。调用 CallObjectMethod 并传入这个 jmethodId 便可实现非 SDK 接口的调用。至此，成功绕过 ART 虚拟机对非 SDK 接口的调用限制。

完整流程
====

完整代码流程如下：

```
struct FakeJNIEnv {
    void* vtb_;
    void *const self_;    // Link to Thread::Current().
    void *const vm_;      // The invocation interface JavaVM.
};
 
void bypassHiddenApi(JNIEnv *env) {
    auto* fakeEnv = reinterpret_cast(env);
    void* current_thread = fakeEnv->self_;
 
    const char *findClassMethod_func_name = "_ZN3art6mirror5Class15FindClassMethodENSt3__117basic_string_viewIcNS2_11char_traitsIcEEEES6_NS_11PointerSizeE";
    void *art_so_handle = bp_dlopen("libart.so", RTLD_NOW);
    void* address = bp_dlsym(art_so_handle, findClassMethod_func_name);
    auto findClassMethod = reinterpret_cast(address);
 
    const char *decodeJObject_sig = "_ZNK3art6Thread13DecodeJObjectEP8_jobject";
    void *art_so_address = bp_dlopen("libart.so", RTLD_NOW);
    auto decodeJObject_func = reinterpret_cast(bp_dlsym(art_so_address, decodeJObject_sig));
 
    const char *VMRuntime_class_name = "dalvik/system/VMRuntime";
    jclass vmRumtime_class = env->FindClass(VMRuntime_class_name);
    void *VMRuntime_mirror_class_ObjPtr = decodeJObject_func(current_thread, vmRumtime_class);
 
    size_t pointer_size = sizeof(void*);
    void *getRuntime_art_method = findClassMethod(VMRuntime_mirror_class_ObjPtr,
                                                             "getRuntime",
                                                             "()Ldalvik/system/VMRuntime;",
                                                             pointer_size);
    jobject vmRuntime_instance = env->CallStaticObjectMethod(vmRumtime_class, (jmethodID)getRuntime_art_method);
 
    const char *target_char = "L";
    jstring mystring = env->NewStringUTF(target_char);
    jclass cls = env->FindClass("java/lang/String");
    jobjectArray jarray = env->NewObjectArray(1, cls, nullptr);
    env->SetObjectArrayElement(jarray, 0, mystring);
 
    void *setHiddenApiExemptions_art_method = findClassMethod(VMRuntime_mirror_class_ObjPtr,
                                                                         "setHiddenApiExemptions",
                                                                         "([Ljava/lang/String;)V",
                                                                         pointer_size);
    env->CallVoidMethod(vmRuntime_instance, (jmethodID)setHiddenApiExemptions_art_method, jarray);
} 
```

上面的实现中，最终是反射调用了两个隐藏接口：

```
VMRuntime runtime = VMRuntime.*getRuntime*();
runtime.setHiddenApiExemptions(new String[]{"L"});

```

`ShouldDenyAccessToMember`的代码实现中，检查了 member 对应的 class 名称前缀是否包含在 GetHiddenApiExemptions 返回的 vector 中。所以的 class 名称都以 “L” 开头，设置 "L" 到这个 HiddenApiExemptions 中，`ShouldDenyAccessToMember`函数就一直返回 false。

源码和用法
=====

完整的源码发布到 github：[bypassHiddenApiRestriction](https://github.com/WindySha/bypassHiddenApiRestriction)

 

使用方法：build.gradle 中添加以下依赖：

```
allprojects {
    repositories {
        mavenCentral()
    }
}

```

```
dependencies {
    implementation 'io.github.windysha:bypassHiddenApiRestriction:1.0.2'
}

```

App 入口代码添加以下代码即可：

```
import com.wind.hiddenapi.bypass.HiddenApiBypass
 
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
    HiddenApiBypass.startBypass();
}

```

[[注意] 招人！base 上海，课程运营、市场多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)

[#基础理论](forum-161-1-117.htm) [#系统相关](forum-161-1-126.htm) [#源码分析](forum-161-1-127.htm)
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.canyie.top](https://blog.canyie.top/2020/06/10/hiddenapi-restriction-policy-on-android-r/)

2018 年发布的 Android 9 中引入了[对隐藏 API 的限制](https://developer.android.google.cn/distribute/best-practices/develop/restrictions-non-sdk-interfaces)，这对整个 Android 生态来说当然是一件好事，但也严重限制了以往我们通过反射等手段实现的 “黑科技”（如插件化等），所以开发者们纷纷寻找手段绕过这个限制，比如我曾经提出了两个绕过方法，其中一个便是几乎完美的[双重反射（即 “元反射”，现在来看叫“套娃反射” 比较好）](https://zhuanlan.zhihu.com/p/59455212)；而在即将发布的 Android R 中把这个方法封杀了~（谷歌：禁止套娃！）~，因此我重新研究了 Android R 中的限制策略。

[](#上有政策 "上有政策")上有政策
--------------------

常言道，知己知彼，百战百胜。要想破解这个限制，就必须去搞懂系统是怎么施加的限制；ok，废话不多说，let’s go！  
以我们在 Java 层通过反射获取一个 Method 为例，`Class.getMethod/getDeclaredMethod`最终都会进入一个 native 方法`getDeclaredMethodInternal`，这个方法的实现如下：

```
static jobject Class_getDeclaredMethodInternal(JNIEnv* env, jobject javaThis, jstring name, jobjectArray args) {
  
  Handle<mirror::Method> result = hs.NewHandle(
      mirror::Class::GetDeclaredMethodInternal<kRuntimePointerSize>(
          soa.Self(),
          klass,
          soa.Decode<mirror::String>(name),
          soa.Decode<mirror::ObjectArray<mirror::Class>>(args),
          GetHiddenapiAccessContextFunction(soa.Self())));
  if (result == nullptr || ShouldDenyAccessToMember(result->GetArtMethod(), soa.Self())) {
    return nullptr;
  }
  return soa.AddLocalReference<jobject>(result.Get());
}
```

我们可以发现，如果 ShouldDenyAccessToMember 返回 true，那么就会返回 null，上层就会抛出方法找不到的异常。这里和 Android P 没什么不同，只是把 ShouldBlockAccessToMember 改了个名而已。  
ShouldDenyAccessToMember 会调用到`hiddenapi::ShouldDenyAccessToMember`，该函数是这样实现的：

```
template<typename T>
inline bool ShouldDenyAccessToMember(T* member,
                                     const std::function<AccessContext()>& fn_get_access_context,
                                     AccessMethod access_method)
    REQUIRES_SHARED(Locks::mutator_lock_) {

  const uint32_t runtime_flags = GetRuntimeFlags(member);

  
  if ((runtime_flags & kAccPublicApi) != 0) {
    return false;
  }

  
  
  const AccessContext caller_context = fn_get_access_context();
  const AccessContext callee_context(member->GetDeclaringClass());

  
  if (caller_context.CanAlwaysAccess(callee_context)) {
    return false;
  }

  
  switch (caller_context.GetDomain()) {
    case Domain::kApplication: {
      DCHECK(!callee_context.IsApplicationDomain());

      
      EnforcementPolicy policy = Runtime::Current()->GetHiddenApiEnforcementPolicy();
      if (policy == EnforcementPolicy::kDisabled) {
        return false;
      }

      
      return detail::ShouldDenyAccessToMemberImpl(member, api_list, access_method);
    }
    
  }
```

这个函数还是比较长的，我们一步步分析。

1.  判断目标成员是否是公开 API，如果是那么直接通过，具体可以看见是通过`GetRuntimeFlags`获取，这个 flags 其实是储存在该成员的`access_flags_`中（其实这里不太完全，暂且这么认为吧）
2.  获取调用者的 Domain，判断是否可信，如果可信直接通过
3.  以上条件都不满足，根据 Domain 走不同的实现，我们应用的代码对应的 Domain 是 kApplication，主要看第一个 case 就行。

第二步中获取调用者的函数中核心部分如下：

```
bool VisitFrame() override REQUIRES_SHARED(Locks::mutator_lock_) {
    ArtMethod *m = GetMethod();
    if (m == nullptr) {
        
        caller = nullptr;
        return false;
    } else if (m->IsRuntimeMethod()) {
        
        return true;
    }

    ObjPtr<mirror::Class> declaring_class = m->GetDeclaringClass();
    if (declaring_class->IsBootStrapClassLoaded()) {
        
        if (declaring_class->IsClassClass()) {
            return true;
        }

        
        ObjPtr<mirror::Class> lookup_class = GetClassRoot<mirror::MethodHandlesLookup>();
        if ((declaring_class == lookup_class || declaring_class->IsInSamePackage(lookup_class))
                && !m->IsClassInitializer()) {
            return true;
        }

        
        
        ObjPtr<mirror::Class> proxy_class = GetClassRoot<mirror::Proxy>();
        if (declaring_class->IsInSamePackage(proxy_class) && declaring_class != proxy_class) {
            if (Runtime::Current()->isChangeEnabled(kPreventMetaReflectionBlacklistAccess)) {
                return true;
            }
        }
    }

    caller = m;
    return false;
}
```

根据 caller 选择 Domain：

```
static Domain ComputeDomain(ObjPtr<mirror::ClassLoader> class_loader, const DexFile* dex_file) {
    if (dex_file == nullptr) {
        
        return ComputeDomain( class_loader.IsNull());
    }
    
    return dex_file->GetHiddenapiDomain();
}

static Domain ComputeDomain(ObjPtr<mirror::Class> klass, const DexFile* dex_file)
    REQUIRES_SHARED(Locks::mutator_lock_) {

    Domain domain = ComputeDomain(klass->GetClassLoader(), dex_file);

    if (domain == Domain::kApplication && klass->ShouldSkipHiddenApiChecks() && Runtime::Current()->IsJavaDebuggable()) {
        
        domain = ComputeDomain( true);
    }

    return domain;
}
```

dex_file 的 Domain 在第一次加载 Class 时被初始化（注：Android 8 时就已经不允许一个 DexFile 同时加载多个 ClassLoader 了）：

```
static Domain DetermineDomainFromLocation(const std::string& dex_location,
                                          ObjPtr<mirror::ClassLoader> class_loader) {

    if (ArtModuleRootDistinctFromAndroidRoot()) {
        
        if (LocationIsOnArtModule(dex_location.c_str()) 
            || LocationIsOnConscryptModule(dex_location.c_str()) 
            || LocationIsOnI18nModule(dex_location.c_str())) {
            return Domain::kCorePlatform;
        }

        
        if (LocationIsOnApex(dex_location.c_str())) {
            return Domain::kPlatform;
        }
    }

    
    if (LocationIsOnSystemFramework(dex_location.c_str())) {
        return Domain::kPlatform;
    } 

    
    if (LocationIsOnSystemExtFramework(dex_location.c_str())) {
        return Domain::kPlatform;
    }

    
    if (class_loader.IsNull()) {
        LOG(WARNING) << "DexFile " << dex_location << " is in boot class path but is not in a known location";
        return Domain::kPlatform;
    }

  return Domain::kApplication;
}
```

值得注意的是 Android Q 中细分出了三个 Domain：kCorePlatform、kPlatform、kApplication，同时对 kPlatform 访问 kCorePlatform 的情况也做出了一定限制：

```
case Domain::kPlatform: {
  DCHECK(callee_context.GetDomain() == Domain::kCorePlatform);

  
  if ((runtime_flags & kAccCorePlatformApi) != 0) {
    return false;
  }

  
  
  EnforcementPolicy policy = Runtime::Current()->GetCorePlatformApiEnforcementPolicy();
  if (policy == EnforcementPolicy::kDisabled) {
    return false;
  }

  return detail::HandleCorePlatformApiViolation(member, caller_context, access_method, policy);
}
```

Q 中默认是关闭这个功能，R 中不知道；这部分简单了解一下就好，主要还是关注 kApplication 即应用代码访问系统 API 的情况。

```
template<typename T>
bool ShouldDenyAccessToMemberImpl(T* member, ApiList api_list, AccessMethod access_method) {
  Runtime* runtime = Runtime::Current();

  EnforcementPolicy hiddenApiPolicy = runtime->GetHiddenApiEnforcementPolicy();

  MemberSignature member_signature(member);

  
  if (member_signature.DoesPrefixMatchAny(runtime->GetHiddenApiExemptions())) {
    MaybeUpdateAccessFlags(runtime, member, kAccPublicApi);
    return false;
  }
  
  bool deny_access = false;
  
  EnforcementPolicy testApiPolicy = runtime->GetTestApiEnforcementPolicy();
  
  if (hiddenApiPolicy == EnforcementPolicy::kEnabled) {
    if (testApiPolicy == EnforcementPolicy::kDisabled && api_list.IsTestApi()) {
      
      deny_access = false;
    } else {
      
      
      switch (api_list.GetMaxAllowedSdkVersion()) {
        case SdkVersion::kP:
          deny_access = runtime->isChangeEnabled(kHideMaxtargetsdkPHiddenApis);
          break;
        case SdkVersion::kQ:
          deny_access = runtime->isChangeEnabled(kHideMaxtargetsdkQHiddenApis);
          break;
        default:
          deny_access = IsSdkVersionSetAndMoreThan(runtime->GetTargetSdkVersion(), api_list.GetMaxAllowedSdkVersion());
      }
    }
  }

  if (access_method != AccessMethod::kNone) {
    

    
    
    if (!deny_access) {
      MaybeUpdateAccessFlags(runtime, member, kAccPublicApi);
    }
  }

  return deny_access;
}
```

OK，大致流程都已经清晰，梳理一下：

1.  如果目标成员是公开 API，直接通过
2.  获取调用者的 AccessContext，如果可信那么通过
3.  如果访问检查被完全关闭，那么通过
4.  判断目标成员是否在豁免名单里，如果在那么通过
5.  `hiddenApiPolicy == EnforcementPolicy::kJustWarn`，不会对`deny_access`赋值，警告后通过
6.  以上条件都不满足，根据 targetSdkVersion 决定是否需要拒绝访问

[](#下有对策 "下有对策")下有对策
--------------------

把系统的策略搞清楚了，接下来绕过就容易了。  
我们一步一步来：  
首先如果这个 member 的 access flags 里有`kAccPublicApi`，那么系统就认为这是一个公开 API，就不会进行任何限制了，然而我们如果要对 access_flags 动手脚，必须先拿到这个 member，然而系统就是限制了我们去拿这个 member 的过程，死循环了，放弃；

然后，如果调用者是可信的，那么也会通过，有这些情况：

1.  调用者所在的类在系统路径下
2.  调用者所在的类对应的类加载器（ClassLoader）为 null（即被 BootClassLoader 加载的类）
3.  debug 版并且主动设置了跳过限制，对应接口为 VMDebug.allowHiddenApiReflectionFrom(Class<?> klass)

我们有两种方法，一种是直接把自己的类变成系统类，另一种是通过系统 API 发起调用；第二种对应的实现方案就是套娃反射，然而现在已经被谷歌封掉了，我也找不到其他 API，就只剩下把自己的类变成系统类了。  
首先排除只能在 debug mode 工作的 3；而 1 也没法满足，主动修改`dex_file->hiddenapi_domain_`需要先拿到这个 dex_file 指针，而不使用 ART 内部接口的情况下是不方便拿到的，而且修改`hiddenapi_domain_`需要提前知道这个成员变量对应的偏移，先放弃；2 我觉得是这三种方法里最好的，Class 对象直接在 java 层就能拿到，改也可以直接在 java 层改，类似这样：

```
Field classLoaderField = Class.class.getDeclaredField("classLoader");
classLoaderField.setAccessible(true);
classLoaderField.set(MyClass.class, null);
```

然后就可以用这个 MyClass 进行反射。  
问题在于这个 classLoader 变量也是隐藏 API，当然你也可以用 Unsafe 等方案，但终究不保险；我们最好使用公开 API，那有这样的公开 API 吗？  
有！  
dalvik.system.DexFile 中有这样一个[方法](https://developer.android.google.cn/reference/dalvik/system/DexFile#loadClass(java.lang.String,%20java.lang.ClassLoader))：

```
public Class loadClass (String name, ClassLoader loader)
```

第二个参数就是指定该 Class 的 ClassLoader，设置成 null 即可。  
但使用这个方法，你需要自己额外准备一个 dex 文件，加载获得 DexFile 对象后再调用 loadClass，略显繁琐，实际使用的话可以弄个 gradle 脚本，拦截 mergeDexDebug/Release 拿到 dex；另一个问题是 DexFile 是 Deprecated，虽然现在用没问题，但保不准哪天就被谷歌给删了。  
提到这个方法，有的小伙伴可能会想起来，我们的目标是获得一个行为受我们控制且 class_loader==null 的类，[java.lang.reflect.Proxy](https://developer.android.google.cn/reference/java/lang/reflect/Proxy) 中也有类似的接口，那么我们可以用动态代理吗？  
事实证明是不行的，你确实可以获得一个 class_loader==null 的代理类，然而动态代理只是一个桥梁，最后执行动作是用的`InvocationHandler`对象，最终栈回溯的结果取决于这个 InvocationHandler 对应的 Class。  
（注：这就是我当时提出的另一个绕过方法，当时在我自己的贴吧里发了个贴，之后被百度删了，申诉四次没过，呵呵）

似乎从这里入手不太好的样子，我们继续。  
第三步和第五步中，都需要通过`runtime->GetHiddenApiEnforcementPolicy()`的返回值做判断，查看实现可以看见其实就是返回了一个成员变量`hidden_api_policy_`，那我们改这个成员变量不就行了？然而打开 runtime.h 你就会发现这个对象太大了，什么东西都往里面扔，改里面的值存在一定风险；搜索一下你会发现 art 通过 RAII 机制封装了一个`ScopedHiddenApiEnforcementPolicySetting`出来，然而这个类的构造函数并没有导出，我们无法直接调用，先放着。

第四步中提到 art 内部有一个豁免名单，而这个名单同样保存在 runtime 中，和上面的情况一样；不过这个 API 暴露到了 java 层，java 层中有对应的接口（`VMRuntime.setHiddenApiExemptions(String[] exemptions)`），不过这个接口在黑名单内，也无法直接调用。

OK，研究完了，得出结论：没有较为方便通用稳定的方法……  
才怪。  
**我们的最终目标是成功拿到 member，反映到代码里就是 ShouldDenyAccessToMember 返回 false，为此我们可以通过各种方式干扰这个函数的执行，但最终都还是为了让它返回 false。既然我们只是为了这个，那么其实可以用更直观的方式：native hook。**  
查看 art 代码可以发现无论是 P 上的`ShouldBlockAccessToMember`还是 Q 上的`ShouldDenyAccessToMember`，执行流程中都会调用某个关键的且符号已被导出的函数，P 上是`GetMemberActionImpl`，Q 上是`ShouldDenyAccessToMemberImpl`，直接 hook 住，修改它的返回值就 ok 了，目前 Pine 采用了这种方式，具体实现可见[这个 commit](https://github.com/canyie/pine/commit/6d8a578576e15d288e3a6e7cc5179584e4651251)。

[](#总结 "总结")总结
--------------

嗯，大概就是这样啦~  
又放一下咱的 [QQ 群：949888394](https://shang.qq.com/wpa/qunwpa?idkey=25549719b948d2aaeb9e579955e39d71768111844b370fcb824d43b9b20e1c04)~  
感谢你能看到最后~
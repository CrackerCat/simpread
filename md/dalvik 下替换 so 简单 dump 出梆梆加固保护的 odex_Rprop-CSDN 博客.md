> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/rrrfff/article/details/45112283)

*   由于保护技术更迭迅速，不保证本文方法适用于后续或者其它版本的梆梆加固，需要读者自行测试。

        梆梆加固后的 apk，里面的 classes.dex 只是个外壳，负责加载 libDexHelper.so，而真正的 dex 被加密放在了 \ assets\classes0.jar，这不是常规的 jar 文件无法直接解压，我们的目标就是从内存中 dump 出解密后的 classes0.jar/.dex(并进行适当修复)。

        内存 dump 的方案有很多, 比如直接 dd、DexHunter 等，但都有一些缺点，dd 需要内存地址和大小，这就要求进行暴力搜索，但是内存完全可能被处理过，dump 时机很难掌握，于是转到动态调试期望在关键点下断，但由于反调试对抗的存在又会有一堆麻烦；DexHunter 很强大，但它要求定制系统，这对逆向工程来说当然不是事，只是同样有针对 DexHunter 类似原理的对抗，比如检测不安全 ROM，等等这些都会带来不必要麻烦，本文提供的方法似乎直接跳过了这些坑。

        其实，无论是 DexHunter 还是本文提供的方案，最关键的不过是取得代码执行先机，因为代码一但被安全的注入，在壳启动前执行我们的代码，我们就能完成很多有意思的事情了。怎么实现呢？我们知道 libDexHelper.so 是整个壳的核心，修改 libDexHelper.so？万一有自校验咋办，这里就有了两种替换方案，第一种查看依赖较少的 so，将该依赖的 so 换成我们的，比如 liblog.so，但是这要求我们实现和 liblog.so 相应的接口，否则 dlopen 会失败，第二种，也是本文采用的方案，我们直接把它替换了不就好了嘛？  

        首先，把原来的 libDexHelper.so 改成 libDexHelper2.so, 然后我们需要写一个 libDexHelper.so，只需实现 JNI_OnLoad 即可, 在里面加载 libDexHelper2.so 并显式调用 JNI_OnLoad 即可 (这里竟然没有检测文件名，呵呵，不过检测也没多大意义，容易 bypass。另外由于我用的 x86 机，梆梆在. cache 目录下生成对应的 x86 so 才是目标):

```
JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM *jvm, void *unused)
{
	#define SO "/data/data/blog.csdn.net.rrrfff/.cache/libDexHelper-x862.so"
	LOGI("Trying to load lib " SO " 0x0");
	void *hk = dlopen(SO, RTLD_GLOBAL | RTLD_NOW);
	LOGI("Added shared lib " SO " %p %s", hk, hk == NULL ? dlerror() : "");
 
	void *ld = dlsym(hk, __FUNCTION__);
	LOGI("JNI_OnLoad at %p", ld);
	reinterpret_cast<__func_type(JNI_OnLoad)>(ld)(jvm, unused);
}
```

adb push 我们的 so 到相应目录，am start 启动目标 app，运行一切正常，代码被顺利注入，接着我们切入重点，把 dex 相关的 api 统统 hook 一遍看看是咋回事，这里我把 dexFile * 全家桶都给 hook 了，包括 dvmDexFileOpenFromFd，dvmDexFileFree，dexFileParse，dvmDexChangeDex1，dvmDexChangeDex2。

通过被 hook 方法的执行情况，我们大概总结下梆梆加固还原 dex 的大概流程 (dalvik 系统):

> 1.  open 读取 \assets\classes0.jar 并解密出真正的 jar，然后解压到. cache 目录，但此时的 classes0.dex 仍旧是加密的，这步仅. cache 目录下没有对应. dex 时会发生。
> 2.  open 读取 classes0.dex 并解密出真正的 dex，其实是 odex，看文件头可以知道。
> 3.  通过 dvmDexFileOpenFromFd 读取 dex 文件，这里的 fd 就是上一步的 fd，进行内存映射。
> 4.  系统调用 dexFileParse 对映射好的 dex 内存进行解析并返回 DexFile 结构，这里是 dump 的好时机。
> 5.  壳调用 dvmDexChangeDex2 对已解析好的 dex 内存区进行混淆，写入脏数据，所以我们在程序跑起来后再 dump 出来的东西是有噪声的。

        知道了这些之后我们就能愉快的干活了，编写代码在 dexFileParse 时完成 dump(据说梆梆把一堆 read,write,mmap 等 libc 函数 hook 了, 防止读取相关 dex 的区域, 这里绕了个弯，先用一般人不会用到的 memmove 备份内存，然后跳过 dex 头写到文件，其实似乎多此一举):

```
	static info<DvmDex *(*)(const uint8_t* data, size_t length, int flags)> *p0 = NULL;
	p0 = p0->hook(dlsym(dvm, "_Z12dexFileParsePKhji"),
				  [](const uint8_t* data, size_t length, int flags)->DvmDex * {
		LOGI("dexFileParse hit with %p %u %d", data, length, flags);
		if (length > 100000) {
			int  dexv = ::open("/data/data/blog.csdn.net.rrrfff/cache/core.odex", O_CREAT | O_TRUNC | O_WRONLY);
			auto dexp = reinterpret_cast<uint8_t *>(::malloc(length));
			::memmove(dexp, data, length);
			::write(dexv, "bmp", 3);
			::write(dexv, dexp + 3, dexl - 3); // bypass odex header
			::free(dexp);
			::close(dexv);
		} //if
		auto pDvmDex = p4->invoke()(data, length, flags);
		return pDvmDex;
	});
```

        顺利得到 core.odex，jeb 可以正常打开，但是会出现一些异常，估计存在一些反反编译手段，我们用 DexClassLoader 加载试试 (因为我们加载的是 odex, 所以需要在同一目录下放一个 dex, 这里我随便弄了个 zip 包, 系统发现里面没 dex 会加载同目录下的 odex):

```
	DexClassLoader *dexloader = new DexClassLoader(env,
												   "/data/data/blog.csdn.net.rrrfff/cache/core.dex",
												   "/data/data/blog.csdn.net.rrrfff/.cache",
												   "/tmp",
												   DexClassLoader::getSystemLoader(env));
	auto cls  = dexloader->loadClass(env, "blog.csdn.net.rrrfff.TBJWelcomeActivity");
	auto cls2 = dexloader->loadClass(env, "blog.csdn.net.rrrfff.bean.UserInfoModel");
	LOGI("cls = %p %p", cls, cls2);
```

类加载成功, 证明 dump 出的 odex 格式良好 (对 ClassLoader 而言), 为了试试能否正常运行, 这里懒得去进行重打包、签名这些琐碎的工作，我们直接 hook dvmLookupClass 让它返回找不到的类 (去掉对原 SO 的加载后会报类找不到):

```
	static info<void *(*)(const char *descriptor, void *loader, bool unprepOkay)> *p0 = NULL;
	p0 = p0->hook(dlsym(dvm, "_Z14dvmLookupClassPKcP6Objectb"),
				  [](const char *descriptor, void *loader, bool unprepOkay)->void * {
		LOGI("dvmLookupClass hit with %s %p %d", descriptor, loader, unprepOkay);
		static volatile int during_load[10240] = { 0 };
		if (during_load[gettid()] == 0 && strstr(descriptor, "Lblog/csdn/net/rrrfff") == descriptor ) {
			char sdescriptor[128];
			int len = strlen(descriptor);
			::memcpy(sdescriptor, descriptor + 1, len - 2);
			sdescriptor[len -= 2] = 0;
			while (--len >= 0) {
				if (sdescriptor[len] == '/') sdescriptor[len] = '.';
			}
			JNIEnv *env;
			g_jvm->GetEnv(reinterpret_cast<void **>(&env), JNI_VERSION_1_6);
			LOGI("loading %s", sdescriptor);
			during_load[gettid()] = 1;
			auto cls = dexloader->loadClass(env, sdescriptor);
			during_load[gettid()] = 0;
			LOGI("%s loaded at %p", sdescriptor, cls);
			return decode_obj(get_self(), cls);
		} //if
		void *ret = p0->invoke()(descriptor, loader, unprepOkay);
		return ret;
	});
```

看日志可以发现很多类被成功加载了，但这里会报一个 native 方法找不到, com/secneo/apkwrapper/Helper 的 attach 方法, 签名 (Landroid/app/Application;Landroid/content/Context;)V, 看参数有 Application 和 Context，应该是壳完成资源修复等工作的切入点，后续再研究，我们给它注册个:

```
	void(*attach)(JNIEnv *env, jclass, jobject, jobject) = [](JNIEnv *env, jclass, jobject, jobject) {
		DEBUG_LINE_HIT;
	};
	JNINativeMethod gMethods[] = {
		{ "attach", "(Landroid/app/Application;Landroid/content/Context;)V", reinterpret_cast<void *>(attach) }
	};
	env->RegisterNatives(env->FindClass("com/secneo/apkwrapper/Helper"), gMethods, 1);
```

最后在 android.content.ContextWrapper.getResources(ContextWrapper.java:89) 处出现 java.lang.NullPointerException 异常, 可能是资源或者那个地方需要修复, 这里就没去深究了, 因为从 jeb 我们已经得到了想要的结果。
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2021050-1-1.html)

> [md]# frida 基于 bug 的 Stalker 跟踪检测和修复 ## 前言 > Stalker 是 Frida 内部的一个模块用于动态追踪目标程序的执行流程，也就是当我们需要知道：> - 函数是怎么调用 .......

![](https://avatar.52pojie.cn/data/avatar/000/93/31/49_avatar_middle.jpg)ZSAIm _ 本帖最后由 ZSAIm 于 2025-4-2 17:49 编辑_  

frida 基于 bug 的 Stalker 跟踪检测和修复
------------------------------

### 前言

> Stalker 是 Frida 内部的一个模块用于动态追踪目标程序的执行流程，也就是当我们需要知道：
> 
> *   函数是怎么调用的？
> *   指令是怎么一步步执行的？
> *   分支是怎么跳转的？
> *   有哪些函数是频繁被调用的？
> 
> 的时候，会用到`Stalker.follow`跟踪线程将要执行的指令，可以对指令动态插桩修改。对莫名其妙的反调试行为来说，是一个通用的兜底窥探方案。

### stalker 跟踪检测

> 关于 stalker 参考官方文档： [https://frida.re/docs/stalker/](https://frida.re/docs/stalker/)

**注： 这个检测手段仅对于在开启`Stalker.follow`跟踪的线程生效**

*   原理：stalker 会对执行过的内存地址 (`block->real_start`) 存储的指令会进行缓存，当再次进入要执行同一个地址的指令的时候，通过比对内存和缓存中的指令来确定是否需要重编译`block`来确保执行到正确的原指令 - 本应该是如此，但是正如标题所说，这个检测依赖当前`frida-gum`项目的`stalker`的重编译 bug，而这个 bug 就出现在比对和重编译上，导致了当执行同一个内存地址的`block`时候，实际指令都是在跟第一次编译后的缓存`block`的原指令快照进行比对。所以这样的后果就是显而易见的，初次执行该内存地址的指令会一直存在于`stalker`的 "阴影中"，下面我会用一个测试例子来说明这个现象。

以上的场景说明存在简化，实际的问题路径在下面对应的代码:

1.  [指令比对](https://github.com/frida/frida-gum/blob/main/gum/backend-arm64/gumstalker-arm64.c#L2596)
2.  [block 重编译](https://github.com/frida/frida-gum/blob/main/gum/backend-arm64/gumstalker-arm64.c#L2635)

#### 通过下面一个例子来验证这个问题

*   为了利用上面这个 bug，我们可以简单的实现两个函数，一个函数 (`add_99_func`) 是对传递的第一个参数 + 99，另一个函数 (`empty_func`) 是不做任何操作直接返回原参数。

```
uint64_t add_99_func(uint64_t count) {
    return count + 99;
}

// 对应的汇编可以是:
// add x0, x0, #99
// ret
```

```
uint64 empty_func(uint64_t count){
    return count;
}

// 对应的汇编可以是:
// ret
```

*   使用`mmap`来创建指定内存块用于复写和执行指令

```
static void *exec_addr = nullptr;

// insnBytes: 要写入的指令字节数组
// num: 调用func传递的count值
jstring mmap_exec(JNIEnv *env, jobject thiz, jbyteArray insnBytes, jint num) {
    size_t page_size = sysconf(_SC_PAGE_SIZE);
    if (page_size == -1) {
        std::cerr << "Failed to get page size!" << std::endl;
        return nullptr;
    }

    void *start_addr = exec_addr;
    int flags = MAP_ANON | MAP_PRIVATE;
    if (start_addr != nullptr) {
        flags |= MAP_FIXED;
    }
    void *mem = mmap(
            start_addr, page_size, PROT_READ | PROT_WRITE | PROT_EXEC,
             flags, -1, 0); 
    if (mem == MAP_FAILED) {
        std::cerr << "mmap failed!" << std::endl;
        exec_addr = nullptr;
        return nullptr;
    }
    if (exec_addr == nullptr) {
        exec_addr = mem; // 下次也使用同一内存地址来写入指令
    }
    jsize length = env->GetArrayLength((insnBytes));

    jbyte *code = env->GetByteArrayElements(insnBytes, nullptr);
    std::memcpy(mem, code, length);

    aarch64_sync_cache_range(mem, page_size); // 刷新cpu指令和内存数据缓存

    func_t func = reinterpret_cast<func_t>(mem);

    uint64_t sum = func(num); // 调用函数

    munmap(mem, page_size);

    env->ReleaseByteArrayElements(insnBytes, code, 0);

    char buff[128];
    std::sprintf(buff, "[%p] sum[%lu]", mem, sum);

    return env->NewStringUTF(buff);
}

// jni native
__attribute__((visibility("default")))
JNIEXPORT jobjectArray JNICALL
Java_com_example_frida_1stalker_1recompile_1fix_MainActivity_mmapExec(
        JNIEnv *env, jobject thiz, jbyteArray inst1, jbyteArray inst2, jint base_num) {
    jclass stringCls = env->FindClass("java/lang/String");
    const jsize test_num = 20;
    jobjectArray resultArray = env->NewObjectArray(test_num, stringCls, nullptr);
    // 每点击一次按钮，遍历交替复写执行20次
    for(jsize i = 0; i < test_num; i++) {
        jbyteArray inst = i % 2 == 0 ? inst1 : inst2;
        jstring result1 = mmap_exec(env, thiz, inst, base_num);
        env->SetObjectArrayElement(resultArray, i, result1);
    }
    return resultArray;
}
```

*   frida 测试脚本

```
setImmediate(() => {
    Interceptor.attach(
        Module.getExportByName('libdl.so', 'android_dlopen_ext'), {
            onEnter(args) {
                this.filename = args[0].readCString()
                console.error(`[android_dlopen_ext] ${this.filename}`)
            },
            onLeave(retval) {
                const filename: string = this.filename
                if(filename.includes('libtest_frida.so')) {
                    attachMmapExec(Process.findModuleByName(filename)!)
                }
            },
        }
    )

    function attachMmapExec(mod: Module) {
        console.error(`[attachMmapExec] ${JSON.stringify(mod)}`)
        const target = mod.getExportByName("Java_com_example_frida_1stalker_1recompile_1fix_MainActivity_mmapExec")
        Interceptor.attach(target, {
            onEnter(args) {
                console.log(`[mmap_exec] follow => tid[${Process.getCurrentThreadId()}]`)
                Stalker.follow(Process.getCurrentThreadId(), {
                    transform(iterator: StalkerArm64Iterator) {
                        let inst
                        while((inst = iterator.next()) !== null) {
                            // console.log(`[${Process.getCurrentThreadId()}] ${inst}`)
                            iterator.keep()
                        }
                    }
                })
            },
            onLeave(retval) {
                Stalker.unfollow()
                console.error(`[mmap_exec] unfollow => tid[${Process.getCurrentThreadId()}]`)
            },
        })
    }
})
```

*   通过对于同一个内存块进行来回按照`empty_func`，`add_99_func`的顺序来写入对应的的汇编来执行（假设传递的`count=1`）：<sup> [代码 (2)](https://github.com/ZSA233/android-reverse-examples/blob/main/001_frida-stalker-recompile-fix/app/app/src/main/cpp/test_frida.cpp)</sup>
    1.  第一次执行`empty_func(1)`的时候返回值是 1（首次编译，缓存根据 empty_func 生成的插桩指令）
    2.  通过比对指令，第二次进行了重编译，执行`add_99_func(1)`返回 100 (第一次重编译，缓存根据 add_99_func 生成的插桩指令，但是指令快照却是 empty_func)
    3.  第三次执行，预期是执行`empty_func(1)`返回 1，但实际却因为快照是第一次编译的指令（empty_func），所以误认为指令没变动，所以执行了**第一次重编译**的插桩代码 add_99_func，导致了此时会返回 100。
    4.  第四次执行，预期是执行`add_99_func(1)`返回 100，实际上也是执行了`add_99_func(1)`，但是错误的又重编译一次
    5.  ... 接下来的都会是执行`add_99_func(1)`的结果

以下展示了三种测试结果用于说明：

##### 1. 正常无 frida-stalker 跟踪现象

![](https://attach.52pojie.cn/forum/202504/02/173347ixxw0d0xwkz1wztt.jpg)

**003-frida-stalker 修复. jpg** _(254.68 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc2ODMwN3wxMWQzMGVlM3wxNzQzNjQ1NTM2fDIxMzQzMXwyMDIxMDUw&nothumb=yes)

2025-4-2 17:33 上传

##### 2. 使用原版 frida-stalker 跟踪现象

![](https://attach.52pojie.cn/forum/202504/02/173345o5dajzrtl1jjnzd8.jpg)

**002-frida-stalker 原版. jpg** _(268.93 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc2ODMwNnwxYjkwNzBmMnwxNzQzNjQ1NTM2fDIxMzQzMXwyMDIxMDUw&nothumb=yes)

2025-4-2 17:33 上传

##### 3. 使用修复的 frida-stalker 后的跟踪现象

![](https://attach.52pojie.cn/forum/202504/02/173342owwzpsdliww14xpl.jpg)

**001 - 正常执行. jpg** _(244.35 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc2ODMwNXw3MGNlMzM5YnwxNzQzNjQ1NTM2fDIxMzQzMXwyMDIxMDUw&nothumb=yes)

2025-4-2 17:33 上传

### 编译修复 frida-server

#### 拉取源项目和合并修改

```
# 从frida中递归子模块拉取主分支(修改基于16.5.9进行，所以我编译这个版本，其他版本如果没冲突可以试试)
git clone --branch 16.5.9 --single-branch --recurse-submodules https://github.com/frida/frida.git

# 进入子模块下的frida-gum
cd subprojects/frida-gum

# 添加修复的远程代码仓库地址
git remote add zsa233 https://github.com/zsa233/frida-gum.git

# 拉取zsa233仓库，并且将对应的修改分支合并到本地仓库
git fetch zsa233
git merge zsa233/fix/stalker-wrong-recompile

# 后面接着就是编译
```

#### 编译

> 参考官方文档： [https://frida.re/docs/building/#cross](https://frida.re/docs/building/#cross)

```
# 指定ANDROID_NDK_ROOT，我这里下载r25c版本（16.5.9要求
# wget https://dl.google.com/android/repository/android-ndk-r25c-linux.zip
# 其中还需要node.js >= 18之类的编译依赖，根据错误提示解决即可，这里不再赘述
# 
export ANDROID_NDK_VERSION=r25c
# ！！！/mypath/改成自己的ndk路径
export ANDROID_NDK_ROOT=/mypath/android-ndk-$ANDROID_NDK_VERSION/

# 回到frida项目下，配置编译android-arm64平台
./configure --host=android-arm64

# 编译
make

# 将编译成功的frida-server上传
adb push build/subprojects/frida-core/server/frida-server /data/local/tmp/frida-server
# 加上x权限
adb shell chmod u+x /data/local/tmp/frida-server
```

### 测试

> [修复的测试结果如上图](#3-使用修复的frida-stalker后的跟踪现象)

### 资源链接

> 0.  [代码仓库](https://github.com/ZSA233/android-reverse-examples/tree/main/001_frida-stalker-recompile-fix)
> 1.  [测试 app 代码](https://github.com/ZSA233/android-reverse-examples/tree/main/001_frida-stalker-recompile-fix/app)
> 2.  [so 库实现](https://github.com/ZSA233/android-reverse-examples/blob/main/001_frida-stalker-recompile-fix/app/app/src/main/cpp/test_frida.cpp)
> 3.  [frida 注入脚本](https://github.com/ZSA233/android-reverse-examples/blob/main/001_frida-stalker-recompile-fix/index.ts)
> 4.  [编译的 arm64 测试 apk](https://github.com/ZSA233/android-reverse-examples/blob/main/001_frida-stalker-recompile-fix/app-debug.apk)
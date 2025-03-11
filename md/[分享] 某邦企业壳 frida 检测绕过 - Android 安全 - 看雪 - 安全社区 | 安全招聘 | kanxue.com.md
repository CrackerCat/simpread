> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-285932.htm)

> [分享] 某邦企业壳 frida 检测绕过

某企业壳 frida 检测绕过尝试
=================

遇见个汽车 app 是个企业壳 有 frida 检测，导致无法注入，本文记录下绕过的过程（使用的 frida 版本葫芦嗒）

![](https://bbs.kanxue.com/upload/attach/202503/949800_W98UVQ9XSUCASKF.png)

![](https://bbs.kanxue.com/upload/attach/202503/949800_4HR4EZVMND4Q9A4.png)

通过查壳发现，这是某企业壳 知其特点检测主要是在 dexhelp 这个 so 文件中，也就省去了定位检测位置这一步

##### 尝试 hook pthread_create 定位线程 通过杀线程方法干掉他

```
function hook_pthread_create(){
    var pthread_create_addr = Module.findExportByName("libc.so", "pthread_create");
    console.log("pthread_create_addr: ", pthread_create_addr);
    Interceptor.attach(pthread_create_addr,{
        onEnter:function(args){
            console.log(args[2], Process.findModuleByAddress(args[2]).name);
        },onLeave:function(retval){
        }
    });
}
hook_pthread_create();
```

![](https://bbs.kanxue.com/upload/attach/202503/949800_A7BJTSW74N7HT7A.png)

发现没啥用 应该是检测到 `pthread_create` 被 hook 了 既然`pthread_create` 被检测了 那么是否还有其他方法定位线程呢，于是乎 我打开了逆向小助手 gpt 首先我去问了下`pthread_create` 实现原理 gpt 给出了这个回答

![](https://bbs.kanxue.com/upload/attach/202503/949800_HQ98XYBBE8MTPS8.png)

然后 我又去问了 gpt hook `pthread_create` 被检测到了 该怎么办 他建议我去 hook 其他相关函数 比如`pthread_join` clone 通过 gpt 给出的答案 `pthread_create` 的实现会调用 `clone` 系统调用来创建新线程

他既然检测 `pthread_create` 函数头被 hook 那 clone 函数应该没有把 通过 hook clone 函数 得到线程相关信息然后去 nop 干掉线程 是不是 也能一把梭了 说干就干

![](https://bbs.kanxue.com/upload/attach/202503/949800_AP4WNPCN84EAGHM.png)

首先 我去 hook dlopen 在打开 dexhelp 这个 so 的时候 去 hook 下 jnionload 看看它做手脚的地方是否在 init 里面 发现不是 那我只需要去 hook clone 函数 打印出相关线程信息 然后 加载这个 so 的时候 nop 掉它 测试是否能绕过即可

于是开始写 hook 首先 找 gpt 要了一份 clone 实现创建线程的例子

```
#define _GNU_SOURCE
#include #include #include #include #define STACK_SIZE 1024*1024  // 分配线程栈空间
 
// 子线程函数
int thread_func(void *arg) {
    printf("Hello from the new thread!\n");
    return 0;
}
 
int main() {
    char *stack;  // 存放线程栈的空间
    char *stackTop;
 
    // 为线程分配栈
    stack = malloc(STACK_SIZE);
    if (stack == NULL) {
        perror("malloc");
        exit(1);
    }
    stackTop = stack + STACK_SIZE;  // 设置栈顶
 
    // 使用 clone 创建新线程
    int pid = clone(thread_func, stackTop, SIGCHLD, NULL);  // 子进程退出时会发出 SIGCHLD 信号
    if (pid == -1) {
        perror("clone");
        exit(1);
    }
 
    // 等待线程结束
    printf("Waiting for the thread to finish...\n");
    waitpid(pid, NULL, 0);
 
    free(stack);  // 释放栈空间
    return 0;
} 
```

传进去第一个参数是 子线程函数 于是乎 嘎嘎嘎 hook 并且打印 结果发现全是 libc.so 这是咋回事呢 看来还得好好学习 于是乎 我拿了一份 libc.so ida 打开它 看 pthread_create 如何 去调用 clone 函数创建线程的

```
// Hook clone 系统调用
Interceptor.attach(Module.findExportByName(null, 'clone'), {
    onEnter: function (args) {
        // 获取线程函数的地址（即第一个参数）
        var thread_func = args[0];
 
        // 打印线程函数地址
        console.log('Thread function address: ' + thread_func);
 
        // 获取线程函数所在模块（.so 文件）
        var module = Process.findModuleByAddress(thread_func);
        if (module) {
            console.log('Thread function is located in module: ' + module.name);
        } else {
            console.log('Thread function is not in a loaded module or location could not be determined.');
        }
    },
    onLeave: function (retval) {
        // 可以在这里修改 clone 的返回值（如需要）
    }
});
 
输出
Thread function address: 0x749b4140dc
Thread function is located in module: libc.so
Thread function address: 0x749b4140dc
Thread function is located in module: libc.so
Thread function address: 0x749b4140dc
Thread function is located in module: libc.so
```

在上诉 clone hook 处 我打印了下堆栈 看下哪里调用了 代码和 hook 结果如下

```
Interceptor.attach(Module.findExportByName(null, 'clone'), {
    onEnter: function (args) {
        // 获取线程函数地址
        var thread_func = args[0];
 
        // 尝试获取线程函数所在模块
        var module = Process.findModuleByAddress(thread_func);
        if (module) {
            console.log('Thread function is located in module: ' + module.name);
        } else {
 
        }
 
        // 打印调用栈
        console.log('Backtrace:');
        console.log(Thread.backtrace(this.context, Backtracer.ACCURATE)
            .map(DebugSymbol.fromAddress).join('\n'));
    },
    onLeave: function (retval) {
        // 返回值处理（如果需要）
    }
});
 
结果
Thread function is located in module: libc.so
Backtrace:
0x749b413e40 libc.so!pthread_create+0x27c
```

然后 IDA 跳转到 pthread_create+0x27c 此处 调用了函数 E6850 于是进去瞧一瞧 发现是 DCQ 数据

![](https://bbs.kanxue.com/upload/attach/202503/949800_RWRB4FPPTS6A5RM.png)

继续使用 frida hook 读取下 结果是 clone 也就说这里调用了 clone 函数

```
Interceptor.attach(Module.findExportByName(null, 'clone'), {
    onEnter: function (args) {
        // 获取线程函数地址
        var thread_func = args[0];
 
 
 
        // 尝试获取线程函数所在模块
        var module = Process.findModuleByAddress(thread_func);
        if (module) {
            console.log('Thread function is located in module: ' + module.name);
            const address = module.base.add(0xF06E8);
            const value = Memory.readU64(address);
            const symbol = DebugSymbol.fromAddress(ptr(value));
            console.log("name",symbol.name)
 
        } else {
 
        }
 
        // 打印调用栈
 
    },
    onLeave: function (retval) {
        // 返回值处理（如果需要）
    }
});
 
结果 clone
```

![](https://bbs.kanxue.com/upload/attach/202503/949800_ZBVQAKH2KK3CYJN.png)

观察 pthread_create 函数的传参走向 也就是 a3

![](https://bbs.kanxue.com/upload/attach/202503/949800_2PE4SX6B6EBAQ28.png)

![](https://bbs.kanxue.com/upload/attach/202503/949800_8YPVZWMVYF2M8XX.png)

此处的 E6850 是 clone 而 通过 pthread_create 函数创建线程传递的 线程函数 就是 v50 而 clone 函数的第四个参数 v27 + 96 是所需要的线程函数 这样 就可以去试试 写一下 hook clone 代码了

```
var clone = Module.findExportByName('libc.so', 'clone');
Interceptor.attach(clone, {
    onEnter: function(args) {
        // args[3] 子线程的栈地址。如果这个值为 0，可能意味着没有指定栈地址
        if(args[3] != 0){
            var addr = args[3].add(96).readPointer()
            var so_name = Process.findModuleByAddress(addr).name;
            var so_base = Module.getBaseAddress(so_name);
            var offset = (addr - so_base);
            console.log("===============>", so_name, addr,offset, offset.toString(16));
        }
    },
    onLeave: function(retval) {
 
    }
});
```

然后运行后发现 非常阔以 打印出来了我们所需要的参数 非常 nice

![](https://bbs.kanxue.com/upload/attach/202503/949800_WP9WTYMPK2V8PHR.png)

最后 我们 去给他 nop 掉 即可

```
function hook_dlopen(so_name) {
    Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"), {
        onEnter: function (args) {
            var pathptr = args[0];
            if (pathptr !== undefined && pathptr != null) {
                var path = ptr(pathptr).readCString();
                // console.log(path)
                if (path.indexOf(so_name) !== -1) {
                    this.match = true
                }
            }
        },
        onLeave: function (retval) {
            if (this.match) {
                console.log(so_name, "加载成功");
                var base = Module.findBaseAddress("libDexHelper.so")
                patch_func_nop(base.add(322008));
                patch_func_nop(base.add(308756));
 
            }
        }
    });
}
function patch_func_nop(addr) {
    Memory.patchCode(addr, 8, function (code) {
        code.writeByteArray([0xE0, 0x03, 0x00, 0xAA]);
        code.writeByteArray([0xC0, 0x03, 0x5F, 0xD6]);
        console.log("patch code at " + addr)
    });
}
hook_dlopen("libDexHelper.so")
```

最后 看结果

![](https://bbs.kanxue.com/upload/attach/202503/949800_7WCF95RNKXA4X27.png)

[[注意] 看雪招聘，专注安全领域的专业人才平台！](https://job.kanxue.com/)

[#工具脚本](forum-161-1-128.htm)
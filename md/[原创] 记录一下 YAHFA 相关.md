> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267606.htm)

> [原创] 记录一下 YAHFA 相关

这里使用的手机是 google Pixel XL 8.1  
本文的目的是记录一下使用 frida 对 YAHFA 的原理的简单分析

 

首先写一个安卓 demo 如下

```
findViewById(R.id.sample_text).setOnClickListener(v -> {
        try {
            Method doWork1 = MainActivity.class.getDeclaredMethod("doWork1");
            Method doWork2 = MainActivity.class.getDeclaredMethod("doWork2");
            Method doWork3 = MainActivity.class.getDeclaredMethod("doWork3");
 
            calledBefore(doWork1,doWork2,doWork3);
            HookMain.backupAndHook(doWork1,doWork2,doWork3);
            calledAfter(doWork1,doWork2,doWork3);
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        }
    });
}
 
private static void doWork1() {
    Log.i(TAG, "doWork1");
}
private static void doWork2() {
    Log.i(TAG, "doWork2");
}
private static void doWork3() {
    Log.i(TAG, "doWork3");
}
 
public native void calledBefore(Method doWork1, Method doWork2, Method doWork3);
public native void calledAfter(Method doWork1, Method doWork2, Method doWork3);

```

两个 native 方法 (calledBefore/calledAfter) 在 native 层啥也不用做 只是拿到一个反射方法

 

顺带附上 hook 代码, 主要是用 frida 展示 artmethod 指针内存的变化

```
var getArtMethod = new NativeFunction(Module.findExportByName('libnative-lib.so','getArtMethod'),'pointer',['pointer','pointer'])
 
Interceptor.attach(Module.findExportByName('libnative-lib.so','Java_com_lzy_yahfa_MainActivity_calledBefore'),{
    onEnter:function(args){
        LOG("\n----------------------- Before -----------------------\n",LogColor.RED)
        showLog(args[0],args[2],args[3],args[4])
    },
    onLeave:function(ret){
 
    }
})
 
Interceptor.attach(Module.findExportByName('libnative-lib.so','Java_com_lzy_yahfa_MainActivity_calledAfter'),{
    onEnter:function(args){
        LOG("\n----------------------- After -----------------------\n",LogColor.RED)
        showLog(args[0],args[2],args[3],args[4])
    },
    onLeave:function(ret){
 
    }
})
 
function showLog(a0,a1,a2,a3){
 
    LOG(" ----- ORG  ----- ",LogColor.YELLOW)
    var method = getArtMethod(a0,a1)
    seeHexA(method,p_size*8)
    LOG("entry_point_from_quick_compiled_code -> "+method.add(p_size*7).readPointer()+" ---> "+method.add(p_size*7).readPointer().readPointer())
    LOG("\n")
 
    LOG(" ----- Hook  ----- ",LogColor.YELLOW)
    var method = getArtMethod(a0,a2)
    seeHexA(method,p_size*8)   
    LOG("entry_point_from_quick_compiled_code -> "+method.add(p_size*7).readPointer()+" ---> "+method.add(p_size*7).readPointer().readPointer())
    LOG("\n")
 
    LOG(" ----- Back  ----- ",LogColor.YELLOW)
    var method = getArtMethod(a0,a3)
    seeHexA(method,p_size*8)
    LOG("entry_point_from_quick_compiled_code -> "+method.add(p_size*7).readPointer()+" ---> "+method.add(p_size*7).readPointer().readPointer())
}

```

这里我用的是 google 原生系统 8.1  
直接去参考源码得到 [artMethod](https://www.androidos.net.cn/android/8.0.0_r4/xref/art/runtime/art_method.h) 结构体长这样:

```
4        GcRoot declaring_class_;
 
4        std::atomic access_flags_;   
 
4        uint32_t dex_code_item_offset_;
 
4        uint32_t dex_method_index_;
 
2        uint16_t method_index_;
 
2        uint16_t hotness_count_;
 
12        struct PtrSizedFields {
4            ArtMethod** dex_cache_resolved_methods_;
 
4            void* data_;
 
4            void* entry_point_from_quick_compiled_code_;
        } ptr_sized_fields_; 
```

运行起来点击 textview 即可得到一下日志

 

![](https://bbs.pediy.com/upload/attach/202105/868525_ZUJNBPZMT4DYTVQ.png)

 

cpu 三级流水线可知程序运行到 0xf48bc018 的时候  
pc 应该是往后的两条指令, 即为 0xf48bc020

 

![](https://bbs.pediy.com/upload/attach/202105/868525_W3VA5F3YVCGQSHF.png)

 

这里计算一下跳板地址跳到了哪里

 

![](https://bbs.pediy.com/upload/attach/202105/868525_28PE4CXNVFM57PE.png)

 

很明显这个位置就是 hook 函数的 entry_point_from_quick_compiled_code_

 

继续看一下 BackUp 函数的实现

 

![](https://bbs.pediy.com/upload/attach/202105/868525_UDPJR6SYN5FJ6P5.png)

 

第一条指令  
ldr r0, [pc, #0xc]  
就是把 pc+0x2+0xc 位置的值给到了 r0, 即等价于  
ldr r0,=0xf410307c

 

第二条指令  
stmdb sp!, {r0}  
r0 压栈

 

第三条指令  
ldr r0, [pc]  
等价于  
ldr r0,=0xf36174d1 (这个地址就是他的默认入口)

 

第四条指令  
ldm sp!, {pc}  
将 sp 出到 pc, 也就是配置 r0 的值为第一个 org method 指针, 并恢复 pc 为默认的函数入口 0xf36174d1

 

补充一点 artMethod* 的 4-8 字节 access_flags_ 的变化  
由 0x0008000a -> 0x0208010a  
做了两步操作（标志位参考）

1.  kAccCompileDontBother（0x01000000）解释执行
2.  kAccNative（0x0100 native 标志位  
    对应就表现为 (0b === 0x, 强行补齐方便对照)  
    ![](https://bbs.pediy.com/upload/attach/202105/868525_W4HFAF6PRVGWCUE.png)

其他：

1.  这里的 ldm stmdb 进行入栈出栈并不影响栈顶指针
2.  这里的函数入口处对 sp 的读写没关系, 不用关心覆盖问题
3.  从 0xf48bc000 - 0xf48bc024 部分就是 YAHFA 补出来的跳板代码
4.  替换函数用到了跳板, 关键在于 r0 和 pc（参考第一个参数就是 artmethod*）
5.  函数开始的时候使用 sp 不用管覆盖问题，中途用 sp 可以考虑使用超出栈顶指针的部分内存

这是个人的一些理解哈，如果有错的地方 欢迎各位大佬指出

[[注意] 招人！base 上海，课程运营、市场多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)

最后于 2021-5-15 07:44 被唱过阡陌编辑 ，原因：

[#基础理论](forum-161-1-117.htm) [#HOOK 注入](forum-161-1-125.htm)
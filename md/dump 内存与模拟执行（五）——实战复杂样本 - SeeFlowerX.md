> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.seeflower.dev](https://blog.seeflower.dev/archives/171/)

> 前言在上一篇系列文章中，以为绿洲为样本演示了 dump 上下文后模拟执行的整个过程绿洲 com.weibo.xvideo.NativeApi 的 s 算法相对简单，涉及的 JNI 调用较少，整体模拟执行过程还是...

在上一篇系列文章中，以为绿洲为样本演示了 dump 上下文后模拟执行的整个过程

绿洲`com.weibo.xvideo.NativeApi`的`s`算法相对简单，涉及的 JNI 调用较少，整体模拟执行过程还是比较顺利

但这似乎让人对 dump 上下文后模拟执行的优势感知不强，本文将以更加复杂的目标算法为例以提升用户感知~

本文的样本是`快手 10.6.50.26734 32位`

目标是模拟执行`com.kuaishou.android.security.internal.dispatch.JNICLibrary`的`doCommandNative`方法

frida hook
----------

### 获取入参和结果

首先是通过 frida 获取传入参数，不熟悉 frida 的同学可以看看怎么打印`[Ljava.lang.Object;`包裹的数组和`Ljava/lang/Object;`包裹的`[B`

```
function obj_byte_arr_to_hexstr(byte_arr){
    let HexDump = Java.use("com.android.internal.util.HexDump");
    // HexDump.hexStringToByteArray()
    // HexDump.toHexString()
    let buffer = Java.cast(byte_arr, Java.use("[B"));
    return HexDump.toHexString(Java.array('byte', buffer));
}

function hook_ks_doCommandNative(){
    Java.perform(function(){
        let Arrays = Java.use("java.util.Arrays");
        let ArrayClz = Java.use("java.lang.reflect.Array");
        let JNICLibrary = Java.use("com.kuaishou.android.security.internal.dispatch.JNICLibrary");

        JNICLibrary.doCommandNative.overload('int', '[Ljava.lang.Object;').implementation = function(num, objs){
            console.log(`[enter] => ${num} ${objs}`);
            let bflag = false;
            if (num == 10400){
                bflag = true;
            }
            let arr = Arrays.asList(objs);
            for (let index = 0; index < arr.size(); index++) {
                let element = arr.get(index);
                console.log(`[arrarr] ${index} ===>${element}<===`);
                if (index == 0){
                    if (bflag){
                        console.log(`[first arr] ===>${obj_byte_arr_to_hexstr(element)}<===`);
                    }
                    else{
                        for(let index = 0; index < ArrayClz.getLength(element); index++){
                            let item = ArrayClz.get(element, index).toString();
                            console.log(`[first arr] ${index} ===>${item}<===`);
                        }
                    }
                }
            }
            let result = this.doCommandNative(num, objs);
            if (bflag){
                console.log(`[leave] => ${obj_byte_arr_to_hexstr(result)}`);
            }else{
                console.log(`[leave] => ${result}`);
            }
            return result;
        }
    })
}
```

可以得到这些输出，现在入参和结果都有了

```
[enter] => 10418 [Ljava.lang.String;@a2ac698,d7b7d042-d4f2-4012-be60-d97ff2429c17,-1,false,com.yxcorp.gifshow.App@9ac1cd7,,false,
[arrarr] 0 ===>[Ljava.lang.String;@a2ac698<===
[first arr] 0 ===>/rest/n/magicFace/ycnn/models88a33d1370a36dd86aa8af7781ed3bb3<===
[arrarr] 1 ===>d7b7d042-d4f2-4012-be60-d97ff2429c17<===
[arrarr] 2 ===>-1<===
[arrarr] 3 ===>false<===
[arrarr] 4 ===>com.yxcorp.gifshow.App@9ac1cd7<===
[arrarr] 5 ===>null<===
[arrarr] 6 ===>false<===
[arrarr] 7 ===><===
[leave] => e8f989aaaff3b2dc81a0a3a2e2c167d8c96354d8bdb1bfa9
[enter] => 10418 [Ljava.lang.String;@9a6964b,d7b7d042-d4f2-4012-be60-d97ff2429c17,-1,false,com.yxcorp.gifshow.App@9ac1cd7,,false,
[arrarr] 0 ===>[Ljava.lang.String;@9a6964b<===
[first arr] 0 ===>/rest/e/load/styleTemplate52729eed3a702a534fe721efea0e9cb0<===
[arrarr] 1 ===>d7b7d042-d4f2-4012-be60-d97ff2429c17<===
[arrarr] 2 ===>-1<===
[arrarr] 3 ===>false<===
[arrarr] 4 ===>com.yxcorp.gifshow.App@9ac1cd7<===
[arrarr] 5 ===>null<===
[arrarr] 6 ===>false<===
[arrarr] 7 ===><===
[leave] => 7b6a1a393c60214f13333031bc11a5225af0c74b2e222c3a
[enter] => 10418 [Ljava.lang.String;@7a575ab,d7b7d042-d4f2-4012-be60-d97ff2429c17,-1,false,com.yxcorp.gifshow.App@9ac1cd7,,false,
[arrarr] 0 ===>[Ljava.lang.String;@7a575ab<===
[first arr] 0 ===>/rest/n/encode/androidffdb4a81067a6f7e20afbc3ec76652b3<===
[arrarr] 1 ===>d7b7d042-d4f2-4012-be60-d97ff2429c17<===
[arrarr] 2 ===>-1<===
[arrarr] 3 ===>false<===
[arrarr] 4 ===>com.yxcorp.gifshow.App@9ac1cd7<===
[arrarr] 5 ===>null<===
[arrarr] 6 ===>false<===
[arrarr] 7 ===><===
[leave] => 8e9fefccc995d4bae5c6c5c4d00b944fad0532bedbd7d9cf
```

### 获取 doCommandNative 在 native 层的地址

直接上`hook_RegisterNatives.js`即可

*   [https://github.com/lasting-yang/frida_hook_libart/blob/master/hook_RegisterNatives.js](https://github.com/lasting-yang/frida_hook_libart/blob/master/hook_RegisterNatives.js)

```
[RegisterNatives] java_class: com.kuaishou.android.security.internal.dispatch.JNICLibrary name: doCommandNative sig: (I[Ljava/lang/Object;)Ljava/lang/Object; fnPtr: 0xac38d919  fnOffset: 0x4b919  callee: 0xac395547 libkwsgmain.so!0x53547
```

即偏移是`0x4b919`，用 IDA 打开看下，直接就是红色...

不过暂时可以不关心怎么修复，第一条指令是 push 那就基本能确定是这了

![](https://blog.seeflower.dev/images/Snipaste_2022-08-07_23-32-04.png)

dump 上下文
--------

ttyd 启动

```
ttyd -t fontSize=22 -t 'theme={"background": "#282A36", "foreground": "#F8F8F2", "cursor": "#F8F8F2", "selection": "#44475A"}' bash
```

PC 远程访问手机，进入 shell，切换到`su`

设置环境变量

```
export SHELL=/data/data/com.termux/files/usr/bin/bash
export COLORTERM=truecolor
export HISTCONTROL=ignoreboth
export PREFIX=/data/data/com.termux/files/usr
export TERMUX_IS_DEBUGGABLE_BUILD=1
export TERMUX_MAIN_PACKAGE_FORMAT=debian
export PWD=/data/data/com.termux/files/home
export TERMUX_VERSION=0.118.0
export EXTERNAL_STORAGE=/sdcard
export LD_PRELOAD=/data/data/com.termux/files/usr/lib/libtermux-exec.so
export HOME=/data/data/com.termux/files/home
export LANG=en_US.UTF-8
export TERMUX_APK_RELEASE=GITHUB
export TMPDIR=/data/data/com.termux/files/usr/tmp
export TERM=xterm-256color
export SHLVL=2
export PATH=/data/data/com.termux/files/usr/bin:$PATH
```

执行 lldb 命令，附加到目标进程

```
lldb
attach 16714
```

![](https://blog.seeflower.dev/images/Snipaste_2022-08-07_23-36-24.png)

处理`SIGSEGV`信号

```
process handle SIGSEGV -p true -s false
```

下断点

```
breakpoint set -s libkwsgmain.so -a 0x4b919
```

按`c`继续执行，操作 APP 等待断点

![](https://blog.seeflower.dev/images/Snipaste_2022-08-07_23-42-20.png)

断下来的时候，通过`register read r2`看看传入参数，这里对应的也就是 java 层的第一个参数

前面 frida hook 已经知道大部分情况下参数一是`10418`，也就是图中的`0x28b2`

另外注意如果你有多个断点，也就是图中圈起来的`Thread 250`，你可以通过`thread select xxx`来切换线程

因为有时候当前断下的可能不是想要的线程，千万不要搞错了

![](https://blog.seeflower.dev/images/Snipaste_2022-08-07_23-49-16.png)

加载 dump 上下文脚本，执行`dumpctx`开始 dump 上下文

```
command script import lldb_dumper.py
```

![](https://blog.seeflower.dev/images/Snipaste_2022-08-07_23-53-13.png)

等待 dump 完成

![](https://blog.seeflower.dev/images/Snipaste_2022-08-07_23-54-54.png)

之前提到还需要获取 Canary 指针地址，但是经过分析 32 位 so 的目标函数附近似乎暂时没用到，所以跳过

删除断点，结束附加

![](https://blog.seeflower.dev/images/Snipaste_2022-08-07_23-57-23.png)

下载 dump 好的上下文压缩包 (这时间怎么不对... 看了下原来手机时间确实是错的...)

```
lsz DumpContext_20220808_000250.zip
```

![](https://blog.seeflower.dev/images/Snipaste_2022-08-07_23-58-20.png)

模拟执行
----

先搭一个框架如下，本案例是 32 位的，所以写寄存器部分略有不同，仅供参考，其他部分和绿洲案例一致

默认先只载入了`libkwsgmain.so`和`libc.so`有关的内存

另外加了`emulator.traceCode();`看看能模拟执行到什么地方

另外这里要把`pc + 1`作为起始地址，而不是直接传`pc`，否则 unicorn 会不认识指令。。。

```
public class JNICLibrary extends AbstractJni {

    private final AndroidEmulator emulator;
    private final VM vm;

    JNICLibrary() {
        emulator = AndroidEmulatorBuilder.for32Bit()
                .setProcessName("com.smile.gifmaker")
                .addBackendFactory(new Unicorn2Factory(true))
                .build(); // 创建模拟器实例，要模拟32位或者64位，在这里区分
        emulator.getSyscallHandler().setEnableThreadDispatcher(false);
        final Memory memory = emulator.getMemory(); // 模拟器的内存操作接口
        memory.setLibraryResolver(new AndroidResolver(23)); // 设置系统类库解析

        vm = emulator.createDalvikVM(); // 创建Android虚拟机
        vm.setJni(this);
        vm.setVerbose(true); // 设置是否打印Jni调用细节
    }

    public static void main(String[] args) throws Exception {
        JNICLibrary mJNICLibrary = new JNICLibrary();
        mJNICLibrary.load_context("unidbg-android/src/test/resources/DumpContext_20220808_000250");
        mJNICLibrary.doCommandNative();
    }

    int UNICORN_PAGE_SIZE = 0x1000;

    private long align_page_down(long x){
        return x & ~(UNICORN_PAGE_SIZE - 1);
    }
    private long align_page_up(long x){
        return (x + UNICORN_PAGE_SIZE - 1) & ~(UNICORN_PAGE_SIZE - 1);
    }

    private void map_segment(long address, long size, int perms){

        long mem_start = address;
        long mem_end = address + size;
        long mem_start_aligned = align_page_down(mem_start);
        long mem_end_aligned = align_page_up(mem_end);

        if (mem_start_aligned < mem_end_aligned){
            emulator.getBackend().mem_map(mem_start_aligned, mem_end_aligned - mem_start_aligned, perms);
        }
    }

    private void load_context(String dump_dir) throws IOException, DataFormatException, IOException {

        Backend backend = emulator.getBackend();
        Memory memory = emulator.getMemory();

        String context_file = dump_dir + "\\" + "_index.json";
        InputStream is = new FileInputStream(context_file);
        String jsonTxt = IOUtils.toString(is, "UTF-8");
        JSONObject context = JSONObject.parseObject(jsonTxt);
        JSONObject regs = context.getJSONObject("regs");

        backend.reg_write(ArmConst.UC_ARM_REG_R0, Long.decode(regs.getString("r0")));
        backend.reg_write(ArmConst.UC_ARM_REG_R1, Long.decode(regs.getString("r1")));
        backend.reg_write(ArmConst.UC_ARM_REG_R2, Long.decode(regs.getString("r2")));
        backend.reg_write(ArmConst.UC_ARM_REG_R3, Long.decode(regs.getString("r3")));
        backend.reg_write(ArmConst.UC_ARM_REG_R4, Long.decode(regs.getString("r4")));
        backend.reg_write(ArmConst.UC_ARM_REG_R5, Long.decode(regs.getString("r5")));
        backend.reg_write(ArmConst.UC_ARM_REG_R6, Long.decode(regs.getString("r6")));
        backend.reg_write(ArmConst.UC_ARM_REG_R7, Long.decode(regs.getString("r7")));
        backend.reg_write(ArmConst.UC_ARM_REG_R8, Long.decode(regs.getString("r8")));
        backend.reg_write(ArmConst.UC_ARM_REG_R9, Long.decode(regs.getString("r9")));
        backend.reg_write(ArmConst.UC_ARM_REG_R10, Long.decode(regs.getString("r10")));
        backend.reg_write(ArmConst.UC_ARM_REG_R11, Long.decode(regs.getString("r11")));
        backend.reg_write(ArmConst.UC_ARM_REG_R12, Long.decode(regs.getString("r12")));

        backend.reg_write(ArmConst.UC_ARM_REG_SP, Long.decode(regs.getString("sp")));
        backend.reg_write(ArmConst.UC_ARM_REG_LR, Long.decode(regs.getString("lr")));
        backend.reg_write(ArmConst.UC_ARM_REG_PC, Long.decode(regs.getString("pc")));
        backend.reg_write(ArmConst.UC_ARM_REG_CPSR, Long.decode(regs.getString("cpsr")));
        backend.reg_write(ArmConst.UC_ARM_REG_FPSCR, Long.decode(regs.getString("fpscr")));

        memory.setStackPoint(Long.decode(regs.getString("sp")));
        backend.enableVFP();

        JSONArray segments = context.getJSONArray("segments");

        for (int i = 0; i < segments.size(); i++) {
            JSONObject segment = segments.getJSONObject(i);
            String path = segment.getString("name");
            long start = segment.getLong("start");
            long end = segment.getLong("end");
            String content_file = segment.getString("content_file");
            JSONObject permissions = segment.getJSONObject("permissions");
            int perms = 0;
            if (permissions.getBoolean("r")){
                perms |= UnicornConst.UC_PROT_READ;
            }
            if (permissions.getBoolean("w")){
                perms |= UnicornConst.UC_PROT_WRITE;
            }
            if (permissions.getBoolean("x")){
                perms |= UnicornConst.UC_PROT_EXEC;
            }

            String[] paths = path.split("/");
            String module_name = paths[paths.length - 1];

            List<String> white_list = Arrays.asList(new String[]{"libkwsgmain.so", "libc.so"});
            if (white_list.contains(module_name)){
                int size = (int)(end - start);

                map_segment(start, size, perms);
                String content_file_path = dump_dir + "\\" + content_file;

                File content_file_f = new File(content_file_path);
                if (content_file_f.exists()){
                    InputStream content_file_is = new FileInputStream(content_file_path);
                    byte[] content_file_buf = IOUtils.toByteArray(content_file_is);

                    // 解压
                    Inflater decompresser = new Inflater();
                    decompresser.setInput(content_file_buf, 0, content_file_buf.length);
                    byte[] result = new byte[size];
                    int resultLength = decompresser.inflate(result);
                    decompresser.end();

                    backend.mem_write(start, result);
                }
                else {
                    System.out.println("not exists path=" + path);
                    byte[] fill_mem = new byte[size];
                    Arrays.fill( fill_mem, (byte) 0 );
                    backend.mem_write(start, fill_mem);
                }

            }
        }

    }

    private void doCommandNative() {

        emulator.traceCode();

        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv());
        DvmObject<?> thiz = vm.resolveClass("com/kuaishou/android/security/internal/dispatch/JNICLibrary").newObject(null);
        list.add(vm.addLocalObject(thiz));
        DvmObject<?> context = vm.resolveClass("com/yxcorp/gifshow/App", vm.resolveClass("android/app/Application")).newObject(null); // context
        vm.addLocalObject(context);
        list.add(10418);
        StringObject urlObj = new StringObject(vm, "/rest/n/encode/androidffdb4a81067a6f7e20afbc3ec76652b3");
        vm.addLocalObject(urlObj);
        ArrayObject arrayObject = new ArrayObject(urlObj);
        vm.addLocalObject(arrayObject);
        StringObject appkey = new StringObject(vm,"d7b7d042-d4f2-4012-be60-d97ff2429c17");
        vm.addLocalObject(appkey);
        DvmInteger intergetobj = DvmInteger.valueOf(vm, -1);
        vm.addLocalObject(intergetobj);
        DvmBoolean boolobj = DvmBoolean.valueOf(vm, false);
        vm.addLocalObject(boolobj);
        StringObject whitestr = new StringObject(vm,"");
        vm.addLocalObject(whitestr);
        ArrayObject objlist = new ArrayObject(arrayObject, appkey, intergetobj, boolobj, context, null, boolobj, whitestr);
        list.add(vm.addLocalObject(objlist));

//        这里获取 dump 时的 pc 地址作为模拟执行起始地址
        long ctx_addr = emulator.getBackend().reg_read(ArmConst.UC_ARM_REG_PC).longValue();
//        开始模拟执行
        Number result = Module.emulateFunction(emulator, ctx_addr + 1, list.toArray());

    }

}
```

发现执行了一条指令就出现了异常，看下错误的地址，转换到 10 进制，打开`DumpContext`文件夹中的`_index.json`，看看是缺少什么内存

![](https://blog.seeflower.dev/images/Snipaste_2022-08-08_00-14-49.png)

对比发现是缺少`[anon:stack_and_tls:19161]`（**图里指错了**），将这个添加到分配内存的白名单中，再次模拟执行

执行了一点指令，但是出现了新的内存未分配错误，同样还是计算下查找是缺少哪块内存，补上再次模拟执行

![](https://blog.seeflower.dev/images/Snipaste_2022-08-08_00-19-26.png)

接下来依次检查，发现需要添加以下内存分段

*   `[anon:.bss]`
*   `[vdso]`
*   `[vvar]`

然后出现了`Invalid instruction (UC_ERR_INSN_INVALID)`异常，说是指令错误

![](https://blog.seeflower.dev/images/Snipaste_2022-08-08_00-24-17.png)

`mrrc p15, #1, r6, r7, c14`这个指令我也不太明白

不过可以看看这个指令是哪块内存的，可以看到就是前面载入的`[vdso]`

![](https://blog.seeflower.dev/images/Snipaste_2022-08-08_00-26-13.png)

查看前面的指令，可以找到明显不一样范围的地址，对比可以知道是`libkwsgmain.so`

![](https://blog.seeflower.dev/images/Snipaste_2022-08-08_00-28-41.png)

根据内存信息，可以计算出偏移是`hex(0xd790a240 - 3616550912) = 0x7240`

原来是调用`gettimeofday`，那我们去 hook 住 libc 的`gettimeofday`函数就行了

![](https://blog.seeflower.dev/images/Snipaste_2022-08-08_00-30-37.png)

添加一个`hook_libc`函数，根据配置文件找到 libc 的基址，把`gettimeofday`函数 hook 上

hook 之后相当于操作就直接做完了，所以要把 LR 写入 PC 在，这样就会直接返回之前的地址继续执行

```
private void hook_libc() {
    long libc_gettimeofday_addr = 3964436480L + 0x33534;
    emulator.getBackend().hook_add_new(new CodeHook() {
        @Override
        public void hook(Backend backend, long address, int size, Object user) {
            System.out.println("gettimeofday");
            UnidbgPointer tv_ptr = emulator.getContext().getPointerArg(0);
            ByteBuffer tv = ByteBuffer.allocate(8);
            tv.order(ByteOrder.LITTLE_ENDIAN);
            tv.putInt(0,1659890020);
            tv.putInt(4, 2891054);
            byte[] data = tv.array();
            tv_ptr.write(0,data,0,8);
            backend.reg_write(ArmConst.UC_ARM_REG_PC, backend.reg_read(ArmConst.UC_ARM_REG_LR).longValue());
        }

        @Override
        public void onAttach(UnHook unHook) {

        }

        @Override
        public void detach() {

        }
    }, libc_gettimeofday_addr, libc_gettimeofday_addr, null);
}
```

继续模拟执行，出现了新的内存异常，发现是需要加载`[anon:scudo:primary]`

然后是缺少`libc++_shared.so`，添加后再次模拟执行，再次出现`Invalid instruction (UC_ERR_INSN_INVALID)`

经过分析发现是调用 libc.so 的 malloc 出现了问题，我们改成自己实现

```
long libc_malloc_addr = 3964436480L + 0x294F8;
emulator.getBackend().hook_add_new(new CodeHook() {
    @Override
    public void hook(Backend backend, long address, int size, Object user) {
        int msize = backend.reg_read(ArmConst.UC_ARM_REG_R0).intValue();
        MemoryBlock block = emulator.getMemory().malloc(msize, true);
        backend.reg_write(ArmConst.UC_ARM_REG_R0, block.getPointer().toUIntPeer());
        backend.reg_write(ArmConst.UC_ARM_REG_PC, backend.reg_read(ArmConst.UC_ARM_REG_LR).longValue());
    }

    @Override
    public void onAttach(UnHook unHook) {

    }

    @Override
    public void detach() {

    }
}, libc_malloc_addr, libc_malloc_addr, null);
```

**free** emm 就啥也不处理，直接返回

```
long libc_free_addr = 3964436480L + 0x29490;
emulator.getBackend().hook_add_new(new CodeHook() {
    @Override
    public void hook(Backend backend, long address, int size, Object user) {
        backend.reg_write(ArmConst.UC_ARM_REG_PC, backend.reg_read(ArmConst.UC_ARM_REG_LR).longValue());
    }

    @Override
    public void onAttach(UnHook unHook) {
    }

    @Override
    public void detach() {
    }
}, libc_free_addr, libc_free_addr, null);
```

还有`pthread_mutex_lock`和`pthread_mutex_unlock`

```
long libc_pthread_mutex_lock_addr = 3964436480L + 0x812EC;
emulator.getBackend().hook_add_new(new CodeHook() {
    @Override
    public void hook(Backend backend, long address, int size, Object user) {
        backend.reg_write(ArmConst.UC_ARM_REG_R0, 0L);
        backend.reg_write(ArmConst.UC_ARM_REG_PC, backend.reg_read(ArmConst.UC_ARM_REG_LR).longValue());
    }

    @Override
    public void onAttach(UnHook unHook) {

    }

    @Override
    public void detach() {

    }
}, libc_pthread_mutex_lock_addr, libc_pthread_mutex_lock_addr, null);


long libc_pthread_mutex_unlock_addr = 3964436480L + 0x8166C;
emulator.getBackend().hook_add_new(new CodeHook() {
    @Override
    public void hook(Backend backend, long address, int size, Object user) {
        backend.reg_write(ArmConst.UC_ARM_REG_R0, 0L);
        backend.reg_write(ArmConst.UC_ARM_REG_PC, backend.reg_read(ArmConst.UC_ARM_REG_LR).longValue());
    }

    @Override
    public void onAttach(UnHook unHook) {

    }

    @Override
    public void detach() {

    }
}, libc_pthread_mutex_unlock_addr, libc_pthread_mutex_unlock_addr, null);
```

接着是`calloc`

```
long libc_caloc_addr = 3964436480L + 0x2943C;
emulator.getBackend().hook_add_new(new CodeHook() {
    @Override
    public void hook(Backend backend, long address, int size, Object user) {
        int num = backend.reg_read(ArmConst.UC_ARM_REG_R0).intValue();
        int msize = backend.reg_read(ArmConst.UC_ARM_REG_R1).intValue();
        MemoryBlock block = emulator.getMemory().malloc(num * msize, true);
        backend.reg_write(ArmConst.UC_ARM_REG_R0, block.getPointer().toUIntPeer());
        backend.reg_write(ArmConst.UC_ARM_REG_PC, backend.reg_read(ArmConst.UC_ARM_REG_LR).longValue());
    }

    @Override
    public void onAttach(UnHook unHook) {

    }

    @Override
    public void detach() {

    }
}, libc_caloc_addr, libc_caloc_addr, null);
```

这次运行了很久，然后出现一个熟悉的需要补 JNI 环境的错误提示

![](https://blog.seeflower.dev/images/Snipaste_2022-08-08_00-55-21.png)

```
@Override
public boolean callBooleanMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
    switch (signature){
        case "java/lang/Boolean->booleanValue()Z":{
            return (boolean) dvmObject.getValue();
//            return false;
        }
    }
    return super.callBooleanMethodV(vm, dvmObject, signature, vaList);
}
```

然后运行得更久了，最后出现一个内存读取错误，但是很奇怪，地址是`0x78`

![](https://blog.seeflower.dev/images/Snipaste_2022-08-08_00-58-09.png)

通过对比计算地址偏移，发现是`libc++_shared.so`调用`libc.so`的`pthread_getspecific`出现了问题

往前回溯，可以发现是`std::__libcpp_snprintf_l`调用`sub_87BB0`出现的异常

![](https://blog.seeflower.dev/images/Snipaste_2022-08-08_01-01-20.png)

粗略分析，可以知道这里就是在做一个字符串格式化，至于前后的线程有关操作，都用 unicorn 跑了那还管什么，直接跳过两个函数的执行

![](https://blog.seeflower.dev/images/Snipaste_2022-08-08_01-04-09.png)

```
long libcpp_patch_addr = 3620732928L + 0x87BB0;
emulator.getBackend().hook_add_new(new CodeHook() {
    @Override
    public void hook(Backend backend, long address, int size, Object user) {
        backend.reg_write(ArmConst.UC_ARM_REG_PC, backend.reg_read(ArmConst.UC_ARM_REG_LR).longValue());
    }

    @Override
    public void onAttach(UnHook unHook) {

    }

    @Override
    public void detach() {

    }
}, libcpp_patch_addr, libcpp_patch_addr, null);
```

出现了新的内存错误，对比发现需要载入`[anon:scudo:secondary]`

再次模拟执行，这次执行了很久 (traceCode 太耗时了)，然后完整完成了执行

![](https://blog.seeflower.dev/images/Snipaste_2022-08-08_01-10-28.png)

去掉`traceCode`，只保留 JNI 相关的日志，可以看到最终拿到了结果

![](https://blog.seeflower.dev/images/Snipaste_2022-08-08_01-12-40.png)

这个结果和前面的 hook 结果对不上，经过测试发现和`gettimeofday`有关，所以这是正常的

完整代码参见：

*   [https://gist.github.com/SeeFlowerX/94b7c52e8bc3d6ddb9108ac5003c6570](https://gist.github.com/SeeFlowerX/94b7c52e8bc3d6ddb9108ac5003c6570)

模拟执行补环境做了以下几件事：

*   遇到内存异常提示，根据异常地址定位需要加载的内存块
*   遇到指令异常提示，回溯分析是什么函数出现了问题，采用 pacth 或者自行实现的方式解决
*   实现了 libc.so 的`gettimeofday malloc free pthread_mutex_lock pthread_mutex_unlock calloc`
*   补充了一个 JNI 调用
*   对`libc++_shared.so`一处函数进行了 patch

这里和上一篇文章提到的略有出入是因为当时测试的传入参数不一样，不过总体都是差不多的

小结：

*   要实现 dump 上下文，首先要过掉 APP 的反调试，要能拿到上下文的内存数据
    
    *   快手似乎没有反调试，但是其他 APP 可能没有这么顺利
*   dump 上下文后模拟执行需要补的环境很少，实现的都是常见的几个函数
*   文章中通过内存异常地址定位需要载入的内存块的方法比较简陋，这一点完全可以改为写代码预先从配置文件中解析各个模块地址范围，再比较地址落在哪个内存块即可
*   文章中手动实现了几个 libc.so 的函数，更好的方案是 unidbg 加载自带的 libc.so，再把模拟执行过程中的 libc 调用关联过去，这样最省力

  
**广告：星球 & 加群交流**  
![](https://blog.seeflower.dev/images/zsxq.png)
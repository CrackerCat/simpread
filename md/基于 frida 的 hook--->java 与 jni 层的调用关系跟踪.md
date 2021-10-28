> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270005.htm)

> 基于 frida 的 hook--->java 与 jni 层的调用关系跟踪

这个文章的前提是假设已经知道 frida 怎么用的情况下使用的。

 

由于论坛中某位大哥的帖子不是很详细，特此在这里做展开，里面用到的代码都是东拼西凑来的，算半原创吧。主要还是希望能用，不喜欢拷贝的不能用的，所以我就修改好了这个，拷贝就能用，不用自己再去哪里找代码。  
代码是从 github 的一个代码中修改的，忘记地址了，GitHub 中搜索 hook art 就能找到了。

 

注意：  
代码中有两处需要修改  
1-- 函数的过滤 indexof 处，  
2--docall 的符号，  
具体怎么改代码注释中有详细说明

 

完整代码 ， 代码就不用看了，直接拷贝黏贴到你的 frida 脚本上，然后愉快的玩耍吧。

```
const STD_STRING_SIZE = 3 * Process.pointerSize;
class StdString {
    constructor() {
        this.handle = Memory.alloc(STD_STRING_SIZE);
    }
 
    dispose() {
        const [data, isTiny] = this._getData();
        if (!isTiny) {
            Java.api.$delete(data);
        }
    }
 
    disposeToString() {
        const result = this.toString();
        this.dispose();
        return result;
    }
 
    toString() {
        const [data] = this._getData();
        return data.readUtf8String();
    }
 
    _getData() {
        const str = this.handle;
        const isTiny = (str.readU8() & 1) === 0;
        const data = isTiny ? str.add(1) : str.add(2 * Process.pointerSize).readPointer();
        return [data, isTiny];
    }
}
 
// PrettyMethod 默认返回是从参数返回的，默认参数增加在第一个参数，然后作为返回值
function prettyMethod(method_id, withSignature) {
    const result = new StdString();
    // art::ArtMethod::PrettyMethod(art::ArtMethod*,bool)
    Java.api['art::ArtMethod::PrettyMethod'](result, method_id, withSignature ? 1 : 0);
    return result.disposeToString();
}
 
//就hook dlopen的打开函数，
//题外话 ， 一般遇到so在软件运行过程中动态载入情况下，可以hook这个两个函数来过滤，
//逮住载入目标so后，就可以hook，目标so的函数了
function hook_dlopen(module_name, fun) {
    var android_dlopen_ext = Module.findExportByName(null, "android_dlopen_ext");
 
    if (android_dlopen_ext) {
        Interceptor.attach(android_dlopen_ext, {
            onEnter: function (args) {
                var pathptr = args[0];
                if (pathptr) {
                    this.path = (pathptr).readCString();
                    if (this.path.indexOf(module_name) >= 0) {
                        this.canhook = true;
                        console.log("android_dlopen_ext:", this.path);
                    }
                }
            },
            onLeave: function (retval) {
                if (this.canhook) {
                    fun();
                }
            }
        });
    }
    var dlopen = Module.findExportByName(null, "dlopen");
    if (dlopen) {
        Interceptor.attach(dlopen, {
            onEnter: function (args) {
                var pathptr = args[0];
                if (pathptr) {
                    this.path = (pathptr).readCString();
                    if (this.path.indexOf(module_name) >= 0) {
                        this.canhook = true;
                        console.log("dlopen:", this.path);
                    }
                }
            },
            onLeave: function (retval) {
                if (this.canhook) {
                    fun();
                }
            }
        });
    }
    console.log("android_dlopen_ext:", android_dlopen_ext, "dlopen:", dlopen);
}
 
 
function hook_java2jni(){
    var module= Process.getModuleByName("libart.so");
    // bool DoCall(ArtMethod* called_method, Thread* self, ShadowFrame& shadow_frame,
    //const Instruction* inst, uint16_t inst_data, JValue* result)
    // DoCall的符号自行去ida动态调试的时候，在module中搜索libart.so，之后到导出函数搜索docall第一个就是
    //或者是用Linux命令查看so文件的符号表，再过滤
    var ducall=module.getExportByName("_ZN3art11interpreter6DoCallILb0ELb1EEEbPNS_9ArtMethodEPNS_6ThreadERNS_11ShadowFrameEPKNS_11InstructionEtPNS_6JValueE");
    Interceptor.attach(ducall,{
        onEnter:function(args){
 
 
            // 调用方
            var addr_ArtMethod = args[2].add(8).readPointer();
            var method_name = prettyMethod(addr_ArtMethod, 0)
 
            // 遇到反调试的话可能会崩，注意，如果争对这个的反调试，就过掉就好了
            //被调用方，这个是敏感点，说白了获取jni地址后，
            //读取内存数据导入ida然后就可以得到解密后的函数了,不过需要很熟悉汇编才行。
            //配合动态调试的话，很容易就能定位到函数的调用
            var addr_Call_ArtMethod = args[0];
            var call_method_name = prettyMethod(addr_Call_ArtMethod, 0)
 
            // 屏蔽过滤掉系统标准函数,这里通过调用方这边过滤，具体根据实际需要过滤
            if (method_name.indexOf("java.") == -1
                && method_name.indexOf("android.") == -1)
            {
                // 这里也可以搞个堆栈打印，
                console.log("[+]DoCall : " + method_name + "--->", call_method_name);
            }
 
        },
        onLeave:function(retval){
 
        }
    })
}
 
function hook_jni2java() {
    var module_libart = Process.findModuleByName("libart.so");
    var symbols = module_libart.enumerateSymbols();
    var ArtMethod_Invoke = null;
    for (var i = 0; i < symbols.length; i++) {
        var symbol = symbols[i];
        var address = symbol.address;
        var name = symbol.name;
        var indexArtMethod = name.indexOf("ArtMethod");
        var indexInvoke = name.indexOf("Invoke");
        var indexThread = name.indexOf("Thread");
        if (indexArtMethod >= 0
            && indexInvoke >= 0
            && indexThread >= 0
            && indexArtMethod < indexInvoke
            && indexInvoke < indexThread) {
            console.log(name);
            ArtMethod_Invoke = address;
        }
    }
    if (ArtMethod_Invoke) {
        Interceptor.attach(ArtMethod_Invoke, {
            onEnter: function (args) {
                var method_name = prettyMethod(args[0], 0);
                if (!(method_name.indexOf("java.") == 0 || method_name.indexOf("android.") == 0)) {
                    // 这里是二次过滤，主要过滤我感兴趣的东西用的，你们自己根据需要修改
                    //if (method_name.indexOf("QueryApPwdTask") != -1
                    //|| method_name.indexOf("WkSecretKeyNativeNew") != -1
                    //|| method_name.indexOf("ConnectJni") != -1
                    //|| method_name.indexOf("loadNatives") != -1
                    //|| method_name.indexOf("SelfSecurityCheck") != -1
                    )
                    {
                        // c++ 层堆栈 -- 可注释掉，忽略，测试用
                        //console.log("ArtMethod Invoke:" + method_name + '  called from:\n' +
                        //Thread.backtrace(this.context, Backtracer.ACCURATE)
                        //   .map(DebugSymbol.fromAddress).join('\n') + '\n');
 
                        // java 堆栈
                        var Log = Java.use("android.util.Log");
                        var Throwable = Java.use("java.lang.Throwable");
                        console.log(Log.getStackTraceString(Throwable.$new()));
                    }
 
                }
            }
        });
    }
}
 
function main() {
    hook_dlopen("libart.so", hook_jni2java);
    hook_java2jni();
}
 
 
 
//这个函数附加的，想hook也行，有时候会看到意想不到的线索。
// 不需要的可以直接忽略，跟本功能没关系
function hook_start(){
    var libcModule=Process.getModuleByName("libc.so");
    var strstr=libcModule.getExportByName("strstr");
    Interceptor.attach(strstr,{
        onEnter:function(args){
            this.arg0=args[0];
            this.arg1=args[1];
 
            this.method_name=ptr(this.arg0).readUtf8String();
            this.call_name=ptr(this.arg1).readUtf8String();
 
            console.log("method_name:"+ this.method_name," call_name : "+ this.call_name );
 
            if(this.call_name.indexOf("WkSecretKeyNativeNew")!=-1){
                console.log("WkSecretKeyNativeNew:"+ this.call_name +" before");
            }
            if(this.call_name.indexOf("JniMethodStart")!=-1){
                console.log("jnimethod:"+ this.call_name +" before");
            }
            if(this.call_name.indexOf("RegisterNativeflag")!=-1){
                console.log("RegisterNative:"+ this.call_name +" before");
            }
        },onLeave:function(retval){
            if (this.call_name.indexOf("JniMethodStart")!=-1 //此处说明代码进入了JniMethodStart，jni函数即将执行
                && this.method_name.indexOf("MainActivity.onCreate")!=-1){ //即将执行的是MainActivity.onCreate
                //retval.replace(0x1);//返回值不为0，让函数sleep，便于IDA附加调试
            }
        }
    })
}
 
setImmediate(main);

```

[[注意] 欢迎加入看雪团队！base 上海，招聘安全工程师、逆向工程师多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)

[#调试逆向](forum-4-1-1.htm) [#系统底层](forum-4-1-2.htm) [#软件保护](forum-4-1-3.htm) [#其他内容](forum-4-1-10.htm)
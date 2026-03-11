> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2095379-1-1.html)

> 此合集只涵盖了部分用途，且是个人与网上资料结合所制 [md]# App 逆向核心 Hook 脚本合集：全面解析与实用技巧本文档整理了 App 逆向过程中常用的 Frida Hook 脚本，涵 ...

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)seventeenJoy 此合集只涵盖了部分用途，且是个人与网上资料结合所制  

App 逆向核心 Hook 脚本合集：全面解析与实用技巧
----------------------------

本文档整理了 App 逆向过程中常用的 Frida Hook 脚本，涵盖反调试绕过、代理检测绕过、数据抓包、加密解密 Hook、脱壳等核心场景，所有脚本均添加详细注释，便于理解和复用。

### 目录

1.  [基础工具命令](#1-基础工具命令)
    
2.  [绕过 App 更新](#2-绕过app更新)
    
3.  [绕过 Frida 反调试](#3-绕过frida反调试)
    
4.  [绕过 App 代理检测](#4-绕过app代理检测)
    
5.  [通用数据 Hook 脚本](#5-通用数据hook脚本)
    
6.  [SO 文件相关 Hook](#6-so文件相关hook)
    
7.  [弹窗 Hook](#7-弹窗hook)
    
8.  [延迟 Hook 技巧](#8-延迟hook技巧)
    
9.  [加密参数 Hook（Key/IV）](#9-加密参数hookkeyiv)
    
10.  [数据类型转换与输出](#10-数据类型转换与输出)
    
11.  [脱壳相关脚本](#11-脱壳相关脚本)
    

### 1. 基础工具命令

#### 1.1 查看前台 Activity

```
# 查看手机当前前台运行的Activity（精准）
adb shell dumpsys activity top
# 备选：筛选Resumed状态的Activity
adb shell dumpsys activity activities | grep "ResumedActivity"

```

#### 1.2 查看 App 进程

```
# 列出手机上所有可被Frida附加的App进程
frida-ps -Ua

```

### 2. 绕过 App 更新

#### 2.1 临时方案（断网 / 关闭代理）

```
# 核心思路：避免App联网检测更新
# 1. 关闭Charles/Proxy代理，打开App进入目标页面后再开启代理
# 2. 断网启动App，进入目标页面后重新联网

```

#### 2.2 永久方案（Hook 更新函数）

```
# 核心思路：Hook强制更新的核心函数，替换为空实现
# 需结合具体App的更新函数名编写Frida脚本，示例逻辑：
# Java.use("com.xxx.UpdateManager").checkUpdate.implementation = function() { return null; }

```

### 3. 绕过 Frida 反调试

#### 3.1 定位导致闪退的 SO 文件

```
// 脚本功能：Hook安卓底层SO加载函数，打印所有加载的SO文件路径
// 适用场景：App检测Frida后闪退，定位检测逻辑所在的SO文件
Java.perform(function () {
    // 安卓底层加载SO的两个核心函数
    var dlopen = Module.findExportByName(null, "dlopen");
    var android_dlopen_ext = Module.findExportByName(null, "android_dlopen_ext");

    // Hook dlopen函数
    Interceptor.attach(dlopen, {
        onEnter: function (args) {
            var path_ptr = args[0];  // 第一个参数是SO文件路径指针
            var path = ptr(path_ptr).readCString();  // 转成字符串路径
            console.log("[dlopen 加载SO]:", path);  // 打印SO路径
        },
        onLeave: function (retval) {}
    });

    // Hook android_dlopen_ext函数（更常用的SO加载函数）
    Interceptor.attach(android_dlopen_ext, {
        onEnter: function (args) {
            var path_ptr = args[0];
            var path = ptr(path_ptr).readCString();
            console.log("[dlopen_ext 加载SO]:", path);
        },
        onLeave: function (retval) {}
    });
});

```

#### 3.2 绕过 SO 文件检测（重写 SO 路径）

```
// 脚本功能：检测到指定SO文件时，重写加载路径为空，阻止其加载
// 适用场景：无法删除SO文件时，绕过检测逻辑
function hook_dlopen() {
    Interceptor.attach(Module.findExportByName(null, "android_dlopen_ext"), {
        onEnter: function (args) {
            var pathptr = args[0];
            if (pathptr !== undefined && pathptr != null) {
                var path = ptr(pathptr).readCString();
                // 匹配目标SO文件（示例：libmsaoaidsec.so）
                if (path.indexOf('libmsaoaidsec') >= 0) {
                    ptr(pathptr).writeUtf8String("");  // 重写路径为空字符串
                    console.log("已阻止加载检测SO：libmsaoaidsec.so");
                }
            }
        }
    });
}

// 执行Hook
Java.perform(hook_dlopen);

```

### 4. 绕过 App 代理检测

#### 4.1 核心思路

```
# 问题：配置代理后仅能抓到静态资源，无法抓到动态接口请求
# 解决方案：抓底层Socket包，借助SocksDroid转发流量

```

#### 4.2 操作步骤

```
# 1. 手机安装SocksDroid App
# 2. Charles配置：Proxy -> Proxy Settings -> SOCKS Proxy（开启，端口自定义）
# 3. SocksDroid配置：填写Charles的IP和SOCKS端口，开启转发
# 4. 手机关闭系统代理，仅保留SocksDroid转发

```

### 5. 通用数据 Hook 脚本

#### 5.1 Hook Map.put 方法（拦截键值对）

```
// 脚本功能：Hook TreeMap/HashMap的put方法，打印指定Key的Value
// 适用场景：App通过Map存储请求参数/加密数据时，拦截关键数据
Java.perform(function () {
    // 选择目标Map类型（TreeMap/HashMap根据实际情况替换）
    var TreeMap = Java.use('java.util.TreeMap');
    // 重写put方法
    TreeMap.put.implementation = function (key, value) {
        // 过滤指定Key（示例：data）
        if (key == "data") {
            console.log("[Map.put] Key=data, Value=", value);
        }
        // 执行原方法，保证业务正常
        var res = this.put(key, value);
        return res;
    };
});

```

#### 5.2 Hook StringBuilder（拦截字符串拼接）

```
// 脚本功能：Hook字符串拼接类，打印最终拼接结果
// 适用场景：App通过StringBuilder拼接加密字符串/请求参数
Java.perform(function () {
    var StringBuilder = Java.use("java.lang.StringBuilder");
    // 重写toString方法（拼接完成后调用）
    StringBuilder.toString.implementation = function () {
        var res = this.toString();  // 获取拼接后的字符串
        console.log("[StringBuilder 拼接结果]:", res);
        return res;
    };
});

```

#### 5.3 Hook Base64 加密

```
// 脚本功能：Hook安卓原生Base64加密方法，打印加密结果
// 适用场景：拦截App中的Base64加密数据
Java.perform(function () {
    var Base64 = Java.use("android.util.Base64");
    // 重写encodeToString方法（字节数组转Base64字符串）
    Base64.encodeToString.overload('[B', 'int').implementation = function (bArr, val) {
        var res = this.encodeToString(bArr, val);
        console.log("[Base64 加密结果]:", res);
        return res;
    };
});

```

#### 5.4 Hook OkHttp 拦截器

```
// 脚本功能：Hook OkHttp的拦截器添加方法，打印所有拦截器信息
// 适用场景：分析App的网络请求拦截逻辑，定位参数加密位置
Java.perform(function () {
    var Builder = Java.use('okhttp3.OkHttpClient$Builder');
    // 重写addInterceptor方法
    Builder.addInterceptor.implementation = function (inter) {
        // 打印拦截器详情（JSON格式化）
        console.log("[OkHttp 拦截器]:", JSON.stringify(inter));
        return this.addInterceptor(inter);
    };
});
// 执行命令：frida -Uf com.hupu.shihuo -l hook_Interceptor.js -o all_interceptor.txt

```

#### 5.5 Hook NewStringUTF（底层字符串创建）

```
// 脚本功能：Hook安卓底层字符串创建函数，拦截指定长度的字符串
// 适用场景：App通过JNI创建加密字符串，上层Java层无法Hook时使用
Java.perform(function () {
    // 1. 查找libart.so中的NewStringUTF函数地址
    var artModule = Process.getModuleByName("libart.so");
    var symbols = artModule.enumerateSymbols();
    var addrNewStringUTF = null;

    for (var i = 0; i < symbols.length; i++) {
        var symbol = symbols[i];
        // 匹配NewStringUTF（排除CheckJNI版本）
        if (symbol.name.indexOf("NewStringUTF") >= 0 && symbol.name.indexOf("CheckJNI") < 0) {
            addrNewStringUTF = symbol.address;
            console.log("NewStringUTF 地址:", symbol.address, symbol.name);
            break;  // 找到第一个即可
        }
    }

    // 2. Hook NewStringUTF函数
    if (addrNewStringUTF != null) {
        Interceptor.attach(addrNewStringUTF, {
            onEnter: function (args) {
                var c_string = args[1];  // 第一个参数是C字符串指针
                var dataString = c_string.readCString();  // 转成可读字符串
                // 过滤指定长度的字符串（示例：32位）
                if (dataString && dataString.length === 32) {
                    console.log("[NewStringUTF] 字符串:", dataString);
                    // 打印调用栈（定位字符串来源）
                    console.log("调用栈:\n", Thread.backtrace(this.context, Backtracer.ACCURATE).map(DebugSymbol.fromAddress).join('\n'));
                }
            }
        });
    }
});

```

#### 5.6 Hook JNI 动态注册（Java-C 函数映射）

```
// 脚本功能：Hook JNI动态注册函数，打印Java方法与C函数的映射关系
// 适用场景：分析App的JNI函数调用，定位加密逻辑所在的C函数
function hook_RegisterNatives() {
    // 1. 查找libart.so中的RegisterNatives函数
    var symbols = Module.enumerateSymbolsSync("libart.so");
    var addrRegisterNatives = null;

    for (var i = 0; i < symbols.length; i++) {
        var symbol = symbols[i];
        // 匹配RegisterNatives（排除CheckJNI版本）
        if (symbol.name.indexOf("art") >= 0 &&
            symbol.name.indexOf("JNI") >= 0 &&
            symbol.name.indexOf("RegisterNatives") >= 0 &&
            symbol.name.indexOf("CheckJNI") < 0) {
            addrRegisterNatives = symbol.address;
            console.log("RegisterNatives 地址:", symbol.address, symbol.name);
            break;
        }
    }

    // 2. Hook RegisterNatives函数
    if (addrRegisterNatives != null) {
        Interceptor.attach(addrRegisterNatives, {
            onEnter: function (args) {
                var env = args[0];  // JNIEnv指针
                var java_class = args[1];  // Java类指针
                var class_name = Java.vm.tryGetEnv().getClassName(java_class);  // 获取Java类名

                // 过滤目标类（示例：B站Native库）
                var target_class = "com.bilibili.nativelibrary.LibBili";
                if (class_name === target_class) {
                    console.log("\n[RegisterNatives] 目标类:", class_name);
                    console.log("[RegisterNatives] 方法数量:", args[3]);

                    var methods_ptr = ptr(args[2]);  // JNINativeMethod数组指针
                    var method_count = parseInt(args[3]);  // 方法数量

                    // 遍历所有Java-C映射方法
                    for (var i = 0; i < method_count; i++) {
                        // 读取JNINativeMethod的三个字段：方法名、签名、C函数指针
                        var name_ptr = Memory.readPointer(methods_ptr.add(i * Process.pointerSize * 3));
                        var sig_ptr = Memory.readPointer(methods_ptr.add(i * Process.pointerSize * 3 + Process.pointerSize));
                        var fnPtr_ptr = Memory.readPointer(methods_ptr.add(i * Process.pointerSize * 3 + Process.pointerSize * 2));

                        // 转成可读字符串
                        var name = Memory.readCString(name_ptr);  // Java方法名
                        var sig = Memory.readCString(sig_ptr);    // 方法签名
                        var find_module = Process.findModuleByAddress(fnPtr_ptr);  // C函数所在SO
                        var offset = ptr(fnPtr_ptr).sub(find_module.base);  // C函数偏移量

                        // 打印映射关系
                        console.log("Java方法名:", name, 
                                  "签名:", sig, 
                                  "SO文件:", find_module.name, 
                                  "偏移量:", offset);
                    }
                }
            }
        });
    }
}

// 立即执行Hook
setImmediate(hook_RegisterNatives);

```

### 6. SO 文件相关 Hook

#### 6.1 按特征 Hook SO 中的字符串

```
// 脚本功能：Hook NewStringUTF，拦截包含指定特征的字符串，定位所属SO
// 适用场景：已知加密字符串特征，定位其生成的SO文件
Java.perform(function () {
    var symbols = Module.enumerateSymbolsSync("libart.so");
    var addrNewStringUTF = null;

    // 查找NewStringUTF函数
    for (var i = 0; i < symbols.length; i++) {
        var symbol = symbols[i];
        if (symbol.name.indexOf("NewStringUTF") >= 0 && symbol.name.indexOf("CheckJNI") < 0) {
            addrNewStringUTF = symbol.address;
            console.log("NewStringUTF 地址:", symbol.address, symbol.name);
            break;
        }
    }

    // Hook并过滤特征字符串
    if (addrNewStringUTF != null) {
        Interceptor.attach(addrNewStringUTF, {
            onEnter: function (args) {
                var c_string = args[1];
                var dataString = c_string.readCString();
                // 匹配特征字符串（示例：XYAAAAAQAAAAEAAAB）
                if (dataString && dataString.indexOf("XYAAAAAQAAAAEAAAB") != -1) {
                    console.log("[特征字符串]:", dataString);
                    // 打印调用栈（定位SO文件）
                    console.log("调用栈:\n", Thread.backtrace(this.context, Backtracer.ACCURATE).map(DebugSymbol.fromAddress).join('\n'));
                }
            }
        });
    }
});

// 执行命令示例：
// frida -U -f 包名 -l 1-hook-NewStringUTF.js  // spawn模式（自动打开App）
// frida -UF -l 1-hook-NewStringUTF.js        // attach模式（App已打开）

```

### 7. 弹窗 Hook

```
 开始Hook所有弹窗...");    // 1. Hook AlertDialog（最常见的弹窗）    var AlertDialogBuilder = Java.use("android.app.AlertDialog$Builder");    AlertDialogBuilder.show.implementation = function () {        logCallStack("AlertDialog.Builder.show");        return this.show.apply(this, arguments);    };    // 2. Hook 自定义Dialog    var Dialog = Java.use("android.app.Dialog");    Dialog.show.implementation = function () {        var className = this.getClass().getName();        // 过滤系统弹窗（如权限请求）        if (!className.startsWith("android.") && !className.startsWith("com.android.")) {            logCallStack("自定义Dialog.show: " + className);        }        return this.show.apply(this, arguments);    };    // 3. Hook Toast（轻量级提示）    var Toast = Java.use("android.widget.Toast");    Toast.show.implementation = function () {        try {            var text = this.getText() ? this.getText().toString() : "N/A";            console.log("\n[+] Toast内容:", text);            logCallStack("Toast.show");        } catch (e) {}        return this.show.apply(this, arguments);    };    // 4. Hook Snackbar（Material Design提示）    try {        var Snackbar = Java.use("com.google.android.material.snackbar.Snackbar");        Snackbar.show.implementation = function () {            logCallStack("Snackbar.show");            return this.show.apply(this, arguments);        };    } catch (e) {        console.log("[-] App未使用Snackbar，跳过Hook");    }    // 工具函数：打印调用栈（过滤系统类，只显示App代码）    function logCallStack(tag) {        console.log("\n[+] " + tag + " 被调用！");        console.log("调用栈:");        try {            var Exception = Java.use("java.lang.Exception");            var exc = Exception.$new();            var stack = exc.getStackTrace();            // 只打印前20行调用栈            for (var i = 0; i < Math.min(stack.length, 20); i++) {                var line = stack[i].toString();                // 过滤系统类，高亮App代码                if (line.includes("com.denglin.duck") ||                    (!line.includes("android.") && !line.includes("java.") &&                      !line.includes("dalvik.") && !line.includes("sun."))) {                    console.log("  >>> " + line);  // App代码（高亮）                } else {                    console.log("      " + line);  // 系统代码（普通）                }            }        } catch (e) {            console.log("[-] 获取调用栈失败:", e.message);        }    }});

```

### 8. 延迟 Hook 技巧

#### 8.1 加载 SO 后再 Hook

```
// 脚本功能：检测到目标SO加载完成后，再执行Hook逻辑
// 适用场景：SO文件延迟加载，直接Hook会提示找不到函数
function do_hook() {
    // 目标Hook逻辑（示例：Hook libkeyinfo.so的getByteHash函数）
    var addr = Module.findExportByName("libkeyinfo.so", "getByteHash");
    console.log("getByteHash 地址:", addr);
    Interceptor.attach(addr, {
        onEnter: function (args) {
            this.x1 = args[2];  // 保存第二个参数
        },
        onLeave: function (retval) {
            console.log("--------------------");
            console.log("参数值:", Memory.readCString(this.x1));
            console.log("返回值:", Memory.readCString(retval));
        }
    });
}

function load_so_and_hook() {
    // Hook SO加载函数
    var dlopen = Module.findExportByName(null, "dlopen");
    var android_dlopen_ext = Module.findExportByName(null, "android_dlopen_ext");

    // Hook dlopen
    Interceptor.attach(dlopen, {
        onEnter: function (args) {
            var path_ptr = args[0];
            var path = ptr(path_ptr).readCString();
            this.path = path;
        },
        onLeave: function (retval) {
            // 检测到目标SO加载完成
            if (this.path.indexOf("libkeyinfo.so") !== -1) {
                console.log("[dlopen] 目标SO已加载:", this.path);
                do_hook();  // 执行Hook逻辑
            }
        }
    });

    // Hook android_dlopen_ext
    Interceptor.attach(android_dlopen_ext, {
        onEnter: function (args) {
            var path_ptr = args[0];
            var path = ptr(path_ptr).readCString();
            this.path = path;
        },
        onLeave: function (retval) {
            if (this.path.indexOf("libkeyinfo.so") !== -1) {
                console.log("[android_dlopen_ext] 目标SO已加载:", this.path);
                do_hook();
            }
        }
    });
}

// 启动Hook
load_so_and_hook();

// 执行命令：frida -U -f com.achievo.vipshop -l delay_hook.js

```

#### 8.2 时间延迟 Hook

```
// 脚本功能：延迟指定时间后执行Hook，适用于SO/类延迟加载的场景
// 适用场景：上述方法仍Hook不到时，强制延迟
function do_hook() {
    // 延迟10ms执行Hook（可调整时间）
    setTimeout(function () {
        Java.perform(function () {
            // 示例：Hook小红书的网络拦截器
            var XhsHttpInterceptor = Java.use('com.xingin.shield.http.XhsHttpInterceptor');
            XhsHttpInterceptor.initialize.implementation = function (str) {
                console.log("initialize 参数:", str);
                return this.initialize(str);
            };
        });
    }, 10);
}

// 或直接延迟执行（示例：延迟1秒）
setTimeout(function() {
    // 此处写Hook逻辑
    console.log("延迟1秒后执行Hook");
}, 1000);

// 带参数的延迟Hook
setTimeout((x, y)=>{
    console.log("延迟3秒，参数和:", x + y);  // 输出3
}, 3000, 1, 2);

```

#### 8.3 通用延迟 Hook 封装

```
// 脚本功能：封装延迟Hook逻辑，传入SO名和Hook函数即可
// 适用场景：复用延迟Hook逻辑，减少重复代码
function do_hook() {
    // 此处编写具体的Hook逻辑
    console.log("执行目标Hook逻辑");
}

function delay_hook(so_name, hook_func) {
    var dlopen = Module.findExportByName(null, "dlopen");
    var android_dlopen_ext = Module.findExportByName(null, "android_dlopen_ext");

    // Hook dlopen
    Interceptor.attach(dlopen, {
        onEnter: function (args) {
            var path_ptr = args[0];
            var path = ptr(path_ptr).readCString();
            this.path = path;
        },
        onLeave: function (retval) {
            if (this.path.indexOf(so_name) !== -1) {
                console.log("[dlopen] 加载SO:", this.path);
                hook_func();  // 执行Hook
            }
        }
    });

    // Hook android_dlopen_ext
    Interceptor.attach(android_dlopen_ext, {
        onEnter: function (args) {
            var path_ptr = args[0];
            var path = ptr(path_ptr).readCString();
            this.path = path;
        },
        onLeave: function (retval) {
            if (this.path.indexOf(so_name) !== -1) {
                console.log("[android_dlopen_ext] 加载SO:", this.path);
                hook_func();
            }
        }
    });
}

// 调用示例：Hook libkeyinfo.so
delay_hook("libkeyinfo.so", do_hook);
// 执行命令：frida -U -f com.achievo.vipshop -l hook.js

```

### 9. 加密参数 Hook（Key/IV）

```
// 脚本功能：Hook AES/DES加密的Key和IV参数，打印明文/密文
// 适用场景：分析App的对称加密逻辑，获取加密密钥
Java.perform(function () {
    // Hook Cipher初始化方法（Key+IV）
    var Cipher = Java.use("javax.crypto.Cipher");
    var IvParameterSpec = Java.use("javax.crypto.spec.IvParameterSpec");
    var ByteString = Java.use("com.android.okhttp.okio.ByteString");  // 字节数组转字符串工具

    // 重写Cipher.init方法（模式+Key+IV）
    Cipher.init.overload('int', 'java.security.Key', 'java.security.spec.AlgorithmParameterSpec').implementation = function (mode, key, iv) {
        console.log("\n-----------------------Cipher.init---------------------------");
        console.log("加密模式:", mode);  // 1=加密，2=解密
        console.log("Key(字节数组):", key.getEncoded());
        console.log("Key(十六进制):", ByteString.of(key.getEncoded()).hex());
        console.log("IV(字节数组):", Java.cast(iv, IvParameterSpec).getIV());
        console.log("IV(十六进制):", ByteString.of(Java.cast(iv, IvParameterSpec).getIV()).hex());
        // 执行原方法
        this.init(mode, key, iv);
    };

    // 重写Cipher.doFinal方法（加密/解密数据）
    Cipher.doFinal.overload('[B').implementation = function (data) {
        console.log("\n-----------------------AES加密/解密---------------------------");
        console.log("明文(UTF8):", ByteString.of(data).utf8());
        var res = this.doFinal(data);  // 执行加密/解密
        console.log('密文(十六进制):', ByteString.of(res).hex());
        return res;
    };
});

```

### 10. 数据类型转换与输出

#### 10.1 Map 转字符串输出

```
// 脚本功能：将Map对象转成可读字符串，避免直接输出[object Object]
Java.perform(function () {
    let RequestUtils = Java.use("com.shizhuang.duapp.common.utils.RequestUtils");
    RequestUtils["c"].implementation = function (map, j2) {
        // 类型转换：Map -> HashMap（根据实际类型调整）
        let Map = Java.use('java.util.HashMap'); 
        let obj = Java.cast(map, Map);
        // 打印参数
        console.log("map参数类型:", JSON.stringify(map));
        console.log("map参数值:", obj);
        // 执行原方法
        let ret = this.c(map, j2);
        console.log("方法返回值:", ret);
        return ret;
    };
});

```

#### 10.2 字节数组转十六进制字符串

```
// 脚本功能：将字节数组转成十六进制字符串，便于查看加密数据
Java.perform(function () {
    let d = Java.use("tv.danmaku.biliplayerimpl.report.heartbeat.d");
    var ByteString = Java.use('com.android.okhttp.okio.ByteString');  // 工具类
    d["H7"].implementation = function (arg1) {
        let ret = this.H7(arg1);
        console.log('原始返回值:', ret);  // 字节数组：[79,-90,...]
        console.log('十六进制返回值:', ByteString.of(ret).hex());
        return ret;
    };
});

```

#### 10.3 字节数组转 UTF8 字符串

```
// 脚本功能：字节数组转UTF8字符串（适用于明文数据）
var ByteString = Java.use('com.android.okhttp.okio.ByteString');
// 示例：打印返回值的UTF8字符串
console.log("明文:", ByteString.of(res).utf8());

```

#### 10.4 查看对象所属类名

```
// 脚本功能：输出对象的实际类名，定位数据类型
// 示例：bVar是目标对象
console.log("对象类名:", JSON.stringify(bVar));
// 输出示例："<instance: com.bilibili.api.a$b, $className: tv.danmaku.bili.utils.p$a>"
// 关键信息：$className 是实际类名

```

#### 10.5 控制输出长度（避免刷屏）

```
// 脚本功能：限制输出长度，只打印前16字节
console.log('getMD5返回值:', hexdump(retval, {length: 16}), "\n");

```

### 11. 脱壳相关脚本

#### 11.1 枚举 DexClassLoader（定位加载的 Dex）

```
// 脚本功能：Hook DexClassLoader，打印加载的Dex文件路径
// 适用场景：分析加壳App的Dex加载逻辑，定位脱壳点
function enumerateClassLoader() {
    Java.perform(function () {
        var dexclassLoader = Java.use("dalvik.system.DexClassLoader");
        // 重写DexClassLoader构造方法
        dexclassLoader.$init.implementation = function (dexPath, optimizedDirectory, librarySearchPath, parent) {
            console.log("\n-----------------------------------------");
            console.log("DexClassLoader实例:", JSON.stringify(this));
            console.log("Dex文件路径:", dexPath);
            console.log("优化目录:", optimizedDirectory);
            console.log("SO搜索路径:", librarySearchPath);
            console.log("父ClassLoader:", parent);
            // 执行原构造方法
            this.$init(dexPath, optimizedDirectory, librarySearchPath, parent);
        };
    });
}

// 立即执行
setImmediate(enumerateClassLoader);
// 执行命令：frida -U -f com.nb.loaderdemo -l loader.js

```

#### 11.2 读取 DexPathList 中的 Dex 文件

```
// 脚本功能：深入读取DexClassLoader的DexPathList，获取所有加载的Dex
Java.perform(function () {
    var dexclassLoader = Java.use("dalvik.system.DexClassLoader");
    dexclassLoader.$init.implementation = function (dexPath, optimizedDirectory, librarySearchPath, parent) {
        this.$init(dexPath, optimizedDirectory, librarySearchPath, parent);

        console.log("\n1. 当前DexClassLoader:", this);

        // 1. 获取BaseDexClassLoader的pathList字段
        var cls = this.getClass().getSuperclass().getSuperclass();  // 父类的父类
        var field1 = cls.getDeclaredField("pathList");
        field1.setAccessible(true);  // 突破私有字段限制
        var pathList = field1.get(this);
        var realPathList = Java.cast(pathList, Java.use("dalvik.system.DexPathList"));
        console.log("2. pathList字段:", realPathList);

        // 2. 获取所有Dex文件路径
        console.log("3. 所有Dex文件路径:");
        var dexPathArrayList = realPathList.getDexPaths();
        var realDexPathArrayList = Java.cast(dexPathArrayList, Java.use("java.util.ArrayList"));
        for (var i = 0; i < realDexPathArrayList.size(); i++) {
            console.log("\t", realDexPathArrayList.get(i));
        }

        // 3. 获取dexElements数组（核心Dex数组）
        var clsDexPathList = Java.use("dalvik.system.DexPathList");
        var field2 = clsDexPathList.class.getDeclaredField("dexElements");
        field2.setAccessible(true);
        var dexElements = field2.get(realPathList);
        var elementArray = Java.cast(dexElements, Java.use("[Ldalvik.system.DexPathList$Element;"));
        console.log("4. dexElements数组:", elementArray);

        // 4. 遍历dexElements，打印每个Dex文件
        console.log("5. dexElements中的Dex文件:");
        var ArrayClz = Java.use("java.lang.reflect.Array");
        var len = ArrayClz.getLength(elementArray);
        for (var i = 0; i < len; i++) {
            var elementObject = ArrayClz.get(elementArray, i);
            var element = Java.cast(elementObject, Java.use("dalvik.system.DexPathList$Element"));
            var dexFile = element.dexFile.value;
            var mFileName = dexFile.mFileName.value;
            console.log("\t", mFileName);
        }
    };
});

```

#### 11.3 FART 脱壳辅助脚本

```
// 脚本功能：主动调用FART的脱壳方法，支持自定义ClassLoader
// 适用场景：加壳App使用自定义ClassLoader加载Dex，FART默认脱壳失败
Java.perform(function () {
    // Hook DexClassLoader构造方法
    var dexclassLoader = Java.use("dalvik.system.DexClassLoader");
    dexclassLoader.$init.implementation = function (dexPath, optimizedDirectory, librarySearchPath, parent) {
        this.$init(dexPath, optimizedDirectory, librarySearchPath, parent);
        // 主动调用FART的脱壳方法，传入自定义ClassLoader
        var ActivityThread = Java.use("android.app.ActivityThread");
        ActivityThread.fartwithClassloader(this);
        console.log("已触发FART脱壳，ClassLoader:", this);
    };
});

// 执行命令：frida -U -f com.nb.loaderdemo2 -l 5.call_classloader.js

```

#### 11.4 Frida-dexdump 脱壳（命令行）

```
# 功能：自动脱壳并导出Dex文件
# 执行命令：
frida-dexdump -U -f [App包名]
# 示例：frida-dexdump -U -f com.xxx.app

# 后续处理：
# 1. 导出的Dex文件默认保存在「包名」命名的文件夹中
# 2. 用adb将Dex文件传到手机：adb push 文件夹名 /storage/emulated/0/
# 3. 使用MT管理器修复Dex（删除异常文件，执行Dex修复）

```

### 备注

1.  所有脚本需根据目标 App 的实际类名 / 方法名 / SO 名调整后使用；
    
2.  Frida 脚本执行前需确保手机已连接 Frida 服务（`frida-server`运行中）；
    
3.  脱壳后的 Dex 文件可能需要修复（如使用 dexfixer），才能正常反编译。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)seventeenJoy _ 本帖最后由 seventeenJoy 于 2026-3-9 21:03 编辑_  
更正弹窗 Hook 相关的代码  
[JavaScript] _纯文本查看_ _复制代码_

```
// 脚本功能：Hook App中所有弹窗（AlertDialog/Toast/Snackbar），打印调用栈
// 适用场景：定位弹窗触发逻辑，分析App的提示/验证逻辑
Java.perform(function () {
    console.log(" 开始Hook所有弹窗...");
 
    // 1. Hook AlertDialog（最常见的弹窗）
    var AlertDialogBuilder = Java.use("android.app.AlertDialog$Builder");
    AlertDialogBuilder.show.implementation = function () {
        logCallStack("AlertDialog.Builder.show");
        return this.show.apply(this, arguments);
    };
 
    // 2. Hook 自定义Dialog
    var Dialog = Java.use("android.app.Dialog");
    Dialog.show.implementation = function () {
        var className = this.getClass().getName();
        // 过滤系统弹窗（如权限请求）
        if (!className.startsWith("android.") && !className.startsWith("com.android.")) {
            logCallStack("自定义Dialog.show: " + className);
        }
        return this.show.apply(this, arguments);
    };
 
    // 3. Hook Toast（轻量级提示）
    var Toast = Java.use("android.widget.Toast");
    Toast.show.implementation = function () {
        try {
            var text = this.getText() ? this.getText().toString() : "N/A";
            console.log("\n[+] Toast内容:", text);
            logCallStack("Toast.show");
        } catch (e) {}
        return this.show.apply(this, arguments);
    };
 
    // 4. Hook Snackbar（Material Design提示）
    try {
        var Snackbar = Java.use("com.google.android.material.snackbar.Snackbar");
        Snackbar.show.implementation = function () {
            logCallStack("Snackbar.show");
            return this.show.apply(this, arguments);
        };
    } catch (e) {
        console.log("[-] App未使用Snackbar，跳过Hook");
    }
 
    // 工具函数：打印调用栈（过滤系统类，只显示App代码）
    function logCallStack(tag) {
        console.log("\n[+] " + tag + " 被调用！");
        console.log("调用栈:");
        try {
            var Exception = Java.use("java.lang.Exception");
            var exc = Exception.$new();
            var stack = exc.getStackTrace();
            // 只打印前20行调用栈
            for (var i = 0; i < Math.min(stack.length, 20); i++) {
                var line = stack[i].toString();
                // 过滤系统类，高亮App代码
                if (line.includes("com.denglin.duck") ||
                    (!line.includes("android.") && !line.includes("java.") &&
                     !line.includes("dalvik.") && !line.includes("sun."))) {
                    console.log("  >>> " + line);  // App代码（高亮）
                } else {
                    console.log("      " + line);  // 系统代码（普通）
                }
            }
        } catch (e) {
            console.log("[-] 获取调用栈失败:", e.message);
        }
    }
});

```

另外我们可以使用算法助手来勾选应用，进行 log 打印和弹窗检测并打印调用栈 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) v89989898 感谢分享，收藏了 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) blueluckycard 感谢，大神，收藏一下 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) Dc1c1c1 ![](https://static.52pojie.cn/static/image/smiley/default/42.gif) 点赞支持，已收藏 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) njlz2012 谢谢分享，先收藏了，后面学 frida hook 来用 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) Galaxyou 刚好需要，我还寻思发帖子问下各位大佬呢![](https://static.52pojie.cn/static/image/smiley/default/lol.gif) ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) rainerosion 感谢大神分享。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)yushangwl 感谢大佬，先收藏备用。。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)西瓜丶 感谢，已收藏学习 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) schip 非常实用的文章，感谢分享！
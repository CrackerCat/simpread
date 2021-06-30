> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268301.htm)

> [原创] 某知笔记服务端 docker 镜像授权分析

启动 docker 镜像
------------

根据其官网 [https://www.wiz.cn/zh-cn/docker](https://www.wiz.cn/zh-cn/docker) 说明, 使用如下命令即可启动

```
docker pull wiznote/wizserver:latest
docker run --name wiz --restart=always -it -d \
   -v `pwd`/wizdata:/wiz/storage \
   -v /etc/localtime:/etc/localtime \
   -p 80:80 -p 9269:9269/udp  wiznote/wizserver

```

初探授权
----

*   启动好之后，默认授权限制只能有 5 个用户。
    
    ![](https://bbs.pediy.com/upload/attach/202106/353809_8U5YEK2E4892XP3.png)
    
*   进入 docker 容器，查看基本信息。
    
    执行 `docker exec -it wiz bash` 进入容器，可以看到
    
    *   后端主要使用 node 实现并用 pm2 管理，nginx 反代，数据库 mysql
    *   node 版本 v8.11.2，v8 版本 6.2.414.54
        
        ```
        # ps -aef
        UID        PID  CMD
        root         1  bash /wiz/app/entrypoint.sh
        root        33  /usr/bin/redis-server 127.0.0.1:6379
        mysql       53  /usr/sbin/mysqld
        root        59  nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
        nginx       60  nginx: worker process
        nginx       61  nginx: worker process
        root       119  PM2 v4.5.0: God Daemon (/root/.pm2)
        root       129  node /root/.pm2/modules/pm2-logrotate/node_modules/pm2-logrotate/app.js
        root       137  node /wiz/app/wizserver/app.js
        root       157  node /wiz/app/wizserver/app.js
        root       173  node /wiz/app/wizserver/app.js
        root       189  node /wiz/app/wizserver/app.js
        root       205  node /wiz/app/wizserver/app.js
        root       221  node /wiz/app/wizserver/app.js
        root       240  node /wiz/app/wizserver/app.js
        root       246  crond -n
        root       256  bash
        root       270  ps -aef
        # node -v
        v8.11.2
        # node -p process.versions.v8
        6.2.414.54
        # pm2 list
        ┌─────┬──────────────────┬
        │ id  │ name             │
        ├─────┼──────────────────┼
        │ 1   │ as               │
        │ 6   │ cloud            │
        │ 4   │ index            │
        │ 2   │ note             │
        │ 7   │ office           │
        │ 5   │ search           │
        │ 3   │ ws               │
        └─────┴──────────────────┴
        
        ```
        
*   分析代码发现后缀为 jsc 的文件
    
    ```
    # ll /wiz/app/wizserver/common/
    total 536
    -rw-r--r-- 1 root root  8792 Dec 23  2020 alicloud_tools.jsc
    -rw-r--r-- 1 root root  9984 Dec 23  2020 client_tools.jsc
    -rw-r--r-- 1 root root  7024 Dec 23  2020 cookie_tools.jsc
    -rw-r--r-- 1 root root  2352 Dec 23  2020 date_tools.jsc
    -rw-r--r-- 1 root root 27528 Dec 23  2020 db_tools.jsc
    ......
    
    ```
    
    进一步分析发现，这是通过 bytenode[https://github.com/bytenode/bytenode](https://github.com/bytenode/bytenode) 工具编译成了 v8 的字节码。
    
    在各种搜索之后发现，目前还没有相关的反汇编、反编译工具。
    
    唯一相关项目 [https://github.com/PositiveTechnologies/ghidra_nodejs](https://github.com/PositiveTechnologies/ghidra_nodejs) 是 GHIDRA 的插件，但 node 版本和我们不匹配
    

[](#反汇编v8字节码，改造d8)反汇编 v8 字节码，改造 d8
----------------------------------

通过 v8 源码分析发现，v8 本身是有反汇编功能，node 可通过`--print-bytecode`参数开启。

 

但只能反汇编源码，无法反汇编 jsc 文件。为了反汇编字节码只能通过修改 v8 来实现。

 

d8 [https://v8.dev/docs/d8](https://v8.dev/docs/d8) 是 v8 的开发工具，为了简单起见决定对 d8 进行修改。

### 反汇编 jsc 思路

*   jsc 实际是由`v8::internal::CodeSerializer::Serialize`方法生成
*   反汇编需要调用`v8::internal::BytecodeArray::Disassemble`方法生成

反序列`v8::internal::CodeSerializer::Deserialize`原型

```
MUST_USE_RESULT static MaybeHandle Deserialize(
    Isolate* isolate, ScriptData* cached_data, Handle source); 
```

其中参数`cached_data`，可通过 ScriptData 的构造函数`ScriptData(const byte* data, int length);`构造

 

其中返回值`SharedFunctionInfo`对象有`bytecode_array`方法，可以获得 BytecodeArray 来进行反汇编

```
BytecodeArray* SharedFunctionInfo::bytecode_array() const {
  DCHECK(HasBytecodeArray());
  return BytecodeArray::cast(function_data());
}

```

于是思路自然而然就有了

*   读取 jsc，构造`ScriptData`对象
*   反序列化，获取`SharedFunctionInfo`对象
*   反汇编，通过`bytecode_array`获取`BytecodeArray`, 并调用`Disassemble`反汇编

### 搭建 v8 编译环境

```
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
export PATH=`pwd`/depot_tools:"$PATH"
export HTTPS_PROXY=http://172.31.0.1:18080  # 设置代理
gclient sync                                # 更新工具链
fetch v8                                    # 获取v8代码
cd v8
git checkout 6.2.414.46                     # 切换至需要的分支
gclient sync                                # 根据分支再次更新工具链
tools/dev/v8gen.py x64.release -- \
  v8_enable_disassembler=true \
  v8_enable_object_print=true               # 配置编译选项
ninja -C out.gn/x64.release d8              # 编译d8

```

`v8_enable_disassembler`和`v8_enable_object_print`一定要开启，否则反汇编时不显示常量内容

### 修改 d8 代码 (d8.cc)

在 Shell 类增加 LoadJSC 方法

```
static void Disassemble(v8::internal::BytecodeArray* bytecode) {
  internal::OFStream os(stdout);
  bytecode->Disassemble(os);
  auto consts = bytecode->constant_pool();
  for (int i = 0; i < consts->length(); i++) {
    auto obj = consts->get(i);
    if (obj->IsSharedFunctionInfo()) {
      auto shared = v8::internal::SharedFunctionInfo::cast(obj);
      os << "Function name " << shared->name()->ToCString().get() << "\n";
      Disassemble(shared->bytecode_array());
    }
  }
}
void Shell::LoadJSC(const v8::FunctionCallbackInfo& args) {
  auto isolate = reinterpret_cast(args.GetIsolate());
  for (int i = 0; i < args.Length(); i++) {
    String::Utf8Value filename(args.GetIsolate(), args[i]);
    if (*filename == NULL) {
      Throw(args.GetIsolate(), "Error loading file");
      return;
    }
    int length = 0;
    auto filedata = reinterpret_cast(ReadChars(*filename, &length));
    if (filedata == NULL) {
      Throw(args.GetIsolate(), "Error reading file");
      return;
    }
    auto scriptdata = new i::ScriptData(filedata, length);
    auto source = isolate->factory()
                      ->NewStringFromUtf8(i::CStrVector("source"))
                      .ToHandleChecked();
    auto fun = i::CodeSerializer::Deserialize(isolate, scriptdata, source)
                   .ToHandleChecked();
    Disassemble(fun->bytecode_array());
  }
} 
```

注册为全局函数，在 Shell::CreateGlobalTemplate 中添加代码

```
global_template->Set(
    String::NewFromUtf8(isolate, "loadjsc", NewStringType::kNormal)
        .ToLocalChecked(),
    FunctionTemplate::New(isolate, LoadJSC));

```

由于目标 v8 版本为 6.2.414.54，但 v8 的 git 仓库中并没有这个版本

 

所以用了最接近的版本 6.2.414.46，但反序列时一些校验无法通过

*   SerializedCodeData::SanityCheck
*   Deserializer::Initialize

需要修改以上两个方法，代码过于丑陋就不贴了。

 

修改后使用`ninja -C out.gn/x64.release d8`命令重新编译即可。

### 使用 loadjsc 进行反汇编

使用命令`v8/out.gn/x64.release/d8 -e "loadjsc('xxx.jsc')"` 即可反汇编 jsc 文件

 

例如源码`test.js`

```
function xxx(){
    console.log("asdasd")
}
xxx()

```

其通过 wiz 的 docker 里 bytenode 编译为 test.jsc

 

命令 `/wiz/app/wizserver/node_modules/.bin/bytenode -c test.js`

 

再通过 d8 反汇编, 结果如下:

 

命令 `./d8 -e "loadjsc('test.jsc')"`

```
Parameter count 1
Frame size 8
    0 E> 0x17c42dd2c522 @    0 : 91                StackCheck
    0 S> 0x17c42dd2c523 @    1 : 6e 00 00 00       CreateClosure [0], [0], #0
         0x17c42dd2c527 @    5 : 1e fb             Star r0
  115 S> 0x17c42dd2c529 @    7 : 95                Return
Constant pool (size = 1)
0x17c42dd2c531: [FixedArray] in OldSpace
 - map = 0x1b27f0f82309 - length: 1
           0: 0x17c42dd2c549 Handler Table (size = 16)
Function name
Parameter count 6
Frame size 8
         0x17c42dd2c742 @    0 : 6e 00 00 02       CreateClosure [0], [0], #2
         0x17c42dd2c746 @    4 : 1e fb             Star r0
   10 E> 0x17c42dd2c748 @    6 : 91                StackCheck
  106 S> 0x17c42dd2c749 @    7 : 4f fb 01          CallUndefinedReceiver0 r0, [1]
         0x17c42dd2c74c @   10 : 04                LdaUndefined
  113 S> 0x17c42dd2c74d @   11 : 95                Return
Constant pool (size = 1)
0x17c42dd2c751: [FixedArray] in OldSpace
 - map = 0x1b27f0f82309 - length: 1
           0: 0x17c42dd2c769 Handler Table (size = 16)
Function name xxx
Parameter count 1
Frame size 24
   74 E> 0x17c42dd2c862 @    0 : 91                StackCheck
   82 S> 0x17c42dd2c863 @    1 : 0a 00 02          LdaGlobal [0], [2]
         0x17c42dd2c866 @    4 : 1e fa             Star r1
   90 E> 0x17c42dd2c868 @    6 : 20 fa 01 04       LdaNamedProperty r1, [1], [4]
         0x17c42dd2c86c @   10 : 1e fb             Star r0
         0x17c42dd2c86e @   12 : 09 02             LdaConstant [2]
         0x17c42dd2c870 @   14 : 1e f9             Star r2
   90 E> 0x17c42dd2c872 @   16 : 4c fb fa f9 00    CallProperty1 r0, r1, r2, [0]
         0x17c42dd2c877 @   21 : 04                LdaUndefined
  104 S> 0x17c42dd2c878 @   22 : 95                Return
Constant pool (size = 3)
0x17c42dd2c881: [FixedArray] in OldSpace
 - map = 0x1b27f0f82309 - length: 3
           0: 0x2e3b60d34a19 1: 0x2e3b60d08e41 2: 0x17c42dd2c8e9 Handler Table (size = 16) 
```

分析授权逻辑
------

### 前台获取授权信息的 api 接口

![](https://bbs.pediy.com/upload/attach/202106/353809_RFAXJRWP8EAT76V.png)

 

返回数据

```
{
    "returnCode": 200,
    "returnMessage": "OK",
    "externCode": "",
    "result": {
        "start": 1571192332036,
        "end": 4733432332036,
        "count": 5,
        "oem": "wiz",
        "type": "license_free",
        "version": 1,
        "key": "4bc0cc40-efbb-11e9-bef0-0faf68675f7c",
        "ext": {}
    }
}

```

### 定位接口对应的文件

根据上述 url，`http://127.0.0.1/as/admin/licence?clientType=web&clientVersion=4.0&lang=zh-cn`

 

结合后端的目录结构，首先对`admin_router.jsc`文件进行反汇编分析

```
# pwd
/wiz/app/wizserver/as/admin
# ll
total 128
-rw-r--r-- 1 root root  7408 Dec 23  2020 admin_file_utils.jsc
-rw-r--r-- 1 root root 80720 Dec 23  2020 admin_router.jsc
-rw-r--r-- 1 root root  1680 Dec 23  2020 admin_static_files.jsc
-rw-r--r-- 1 root root  3384 Dec 23  2020 ldap_settings.jsc
drwxr-xr-x 4 root root  4096 Dec 23  2020 meta_files
-rw-r--r-- 1 root root  3304 Dec 23  2020 middleware.jsc
-rw-r--r-- 1 root root 15240 Dec 23  2020 oem_utils.jsc
-rw-r--r-- 1 root root  6304 Dec 23  2020 wizbox_search.jsc

```

#### admin_router.jsc

![](https://bbs.pediy.com/upload/attach/202106/353809_MD6N55BGY2ZV3NG.png)

 

![](https://bbs.pediy.com/upload/attach/202106/353809_RRJDQWN8PH37DAV.png)

 

![](https://bbs.pediy.com/upload/attach/202106/353809_43BWMW48SZ3T7Q6.png)

 

![](https://bbs.pediy.com/upload/attach/202106/353809_DXQEX3YE6FJQUQP.png)

 

粗略分析，发现`admin_router`调用了`addLicence`和`getLicence`两个方法

 

而同时包含这两个方法的文件是`wiz_name_utils.jsc`

#### wiz_name_utils.jsc

![](https://bbs.pediy.com/upload/attach/202106/353809_8BX8XEJS5HNEWVQ.png)

 

![](https://bbs.pediy.com/upload/attach/202106/353809_9J6P8HDQNY2PA4K.png)

 

发现 wiz_name_utils.jsc 应该是调用了 node-rsa 库，通过 RSA 算法来计算相关授权

#### node-rsa

为了验证猜测，通过修改`/wiz/app/wizserver/node_modules/node-rsa/src/NodeRSA.js`的构造函数，来打印一下公钥

 

![](https://bbs.pediy.com/upload/attach/202106/353809_8DXKZW73RFK4YDW.png)

```
pm2 restart as  #pm2重启as服务，并访问网页授权页面
tail -f /root/.pm2/logs/as-out.log #查看as日志

```

![](https://bbs.pediy.com/upload/attach/202106/353809_HF6RZRXWT2FM52C.png)

 

为了进一步验证，我们修改解密函数，打印返回值

 

![](https://bbs.pediy.com/upload/attach/202106/353809_ZCZP8ED7X5AEM7P.png)

 

再次重启 as 服务，并访问网页授权页面

```
2021-06-29 18:15 +08:00: {"start":1571192332036,"end":4733432332036,"count":5,"oem":"wiz","type":"license_free","version":1,"key":"4bc0cc40-efbb-11e9-bef0-0faf68675f7c","ext":{}}
2021-06-29 18:15 +08:00: e58678339769ba7a9139202655ab345f

```

解密后的数据与网页接口返回的数据一致，证明猜测位置正确

解除授权封印
------

通过上面的分析，思路就很清楚了。我们只要在特定的时候修改`NodeRSA.prototype.decryptPublic`的返回值可以控制授权

```
NodeRSA.prototype.decryptPublic = function (buffer, encoding) {
    var data = this.$$decryptKey(true, buffer, encoding);
    try{
        var v = JSON.parse(data);
        if(v.count == 5){
            v.count = 99999;
            v.type = 'license_vip';
            data = Buffer.from(JSON.stringify(v));
        }
    }catch(e){
    }
    return data;
};

```

最终结果:

 

![](https://bbs.pediy.com/upload/attach/202106/353809_ZRZXAMCV64XA43N.png)

[[看雪官方培训] Unicorn Trace 还原 Ollvm 算法！《安卓高级研修班》2021 年 9 月班火热招生！！](https://bbs.pediy.com/thread-267018.htm)

[#软件保护](forum-4-1.htm?tagids=3)
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282780.htm)

> [原创] 某东东的 jdgs 算法分析 -- 适合进阶学习

某东东的 jdgs 算法分析
==============

  这个贴主要还是对算法本身结构部分描述会多点，憋问，问就是过去太久了，很多逆向过程不一定能还原（主要是懒，不想原路再走一遍），所以可能有部分跳跃的内容，会给具体代码，但对应的偏移地址和具体信息没有，给大家一个锻炼自己的机会 (•_•)

  继续观看前申明：本文涉及的内容是为给大家提供适合大家巩固基础及进阶更高的技术，别做不好的事情哦。

算法分析结构划分
--------

1、查找 java 调用 jdgs 算法位置，frida 主动调用获取参数；  
2、unidbg 模拟算法 so 执行；  
3、枯燥的边调边复现算法；

java 调用部分
---------

  这部分直接参考其他佬的，挺详细的 [[原创] 某东 APP 逆向分析 + Unidbg 算法模拟](https://bbs.kanxue.com/thread-276430.htm) ；

unidbg 模拟执行 jdgs 流程
-------------------

  jdgs 算法的 unidbg 模拟执行上面链接里的结果出现以下情况问题解决：  
![](https://bbs.kanxue.com/upload/tmp/800845_UPHWRD4PPG6NUF4.webp)

### 问题一：

![](https://bbs.kanxue.com/upload/tmp/800845_TZUZN3E9W36AKT5.webp)  
![](https://bbs.kanxue.com/upload/tmp/800845_2QHCYWEFTWAD48W.webp)  
  看到 java 层和 so 层都对 i2 出现了不同参数对应不同功能的分支就要打起精神了，需要判断在走 i2=101 主体算法获取 jdgs 结果之前是否有走其他流程，明显是的，它执行了 i2=103 的 init 初始化部分，你在分析 java 层调用 jdgs native 算法的时候会看到 init 部分，so 层分析时也能看到 i2 值会走不同的分支；  
所以需要在 unidbg 里提前执行一步 init：

```
// jdgs初始化
public void doInit()
{
    //init
    System.out.println("=== init begin ====");
    Object[] init_param = new Object[1];
    Integer init = 103;
    DvmObject ret = module.callStaticJniMethodObject(
         emulator, "main(I[Ljava/lang/Object;)[Ljava/lang/Object;",
         init,
         ProxyDvmObject.createObject(vm, init_param));
    System.out.println("=== init end ====");
}

```

### 问题二：

  jdgs 算法过程会调用具体的两个资源文件，位置在解压文件夹 assets 里，后缀是. jdg.jpg 和. jdg.xbt，通过 unidbg 自带的 HookZz 框架将这两个本地文件先入内存再写入到寄存器里（这部分我不贴代码了，新手可以练练手）；

### 问题三：

  这个问题就是需要你手动去利用 unidbg 调试算法过程，去查看控制台报红日志代码位置点在哪，追溯为什么会走这个报红日志，去手动修改这些点，这里我就直接贴代码给大家：

```
//patch C0 53 00 B5, 将反转条件跳转CBZ-->CBNZ  会报：[main]E/JDG: [make_header:491] [-] Error pp not init
byte[] CBNZ = new byte[]{(byte) 0xC0, (byte)0x53, (byte)0x00, (byte)0xB5};
UnidbgPointer p = UnidbgPointer.pointer(emulator,dm.getModule().base + 地址);
p.write(CBNZ);
MyDbg.addBreakPoint(dm.getModule(), 地址, (emulator, address) -> {
    System.out.println("====== patch 反转条件跳转CBZ-->CBNZ ======");
    return true;
});
 
//干掉一个free  （这个会影响结果） 会报：[main]E/JDG: [make_header:491] [-] Error pp not init
byte[] NOP = new byte[]{(byte)0xC1F, (byte)0xC20, (byte)0xC03, (byte)0xD5};
UnidbgPointer p = UnidbgPointer.pointer(emulator,dm.getModule().base + 地址 );
p.write(NOP);
MyDbg.addBreakPoint(dm.getModule(), 地址, (emulator, address) -> {
    System.out.println("======= 干掉一个free =======");
    return true;
});

```

  这些问题主要还是动手能力的体现，就算天赋异禀，也要老老实实的动手；  
  最终会看到满意的结果：  
![](https://bbs.kanxue.com/upload/attach/202408/800845_MNPK258T8U6HDY9.webp)

jdgs 算法分析
---------

  首先看下 jdgs 的结果

```
{
"b1":".jdg.xbt文件名",
"b2":"***",
"b3":"***",
"b4":"yY8lbpaUOZeQ3fyCiccRrM66O+Nzo/mhwP4wIa8C8JOZ6aJgSdfTJl2a6Q4oeMBx+2P4ySmoN/AtDHutJNGd/lImZaXQkwd00ZyfFGn2PmTk4uorMcnQUrKbmPRHlcKx6iOwmt8RoYf9C7l7bGWQ/COl6HcUT199wCWGjI5+u4mxfvLmiCSqhJ8qbLgVx9KQrRLXW1oDY1sf1RdNl1cYe6GfpF8kwgNMQJif9EIUBw0Td64cduT7MKAFjA3oew02IyWX2aSJaOuWaULTUqO4al9SIyRYojxQCEiMzF5UMxV6Zwu2lw1uZ6+22fJgxbEBv2LeGUpPPzXGF6E2vC0vb9sE5in3CkrKHwM+QfA5CasSPwpAmzQyr5iGyl9o6g==",
"b5":"7e640fcb8293d390b3758974b75e9dad5082bed9",
"b7":"1724176633106",
"b6":"30ed898f8d129b6d16c3f0c49efae07e8de4ee0e"}

```

  通过重复抓包和执行，确定固定值 b1（.jdg.xbt 文件名）、b2、b3、b7(时间戳，分析过程通过 hook 固定)，  
需要分析 b4、b5、b6，其实实际走完算法，主要是考验你对标准算法的熟悉程度（ida 脚本 Findcrypt），因为并没有出现魔改算法，自定义算法也没混淆，难度不大，但详细写篇幅有点大了，适合新手进阶，所以我说下算法具体实现，就不参照 ida 和 unidbg 调试过程手摸手复现；

### 分析前固定时间戳

  经验之谈，分析算法过程，时间戳一般都是在算法中主要变动参数之一，为了减小分析影响，我们可以选择固定时间戳值：  
  so 直接搜索获取时间戳的常见函数名进行回溯找到时间戳生成位置：  
![](https://bbs.kanxue.com/upload/tmp/800845_TCRVNMQTPVR949U.webp)  
  然后通过 unidbg 的 HookZz 实现固定

```
// 固定时间戳 修改获取毫秒时间戳系统函数返回值十六进制
        hook.replace(dm.getModule().base + 地址, new ReplaceCallback() {
            @Override
            public HookStatus onCall(Emulator emulator, long originFunction) {
                return super.onCall(emulator, originFunction);
            }
 
            @Override
            public HookStatus onCall(Emulator emulator, HookContext context, long originFunction) {
                System.out.println("\n=========== HooZz 修改固定时间戳 =========\n");
                return super.onCall(emulator, context, originFunction);
            }
 
            @Override
            public void postCall(Emulator emulator, HookContext context) {
                long a = (long) emulator.getBackend().reg_read(Arm64Const.UC_ARM64_REG_X0);
                System.out.println("修改前时间戳："+Long.toHexString(a));
                emulator.getBackend().reg_write(Arm64Const.UC_ARM64_REG_W0,0x18f70ef8d12L);
                System.out.println("修改后时间戳："+ Long.toString(0x18f70ef8d12L,16));
            }
        },true);

```

### b4

首先传参拼接一串 json：e1: 参数三 eid，e2：参数二 finger 和一些常量，e3：时间戳  
这一串 json 会进行压缩操作, 返回值：comp_json

```
# 压缩算法
def fun_compress(self, json):
    # json_len=len(json)
    # # 使用compressBound计算压缩后的最大可能字节数
    # comp_bound = zlib.compressBound(json_len)
    # 使用compress方法压缩数据
    comp_data = zlib.compress(json.encode('utf-8'))
    return bytearray(comp_data)

```

接下来是获取一块 0x100 自定义加密数据: buf_sf_0x100

```
# salt = 时间戳+一段0x28固定值
def fun_sf(self, salt):
    salt = bytearray(salt, "utf-8")
    # 使用列表推导式创建一个从0到255的整数列表
    int_list = [i for i in range(256)]
    # 将整数列表转换为 bytearray
    ret_arr = bytearray(int_list)  # X0
 
    var2 = 0  # W11
 
    salt_len = len(salt)  # W10
    for i in range(0x100):
        # print(f"{i:02x}")
        salt_chunk = int(i / salt_len)  # W13                       SDIV            W13, W10, W2
        ret_i = ret_arr[i]  # W12                                   LDRB            W12, [X0,X10]
        salt_chunk = i - salt_chunk * salt_len  # W13               MSUB            W13, W13, W2, W10
        salt_chunk = salt[salt_chunk]  # W13                        LDRB            W13, [X1,W13,UXTW]
        var2 = ret_i + var2  # W11                                  ADD             W11, W12, W11
        var2 = var2 + salt_chunk  # W11                             ADD             W11, W11, W13
 
        salt_chunk = var2 & 0xff  # X13                             AND             X13, X11, #0xFF
        ret_arr[i] = ret_arr[salt_chunk]  # W14                     LDRB            W14, [X0,X13]
        #                                                           W13   STRB            W14, [X0,X10]
        ret_arr[salt_chunk] = ret_i  # W12                          STRB            W12, [X0,X13]
 
    return ret_arr

```

然后将 buf_sf_0x100 和 comp_json 进行处理获得新的：comp_json

```
# 寄存器格式为dword格式
def tool_range0xff(self, var):
       return var & 0xff
def fun_xor(self, buf_sf, comp_json):
       buf_sf.append(0)  # 扩容到0x102
       buf_sf.append(0)  # 扩容到0x102
       self.jdgstools.tool_bytearray2str(buf_sf)
       comp_json_len = len(comp_json)
       i = 0
       # try:
       while True:
           comp_json_len -= 1  # SUBS            X10, X10, #1            ; X0=X0-1=--len
           buf_0x100 = self.jdgstools.tool_range0xff(
               buf_sf[0x100])  # LDRB            W11, [X0,#0x100]        ; W11=X0[0x100]=buf[0x100]
           buf_0x101 = self.jdgstools.tool_range0xff(
               buf_sf[0x101])  # LDRB            W12, [X0,#0x101]        ; W12=buf_0x101 =X0[0x101]=*(buf + 0x101);
 
           buf_0x100_i = self.jdgstools.tool_range0xff(
               buf_0x100 + 1)  # ADD             W11, W11, #1            ; W11=W11+1=buf[0x100]+1
 
           buf_sf[0x100] = buf_0x100_i  # STRB            W11, [X0,#0x100]        ; X0[0x100]=W11
 
           buf_0x100_i = buf_0x100_i & 0xff  # AND             X11, X11, #0xFF         ; X11=W11&0xff
           buf_var = self.jdgstools.tool_range0xff(
               buf_sf[buf_0x100_i])  # LDRB            W13, [X0,X11]           ; W13=X0[X11]
           buf_0x101 = buf_0x101 + buf_var  # ADD             W12, W12, W13           ; W12=W12+W13
 
           buf_sf[0x101] = self.jdgstools.tool_range0xff(
               buf_0x101)  # STRB            W12, [X0,#0x101]        ; X0[0x101]=W12
           buf_0x101 = buf_0x101 & 0xff  # AND             X12, X12, #0xFF         ; X12=X12&0xff
 
           buf_var = self.jdgstools.tool_range0xff(
               buf_sf[buf_0x101])  # LDRB            W13, [X0,X12]           ; W13=X0[X12]
           var = self.jdgstools.tool_range0xff(
               buf_sf[buf_0x100_i])  # LDRB            W14, [X0,X11]           ; W14=X0[X11]
           buf_sf[buf_0x100_i] = buf_var  # STRB            W13, [X0,X11]           ; X0[X11]=W13
           buf_sf[buf_0x101] = var  # STRB            W14, [X0,X12]           ; X0[X12]=W14
           buf_0x100_i = self.jdgstools.tool_range0xff(
               buf_sf[0x100])  # LDRB            W11, [X0,#0x100]        ; W11=X0[0x100]
           buf_0x101 = self.jdgstools.tool_range0xff(
               buf_sf[0x101])  # LDRB            W12, [X0,#0x101]        ; W12=X0[0x101]
           buf_0x100_i = self.jdgstools.tool_range0xff(
               buf_sf[buf_0x100_i])  # LDRB            W11, [X0,X11]           ; W11=X0[X11]
           buf_0x101 = self.jdgstools.tool_range0xff(
               buf_sf[buf_0x101])  # LDRB            W12, [X0,X12]           ; W12=X0[X12]
 
           var_comp_json = self.jdgstools.tool_range0xff(
               comp_json[i])  # LDRB            W13, [X1],#1            ; W13=*X1+1
           # i += 1
 
           buf_0x100_i = buf_0x101 + buf_0x100_i  # ADD             W11, W12, W11           ; W11=W12+W11
           buf_0x100_i = buf_0x100_i & 0xFF  # AND             X11, X11, #0xFF         ; X11=X11&0xFF
 
           buf_0x100_i = self.jdgstools.tool_range0xff(
               buf_sf[buf_0x100_i])  # LDRB            W11, [X0,X11]           ; W11=X0[X11]
           buf_0x100_i = buf_0x100_i ^ var_comp_json  # EOR             W11, W11, W13           ; W11=W11^W13
 
           comp_json[i] = buf_0x100_i  # STRB            W11, [X2],#1            ; *X2+1=W11
 
           i += 1
           # print(f"i:{hex(i)}  {hex(buf_0x100_i)}")
           # comp_json[i] = buf_sf[buf_sf[buf_sf[0x101]] + buf_sf[buf_sf[0x100]]] ^ var_comp_json
 
           if comp_json_len == 0:
               break
       # except:
       #     print("error i:",i)
 
       return comp_json

```

最后对 comp_json 进行 base64 即可获得 b4

```
def fun_base64(self, comp_json):
    ret = base64.b64encode(comp_json)
    return ret
b4 = comp_json.decode('utf-8')

```

### b5

首先对 b1 进行计算

```
self.b1 = ".jdg.xbt文件名"  # jpg文件名 版本固定
# xbt字节加密iv 版本固定（需要判断下）
self.xbt_eny = "5A 36 58 38 65 66 74 42 4E 6D 53 35 56 4B 6F 47 71 53 2F 71 34 70 44 53 36 76 72 32 53 4B 76 74 34 61 31 49 65 61 37 67 6A 54 35 52 64 32 4C 2F 65 39 76 78 4D 6D 74 69 78 58 57 75 72 75 2B 68"
 
    # 这个函数xbt & body 字节加密
    def fun_body_eny(self, comm_body, enc):
        # print("准备加密body：",comm_body)
        logger.debug("准备加密body：{}".format(comm_body))
        comm_arr = bytearray(comm_body, 'utf-8')
        enc_arr = self.jdgstools.tool_str2bytearr(enc)
        comm_len = len(comm_arr)
        enc_len = len(enc_arr)
 
        buf = bytearray(comm_arr)
 
        i = 0
 
        if comm_len <= enc_len:
            while True:
 
                v14 = self.jdgstools.tool_range0xff(enc_arr[i])
 
                v13 = self.jdgstools.tool_range0xff(i // comm_len)
                var = self.jdgstools.tool_range0xff(i - v13 * comm_len)
                v15 = self.jdgstools.tool_range0xff(comm_arr[var])
 
                i += 1
 
                buf[var] = self.jdgstools.tool_range0xff(v14 ^ v15)
                # print(f"i:{hex(i)},v14:{hex(v14)},v15:{hex(v15)},v14 ^ v15:{hex(v14 ^ v15)}")
                if i == enc_len:
                    break
        else:
            while True:
                v11 = self.jdgstools.tool_range0xff(comm_arr[i])
 
                var = self.jdgstools.tool_range0xff(i - (i // enc_len) * enc_len)
                v12 = self.jdgstools.tool_range0xff(enc_arr[var])
 
                buf[i] = self.jdgstools.tool_range0xff(v11 ^ v12)
                i += 1
                if i == comm_len:
                    break
 
        return buf
 
self.body_eny = self.fun_body_eny(self.b1, self.xbt_eny)
    

```

然后对参数一的 comm_body 也进行同样处理，

```
# comm_body加密
body_eny = self.fun_body_eny(body, self.body_eny)

```

然后对 body_eny 进行 md5 得到 md5_text

```
# md5算法
def fun_md5(self, buf):
    var_mad5 = hashlib.md5()
    var_mad5.update(buf.encode("utf-8"))
 
    return var_mad5.hexdigest()

```

然后通过 Findcrpy 知道 md5_text 要进行 aes 加密得到 aes_text，key 和 iv 是内存值，不难找

```
def fun_aes(self, plaintext, key, iv):
 
    # 对明文进行填充，使其长度为16的倍数
    padded_plaintext = pad(plaintext, AES.block_size)
    # 创建AES的CBC模式对象
    cipher = AES.new(key, AES.MODE_CBC, iv)
    # 加密 bytes
    ciphertext = cipher.encrypt(padded_plaintext)
    return ciphertext

```

接下来是 sha1 加密算法，明文 comm_body+" "+aes_text

```
# sha1加密
def fun_sha1(self, commbody_aes):
    sha1 = hashlib.sha1()
    sha1.update(commbody_aes)
    # print(sha1.hexdigest())
    return sha1.hexdigest()

```

结果就是 b5

### b6

b6 的参数是拼接值

```
# b6_data = '{"b1":"{}","b2":"{}","b3":"{}","b4":"{}","b5":"{}","b7":"{}","b6":"{}"}' % b1,b2,b3,b4,b5,b7

```

然后和 b5 相同的算法结构，得到 b6

结尾
==

  这个算法过程其实非常适合新手进阶，并没有混淆和魔改，但遇到的问题都非常典型，文章的本身也是抱着锻炼的想法写的，不喜勿喷，希望大家可以互相交流，一起进步。

[[竞赛]2024 KCTF 大赛征题截止日期 08 月 10 日！](https://bbs.kanxue.com/thread-281194.htm)

[#逆向分析](forum-161-1-118.htm) [#协议分析](forum-161-1-120.htm)
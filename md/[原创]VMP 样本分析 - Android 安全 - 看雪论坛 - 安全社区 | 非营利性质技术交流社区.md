> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-288854.htm)

> [原创]VMP 样本分析

*   **unidbg 搭架子**
    

```
package com.tk.run;
 
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Emulator;
import com.github.unidbg.Module;
import com.github.unidbg.arm.backend.Unicorn2Factory;
import com.github.unidbg.file.FileResult;
import com.github.unidbg.file.IOResolver;
import com.github.unidbg.file.linux.AndroidFileIO;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.linux.android.dvm.DalvikModule;
import com.github.unidbg.linux.android.dvm.DvmClass;
import com.github.unidbg.linux.android.dvm.VM;
import com.github.unidbg.linux.android.dvm.array.ByteArray;
import com.github.unidbg.linux.android.dvm.wrapper.DvmInteger;
import com.github.unidbg.linux.file.ByteArrayFileIO;
import com.github.unidbg.linux.file.SimpleFileIO;
import com.github.unidbg.memory.Memory;
import com.github.unidbg.memory.MemoryBlock;
import com.github.unidbg.pointer.UnidbgPointer;
import com.github.unidbg.virtualmodule.android.AndroidModule;
import com.sun.jna.JNIEnv;
 
import java.io.File;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.PrintStream;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;
 
public class fucktk extends AbstractJni implements IOResolver {
    private final AndroidEmulator emulator;
    private final Module module;
    private final VM vm;
 
    @Override
    public FileResult resolve(Emulator emulator, String pathname, int oflags) {
        System.out.println("file open:" + pathname);
        return null;
    }
 
    public fucktk() {
        System.out.print("enter");
        emulator = AndroidEmulatorBuilder
                .for64Bit()
                .addBackendFactory(new Unicorn2Factory(true))
                .setProcessName("com.zhiliaoapp.musically")
                .build();
        Memory memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23));
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/resources/app/demo.apk"));
        vm.setJni(this);
        vm.setVerbose(true);
        emulator.getSyscallHandler().addIOResolver(this);
        new AndroidModule(emulator, vm).register(memory);
        DalvikModule dm = vm.loadLibrary("Encryptor", true);
        module = dm.getModule();
        System.out.print("base :" + module.base + "\n");
        System.out.print("size :" + module.size + "\n");
        dm.callJNI_OnLoad(emulator);
        trace2file();
    }
 
    private void trace2file() {
        String traceFile = "unidbg-android/src/test/resources/tk/trace/trace.txt";
        PrintStream traceStream = null;
        try{
            traceStream = new PrintStream(new FileOutputStream(traceFile), true);
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        }
        emulator.traceCode(module.base, module.base + module.size).setRedirect(traceStream);
    }
 
    public static String bytesToHex(byte[] bytes) {
        StringBuilder sb = new StringBuilder();
        for (byte b : bytes) {
            int unsignedInt = b & 0xff;
            String hex = Integer.toHexString(unsignedInt);
            if (hex.length() == 1) {
                sb.append('0');
            }
            sb.append(hex);
        }
        return sb.toString();
    }
 
    private void call_jni() {
        List params = new ArrayList<>();
        params.add(vm.getJNIEnv());
        params.add(0);
        byte[] b = {0x12,0x19,(byte)0xB3,0x18,0x1B,0x15,(byte)0xB2,0x1B,0x11,(byte)0xB0,0x12,0x12,0x13,(byte)0xB3,0x13,0x13};
        ByteArray by = new ByteArray(vm, b);
        params.add(vm.addLocalObject(by));
        params.add(16);
        Number number = module.callFunction(emulator, 0x7d88, params.toArray());
 
        if (number != null) {
            ByteArray resultArr = vm.getObject(number.intValue());
            System.out.println("result:"+ bytesToHex(resultArr.getValue()));
        }
    }
 
    public static void main(String[] args) {
        fucktk fk = new fucktk();
        fk.call_jni();
    }
} 
```

搭完架子后直接拿到 trace 方便后续分析

*   **入口函数分析**
    

**![](https://bbs.kanxue.com/upload/attach/202510/887351_XGGSJG5YHFP2NZU.webp)**

主要调用 sub_2BD4，进行加密

![](https://bbs.kanxue.com/upload/attach/202510/887351_R3YJUR2U7BHM7AK.webp)

sub_2BD4 主要调用 sub_2D28，入参 dword_B2C0 为一组内存数据，v5 为 sub_2BD4 的四个入参，off_21CE0 为一个长度为 8 的字节变量，v6 为一个中转函数，继续往下可以看到主加密的函数 sub_2D82 的 cfg 如下：

**![](https://bbs.kanxue.com/upload/attach/202510/887351_3SPMP2JCE8AMGUQ.webp)**

第一眼看上去的感觉像控制流平坦化，因为有序，主分发器，基本块，跟预处理器。但是伪 C 很明显可以看到 switch 的 case 不像随机数，而标准的控制流平坦化都是随机数。一般魔改也不会去修改这个 case 值。入口处有编码数组，所以猜是虚拟机函数

*   **静态分析**
    

一般虚拟机函数肯定要解析 opcode，先看下跳转处的汇编

![](https://bbs.kanxue.com/upload/attach/202510/887351_JKW5RTPKHM89UEM.webp)

结合 trace 搜索可以看到

```
[09:39:47 245][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0xdfbdfc28 => w10=0x28
[09:39:47 261][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40003044
[09:39:47 279][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x23bf023b => w10=0x3b
[09:39:47 282][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40002f70
[09:39:47 316][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x23be003b => w10=0x3b
[09:39:47 319][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40002f70
[09:39:47 343][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x1fb70e3b => w10=0x3b
[09:39:47 345][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40002f70
[09:39:47 349][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x1fb60c3b => w10=0x3b
[09:39:47 350][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40002f70
[09:39:47 353][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x1fb50a3b => w10=0x3b
[09:39:47 354][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40002f70
[09:39:47 356][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x1fb4083b => w10=0x3b
[09:39:47 358][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40002f70
[09:39:47 380][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x1fb3063b => w10=0x3b
...
[09:39:48 825][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x4000333c
[09:39:48 827][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0xbb1063b => w10=0x3b
[09:39:48 828][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40002f70
[09:39:48 829][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0xba50628 => w10=0x28
[09:39:48 829][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40003044
[09:39:48 830][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x8200c62f => w10=0x2f
[09:39:48 831][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x4000333c
[09:39:48 832][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x8320f5af => w10=0x2f
[09:39:48 832][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x4000333c
[09:39:48 833][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x240262f => w10=0x2f
[09:39:48 834][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x4000333c
[09:39:48 835][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0xba50428 => w10=0x28
[09:39:48 836][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40003044
[09:39:48 837][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0xbb4043b => w10=0x3b
[09:39:48 837][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40002f70
[09:39:48 838][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x8200c62f => w10=0x2f
[09:39:48 838][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x4000333c
[09:39:48 839][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x8320f5af => w10=0x2f
[09:39:48 840][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x4000333c
[09:39:48 841][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x240262f => w10=0x2f
[09:39:48 841][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x4000333c
[09:39:48 843][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0xba50228 => w10=0x28
[09:39:48 843][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40003044
[09:39:48 844][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0xbb5023b => w10=0x3b
[09:39:48 845][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40002f70
[09:39:48 846][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x8200c62f => w10=0x2f
[09:39:48 846][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x4000333c
[09:39:48 847][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x8320f5af => w10=0x2f
[09:39:48 847][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x4000333c
[09:39:48 849][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x240262f => w10=0x2f
[09:39:48 849][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x4000333c
[09:39:48 851][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x1fb00002 => w10=0x2
[09:39:48 851][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40002e50
[09:39:48 852][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x1fb10202 => w10=0x2
[09:39:48 852][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40002e50
[09:39:48 853][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x1fb20402 => w10=0x2
[09:39:48 854][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40002e50
[09:39:48 855][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x1fb30602 => w10=0x2
[09:39:48 855][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40002e50
[09:39:48 856][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x1fb40802 => w10=0x2
[09:39:48 857][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40002e50
[09:39:48 857][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x1fb50a02 => w10=0x2
[09:39:48 858][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40002e50
[09:39:48 859][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x1fb60c02 => w10=0x2
[09:39:48 859][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40002e50
[09:39:48 860][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x1fb70e02 => w10=0x2
[09:39:48 861][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40002e50
[09:39:48 862][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x23be0002 => w10=0x2
[09:39:48 862][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40002e50
[09:39:48 863][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x23bf0202 => w10=0x2
[09:39:48 864][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40002e50
[09:39:48 864][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x3e006af => w10=0x2f
[09:39:48 865][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x4000333c
[09:39:48 869][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x23bd0428 => w10=0x28
[09:39:48 869][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40003044

```

opcode 每次取 4 个字节，统计一下对应的 opcode 与跳转地址

0x3b     x2=0x40002f70

0x28     x2=0x40003044

0x2f      x2=0x4000333c

0x2       x2=0x40002e50

0xf        x2=0x40002ecc

0x4       x2=0x40002fec

0x3e     x2=0x40002fec

0x15  21   x2=0x40003044

0x36  54   x2=0x40002f70

0x2a  42   x2=0x400030a0

顺着每条 opcode 解析往下走不难发现 PC 指针的位置如下：

![](https://bbs.kanxue.com/upload/attach/202510/887351_GXPXN9JU8DF2CC7.jpg)

再看下对应的 trace 就比较明显了

```
[15:45:31 882][libEncryptor.so 0x03c24] [a90240f9] 0x40003c24: "ldr x9, [x21]" x21=0xbffff538 => x9=0x4000b2c4 #从x21取pc
[15:45:31 882][libEncryptor.so 0x03c28] [68025ef8] 0x40003c28: "ldur x8, [x19, #-0x20]" x8=0x0 x19=0xbffff670 => x8=0x0
[15:45:31 882][libEncryptor.so 0x03c2c] [20110091] 0x40003c2c: "add x0, x9, #4" x9=0x4000b2c4 => x0=0x4000b2c8 #pc加4
[15:45:31 882][libEncryptor.so 0x03c30] [1f0900f1] 0x40003c30: "cmp x8, #2" => nzcv: N=1, Z=0, C=0, V=0 x8=0x0
[15:45:31 883][libEncryptor.so 0x03c34] [a00200f9] 0x40003c34: "str x0, [x21]" x0=0x4000b2c8 x21=0xbffff538 => x0=0x4000b2c8 #存pc到x21
[15:45:31 883][libEncryptor.so 0x03c38] [e0000054] 0x40003c38: "b.eq #0x40003c54" nzcv: N=1, Z=0, C=0, V=0
[15:45:31 883][libEncryptor.so 0x03c3c] [1f0d00f1] 0x40003c3c: "cmp x8, #3" => nzcv: N=1, Z=0, C=0, V=0 x8=0x0
[15:45:31 883][libEncryptor.so 0x03c40] [80110054] 0x40003c40: "b.eq #0x40003e70" nzcv: N=1, Z=0, C=0, V=0
[15:45:31 883][libEncryptor.so 0x03c44] [1f0500f1] 0x40003c44: "cmp x8, #1" => nzcv: N=1, Z=0, C=0, V=0 x8=0x0
[15:45:31 883][libEncryptor.so 0x03c48] [40110054] 0x40003c48: "b.eq #0x40003e70" nzcv: N=1, Z=0, C=0, V=0
[15:45:31 883][libEncryptor.so 0x03c4c] [1f0014eb] 0x40003c4c: "cmp x0, x20" x20=0x40002d1c => nzcv: N=0, Z=0, C=1, V=0 x0=0x4000b2c8
[15:45:31 883][libEncryptor.so 0x03c50] [a0110054] 0x40003c50: "b.eq #0x40003e84" nzcv: N=0, Z=0, C=1, V=0
[15:45:31 883][libEncryptor.so 0x03c54] [008cffb5] 0x40003c54: "cbnz x0, #0x40002dd4" x0=0x4000b2c8
[15:45:31 883][libEncryptor.so 0x02dd4] [61025ef8] 0x40002dd4: "ldur x1, [x19, #-0x20]" x1=0x0 x19=0xbffff670 => x1=0x0
[15:45:31 883][libEncryptor.so 0x02dd8] [0c0040b9] 0x40002dd8: "ldr w12, [x0]" x0=0x4000b2c8 => w12=0x23be003b #取出下一条encode
[15:45:31 883][libEncryptor.so 0x02ddc] [3f0800f1] 0x40002ddc: "cmp x1, #2" => nzcv: N=1, Z=0, C=0, V=0 x1=0x0
[15:45:31 884][libEncryptor.so 0x02de0] [40a30054] 0x40002de0: "b.eq #0x40004248" nzcv: N=1, Z=0, C=0, V=0
[15:45:31 884][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x23be003b => w10=0x3b #下一条指令的opcode

```

后面就要开始分析指令了，大概初步扫了一圈，柿子挑软的捏，先找个熟悉的指令开始

opcode：0x28

```
.text:0000000000002DF0                 LSR             W11, W12, #0xB ; 字节码右移11位
.text:0000000000002DF4                 AND             W14, W11, #2 ; 取字节码第12位  ①
.text:0000000000002DF8                 LDRSW           X3, [X28,X10,LSL#2] ; x3=[x28 + x10 * 4] x28为跳转表地址 获取偏移的index
.text:0000000000002DFC                 AND             W16, W12, #0x10000000 ; 取w12[28]
.text:0000000000002E00                 AND             W2, W11, #4 ; 取字节码第13位 ②
.text:0000000000002E04                 BFXIL           W14, W12, #0x1F, #1 ; w14 = (w12 >> 31) & 0x1 取字节码第31位 ③
.text:0000000000002E08                 ORR             W14, W14, W2
.text:0000000000002E0C                 AND             W2, W11, #8 ; 取字节码第14位 ④
.text:0000000000002E10                 AND             W10, W11, #0x10 ; 取字节码第15位 ⑤
.text:0000000000002E14                 LSR             W11, W16, #0x1A
.text:0000000000002E18                 AND             W15, W12, #0x20000000 ; 取w12[29]
.text:0000000000002E1C                 ORR             W2, W14, W2
.text:0000000000002E20                 BFXIL           W11, W12, #0x1A, #2
.text:0000000000002E24                 AND             W14, W12, #0x40000000
.text:0000000000002E28                 ORR             W10, W2, W10 ; ①|②|③|④|⑤
.text:0000000000002E2C                 ORR             W11, W11, W15,LSR#26
.text:0000000000002E30                 ADD             X2, X3, X28 ; 计算跳转地址x2
.text:0000000000002E34                 UBFX            W9, W12, #0x15, #5 ; w9 = w12 >> 21 & 0x1f 取w12[21..25] 源寄存器索引
.text:0000000000002E38                 UBFX            W8, W12, #0x10, #5 ; w9 = w12 >> 16 & 0x1f 取w12[16..20] 目标寄存器索引
.text:0000000000002E3C                 AND             W13, W12, #0x80000000 ; 取w12[31]
.text:0000000000002E40                 AND             W18, W12, #0x4000000 ; 取w12[26]
.text:0000000000002E44                 AND             W17, W12, #0x8000000 ; 取w12[27]
.text:0000000000002E48                 ORR             W11, W11, W14,LSR#26
.text:0000000000002E4C                 BR              X2      ; switch jump
 
.text:0000000000003044                 AND             W0, W12, #0xF000 ; w0 = w12 & 0xf000 取w12[12..15]
.text:0000000000003048                 UBFX            W10, W12, #6, #6 ; w10 = w12 >> 6 & 0x3f 取w12[6..11]
.text:000000000000304C                 BFXIL           W0, W18, #0x14, #0xC ; w0 = (W0 & 0xFFFFF000) | (w18 >> 0x20 & 0xfff) 取w18[20..31] w18取w12[26]
.text:0000000000003050                 ORR             W10, W0, W10 ; w10 = w0 | w10
.text:0000000000003054                 ORR             W10, W10, W17,LSR#20 ; w10 = w10 | w17 >> 20 w17取字节码w12[27]
.text:0000000000003058                 ORR             W10, W10, W16,LSR#20 ;  w10 = w10 | w16 >> 20 w16取字节码w12[28]
.text:000000000000305C                 ORR             W10, W10, W15,LSR#20 ;  w10 = w10 | w15 >> 20 w15取字节码w12[29]
.text:0000000000003060                 AND             W11, W12, #0x3F
.text:0000000000003064                 ORR             W10, W10, W14,LSR#20 ;  w10 = w10 | w14 >> 20 w14取字节码w12[30]
.text:0000000000003068                 CMP             W11, #0xD
.text:000000000000306C                 ORR             W10, W10, W13,LSR#20 ;  w10 = w10 | w13 >> 20 w13取字节码w12[31]
.text:0000000000003070                 B.EQ            loc_3144
.text:0000000000003074                 CMP             W11, #0x28 ; '('
.text:0000000000003078                 B.EQ            loc_3144
 
.text:0000000000003144                 ADD             X11, X21, #8;x11 = x21 + 8 x21是context 加8偏移后x11是虚拟寄存器表
.text:0000000000003148                 LDR             X9, [X11,W9,UXTW#3] ;x9 = [x11 + (w9_uxth << 3)] w9是源寄存器索引
.text:000000000000314C                 ADD             X9, X9, W10,SXTH ;x9 = x9 + w10_sxth w10是立即数
.text:0000000000003150                 STR             X9, [X11,W8,UXTW#3] ;[x11 + (w8_uxth << 3)] = x9 w8是目标寄存器索引
.text:0000000000003154                 B               def_2E4C

```

1. 先从虚拟源寄存器 w9 取出数据，2. 加上立即数 w10，3. 将结果存入虚拟目的寄存器 w8 ，其中每个虚拟寄存器占 8 个字节，x[index] = x21 + 8 + index * 8，实现指令 add xd, xn, #imm

opcode：0x15

```
.text:0000000000003044                 AND             W0, W12, #0xF000 ; w0 = w12 & 0xf000 取w12[12..15]
.text:0000000000003048                 UBFX            W10, W12, #6, #6 ; w10 = w12 >> 6 & 0x3f 取w12[6..11]
.text:000000000000304C                 BFXIL           W0, W18, #0x14, #0xC ; w0 = (W0 & 0xFFFFF000) | (w18 >> 0x20 & 0xfff) 取w18[20..31] w18取w12[26]
.text:0000000000003050                 ORR             W10, W0, W10 ; w10 = w0 | w10
.text:0000000000003054                 ORR             W10, W10, W17,LSR#20 ; w10 = w10 | w17 >> 20 w17取字节码w12[27]
.text:0000000000003058                 ORR             W10, W10, W16,LSR#20 ;  w10 = w10 | w16 >> 20 w16取字节码w12[28]
.text:000000000000305C                 ORR             W10, W10, W15,LSR#20 ;  w10 = w10 | w15 >> 20 w15取字节码w12[29]
.text:0000000000003060                 AND             W11, W12, #0x3F
.text:0000000000003064                 ORR             W10, W10, W14,LSR#20 ;  w10 = w10 | w14 >> 20 w14取字节码w12[30]
.text:0000000000003068                 CMP             W11, #0xD
.text:000000000000306C                 ORR             W10, W10, W13,LSR#20 ;  w10 = w10 | w13 >> 20 w14取字节码w12[31]
.text:0000000000003070                 B.EQ            loc_3144
.text:0000000000003074                 CMP             W11, #0x28 ; '('
.text:0000000000003078                 B.EQ            loc_3144
.text:000000000000307C                 CMP             W11, #0x15
.text:0000000000003080                 B.NE            def_2E4C 
.text:0000000000003084                 ADD             X11, X21, #8 ;X11 = X21 + 8
.text:0000000000003088                 LSL             X9, X9, #3 ;X9 = X9 << 3
.text:000000000000308C                 LDRSW           X9, [X11,X9] ;取虚拟寄存器值x9
.text:0000000000003090                 SXTH            W10, W10
.text:0000000000003094                 ADD             X9, X9, W10,SXTW ; x9 = x9 + w10_sxtw
.text:0000000000003098                 STR             X9, [X11,W8,UXTW#3] ;存结果到虚拟寄存器W8
.text:000000000000309C                 B               def_2E4C

```

与 0x28 实现一样的 add 指令，只是取虚拟寄存器的顺序不一样即调整了 x21 加 8 与加 index * 8 的先后顺序

opcode：0x2

```
.text:0000000000002E50 loc_2E50                                ; CODE XREF: sub_2D28+124↑j
.text:0000000000002E50                                         ; DATA XREF: .rodata:jpt_2E4C↓o ...
.text:0000000000002E50                 AND             W10, W12, #0x3F ; jumptable 0000000000002E4C cases 0,2,8,18,31,33,34,39,43,48,49,51,55
.text:0000000000002E54                 CMP             W10, #0x37 ; '7'
.text:0000000000002E58                 B.HI            def_2E4C ; jumptable 0000000000002E4C default case, cases 1,6,7,9,12,19,24,25,28-30,37,38,46,52,53,57,60,61
.text:0000000000002E58                                         ; jumptable 0000000000002E94 default case, cases 1,3-7,9-17,19-32,35-38,40-42,44-47,50,52-54
.text:0000000000002E58                                         ; jumptable 0000000000002F18 default case, cases 4,6-14,16,18-25,27-41,43,46-57,59-62
.text:0000000000002E58                                         ; jumptable 0000000000002F58 default case, cases 4,6-14,16-25,27-43,46-57,59-62
.text:0000000000002E58                                         ; jumptable 0000000000002FB8 default case, cases 12,13,15-19,21,23-35,37-53,55-58
.text:0000000000002E58                                         ; jumptable 0000000000003228 default case, cases 4,6-14,16,18-25,27-41,43,46-57,59-62
.text:0000000000002E58                                         ; jumptable 0000000000003270 default case, cases 4,6-14,16-25,27-43,46-57,59-62
.text:0000000000002E58                                         ; jumptable 0000000000003360 cases 0,3,9,15,46,61
.text:0000000000002E58                                         ; jumptable 00000000000040C4 cases 4,6-14,16,18-25,27-41,43,46-57,59-62
.text:0000000000002E58                                         ; jumptable 0000000000004104 default case, cases 4,6-14,16-25,27-43,46-57,59-62
.text:0000000000002E5C                 UBFX            W11, W12, #6, #6 ; W11 = (W12 >> 6) & 0x3F w11取w12[6..11]
.text:0000000000002E60                 AND             W12, W12, #0xF000 ; w12 = w12 & 0xf000 取w12[12..15]
.text:0000000000002E64                 BFXIL           W12, W18, #0x14, #0xC ; w12 = (W12 & 0xFFFFF000) | (w18 >> 0x20 & 0xfff) 取w18[20..31] w18取w12[26]
.text:0000000000002E68                 ORR             W11, W12, W11 ; w11 = w12 | w11
.text:0000000000002E6C                 ADD             X9, X21, W9,UXTW#3 ; x9 = x21 + w9_uxtw << 3
.text:0000000000002E70                 LDRSW           X10, [X22,X10,LSL#2]
.text:0000000000002E74                 ORR             W11, W11, W17,LSR#20 ; w10 = w10 | w17 >> 20 w17取字节码w12[27]
.text:0000000000002E78                 LDR             X9, [X9,#8] ;取源寄存器地址X9
.text:0000000000002E7C                 ORR             W11, W11, W16,LSR#20 ; w10 = w10 | w16 >> 20 w16取字节码w12[28]
.text:0000000000002E80                 ORR             W11, W11, W15,LSR#20 ; w10 = w10 | w15 >> 20 w15取字节码w12[29]
.text:0000000000002E84                 ORR             W11, W11, W14,LSR#20 ; w10 = w10 | w14 >> 20 w14取字节码w12[30]
.text:0000000000002E88                 ORR             W11, W11, W13,LSR#20 ; w10 = w10 | w13 >> 20 w13取字节码w12[31]
.text:0000000000002E8C                 ADD             X10, X10, X22
.text:0000000000002E90                 ADD             X9, X9, W11,SXTH ;x9 = x9 + w11_sxth,w11为立即数
.text:0000000000002E94                 BR              X10     ; switch jump
 
.text:00000000000033D8 loc_33D8                                ; CODE XREF: sub_2D28+16C↑j
.text:00000000000033D8                                         ; DATA XREF: .rodata:000000000000AA68↓o
.text:00000000000033D8                 LDR             X9, [X9] ; jumptable 0000000000002E94 case 2 ;从x9的偏移地址取数据
.text:00000000000033DC                 B               loc_3C1C
 
.text:0000000000003C1C loc_3C1C                                ; CODE XREF: sub_2D28+6AC↑j
.text:0000000000003C1C                                         ; sub_2D28+6B4↑j ...
.text:0000000000003C1C                 ADD             X8, X21, W8,UXTW#3
.text:0000000000003C20
.text:0000000000003C20 loc_3C20                                ; CODE XREF: sub_2D28+1A0↑j
.text:0000000000003C20                                         ; sub_2D28+6EC↑j ...
.text:0000000000003C20                 STR             X9, [X8,#8] ;存储结果到虚拟寄存器w8

```

1. 先从虚拟寄存器 w9 取出数据 ，2. 加上立即数 w11， 3. 从偏移地址取处数据， 4. 存入虚拟寄存器 x8，实现指令 ldr xd, [xn, #imm]

opcode：0x2f 0x30

```
.text:000000000000333C                 AND             W13, W12, #0xFFF ; jumptable 0000000000002E4C case 47
.text:0000000000003340                 SUB             W14, W13, #0x6F ; 'o'
.text:0000000000003344                 EXTR            W14, W14, W14, #6 ;二级opcode
.text:0000000000003348                 CMP             W14, #0x3E ; switch 63 cases
.text:000000000000334C                 B.HI            def_3360 ; jumptable 0000000000003360 default case, cases 4,5,7,8,17-20,25,27-30,33,35,37,39,42,43,47,50,53-58
.text:0000000000003350                 ADRL            X15, jpt_3360
.text:0000000000003358                 LDRSW           X14, [X15,X14,LSL#2]
.text:000000000000335C                 ADD             X14, X14, X15
.text:0000000000003360                 BR              X14     ; switch jump ;二级opcode跳转
 
.text:0000000000003A60                 CMP             W13, #0xC6E ; jumptable 0000000000003360 cases 10,48
.text:0000000000003A64                 B.GT            loc_3A94
.text:0000000000003A68                 CMP             W13, #0x8EE
.text:0000000000003A6C                 B.GT            loc_3D4C
.text:0000000000003A70                 CMP             W13, #0x2EF
.text:0000000000003A74                 B.EQ            loc_4288
.text:0000000000003A78                 CMP             W13, #0x3EF
.text:0000000000003A7C                 B.EQ            loc_429C
.text:0000000000003A80                 CMP             W13, #0x46F
.text:0000000000003A84                 B.NE            def_2E4C
.text:0000000000003A88                 ADD             X9, X21, #8
.text:0000000000003A8C                 LDR             X8, [X9,W8,UXTW#3]
.text:0000000000003A90                 B               loc_42D4
 
.text:0000000000003A94                 MOV             W9, #0x2008EE
.text:0000000000003A9C                 CMP             W13, W9
.text:0000000000003AA0                 B.GT            loc_3D78
.text:0000000000003AA4                 CMP             W13, #0xC6F
.text:0000000000003AA8                 B.EQ            loc_42B0
.text:0000000000003AAC                 CMP             W13, #0xCAF
.text:0000000000003AB0                 B.EQ            loc_42C4
.text:0000000000003AB4                 CMP             W13, #0xF2F
.text:0000000000003AB8                 B.NE            def_2E4C
 
.text:00000000000042B0                 ADD             X9, X21, #8
.text:00000000000042B4                 LSL             X8, X8, #3
.text:00000000000042B8                 LDR             W8, [X9,X8] ;取虚拟寄存器w8
.text:00000000000042BC                 LSL             W8, W8, W11 ;左移操作
.text:00000000000042C0                 B               loc_432C
 
.text:000000000000432C                 SXTW            X8, W8
.text:0000000000004330                 STR             X8, [X9,W10,UXTW#3] ;存入虚拟寄存器w10
.text:0000000000004334                 B               def_2E4C

```

这个有两级 opcode，1. 取虚拟寄存器 w8，2. 左移 w11，3. 结果存入 w10 的虚拟寄存器，实现指令 lsl xd, xn, #imm

opcode：0x2f 0x26

```
.text:0000000000003364                 AND             W11, W12, #0xFFF ; jumptable 0000000000003360 cases 6,13,26,32,36,38,40,51
.text:0000000000003368                 CMP             W11, #0x86E
.text:000000000000336C                 B.LE            loc_3B4C
.text:0000000000003370                 CMP             W11, #0x9EE
.text:0000000000003374                 B.LE            loc_37FC
.text:0000000000003378
.text:0000000000003378 loc_3378                                ; CODE XREF: sub_2D28+AD0↓j
.text:0000000000003378                 CMP             W11, #0x9EF
.text:000000000000337C                 B.EQ            loc_380C
 
.text:000000000000380C                 ADD             X11, X21, #8
.text:0000000000003810                 LDR             X9, [X11,W9,UXTW#3] ; 获取虚拟寄存器w9的值
.text:0000000000003814                 LDR             X8, [X11,W8,UXTW#3] ; 获取虚拟寄存器w8的值
.text:0000000000003818                 ADD             X8, X8, X9 ; x8 = x8 + x9
.text:000000000000381C                 STR             X8, [X11,W10,UXTW#3] ; 结果存入虚拟寄存器w10
.text:0000000000003820                 B               def_2E4C

```

非常明显实现指令 add xd, xn, xm

opcode 0x2f 0x15

```
.text:000000000000333C                 AND             W13, W12, #0xFFF ; jumptable 0000000000002E4C case 47
.text:0000000000003340                 SUB             W14, W13, #0x6F ; 'o'
.text:0000000000003344                 EXTR            W14, W14, W14, #6
.text:0000000000003348                 CMP             W14, #0x3E ; switch 63 cases
.text:000000000000334C                 B.HI            def_3360 ; jumptable 0000000000003360 default case, cases 4,5,7,8,17-20,25,27-30,33,35,37,39,42,43,47,50,53-58
.text:0000000000003350                 ADRL            X15, jpt_3360
.text:0000000000003358                 LDRSW           X14, [X15,X14,LSL#2]
.text:000000000000335C                 ADD             X14, X14, X15
.text:0000000000003360                 BR              X14     ; switch jump
 
.text:0000000000004094                 CBNZ            X1, def_2E4C ; jumptable 0000000000003360 case 21
.text:0000000000004098                 AND             W13, W12, #0xFFF
.text:000000000000409C                 SUB             W14, W13, #3 ; switch 61 cases
.text:00000000000040A0                 CMP             W14, #0x3C
.text:00000000000040A4                 MOV             W8, WZR
.text:00000000000040A8                 B.HI            def_40C4
 
.text:0000000000004470                 CMP             W13, #0x5AF ; jumptable 00000000000040C4 default case
.text:0000000000004474                 B.EQ            loc_45F0
 
.text:00000000000045F0                 MOV             W8, W10
.text:00000000000045F4
.text:00000000000045F4 loc_45F4                                ; CODE XREF: sub_2D28+1754↑j
.text:00000000000045F4                 ADD             X9, X21, W9,UXTW#3
.text:00000000000045F8                 LDR             X9, [X9,#8];取虚拟寄存器w9
.text:00000000000045FC                 MOV             W12, WZR
.text:0000000000004600                 MOV             W10, WZR
.text:0000000000004604
.text:0000000000004604 loc_4604                                ; CODE XREF: sub_2D28+1804↑j
.text:0000000000004604                                         ; sub_2D28+1864↑j
.text:0000000000004604                 MOV             W11, #1
.text:0000000000004608
.text:0000000000004608 loc_4608                                ; CODE XREF: sub_2D28+1884↑j
.text:0000000000004608                 CMP             W10, #0
.text:000000000000460C                 CSET            W13, NE
.text:0000000000004610                 AND             W12, W12, W13
.text:0000000000004614                 STUR            X9, [X19,#-0x18];设置虚拟寄存器x9为跳转地址
.text:0000000000004618                 TBNZ            W12, #0, loc_4628
.text:000000000000461C                 CBZ             W10, loc_4628
.text:0000000000004620                 AND             W9, W11, W13
.text:0000000000004624                 CBZ             W9, loc_3700
.text:0000000000004628
.text:0000000000004628 loc_4628                                ; CODE XREF: sub_2D28+18F0↑j
.text:0000000000004628                                         ; sub_2D28+18F4↑j
.text:0000000000004628                 CMP             W12, #0
.text:000000000000462C                 MOV             W9, #1
.text:0000000000004630                 CINC            X9, X9, EQ
.text:0000000000004634                 STUR            X9, [X19,#-0x20];设置跳转标志位为2
.text:0000000000004638                 CMP             W8, #1
.text:000000000000463C                 B.LT            def_2E4C
.text:0000000000004640                 B               loc_3708
 
.text:0000000000003708                 ADD             X9, X0, #8
.text:000000000000370C                 ADD             X8, X21, W8,SXTW#3
.text:0000000000003710                 STR             X9, [X8,#8]
.text:0000000000003714                 STUR            X9, [X19,#-0x10];pc + 8 LR寄存器
.text:0000000000003718                 B               def_2E4C

```

1. 取出虚拟寄存器 w9，2. 将 w9 的值存入 [X19,#-0x18]，3. 设置[X19,#-0x20] 为 2，4. 设置 [X19,#-0x10] 为当前 PC+8，第一感觉就是跟跳转相关，找 trace 分析一下，在 trace 中搜索 0x02de4|0x40002e4c|0x03344|0x02dd8|\[x19, #-0x18\]|\[x19, #-0x20\]随便抽取一段分析

```
[15:45:31 956][libEncryptor.so 0x02dd8] [0c0040b9] 0x40002dd8: "ldr w12, [x0]" x0=0x4000b330 => w12=0x2000cf #1.从0x4000b330取encode
[15:45:31 956][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x2000cf => w10=0xf #解析opcode=0xf
[15:45:31 956][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40002ecc
[15:45:31 977][libEncryptor.so 0x03648] [68821ef8] 0x40003648: "stur x8, [x19, #-0x18]" x8=0x4000b340 x19=0xbffff670 => x8=0x4000b340 #存0x4000b340到[x19, #-0x18]
[15:45:31 979][libEncryptor.so 0x03668] [68021ef8] 0x40003668: "stur x8, [x19, #-0x20]" x8=0x2 x19=0xbffff670 => x8=0x2 #存0x2到[x19, #-0x20]
[15:45:31 979][libEncryptor.so 0x03c28] [68025ef8] 0x40003c28: "ldur x8, [x19, #-0x20]" x8=0x2 x19=0xbffff670 => x8=0x2
[15:45:31 979][libEncryptor.so 0x02dd4] [61025ef8] 0x40002dd4: "ldur x1, [x19, #-0x20]" x1=0xffffffffffff83dc x19=0xbffff670 => x1=0x2
[15:45:31 979][libEncryptor.so 0x02dd8] [0c0040b9] 0x40002dd8: "ldr w12, [x0]" x0=0x4000b334 => w12=0xc6f #2.从0x4000b334取encode
[15:45:31 980][libEncryptor.so 0x0424c] [7a021ef8] 0x4000424c: "stur x26, [x19, #-0x20]" x26=0x3 x19=0xbffff670 => x26=0x3 #存0x3到[x19, #-0x20]
[15:45:31 981][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0xc6f => w10=0x2f #解析opcode=0x2f
[15:45:31 981][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x4000333c
[15:45:31 981][libEncryptor.so 0x03344] [ce198e13] 0x40003344: "ror w14, w14, #6" w14=0xc00 => w14=0x30
[15:45:31 985][libEncryptor.so 0x03c28] [68025ef8] 0x40003c28: "ldur x8, [x19, #-0x20]" x8=0x0 x19=0xbffff670 => x8=0x3 
[15:45:31 986][libEncryptor.so 0x03e70] [60825ef8] 0x40003e70: "ldur x0, [x19, #-0x18]" x0=0x4000b338 x19=0xbffff670 => x0=0x4000b340 #从[x19, #-0x18]取出跳转地址
[15:45:31 986][libEncryptor.so 0x03e74] [7f021ef8] 0x40003e74: "stur xzr, [x19, #-0x20]" x19=0xbffff670
[15:45:31 986][libEncryptor.so 0x02dd4] [61025ef8] 0x40002dd4: "ldur x1, [x19, #-0x20]" x1=0x3 x19=0xbffff670 => x1=0x0
[15:45:31 986][libEncryptor.so 0x02dd8] [0c0040b9] 0x40002dd8: "ldr w12, [x0]" x0=0x4000b340 => w12=0x3ad043b #3.从跳转地址0x4000b340取encode
[15:45:31 987][libEncryptor.so 0x02de4] [8a150012] 0x40002de4: "and w10, w12, #0x3f" w12=0x3ad043b => w10=0x3b #解析opcode=0x3b
[15:45:31 987][libEncryptor.so 0x02e4c] [40001fd6] 0x40002e4c: "br x2" x2=0x40002f70

```

执行一条 opcode 前对应的汇编

![](https://bbs.kanxue.com/upload/attach/202510/887351_426D9CR843KWZ5C.jpg)

执行完一条 opcode 后对应的汇编

![](https://bbs.kanxue.com/upload/attach/202510/887351_AUGAKP62G4NYV37.jpg)

![](https://bbs.kanxue.com/upload/attach/202510/887351_BBT8RMAMKCRAP7E.jpg)

第一条 opcode 处设置 [x19, #-0x18] 为跳转地址，[x19, #-0x20]跳转标志 2，第二条 opcode 解析前判断 [x19, #-0x20] 为 2 则设置 [x19, #-0x20] 为 3，第二条 opcode 处理完成后，判断 [x19, #-0x20] 为 3，则会取出 [x19, #-0x18] 的地址覆盖 pc，其中如果跳转地址为 sub_2d1c（中转函数）则会用到 [x19, #-0x10] 作为 LR 地址使用，第三条 opcode 执行跳转地址，所以实现指令为 blr xn

opcode 0x2f 0x17

```
.text:000000000000333C                 AND             W13, W12, #0xFFF ; jumptable 0000000000002E4C case 47
.text:0000000000003340                 SUB             W14, W13, #0x6F ; 'o'
.text:0000000000003344                 EXTR            W14, W14, W14, #6
.text:0000000000003348                 CMP             W14, #0x3E ; switch 63 cases
.text:000000000000334C                 B.HI            def_3360 ; jumptable 0000000000003360 default case, cases 4,5,7,8,17-20,25,27-30,33,35,37,39,42,43,47,50,53-58
.text:0000000000003350                 ADRL            X15, jpt_3360
.text:0000000000003358                 LDRSW           X14, [X15,X14,LSL#2]
.text:000000000000335C                 ADD             X14, X14, X15
.text:0000000000003360                 BR              X14     ; switch jump
 
.text:0000000000003AD0 loc_3AD0                                ; CODE XREF: sub_2D28+638↑j
.text:0000000000003AD0                                         ; DATA XREF: .rodata:000000000000B008↓o ...
.text:0000000000003AD0                 AND             W11, W12, #0xFFF ; jumptable 0000000000003360 cases 12,23,31,60
.text:0000000000003AD4                 CMP             W11, #0x82E
.text:0000000000003AD8                 B.GT            loc_3DB8
.text:0000000000003ADC                 CMP             W11, #0x36F
.text:0000000000003AE0                 B.EQ            loc_4254
.text:0000000000003AE4                 CMP             W11, #0x62F
.text:0000000000003AE8                 B.NE            def_2E4C ; jumptable 0000000000002E4C default case, cases 1,6,7,9,12,19,24,25,28-30,37,38,46,52,53,57,60,61
.text:0000000000003AE8                                         ; jumptable 0000000000002E94 default case, cases 1,3-7,9-17,19-32,35-38,40-42,44-47,50,52-54
.text:0000000000003AE8                                         ; jumptable 0000000000002F18 default case, cases 4,6-14,16,18-25,27-41,43,46-57,59-62
.text:0000000000003AE8                                         ; jumptable 0000000000002F58 default case, cases 4,6-14,16-25,27-43,46-57,59-62
.text:0000000000003AE8                                         ; jumptable 0000000000002FB8 default case, cases 12,13,15-19,21,23-35,37-53,55-58
.text:0000000000003AE8                                         ; jumptable 0000000000003228 default case, cases 4,6-14,16,18-25,27-41,43,46-57,59-62
.text:0000000000003AE8                                         ; jumptable 0000000000003270 default case, cases 4,6-14,16-25,27-43,46-57,59-62
.text:0000000000003AE8                                         ; jumptable 0000000000003360 cases 0,3,9,15,46,61
.text:0000000000003AE8                                         ; jumptable 00000000000040C4 cases 4,6-14,16,18-25,27-41,43,46-57,59-62
.text:0000000000003AE8                                         ; jumptable 0000000000004104 default case, cases 4,6-14,16-25,27-43,46-57,59-62
.text:0000000000003AEC                 ADD             X11, X21, #8 ; 虚拟寄存器表
.text:0000000000003AF0                 LDR             X9, [X11,W9,UXTW#3] ; 取虚拟寄存器值w9
.text:0000000000003AF4                 LDR             X8, [X11,W8,UXTW#3] ; 取虚拟寄存器值w8
.text:0000000000003AF8                 ORR             X8, X8, X9 ; x8 = x8 | x9
.text:0000000000003AFC                 STR             X8, [X11,W10,UXTW#3] ; 将结果存入虚拟寄存器w10

```

也很明显，取出虚拟寄存器 w8，w9 与后存入 w10 虚拟寄存器，实现指令 orr xd, xn, xt

opcode 0x2f 0x2

```
.text:000000000000333C                 AND             W13, W12, #0xFFF ; jumptable 0000000000002E4C case 47
.text:0000000000003340                 SUB             W14, W13, #0x6F ; 'o'
.text:0000000000003344                 EXTR            W14, W14, W14, #6
.text:0000000000003348                 CMP             W14, #0x3E ; switch 63 cases
.text:000000000000334C                 B.HI            def_3360 ; jumptable 0000000000003360 default case, cases 4,5,7,8,17-20,25,27-30,33,35,37,39,42,43,47,50,53-58
.text:0000000000003350                 ADRL            X15, jpt_3360
.text:0000000000003358                 LDRSW           X14, [X15,X14,LSL#2]
.text:000000000000335C                 ADD             X14, X14, X15
.text:0000000000003360                 BR              X14     ; switch jump
 
.text:000000000000402C                 ADD             X11, X21, #8 ; jumptable 0000000000003360 case 2
.text:0000000000004030                 LDR             X9, [X11,W9,UXTW#3] ;取虚拟寄存器w9
.text:0000000000004034                 LDR             X8, [X11,W8,UXTW#3] ;取虚拟寄存器w8
.text:0000000000004038                 CMP             X9, X8 ;比较两个虚拟寄存器
.text:000000000000403C                 ADD             X8, X11, W10,UXTW#3 ;取虚拟寄存器w10偏移
.text:0000000000004040                 B.CC            loc_422C ;x9 < x8 则跳转  trace x8==x9
.text:0000000000004044
.text:0000000000004044 loc_4044                                ; CODE XREF: sub_2D28+1500↓j
.text:0000000000004044                 STR             XZR, [X8] ; 虚拟寄存器w10清0
.text:0000000000004048                 B               def_2E4C 
 
.text:000000000000422C                 MOV             W9, #1
.text:0000000000004230                 STR             X9, [X8] ;虚拟寄存器w10置1
.text:0000000000004234                 B               def_2E4C

```

1. 取出虚拟寄存器 w9 与 w8，2. 比较 w9 与 w8，如果 w9 <w8 则 w10 置 1，否则 w10 清 0，实现指令 cmp xd, xn 与 cset xd, lt

opcode 0x2f 0x19

```
.text:000000000000404C                 CMP             W13, #0x92E ; jumptable 0000000000003360 default case, cases 4,5,7,8,17-20,25,27-30,33,35,37,39,42,43,47,50,53-58
.text:0000000000004050                 B.LE            loc_43B0
 
.text:00000000000043B0                 CMP             W13, #0x6AE
.text:00000000000043B4                 B.LE            loc_43FC
.text:00000000000043B8                 CMP             W13, #0x76E
.text:00000000000043BC                 B.GT            loc_44C4
.text:00000000000043C0                 CMP             W13, #0x6AF
.text:00000000000043C4                 B.EQ            loc_45B0
 
.text:00000000000045B0                 CBNZ            X1, def_2E4C
.text:00000000000045B4                 ADD             X8, X21, W9,UXTW#3
.text:00000000000045B8                 LDR             X8, [X8,#8] ;取虚拟寄存器w9
.text:00000000000045BC                 ADD             X0, X0, #4
.text:00000000000045C0                 MOV             W9, #2
.text:00000000000045C4                 STP             X9, X8, [X19,#-0x20] ;[x19,#-0x18] = 0 [x19,#-0x20] = 2
.text:00000000000045C8                 STR             X0, [X21]
.text:00000000000045CC                 CBNZ            X0, loc_2DD4
 
.text:0000000000002DD4                 LDUR            X1, [X19,#-0x20]
.text:0000000000002DD8
.text:0000000000002DD8 loc_2DD8                                ; CODE XREF: sub_2D28+A8↑j
.text:0000000000002DD8                 LDR             W12, [X0]
.text:0000000000002DDC                 CMP             X1, #2
.text:0000000000002DE0                 B.EQ            loc_4248

```

取虚拟寄存器 w9 设置到跳转地址 [x19, #-0x18]，根据 trace 日志可以定位到 w9 为 0

![](https://bbs.kanxue.com/upload/attach/202510/887351_N7MF2STJK6HWQGH.jpg)

继续看下 loc_3c54

![](https://bbs.kanxue.com/upload/attach/202510/887351_7YMF2EWC76C3BVG.jpg)

接着看 loc_4654

![](https://bbs.kanxue.com/upload/attach/202510/887351_MUXDNYE7GHV7NVD.jpg)

出栈恢复上下文了，所以实现指令为 ret

opcode 0x3b

```
.text:0000000000002F70                 AND             W10, W12, #0x3F ; jumptable 0000000000002E4C cases 10,11,14,20,22,36,54,59
.text:0000000000002F74                 SUB             W10, W10, #0xA ; switch 50 cases
.text:0000000000002F78                 CMP             W10, #0x31
.text:0000000000002F7C                 B.HI            def_2E4C ; jumptable 0000000000002E4C default case, cases 1,6,7,9,12,19,24,25,28-30,37,38,46,52,53,57,60,61
.text:0000000000002F80                 UBFX            W11, W12, #6, #6 ; W11 = (W12 >> 6) & 0X3F  取W12[6..11]
.text:0000000000002F84                 AND             W12, W12, #0xF000 ; w12 = w12 & 0xf000 取w12[12..15]
.text:0000000000002F88                 BFXIL           W12, W18, #0x14, #0xC ; w11 = (w18 >> 20) & 0xfff
.text:0000000000002F8C                 ORR             W11, W12, W11 ; w11 = w12 | w11
.text:0000000000002F90                 ADD             X9, X21, W9,UXTW#3 ; x9 = x21 + w9 << 3
.text:0000000000002F94                 LDRSW           X10, [X25,X10,LSL#2] ; x10 = [x25 + x10 << 2]
.text:0000000000002F98                 ORR             W11, W11, W17,LSR#20 ; w11 = [w11 | w17 >> 20]
.text:0000000000002F9C                 LDR             X9, [X9,#8] ; x9 = [x9 + 8] ;取虚拟寄存器值w9
.text:0000000000002FA0                 ORR             W11, W11, W16,LSR#20 ; w11 = w11 | w16 >> 20 w16取字节码w12[28]
.text:0000000000002FA4                 ORR             W11, W11, W15,LSR#20 ; w11 = w11 | w15 >> 20 w15取字节码w12[29]
.text:0000000000002FA8                 ORR             W11, W11, W14,LSR#20 ; w11 = w11 | w14 >> 20 w14取字节码w12[30]
.text:0000000000002FAC                 ORR             W11, W11, W13,LSR#20 ; w11 = w11 | w13 >> 20 w13取字节码w12[31]
.text:0000000000002FB0                 ADD             X10, X10, X25 ; x10 = x10 + x25 计算跳转地址
.text:0000000000002FB4                 ADD             X9, X9, W11,SXTH ; x9 = x9 + w11_sxth ;虚拟寄存器w9加立即数 
 
.text:000000000000356C                 ADD             X8, X21, W8,UXTW#3 ; jumptable 0000000000002FB8 case 59
.text:0000000000003570                 LDR             X8, [X8,#8] ;取虚拟寄存器值w8
.text:0000000000003574                 STR             X8, [X9] ;将虚拟寄存器w8的值存入虚拟寄存器w9加立即数的地址中
.text:0000000000003578                 B               def_2E4C ;

```

实现指令 str xd, [xn, #imm]

opcode 0x36

```
.text:0000000000002F70 loc_2F70                                ; CODE XREF: sub_2D28+124↑j
.text:0000000000002F70                                         ; DATA XREF: .rodata:000000000000A988↓o ...
.text:0000000000002F70                 AND             W10, W12, #0x3F ; jumptable 0000000000002E4C cases 10,11,14,20,22,36,54,59
.text:0000000000002F74                 SUB             W10, W10, #0xA ; switch 50 cases
.text:0000000000002F78                 CMP             W10, #0x31
.text:0000000000002F7C                 B.HI            def_2E4C ; jumptable 0000000000002E4C default case, cases 1,6,7,9,12,19,24,25,28-30,37,38,46,52,53,57,60,61
.text:0000000000002F7C                                         ; jumptable 0000000000002E94 default case, cases 1,3-7,9-17,19-32,35-38,40-42,44-47,50,52-54
.text:0000000000002F7C                                         ; jumptable 0000000000002F18 default case, cases 4,6-14,16,18-25,27-41,43,46-57,59-62
.text:0000000000002F7C                                         ; jumptable 0000000000002F58 default case, cases 4,6-14,16-25,27-43,46-57,59-62
.text:0000000000002F7C                                         ; jumptable 0000000000002FB8 default case, cases 12,13,15-19,21,23-35,37-53,55-58
.text:0000000000002F7C                                         ; jumptable 0000000000003228 default case, cases 4,6-14,16,18-25,27-41,43,46-57,59-62
.text:0000000000002F7C                                         ; jumptable 0000000000003270 default case, cases 4,6-14,16-25,27-43,46-57,59-62
.text:0000000000002F7C                                         ; jumptable 0000000000003360 cases 0,3,9,15,46,61
.text:0000000000002F7C                                         ; jumptable 00000000000040C4 cases 4,6-14,16,18-25,27-41,43,46-57,59-62
.text:0000000000002F7C                                         ; jumptable 0000000000004104 default case, cases 4,6-14,16-25,27-43,46-57,59-62
.text:0000000000002F80                 UBFX            W11, W12, #6, #6 ; W11 = (W12 >> 6) & 0X3F  取W12[6..11]
.text:0000000000002F84                 AND             W12, W12, #0xF000 ; w12 = w12 & 0xf000 取w12[12..15]
.text:0000000000002F88                 BFXIL           W12, W18, #0x14, #0xC ; w11 = (w18 >> 20) & 0xfff
.text:0000000000002F8C                 ORR             W11, W12, W11 ; w11 = w12 | w11
.text:0000000000002F90                 ADD             X9, X21, W9,UXTW#3 ; x9 = x21 + w9 << 3
.text:0000000000002F94                 LDRSW           X10, [X25,X10,LSL#2] ; x10 = [x25 + x10 << 2]
.text:0000000000002F98                 ORR             W11, W11, W17,LSR#20 ; w11 = [w11 | w17 >> 20]
.text:0000000000002F9C                 LDR             X9, [X9,#8] ; 取虚拟寄存器w9值
.text:0000000000002FA0                 ORR             W11, W11, W16,LSR#20 ; w11 = w11 | w16 >> 20 w16取字节码w12[28]
.text:0000000000002FA4                 ORR             W11, W11, W15,LSR#20 ; w11 = w11 | w15 >> 20 w15取字节码w12[29]
.text:0000000000002FA8                 ORR             W11, W11, W14,LSR#20 ; w11 = w11 | w14 >> 20 w14取字节码w12[30]
.text:0000000000002FAC                 ORR             W11, W11, W13,LSR#20 ; w11 = w11 | w13 >> 20 w13取字节码w12[31]
.text:0000000000002FB0                 ADD             X10, X10, X25 ; x10 = x10 + x25 计算跳转地址
.text:0000000000002FB4                 ADD             X9, X9, W11,SXTH ; x9 = x9 + w11_sxth
.text:0000000000002FB8                 BR              X10     ; switch jump
 
loc_355C                                ; CODE XREF: sub_2D28+290↑j
.text:000000000000355C                                         ; DATA XREF: .rodata:000000000000ADD8↓o
.text:000000000000355C                 ADD             X8, X21, W8,UXTW#3 ; jumptable 0000000000002FB8 case 54
.text:0000000000003560                 LDR             W8, [X8,#8] ;取虚拟寄存器w8的值
.text:0000000000003564                 STR             W8, [X9] ;存w8到虚拟寄存器w9加偏移的地址
.text:0000000000003568                 B               def_2E4C

```

实现指令 str wd, [xn, #imm]

opcode 0xf

```
.text:0000000000002ECC                 CBNZ            X1, def_2E4C ; jumptable 0000000000002E4C cases 3,5,15,26,44,45,58,63
.text:0000000000002ED0                 AND             W10, W12, #0x3F
.text:0000000000002ED4                 SUB             W1, W10, #3 ; switch 61 cases
.text:0000000000002ED8                 CMP             W1, #0x3C
.text:0000000000002EDC                 B.HI            def_2E4C ; jumptable 0000000000002E4C default case, cases 1,6,7,9,12,19,24,25,28-30,37,38,46,52,53,57,60,61
.text:0000000000002EDC                                         ; jumptable 0000000000002E94 default case, cases 1,3-7,9-17,19-32,35-38,40-42,44-47,50,52-54
.text:0000000000002EDC                                         ; jumptable 0000000000002F18 default case, cases 4,6-14,16,18-25,27-41,43,46-57,59-62
.text:0000000000002EDC                                         ; jumptable 0000000000002F58 default case, cases 4,6-14,16-25,27-43,46-57,59-62
.text:0000000000002EDC                                         ; jumptable 0000000000002FB8 default case, cases 12,13,15-19,21,23-35,37-53,55-58
.text:0000000000002EDC                                         ; jumptable 0000000000003228 default case, cases 4,6-14,16,18-25,27-41,43,46-57,59-62
.text:0000000000002EDC                                         ; jumptable 0000000000003270 default case, cases 4,6-14,16-25,27-43,46-57,59-62
.text:0000000000002EDC                                         ; jumptable 0000000000003360 cases 0,3,9,15,46,61
.text:0000000000002EDC                                         ; jumptable 00000000000040C4 cases 4,6-14,16,18-25,27-41,43,46-57,59-62
.text:0000000000002EDC                                         ; jumptable 0000000000004104 default case, cases 4,6-14,16-25,27-43,46-57,59-62
.text:0000000000002EE0                 AND             W3, W12, #0xF000 ; w3 = w12 & 0xf000 w3取字节码w12[12..15]
.text:0000000000002EE4                 UBFX            W2, W12, #6, #6 ; w2 = w12 >> 6 & 3f w2取w12[6..11]
.text:0000000000002EE8                 BFXIL           W3, W18, #0x14, #0xC ; w3 = (W3 & 0xFFFFF000) | (w18 >> 0x20 & 0xfff) 取w18[20..31] w18取w12[26]
.text:0000000000002EEC                 ORR             W18, W3, W2 ; w18 = w3 | w2 w18包含w12[6..11 26]
.text:0000000000002EF0                 LDRSW           X1, [X27,X1,LSL#2]
.text:0000000000002EF4                 ORR             W17, W18, W17,LSR#20 ; w17 = w18 | w17 >> 20 w17取w12[27]
.text:0000000000002EF8                 ORR             W16, W17, W16,LSR#20 ; w16 = w17 | w16 >> 20 w16取w12[28]
.text:0000000000002EFC                 ORR             W15, W16, W15,LSR#20 ; w15 = w16 | w15 >> 20 w15取w12[29]
.text:0000000000002F00                 ORR             W14, W15, W14,LSR#20 ; w14 = w15 | w14 >> 20 w14取w12[30]
.text:0000000000002F04                 ORR             W13, W14, W13,LSR#20 ; w13 = w14 | w13 >> 20 w13取w12[31]
.text:0000000000002F08                 ADD             X14, X1, X27
.text:0000000000002F0C                 MOV             W10, WZR
.text:0000000000002F10                 MOV             X11, XZR
.text:0000000000002F14                 SBFIZ           W13, W13, #2, #0x10 ; W13 = (sign_extend_32(W13 & 0xFFFF) << 2) ->0011 1111 0000 0000 ？W13 = (sign_extend_32(W13 >> 2 & 0xFFFF))
.text:0000000000002F18                 BR              X14     ; switch jump
 
.text:0000000000002F1C loc_2F1C                                ; CODE XREF: sub_2D28+1F0↑j
.text:0000000000002F1C                                         ; DATA XREF: .rodata:000000000000AB48↓o ...
.text:0000000000002F1C                 ADD             X8, X21, W8,UXTW#3 ; jumptable 0000000000002F18 cases 5,15,44,58
.text:0000000000002F20                 LDR             X11, [X8,#8] ;取虚拟寄存器w8值
 
text:0000000000002F24                                         ; DATA XREF: .rodata:jpt_2F18↓o ...
.text:0000000000002F24                 AND             W8, W12, #0x3F ; jumptable 0000000000002F18 cases 3,26,45,63
.text:0000000000002F28                 SUB             W8, W8, #3 ; switch 61 cases
.text:0000000000002F2C                 CMP             W8, #0x3C
.text:0000000000002F30                 B.HI            def_2E4C 
 
.text:0000000000002F34                 ADRL            X10, jpt_2F58
.text:0000000000002F3C                 MOV             X12, X10
.text:0000000000002F40                 LDRSW           X8, [X12,X8,LSL#2]
.text:0000000000002F44                 ADD             X9, X21, W9,UXTW#3
.text:0000000000002F48                 LDR             X9, [X9,#8] ;取虚拟寄存器w9值
.text:0000000000002F4C                 ADD             W10, W13, #4 ;w10 = w13 + 4
.text:0000000000002F50                 ADD             X12, X8, X12
.text:0000000000002F54                 ADD             X8, X0, W10,SXTW ;x8 = x0(pc) + w10_sxtw ->pc + #imm
.text:0000000000002F58                 BR              X12     ; switch jump
 
.text:00000000000035C8                 CMP             X9, X11 ; jumptable 0000000000002F58 case 15
 
x9 == x11-->
.text:00000000000035CC                 B.NE            loc_3628
.text:00000000000035D0                 B               loc_2F64
 
.text:0000000000002F64                 MOV             W11, WZR
.text:0000000000002F68                 MOV             W9, WZR
.text:0000000000002F6C                 B               loc_3614
 
.text:0000000000003614                 MOV             W10, WZR
.text:0000000000003618                 MOV             W12, #1
.text:000000000000361C                 B               loc_363C
 
.text:000000000000363C                 CMP             W9, #0
.text:0000000000003640                 CSET            W13, NE ; if w9 != 0; w13 = 1 else w13 = 0
.text:0000000000003644                 AND             W11, W11, W13
.text:0000000000003648                 STUR            X8, [X19,#-0x18] ;;[x19, #-0x18] = x8
.text:000000000000364C                 TBNZ            W11, #0, loc_365C ;
.text:0000000000003650                 CBZ             W9, loc_365C ;w9为0
 
.text:000000000000365C                 CMP             W11, #0
.text:0000000000003660                 MOV             W8, #1
.text:0000000000003664                 CINC            X8, X8, EQ ;x8 = 2
.text:0000000000003668                 STUR            X8, [X19,#-0x20] ;[x19,#-0x20] = 2
 
.text:000000000000366C                 CMP             W10, #1
.text:0000000000003670                 B.LT            def_2E4C 
 
x9 != x11-->
.text:00000000000035C8                 CMP             X9, X11
.text:00000000000035CC                 B.NE            loc_3628
 
.text:0000000000003628                 MOV             W9, WZR
 
.text:000000000000362C                 MOV             W12, WZR
.text:0000000000003630                 MOV             W10, WZR
.text:0000000000003634                 ADD             X8, X0, #8 ;x8 = x0(pc) + 8
.text:0000000000003638                 MOV             W11, #1
 
.text:000000000000363C loc_363C                                ; CODE XREF: sub_2D28+890↑j
.text:000000000000363C                                         ; sub_2D28+8F4↑j
.text:000000000000363C                 CMP             W9, #0
.text:0000000000003640                 CSET            W13, NE
.text:0000000000003644                 AND             W11, W11, W13
.text:0000000000003648                 STUR            X8, [X19,#-0x18] ;[x19, #-0x18] = x8
.text:000000000000364C                 TBNZ            W11, #0, loc_365C
.text:0000000000003650                 CBZ             W9, loc_365C
.text:0000000000003654                 AND             W8, W12, W13
.text:0000000000003658                 CBZ             W8, loc_366C
.text:000000000000365C
.text:000000000000365C loc_365C                                ; CODE XREF: sub_2D28+924↑j
.text:000000000000365C                                         ; sub_2D28+928↑j
.text:000000000000365C                 CMP             W11, #0
.text:0000000000003660                 MOV             W8, #1
.text:0000000000003664                 CINC            X8, X8, EQ
.text:0000000000003668                 STUR            X8, [X19,#-0x20] ;[x19,#-0x20] = 2
.text:000000000000366C
.text:000000000000366C loc_366C                                ; CODE XREF: sub_2D28+930↑j
.text:000000000000366C                                         ; sub_2D28+1250↓j
.text:000000000000366C                 CMP             W10, #1
.text:0000000000003670                 B.LT            def_2E4C

```

1. 取虚拟寄存器 w9 与 w8，2.w8 与 w9 值进行比较，当相等时设置 [x19,#- 0x18] = pc + #imm [x19,#-0x20] = 2，不相等设置 [x19,#- 0x18] = pc + 8 [x19,#-0x20] = 2，实现指令 cmp xn, xt 与 b.eq label

opcode 0x2a

```
.text:00000000000030A0                 CBNZ            X1, def_2E4C ; jumptable 0000000000002E4C cases 17,42
.text:00000000000030A4                 AND             W11, W12, #0x3FFF000 ; w11=w12&0x3fff000 w11取w12[12..25]
.text:00000000000030A8                 UBFX            W10, W12, #6, #6 ; w10 = w12 >> 6 & 3f w10取w12[6..11]
.text:00000000000030AC                 BFXIL           W11, W18, #0x14, #0xC ; w11 = (w11 & 0xFFFFF000) | (w18 >> 0x14 & 0xfff) w11取w18[20..31] w18取w12[26]
.text:00000000000030B0                 ORR             W10, W11, W10
.text:00000000000030B4                 ORR             W10, W10, W17,LSR#20
.text:00000000000030B8                 ORR             W10, W10, W16,LSR#20
.text:00000000000030BC                 ORR             W10, W10, W15,LSR#20
.text:00000000000030C0                 ORR             W10, W10, W14,LSR#20
.text:00000000000030C4                 AND             W8, W12, #0x3F
.text:00000000000030C8                 ORR             W10, W10, W13,LSR#20
.text:00000000000030CC                 CMP             W8, #0x11
.text:00000000000030D0                 LSL             W12, W10, #2 ; w12 = w10 << 2
.text:00000000000030D4                 B.EQ            loc_36B0
.text:00000000000030D8                 CMP             W8, #0x2A ; '*'
.text:00000000000030DC                 B.EQ            loc_36B8
 
.text:00000000000036BC loc_36BC                                ; CODE XREF: sub_2D28+98C↑j
.text:00000000000036BC                 LDUR            X9, [X19,#-8] ; encode起始地址，函数入口处存储
.text:00000000000036C0                 MOV             W11, WZR
.text:00000000000036C4                 MOV             W10, WZR
.text:00000000000036C8                 ADD             X12, X9, W12,UXTW ;encode起始地址加上立即数
.text:00000000000036CC                 MOV             W9, #1
.text:00000000000036D0
.text:00000000000036D0 loc_36D0                                ; CODE XREF: sub_2D28+3F0↑j
.text:00000000000036D0                 CMP             W10, #0
.text:00000000000036D4                 CSET            W13, NE
.text:00000000000036D8                 AND             W11, W11, W13
.text:00000000000036DC                 STUR            X12, [X19,#-0x18] ;[X19,#-0x18] = x12
.text:00000000000036E0                 TBNZ            W11, #0, loc_36F0
.text:00000000000036E4                 CBZ             W10, loc_36F0
.text:00000000000036E8                 AND             W9, W9, W13
.text:00000000000036EC                 CBZ             W9, loc_3700
.text:00000000000036F0
.text:00000000000036F0 loc_36F0                                ; CODE XREF: sub_2D28+9B8↑j
.text:00000000000036F0                                         ; sub_2D28+9BC↑j
.text:00000000000036F0                 CMP             W11, #0
.text:00000000000036F4                 MOV             W9, #1
.text:00000000000036F8                 CINC            X9, X9, EQ
.text:00000000000036FC                 STUR            X9, [X19,#-0x20] ;[X19,#-0x20] = 2
.text:0000000000003700
.text:0000000000003700 loc_3700                                ; CODE XREF: sub_2D28+9C4↑j
.text:0000000000003700                                         ; sub_2D28+18FC↓j
.text:0000000000003700                 CMP             W8, #1
.text:0000000000003704                 B.LT            def_2E4C

```

主要将 vmp 编码的起始地址加上一个常数存到 [x19,#- 0x18], [x19,#- 0x20] 设置为 2，实现指令 b labe

opcode 0x4

```
.text:0000000000002FEC                 AND             W0, W12, #0xF000 ; jumptable 0000000000002E4C cases 4,32,56,62
.text:0000000000002FF0                 UBFX            W10, W12, #6, #6
.text:0000000000002FF4                 BFXIL           W0, W18, #0x14, #0xC
.text:0000000000002FF8                 ORR             W10, W0, W10
.text:0000000000002FFC                 ORR             W10, W10, W17,LSR#20
.text:0000000000003000                 ORR             W10, W10, W16,LSR#20
.text:0000000000003004                 ORR             W10, W10, W15,LSR#20
.text:0000000000003008                 AND             W11, W12, #0x3F
.text:000000000000300C                 ORR             W10, W10, W14,LSR#20
.text:0000000000003010                 CMP             W11, #0x37 ; '7'
.text:0000000000003014                 ORR             W10, W10, W13,LSR#20
.text:0000000000003018                 B.GT            loc_311C
.text:000000000000301C                 CMP             W11, #4
.text:0000000000003020                 B.EQ            loc_357C
 
.text:000000000000357C                 LSL             W9, W10, #0x10 ;w9 = w10 << 0x10 w10为立即数
 
.text:0000000000003580                 SXTW            X9, W9 ;x9 = w9_sxtw
.text:0000000000003584                 B               loc_3C1C
 
.text:0000000000003C1C                 ADD             X8, X21, W8,UXTW#3
.text:0000000000003C20
.text:0000000000003C20 loc_3C20                                ; CODE XREF: sub_2D28+1A0↑j
.text:0000000000003C20                                         ; sub_2D28+6EC↑j ...
.text:0000000000003C20                 STR             X9, [X8,#8] ;存x9到虚拟寄存器w8

```

实现指令 movz  wn, #imm, lsl #16 与 sxtw xn, wn

opcode 0x3e

```
.text:0000000000002FEC                 AND             W0, W12, #0xF000 ; jumptable 0000000000002E4C cases 4,32,56,62
.text:0000000000002FF0                 UBFX            W10, W12, #6, #6
.text:0000000000002FF4                 BFXIL           W0, W18, #0x14, #0xC
.text:0000000000002FF8                 ORR             W10, W0, W10
.text:0000000000002FFC                 ORR             W10, W10, W17,LSR#20
.text:0000000000003000                 ORR             W10, W10, W16,LSR#20
.text:0000000000003004                 ORR             W10, W10, W15,LSR#20
.text:0000000000003008                 AND             W11, W12, #0x3F
.text:000000000000300C                 ORR             W10, W10, W14,LSR#20
.text:0000000000003010                 CMP             W11, #0x37 ; '7'
.text:0000000000003014                 ORR             W10, W10, W13,LSR#20
.text:0000000000003018                 B.GT            loc_311C
 
.text:000000000000311C                 CMP             W11, #0x38 ; '8'
.text:0000000000003120                 B.EQ            loc_3588
.text:0000000000003124                 CMP             W11, #0x3E ; '>'
.text:0000000000003128                 B.NE            def_2E4C 
.text:000000000000312C                 ADD             X11, X21, #8
.text:0000000000003130                 LDR             X9, [X11,W9,UXTW#3] ;取虚拟寄存器值w9
.text:0000000000003134                 AND             W10, W10, #0xFFFF ;w10 = w10 & 0xffff
.text:0000000000003138                 ORR             X9, X9, X10 ;x9 = x9 | x10
.text:000000000000313C                 STR             X9, [X11,W8,UXTW#3] ;存到虚拟寄存器w8
.text:0000000000003140                 B               def_2E4C

```

实现指令 or xd, xn, #imm

*   **指令还原**
    

常规的做法是用 llvmlite 去做，这里介绍一种简单粗暴的处理方法，我们先用 python 脚本解析还原指令

```
import struct
 
g_op1 = -1
g_op2 = -1
g_op_type = -1
g_op_str = ""
g_bl= -1
g_pc=0
 
def sxtw_extend(value_32):
    value_32 = value_32 & 0xFFFFFFFF  
    if value_32 & 0x80000000:
        #extended = value_32 | 0xFFFFFFFF00000000 
        extended = -1 * (value_32 - 0x100000000)
        sign = "-"        
    else:
        extended = value_32 & 0x00000000FFFFFFFF 
        sign = "+"
    return extended,sign
 
def sxth_extend(value_32):
    low_16 = value_32 & 0xFFFF           
    if low_16 & 0x8000:              
        extended = -1 * (low_16 - 0x10000)
        sign = "-"
    else:
        extended = low_16
        sign = "+"
    return extended,sign
 
def vmp_interpreter(data):
    #init data
    global g_op1
    global g_op2
    global g_op_type
    global g_op_str
    global g_bl
    global g_pc
    g_op1 = -1;
    g_op2 = -1
    g_op_type = -1
    g_op_str = ""
    g_bl = -1
     
    #data handle
    g_op1 = data & 0x3f
    if g_op1 == 0x2f:
        g_op_type = 2
        g_op2 = ((data & 0xfff) - 0x6f) >> 6
    else:
        g_op_type = 1
 
    w9 = (data >> 21) & 0x1f
    w8 = (data >> 16) & 0x1f
    w13 = data & 0x80000000
    w14 = data & 0x40000000
    w15 = data & 0x20000000
    w16 = data & 0x10000000
    w17 = data & 0x8000000
    w18 = data & 0x4000000
 
    if g_op1 == 0x28:
        w0 = data & 0xf000
        w10 = (data >> 6) & 0x3f
        w0 = (w0 & 0xFFFFF000) | ((w18 >> 0x14) & 0xfff)
        w10 = w0 | w10
        w10 = w10 | w17 >> 20
        w10 = w10 | w16 >> 20
        w10 = w10 | w15 >> 20
        w10 = w10 | w14 >> 20
        w10 = w10 | w13 >> 20
        imm,sign = sxth_extend(w10)
        if sign == "-":
            #print("add x%d, x%d, #-0x%x" % (w8, w9, imm))
            g_op_str = "add x%d, x%d, #-0x%x" % (w8, w9, imm)
        else:
            #print("add x%d, x%d, #0x%x" % (w8, w9, imm))
            g_op_str = "add x%d, x%d, #0x%x" % (w8, w9, imm)
    elif g_op1 == 0x15:
        w0 = data & 0xf000
        w10 = (data >> 6) & 0x3f
        w0 = (w0 & 0xFFFFF000) | ((w18 >> 0x14) & 0xfff)
        w10 = w0 | w10
        w10 = w10 | w17 >> 20
        w10 = w10 | w16 >> 20
        w10 = w10 | w15 >> 20
        w10 = w10 | w14 >> 20
        w10 = w10 | w13 >> 20
        imm,sign = sxth_extend(w10)
        if sign == "-":
            g_op_str = "add x%d, x%d, #-0x%x" % (w8, w9, imm)
        else:
            g_op_str = "add x%d, x%d, #0x%x" % (w8, w9, imm)
    elif  g_op1 == 0x2:
        w11 = (data >> 6) & 0x3F
        w12 = data & 0xf000
        w12 = (w12 & 0xFFFFF000) | (w18 >> 0x14 & 0xfff)
        w11 = w11 | w12
        w11 = w11 | (w17 >> 20)
        w11 = w11 | (w16 >> 20)
        w11 = w11 | (w15 >> 20)
        w11 = w11 | (w14 >> 20)
        w11 = w11 | (w13 >> 20)
        imm,sign = sxth_extend(w11)
        if sign == "-":
            g_op_str = "ldr x%d, [x%d, #-0x%x]" % (w8, w9, imm)
        else:
            g_op_str = "ldr x%d, [x%d, #0x%x]" % (w8, w9, imm)
    elif  g_op1 == 0x3b:
        w11 = (data >> 6) & 0x3F
        w12 = data & 0xf000
        w12 = (w12 & 0xFFFFF000) | (w18 >> 0x14 & 0xfff)
        w11 = w11 | w12
        w11 = w11 | (w17 >> 20)
        w11 = w11 | (w16 >> 20)
        w11 = w11 | (w15 >> 20)
        w11 = w11 | (w14 >> 20)
        w11 = w11 | (w13 >> 20)
        imm,sign = sxth_extend(w11)
        if sign == "-":
            g_op_str = "str x%d, [x%d, #-0x%x]" % (w8, w9, imm)
        else:
            g_op_str = "str x%d, [x%d, #0x%x]" % (w8, w9, imm)
    elif  g_op1 == 0x36:
        w11 = (data >> 6) & 0x3F
        w12 = data & 0xf000
        w12 = (w12 & 0xFFFFF000) | (w18 >> 0x14 & 0xfff)
        w11 = w11 | w12
        w11 = w11 | (w17 >> 20)
        w11 = w11 | (w16 >> 20)
        w11 = w11 | (w15 >> 20)
        w11 = w11 | (w14 >> 20)
        w11 = w11 | (w13 >> 20)
        imm,sign = sxth_extend(w11)
        if sign == "-":
            g_op_str = "str w%d, [x%d, #-0x%x]" % (w8, w9, imm)
        else:
            g_op_str = "str w%d, [x%d, #0x%x]" % (w8, w9, imm)
    elif  g_op1 == 0x3e:
        w0 = data & 0xf000
        w10 = (data >> 6) & 0x3F
        w0 = (w0 & 0xFFFFF000) | (w18 >> 0x14 & 0xfff)
        w10 = w10 | w0
        w10 = w10 | (w17 >> 20)
        w10 = w10 | (w16 >> 20)
        w10 = w10 | (w15 >> 20)
        w10 = w10 | (w14 >> 20)
        w10 = w10 | (w13 >> 20)
        g_op_str = "orr x%d, x%d, #0x%x" % (w8, w9, w10 & 0xffff)
    elif  g_op1 == 0x4:
        w0 = data & 0xf000
        w10 = (data >> 6) & 0x3F
        w0 = (w0 & 0xFFFFF000) | (w18 >> 0x14 & 0xfff)
        w10 = w10 | w0
        w10 = w10 | (w17 >> 20)
        w10 = w10 | (w16 >> 20)
        w10 = w10 | (w15 >> 20)
        w10 = w10 | (w14 >> 20)
        w10 = w10 | (w13 >> 20)
        #w10 = w10 << 0x10
        #imm,sign = sxtw_extend(w10)
        #if sign == "-":
            #g_op_str = "mov x%d, #-0x%x" % (w8, imm)
        #else:
            #g_op_str = "mov x%d, #0x%x" % (w8, imm)
        imm = w10
        g_op_str = "movz  w%d, #0x%x, lsl #16 --> sxtw x%d, w%d" %(w8, imm, w8, w8)
    elif g_op1 == 0xf:
        w3 = data & 0xf000
        w2 = (data >> 6) & 0x3f
        w3 = (w3 & 0xFFFFF000) | ((w18 >> 0x14) & 0xfff)
        w18 = w3 | w2
        w17 = w18 | w17 >> 20
        w16 = w17 | w16 >> 20
        w15 = w16 | w15 >> 20
        w14 = w15 | w14 >> 20
        w13 = w14 | w13 >> 20
        imm,sign = sxth_extend((w13 & 0xffff) << 2)
        imm = imm + 4
        imm,sign = sxtw_extend(imm)
        imm = imm + g_pc
        g_op_str = "cmp x%d, x%d -> b.eq #0x%x" % (w9, w8, imm)
        g_bl = 1
    elif g_op1 == 0x2a:
        w11 = data & 0x3fff000
        w10 = (data >> 6) & 0x3f
        w11 = (w11 & 0xFFFFF000) | (w18 >> 0x14 & 0xfff)
        w10 = w10 | w11
        w10 = w10 | (w17 >> 20)
        w10 = w10 | (w16 >> 20)
        w10 = w10 | (w15 >> 20)
        w10 = w10 | (w14 >> 20)
        w10 = w10 | (w13 >> 20)
        w12 = w10 << 2
        x12,sign = sxtw_extend(w12)
        x12 = 0x4000b2c0 + x12
        g_op_str = "b #0x%x" % (x12)
        g_bl = 1
    elif g_op1 == 0x2f and g_op2 == 0x17:
        w10 = data & 0x3f
        w11 = data >> 0xb
        w14 = w11 & 2
        w2 = w11 & 4
        w14 = (w14 & 0xFFFFFFFE) | (data >> 31) & 0x1
        w14 = w14 | w2
        w2 = w11 & 8
        w10 = w11 & 0x10
        w2 = w14 | w2
        w10 = w2 | w10
        g_op_str = "orr x%d, x%d, x%d" % (w10, w8, w9)
    elif g_op1 == 0x2f and g_op2 == 0x2:
        g_op_str = "cmp x%d, x%d -> cset x%d, lt" % (w9, w8, w8)
    elif g_op1 == 0x2f and g_op2 == 0x30:
        w10 = data & 0x3f
        w11 = data >> 0xb
        w14 = w11 & 2
        w2 = w11 & 4
        w14 = (w14 & 0xFFFFFFFE) | (data >> 31) & 0x1
        w14 = w14 | w2
        w2 = w11 & 8
        w10 = w11 & 0x10
        w2 = w14 | w2
        w10 = w2 | w10
        imm,sign = sxtw_extend(w10)
        if w10 == w8 and imm == 0:
            g_op_str = "nop"
        else:
            g_op_str = "lsl x%d, x%d, #0x%x" %(w10, w8, imm)
    elif g_op1 == 0x2f and g_op2 == 0x26:
        w10 = data & 0x3f
        w11 = data >> 0xb
        w14 = w11 & 2
        w2 = w11 & 4
        w14 = (w14 & 0xFFFFFFFE) | (data >> 31) & 0x1
        w14 = w14 | w2
        w2 = w11 & 8
        w10 = w11 & 0x10
        w2 = w14 | w2
        w10 = w2 | w10
        g_op_str = "add x%d, x%d, x%d" % (w10, w8, w9)
    elif g_op1 == 0x2f and g_op2 == 0x15:
        g_op_str = "blr x%d" % (w9)
        g_bl = 1
    elif g_op1 == 0x2f and g_op2 == 0x19:
        g_op_str = "ret"
        g_bl = 1
 
def read_binary_with_index(input_path, output_path, offset, length):
    global g_op1
    global g_op2
    global g_op_type
    global g_op_str
    global g_bl
    global g_pc
    with open(input_path, 'rb') as src_file, open(output_path, 'w') as dest_file:
        content = []
        insert = 0
        flag = 0
        src_file.seek(offset)
        off = offset + 0x40000000
        #off = 0
        chunks = (length + 3) // 4
        index_counter = 0
         
        for _ in range(chunks):
            data = src_file.read(4)
            if not data:
                break
  
            little_endian_byte_array = struct.pack('I', data)[0])
            hex_data = little_endian_byte_array.hex()
            vmp_interpreter(int.from_bytes(little_endian_byte_array, 'big'))
            if flag == 0:
                content.append("index   address     encode    op1    op2        asm\n")
                flag = 1
            if g_op_type == 1:
                if insert == 1:
                    content.insert(index_counter, f"{index_counter:<3}   {hex(off)}   {hex_data}   {hex(g_op1):<4}          {g_op_str}\n")
                    insert = 0
                else:
                    content.append(f"{index_counter:<3}   {hex(off)}   {hex_data}   {hex(g_op1):<4}          {g_op_str}\n")
                #dest_file.write(f"{index_counter:<3}   {hex(off)}   {hex_data}   {hex(g_op1):<4}          {g_op_str}\n")
            else:
                if insert == 1:
                    content.insert(index_counter, f"{index_counter:<3}   {hex(off)}   {hex_data}   {hex(g_op1):<4}   {hex(g_op2):<4}   {g_op_str}\n")
                    insert = 0
                else:
                    content.append(f"{index_counter:<3}   {hex(off)}   {hex_data}   {hex(g_op1):<4}   {hex(g_op2):<4}   {g_op_str}\n")
                #dest_file.write(f"{index_counter:<3}   {hex(off)}   {hex_data}   {hex(g_op1):<4}   {hex(g_op2):<4}   {g_op_str}\n")
            off = off + 4
            g_pc = off
            if g_bl == 1:
                insert = 1
            index_counter = index_counter + 1
        dest_file.writelines(content)
 
if __name__ == "__main__":
    read_binary_with_index(
        input_path="libEncryptor.so",
        output_path="output.txt",
        offset=0xb2c0,
        length=0x2f4
    ) 
```

运行还原之后结果如下

```
index   address     encode    op1    op2        asm
0     0x4000b2c0   dfbdfc28   0x28          add x29, x29, #-0x210 #开栈并保存上下文
1     0x4000b2c4   23bf023b   0x3b          str x31, [x29, #0x208]
2     0x4000b2c8   23be003b   0x3b          str x30, [x29, #0x200]
3     0x4000b2cc   1fb70e3b   0x3b          str x23, [x29, #0x1f8]
4     0x4000b2d0   1fb60c3b   0x3b          str x22, [x29, #0x1f0]
5     0x4000b2d4   1fb50a3b   0x3b          str x21, [x29, #0x1e8]
6     0x4000b2d8   1fb4083b   0x3b          str x20, [x29, #0x1e0]
7     0x4000b2dc   1fb3063b   0x3b          str x19, [x29, #0x1d8]
8     0x4000b2e0   1fb2043b   0x3b          str x18, [x29, #0x1d0]
9     0x4000b2e4   1fb1023b   0x3b          str x17, [x29, #0x1c8]
10    0x4000b2e8   1fb0003b   0x3b          str x16, [x29, #0x1c0]
11    0x4000b2ec   00e0862f   0x2f   0x17   orr x16, x0, x7 #x16 = 0x40002d1c 中转函数地址
12    0x4000b2f0   008b0402   0x2           ldr x11, [x4, #0x10]
13    0x4000b2f4   008c0002   0x2           ldr x12, [x4, #0x0]
14    0x4000b2f8   00c20002   0x2           ldr x2, [x6, #0x0]
15    0x4000b2fc   00c30202   0x2           ldr x3, [x6, #0x8]
16    0x4000b300   00c50402   0x2           ldr x5, [x6, #0x10]
17    0x4000b304   00c70602   0x2           ldr x7, [x6, #0x18]
18    0x4000b308   00c80802   0x2           ldr x8, [x6, #0x20]
19    0x4000b30c   00c90a02   0x2           ldr x9, [x6, #0x28]
20    0x4000b310   008d0602   0x2           ldr x13, [x4, #0x18]
21    0x4000b314   00ca0c02   0x2           ldr x10, [x6, #0x30]
22    0x4000b318   00c60e02   0x2           ldr x6, [x6, #0x38]
23    0x4000b31c   00940202   0x2           ldr x20, [x4, #0x8]
24    0x4000b320   06810da8   0x28          add x1, x20, #0x76
25    0x4000b324   0ba1003b   0x3b          str x1, [x29, #0x80]
26    0x4000b328   01a40002   0x2           ldr x4, [x13, #0x0]
27    0x4000b32c   808100ef   0x2f   0x2    cmp x4, x1 -> cset x1, lt
29    0x4000b334   00000c6f   0x2f   0x30   nop
28    0x4000b330   002000cf   0xf           cmp x1, x0 -> b.eq #0x4000b340
31    0x4000b33c   01a0003b   0x3b          str x0, [x13, #0x0]
30    0x4000b338   08000c6a   0x2a          b #0x4000b584
32    0x4000b340   03ad043b   0x3b          str x13, [x29, #0x10]
33    0x4000b344   f801f844   0x4           movz  w1, #0xffa1, lsl #16 --> sxtw x1, w1
34    0x4000b348   6021083e   0x3e          orr x1, x1, #0x620
35    0x4000b34c   00c129ef   0x2f   0x26   add x4, x1, x6
36    0x4000b350   03a40e3b   0x3b          str x4, [x29, #0x38]
37    0x4000b354   014129ef   0x2f   0x26   add x4, x1, x10
38    0x4000b358   03a40c3b   0x3b          str x4, [x29, #0x30]
39    0x4000b35c   012129ef   0x2f   0x26   add x4, x1, x9
40    0x4000b360   03a40a3b   0x3b          str x4, [x29, #0x28]
41    0x4000b364   010129ef   0x2f   0x26   add x4, x1, x8
42    0x4000b368   03a4063b   0x3b          str x4, [x29, #0x18]
43    0x4000b36c   00e129ef   0x2f   0x26   add x4, x1, x7
44    0x4000b370   03a4023b   0x3b          str x4, [x29, #0x8]
45    0x4000b374   00a1b9ef   0x2f   0x26   add x22, x1, x5
46    0x4000b378   8061b9ef   0x2f   0x26   add x23, x1, x3
47    0x4000b37c   0041f9ef   0x2f   0x26   add x30, x1, x2
48    0x4000b380   00110828   0x28          add x17, x0, #0x20
49    0x4000b384   1bb10c3b   0x3b          str x17, [x29, #0x1b0]
50    0x4000b388   1ba50c28   0x28          add x5, x29, #0x1b0
51    0x4000b38c   03c0262f   0x2f   0x17   orr x4, x0, x30  #x30 = 0x40002c50 中转后的实际运行函数地址
52    0x4000b390   8200c62f   0x2f   0x17   orr x25, x0, x16
53    0x4000b394   03ab083b   0x3b          str x11, [x29, #0x20]
55    0x4000b39c   03ac003b   0x3b          str x12, [x29, #0x0]
54    0x4000b398   8320f5af   0x2f   0x15   blr x25  #sub_2D1C ->sub_2c50  malloc
56    0x4000b3a0   1bb10a3b   0x3b          str x17, [x29, #0x1a8]
57    0x4000b3a4   1bb50e02   0x2           ldr x21, [x29, #0x1b8]
58    0x4000b3a8   1bb5083b   0x3b          str x21, [x29, #0x1a0]
59    0x4000b3ac   1ba50828   0x28          add x5, x29, #0x1a0
60    0x4000b3b0   8200c62f   0x2f   0x17   orr x25, x0, x16
62    0x4000b3b8   02e0262f   0x2f   0x17   orr x4, x0, x23 #x4 = 0x40002c6c 中转后的实际运行函数地址
61    0x4000b3b4   8320f5af   0x2f   0x15   blr x25 #sub_2D1C ->sub_2c6c  random随机因子
63    0x4000b3bc   1ba50428   0x28          add x5, x29, #0x190
64    0x4000b3c0   00170428   0x28          add x23, x0, #0x10
65    0x4000b3c4   1bb7043b   0x3b          str x23, [x29, #0x190]
66    0x4000b3c8   8200c62f   0x2f   0x17   orr x25, x0, x16
68    0x4000b3d0   03c0262f   0x2f   0x17   orr x4, x0, x30 #x4=0x40002c50
67    0x4000b3cc   8320f5af   0x2f   0x15   blr x25 #sub_2D1C ->sub_2c50 malloc
69    0x4000b3d4   1ba50028   0x28          add x5, x29, #0x180
70    0x4000b3d8   1bb7003b   0x3b          str x23, [x29, #0x180]
71    0x4000b3dc   1bb30602   0x2           ldr x19, [x29, #0x198]
72    0x4000b3e0   8200c62f   0x2f   0x17   orr x25, x0, x16
74    0x4000b3e8   03c0262f   0x2f   0x17   orr x4, x0, x30
73    0x4000b3e4   8320f5af   0x2f   0x15   blr x25 #sub_2D1C ->sub_2c50 malloc
75    0x4000b3ec   17a50428   0x28          add x5, x29, #0x150
76    0x4000b3f0   17b5043b   0x3b          str x21, [x29, #0x150]
77    0x4000b3f4   17b1063b   0x3b          str x17, [x29, #0x158]
78    0x4000b3f8   17b3083b   0x3b          str x19, [x29, #0x160]
79    0x4000b3fc   17b70a3b   0x3b          str x23, [x29, #0x168]
80    0x4000b400   17b70e3b   0x3b          str x23, [x29, #0x178]
81    0x4000b404   1bb20202   0x2           ldr x18, [x29, #0x188]
82    0x4000b408   17b20c3b   0x3b          str x18, [x29, #0x170]
83    0x4000b40c   8200c62f   0x2f   0x17   orr x25, x0, x16
85    0x4000b414   02c0262f   0x2f   0x17   orr x4, x0, x22
84    0x4000b410   8320f5af   0x2f   0x15   blr x25 #sub_2D1C ->sub_2CA4 另外一套VMP，原理一致
86    0x4000b418   06970028   0x28          add x23, x20, #0x40
87    0x4000b41c   17a50028   0x28          add x5, x29, #0x140
88    0x4000b420   17b7003b   0x3b          str x23, [x29, #0x140]
89    0x4000b424   8200c62f   0x2f   0x17   orr x25, x0, x16
91    0x4000b42c   03c0262f   0x2f   0x17   orr x4, x0, x30
90    0x4000b428   8320f5af   0x2f   0x15   blr x25 #sub_2D1C->sub_2c50 malloc
92    0x4000b430   07be0028   0x28          add x30, x29, #0x40
93    0x4000b434   17b60202   0x2           ldr x22, [x29, #0x148]
94    0x4000b438   13b40c3b   0x3b          str x20, [x29, #0x130]
95    0x4000b43c   03b10002   0x2           ldr x17, [x29, #0x0]
96    0x4000b440   13b10a3b   0x3b          str x17, [x29, #0x128]
97    0x4000b444   13be0e3b   0x3b          str x30, [x29, #0x138]
98    0x4000b448   03a40202   0x2           ldr x4, [x29, #0x8]
99    0x4000b44c   8200c62f   0x2f   0x17   orr x25, x0, x16
101   0x4000b454   13a50a28   0x28          add x5, x29, #0x128
100   0x4000b450   8320f5af   0x2f   0x15   blr x25 #sub_2D1C->sub_6da8 加密相关 sha相关
102   0x4000b458   13a50428   0x28          add x5, x29, #0x110
103   0x4000b45c   04010028   0x28          add x1, x0, #0x40
104   0x4000b460   13be063b   0x3b          str x30, [x29, #0x118]
105   0x4000b464   13b6043b   0x3b          str x22, [x29, #0x110]
106   0x4000b468   13a1083b   0x3b          str x1, [x29, #0x120]
107   0x4000b46c   03be0602   0x2           ldr x30, [x29, #0x18]
108   0x4000b470   8200c62f   0x2f   0x17   orr x25, x0, x16
110   0x4000b478   03c0262f   0x2f   0x17   orr x4, x0, x30
109   0x4000b474   8320f5af   0x2f   0x15   blr x25 #sub_2D1C->sub_2cc8 memcpy
111   0x4000b47c   0fa50e28   0x28          add x5, x29, #0xf8
112   0x4000b480   06c10028   0x28          add x1, x22, #0x40
113   0x4000b484   13b1003b   0x3b          str x17, [x29, #0x100]
114   0x4000b488   8240862f   0x2f   0x17   orr x17, x0, x18
115   0x4000b48c   0fa10e3b   0x3b          str x1, [x29, #0xf8]
116   0x4000b490   13b4023b   0x3b          str x20, [x29, #0x108]
117   0x4000b494   0260a62f   0x2f   0x17   orr x20, x0, x19
118   0x4000b498   8200c62f   0x2f   0x17   orr x25, x0, x16
120   0x4000b4a0   03c0262f   0x2f   0x17   orr x4, x0, x30
119   0x4000b49c   8320f5af   0x2f   0x15   blr x25 #sub_2D1C->sub_2cc8 memcpy
121   0x4000b4a4   00010815   0x15          add x1, x0, #0x20
122   0x4000b4a8   0fb50a3b   0x3b          str x21, [x29, #0xe8]
123   0x4000b4ac   03b30802   0x2           ldr x19, [x29, #0x20]
124   0x4000b4b0   0fb3083b   0x3b          str x19, [x29, #0xe0]
125   0x4000b4b4   0fa10c36   0x36          str w1, [x29, #0xf0]
126   0x4000b4b8   03a40a02   0x2           ldr x4, [x29, #0x28]
127   0x4000b4bc   8200c62f   0x2f   0x17   orr x25, x0, x16
129   0x4000b4c4   0fa50828   0x28          add x5, x29, #0xe0
128   0x4000b4c0   8320f5af   0x2f   0x15   blr x25 #sub_2D1C->sub_2cd8 memcpy
130   0x4000b4c8   026109a8   0x28          add x1, x19, #0x26
131   0x4000b4cc   00020415   0x15          add x2, x0, #0x10
132   0x4000b4d0   0ba30028   0x28          add x3, x29, #0x80
133   0x4000b4d4   0bb40a3b   0x3b          str x20, [x29, #0xa8]
134   0x4000b4d8   0ba20c36   0x36          str w2, [x29, #0xb0]
135   0x4000b4dc   0bb10e3b   0x3b          str x17, [x29, #0xb8]
136   0x4000b4e0   0fb6003b   0x3b          str x22, [x29, #0xc0]
137   0x4000b4e4   0fb7023b   0x3b          str x23, [x29, #0xc8]
138   0x4000b4e8   0fa1043b   0x3b          str x1, [x29, #0xd0]
139   0x4000b4ec   0fa3063b   0x3b          str x3, [x29, #0xd8]
140   0x4000b4f0   0ba10002   0x2           ldr x1, [x29, #0x80]
141   0x4000b4f4   fc21f6a8   0x28          add x1, x1, #-0x26
142   0x4000b4f8   0ba1003b   0x3b          str x1, [x29, #0x80]
143   0x4000b4fc   03a40c02   0x2           ldr x4, [x29, #0x30]
144   0x4000b500   8200c62f   0x2f   0x17   orr x25, x0, x16
146   0x4000b508   0ba50a28   0x28          add x5, x29, #0xa8
145   0x4000b504   8320f5af   0x2f   0x15   blr x25 #sub_2D1C->sub_2cf8 aes相关
147   0x4000b50c   0ba20002   0x2           ldr x2, [x29, #0x80]
149   0x4000b514   00000c6f   0x2f   0x30   nop
148   0x4000b510   0040014f   0xf           cmp x2, x0 -> b.eq #0x4000b528
150   0x4000b518   004109a8   0x28          add x1, x2, #0x26
151   0x4000b51c   03a20402   0x2           ldr x2, [x29, #0x10]
153   0x4000b524   0041003b   0x3b          str x1, [x2, #0x0]
152   0x4000b520   0800072a   0x2a          b #0x4000b530
154   0x4000b528   03a10402   0x2           ldr x1, [x29, #0x10]
155   0x4000b52c   0020003b   0x3b          str x0, [x1, #0x0]
156   0x4000b530   0bb6083b   0x3b          str x22, [x29, #0xa0]
157   0x4000b534   0ba50828   0x28          add x5, x29, #0xa0
158   0x4000b538   03b20e02   0x2           ldr x18, [x29, #0x38]
159   0x4000b53c   8200c62f   0x2f   0x17   orr x25, x0, x16
161   0x4000b544   0240262f   0x2f   0x17   orr x4, x0, x18
160   0x4000b540   8320f5af   0x2f   0x15   blr x25 #sub_2D1C->sub_2d14 free
162   0x4000b548   0bb1063b   0x3b          str x17, [x29, #0x98]
163   0x4000b54c   0ba50628   0x28          add x5, x29, #0x98
164   0x4000b550   8200c62f   0x2f   0x17   orr x25, x0, x16
166   0x4000b558   0240262f   0x2f   0x17   orr x4, x0, x18
165   0x4000b554   8320f5af   0x2f   0x15   blr x25 #sub_2D1C->sub_2d14 free
167   0x4000b55c   0ba50428   0x28          add x5, x29, #0x90
168   0x4000b560   0bb4043b   0x3b          str x20, [x29, #0x90]
169   0x4000b564   8200c62f   0x2f   0x17   orr x25, x0, x16
171   0x4000b56c   0240262f   0x2f   0x17   orr x4, x0, x18
170   0x4000b568   8320f5af   0x2f   0x15   blr x25 #sub_2D1C->sub_2d14 free
172   0x4000b570   0ba50228   0x28          add x5, x29, #0x88
173   0x4000b574   0bb5023b   0x3b          str x21, [x29, #0x88]
174   0x4000b578   8200c62f   0x2f   0x17   orr x25, x0, x16
176   0x4000b580   0240262f   0x2f   0x17   orr x4, x0, x18
175   0x4000b57c   8320f5af   0x2f   0x15   blr x25 #sub_2D1C->sub_2d14 free
177   0x4000b584   1fb00002   0x2           ldr x16, [x29, #0x1c0] #恢复栈平衡并恢复上下文
178   0x4000b588   1fb10202   0x2           ldr x17, [x29, #0x1c8]
179   0x4000b58c   1fb20402   0x2           ldr x18, [x29, #0x1d0]
180   0x4000b590   1fb30602   0x2           ldr x19, [x29, #0x1d8]
181   0x4000b594   1fb40802   0x2           ldr x20, [x29, #0x1e0]
182   0x4000b598   1fb50a02   0x2           ldr x21, [x29, #0x1e8]
183   0x4000b59c   1fb60c02   0x2           ldr x22, [x29, #0x1f0]
184   0x4000b5a0   1fb70e02   0x2           ldr x23, [x29, #0x1f8]
185   0x4000b5a4   23be0002   0x2           ldr x30, [x29, #0x200]
186   0x4000b5a8   23bf0202   0x2           ldr x31, [x29, #0x208]
188   0x4000b5b0   23bd0428   0x28          add x29, x29, #0x210
187   0x4000b5ac   03e006af   0x2f   0x19   ret

```

由于样本这种比较简单的 vmp 结合 trace 直接就能分析，但是对于复杂的样本可能就不是这么简单了，所以还是要转成 bin，放到 IDA 里面，下面介绍一种比较简单的做法，我们先写一个测试程序

```
#include  int replace() {
    int a = 1;
    int b = 1;
    return a + b;
}
 
int main(void) {
    replace();
    return 0;
} 
```

用 ndk 的工具链编译成汇编代码 aarch64-linux-android21-clang main.c -S

```
    .text
    .file   "main.c"
    .globl  replace                         // -- Begin function replace
    .p2align    2
    .type   replace,@function
replace:                                // @replace
    .cfi_startproc
// %bb.0:
    sub sp, sp, #16
    .cfi_def_cfa_offset 16
    mov w8, #1
    str w8, [sp, #12]
    str w8, [sp, #8]
    ldr w8, [sp, #12]
    ldr w9, [sp, #8]
    add w0, w8, w9
    add sp, sp, #16
    ret
.Lfunc_end0:
    .size   replace, .Lfunc_end0-replace
    .cfi_endproc
                                        // -- End function
    .globl  main                            // -- Begin function main
    .p2align    2
    .type   main,@function
main:                                   // @main
    .cfi_startproc
// %bb.0:
    sub sp, sp, #32
    stp x29, x30, [sp, #16]             // 16-byte Folded Spill
    add x29, sp, #16
    .cfi_def_cfa w29, 16
    .cfi_offset w30, -8
    .cfi_offset w29, -16
    mov w8, wzr
    str w8, [sp, #8]                    // 4-byte Folded Spill
    stur    wzr, [x29, #-4]
    bl  replace
    ldr w0, [sp, #8]                    // 4-byte Folded Reload
    ldp x29, x30, [sp, #16]             // 16-byte Folded Reload
    add sp, sp, #32
    ret
.Lfunc_end1:
    .size   main, .Lfunc_end1-main
    .cfi_endproc
                                        // -- End function
    .ident  "Android (9352603, based on r450784d1) clang version 14.0.7 (https://android.googlesource.com/toolchain/llvm-project 4c603efb0cca074e9238af8b4106c30add4418f6)"
    .section    ".note.GNU-stack","",@progbits

```

接着把 replace 函数里面替换成我们的还原的汇编函数稍微适配一下

```
    .text
    .file   "main.c"
    .globl  replace                         // -- Begin function replace
    .p2align    2
    .type   replace,@function
replace:                                // @replace
    .cfi_startproc
// %bb.0:
    add x29, x29, #-0x210
    str x31, [x29, #0x208]
    str x30, [x29, #0x200]
    str x23, [x29, #0x1f8]
    str x22, [x29, #0x1f0]
    str x21, [x29, #0x1e8]
    str x20, [x29, #0x1e0]
    str x19, [x29, #0x1d8]
    str x18, [x29, #0x1d0]
    str x17, [x29, #0x1c8]
    str x16, [x29, #0x1c0]
    orr x16, x0, x7
    ldr x11, [x4, #0x10]
    ldr x12, [x4, #0x0]
    ldr x2, [x6, #0x0]
    ldr x3, [x6, #0x8]
    ldr x5, [x6, #0x10]
    ldr x7, [x6, #0x18]
    ldr x8, [x6, #0x20]
    ldr x9, [x6, #0x28]
    ldr x13, [x4, #0x18]
    ldr x10, [x6, #0x30]
    ldr x6, [x6, #0x38]
    ldr x20, [x4, #0x8]
    add x1, x20, #0x76
    str x1, [x29, #0x80]
    ldr x4, [x13, #0x0]
    cmp x4, x1
    cset x1, lt
    nop
    cmp x1, x0
    b.eq #0xc
    str x0, [x13, #0x0]
    b #0x254
    str x13, [x29, #0x10]
    movz  w1, #0xffa1, lsl #16
    sxtw x1, w1
    mov x23, #0x620
    orr x1, x1, x23
    add x4, x1, x6
    str x4, [x29, #0x38]
    add x4, x1, x10
    str x4, [x29, #0x30]
    add x4, x1, x9
    str x4, [x29, #0x28]
    add x4, x1, x8
    str x4, [x29, #0x18]
    add x4, x1, x7
    str x4, [x29, #0x8]
    add x22, x1, x5
    add x23, x1, x3
    add x30, x1, x2
    add x17, x0, #0x20
    str x17, [x29, #0x1b0]
    add x5, x29, #0x1b0
    orr x4, x0, x30
    orr x25, x0, x16
    str x11, [x29, #0x20]
    str x12, [x29, #0x0]
    blr x25
    str x17, [x29, #0x1a8]
    ldr x21, [x29, #0x1b8]
    str x21, [x29, #0x1a0]
    add x5, x29, #0x1a0
    orr x25, x0, x16
    orr x4, x0, x23
    blr x25
    add x5, x29, #0x190
    add x23, x0, #0x10
    str x23, [x29, #0x190]
    orr x25, x0, x16
    orr x4, x0, x30
    blr x25
    add x5, x29, #0x180
    str x23, [x29, #0x180]
    ldr x19, [x29, #0x198]
    orr x25, x0, x16
    orr x4, x0, x30
    blr x25
    add x5, x29, #0x150
    str x21, [x29, #0x150]
    str x17, [x29, #0x158]
    str x19, [x29, #0x160]
    str x23, [x29, #0x168]
    str x23, [x29, #0x178]
    ldr x18, [x29, #0x188]
    str x18, [x29, #0x170]
    orr x25, x0, x16
    orr x4, x0, x22
    blr x25
    add x23, x20, #0x40
    add x5, x29, #0x140
    str x23, [x29, #0x140]
    orr x25, x0, x16
    orr x4, x0, x30
    blr x25
    add x30, x29, #0x40
    ldr x22, [x29, #0x148]
    str x20, [x29, #0x130]
    ldr x17, [x29, #0x0]
    str x17, [x29, #0x128]
    str x30, [x29, #0x138]
    ldr x4, [x29, #0x8]
    orr x25, x0, x16
    add x5, x29, #0x128
    blr x25
    add x5, x29, #0x110
    add x1, x0, #0x40
    str x30, [x29, #0x118]
    str x22, [x29, #0x110]
    str x1, [x29, #0x120]
    ldr x30, [x29, #0x18]
    orr x25, x0, x16
    orr x4, x0, x30
    blr x25
    add x5, x29, #0xf8
    add x1, x22, #0x40
    str x17, [x29, #0x100]
    orr x17, x0, x18
    str x1, [x29, #0xf8]
    str x20, [x29, #0x108]
    orr x20, x0, x19
    orr x25, x0, x16
    orr x4, x0, x30
    blr x25
    add x1, x0, #0x20
    str x21, [x29, #0xe8]
    ldr x19, [x29, #0x20]
    str x19, [x29, #0xe0]
    str w1, [x29, #0xf0]
    ldr x4, [x29, #0x28]
    orr x25, x0, x16
    add x5, x29, #0xe0
    blr x25
    add x1, x19, #0x26
    add x2, x0, #0x10
    add x3, x29, #0x80
    str x20, [x29, #0xa8]
    str w2, [x29, #0xb0]
    str x17, [x29, #0xb8]
    str x22, [x29, #0xc0]
    str x23, [x29, #0xc8]
    str x1, [x29, #0xd0]
    str x3, [x29, #0xd8]
    ldr x1, [x29, #0x80]
    add x1, x1, #-0x26
    str x1, [x29, #0x80]
    ldr x4, [x29, #0x30]
    orr x25, x0, x16
    add x5, x29, #0xa8
    blr x25
    ldr x2, [x29, #0x80]
    nop
    cmp x2, x0
    b.eq #0x14
    add x1, x2, #0x26
    ldr x2, [x29, #0x10]
    str x1, [x2, #0x0]
    b #0xc
    ldr x1, [x29, #0x10]
    str x0, [x1, #0x0]
    str x22, [x29, #0xa0]
    add x5, x29, #0xa0
    ldr x18, [x29, #0x38]
    orr x25, x0, x16
    orr x4, x0, x18
    blr x25
    str x17, [x29, #0x98]
    add x5, x29, #0x98
    orr x25, x0, x16
    orr x4, x0, x18
    blr x25
    add x5, x29, #0x90
    str x20, [x29, #0x90]
    orr x25, x0, x16
    orr x4, x0, x18
    blr x25
    add x5, x29, #0x88
    str x21, [x29, #0x88]
    orr x25, x0, x16
    orr x4, x0, x18
    blr x25
    ldr x16, [x29, #0x1c0]
    ldr x17, [x29, #0x1c8]
    ldr x18, [x29, #0x1d0]
    ldr x19, [x29, #0x1d8]
    ldr x20, [x29, #0x1e0]
    ldr x21, [x29, #0x1e8]
    ldr x22, [x29, #0x1f0]
    ldr x23, [x29, #0x1f8]
    ldr x30, [x29, #0x200]
    ldr x31, [x29, #0x208]
    add x29, x29, #0x210
    ret
.Lfunc_end0:
    .size   replace, .Lfunc_end0-replace
    .cfi_endproc
                                        // -- End function
    .globl  main                            // -- Begin function main
    .p2align    2
    .type   main,@function
main:                                   // @main
    .cfi_startproc
// %bb.0:
    sub sp, sp, #32
    stp x29, x30, [sp, #16]             // 16-byte Folded Spill
    add x29, sp, #16
    .cfi_def_cfa w29, 16
    .cfi_offset w30, -8
    .cfi_offset w29, -16
    mov w8, wzr
    str w8, [sp, #8]                    // 4-byte Folded Spill
    stur    wzr, [x29, #-4]
    bl  replace
    ldr w0, [sp, #8]                    // 4-byte Folded Reload
    ldp x29, x30, [sp, #16]             // 16-byte Folded Reload
    add sp, sp, #32
    ret
.Lfunc_end1:
    .size   main, .Lfunc_end1-main
    .cfi_endproc
                                        // -- End function
    .ident  "Android (9352603, based on r450784d1) clang version 14.0.7 (https://android.googlesource.com/toolchain/llvm-project 4c603efb0cca074e9238af8b4106c30add4418f6)"
    .section    ".note.GNU-stack","",@progbits

```

接续编译一下 aarch64-linux-android21-clang main.s 会生成 a.out 的二进制文件，就可以导入到 IDA 了，f5 可以看到伪 C 代码

```
__int64 __fastcall replace(__int64 result, __int64 a2, __int64 a3, __int64 a4, __int64 *a5, __int64 a6, _QWORD *a7, __int64 a8)
{
  __int64 v8; // x16
  __int64 v9; // x17
  __int64 v10; // x18
  __int64 v11; // x19
  __int64 v12; // x20
  __int64 v13; // x21
  __int64 v14; // x22
  __int64 v15; // x23
  __int64 v16; // x29
  __int64 v17; // x30
  __int64 v18; // x29
  __int64 v19; // x16
  __int64 v20; // x11
  __int64 v21; // x12
  __int64 v22; // x7
  __int64 v23; // x8
  __int64 v24; // x9
  __int64 *v25; // x13
  __int64 v26; // x10
  __int64 v27; // x6
  __int64 v28; // x20
  __int64 v29; // x0
  __int64 v30; // x17
  __int64 v31; // x21
  __int64 v32; // x16
  __int64 v33; // x0
  __int64 v34; // x23
  __int64 v35; // x16
  __int64 v36; // x0
  __int64 v37; // x19
  __int64 v38; // x16
  __int64 v39; // x0
  __int64 v40; // x17
  __int64 v41; // x16
  __int64 v42; // x0
  __int64 v43; // x23
  __int64 v44; // x16
  __int64 v45; // x0
  __int64 v46; // x22
  __int64 v47; // x16
  __int64 v48; // x0
  __int64 v49; // x16
  __int64 v50; // x0
  __int64 v51; // x17
  __int64 v52; // x20
  __int64 v53; // x16
  __int64 v54; // x0
  __int64 v55; // x19
  __int64 v56; // x16
  __int64 v57; // x0
  __int64 v58; // x17
  __int64 v59; // x16
  __int64 v60; // x0
  __int64 v61; // x16
  __int64 v62; // x2
  __int64 v63; // x0
  __int64 v64; // x17
  __int64 v65; // x16
  __int64 v66; // x0
  __int64 v67; // x16
  __int64 v68; // x0
  __int64 v69; // x16
 
  v18 = v16 - 528;
  *(_QWORD *)(v18 + 520) = 0LL;
  *(_QWORD *)(v18 + 512) = v17;
  *(_QWORD *)(v18 + 504) = v15;
  *(_QWORD *)(v18 + 496) = v14;
  *(_QWORD *)(v18 + 488) = v13;
  *(_QWORD *)(v18 + 480) = v12;
  *(_QWORD *)(v18 + 472) = v11;
  *(_QWORD *)(v18 + 464) = v10;
  *(_QWORD *)(v18 + 456) = v9;
  *(_QWORD *)(v18 + 448) = v8;
  v19 = result | a8;
  v20 = a5[2];
  v21 = *a5;
  v22 = a7[3];
  v23 = a7[4];
  v24 = a7[5];
  v25 = (__int64 *)a5[3];
  v26 = a7[6];
  v27 = a7[7];
  v28 = a5[1];
  *(_QWORD *)(v18 + 128) = v28 + 118;
  if ( *v25 < v28 + 118 == result )
  {
    *(_QWORD *)(v18 + 16) = v25;
    *(_QWORD *)(v18 + 56) = v27 - 6224352;
    *(_QWORD *)(v18 + 48) = v26 - 6224352;
    *(_QWORD *)(v18 + 40) = v24 - 6224352;
    *(_QWORD *)(v18 + 24) = v23 - 6224352;
    *(_QWORD *)(v18 + 8) = v22 - 6224352;
    *(_QWORD *)(v18 + 432) = result + 32;
    *(_QWORD *)(v18 + 32) = v20;
    *(_QWORD *)v18 = v21;
    v29 = ((__int64 (*)(void))(result | v19))();
    *(_QWORD *)(v18 + 424) = v30;
    v31 = *(_QWORD *)(v18 + 440);
    *(_QWORD *)(v18 + 416) = v31;
    v33 = ((__int64 (*)(void))(v29 | v32))();
    v34 = v33 + 16;
    *(_QWORD *)(v18 + 400) = v33 + 16;
    v36 = ((__int64 (*)(void))(v33 | v35))();
    *(_QWORD *)(v18 + 384) = v34;
    v37 = *(_QWORD *)(v18 + 408);
    v39 = ((__int64 (*)(void))(v36 | v38))();
    *(_QWORD *)(v18 + 336) = v31;
    *(_QWORD *)(v18 + 344) = v40;
    *(_QWORD *)(v18 + 352) = v37;
    *(_QWORD *)(v18 + 360) = v34;
    *(_QWORD *)(v18 + 376) = v34;
    *(_QWORD *)(v18 + 368) = *(_QWORD *)(v18 + 392);
    v42 = ((__int64 (*)(void))(v39 | v41))();
    v43 = v28 + 64;
    *(_QWORD *)(v18 + 320) = v28 + 64;
    v45 = ((__int64 (*)(void))(v42 | v44))();
    v46 = *(_QWORD *)(v18 + 328);
    *(_QWORD *)(v18 + 304) = v28;
    *(_QWORD *)(v18 + 296) = *(_QWORD *)v18;
    *(_QWORD *)(v18 + 312) = v18 + 64;
    v48 = ((__int64 (*)(void))(v45 | v47))();
    *(_QWORD *)(v18 + 280) = v18 + 64;
    *(_QWORD *)(v18 + 272) = v46;
    *(_QWORD *)(v18 + 288) = v48 + 64;
    v50 = ((__int64 (*)(void))(v48 | v49))();
    *(_QWORD *)(v18 + 256) = v51;
    *(_QWORD *)(v18 + 248) = v46 + 64;
    *(_QWORD *)(v18 + 264) = v28;
    v52 = v50 | v37;
    v54 = ((__int64 (*)(void))(v50 | v53))();
    *(_QWORD *)(v18 + 232) = v31;
    v55 = *(_QWORD *)(v18 + 32);
    *(_QWORD *)(v18 + 224) = v55;
    *(_DWORD *)(v18 + 240) = v54 + 32;
    v57 = ((__int64 (*)(void))(v54 | v56))();
    *(_QWORD *)(v18 + 168) = v52;
    *(_DWORD *)(v18 + 176) = v57 + 16;
    *(_QWORD *)(v18 + 184) = v58;
    *(_QWORD *)(v18 + 192) = v46;
    *(_QWORD *)(v18 + 200) = v43;
    *(_QWORD *)(v18 + 208) = v55 + 38;
    *(_QWORD *)(v18 + 216) = v18 + 128;
    *(_QWORD *)(v18 + 128) -= 38LL;
    v60 = ((__int64 (*)(void))(v57 | v59))();
    v62 = *(_QWORD *)(v18 + 128);
    if ( v62 == v60 )
      **(_QWORD **)(v18 + 16) = v60;
    else
      **(_QWORD **)(v18 + 16) = v62 + 38;
    *(_QWORD *)(v18 + 160) = v46;
    v63 = ((__int64 (*)(void))(v60 | v61))();
    *(_QWORD *)(v18 + 152) = v64;
    v66 = ((__int64 (*)(void))(v63 | v65))();
    *(_QWORD *)(v18 + 144) = v52;
    v68 = ((__int64 (*)(void))(v66 | v67))();
    *(_QWORD *)(v18 + 136) = v31;
    result = ((__int64 (*)(void))(v68 | v69))();
  }
  else
  {
    *v25 = result;
  }
  return result;
}

```

*   **总结**

整个样本在 vmp 中还是算简单的，毕竟虚拟机解释器跟 encode 编码部分都没有加密，比较适合学习。后面还原出的原始汇编转成 bin 后，也可以用 bin2llvm 转成 ir 进行分析与处理。

*   **样本**

https://pan.quark.cn/s/2eb36aea6148

[[培训] 传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

最后于 11 小时前 被 HandsomeBro 编辑 ，原因：

[#逆向分析](forum-161-1-118.htm)
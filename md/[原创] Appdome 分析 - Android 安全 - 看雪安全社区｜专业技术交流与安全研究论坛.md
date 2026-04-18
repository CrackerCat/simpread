> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-290866.htm)

> [原创] Appdome 分析

> 包名：com.miniclip.eightballpool (版本：56.19.0)

0x0 前言
------

幾個月前有朋友分享了一個 Appdome 的樣本給我，聽說很強，過年的時候看了一下，發現確實很強，嘗試分析了一段時間也沒看出什麼東西來，就沒有再繼續看了。

直到上個月在做某家公司的筆試題時，給的樣本又是 Appdome 保護的，只能說緣分真是妙不可言。

可能是因為這次有明確的需求和動力，加上嘗試了用 AI 來輔助逆向 (AI 太強了…)，最終陸陸續續的完成了對 Appdome 的分析。

筆試的產物我交了個面具模塊，但很抽象的是它們竟然連面具模塊都裝不明白？？說我給的模塊「有問題」，但我尋思我在普通 Magisk 和 KernelSU 上都能正常跑。後來發現他似乎是 zygisk 不會開，外加試圖加載一個非`.zip`的面具模塊 ( 似乎是把我發它的`.zip`給解壓了？？ )，才導致「裝不上」。果然這世界就是個巨大的草台班子嗎……

最終的面試沒有通過，大概是老天爺不忍看我去吃屎吧，或許是件好事。

0x1 常規分析
--------

常規分析的目的：找到檢測 so / 一些可疑的東西，以及熟悉一下它的代碼邏輯。

hook `android_dlopen_ext`，發現它加載了個奇怪的 so，閃退前會把這個 so 刪掉。

```
[android_dlopen_ext]  /data/user/0/com.miniclip.eightballpool//j9g29zqf0cfqd3vvu2bw/1712A600C28B40B0B9FD41DDFEFDC223/7006e94051c3/8peQFPY8peRAVCWABhddi9W5QtKCqKuZAsUc0k9xOkoP69jApP


```

猜測是通過`libc.so!remove`來刪除的，hook 阻止它的刪除行為。

```
function hook_remove() {
    Interceptor.replace(Module.findExportByName("libc.so", "remove"), new NativeCallback((arg0) => {
        let path = arg0.readCString();
        if (path.indexOf("8peQFPY8peRAVCWABhddi9W5QtKCqKuZAsUc0k9xOkoP69jApP") != -1) {
            console.log("[remove] path: ", path);
        }
        return 0;
    }, "int", ["pointer"]))
}


```

有兩個`8peQFPY8peRAVCWABhddi9W5QtKCqKuZAsUc0k9xOkoP69jApP`，將它們提取到本地。

```
[Remote::com.miniclip.eightballpool ]-> [remove] path:  /data/user/0/com.miniclip.eightballpool//j9g29zqf0cfqd3vvu2bw/D1CFF3B7BCA94FFDBEF7193A94CFCD6C/2c0462959561/temp/8peQFPY8peRAVCWABhddi9W5QtKCqKuZAsUc0k9xOkoP69jApP
[remove] path:  /data/user/0/com.miniclip.eightballpool/j9g29zqf0cfqd3vvu2bw/D1CFF3B7BCA94FFDBEF7193A94CFCD6C/2c0462959561/temp/8peQFPY8peRAVCWABhddi9W5QtKCqKuZAsUc0k9xOkoP69jApP
[remove] path:  /data/user/0/com.miniclip.eightballpool/j9g29zqf0cfqd3vvu2bw/D1CFF3B7BCA94FFDBEF7193A94CFCD6C/2c0462959561/8peQFPY8peRAVCWABhddi9W5QtKCqKuZAsUc0k9xOkoP69jApP


```

對比後發現`temp/`目錄下那個是解密前的，另一個是解密後的。

![](https://bbs.kanxue.com/upload/tmp/946537_3FS8J6V54N43XJ6.webp)

簡單分析後，可知這個奇怪的 so 並非檢測 so(依據：拉入 IDA 可見很多近乎源碼的邏輯，檢測 so 不可能會這樣)。

hook `dlopen`，發現在`libloader.so`時會閃退，結合`apkid`查殼工具，最終可以鎖定`libloader.so`就是檢測 so，加固名為 Appdome。

```
> apkid split_config.arm64_v8a.apk
[+] APKiD 3.0.0 :: from RedNaga :: rednaga.io
[*] split_config.arm64_v8a.apk!lib/arm64-v8a/libcrashlytics-common.so
 |-> anti_hook : syscalls
[*] split_config.arm64_v8a.apk!lib/arm64-v8a/libloader.so
 |-> anti_hook : syscalls
 |-> protector : Appdome
[*] split_config.arm64_v8a.apk!lib/arm64-v8a/libnms.so
 |-> anti_hook : syscalls


```

在`libloader.so` dlopen 完的時機阻塞，過一段時間仍會閃退？是否說明相關檢測點在`JNI_OnLoad`等時機就已經執行了？

hook `libloader.so!JNI_OnLoad`，不但沒有觸發，而且直接報錯：

```
Process crashed: java.lang.UnsatisfiedLinkError: No implementation found for void qMivB4.tmY0B4.aU4no0.ujd3D0.setApplication(java.lang.Object) (tried Java_qMivB4_tmY0B4_aU4no0_ujd3D0_setApplication and Java_qMivB4_tmY0B4_aU4no0_ujd3D0_setApplication__Ljava_lang_Object_2)

***
FATAL EXCEPTION: main
Process: com.miniclip.eightballpool, PID: 5447
java.lang.UnsatisfiedLinkError: No implementation found for void qMivB4.tmY0B4.aU4no0.ujd3D0.setApplication(java.lang.Object) (tried Java_qMivB4_tmY0B4_aU4no0_ujd3D0_setApplication and Java_qMivB4_tmY0B4_aU4no0_ujd3D0_setApplication__Ljava_lang_Object_2)
        at qMivB4.tmY0B4.aU4no0.ujd3D0.setApplication(Native Method)
        at qMivB4.tmY0B4.aU4no0.tETeU3.zd6ub6(Unknown Source:18)
        at qMivB4.tmY0B4.aU4no0.o8PB13.uzzmc7(Unknown Source:16)
        at com.vungle.ads.VungleProvider.onCreate(Unknown Source:0)
        at android.content.ContentProvider.attachInfo(ContentProvider.java:2451)
        at android.content.ContentProvider.attachInfo(ContentProvider.java:2421)
        at android.app.ActivityThread.installProvider(ActivityThread.java:7488)
        at android.app.ActivityThread.installContentProviders(ActivityThread.java:6999)
        at android.app.ActivityThread.handleBindApplication(ActivityThread.java:6770)
        at android.app.ActivityThread.-$$Nest$mhandleBindApplication(Unknown Source:0)
        at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2134)
        at android.os.Handler.dispatchMessage(Handler.java:106)
        at android.os.Looper.loopOnce(Looper.java:201)
        at android.os.Looper.loop(Looper.java:288)
        at android.app.ActivityThread.main(ActivityThread.java:7898)
        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:548)
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:936)


```

`.init_array`中只有`.init_array[3]`( 地址：`0x124BA8` ) 看上去與檢測有關，嘗試 hook 發現只有 enter 沒有 leave。

記`0x124BA8`為`init_array_func4_124BA8`。

![](https://bbs.kanxue.com/upload/tmp/946537_5R62JYVG52KZX8M.webp)

嘗試 trace `init_array_func4_124BA8`，但沒什麼發現，唯一比較有用的是 trace 時自動識別出來的字符串解密函數`sub_a8ad4()`，記為`decrypt_str`。

![](https://bbs.kanxue.com/upload/tmp/946537_74HPHSVUBAVNYN2.webp)

沒什麼思路的時候，我習慣動調看看。

動調時，會在`0x243B48`函數中觸發`abort()`。

![](https://bbs.kanxue.com/upload/tmp/946537_Z8J7P5X3FNRUE33.webp)

lr 在`0x80C10`，分析後發現是因為`byte_777EDC`不為`0`導致沒有走`decrypt_str()`。

![](https://bbs.kanxue.com/upload/tmp/946537_WUPM5C2KVS24WZM.webp)

在動調斷在`init_array_func4_124BA8`時對`byte_777EDC`下硬斷，看看哪裡寫入了這個地址。

```
[11686|11686] event_addr:0x79cedadedc hit_count:1, Backtrace:
  #00 pc 00000000000a94ac  /data/app/~~aaNNmfGcCRd1LmRDjQxCFQ==/com.miniclip.eightballpool-RmsVxckNmiJsu6bdVfYeng==/lib/arm64/libloader.so
  #01 pc 000000000007a1fc  /data/app/~~aaNNmfGcCRd1LmRDjQxCFQ==/com.miniclip.eightballpool-RmsVxckNmiJsu6bdVfYeng==/lib/arm64/libloader.so
  #02 pc 0000000000072268  /data/app/~~aaNNmfGcCRd1LmRDjQxCFQ==/com.miniclip.eightballpool-RmsVxckNmiJsu6bdVfYeng==/lib/arm64/libloader.so
  #03 pc 0000000000061908  /data/app/~~aaNNmfGcCRd1LmRDjQxCFQ==/com.miniclip.eightballpool-RmsVxckNmiJsu6bdVfYeng==/lib/arm64/libloader.so
  #04 pc 0000000000062188  /data/app/~~aaNNmfGcCRd1LmRDjQxCFQ==/com.miniclip.eightballpool-RmsVxckNmiJsu6bdVfYeng==/lib/arm64/libloader.so
  #05 pc 0000000000121438  /data/app/~~aaNNmfGcCRd1LmRDjQxCFQ==/com.miniclip.eightballpool-RmsVxckNmiJsu6bdVfYeng==/lib/arm64/libloader.so
  #06 pc 800c6c05dbb827e2  <unknown>


```

根據上述調用棧定位到`0x7A1FC`，發現只有在動調時，`0x7A1FC`處的`decrypt_str()`才會解密失敗，導致覆蓋掉上面的`byte_777EDC`。

![](https://bbs.kanxue.com/upload/tmp/946537_GZDXSAPT9JZMP3S.webp)

分析`decrypt_str()`，發現其中會調用`get_bytes()`讀取`init_array_func4_124BA8()`前`0x20`字節作為 key 來進行解密。

![](https://bbs.kanxue.com/upload/tmp/946537_4R9ANM8HS4MX6SY.webp)

而由於我對`init_array_func4()`下了斷點，因此會連鎖導致上述一連串問題。

![](https://bbs.kanxue.com/upload/tmp/946537_K4A3M7R29M868W4.webp)

通過 hook `get_bytes()`來 bypass 掉這問題，之後就能繼續動調。

```
function hook_get_bytes(base) {
    let init_array_func4_bytes = [
        0xC0, 0x32, 0x00, 0xF0, 0x01, 0x00, 0x00, 0x90, 0x00, 0xB0,
        0x30, 0x91, 0x21, 0x10, 0x3D, 0x91, 0x12, 0x31, 0x04, 0x14,
        0xFF, 0x03, 0x02, 0xD1, 0xFD, 0x7B, 0x03, 0xA9, 0xFA, 0x67,
        0x04, 0xA9
    ]

    Interceptor.attach(base.add(0x243880), {
        onEnter: function (args) {
            
            if (args[1].sub(base) == 0x124BA8) {
                // console.log("[get_bytes] a1: ", (args[1].sub(base)));
                // console.log("[get_bytes] a1: ", hexdump(args[1], {length: args[2].toInt32()}));
                this.flag = true;
                this.len = args[2].toInt32();
            }
        },
        onLeave: function (retval) {
            if (this.flag) {
                retval.writeByteArray(init_array_func4_bytes)
                // console.log("[get_bytes] retval: ", hexdump(retval))
            }
        }
    })
}


```

在某處，發現它調用`ng1ok_pthread_create()`創建來創建線程。

![](https://bbs.kanxue.com/upload/tmp/946537_YCGUF35KH2QTURN.webp)

而且是創建了一堆子線程，比較特別的是線程回調都是同一個，記為`pthread_func`。

![](https://bbs.kanxue.com/upload/tmp/946537_A7ZMA28M9W438H6.webp)

除此之外就沒有看到其他可疑的東西了，因此猜測檢測邏輯都在`pthread_func`中。

0x2 檢測線程分析
----------

分析`pthread_func()`，邏輯如下：

調用`mb_get_target_func()`獲取真正的檢測函數 (不同線程返回不同的檢測函數)，記獲取的檢測函數為`target_func`。

![](https://bbs.kanxue.com/upload/tmp/946537_MVWG75SZECQ5CJC.webp)在最後間接調用`target_func`。

![](https://bbs.kanxue.com/upload/tmp/946537_XWYB7HZAQ4NAHMW.webp)

因此可以通過 hook `mb_get_target_func()`來記錄所有檢測函數。

注：`mb_get_target_func()`返回的不一定是函數地址，因此要用 lr 來過濾，當 lr 為`pthread_func()`所屬地址時才打印。

```
function hook_get_target_func(base) {
    Interceptor.attach(base.add(0x10B810), {
        onEnter: function (args) {
            if (this.context.lr.sub(base) == 0x25387C) {
                this.flag = true;
            }
        },
        onLeave: function (retval) {
            if (this.flag) {
                let func_addr = retval.readPointer();
                console.log("[hook_get_target_func] func: ", func_addr.sub(base));
            }
        }
    })
}


```

逐一分析`mb_get_target_func()`返回的檢測函數，最終能找到一個方法 bypass 掉所有檢測點，各位讀者可以自行嘗試。

0x3 libloader hook 檢測分析
-----------------------

明顯存在對`libloader.so`的 hook 檢測，硬斷可知有兩處 hook 檢測：

```
[15777|16005] event_addr:0x782c8c7124 hit_count:23, Backtrace:
  #00 pc 000000000048e3a0  /data/app/~~8-jKhW4Si94CBkZuAPhwgg==/com.miniclip.eightballpool-i3PwTB1rdknDeh9vD890nw==/lib/arm64/libloader.so
  #01 pc 000000000048d200  /data/app/~~8-jKhW4Si94CBkZuAPhwgg==/com.miniclip.eightballpool-i3PwTB1rdknDeh9vD890nw==/lib/arm64/libloader.so
  #02 pc 000000000048d200  /data/app/~~8-jKhW4Si94CBkZuAPhwgg==/com.miniclip.eightballpool-i3PwTB1rdknDeh9vD890nw==/lib/arm64/libloader.so
  #03 pc 00000000000cd24c  /data/app/~~8-jKhW4Si94CBkZuAPhwgg==/com.miniclip.eightballpool-i3PwTB1rdknDeh9vD890nw==/lib/arm64/libloader.so
  #04 pc 00000000000c70fc  /data/app/~~8-jKhW4Si94CBkZuAPhwgg==/com.miniclip.eightballpool-i3PwTB1rdknDeh9vD890nw==/lib/arm64/libloader.so
  #05 pc 0000000000096f5c  /data/app/~~8-jKhW4Si94CBkZuAPhwgg==/com.miniclip.eightballpool-i3PwTB1rdknDeh9vD890nw==/lib/arm64/libloader.so
  #06 pc 000000000012acc4  /data/app/~~8-jKhW4Si94CBkZuAPhwgg==/com.miniclip.eightballpool-i3PwTB1rdknDeh9vD890nw==/lib/arm64/libloader.so
  #07 pc 00000000000cc32c  /data/app/~~8-jKhW4Si94CBkZuAPhwgg==/com.miniclip.eightballpool-i3PwTB1rdknDeh9vD890nw==/lib/arm64/libloader.so
  #08 pc 000000000022dde8  /data/app/~~8-jKhW4Si94CBkZuAPhwgg==/com.miniclip.eightballpool-i3PwTB1rdknDeh9vD890nw==/lib/arm64/libloader.so
  #09 pc 000000000022a558  /data/app/~~8-jKhW4Si94CBkZuAPhwgg==/com.miniclip.eightballpool-i3PwTB1rdknDeh9vD890nw==/lib/arm64/libloader.so
  #10 pc 000000000022a968  /data/app/~~8-jKhW4Si94CBkZuAPhwgg==/com.miniclip.eightballpool-i3PwTB1rdknDeh9vD890nw==/lib/arm64/libloader.so

[15777|15991] event_addr:0x782c8c7124 hit_count:24, Backtrace:
  #00 pc 000000000048fe10  /data/app/~~8-jKhW4Si94CBkZuAPhwgg==/com.miniclip.eightballpool-i3PwTB1rdknDeh9vD890nw==/lib/arm64/libloader.so
  #01 pc 000000000048e838  /data/app/~~8-jKhW4Si94CBkZuAPhwgg==/com.miniclip.eightballpool-i3PwTB1rdknDeh9vD890nw==/lib/arm64/libloader.so
  #02 pc 000000000048e838  /data/app/~~8-jKhW4Si94CBkZuAPhwgg==/com.miniclip.eightballpool-i3PwTB1rdknDeh9vD890nw==/lib/arm64/libloader.so
  #03 pc 0000000000161100  /data/app/~~8-jKhW4Si94CBkZuAPhwgg==/com.miniclip.eightballpool-i3PwTB1rdknDeh9vD890nw==/lib/arm64/libloader.so
  #04 pc 0000000000162a68  /data/app/~~8-jKhW4Si94CBkZuAPhwgg==/com.miniclip.eightballpool-i3PwTB1rdknDeh9vD890nw==/lib/arm64/libloader.so
  #05 pc 0000000000161670  /data/app/~~8-jKhW4Si94CBkZuAPhwgg==/com.miniclip.eightballpool-i3PwTB1rdknDeh9vD890nw==/lib/arm64/libloader.so
  #06 pc 0000000000173144  /data/app/~~8-jKhW4Si94CBkZuAPhwgg==/com.miniclip.eightballpool-i3PwTB1rdknDeh9vD890nw==/lib/arm64/libloader.so
  #07 pc 000000000016db24  /data/app/~~8-jKhW4Si94CBkZuAPhwgg==/com.miniclip.eightballpool-i3PwTB1rdknDeh9vD890nw==/lib/arm64/libloader.so
  #08 pc 0000000000161794  /data/app/~~8-jKhW4Si94CBkZuAPhwgg==/com.miniclip.eightballpool-i3PwTB1rdknDeh9vD890nw==/lib/arm64/libloader.so


```

嘗試 hook `0x48e3a0`和`0x48fe10`所屬的函數 ( 記為`sha1_transform1`和`sha1_transform2` )，會 hook 失敗。

![](https://bbs.kanxue.com/upload/tmp/946537_QJTEEY5FXNVZ88F.webp)

怎麼 hook？來到`0x48e3a0`，看這地址的交叉引用。

![](https://bbs.kanxue.com/upload/tmp/946537_ZJ6W7F6RAU6X4QQ.webp)

去到交叉引用處，一路向上，找到像函數頭的地方 ( `stp x29, x30` )，hook 這個地址，能成功。

![](https://bbs.kanxue.com/upload/tmp/946537_84FXXSGP7G8XTEC.webp)

bypass 思路：手動`mmap`一個假的`libloader.so`，然後在讀取`libloader.so`的地方進行重定向，來個偷天換日即可。

但之後仍會閃退，說明大概率還有其他地方在做校驗，按同樣方法分析和繞過即可。

0x4 Frida hook 檢測分析
-------------------

在後台啟動 frida-server，之後手動打開遊戲，能正常運行。

以 spawn 方式啟動遊戲，不進行任何 hook，會觸發檢測，提示`Security Threat Detected`：

![](https://bbs.kanxue.com/upload/tmp/946537_PYTE2RV3DQFJY5A.webp)

先從字符串解密函數入手，hook `decrypt_str()`，記錄閃退前解密的字符串，在其中搜`threat`，有 3 個，後 2 個其實屬於同一個函數。

`threatEventsScore`所屬函數是`sub_2E3AC8`，`threatCode`所屬函數是`sub_2E0B78`。

```
[decrypt_str] str: threatEventsScore (lr: 0x2e5358)

[decrypt_str] str: threatCode (lr: 0x2e1060)
[decrypt_str] str: threatCode (lr: 0x2e266c)


```

嘗試 hook 這兩個函數，然後阻塞，發現均不會觸發`Security Threat Detected`。

說明要麼是這兩個函數在檢測，要麼是它們的上層函數，總之它們都必然是檢測鏈路中的一環。

```
function hook_test(base) {
    // Interceptor.attach(base.add(0x2E3AC8), {
    Interceptor.attach(base.add(0x2E0B78), {
        onEnter: function (args) {
            console.log("[hook_test] a0: ", args[0]);
            Thread.sleep(100);
            // print_stack(this.context);
        },
        onLeave: function (retval) {
            console.log("[hook_test] retval: ", retval);
        }
    })

}


```

調用棧分別如下：

```
[0x2E0B78]
called from:
0x764c4edd00 libloader.so!0x2e2d00  // func: sub_2E2AFC
0x764c4edd00 libloader.so!0x2e2d00
0x764c478acc libloader.so!0x26dacc
0x764c477f7c libloader.so!0x26cf7c
0x764c473d08 libloader.so!0x268d08
0x764c5397e0 libloader.so!0x32e7e0
0x764c2fc2bc libloader.so!0xf12bc
0x764c45e938 libloader.so!0x253938
0x797f6cc4e0 libc.so!_ZL15__pthread_startPv+0xd0
0x797f65f934 libc.so!__start_thread+0x44

[0x2E3AC8]
called from:
0x76497036f0 libloader.so!0x2e36f0
0x76497036f0 libloader.so!0x2e36f0
0x764965109c libloader.so!0x23109c
0x7649703278 libloader.so!0x2e3278
0x7649673938 libloader.so!0x253938
0x797f6cc4e0 libc.so!_ZL15__pthread_startPv+0xd0
0x797f65f934 libc.so!__start_thread+0x44


```

將上述思路給 AI，讓它分析得出的結論如下：

```
• 結論
  真正的 Frida 相關檢測點不在 sub_2E0B78 或 sub_2E3AC8。這兩個更像是上報/配置層。和你這條調用鏈對上的實際檢測鏈是：

  pthread_func(0x253938) -> sub_F0DD8(0xF0DD8) -> sub_32E0F8(0x32E0F8) -> sub_268CB8(0x268CB8) -> sub_26952C(0x26952C)
  -> sub_26D158(0x26D158) -> sub_2E2AFC(0x2E2AFC) -> sub_2E0B78(0x2E0B78)

  其中真正落到 threatCode 的核心點是 sub_26D158 @ 0x26D158。sub_26952C 負責從 /proc/<pid>/maps 類內容裡篩選候選映射，
  sub_268CB8 是這條檢測的入口。

  證據

  - .ida-mcp/268cb8.c:120 先讀 /proc/<pid>/...，成功後在 .ida-mcp/268cb8.c:123 把內容交給 sub_26952C；失敗時在 .ida-mcp/268cb8.c:396 直接走 sub_2E2AFC(..., 2, ...)。
  - .ida-mcp/26952c.c:2815 用 maps 格式做解析；在 .ida-mcp/26952c.c:2816 明確只處理帶 pathname 的 x 權限映射；之後在 .ida-mcp/26952c.c:3215 把每個候選映射交給 sub_26D158。
  - .ida-mcp/26d158.c:171 打開 maps 中對應文件路徑，.ida-mcp/26d158.c:190 mmap 對應文件段，.ida-mcp/26d158.c:207 對進程內存段做校驗；一旦有差異，在 .ida-mcp/26d158.c:237 進入上報分支，並在 .ida-mcp/26d158.c:465 調 sub_2E2AFC 生成threat 事件。

  原理
  sub_2E3284 只是載入/校驗 adconfig_devevents，再調 sub_2E3AC8 構造 threatEventsScore；sub_2E0B78 則負責序列化
  threatCode。所以你 hook 這兩個函數後不再彈 Security Threat Detected，本質上是截斷了“上報鏈”，不是把檢測源頭幹掉了。

  從 maps 解析 + 文件映射段對比的結構看，這裡做的是 inline hook / 代碼完整性檢測。推斷 Frida spawn 命中的不是單純的
  frida 字串或端口檢測，而是注入後某些可執行映射與磁盤文件對應段不一致。

  如果你要，我下一步可以繼續把 sub_26952C 裡那幾組解密過濾字串還原出來，進一步鎖定到底在比對哪些 .so / 哪類映射。


```

它說`sub_268CB8`是 frida 檢測的入口，我嘗試阻塞這個函數。

```
Interceptor.attach(base.add(0x268CB8), {
    onEnter: function (args) {
        console.log("call 268CB8")
        Thread.sleep(1000)
    },
    onLeave: function (retval) {
        // console.log("[hook_test] retval: ", retval);
    }
})


```

發現能進入遊戲，之後雖然會閃退，但觸發的是其他檢測 ( `Malicious Injection Detected` )，說明 frida 檢測入口大概率如 AI 所說是在`sub_268CB8`。

![](https://bbs.kanxue.com/upload/tmp/946537_48HXHKHUGGU8A69.webp)

按照 AI 給出的證據來驗證。

```
.ida-mcp/268cb8.c:120 先讀 /proc/<pid>/...，成功後在 .ida-mcp/268cb8.c:123 把內容交給 sub_26952C；失敗時在 .ida-mcp/268cb8.c:396 直接走 sub_2E2AFC(..., 2, ...)。


```

它說`0x268cb8.c:123`處會把`/proc/&lt;pid&gt;/...`的內容傳給`sub_26952C()`，hook 驗證的確如此。

![](https://bbs.kanxue.com/upload/tmp/946537_SGQBNBVJC23URJH.webp)

trace 看看`sub_26952C()`到底在驗證什麼。

分析 trace 日志，可知大概率是在校驗`libart.so`和`libc.so`等系統庫是否被 hook

![](https://bbs.kanxue.com/upload/tmp/946537_D243PRPT3VP2U9T.webp)

![](https://bbs.kanxue.com/upload/tmp/946537_WJZEWPCPK85VYU6.webp)

其中應該是通過`process_vm_readv`來讀取內存中的`libart.so`和`libc.so`，然後與本地文件進行 sha256 對比。

![](https://bbs.kanxue.com/upload/tmp/946537_DT6XNBXPBXFY2PS.webp)

當然，它並不止檢測了`libart.so`和`libc.so`，還有其他幾個系統庫。

0x5 其他檢測點
---------

還有很多其他檢測點，讀者有興趣可以分析看看。

1.  Root 檢測
    
    ```
    // 1. Magisk path
    path:  /sbin/magisk
    path:  /sbin/magiskhide
    path:  /sbin/magiskinit
    path:  /sbin/magiskpolicy
    path:  /sbin/.magisk
    path:  /bin/magisk
    path:  /bin/magiskhide
    path:  /bin/magiskinit
    path:  /bin/magiskpolicy
    path:  /bin/.magisk
    
    // 2. mounts
    [sys_openat] path:  /proc/27712/mounts
    
    // 3. Magisk apk custom check
    
    
    ```
    
2.  frida 檢測 (2)：檢測了 frida 的默認 hook 行為
    
3.  簽名校驗
    
4.  注入檢測
    

……

0x6 遊戲功能分析
----------

遊戲邏輯在`libgame-BPM-GooglePlay-Gold-Release-Module-3919.so`中，它其實就是`8peQFPY8peRAVCWABhddi9W5QtKCqKuZAsUc0k9xOkoP69jApP`(解密前的 so 文件就是從 lib 中獲取的)。

發現一張表，似乎對應遊戲的業務邏輯。

![](https://bbs.kanxue.com/upload/tmp/946537_NXEPJJKVX3JCVH6.webp)

該表每項元素的結構如下：

```
struct FunctionTableItem {
    void* functionName;
    void* paramsFormat; 
    void* functionAddress;
}


```

嘗試 hook `canMoveBalls()`，將返回值置`1`。

```
Interceptor.attach(base.add(0x2DC11E8), {
    onEnter: function (args) {
        console.log("[f_canMoveBalls] a0: ", args[0])
        print_stack2(this.context)
    },
    onLeave: function (retval) {
        retval.replace(1);
        console.log("[f_canMoveBalls] retval: ", (retval))
    }
})


```

結果在遊戲中我能隨意移動任何球，說明上述推測正確。

![](https://bbs.kanxue.com/upload/tmp/946537_V4SUY3X2W5S5VJ7.webp)

同理，調用遊戲中與小球座標有關的 API，就能實現以下效果：

![](https://bbs.kanxue.com/upload/tmp/946537_TKZNA9RTNW3BGBU.webp)

0x7 結語
------

本文只對 Appdome 做了部份的分析，留下了一些「空白」，各位讀者可自行分析，或者加入我的「Ng1ok Hub」去查看完整的分析文章。

![](https://bbs.kanxue.com/upload/tmp/946537_DHGSSY92HZABKJE.webp)

(生活不易，給自己打個廣告 ^^)

「Ng1ok Hub」是我用 Notion 建的個人的知識付費平台 (類似「知識星球」，不用它是因為我用不了…)，目前有 Appdome 的詳細分析文章，包括繞過檢測的思路和方法，以及文中用到的調用流 trace 工具 ( nggtrace )。之後會繼續分享一些其他好玩的樣本 ( 如果有的話 )、檢測、工具等 ( 安卓逆向 / 遊戲安全相關的 )。

有相關需求 / 委託也歡迎聯繫我！！！

tg / vx(同號)：`ngiokweng`

[传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm)
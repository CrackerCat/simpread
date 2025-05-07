> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2029498-1-1.html)

> [md]## 前言前前後後大概分析了這樣本 4 次左右，前 3 次都以失敗告終，或許對於普通人來說，失敗才是人生的主旋律，接觸逆向後對這句話越來越有感觸。本文主要分 ...

![](https://avatar.52pojie.cn/data/avatar/001/89/82/08_avatar_middle.jpg)ngiokweng  

### 前言

前前後後大概分析了這樣本 4 次左右，前 3 次都以失敗告終，或許對於普通人來說，失敗才是人生的主旋律，接觸逆向後對這句話越來越有感觸。

本文主要分析的目標是 frida/hook 檢測。

### 閃退情況描述

frida hook 後會立即閃退，hook `dlopen`後可知是在加載`lib__6dba__.so`時閃退，具體是在`lib__6dba__.so`的`.init_array`裡。而`.init_array`中只有一個`start`函數。

frida hook 了一次之後，下次就算不 hook 正常打開 APP 也會閃退，大概率檢測了 frida 的 maps 特徵。

### start 分析

一開始會調用`get_custom_scetion`獲取`lib__6dba__.so`中的加密數據。

![](https://attach.52pojie.cn/forum/202505/06/225100adz7t07k0du87f0e.png)

**image.png** _(24.73 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY2N3w5OTcyZThjM3wxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

具體實現如下：

首先用`openat`、`lseek`、`read`等系統調用打開並讀取`lib__6dba__.so`，然後遍歷獲取最後一個 loadable segment 的結束地址，記為`last_loadseg_end`。

![](https://attach.52pojie.cn/forum/202505/06/225103tmis153u3q77og79.png)

**image1.png** _(32.38 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY2OHwzMGQ5OGU5MnwxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

用 010 查看`last_loadseg_end`偏移指向的數據，可以看出明顯是一些高熵數據，記這些數據為`enc_data`。

![](https://attach.52pojie.cn/forum/202505/06/225104opoqhjhqegxe5ty5.png)

**image2.png** _(49.2 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY2OXwzNTdkYjE2NXwxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

繼續向下看，它又遍歷 shdr table 獲取自定義的一個 section。

![](https://attach.52pojie.cn/forum/202505/06/225104xrdp1spdvddsvmjv.png)

**image3.png** _(37.7 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY3MHxlM2E2NmE3MHwxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

從 010 可以看出，該 section 同樣是指向上述`last_loadseg_end`那附近。

![](https://attach.52pojie.cn/forum/202505/06/225105l5w1j5a2zajz56pw.png)

**image4.png** _(20.56 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY3MXw0MmY4MTJmYnwxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

雖然不知為何要分別通過 phdr 和 shdr 來定位`enc_data`，但總的來說`get_custom_scetion`函數的功能就是獲取`enc_data`。

回到`start`函數，獲取完`enc_data`後，調用`decrypt1`和`decrypt2`來解密。

![](https://attach.52pojie.cn/forum/202505/06/225105dzl8uu8qazootl8a.png)

**image5.png** _(48.06 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY3Mnw5Njg0MzVhN3wxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

解密出來的數據其實是一些可執行的邏輯，由於它是通過`mmap`映射 + `mprotect`賦予可執行權限的方式來執行，因此記這種形式為 mmap 模塊，根據創建順序記為`mmap1`模塊、`mmap2`模塊、…，如此類推。frida 的檢邏邏輯明顯就在其中。

![](https://attach.52pojie.cn/forum/202505/06/225105wekkbkjddcwkwb7w.png)

**image6.png** _(33.11 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY3M3xiYWU2M2RmYnwxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

注：該保護使用了大量的系統調用 ( 上述的`mmap`和`mprotect`都是指系統調用 )，一些基礎函數如`strcpy`、`strlen`、`memset`等都是自實現的。

### hook & dump mmap 模塊

一開始我選擇通過動調來分析上述的`mmap1`模塊，發現`mmap1`中會創建和調用`mmap2`、`mmap3`、`mmap4`模塊，同理`mmap2 ~ 4`模塊又分別會創建和調用更多的 mmap 模塊，如此一來使得動調難以分析 (最主要是因為在 mmap 模塊中記錄的注釋、重命名變量名、函數名等都無法持久地保存)。

但動調也並非毫無收獲，可以得知以下幾點：

1.  每個 mmap 模塊的結構是非常相似的 (動調後會明白這句話的意思)。
2.  每個 mmap 模塊的大部份函數實現是一樣的，如字符串解密函數。
3.  每個 mmap 模塊都有封裝系統調用，因此可以很方便地 hook。
4.  每個 mmap 模塊創建 & 調用另一個 mmap 模塊的方法是一樣的，都是通過`mmap` + `mprotect`系統調用

由於難以動調，只好以純 hook 的方式來分析，在此之前要先將所有 mmap 模塊 dump 下來，遊戲閃退前創建的 mmap 模塊共有`13`個。

可以通過 frida 或 qbdi 等方式來 dump 和 trace 所有 mmap 模塊，dump 文件記為`mmap_<base>_<size>_<idx>.bin`，trace 文件記為`log.txt`(主要記錄函數調用關系，用利用 qbdi 可以很方便實現)。

然後按字節特徵來判斷`mmap1 ~ mmap13`，獲取分別的基址，以此進行 hook。hook `mmap1 ~ 4`的例子如下所示。

```
let hooked = false;
let mmap_history = {}
function hook_func_init(soName) {
    if (hooked) return;
    hooked = true;

    function hook_syscall() {
        function is_mmap1 (addr) {
            let byte_arr = [
                0xF0, 0x7B, 0xBF, 0xA9, 0x30, 0x01, 0x00, 0xB0, 0x11, 0x86, 
                0x42, 0xF9, 0x10, 0x22, 0x14, 0x91, 0x20, 0x02, 0x1F, 0xD6
            ]
            let offset = 0x440;
            for(let i = 0; i < byte_arr.length; i++) {
                if (addr.add(offset).add(i).readU8() != byte_arr[i]) return false;
            }
            return true;
        }

        function hook_mmap1(mmap_base) {
            Interceptor.attach(mmap_base.add(0xF9B0), {
                onEnter: function(args) {
                    this.sysno = args[7];
                    this.a0 = args[0]
                    this.a1 = args[1]
                    this.a2 = args[2]
                },
                onLeave: function(retval) {
                    if (this.sysno == 0xde) {
                        // console.log("[hook_mmap1_syscall] mmap addr: ", retval, "size: ", this.a1, "prot: ", this.a2);
                        mmap_history[retval] = this.a1;
                    }

                    if (this.sysno == 0xe2) {
                        console.log("[hook_mmap1_syscall] mprotect addr: ", this.a0, "size: ", this.a1 ,"prot: ", this.a2);
                        if (mmap_history[this.a0]) {
                            console.log(`\t[hook_mmap1_syscall] mmap addr: ${this.a0}  size: ${mmap_history[this.a0]}`);
                        }
                        if (is_mmap2(this.a0)) {
                            hook_mmap2(this.a0);
                        }
                        if (is_mmap3(this.a0)) {
                            hook_mmap3(this.a0);
                        }
                        if (is_mmap4(this.a0)) {
                            hook_mmap4(this.a0);
                        }
                    }
                }
            })
        }

        Interceptor.attach(base.add(0x5C84), {
            onEnter: function(args) {
                // console.log("[svc] sysno: ", args[7]);
                this.sysno = args[7];
                this.a0 = args[0]
                this.a1 = args[1]
                this.a2 = args[2]
            },
            onLeave: function(retval) {
                if (this.sysno == 0xde) {
                    // console.log("mmap addr: ", retval, "size: ", this.a1, "prot: ", this.a2);
                    mmap_history[retval] = this.a1;
                }

                if (this.sysno == 0xe2) {
                    console.log("[syscall] mprotect addr: ", this.a0, "size: ", this.a1 ,"prot: ", this.a2);
                    if (mmap_history[this.a0]) {
                        console.log(`\t[syscall] mmap addr: ${this.a0}  size: ${mmap_history[this.a0]}`);
                    }
                    if (is_mmap1(this.a0)) {
                        hook_mmap1(this.a0);
                    }
                }
            }
        })
    }

    var base = Module.findBaseAddress(soName);
    console.log("[hook_func_init] base: ", base);

    hook_syscall();

}
```

### mmap13 模塊分析

閃退前的最後一個模塊是`mmap13`，大概率會包含檢測 frida 的邏輯，因此重點分析這個模塊。

#### 基礎分析

先找到字符串解密函數，其特徵如下，返回值就是解密後的字符串：

![](https://attach.52pojie.cn/forum/202505/06/225105jtcpc62mc45zma1m.png)

**image7.png** _(28.64 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY3NHxhNDJkMmE5MXwxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

hook 輸出如下：

```
[hook_mmap13_decrypt_str] retval:  %s/lib
[hook_mmap13_decrypt_str] retval:  %s/lib
[hook_mmap13_decrypt_str] retval:  /lib
[hook_mmap13_decrypt_str] retval:  arm64
[hook_mmap13_decrypt_str] retval:  %s/%s
[hook_mmap13_decrypt_str] retval:  %s/%s
[hook_mmap13_decrypt_str] retval:  %s/%s
[hook_mmap13_decrypt_str] retval:  /proc/self/maps
[hook_mmap13_decrypt_str] retval:  %s/%s
[hook_mmap13_decrypt_str] retval:  %s/%s
[hook_mmap13_decrypt_str] retval:  %s/%s
[hook_mmap13_decrypt_str] retval:  %s/%s
[hook_mmap13_decrypt_str] retval:  %s/%s
[hook_mmap13_decrypt_str] retval:  %s/%s
```

比較可疑的是`/proc/self/maps`，打印調用棧發現在`mmap13!0xF348`，而該地址所在函數的交叉引用在`0x4D28`。

![](https://attach.52pojie.cn/forum/202505/06/225105zve1grgarzh1vr7h.png)

**image8.png** _(39.4 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY3NXxjYWQ3NTZhZXwxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

`bl sub_F2E8`所在地址是`0x4D28`，加上`mmap13`的基址是`0x7AB2D21D28`。

![](https://attach.52pojie.cn/forum/202505/06/225105jtt77r5iit7wyn8r.png)

**image9.png** _(16.98 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY3NnwzZTkyZDczNHwxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

在`log.txt`裡搜`0x7AB2D21D28`找到對應地方查看函數調用關系，發現以下函數調用順序：

1.  `openat` + `lseek` + `read`讀取了`/proc/self/maps`中的數據。
2.  通過`vsnprintf`拼接了 APP 自身 3 個 so 庫的完整路徑，其中就包括`lib__6dba__.so`的完整路徑。

```
0x14043be0 (0x7ab2d2bbe0): sub_1404712c() {
    0x140470c4 (0x7ab2d2f0c4): sub_14047068() {
        0x14047080 (0x7ab2d2f080): [SVC] sysno(0x38) -> openat(-100, "/proc/self/maps") => fd: 0x27
    }
}
0x14043c1c (0x7ab2d2bc1c): sub_140439a0() {
    0x140439b4 (0x7ab2d2b9b4): sub_14045e70() {
    }
    0x140439d4 (0x7ab2d2b9d4): sub_1404709c() {
        0x140470c4 (0x7ab2d2f0c4): sub_14047068() {
            0x14047080 (0x7ab2d2f080): [SVC] sysno(0xde) -> mmap(0x0, 0x80000, 0x3) => mmap address: 0x7ab1660000
        }
    }
}
0x14043c40 (0x7ab2d2bc40): sub_14043a44() {
    0x14043aa8 (0x7ab2d2baa8): sub_1404709c() {
        0x140470c4 (0x7ab2d2f0c4): sub_14047068() {
            0x14047080 (0x7ab2d2f080): [SVC] sysno(0x3e) -> lseek
        }
    }
    0x14043ad4 (0x7ab2d2bad4): sub_1404709c() {
        0x140470c4 (0x7ab2d2f0c4): sub_14047068() {
            0x14047080 (0x7ab2d2f080): [SVC] sysno(0x3f) -> read(0x27, "12c00000-12c40000 rw-p 00000000 ", 0x80000) => real read bytes: 0xf96
        }
    }
    0x14043ad4 (0x7ab2d2bad4): sub_1404709c() {
        0x140470c4 (0x7ab2d2f0c4): sub_14047068() {
            0x14047080 (0x7ab2d2f080): [SVC] sysno(0x3f) -> read(0x27, "71124000-71125000 rw-p 0003d000 ", 0x7f06a) => real read bytes: 0xfb7
        }
    }

// ...

0x1403b658 (0x7ab2d23658): sub_14040bb8() {
    0x14040c48 (0x7ab2d28c48): sub_14040ab0() {
        0x14040af4 (0x7ab2d28af4): sub_14037590() {     // nglog: mmap13 => 0xBAF4
            0x14040af4 (0x7ab2d28af4): [ExternalCall] vsnprintf("/data/app/jp.gungho.padHT-RB7leURHfwOLGhr-1wOUew==/lib/arm64/lib__6dba__.so", 0x400, "%s/%s") => res: 0x4b
        }
        }
}
```

由此猜測可能是在檢查自身的 so 庫有沒有被 hook。

嘗試 hook `mmap13`的`vsnprintf`，將`lib__6dba__.so`替換為另一個沒被 hook 的庫`libpad.so` (這個庫也是 APP 本身的)。

```
function hook_vsnprintf () {
    Interceptor.attach(mmap_base.add(0xBBB8), {
        onEnter: function (args) {
            this.a0 = args[0];
        },
        onLeave: function (retval) {
            if (this.a0.readCString().indexOf("lib__6dba__.so") != -1) {
                console.log("replace!!!!!!!!!!!!")
                Memory.writeUtf8String(this.a0, this.a0.readCString().replace("lib__6dba__.so", "libpad.so"))
            }
            console.log("[mmap13_vsnprintf] this.a0: ", this.a0.readCString());
        }
    })
}
```

替換前，`vsnprintf`的輸出如下：

```
[mmap13_vsnprintf] this.a0:  /data/user/0/jp.gungho.padHT/lib
[mmap13_vsnprintf] this.a0:  /data/user/0/jp.gungho.padHT/lib
[mmap13_vsnprintf] this.a0:  /data/app/jp.gungho.padHT-RB7leURHfwOLGhr-1wOUew==/lib/arm64/libopenal.so
[mmap13_vsnprintf] this.a0:  /data/app/jp.gungho.padHT-RB7leURHfwOLGhr-1wOUew==/lib/arm64/libpad.so
[mmap13_vsnprintf] this.a0:  /data/app/jp.gungho.padHT-RB7leURHfwOLGhr-1wOUew==/lib/arm64/lib__6dba__.so
[mmap13_vsnprintf] this.a0:  /data/app/jp.gungho.padHT-RB7leURHfwOLGhr-1wOUew==/lib/arm64/libopenal.so
[mmap13_vsnprintf] this.a0:  /data/app/jp.gungho.padHT-RB7leURHfwOLGhr-1wOUew==/lib/arm64/libpad.so
[mmap13_vsnprintf] this.a0:  /data/app/jp.gungho.padHT-RB7leURHfwOLGhr-1wOUew==/lib/arm64/lib__6dba__.so
[mmap13_vsnprintf] this.a0:  /data/app/jp.gungho.padHT-RB7leURHfwOLGhr-1wOUew==/lib/arm64/libopenal.so
[mmap13_vsnprintf] this.a0:  /data/app/jp.gungho.padHT-RB7leURHfwOLGhr-1wOUew==/lib/arm64/libpad.so
[mmap13_vsnprintf] this.a0:  /data/app/jp.gungho.padHT-RB7leURHfwOLGhr-1wOUew==/lib/arm64/lib__6dba__.so
```

替換後，`vsnprintf`的輸出如下，可以看到多了兩行關於`libc.so`的日志

```
[mmap13_vsnprintf] this.a0:  /data/user/0/jp.gungho.padHT/lib
[mmap13_vsnprintf] this.a0:  /data/user/0/jp.gungho.padHT/lib
[mmap13_vsnprintf] this.a0:  /data/app/jp.gungho.padHT-RB7leURHfwOLGhr-1wOUew==/lib/arm64/libopenal.so
[mmap13_vsnprintf] this.a0:  /data/app/jp.gungho.padHT-RB7leURHfwOLGhr-1wOUew==/lib/arm64/libpad.so
replace!!!!!!!!!!!!
[mmap13_vsnprintf] this.a0:  /data/app/jp.gungho.padHT-RB7leURHfwOLGhr-1wOUew==/lib/arm64/libpad.so
[mmap13_vsnprintf] this.a0:  /data/app/jp.gungho.padHT-RB7leURHfwOLGhr-1wOUew==/lib/arm64/libopenal.so
[mmap13_vsnprintf] this.a0:  /data/app/jp.gungho.padHT-RB7leURHfwOLGhr-1wOUew==/lib/arm64/libpad.so
replace!!!!!!!!!!!!
[mmap13_vsnprintf] this.a0:  /data/app/jp.gungho.padHT-RB7leURHfwOLGhr-1wOUew==/lib/arm64/libpad.so
[mmap13_vsnprintf] this.a0:  /data/app/jp.gungho.padHT-RB7leURHfwOLGhr-1wOUew==/lib/arm64/libopenal.so
[mmap13_vsnprintf] this.a0:  /data/app/jp.gungho.padHT-RB7leURHfwOLGhr-1wOUew==/lib/arm64/libpad.so
replace!!!!!!!!!!!!
[mmap13_vsnprintf] this.a0:  /data/app/jp.gungho.padHT-RB7leURHfwOLGhr-1wOUew==/lib/arm64/libpad.so
[mmap13_vsnprintf] this.a0:  /vendor/lib64/libc.so
[mmap13_vsnprintf] this.a0:  /system/lib64/libc.so
```

用同樣方法將`libc.so`替換為`libz.so`，發現 APP 終於不會在`mmap13`模塊之後馬上閃退，反而又再創建了其他模塊。

![](https://attach.52pojie.cn/forum/202505/06/225105ebtiw7vd72oggeyi.png)

**image10.png** _(83.52 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY3N3xjM2M1YWRlMXwxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

簡單小結，`mmap13`模塊應該是先檢測了`lib__6dba__.so`(APP 本身的 so 庫) 有沒有被 hook，若前者通過檢測，則再檢測`libc.so`(系統 so 庫) 有沒有被 hook，都通過後才會創建新模塊進行其他檢測，否則就用某些手段讓程序退出。

手動 patch `mmap13`模塊後，多了很多新模塊，是在`mmap3`模塊裡創建的，索引由`14`開始，共有`mmap14 ~ mmap30`模塊。

```
if (this.sysno == 0xe2) {
    console.log("[hook_mmap3_syscall] mprotect addr: ", this.a0, "size: ", this.a1 ,"prot: ", this.a2);
    if (mmap_history[this.a0]) {
        console.log(`\t[hook_mmap3_syscall] mmap addr: ${this.a0}  size: ${mmap_history[this.a0]}`);
        // after patch mmap13 detect, use this to dump new mmap module
        if (is_hook_mmap13) {
            saveData(`/data/data/jp.gungho.padHT/mmap_${this.a0}_${mmap_history[this.a0]}_${idx++}.bin`, this.a0, mmap_history[this.a0].toInt32());
        }
    }
    // ...
```

#### local lib 檢測分析

上一小節通過 trace 日志 + 經驗猜測的方式成功 bypass 了`lib__6dba__.so`中的 hook 檢測，這一小節嘗試分析看看具體的檢測原理。

hook `mmap13`模塊封裝的 syscall，在系統調用是`openat`且 path 包含`lib__6ba__.so`時打印調用棧，然後一路向上跟，最終發現是在`mmap13!0x3BF0`裡打開`lib__6ba__.so`的。

詳細調用鏈如下：( `ins addr`代表指令地址，`func addr`代表函數起始地址 )

```
0x394C(ins addr) -> 0x485C(ins addr) -> 0x433C(ins addr) -> 0x3F7C(ins addr) -> 0x3BF0(func addr)
```

`0x394C`( 調用`sub_4684`的指令地址 ) 附近的邏輯如下，記所在函數為`mmap13_main`。

測試發現，按上述「hook mmap13 的`vsnprintf`，將`lib__6dba__.so`替換為另一個沒被 hook 的庫`libpad.so`」後，`sub_4C20`函數會返回`1`，否則返回`0`。

由此可知`sub_4C20`要麼是具體的檢測函數，要麼是處理檢測結果的函數。記`sub_4C20`為`mb_detect_func`。

![](https://attach.52pojie.cn/forum/202505/06/225105c6y07eg06w66p63j.png)

**image11.png** _(26.23 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY3OHxjM2NjODdiOHwxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

進入`mb_detect_func`分析，一路通過 hook 驗證，會發現`get_so_info`這個比較關鍵的函數。

一開始以為`get_so_info`是具體的檢測函數，因為 hook 發現`get_so_info`共調用了 3 次，而且 hook `mmap13`的`openat`系統調用時，看到它打開了 3 個自身的 so 庫，正好與之對應。由此猜測前 2 次`get_so_info`執行後的`a1`為`0`是因為我沒有 hook `libopenal.so`和`libpad.so`，而第 3 次不為`0`是因為 hook 了`lib__6dba__.so`被檢測到。

![](https://attach.52pojie.cn/forum/202505/06/225106dnacox8oiuicnzf8.png)

**image12.png** _(27.1 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY3OXw1NGM2MmI4M3wxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

```
// hook mmap13 openat log:
[hook_mmap13_openat] a1:  /data/app/jp.gungho.padHT-RB7leURHfwOLGhr-1wOUew==/lib/arm64/libopenal.so
[hook_mmap13_openat] a1:  /data/app/jp.gungho.padHT-RB7leURHfwOLGhr-1wOUew==/lib/arm64/libpad.so
[hook_mmap13_openat] a1:  /data/app/jp.gungho.padHT-RB7leURHfwOLGhr-1wOUew==/lib/arm64/lib__6dba__.so

// hook get_so_info log:
[mmap13_get_so_info] this.a1.readPointer:  0x0
[mmap13_get_so_info] this.a1.readPointer:  0x0
[mmap13_get_so_info] this.a1.readPointer:  0x7cfb84e000
```

但後來詳細分析`get_so_info`後發現它其實只是在解析、保存`/proc/pid/maps`裡的信息 ( `so_info[0]`保存著 so 的二進制信息 )，前 2 次的`a1`為`0`是因為這時機還未加載那兩個 lib 庫，因此才為`0`。

繼續向下看，`so_info`( `so_img` ) 之後會傳入`do_something1`函數，返回值保存在`dest`，然後會與`*(_DWORD*)(v8+0x3C)`對比，若不相等會導致最終走向`wrong_branch`。

由此猜測`*(_DWORD *)(v8 + 0x3C)`應該是原始`lib__6dba_.so` .text 段的 hash 值，dest 是`/proc/pid/maps`裡`lib__6dba__.so` .text 段的 hash 值。

![](https://attach.52pojie.cn/forum/202505/06/225106m00eddaodad80301.png)

**image13.png** _(28.25 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY4MHw0NDQ5NDQ4MHwxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

進入`do_something1`，一開始在通過`so_img`解析重定向表，但沒看出來有什麼用。

![](https://attach.52pojie.cn/forum/202505/06/225106affmko1wjx7wdwow.png)

**image14.png** _(34.6 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY4MXxiMmQyMjBlYnwxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

繼續向下可以看到關鍵的 while 循環。

![](https://attach.52pojie.cn/forum/202505/06/225106ztxl8xcqkchhqp1m.png)

**image15.png** _(27.47 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY4MnxhODdhNDg2OHwxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

其中的`hash_sum`是一堆計算，應該是在計算類似哈希值的東西，嘗試 hook 該函數會發現`args[0]`曾出現過`lib__6dba__.so`的. text 段，`args[1]`是. text 段的大小，`args[2]`保存計算結果。

![](https://attach.52pojie.cn/forum/202505/06/225106z6d677dp6799ipcw.png)

**image16.png** _(63.8 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY4M3wzMzkwYzMxOXwxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

而後發現，針對自身的每個 so，總共會調用 2 次`hash_sum`(在兩處不同的位置) 來計算哈希值：

1.  第 1 次會對整個文件進行哈希，從下圖第 1 部份可以看出，`0x1860df`正是`lib__6dba__.so`的文件大小，而且在此之前調用`openat`打開了`lib__6dba__.so`。調用棧在`mmap13!0x3DF0`。
2.  第 2 次會對. text 段進行哈希，從下圖第 2 部份可以看出，`0xcaf4`正是`lib__6dba__.so`的. text 段大小，而且在此之前調用`openat`打開了`/proc/self/maps`，因此可知這部份是從其中獲取的。調用棧在`mmap13!0x6B98`，這正是上述的`do_something1`那裡。

第 1 次大概是為了校驗完整性之類的，第 2 次顯然就是在校驗是否被 hook，這樣使得常規的 IO 重定向似乎無法直接繞過？

![](https://attach.52pojie.cn/forum/202505/06/225106vspjlancsa9ppjja.png)

**image17.png** _(85.13 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY4NHwxNTRjZDllM3wxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

小結：對於 local lib(APP 自身的庫)，會調用`hash_sum`函數進行校驗，與之對比的值應該是提前計算好內置到 so 中的。

#### system lib 檢測分析

通過上述的 local lib 檢測後，才會繼續調用`check_libc`函數來檢測`libc.so`(貌似只檢測了 libc 這個系統庫)。下圖所在函數是`mmap13_main`。

![](https://attach.52pojie.cn/forum/202505/06/225106a99669e1c99rw9rx.png)

**image18.png** _(28.4 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY4NXwzOTRhNDc4Y3wxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

`check_libc`中調用了`do_something2`函數。

![](https://attach.52pojie.cn/forum/202505/06/225106lszloe3zlz3wunxx.png)

**image19.png** _(33.18 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY4NnxkMTI0MzhhNnwxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

接下來詳細分析`do_something2`函數。

首先調用`parse_elf_data`函數來解析指定 so，`args[0]`是`libc.so`映像的地址 (該映像是在此之前通過 openat 系統調用打開 & 讀取的)。解析結果保存在`soinfo`中 (這並非 linker 那個 soinfo)。

![](https://attach.52pojie.cn/forum/202505/06/225106dinp7pns57dpsmh2.png)

**image20.png** _(31.11 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY4N3w1MmMxODMxOHwxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

然後解密了一個關鍵字符串`.text`，傳入了`get_section_info`函數，它會返回`libc.so`的`.bss`段中的某段數據，其中包含指定 section 的信息，記為`section_info`。

如`*(section_info+0x10)`就是指定 section 的 offset。

![](https://attach.52pojie.cn/forum/202505/06/225106j1md7jj7d804w0s1.png)

**image21.png** _(27.06 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY4OHxjYzM2NWI1NHwxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

之後會遍歷`maps_item`( `/proc/pid/maps`的每一行我稱為一個`maps_item` )，當遍歷到`libc.so`的`.text`段的下一段時，才會滿足下圖的第 1 個 if 條件。

正常手機沒有啟動過 frida 時，會滿足第 2 個 if 條件 ( 即`.text`段的下一段一定大於等於`.text`段結束的位置 )，最終走到真正檢測 libc 的地方。

![](https://attach.52pojie.cn/forum/202505/06/225106nq0nt9pfuz0eex10.png)

**image22.png** _(38.36 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY4OXxiY2NkZTNkY3wxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

當不滿足上述第 2 個 if 條件時，會走下圖這裡，而且會循環多次。

第 1 個紅框代表最多循環 10 次，若遍歷完`.text`段的後 10 個`maps_item`仍沒有發現大於`.text`段結束的，代表有問題，最終會導致程序走向閃退的錯誤分支。

正常沒有被 frida 干預的程序流會在第 2 個紅框那裡直接`goto LABEL 49`。

![](https://attach.52pojie.cn/forum/202505/06/225106ldo608k09bzwkwxk.png)

**image23.png** _(68.98 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY5MHw5NDVkNDAzMXwxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

而`goto LABEL 49`最終會走到這裡，調用`do_check_libc`進行真正的 libc 校驗。

![](https://attach.52pojie.cn/forum/202505/06/225106b3o8jc7c57hi3wwa.png)

**image24.png** _(24.51 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY5MXxmOGZlOGI3MnwxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

`do_check_libc`函數裡有些關鍵字符串信息，如下。

![](https://attach.52pojie.cn/forum/202505/06/225107h9bovb2wow7z5at9.png)

**image25.png** _(26.31 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc3NTY5Mnw2MzYzNGM2N3wxNzQ2NTgwMDYxfDB8MjAyOTQ5OA%3D%3D&nothumb=yes)

2025-5-6 22:51 上传

而`do_check_libc`的具體原理，有興趣的靚仔可以自己分析看看。

### 完全繞過所有 hook 檢測的思路

通過 hook `mmap13`模塊的`vsnprintf`繞過對`lib__6dba__.so`和`libc.so`的校驗後，會加載`libopenal.so`和`libpad.so`(它們是 APP 自身的 so 庫)，然後發現這兩個 so 庫同樣存在與`lib__6dba__.so`一樣的`start`函數，同樣存在上述的 mmap 模塊檢測，同樣會檢驗 local lib 和 system lib。

好消息是它們大致使用了相同的 mmap 模塊來進行檢測，不同的只有 mmap 模塊創建的數量，如`libopenal.so`創建的`mmap11`模塊其實是`lib__6dba__.so`創建的`mmap13`模塊。

而 mmap 模塊會調用`vsnprintf`來拼接庫的完整路徑，因此可以 hook `vsnprintf`來改變指定庫路徑，重定位到其他沒有被 hook 的庫，以此來繞過檢測。具體方式在上文中已經給出，就不再重複。

### 結語

這個遊戲的保護是我遇到數一數二難的，難點在於它十分麻煩，且只能以 hook 的方式來調試，但找對方法後還是可以一點一點分析並解決的，不至於像一些 VM 那樣無從下手。

同時本文只大致分析了其中的一個模塊，各位讀者有興趣可以自己看看其他模塊，大概有 29 個模塊，也是挺有意思的。

![](https://avatar.52pojie.cn/data/avatar/001/89/82/08_avatar_middle.jpg)ngiokweng 樣本: jp.gungho.padHT  
版本: 22.1.0![](https://avatar.52pojie.cn/images/noavatar_middle.gif)yunxin0yu wow，太强了楼主，学习到了 ![](https://avatar.52pojie.cn/data/avatar/001/89/82/08_avatar_middle.jpg) ngiokweng

> [yunxin0yu 发表于 2025-5-6 23:14](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=52995474&ptid=2029498)  
> wow，太强了楼主，学习到了

大晚上還在學習![](https://static.52pojie.cn/static/image/smiley/laohu/laohu33.gif)
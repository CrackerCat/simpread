> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-288477.htm)

> [原创] 初窺 NP 手遊保護

0. 前言
=====

某天在應用商店挑選「幸運兒」時，一不小心選到了 NP 保護的，我想這便是天意。

自那天起便開啟了這段漫長的分析之旅，數以百次的調試，最後留下本文。

樣本: anAuZ29vZHNtaWxlLnRvdWhvdWxvc3R3b3JkX2FuZHJvaWQ=

1. `libcompatible.so`分析
=======================

NP 的關鍵 so 為`libcompatible.so`、`libstub.so`和`libengine.so`，三者之間環環相扣，先看`libcompatible.so`。

1.1 導出符號混淆
----------

通過`readelf`找到`.init_proc`在`0x154028`。

```
# readelf -d libcompatible.so
readelf: Error: no .dynamic section in the dynamic segment
 
Dynamic section at offset 0x1abd80 contains 29 entries:
  Tag        Type                         Name/Value
 0x000000000000000c (INIT)               0x154028

```

發現`.init_proc`被一些導出符號分割了，導致 IDA 無法 F5。

![](https://bbs.kanxue.com/upload/attach/202509/946537_MG2HDKP94WNRN7G.webp)

用乐佬那篇文章的解決思路，手動把位於`.init_proc`範圍內的導出符號置空即可。

```
import struct
 
input_path = r"libcompatible.so"
output_path = r"libcompatible_p.so"
 
sym_start = 0x2F8
sym_end = 0xACD8
sym_size = 0x18
 
init_proc_start = 0x154028
init_proc_end = 0x15B06C
 
data = None
with open(input_path, mode = "rb") as f:
    data = bytearray(f.read())
 
for sym in range(sym_start, sym_end, sym_size):
    st_value = struct.unpack("= init_proc_start and st_value < init_proc_end):
        data[sym + 8: sym + 8 + 8] = struct.pack("
```

patch 後就能順利 F5 了，發現`.init_proc`中有如下虛假控制流。

![](https://bbs.kanxue.com/upload/attach/202509/946537_76PSF5D6C6SM5C2.webp)

bcf 的解決思路可參考 oacia 大佬的這篇文章：[https://oacia.dev/ollvm-study/](elink@ea6K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6G2j5h3y4A6j5g2)9J5k6h3c8W2N6W2)9J5c8X3!0D9L8s2k6E0i4K6u0V1M7%4c8#2k6s2W2Q4x3V1j5`.)

1.2 init_proc 分析
----------------

計算獲取`libcompatible.so`的基址，保存在`libcompatible_base`中。然後調用`give_load_seg_rwx()`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_NZHFVEDK67ZUDN5.webp)

`give_load_seg_rwx()`的實現如下，根據`libcompatible_base`解析 & 遍歷 phdr，記錄最後一個 loadable seg 的結束位置，記為`last_loadseg_end`。

將`last_loadseg_end`對齊內存頁大小後的值作為`mprotect()`的 size，保證基本上全部代碼段、數據段都有 rwx 權限。

![](https://bbs.kanxue.com/upload/attach/202509/946537_WD7NUEYKH7SJRFB.webp)

回到`.init_proc`繼續向下看。

調用了`mmap`系統調用，映射了一頁`rwx`權限的內存，記為這片內存為`mmap_buf`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_GHN6HTWGPQAVTCE.webp)

調用`decrypt_something1()`，解密了一些數據，結果存放在`data_from_dec`中。

![](https://bbs.kanxue.com/upload/attach/202509/946537_VQCXEQX25FVZMQ8.webp)

調用`decrypt_something2()`，根據`data_from_dec`來解密`args[1]`。記解密後的數據為`data_from_dec2`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_DUVJXM5AFUY95D5.webp)

然後會根據`data_from_dec2`來對`mmap_buf`賦值。

![](https://bbs.kanxue.com/upload/attach/202509/946537_BV6R77R2K62CWM4.webp)

由於`mmap_buf`有執行權，因此嘗試將賦予`mmap_buf`的數據解析為代碼，發現是 svc。

![](https://bbs.kanxue.com/upload/attach/202509/946537_NUYKQVEP4HFDUJJ.webp)

之後 svc 會被保存到全局變量中。

![](https://bbs.kanxue.com/upload/attach/202509/946537_CD4UZ9Z496MNNM2.webp)

完成 svc 的賦值後，調用了`cache_maintenance()`來更新`mmap_buf`的 cache 緩存。

![](https://bbs.kanxue.com/upload/attach/202509/946537_M39RSC4VM66RVWM.webp)

調用`decrypt_some_str()`解密了一些字符串，保存在全局變量中。

它的字符串解密邏輯如下：

1.  調用`parse_data()`解析`args[1]`，結果存在`args[0]`中。
2.  調用`get_data_string()`從上述的`args[0]`提取字符串地址。

解密了一些`ro.`屬性，大概是在做兼容。

![](https://bbs.kanxue.com/upload/attach/202509/946537_M324BMBMNEPJ39U.webp)

之後獲取了當前 so 的`.dynamic`段。

![](https://bbs.kanxue.com/upload/attach/202509/946537_QGWEVNR5M9WYZUJ.webp)

解析`.dynamic`，獲取重定向相關信息，如 RELA 表，JMPREL 表等。

![](https://bbs.kanxue.com/upload/attach/202509/946537_329BU6PMG57GF5P.webp)

清空重定向表。

![](https://bbs.kanxue.com/upload/attach/202509/946537_69XCP5JYAVTTAPU.webp)

![](https://bbs.kanxue.com/upload/attach/202509/946537_R9KD8EGUHNE4RAB.webp)

最後會分別調用`I1 ~ I3`這 3 個導出函數 (是在上面被解密的)，調用後會把函數加密回去 ( 那些異或操作一開始以為是解密，後來才發現是加密 )。  
![](https://bbs.kanxue.com/upload/attach/202509/946537_U39UZQ3JA56V8HC.webp)

簡單看了下`I1`和`I2`，沒有太特別的地方，重點在`I3`這個導出函數。

1.3 I3 函數分析
-----------

`sub_7A635BF48C()`設置了某些導出符號的其中幾個字節，不知在干什麼。

之後`mmap`了一片 rw 權限的內存，記為`mmap_buf2`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_ZMH93XW2UX76PCA.webp)

對`mmap_buf2`進行賦值。

![](https://bbs.kanxue.com/upload/attach/202509/946537_GGYX3V9UNNE4BZK.webp)

之後會來到下圖的地方，若 F7 單步步入紅框慢慢跟，IDA 似乎會 crash，而 F8 直接步過則不會？

![](https://bbs.kanxue.com/upload/attach/202509/946537_6ESFZCATB8AYDT9.webp)

把某個函數賦給了`some_func1`變量

![](https://bbs.kanxue.com/upload/attach/202509/946537_PWZAJ7WDATJ536F.webp)

### 1.3.1 解密 libcompatible.so 的 JNI_OnLoad

之後就來到第 1 處 fla，顯然是關鍵邏輯處。

![](https://bbs.kanxue.com/upload/attach/202509/946537_UVBMVGTAGJZDU5F.webp)

簡單看看這個 fla。

這個地方在解密一段代碼，把`v144`指向的地址減去基址，得到的偏移記為`off`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_BY8KTXTKGXYPRQ4.webp)

從 dump 出來的`libcompatible.so`中搜`off`，發現`off`在`JNI_OnLoad()`中，由此可知上述解密的代碼正是`JNI_OnLoad()`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_X8QW7GKNBGHPDRH.webp)

第 2 處解密`JNI_OnLoad()`的地方，由此可知`JNI_OnLoad()`是分段解密的。

![](https://bbs.kanxue.com/upload/attach/202509/946537_2GCGFBBWXU6GH2K.webp)

第 3 處解密`JNI_OnLoad()`的地方。

![](https://bbs.kanxue.com/upload/attach/202509/946537_S5BPQQ3UV2Z2W7Z.webp)

`JNI_OnLoad()`的解密應該只分成了 3 段，解密完後會調用`cache_maintenance()`，似乎每次解密完代碼後都會調用該函數做 cache 相關的處理。

`cache_maintenance()`的`args[0]`是被解密代碼的起始地址，`args[1]`是結束地址，這個地址範圍中包含多個解密後的函數。  
![](https://bbs.kanxue.com/upload/attach/202509/946537_WSFVFKPUP8F4C56.webp)

### 1.3.2 一些檢測邏輯

然後就是第 2 處 fla，記為`fla2`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_YQX6BNF6QFXZ3HA.webp)

同樣是一段代碼解密邏輯，解密的是`0xF70D0`處的代碼。

![](https://bbs.kanxue.com/upload/attach/202509/946537_YTH9ZTBT48NN94D.webp)

解密後發現只是一個功能函數。

![](https://bbs.kanxue.com/upload/attach/202509/946537_RNED5KV52SSS8SR.webp)

`fla2`中第 2 處代碼解密邏輯。

![](https://bbs.kanxue.com/upload/attach/202509/946537_878CZQYDZV8C4ZD.webp)

`fla2`中第 3 處代碼解密邏輯。

![](https://bbs.kanxue.com/upload/attach/202509/946537_65B2EPUYQTSNE2E.webp)

注：上述的代碼解密邏輯可能會被交替調用來對同一個函數進行解密。

解密完成後，同樣調用了`cache_maintenance()`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_F5BNQEHRKQCZBFX.webp)

然後調用了`check_something()`進行一些檢測，其中又大概可分成`check1() ~ check5()`5 個小檢測。

![](https://bbs.kanxue.com/upload/attach/202509/946537_5TTFGZHWHRPN7D4.webp)

先看`check1()`，一開始調用了`prctl("PR_SET_DUMPABLE")`，不知為何。

然後調用`newfstatat`系統調用獲取`/proc/<pid>/environ`相關信息。

![](https://bbs.kanxue.com/upload/attach/202509/946537_NGCFSQ6NA5M86ZD.webp)

將`statbuf`設置為`struct stat statbuf`類型後，可以看出調用`newfstatat()`是為了獲取時間戳來實現時間檢測。

*   `tv_sec`是`/proc/<pid>/environ` 的**最後修改時間**（`st_mtim`）。
*   `statbuf.st_atim` 是 `/proc/<pid>/exe` 的**最後存取時間**（`st_atim`）。
*   因此`v12 == 1` 代表 **environ 的修改時間晚於 exe 的存取時間。**

![](https://bbs.kanxue.com/upload/attach/202509/946537_M5UPB2Y72GMHEA4.webp)

![](https://bbs.kanxue.com/upload/attach/202509/946537_PMY9P4X7X575HVZ.webp)

接下來看`check2()`。

解密了`"magisk"`，然後調用`openat`系統調用打開`"/proc/self/mounts"`，顯然是在檢測 magisk。

![](https://bbs.kanxue.com/upload/attach/202509/946537_FMT3X6SECXYE329.webp)

調用`read`系統調用讀取`/proc/self/mounts`，每次讀 2048 字節，其中包含換行符，要手動處理。

處理完後就是檢查`/proc/self/mounts`每行是否包含 magisk 字樣，以此來檢測 magisk。

![](https://bbs.kanxue.com/upload/attach/202509/946537_KCXWFE9DX38N897.webp)

然後`check3()`是經典的 root 檢測。

![](https://bbs.kanxue.com/upload/attach/202509/946537_HQM6HA3UZN7YJSR.webp)

![](https://bbs.kanxue.com/upload/attach/202509/946537_HKRAMRQ7H3W8FJH.webp)

`check4()`是動調檢測。

![](https://bbs.kanxue.com/upload/attach/202509/946537_VCRVEBAZ86NWWXM.webp)

最後是`check5()`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_55D55MYXK7NSGNT.webp)

`check5_func1()`打開了`/proc/self/maps`，解密了解析 maps 文件的格式化字符串，保存在`a1`中。

![](https://bbs.kanxue.com/upload/attach/202509/946537_E4EAK4JPDVWCBQG.webp)

`check5_func2()`調用 read 系統調用讀取 maps 文件，同樣手動處理換行符後，調用自實現的`sscanf`解析 maps 信息，最後保存到`a1`中。

![](https://bbs.kanxue.com/upload/attach/202509/946537_MJJ72KR99PMSJYG.webp)

調用完`check5_func2()`後，解密出了`"/system/bin/app_process"`字串，然後與`maps_lib_path`對比。

這裡是為了匹配 maps 中的`/system/bin/app_process`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_JMXKKJGBZQGDWQU.webp)

匹配成功後，調用`process_app_process()`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_VBRMRUZJ82B33GN.webp)

`process_app_process()`會遍歷內存中的`/system/bin/app_process64`，看看其中是否存在`magisk`字串。

![](https://bbs.kanxue.com/upload/attach/202509/946537_U4RK6WRPG5RKN4W.webp)

`process_app_process()`檢查完之後，會調用`check5_func3()`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_B636DDU32NQ6ZAW.webp)

`check5_func3()`同樣是一些 magisk 檢測，如下：

1.  判斷 PATH 環境變量中是否包含`.magisk/`

![](https://bbs.kanxue.com/upload/attach/202509/946537_PDJUXZUFU9XMXXJ.webp)

1.  判斷 PATH 環境變量中是否包含`MAGISK`

![](https://bbs.kanxue.com/upload/attach/202509/946537_NMFWXMVPV87NKHG.webp)

1.  判斷包名結尾是否`_zygote`

![](https://bbs.kanxue.com/upload/attach/202509/946537_SG5UEKA3M629Z8F.webp)

至此分析完`check_something()`。

### 1.3.3 調用. init_array 函數 & 收尾

回到`I3`函數向下看。

有個 while 循環不斷從一個類似函數列表的地方取函數並調用，這個函數列表大概就是解密後的`.init_array`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_FYT897R8J5M9HZF.webp)

`.init_array[0]`中調用了`unsetenv("LD_PRELOAD")`，這大概是一種反注入的機制。`unsetenv()`還會再遍歷一次 PATH 環境變量，確保`LD_PRELOAD`真的被 unset 掉，否則會直接`exit()`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_B9J5TX38G3N6YQU.webp)

除了`.init_array[0]`外，其他`.init_array`函數似乎都與檢測無關。

1.4 JNI_OnLoad 分析 (part1)
-------------------------

由上述分析可知，`I3`函數執行完後，`JNI_OnLoad()`也被解密完成。因此在那時機下斷點跳過去分析`JNI_OnLoad()`。

開始分析`JNI_OnLoad()`。

調用了`mprotect`系統調用，賦予某片內存 rwx 的權限。

![](https://bbs.kanxue.com/upload/attach/202509/946537_KVJAKDY8AKMQ7HT.webp)

保存了`JNI_OnLoad`的前`0x10`字節，暫時未知用來做什麼。

![](https://bbs.kanxue.com/upload/attach/202509/946537_SXKHJ9BWYJMUQUH.webp)

然後就是熟悉的控制流。

![](https://bbs.kanxue.com/upload/attach/202509/946537_K6G8UW7UXRNABBB.webp)

同樣是一些函數解密的邏輯。

![](https://bbs.kanxue.com/upload/attach/202509/946537_P3TJDCHXMR4XHPC.webp)

![](https://bbs.kanxue.com/upload/attach/202509/946537_XSEXVNFHECG4UBX.webp)

最終同樣會調用`cache_maintenance()`。

之後調用了`sub_777F32BB58()`，其中會間接調用`a1 + 48`指向的函數。

![](https://bbs.kanxue.com/upload/attach/202509/946537_NK7R29S9KW84J74.webp)

跟進去後發現是`GetEnv()`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_S55DE4XUPV7N4Q5.webp)

然後又判斷了一次 package name 是否以`_zygote`結尾。

![](https://bbs.kanxue.com/upload/attach/202509/946537_YFXJGKHPPXZUPEU.webp)

之後調用了`JNI_OnLoad_func1()`進行一些檢測。

![](https://bbs.kanxue.com/upload/attach/202509/946537_D8F5DYQJRQMXN76.webp)

`JNI_OnLoad_func1()`中解密了以下字符串：(都是模擬器的特徵)

*   `"lib3btrans.so"`
*   `"libhoudini.so"`
*   `"/lib/arm/nb/libc.so"`
*   `"/lib64/arm64/nb/libc.so"`

檢測的邏輯同樣是按上述方式解析`/proc/self/maps`後，看看是否存在這些 so，是則代表是模擬器。

![](https://bbs.kanxue.com/upload/attach/202509/946537_H7ZVRC4BDW8UCN5.webp)

之後反射調用了一個 Java 層的函數。

![](https://bbs.kanxue.com/upload/attach/202509/946537_2RD92XBYBN96XW7.webp)

發現只是一個固定返回`true`函數？

![](https://bbs.kanxue.com/upload/attach/202509/946537_SFC5F7P6EUWM6JW.webp)

然後動態注冊了一些 JNI 函數。

![](https://bbs.kanxue.com/upload/attach/202509/946537_3P4MFWRBTWKB6F4.webp)

第 1 個`register_natives()`注冊了以下 Java 類的 native 函數：

*   `com/inca/security/Native/AppGuardPreAssistantNative`
*   `com/inca/security/Scalar/SecureType`
*   `com/inca/security/DexProtect/SecureApplication`

第 2 個`register_natives()`注冊了以下 Java 類的 native 函數：

*   `com/inca/security/Proxy/iIiIiIiIii`

![](https://bbs.kanxue.com/upload/attach/202509/946537_HV478PHGTBPV98T.webp)

然後又解密了一個函數 ( 下圖的`sub_777E313F98()` ) ，記該函數為`JNI_OnLoad_decfunc1()`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_2EDAPPKN5DJWVKM.webp)

1.5 JNI_OnLoad_decfunc1 分析 (part1)
----------------------------------

進行了一些運算後，調用了`KM4PI0Z7J8QMILO5G6P6()`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_9USHQN797W4YU8F.webp)

`KM4PI0Z7J8QMILO5G6P6()`記錄了以下目錄的「最後存取時間」之和，但不知有什麼用。

*   `"/storage/emulated/0/Music/"`
*   `"/storage/emulated/0/Android/"`

![](https://bbs.kanxue.com/upload/attach/202509/946537_WW2WDWU8GTDA63Q.webp)

之後又在一處 fla 中解密了某個函數，記該函數為`big_func()`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_3UCFNWVJPYQGCX6.webp)

1.6 `big_func`分析
----------------

之所以叫`big_func()`，是因為該函數的 F5 偽代碼有六千多行。

### 1.6.1 sapi 初始化

一開始解密了一些與 libc 相關的字串，通過`sprintf()`組裝後傳入了`parse_libc()`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_EN6W9WS7CMXTZ9M.webp)

`parse_libc()`先從`/proc/self/maps`獲取`libc.so`的相關信息，然後調用`do_parse_libc()`進行解析。

![](https://bbs.kanxue.com/upload/attach/202509/946537_A2Q9JXG9RH735TM.webp)

`do_parse_libc()`開頭解析了 libc 的 dynamic。

![](https://bbs.kanxue.com/upload/attach/202509/946537_VE64E82H3D5C6BQ.webp)

`do_parse_libc()`最後分別調用了`get_libc_info()`來將 ELF GNU Hash Table 和 dynamic 等信息保存在`args[0]`中。

然後調用`save_some_libc_sym()`獲取了 libc 中的一些符號偏移，如`clone()`、`__libc_sysinfo`、`_ZL13g_thread_list`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_8ZQ5U2SSXAVKTJC.webp)

回到`big_func()`繼續向下看。之後調用了`lookup_libc_sym()`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_A7ZYWV6EXRUGCSQ.webp)

`lookup_libc_sym()`中調用了`find_symbol_by_name()`來根據函數名獲取對應的符號地址 (原理是 gnu hash)，保存在全局變量`g_libc_funcs`中。

![](https://bbs.kanxue.com/upload/attach/202509/946537_KFR7KZ2N35CK5UU.webp)

![](https://bbs.kanxue.com/upload/attach/202509/946537_BT7MY32ZTX9RUTU.webp)

之後又解密了一些函數來加載自己的 libc，主要邏輯在`load_mylibc()`中，加載後的 libc 記為`mylibc`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_FRJSHS6XTVK79CT.webp)

`load_mylibc()`主要干了以下事情：

1.  調用`LoadSection()`加載所需的 section 信息。

![](https://bbs.kanxue.com/upload/attach/202509/946537_WWNXV3JYNK4BGGA.webp)

1.  調用`ClearShdr()`

![](https://bbs.kanxue.com/upload/attach/202509/946537_KHG59T2U2PKJ6UB.webp)

1.  重定向：

![](https://bbs.kanxue.com/upload/attach/202509/946537_57VARWCJDS2TTGZ.webp)

注：分析過程中會發現在`LoadSection()`和`ClearShdr()`最後都有以下函數調用，其`args[3]`似乎就代表了所在函數的功能。

```
// 在LoadSection()最後:
sub_777F3E56B0(1u, 0x64u, 9u, (__int64)"LoadSection", 0x120u);
 
// 在ClearShdr()最後：
sub_777F3E56B0(1u, 0x64u, 9u, (__int64)"ClearShdr", 0x344u);

```

`load_mylibc()`執行完之後，從`mylibc`獲取了一些函數單獨保存下來，如`malloc`、`calloc`、`realloc`等等。

![](https://bbs.kanxue.com/upload/attach/202509/946537_W9UWVTJE2F8PK82.webp)

之後調用`prepare_for_inline_hook()`來保存指定 mylibc 函數的前 0x32 字節，結果保存在`args[0]`中，然後傳入`inline_hook()`進行 hook。

![](https://bbs.kanxue.com/upload/attach/202509/946537_BA8HN3Y43MWMXPP.webp)

以 mylibc 的`calloc()`為例，在`inline_hook()`後，前 0x10 字節被改為跳到原 libc 的`calloc()`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_MYZRH8RFK8EWWNY.webp)

之後又繼續從`mylibc`獲取了一堆函數單獨保存到全局變量中，不一一列出了。

但獲取的目的並不是為了像上面那樣 inline hook，而大概是之後會用到，所以提前保存下來。

![](https://bbs.kanxue.com/upload/attach/202509/946537_EUPP7T54DHF7CKG.webp)

1.7 `JNI_OnLoad_decfunc1`分析 (part2)
-----------------------------------

`big_func()`執行完後，回到`JNI_OnLoad_decfunc1()`。

同樣的形式解密了一段代碼。

![](https://bbs.kanxue.com/upload/attach/202509/946537_H38ZUU5F9ETFP83.webp)

### 1.7.1 anti xposed & anti debugging

`check_xposed()`中遍歷了 maps，看看是否存在`XposedBridge.jar`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_EZDF5GYHKJNQAGP.webp)

之後會來到一個反調試的函數，記為`anti_debugging()`

![](https://bbs.kanxue.com/upload/attach/202509/946537_N4EFNJH5DWK89NC.webp)

調用`mylibc`的`pthread_create()`創建了一個線程。

![](https://bbs.kanxue.com/upload/attach/202509/946537_8GJAY4J64GTMH6Z.webp)

調用`mylibc`的`fork()`創建了兩個子線程，之後調用`signal(17, 1)`忽略子線程結束時送出的`SIGCHILD`信號。

嘗試動調此處邏輯，但總會發生一些非預期的結果，因此無法詳細分析`anti_debugging()`的具體實現原理。

唯一確定的是它調用了兩次 mylibc 的`fork()`，使得此時機後將 IDA 無法 attach，記這種反調試為三進程保護。

![](https://bbs.kanxue.com/upload/attach/202509/946537_JFQJHFYQMJ7AC5V.webp)

之後調用`mylibc`裡的`pthread_create()`創建一個線程，線程回調函數記為`JOdf1_pthread_func3()`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_ZF6VDW9S4C7XXPK.webp)

### 1.7.2 JOdf1_pthread_func3 分析

一開始調用了`check5()` (上面已經分析過該函數，不再重複)。

然後調用`NativeBridge_check()`進行 NativeBridge 注入檢測。

![](https://bbs.kanxue.com/upload/attach/202509/946537_FZC696A425GZXRF.webp)

NativeBridge 注入檢測的原理參考[這篇文章](https://bbs.kanxue.com/thread-286536.htm)，檢測流程如下：

1.  獲取`ro.dalvik.vm.native.bridge`屬性值，不為空則繼續後續流程 ( 但其實即使在正常真機的環境下，返回的也是`"0"`，同樣會繼續後續的檢測流程 )。

![](https://bbs.kanxue.com/upload/attach/202509/946537_ZPREF6BJN9VY8JW.webp)

1.  從 maps 獲取`libnativebridge.so`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_NCKUD9F2PAZ79PH.webp)

1.  調用`libnativebridge.so`的`NativeBridgeError()`，若返回值非`0`代表檢測到。

![](https://bbs.kanxue.com/upload/attach/202509/946537_JUBAS8GDNYNF669.webp)

1.8 JNI_OnLoad 分析 (part2)
-------------------------

`JNI_OnLoad_decfunc1()`執行完後，回到`JNI_OnLoad()`，又調用了`mylibc`的`pthread_create()`創建了線程。

之後會間接調用一個不正常地址導致 SIGSEV？但 pass to app 後可以繼續執行，過段時間後才會真正 crash？

![](https://bbs.kanxue.com/upload/attach/202509/946537_7UP7BGWDDN79YUP.webp)

2. bypass 三進程保護
===============

要先 bypass 掉`libcompatible.so`的反調試，也能繼續動調後面的`libstub.so`。

2.1 bypass 嘗試
-------------

嘗試直接 patch 掉`anti_debugging()`。

```
function hook_anti_debugging(base) {
    Interceptor.replace(base.add(0x18C148), new NativeCallback(function (arg0) {
        console.log("[hook_anti_debugging] a0: ", hexdump(arg0.readPointer()));
        return ptr(0);
    }, 'pointer', ['pointer']));

}


```

但報了`SIGSEGV`的錯，報錯時的 PC 在`0x7130`，十分奇怪。

![](https://bbs.kanxue.com/upload/attach/202509/946537_BGW2GB4WYD3QBV9.webp)

用 frida + IDA 動調後發現報錯點在下圖這裡。

![](https://bbs.kanxue.com/upload/attach/202509/946537_63VC3J2QMP5WS5F.webp)

這種情況有兩種可能性：

1.  `0x7130`存放在某個全局變量`a`中，而`anti_debugging()`中會修改`a`的值為某個函數地址。
2.  `anti_debugging()`中注冊了`SIGSEGV`的信號回調函數。

而情況 1 基本可以排除掉，因為直接動調 ( 沒有 patch `anti_debugging()` )，走到上圖位置時同樣是`0x7130`，而且用 trace 排查時，顯示的也是`0x7130`。

因此大概率是情況 2，而且調用的大概率也是`mylibc`的`sigaction`。

嘗試 hook mylibc 的`sigaction()`，但沒有觸發。

改為 hook 原 libc 的`sigaction`，反而有觸發，看來 NP 無法使用 mylibc 的`sigaction()`來注冊信號回調？

```
Interceptor.attach(Module.findExportByName("libc.so", "sigaction"), {
    onEnter: function (args) {
        console.log("[hook_sigaction] a0: ", (args[0]));
        if (args[1].toInt32()) {
            console.log("[hooksigaction] a1: ", hexdump(args[1]));
            console.log("[hook_sigaction] a1: ", JSON.stringify(Process.findModuleByAddress(args[1].add(0x8).readPointer())));
            console.log("[hook_sigaction] a1: ", ptr(args[1].add(0x8).readPointer() - base));
        }
    
    },
    onLeave: function (retval) {
        console.log("[hook_mylibc_sigaction] retval: ", retval);
    }
});


```

輸出如下，`0xb`正是`SIGSEGV`，信號回調函數在`libcompatible.so!0xaa994`。

```
[hook_sigaction] a0:  0xb
[hooksigaction] a1:               0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
7fc4082ba0  04 00 00 08 00 00 00 00 94 99 81 9e 71 00 00 00  ............q...
7fc4082bb0  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
7fc4082bc0  12 ba 85 03 7c 62 4e 70 00 00 00 00 00 00 00 00  ....|bNp........
7fc4082bd0  80 2d 08 c4 7f 00 00 00 2c be 8f 9e 71 00 00 00  .-......,...q...
7fc4082be0  ac f4 8f 9e 71 00 00 00 9c 38 ba 84 00 00 00 00  ....q....8......
7fc4082bf0  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
7fc4082c00  50 36 92 9e 71 00 00 00 00 00 00 00 00 00 00 00  P6..q...........
7fc4082c10  70 56 93 9e 71 00 00 00 b4 56 93 9e 71 00 00 00  pV..q....V..q...
7fc4082c20  7c 31 92 9e 71 00 00 00 ac f4 8f 9e 71 00 00 00  |1..q.......q...
7fc4082c30  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
7fc4082c40  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
7fc4082c50  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
7fc4082c60  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
7fc4082c70  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
7fc4082c80  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
7fc4082c90  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
[hook_sigaction] a1:  {"name":"libcompatible.so","base":"0x719e76f000","size":1863680,"path":"/data/app/jp.goodsmile.touhoulostword_android-KNfNiSYf49IV5K_x4TTh3Q==/lib/arm64/libcompatible.so"} 
[hook_sigaction] a1:  0xaa994
[hook_mylibc_sigaction] retval:  0x0


```

`0xaa994`如下，記為`SIGSEGV_cb()`

![](https://bbs.kanxue.com/upload/attach/202509/946537_96SW9S45RSSB4RD.webp)

在`sub_187310()`中可以看到`0x7130`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_GZJC9JRP7MEZS4Z.webp)

`SIGSEGV_cb()`最終會根據 fault address 分發不同的函數，之後會看到。

3. libstub.so 分析
================

在 bypass 掉三進程保護後，終於可以動調分析後續的流程，首先是`libstub.so`的加載。

3.1 SoLibraryStart 分析
---------------------

`libstub.so`的. init_proc 中會調用`libcompatible.so!SoLibraryStart()`來解密。

`SoLibraryStart()`中的`qword_1B41E0`固定是`0x4580`，因此必然會觸發 SIGSEGV 的信號回調`SIGSEGV_cb()`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_JXNBQQA2A23MCS3.webp)

`SIGSEGV_cb()`會根據導致異常的地址來執行對應的函數，如`0x4580`會執行`func_4580()`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_S2ZWUVGA96KVW7J.webp)

3.2 func_4580 分析 (part1)
------------------------

打斷點跳到`func_4580()`進行分析。

調用了`check_and_decrypt_libstub()`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_RHGDDSFUD7722QE.webp)

`check_and_decrypt_libstub()`中會先調用`check_enc_flag()`確定`libstub.so`是加密的。

![](https://bbs.kanxue.com/upload/attach/202509/946537_4JZ3BVC5QURQHPM.webp)

`check_enc_flag()`具體實現如下，打開本地的`libstub.so`並檢查最後`20`字節是否與固定的加密標誌相等。

![](https://bbs.kanxue.com/upload/attach/202509/946537_EF7ZFH78JKVU6CE.webp)

而`decrypt_libstub_data()`一開始先獲讀取了`libstub[0:0xB758]`，然後再讀取`libstub[0x1CAA8: 0x1CAA8 + 0x3EFEE]`，記為後者為`libstub_enc_data1`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_EVPBGQ555QJ72WG.webp)

注：從 010 可知`0x1CAA8`正好在 section header 後面。

![](https://bbs.kanxue.com/upload/attach/202509/946537_8ZYWUF9PTZW47XM.webp)

第 1 處解密`libstub_enc_data1`的地方。

![](https://bbs.kanxue.com/upload/attach/202509/946537_CMW39GG766DYBGW.webp)

第 2 處解密`libstub_enc_data1`的地方。

![](https://bbs.kanxue.com/upload/attach/202509/946537_753PUBBB52NBPZF.webp)

第 3 處解密`libstub_enc_data1`的地方。

![](https://bbs.kanxue.com/upload/attach/202509/946537_A49GH7BVTUZ644F.webp)

調用`LZ4_decompress_fast()`對解密後的`libstub_enc_data1`進行解壓。

![](https://bbs.kanxue.com/upload/attach/202509/946537_HS6A7F44X8W3SHN.webp)

解壓後的數據如下，可以看到包含一些重定向信息，記為`dec_data1`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_QRN3SV76GA2XR46.webp)

讀取了`libstub[0x5BA96:0x5BA96 + 0x15C0]`，記為`libstub_enc_data2`，然後解密，再解壓。

![](https://bbs.kanxue.com/upload/attach/202509/946537_FNFCHWVUCGMG2VM.webp)

解壓後的數據如下，可以看出應該是符號表，記為`sym`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_GVYR6552UJ5KTR9.webp)

讀取了`libstub[0x5D056:0x5D056+ 0x1230]`，記為`libstub_enc_data3`，然後解密，再解壓。

![](https://bbs.kanxue.com/upload/attach/202509/946537_WJY7S2R5H7DHNDS.webp)

解壓後的數據如下，可以看出是字符串表，記為`strtab`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_HP569SMAFY72PVB.webp)

讀取了`libstub[0x5E286:0x5E286 + 0x29B0]`，記為`libstub_enc_data4`，然後解密，再解壓。

![](https://bbs.kanxue.com/upload/attach/202509/946537_3SBJVPPJ2KXRTCU.webp)

解壓後的數據如下，可以看到同樣是一些重定向信息 (403 重定向)，記為`relocate1`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_D6CZCGHQPJTZ9RZ.webp)

讀取了`libstub[0x60C36:0x60C36 + 0x4F0]`，記為`libstub_enc_data5`，然後解密，再解壓。

![](https://bbs.kanxue.com/upload/attach/202509/946537_P85RSYS7GUMEYHX.webp)

解壓後的數據如下，可以看到同樣是一些重定向信息 (402 重定向)，記為`relocate2`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_T4C95QQ9RA5PR4N.webp)

讀取了`libstub[0x61126:0x61126 + 0x30]`，記為`libstub_enc_data6`，然後解密，再解壓…

![](https://bbs.kanxue.com/upload/attach/202509/946537_DH82DVDMZ77J3NN.webp)

解壓後的數據如下，可以看出是`libstub.so`的依賴庫。

![](https://bbs.kanxue.com/upload/attach/202509/946537_PB9J36HBZ273F8W.webp)

最後讀取了`libstub[0x61146:0x61146 + 0x20]`，但似乎沒有用到，大概只是加密數據的結束標誌？

![](https://bbs.kanxue.com/upload/attach/202509/946537_HUG3DYBDTEH4V6A.webp)

至此大致分析完`check_and_decrypt_libstub()`。回到`func_4580()`。

調用`set_w_perm_to_libstub()`賦予`libstub.so`可寫的權限。`dlopen_needs()`會調用`dlopen()`加載`libstub.so`的所有`DT_NEED`庫。

`MEMORY[0x7510]()`會觸發`SIGSEGV_cb()`，然後跳到`func_7510()`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_G6KBPZA6HJ94RG6.webp)

3.3 func_7510 分析
----------------

看到`"ldrRestoreDynSymInfo"`，見名知意，大概是在恢復`libstub.so`的一些動態符號信息。

![](https://bbs.kanxue.com/upload/attach/202509/946537_VUDGSAEFJF9ZHA2.webp)

忽略前面一些不太知道在干什麼的邏輯後，來到下圖這裡，解密了一些符號名。

![](https://bbs.kanxue.com/upload/attach/202509/946537_HR5ZQ93SAU92YZP.webp)

![](https://bbs.kanxue.com/upload/attach/202509/946537_W2B6GU8UU4TFYQC.webp)

![](https://bbs.kanxue.com/upload/attach/202509/946537_KSP3JTWEUWEC7B4.webp)

然後會對比`libstub.so`的字符串表 (上面解密出來的那個)，若包含上述的符號名，則會把對應函數 ( 如下圖的`some_func2()`，它是`libcompatible.so`的函數 ) 保存到某個地方。

猜測是`libstub.so`之後會用到`libcompatible.so`中的一些函數，因而用這種方式提前保存函數地址到某處。

![](https://bbs.kanxue.com/upload/attach/202509/946537_SQCYY9CDD658X9R.webp)

3.4 func_4580 分析 (part2)
------------------------

`func_7510()`之後，回到`func_4580()`會執行重定向的邏輯，然後調用. init_array 的函數。

### 3.4.1 relocate

一開始解密了`"dlopen"`、`"dlsym"`、`"il2cpp.so"`等字符串，不知道干什麼。

![](https://bbs.kanxue.com/upload/attach/202509/946537_AQP9CHG6BUQ596G.webp)

然後就是熟悉的重定向操作：

![](https://bbs.kanxue.com/upload/attach/202509/946537_7Q6BX3RGEV27SHX.webp)

![](https://bbs.kanxue.com/upload/attach/202509/946537_9VTCBPEYDMKXFEY.webp)

注：重定向的對象並非 maps 中的`libstub.so`，而是內存中的`libstub.so`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_Y7TAA8YW4AHJWDT.webp)

通過 maps 查看內存中的`libstub.so`範圍，然後把這片內存 dump 下來，記為`libstub_dump.so`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_NCMP32YKQWGBZHZ.webp)

```
import idaapi
data = idaapi.dbg_read_memory(0x7c61330000, 0x7c613d2000 - 0x7c61330000)
fp = open(r'libstub_dump.so', 'wb')
fp.write(data)
fp.close()

```

### 3.4.2 call_constructors

調用`libstub.so`的`.init_array`函數。

![](https://bbs.kanxue.com/upload/attach/202509/946537_PJEFWCBP3AU3RU4.webp)

`.init_array`的偏移為`0x8FDE0`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_XE3Z9CTA89RKKWG.webp)

3.5 修復 libstub.so
-----------------

將`dec_data1` dump 下來會發現，其中包含了`relocate1`和`relocate2`，還有. dynamic 信息。

![](https://bbs.kanxue.com/upload/attach/202509/946537_WH9HJF4Q2FBU9B5.webp)

至於解密後的代碼段和數據段，應該已經在`libstub_dump.so`中，但`libstub_dump.so`中並不包含`strtab`、`sym`和. dynamic 的信息。

也不包含 shdr 和 shstrtab 字符串表，這兩者可從原`libstub.so`中獲取，前者在修復時直接覆蓋到原位置，後者隨便寫到最後的一片空區域即可。

![](https://bbs.kanxue.com/upload/attach/202509/946537_CSXMGTTK7U36PDE.webp)

從原`libstub.so`提取`shstrtab`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_4P2XRXYHFCAKQSG.webp)

section 修複，大概只有`.dynstr`、`.dynamic`、`.shstrtab`是必要的，第 0 個 section 要為 0，拉入 IDA 時才不會報錯。

![](https://bbs.kanxue.com/upload/attach/202509/946537_PSQ7XHWSVAETQ8N.webp)

把修復後的 libstub.so 拉入 IDA，發現只有少量的符號。

![](https://bbs.kanxue.com/upload/attach/202509/946537_MNUEDSJR75F8CJA.webp)

3.6 init_array_func1 分析
-----------------------

大概只有第 1 個. init_array 函數是檢測的邏輯，同樣也是對環境變量的檢測，具體做法是在`unsetenv("LD_PRELOAD")`後，遍歷`environ`數組，若發現其中仍有`LD_PRELOAD`則直接`exit()`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_S9SWYXVP6R5SG5J.webp)

3.7 JNI_OnLoad 分析
-----------------

保存了一些`libdl.so`、`libc.so`的函數到某個全局變量中。

![](https://bbs.kanxue.com/upload/attach/202509/946537_UUX2D5YJA8P6ZWB.webp)

嘗試尋找`"com/inca/security/DexProtect/SecureApplication"`類，但在本例會返回`0`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_BJXF8NYWWD9H7SQ.webp)

動態注冊了`"com/inca/security/Native/AppGuardAssistantNative"`、`"com/inca/security/AppGuard/TestCase"`中的一些 JNI 函數。

![](https://bbs.kanxue.com/upload/attach/202509/946537_T6BT7WQJHBSQ3PE.webp)

第 1 處`register_func()`動態注冊了 9 個 JNI 函數，如`startEngine()`、`stopEngine()`等等。

![](https://bbs.kanxue.com/upload/attach/202509/946537_3M44AN35R6JS465.webp)

第 2 處`register_func()`動態注冊了 4 個 JNI 函數，如下圖所示。

![](https://bbs.kanxue.com/upload/attach/202509/946537_8HWU3ZF33SM6BAR.webp)

最後創建了兩個線程，暫時不知有什麼用。

![](https://bbs.kanxue.com/upload/attach/202509/946537_W95WGUTYKBCQCBT.webp)

3.8 startEngine 分析
------------------

`startEngine()`被控制流混淆了，如下所示。

![](https://bbs.kanxue.com/upload/attach/202509/946537_ZRWHWSDTHPA65R6.webp)

同時其中充斥著大量的空函數，或許也是一種混淆。

![](https://bbs.kanxue.com/upload/attach/202509/946537_C2XDYXHNM8KC2GF.webp)

分析的思路是，忽略上面這樣的空函數，關注那些有內容的函數，下斷點跳過去調試。

最終可把`stratEngine()`分成三部份，第一部份如下。

![](https://bbs.kanxue.com/upload/attach/202509/946537_VJRKHET6PZP55GF.webp)

第二部份。

![](https://bbs.kanxue.com/upload/attach/202509/946537_FJVXBDQHPNBN3HQ.webp)

第三部份。

![](https://bbs.kanxue.com/upload/attach/202509/946537_NE3J86PBYX25TVU.webp)

在這三個部份中，都會調用某片匿名內存中的函數，一開始沒有多想，直接單步跟了進去調試，後來才想起這會不會就是`libengine.so`？

記那片匿名內存為`mem_libengine`，對比`mem_libengine`和`libengine.so`後發現，它們前面的字節的確是一樣的。( 本以為`startEngine()`會是加載`libengine.so`的邏輯，其實不然 )。

![](https://bbs.kanxue.com/upload/attach/202509/946537_USMPRJUM3TPZ86H.webp)

![](https://bbs.kanxue.com/upload/attach/202509/946537_4EPCMJ2DNFKY32B.webp)

4. libengine.so 分析
==================

4.1 哪裡加載的 libengine.so？
-----------------------

知道了`mem_libengine`就是`libengine.so`後，以此作為入手點，分析哪裡加載的。

在`libstub.so!JNI_OnLoad` leave 時機，maps 中仍未有`mem_libengine`。在`getVersion`、`setContext`時，maps 中有`mem_libengine`。

而在上面的分析中提過，`libstub.so!JNI_OnLoad`最後創建了一些線程，最終確定大概率是在`sub_53AC4`中加載的`libengine.so`。

通過 frida stalker 確定了`sub_53AC4`中會調用`libcompatible.so!0xA1990`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_9ZW85XHF63BNPSC.webp)

4.2 libengine.so 加載
-------------------

接下來看看`libcompatible.so!0xA1990`是怎麼加載`libengine.so`的。

首先調用了`LoadEngineLibrary()`，其中一開始拼接了`libengine.so`的絕對路徑。

![](https://bbs.kanxue.com/upload/attach/202509/946537_7KE9XEQ87WSX523.webp)

然後調用`sys_mmap()`映射一片匿名內存，調用`sys_lseek()` + `sys_read()`讀取`libengine.so`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_38VFBEKKJ4WRPHG.webp)

`LoadEngineLibrary()`之後會來到熟悉的解密函數 ( 在上方被我命名為`check_and_decrypt_libstub` )。

![](https://bbs.kanxue.com/upload/attach/202509/946537_2HTW9QAH6X73FEY.webp)

注：邏輯基本相同，不再重複分析，將所需數據 dump 下來。

![](https://bbs.kanxue.com/upload/attach/202509/946537_4UZXPCCZ8PR54RD.webp)

之後同樣調用了`dlopen_needs()`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_VHG3PM4UVRCWP6B.webp)

然後是`ldrRestoreDynSymInfo()`，但這次是直接調用，而非通過信號回調的形式來間接調用。

![](https://bbs.kanxue.com/upload/attach/202509/946537_QBC3M4KQSXJ3ZCV.webp)

然後是`relocate()`。

![](https://bbs.kanxue.com/upload/attach/202509/946537_KUH25ZYFBV4DKAH.webp)

可以在 403 重定向的位置，把內存中的`libengine.so` dump 下來。

![](https://bbs.kanxue.com/upload/attach/202509/946537_FM2F9UYP4QSJ4EB.webp)

注：`libengine_base`是由`*(a1 + 2680)`而來，`2560`則是`libstub.so`的。

![](https://bbs.kanxue.com/upload/attach/202509/946537_CHJS8VH5FUY2ZFB.webp)

4.3 修復 libengine.so
-------------------

大致跟修復`libstub.so`一樣，都需要從 dump 出來的 libengine 中提取`.dynamic`和`shstrtab`。

不同的是這次連 ehdr 和 phdr 都沒有，需要手動補上，而且 dump 出來的`libengine.so`中也不帶重定向信息，也需要手動補。

好消息是修復相對簡單，壞消息是只有一些無用的符號！！！

![](https://bbs.kanxue.com/upload/attach/202509/946537_TC5368CGSNA5FQ5.webp)

4.4 一些檢測
--------

通過 hook `libengine.so`的`get_data_string()`簡單看看其中的一些檢測點。

1.  第 N 次的環境變量檢測。

```
[libengine_get_data_string2]  LD_PRELOAD
[libengine_get_data_string2]  LD_PRELOAD
[libengine_get_data_string2]  LD_PRELOAD
[libengine_get_data_string2]  LD_PRELOAD
[libengine_get_data_string2]  LD_PRELOAD

```

1.  第 N 次的模擬器檢測。

```
[libengine_get_data_string2]  libhoudini.so
[libengine_get_data_string2]  lib3btrans.so
[libengine_get_data_string2]  /lib/arm/nb/libc.so
[libengine_get_data_string2]  /lib64/arm64/nb/libc.so

```

1.  一堆作弊工具。

```
[libengine_get_data_string2]  Unknown
[libengine_get_data_string2]  UserPattern
[libengine_get_data_string2]  GameGuardian
[libengine_get_data_string2]  GameKiller
[libengine_get_data_string2]  TeamCrakMemoryDump
[libengine_get_data_string2]  YouXiXiuGaiQi
[libengine_get_data_string2]  ChaoHaoWanXiuGaiQi
[libengine_get_data_string2]  ParadiseIslandTraniner
[libengine_get_data_string2]  QuanQuanYouXiZhouShou
[libengine_get_data_string2]  HuangHL
[libengine_get_data_string2]  GameCIH
[libengine_get_data_string2]  GameSpeeder
[libengine_get_data_string2]  SBTools
[libengine_get_data_string2]  GameCheater
[libengine_get_data_string2]  GameMaster
[libengine_get_data_string2]  HexEditor
[libengine_get_data_string2]  HaXplorer
[libengine_get_data_string2]  DaxAttack
[libengine_get_data_string2]  GMDSpeedTime
[libengine_get_data_string2]  CoolBoyTimer
[libengine_get_data_string2]  Freedom
[libengine_get_data_string2]  RootCloak
[libengine_get_data_string2]  LuckyPatcher
[libengine_get_data_string2]  CheatEngine
[libengine_get_data_string2]  BaMenYouXiXiuGaiShenQi
[libengine_get_data_string2]  SpeedWizard
[libengine_get_data_string2]  XiongMaoXiaYouXiZhuShou
[libengine_get_data_string2]  TransGameHacker
[libengine_get_data_string2]  AppCIH
[libengine_get_data_string2]  xxAssistant
[libengine_get_data_string2]  RepetiTouch
[libengine_get_data_string2]  BotMaker
[libengine_get_data_string2]  AnJianJingLing
[libengine_get_data_string2]  YouXiFengWo
[libengine_get_data_string2]  FingerReplayer
[libengine_get_data_string2]  HiroMacro
[libengine_get_data_string2]  SmartMacro
[libengine_get_data_string2]  TheToucher
[libengine_get_data_string2]  AutoTouch
[libengine_get_data_string2]  ScreencastVideoRecorder
[libengine_get_data_string2]  SCRScreenRecorder
[libengine_get_data_string2]  LiziModifier
[libengine_get_data_string2]  ZhuoMuNiao
// ...

```

1.  一堆奇怪的包名。

```
[libengine_get_data_string2]  com.vqs.iphoneassess
[libengine_get_data_string2]  com.huluxia.gametools
[libengine_get_data_string2]  com.huluxiakajkia
[libengine_get_data_string2]  com.huluxia.gametoolsdwaf
[libengine_get_data_string2]  com.huati
[libengine_get_data_string2]  com.zhushou.cc
[libengine_get_data_string2]  com.jbelf.imei
[libengine_get_data_string2]  com.mostwanted.crackzxc
[libengine_get_data_string2]  com.paojiao.youxia
[libengine_get_data_string2]  com.yx.youxia
[libengine_get_data_string2]  com.yx.youxia_shuih
[libengine_get_data_string2]  com.medroid.sbjiaqiangban
[libengine_get_data_string2]  com.burakgon.gamebooster2
[libengine_get_data_string2]  com.anzhuo.GameToos
[libengine_get_data_string2]  mobi.infolife.gamebooster
[libengine_get_data_string2]  com.android.gamespeedup
[libengine_get_data_string2]  com.iplay.assistant
[libengine_get_data_string2]  org.game.master
[libengine_get_data_string2]  com.mjmcmdjijkjnjn.wdjdy
[libengine_get_data_string2]  com.mnmfmnmfnnmhmo.wdmc
[libengine_get_data_string2]  com.jjjnjmjljnjljpjmjl.wd
[libengine_get_data_string2]  com.kkckchbjbe.ywwd
[libengine_get_data_string2]  cn.com.opda.gamemaster
[libengine_get_data_string2]  com.surcumference.xsposed
// ...

```

1.  一堆可執行文件。

```
[libengine_get_data_string2]  /dev/input
[libengine_get_data_string2]  /system/bin/input
[libengine_get_data_string2]  /system/bin/monkey
[libengine_get_data_string2]  /proc/self/maps
[libengine_get_data_string2]  /proc/%d/maps
[libengine_get_data_string2]  /proc/self/mem
[libengine_get_data_string2]  /proc/%d/mem
[libengine_get_data_string2]  /system/bin/dnsmasq
[libengine_get_data_string2]  /system/bin/dnsmasq
[libengine_get_data_string2]  /system/bin/efsks
[libengine_get_data_string2]  /system/bin/efsks
[libengine_get_data_string2]  /system/bin/pam_server
[libengine_get_data_string2]  /system/bin/pam_server
[libengine_get_data_string2]  pam_server
[libengine_get_data_string2]  pam_server
[libengine_get_data_string2]  /system/bin/pppd
[libengine_get_data_string2]  /system/bin/pppd
[libengine_get_data_string2]  /system/bin/ip6tables
[libengine_get_data_string2]  /system/bin/ip6tables
[libengine_get_data_string2]  /system/bin/iptables
[libengine_get_data_string2]  /system/bin/iptables
[libengine_get_data_string2]  /system/bin/ip
[libengine_get_data_string2]  /system/bin/ip
[libengine_get_data_string2]  /system/xbin/spritebud
[libengine_get_data_string2]  /system/xbin/spritebud
[libengine_get_data_string2]  /system/xbin/vold
[libengine_get_data_string2]  /system/xbin/vold
[libengine_get_data_string2]  /system/xbin/netd
[libengine_get_data_string2]  /system/xbin/netd
[libengine_get_data_string2]  /system/xbin/installd
[libengine_get_data_string2]  /system/xbin/installd
[libengine_get_data_string2]  /sbin/ueventd
[libengine_get_data_string2]  /sbin/ueventd
[libengine_get_data_string2]  /system/xbin/tcd
[libengine_get_data_string2]  /system/xbin/tcd
[libengine_get_data_string2]  /sbin/fsck_msdos
[libengine_get_data_string2]  /sbin/fsck_msdos
// ...

```

1.  Magisk

```
[libengine_get_data_string2]  Magisk v16.3
[libengine_get_data_string2]  24576:Q3LvBhUvgZiPYOvdwpohg+A6ht6vDM3Wr/ZLpzuzK:iqCc/Qo3WN0K
[libengine_get_data_string2]  24576:ODIjkIhUvgZicFNM3WFPn7IfyG+K6BACjYDlZfjzW:vw3WF7ji0EY
[libengine_get_data_string2]  192:b60cVwbwmw1m1SMzRvyOEgRphVdKT+GxLSpyijl:u0cV+wmw1m1SMzRvyOEgRphVdY5LSXjl
[libengine_get_data_string2]  192:01gS1jU9ywbtTgRt7r+k1Sv1QspVmT+3Pv3WWvP8h:01gSi9ywbtTgRRr+k1Sv1QspV0CPvWWu
[libengine_get_data_string2]  Magisk v16.4
[libengine_get_data_string2]  24576:TRAT0nEPHDSDC3N1Jd6/HIXaXfOO2SYC05PWIuul:aeDCfJd6/HIXa8PWIvl
[libengine_get_data_string2]  24576:i0yPElnEPHDSV4a5PWbzEDCHWJqV8/UcX+3fKh2pdG1Vtm:8reyAPWkDC+qV8/UcX+As
[libengine_get_data_string2]  192:x5kTwoH/w1m1SMzRvyOEgRphVdKT+GcLSpl+2i2pb:x2TwoH/w1m1SMzRvyOEgRphVdY0LSi2X
[libengine_get_data_string2]  192:x5kTwoH/w1m1SMzRvyOEgRphVdKT+GcLSpl+2i2pb:x2TwoH/w1m1SMzRvyOEgRphVdY0LSi2X
// ...
[libengine_get_data_string2]  MagiskManager v5.7.0
[libengine_get_data_string2]  24576:TRAT0nEPHDSDC3N1Jd6/HIXaXfOO2SYC05PWIuul:aeDCfJd6/HIXa8PWIvl
[libengine_get_data_string2]  24576:i0yPElnEPHDSV4a5PWbzEDCHWJqV8/UcX+3fKh2pdG1Vtm:8reyAPWkDC+qV8/UcX+As
[libengine_get_data_string2]  192:x5kTwoH/w1m1SMzRvyOEgRphVdKT+GcLSpl+2i2pb:x2TwoH/w1m1SMzRvyOEgRphVdY0LSi2X
[libengine_get_data_string2]  192:a1gSyU9ywbtTgRt7r+k1Sv1eH6spVmT+wPv3WWvptY:a1gS/9ywbtTgRRr+k1Sv1eH6spV0FPvs
// ...

```

1.  又一堆作弊工具

```
[libengine_get_data_string2]  YouXiFengWo v8.0.0
[libengine_get_data_string2]  98304:XXQYQqjBZK5eP/a8eOClHMyj4+/nc/FO1X7H:VJK5K/axTnc/FO9z
[libengine_get_data_string2]  98304:/U5oslClHMmj4+/O4qq8amgdG74GqDpU:M5oHTO4kamgQ4GYU
[libengine_get_data_string2]  1536:T7riB4iWRZdXOyOzdd0mFbOn6RYj/tIbGoH:XbeyOzddW6/bGoH
[libengine_get_data_string2]  1536:T7riB4iWRZdXOyOzdd0mFbOn6RYj/tIbGoH:XbeyOzddW6/bGoH
[libengine_get_data_string2]  LuckyPatcher v4.1.9
[libengine_get_data_string2]  12288:k7glLaM1QMzMuK4PFSGOtB8j1n3A8ldEGt/tt3R3Mt/+8+C+ZtEt/3U33tR+i+PM:0M19DFLOD8j1blb41dXzo0agrJ0Wcug
[libengine_get_data_string2]  24576:agtzGv0XzvsaCEdoupLodt9KIIqCh8SOnSPNcHpeZ0s7:yYdCQpLodtgOnW+Yz
[libengine_get_data_string2]  192:psgSGHUf7u3yFOVrT8dK171SX14gDwFuL/D1+h8CYpm:psgSdf7u3yFOVrT8dK171SX14gDouL/A
[libengine_get_data_string2]  192:5sgSGHUf7u3yFOVrT8dK171SX14gDwFuL/D1NAOtr84mn:5sgSdf7u3yFOVrT8dK171SX14gDouL/Y
[libengine_get_data_string2]  GameHacker v2.6.3
[libengine_get_data_string2]  6144:jV4GcAmz53H1eaMkCtqJr74+/zWZ7ZOR6p9xK5LeEXmkiMxh5UBWF9dCEOCiqop+:jacqNYZ7ZOR6pgLL8GhIWx9yJHXQ
[libengine_get_data_string2]  6144:jV4GcAmz53H1eaMkCtqJr74+/zWZ7ZOR6p9xK5LeEXmkiMxh5UBWF9dCEOCiqop+:jacqNYZ7ZOR6pgLL8GhIWx9yJHXQ
[libengine_get_data_string2]  96:ep3JWcOdtBV7ngScytHD6h+DTypBxRl3stWtEtVr3StC1tVtRHt/5tatWntBZt8s:eZOBhgSt4qTypBxRl3Wr7vhAQWbTij9D
[libengine_get_data_string2]  96:ep3JWcOdtBV7ngScytHD6h+DTypBxRl3stWtEtVr3StC1tVtRHt/5tatWntBZt8s:eZOBhgSt4qTypBxRl3Wr7vhAQWbTij9D
[libengine_get_data_string2]  GameKiller v3.4.3
[libengine_get_data_string2]  98304:swAXGSs/B1OsdbM1Ccc5JiOienK628BjFMihcTgFR:NAXGSs/BYisc5JioDagR
[libengine_get_data_string2]  98304:D1Osd9K628B6eIj4eDhqT/+0BFGnktk7uQ/:DYD9rwi0BFUX7N
[libengine_get_data_string2]  768:zXgSdA13ryngSai6zO0o8co3ukZmtfrQN5Ato25o6kckWkQkBkSckkk/kikrkpk7:cRAkHtbySHvc5IRgoBnbIlbIO
[libengine_get_data_string2]  768:yXgSFA13ryngSai6zO0o8co3ukZmtfrQN5Ato25o6kckWkQkBkSckkk/kikrkpki:RRAkHtbySHvc5IRgoBnbwleUQ
[libengine_get_data_string2]  xxAssistant v2.4.0
[libengine_get_data_string2]  49152:JbZEnH3yH8BBZGVDg3BaCe3JeiQw8zWC0s376e5HO17A0X8OBctF+pSCzbdLyUZC:JKnHAJv8R0s/ROA0iFUvdmUg
[libengine_get_data_string2]  49152:ut4+T3yH8BBZGVDg3BaCwAOBctF+p9m+k6oMNZ+69P+YHigdr0bZSd9MITAMltTH:K4CdF7L6FNZ+KHiw0VstAMltRL1
[libengine_get_data_string2]  768:pDmpgSqyD9mtGvOeSHSbzMNUXZZtPwci2ArFZQzRTBvy57U3RV5FYQwDbo60mSvu:9gMztdc81Se
[libengine_get_data_string2]  768:/DmpgSyyD9mtGvOeSHSbzMNUXZZtPwci2ArFZQzRTBvy57U3RV5FYQwDbo60mSvZ:L4MzDyQH7M
// ...

```

1.  一些模擬器特徵

```
[libengine_get_data_string2]  GenyMotion
[libengine_get_data_string2]  genymotion
[libengine_get_data_string2]  GenyMotion
[libengine_get_data_string2]  genymotion
[libengine_get_data_string2]  Windroy
[libengine_get_data_string2]  windroy
[libengine_get_data_string2]  Windroy
[libengine_get_data_string2]  windroy
[libengine_get_data_string2]  Droid4X
[libengine_get_data_string2]  droid4x
[libengine_get_data_string2]  Droid4X
[libengine_get_data_string2]  droid4x
[libengine_get_data_string2]  TianTian
[libengine_get_data_string2]  TianTian
[libengine_get_data_string2]  TianTian
[libengine_get_data_string2]  TianTian
[libengine_get_data_string2]  CanPlay
[libengine_get_data_string2]  canplay
[libengine_get_data_string2]  CanPlay
[libengine_get_data_string2]  canplay
[libengine_get_data_string2]  VPhoneGaGa
[libengine_get_data_string2]  VPhoneGaGa
[libengine_get_data_string2]  BlueStacks
[libengine_get_data_string2]  com.bluestacks.*
[libengine_get_data_string2]  BlueStacks
[libengine_get_data_string2]  com.bluestacks.*
[libengine_get_data_string2]  MEmu
[libengine_get_data_string2]  com.microvirt.*
[libengine_get_data_string2]  MEmu
[libengine_get_data_string2]  com.microvirt.*
[libengine_get_data_string2]  Andy
[libengine_get_data_string2]  org.greatfruit.andy.*
[libengine_get_data_string2]  Andy
[libengine_get_data_string2]  org.greatfruit.andy.*
[libengine_get_data_string2]  DuOS
[libengine_get_data_string2]  com.ami.syncduosservices
[libengine_get_data_string2]  DuOS
[libengine_get_data_string2]  com.ami.syncduosservices
[libengine_get_data_string2]  TianTian
[libengine_get_data_string2]  com.tiantian.ime
[libengine_get_data_string2]  TianTian
[libengine_get_data_string2]  com.tiantian.ime
[libengine_get_data_string2]  RemixOS
[libengine_get_data_string2]  com.jide.*
[libengine_get_data_string2]  RemixOS
[libengine_get_data_string2]  com.jide.*
[libengine_get_data_string2]  CanPlay
[libengine_get_data_string2]  com.huidong.canplay
[libengine_get_data_string2]  CanPlay
[libengine_get_data_string2]  com.huidong.canplay
[libengine_get_data_string2]  Nox
[libengine_get_data_string2]  /system/bin/nox
[libengine_get_data_string2]  Nox
[libengine_get_data_string2]  /system/bin/nox
[libengine_get_data_string2]  TianTian
[libengine_get_data_string2]  /system/bin/ttVM-prop
[libengine_get_data_string2]  TianTian
[libengine_get_data_string2]  /system/bin/ttVM-prop
[libengine_get_data_string2]  MoMo
[libengine_get_data_string2]  /system/app/MOMOStore/MOMOStore.apk
[libengine_get_data_string2]  MoMo
[libengine_get_data_string2]  /system/app/MOMOStore/MOMOStore.apk

```

5. 結語
=====

看到這的讀者應該可以大致感受到 NP 的強大，自實現 linker、自加載 libc、信號回調的應用、反調試、各種檢測點、混淆，任何一點都值得仔細分析。除了這些，其實還有很多東西沒有分析到，但礙於時間和水平有限，筆者也只能分析到這裡，後續有時間或許可以再看看其他部份。

最後感謝乐佬的那篇文章，時至今日還是很有參考價值，還要感謝 Code 大佬，沒有他的那句提點，或許就沒有這篇文章。

歡迎找我交流安卓手遊安全相關的內容，不論是攻方或防方的 ^^ (dng6IGFzZDI4NzYxMzA5)。

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-173.htm)

[#逆向分析](forum-161-1-118.htm) [#混淆加固](forum-161-1-121.htm)
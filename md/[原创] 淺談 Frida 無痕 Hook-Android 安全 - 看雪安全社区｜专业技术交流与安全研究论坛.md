> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-291266.htm)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

前段時間有大佬開源了一套名為 wxshadow 的無痕 hook 方案，它主要通過頁表的一些機制來實現，詳細實現推薦去看源碼，非常厲害的思路！！( 原文：「[**[原创]linux/android 利用 shadow 内存无痕 hook 方法**](https://bbs.kanxue.com/thread-290304.htm)」 )

wxshadow 是一個提供了無痕 patch 接口的 KPM 模塊，而非一個可以開箱即用的 hook 框架，因此我就在想，能否把它集成到 Frida 中，給 Frida 裝上無痕 Hook 功能？本文簡單聊聊它的可行性。

一些個人認為要先了解的內核知識，來自「《linux 内核深度解析》余华兵」第三章。

頁表用來把虛擬頁映射到物理頁，並且存放頁的保護位。

在 Linux4.11 版本之前，Linux 內核把頁表分為四級：

1.  頁全局目錄 (Page Global Directory，PGD)
2.  頁上層目錄 (Page Upper Directory，PUD)
3.  頁中間目錄 (Page Middle Directory，PMD)
4.  直接頁表 (Page Table，PT ，表中的元素稱為 Page Table Entry，簡稱 PTE)

4.11 版本把頁表擴展到五級，在 PGD 和 PUD 之間增加了頁四級目錄 (Page 4th Directory，P4D)。

可以通過`CONFIG_PGTABLE_LEVELS`宏來配置頁表的級數，以我的 P6 為例，是 3 級頁表。

```
oriole:/ 
CONFIG_PGTABLE_LEVELS=3


```

每個進程有獨立的頁表，進程的`mm_struct`實例成員`pgd`指向全局目錄，前面四級頁表的表項存放下一級頁表的起始地址，直接頁表的表項存放頁幀號 (Page Frame Number，PFN)。

頁幀號 + 頁內編移 = 物理地址

![](https://bbs.kanxue.com/upload/attach/202605/946537_UMY4AGVWMW6EXP3.webp)

ARM64 把頁表稱為轉換表 (translation table)，最多 4 級。

1.  頁長度是 4KB：使用`4`級轉換表，`0`級轉換表對應頁全局目錄，`1`級轉換表對應頁上層目錄，如此類推 ( 沒有`P4D` )。
    
    48 位虛擬地址被分解為如下所示：
    
    ![](https://bbs.kanxue.com/upload/attach/202605/946537_HCUFR8EY73CP7WK.webp)
    
2.  頁長度是 64KB：使用`3`級頁表，`1`級轉換表對應頁全局目錄，`2`級對應頁中間目錄，`3`級對應直接頁表。
    
    ![](https://bbs.kanxue.com/upload/attach/202605/946537_8A3QX5H8UHR4A4V.webp)
    

ARM64 把表項稱為描述符 (descriptor)，長度為`64`位。描述符的第`0`位代表當前描述符是否有效，`0`表示無效，`1`表示有效。第`1`位指定描述符的類型，具體如下：

*   在`0 ~ 2`級轉換表中，`0`表示塊 (block) 描述符，存放一個內存塊 ( 即巨型頁 ) 的起始地址，`1`表示表 (table) 描述符，存放下一級轉換表的地址。
*   在第`3`級轉換表中，`0`表示保留，`1`表示頁描述符。

`3`級轉換表的頁描述符如下圖所示：

![](https://bbs.kanxue.com/upload/attach/202605/946537_VXC72NNDKPNE2D9.webp)

在塊描述符和頁描述符中，內存屬性被拆分成一個高屬性塊和一個低屬性塊，如下圖所示：

![](https://bbs.kanxue.com/upload/attach/202605/946537_9YQGRB5UHDTVM72.webp)

wxshadow 利用了其中幾個關鍵屬性來實現：

*   第`54`位：在 EL0 中表示 UXN(Unprivileged execute-Never)，即不允許 EL0 執行內核代碼；在其他異常級別，表示 XN( execute-Never )，不允許執行。
*   第`6 ~ 7`位：`AP[2:1]`(Data Access Permissions，數據訪問權限)。在階段 1 轉換中，`AP[2]`用來選擇只讀或讀寫，`1`表示只讀，`0`表示讀寫；`AP[1]`用來選擇是否允許 EL0 訪問，`1`表示允許，`0`表示不允許。在非異常級別`1`和`0`轉換機制的階段 2 轉換中，`AP[2:1]`為`00`表示不允許訪問，`01`表示只讀，`10`表示只寫，`11`表示讀寫。

當運行內存需求量較大的應用程序時，如果使用長度為 4KB 的頁，將會產生較多的 TLB 未命中和缺頁異常，大大影響應用程序的性能。而如果使用長度為 2MB，什至更大的巨型頁，可以大幅改善此問題，這正是內核引入巨型頁 (Huge Page) 的直接原因。

ARM64 支持巨型頁的方式有兩種：

1.  通過塊描述符來支持
    
    ![](https://bbs.kanxue.com/upload/attach/202605/946537_CAUW84AFQTHT4F5.webp)
    
2.  通過頁 / 塊描述符的連續位來支持
    
    ![](https://bbs.kanxue.com/upload/attach/202605/946537_USD2URZ7MRRYW5X.webp)
    

wxshadow 提供了`PR_WXSHADOW_PATCH`和`PR_WXSHADOW_RELEASE`，前者可以用於替代 frida 的 patch 行為，後者用於清理現場。

```
case PR_WXSHADOW_PATCH:
    pid = (pid_t)arg2;
    mm = resolve_pid_to_mm(pid);
    if (!mm) { args->ret = -3; args->skip_origin = 1; break; }
    ret = wxshadow_do_patch(mm, arg3, (void __user *)arg4, arg5);
    kfunc_mmput(mm);
    args->ret = ret;
    args->skip_origin = 1;
    break;

case PR_WXSHADOW_RELEASE:
    pid = (pid_t)arg2;
    mm = resolve_pid_to_mm(pid);
    if (!mm) { args->ret = -3; args->skip_origin = 1; break; }
    if (arg3 == 0) {
        ret = wxshadow_release_pages_for_mm(mm, "release_all");
    } else {
        ret = wxshadow_do_release(mm, arg3);
    }
    kfunc_mmput(mm);
    args->ret = ret;
    args->skip_origin = 1;
    break;


```

實際嘗試時，遇到了兩個問題。

**問題一：會發生 livelock 的情況。**

原因是 Frida hook 會把目標地址 patch 成如下形式，用到了 arm64 的 ldr literal 技術 (代碼 / 數據在相鄰位置)。

```
ldr x16, [pc, #8]
br  x16
.dword on_enter_trampoline


```

若`PR_WXSHADOW_PATCH`時仍然是按這種形式來 patch，會導致 livelock (APP 卡死)

因為 wxshadow 本質上維護了`original(r--)`(讀) 和`wxshadow(--x)`(執行) 兩種視圖，當 wxshadow 直接套用到上述 patch 中，就會發生以下情況：

```
; 當前視圖: wxshadow(--x)
exec: ldr x16, [pc, #8]
; 觸發read fault, 切換到original(r--)
read: on_enter_trampoline
; 讀完後, 回到ldr指令, 觸發exec fault, 切換到wxshadow(--x)
exec: ldr x16, [pc, #8]

; 死循環...


```

而 wxshadow 的作用範圍是一整頁，因此對同頁下的 ldr literal 也要做處理。

**問題二：wxshadow 無法對`libart.so`大部份函數進行 hook。**

這與`libart.so`在某次更新時改用巨型頁 (Huge PMD) 有關，而 wxshadow 是以 PTE 為單位進行管理，雖然其中有對巨型頁做處理，但測試下來發現是不夠的。

```
Compile libart.so and libart-compiler.so with 2MB section alignment.

Adds the appropriate linker flags for libart and libart-compiler to have
2MB section alignment. This allows the executable segment of these
libraries to be backed by transparent hugepages on supporting systems.


```

解決思路是手動把 2MB 的 PMD 分割成 512 個 4KB 的 PTE，實測可行。

基於 wxshadow 提供能力，能很好地隱藏 Frida 本身的一些特徵：

1.  默認 hook 行為：以 Frida16.5.9 為例，默認會 hook `libc.so`、`libselinux.so`和`libandroid_runtime.so`，用到 Java 相關 API 時，會 hook `libart.so`。
    
    注：之後的某個版本開始還會默認 hook `linker`
    
2.  maps 特徵：啟動一次 frida-server 後，maps 特徵 ( rwxp 段、N 段的`libc.so`等 ) 會永久留下，即使關掉 frida-server 也無用，必須要重開機才行。
    
    原因是 frida-server 的那些默認 hook 行為是對 zygote 進程做的，而 zygote 進程是所有 APP 進程的父進程。
    

除此之外，Frida 還有 memfd、注入線程等特徵，可以在內核層做繞過。

總結：Frida 集成 wxshadow 是完全可行的，遇到的各種問題基本都有解法。

兼容性和穩定性如何？用一些樣本來測測看。

以前分析過這個保護，那時候還不知道它的名字叫`CrackProof`，個人認為是個很強的保護 ( 分析文章：[https://bbs.kanxue.com/thread-286746.htm](https://bbs.kanxue.com/thread-286746.htm) )

遊戲邏輯在`libpad.so`裡，之前分析的時候這個 so 似乎還沒有加密，現在終於加密了，但也是直接 dump 就行的整體加密。

嘗試實現秒殺功能，對於這種非常規遊戲引擎開發的遊戲，讓 AI 來找簡直再合適不過：

```
使用ida-pro-mcp, 分析"玩家攻擊" 相關的邏輯在哪?


```

然後用魔改的 Frida 對 AI 給出的地址進行 hook，成功實現秒殺。

![](https://bbs.kanxue.com/upload/attach/202605/946537_5V5XG8PFVWQ6GCH.webp)

運行得也挺穩定，未曾出現過閃退的問題。

之前也分析過，這次用另一個樣本來測 ( [https://bbs.kanxue.com/thread-288477.htm](https://bbs.kanxue.com/thread-288477.htm) )

記得 NP 裡是有自定義 linker 保護的，沒想到直接 dump `libil2cpp.so`也行。把 gm 文件也 dump 出來後，直接用 Il2cppDumper 就能把 SDK 給 dump 出來。

同樣嘗試實現秒殺功能，同樣交給 AI 去分析，結合 ida 和 dump.cs，也是很簡單就找到對應的邏輯。

![](https://bbs.kanxue.com/upload/attach/202605/946537_5PZFPJUD9E2D57H.webp)

魔改 Frida spawn 這個 APP 時，有機會 spawn 失敗 (Frida 閃退 / APP 卡住，並非被檢測到)，但機率比較低，屬於可接受範圍之內。

小結：除此之外還測試了幾個 Appdome 保護的樣本，雖然都有機會發生上述 spawn 失敗的情況，但基本不影響使用，說明 wxshadow 已經是個非常成熟的方案。

(如果有其他檢測強的樣本，歡迎分享給我看看 ^^)

一開始選用了`17.9.3`這個版本來進行魔改，改到後面發現會莫名其妙地崩潰，一度讓我以為是改崩了，但後來用原版的`17.9.3`去測試也一樣崩，說明是這個版本有問題？？？最後還是用回了`16.5.9`這個版本，以前一直用的這個版本，個人認為是比較穩定的一個版本。

對 Frida 魔改集成 wxshadow 的所有工作，可以完全交給 AI 來完成。在 AI 的幫助下，試錯成本低了很多，只要有思路，AI 總能給你一個滿意的答覆，或許時代真的變了吧。

( 最近在看工作機會，有大佬公司缺人的可以私聊 / vx：`ngiokweng` )

[[培训]《冰与火的战歌：Windows 内核攻防实战》！从零到实战，融合 AI 与 Windows 内核攻防全技术栈，打造具备自动化能力的内核开发高手。](https://www.kanxue.com/book-section_list-227.htm)
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-289567.htm)

> [原创] 一种基于 ART 内存特征的 LSPosed/Xposed / 分身环境 完美检测方案

[原创] 一种基于 ART 内存特征的 LSPosed/Xposed / 分身环境 完美检测方案

发表于: 3 小时前 155

[举报](javascript:void(0);)

### [原创] 一种基于 ART 内存特征的 LSPosed/Xposed / 分身环境 完美检测方案

 [![](http://passport.kanxue.com/upload/avatar/880/735880.png?1499251741)](user-home-735880.htm) [世界美景](user-home-735880.htm) ![](https://bbs.kanxue.com/view/img/rank/9.png) 1  ![](http://passport.kanxue.com/pc/view/img/sun.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) [ 举报](javascript:void(0);) 3 小时前  155

前言：猫鼠游戏的困局
----------

在 Android 反作弊（Anti-Cheat）的战场上，检测 Xposed 类框架（LSPosed, EdXposed 等）一直是最核心的对抗环节。

传统的检测手段通常依赖于：

1.  **文件检测**：扫描 /data/app 下的异常 APK 或 /proc/self/maps 中的异常 SO。
    
2.  **符号检测**：尝试 dlopen 或 dlsym 寻找框架特有的导出函数。
    
3.  **堆栈回溯**：[在 Java /JNI 层制造异常](https://bbs.kanxue.com/thread-289531.htm)，检测堆栈中是否有 LSPHooker 或 XposedBridge。
    

然而，随着 **Shamiko** 等隐藏模块的出现，以及 **Magisk/KernelSU** 带来的内核级隐藏能力，上述手段正变得越来越无力。攻击者可以 Hook open、read、dlopen 甚至系统调用，给反作弊 SDK 返回一份 “完美” 的虚假数据。

**我们是否能跳出 API 调用的维度，直接从虚拟机内存的物理本质上抓出作弊框架？**

答案是肯定的。本文将分享一种 **Tier 0 级别** 的检测方案：**基于 ART 内存布局特征的 ClassLoader 计数检测**。

核心原理：ART 的软肋与死局

### 1. ClassLinker：虚拟机的 “户籍办”

在 Android 的 ART 虚拟机中，Runtime 结构体持有一个核心组件——**ClassLinker**。它的职责是管理所有的类加载器（ClassLoader）和类。

在 ClassLinker 的 C++ 对象内部，维护了一个链表：**class_loaders_**。  
这是一个 std::list，记录了当前进程中所有 **存活** 的 ClassLoader。

### 2. LSPosed 的 “生存悖论” (The GC Paradox)

LSPosed 要实现模块注入，必须创建自己的 PathClassLoader 或 DexClassLoader 来加载模块代码。

这里存在一个**无法解决的死局**：

*   **为了生存**：LSPosed 创建的 ClassLoader 必须被注册到 ClassLinker 的 class_loaders_ 链表中。如果它试图将自己从链表中移除（隐藏），ART 的垃圾回收机制（GC）会认为该 ClassLoader 不可达，进而将其回收。**一旦回收，模块代码被卸载，Hook 瞬间失效，甚至导致 App 崩溃。**
    
*   **为了隐藏**：它必须从链表中消失。
    

**结论**：LSPosed **不得不** 赖在这个链表里。只要它在，我们就能抓到它。

技术实现：内存盲扫 (Blind Memory Scanning)
---------------------------------

为了绕过所有的 Hook（包括 PLT Hook, Inline Hook, Syscall Hook），本方案**不调用任何系统 API**（如 GetClassLinker），而是直接进行 **C++ 内存指针运算**。

### Step 1: 寻找 Runtime (The Entry)

通过标准的 JNI 接口获取 JavaVM，进而拿到 Runtime 指针。这是极其稳定的，几乎所有 Android 版本通用。

```
JavaVM* vm = nullptr;
env->GetJavaVM(&vm);
void* runtime = *((void**)((uintptr_t)vm + sizeof(void*)));

```

### tep 2: 全动态特征扫描 (The Scanner)

由于不同 Android 版本和厂商 ROM 的 Runtime 结构体布局不同，硬编码偏移量（Offset）是不可靠的。我们采用**运行时特征扫描**：

**特征 A：VTable 校验**  
ClassLinker 是一个 C++ 对象，其首地址一定是虚函数表（VTable）指针。该 VTable 地址必然位于 libart.so 的只读数据段（.rodata）内。

**特征 B：双向循环链表**  
class_loaders_ 是 std::list，其底层是双向循环链表。必然满足以下指针关系：

```
node->next->prev == node
node->prev->next == node

```

**特征 C：数量合理性**  
正常的 App 启动后，至少包含 BootClassLoader 和 PathClassLoader。因此，链表节点数必然 >= 2。

结合上述特征，我们在 Runtime 内存范围内进行暴力搜索：

```
// 伪代码演示
for (int offset = 0; offset < 0x500; offset += PTR_SIZE) {
    void* candidate = *(void**)(runtime + offset);
    if (IsVTableValid(candidate)) { // 特征A
        if (HasValidList(candidate)) { // 特征B & C
            // 锁定 ClassLinker 和 List 的偏移！
            g_ClassLinkerOffset = offset;
            break;
        }
    }
}

```

### Step 3: 计数判定 (The Verdict)

一旦锁定链表位置，直接遍历并计数。

*   **纯净环境**：通常只有 **2-3** 个 ClassLoader（Boot + App + WebView）。
    
*   **注入环境**：LSPosed 会为框架自身、每个模块、以及沙箱环境创建额外的 ClassLoader。
    
*   在实际测试中，LSPosed 环境下的 ClassLoader 数量通常高达 **13-15 个，分身环境实测只会比正常环境 + 1**。
    

**判定逻辑**：Count > 10 即视为异常。

检测代码 (C++):

```
//通过内存暴力搜索，找到 Runtime 对象中的 ClassLinker 指针，再进一步定位 class_loaders_ 链表。
namespace anti_ClassLinker {
 
    // 1. 基础结构与工具
    // =============================================================
 
    // 双向链表节点 (64位)
    struct ListNode {
        ListNode* next;
        ListNode* prev;
    };
 
    // 全局静态变量：缓存扫描到的偏移量
    // 初始化为 -1，表示尚未扫描
    static int g_ClassLinkerOffset = -1;
    static int g_ListOffset = -1;
 
    // 内存安全检查：防止读取非法地址导致 SIGSEGV
    static bool IsAddressReadable(void* addr) {
        if (!addr) return false;
        unsigned char vec = 0;
        size_t page_size = getpagesize();
        // 对齐到页边界
        uintptr_t align_addr = (uintptr_t)addr & ~(page_size - 1);
        // mincore 检查该页是否在物理内存中
        return mincore((void*)align_addr, page_size, &vec) == 0 && (vec & 1);
    }
 
    // 解析 /proc/self/maps 获取 libart.so 的内存范围
    // 用于校验 VTable 是否合法
    static bool GetArtMemoryRange(uintptr_t* start, uintptr_t* end) {
        FILE* fp = fopen("/proc/self/maps", "r");
        if (!fp) return false;
        char line[512];
        bool found = false;
        while (fgets(line, sizeof(line), fp)) {
            if (strstr(line, "/libart.so")) { // 匹配 libart.so 路径
                unsigned long s, e;
                if (sscanf(line, "%lx-%lx", &s, &e) == 2) {
                    if (!found) *start = s; // 记录起始地址
                    *end = e; // 不断更新结束地址，直到最后一段
                    found = true;
                }
            }
        }
        fclose(fp);
        return found;
    }
 
    // =============================================================
    // 2. 核心扫描逻辑 (只在初始化时运行一次)
    // =============================================================
 
    // 在 Runtime 内存中暴力搜索 ClassLinker 和 class_loaders_ 链表
    static void ScanOffsets(void* runtime) {
        uintptr_t art_start = 0, art_end = 0;
        if (!GetArtMemoryRange(&art_start, &art_end)) {
            LOGE("[-] 无法获取 libart.so 内存映射");
            return;
        }
 
        LOGD(" [初始化] 开始全动态扫描 Art 内存特征...");
        uintptr_t runtime_addr = (uintptr_t)runtime;
 
        // --- 外层循环：扫描 Runtime 成员，寻找疑似 ClassLinker ---
        // 范围：0 ~ 0x500 (通常在 0x200-0x350 之间)
        for (int cl_off = 0; cl_off < 0x500; cl_off += 8) {
            void** ptr_candidate = (void**)(runtime_addr + cl_off);
            if (!IsAddressReadable(ptr_candidate)) continue;
 
            void* potential_obj = *ptr_candidate;
            if (!IsAddressReadable(potential_obj)) continue;
 
            // [特征1] 验证 VTable
            // C++ 对象的头 8 字节是指向 VTable 的指针
            void** vptr = (void**)potential_obj;
            if (!IsAddressReadable(vptr)) continue;
            uintptr_t vtable = (uintptr_t)*vptr;
 
            // VTable 地址必须落在 libart.so 的内存区间内
            if (vtable < art_start || vtable > art_end) continue;
 
            // --- 内层循环：在疑似对象中寻找 class_loaders_ 链表 ---
            // 范围：0 ~ 0x500 (通常在 0x50-0x300 之间)
            uintptr_t obj_addr = (uintptr_t)potential_obj;
            for (int list_off = 0; list_off < 0x500; list_off += 8) {
                ListNode* head = (ListNode*)(obj_addr + list_off);
                if (!IsAddressReadable(head)) continue;
 
                ListNode* next = head->next;
                ListNode* prev = head->prev;
 
                if (!IsAddressReadable(next) || !IsAddressReadable(prev)) continue;
 
                // [特征2] 双向链表闭环检测
                // head->next->prev == head  且  head->prev->next == head
                if (next->prev == head && prev->next == head) {
 
                    // 排除空链表 (next == head)，因为 class_loaders_ 必不为空
                    if (next == head) continue;
 
                    // [特征3] 节点数量验证
                    // 正常的 App 至少有 2 个 Loader (Boot + Path)
                    int count = 0;
                    ListNode* curr = next;
                    bool is_valid_list = true;
 
                    // 遍历计数，同时防止死循环
                    while (curr != head) {
                        count++;
                        if (count > 200) { is_valid_list = false; break; }
                        if (!IsAddressReadable(curr) || !IsAddressReadable(curr->next)) {
                            is_valid_list = false; break;
                        }
                        curr = curr->next;
                    }
 
                    if (is_valid_list && count >= 2) {
                        //  完美匹配！同时满足 VTable 合法 + 链表结构合法 + 数量合理
                        g_ClassLinkerOffset = cl_off;
                        g_ListOffset = list_off;
 
                        LOGD("✅ [锁定] 动态偏移计算完成!");
                        LOGD("    -> ClassLinker Offset: 0x%x", g_ClassLinkerOffset);
                        LOGD("    -> List Offset: 0x%x", g_ListOffset);
                        LOGD("    -> 当前 Loader 数量: %d", count);
                        return; // 找到即停止
                    }
                }
            }
        }
        LOGE("[-] 扫描失败，未找到符合特征的结构");
    }
 
    // =============================================================
    // 3. 对外接口：获取 ClassLoader 数量
    // =============================================================
 
    int getClassLoaderCount(JNIEnv* env) {
 
        // 1. 获取 Runtime 实例 (JavaVM + sizeof(void*))
        JavaVM* vm = nullptr;
        if (env->GetJavaVM(&vm) != JNI_OK || !vm) return -1;
        void* runtime = *((void**)((uintptr_t)vm + sizeof(void*)));
        if (!IsAddressReadable(runtime)) return -1;
 
        // 2. 如果偏移未初始化，执行一次扫描
        if (g_ClassLinkerOffset == -1 || g_ListOffset == -1) {
            ScanOffsets(runtime);
            // 如果扫完还是 -1，说明失败
            if (g_ClassLinkerOffset == -1) return -1;
        }
 
        // 3. 极速读取模式 (直接利用偏移)
        void* class_linker = *(void**)((uintptr_t)runtime + g_ClassLinkerOffset);
        if (!IsAddressReadable(class_linker)) return -1;
 
        ListNode* head = (ListNode*)((uintptr_t)class_linker + g_ListOffset);
        if (!IsAddressReadable(head)) return -1;
 
        // 4. 遍历链表
        // 再次校验链表完整性，防止运行时结构变化
        if (!IsAddressReadable(head->next) || head->next->prev != head) {
            LOGE("[-] 链表结构在运行时损坏，重置偏移");
            g_ClassLinkerOffset = -1; // 触发下次重新扫描
            return -1;
        }
 
        int count = 0;
        ListNode* curr = head->next;
        while (curr != head) {
            count++;
            if (count > 500) break; // 防死循环
 
            curr = curr->next;
        }
 
        return count;
    }
 
 
}
 
 
 
//调用部分
int count = anti_ClassLinker::getClassLoaderCount(env);
 
if (count > 0) {
    // 简单的判定逻辑打印
    if (count > 10) {
        LOGE("???????????? 异常! 发现 %d 个 ClassLoader (正常值 5)", count);
    } else {
        LOGD("✅ 正常. 发现 %d 个 ClassLoader", count);
    }
} else {
    LOGE("[-] 获取失败");
}
 
return count;

```

* * *

稳定性评估与风险控制：在 “暴力” 中寻找平衡
-----------------------

必须承认，**内存盲扫（Memory Scanning）** 即使在 PC 端反作弊中也属于激进（Aggressive）手段，在碎片化极度严重的 Android 生态中更是如此。虽然我在理论层面构建了多重防护，但面对魔改的 ROM 和千奇百怪的设备，**我保持极度谨慎的态度，反正我的 SDK 目前不敢上线使用哈哈哈**。

### 1. 理论层面的 “三道护盾”

为了将 Crash 风险降至最低，我在代码实现上极其克制：

*   **系统级护盾 (**：这是最核心的安全机制。在对任何指针进行解引用（Dereference）之前，强制调用 mincore 系统检测该内存页是否映射在物理内存中。这从根本上阻断了 99% 因访问野指针或非法地址导致的 SIGSEGV 崩溃。
    
*   **零侵入（Read-Only）**：全程仅进行 “读取” 操作，绝不尝试写入或修改任何内存数据，确保不会破坏 ART 虚拟机的内部状态。
    
*   **去符号化**：完全移除对 xdl、dlsym 或私有系统库的依赖，规避了 Android 7.0+ 命名空间隔离带来的兼容性崩坏，也减少了因系统库版本差异导致的符号查找失败。
    

### 2. 现实世界的挑战（Risk Warning）

尽管有上述防护，但 **“全量上线” 仍需三思**。由于我们采用了暴力枚举（从 Runtime 指针偏移 0 扫到 0x500）的方式，以下风险客观存在：

*   **OEM 厂商魔改**：部分深度定制的 ROM（如某些游戏手机或车机系统）可能大幅修改了 Runtime 或 ClassLinker 的内存布局，导致特征扫描误判，虽然不会崩，但可能导致**检测失效**（返回 -1）。
    
*   **并发竞争（Race Condition）**：在遍历链表时，尽管有 mincore 保护，但理论上存在极低概率的 “Time-of-Check to Time-of-Use” 风险，即在检测可读和实际读取的微小时间窗内，内存页被系统回收（虽然在主线程或加锁环境下极少发生）。
    
*   **性能抖动**：在 App 启动瞬间进行内存扫描，虽然耗时通常在毫秒级，但在某些低端机型上可能会产生极其微弱的 CPU 峰值。
    

*     
    

* * *

为什么 Shamiko 无法隐藏？
-----------------

Shamiko 的隐藏原理主要是 Hook 系统查询接口（如隐藏文件、隐藏 Maps 条目）。

但是，本方案**直接读取的是堆内存中的 C++ 对象**。

*   攻击者无法 Hook CPU 的内存加载指令（LDR）。
    
*   攻击者无法在不破坏 GC 的前提下修改 ART 内部链表结构。
    
*   **除非攻击者针对本 App 的检测函数进行专门的逆向和 Inline Hook**（成本极高），否则在通用隐藏层面，这是一个**无解的死局**。
    

总结
--

反作弊与作弊的对抗螺旋上升。当 API 层的 Hook 已经泛滥时，下沉到虚拟机内存布局层面进行 **“降维打击”**，往往能收到奇效。

通过**动态计算偏移 + 内存特征校验**，我们实现了一个无需权限、无需符号、难以隐藏的通用检测方案。只要 ART 虚拟机还是用 C++ 写的，只要 GC 机制还在运行，这套逻辑就依然有效。

(注：本文仅供安全研究与技术交流，请勿用于恶意用途)

  

[传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

最后于 2 小时前 被世界美景编辑 ，原因： 格式乱了 删除重复内容 [#混淆加固](forum-161-1-121.htm) [#HOOK 注入](forum-161-1-125.htm) [#其他](forum-161-1-129.htm)
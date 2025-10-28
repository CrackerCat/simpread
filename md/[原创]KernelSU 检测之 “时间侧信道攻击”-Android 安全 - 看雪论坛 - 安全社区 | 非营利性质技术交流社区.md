> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-288928.htm)

> [原创]KernelSU 检测之 “时间侧信道攻击”

KernelSU 检测之 “时间侧信道攻击”
======================

背景
--

​ 偶然间拿到一个海外 Bank 的样本, 打开后出现崩溃的现象，但是我的手机只刷了 KernelSU，并且还魔改过的，以前一直都是过五关斩六将，但是这个 app 为什么会崩溃呢，难道说是检测到我的环境装了 KernelSU？带着这个疑问对这个样本进行了一系列惨无人道的摧残，发现确实检测了 KernelSU, 并且不是普通的检测方式，而是鲜少听闻的” 侧信道攻击 “检测。

在开始之前说下什么是 “侧信道攻击”。

什么是 “侧信道攻击”？
------------

### 核心概念：不正面强攻，而是 “旁敲侧击”

想象一下，你想知道一个保险箱的密码，但你并不去尝试破解它的机械结构或电子锁。

*   **传统攻击（正面攻击）**：你试图用听诊器听密码锁的声音，或者用炸药强行炸开。
*   **侧信道攻击（旁敲侧击）**：你不直接碰锁，而是去：
    *   **测量**：转动密码盘时，保险箱发出的微弱声音或振动。
    *   **观察**：主人输入密码时，他手指的运动轨迹和按键上的指纹磨损。
    *   **计时**：记录主人从开始输入到打开箱门所花费的总时间。
    *   **分析功耗**：如果这是一个电子锁，你测量它在尝试不同密码时的微小电量消耗差异。

通过分析这些**间接信息**，你很有可能推断出正确的密码。这种通过收集和分析系统在运行过程中产生的**物理间接信息**，而不是直接攻击算法或代码本身，来获取秘密信息的攻击方式，就叫做**侧信道攻击**。

### 常见侧信道攻击类型一览表

<table><thead><tr><th>攻击类型</th><th>攻击原理</th><th>典型例子</th><th>关键特点</th></tr></thead><tbody><tr><td><strong>计时攻击</strong></td><td>测量操作执行时间，通过时间差异推断数据。例如，字符串比较时，早期返回错误会导致更短的执行时间。</td><td>1. 分析网站登录验证，猜测用户名和密码。 2. 通过测量 RSA 解密操作的时间来推测私钥。</td><td>纯软件攻击，无需物理接触。依赖代码逻辑不是 “恒定时间” 的。</td></tr><tr><td><strong>功耗分析攻击</strong></td><td>测量设备运行时芯片的功耗波动。处理不同数据（0 或 1）时功耗有微小差异，分析这些 “功耗轨迹” 可还原密钥。</td><td>1. 攻击智能卡、加密狗。 2. 攻击 POS 机，窃取信用卡 PIN 码。</td><td>需要物理接近和精密仪器（如示波器）。分为简单功耗分析（SPA）和差分功耗分析（DPA）。</td></tr><tr><td><strong>电磁分析攻击</strong></td><td>测量设备运行时散发的电磁辐射。与功耗分析类似，电磁辐射也携带着与当前操作和数据相关的信息。</td><td>1. 通过探测手机 CPU 的电磁辐射，还原其正在运行的加密算法和密钥。 2. 攻击接触式 / 非接触式智能卡。</td><td>是一种 “非侵入式” 攻击，可以在一定距离外进行。信息量可能比功耗更丰富。</td></tr><tr><td><strong>声音攻击</strong></td><td>录制设备运行时的声音，分析其机械部件的声学特征，来还原处理的信息。</td><td>1. 通过点阵打印机的声音还原打印内容。 2. 通过键盘敲击声识别输入的按键。 3. 通过硬盘寻道声推测读取的数据。</td><td>利用的是机械振动产生的声音，对麦克风灵敏度要求高。</td></tr><tr><td><strong>缓存攻击</strong></td><td>利用 CPU 缓存的访问时间差异（缓存命中快，未命中慢）。通过监控自身程序的访问速度，来探测受害者进程的缓存使用情况，从而泄露信息。</td><td>1. <strong>Spectre（幽灵）</strong> 和 <strong>Meltdown（熔断）</strong> 漏洞就是利用缓存侧信道跨进程窃取内核内存数据。 2. 在云环境中，攻击同一物理主机的其他虚拟机。</td><td>是现代 CPU 微架构中最重要的侧信道之一，威胁云安全。</td></tr><tr><td><strong>光学攻击</strong></td><td>使用高分辨率相机等设备，捕捉设备运行时泄露的光学信息。</td><td></td><td></td></tr></tbody></table>

分析
--

样本 so 用了 AppGuard 加固, 且应用自身的 so 被加密了, 节约时间就直接步入正题。

检测方式用的是 “时间侧信道攻击”

### 检测代码

这是 ksu 的检测代码，接下来对其进行拆解

```
__int64 __fastcall ksu_check(void *a1)
{
  __int64 v1; // x0
  __int64 *v2; // x22
  __int64 v3; // x19
  __int64 v4; // x0
  void **v5; // x21
  __int64 v6; // x20
  __int64 v7; // x0
  __int64 i; // x20
  __int64 v9; // x19
  __int64 v10; // x0
  __int64 j; // x20
  __int64 v12; // x19
  __int64 (__fastcall *v13)(const void *, const void *); // x19
  __int64 v14; // x20
  void *v15; // x0
  int32x2_t v16; // d0
  int64x2_t v17; // q1
  uint64x2_t *v18; // x8
  int64x2_t *v19; // x9
  int32x2_t v20; // d2
  uint64x2_t v21; // q3
  uint64x2_t v22; // q4
  int32x2_t v23; // d0
  unsigned __int32 v24; // s0
 
  LODWORD(off_132FC0) = sub_67AF8(a1);
  if ( off_132FC0 >= 2 )
  {
    sub_67E1C();
    if ( off_132FD8 )
      sub_67F8C();
  }
  v1 = sub_539B0(80000LL);
  v2 = d1_ptr;
  v3 = v1;
  *d1_ptr = v1;
  v4 = sub_539B0(80000LL);
  v5 = d2_ptr[0];
  v6 = v4;
  *d2_ptr[0] = v4;
  sub_54EB0(v3, 0LL, 80000LL);
  v7 = sub_54EB0(v6, 0LL, 80000LL);
  for ( i = 0LL; i != 80000; i += 8LL )
  {
    v9 = sub_67AD8(v7);
    v10 = sub_54E70(48LL, 0xFFFFFFFFLL, 0LL, 0xFFFFFFFFLL, 0LL);
    v7 = sub_67AD8(v10);
    *(*v2 + i) = v7 - v9;
  }
  for ( j = 0LL; j != 80000; j += 8LL )
  {
    v12 = sub_67AD8(v7);
    v7 = sub_67AD8(linux_eabi_syscall(__NR_fchownat, -1, 0LL, 0, 0, -1));
    *(*v5 + j) = v7 - v12;
  }
  v13 = compare;
  v14 = 10000LL;
  sub_557B0(*v2, 10000LL, 8LL, compare);
  sub_557B0(*v5, 10000LL, 8LL, v13);
  v15 = *v2;
  v16.n64_u64[0] = 0LL;
  v17 = vdupq_n_s64(1uLL);
  v18 = (*v2 + 16);
  v19 = (*v5 + 16);
  v20.n64_u64[0] = 0LL;
  do
  {
    v21 = v18[-1];
    v22 = *v18;
    v18 += 2;
    v14 -= 4LL;
    v16.n64_u64[0] = vsub_s32(v16, vmovn_s64(vcgtq_u64(v21, vaddq_s64(v19[-1], v17)))).n64_u64[0];
    v20.n64_u64[0] = vsub_s32(v20, vmovn_s64(vcgtq_u64(v22, vaddq_s64(*v19, v17)))).n64_u64[0];
    v19 += 2;
  }
  while ( v14 );
  v23.n64_u64[0] = vadd_s32(v20, v16).n64_u64[0];
  v24 = vadd_s32(v23, vdup_lane_s32(v23, 1)).n64_u32[0];
  if ( v24 > 0x1B58 )
  {
    sub_681AC(v24);
    v15 = *v2;
  }
  operator delete(v15);
  operator delete(*v5);
  return 0LL;
}

```

### 拆解分析

#### 阶段 1：环境初始化

```
LODWORD(off_132FC0) = sub_67AF8(a1); // 获取 CPU 核心数
if ( off_132FC0 >= 2 )
{
  sub_67E1C();  // 分析 CPU 性能
  if ( off_132FD8 )
    sub_67F8C();// 绑定到高性能核心
}

```

作用：

*   获取核心数，评估性能并识别大核
    
*   绑定到高性能核心以稳定测量
    

#### 阶段 2：内存分配

```
v1 = sub_539B0(80000LL);//sub_539B0-->malloc
v2 = d1_ptr;
v3 = v1;
*d1_ptr = v1;
v4 = sub_539B0(80000LL);
v5 = d2_ptr[0];
v6 = v4;
*d2_ptr[0] = v4;
sub_54EB0(v3, 0LL, 80000LL);//sub_54EB0-->memset
v7 = sub_54EB0(v6, 0LL, 80000LL);

```

作用：为时间数据分配并清零内存

#### 阶段 3：收集时间

这里会收集两个函数的执行耗时

```
for ( i = 0LL; i != 80000; i += 8LL )
{
  v9 = sub_67AD8(v7);//开始时间
  v10 = sub_54E70(48LL, 0xFFFFFFFFLL, 0LL, 0xFFFFFFFFLL, 0LL);//sub_54E70-->syscall
  v7 = sub_67AD8(v10);//结束时间
  *(*v2 + i) = v7 - v9;
}
for ( j = 0LL; j != 80000; j += 8LL )
{
  v12 = sub_67AD8(v7);//开始时间
  v7 = sub_67AD8(linux_eabi_syscall(__NR_fchownat, -1, 0LL, 0, 0, -1));
  *(*v5 + j) = v7 - v12;//结束时间
}

```

```
unsigned __int64 sub_67AD8()//提供纳秒级计数器值
{
  unsigned __int64 result; // x0
 
  __isb(0xFu);// 指令同步屏障
  result = _ReadStatusReg(CNTVCT_EL0);// 读取虚拟计数器
  __isb(0xFu);// 指令同步屏障
  return result;
}

```

分别收集__NR_faccessat 和__NR_fchownat 的执行时间

#### 阶段 4：排序数组

```
v13 = compare;
 v14 = 10000LL;
 sub_557B0(*v2, 10000LL, 8LL, compare); //sub_557B0-->qsort
 sub_557B0(*v5, 10000LL, 8LL, v13);

```

作用：稳定比较，减小极值影响

#### 阶段 5：NEON 向量化比较

```
do
 {
   v21 = v18[-1];
   v22 = *v18;
   v18 += 2;
   v14 -= 4LL;
   v16.n64_u64[0] = vsub_s32(v16, vmovn_s64(vcgtq_u64(v21, vaddq_s64(v19[-1], v17)))).n64_u64[0];
   v20.n64_u64[0] = vsub_s32(v20, vmovn_s64(vcgtq_u64(v22, vaddq_s64(*v19, v17)))).n64_u64[0];
   v19 += 2;
 }
 while ( v14 );
 v23.n64_u64[0] = vadd_s32(v20, v16).n64_u64[0];
 v24 = vadd_s32(v23, vdup_lane_s32(v23, 1)).n64_u32[0];

```

作用：并行比较并累加异常计数

这段代码理解起来可能比较吃力, 换个如下实现方式就非常清晰了

```
uint32_t anomaly = 0;
for (int i = 0; i < NUM_SAMPLES; i++) {
    if (baseline[i] > syscall_array[i] + 1) {
        anomaly++;
    }
}

```

就是单纯的比较 NR_faccessat 数组和 NR_fchownat 数组里的元素大小，当出现 NR_faccessat>NR_fchownat 的情况是就记录为一次异常, 往后依次自增。

用 NEON 的目的是为了提升代码执行性能。

#### 阶段 6：判断与处理

```
if ( v24 > 0x1B58 )// 阈值 0x1B58 = 7000
{
  sub_681AC(v24); // 检测到 KernelSU，触发处理
  v15 = *v2;
}

```

作用: 超过阈值时判定存在 hook，并触发回调

#### 阶段 7：清理资源

```
operator delete(v14);
operator delete(*v5);
return 0;

```

### KernelSU 源码针对分析

针对这块的检测我们可以简单看看 KernelSU 的 hook

![](https://bbs.kanxue.com/upload/attach/202510/947335_FFM53C3JCC7KVKR.webp)

实现方式

![](https://bbs.kanxue.com/upload/attach/202510/947335_FVSA6CZVVFJQ4PU.webp)

#### 为何选择 faccessat 检测

*   选择 faccessat 因其在 KSU hook 列表中，且满足 “简单、快速、失败无副作用” 的要求
    
*   其他 hook 调用要么更复杂，要么有副作用，不适合用作可靠的基线或被测调用
    

一般来说 faccessat 的执行速度是要比 fchownat 快的，如果 faccessat 出现大量慢于 fchownat 的情况，那么说明环境有异常。

### 实践

Android 15

![](https://bbs.kanxue.com/upload/attach/202510/947335_X39ERM68M4P8MJ7.webp)

Android 10

![](https://bbs.kanxue.com/upload/attach/202510/947335_NFJFV5ZRS6DRJDE.webp)

### 小结

_// 正常路径_

syscall -> kernel -> 快速返回

时间：50-100 纳秒

_// 有任何拦截后_

syscall -> kernel -> hook 层 -> 返回

时间：200-500 纳秒

总结
--

“侧信道攻击” 防护确实非常有效，感觉针对这种防护的方式只有进一步去魔改对应的 hook 函数了, 要么就针对处理检测函数。

[[培训] 传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

最后于 55 分钟前 被九天 666 编辑 ，原因：

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#NDK 分析](forum-161-1-119.htm) [#HOOK 注入](forum-161-1-125.htm)

上传的附件：

*   [ksucheck-release.apk](javascript:void(0);) （4.94MB，4 次下载）
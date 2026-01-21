> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-289811.htm)

> [原创] 侧信道检测 APatch Root

摘要
==

APatch 在内核中 HOOK 了 truncate(45 号) 系统调用，作为应用层请求 ROOT 权限的接口。如下是相关代码，它的含义如下：

• 挑选 truncate 作为后门调用, 安装 HOOK  
• 取调用的 key 与 cmd 参数, 如果参数校验通过, 进入后门调用, 否则进入正常调用

![](https://bbs.kanxue.com/upload/attach/202601/783210_JCTSPXMRSAZUDRV.webp)

由此可以得出检验方法：

1.  设置 key, cmd 参数, 尽可能多的触发后门调用中的代码, 计算执行时间为 T1
2.  设置 key, cmd 参数, 尽可能少的触发后门调用中的代码, 计算执行时间为 T2
3.  在正常环境中, T1 与 T2 应该接近, 因为都会触发 truncate 的参数检验提前返回
4.  在 APatch 环境中, T1 与 T2 应该有较大相差值

实现细节
====

检测代码如下, 基本逻辑如下:

• 要将执行线程绑定到固定 CPU 核心上, 避免线程切换 CPU 核心产生影响  
• 设置参数 1 为 su, 参数 2 在 1000-1005 执行若干次, 计算时间 T1  
• 设置参数 1 为 su, 参数 2 在 994-999 执行若干次, 计算时间 T2  
• 在 APatch 环境中, T1 与 T2 存在较大差异, 因为 T1 会执行更多后门代码, T2 在后门代码开头即会退出  
• 在正常环境中, T1 与 T2 基本无差异, 因为会在 truncate 函数入口处由于 su 文件不存在退出

```
#define _GNU_SOURCE
 
#include #include #include static inline uint64_t get_ticks() {
  uint64_t v;
  asm volatile("isb; mrs %0, cntvct_el0; isb" : "=r"(v) : : "memory");
  return v;
}
 
void bind_to_cpu(int cpu_id) {
  cpu_set_t cpuset;
  CPU_ZERO(&cpuset);
  CPU_SET(cpu_id, &cpuset);
 
  if (sched_setaffinity(0, sizeof(cpu_set_t), &cpuset) == -1) {
    perror("sched_setaffinity");
  } else {
    printf("Thread successfully bound to CPU %d\n", cpu_id);
  }
}
 
int get_current_cpu() {
  int cpu = sched_getcpu();
  if (cpu == -1) {
    perror("sched_getcpu");
    return -1;
  }
  return cpu;
}
 
uint64_t runner(int cmd) {
  char buffer[3] = "su";
 
  uint64_t t1 = get_ticks();
  for (int i = 0; i < 10000; i++) {
    syscall(45, buffer, cmd);
  }
  uint64_t t2 = get_ticks();
  return t2 - t1;
}
 
int main(int argc, char *argv[]) {
  // 绑定cpu核心, 避免切换cpu核心对结果的影响
  bind_to_cpu(0);
 
  uint64_t d1 = 0, d2 = 0;
  for (int i = 0; i < 5; i++) {
    d2 += runner(0x999 - i);
  }
  for (int i = 0; i < 5; i++) {
    d1 += runner(0x1000 + i);
  }
  printf("Duration for cmd 0x1000: %lu tk\n", d1);
  printf("Duration for cmd 0x500 : %lu tk\n", d2);
  uint64_t diff = d1 > d2 ? d1 - d2 : d2 - d1;
  printf("Delta percentage: %.2f%%\n",
         (double)diff / ((double)(d1 + d2) / 2) * 100.0);
  return 0;
} 
```

如下图所示, APatch 环境中 T1 T2 差异较明显, 正常环境中 T1 T2 差异不明显  
![](https://bbs.kanxue.com/upload/attach/202601/783210_DMJZG4P8YQGQHKN.webp)  
![](https://bbs.kanxue.com/upload/attach/202601/783210_CBSCER9BH7PT8BS.webp)

[传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

[#系统相关](forum-161-1-126.htm)
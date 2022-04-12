> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-272284.htm)

> [原创]Go 解析

Go 解析
=====

闲谈: 在 IDA7.5 有插件支持反编译 Go, IDA7.6 已经支持的情况下, 为什么写这篇文章呢, 就算能反编译, 也是需要对照汇编的, 对于反编译出来的很多函数, 在调试时也要很快的知道哪些才是真正存储数据的地方, 增加分析效率.

Ready
=====

Go 源码下载
-------

[https://github.com/golang/go](https://github.com/golang/go)

main_main 查找
------------

这里简单提一下 main_main 的查找  
* 有符号，直接 ctrl+f，IDA 中搜索 main_main 即可

 

* 无符号：

1.  字符串查找

runtime_main 调用 main_main

*   runtime_mainIDA 反编译代码参考
    
    ```
    void runtime_main()
    {
      PVOID ArbitraryUserPointer; // rax
      __int64 v1; // rdx
      __int64 i; // rax
      __int64 v3; // [rsp+0h] [rbp-50h]
      __int64 v4; // [rsp+0h] [rbp-50h]
      __int64 v5; // [rsp+0h] [rbp-50h]
      __int64 v6; // [rsp+10h] [rbp-40h]
      __int64 v7; // [rsp+20h] [rbp-30h] BYREF
      __int64 v8; // [rsp+28h] [rbp-28h]
      __int64 v9; // [rsp+30h] [rbp-20h]
      __int128 v10; // [rsp+38h] [rbp-18h]
      void *retaddr; // [rsp+50h] [rbp+0h] BYREF
     
      while ( (unsigned __int64)&retaddr <= *(_QWORD *)(*(_QWORD *)NtCurrentTeb()->NtTib.ArbitraryUserPointer + 16LL) )
        runtime_morestack_noctxt();
      v10 = 0LL;
      HIBYTE(v7) = 0;
      v9 = *(_QWORD *)NtCurrentTeb()->NtTib.ArbitraryUserPointer;
      *(_QWORD *)(**(_QWORD **)(v9 + 48) + 304LL) = 0LL;
      runtime_maxstacksize = 1000000000LL;
      runtime_maxstackceiling = 2000000000LL;
      runtime_mainStarted = 1;
      _InterlockedExchange((volatile __int32 *)&unk_5683A0, 1);
      runtime_systemstack((__int64)&off_4D6020);
      ArbitraryUserPointer = NtCurrentTeb()->NtTib.ArbitraryUserPointer;
      ++*(_DWORD *)(*(_QWORD *)(*(_QWORD *)ArbitraryUserPointer + 48LL) + 580LL);
      v1 = *(_QWORD *)NtCurrentTeb()->NtTib.ArbitraryUserPointer;
      *(_QWORD *)(*(_QWORD *)(v1 + 48) + 312LL) = v1;
      *(_QWORD *)(v1 + 216) = *(_QWORD *)(v1 + 48);
      if ( *(_UNKNOWN **)(v9 + 48) != &runtime_m0 )
        goto LABEL_28;
      byte_568678 = 1;
      runtime_nanotime1(v3);
      runtime_runtimeInitTime = v4;
      if ( !v4 )
      {
    LABEL_27:
        runtime_throw((__int64)"nanotime returning zero", 23LL);
    LABEL_28:
        runtime_throw((__int64)"runtime.main not on m0", 22LL);
        runtime_deferreturn(v5);
        return;
      }
      if ( dword_5AB048 )
      {
        qword_5AAE88 = *(_QWORD *)(*(_QWORD *)NtCurrentTeb()->NtTib.ArbitraryUserPointer + 152LL);
        runtime_inittrace = 1;
      }
      runtime_doInit((__int64)&runtime__inittask);
      HIWORD(v7) = 257;
      *((_QWORD *)&v10 + 1) = &off_4D6028;
      *(_QWORD *)&v10 = (char *)&v7 + 6;
      runtime_gcenable();
      v6 = runtime_makechan((__int64)&unk_4B2580, 0LL);
      if ( runtime_writeBarrier )
        runtime_gcWriteBarrier();
      else
        runtime_main_init_done = v6;
      if ( !runtime_iscgo )
        goto LABEL_12;
      if ( !cgo_thread_start )
      {
    LABEL_26:
        runtime_throw((__int64)"_cgo_thread_start missing", 25LL);
        goto LABEL_27;
      }
      if ( !cgo_notify_runtime_init_done )
      {
        runtime_throw((__int64)"_cgo_notify_runtime_init_done missing", 37LL);
        goto LABEL_26;
      }
      runtime_startTemplateThread();
      runtime_cgocall(cgo_notify_runtime_init_done, 0LL, v6);
    LABEL_12:
      runtime_doInit((__int64)&main__inittask);
      runtime_inittrace = 0;
      runtime_closechan(runtime_main_init_done);
      BYTE6(v7) = 0;
      runtime_unlockOSThread();
      if ( !runtime_isarchive && !runtime_islibrary )
      {
        main_main();
        if ( runtime_runningPanicDefers )
        {
          for ( i = 0LL; i < 1000 && runtime_runningPanicDefers; i = v8 + 1 )
          {
            v8 = i;
            runtime_mcall();
          }
        }
        if ( runtime_panicking )
          v7 = runtime_gopark(0LL, 0LL, 4104, 1LL);
        runtime_exit(0);
        while ( 1 )
          MEMORY[0] = 0;
      }
      HIBYTE(v7) = 0;
      runtime_main_func2(v10);
    }
    
    ```
    

可以利用查找那些 runtime_main 中的字符串在无符号的 Go 程序中找到 runtime_main，从而找到 main_main

 

![](https://bbs.pediy.com/upload/attach/202204/921830_464F4PY7JQ7VS9S.png)

1.  编译一份有符号的，同位数的 Go 可执行程序 bindiff 恢复符号找到 runtime_main 或一些关键函数

**notice：**

 

本文的数据均在 64 位 环境下得出.

 

可关掉编译优化

```
-gcflags "-N -l"
 
go build -o hello.exe -gcflags "-N -l" hello.go

```

Go 数据结构解析
=========

字符串
---

![](https://bbs.pediy.com/upload/attach/202204/921830_W92CV2BY8PT5QRW.png)  
Go 程序会将字符串全部存放到一块连续的内存当中，使用的时候会利用 runtime_convTstring 来构造真正的字符串内存格式

```
struct String{
    char * strPtr;   //指向目标字符串的开始地址
    int64 size;      //字符串大小
}

```

![](https://bbs.pediy.com/upload/attach/202204/921830_DX5YXTJ8SYFFFGT.png)

数组
--

```
x := []int{1, 2, 3, 4, 5}

```

![](https://bbs.pediy.com/upload/attach/202204/921830_6TYV34782BU4TSE.png)  
通常会有一个地址作为 runtime_newobject 的参数，这个地址存放的是我们将要构造的数组的总大小，比如这里 qword_234820 地址处存放的就是 40(5 * 8)

 

runtime_newobject 里面会调用 runtime_mallocgc 分配内存

 

再之后会将 runtime_newobject 返回的指向这块内存的指针，存放到当前函数栈中的一个局部变量中，以后对这个数组的操作都是通过这个局部变量来的，比如这里的 [rsp+168](pointer)

 

再下方就是对那块内存赋值, 也就是对数组初始化

```
y := []float32{1000.0, 2.0, 3.4, 7.0, 50.0}

```

![](https://bbs.pediy.com/upload/attach/202204/921830_7SJG9DKCDWB5H2P.png)

slice 切片
--------

### 内存结构

一个 slice 是一个数组某个部分的引用。

 

在内存中，它是一个包含 3 个域的结构体：

```
{
    指向slice中第一个元素的指针
    slice的长度
    slice的容量
}

```

这里解释一下长度和容量的概念：

 

长度：下标操作的上界，如 x[i] 中 i 必须小于长度

 

容量：分割操作的上界，如 x[i:j] 中 j 不能大于容量

 

![](https://bbs.pediy.com/upload/attach/202204/921830_W8G7SJ489NP6PAU.png)  
数组的 slice 并不会实际复制一份数据，它只是创建一个新的数据结构，包含了另外的一个指针，一个长度和一个容量数据。

 

比如上面的分割表达式`x[1:3]`并不分配更多的数据：它只是写了一个新的 slice 结构的属性来引用相同的存储数据。

 

在例子中，长度为 2，只有 y[0] 和 y[1] 是有效的索引，但是容量为 4，y[0:4] 是一个有效的分割表达式。

 

![](https://bbs.pediy.com/upload/attach/202204/921830_EDEAUPB4HVADWV7.png)  
这里先创建一个数组，然后 rax 存放的是构造好的数组的首地址，add rax, 8 之后就会指向数组的下标 1 对应的元素，也就是 3 存放的地址，也就是我们切片 [1:3] 的第一个元素的指针，之后

 

.text:00000000004A79A8 mov [rsp], rax  
.text:00000000004A79AC mov qword ptr [rsp+8], 2  
.text:00000000004A79B5 mov qword ptr [rsp+16], 4

 

这里就在内存中构造出来了一个切片的结构

 

调用了 runtime_convTslice，返回值是一个指向新分配的内存的指针，这个新分配的内存存放的元素数量就是切片的容量，比如这里就存放了 4 个元素，[3, 5, 7, 11] (从切片的那个元素开始，往后取容量那么多个)，所以这里就是为什么 y[0:4] 是一个有效的分割表达式。

 

![](https://bbs.pediy.com/upload/attach/202204/921830_RXG8K69A8KG4DK3.png)

### **make**

`slice := make([]int, len)`

```
slice1 := make([]int, 5)
slice1[3] = 66

```

![](https://bbs.pediy.com/upload/attach/202204/921830_7J79RU2NHATXWNS.png)  
make 的底层会调用 runtime_makeslice 分配 Array，真正的切片结构是后面的 runtime_convTslice 时创建的

```
// runtime_makeslice
func makeslice(et *_type, len, cap int) unsafe.Pointer {
    mem, overflow := math.MulUintptr(et.size, uintptr(cap))
    if overflow || mem > maxAlloc || len < 0 || len > cap {
        // NOTE: Produce a 'len out of range' error instead of a
        // 'cap out of range' error when someone does make([]T, bignumber).
        // 'cap out of range' is true too, but since the cap is only being
        // supplied implicitly, saying len is clearer.
        // See golang.org/issue/4085.
        mem, overflow := math.MulUintptr(et.size, uintptr(len))
        if overflow || mem > maxAlloc || len < 0 {
            panicmakeslicelen()
        }
        panicmakeslicecap()
    }
 
    return mallocgc(mem, et, true)
}

```

makeslice 返回的是一个指向实际数据的指针（不含管理 slice 的结构体）相当于 `malloc(sizeof(Type) * len)`

 

在访问 slice 中元素时，一般会检测下标是否小于 len，如果越界则调用`runtime_panicIndex`

### ****append/copy****

```
slice1 = append(slice1, 123)

```

append 的时候会检测目标 slice1.len + 1 与 slice1.cap 的大小关系

 

若 slice1.len + 1 > slice1.cap 则调用 runtime_growslice 扩容

 

在对 slice 进行 append 等操作时，可能会造成 slice 的自动扩容。其扩容时的大小增长规则是：

*   如果新的大小是当前大小 2 倍以上，则大小增长为新大小
*   否则循环以下操作：如果当前大小小于 1024，按每次 2 倍增长，否则每次按当前大小 1/4 增长。直到增长的大小超过或等于新大小。

copy 就是复制一个新的切片

### ****切片截取****

```
myvar := slice1[1:3]

```

myvar 的数据结构是新一个新的切片 struct.

map
---

### 内存结构

Go 语言的 map 使用 Hash 表作为底层实现，可以在 $GOROOT/src/pkg/runtime/hashmap.goc 找到它的实现，其中 `[runtime.hmap](https://draveness.me/golang/tree/runtime.hmap)`是最核心的结构体，我们先来了解一下该结构体的内部字段：

```
type hmap struct {
    count     int     //map中键值对数量
    flags     uint8   //map当前是否处于写入状态登
    B         uint8   //2的B次幂表示当前map中桶的数量
    noverflow uint16  //map中溢出桶的数量，当溢出桶太多时，map会进行等量扩容
    hash0     uint32  //生成hash的随机数种子
 
    buckets    unsafe.Pointer  //当前map对应的桶的指针
    oldbuckets unsafe.Pointer  //map扩容时指向旧桶的指针，当所有旧桶中的数据转移到新桶时，清空
    nevacuate  uintptr         //扩容时，用于标记当前旧桶中小于nevacute的数据都已经转移到了新桶
 
    extra *mapextra            //存储map的溢出桶
}
 
type mapextra struct {
    overflow    *[]*bmap
    oldoverflow *[]*bmap
    nextOverflow *bmap
}

```

这个 hash 结构使用的是一个可扩展哈希的算法，由 hash 值 mod 当前 hash 表大小决定某一个值属于哪个桶，而 hash 表大小是 2 的指数，即上面结构体中的 2^B。每次扩容，会增大到上次大小的两倍。结构体中有一个 buckets 和一个 oldbuckets 是用来实现增量扩容的。正常情况下直接使用 buckets，而 oldbuckets 为空。如果当前哈希表正在扩容中，则 oldbuckets 不为空，并且 buckets 大小是 oldbuckets 大小的两倍。

 

具体的 Bucket 结构如下所示：

```
type bmap struct {
    tophash [8]uint8 //存储Hash值的高8位
    data []byte    //key value数据：key/key/key.../value/value/value...
    overflow *bmap    //溢出bucket的地址
}

```

*   BUCKETSIZE 是用宏定义的 8，每个 bucket 中存放最多 8 个 key/value 对, 如果多于 8 个，那么会申请一个新的 bucket，并将它与之前的 bucket 链起来。
*   按 key 的类型采用相应的 hash 算法得到 key 的 hash 值。将 hash 值的低位当作 Hmap 结构体中 buckets 数组的 index，找到 key 所在的 bucket。将 hash 的高 8 位存储在了 bucket 的 tophash 中。**注意，这里高 8 位不是用来当作 key/value 在 bucket 内部的 offset 的，而是作为一个主键，在查找时对 tophash 数组的每一项进行顺序匹配的**。先比较 hash 值高位与 bucket 的 tophash[i] 是否相等，如果相等则再比较 bucket 的第 i 个的 key 与所给的 key 是否相等。如果相等，则返回其对应的 value，反之，在 overflow buckets 中按照上述方法继续寻找。
*   data 区存放的是 key—value 数据，其中 keys 放在一起，values 放在一起，如此存储是为了节省字节对齐带来的空间浪费。例如 map[int64]int8。
*   overflow 指针指向的是下一个 bucket，据此将所有冲突的键连接起来

### 赋值和访问

赋值

```
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer{}

```

3 个参数，第三个参数是 key 的指针，rsp + 16， 返回值是 key 对应的数据指针.

 

访问

```
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {}

```

和赋值同理，区别是这个不会为不存在的 key 创建 key pair.

```
package main
 
import "fmt"
 
func main() {
    countryCapitalMap := map[string]string{"France": "Paris", "Italy": "Rome", "Japan": "Tokyo", "India": "New delhi"}
 
    fmt.Println("France首都是", countryCapitalMap ["France"])
}

```

![](https://bbs.pediy.com/upload/attach/202204/921830_UP2N3PZ52W5QE2V.png)  
比如上方在执行完  
![](https://bbs.pediy.com/upload/attach/202204/921830_T6QAW3ZKSX5E7C7.png)  
之后，map buckets 是存放了一个键值对的，可以过去查看 v5(`h *hmap`) ，0x000000C00014FE38 来查看

 

![](https://bbs.pediy.com/upload/attach/202204/921830_BRJSM4484E8YSA2.png)  
可以看见 buckets unsafe.Pointer 为 byte_C00014FE68

 

![](https://bbs.pediy.com/upload/attach/202204/921830_29SZWRQMDKJ9Q26.png)  
前面 8 字节的是 tophash 数组，目前只存放了一个键值对，就只有一个存放了，hash(”France”) 的高 8bit，往下就是存放键值对

```
package main
 
import "fmt"
 
func main() {
    var countryCapitalMap map[string]string /*创建集合, 默认map是nil*/
    //如果不初始化 map，那么就会创建一个 nil map, nil map 不能用来存放键值对
    countryCapitalMap = make(map[string]string)
 
    countryCapitalMap [ "France" ] = "巴黎"
    countryCapitalMap [ "Italy" ] = "罗马"
    countryCapitalMap [ "Japan" ] = "东京"
    countryCapitalMap [ "India " ] = "新德里"
 
    //或者：countryCapitalMap := map[string]string{"France": "Paris", "Italy": "Rome", "Japan": "Tokyo", "India": "New delhi"}
 
    /*使用键输出地图值 */
    for country := range countryCapitalMap {
        fmt.Println(country, "首都是", countryCapitalMap [country])
    }
}

```

```
void __cdecl main_main()
{
  int v0; // [rsp+10h] [rbp-230h]
  __int64 v1; // [rsp+20h] [rbp-220h]
  __int64 v2; // [rsp+20h] [rbp-220h]
  __int64 v3; // [rsp+20h] [rbp-220h]
  __int64 v4; // [rsp+20h] [rbp-220h]
  _QWORD *v5; // [rsp+30h] [rbp-210h]
  __int64 *v6; // [rsp+30h] [rbp-210h]
  __int64 v7; // [rsp+38h] [rbp-208h]
  __int64 v8; // [rsp+40h] [rbp-200h]
  __int64 v9; // [rsp+48h] [rbp-1F8h]
  __int64 v10; // [rsp+50h] [rbp-1F0h]
  __int64 v11; // [rsp+58h] [rbp-1E8h]
  __int64 v12; // [rsp+60h] [rbp-1E0h]
  _QWORD v13[5]; // [rsp+68h] [rbp-1D8h] BYREF
  __int64 v14; // [rsp+90h] [rbp-1B0h] BYREF
  __int128 v15; // [rsp+98h] [rbp-1A8h] BYREF
  __int128 v16; // [rsp+A8h] [rbp-198h]
  __int128 v17; // [rsp+B8h] [rbp-188h]
  __int64 v18[12]; // [rsp+C8h] [rbp-178h] BYREF
  char v19; // [rsp+128h] [rbp-118h] BYREF
 
  while ( (unsigned __int64)&v14 <= *(_QWORD *)(*(_QWORD *)NtCurrentTeb()->NtTib.ArbitraryUserPointer + 16LL) )
    runtime_morestack_noctxt();
  v15 = 0LL;
  v16 = 0LL;
  v17 = 0LL;
  ((void (*)(void))loc_46651C)();
  *(_QWORD *)&v16 = &v19;
  runtime_fastrand();
  HIDWORD(v15) = v0;
  runtime_mapassign_faststr((__int64)&unk_4B7040, (__int64)&v15, (__int64)"France", 6LL);
  v5[1] = 6LL;
  if ( runtime_writeBarrier )
    runtime_gcWriteBarrier();
  else
    *v5 = "巴黎";
  runtime_mapassign_faststr((__int64)&unk_4B7040, (__int64)&v15, (__int64)"Italy", 5LL);
  v5[1] = 6LL;
  if ( runtime_writeBarrier )
    runtime_gcWriteBarrier();
  else
    *v5 = "罗马";
  runtime_mapassign_faststr((__int64)&unk_4B7040, (__int64)&v15, (__int64)"Japan", 5LL);
  v5[1] = 6LL;
  if ( runtime_writeBarrier )
    runtime_gcWriteBarrier();
  else
    *v5 = "东京";
  v8 = runtime_mapassign_faststr((__int64)&unk_4B7040, (__int64)&v15, (__int64)"India ", 6LL);
  v5[1] = 9LL;
  if ( runtime_writeBarrier )
    runtime_gcWriteBarrier();
  else
    *v5 = "新德里";
  ((void (*)(void))loc_466551)();
  runtime_mapiterinit((__int64)&unk_4B7040, &v15, (__int64)v18);
  while ( v18[0] )
  {
    v11 = *(_QWORD *)v18[0];
    v10 = *(_QWORD *)(v18[0] + 8);
    runtime_convTstring(*(_QWORD *)v18[0], v10, v1);
    v12 = v2;
    v9 = runtime_mapaccess1_faststr((__int64)&unk_4B7040, (__int64)&v15, v11, v10, (__int64)v5);
    v7 = runtime_convTstring(*v6, v6[1], v3);
    v13[0] = &unk_4B3260;
    v13[1] = v12;
    v13[2] = &unk_4B3260;
    v13[3] = &off_4EF518;
    v13[4] = &unk_4B3260;
    v14 = v4;
    fmt_Fprintln((__int64)&go_itab__os_File_io_Writer, os_Stdout, (__int64)v13, 3LL, 3LL, v7, v8, v9);
    runtime_mapiternext((__int64)v18);
  }
}

```

### **查找过程**

1.  根据 key 计算出 hash 值。
2.  如果存在 old table, 首先在 old table 中查找，如果找到的 bucket 已经 evacuated，转到步骤 3。 反之，返回其对应的 value。
3.  在 new table 中查找对应的 value。

这里一个细节需要注意一下。不认真看可能会以为低位用于定位 bucket 在数组的 index，那么高位就是用于 key/valule 在 bucket 内部的 offset。事实上高 8 位不是用作 offset 的，而是用于加快 key 的比较的。

```
do { //对每个桶b
    //依次比较桶内的每一项存放的tophash与所求的hash值高位是否相等
    for(i = 0, k = b->data, v = k + h->keysize * BUCKETSIZE; i < BUCKETSIZE; i++, k += h->keysize, v += h->valuesize) {
        if(b->tophash[i] == top) {
            k2 = IK(h, k);
            t->key->alg->equal(&eq, t->key->size, key, k2);
            if(eq) { //相等的情况下再去做key比较...
                *keyp = k2;
                return IV(h, v);
            }
        }
    }
    b = b->overflow; //b设置为它的下一下溢出链
} while(b != nil);

```

### **插入过程**

1.  根据 key 算出 hash 值，进而得出对应的 bucket。
2.  如果 bucket 在 old table 中，将其重新散列到 new table 中。
3.  在 bucket 中，查找空闲的位置，如果已经存在需要插入的 key，更新其对应的 value。
4.  根据 table 中元素的个数，判断是否 grow table。
5.  如果对应的 bucket 已经 full，重新申请新的 bucket 作为 overbucket。
6.  将 key/value pair 插入到 bucket 中。

这里也有几个细节需要注意一下。

 

在扩容过程中，oldbucket 是被冻结的，查找时会在 oldbucket 中查找，但不会在 oldbucket 中插入数据。如果在 oldbucket 是找到了相应的 key，做法是将它迁移到新 bucket 后加入 evalucated 标记。并且还会额外的迁移另一个 pair。

 

然后就是只要在某个 bucket 中找到第一个空位，就会将 key/value 插入到这个位置。也就是位置位于 bucket 前面的会覆盖后面的 (类似于存储系统设计中做删除时的常用的技巧之一，直接用新数据追加方式写，新版本数据覆盖老版本数据)。找到了相同的 key 或者找到第一个空位就可以结束遍历了。不过这也意味着做删除时必须完全的遍历 bucket 所有溢出链，将所有的相同 key 数据都删除。所以目前 map 的设计是为插入而优化的，删除效率会比插入低一些。

### **删除过程**

删除元素实际上也是先查找元素，如果元素存在则把元素从相应的 bucket 中删除，如果不存在则什么也不做。

 

**notice**

 

Go 调用汇编和 C：[https://tiancaiamao.gitbooks.io/go-internals/content/zh/03.1.html](https://tiancaiamao.gitbooks.io/go-internals/content/zh/03.1.html)

G,M,P 模型
========

在讲解函数调用之前，我们讲解一下 GMP 结构体

 

并发：一个逻辑流的执行在时间上与另一个流重叠，叫并发流，多个流并发地执行被成为并发。

 

并行：如果两个流并发地运行在不同的处理器核或者计算机上，那么我们成为它们为并行地执行。

 

并行流是并发流的真子集。

 

多线程或多进程是并行的基本条件，但单线程也可以用协程 (coroutine) 做到并发。简单将 Goroutine 归纳为协程并不合适，因为它运行时会创建多个线程来执行并发任务，且任务单元可被调度到其它线程执行。这更像是多线程和协程的结合体，能最大限度提升执行效率，发挥多核处理器能力。

 

Go 的并发实现非常简单，使用一个 go 关键字即可

```
package main
 
import (
        "fmt"
        "time"
)
 
func say(s string) {
        for i := 0; i < 5; i++ {
                time.Sleep(100 * time.Millisecond)
                fmt.Println(s)
        }
}
 
func main() {
        go say("world")
        say("hello")
}

```

Go 语言虽然使用一个 Go 关键字即可实现并发编程，但 Goroutine 被调度到后端之后，具体的实现比较复杂。**Go 调度器组成。**

G
-

G 是 Goroutine 的缩写，相当于操作系统中的进程控制块，在这里就是 Goroutine 的控制结构，是对 Goroutine 的抽象。其中包括执行的函数指令及参数；G 保存的任务对象；线程上下文切换，现场保护和现场恢复需要的寄存器 (SP、IP) 等信息。

```
type g struct {
    // Stack parameters.
    // stack describes the actual stack memory: [stack.lo, stack.hi).
    // stackguard0 is the stack pointer compared in the Go stack growth prologue.
    // It is stack.lo+StackGuard normally, but can be StackPreempt to trigger a preemption.
    // stackguard1 is the stack pointer compared in the C stack growth prologue.
    // It is stack.lo+StackGuard on g0 and gsignal stacks.
    // It is ~0 on other goroutine stacks, to trigger a call to morestackc (and crash).
    // 记录该goroutine使用的栈
    stack       stack   // offset known to runtime/cgo
 
    //下面两个成员用于栈溢出检查，实现栈的自动伸缩，抢占调度也会用到stackguard0
    stackguard0 uintptr // offset known to liblink
      stackguard1 uintptr // offset known to liblink
 
        _panic         *_panic // innermost panic - offset known to liblink
        _defer         *_defer // innermost defer
 
        // 此goroutine正在被哪个工作线程执行
        m              *m      // current m; offset known to arm liblink
        //这个字段跟调度切换有关，G切换时用来保存上下文，保存什么，看下面gobuf结构体
        sched          gobuf
        syscallsp      uintptr        // if status==Gsyscall, syscallsp = sched.sp to use during gc
        syscallpc      uintptr        // if status==Gsyscall, syscallpc = sched.pc to use during gc
        stktopsp       uintptr        // expected sp at top of stack, to check in traceback
        param          unsafe.Pointer // passed parameter on wakeup，wakeup唤醒时传递的参数
        // 状态Gidle,Grunnable,Grunning,Gsyscall,Gwaiting,Gdead
      atomicstatus   uint32
        stackLock      uint32 // sigprof/scang lock; TODO: fold in to atomicstatus
        goid           int64
 
        //schedlink字段指向全局运行队列中的下一个g，
        //所有位于全局运行队列中的g形成一个链表
        schedlink      guintptr
        waitsince      int64      // approx time when the g become blocked
        waitreason     waitReason // if status==Gwaiting，g被阻塞的原因
        //抢占信号，stackguard0 = stackpreempt，如果需要抢占调度，设置preempt为true
        preempt        bool       // preemption signal, duplicates stackguard0 = stackpreempt
        paniconfault   bool       // panic (instead of crash) on unexpected fault address
        preemptscan    bool       // preempted g does scan for gc
        gcscandone     bool       // g has scanned stack; protected by _Gscan bit in status
        gcscanvalid    bool       // false at start of gc cycle, true if G has not run since last scan; TODO: remove?
        throwsplit     bool       // must not split stack
        raceignore     int8       // ignore race detection events
        sysblocktraced bool       // StartTrace has emitted EvGoInSyscall about this goroutine
        sysexitticks   int64      // cputicks when syscall has returned (for tracing)
        traceseq       uint64     // trace event sequencer
        tracelastp     puintptr   // last P emitted an event for this goroutine
        // 如果调用了 LockOsThread，那么这个 g 会绑定到某个 m 上
      lockedm        muintptr
        sig            uint32
        writebuf       []byte
        sigcode0       uintptr
        sigcode1       uintptr
        sigpc          uintptr
        // 创建这个goroutine的go表达式的pc
        gopc           uintptr         // pc of go statement that created this goroutine
        ancestors      *[]ancestorInfo // ancestor information goroutine(s) that created this goroutine (only used if debug.tracebackancestors)
        startpc        uintptr         // pc of goroutine function
        racectx        uintptr
        waiting        *sudog         // sudog structures this g is waiting on (that have a valid elem ptr); in lock order
        cgoCtxt        []uintptr      // cgo traceback context
        labels         unsafe.Pointer // profiler labels
        timer          *timer         // cached timer for time.Sleep,为 time.Sleep 缓存的计时器
        selectDone     uint32         // are we participating in a select and did someone win the race?
 
        // Per-G GC state
 
        // gcAssistBytes is this G's GC assist credit in terms of
        // bytes allocated. If this is positive, then the G has credit
        // to allocate gcAssistBytes bytes without assisting. If this
        // is negative, then the G must correct this by performing
        // scan work. We track this in bytes to make it fast to update
        // and check for debt in the malloc hot path. The assist ratio
        // determines how this corresponds to scan work debt.
        gcAssistBytes int64
}

```

stack 描述了当前 goroutine 的栈内存范围 [stack.lo, stack.hi)，其中 stack 的数据结构：

```
// Stack describes a Go execution stack.
// The bounds of the stack are exactly [lo, hi),
// with no implicit data structures on either side.
// 描述 goroutine 执行栈
// 栈边界为[lo, hi)，左包含右不包含，即 lo≤stack
```

stackguard0 和 stackguard1 均是一个栈指针，用于扩容场景，前者用于 Go stack ，后者用于 C stack。

M
-

M 是一个线程或称为 Machine，所有 M 是有线程栈的。如果不对该线程栈提供内存的话，系统会给该线程栈提供内存 (不同操作系统提供的线程栈大小不同)。当指定了线程栈，则 M.stack→G.stack，M 的 PC 寄存器指向 G 提供的函数，然后去执行。

```
type m struct {   
    /*
        1.  所有调用栈的Goroutine,这是一个比较特殊的Goroutine。
        2.  普通的Goroutine栈是在Heap分配的可增长的stack,而g0的stack是M对应的线程栈。
        3.  所有调度相关代码,会先切换到该Goroutine的栈再执行。
    */
    g0       *g
    curg     *g         // M当前绑定的结构体G
 
    // SP、PC寄存器用于现场保护和现场恢复
    vdsoSP uintptr
    vdsoPC uintptr
 
    // 省略…}

```

P
-

P(Processor) 是一个抽象的概念，并不是真正的物理 CPU。所以当 P 有任务时需要创建或者唤醒一个系统线程来执行它队列里的任务。所以 P/M 需要进行绑定，构成一个执行单元。

 

P 决定了同时可以并发任务的数量，可通过 GOMAXPROCS 限制同时执行用户级任务的操作系统线程。可以通过 runtime.GOMAXPROCS 进行指定。在 Go1.5 之后 GOMAXPROCS 被默认设置可用的核数，而之前则默认为 1。

Go 调度器调度过程
==========

首先创建一个 G 对象，G 对象保存到 P 本地队列或者是全局队列。

 

P 此时去唤醒一个 M。P 继续执行它的执行序。

 

M 寻找是否有空闲的 P，如果有则将该 G 对象移动到它本身。

 

接下来 M 执行一个调度循环 (调用 G 对象 -> 执行 ->清理线程→继续找新的 Goroutine 执行)。

 

M 执行过程中，随时会发生上下文切换。

 

当发生上下文切换时，需要对执行现场进行保护，以便下次被调度执行时进行现场恢复。

 

Go 调度器 M 的栈保存在 G 对象上，只需要将 M 所需要的寄存器 (SP、PC 等) 保存到 G 对象上就可以实现现场保护。当这些寄存器数据被保护起来，就随时可以做上下文切换了，在中断之前把现场保存起来。如果此时 G 任务还没有执行完，M 可以将任务重新丢到 P 的任务队列，等待下一次被调度执行。当再次被调度执行时，M 通过访问 G 的 vdsoSP、vdsoPC 寄存器进行现场恢复(从上次中断位置继续执行)。

 

后面分析函数的开头和结尾的时候可以对照。

连续栈
===

Go 语言支持 goroutine，每个 goroutine 需要能够运行，所以它们都有自己的栈。假如每个 goroutine 分配固定栈大小并且不能增长，太小则会导致溢出，太大又会浪费空间，无法存在许多的 goroutine。

 

为了解决这个问题，goroutine 可以初始时只给栈分配很小的空间，然后随着使用过程中的需要自动地增长。这就是为什么 Go 可以开千千万万个 goroutine 而不会耗尽内存。

 

Go1.3 版本之后则使用的是 continuous stack，下面将具体分析一下这种技术。

基本原理
----

每次执行函数调用时 Go 的 runtime 都会进行检测，若当前栈的大小不够用，则会触发 “中断”，从当前函数进入到 Go 的运行时库，Go 的运行时库会保存此时的函数上下文环境，然后分配一个新的足够大的栈空间，将旧栈的内容拷贝到新栈中，并做一些设置，使得当函数恢复运行时，函数会在新分配的栈中继续执行，仿佛整个过程都没发生过一样，这个函数会觉得自己使用的是一块大小“无限” 的栈空间。

实际分析
----

![](https://bbs.pediy.com/upload/attach/202204/921830_KKC3RHNKSJUZTSF.png)  
![](https://bbs.pediy.com/upload/attach/202204/921830_36SNSYKPHR2XXZU.png)  
Go 语言和 C 不同，不是使用栈指针寄存器和栈基址寄存器确定函数的栈的。

 

在 Go 的运行时库中，每个 goroutine 对应一个结构体 G，大致相当于进程控制块的概念。

 

这个结构体中存了 stackbase 和 stackguard，用于确定这个 goroutine 使用的栈空间信息。

 

每个 Go 函数调用的前几条指令，先比较栈指针寄存器跟 g->stackguard(偏移 0x10)，检测是否发生栈溢出。

 

如果栈指针寄存器值超越了 stackguard 就需要扩展栈空间。

 

函数开头首先 gs:0x28，获取到了 g 结构体地址

 

后面的 g+0x10 也就是 g->stackguard 的地址

 

所以是把 rsp 和 g->stackguard 的值比较

 

如果 SP 大于 g->stackguard 了，则会跳转到函数结尾调用 runtime.morestack 函数。函数开头几条指令的作用就是检测栈是否溢出。

 

runtime.morestack 作用：

 

将一些信息存在 M 结构体中，这些信息包括当前栈桢，参数，当前函数调用，函数返回地址（两个返回地址，一个是 runtime.morestack 的函数地址，一个是 f 的返回地址）。通过这些信息可以把新栈和旧栈链起来。

```
void runtime.morestack() {
    if(g == g0) {
        panic();
    } else {
        m->morebuf.gobuf_pc = getCallerCallerPC();
        void *SP = getCallerSP();
        m->morebuf.gobuf_sp = SP;
        m->moreargp = SP;
        m->morebuf.gobuf_g = g;
        m->morepc = getCallerPC();
 
        void *g0 = m->g0;
        g = g0;
        setSP(g0->g_sched.gobuf_sp);
        runtime.newstack();
    }
}

```

函数调用 (支持多返回值)
=============

无论是 x86 还是 x86-64， 都采用栈传递参数

 

返回值传递不通过 eax/rax 等寄存器，也是通过栈。

 

位置是最后一个参数的下面。例如最后一个参数的地址是 `rsp + 0x8`, 则:

*   第一个返回值：rsp + 0x10
*   第二个返回值：rsp + 0x18
*   以此类推

[](#一个参数，多个返回值)一个参数，多个返回值
-------------------------

```
package main
 
import "fmt"
 
func main() {
    var a, b int
    a, b = sayHello(123)
    a = a + 1
    b = b + 2
    fmt.Println(a, b)
}
 
func sayHello(a int)(i,j int){
    i = a + 1
    fmt.Println("execute half")
    j = a + 2
    return
}

```

![](https://bbs.pediy.com/upload/attach/202204/921830_88Y8CF2Q9E4TC73.png)

[](#多个参数，多个返回值)多个参数，多个返回值
-------------------------

```
package main
 
import (
    "fmt"
)
 
func main() {
    var a, b int
    a, b = sayHello(1234, 5678)
    a = a + 1
    b = b + 2
    fmt.Println(a, b)
}
 
func sayHello(a int, b int)(i,j int){
    i = a + 1
    fmt.Println("execute half")
    j = b + 2
    return
}

```

![](https://bbs.pediy.com/upload/attach/202204/921830_U6UT84WCCVHME4K.png)  
返回值都会存放到 main 函数的栈中，main 的局部变量

 

参考：

 

[https://tiancaiamao.gitbooks.io/go-internals/content/zh](https://tiancaiamao.gitbooks.io/go-internals/content/zh)

 

[https://blog.csdn.net/star_of_science/article/details/121802354](https://blog.csdn.net/star_of_science/article/details/121802354)  
[https://panda0s.top/2021/04/14/Golang-underlying-data-representaion/#Golang-underlying-data-representaion](https://panda0s.top/2021/04/14/Golang-underlying-data-representaion/#Golang-underlying-data-representaion)

 

[https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-hashmap/)

 

[https://tiancaiamao.gitbooks.io/go-internals/content/zh/03.1.html](https://tiancaiamao.gitbooks.io/go-internals/content/zh/03.1.html)

 

[https://www.1024sou.com/article/287980.html](https://www.1024sou.com/article/287980.html)

 

[https://segmentfault.com/a/1190000040933033](https://segmentfault.com/a/1190000040933033)

[【公告】 [2022 大礼包]《看雪论坛精华 22 期》发布！收录近 1000 余篇精华优秀文章!](https://bbs.pediy.com/thread-271749.htm)

[#其他内容](forum-4-1-10.htm)
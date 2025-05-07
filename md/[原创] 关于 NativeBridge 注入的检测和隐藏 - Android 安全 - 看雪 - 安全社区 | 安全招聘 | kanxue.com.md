> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-286536.htm)

> [原创] 关于 NativeBridge 注入的检测和隐藏

前言
--

在装有`Riru/LSPosed`的机子上, 突然发现`中国移动(com.greenpoint.android.mc10086.activity)`(中国电信也是) 打不开了, 一直卡在启动页上, 让我看看是怎么回事.

1.  停用`LSPosed`依旧打不开
2.  停用`Riru`能打开
3.  注释`jni_hooks`依旧打不开
4.  注释`loader`(空 loader.so) 也不行
5.  注释`system.prop`中的`ro.dalvik.vm.native.bridge=libriruloader.so` 正常
6.  随意`resetprop ro.dalvik.vm.native.bridge xx`后`am restart`, 还原再打开也不行
7.  只有`zygote`启动前没有`ro.dalvik.vm.native.bridge`值或者为`0`才正常证明检测了`ro.dalvik.vm.native.bridge`的值, 或者检测到了`resetprop`的修改

关于`resetprop`的检测可以看这个: https://t.me/nullptr_dev/60  
![](https://bbs.kanxue.com/upload/attach/202504/98159_PDT5JSAFK2RBXM2.webp)  
不过也证明不是, 因为改了之后再还原, 依旧打不开, 唯二的可能是就`zygote`中记录了参数值 (也证明不是, 因为整个内存搜索了没有参数被记录) 或者有标识成已尝试加载 NativeBridge.

确认
--

### NativeBrige 流程

具体就不啰嗦了, 可以看看这个文章: https://blog.csdn.net/Roland_Sun/article/details/49688311

关键看 [LoadNativeBridge](elink@5f2K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6U0M7#2)9J5k6h3q4F1k6s2u0G2K9h3c8Q4x3X3g2U0L8$3#2Q4x3V1k6S2L8X3c8J5L8$3W2V1i4K6u0r3M7r3I4S2N6r3k6G2M7X3#2Q4x3V1k6K6N6i4m8W2M7Y4m8J5L8$3A6W2j5%4c8Q4x3V1k6E0j5h3W2F1i4K6u0r3i4K6u0n7i4K6u0r3L8h3q4A6L8W2)9K6b7h3q4J5N6q4)9J5c8X3I4A6j5X3&6S2N6r3W2$3k6h3u0J5K9h3c8Y4k6g2)9J5c8X3&6S2N6r3W2$3k6g2)9#2k6X3u0J5K9h3c8Y4k6g2)9J5k6h3y4U0i4K6y4n7k6s2u0U0i4K6y4p5y4U0p5I4z5e0M7K6y4U0b7K6y4U0N6U0z5h3f1@1x3o6c8U0y4$3c8S2y4U0V1H3x3o6j5#2z5r3j5I4j5U0p5$3j5K6b7J5k6o6m8V1j5g2)9K6b7X3I4Q4x3@1b7J5x3K6y4Q4x3@1k6Z5L8q4)9K6c8s2A6Z5i4K6u0V1j5$3^5`.), 只要`ro.dalvik.vm.native.bridge`不空或者`0`都会执行到这  
![](https://bbs.kanxue.com/upload/attach/202504/98159_VRMJ9D7PA4A5PGT.webp)  
看出来了吧, 只要有值 (不是''或者 0) 就会调用 [CloseNativeBridge](elink@152K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6U0M7#2)9J5k6h3q4F1k6s2u0G2K9h3c8Q4x3X3g2U0L8$3#2Q4x3V1k6S2L8X3c8J5L8$3W2V1i4K6u0r3M7r3I4S2N6r3k6G2M7X3#2Q4x3V1k6K6N6i4m8W2M7Y4m8J5L8$3A6W2j5%4c8Q4x3V1k6E0j5h3W2F1i4K6u0r3i4K6u0n7i4K6u0r3L8h3q4A6L8W2)9K6b7h3q4J5N6q4)9J5c8X3I4A6j5X3&6S2N6r3W2$3k6h3u0J5K9h3c8Y4k6g2)9J5c8X3&6S2N6r3W2$3k6g2)9#2k6X3u0J5K9h3c8Y4k6g2)9J5k6h3y4U0i4K6y4n7k6s2u0U0i4K6y4p5y4U0p5I4z5e0M7K6y4U0b7K6y4U0N6U0z5h3f1@1x3o6c8U0y4$3c8S2y4U0V1H3x3o6j5#2z5r3j5I4j5U0p5$3j5K6b7J5k6o6m8V1j5g2)9K6b7X3u0H3N6W2)9K6c8o6q4Q4x3@1u0T1M7s2c8Q4x3@1b7I4i4K6y4n7L8q4)9K6c8o6t1J5y4#2)9K6c8X3S2D9i4K6y4p5P5X3S2Q4x3X3c8U0L8W2)9J5y4X3q4E0M7q4)9K6b7X3N6K6L8W2)9K6c8p5y4D9L8%4y4W2e0X3q4@1K9i4k6W2b7Y4u0A6k6r3N6W2i4K6t1$3j5h3#2H3i4K6y4n7k6%4y4Q4x3@1c8w2h3g2c8t1c8g2)9J5y4e0y4m8i4K6t1#2x3V1k6Q4x3U0f1J5c8X3E0&6N6r3S2W2i4K6t1#2x3@1q4Q4x3U0f1J5c8W2)9J5y4e0u0r3j5h3&6V1M7X3!0A6k6q4)9J5k6h3N6G2L8$3N6D9k6i4y4G2N6i4u0U0k6g2)9J5k6h3y4G2L8g2)9J5y4e0u0r3M7r3I4S2N6r3k6G2M7X3#2Q4x3U0f1J5c8Y4y4#2M7r3g2J5M7s2u0G2K9X3g2U0N6q4)9J5y4e0u0r3L8h3q4A6L8W2)9J5y4e0u0r3i4K6t1#2x3V1k6E0j5h3W2F1i4K6t1#2x3@1k6D9j5h3&6Y4i4K6t1#2x3@1c8U0i4K6t1#2x3U0f1J5b7W2)9J5y4e0t1#2x3V1u0Q4x3U0f1K6c8Y4m8S2N6r3S2Q4x3U0f1K6c8r3q4J5N6q4)9J5y4e0u0r3L8r3W2T1L8X3q4@1K9i4k6W2j5Y4u0A6k6r3N6W2i4K6t1#2x3V1k6F1j5i4c8A6N6X3g2Q4y4h3k6T1M7X3W2V1k6$3g2Q4x3X3g2U0j5#2)9J5y4e0t1K6h3s2y4H3g2V1#2i4i4K6g2X3h3V1!0V1i4K6g2X3N6V1k6A6d9q4A6o6g2Y4c8o6P5e0W2b7x3i4g2S2L8h3y4m8k6U0g2@1g2h3y4K9j5i4A6f1f1o6c8F1d9r3x3`.)  
看看 CloseNativeBridge

```
......
// Whether we had an error at some point.
static bool had_error = false;
......
 
static void CloseNativeBridge(bool with_error) {
  state = NativeBridgeState::kClosed;
  had_error |= with_error;
  ReleaseAppCodeCacheDir();
}
 
.........
 
bool NativeBridgeError() {
  return had_error;
}
```

`had_error`关键的值, 只要设置为 true, 就再也不会是 false 了  
`NativeBridgeError`会返回`had_error`  
写个代码试下:

```
SandHook::ElfImg nativeBridge("libnativebridge.so");
using NativeBridgeError_t = bool();
auto *NativeBridgeError = nativeBridge.getSymbAddress("NativeBridgeError");
LOGI("NativeBridgeError[%p]=%d", NativeBridgeError, NativeBridgeError());
 
bool *nbm = (bool *) ((uintptr_t) nativeBridge.getBase() + 0x5288);
*nbm = 0;
LOGI("NativeBridgeError, from memory[%p]=%d", nbm, *nbm); 
```

`这个0x5288是从IDA写复制来的`  
![](https://bbs.kanxue.com/upload/attach/202504/98159_DD62TV478HQ29WQ.webp)  
通过`resetprop ro.dalvik.vm.native.bridge [0/xxx]`再次测试, **确认是这个检测点**

```
20:23:53.066 10026-10061 NativeGuard   I  dl_iterate_phdr, get module base /apex/com.android.art/lib64/libnativebridge.so: 0x7b13725000
20:23:53.066 10026-10061 NativeGuard   I  NativeBridgeError[0x7b13727ef0]=1
20:23:53.066 10026-10061 NativeGuard   I  NativeBridgeError, from memory[0x7b1372a288]=0
20:23:53.066 10026-10061 NativeGuard   I  NativeBridgeError check[0x7b13727ef0]=0
```

修复
--

那这个`0x5288`怎么搞出来呢? had_error 是个 static, 没有符号, NativeBridgeError 返回的也是值, 不是指针, 那只能得到 NativeBridgeError 指针后, 拿到汇编 hex, 再去转换汇编码, 得到地址, 修复数据, 说干就干, 在线测试: https://shell-storm.org/online/Online-Assembler-and-Disassembler/?opcodes=08+00+00+F0+00+21+4A+39&arch=arm64&endianness=little&baddr=0x00000000&dis_with_addr=True&dis_with_raw=True&dis_with_ins=True#disassembly

### 为什么要放弃

> 太难

### 别的方法?

![](https://bbs.kanxue.com/upload/attach/202504/98159_KJS7R3B87EFYBPK.webp)  
这个值好像正好在. bss 块的头上, 直接尝试读取. bss 的指针试下  
![](https://bbs.kanxue.com/upload/attach/202504/98159_RC475AHBEYE435K.webp)

> 这里偷懒一下, 正常应该通过文件获取`ehdr`指针, 再获取. bss 块的首地址的

```
// fix native bridge detection
void hideNativeBridgeError() {
    // fix native bridge detection
    dl_iterate_phdr([](struct dl_phdr_info *info, size_t map_size, void *data) -> int {
        if (strstr(info->dlpi_name, "libnativebridge.so")) {
            for (int i = 0; i < info->dlpi_phnum; i++) {
                auto phdr = &info->dlpi_phdr[i];
                if (phdr->p_type == PT_LOAD && (phdr->p_flags & PF_W) && (phdr->p_flags & PF_R) && phdr->p_filesz == 0) {
                    LOGI("Riru found NativeBridge.had_error: 0x%lx", (off_t) phdr->p_vaddr);
                    void *nb_point = reinterpret_cast(info->dlpi_addr);
                    bool *nb_error_point = reinterpret_cast(info->dlpi_addr + phdr->p_vaddr);
                    LOGI("Riru NativeBridge: %p, NativeBridgeError: %p ==> %d", nb_point, nb_error_point, *nb_error_point);
                    if (*nb_error_point != 0) {
                        *nb_error_point = 0;
                        LOGI("Riru fix NativeBridgeError: %p ==> %d", nb_error_point, *nb_error_point);
                    }
                    break;
                }
            }
            return 1;
        }
        return 0;
    }, nullptr);
} 
```

放进 RIRU 测试下

```
...
19:45:37.350   705-705   Riru64        D  jniRegisterNativeMethods com/android/internal/os/Zygote
19:45:37.350   705-705   Riru64        I  Riru found NativeBridge.had_error: 0x5288
19:45:37.350   705-705   Riru64        I  Riru NativeBridge: 0x7dbb1a9000, NativeBridgeError: 0x7dbb1ae288 ==> 1
19:45:37.350   705-705   Riru64        I  Riru fix NativeBridgeError: 0x7dbb1ae288 ==> 0
19:45:37.350   705-705   Riru64        I  replaced com.android.internal.os.Zygote#nativeForkAndSpecialize
...
```

这个注意下, 要 dlopebnn 执行完后再修复, 一般放在`onRegisterZygote`方法里

打开了, 也结束了
---------

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-173.htm)

[#逆向分析](forum-161-1-118.htm) [#HOOK 注入](forum-161-1-125.htm) [#工具脚本](forum-161-1-128.htm)
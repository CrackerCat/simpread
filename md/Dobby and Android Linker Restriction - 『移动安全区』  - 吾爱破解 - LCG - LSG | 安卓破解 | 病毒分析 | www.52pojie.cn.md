> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/forum.php?mod=viewthread&tid=1232364&page=1&extra=#pid33218773) ![](https://avatar.52pojie.cn/data/avatar/000/74/77/09_avatar_middle.jpg)jmpews  _ 本帖最后由 jmpews 于 2020-7-28 20:37 编辑_  

Dobby and Linker Android Restriction
------------------------------------

> Google Pixel 2, Android 10, ARM64

#### Prologue

看到很多文章在讲, 然后可能最近也需要, 就抽空实现了下.

对 Android 不熟悉, 可能存在实现不合理 / 不恰当 / 错误的地方, 欢迎提出各种意见 / 批评, 非常感谢.

(markdown 编辑器不识别语言标记导致排版乱掉

[Dobby/builtin-plugin/AndroidRestriction/android_restriction.cc](https://github.com/jmpews/Dobby/blob/master/builtin-plugin/AndroidRestriction/android_restriction.cc)

[Dobby/SymbolResolver/elf/dobby_symbol_resolver.cc](https://github.com/jmpews/Dobby/blob/master/builtin-plugin/SymbolResolver/elf/dobby_symbol_resolver.cc)

#### 0xff. process module map

```
static std::vector<RuntimeModule> get_process_map_with_linker_iterator() {
  dl_iterate_phdr(
      [](dl_phdr_info *info, size_t size, void *data) {
        RuntimeModule module = {0};
        if (info->dlpi_name && info->dlpi_name[0] == '/')
          strcpy(module.path, info->dlpi_name);
        module.load_address = (void *)info->dlpi_addr;c
        ProcessModuleMap.push_back(module);
        return 0;
      },
      NULL);

  return ProcessModuleMap;
}

std::vector<RuntimeModule> ProcessRuntimeUtility::GetProcessModuleMap() {
  if (!ProcessMemoryLayout.empty()) {
    ProcessMemoryLayout.clear();
  }
  return get_process_map_with_linker_iterator();
}

```

[go to detail](https://github.com/jmpews/Dobby/blob/master/source/UserMode/PlatformUtil/Linux/ProcesssRuntimeUtility.cc#L98)

#### 0x0. linker solist

```
std::vector<soinfo_t> linker_solist;
std::vector<soinfo_t> linker_get_solist() {
  if (!linker_solist.empty()) {
    linker_solist.clear();
  }

  static soinfo_t (*solist_get_head)() = NULL;
  if (!solist_get_head)
    solist_get_head = (soinfo_t(*)())resolve_elf_internal_symbol(LINKER_PATH, "__dl__Z15solist_get_headv");

  static soinfo_t (*solist_get_somain)() = NULL;
  if (!solist_get_somain)
    solist_get_somain = (soinfo_t(*)())resolve_elf_internal_symbol(LINKER_PATH, "__dl__Z17solist_get_somainv");

  static addr_t *solist_head = NULL;
  if (!solist_head)
    solist_head = (addr_t *)solist_get_head();

  static addr_t somain = 0;
  if (!somain)
    somain = (addr_t)solist_get_somain();

    // Generate the name for an offset.
#define PARAM_OFFSET(type_, member_) __##type_##__##member_##__offset_
#define STRUCT_OFFSET PARAM_OFFSET
  int STRUCT_OFFSET(solist, next) = 0;
  for (size_t i = 0; i < 16; i++) {
    if (*(addr_t *)((addr_t)solist_head + i * 8) == somain) {
      STRUCT_OFFSET(solist, next) = i * 8;
      break;
    }
  }

  linker_solist.push_back(solist_head);

  addr_t sonext = 0;
  sonext        = *(addr_t *)((addr_t)solist_head + STRUCT_OFFSET(solist, next));
  while (sonext) {
    linker_solist.push_back((void *)sonext);
    sonext = *(addr_t *)((addr_t)sonext + STRUCT_OFFSET(solist, next));
  }

  return linker_solist;
}

```

[go to detail](https://github.com/jmpews/Dobby/blob/master/builtin-plugin/AndroidRestriction/android_restriction.cc#L49)

#### 0x1. symbol resolver

```
// impl at "android_restriction.cc"
extern std::vector<void *> linker_get_solist();
void *DobbySymbolResolver(const char *image_name, const char *symbol_name_pattern) {
  void *result = NULL;

  auto solist = linker_get_solist();
  for (auto soinfo : solist) {
    uintptr_t handle = linker_soinfo_to_handle(soinfo);
    if (image_name == NULL || strstr(linker_soinfo_get_realpath(soinfo), image_name) != 0) {
      DLOG("DobbySymbolResolver::dlsym: %s", linker_soinfo_get_realpath(soinfo));
      result = dlsym((void *)handle, symbol_name_pattern);
      if (result)
        return result;
    }
  }

  result = resolve_elf_internal_symbol(image_name, symbol_name_pattern);
  return result;
}

```

```
void *__loader_dlopen = DobbySymbolResolver(NULL, "__loader_dlopen");
DobbyHook((void *)__loader_dlopen, (void *)fake_loader_dlopen, (void **)&orig_loader_dlopen);

```

[go to detail](https://github.com/jmpews/Dobby/blob/master/builtin-plugin/SymbolResolver/elf/dobby_symbol_resolver.cc#L145)

#### 0x2. hook dlopen

```
void *__loader_dlopen = DobbySymbolResolver(NULL, "__loader_dlopen");
DobbyHook((void *)__loader_dlopen, (void *)fake_loader_dlopen, (void **)&orig_loader_dlopen);

```

[go to detail](https://github.com/jmpews/Dobby/blob/master/builtin-plugin/ApplicationEventMonitor/dynamic_loader_monitor.cc#L83)

#### 0x3. linker dlopen(fake caller address)

```
#if defined(__LP64__)
  lib = "/system/lib64/libandroid_runtime.so";
#else
  lib          = "/system/lib/libandroid_runtime.so";
#endif
void *handle = NULL;
handle       = linker_dlopen(lib, RTLD_LAZY);
void *vm;
vm = dlsym(handle, "_ZN7android14AndroidRuntime7mJavaVME");

```

[go to detail](https://github.com/jmpews/Dobby/blob/master/builtin-plugin/AndroidRestriction/android_restriction.cc#L38)

#### 0x4. disable android namespace restriction

```
void linker_disable_namespace_restriction() {
  linker_iterate_soinfo(iterate_soinfo_cb);

  // no need for this actually
  void *linker_namespace_is_is_accessible_ptr =
      resolve_elf_internal_symbol(LINKER_PATH, "__dl__ZN19android_namespace_t13is_accessibleERKNSt3__112basic_"
                                               "stringIcNS0_11char_traitsIcEENS0_9allocatorIcEEEE");
  DobbyHook(linker_namespace_is_is_accessible_ptr, (void *)linker_namespace_is_is_accessible,
            (void **)&orig_linker_namespace_is_is_accessible);

  LOG("disable namespace restriction done");
}

```

[go to detail](https://github.com/jmpews/Dobby/blob/master/builtin-plugin/AndroidRestriction/android_restriction.cc#L166)

#### Reference

[frida](https://github.com/frida/frida-gum/blob/master/gum/backend-linux/gumandroid.c)

[quarkslab](https://github.com/quarkslab/android-restriction-bypass)

![](https://avatar.52pojie.cn/data/avatar/000/74/77/09_avatar_middle.jpg)jmpews 

> [FraMeQ 发表于 2020-8-5 13:44](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=33417874&ptid=1232364)  
> 我想知道现在 Android 稳定吗。。之前 Android 编译都编译不通过，还得只能能用老 commit 的 HookZz

  
稳了... 哈哈哈哈哈 之前我不太关注 Android![](https://avatar.52pojie.cn/data/avatar/000/37/13/87_avatar_middle.jpg)FraMeQ  我想知道现在 Android 稳定吗。。之前 Android 编译都编译不通过，还得只能能用老 commit 的 HookZz ![](https://avatar.52pojie.cn/data/avatar/000/76/89/11_avatar_middle.jpg) 空白菌 活抓 zz  
活抓 zz  
活抓 zz![](https://avatar.52pojie.cn/data/avatar/000/93/53/52_avatar_middle.jpg)xixicoco  完全没看懂啊 ![](https://avatar.52pojie.cn/data/avatar/001/44/03/12_avatar_middle.jpg) mzrme 完全没看懂啊 ![](https://avatar.52pojie.cn/data/avatar/000/22/28/08_avatar_middle.jpg) a6608816 这得是一定编程基础的才看得懂吧 ![](https://avatar.52pojie.cn/data/avatar/001/34/29/87_avatar_middle.jpg) xy20200214 Dobby a lightweight, multi-platform, multi-architecture exploit hook framework. ![](https://avatar.52pojie.cn/data/avatar/001/45/75/01_avatar_middle.jpg) 留住那片心 全是英文看不懂![](https://avatar.52pojie.cn/data/avatar/000/35/53/29_avatar_middle.jpg)海尔波普彗星  英文学的好
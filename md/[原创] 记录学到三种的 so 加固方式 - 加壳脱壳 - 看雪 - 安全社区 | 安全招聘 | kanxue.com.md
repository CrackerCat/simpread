> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-286878.htm)

> [原创] 记录学到三种的 so 加固方式

简单分享一下学习到的 So 加固的方案，前前后后实现差不多用了两周的时间，实践下来也学到了很多关于 So 的知识。如果文章中有讲述错误的地方，欢迎各位大佬斧正，谢谢。本文章的代码基于[关于 SO 加密对抗的两种实现方式](https://bbs.kanxue.com/thread-285650.htm)  
在看本篇文章之前，最好需要了解一下 ELF 文件格式，以及 So 的加载流程，这里推荐 oacia 大佬的两篇文章。[ELF 结构分析及 ElfReader](elink@7d2K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6G2j5h3y4A6j5g2)9J5k6h3c8W2N6W2)9J5c8V1g2D9k6W2u0W2j5h3c8W2M7W2)9J5c8R3`.`.) 和[安卓 so 加载流程源码分析](elink@0edK9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6G2j5h3y4A6j5g2)9J5k6h3c8W2N6W2)9J5c8X3q4F1k6s2u0G2K9h3c8Q4x3X3c8D9L8$3q4V1i4K6u0V1M7$3!0Q4x3V1j5`.)。

1. 第一种加密方式：加密函数
---------------

下面是编译为 libmathlib.so 前的源代码，我们将要加密`int mymyadd(int a, int b)`

```
#include <android/log.h>  
  
// 定义日志标签  
#define LOG_TAG "MathLib"  
#define LOGD(...) __android_log_print(ANDROID_LOG_DEBUG, LOG_TAG, __VA_ARGS__)  
  
extern "C"  
int mymyadd(int a, int b) {  
    int result = a + b;  
    LOGD("Add: %d + %d = %d", a, b, result);  
    return result;  
}  
extern "C"  
int subtract(int a, int b) {  
    int result = a - b;  
    LOGD("Subtract: %d - %d = %d", a, b, result);  
    return result;  
}
```

### 1.1. 加密效果

未加密前  
![](https://bbs.kanxue.com/upload/attach/202505/998309_FK54ZVHGBR7NHH8.png)  
加密后，IDA 自然无法正确识别出函数  
![](https://bbs.kanxue.com/upload/attach/202505/998309_NN8TAW6TJSMXMFE.png)

### 1.2. 实现原理

加密函数，首先自然要从 ELF 文件中找到函数的位置以及函数的大小。  
这里看一下源码中 dlsym 函数怎么处理的。  
调用了`__loader_dlsym(handle, symbol, caller_addr)`

```
// bionic/libdl/libdl.cpp
void* dlsym(void* handle, const char* symbol) {
  const void* caller_addr = __builtin_return_address(0);
  return __loader_dlsym(handle, symbol, caller_addr);
}
```

调用了`dlsym_impl(handle, symbol, nullptr, caller_addr);`

```
// bionic/linker/dlfcn.cpp
void* __loader_dlsym(void* handle, const char* symbol, const void* caller_addr) {
  return dlsym_impl(handle, symbol, nullptr, caller_addr);
}
```

调用了`do_dlsym(handle, symbol, version, caller_addr, &result)`

```
// bionic/linker/dlfcn.cpp
void* dlsym_impl(void* handle, const char* symbol, const char* version, const void* caller_addr) {
  ScopedPthreadMutexLocker locker(&g_dl_mutex);
  g_linker_logger.ResetState();
  void* result;
  if (!do_dlsym(handle, symbol, version, caller_addr, &result)) {
    __bionic_format_dlerror(linker_get_error_buffer(), nullptr);
    return nullptr;
  }
 
  return result;
}
```

`if (handle == RTLD_DEFAULT || handle == RTLD_NEXT)`是判断特殊句柄，所以我们的情况走的是`else`语句。那么调用了`dlsym_handle_lookup(si, &found, sym_name, vi)`

```
// bionic/linker/linker.cpp
bool do_dlsym(void* handle,
              const char* sym_name,
              const char* sym_ver,
              const void* caller_addr,
              void** symbol) {
  ScopedTrace trace("dlsym");
#if !defined(__LP64__)
  if (handle == nullptr) {
    DL_SYM_ERR("dlsym failed: library handle is null");
    return false;
  }
#endif
 
  soinfo* found = nullptr;
  const ElfW(Sym)* sym = nullptr;
  soinfo* caller = find_containing_library(caller_addr);
  android_namespace_t* ns = get_caller_namespace(caller);
  soinfo* si = nullptr;
  if (handle != RTLD_DEFAULT && handle != RTLD_NEXT) {
    si = soinfo_from_handle(handle);
  }
 
  LD_LOG(kLogDlsym,
         "dlsym(handle=%p(\"%s\"), sym_name=\"%s\", sym_ver=\"%s\", caller=\"%s\", caller_ns=%s@%p) ...",
         handle,
         si != nullptr ? si->get_realpath() : "n/a",
         sym_name,
         sym_ver,
         caller == nullptr ? "(null)" : caller->get_realpath(),
         ns == nullptr ? "(null)" : ns->get_name(),
         ns);
 
  auto failure_guard = android::base::make_scope_guard(
      [&]() { LD_LOG(kLogDlsym, "... dlsym failed: %s", linker_get_error_buffer()); });
 
  if (sym_name == nullptr) {
    DL_SYM_ERR("dlsym failed: symbol name is null");
    return false;
  }
 
  version_info vi_instance;
  version_info* vi = nullptr;
 
  if (sym_ver != nullptr) {
    vi_instance.name = sym_ver;
    vi_instance.elf_hash = calculate_elf_hash(sym_ver);
    vi = &vi_instance;
  }
 
  if (handle == RTLD_DEFAULT || handle == RTLD_NEXT) {
    sym = dlsym_linear_lookup(ns, sym_name, vi, &found, caller, handle);
  } else {
    if (si == nullptr) {
      DL_SYM_ERR("dlsym failed: invalid handle: %p", handle);
      return false;
    }
    sym = dlsym_handle_lookup(si, &found, sym_name, vi);
  }
 
  if (sym != nullptr) {
    uint32_t bind = ELF_ST_BIND(sym->st_info);
    uint32_t type = ELF_ST_TYPE(sym->st_info);
 
    if ((bind == STB_GLOBAL || bind == STB_WEAK) && sym->st_shndx != 0) {
      if (type == STT_TLS) {
        // For a TLS symbol, dlsym returns the address of the current thread's
        // copy of the symbol.
        const soinfo_tls* tls_module = found->get_tls();
        if (tls_module == nullptr) {
          DL_SYM_ERR("TLS symbol \"%s\" in solib \"%s\" with no TLS segment",
                     sym_name, found->get_realpath());
          return false;
        }
        void* tls_block = get_tls_block_for_this_thread(tls_module, /*should_alloc=*/true);
        *symbol = static_cast(tls_block) + sym->st_value;
      } else {
        *symbol = reinterpret_cast(found->resolve_symbol_address(sym));
      }
      failure_guard.Disable();
      LD_LOG(kLogDlsym,
             "... dlsym successful: sym_name=\"%s\", sym_ver=\"%s\", found in=\"%s\", address=%p",
             sym_name, sym_ver, found->get_soname(), *symbol);
      return true;
    }
 
    DL_SYM_ERR("symbol \"%s\" found but not global", symbol_display_name(sym_name, sym_ver).c_str());
    return false;
  }
 
  DL_SYM_ERR("undefined symbol: %s", symbol_display_name(sym_name, sym_ver).c_str());
  return false;
} 
```

进入 if 语句也是调用了`dlsym_handle_lookup_impl`

```
// linker/linker.cpp
static const ElfW(Sym)* dlsym_handle_lookup(soinfo* si,
                                            soinfo** found,
                                            const char* name,
                                            const version_info* vi) {
  // According to man dlopen(3) and posix docs in the case when si is handle
  // of the main executable we need to search not only in the executable and its
  // dependencies but also in all libraries loaded with RTLD_GLOBAL.
  //
  // Since RTLD_GLOBAL is always set for the main executable and all dt_needed shared
  // libraries and they are loaded in breath-first (correct) order we can just execute
  // dlsym(RTLD_DEFAULT, ...); instead of doing two stage lookup.
  if (si == solist_get_somain()) {
    return dlsym_linear_lookup(&g_default_namespace, name, vi, found, nullptr, RTLD_DEFAULT);
  }
 
  SymbolName symbol_name(name);
  // note that the namespace is not the namespace associated with caller_addr
  // we use ns associated with root si intentionally here. Using caller_ns
  // causes problems when user uses dlopen_ext to open a library in the separate
  // namespace and then calls dlsym() on the handle.
  return dlsym_handle_lookup_impl(si->get_primary_namespace(), si, nullptr, found, symbol_name, vi);
}
```

调用了`current_soinfo->find_symbol_by_name(symbol_name, vi);`

```
// bionic/linker/linker.cpp
static const ElfW(Sym)* dlsym_handle_lookup_impl(android_namespace_t* ns,
                                                 soinfo* root,
                                                 soinfo* skip_until,
                                                 soinfo** found,
                                                 SymbolName& symbol_name,
                                                 const version_info* vi) {
  const ElfW(Sym)* result = nullptr;
  bool skip_lookup = skip_until != nullptr;
 
  walk_dependencies_tree(root, [&](soinfo* current_soinfo) {
    if (skip_lookup) {
      skip_lookup = current_soinfo != skip_until;
      return kWalkContinue;
    }
 
    if (!ns->is_accessible(current_soinfo)) {
      return kWalkSkip;
    }
 
    result = current_soinfo->find_symbol_by_name(symbol_name, vi);
    if (result != nullptr) {
      *found = current_soinfo;
      return kWalkStop;
    }
 
    return kWalkContinue;
  });
 
  return result;
}
```

这里根据设置了`FLAG_GNU_HASH`标志位选择使用`GNU哈希查找`还是`ELF哈希查找`。GNU 哈希是一种更现代、更高效的符号查找方法，特别针对大型库进行了优化。

```
// bionic/linker/linker_soinfo.cpp
const ElfW(Sym)* soinfo::find_symbol_by_name(SymbolName& symbol_name,
                                             const version_info* vi) const {
  return is_gnu_hash() ? gnu_lookup(symbol_name, vi) : elf_lookup(symbol_name, vi);
}
```

这里我将两个函数都粘贴出来，我们选择学习`GNU哈希查找`

```
// bionic/linker/linker_soinfo.cpp
const ElfW(Sym)* soinfo::gnu_lookup(SymbolName& symbol_name, const version_info* vi) const {
  const uint32_t hash = symbol_name.gnu_hash();
 
  constexpr uint32_t kBloomMaskBits = sizeof(ElfW(Addr)) * 8;
  const uint32_t word_num = (hash / kBloomMaskBits) & gnu_maskwords_;
  const ElfW(Addr) bloom_word = gnu_bloom_filter_[word_num];
  const uint32_t h1 = hash % kBloomMaskBits;
  const uint32_t h2 = (hash >> gnu_shift2_) % kBloomMaskBits;
 
  LD_DEBUG(lookup, "SEARCH %s in %s@%p (gnu)",
           symbol_name.get_name(), get_realpath(), reinterpret_cast(base));
 
  // test against bloom filter
  if ((1 & (bloom_word >> h1) & (bloom_word >> h2)) == 0) {
    return nullptr;
  }
 
  // bloom test says "probably yes"...
  uint32_t n = gnu_bucket_[hash % gnu_nbucket_];
 
  if (n == 0) {
    return nullptr;
  }
 
  const ElfW(Versym) verneed = find_verdef_version_index(this, vi);
  const ElfW(Versym)* versym = get_versym_table();
 
  do {
    ElfW(Sym)* s = symtab_ + n;
    if (((gnu_chain_[n] ^ hash) >> 1) == 0 &&
        check_symbol_version(versym, n, verneed) &&
        strcmp(get_string(s->st_name), symbol_name.get_name()) == 0 &&
        is_symbol_global_and_defined(this, s)) {
      return symtab_ + n;
    }
  } while ((gnu_chain_[n++] & 1) == 0);
 
  return nullptr;
} 
```

```
// bionic/linker/linker_soinfo.cpp
const ElfW(Sym)* soinfo::elf_lookup(SymbolName& symbol_name, const version_info* vi) const {
  uint32_t hash = symbol_name.elf_hash();
 
  LD_DEBUG(lookup, "SEARCH %s in %s@%p h=%x(elf) %zd",
           symbol_name.get_name(), get_realpath(),
           reinterpret_cast(base), hash, hash % nbucket_);
 
  const ElfW(Versym) verneed = find_verdef_version_index(this, vi);
  const ElfW(Versym)* versym = get_versym_table();
 
  for (uint32_t n = bucket_[hash % nbucket_]; n != 0; n = chain_[n]) {
    ElfW(Sym)* s = symtab_ + n;
 
    if (check_symbol_version(versym, n, verneed) &&
        strcmp(get_string(s->st_name), symbol_name.get_name()) == 0 &&
        is_symbol_global_and_defined(this, s)) {
      return symtab_ + n;
    }
  }
 
  return nullptr;
} 
```

根据函数得知，需要计算得到`gnu_hash`以及其他例如布隆过滤器掩码字数`gnu_maskwords_`，布隆过滤器数组`gnu_bloom_filter_`，符号表地址`symtab_`等。  
其中`gnu_hash`值简单，只需要按照下面的函数计算即可

```
static uint32_t gnu_hash(const char *s0)
{
    const unsigned char *s = (void *)s0;
    uint_fast32_t h = 5381;
    for (; *s; s++)
        h += h*32 + *s;
    return h;
}
```

至于`gnu_maskwords_`，`gnu_bloom_filter_`，符号表地址`symtab_`等  
按照源码中的提示，就更好解决了。通过`ELF Header`定位到`Program Header Table`，遍历`Program Header Table`，找到类型为`PT_DYNAMIC`的动态段的偏移量和大小，这个段包含了动态链接的关键信息。GNU 哈希表地址和符号表地址都可以通过分析动态段中的条目得到

```
// bionic/linker/linker.cpp
case DT_GNU_HASH:
        gnu_nbucket_ = reinterpret_cast(load_bias + d->d_un.d_ptr)[0];
        // skip symndx
        gnu_maskwords_ = reinterpret_cast(load_bias + d->d_un.d_ptr)[2];
        gnu_shift2_ = reinterpret_cast(load_bias + d->d_un.d_ptr)[3];
 
        gnu_bloom_filter_ = reinterpret_cast(load_bias + d->d_un.d_ptr + 16);
        gnu_bucket_ = reinterpret_cast(gnu_bloom_filter_ + gnu_maskwords_);
        // amend chain for symndx = header[1]
        gnu_chain_ = gnu_bucket_ + gnu_nbucket_ -
            reinterpret_cast(load_bias + d->d_un.d_ptr)[1];
 
        if (!powerof2(gnu_maskwords_)) {
          DL_ERR("invalid maskwords for gnu_hash = 0x%x, in \"%s\" expecting power to two",
              gnu_maskwords_, get_realpath());
          return false;
        }
        --gnu_maskwords_;
 
        flags_ |= FLAG_GNU_HASH;
        break; 
```

### 1.3. 实现代码

具体分析已经完毕，实现代码如下

```
int harden_handle(char *src_path, char *name, char *dst_path, int mode) {
    Elf64_Ehdr header_;
    Elf64_Phdr phdr_;
    Elf64_Dyn dyn_;
    int dyn_off;
    int dyn_size;
    int dyn_count;
    Elf64_Addr dyn_symtab;
    Elf64_Addr dyn_strtab;
    Elf64_Addr dyn_gnuhash;
    int dyn_strsz;
    uint32_t symndex;
    // GNU哈希表相关变量
    uint32_t gnu_nbucket_;
    uint32_t gnu_maskwords_;
    uint32_t gnu_shift2_;
    Elf64_Addr *gnu_bloom_filter_;
    uint32_t* gnu_bucket_;
    uint32_t* gnu_chain_;
    bool has_gnu_hash = false;
 
    // 打开源文件
    int fd = open(src_path, O_RDONLY);
    if (fd == -1) {
        LOGE("error opening source file");
 
        return -1;
    }
 
    //读elf头
    int ret = read(fd, &header_, sizeof(header_));
    if (ret < 0) {
        LOGE("error read file!");
    }
 
    //读程序头
    lseek(fd, header_.e_phoff, SEEK_SET);
    for (int i = 0; i <  header_.e_phnum; i++) {
        ret = read(fd, &phdr_, sizeof(phdr_));
        if (ret < 0) {
            LOGE("error read file!");
        }
 
        if (phdr_.p_type != PT_DYNAMIC) {
            continue;
        }
        dyn_off = phdr_.p_offset;
        dyn_size = phdr_.p_filesz;
        dyn_count = phdr_.p_memsz / (8 * 2);
    }
 
    //读动态段
    lseek(fd, dyn_off, SEEK_SET);
    for(int i = 0; i < dyn_count; i++) {
        ret = read(fd, &dyn_, sizeof(dyn_));
        if (ret < 0) {
            LOGI("error read file!");
        }
 
        switch (dyn_.d_tag) {
            case DT_SONAME:
                break;
            case DT_GNU_HASH:
                dyn_gnuhash = dyn_.d_un.d_ptr;
                break;
            case DT_SYMTAB:
                dyn_symtab =  dyn_.d_un.d_ptr;
                break;
            case DT_SYMENT:
                break;
            case DT_STRTAB:
                dyn_strtab =  dyn_.d_un.d_ptr;
                break;
            case DT_STRSZ:
                dyn_strsz = dyn_.d_un.d_val;
                break;
        }
    }
 
    char *dynstr = (char*) malloc(dyn_strsz);
    if(dynstr == NULL){
        LOGE("malloc failed");
    }
 
    lseek(fd, dyn_strtab, SEEK_SET);
    ret = read(fd, dynstr, dyn_strsz);
    if (ret < 0) {
        LOGE("read .dynstr failed");
    }
 
    //Gnu Hash桶查找法
    gnu_nbucket_ = dyn_gnuhash[0];
    uint32_t symndx = dyn_gnuhash[1];
    gnu_maskwords_ = dyn_gnuhash[2];
    gnu_shift2_ = dyn_gnuhash[3];
 
    gnu_bloom_filter_ = reinterpret_cast(dyn_gnuhash + 16);//布隆滤波器地址
    gnu_bucket_ = reinterpret_cast(gnu_bloom_filter_ + gnu_maskwords_);//bucket起始地址
    gnu_chain_ = gnu_bucket_ + gnu_nbucket_ - symndx;//chain起始地址减去index偏差
 
    uint32_t hash = gnu_hash(name);
    uint32_t h2 = hash >> gnu_shift2_;
    uint32_t bloom_mask_bits = sizeof(Elf64_Addr) * 8;
    uint32_t word_num = (hash / bloom_mask_bits) & gnu_maskwords_;
 
    uint32_t val = hash % gnu_nbucket_;
 
    // 检查哈希桶
    uint32_t n = gnu_bucket_[hash % gnu_nbucket_];
    if (n == 0) {
        printf("符号'%s'所在哈希桶为空\n", name);
        return false;
    }
 
    Elf64_Sym s;
    uint32_t chain;
    do {
        lseek(fd, dyn_symtab + n * sizeof(Elf64_Sym), SEEK_SET);
        ret = read(fd, &s, sizeof(Elf64_Sym));
        if (ret < 0) {
            LOGI("read gnuhash failed");
        }
        LOGI("name = %d %s", s.st_name, dynstr + s.st_name);
 
        lseek(fd, reinterpret_cast(gnu_chain_ + n), SEEK_SET);
        ret = read(fd, &chain, sizeof(chain));
        if (ret < 0) {
            LOGI("read gnuhash failed");
        }
 
        if (((chain ^ hash) >> 1) == 0  && strcmp(dynstr + s.st_name, name) == 0) {
            LOGI("found function(%s) at %p(%zd)", name, reinterpret_cast(s.st_value), static_cast(s.st_size));
            break;
        }
 
        n++;
        lseek(fd, reinterpret_cast(gnu_chain_ + n), SEEK_SET);
        ret = read(fd, &chain, sizeof(chain));
        if (ret < 0) {
            LOGI("read gnuhash failed");
        }
    } while((chain & 1) == 0);
 
    uint32_t  size = get_file_size(src_path);
    char *file_buf = (char *)malloc(size);
    if (file_buf == NULL) {
        LOGI("file buf malloc failed");
    }
 
    lseek(fd, 0, SEEK_SET);
    ret = read(fd, file_buf, size);
    if (ret < 0) {
        LOGI("read file buf failed");
    }
 
    close(fd);
 
    //so修改与保存
    char save_path[128] = {0};
    fd = open(dst_path, O_RDWR | O_CREAT, 0644);
    if (fd == -1) {
        LOGI("error opening file, %s", dst_path);
        return -1;
    }
    char *encrypt_buf = (char *)malloc(s.st_size);
 
    encrypt((unsigned char *) RC4_KEY, reinterpret_cast(encrypt_buf),
            reinterpret_cast(&file_buf[s.st_value]), FUNC_SIZE);
    memcpy(&file_buf[s.st_value], encrypt_buf,  FUNC_SIZE);
    if (mode == MODE_DUPLEX) {
        for (int i = 0; i < header_.e_phnum; i++) {
            file_buf[header_.e_phoff + i * sizeof(phdr_)] = file_buf[header_.e_phoff + i * sizeof(phdr_)] ^ XOR_MAGIC;
            file_buf[header_.e_phoff + i * sizeof(phdr_) + 1] = file_buf[header_.e_phoff + i * sizeof(phdr_) + 1] ^ XOR_MAGIC;
            file_buf[header_.e_phoff + i * sizeof(phdr_) + 2] = file_buf[header_.e_phoff + i * sizeof(phdr_) + 2] ^ XOR_MAGIC;
            file_buf[header_.e_phoff + i * sizeof(phdr_) + 3] = file_buf[header_.e_phoff + i * sizeof(phdr_) + 3] ^ XOR_MAGIC;
        }
    }
    write(fd, file_buf, size);
    free(encrypt_buf);
    close(fd);
 
    return 0;
} 
```

### 1.4. 调用流程

注意这里原作者定义了`FUNC_SIZE`常量，如果不想这么做就得跟加密一样去解析 ELF 文件找动态段来遍历获取函数大小。  
1.`dlopen`打开库文件用`dlsym`获取函数地址  
2. 修改内存权限，解密覆盖函数地址范围，恢复内存权限  
3. 调用函数运行  
4. 修改内存权限，加密覆盖函数地址范围，恢复内存权限

```
extern "C"
JNIEXPORT void JNICALL
Java_com_example_mylinkerwith21_MainActivity_loadSO(JNIEnv *env, jobject thiz,
                                                    jstring jEncryptedFileDirectoryPath) {
    void *lib;
    uint64_t start;
    uint64_t end;
    ssize_t page;
    FUNC lib_func = NULL;
    RC4_CTX ctx;
    char buf[512] = {0};
    int result;
 
    LOGI("loadSO called!");
 
    const char *encryptedFileDirectoryPath = env->GetStringUTFChars(jEncryptedFileDirectoryPath, nullptr);
 
    if (encryptedFileDirectoryPath == nullptr) {
        LOGE("Failed to get string characters for encrypted file directory path");
        return;
    }
 
    // 构建加密文件的完整加载路径 (从私有目录)
    const char* encryptedFileName = "libmathlib_encrypt.so";
    // 计算所需缓冲区大小: 目录路径长度 + 1 (斜杠) + 文件名长度 + 1 (null terminator)
    size_t requiredPathSize = strlen(encryptedFileDirectoryPath) + 1 + strlen(encryptedFileName) + 1;
    char* encryptedLoadPath = (char*)malloc(requiredPathSize);
 
    if (encryptedLoadPath == nullptr) {
        LOGE("Failed to allocate memory for encrypted load path");
        env->ReleaseStringUTFChars(jEncryptedFileDirectoryPath, encryptedFileDirectoryPath);
        return; // 返回，内存分配失败
    }
 
    // 拼接路径
    sprintf(encryptedLoadPath, "%s/%s", encryptedFileDirectoryPath, encryptedFileName);
    LOGI("Loading from path: %s", encryptedLoadPath);
 
 
    // 使用构建的私有目录路径下的加密文件路径进行 dlopen
    lib = dlopen(encryptedLoadPath, RTLD_LAZY);
 
    // 释放动态分配的内存
    free(encryptedLoadPath);
 
    // 释放JNI字符串资源
    env->ReleaseStringUTFChars(jEncryptedFileDirectoryPath, encryptedFileDirectoryPath);
 
 
    if (lib == NULL) {
        LOGE("%s dlopen failed: %s\n", encryptedFileName, dlerror());
        return; // dlopen 失败，直接返回
    }
 
    // --- 后面的逻辑与原始代码类似，操作dlopen加载的库 ---
 
    //获取库函数
    lib_func = reinterpret_cast(dlsym(lib, "mymyadd"));
    if (!lib_func) {
        LOGI("can't find module symbol 'mymyadd': %s\n", dlerror());
        dlclose(lib); // dlsym 失败，关闭库
        return; // dlsym 失败，返回
    }
 
    LOGI("loadso lib_func = %p", lib_func);
 
    //修改内存读写权限
    // 注意：parse_maps_for_lib 函数需要根据加载的库名来查找，这里是 "libmathlib_encrypt.so"
    // 确保 parse_maps_for_lib 能够正确找到加载的库的内存映射
    parse_maps_for_lib("libmathlib_encrypt.so", &start, &end); // 使用加载的库名
 
    if (start == 0 && end == 0) {
        LOGI("Failed to find memory map for libmathlib_encrypt.so");
        dlclose(lib); // 查找内存映射失败，关闭库
        return;
    }
 
 
    page = end - start;
    // 确保 page 是页对齐的，mprotect 要求地址和大小都是页对齐的
    // 简单的示例省略了页对齐处理，实际生产代码需要考虑
    if (mprotect((void*)start, page, PROT_READ | PROT_WRITE | PROT_EXEC) == -1) {
        LOGI("mprotect failed for PROT_READ | PROT_WRITE | PROT_EXEC: %s", strerror(errno)); // 添加错误信息
        dlclose(lib); // mprotect 失败，关闭库
        return; // mprotect 失败，返回
    }
 
    //解密覆盖
    // 确保 RC4_KEY 和 FUNC_SIZE 在某个头文件中定义且可访问
    // 确保 buf 大小足够 FUNC_SIZE
    rc4_init(&ctx, (unsigned char *) RC4_KEY, strlen(reinterpret_cast(RC4_KEY)));
    rc4_run(&ctx, reinterpret_cast(buf), reinterpret_cast(lib_func), FUNC_SIZE);
    memcpy((void*)lib_func, buf, FUNC_SIZE);
 
    //权限还原
    if (mprotect((void*)start, page, PROT_READ | PROT_EXEC) == -1) {
        LOGI("mprotect failed for PROT_READ | PROT_EXEC: %s", strerror(errno)); // 添加错误信息
        // 即使还原权限失败，也要尝试调用函数，但记录错误
    }
 
    //调用库函数
    result = lib_func(1, 2);
    LOGI("loadso add result = %d", result);
 
    // --- 清理阶段 ---
 
    // 修改内存读写权限以便加密还原
    if (mprotect((void*)start, page, PROT_READ | PROT_WRITE | PROT_EXEC) == -1) {
        LOGI("mprotect failed for PROT_READ | PROT_WRITE | PROT_EXEC during re-encryption: %s", strerror(errno));
        // 如果这里失败了，后续的加密和还原权限操作可能也会失败，但是仍然尝试关闭库
    } else {
        //加密还原覆盖
        rc4_init(&ctx, (unsigned char *) RC4_KEY, strlen(reinterpret_cast(RC4_KEY)));
        rc4_run(&ctx, reinterpret_cast(buf), reinterpret_cast(lib_func), FUNC_SIZE);
        memcpy((void*)lib_func, buf, FUNC_SIZE);
 
        //权限还原
        if (mprotect((void*)start, page, PROT_READ | PROT_EXEC) == -1) {
            LOGI("mprotect failed for PROT_READ | PROT_EXEC after re-encryption: %s", strerror(errno));
        }
    }
 
 
    // 关闭库
    dlclose(lib);
} 
```

2. 第二种加密方式：异或段头的 type
---------------------

### 2.1. 加密效果

![](https://bbs.kanxue.com/upload/attach/202505/998309_BTJV9ABFQBAB5RQ.png)

### 2.2. 实现原理

在方式一的基础上，对每个段头的 type 进行了异或处理。

### 2.3. 实现代码

加密代码前面已经贴出来了，就是当传入参数`mode = MODE_DUPLEX`时参与的加密。该方法具体的调用流程可以看原作者的文章，自定义 linker 对 so 加载分析得非常好。  
只是这种方式的加密下，使用 IDA 仍能打开 so 文件静态分析，只修改了`Program Header Table`，而没有修改`Section Header Table`，所以 IDA 仍然能通过节头表获取大部分需要的信息。因此我在这里分享第三种 so 加密方法，自定义 ELF 文件格式，并且根据 android 源码自定义 linker 加载。

3. 第三种加密方式：自定义 ELF 文件格式
-----------------------

### 3.1. 加密效果

该方法加密后的 so 文件 IDA 肯定是无法解析的。  
![](https://bbs.kanxue.com/upload/attach/202505/998309_43GC9SJKS94TGA5.png)

左边部分为原本标准格式未加密 so 文件，右边为自定义格式且 RC4 加密后 so 文件  
![](https://bbs.kanxue.com/upload/attach/202505/998309_93ZSAAWWKPC8PFS.png)

左边部分为原本标准格式未加密 so 文件，右边为自定义格式且 RC4 解密后 so 文件  
![](https://bbs.kanxue.com/upload/attach/202505/998309_DJPCKVF6JF6Q3W5.png)

### 3.2. 实现代码

其中自定义的文件头`Custom_Elf64_Ehdr`自然可以不用与标准的文件头一样，这里我为了方便实现就没有变动。实际情况中可以调换顺序，调用时用计算的偏移，或者直接删除没有用到的部分都可以。`e_ident`也可以不只改魔数头，其他删掉都可以，毕竟 linker 都有我们自己实现了，也没必要叫校验了。

```
#pragma pack(push, 1) 
typedef struct custom_elf64_file{ 
    Elf64_Off   elf_header_off;  
    Elf64_Half  elf_header_size;  
    Elf64_Off   elf_program_header_table_off; 
    Elf64_Half  elf_program_header_table_num;  
    Elf64_Half  elf_program_header_table_size;  
}Custom_Elf64_File; 
#pragma pack(pop)
 
typedef struct 
{ 
    unsigned char   e_ident[EI_NIDENT];    /* Magic number and other info 16 bytes*/ 
    Elf64_Half  e_type;          /* Object file type 2 bytes*/ 
    Elf64_Half  e_machine;    /* Architecture 2 bytes*/ 
    Elf64_Word  e_version;    /* Object file version 4 bytes*/ 
    Elf64_Addr  e_entry;      /* Entry point virtual address 8 bytes*/ 
    Elf64_Off   e_phoff;      /* Program header table file offset 8 bytes*/ 
    Elf64_Off   e_shoff;      /* Section header table file offset */ 
    Elf64_Word  e_flags;      /* Processor-specific flags */ 
    Elf64_Half  e_ehsize;     /* ELF header size in bytes */ 
    Elf64_Half  e_phentsize;      /* Program header table entry size */ 
    Elf64_Half  e_phnum;      /* Program header table entry count */ 
    Elf64_Half  e_shentsize;      /* Section header table entry size */ 
    Elf64_Half  e_shnum;      /* Section header table entry count */ 
    Elf64_Half  e_shstrndx;       /* Section header string table index */ 
} Custom_Elf64_Ehdr;
 
int do_pack(char *inputfile_buffer, size_t inputfile_size, char *outfile_buffer, size_t outfile_size) 
{ 
    if (NULL == inputfile_buffer || 0 == inputfile_size || NULL == outfile_buffer) { 
        return -1; 
    } 
   
    // Validate input file is an ELF file 
    Elf64_Ehdr* orig_ehdr = (Elf64_Ehdr *)inputfile_buffer; 
   
    // 验证ELF魔数 
    if (memcmp(orig_ehdr->e_ident, ELFMAG, SELFMAG) != 0) { 
        return -2; 
    } 
   
    // Calculate sizes and offsets 
    size_t phdr_table_size = orig_ehdr->e_phnum * orig_ehdr->e_phentsize; 
    size_t enc_ehdr_offset = inputfile_size; 
    size_t enc_phdr_offset = enc_ehdr_offset + orig_ehdr->e_ehsize; 
    size_t final_size = enc_phdr_offset + phdr_table_size; 
   
    // Check if output buffer is large enough 
    if (final_size > outfile_size) { 
        return -3; 
    } 
   
    // 1. Copy original file to output buffer 
    memcpy(outfile_buffer, inputfile_buffer, inputfile_size); 
   
    // 2. Create custom ELF file header 
    Custom_Elf64_File custom_file = {0}; 
    custom_file.elf_header_off = enc_ehdr_offset; 
    custom_file.elf_header_size = orig_ehdr->e_ehsize; 
    custom_file.elf_program_header_table_off = enc_phdr_offset; 
    custom_file.elf_program_header_table_num = orig_ehdr->e_phnum; 
    custom_file.elf_program_header_table_size = phdr_table_size; 
   
    // 3. Create modified ELF header based on original 
    Custom_Elf64_Ehdr custom_ehdr = {0}; 
    memcpy(&custom_ehdr, orig_ehdr, sizeof(Elf64_Ehdr)); 
   
    // 只修改魔数头，保留其他e_ident字段 
    memcpy(custom_ehdr.e_ident, ".csf", 4); // Change magic number 
   
    // 4. Encrypt and store the modified ELF header at end of file    encrypt((unsigned char *)RC4_KEY, RC4_KEY_LEN, 
            reinterpret_cast(outfile_buffer + enc_ehdr_offset), 
            reinterpret_cast(&custom_ehdr), 
            orig_ehdr->e_ehsize); 
   
    // 5. Encrypt and store the original program header table at end of file 
    encrypt((unsigned char *)RC4_KEY, RC4_KEY_LEN, 
            reinterpret_cast(outfile_buffer + enc_phdr_offset), 
            reinterpret_cast(inputfile_buffer + orig_ehdr->e_phoff), 
            phdr_table_size); 
   
    // 6. Find and encrypt all PT_LOAD segments in place 
    Elf64_Phdr *phdr = (Elf64_Phdr *)(inputfile_buffer + orig_ehdr->e_phoff); 
    for (int i = 0; i < orig_ehdr->e_phnum; i++) { 
        if (phdr[i].p_type == PT_LOAD) { 
            // Validate segment boundaries 
            if (phdr[i].p_offset + phdr[i].p_filesz > inputfile_size) { 
                return -4; 
            } 
            LOGI("第%d个PT_LOAD段，偏移值为%llu，大小为%llu", i, phdr[i].p_offset, phdr[i].p_filesz); 
   
            encrypt((unsigned char *)RC4_KEY, RC4_KEY_LEN, 
                    reinterpret_cast(outfile_buffer + phdr[i].p_offset), 
                    reinterpret_cast(outfile_buffer + phdr[i].p_offset), 
                    phdr[i].p_filesz); 
        } 
    } 
   
    // 7. Zero out original ELF header in output buffer 
    memset(outfile_buffer, 0, sizeof(Elf64_Ehdr)); 
   
    // 8. Zero out original program header table in output buffer 
    memset(outfile_buffer + orig_ehdr->e_phoff, 0, phdr_table_size); 
   
    // 9. Write custom file header structure at beginning of output file 
    memcpy(outfile_buffer, &custom_file, sizeof(Custom_Elf64_File)); 
   
    return 0; 
} 
   
int unpack(char *elf_pack_data, size_t file_size, Custom_Elf64_File my_elf64_file) 
{ 
    // 输入验证 
    if (!elf_pack_data) { 
        return -1; 
    } 
   
    // 检查头值的合理性 
    if (my_elf64_file.elf_header_size == 0 || 
        my_elf64_file.elf_program_header_table_size == 0 || 
        my_elf64_file.elf_program_header_table_num == 0) { 
        return -2; 
    } 
   
    // 检查文件偏移是否在文件范围内 
    if (my_elf64_file.elf_header_off >= file_size || 
        my_elf64_file.elf_program_header_table_off >= file_size) { 
        return -3; 
    } 
   
    // 0. 保存Custom_Elf64_File原始数据副本 
    Custom_Elf64_File original_custom_file; 
    memcpy(&original_custom_file, elf_pack_data, sizeof(Custom_Elf64_File)); 
   
    // 1. 分配解密缓冲区 
    char *decrypted_ehdr = (char *)malloc(my_elf64_file.elf_header_size); 
    if (!decrypted_ehdr) { 
        return -4; 
    } 
   
    char *decrypted_phdr = (char *)malloc(my_elf64_file.elf_program_header_table_size); 
    if (!decrypted_phdr) { 
        free(decrypted_ehdr); 
        return -5; 
    } 
   
    // 2. 解密ELF头 
    decrypt((unsigned char *)RC4_KEY, RC4_KEY_LEN, 
            reinterpret_cast(decrypted_ehdr), 
            reinterpret_cast(elf_pack_data + my_elf64_file.elf_header_off), 
            my_elf64_file.elf_header_size); 
   
    // 验证解密后的ELF头 
    Elf64_Ehdr *decrypted_elf_header = (Elf64_Ehdr *)decrypted_ehdr; 
    if (memcmp(decrypted_elf_header->e_ident, ".csf", 4) != 0) { 
        free(decrypted_ehdr); 
        free(decrypted_phdr); 
        return -6; 
    } 
   
    // 3. 解密程序头表 
    decrypt((unsigned char *)RC4_KEY, RC4_KEY_LEN, 
            reinterpret_cast(decrypted_phdr), 
            reinterpret_cast(elf_pack_data + my_elf64_file.elf_program_header_table_off), 
            my_elf64_file.elf_program_header_table_size); 
   
    // 4. 分析程序头表，找出所有LOAD段 
    Elf64_Phdr *decrypted_ph_table = (Elf64_Phdr *)decrypted_phdr; 
   
    // 验证头表条目数量 
    if (decrypted_elf_header->e_phnum != my_elf64_file.elf_program_header_table_num) { 
        free(decrypted_ehdr); 
        free(decrypted_phdr); 
        return -7; 
    } 
   
    // 5. 验证程序头表 
    for (int i = 0; i < my_elf64_file.elf_program_header_table_num; i++) { 
        Elf64_Phdr *current_phdr = &decrypted_ph_table[i]; 
   
        if (current_phdr->p_type == PT_LOAD) { 
            // 检查段边界 
            if (current_phdr->p_offset + current_phdr->p_filesz > file_size) { 
                free(decrypted_ehdr); 
                free(decrypted_phdr); 
                return -8; 
            } 
        } 
    } 
   
    // 6. 解密所有LOAD段 
    for (int i = 0; i < my_elf64_file.elf_program_header_table_num; i++) { 
        Elf64_Phdr *current_phdr = &decrypted_ph_table[i]; 
   
        if (current_phdr->p_type == PT_LOAD && current_phdr->p_filesz > 0) { 
            // 解密段 
            decrypt((unsigned char *)RC4_KEY, RC4_KEY_LEN, 
                    reinterpret_cast(elf_pack_data + current_phdr->p_offset), 
                    reinterpret_cast(elf_pack_data + current_phdr->p_offset), 
                    current_phdr->p_filesz); 
        } 
    } 
   
    // 7. 将解密后的ELF头和程序头表存储在原位置，供restore函数使用 
    memcpy(elf_pack_data + my_elf64_file.elf_header_off, decrypted_ehdr, my_elf64_file.elf_header_size); 
    memcpy(elf_pack_data + my_elf64_file.elf_program_header_table_off, decrypted_phdr, my_elf64_file.elf_program_header_table_size); 
   
    // 8. 恢复原始的Custom_Elf64_File数据 
    memcpy(elf_pack_data, &original_custom_file, sizeof(Custom_Elf64_File)); 
   
    // 9. 清理资源 
    free(decrypted_ehdr); 
    free(decrypted_phdr); 
   
    return 0; 
}
 
int restore_elf(char *decrypted_data, size_t file_size, Custom_Elf64_File my_elf64_file, char **restored_data, size_t *restored_size) 
{ 
    // 输入验证 
    if (!decrypted_data || !restored_data || !restored_size) { 
        return -1; 
    } 
   
    // 获取解密后的ELF头和程序头表 
    Elf64_Ehdr *decrypted_elf_header = (Elf64_Ehdr *)(decrypted_data + my_elf64_file.elf_header_off); 
    Elf64_Phdr *decrypted_phdr_table = (Elf64_Phdr *)(decrypted_data + my_elf64_file.elf_program_header_table_off); 
   
    // 验证解密后的ELF头 
    if (memcmp(decrypted_elf_header->e_ident, ".csf", 4) != 0) { 
        return -2; 
    } 
   
    // 计算恢复后的文件大小 
    // 1. 考虑所有段的结尾 
    size_t max_segment_end = 0; 
    for (int i = 0; i < my_elf64_file.elf_program_header_table_num; i++) { 
        Elf64_Phdr *current_phdr = &decrypted_phdr_table[i]; 
        size_t segment_end = current_phdr->p_offset + current_phdr->p_filesz; 
        if (segment_end > max_segment_end) { 
            max_segment_end = segment_end; 
        } 
    } 
   
    // 2. 考虑节表的结尾 (如果存在) 
    size_t section_table_end = 0; 
    if (decrypted_elf_header->e_shoff > 0 && decrypted_elf_header->e_shnum > 0) { 
        section_table_end = decrypted_elf_header->e_shoff + 
                            decrypted_elf_header->e_shnum * decrypted_elf_header->e_shentsize; 
    } 
   
    // 取最大值作为文件大小的基础 
    size_t new_file_size = (max_segment_end > section_table_end) ? max_segment_end : section_table_end; 
   
    // 确保文件大小至少包含ELF头和程序头表 
    size_t min_size = decrypted_elf_header->e_phoff + 
                      decrypted_elf_header->e_phnum * decrypted_elf_header->e_phentsize; 
   
    if (new_file_size < min_size) { 
        new_file_size = min_size; 
    } 
   
    // 确保新文件大小不超过原文件中不包含尾部附加数据的大小 
    if (new_file_size > my_elf64_file.elf_header_off && my_elf64_file.elf_header_off > min_size) { 
        new_file_size = my_elf64_file.elf_header_off; 
    } 
   
    // 分配新的内存来存储恢复的ELF文件 
    *restored_data = (char *)malloc(new_file_size); 
    if (!*restored_data) { 
        return -3; 
    } 
   
    // 复制基本数据（不包括末尾额外数据） 
    memcpy(*restored_data, decrypted_data, new_file_size); 
   
    // 创建完整的ELF头 
    Elf64_Ehdr restored_ehdr = {0}; 
    memcpy(&restored_ehdr, decrypted_elf_header, sizeof(Elf64_Ehdr)); 
   
    // 只恢复标准ELF魔数，保留其他e_ident字段原值 
    memcpy(restored_ehdr.e_ident, ELFMAG, SELFMAG);  // 标准魔数: 0x7F ELF 
   
    // 写入ELF头 
    memcpy(*restored_data, &restored_ehdr, sizeof(Elf64_Ehdr)); 
   
    // 确定程序头表的位置和大小 
    Elf64_Off phdr_restore_offset = restored_ehdr.e_phoff; 
   
    // 如果e_phoff不合理，使用标准位置 
    if (phdr_restore_offset < sizeof(Elf64_Ehdr) || phdr_restore_offset >= new_file_size) { 
        phdr_restore_offset = sizeof(Elf64_Ehdr); // 通常是64字节 
        // 更新ELF头中的程序头表偏移 
        ((Elf64_Ehdr*)(*restored_data))->e_phoff = phdr_restore_offset; 
    } 
   
    // 确保程序头表不会超出文件边界 
    size_t phdr_table_size = decrypted_elf_header->e_phnum * decrypted_elf_header->e_phentsize; 
    if (phdr_restore_offset + phdr_table_size > new_file_size) { 
        free(*restored_data); 
        *restored_data = NULL; 
        *restored_size = 0; 
        return -4; 
    } 
   
    // 复制程序头表 
    memcpy(*restored_data + phdr_restore_offset, decrypted_phdr_table, phdr_table_size); 
   
    // 设置恢复后的文件大小 
    *restored_size = new_file_size; 
   
    // 最后验证恢复的ELF文件 
    Elf64_Ehdr *final_ehdr = (Elf64_Ehdr *)(*restored_data); 
    if (memcmp(final_ehdr->e_ident, ELFMAG, SELFMAG) != 0) { 
        free(*restored_data); 
        *restored_data = NULL; 
        *restored_size = 0; 
        return -5; 
    } 
   
    return 0; 
} 
```

那么现在有两个选择，一种是恢复为标准 ELF 文件格式后加载，另一种则是直接从自定义 ELF 文件格式加载。我选择从自定义 ELF 文件格式加载。  
但是如果想学习自定义 ELF 文件格式加载，建议先恢复为标准 ELF 文件格式，加载成功后，对照修改。

### 3.3. 实现原理

按照 android 加载 so 的流程来

#### 3.3.1. 读取 so 文件信息

这里传入的`file_offset`是`ELF Header`的偏移，一般标准 ELF 文件的偏移都是 0，所以这里直接置 0，而自定义的 ELF 的`ELF Header`偏移则用`origin_file_size_`保存一下，后面有用

```
bool ElfReader::DIY_Read(const char* name, int fd, off64_t file_offset, off64_t file_size) { 
    if (did_read_) { 
        return true; 
    } 
    name_ = name; 
    fd_ = fd; 
    file_offset_ = 0; 
    file_size_ = file_size; 
    origin_file_size_ = file_offset; 
   
    if (DIY_ReadElfHeader() && 
        DIY_VerifyElfHeader() && 
        ReadProgramHeaders() && 
        ReadSectionHeaders() && 
        ReadDynamicSection()) { 
        did_read_ = true; 
    } 
   
    return did_read_; 
}
```

#### 3.3.2. 读取 so 文件信息

在这里传入的参数`file_offset`是`ELF Header`的偏移，正常 so 文件的偏移都是 0，所以我直接置 0，这样可以保证后面映射到内存时不出错，而把自定义的格式`ELF Header`的偏移使用`origin_file_size_`保存了起来。

```
bool ElfReader::DIY_Read(const char* name, int fd, off64_t file_offset, off64_t file_size) { 
    if (did_read_) { 
        return true; 
    } 
    name_ = name; 
    fd_ = fd; 
    file_offset_ = 0; 
    file_size_ = file_size; 
    origin_file_size_ = file_offset; 
   
    if (DIY_ReadElfHeader() && 
        DIY_VerifyElfHeader() && 
        ReadProgramHeaders() && 
        ReadSectionHeaders() && 
        DIY_ReadDynamicSection()) { 
        did_read_ = true; 
    } 
   
    return did_read_; 
}
```

#### 3.3.3. 读取 so 文件头

在这里就可以把刚才保存的`origin_file_size_`使用起来，使得能够正确的读到 so 文件头。

```
bool ElfReader::DIY_ReadElfHeader() { 
    LOGI("enter DIY_ReadElfHeader"); 
    ssize_t rc = TEMP_FAILURE_RETRY(pread64(fd_, &header_, sizeof(header_), origin_file_size_)); 
    if (rc < 0) { 
        LOGE("can't read file \"%s\": %s", name_.c_str(), strerror(errno)); 
        return false; 
    } 
    if (rc != sizeof(header_)) { 
        LOGE("\"%s\" is too small to be an ELF executable: only found %zd bytes", name_.c_str(), 
               static_cast(rc)); 
        return false; 
    } 
    return true; 
} 
```

#### 3.3.4. 校验 so 文件头

跟源码差不多，只是 ELF 的魔数头需要替换为我们自定义的`.csf`，并且这个校验步骤完全可以省略掉。

#### 3.3.5. 读取程序头

```
bool ElfReader::ReadProgramHeaders() { 
    phdr_num_ = header_.e_phnum; 
   
    // Like the kernel, we only accept program header tables that 
    // are smaller than 64KiB.    if (phdr_num_ < 1 || phdr_num_ > 65536/sizeof(ElfW(Phdr))) { 
        LOGE("\"%s\" has invalid e_phnum: %zd", name_.c_str(), phdr_num_); 
        return false; 
    } 
   
    // Boundary checks 
    size_t size = phdr_num_ * sizeof(ElfW(Phdr)); 
    if (!CheckFileRange(header_.e_phoff, size, alignof(ElfW(Phdr)))) { 
        LOGE("\"%s\" has invalid phdr offset/size: %zu/%zu", 
                       name_.c_str(), 
                       static_cast(header_.e_phoff), 
                       size); 
        return false; 
    } 
   
    if (!phdr_fragment_.Map(fd_, file_offset_, header_.e_phoff, size)) { 
        LOGE("\"%s\" phdr mmap failed: %m", name_.c_str()); 
        return false; 
    } 
   
    phdr_table_ = static_cast(phdr_fragment_.data()); 
    return true; 
} 
```

#### 3.3.6. 读取节头

代码部分跟源码一样

```
bool ElfReader::ReadSectionHeaders() {
  shdr_num_ = header_.e_shnum;
 
  if (shdr_num_ == 0) {
    DL_ERR_AND_LOG("\"%s\" has no section headers", name_.c_str());
    return false;
  }
 
  size_t size = shdr_num_ * sizeof(ElfW(Shdr));
  if (!CheckFileRange(header_.e_shoff, size, alignof(const ElfW(Shdr)))) {
    DL_ERR_AND_LOG("\"%s\" has invalid shdr offset/size: %zu/%zu",
                   name_.c_str(),
                   static_cast(header_.e_shoff),
                   size);
    return false;
  }
 
  if (!shdr_fragment_.Map(fd_, file_offset_, header_.e_shoff, size)) {
    DL_ERR("\"%s\" shdr mmap failed: %m", name_.c_str());
    return false;
  }
 
  shdr_table_ = static_cast(shdr_fragment_.data());
  return true;
} 
```

#### 3.3.7. 接下来进入 so 的加载

其中主要说一下`FindPhdr()`, 在这个程序中有一个很坑的点在于后面的一步`si_->phdr = elfreader->loaded_phdr();`。在`Program Header Table`中如果有`p_type == PT_PHDR`的段，那么**该段类型的数组元素如果存在的话，则给出了程序头部表自身的大小和位置，既包括在文件中也包括在内存中的信息**。也就是说段中的`p_vaddr`和`p_offset`的值是原来`Program Header Table`的偏移值，这就会导致`si_->phdr = elfreader->loaded_phdr();`指向的是错误的内存。这里我的解决办法是直接判断是不是自定义格式的 ELF 文件，如果是则让`loaded_phdr_ = phdr_table_`。

```
bool ElfReader::Load() {
    CHECK(did_read_);
    if (did_load_) {
        return true;
    }
    if (ReserveAddressSpace() && //创建足够大加载段的内存空间
        LoadSegments()&& //段加载
        FindPhdr()) {//寻找程序头
        did_load_ = true;
    }
  
    return did_load_;
}
```

```
bool ElfReader::FindPhdr() {
  const ElfW(Phdr)* phdr_limit = phdr_table_ + phdr_num_;
    if (memcmp(header_.e_ident, reinterpret_cast(DIY_ELFMAG), SELFMAG) == 0) { 
    LOGE("\"%s\" has bad ELF magic: %02x%02x%02x%02x", name_.c_str(), 
    header_.e_ident[0], header_.e_ident[1], header_.e_ident[2], header_.e_ident[3]); 
    loaded_phdr_ = phdr_table_; 
    return true; 
}
 
  // If there is a PT_PHDR, use it directly.
  for (const ElfW(Phdr)* phdr = phdr_table_; phdr < phdr_limit; ++phdr) {
    if (phdr->p_type == PT_PHDR) {
      return CheckPhdr(load_bias_ + phdr->p_vaddr);
    }
  }
 
  // Otherwise, check the first loadable segment. If its file offset
  // is 0, it starts with the ELF header, and we can trivially find the
  // loaded program header from it.
  for (const ElfW(Phdr)* phdr = phdr_table_; phdr < phdr_limit; ++phdr) {
    if (phdr->p_type == PT_LOAD) {
      if (phdr->p_offset == 0) {
        ElfW(Addr)  elf_addr = load_bias_ + phdr->p_vaddr;
        const ElfW(Ehdr)* ehdr = reinterpret_cast(elf_addr);
        ElfW(Addr)  offset = ehdr->e_phoff;
        return CheckPhdr(reinterpret_cast(ehdr) + offset);
      }
      break;
    }
  }
 
  DL_ERR("can't find loaded phdr for \"%s\"", name_.c_str());
  return false;
} 
```

#### 3.3.8. 借鸡生蛋

我们用 dlopen 打开一个空壳 so，然后填入我们自己 so 的信息，达到替换的操作。这里的关键点在于实现`soinfo_from_handle`函数，这个函数由于并没有导出，所以需要自己实现。

```
FILE *fp = fopen("/proc/self/maps", "r"); 
if (!fp) { 
    LOGI("Failed to open /proc/self/maps"); 
    close(fd); 
    return; 
} 
   
parse_maps_for_lib("linker64", &start, &end);  //linker64地址解析
LOGI("linker64 start = 0x%lx end = 0x%lx", start, end); 
uintptr_t sym_offset = get_soinfo_handles_map_offset(start, end); 
if (sym_offset == 0) {
    LOGE("Failed to get offset"); 
    return; 
} else{ 
    LOGI("soinfo_handles_map offset = 0x%lx", sym_offset); 
} 
   
soinfo_handles_map = reinterpret_cast*>((char*)start + sym_offset); 
if (soinfo_handles_map == nullptr) { 
    LOGI("获取soinfo_handles_map失败"); 
    return; 
} 
void *handle = dlopen(path_fake, RTLD_LAZY);  //打开空壳so，创建系统内部的soinfo，获取对应的句柄
if (handle == NULL) { 
    LOGI("%s dlopen failed: %s\n", path_fake, dlerror()); 
    fclose(fp); 
    close(fd); 
    return; 
} 
   
si_ = soinfo_from_handle(soinfo_handles_map, static_cast*>(handle)); 
LOGI("si_ %p\n", si_);
//参照安卓源码，数据回填，偷梁换柱
si_->base = elfreader->load_start(); 
si_->size = elfreader->load_size(); 
si_->load_bias = elfreader->load_bias(); 
si_->phnum = elfreader->phdr_count(); 
si_->phdr = elfreader->loaded_phdr(); 
```

我们仔细看一下这个函数的源码，发现其实只用到了`g_soinfo_handles_map`这个全局变量是不知道的。再参考原文使用的是偏移地址，那么我们完善一下，解析`linker64`这个 ELF 文件，就可以适用于不同的安卓版本了。

```
static soinfo* soinfo_from_handle(void* handle) {
  if ((reinterpret_cast(handle) & 1) != 0) {
    auto it = g_soinfo_handles_map.find(reinterpret_cast(handle));
    if (it == g_soinfo_handles_map.end()) {
      return nullptr;
    } else {
      return it->second;
    }
  }
 
  return static_cast(handle);
} 
```

```
uintptr_t get_soinfo_handles_map_offset(uint64_t start, uint64_t end) { 
    int fd = open("/system/bin/linker64", O_RDONLY); 
    if (fd < 0) { 
        LOGI("Failed to open linker64: %s", strerror(errno)); 
        return 0; 
    } 
   
    struct stat sb; 
    fstat(fd, &sb); 
    void* file_start = mmap(NULL, sb.st_size, PROT_READ, MAP_PRIVATE, fd, 0); 
    if (file_start == MAP_FAILED) { 
        LOGI("mmap failed: %s", strerror(errno)); 
        close(fd); 
        return 0; 
    } 
   
    // Parse ELF header 
    Elf64_Ehdr header; 
    memcpy(&header, file_start, sizeof(Elf64_Ehdr)); 
    int secoff = header.e_shoff; 
    int secsize = header.e_shentsize; 
    int secnum = header.e_shnum; 
    int secstr = header.e_shstrndx; 
   
    // Get section name string table 
    Elf64_Shdr strtab; 
    memcpy(&strtab, (char*)file_start + secoff + secstr * secsize, sizeof(Elf64_Shdr)); 
    int strtaboff = strtab.sh_offset; 
    char* strtabchar = new char[strtab.sh_size]; 
    memcpy(strtabchar, (char*)file_start + strtaboff, strtab.sh_size); 
   
    // Find symbol table and string table 
    Elf64_Shdr enumsec; 
    int symtab_off = 0, symtab_size = 0; 
    int strtab_off = 0, strtab_size = 0; 
   
    for (int n = 0; n < secnum; n++) { 
        memcpy(&enumsec, (char*)file_start + secoff + n * secsize, sizeof(Elf64_Shdr)); 
        if (strcmp(&strtabchar[enumsec.sh_name], ".symtab") == 0) { 
            symtab_off = enumsec.sh_offset; 
            symtab_size = enumsec.sh_size; 
            LOGI("Found .symtab: offset=0x%x, size=%d", symtab_off, symtab_size); 
        } 
        if (strcmp(&strtabchar[enumsec.sh_name], ".strtab") == 0) { 
            strtab_off = enumsec.sh_offset; 
            strtab_size = enumsec.sh_size; 
            LOGI("Found .strtab: offset=0x%x, size=%d", strtab_off, strtab_size); 
        } 
    } 
   
    delete[] strtabchar; 
   
    if (symtab_off == 0 || strtab_off == 0) { 
        LOGI("Required sections not found"); 
        munmap(file_start, sb.st_size); 
        close(fd); 
        return 0; 
    } 
   
    // Search for soinfo_handles_map 
    uintptr_t symbol_offset = 0; 
    char* symbol_strtab = new char[strtab_size]; 
    memcpy(symbol_strtab, (char*)file_start + strtab_off, strtab_size); 
   
    for (int n = 0; n < symtab_size; n += sizeof(Elf64_Sym)) { 
        Elf64_Sym sym; 
        memcpy(&sym, (char*)file_start + symtab_off + n, sizeof(Elf64_Sym)); 
   
        if (sym.st_name != 0) { 
            const char* name = symbol_strtab + sym.st_name; 
            if (strstr(name, "soinfo_handles_map")) { 
                symbol_offset = sym.st_value; 
                LOGI("Found symbol %s: offset=0x%lx", name, symbol_offset); 
                break; 
            } 
        } 
    } 
   
    delete[] symbol_strtab; 
    munmap(file_start, sb.st_size); 
    close(fd); 
   
    return symbol_offset; 
}
```

后面的`预链接`与`链接`过程则与原文一样了。  
调用日志  
![](https://bbs.kanxue.com/upload/attach/202505/998309_6FVKMG9JAV8B35G.png)  
![](https://bbs.kanxue.com/upload/attach/202505/998309_PWMVAJM5TQ26G88.png)

### 3.4. 看一下 soinfolist

我们使用`libjoke.so`作为我们的壳，所以我们自然要在 soinfolist 中找的是 libjoke.so，使用 frida 打印一下。红线上面部分是 soinfolist 遍历的结构，下面部分是打印的是 ELF 文件的前 64 字节数据，正是我们之前`3.1节`图中使用 RC4 解密后传入的数据。

![](https://bbs.kanxue.com/upload/attach/202505/998309_W4JDEQ5QRG2Y26K.png)

4. 总结
-----

这里三种方法可以并不止局限于一种，可以搭配使用。既然都是自定义 linker 加载了，自然 so 可以放在更隐蔽的地方。在这里`call_constructors`没有实现，有兴趣的朋友可以自己实现一下。具体完整代码我就不贴出来了，里面各种各样调试信息绕来绕去没删除，写得太乱了。

5. 参考链接
-------

*   https://bbs.kanxue.com/thread-285650.htm
*   https://oacia.dev/ElfReader/
*   https://oacia.dev/android-load-so/
*   https://www.cnblogs.com/revercc/p/17100308.html

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-173.htm)
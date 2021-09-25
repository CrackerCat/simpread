> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269484.htm)

> 《基于 linker 实现 so 加壳技术基础》上篇

[](#《基于linker实现so加壳技术基础》上篇)《基于 linker 实现 so 加壳技术基础》上篇
-----------------------------------------------------

### 前言

本篇是一个追随技术的人照着网上为数不多的资料，在探索过程中发现了许多意想不到的问题，搜了好多文章发现对于这方面的记载很少，甚至连一个实现的蓝本都没找到，写下这篇文章希望能帮到像我一样对 so 壳感兴趣的伙伴在做实践的时候能有一个抓手，只有了解了加壳原理才能更好的脱壳，其实 so 文件在系统当中都对应了一个 soinfo 指针如果能找到 soinfo 指针那么脱掉 so 壳也不是不可能，如果有机会的话后面会出 so 脱壳相关的知识，其实最难的点在于如何找到当前 so 的 soinfo 指针这个，源自 aosp8.1.0_r33.

### 思路

回想 dex 可以通过动态加载插件的方式完成加壳，例如一代壳，那么 so 是否也有这种技术呢，答案是肯定的，只是用 c 实现的 so 没有办法使用 java 中的 classloader 这种，快速便捷的类加载方式，那么我们现在就遇到了 2 个问题就是如何将 so 加载到内存中，如何使用 so 中的函数, 那么其实我们现在就可以从安卓源码入手，看一看 so 是如何加载到系统中的

### 装载

从 System.loadLibrary 入手（ps：我们平常加载 so 就是用到这两个函数一个是 System.loadLibrary 传入 Classloader 所在的相对路径，一个是 System.load 传入绝对路径, 最后发现他们在 native 层都会汇聚到一点）, 一直跟下去就会发现最后 Runtime 下的私有函数 doLoad 中调用了 nativeLoad 来进入 native 层

```
public static void loadLibrary(String libname) {
     Runtime.getRuntime().loadLibrary0(VMStack.getCallingClassLoader(), libname);
   }
.....
    private String doLoad(String name, ClassLoader loader) {
.....
return nativeLoad(name, loader, librarySearchPath);
    }

```

那么顺利进入 so 层，进入了`libcore/ojluni/src/main/native/Runtime.c`中的 Runtime_nativeLoad 方法, 一直跟下去最终在`system/core/libnativeloader/native_loader.cpp`中找到了 OpenNativeLibrary 方法找到了 dlopen 和 android_dlopen_ext（ps: 通过分析发现这两个函数最终也汇聚于 linker 下的 do_dlopen）

```
Runtime_nativeLoad(JNIEnv* env, jclass ignored, jstring javaFilename,
                 jobject javaLoader, jstring javaLibrarySearchPath)
{
   return JVM_NativeLoad(env, javaFilename, javaLoader, javaLibrarySearchPath);
}
......c
void* OpenNativeLibrary(...) {
#if defined(__ANDROID__)
  UNUSED(target_sdk_version);
 if (class_loader == nullptr) {
   *needs_native_bridge = false;
   return dlopen(path, RTLD_NOW);
  }
.....
 void* handle = android_dlopen_ext(path, RTLD_NOW, &extinfo);//当classloader不为空的情况下调用android_dlopen_ext继续跟下去其实也可以看到它的实现最终也是do_dlopen

```

那么其实现在就清楚了，其实 java 层的 loadLibrary 也是通过 linker 中的 do_dlopen 来 j 将 so 文件加载到内存中的，那么下面开始分析 do_dlopen 中的 so 的加载流程。

```
void* do_dlopen(......) {
...
soinfo* si = find_library(ns, translated_name, flags, extinfo, caller)//通过find_library函数拿到当前handle对应的soinfo
...
             }

```

那么继续跟进 find_library 函数，发现如下关键点代码

```
...
static soinfo* find_library(android_namespace_t* ns,
                          const char* name, int rtld_flags,
                           const android_dlextinfo* extinfo,
                           soinfo* needed_by) {
...
if (!find_libraries(const_cast(task->get_start_from()),task,&zip_archive_cache,&load_tasks,rtld_flags,search_linked_namespaces || is_dt_needed))//关键函数详细分析
...
 
} 
```

在 find_libraries 函数中, 通过 find_library_internal，来搞定所有需要的 so 和初始化我们的 so 等一些列工作

```
bool find_libraries(...) {
 
 
 
...
 if (!find_library_internal(....）{
   return false；
 }
...
 
 
                   }

```

一直跟下去，最终在 load_library 函数中发现了调用了 LoadTask 对象的 read 函数，之后又调用了 ElfReader 对象的 read 函数来初始化 LoadTask

```
static bool load_library(...) {
...
if (!task->read(realpath.c_str(), file_stat.st_size))//通过ElfReader对象的read函数初始化我们的LoadTask对象
...
 for (const ElfW(Dyn)* d = elf_reader.dynamic(); d->d_tag != DT_NULL; ++d) {
    if (d->d_tag == DT_RUNPATH) {
      si->set_dt_runpath(elf_reader.get_string(d->d_un.d_val));
    }
    if (d->d_tag == DT_SONAME) {
      si->set_soname(elf_reader.get_string(d->d_un.d_val));
    }
  }
 
  for_each_dt_needed(task->get_elf_reader(), [&](const char* name) {
    load_tasks->push_back(LoadTask::create(name, si, ns, task->get_readers_map()));
  });
 
 
                         }

```

LoadTask 对象的 read 方法十分简洁这里就不贴代码了。在`/bionic/linker/linker_phdr.cpp`目录中找到了 ElfReader 类中 read 函数的实现, 这个函数很重要所以就全贴上了，可以看到，这里将我们打开 so 文件的 fd 以及 file_size 等一系列信息复制给了 ElfReader 对象然后做了 5 件事  
1：读取 elf 头  
2：验证 elf 头  
3：读取程序头  
4：读取节头  
5：读取 Dynamic 节  
但是我们是做一个简单的壳肯定要去掉这些对于加载用不到的地方，所以这里可以只实现读取程序头和 elf 头，因为 Execution View 下一定会使用程序头加载段而节信息可有可无

```
bool ElfReader::Read(const char* name, int fd, off64_t file_offset, off64_t file_size) {
  if (did_read_) {
    return true;
  }
  name_ = name;
  fd_ = fd;
  file_offset_ = file_offset;
  file_size_ = file_size;
 
  if (ReadElfHeader() &&
      VerifyElfHeader() &&
      ReadProgramHeaders() &&
      ReadSectionHeaders() &&
      ReadDynamicSection()) {
    did_read_ = true;
  }
 
  return did_read_;
}

```

下面就是我模仿安卓源码写的一个 ReadProgramHeader 的实现, 首先要实现一个 loader 类，模仿源码中的 ElfReader 类给他填上属性。，然后实现 ReadProgramHeader 方法.

```
class load {
public:
    const char *name_;
    int fd_;
    ElfW(Ehdr) header_;
    size_t phdr_num_;
    void *phdr_mmap_;
    ElfW(Phdr) *phdr_table_;
    ElfW(Addr) phdr_size_;
    void *load_start_;
    size_t load_size_;
    ElfW(Addr) load_bias_;
    const ElfW(Phdr) *loaded_phdr_;
    void *st;
public:
    load(void *sta)
            : phdr_num_(0), phdr_mmap_(NULL), phdr_table_(NULL), phdr_size_(0),
              load_start_(NULL), load_size_(0), load_bias_(0),
              loaded_phdr_(NULL), st(sta) {
    }
 
   bool loadhead(){
 
 
     return   memcpy(&(header_),st,sizeof(header_));//赋值elf头
 
 
    };
   bool ReadProgramHeader() {
        phdr_num_ = header_.e_phnum;//由于我们没有执行ReadElfHeader函数所以一会要手动给这个header赋值
        ElfW(Addr) page_min = PAGE_START(header_.e_phoff);//页对齐
        ElfW(Addr) page_max = PAGE_END(header_.e_phoff + (phdr_num_ * sizeof(ElfW(Phdr))));
        ElfW(Addr) page_offset = PAGE_OFFSET(header_.e_phoff);//获得程序头的偏移
        void **c = reinterpret_cast((char *) (st) + page_min);
        phdr_table_ = reinterpret_cast(reinterpret_cast(c) + page_offset);//获得程序头实际地址
        return true;
    };
    } 
```

需要将我们的真正的 so 加载到内存中 (如果是壳的话可以是加密文件或者直接贴在 dex 文件最后有很多方法这里不再讨论，这里只讨论最基础的动态加载），使用 mmap 就好

```
int fd;
void *start;
struct stat sb;
fd = open("/data/local/tmp/1.so", O_RDONLY); //打开获得我们插件so的fd
fstat(fd, &sb);
start = static_cast(mmap(NULL, sb.st_size, PROT_READ, MAP_PRIVATE, fd, 0));//用mmap把文件加载到内存中
load a(start);//构造load对象 
```

然后 read 函数就完成了其他部分我们不需要看。结束后 load_library 函数中第二个关键的部分就是对依赖 so 的加载，这里重点看 for_each_dt_needed 函数，逻辑十分的简单，就是从 dynamic 段中寻找类型为 0x1（在 elf.h 中 #define DT_NEEDED 1）的段然后将这个 so 放入到 LoadTask 对象的加载队列中，最终通过 read 函数将每一个未被加载的 so 加载到内存中，并且返回所有的 soinfo 指针。

```
static void for_each_dt_needed(const ElfReader& elf_reader, F action) {
  for (const ElfW(Dyn)* d = elf_reader.dynamic(); d->d_tag != DT_NULL; ++d) {
   if (d->d_tag == DT_NEEDED) {
      action(fix_dt_needed(elf_reader.get_string(d->d_un.d_val), elf_reader.name()));
    }
 }
}

```

接着继续 find_library 函数，发现之后又调用了 task 的 load 函数。跟踪进去 load 函数中发现分 2 块内容，第一块内容就是调用 ElfReader 类的 load 方法装载 so，第二部分就是对于有 soinfo 的修正，代码十分简洁这里就不贴了，那么现在的任务就是看看 ElfReader 类的 load 方法是如何实现的。load 函数就少了一点只做了 3 件事  
1：申请段加载空间  
2：装载段  
3：找到 Phdr 在内存中的地址  
那么这里我们就要将这三个方法全部都实现了

```
bool ElfReader::Load(const android_dlextinfo* extinfo) {
  CHECK(did_read_);
 if (did_load_) {
    return true;
  }
  if (ReserveAddressSpace(extinfo) &&
    LoadSegments() &&
      FindPhdr()) {
   did_load_ = true;
 }
 
  return did_load_;
}

```

首先是申请加载空间, 期间有一个 phdr_table_get_load_size 方法我们直接从源码里面抄过来即可

```
size_t phdr_table_get_load_size(const ElfW(Phdr)* phdr_table, size_t phdr_count,
                                ElfW(Addr)* out_min_vaddr) {
    ElfW(Addr) min_vaddr = UINTPTR_MAX;
    ElfW(Addr) max_vaddr = 0;
 
    bool found_pt_load = false;
    for (size_t i = 0; i < phdr_count; ++i) {
        const ElfW(Phdr)* phdr = &phdr_table[i];
 
        if (phdr->p_type != PT_LOAD) {
            continue;
        }
        found_pt_load = true;
 
        if (phdr->p_vaddr < min_vaddr) {
            min_vaddr = phdr->p_vaddr;
        }
 
        if (phdr->p_vaddr + phdr->p_memsz > max_vaddr) {
            max_vaddr = phdr->p_vaddr + phdr->p_memsz;
        }
    }
    if (!found_pt_load) {
        min_vaddr = 0;
    }
 
    min_vaddr = PAGE_START(min_vaddr);
    max_vaddr = PAGE_END(max_vaddr);
 
    if (out_min_vaddr != nullptr) {
        *out_min_vaddr = min_vaddr;
    }
 
    return max_vaddr - min_vaddr;
}
 
 
   bool ReserveAddressSpace() {
 
        ElfW(Addr) min_vaddr;
        load_size_ = phdr_table_get_load_size(phdr_table_, phdr_num_, &min_vaddr);//获得段总大小
 
        uint8_t *addr = reinterpret_cast(min_vaddr);
        void *start;
 
        int mmap_flags = MAP_PRIVATE | MAP_ANONYMOUS;
 
 
 
        start = start = mmap(addr, load_size_, PROT_NONE, mmap_flags, -1, 0);;//直接申请空间需要页对其所以好像不能用malloc
 
        load_start_ = start;
        load_bias_ = reinterpret_cast(load_start_) - addr;
        __android_log_print(6,"r0ysue","%p 111111  %x",load_bias_,load_size_);
        return true;
 
    }; 
```

接下来就是装载段信息, 这部分也是完全仿照安卓源码的写法，将段头中所有类型为 0x1 的段加载到内存当中 (#define PT_LOAD 1) 我这里直接用 memcpy 了主要是不想从文件中加载 so 想的是我这种方案直接内存加载 so, 并且用 mprotect 申请权限，最后填 0 占位

```
   bool LoadSegments() {
        for (size_t i = 0; i < phdr_num_; ++i) {
            const ElfW(Phdr) *phdr = &phdr_table_[i];
            if (phdr->p_type != PT_LOAD) {
                continue;
            }
            // Segment addresses in memory.
            ElfW(Addr) seg_start = phdr->p_vaddr + load_bias_;
            ElfW(Addr) seg_end = seg_start + phdr->p_memsz;
            ElfW(Addr) seg_page_start = PAGE_START(seg_start);
            ElfW(Addr) seg_page_end = PAGE_END(seg_end);
            ElfW(Addr) seg_file_end = seg_start + phdr->p_filesz;
            // File offsets.
            ElfW(Addr) file_start = phdr->p_offset;
            ElfW(Addr) file_end = file_start + phdr->p_filesz;
 
            ElfW(Addr) file_page_start = PAGE_START(file_start);
            ElfW(Addr) file_length = file_end - file_page_start;
            long* pp= reinterpret_cast(seg_page_start);
            __android_log_print(6,"r0ysue","%p 111111",load_bias_);
            __android_log_print(6,"r0ysue","%p 111111",seg_page_end);
            mprotect(reinterpret_cast(seg_page_start), seg_page_end-seg_page_start, PROT_WRITE);//申请访问权限
 
            if (file_length != 0) {
                void* c=(char*)st+file_page_start;
 
                memcpy(reinterpret_cast(seg_page_start), c, file_length);//我把mmap改成了memcpy因为安卓源码中用了fd我期望全使用内存加载的方式所以有fd的地方我都改了
            }
            if ((phdr->p_flags & PF_W) != 0 && PAGE_OFFSET(seg_file_end) > 0) {
                memset(reinterpret_cast(seg_file_end), 0, PAGE_SIZE - PAGE_OFFSET(seg_file_end));
            }
            seg_file_end = PAGE_END(seg_file_end);
            if (seg_page_end > seg_file_end) {
                void* zeromap = mmap(reinterpret_cast(seg_file_end),
                                     seg_page_end - seg_file_end,
                                     PFLAGS_TO_PROT(phdr->p_flags),
                                     MAP_FIXED|MAP_ANONYMOUS|MAP_PRIVATE,
                                     -1,
                                     0);
                __android_log_print(6,"r0ysue","duiqi %p ",zeromap);
 
            }
//            __android_log_print(6,"r0ysue","%p 111111",seg_file_end);
        }
 
        return true;
    }; 
```

最后实现 FindPhdr 方法，其中引用了 CheckPhdr 方法，这里 FindPHPtr 函数主要分 2 类如果直接就有程序头段那么直接用就好否则就找虚拟地址为 0 的段从文件头解析它，全部也是直接超过来就好，但是注意这个要抄在 load 类里面，上面的 phdr_table_get_load_size 写在类的外面也无关紧要。

```
bool CheckPhdr(ElfW(Addr) loaded) {
      const ElfW(Phdr) *phdr_limit = phdr_table_ + phdr_num_;
      ElfW(Addr) loaded_end = loaded + (phdr_num_ * sizeof(ElfW(Phdr)));
      for (ElfW(Phdr) *phdr = phdr_table_; phdr < phdr_limit; ++phdr) {
          if (phdr->p_type != PT_LOAD) {
              continue;
          }
          ElfW(Addr) seg_start = phdr->p_vaddr + load_bias_;
          ElfW(Addr) seg_end = phdr->p_filesz + seg_start;
          if (seg_start <= loaded && loaded_end <= seg_end) {
              loaded_phdr_ = reinterpret_cast(loaded);
              return true;
          }
      }
 
      return false;
  };
 
 bool FindPHPtr(){
      const ElfW(Phdr)* phdr_limit = phdr_table_ + phdr_num_;
 
      for (const ElfW(Phdr)* phdr = phdr_table_; phdr < phdr_limit; ++phdr) {
          if (phdr->p_type == PT_PHDR) {
              return CheckPhdr(load_bias_ + phdr->p_vaddr);//主要检测这个段是否越界
          }
      }
 
      for (const ElfW(Phdr)* phdr = phdr_table_; phdr < phdr_limit; ++phdr) {
          if (phdr->p_type == PT_LOAD) {
              if (phdr->p_offset == 0) {
                  ElfW(Addr)  elf_addr = load_bias_ + phdr->p_vaddr;
                  const ElfW(Ehdr)* ehdr = reinterpret_cast(elf_addr);
                  ElfW(Addr)  offset = ehdr->e_phoff;
                  return CheckPhdr((ElfW(Addr))ehdr + offset);
              }
              break;
          }
      }
 
 
      return false;
 
  }; 
```

至此我们已经写完了装载过程，至于引用的其他 so 我们写一个循环直接重走一边即可，由于我这里没有引用所以先不展示这一方面的内容了，这里还有一段 Load 函数结尾需要将我们上面读到的内容存储到 link 维护的本 so 的 soinfo 中，而由于我们是加壳所以要做的就是将刚才得到的段信息替换掉本 so 的 soinfo，类似于 dex 整体加壳（一代壳）的 classloader 的修正，接下来进入获得 soinfo 部分，会写在下篇里面

[第五届安全开发者峰会（SDC 2021）10 月 23 日上海召开！限时 2.5 折门票 (含自助午餐 1 份）](https://www.bagevent.com/event/6334937)

[#基础理论](forum-161-1-117.htm) [#混淆加固](forum-161-1-121.htm) [#源码分析](forum-161-1-127.htm)
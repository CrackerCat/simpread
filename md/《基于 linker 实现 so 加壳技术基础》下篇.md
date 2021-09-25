> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269485.htm)

> 《基于 linker 实现 so 加壳技术基础》下篇

[](#《基于linker实现so加壳技术基础》下篇)《基于 linker 实现 so 加壳技术基础》下篇
-----------------------------------------------------

### 获得 linker 维护的本 so 的 soinfo

但是问题又来了如何获得当前 so 的 soinfo 指针的基址呢？翻阅网上的资料说可以 dlopen 打开 self，我看了一下那是安卓 7 之前的方法安卓 8.1 不支持了（555 这不是坑人嘛咋搞），于是我阅读安卓源码发现了获得 soinfo 的方法，这是一套组合拳，可以先 dlopen 自己然后再用 soinfo_from_handle 函数来把 handle 转换成 soinfo，正当我性高彩烈的打开 ida 查看它的 symble 的时候，发现没有这个函数，他不是导出函数（sblinker 5555），坑人呢呀，那么就只能照着 ida 一点一点的翻译它的代码了，找一个调用它的稍微短一点的函数，我找到的是 do_dlclose 函数，那么中间那一大坨就是 soinfo_from_handle 的实现了，返回值就是 soinfo_unload，的参数，接着我傻眼了，f5 之后这玩意没参数（逆天 f5），只能看汇编了，还好不长，就是这个 x12+0x18 中的地址值，切过去一看就是 v7[3] 那么就对了，我就可以写一个属于自己的 handle 转 soinfo

```
void* dlopen(const char* filename, int flag);
static soinfo* soinfo_from_handle(void* handle)

```

![](https://bbs.pediy.com/upload/attach/202109/799845_Z7RDXHUHPX7GHUE.png)  
![](https://bbs.pediy.com/upload/attach/202109/799845_MCV62FPURAPJQVJ.png)  
就是如下的这个函数，有些东西不好处理，比如它搞了好多全局变量，所以我们要从 maps 里面扫描 linker 的基址，剩下的直接抄就好了

```
_QWORD * getsoinfo(unsigned __int64 a1,void* base){
    unsigned int v2; // w19
    unsigned __int64 v3; // x11
    __int64 v4; // x9
    __int64 v5; // x10
    _QWORD *v6; // x12
    uint64 *bas1e= reinterpret_cast((char *) base + 0xFD468);
    uint64 *bas2= reinterpret_cast((char *) base + 0xFD460);
    _QWORD qword_FD468=*bas1e;
    _QWORD _dl_g_soinfo_handles_map=*bas2;
    unsigned __int64 v7; // x13
    __int64 v8; // x20
    __int64 v9; // x0
    __int64 v11; // [xsp+0h] [xbp-20h] BYREF
    char v12[8]; // [xsp+8h] [xbp-18h] BYREF
    if ( (a1 & 1) != 0 )
    {
        if ( qword_FD468 )
        {
            v3 = a1 - a1 / qword_FD468 * qword_FD468;
            v4 = qword_FD468 - 1;
            v5 = (qword_FD468 - 1) & qword_FD468;
            if ( qword_FD468 > a1 )
                v3 = a1;
            if ( !v5 )
                v3 = v4 & a1;
            v6 = *(_QWORD **)(_dl_g_soinfo_handles_map + 8 * v3);
            if ( v6 )
            {
                while ( 1 )
                {
                    v6 = (_QWORD *)*v6;
                    if ( !v6 )
                        break;
                    v7 = v6[1];
                    if ( v7 == a1 )
                    {
                        if ( v6[2] == a1 )
                        {
                            if ( v6[3] )
 
                                break;
                        }
                    }
                    else
                    {
                        if ( v5 )
                        {
                            if ( v7 >= qword_FD468 )
                                v7 -= v7 / qword_FD468 * qword_FD468;
                        }
                        else
                        {
                            v7 &= v4;
                        }
                        if ( v7 != v3 )
                            break;
                    }
                }
            }
        }
    }
    _QWORD * st= reinterpret_cast((char *) (v6[3]) );
    return st;
 
} 
```

```
void* ax=dlopen("libnative-lib.so",RTLD_NOW);
  __android_log_print(6,"r0ysue","%s",strerror(errno));
  char line[1024];
  int *startr;
  int *end;
  int n=1;
  FILE *fp=fopen("/proc/self/maps","r");
  while (fgets(line, sizeof(line), fp)) {
      if (strstr(line, "linker64") ) {
          __android_log_print(6,"r0ysue","%s", line);
          if(n==1){
              startr = reinterpret_cast(strtoul(strtok(line, "-"), NULL, 16));
              end = reinterpret_cast(strtoul(strtok(NULL, " "), NULL, 16));
 
 
          }
          else{
              strtok(line, "-");
              end = reinterpret_cast(strtoul(strtok(NULL, " "), NULL, 16));
          }
          n++;
 
 
      }
 
 
  }
 
   void** old_soinfo= reinterpret_cast(getsoinfo((unsigned __int64) ax, startr)); 
```

### 链接 & soinfo 的修正

这里修正 soinfo 直接用了结构体的 ->，由于我没有实现 soinfo 类所以这篇文章就到这里了。。。。。。。那是不可能的肉丝老师教我们永远不放弃，没有条件要创造条件也要解决这个问题，既然没实现 soinfo 我就用笨方法来实现就是 c 的偏移，而一个一个数 soinfo 当中的变量大小太过于麻烦，因为它的变量实在是太多了（555），于是我想到可以使用 ida 来辅助查看它的偏移，先直接查看 LoadTask 对象的 Load 函数  
![](https://bbs.pediy.com/upload/attach/202109/799845_EZD87TS62S3ZMF6.png)  
那么其实就是这里，只需要一一对应即可，也就是说

```
si_->base = *(si+16)
si_->size = *(si+24)
si_->load_bias =* (si+256)
si_->phnum = *(si+8)
si_->phdr = *(si)

```

那么修正代码就是

```
memcpy(&secstr,(char*)(start)+bb.sh_offset,bb.sh_size);
 mprotect((void*)PAGE_START((ElfW(Addr))((char *)start)),a.load_size_,PROT_WRITE|PROT_READ|PROT_EXEC);//申请读写执行权限因为我们要执行插件so的代码所以要执行权限
 __android_log_print(6,"r0ysue","size %s",strerror(errno));
 *reinterpret_cast((char *) old_soinfo + 16) = reinterpret_cast(a.load_start_);
 *(int*)((char*)(old_soinfo)+24)= a.load_size_;
 *reinterpret_cast((char *) old_soinfo + 256) = reinterpret_cast(start);
 *(int*)((char*)(old_soinfo)+8) = a.phdr_num_;
 *reinterpret_cast((char *) old_soinfo )= (uint64) a.loaded_phdr_; 
```

接下来就是链接过程，要将函数的绝对地址填上去，并且将引用的其他 so 的函数地址也填上去，这里安卓源码实现的函数是 prelink_image，非常的长仔细读一下就知道，它其实是可以抄的，这里我们主要修正的是导入表、导出表、重定向表、符号表、字符串表、重定位表、异常处理, 但是其实可以照着安卓源码和 ida 全部把它抄上，这里我从 elf 头开始获得了程序头然后再程序头中寻找 Dynamic 段，因为这些表都在动态段中，至于起始地址直接用 mmap 将上面 load 得到的 load_bias_ 映射过来即可

```
Elf64_Ehdr aa;
void* start= mmap(reinterpret_cast(a.load_bias_), sb.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
memcpy(&aa,start,sizeof(Elf64_Ehdr));//elf头解析，其实直接用a里面的也行我这里忘了
int secoff= aa.e_shoff;
int secsnum=aa.e_shnum;
Elf64_Shdr bb;
Elf64_Phdr cc;
memcpy (&cc,((char*)(start)+aa.e_phoff),sizeof(Elf64_Phdr));//将程序头表存入cc里面
for(int y=0;y
```

接下来就开始漫长的修正过程了，可以对照着 ida 都抄源码，主要对照着上面的段都要修复成功。主要就是要将相对地址转化为绝对地址，内容部分使用 Elf64_Dyn 这个结构体对他进行解析就好, 也就是 d_tag 等于 0x6ffffef5 时的导出表（so 一定要导出给 art 使用），等于 5 时的字符串表，等于 6 时的符号表等等这些都要修正，最终我只取了几个我的 so 中有的段类型进行修正

```
if(dd.d_tag==0x6ffffef5 ){//对导出表进行修正这个很重要导出失败则无法运行
                size_t   gnu_nbucket_ = reinterpret_cast((char*)start + dd.d_un.d_ptr)[0];
                // skip symndx
                uint32_t     gnu_maskwords_ = reinterpret_cast((char*)start + dd.d_un.d_ptr)[2];
                uint32_t  gnu_shift2_ = reinterpret_cast((char*)start + dd.d_un.d_ptr)[3];
 
                ElfW(Addr)*  gnu_bloom_filter_ = reinterpret_cast((char*)start + dd.d_un.d_ptr + 16);
                uint32_t*  gnu_bucket_ = reinterpret_cast(gnu_bloom_filter_ + gnu_maskwords_);
                // amend chain for symndx = header[1]
                uint32_t*  gnu_chain_ = reinterpret_cast( gnu_bucket_ +
                                                                     gnu_nbucket_-reinterpret_cast(
                                                                             (char *) start +
                                                                             dd.d_un.d_ptr)[1]);
                --gnu_maskwords_;
                uint32_t  flags_ = FLAG_GNU_HASH|flags_;
                *reinterpret_cast((char *) old_soinfo + 344) = gnu_nbucket_;
                *reinterpret_cast((char *) old_soinfo + 368) = gnu_maskwords_;
                *reinterpret_cast((char *) old_soinfo + 372) = gnu_shift2_;
                *reinterpret_cast<  ElfW(Addr)* *>((char *) old_soinfo +  376) = gnu_bloom_filter_;
                *reinterpret_cast((char *) old_soinfo + 352) = gnu_bucket_;
                *reinterpret_cast((char *) old_soinfo + 360) = gnu_chain_;
                *reinterpret_cast((char *) old_soinfo + 48) = *reinterpret_cast((char *) old_soinfo + 48) |FLAG_GNU_HASH;
 
            }
             if(dd.d_tag==2 ){
                *reinterpret_cast((char *) old_soinfo + 48)=dd.d_un.d_val / sizeof(ElfW(Rela));
             }
             if(dd.d_tag==0x17 ){//导入表修正
                 *reinterpret_cast((char *) old_soinfo + 104)= reinterpret_cast(
                         (char *) start + dd.d_un.d_ptr);
             }
             if(dd.d_tag==7){//重定位修正
                 *reinterpret_cast((char *) old_soinfo + 120)= reinterpret_cast(
                         (char *) start + dd.d_un.d_ptr);
             }
             if(dd.d_tag==5){//对字符串表进行修正
                 *reinterpret_cast((char *) old_soinfo + 56) = reinterpret_cast< char*>((char *) start+dd.d_un.d_ptr);
             }
             if(dd.d_tag==6){//对符号表进行修正
                 *reinterpret_cast((char *) old_soinfo + 64) = reinterpret_cast(
                         (char *) start + dd.d_un.d_ptr);
             }
             if(dd.d_tag==10){
                 *reinterpret_cast((char *) old_soinfo + 336) = reinterpret_cast(
                         (char *) start + dd.d_un.d_ptr);
             }
             if(dd.d_tag==8){
                 *reinterpret_cast((char *) old_soinfo + 336) =  dd.d_un.d_val / sizeof(ElfW(Rela));
             }
 
             if(dd.d_tag==0x6ffffff0){
                 *reinterpret_cast((char *) old_soinfo + 440) =  reinterpret_cast((char*)start + dd.d_un.d_ptr);
             }
             if(dd.d_tag==0x6fffffff){
                 *reinterpret_cast((char *) old_soinfo + 472) =  dd.d_un.d_val;
             }
 
             if(dd.d_tag==0x6ffffffe){
                 *reinterpret_cast((char *) old_soinfo + 464) = reinterpret_cast(
                         (char *) start + dd.d_un.d_ptr);
             }
 
 
              if(dd.d_tag==1){
                 mynedd[needed]=dd.d_un.d_val;
                 needed++;
 
             } 
```

这样其实如果我们被加固的 so 如果没有引用外部函数就可以正常使用了（哪个 so 可能没有外部函数呀），因为我们已经修复了导出表，但是为了追求完整性还需要补依赖, 比如我要是在被加壳的 so 中引用了 printf 或者__android_log_print 就会报错  
![](https://bbs.pediy.com/upload/attach/202109/799845_JH274SC6RSE8HBR.png)

### 修正依赖函数地址

由于我上面未实现 neededso 的装载与链接为了方便所以我下面对于依赖 so 的加载都采用 dlopen 和 dlsym 这种方式。这里可以看安卓源码中的 link_image 函数他调用了 relocate 来修复 JMPREL Relocation Table 表，所以我们跟进去看一下，其实这里就很清楚了，用迭代的方法获得 so 中引用的地址并且根据类型瑱回去我们的 so 当中。

```
bool soinfo::relocate(const VersionTracker& version_tracker, ElfRelIteratorT&& rel_iterator,
                    const soinfo_list_t& global_group, const soinfo_list_t& local_group) {
....
    ElfW(Word) type = ELFW(R_TYPE)(rel->r_info);
   ElfW(Word) sym = ELFW(R_SYM)(rel->r_info);
....
  if (!soinfo_do_lookup(this, sym_name, vi, &lsi, global_group, local_group, &s)) {
      return false;
      }
....
 switch (type) {
 
   ...
 }
 
 
                    }

```

由于我没有实现 soinfo 所以只能另辟蹊径，从原理出发用 dlopen 和 dlsym 另写一套方案。首先把上面的符号表和字符串表用起来，然后照着源码实现一个遍历的类（不实现用循环也可以，但是直接 ctrl+cv 就好了还不用动脑仁何乐而不为呢），而且要用到上面的导入库表，当然不知道安卓源码咋抽风了，就是没有 R_SYM 和 R_TYPE 这两个类型的定义我只能自己导入了，其实这两个就是对 info 的解析十分的简单

```
class plain_reloc_iterator {
 
public:
    plain_reloc_iterator(rel_t* rel_array, size_t count)
            : begin_(rel_array), end_(begin_ + count), current_(begin_) {}
 
    bool has_next() {
        return current_ < end_;
    }
 
    rel_t* next() {
        return current_++;
    }
public:
    rel_t* const begin_;
    rel_t* const end_;
    rel_t* current_;
 
 
};
 
#define ELFW(what) ELF64_ ## what
 
#define R_TYPE(sym) ((((Elf64_Xword)sym) << 32)
#define R_SYM(type) ((type) & 0xffffffff))
 
 
    char* strtab_= *reinterpret_cast((char *) old_soinfo + 56) ;//字符串表基址
    Elf64_Sym* symtab_= *reinterpret_cast((char *) old_soinfo + 64);//符号表基址
    plain_reloc_iterator myit(
            reinterpret_cast(*reinterpret_cast(
                    (char *) old_soinfo + 104)), *reinterpret_cast((char *) old_soinfo + 48));
    __android_log_print(6,"r0ysue","finish xxx%x",*reinterpret_cast((char *) old_soinfo + 48)); 
```

最后写一个循环回填就好了

```
    for (size_t idx = 0; myit.has_next(); ++idx) {
        const auto rel = myit.next();
 
 
        ElfW(Word) type = ELFW(R_TYPE)(rel->r_info);
        ElfW(Word) sym = ELFW(R_SYM)(rel->r_info);
 
 
        ElfW(Addr) sym_addr = 0;
        const char *sym_name = nullptr;
        const Elf64_Sym *s = nullptr;
        if (type == 0) {//不处理类型为0的部分
            continue;
        }
        sym_name = reinterpret_cast(strtab_+symtab_[sym].st_name);//根据get_string函数改编
 
 
        for(int s=0;s(dlsym(handle, sym_name));
            if(sym_addr==0)
                continue;
            else
//            __android_log_print(6, "r0ysue", "finish xxwwwwwwwwwwwwwwwx%p %s", sym_addr,sym_name);
break;
        }
 
        switch (type) {
            case 1026://我只有0x402类型的部分所以就简化处理了
                *reinterpret_cast((char *) start+ rel->r_offset)  = (sym_addr );
                break;
 
        }
 
    } 
```

跟到这里其实就完成了，下面看一下结果

```
//插件so当中的代码
extern "C"
JNIEXPORT jint JNICALL
Java_com_roysue_elfso_MainActivity_add(JNIEnv *env, jobject thiz, jint a, jint b) {
    printf("cxzcxzcxz");
    __android_log_print(6,"r0ysue","i am from 1.so %p",a);
   return a+b;
}

```

最后日志, 这样就完成和 art 的交互，后面还有执行 init_arry 函数和 Jni_Onload 也是十分的简单我就不实现了  
![](https://bbs.pediy.com/upload/attach/202109/799845_42TZAN3BW59UUCT.png)

### 总结

本篇文章只是一个基础用于对新手的 so 加壳入门，我粗略的实现了一个简单的 so 壳，算是我踩到的许多坑，其中导出表的修复就花费了好久的时间最终才成功，感谢大家观看

 

附件加壳 demo  
链接：https://pan.baidu.com/s/1MZSjotH8cs7wrOIAiZM5NQ  
提取码：kjvm

 

![](https://bbs.pediy.com/upload/attach/202109/799845_CFKTKD34FMREXDS.png)

[第五届安全开发者峰会（SDC 2021）10 月 23 日上海召开！限时 2.5 折门票 (含自助午餐 1 份）](https://www.bagevent.com/event/6334937)

[#基础理论](forum-161-1-117.htm) [#混淆加固](forum-161-1-121.htm) [#源码分析](forum-161-1-127.htm)
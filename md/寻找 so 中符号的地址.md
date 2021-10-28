> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270015.htm)

> 寻找 so 中符号的地址

*   [寻找 so 中符号的地址](#寻找so中符号的地址)
    *   [总述](#总述)
    *   [通过程序头获得符号地址](#通过程序头获得符号地址)
    *   [通过节头获得符号地址](#通过节头获得符号地址)
    *   [模仿安卓通过 hash 寻找符号](#模仿安卓通过hash寻找符号)
*   [总结](#总结)

寻找 so 中符号的地址
------------

### 总述

我们在使用 so 中的函数的时候可以使用 dlopen 和 dlsym 配合来寻找该函数的起始地址，但是在安卓高版本中 android 不允许打开白名单之外的 so，这就让我们很头疼，比如我们 hook libart.so 中的函数都没有办法来找到函数的具体位置，所以有了此文，这里介绍 3 种方法来获得符号的地址，网上方案挺多的我这里主要介绍原理

### 通过程序头获得符号地址

首先是如何找到 so 的首地址，这个 android 系统中提供了 maps 文件来记录 so 的内存分步，所以我们可以遍历 maps 文件来寻找 so 的首地址，如下

```
char line[1024];
 int *start;
 int *end;
 int n=1;
 FILE *fp=fopen("/proc/self/maps","r");
 while (fgets(line, sizeof(line), fp)) {
     if (strstr(line, "libart.so") ) {
         __android_log_print(6,"r0ysue","");
         if(n==1){
             start = reinterpret_cast(strtoul(strtok(line, "-"), NULL, 16));
             end = reinterpret_cast(strtoul(strtok(NULL, " "), NULL, 16));
         }
         else{
             strtok(line, "-");
             end = reinterpret_cast(strtoul(strtok(NULL, " "), NULL, 16));
         }
         n++;
     }
 } 
```

通过 elf 头结构我们可以找到程序头的地址，ndk 中自带了 elf.h 就很友好，就是 e_phoff 是相对于我们上面扫到的 so 首地址的偏移，e_phnum 是我们的程序头表中结构体的总个数，程序头中存着 elf 装载信息，如下图  
![](https://bbs.pediy.com/upload/attach/202110/799845_XCNQZCRM4NM33Y7.png)  
这里有一个问题就是上面的地址是 so 的起始地址，不是 load_bias, 所以我们在计算物理偏移的时候要减去一个首段的物理偏移，这里需要遍历程序头，得到第一个 e_type 为 1 的段记录下它的 p_vaddr。其中对我们索引符号地址有用的就是 Dynamic Segment, 也就是 type 为 2 的段，这部分可以写一个循环来找到，去记录下其中的字符串表和符号表就可以了  
![](https://bbs.pediy.com/upload/attach/202110/799845_NEBDPWYVNVTYUDS.png)

```
Elf64_Ehdr header;
 memcpy(&header, startr, sizeof(Elf64_Ehdr));
memcpy(&cc, ((char *) (startr) + header.e_phoff), sizeof(Elf64_Phdr));
 for (int y = 0; y < header.e_phnum; y++) {//寻找首段偏移
     memcpy(&cc, (char *) (startr) + header.e_phoff + sizeof(Elf64_Phdr) * y,
            sizeof(Elf64_Phdr));
     if (cc.p_type == 1) {
         phof =cc.p_paddr
         break;
     }
 
 }
for (int y = 0; y < header.e_phnum; y++) {
     memcpy(&cc, (char *) (startr) + header.e_phoff + sizeof(Elf64_Phdr) * y,
            sizeof(Elf64_Phdr));
     if (cc.p_type == 2) {
         Elf64_Dyn dd;
         for (y = 0; y == 0 || dd.d_tag != 0; y++) {
             memcpy(&dd, (char *) (startr) + cc.p_offset + y * sizeof(Elf64_Dyn) + 0x1000,
                    sizeof(Elf64_Dyn));
 
             if (dd.d_tag == 5) {//符号表
                 strtab_ = reinterpret_cast< char *>((char *) startr + dd.d_un.d_ptr - phof);
             }
             if (dd.d_tag == 6) {//字符串表
                 symtab_ = reinterpret_cast((
                         (char *) startr + dd.d_un.d_ptr - phof));
             }
             if (dd.d_tag == 10) {//字符串表大小
                 strsz = dd.d_un.d_val;
             }
 
 
         }
     }
 } 
```

接下来遍历符号表就可以了，这里有一个问题就是如何确定符号表的大小，这里观察一下 ida 反编译的结果, 发现符号表后面接的就是字符串表，那么用字符串表的首地址减去符号表的首地址就是符号表的大小, 之后再用 Elf64_Sym 结构体解析，st_value 就是该函数相对于 load_bias 的物理偏移，所以我们最后. 再减去之前记录的首段偏移即可  
![](https://bbs.pediy.com/upload/attach/202110/799845_5NV7BAHMSQBWETM.png)

```
  char strtab[strsz];
   memcpy(&strtab, strtab_, strsz);
   Elf64_Sym mytmpsym;
   for (n = 0; n < (long) strtab_ - (long) symtab_; n = n + sizeof(Elf64_Sym)) {//遍历符号表
   memcpy(&mytmpsym,(char*)symtab_+n,sizeof(Elf64_Sym));
if(strstr(strtab_+mytmpsym.st_name,"artFindNativeMethod"))
  {    __android_log_print(6,"r0ysue","%p %s",mytmpsym.st_value,strtab_+mytmpsym.st_name);
       break;
  }
   }
   return (char*)start+mytmpsym.st_value-phof;

```

### 通过节头获得符号地址

通过 elf 头结构我们也可以找到节头的地址，也就是 e_shoff，节头表相对于程序头表就友好许多，它的项非常多，唯一不好的一点就是它不会加载到内存中，所以 Execution View 中就没有这个东西，所以我们只能通过绝对路径找到它，手动解析文件

```
int fd;
void *start;
struct stat sb;
fd = open(lib, O_RDONLY);
fstat(fd, &sb);
start = mmap(NULL, sb.st_size, PROT_READ, MAP_PRIVATE, fd, 0);

```

在这种解析方式中我们在 elf 头中需要的值是 e_shoff、e_shentsize、e_shnum、e_shstrndx，就是节头表偏移，节大小，节个数，节头表字符串，不过我们最终的目标仍然是拿到符号表和字符串表，也就是下面的 symtab 和 strtab 中的 sh_offset  
![](https://bbs.pediy.com/upload/attach/202110/799845_Z8783JVB9XKFFJR.png)

```
Elf64_Ehdr header;
   memcpy(&header, start, sizeof(Elf64_Ehdr));
   int secoff = header.e_shoff;
   int secsize = header.e_shentsize;
   int secnum = header.e_shnum;
   int secstr = header.e_shstrndx;
   Elf64_Shdr strtab;
   memcpy(&strtab, (char *) start + secoff + secstr * secsize, sizeof(Elf64_Shdr));
   int strtaboff = strtab.sh_offset;
   char strtabchar[strtab.sh_size];
 
   memcpy(&strtabchar, (char *) start + strtaboff, strtab.sh_size);
   Elf64_Shdr enumsec;
   int symoff = 0;
   int symsize = 0;
   int strtabsize = 0;
   int stroff = 0;
   for (int n = 0; n < secnum; n++) {
 
       memcpy(&enumsec, (char *) start + secoff + n * secsize, sizeof(Elf64_Shdr));
 
 
       if (strcmp(&strtabchar[enumsec.sh_name], ".symtab") == 0) {
           symoff = enumsec.sh_offset;
           symsize = enumsec.sh_size;
 
       }
       if (strcmp(&strtabchar[enumsec.sh_name], ".strtab") == 0) {
           stroff = enumsec.sh_offset;
           strtabsize = enumsec.sh_size;
 
       }
 
 
   }

```

最后和上面一样遍历符号表即可可得到物理偏移

```
int realoff=0;
char relstr[strtabsize];
Elf64_Sym tmp;
memcpy(&relstr, (char *) start + stroff, strtabsize);
 
for (int n = 0; n < symsize; n = n + sizeof(Elf64_Sym)) {
    memcpy(&tmp, (char *)start + symoff+n, sizeof(Elf64_Sym));
    if(tmp.st_name!=0&&strstr(relstr+tmp.st_name,sym)){
        realoff=tmp.st_value;
        break;
    }
}
return realoff;

```

这种方式能够找到非导出符号的地址，还是有一定作用的，比如我在寻找 soinfo 地址的时候就用到了寻找 soinfo_map 在 linker 中的相对地址

### 模仿安卓通过 hash 寻找符号

这种方式就是 dlsym 的官方写法，由于 libart.so 这种 so 自动就会加载到内存种所以就不需要 dlopen 了，我们只需要在 map 里面找到它的首地址就可以了，代码和上面一样就不贴了, 这里我们主要看看官方如何实现的，一路追踪 do_dlopen 最终找到了函数 soinfo::gnu_lookup，这里面是他的主要实现逻辑，我们只需要实现它即可, 这里多了 4 个项我们之前没有提到，就是它的导出表 4 项，所以这种方法只能找到导出表当中的函数或者变量

```
size_t gnu_nbucket_ = 0;
    // skip symndx
    uint32_t gnu_maskwords_ = 0;
    uint32_t gnu_shift2_ = 0;
    ElfW(Addr) *gnu_bloom_filter_ = nullptr;
    uint32_t *gnu_bucket_ = nullptr;
    uint32_t *gnu_chain_ = nullptr;
    int phof = 0;
    Elf64_Ehdr header;
    memcpy(&header, startr, sizeof(Elf64_Ehdr));
    uint64 rel = 0;
    size_t size = 0;
    long *plt = nullptr;
    char *strtab_ = nullptr;
    Elf64_Sym *symtab_ = nullptr;
    Elf64_Phdr cc;
    memcpy(&cc, ((char *) (startr) + header.e_phoff), sizeof(Elf64_Phdr));
    for (int y = 0; y < header.e_phnum; y++) {
        memcpy(&cc, (char *) (startr) + header.e_phoff + sizeof(Elf64_Phdr) * y,
               sizeof(Elf64_Phdr));
        if (cc.p_type == 6) {
            phof = cc.p_paddr - cc.p_offset;//改用程序头的偏移获得首段偏移用之前的方法也行
        }
    }
    for (int y = 0; y < header.e_phnum; y++) {
        memcpy(&cc, (char *) (startr) + header.e_phoff + sizeof(Elf64_Phdr) * y,
               sizeof(Elf64_Phdr));
        if (cc.p_type == 2) {
            Elf64_Dyn dd;
            for (y = 0; y == 0 || dd.d_tag != 0; y++) {
                memcpy(&dd, (char *) (startr) + cc.p_offset + y * sizeof(Elf64_Dyn) + 0x1000,
                       sizeof(Elf64_Dyn));
 
                if (dd.d_tag == 0x6ffffef5) {//0x6ffffef5为导出表项
 
                    gnu_nbucket_ = reinterpret_cast((char *) startr + dd.d_un.d_ptr -
                                                                phof)[0];
                    // skip symndx
                    gnu_maskwords_ = reinterpret_cast((char *) startr + dd.d_un.d_ptr -
                                                                  phof)[2];
                    gnu_shift2_ = reinterpret_cast((char *) startr + dd.d_un.d_ptr -
                                                               phof)[3];
 
                    gnu_bloom_filter_ = reinterpret_cast((char *) startr +
                                                                       dd.d_un.d_ptr + 16 - phof);
                    gnu_bucket_ = reinterpret_cast(gnu_bloom_filter_ + gnu_maskwords_);
                    // amend chain for symndx = header[1]
                    gnu_chain_ = reinterpret_cast( gnu_bucket_ +
                                                               gnu_nbucket_ -
                                                               reinterpret_cast(
                                                                       (char *) startr +
                                                                       dd.d_un.d_ptr - phof)[1]);
 
                }
                if (dd.d_tag == 5) {
                    strtab_ = reinterpret_cast< char *>((char *) startr + dd.d_un.d_ptr - phof);
                }
                if (dd.d_tag == 6) {
                    symtab_ = reinterpret_cast((
                            (char *) startr + dd.d_un.d_ptr - phof));
                }
 
            }
        }
    } 
```

之后模仿 gnu_lookup 函数即可, hashmap 的查询方法

```
char* name_=symname;//直接抄的安卓源码
   uint32_t h = 5381;
   const uint8_t* name = reinterpret_cast(name_);
   while (*name != 0) {
       h += (h << 5) + *name++; // h*33 + c = h + h * 32 + c = h + h << 5 + c
   }
   int index=0;
   uint32_t h2 = h >> gnu_shift2_;
   uint32_t bloom_mask_bits = sizeof(ElfW(Addr))*8;
   uint32_t word_num = (h / bloom_mask_bits) & gnu_maskwords_;
   ElfW(Addr) bloom_word = gnu_bloom_filter_[word_num];
   n = gnu_bucket_[h % gnu_nbucket_];
   do {
       Elf64_Sym * s = symtab_ + n;
       char * sb=strtab_+ s->st_name;
       if (strcmp(sb ,reinterpret_cast(name_)) == 0 ) {
           break;
       }
   } while ((gnu_chain_[n++] & 1) == 0);
   Elf64_Sym * mysymf=symtab_+n;
   long* finaladdr= reinterpret_cast(sb->st_value + (char *) start-phof);
   return finaladdr; 
```

总结
--

这里介绍了三种得到符号地址的方法，都比较简单，只是我们写 hook 或者主动调用框架的一个基础，只有深刻的了解了 elf 格式才能完成我们的目标  
有兴趣可以加微信：roysu3 一起学习呀

[[培训] 优秀毕业生寄语：恭喜 id 咸鱼炒白菜拿到远超 3W 月薪的 offer，《安卓高级研修班》火热招生！！！](https://zhuanlan.kanxue.com/article-16096.htm)

[#基础理论](forum-161-1-117.htm)
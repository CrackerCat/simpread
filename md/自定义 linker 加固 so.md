> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1743378-1-1.html)

> [md] 其实自实现 linker 加固 so 与之前研究 windows 平台的 PE 文件的加密壳原理很相似。

![](https://avatar.52pojie.cn/data/avatar/001/28/98/46_avatar_middle.jpg)镇北看雪 _ 本帖最后由 镇北看雪 于 2023-2-8 17:05 编辑_  

其实自实现 linker 加固 so 与之前研究 windows 平台的 PE 文件的加密壳原理很相似。主要就是自定义文件格式加密 so，然后壳代码实现将加密的 so 文件加载，链接重定位并修正 soinfo（三部曲）。

自定义文件格式
=======

格式可以自己定义，只要在壳代码加载 so 时能够知道正确的格式就可以，下面是对标准的 ELF 文件格式的各部分数据进行简单的重组加密。

*   将原 so 文件的`Elf64_Ehdr`文件头进行重新定义为`Custom_Elf64_Ehdr`并使用 rc4 加密保存在 elf 文件末尾，旧的`Elf64Ehdr`被置 0。
*   将原 so 文件的`program header table`进行 rc4 加密后保存在 elf 文件末尾，旧的`program header table`被置 0。
*   将所有的`PT_LOAD`段进行 rc4 加密保存，文件偏移不变。
*   在被置 0 后的原`Elf64_Ehdr`文件头和`program header table`位置写入自定义的`Custom_Elf64_File`文件头，此结构用来保存加密后放置在文件末尾的`Custom_Elf64_Ehdr`和`program header table`的文件偏移和大小等信息。  
    ![](https://attach.52pojie.cn/forum/202302/08/125729oh4uudbq5qmwhjo6.png)
    
    **2052882-20230208002542048-755167136.png** _(38.8 KB, 下载次数: 1)_
    
    [下载附件](forum.php?mod=attachment&aid=MjU4OTMyMnwzM2Y1ZTU1YnwxNjc1OTE5ODI2fDB8MTc0MzM3OA%3D%3D&nothumb=yes)
    
    2023-2-8 12:57 上传
    

加壳代码
====

加壳代码通过解析一个标准的 so 文件将其格式保存为上面自定义的文件格式。

```
int do_pack(const char *inputfile_buffer, size_t inputfile_size, char *outfile_buffer, size_t outfile_size) 
{
    if(NULL == inputfile_buffer || 0 == inputfile_size || NULL == outfile_buffer)    return -1;

    Elf64_Ehdr* elf64_header = inputfile_buffer;
    //自定义的elf文件头（此结构在加密文件的开始，不进行加密）
    Custom_Elf64_File my_elf64_file = {0};
    my_elf64_file.elf_header_off = inputfile_size + elf64_header->e_phnum * elf64_header->e_phentsize;
    my_elf64_file.elf_header_size = elf64_header->e_ehsize;
    my_elf64_file.elf_program_header_table_num = elf64_header->e_phnum;
    my_elf64_file.elf_program_header_table_off = inputfile_size;
    my_elf64_file.elf_program_header_table_size = elf64_header->e_phnum * elf64_header->e_phentsize;

    //将原elf文件头字段进行重排
    Custom_Elf64_Ehdr my_elf64_header = {0};
    my_elf64_header.e_ehsize = elf64_header->e_ehsize;
    my_elf64_header.e_entry = elf64_header->e_entry;
    my_elf64_header.e_flags = elf64_header->e_flags;
    my_elf64_header.e_machine = elf64_header->e_machine;
    my_elf64_header.e_shentsize = elf64_header->e_shentsize;
    my_elf64_header.e_shnum = elf64_header->e_shnum;
    my_elf64_header.e_shoff = elf64_header->e_shoff;
    my_elf64_header.e_shstrndx = elf64_header->e_shstrndx;
    my_elf64_header.e_phentsize = elf64_header->e_phentsize;
    my_elf64_header.e_phnum = elf64_header->e_phnum;
    my_elf64_header.e_phoff = my_elf64_file.elf_program_header_table_off;
    strcpy(my_elf64_header.e_ident, ".csf");

    //加密原elf文件头
    RC4(&my_elf64_header, my_elf64_header.e_ehsize, rc4_key, strlen(rc4_key), outfile_buffer + my_elf64_file.elf_header_off);
    //加密progrem table header
    RC4(inputfile_buffer + elf64_header->e_phoff, elf64_header->e_phnum * elf64_header->e_phentsize, rc4_key, strlen(rc4_key), outfile_buffer + my_elf64_file.elf_program_header_table_off);
    //加密所有的PT_LOAD区段
    Elf64_Phdr *p = inputfile_buffer + elf64_header->e_phoff;
    for(int i = 0; i < elf64_header->e_phnum; i++){
        if(p->p_type == PT_LOAD){
            RC4(inputfile_buffer + p->p_offset, p->p_filesz, rc4_key, strlen(rc4_key), outfile_buffer + p->p_offset);
        }
        p = (char*)p + sizeof(Elf64_Phdr); 
    }

    //将原elf文件头抹去并替换为自定义elf文件头
    memset(outfile_buffer, 0, sizeof(Elf64_Ehdr));
    memcpy(outfile_buffer, &my_elf64_file, sizeof(Elf64_Ehdr));
    return 0;
}

```

用 ida 查看加固后的 so 是肯定无法解析的，使用 010editor 看一下加固后的 so，其数据全部都被加密了。

![](https://attach.52pojie.cn/forum/202302/08/125727m2ehi32izhfdimdx.png)

**2052882-20230208004843603-1075944142.png** _(61.62 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjU4OTMyMXw2NDljMWVmOHwxNjc1OTE5ODI2fDB8MTc0MzM3OA%3D%3D&nothumb=yes)

2023-2-8 12:57 上传

这里加壳代码并没有将加固 so 与壳 so 合并为一个 so 文件，现在的自定义 linker 加壳都是使用这种方式。

![](https://attach.52pojie.cn/forum/202302/08/170322n6oc68glv9rpz03c.png)

**图片. png** _(17.33 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjU4OTM4NXwwM2E3ZmM5YnwxNjc1OTE5ODI2fDB8MTc0MzM3OA%3D%3D&nothumb=yes)

2023-2-8 17:03 上传

壳 so（自定义 linker）
================

壳 so 负责将加固后的 so 文件解密，解析并加载，然后链接重定位，最后将壳 so 自己的 soinfo 修正为加固 so。

解密文件
----

```
//此结构在文件中不加密
typedef struct custom_elf64_file{
    Elf64_Off   elf_header_off;
    Elf64_Half  elf_header_size;
    Elf64_Off   elf_program_header_table_off;
    Elf64_Half  elf_program_header_table_num;
    Elf64_Half  elf_program_header_table_size;
}Custom_Elf64_File;
void unpack(char *elf_pack_data, Custom_Elf64_File my_elf64_file)
{
    //对重排列后原elf头进行解密
    RC4(reinterpret_cast<unsigned char *>(elf_pack_data + my_elf64_file.elf_header_off), my_elf64_file.elf_header_size,
        reinterpret_cast<unsigned char *>(rc4_key), strlen(rc4_key),
        reinterpret_cast<unsigned char *>(elf_pack_data + my_elf64_file.elf_header_off));
    //对progrem table header进行解密
    RC4(reinterpret_cast<unsigned char *>( elf_pack_data +
                                          my_elf64_file.elf_program_header_table_off),  my_elf64_file.elf_program_header_table_size,
        reinterpret_cast<unsigned char *>(rc4_key), strlen(rc4_key),
        reinterpret_cast<unsigned char *>( elf_pack_data +
                                          my_elf64_file.elf_program_header_table_off));

    //对各个PT_LOAD段进行解密
    Elf64_Phdr *p = reinterpret_cast<Elf64_Phdr *>( elf_pack_data +
                                                   my_elf64_file.elf_program_header_table_off);
    for(int i = 0; i < my_elf64_file.elf_program_header_table_num; i++){
        if(p->p_type == PT_LOAD){
            RC4(reinterpret_cast<unsigned char *>(elf_pack_data + p->p_offset), p->p_filesz,
                reinterpret_cast<unsigned char *>(rc4_key), strlen(rc4_key),
                reinterpret_cast<unsigned char *>(elf_pack_data + p->p_offset));
        }
        p = reinterpret_cast<Elf64_Phdr *>((char *) p + sizeof(Elf64_Phdr));
    }
}

```

解析自定义格式的 ELF 文件并加载
------------------

获取自定义格式的 ELF 文件头部的`Custom_Elf64_File`文件头，此文件头中保存了原始 elf 文件的文件头和 program header table 文件偏移和大小。 解析自定义格式的 ELF 文件并加载到内存中。将 program header table 保存在其他地方供之后使用。

```
Custom_Elf64_File my_elf64_file = {0};
memcpy((void *)&my_elf64_file, new_so_file_data, sizeof(Custom_Elf64_File));

//加载elf文件
load LoadElf(fd, my_elf64_file,new_so_file_data);
if (!LoadElf.ReadElfHeader() ||
    !LoadElf.ReadProgramHeader() ||
    !LoadElf.ReserveAddressSpace() ||
    !LoadElf.LoadSegments()) {                         
    munmap(new_so_file_data, sb.st_size);
    close(fd);
    return false;
}

//将内存中的program header table放在其他地方
Elf64_Phdr *program_header_table = (Elf64_Phdr*)malloc( my_elf64_file.elf_program_header_table_size);
memcpy(program_header_table, (char*)new_so_file_data + my_elf64_file.elf_program_header_table_off, my_elf64_file.elf_program_header_table_size);
LoadElf.SetPHPtr(program_header_table);


```

获取壳 so 的 soinfo
---------------

linker 程序通过函数`soinfo_from_handle`利用 handle 从`g_soinfo_handles_map`表中获取到对应的 soinfo，但是此函数并未导出，参考 https://www.cnblogs.com/r0ysue/p/15331621.html 调用 ida 反编译的代码。需要注意的是`g_soinfo_handles_map`在不同的 Android 版本中偏移不同。

![](https://attach.52pojie.cn/forum/202302/08/125725q169vttb58gt23tv.png)

**2052882-20230208011404720-1328327979.png** _(25.31 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjU4OTMyMHwzMjFiYWE0OHwxNjc1OTE5ODI2fDB8MTc0MzM3OA%3D%3D&nothumb=yes)

2023-2-8 12:57 上传

获取到壳 so 的 soinfo 后将其修正为加壳 so。

加载所有的依赖项
--------

解析出加固 so 所有的依赖项并加载，注意因为链接器命名空间的问题需要再修改壳 so 的 soinfo 之前加载依赖项，具体原因参考`https://www.cnblogs.com/revercc/p/17097115.html`.

prelink_image 解析. dynamic
-------------------------

参考 android 源码的 prelink_image 函数，解析加固 so 的. dynamic 节区信息并对 soinfo 进行修正。

![](https://attach.52pojie.cn/forum/202302/08/125723rngfj4llznfhfqqi.png)

**2052882-20230208012118285-461002886.png** _(95.54 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjU4OTMxOXwwMDVmMjc5MnwxNjc1OTE5ODI2fDB8MTc0MzM3OA%3D%3D&nothumb=yes)

2023-2-8 12:57 上传

对 soinfo 中其他信息进行修正

```
old_soinfo->size = LoadElf.load_size_;
old_soinfo->phnum = LoadElf.phdr_num_;
old_soinfo->phdr = LoadElf.loaded_phdr_;
//修改soinfo对应的文件信息
old_soinfo->st_dev_ = sb.st_dev;
old_soinfo->st_ino_ = sb.st_ino;
old_soinfo->file_offset_ = 0;

```

加载所有的依赖库
--------

通过 dlopen 加载所有的依赖库

```
//加载所有的依赖库
for(int s=0;s<needed;s++) {
    if(NULL == dlopen(old_soinfo->strtab_ + mynedd[s],RTLD_NOW)){
        goto exend;
    }
}


```

在加载完依赖项之后再修正壳 soinfo 的 base 和 load_bias，原因是为了避免以为链接器命名空间产生的问题，具体问题分析参考 [https://www.cnblogs.com/revercc/p/17097115.html](https://www.cnblogs.com/revercc/p/17097115.html)

```
old_soinfo->base = LoadElf.load_start_;
old_soinfo->load_bias = LoadElf.load_bias_;

```

link_image 链接重定位
----------------

调用`phdr_table_unprotect_segments`去除加固 so 各个段的保护属性后调用`link_image`进行链接重定位，重定位完成之后再通过`phdr_table_protect_segments`恢复各个段的属性。

```
//去除各个段的保护
phdr_table_unprotect_segments(LoadElf.phdr_table_, LoadElf.phdr_num_, LoadElf.load_bias_));

//进行链接重定位
link_image(old_soinfo, &LoadElf, needed, mynedd);

//恢复各个段的属性
phdr_table_protect_segments(LoadElf.phdr_table_, LoadElf.phdr_num_, LoadElf.load_bias_));

```

`link_image`需要对`.rel.plt`节区中的`R_AARCH64_JUMP_SLOT`重定位类型进行重定位，对`.rel.dyn`节区中的`R_AARCH64_RELATIVE, R_AARCH64_ABS64, R_AARCH64_GLOB_DAT`重定位类型进行重定位。同样可以参考 android 源码

![](https://attach.52pojie.cn/forum/202302/08/125721gv99m74t447enq57.png)

**2052882-20230208013456955-263340328.png** _(81.33 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjU4OTMxOHw4ZTAxYzEwMXwxNjc1OTE5ODI2fDB8MTc0MzM3OA%3D%3D&nothumb=yes)

2023-2-8 12:57 上传

脱壳
==

因为加固 so 被加载到内存中时其默认加载位置处的 elf 文件头和 program header table 都被抹去了。所以如果尝试 dump 整块内存，并利用映射基地址处的 elf 文件头去解析还原 elf 文件是无法实现的。（upx 的脱壳方式）

![](https://attach.52pojie.cn/forum/202302/08/125719for7csrcmiml6xyq.png)

**2052882-20230208105138513-563589762.png** _(108.52 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=MjU4OTMxN3w0MTNkYjcwOXwxNjc1OTE5ODI2fDB8MTc0MzM3OA%3D%3D&nothumb=yes)

2023-2-8 12:57 上传

要想脱壳就需要通过 soinfo_list 获取对应的 soinfo，soinfo 中保存了加载到内存中 elf 文件的 program header table，基地址，映射大小等信息。通过 soinfo 中保存的 elf 文件内存信息去 dump 内存并进行修复，这种方式其实和 pe 文件利用 LordPE 寻找到映射文件基地址和重定位表并进行 dump 修复很相似。

![](https://attach.52pojie.cn/forum/202302/08/125717boop77l33bb6w6oh.png)

**2052882-20230208105730570-235626711.png** _(41.88 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjU4OTMxNnw3NjRhZGFlMnwxNjc1OTE5ODI2fDB8MTc0MzM3OA%3D%3D&nothumb=yes)

2023-2-8 12:57 上传

参考：  
[https://www.cnblogs.com/theseventhson/p/16366038.html](https://www.cnblogs.com/theseventhson/p/16366038.html)![](https://avatar.52pojie.cn/images/noavatar_middle.gif)codown2017 看了大佬的分析，受益匪浅。膜拜 ![](https://avatar.52pojie.cn/data/avatar/000/35/89/70_avatar_middle.jpg) Light 紫星 大佬，学习到了，这种脱壳感觉可以在 load 之后的时候进行，不知道能不能脱掉![](https://avatar.52pojie.cn/data/avatar/001/28/98/46_avatar_middle.jpg)镇北看雪

> [Light 紫星 发表于 2023-2-8 15:26](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=45575114&ptid=1743378)  
> 大佬，学习到了，这种脱壳感觉可以在 load 之后的时候进行，不知道能不能脱掉

是的，主要是对 elf 文件格式的一个熟悉 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) a1046830 好东西 学习学习![](https://avatar.52pojie.cn/data/avatar/000/41/11/24_avatar_middle.jpg)雁字回时月 man 楼 大佬多出点这种教程![](https://avatar.52pojie.cn/data/avatar/000/73/84/40_avatar_middle.jpg)莫问刀 大佬多出点这种教程![](https://avatar.52pojie.cn/images/noavatar_middle.gif)不会想象的明天 这种东西值得学习，谢谢啦，辛苦 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) dl0618 感谢分享 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) qq882011 谢谢分享。。。。
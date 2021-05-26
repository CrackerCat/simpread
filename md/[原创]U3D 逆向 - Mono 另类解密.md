> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-267808.htm)

> [原创]U3D 逆向 - Mono 另类解密

        很久不来看雪了，这段时间过得真的是一言难尽。许久的发呆后决定干点有意义的事，于是乎就顺手来写一篇技术文章，算是本年的打卡。

        如题所示，U3D 的 Mono 解密，这个玩意我昨晚在我的技术群里貌似看到说 U3D 是要弃用了 Mono 这个机制，也不知道是真是假，趁着还没弃用，我也就来说说这个机制的一个另类解密。

       关于 Mono 这东西我就不啰嗦了，我就直接进入正题。众所周知，Mono 的加密主要是针对 **Assembly-CSharp.dll**，此 DLL 包含了游戏的所有功能性函数，并且可以通过工具 dnSpy.exe 加载后进行查看。

     ![](https://bbs.pediy.com/upload/attach/202105/826671_NZ3DPDQKXWMF4G7.jpg)

此 DLL 公开等于说源码公开，可以通过 C# 工程引入该 DLL 自写一个 GameObject 注入到游戏里调用游戏自带函数实现作弊。![](https://bbs.pediy.com/upload/attach/202105/826671_TZAX2EZ4ZZ4VU2P.jpg)![](https://bbs.pediy.com/upload/attach/202105/826671_DMVS37WHSDD9DEB.jpg)

    大多的加密手段就是对这 DLL 进行二进制处理，也就是把文件字节给进行处理。对于这种加密的处理方式很简单，Mono.dll 是 U3D 用来初始化并加载 dll 的一个模块，他里面有一个函数 **mono_image_open_from_data_with_name**，这里放一下他的函数代码

```
MonoImage *
mono_image_open_from_data_with_name (char *data, guint32 data_len, gboolean need_copy, MonoImageOpenStatus *status, gboolean refonly, const char *name)
{
    return mono_image_open_from_data_internal (data, data_len, need_copy, status, refonly, FALSE, name);
}

```

```
MonoImage *
mono_image_open_from_data_internal (char *data, guint32 data_len, gboolean need_copy, MonoImageOpenStatus *status, gboolean refonly, gboolean metadata_only, 
                const char *name)
{
    MonoCLIImageInfo *iinfo;
    MonoImage *image;
    char *datac;
 
    if (!data || !data_len) {
        if (status)
            *status = MONO_IMAGE_IMAGE_INVALID;
        return NULL;
    }
    datac = data;
    if (need_copy) {
        datac = (char *)g_try_malloc (data_len);
        if (!datac) {
            if (status)
                *status = MONO_IMAGE_ERROR_ERRNO;
            return NULL;
        }
        memcpy (datac, data, data_len);
    }
 
    image = g_new0 (MonoImage, 1);
    image->raw_data = datac;
    image->raw_data_len = data_len;
    image->raw_data_allocated = need_copy;
    image->name = (name == NULL) ? g_strdup_printf ("data-%p", datac) : g_strdup(name);
    iinfo = g_new0 (MonoCLIImageInfo, 1);
    image->image_info = iinfo;
    image->ref_only = refonly;
    image->metadata_only = metadata_only;
    image->ref_count = 1;
 
    image = do_mono_image_load (image, status, TRUE, TRUE);
    if (image == NULL)
        return NULL;
 
    return register_image (image);
}

```

        看不懂没关系，我们只需要关注他的 **data,data_len,name** 这三个参数，这三个分别表示当前被加载模块的二进制内容，二进制长度，模块名。大部分游戏厂商都会在这里判断模块名是否为 Assembly-CSharp，然后进行二进制内容解密。那么只需要用调试工具在这个函数下段，然后在这里分析结束的位置，然后直接 dump 即可。这里放一下大概代码

![](https://bbs.pediy.com/upload/attach/202105/826671_YD4X8Z2GWQCBCDM.jpg)

        直接在解密完毕的位置下断，然后把 rdi 给 dump 出来就行，得到的就是解密后的 dll。也可以自写脚本。

![](https://bbs.pediy.com/upload/attach/202105/826671_F974FNTB53ZAGES.jpg)

____________________________________________________________________________________________________________________________________________________________

上面介绍了 Assembly-CSharp 的一种加密和解密方式，虽说是加密，但是非常下饭，基本有手就行。今天就来说说另一种加密方式。小白鼠是国内某款游戏，现在应该应该不开放了，但是游戏依旧躺在我硬盘里。该游戏同样是加密 Assembly-CSharp，但是有个不同点就是该文件只有 1kb，这个就很可疑，因为这等于说这文件是空的，用十六进制打开文件也能看出里面无内容，那么游戏很可能是联网获取新的，或者是内存释放。那么除了去分析后解密，就没有办法拿到解密后的文件吗，答案是否定的。这个时候就需要上点硬技术了。

    回到 Mono 的源码，我们需要去分析一下他的一个流程。这里我就不去分析了，有一个帖子已经说得很详细。[](https://bbs.pediy.com/thread-264444.htm)[[原创] 关于 Unity 游戏 Mono 注入型外挂的一个检测思路 - 编程技术 - 看雪论坛 - 安全社区 | 安全招聘 | bbs.pediy.com](https://bbs.pediy.com/thread-264444.htm)

    大家可以看看这个帖子，主要是去熟悉一下 mono 的一个流程。我也直接请出主角。

```
static MonoImage *
register_image (MonoImage *image)
{
    MonoImage *image2;
    GHashTable *loaded_images = get_loaded_images_hash (image->ref_only);           //       重点关注对象
 
    mono_images_lock ();
    image2 = (MonoImage *)g_hash_table_lookup (loaded_images, image->name);
 
    if (image2) {
        /* Somebody else beat us to it */
        mono_image_addref (image2);
        mono_images_unlock ();
        mono_image_close (image);
        return image2;
    }
 
    GHashTable *loaded_images_by_name = get_loaded_images_by_name_hash (image->ref_only);
    g_hash_table_insert (loaded_images, image->name, image);                       //       重点关注对象
    if (image->assembly_name && (g_hash_table_lookup (loaded_images_by_name, image->assembly_name) == NULL))
        g_hash_table_insert (loaded_images_by_name, (char *) image->assembly_name, image);
    mono_images_unlock ();
 
    return image;
}

```

这里可以清楚的看到，U3D 直接把需要的加载的模块插入了一个 **HashTable**，然后完成模块的一个装载。这里我们先不管这个模块进行了如何加密和解密，既然你要扔给 U3D 托管，那么你不可能扔了一个加密模块的给 U3D 托管，除非你的 U3D 引擎进行了一个大规模的魔改。所以说，我们只需要找到这个 HashTable，然后去遍历一下里面的模块，我们是不是就能拿到解密后的 Assembly-CSharp，答案是肯定的。那么现在重点对象从 如何解密 Assembly-CSharp 变成了 如何使用 **loaded_images**。

    我们应该如何去找到这个 loaded_images？没办法，上分析。调试器打开游戏跳到 **mono_image_open_from_data_with_name** 这个函数然后对照源码进行分析。

![](https://bbs.pediy.com/upload/attach/202105/826671_9UVYK43EYFR23D4.jpg)

         从源码得知，register_image 是最后一个调用的函数，我们直接找到最后一个函数，进去后分析得知。

![](https://bbs.pediy.com/upload/attach/202105/826671_H6U7KTN57YX8X87.jpg)![](https://bbs.pediy.com/upload/attach/202105/826671_NW7SGA5UYAWT2M8.jpg)  

    拿到 loaded_images 后，我们下一步就是要去遍历这个 HashTabel, 两个办法，一个是自己去搭建 **GHashTable** 库，然后自己去跑一遍。第二个办法，自己去分析 g_hash_table_insert 函数的流程。我选择后者 =。=

        Ida 打开 mono.dll 后搜索 **g_hash_table_insert**，有一个

        ![](https://bbs.pediy.com/upload/attach/202105/826671_9GC768NM768SW2S.jpg)

            进去 F5 后无脑暴力分析。

![](https://bbs.pediy.com/upload/attach/202105/826671_SYHJ4FYGKN2H5AP.jpg)

        分析了一个大概流程后，打开 CE，用数据结构工具分析验证看看

![](https://bbs.pediy.com/upload/attach/202105/826671_CKENMD5RHGGU2TQ.jpg)

        很明显，这里应该就是模块列表，我们再随便进去一个去看看。**通过刚刚 IDA 的分析，0x0 是模块名，0x8 是 image。**

**![](https://bbs.pediy.com/upload/attach/202105/826671_RF8X85ZGEC8QU6X.jpg)**

**![](https://bbs.pediy.com/upload/attach/202105/826671_Z5J4AE4BUW3V8KM.jpg)**

            和我们刚刚分析的一致，现在我们还差一个东西，那就是 image 的结构，我们在源码查找一下。

```
struct _MonoImage {
           ………
           guint32 raw_data_len;
           char *raw_data;                   //模块二进制
           char *name;              //模块名
           …………
}

```

        这里就贴出重要的。问题来了，我们怎么知道他的偏移。别忘了，这里有

```
MonoImage *
mono_image_open_from_data_internal (char *data, guint32 data_len, gboolean need_copy, MonoImageOpenStatus *status, gboolean refonly, gboolean metadata_only, 
                const char *name)
{
    MonoCLIImageInfo *iinfo;
    MonoImage *image;
    char *datac;
 
    if (!data || !data_len) {
        if (status)
            *status = MONO_IMAGE_IMAGE_INVALID;
        return NULL;
    }
    datac = data;
    if (need_copy) {
        datac = (char *)g_try_malloc (data_len);
        if (!datac) {
            if (status)
                *status = MONO_IMAGE_ERROR_ERRNO;
            return NULL;
        }
        memcpy (datac, data, data_len);
    }
 
    image = g_new0 (MonoImage, 1);
    image->raw_data = datac;                                                                       <<<<<<<<<<<<<<<
    image->raw_data_len = data_len;                                                                <<<<<<<<<<<<<<<
    image->raw_data_allocated = need_copy;
    image->name = (name == NULL) ? g_strdup_printf ("data-%p", datac) : g_strdup(name);            <<<<<<<<<<<<<<<
    iinfo = g_new0 (MonoCLIImageInfo, 1);
    image->image_info = iinfo;
    image->ref_only = refonly;
    image->metadata_only = metadata_only;
    image->ref_count = 1;
 
    image = do_mono_image_load (image, status, TRUE, TRUE);
    if (image == NULL)
        return NULL;
 
    return register_image (image);
}

```

    我们 ida 返回去分析

```
__int64 __fastcall sub_18006CA20(__int64 data, unsigned int data_len, int need_cpy, _DWORD *status, __int64 refonly, char metadata_only, __int64 name)
{
  __int64 len; // r14
  _DWORD *v8; // rdi
  char v9; // si
  __int64 data_; // rbx
  __int64 data__; // rbp
  __int64 v12; // rax
  __int64 result; // rax
  __int64 v14; // rbx
  __int64 m_name; // rax
  signed __int64 v16; // rax
  __int64 v18; // rax
  __int64 v19; // rax
 
  len = data_len;
  v8 = status;
  v9 = need_cpy;
  data_ = data;
  if ( data && data_len )
  {
    data__ = data;
    if ( need_cpy )
    {
      v12 = sub_180004B80(data_len);
      data__ = v12;
      if ( !v12 )
      {
        if ( v8 )
          *v8 = 1;
        return 0i64;
      }
      sub_180314D40(v12, data_, len);
    }
    v14 = sub_180004AE0(1856i64);
    *(_BYTE *)(v14 + 0x1C) &= 0xFDu;
    *(_BYTE *)(v14 + 0x1C) |= 2 * (v9 & 1);
    *(_QWORD *)(v14 + 0x10) = data__;   //data
    *(_DWORD *)(v14 + 0x18) = len;      //data_len
    if ( name )
    {
      v16 = -1i64;
      while ( *(_BYTE *)(name + v16++ + 1) != 0 )
        ;
      m_name = sub_180004A10(name, (unsigned int)(v16 + 1));
    }
    else
    {
      m_name = sub_180006230("data-%p", data__);
    }
    *(_QWORD *)(v14 + 0x20) = m_name;    //name
    v18 = sub_180004AE0(408i64);
    *(_BYTE *)(v14 + 0x1C) &= 0xBFu;
    *(_BYTE *)(v14 + 0x1D) &= 0xFEu;
    *(_QWORD *)(v14 + 0x50) = v18;
    *(_DWORD *)v14 = 1;
    *(_BYTE *)(v14 + 0x1C) |= (refonly & 1) << 6;
    *(_BYTE *)(v14 + 0x1D) |= metadata_only & 1;
    v19 = sub_1800699B0(v14, v8, 1i64);
    if ( !v19 )
      return 0i64;
    result = sub_18006D6D0(v19);
  }
  else
  {
    if ( status )
      *status = 3;
    result = 0i64;
  }
  return result;
}

```

    可知

```
0x10 raw_data
0x18 raw_data_len
0x20 name

```

        我们再用 CE 看看  

![](https://bbs.pediy.com/upload/attach/202105/826671_YWXF8RVCXSHT2W7.jpg)

        可以看到，直接是一个标准的 PE 文件，说明这里存的是二进制内容。

        ![](https://bbs.pediy.com/upload/attach/202105/826671_TMQVP2G2QTY58YQ.jpg)

        到这里，我们就已经分析完了 loaded_images 的一个内存结构。接下来就是写一个遍历工具，然后打开游戏，运行工具，等待 DLL 生成。这里我用易语言写了一个。

        ![](https://bbs.pediy.com/upload/attach/202105/826671_9478QHY6SE9PYQW.jpg)

                        ![](https://bbs.pediy.com/upload/attach/202105/826671_WDQP64EPHVCW66X.jpg)

        **运行。。。**

            原文件

            ![](https://bbs.pediy.com/upload/attach/202105/826671_29NX2E3BPDRRCV7.jpg)

        dump 文件  

        ![](https://bbs.pediy.com/upload/attach/202105/826671_QV8VFXYKPEUEVVA.jpg)

  **Dnspy 打开**

   ![](https://bbs.pediy.com/upload/attach/202105/826671_3ET3AUPEJYR7ARM.jpg)

    到这里，就已经完美提取了。这里的话，个人觉得，也可以提取一些第三方挂钩 mono 的 DLL，但是本人没去试过。有兴趣的可以去试试。输入法和键盘同时出了问题，打字很难受，如果文章中存在错别字，，，见谅。。。。

                                                                                                   ![](https://bbs.pediy.com/upload/attach/202105/826671_SYPERDG4EFCDB8R.jpg)

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

上传的附件：

*   [mono 部分源码. zip](javascript:void(0)) （1.42MB，1 次下载）
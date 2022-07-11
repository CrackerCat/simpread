> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271557.htm)

> [原创] 简单完善《ARM64 OLLVM 反混淆》中的脚本

简单完善了一下无名侠大佬这篇文章：[ARM64 OLLVM 反混淆](https://bbs.pediy.com/thread-252321.htm)中的脚本。

 

主要改动：  
1. 增加 patch 二进制文件的代码；  
2. 研究代码可以发现 BL 指令调用的函数全部都是真实的逻辑，所以不根据 BL 指令分割基本块，并且含有 BL 指令一定是真实块；

```
......
    if (len(i.groups) > 0 and i.mnemonic != "bl") or i.mnemonic == 'ret':
        isNew = True
        block_item["eaddress"] = i.address
        block_item['ins'] = insStr
        block_item['insns'] = insStrlist
......
#如果某个代码块含有BL指令那一定是真实块
for b in list_blocks:
    if re.search( r'\s+bl\s+', list_blocks[b]['ins']):
        real_blocks.append(list_blocks[b]['saddress'])
......
```

3. 修改到 python3 环境下运行。  
修复效果：  
![](https://bbs.pediy.com/upload/attach/202202/734571_JADMFQMFZX233GX.png)

[看雪招聘平台创建简历并且简历完整度达到 90% 及以上可获得 500 看雪币～](https://job.kanxue.com/position-list.htm)

[#混淆加固](forum-161-1-121.htm)

上传的附件：

*   [fix.py](javascript:void(0)) （12.22kb，102 次下载）
*   [libvdog.so](javascript:void(0)) （2.02MB，52 次下载）
*   [libvdog.fix.so](javascript:void(0)) （2.02MB，42 次下载）
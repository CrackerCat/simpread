> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-253533.htm)

感谢无名侠大佬提供的思路

感谢《利用符号执行去除控制流平坦化》提供的思路

  
代码公开了，各种拼凑，比较龊，轻喷。希望能够有所帮助[deollvm64](https://github.com/GeT1t/deollvm64)   

### 在无名侠大佬 发表了《ARM64 OLLVM 反混淆》之后，尝试跑了一下附件的脚本，发现有很大缺陷。的确这篇文章只是提供了大体思路，后续优化的还需要很多。

### 仔细研究了现有资料，分享分享自己的心得，抛砖引玉。

  

先简单的总结下反 ollvm 混淆的思路

### 1. 获取真实块、序言、retn 块和无用块（也就是区别出真实块和垃圾块），通过静态分析提取真实块内在的某些规则

2. 确定真实块、序言和 retn 块的前后关系，利用符号执行或者 Unicorn 模拟执行，记录一个真实块到下一个真实块的关联 3. 修复真实块之间的关联

  

实际在反 ollvm 遇到最大的难题是 如何得到所有的真实块

  

遇到的一些问题：

1. 现有的文章 ollvm 混淆的 cfg 图和实际遇到的差别很大？

### ![](https://bbs.pediy.com/upload/attach/201908/682449_9EVSX83A7DXC5D8.jpg)  

  
![](https://bbs.pediy.com/upload/attach/201908/682449_RXXJ9PXC4TDQACP.jpg)  
  
    可以看到这两个生成的 cfg 图是完全不一样的，深究原因发现是因为在编译 Debug 时编译器并没有进行优化，而在编译 Release 的时候编译器优化指令后 "预分发器" 被优化掉了，生成的 cfg 就不同了，那么采用 《利用符号执行去除控制流平坦化》文章中寻找真实块是不可行的。  
2. 采用《ARM64 OLLVM 反混淆》文章中的方式寻找到的真实块存在无用块？     文章中采用寻找真实块的方式是： 如果某个代码块的引用大于 1，就可能是分发器，引用该分发器的代码块就可能为真实块。 那么思路其实是先找分发器再找真实块。那么这样就存在一个问题：如果子分发器后面再跟随子分发器，那么子分发器也被识别为了真实块？代码中不得已加入了特征码将这些子分发器移除。仔细研究最后发现这种方式还会漏掉某些真实块。那么这种方式其实也不是特别靠谱  
那么还有没有其他方式去寻找真实块？ 其实在研究真实块的逻辑关系时，debug 版本给了很多可以参考的位置  
第一次的思路： 《利用符号执行去除控制流平坦化》文章中寻找真实块是通过预分发器，那么既然预分发器已经被优化掉了，引用主分发器的是否就是真实块？ 那么这个思路就是：先找到主分发器，在查看谁引用了主分发器。被引用次数最多的块一定就是主分发器  
结果显而是失败的，原因在于在 release 下并不是所有的真实块都是引用主分发器，一些真实块居然引用了次分发器？  
第二次的思路： 其实第一次思路并没有出现错误的识别，而是出现了漏识别，那么我们可以在第一次的思路上进行优化： 从主分发器开始遍历接下来引用的块，记录每一次遍历到的块，如果发现这个块在记录列表里，那么他上次遍历到的块是真实块。 ![](https://bbs.pediy.com/upload/attach/201908/682449_3V5CF6TMKVX5RKM.jpg)  
类似二叉树，从上往下找，发现最后的节点的下一个节点是之前遍历到的，那么这个节点就是真实块。 这个思路能够将引用次分发器的真实块寻找到

```
get_relevant_nodes(supergraph, main_dispatcher_node, [])
 
def get_relevant_nodes(supergraph, node, founded_node):
    global relevant_nodes
    branch_nodes = list(supergraph.successors(node))
 
    founded_node.append(node)      
    for i in branch_nodes:
        if i not in founded_node:
            get_relevant_nodes(supergraph, i, founded_node)

```

  
大部分的真实块已经被识别到了，但是还是存在没有找到真实块，原因在于被 release 优化的部分存在一个逻辑关系，真实块的前继块有可能还是真实块，举一个例子  
![](https://bbs.pediy.com/upload/attach/201908/682449_YX6XZZS4EMR4C9C.jpg)  
  
这三个都是真实块，原因在于 release 将两个 puts 优化为了一个 puts。  
第三次的思路：这一次只需要找到真实块前面的真实块，规则是真实块的前继块的后继如果只有一个，那么则也是真实块。有点绕，看代码

```
def get_relevant_nodes(supergraph, node, founded_node):
    global relevant_nodes
    branch_nodes = list(supergraph.successors(node))
 
    if len(branch_nodes) == 1 and branch_nodes[0] in founded_node:
        if node in relevant_nodes:
            for i in supergraph.predecessors(node):
                relevant_nodes.append(i)
        else:
            relevant_nodes.append(node)
    else:
        founded_node.append(node)      
        for i in branch_nodes:
            if i not in founded_node:
                get_relevant_nodes(supergraph, i, founded_node)

```

  
通过这三次优化 已经能够将使用 -fla 参数 被 ollvm 混淆的的真实块识别出来了，并且已经修复（这个是拿真实的样本测试的）![](https://bbs.pediy.com/upload/attach/201908/682449_3W9F3DXNUPDJ4BX.jpg)  
仍未解决的点：1. 在使用 -bcf (虚假控制流) 参数之后 有一些块是死循环，而本来一个真实块被死循环打乱成为了两个真实块，例如  
![](https://bbs.pediy.com/upload/attach/201908/682449_BS3N7UKDFKD6GKG.jpg)  
这个时候真实块 1 没有被识别出来，需要更进一步的优化。  
这个没太深入研究了，可以直接将真实块 1 手动的加入到真实块列表中  
2. 在修复真实块的条件跳转逻辑时，有时候条件完全是相反的，这个后续还需要优化  
参考 1. [利用符号执行去除控制流平坦化](https://security.tencent.com/index.php/blog/msg/112)  
2. [ARM64 OLLVM 反混淆](https://bbs.pediy.com/thread-252321.htm)3.[https://github.com/cq674350529/deflat](https://github.com/cq674350529/deflat) 

[看雪学院推出的专业资质证书《看雪安卓应用安全能力认证 v1.0》（中级和高级）！](https://bbs.pediy.com/thread-265424.htm)

最后于 2020-3-8 23:16 被 FraMeQ 编辑 ，原因：
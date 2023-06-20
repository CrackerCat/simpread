> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1798622-1-1.html)

> [md]# 新手学习 Vmp 之控制流程图生成控制流程图的生成对于反混淆分析来说是非常重要的一步，这里记录一下我研究的过程，以 Vmp2 为例子。

![](https://avatar.52pojie.cn/data/avatar/000/48/36/37_avatar_middle.jpg)fjqisba _ 本帖最后由 fjqisba 于 2023-6-17 16:20 编辑_  

新手学习 Vmp 之控制流程图生成
=================

控制流程图的生成对于反混淆分析来说是非常重要的一步，这里记录一下我研究的过程，以 Vmp2 为例子。

这里我的环境准备如下:

Visual Studio + IDA SDK + Capstone + Unicorn + Graphviz

IDA SDK 插件环境，主要是有一些 API 可以调用，方便编写代码，X64Dbg 插件环境可以替代之。

Capstone，一个很不错的反汇编引擎，IDA 自带的反汇编引擎不太好用，用这个替代之。

Unicorn，指令模拟执行，用来跟踪指令。

Graphviz，一个绘图工具，可以将控制流程图可视化。

要生成流程图，首先使用 unicron 引擎对指令进行跟踪，大致步骤如下:

1、使用 uc_mem_map 和 uc_mem_write 函数填充内存区域和堆栈

2、uc_hook_add 设置监视函数，每次执行指令前检查退出条件，例如当前指令位于 text 区段且上一条指令是 ret 的时候，基本上就是 vmp 结束的时候了。

3、uc_emu_start 进行 trace，拿到所有的指令跟踪数组。

之后是根据这些地址动态生成控制流程图，这里需要了解一下基本块这个概念。

核心逻辑如下:

```
bool VmpTraceFlowGraph::GenerateBasicFlowData(std::vector<ea_t>& traceList)
{
        if (!traceList.size()) {
                return false;
        }
        cs_insn* curIns;
        VmpTraceFlowNode* currentNode = createNode(traceList[0]);;
        for (unsigned int n = 0; n < traceList.size(); ++n) {
                const ea_t& curAddr = traceList[n];
                if (!DisasmManager::DecodeInstruction(curAddr, curIns)) {
                        return false;
                }
                //不管是什么指令,都立即追加到当前基本块
                if (!currentNode->bTraced) {
                        currentNode->addrList.push_back(curAddr);
                        updateInstructionToBlockMap(curAddr, currentNode);
                }
                //判断是否为终止指令
                if (isEndIns(curIns)) {
                        //检查是否为最后一条指令
                        if (n + 1 >= traceList.size()) {
                                break;
                        }
                        currentNode->bTraced = true;
                        //这里开始进行核心判断
                        ea_t nextNodeAddr = traceList[n + 1];
                        VmpTraceFlowNode* nextNode = instructionToBlockMap[nextNodeAddr];
                        linkEdge(curAddr, nextNodeAddr);
                        //下一个节点是新节点
                        if (!nextNode) {
                                currentNode = createNode(nextNodeAddr);
                        }
                        //已访问过该节点,且节点指向Block头部
                        else if (nextNode->nodeEntry == nextNodeAddr) {
                                currentNode = nextNode;
                        }
                        else {
                                //节点指向已有区块其它地址,需要对区块进行分割
                                currentNode = splitBlock(nextNode, nextNodeAddr);
                        }
                }
        }
        return true;
}

```

再进行节点合并优化，核心代码是这样的:

```
void VmpTraceFlowGraph::MergeNodes()
{
        //已确定无法合并的节点
        std::set<ea_t> badNodeList;
        bool bUpdateNode;
        do
        {
                bUpdateNode = false;
                std::map<ea_t, VmpTraceFlowNode>::iterator it = nodeMap.begin();
                while (it != nodeMap.end()) {
                        ea_t nodeAddr = it->first;
                        if (badNodeList.count(nodeAddr)) {
                                it++;
                                continue;
                        }
                        //判断合并条件
                        //条件1,指向子节点的边只有1条
                        if (toEdges[nodeAddr].size() == 1) {
                                ea_t fromAddr = *toEdges[nodeAddr].begin();
                                VmpTraceFlowNode* fatherNode = instructionToBlockMap[fromAddr];
                                //条件2,父节点指向的边也只有1条
                                if (fromEdges[fromAddr].size() == 1) {
                                        //条件3,子节点不能指向父节点
                                        if (!fromEdges[nodeAddr].count(fatherNode->addrList[fatherNode->addrList.size() - 1])) {
                                                executeMerge(fatherNode, &it->second);
                                                bUpdateNode = true;
                                                it = nodeMap.erase(it);
                                                continue;
                                        }
                                }
                        }
                        badNodeList.insert(nodeAddr);
                        it++;
                }
        } while (bUpdateNode);
}

```

最后是将流程图转换成 dot 语言，核心代码如下:

```
std::string VmpTraceFlowGraph::DumpGraph()
{
        std::stringstream ss;
        ss << "strict digraph \"hello world\"{\n";
        cs_insn* tmpIns = 0x0;

        char addrBuffer[0x10];
        for (std::map<ea_t, VmpTraceFlowNode>::iterator it = nodeMap.begin(); it != nodeMap.end(); ++it) {
                VmpTraceFlowNode& node = it->second;
                sprintf_s(addrBuffer, sizeof(addrBuffer), "%08X", it->first);
                ss << "\"" << addrBuffer << "\"[label=\"";
                for (unsigned int n = 0; n < node.addrList.size(); ++n) {
                        //测试代码
                        if (n > 20 && (n != node.addrList.size() - 1)) {
                                continue;
                        }
                        DisasmManager::DecodeInstruction(node.addrList[n], tmpIns);
                        sprintf_s(addrBuffer, sizeof(addrBuffer), "%08X", node.addrList[n]);
                        ss << addrBuffer << "\t" << tmpIns->mnemonic << " " << tmpIns->op_str << "\\n";
                }
                ss << "\"];\n";
        }

        for(std::map<ea_t, std::unordered_set<ea_t>>::iterator it = fromEdges.begin(); it != fromEdges.end(); ++it){
                std::unordered_set<ea_t>& edgeList = it->second;
                for (std::unordered_set<ea_t>::iterator edegIt = edgeList.begin(); edegIt != edgeList.end(); ++edegIt) {
                        VmpTraceFlowNode* fromBlock = instructionToBlockMap[it->first];
                        sprintf_s(addrBuffer, sizeof(addrBuffer), "%08X", fromBlock->nodeEntry);
                        ss << "\"" << addrBuffer << "\" -> ";
                        sprintf_s(addrBuffer, sizeof(addrBuffer), "%08X", *edegIt);
                        ss << "\"" << addrBuffer << "\";\n";
                }
        }
        ss << "\n}";
        return ss.str();
}

```

得到文件后，调用 dot 命令行打印出流程图

```
dot graph.txt -T png -o vmp2.png

```

最后得到的结果是这样的  
![](https://attach.52pojie.cn/forum/202306/17/161547scznznju4pkpczwp.png)

**vmp2.png** _(1.16 MB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjYyMjE1Mnw5Yjk0NWFjZnwxNjg3MjEzMTg1fDB8MTc5ODYyMg%3D%3D&nothumb=yes)

2023-6-17 16:15 上传

最后是参考代码  
 ![](https://static.52pojie.cn/static/image/filetype/zip.gif) [TestVmp.7z](forum.php?mod=attachment&aid=MjYyMjE1NHw3YmFhNGQ5NHwxNjg3MjEzMTg1fDB8MTc5ODYyMg%3D%3D) _(4.7 KB, 下载次数: 19, 售价: 5 CB 吾爱币)_ 2023-6-17 16:19 上传 点击文件名下载附件 C++ 售价: 5 CB 吾爱币  [[记录]](forum.php?mod=misc&action=viewattachpayments&aid=2622154) 下载积分: 吾爱币 -1 CB ![](https://avatar.52pojie.cn/images/noavatar_middle.gif)sweetcat 来看看!!!!![](https://avatar.52pojie.cn/images/noavatar_middle.gif)gly198752 学习了，大佬大佬 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) sxlcsh 支持发帖 辛苦啦 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) sec-u 支持发帖 辛苦啦![](https://avatar.52pojie.cn/images/noavatar_middle.gif)空欢 支持发帖 辛苦啦 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) Lemon1001 谢谢分享，学习一下 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) moruye 楼主辛苦，刚好下载学习研究下 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) zjh889 谢谢大师分享，俺学习看看！![](https://avatar.52pojie.cn/images/noavatar_middle.gif)至尊 Cracker 编程写流程图，可以的
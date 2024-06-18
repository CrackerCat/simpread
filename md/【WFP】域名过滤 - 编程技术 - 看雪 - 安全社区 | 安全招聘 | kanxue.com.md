> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282189.htm)

> 【WFP】域名过滤

前言
==

近段时间做了一些与网络管控的需求。感谢 @zx_838741 的入门贴：  
[WFP 网络过滤驱动——限制网站访问](https://bbs.kanxue.com/thread-265597.htm)  
帖子中看到了这么一个思路：  
![](https://bbs.kanxue.com/upload/attach/202406/905357_VPA4K6BXNX2H2PJ.webp)就想着去实现一下。

原理
==

在 **FWPM_LAYER_DATAGRAM_DATA_V4** 层拦截 dns 流量，从而解析出域名。他在整个 WFP 生命周期中的位置是，如下所示：

```
客户端->服务器
bind:      FWPM_LAYER_ALE_RESOURCE_ASSIGNMENT_V4
send:      FWPM_LAYER_ALE_AUTH_CONNECT_V4
Data：     FWPM_LAYER_DATAGRAM_DATA_V4      //// 这里
UDP：      FWPM_LAYER_OUTBOUND_TRANSPORT_V4
IP ：      FWPM_LAYER_OUTBOUND_IPPACKET_V4
 
服务器->客户端
IP:  FWPM_LAYER_INBOUND_IPPACKET_V4
UDP：FWPM_LAYER_ALE_AUTH_RECV_ACCEPT_V4
Data：FWPM_LAYER_DATAGRAM_DATA_V4         ////  这里

```

代码
==

### 0：WFP 经典步骤，注册回调，添加回调，添加子层，添加过滤引擎;

```
NTSTATUS RegisterNetworkFilterUDP(PDEVICE_OBJECT DeviceObject)
{
     
    NTSTATUS status = STATUS_SUCCESS;
 
    // open filter engine session
    status = FwpmEngineOpen(NULL, RPC_C_AUTHN_WINNT, NULL, NULL, &EngHandle);
    if (!NT_SUCCESS(status)) {
        DbgPrint("[*] failed to open filter engine\n");
        return status;
    }
 
    // register callout in filter engine
    FWPS_CALLOUT callout = {};
    callout.calloutKey = EXAMPLE_CALLOUT_UDP_GUID;
    callout.flags = 0;
    callout.classifyFn = ClassifyCallback;
    callout.notifyFn = NotifyCallback;
    callout.flowDeleteFn = nullptr;
     
    status =  FwpsCalloutRegister(DeviceObject, &callout, &CalloutId);
    if (!NT_SUCCESS(status))
    {
        DbgPrint("[*] failed to register callout in filter engine\n");
        return status;
    }
 
    // add callout to the system
    FWPM_CALLOUT calloutm = { };
    calloutm.flags = 0;                         
    calloutm.displayData.name = L"example callout udp";
    calloutm.displayData.description = L"example PoC callout for udp ";
    calloutm.calloutKey = EXAMPLE_CALLOUT_UDP_GUID;
    calloutm.applicableLayer = FWPM_LAYER_DATAGRAM_DATA_V4; //dns流量拦截层
     
 
    status =  FwpmCalloutAdd(EngHandle, &calloutm, NULL, &SystemCalloutId);
    if (!NT_SUCCESS(status)) {
        DbgPrint("[*] failed to add callout to the system \n");
        return status;
    }
 
    // create a sublayer to group filters (not actually required
    FWPM_SUBLAYER sublayer = {};
    sublayer.displayData.name = L"PoC sublayer example filters";
    sublayer.displayData.name = L"PoC sublayer examle filters";
    sublayer.subLayerKey = EXAMPLE_FILTERS_SUBLAYER_GUID;
    sublayer.weight = 65535;
 
 
    status =  FwpmSubLayerAdd(EngHandle, &sublayer, NULL);
    if (!NT_SUCCESS(status)) {
        DbgPrint("[*] failed to create a sublayer\n");
        return status;
    }
 
    // add a filter that references our callout with no conditions
    UINT64 weightValue = 0xFFFFFFFFFFFFFFFF;                               
    FWP_VALUE weight = {};
    weight.type = FWP_UINT64;
    weight.uint64 = &weightValue;
 
    // process every packet , no conditions
    FWPM_FILTER_CONDITION conditions[1] = { 0 };                                        \
 
        FWPM_FILTER filter = {};
    filter.displayData.name = L"example filter callout udp";
    filter.displayData.name = L"example filter calout udp";
    filter.layerKey = FWPM_LAYER_DATAGRAM_DATA_V4; //dns流量拦截层
    filter.subLayerKey = EXAMPLE_FILTERS_SUBLAYER_GUID;
    filter.weight = weight;
    filter.numFilterConditions = 0;
    filter.filterCondition = conditions;
    filter.action.type = FWP_ACTION_CALLOUT_INSPECTION;
    filter.action.calloutKey = EXAMPLE_CALLOUT_UDP_GUID;
     
 
    return FwpmFilterAdd(EngHandle, &filter, NULL, &FilterId);
}

```

### 1：ClassifyCallback 回调中获取 dns 流量

没啥好说的，就是使用 NdisGetDataBuffer 获取 dns 流量存入。  
回调前面有三个判断：是不是 UDP 包，端口是不是 53，数据包出去的还是进来的

```
VOID ClassifyCallback(
    const FWPS_INCOMING_VALUES* inFixedValues,
    const FWPS_INCOMING_METADATA_VALUES* inMetaValues,
    void* layerData,
    const void* classifyContext,
    const FWPS_FILTER* filter,
    UINT64 flowContext,
    FWPS_CLASSIFY_OUT* classifyOut
) {
    classifyOut->actionType = FWP_ACTION_PERMIT;
 
    if (inFixedValues->incomingValue[FWPS_FIELD_DATAGRAM_DATA_V4_IP_PROTOCOL].value.uint8 != IPPROTO_UDP ||
        inFixedValues->incomingValue[FWPS_FIELD_DATAGRAM_DATA_V4_IP_REMOTE_PORT].value.uint16 != 53 ||
        inFixedValues->incomingValue[FWPS_FIELD_DATAGRAM_DATA_V4_DIRECTION].value.uint32 != FWP_DIRECTION_OUTBOUND)
    {
        return;
    }
    ULONG UdpHeaderLength = 0;
    BOOL isOutBound = FALSE;
    PNET_BUFFER pNetBuffer = NET_BUFFER_LIST_FIRST_NB((PNET_BUFFER_LIST)layerData);
    if (pNetBuffer == NULL)
    {
        goto end;
    }
    ULONG UdpContentLength = NET_BUFFER_DATA_LENGTH(pNetBuffer);
    if (UdpContentLength == 0)
    {
        goto end;
    }
 
    UINT8 direction =inFixedValues->incomingValue[FWPS_FIELD_DATAGRAM_DATA_V4_DIRECTION].value.uint8;
    if (direction == FWP_DIRECTION_OUTBOUND)
    {
        UdpHeaderLength = inMetaValues->transportHeaderSize;
        if (UdpHeaderLength == 0 || UdpContentLength < UdpHeaderLength) {
            goto end;
        }
        UdpContentLength -= UdpHeaderLength;
        isOutBound = TRUE;
    }
    else
    {
        goto end;
    }
 
    PVOID UdpContent = ExAllocatePoolWithTag(NonPagedPool, UdpContentLength + UdpHeaderLength, 'Tsnd');
    if (UdpContent == NULL)
    {
        goto end;
    }
 
    PVOID ndisBuffer = NdisGetDataBuffer(pNetBuffer, UdpContentLength + UdpHeaderLength, UdpContent, 1, 0);
    if (ndisBuffer == nullptr) {
        goto end;
    }
    ResolveDnsPacket(ndisBuffer, UdpContentLength + UdpHeaderLength, UdpHeaderLength);
end:
    return;
}

```

### 2. 从 DNS 流量中解析出域名

先上代码

```
VOID ResolveDnsPacket(void* packet, size_t packetSize , size_t udpHeaderLen){
    PHYSICAL_ADDRESS highestAcceptableWriteBufferAddr;
    highestAcceptableWriteBufferAddr.QuadPart = MAXULONG64;
    if (packetSize < sizeof(DNS_HEADER))
    {
        return;
    }
    DNS_HEADER* dnsHeader = (DNS_HEADER*)packet;
    if (dnsHeader->IsResponse)
    {
        return;
    }
    size_t dnsDataLength = packetSize - sizeof(DNS_HEADER);
 
    if (dnsDataLength >= packetSize) {
        return;
    }
 
    char* dnsData = (char*)packet + sizeof(DNS_HEADER) + udpHeaderLen;
    char* domainName = reinterpret_cast(MmAllocateContiguousMemory(128, highestAcceptableWriteBufferAddr));
    if (domainName == NULL)
    {
        return;
    }
 
    memset(domainName, '\0', 128);
 
    bool isSuccess = TRUE;
    size_t domainNameLength = 0;
 
    while (dnsDataLength > 0) {
        const char length = *dnsData;
        if (length == 0) {
            break;
        }
        if (length >= packetSize || length > 256)
        {
            break;
        }
 
        if (domainNameLength + 1 > 128)
        {
            isSuccess = FALSE;
            break;
        }
        char domainNameStr = *(dnsData + 1);
        // 检查第一个字符是否是可读字符
        if (isprint(domainNameStr) == FALSE) {
            isSuccess = FALSE;
            break;
        }
        if (domainNameLength != 0) {
            domainName[domainNameLength] = '.';
            domainNameLength++;
        }
        memcpy(domainName + domainNameLength, dnsData + 1, length);
        domainNameLength += length;
        dnsDataLength -= *dnsData + 1;
        dnsData += *dnsData + 1;
    }
    if (isSuccess)
    {
        // TODO: OnDnsQueryEvent;
        DbgPrint("ResolverDnsPacket: %s \n", domainName);
    }
    MmFreeContiguousMemory(domainName);
} 
```

在前面使用 NdisGetDataBuffer 获取了 DNS 数据包，让我们看看 DNS 数据包长啥样，从而进一步从数据中解析出域名。

### 3. 从 DNS 流量分析

##### 获取的 DNS 流量数据如下所示：

![](https://bbs.kanxue.com/upload/attach/202406/905357_2YA78PBVHNZZM67.webp)

##### 前 8 个字节是 UDP 报文头：

![](https://bbs.kanxue.com/upload/attach/202406/905357_FHVY7DYD543EUWJ.webp)  
![](https://bbs.kanxue.com/upload/attach/202406/905357_HUKM5XZC48VD6A6.webp)

##### 接着就是 12 个字节的 DNS 数据头：

![](https://bbs.kanxue.com/upload/attach/202406/905357_TWZGSUP7WMGDCHA.webp)  
![](https://bbs.kanxue.com/upload/attach/202406/905357_V3HYYF3C9NSXXUX.webp)  
头部详细说明：

```
事务ID (Transaction ID): 8F FE
表示唯一的事务ID
 
标志 (Flags): 01 00
QR (1位): 0，表示这是一个查询。
Opcode (4位): 0，表示这是一个标准查询。
AA (1位): 0，表示不是授权回答。
TC (1位): 0，表示消息未被截断。
RD (1位): 1，表示请求递归查询。
RA (1位): 0，表示服务器不支持递归查询。
Z (3位): 000，保留位。
RCODE (4位): 0000，表示响应码为无错误。
 
问题数 (Number of Questions): 00 01
表示在查询部分中有1个问题。
 
回答数 (Number of Answer RRs): 00 00
表示回答部分中没有资源记录。
 
权威部分记录数 (Number of Authority RRs): 00 00
表示权威部分中没有资源记录。
 
附加部分记录数 (Number of Additional RRs): 00 00
表示附加部分中没有资源记录

```

##### 最后包含域名的 body 部分

![](https://bbs.kanxue.com/upload/attach/202406/905357_NAU37QSYRTWU8NT.webp)

```
03                 www的长度
77 77 77           表示 www
06                 kanxue的长度
6B 61 6E 78 75 65  表示kanxue
03                 com的长度
63 6F 6D           表示com

```

效果截图
====

![](https://bbs.kanxue.com/upload/attach/202406/905357_UQ5Q474CEF2HPQK.webp)

其他
==

1.  本章的方法是通过过滤 dns 请求的 udp 流程，解析出域名。后面还有可以过滤 dns 响应的 udp 数据，不仅可以解析出域名，还能解析出域名服务器返回的 ip，这个后续有时间再搞吧
2.  如果域名解析使用其他的怪招，比如加密，tcp 协议等，这个方式还是获取不到域名的。
3.  其他比较好用的方法：[简单 hook dns 缓存](https://bbs.kanxue.com/thread-265875.htm)

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-174.htm)

最后于 5 小时前 被编程两年半编辑 ，原因：

[#系统内核](forum-41-1-131.htm) [#驱动开发](forum-41-1-132.htm) [#开源分享](forum-41-1-134.htm) [#开发技巧](forum-41-1-135.htm)
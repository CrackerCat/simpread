> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-277402.htm)

> [原创] 聊聊大厂设备指纹其二 & Hunter 环境检测思路详解!

[原创] 聊聊大厂设备指纹其二 & Hunter 环境检测思路详解!

20 小时前 3058

### [原创] 聊聊大厂设备指纹其二 & Hunter 环境检测思路详解!

 [![](http://passport.kanxue.com/upload/avatar/934/819934.png?1541667727)](user-home-819934.htm) [珍惜 Any](user-home-819934.htm) ![](https://bbs.kanxue.com/view/img/rank/11.png) 1  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 20 小时前  3058

目录

*            [前言：](#前言：)
*            [设备指纹：](#设备指纹：)
*                    Android Id
*                            [方法 1：](#方法1：)
*                            [方法 2：](#方法2：)
*                            [方法 3：](#方法3：)
*                            [方法 4：](#方法4：)
*                            [方法 5：](#方法5：)
*                    [硬盘字节总大小：](#硬盘字节总大小：)
*                    [Mac 地址：](#mac地址：)
*                    [附近网卡信息：](#附近网卡信息：)
*                    [IPV6：](#ipv6：)
*                    [系统内核信息：](#系统内核信息：)
*                    [包名随机路径：](#包名随机路径：)
*                    [系统账号：](#系统账号：)
*            [环境检测：](#环境检测：)
*                    Apk 签名:
*                            [Native 层获取签名方法：](#native层获取签名方法：)
*                            [检测：](#检测：)
*                            [对抗：](#对抗：)
*                            [Java 层获取签名方法：](#java层获取签名方法：)
*                            [检测：](#检测：)
*                                    [检测 CREATOR 是否被替换](#检测creator是否被替换)
*                            [对抗：](#对抗：)
*                                    [Hook 服务端解析签名方法](#hook服务端解析签名方法)
*                                    [替换 CREATOR](#替换creator)
*                                    Binder transact 方法
*                    [模拟器检测：](#模拟器检测：)
*                            [Seccomp 检测架构：](#seccomp检测架构：)
*                            [检测温度挂载文件：](#检测温度挂载文件：)
*                            [模拟器特征文件：](#模拟器特征文件：)
*                            [模拟器基础特征：](#模拟器基础特征：)
*                    [检测云手机：](#检测云手机：)
*                            [检测电流 & 电压：](#检测电流&电压：)
*                            [检测摄像头 & 传感器相关：](#检测摄像头&传感器相关：)
*                            [检测传感器个数：](#检测传感器个数：)
*                            [检测传感器名称：](#检测传感器名称：)
*                            [检测挂载文件：](#检测挂载文件：)
*                            检测 ROM 是否 Match:
*                                    [检测环境信息：](#检测环境信息：)
*                                    [检测服务列表：](#检测服务列表：)
*                    检测当前环境是否被 Hook:
*                    [检测沙箱：](#检测沙箱：)
*                            检测多余线程 PID:
*                                    [对抗：](#对抗：)
*                            [执行 ps 命令：](#执行ps命令：)
*                                    [对抗：](#对抗：)
*                            内存 Choose:
*                    [检测 Google&lineageos](#检测google&lineageos)
*                    [检测 IDA 和反调试：](#检测ida和反调试：)
*                            service list
*            [后语：](#后语：)

[](#前言：)前言：
-----------

昨天发现自己有点发烧，今天测一下果然阳了，大早上起来睡不着 。

 

请假以后正好闲着没事，完善一下之前文章 https://bbs.kanxue.com/thread-273759.htm 。

 

之前一直想写来着，但是一直没时间 。之前这篇文章介绍了。设备指纹基础对抗思路和原理 。很多都是基础部分 。这一篇将更详细的介绍一些常见的检测思路和方法，还有一些主流对抗手段 。

 

其中包括很多大厂头疼的一些事情，如何手机恢复出厂设置也能保证设备指纹不发生变化，保证稳定性 。这篇文章里面也都会详细介绍 。

 

如果没看过我之前的这篇文章，所以没看过的同学可以先看看这个文章 。之前有的这篇文章里面就不多二次介绍了 。

 

还有就是一个设备指纹大厂会使用多种方式去获取，那么我们应该如何进行对抗 ，我也会在文章里面说一下我自己的见解和方案，如何在一个 “最佳” 点去解决问题 ，当然如果你有更好的方案，也可以私聊我。

[](#设备指纹：)设备指纹：
---------------

### Android Id

聊到设备指纹最经典的一个字段就是 Android id，就我目前所知，他的获取方式不下 5 种 。分别介绍一下 。

#### [](#方法1：)方法 1：

最基础的 Android id 获取方式 ，这个不多说，直接 Hook 就行 。

```
//原始获取android id
String androidId = Settings.Secure.getString(context.getContentResolver(), Settings.Secure.ANDROID_ID);
CLog.i(String.format("android_id -> 2222 %s", androidId));

```

#### [](#方法2：)方法 2：

第一种获取以后，系统会把 Android id 保存起来，保存到一个 HashMap 里面，防止多次 IPC 初始化 ，所以为了验证第一种方法的准确性，可以二次获取 cache ，

 

和上面的 Android id 进行对比 。9.0 以上需要绕过 Android id 的反射限制 。具体获取方法如下：

```
//通过反射查询android id cache
ArrayMap mValues = null;
try {
    Field sNameValueCache = Settings.Secure.class.getDeclaredField("sNameValueCache");
    sNameValueCache.setAccessible(true);
    Object sLockSettings = sNameValueCache.get(null);
    Field fieldmValues = sLockSettings.getClass().getDeclaredField("mValues");
    fieldmValues.setAccessible(true);
    mValues = (ArrayMap) fieldmValues.get(sLockSettings);
    String android_id = (String)mValues.get("android_id");
    CLog.i(String.format("android_id -> 3333 %s", android_id));
} catch (Throwable e) {
    e.printStackTrace();
} 
```

#### [](#方法3：)方法 3：

方法 3 也是很基础的 Api ，主要通过 ContentResolver 进行间接获取 。很多大厂也都在使用 。

```
try {
    Bundle callResult = context.getContentResolver().call(
            Uri.parse("content://settings/secure"), "GET_secure", "android_id", new Bundle()
    );
    String androidIdValue = callResult.getString("value");
    CLog.i(String.format("android_id -> 1111 %s", androidIdValue));
} catch (Exception e) {
    CLog.e(e.toString(), e);
}

```

#### [](#方法4：)方法 4：

通过 query 命令去查询，获取 Android id ，这种方式底层走的也是 ContentResolver

```
//通过content命令查询android id
String android_id = NativeEngine.popen(
        "content query --uri content://settings/secure --where \"name=\\'android_id\\'\"",
        "");
CLog.i(String.format("android_id -> 4444 %s", android_id));

```

#### [](#方法5：)方法 5：

方法 4 的代码反射实现，我自己测试在高版本总是有问题 ，不是很稳定 ，所以这里面就不发了 。

### [](#硬盘字节总大小：)硬盘字节总大小：

在设备指纹里面，如果想回复出厂设置也能保证原有的设备信息 ，这个字段可以在服务端的相似度算法里面占比很重 ，可以以型号进行分类。，我之前测试过，回复出厂设置指纹也不发生变化的设备指纹核心的设备指纹就几个 。

 

比如硬盘大小，ipv6 ，还有一个就是 MAC 地址，这几个设备指纹也是很核心的设备指纹 。首先先介绍硬盘字节大小。 也是三种获取方法，但是方法底层都是一条系统调用。所以如果要进行对抗的话，只需要在 SVC 层进行处理即可 。获取三种方法如下 ，不建议分别进行处理，可能会导致有地方泄漏，特别是直接开启一条进程通过 execve 去执行，然后管道传过来，导致很容易 Hook 不全。

```
jclass pJclass = env->FindClass("android/os/StatFs");
jmethodID id = env->GetMethodID(pJclass, "", "(Ljava/lang/String;)V");
jobject pJobject =
        env->NewObject(pJclass, id, env->NewStringUTF("/storage/emulated/0"));
 
jlong i = env->CallLongMethod(pJobject, env->GetMethodID(pJclass, "getTotalBytes", "()J"));
LOG(ERROR) << "Java获取getTotalBytes "<
```

我之前也是参考的平头哥，Hook 的 Java 方法，但是发现 Native 层并不能全量拦截 。后来弃用 。平头哥方法代码如下：

```
private void relocateRomMemInfo(VirtualBaseInfo virtualBaseInfo) {
    final String androidSystemOsClassName = "android.system.Os";
    Class androidSystemOsClass =
            RposedHelpers.findClassIfExists(androidSystemOsClassName, ClassLoader.getSystemClassLoader());
    if (androidSystemOsClass == null) {
        CLog.e(">>>>>>>>>>>>> vfs android.system.Os not found");
        return;
    }
    RposedHelpers.findAndHookMethod(androidSystemOsClass, "statvfs", String.class, new RC_MethodHook() {
        @Override
        protected void afterHookedMethod(MethodHookParam param) throws Throwable {
            if(param.getResult() == null) {
                return;
            }
            StructStatVfs structStatVfs = (StructStatVfs) param.getResult();
            if (structStatVfs.f_blocks == 0) {
                return;
            }
            if(param.args[0] instanceof String) {
                long mockRomOut = getMockRomOut(structStatVfs.f_blocks, virtualBaseInfo, structStatVfs.f_blocks);
                CLog.i(TAG, "vfs statvfs path:" + param.args[0] + " f_blocks:" + structStatVfs.f_blocks + " mock value:" + mockRomOut);
                RposedHelpers.setObjectField(
                        structStatVfs,
                        "f_blocks",
                        mockRomOut
                );
            }
        }
    });
}

```

上面这种方式只适合 Java 获取，具体三种获取方法可以看下面 。

 

这三种方法底层走的都是 statfs64 或者 statfs 函数，对抗的话也很简单，直接在 statfs64 或者 statfs 的 after 里面对参数 2 进行替换和复写即可 。

 

对抗如下，具体如何拦截 svc 代码可以参考 proot , 可以参考我之前写的文章 https://bbs.kanxue.com/thread-273160.htm

 

这种方法的好处就是不管如何开进程，都可以在底层统一处理 。

```
        //int statfs(const char *path, struct statfs *buf);
        //int statfs64(const char *path, struct statfs64 *buf);
        case SC_statfs:
        case SC_statfs64: {
            if (isMockFingerptint()) {
                if ((int) syscall_result < 0) {
                    break;
                }
//              f_type：文件系统类型。
//              f_bsize：文件系统块的大小。
//              f_blocks：文件系统中的总块数。
//              f_bfree：文件系统中的可用块数。
//              f_bavail：非超级用户可获取的块数。
//              f_files：文件系统中的总文件节点数。
//              f_ffree：文件系统中的可用文件节点数。
//              f_fsid：文件系统标识。
//              f_namelen：文件名的最大长度。
                char pathBuff[PATH_MAX];
                word_t pPath = peek_reg(tracee, ORIGINAL, SYSARG_1);
                int ret = read_string(tracee, pathBuff, pPath, PATH_MAX);
                if(ret < 0){
                    break;
                }
                if(get_sysnum(tracee, ORIGINAL) == SC_statfs64){
                    struct statfs64 fs = {};
                    word_t arg2 = peek_reg(tracee, ORIGINAL, SYSARG_2);
                    read_data(tracee,&fs,arg2,sizeof (struct statfs64));
                    NativeFingerHandler::StatfsHandler64(pathBuff,&fs);
                    write_data(tracee,arg2,&fs,sizeof (struct statfs64));
                } else{
                    struct statfs fs = {};
                    word_t arg2 = peek_reg(tracee, ORIGINAL, SYSARG_2);
                    read_data(tracee,&fs,arg2,sizeof (struct statfs));
                    NativeFingerHandler::StatfsHandler32(pathBuff,&fs);
                    write_data(tracee,arg2,&fs,sizeof (struct statfs));
                }
            }
            break;
        }

```

### [](#mac地址：)Mac 地址：

这个没啥好说的，基础字段，Java 层获取，netlink 获取，命令行获取 。读文件获取，四种获取方法，和上面类似 ，直接在 svc 的 recvmsg ，recv，recvfrom 的 after 进行数据包替换即可。netlink 获取 mac 方法可以参考我之前的代码 。

 

https://bbs.kanxue.com/thread-271698.htm

 

如果判断是 netlink 的消息，并且是获取网卡类型直接对里面的数据包解析和替换即可 。

```
case SC_recvmsg: {
    //LOGI("start handle SC_recvmsg systexit after")
    if (isMockFingerptint()) {
        NetlinkMacHandler::netlinkHandler_recmsg(tracee);
    }
    break;
}
case SC_recv:
case SC_recvfrom: {
    //LOGE("start handle SC_recvfrom systexit after")
    //recv底层走的recvfrom,所以不需要处理recvfrom
    if (isMockFingerptint()) {
        NetlinkMacHandler::netlinkHandler_recv(tracee);
    }
    break;
}

```

在读文件获取这块因为网卡信息已经在内存里面 ，所以直接 IO 重定向过去即可 。

 

常用的获取网卡信息的文件 ，以 wlan0 为例子 , 场景的获取目录如下：可以 cat 获取，也可以直接读文件 。

```
/sys/class/net/wlan0/address
/sys/devices/virtual/net/wlan0/address
...

```

### [](#附近网卡信息：)附近网卡信息：

这个字段主要是监控群控的一些信息的，主要作用是获取当前 wifi 附近的人 MAC 信息的 。

 

比如大厂一般检测群控的手段就是获取附近的网卡，如果有聚集性就可以认为是群控 。获取的方式也也跟上面一样 。五种获取方法 。

 

获取方法底层也是和 MAC 获取方法一样 ，底层都是 netlink，比如可以直接执行 popen 获取 ，

```
popen("ip neigh show", "r");

```

也可以直接直接读文件 ，路径如下：

```
/proc/net/arp

```

还可以直接 netlink 获取 ，在收到消息以后判断消息类型是 hdr->nlmsg_type == RTM_NEWNEIGH 直接进行替换即可 。

 

直接在 recv 收到消息以后对数据里面的 buff 进行替换即可 。主要核心代码如下 。包括上面的 mac 地址替换 。

```
static void _getifaddrs_callback(void *context, nlmsghdr *hdr) {
    auto **out = reinterpret_cast(context);
 
    //首先先判断消息类型是不是RTM_NEWLINK类型
    if (hdr->nlmsg_type == RTM_NEWLINK) {
        auto *ifi = reinterpret_cast(NLMSG_DATA(hdr));
 
        ifaddrs_storage new_addr(out);
        new_addr.interface_index = ifi->ifi_index;
        new_addr.ifa.ifa_flags = ifi->ifi_flags;
 
        // Get the interface name
        char ifname[IFNAMSIZ];
        if_indextoname(ifi->ifi_index, ifname);
 
        // Go through the various bits of information and find the name.
        rtattr *rta = IFLA_RTA(ifi);
        //获取这个消息的长度
        size_t rta_len = IFLA_PAYLOAD(hdr);
        //这块是判断这个消息是否是合格的消息
        while (RTA_OK(rta, rta_len)) {
            if (rta->rta_type == IFLA_ADDRESS){
                if (RTA_PAYLOAD(rta) < sizeof(new_addr.addr)) {
                    void *data = RTA_DATA(rta);
                    //修改mac地址
                    setMacInData(data, ifname, ZHENXI_RUNTIME_NETLINK_MAC, false);
                    new_addr.SetAddress(AF_PACKET, data, RTA_PAYLOAD(rta));
                    new_addr.SetPacketAttributes(ifi->ifi_index, ifi->ifi_type,
                                                 RTA_PAYLOAD(rta));
                }
            }
            else if (rta->rta_type == IFLA_BROADCAST) {
                if (RTA_PAYLOAD(rta) < sizeof(new_addr.ifa_ifu)) {
                    void *data = RTA_DATA(rta);
                    size_t byteCount = RTA_PAYLOAD(rta);
                    new_addr.SetBroadcastAddress(AF_PACKET, data, byteCount);
                    new_addr.SetPacketAttributes(ifi->ifi_index, ifi->ifi_type,
                                                 RTA_PAYLOAD(rta));
                }
            }
            else if (rta->rta_type == IFLA_IFNAME) {
                if (RTA_PAYLOAD(rta) < sizeof(new_addr.name)) {
                    memcpy(new_addr.name, RTA_DATA(rta), RTA_PAYLOAD(rta));
                    new_addr.ifa.ifa_name = new_addr.name;
                }
            }
            rta = RTA_NEXT(rta, rta_len);
        }
    }
    else if (hdr->nlmsg_type == RTM_NEWADDR) {
        //这个类型在获取网卡的时候未发现调用
        auto *msg = reinterpret_cast(NLMSG_DATA(hdr));
 
 
        // We should already know about this from an RTM_NEWLINK message.
        const auto *addr = reinterpret_cast(*out);
 
        while (addr != nullptr && addr->interface_index != static_cast(msg->ifa_index)) {
            //LOGE("Current interface index: %d", addr->interface_index); // 添加当前接口索引日志
            addr = reinterpret_cast(addr->ifa.ifa_next);
        }
        // If this is an unknown interface,
        // ignore whatever we're being told about it.
        if (addr == nullptr) {
            //LOGE ("_getifaddrs_callback RTM_NEWADDR return")
            return;
        }
        ifaddrs_storage new_addr(out);
        strcpy(new_addr.name, addr->name);
        new_addr.ifa.ifa_name = new_addr.name;
        new_addr.ifa.ifa_flags = addr->ifa.ifa_flags;
        new_addr.interface_index = addr->interface_index;
        // Go through the various bits of information and find the address
        // and any broadcast/destination address.
        rtattr *rta = IFA_RTA(msg);
        size_t rta_len = IFA_PAYLOAD(hdr);
        while (RTA_OK(rta, rta_len)) {
            LOGE("RTA type: %d", rta->rta_type);
            if (rta->rta_type == IFA_ADDRESS) {
                //LOGE ("_getifaddrs_callback RTM_NEWADDR IFA_ADDRESS %d ",msg->ifa_family)
                if (msg->ifa_family == AF_INET || msg->ifa_family == AF_INET6) {
                    void *data = RTA_DATA(rta);
                    // 确保 RTA_DATA(rta) 的大小是正确的
                    if (msg->ifa_family == AF_INET6 && RTA_PAYLOAD(rta) < sizeof(struct in6_addr)) {
                        LOGE("RTA_PAYLOAD size is less than sizeof(struct in6_addr)");
                        return;
                    }
                    struct in6_addr addr_v6_address{};
                    memcpy(&addr_v6_address, RTA_DATA(rta), sizeof(struct in6_addr));
                    char str[INET6_ADDRSTRLEN];
                    inet_ntop(AF_INET6, &addr_v6_address, str, sizeof(str));
                    LOGE("RTM_NEWADDR&IFA_ADDRESS&AF_INET6 111 %s", str)
 
                    size_t byteCount = RTA_PAYLOAD(rta);
                    LOGE ("RTM_NEWADDR&IFA_ADDRESS %zu  %s ",
                          byteCount, getpData(data, byteCount).c_str())
                    new_addr.SetAddress(msg->ifa_family, data, byteCount);
                    new_addr.SetNetmask(msg->ifa_family, msg->ifa_prefixlen);
                }
            }
            else if (rta->rta_type == IFA_BROADCAST) {
                if (msg->ifa_family == AF_INET||msg->ifa_family == AF_INET6) {
                    void *data = RTA_DATA(rta);
                    size_t byteCount = RTA_PAYLOAD(rta);
                    LOGE ("RTM_NEWADDR&IFA_BROADCAST %zu  %s ",
                          byteCount, getpData(data, byteCount).c_str())
                    new_addr.SetBroadcastAddress(msg->ifa_family, data,
                                                 byteCount);
                }
            }
            else if (rta->rta_type == IFA_LOCAL) {
                //LOGE ("_getifaddrs_callback RTM_NEWADDR IFA_LOCAL %d ",msg->ifa_family)
                if (msg->ifa_family == AF_INET || msg->ifa_family == AF_INET6) {
                    void *data = RTA_DATA(rta);
                    struct in6_addr addr_v6_local{};
                    memcpy(&addr_v6_local, RTA_DATA(rta), sizeof(struct in6_addr));
                    char str[INET6_ADDRSTRLEN];
                    inet_ntop(AF_INET6, &addr_v6_local, str, sizeof(str));
                    LOGE("RTM_NEWADDR&IFA_ADDRESS&AF_INET6 222 %s", str)
 
                    size_t byteCount = RTA_PAYLOAD(rta);
                    LOGE ("RTM_NEWADDR&IFA_LOCAL %zu  %s ",
                          byteCount, getpData(data, byteCount).c_str())
                    new_addr.SetLocalAddress(msg->ifa_family, data, byteCount);
                }
            }
            rta = RTA_NEXT(rta, rta_len);
        }
    }
    else if (hdr->nlmsg_type == RTM_NEWNEIGH) {
        // RTM_NEWNEIGH 类型消息为网上邻居(arp表)，需要进行随机化
        auto *ifinfo = reinterpret_cast(NLMSG_DATA(hdr));
        rtattr *rta = NDA_RTA(ifinfo);
        size_t rta_len = NDA_PAYLOAD(hdr);
 
        int if_index = ifinfo->ndm_ifindex;
        char if_name[IFNAMSIZ];
        if_indextoname(if_index, if_name);
        //遍历具体的消息类型
        while (RTA_OK(rta, rta_len)) {
            //a neighbor cache n/w layer destination address
            //邻居缓存nw层目标地址,ip地址区分32和64
            //ip地址,ip可以是v4也可以是v6
            if (rta->rta_type == NDA_DST) {
                if (ifinfo->ndm_family == AF_INET) {
                    //32
                    struct in_addr addr{};
                    memcpy(&addr, RTA_DATA(rta), sizeof(struct in_addr));
                    char *ntoa = inet_ntoa(addr);
                    //LOGE("NDA_DST&AF_INET   %s", inet_ntoa(addr))
                } else if (ifinfo->ndm_family == AF_INET6) {
                    //64
                    struct in6_addr addr{};
                    memcpy(&addr, RTA_DATA(rta), sizeof(struct in6_addr));
                    char str[INET6_ADDRSTRLEN];
                    inet_ntop(AF_INET6, &addr, str, sizeof(str));
                    //LOGE("NDA_DST&AF_INET6  %s", str);
                }
            } else if (rta->rta_type == NDA_LLADDR) {
                //网卡地址
                auto *data = RTA_DATA(rta);
                setMacInData(data, if_name, ZHENXI_RUNTIME_NETLINK_NEIGH, true);
            }
            rta = RTA_NEXT(rta, rta_len);
        }
    }
    else if (hdr->nlmsg_type == RTM_GETADDR) {
        //LOGE("RTM_GETADDR ")
        auto *ifa = reinterpret_cast(NLMSG_DATA(hdr));
 
        // Get the interface name
        char ifname[IFNAMSIZ];
        if_indextoname(ifa->ifa_index, ifname);
 
        // Process the attributes
        rtattr *rta = IFA_RTA(ifa);
        size_t rta_len = IFA_PAYLOAD(hdr);
        while (RTA_OK(rta, rta_len)) {
            if (rta->rta_type == IFA_ADDRESS) {
                if (ifa->ifa_family == AF_INET6) {
                    // Ensure RTA_DATA(rta) size is correct
                    if (RTA_PAYLOAD(rta) < sizeof(struct in6_addr)) {
                        LOGE("RTM_GETADDR RTA_PAYLOAD size is less than sizeof(struct in6_addr)");
                        return;
                    }
                    struct in6_addr addr_v6_address{};
                    memcpy(&addr_v6_address, RTA_DATA(rta), sizeof(struct in6_addr));
                    char str[INET6_ADDRSTRLEN];
                    inet_ntop(AF_INET6, &addr_v6_address, str, sizeof(str));
                    LOGE("RTM_GETADDR RTM_GETADDR&IFA_ADDRESS&AF_INET6 %s", str);
                }
            }
            rta = RTA_NEXT(rta, rta_len);
        }
    }
} 
```

### [](#ipv6：)IPV6：

设个设备指纹也是很核心的设备指纹 ，这个玩意底层获取也是 netlink，但是 netlink 获取，但是这块处理很不好处理 ，我暂时也没进行处理 。

 

常用的获取方式比如，Java 获取 ，命令获取。如果需要进行替换的话 ，只需要处理命令行和 Java 的 Hook 即可 。

 

**命令行可以在对方执行命令之前，将命令换成 cat 命令，去 cat 自己提前 Mock 好的文件，效果是一样的 。**

 

当然，还有另一种思路，其实这个字段可以服务端获取，客户端二次上报，进行匹配 。

```
try {
    NetworkInterface networkInterface;
    InetAddress inetAddress;
    for (Enumeration en = NetworkInterface.getNetworkInterfaces(); en.hasMoreElements(); ) {
        networkInterface = en.nextElement();
        for (Enumeration enumIpAddr = networkInterface.getInetAddresses(); enumIpAddr.hasMoreElements(); ) {
            inetAddress = enumIpAddr.nextElement();
            if (inetAddress instanceof Inet6Address) {
                CLog.e("Java 获取 ipv6 " + inetAddress.getHostAddress());
            }
        }
    }
} catch (Throwable ex) {
    CLog.e("printf ipv6 info error " + ex);
} 
```

命令行获取如下，ip 命令获取如下 。

```
ip -6 addr show

```

打印的内容如下：

```
1: lo: mtu 65536 state UNKNOWN qlen 1000
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
3: dummy0: mtu 1500 state UNKNOWN qlen 1000
    inet6 fe80::b86c:79ff:fe96:4945/64 scope link
       valid_lft forever preferred_lft forever
10: rmnet_data0@rmnet_ipa0: mtu 1500 state UNKNOWN qlen 1000
    inet6 fe80::2ad1:b5a0:792b:9ec4/64 scope link
       valid_lft forever preferred_lft forever
30: wlan0: mtu 1500 state UP qlen 3000
    inet6 fe80::8670:a04c:b8cf:467c/64 scope link stable-privacy
       valid_lft forever preferred_lft forever 
```

### [](#系统内核信息：)系统内核信息：

这玩意底层走的都是 uname 函数 ，直接对 uname 系统调用处理即可 。获取方法比如, 也可以直接 svc 调用 uname 函数 ，也可以直接根据命令行 ，

 

修改的话也很简单，直接在 uname 的 after 里面直接对数据进行替换即可 。

```
uname -a

```

### [](#包名随机路径：)包名随机路径：

**这个是一个非常非常核心的字段，就是 / data/app / 随机 Base64 路径 / base.apk**

 

**这个随机路径就是设备指纹，比如一些大厂会玩，读取你微信的随机路径，获取微信的包信息，然后获取里面的随机路径**

 

**比如微信，快手，京东，淘宝这种随机路径 ，作为核心的唯一设备指纹，只要你不卸载微信，或者其他大厂 apk ，你得设备指纹永远不发生变化，无论你如何修改他自己 Apk 里面的信息，跟他都不产生任何影响 。**

### [](#系统账号：)系统账号：

一般尝试比如小米之类的，登入了指定账号，可以得到一个账号的 id 信息 ，这个也需要处理一下 。最好的办法是不登入账号 。

[](#环境检测：)环境检测：
---------------

检测环境大多数围绕 Hunter 的源码检测思路去复现 ，很多都是 Hunter 的源码 ，很多也都是行业内没有公开的一些检测思路 ，现在市面上检测已经很多没更新了 ，加速行业内卷，我辈刻不容缓 。 ：）

### Apk 签名:

提到环境检测不得不说的就是 Apk 重打包检测 ，现在检测方法千奇百怪，我这边也是一一罗列一下，把一些可能存在的风险点，检测和绕过的原理详细叙述一下 。当然如果你有更好的思路也可以在文章下面留言。

 

**想要绕过签名检测最好的办法或者说成本最低有效的办法就是修改完毕以后不签名配合核心破解直接安装。**

> 核心破解是 lsp 的模块，lsp 商店直接下载，Hook apk 的系统签名解析方法，直接绕过签名检测流程 ，已实现不签名直接安装 。

 

首先先说一下大厂或者一些企业壳的检测点，Java 层基础的获取签名的方法这块就不一一叙述了 。

#### [](#native层获取签名方法：)Native 层获取签名方法：

#### [](#检测：)检测：

这块以 Hunter 源码开始介绍 。

 

核心就三部分 。

*   svc openat 读 apk，去解析签名 。
*   检测打开的 fd，对 fd 的路径进行反查，这块有个细节 buff[len] = '\0'; 就是加这个，如果攻击者没修改 readlinkat 的返回值，就可以检测出来 。
*   检测完毕路径以后对这个文件的权限进行反查，正常 apk 是在系统下的，权限 GID 和 UID 应该是 1000 ，如果攻击者忘记修改权限也可以检测出来 。

```
const char *path = getApkPath(env, context);
//check svc apk sign
const string &string = checkSign(env, path).substr(0, 10);
LOG(INFO) << "apk sign  " << string;
if (string == Base64Utils::VTDecode("TFtCRU58UERAUQ==")) {
    //check sign success,but maybe svc io hook
    //check apk path
    int fd = my_openat(AT_FDCWD, reinterpret_cast(path),
                       O_RDONLY | O_CLOEXEC,
                       0640);
    //check apk path
    char buff[PATH_MAX] = {0};
    std::string fdPath("/proc/");
    fdPath.append(to_string(getpid())).append("/fd/").append(to_string(fd));
    long len = raw_syscall(__NR_readlinkat, AT_FDCWD, fdPath.c_str(), buff, PATH_MAX);
    if (len < 0) {
        return getItemData(env, "APK签名验证失败",
                           "readlinkat error", true,
                           RISK_LEAVE_DEADLY, TAG_REPACKAGE);
    }
    //截断,如果攻击者hook了readlinkat,只修改了参数,没修改返回值也可以检测出来。
    buff[len] = '\0';
    LOG(INFO) << "check apk sign path " << buff;
    if (my_strcmp(path, buff) == 0) {
        LOG(INFO) << "check apk sign path success ";
        //start check memory&location inode
        struct stat statBuff = {0};
        long stat = raw_syscall(__NR_fstat, fd, &statBuff);
        if (stat < 0) {
            LOG(ERROR) << "check apk sign path fail __NR_fstat<0";
            return getItemData(env, "APK签名验证失败",
                               "fstat error", true, RISK_LEAVE_DEADLY, TAG_REPACKAGE);
        }
        //check uid&gid (1000 = system group)
        if (statBuff.st_uid != 1000 && statBuff.st_gid != 1000) {
            LOG(ERROR) << "check apk sign gid&uid fail ";
            return getItemData(env, "APK签名验证失败",
                               nullptr, true, RISK_LEAVE_DEADLY, TAG_REPACKAGE);
        }
        size_t inode = getFileInMapsInode(path);
        if (statBuff.st_ino != inode) {
            LOG(ERROR) << "check apk sign inode fail "<"<>>>>>>>>> check apk sign success! uid-> " << statBuff.st_uid
                   << " gid-> "
                   << statBuff.st_gid;
    } else {
        LOG(ERROR) << "check apk sign path fail ";
        return getItemData(env, "APK签名验证失败",
                           nullptr, true, RISK_LEAVE_DEADLY, TAG_REPACKAGE);
 
    }
    LOG(INFO) << "check apk sign success";
 
    return nullptr;
} 
```

#### [](#对抗：)对抗：

针对上面的检测对抗也很简单，对 svc 的 openat 拦截了以后，对 readlinkat 和 stat 函数进行处理即可 。很轻松即可绕过检测 。很多加壳基本都是检测 ROOT 检测 LSP 调用栈之类的 ，并不只是单一的去检测签名一个纬度 。比如发现了开启了 seccomp 就会闪退，发现 Root 就会闪退 。

#### [](#java层获取签名方法：)Java 层获取签名方法：

#### [](#检测：)检测：

##### 检测 CREATOR 是否被替换

这里先说一下 Hunter 的 Java 层检测签名的方法，这块相当于反射 CREATOR 变量，这个变量是保存一些 IPC 通讯的东西 。

 

很多攻击者会用 Lspatch 进行打包 ，对变量进行替换，这时候我们去检测这个变量的 Classloader 是不是系统 ClassLoader 。

 

防止被替换 。

```
try {
    Field creatorField = PackageInfo.class.getField("CREATOR");
    creatorField.setAccessible(true);
    Object creator = creatorField.get(null);
    if (creator != null) {
        ClassLoader creatorClassloader = creator.getClass().getClassLoader();
        ClassLoader sysClassloader = ClassLoader.getSystemClassLoader();
        if (creatorClassloader == null || sysClassloader == null) {
            return null;
        }
        //系统的是bootclassloader
        //用户创建的都是pathclassloader
        //如果相等则认为系统的被替换
        if (sysClassloader.getClass().getName().
                equals(creatorClassloader.getClass().getName())) {
            return new ListItemBean("Apk签名验证失败！",
                    ListItemBean.RiskLeave.Deadly,
                    "Apk签名方法被替换!\n"
                            + creatorClassloader.getClass().getName() + "\n"
                            + sysClassloader.getClass().getName());
        }
        return null;
    }
} catch (Throwable e) {
    CLog.e("checkApkPackageInfoCreator error " + e);
}

```

#### [](#对抗：)对抗：

对抗的方法主要下面三种：

 

现在大厂一般会直接通过 IPC 直接和 PMS 进行通讯 ，不过这种思路也很好过 ，我这边也是参考的 Lspatch 。

 

这块有两个很核心的思路 ，就是拦截 Binder IPC 通讯的方法，和 Hook 服务端的签名解析方法 。代码如下：

##### Hook 服务端解析签名方法

在系统进行签名解析的时候进行签名替换替换 。

```
private static void hookPackageParser(Signature[] fakeSignature) {
    try {
        RposedBridge.hookAllMethods(
                RposedHelpers.findClass("android.content.pm.PackageParser", ClassLoader.getSystemClassLoader()),
                "generatePackageInfo", new RC_MethodHook() {
                    @Override
                    protected void afterHookedMethod(MethodHookParam param) {
                        PackageInfo packageInfo = (PackageInfo) param.getResult();
                        if (packageInfo == null) return;
                        if (packageInfo.packageName.equals(RuntimeToolKit.packageName)) {
                            if (packageInfo.signatures != null && packageInfo.signatures.length > 0) {
                                CLog.i("PackageParser signature info (method 1)");
                                packageInfo.signatures[0] = fakeSignature[0];
                            }
                            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
                                if (packageInfo.signingInfo != null) {
                                    CLog.i("PackageParser signature info (method 2)");
                                    Signature[] signaturesArray = packageInfo.signingInfo.getApkContentsSigners();
                                    if (signaturesArray != null && signaturesArray.length > 0) {
                                        signaturesArray[0] = fakeSignature[0];
                                    }
                                }
                            }
                        }
                    }
                });
    } catch (Throwable e) {
        CLog.e("hook apkSign PackageParser -> generatePackageInfo " + e.getMessage());
    }
}

```

##### 替换 CREATOR

这个思路也是抄的 lspatch ，这种思路可以通过检测 CREATOR 变量 Classloader 的方式检测出来 。

```
public static void byPassSignatureForCREATOR(Signature[] fakeSignature) {
    if (fakeSignature == null) {
        return;
    }
    Parcelable.Creator originalCreator = PackageInfo.CREATOR;
    Parcelable.Creator proxiedCreator = new Parcelable.Creator() {
        @Override
        public PackageInfo createFromParcel(Parcel source) {
            PackageInfo packageInfo = originalCreator.createFromParcel(source);
            if (packageInfo.packageName.equals(RuntimeToolKit.packageName)) {
                if (packageInfo.signatures != null && packageInfo.signatures.length > 0) {
                    //CLog.i("CREATOR Replace signature info (method 1)");
                    packageInfo.signatures[0] = fakeSignature[0];
                }
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
                    if (packageInfo.signingInfo != null) {
                        //CLog.i("CREATOR Replace signature info (method 2)");
                        Signature[] signaturesArray = packageInfo.signingInfo.getApkContentsSigners();
                        if (signaturesArray != null && signaturesArray.length > 0) {
                            signaturesArray[0] = fakeSignature[0];
                        }
                    }
                }
            }
            return packageInfo;
        }
 
        @Override
        public PackageInfo[] newArray(int size) {
            return originalCreator.newArray(size);
        }
    };
    RposedHelpers.setStaticObjectField(PackageInfo.class, "CREATOR", proxiedCreator);
    try {
        Map) RposedHelpers.getStaticObjectField(Parcel.class, "mCreators");
        mCreators.clear();
    } catch (NoSuchFieldError ignore) {
    } catch (Throwable e) {
        CLog.e("fail to clear Parcel.mCreators", e);
    }
    try {
        Map)
                RposedHelpers.getStaticObjectField(Parcel.class,
                        "sPairedCreators");
        sPairedCreators.clear();
    } catch (NoSuchFieldError ignore) {
    } catch (Throwable e) {
        CLog.e("fail to clear Parcel.sPairedCreators", e);
    }
} 
```

##### Binder transact 方法

有很多大厂会自己伪造 IPC 和服务端进行通讯，用这种方法可以进行签名的修改和替换，实现思路主要就是 Hook binder 通讯解析的方法 。判断如果传输类型数据是获取签名的话就进行替换 。主要思路就是 Hook 传输数据的 “水管”，对水管里面的内容进行替换 。

 

不知道为什么 lspatch 新版本把这段代码删除了 ，只采用了替换 CREATOR 和 Hook 服务端解析签名的方法 。

 

代码如下：

```
RposedHelpers.findAndHookMethod("android.os.BinderProxy",
        context.getClassLoader(),
        "transact",
        int.class,
        Parcel.class,
        Parcel.class,
        int.class,
        new RC_MethodHook() {
            @Override
            protected void afterHookedMethod(MethodHookParam param) {
                try {
                    Object object = param.thisObject;
                    int id = (int) param.args[0];
                    Parcel write = (Parcel) param.args[1];
                    Parcel out = (Parcel) param.args[2];
                    // forward check
                    if (write == null || out == null) {
                        return;
                    }
                    // prevent recurise call (防止递归)
                    if (id == IBinder.INTERFACE_TRANSACTION) {
                        //IBinder协议事务代码：询问事务的收件人端以获取其规范接口描述符。
                        return;
                    }
 
                    String desc = (String) RposedHelpers.callMethod(object, "getInterfaceDescriptor");
                    if (desc == null || !desc.equals("android.content.pm.IPackageManager")) {
                        return;
                    }
                    if (id == TRANSACTION_getPackageInfo_ID) {
                        out.readException();
                        if (0 != out.readInt()) {
                            PackageInfo packageInfo = PackageInfo.CREATOR.createFromParcel(out);
                            if (packageInfo.packageName.equals(context.getApplicationInfo().packageName)) {
                                if (fakeSignature[0] == null) {
                                    CLog.e(">>>>>>>>>> byPassSignature fakeSignature == null");
                                    System.exit(0);
                                    return;
                                }
                                //CLog.i( "org data size ->  "+out.dataSize());
 
                                if (packageInfo.signatures != null && packageInfo.signatures.length > 0) {
                                    packageInfo.signatures[0] = fakeSignature[0];
                                    //CLog.i(" byPassSignatureByLSPosed 1 !!");
                                }
                                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
                                    if (packageInfo.signingInfo != null) {
                                        Signature[] signaturesArray = packageInfo.signingInfo.getApkContentsSigners();
                                        if (signaturesArray != null && signaturesArray.length > 0) {
                                            signaturesArray[0] = fakeSignature[0];
                                            //CLog.i( " byPassSignatureByLSPosed 2 !!");
                                        }
                                    }
                                }
 
                                out.setDataPosition(0);
                                out.setDataSize(0);
                                out.writeNoException();
                                out.writeInt(1);
                                packageInfo.writeToParcel(out, PARCELABLE_WRITE_RETURN_VALUE);
                            }
                        }
 
                        // reset pos
                        out.setDataPosition(0);
                    }
                }
                catch (Throwable err) {
                    CLog.e(">>>>>>>>>> byPassSignatureByLSPosed error " + err.getMessage());
                }
            }
        });

```

还有很多很好思路的代码 ，比如 ISO 线程去读取 apk 签名信息，防止 SVC 被 IO 重定向掉 。

 

当然检测签名的方法肯定不止这些，如果有更好的，我没写的，也可以在文章下面留言 。

### [](#模拟器检测：)模拟器检测：

Java 层基础的获取 api 架构啥的，这块就不一一叙述了 。

#### [](#seccomp检测架构：)Seccomp 检测架构：

主要思路来自 B 哥 ，感谢 B 个 ，先安装 seccomp 架子 。

```
void install_check_arch_seccomp() {
    struct sock_filter filter[15] = {
            BPF_STMT(BPF_LD + BPF_W + BPF_ABS, (uint32_t) offsetof(struct seccomp_data, nr)),
            BPF_JUMP(BPF_JMP + BPF_JEQ, __NR_getpid, 0, 12),
            BPF_STMT(BPF_LD + BPF_W + BPF_ABS, (uint32_t) offsetof(struct seccomp_data, args[0])),
            BPF_JUMP(BPF_JMP + BPF_JEQ, DetectX86Flag, 0, 10),
            BPF_STMT(BPF_LD + BPF_W + BPF_ABS, (uint32_t) offsetof(struct seccomp_data, arch)),
            BPF_JUMP(BPF_JMP + BPF_JEQ, AUDIT_ARCH_X86_64, 0, 1),
            BPF_STMT(BPF_RET + BPF_K, SECCOMP_RET_ERRNO | (864 & SECCOMP_RET_DATA)),
            BPF_JUMP(BPF_JMP + BPF_JEQ, AUDIT_ARCH_I386, 0, 1),
            BPF_STMT(BPF_RET + BPF_K, SECCOMP_RET_ERRNO | (386 & SECCOMP_RET_DATA)),
            BPF_JUMP(BPF_JMP + BPF_JEQ, AUDIT_ARCH_ARM, 0, 1),
            BPF_STMT(BPF_RET + BPF_K, SECCOMP_RET_ERRNO | (0xA32 & SECCOMP_RET_DATA)),
            BPF_JUMP(BPF_JMP + BPF_JEQ, AUDIT_ARCH_AARCH64, 0, 1),
            BPF_STMT(BPF_RET + BPF_K, SECCOMP_RET_ERRNO | (0xA64 & SECCOMP_RET_DATA)),
            BPF_STMT(BPF_RET + BPF_K, SECCOMP_RET_ERRNO | (6 & SECCOMP_RET_DATA)),
            BPF_STMT(BPF_RET + BPF_K, SECCOMP_RET_ALLOW)
 
    };
    struct sock_fprog program = {
            .len = (unsigned short) (sizeof(filter) / sizeof(filter[0])),
            .filter = filter
    };
    errno = 0;
    if (prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0)) {
        LOG(ERROR) << "prctl(PR_SET_NO_NEW_PRIVS) " << strerror(errno);
    }
    errno = 0;
    if (prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &program)) {
        LOG(ERROR) << "prctl(PR_SET_SECCOMP) " << strerror(errno);
    }
}

```

配合上面的代码 。启动调用 getpid， 上面的架子会对 getpid 函数进行拦截 ，然后架构进行判断 。

```
string check_arch_by_seccomp() {
    if (get_sdk_level() < __ANDROID_API_N_MR1__){
        return "";
    }
    errno = 0;
    syscall(__NR_getpid, DetectX86Flag);
    if (errno == 386) {
        return "I386设备";
    } else if (errno == 864) {
        return "X86_64设备";
    } else if (errno == 0xA32 || errno == 0xA64) {
        return "";
    }else if (errno == 0) {
        //可能是没有开启seccomp
        return "";
    }
    return ("疑似X86模拟器设备"+ to_string(errno));
}

```

#### [](#检测温度挂载文件：)检测温度挂载文件：

```
int thermal_check() {
    DIR *dir_ptr;
    int count = 0;
    struct dirent *entry;
    if ((dir_ptr = opendir("/sys/class/thermal/")) != nullptr) {
        while ((entry = readdir(dir_ptr))) {
            if (!strcmp(entry->d_name, ".") || !strcmp(entry->d_name, "..")) {
                continue;
            }
            char *tmp = entry->d_name;
            if (strstr(tmp, "thermal_zone") != nullptr) {
                count++;
            }
        }
        closedir(dir_ptr);
    } else {
        count = -1;
    }
    return count;
}

```

#### [](#模拟器特征文件：)模拟器特征文件：

```
string simulator_files_check() {
    if (file_exist("/system/bin/androVM-prop")) {//检测androidVM
        return "/system/bin/androVM-prop";
    } else if (file_exist("/system/bin/microvirt-prop")) {//检测逍遥模拟器--新版本找不到特征
        return "/system/bin/microvirt-prop";
    } else if (file_exist("/system/lib/libdroid4x.so")) {//检测海马模拟器
        return "/system/lib/libdroid4x.so";
    } else if (file_exist("/system/bin/windroyed")) {//检测文卓爷模拟器
        return "/system/bin/windroyed";
    } else if (file_exist("/system/bin/nox-prop")) {//检测夜神模拟器--某些版本找不到特征
        return "/system/bin/nox-prop";
    } else if (file_exist("system/lib/libnoxspeedup.so")) {//检测夜神模拟器
        return "system/lib/libnoxspeedup.so";
    } else if (file_exist("/system/bin/ttVM-prop")) {//检测天天模拟器
        return "/system/bin/ttVM-prop";
    } else if (file_exist("/data/.bluestacks.prop")) {//检测bluestacks模拟器  51模拟器
        return "/data/.bluestacks.prop";
    } else if (file_exist("/system/bin/duosconfig")) {//检测AMIDuOS模拟器
        return "/system/bin/duosconfig";
    } else if (file_exist("/system/etc/xxzs_prop.sh")) {//检测星星模拟器
        return "/system/etc/xxzs_prop.sh";
    } else if (file_exist("/system/etc/mumu-configs/device-prop-configs/mumu.config")) {//网易MuMu模拟器
        return "/system/etc/mumu-configs/device-prop-configs/mumu.config";
    } else if (file_exist("/system/priv-app/ldAppStore")) {//雷电模拟器
        return "/system/priv-app/ldAppStore";
    } else if (file_exist("system/bin/ldinit") && file_exist("system/bin/ldmountsf")) {//雷电模拟器
        return "system/bin/ldinit";
    } else if (file_exist("/system/app/AntStore") && file_exist("/system/app/AntLauncher")) {//小蚁模拟器
        return "/system/app/AntStore";
    } else if (file_exist("vmos.prop")) {//vmos虚拟机
        return "vmos.prop";
    } else if (file_exist("fstab.titan") && file_exist("init.titan.rc")) {//光速虚拟机
        return "fstab.titan";
    } else if (file_exist("x8.prop")) {//x8沙箱和51虚拟机
        return "x8.prop";
    } else if (file_exist("/system/lib/libc_malloc_debug_qemu.so")) {//AVD QEMU
        return "/system/lib/libc_malloc_debug_qemu.so";
    }
    LOGD("simulator file check info not find  ");
    return "";
}

```

#### [](#模拟器基础特征：)模拟器基础特征：

这块思路主要来自非虫 ，在次感谢 。

```
      public static ListItemBean checkEmulator(Context context) {
        ArrayList choose = new ArrayList<>();
//        try {
//            String[] strArr = {
//                    "/boot/bstmods/vboxguest.ko",
//                    "/boot/bstmods/vboxsf.ko",
//                    "/dev/mtp_usb",
//                    "/dev/qemu_pipe",
//                    "/dev/socket/baseband_genyd",
//                    "/dev/socket/genyd",
//                    "/dev/socket/qemud",
//                    "/dev/socket/windroyed-audio",
//                    "/dev/socket/windroyed-camera",
//                    "/dev/socket/windroyed-gps",
//                    "/dev/socket/windroyed-sensors",
//                    "/dev/vboxguest",
//                    "/dev/vboxpci",
//                    "/dev/vboxuser",
//                    "/fstab.goldfish",
//                    "/fstab.nox",
//                    "/fstab.ranchu-encrypt",
//                    "/fstab.ranchu-noencrypt",
//                    "/fstab.ttVM_x86",
//                    "/fstab.vbox86",
//                    "/init.goldfish.rc",
//                    "/init.magisk.rc",
//                    "/init.nox.rc",
//                    "/init.ranchu-encrypt.rc",
//                    "/init.ranchu-noencrypt.rc",
//                    "/init.ranchu.rc",
//                    "/init.ttVM_x86.rc",
//                    "/init.vbox86.rc",
//                    "/init.vbox86p.rc",
//                    "/init.windroye.rc",
//                    "/init.windroye.sh",
//                    "/init.x86.rc",
//                    "/proc/irq/20/vboxguest",
//                    "/sdcard/Android/data/com.redfinger.gamemanage",
//                    "/stab.andy",
//                    "/sys/bus/pci/drivers/vboxguest",
//                    "/sys/bus/pci/drivers/vboxpci",
//                    "/sys/bus/platform/drivers/qemu_pipe",
//                    "/sys/bus/platform/drivers/qemu_pipe/qemu_pipe",
//                    "/sys/bus/platform/drivers/qemu_trace",
//                    "/sys/bus/virtio/drivers/itolsvmlgtp",
//                    "/sys/bus/virtio/drivers/itoolsvmhft",
//                    "/sys/class/bdi/vboxsf-1",
//                    "/sys/class/bdi/vboxsf-2",
//                    "/sys/class/bdi/vboxsf-3",
//                    "/sys/class/misc/qemu_pipe",
//                    "/sys/class/misc/vboxguest",
//                    "/sys/class/misc/vboxuser",
//                    "/sys/devices/platform/qemu_pipe",
//                    "/sys/devices/virtual/bdi/vboxsf-1",
//                    "/sys/devices/virtual/bdi/vboxsf-2",
//                    "/sys/devices/virtual/bdi/vboxsf-3",
//                    "/sys/devices/virtual/misc/qemu_pipe",
//                    "/sys/devices/virtual/misc/vboxguest",
//                    "/sys/devices/virtual/misc/vboxpci",
//                    "/sys/devices/virtual/misc/vboxuser",
//                    "/sys/fs/selinux/booleans/in_qemu",
//                    "/sys/kernel/debug/bdi/vboxsf-1",
//                    "/sys/kernel/debug/bdi/vboxsf-2",
//                    "/sys/kernel/debug/x86",
//                    "/sys/module/qemu_trace_sysfs",
//                    "/sys/module/vboxguest",
//                    "/sys/module/vboxguest/drivers/pci:vboxguest",
//                    "/sys/module/vboxpcism",
//                    "/sys/module/vboxsf",
//                    "/sys/module/vboxvideo",
//                    "/sys/module/virtio_pt/drivers/virtio:itoolsvmhft",
//                    "/sys/module/virtio_pt_ie/drivers/virtio:itoolsvmlgtp",
//                    "/sys/qemu_trace",
//                    "/system/app/GenymotionLayout",
//                    "/system/bin/OpenglService",
//                    "/system/bin/androVM-vbox-sf",
//                    "/system/bin/droid4x",
//                    "/system/bin/droid4x-prop",
//                    "/system/bin/droid4x-vbox-sf",
//                    "/system/bin/droid4x_setprop",
//                    "/system/bin/enable_nox",
//                    "/system/bin/genymotion-vbox-sf",
//                    "/system/bin/microvirt-prop",
//                    "/system/bin/microvirt-vbox-sf",
//                    "/system/bin/microvirt_setprop",
//                    "/system/bin/microvirtd",
//                    "/system/bin/mount.vboxsf",
//                    "/system/bin/nox",
//                    "/system/bin/nox-prop",
//                    "/system/bin/nox-vbox-sf",
//                    "/system/bin/nox_setprop",
//                    "/system/bin/noxd",
//                    "/system/bin/noxscreen",
//                    "/system/bin/noxspeedup",
//                    "/system/bin/qemu-props",
//                    "/system/bin/qemud",
//                    "/system/bin/shellnox",
//                    "/system/bin/ttVM-prop",
//                    "/system/bin/windroyed",
//                    "/system/droid4x",
//                    "/system/etc/init.droid4x.sh",
//                    "/system/etc/init.tiantian.sh",
//                    "/system/lib/egl/libEGL_emulation.so",
//                    "/system/lib/egl/libEGL_tiantianVM.so",
//                    "/system/lib/egl/libEGL_windroye.so",
//                    "/system/lib/egl/libGLESv1_CM_emulation.so",
//                    "/system/lib/egl/libGLESv1_CM_tiantianVM.so",
//                    "/system/lib/egl/libGLESv1_CM_windroye.so",
//                    "/system/lib/egl/libGLESv2_emulation.so",
//                    "/system/lib/egl/libGLESv2_tiantianVM.so",
//                    "/system/lib/egl/libGLESv2_windroye.so",
//                    "/system/lib/hw/audio.primary.vbox86.so",
//                    "/system/lib/hw/audio.primary.windroye.so",
//                    "/system/lib/hw/audio.primary.x86.so",
//                    "/system/lib/hw/autio.primary.nox.so",
//                    "/system/lib/hw/camera.vbox86.so",
//                    "/system/lib/hw/camera.windroye.jpeg.so",
//                    "/system/lib/hw/camera.windroye.so",
//                    "/system/lib/hw/camera.x86.so",
//                    "/system/lib/hw/gps.nox.so",
//                    "/system/lib/hw/gps.vbox86.so",
//                    "/system/lib/hw/gps.windroye.so",
//                    "/system/lib/hw/gralloc.nox.so",
//                    "/system/lib/hw/gralloc.vbox86.so",
//                    "/system/lib/hw/gralloc.windroye.so",
//                    "/system/lib/hw/sensors.nox.so",
//                    "/system/lib/hw/sensors.vbox86.so",
//                    "/system/lib/hw/sensors.windroye.so",
//                    "/system/lib/init.nox.sh",
//                    "/system/lib/libGM_OpenglSystemCommon.so",
//                    "/system/lib/libc_malloc_debug_qemu.so",
//                    "/system/lib/libclcore_x86.bc",
//                    "/system/lib/libdroid4x.so",
//                    "/system/lib/libnoxd.so",
//                    "/system/lib/libnoxspeedup.so",
//                    "/system/lib/modules/3.10.30-android-x86.hd+",
//                    "/system/lib/vboxguest.ko",
//                    "/system/lib/vboxpcism.ko",
//                    "/system/lib/vboxsf.ko",
//                    "/system/lib/vboxvideo.ko",
//                    "/system/lib64/egl/libEGL_emulation.so",
//                    "/system/lib64/egl/libGLESv1_CM_emulation.so",
//                    "/system/lib64/egl/libGLESv2_emulation.so",
//                    "/vendor/lib64/egl/libEGL_emulation.so",
//                    "/vendor/lib64/egl/libGLESv1_CM_emulation.so",
//                    "/vendor/lib64/egl/libGLESv2_emulation.so",
//                    "/vendor/lib64/libandroidemu.so",
//                    "/system/lib64/hw/gralloc.ranchu.so",
//                    "/system/lib64/libc_malloc_debug_qemu.so",
//                    "/system/usr/Keylayout/droid4x_Virtual_Input.kl",
//                    "/system/usr/idc/Genymotion_Virtual_Input.idc",
//                    "/system/usr/idc/droid4x_Virtual_Input.idc",
//                    "/system/usr/idc/nox_Virtual_Input.idc",
//                    "/system/usr/idc/windroye.idc",
//                    "/system/usr/keychars/nox_gpio.kcm",
//                    "/system/usr/keychars/windroye.kcm",
//                    "/system/usr/keylayout/Genymotion_Virtual_Input.kl",
//                    "/system/usr/keylayout/nox_Virtual_Input.kl",
//                    "/system/usr/keylayout/nox_gpio.kl",
//                    "/system/usr/keylayout/windroye.kl",
//                    "system/etc/init/ndk_translation_arm64.rc",
//                    "/system/xbin/noxsu",
//                    "/ueventd.android_x86.rc",
//                    "/ueventd.andy.rc",
//                    "/ueventd.goldfish.rc",
//                    "/ueventd.nox.rc",
//                    "/ueventd.ranchu.rc",
//                    "/ueventd.ttVM_x86.rc",
//                    "/ueventd.vbox86.rc",
//                    "/vendor/lib64/libgoldfish-ril.so",
//                    "/vendor/lib64/libgoldfish_codecs_common.so",
//                    "/vendor/lib64/libstagefright_goldfish_avcdec.so",
//                    "/vendor/lib64/libstagefright_goldfish_vpxdec.so",
//                    "/x86.prop"
//            };
//            for (int i = 0; i < 7; i++) {
//                String f = strArr[i];
//                if (new File(f).exists())
//                    choose.add(f);
//            }
//        } catch (Exception e) {
//            e.printStackTrace();
//        }
 
        try {
            String[] myArr = {
                    "generic",
                    "vbox"
            };
            for (String str : myArr) {
                if (Build.FINGERPRINT.contains(str))
                    choose.add(Build.FINGERPRINT);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
 
        try {
            String[] myArr = {
                    "google_sdk",
                    "emulator",
                    "android sdk built for",
                    "droid4x"
            };
            for (String str : myArr) {
                if (Build.MODEL.contains(str))
                    choose.add(Build.MODEL);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
 
        try {
            String[] myArr = {
                    "Genymotion"
            };
            for (String str : myArr) {
                if (Build.MANUFACTURER.contains(str))
                    choose.add(Build.MANUFACTURER);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
 
        try {
            String[] myArr = {
                    "google_sdk", "sdk_phone", "sdk_x86", "vbox86p", "nox"
            };
            for (String str : myArr) {
                if (Build.PRODUCT.toLowerCase(Locale.ROOT).contains(str))
                    choose.add(Build.PRODUCT);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
 
        try {
            String[] myArr = {
                    "nox"
            };
            for (String str : myArr) {
                if (Build.BOARD.toLowerCase(Locale.ROOT).contains(str))
                    choose.add(Build.BOARD);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
 
        try {
            String[] myArr = {
                    "nox"
            };
            for (String str : myArr) {
                if (Build.BOOTLOADER.toLowerCase(Locale.ROOT).contains(str))
                    choose.add(Build.BOOTLOADER);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
 
        try {
            String[] myArr = {
                    "ranchu", "vbox86", "goldfish"
            };
            for (String str : myArr) {
                if (Build.HARDWARE.equalsIgnoreCase(str))
                    choose.add(Build.HARDWARE);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
 
        try {
            Enumeration networkInterfaces = NetworkInterface.getNetworkInterfaces();
            while (networkInterfaces.hasMoreElements()) {
                NetworkInterface ele = networkInterfaces.nextElement();
                if (ele != null) {
                    Enumeration inetAddresses = ele.getInetAddresses();
                    while (inetAddresses.hasMoreElements()) {
                        InetAddress nextElement = inetAddresses.nextElement();
                        if (!nextElement.isLoopbackAddress() &&
                                (nextElement instanceof Inet4Address)) {
                            String ip = nextElement.getHostAddress();
                            if (ip == null) continue;
                            if (ip.equalsIgnoreCase("10.0.2.15") ||
                                    ip.equalsIgnoreCase("10.0.2.16")) {
                                choose.add(ip);
                            }
                        }
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
 
//        try {
//            String[] qemuProps = {
//                    "ro.kernel.qemu.avd_name",
//                    "ro.kernel.qemu.gles",
//                    "ro.kernel.qemu.gltransport",
//                    "ro.kernel.qemu.opengles.version",
//                    "ro.kernel.qemu.uirenderer",
//                    "ro.kernel.qemu.vsync",
//                    "ro.qemu.initrc",
//                    "init.svc.qemu-props",
//                    "qemu.adb.secure",
//                    "qemu.cmdline",
//                    "qemu.hw.mainkeys",
//                    "qemu.logcat",
//                    "ro.adb.qemud",
//                    "qemu.sf.fake_camera",
//                    "qemu.sf.lcd_density",
//                    "qemu.timezone",
//                    "init.svc.goldfish-logcat",
//                    "ro.boottime.goldfish-logcat",
//                    "ro.hardware.audio.primary",
//                    "init.svc.ranchu-net",
//                    "init.svc.ranchu-setup",
//                    "ro.boottime.ranchu-net",
//                    "ro.boottime.ranchu-setup",
//                    "init.svc.droid4x",
//                    "init.svc.noxd",
//                    "init.svc.qemud",
//                    "init.svc.goldfish-setup",
//                    "init.svc.goldfish-logcat",
//                    "init.svc.ttVM_x86-setup",
//                    "vmos.browser.home",
//                    "vmos.camera.enable",
//                    "ro.trd_yehuo_searchbox",
//                    "init.svc.microvirtd",
//                    "init.svc.vbox86-setup",
//                    "ro.ndk_translation.version",
//                    "redroid.width",
//                    "redroid.height",
//                    "redroid.fps",
//                    "ro.rf.vmname"
//            };
//
//            for (String str : qemuProps) {
//                String val = SystemPropertiesUtils.getProperty(str, null);
//                if (val != null) {
//                    choose.add(str);
//                }
//            }
//        } catch (Throwable e) {
//            e.printStackTrace();
//        }
        //判断是否存在指定硬件
        PackageManager pm = null;
        try {
            pm = context.getPackageManager();
            String[] features = {
                    //PackageManager.FEATURE_RAM_NORMAL,//这个存在问题,自己组装的手机可能导致这个痕迹找不到
                    PackageManager.FEATURE_BLUETOOTH,
                    PackageManager.FEATURE_CAMERA_FLASH,
                    PackageManager.FEATURE_TELEPHONY
            };
            for (String feature : features) {
                if (!pm.hasSystemFeature(feature)) {
                    choose.add(feature);
                }
            }
        } catch (Throwable ignored) {
    }
 
    try {
        String[] emuPkgs = {
                "com.google.android.launcher.layouts.genymotion",
                "com.bluestacks",
                "com.bignox.app"
        };
 
        for (String pkg : emuPkgs) {
            try {
                if (pm != null) {
                    pm.getPackageInfo(pkg, 0);
                }
                choose.add(pkg);
            } catch (Throwable e) {
                //e.printStackTrace();
            }
        }
    } catch (Throwable ignored) {
 
    }
 
    try {
        SensorManager sensor = (SensorManager) context.getSystemService(Context.SENSOR_SERVICE);
        int sensorSize = sensor.getSensorList(Sensor.TYPE_ALL).size();
        for (int i = 0; i < sensorSize; i++) {
            Sensor s = sensor.getDefaultSensor(i);
            if (s != null && s.getName().contains("Goldfish")) {
                choose.add(s.getName());
            }
        }
    } catch (Throwable ignored) {
 
    }
 
    try {
        if (checkSelfPermission(context, "android.permission.READ_SMS") == 0 ||
                checkSelfPermission(context, "android.permission.READ_PHONE_NUMBERS") == 0 ||
                    checkSelfPermission(context, "android.permission.READ_PHONE_STATE") == 0) {
            TelephonyManager telephonyManager = (TelephonyManager) context.getSystemService(Context.TELEPHONY_SERVICE);
            String phoneNumber = telephonyManager.getLine1Number();
 
            String[] phoneNumbers = {
                    "15555215554",
                    "15555215556",
                    "15555215558",
                    "15555215560",
                    "15555215562",
                    "15555215564",
                    "15555215566",
                    "15555215568",
                    "15555215570",
                    "15555215572",
                    "15555215574",
                    "15555215576",
                    "15555215578",
                    "15555215580",
                    "15555215582",
                    "15555215584"
            };
            if(phoneNumber!=null) {
                for (String phone : phoneNumbers) {
                    if (phoneNumber.equalsIgnoreCase(phone)) {
                        choose.add(phone);
                        break;
                    }
                }
            }
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
 
    if (choose.size() > 0) {
        ListItemBean item = new ListItemBean("检测到APK运行在虚拟机&模拟器中",
                ListItemBean.RiskLeave.Deadly,
                choose.toString()
        );
        for (String str : choose) {
            item.putData(str);
        }
        return item;
    }
    return null;
} 
```

### [](#检测云手机：)检测云手机：

这块思路还是很多的，不同的云手机检测的思路也不一样 。大部分云手机做的还是很好的，很多都可以过掉 Hunter 的检测 。

#### 检测电流 & 电压：

```
private final BroadcastReceiver batteryInfoReceiver = new BroadcastReceiver() {
    @Override
    public void onReceive(Context context, Intent intent) {
 
        // 电池状态
        int plugged = intent.getIntExtra(BatteryManager.EXTRA_PLUGGED, -1);
 
        // 电压（以毫伏为单位）
        int voltage = intent.getIntExtra(BatteryManager.EXTRA_VOLTAGE, -1);
 
        // 获取电池电流（毫安）
        int currentNow = -1;
        if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.LOLLIPOP) {
            BatteryManager batteryManager = (BatteryManager) context.getSystemService(Context.BATTERY_SERVICE);
            currentNow = batteryManager.getIntProperty(BatteryManager.BATTERY_PROPERTY_CURRENT_NOW);
        }
 
        // 判断是否在充电
        if (plugged == BatteryManager.BATTERY_PLUGGED_AC || plugged == BatteryManager.BATTERY_PLUGGED_USB || plugged == BatteryManager.BATTERY_PLUGGED_WIRELESS) {
            // 在充电
            if (voltage != -1 && currentNow != -1) {
                float voltageInVolts = voltage / 1000f; // 将电压转换为伏特
                float currentInAmperes = currentNow / 1000000f; // 将电流转换为安培
                float chargingPower = voltageInVolts * currentInAmperes; // 计算充电功率（瓦特）
                CLog.i(String.format("充电功率: %.2fW", chargingPower));
                if (Math.abs(chargingPower) > 300) {
                    CLog.e("充电功率过高");
                    handlerItemData(new ListItemBean(
                            "电池异常:充电功率过高(可能是云手机)",
                            ListItemBean.RiskLeave.Deadly,
                            "检测到过大的充电功率 -> " + String.format("%.2fW", Math.abs(chargingPower))
                    ));
                }
            }
        }
 
    }
};

```

#### 检测摄像头 & 传感器相关：

判断摄像头有个数 。

```
try {
    CameraManager manager = (CameraManager) context.getSystemService(Context.CAMERA_SERVICE);
    String[] cameraIds = manager.getCameraIdList();
    //摄像头个数
    CLog.i("cameraIds -> "+ Arrays.toString(cameraIds));
    if(cameraIds.length < CAMERA_MINIMUM_QUANTITY_LIMIT){
        items.add(
                new ListItemBean(
                "当前手机可能是模拟器&云手机",
                ListItemBean.RiskLeave.Warn,
                "camera size -> "+cameraIds.length
        ));
    }
} catch (Throwable ignored) {
 
}

```

#### [](#检测传感器个数：)检测传感器个数：

这块思路就是直接获取个数，少于 10 个可以直接认定为黑产 。 我目前没发现那个手机少于 10 个传感器 ，这块如果可能的话可以尝试调用一下传感器，保证传感器是否可用 ，防止云手机以假乱真 。

```
try {
    //3,检测传感器类型,支持的全部类型传感器
    SensorManager sm = (SensorManager) context.getSystemService(SENSOR_SERVICE);
    List sensorlist = sm.getSensorList(Sensor.TYPE_ALL);
 
    ArrayList sensorTypeS = new ArrayList<>();
    for (Sensor sensor : sensorlist) {
        //获取传感器类型
        int type = sensor.getType();
        if (!sensorTypeS.contains(type)) {
            //发现一种类型则添加一种类型
            sensorTypeS.add(type);
        }
    }
    //小米k40 51个传感器类型
    //普通的pix 27个
    //华为荣耀20 18个传感器
    CLog.e("sensor types size -> " + sensorlist.size());
    //我们认为传感器少于20个则认为是风险设备
    if (sensorlist.size() < SENSOR_MINIMUM_QUANTITY_LIMIT) {
        items.add(new ListItemBean(
                "当前手机可能是模拟器&云手机",
                ListItemBean.RiskLeave.Warn,
                "sensor size -> ("+ sensorlist.size()+") \n" +
                "sensor type size -> ("+sensorTypeS.size()+") \n"
                //+ "sensor info -> \n"+ Sensorlist   //打印全部传感器信息
        ));
    } 
```

#### [](#检测传感器名称：)检测传感器名称：

这块检测思路主要是检测传感器的名称，正常小米之类的手机他是不可能存在叫什么 AOSP 的传感器的 。

 

这种 AOSP 基本都是自己编译的 ROM ，所以这块也可以作为监测点 。可以上报传感器的一些名称信息，也是环境检测一个很重要的抓手 。

 

一般小白肯定不会说去改传感器名称 。

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
    ArrayList aospSensor = new ArrayList<>();
    for(Sensor sensor:sensorlist){
        if(sensor.getVendor().contains("AOSP")){
            aospSensor.add(sensor);
        }
    }
    if (aospSensor.size()>3) {
        CLog.e("传感器参数是否异常(生产厂商为AOSP)");
        items.add(new ListItemBean(
                "当前手机可能是模拟器&云手机",
                ListItemBean.RiskLeave.Warn,
                aospSensor.size()
                        +"/"+sensorlist.size()+"传感器参数异常 -> "+ aospSensor
        ));
    }
} 
```

#### [](#检测挂载文件：)检测挂载文件：

这块就是去遍历 mounts 下面这几个文件，检测里面是否包含 docker 关键字 ，防止一些云手机搞虚拟化，通过使用 docker 进行挂载 。

 

这块也是很好的监测点 。

```
String[] marks = {"docker"};
//检测proc/mounts是否包含docker关键字
String mark = NativeEngine.getZhenxiInfoK("/proc/mounts",marks );
if(mark == null){
    mark = NativeEngine.getZhenxiInfoK("/proc/self/mountstats", marks);
    if(mark == null){
        mark = NativeEngine.getZhenxiInfoK("/proc/self/mountinfo", marks);
    }
}
if(mark!=null){
    items.add(new ListItemBean(
            "当前手机可能是模拟器&云手机",
            ListItemBean.RiskLeave.Warn,
            "(mounts异常)\n"+mark
    ));
}

```

#### 检测 ROM 是否 Match:

##### [](#检测环境信息：)检测环境信息：

这块思路主要好多种 ，主要是为了防止一些自定义 ROM ，通过修改机型的方法，绕过自定义 ROM 检测逃逸 。

 

可以直接执行 getprop 把所有的环境信息都拿到手 ，如果是小米手机，里面环境信息里面，肯定是有 MIUI 关键字。

 

比如小米的手机，我会去检测是否包含这几个关键环境信息 。

```
private static final String KEY_MIUI_VERSION_NAME = "ro.miui.ui.version.name";
private static final String KEY_MIUI_VERSION_CODE = "ro.miui.ui.version.code";
private static final String KEY_MIUI_INTERNAL_STORAGE = "ro.miui.internal.storage";

```

这块可以采集以后服务端进行判断 。防止自定义 ROM 机型伪造 。

##### [](#检测服务列表：)检测服务列表：

这块还是执行 service list ，一般小米手机之类的，都会有小米的系统服务，这种东西很难去伪造，如果他伪造了假的 。你就尝试调用即可 。

 

这块还是建议上传到服务端，由服务端算法同学去根据相似度算法去推断 ，不要再本地进行判断 ，因为 Hunter 是非联网 Apk，所以只是在客户端打了个样子 。

### 检测当前环境是否被 Hook:

这块检测方法千奇百怪，首先最基本 maps 去检测 frida 或者根据调用栈检测 lsp 特征 ，基础的检测方案不说了 。因为我觉得并不是一个很好的方案 。改个名就绕过了 。

 

比如 frida 特征三件套 。检测思路主要

```
static const char *FRIDA_THREAD_GUM_JS_LOOP = "gum-js-loop";
static const char *FRIDA_THREAD_GMAIN = "gmain";
static const char *FRIDA_NAMEDPIPE_LINJECTOR = "linjector";

```

Hook 检测，我们其实只需要检测内存没有被修改即可 。这块需要介绍一下基础原理。和实现的伪代码 。

 

正常我们知道一个 SO 加载到内存里，本质上是通过 mmap 把 so 分配到内存里面 ，比如 A 函数的指令是 BBB，那么加载到内存里面应该也是 BBB 。

 

记住上面这句话 ，我们就可以对内存里面的指令转换成一个 int 值，然后累加 。如果内存没有被修改 ，累加值文件里面和内存里面的值应该是一样的 。

 

因为现在 Hook 基本都是 text 段和 plt 端，一个 inlinehook 一个 got 表 。当然 Frida 可能会延迟启动 ，所以开启一条检测线程，进行轮训操作。

 

这块还有一个设计问题，很多开发者也都没注意到 ，就是我开启的这一条线程，被攻击者 anti 掉 应该怎么办呢？

 

因为想要 anti 掉一条检测线程，方法太多了，N 种方法，比如监听全部线程的文件读写，看看那个线程在读取文件 ，只做这一件事，基本八九不离十 ，

 

也可以直接 Hook 开启线程的方法，对开启的线程进行 anti 。当然这种思路 就没有好的对抗或者检测办法了么？

> **其实很简单 ，只需要把在你的检测代码里，对某个变量进行赋值 ，修改 flag 即可 。**
> 
> **然后第一次检测完毕以后将主进程的某个变量标志为 true，可以使用__NR_process_vm_writev ，又因为是异步的关系，主线程可以延迟 2 秒对这个标识进行获取，判断是否为 true ，**
> 
> **以确保检测线程成功开启 。**

 

具体检测流程如下，以检测 libc 为例子，路径可以换成自己需要的路径 ：

 

首先获取本地 So 文件的累加值 ，返回 execSection 结构体 。

```
//记录可执行段的结构体,一个是plt段一个是text段
//所以对应的数量是2
typedef struct stExecSection {
    int execSectionCount;
    unsigned long offset[2];
    unsigned long memsize[2];
    unsigned long checksum[2];
    unsigned long startAddrinMem;
    bool isSuccess = false;
} execSection;
 
 
/**
 * 获取本地文件的 Check sum
 * 读取耗时操作,只初始化一次保存到本地。
 */
execSection fetch_checksum_of_library(const char *filePath) {
    execSection section = {0};
    Elf_Ehdr ehdr;
    Elf_Shdr sectHdr;
    int fd;
    int execSectionCount = 0;
    fd = my_openat(AT_FDCWD, filePath, O_RDONLY, 0);
    if (fd < 0) {
        return section;
    }
 
    my_read(fd, &ehdr, sizeof(Elf_Ehdr));
    my_lseek(fd, (off_t) ehdr.e_shoff, SEEK_SET);
 
    unsigned long memSize[2] = {0};
    unsigned long offset[2] = {0};
 
    //查找section的plt和text开始位置和长度
    for (int i = 0; i < ehdr.e_shnum; i++) {
        my_memset(§Hdr, 0, sizeof(Elf_Shdr));
        my_read(fd, §Hdr, sizeof(Elf_Shdr));
        //通常 PLT and Text 一般都是可执行段
        if (sectHdr.sh_flags & SHF_EXECINSTR) {
            offset[execSectionCount] = sectHdr.sh_offset;
            memSize[execSectionCount] = sectHdr.sh_size;
            execSectionCount++;
            if (execSectionCount == 2) {
                break;
            }
        }
    }
    if (execSectionCount == 0) {
        LOG(INFO) << "get elf section error " << filePath;
        my_close(fd);
        return section;
    }
 
    //记录个数
    section.execSectionCount = execSectionCount;
 
    section.startAddrinMem = 0;
    for (int i = 0; i < execSectionCount; i++) {
        my_lseek(fd, (off_t) offset[i], SEEK_SET);
        //void * buffer = alloca(memSize[i] * sizeof(uint8_t));
        //存放text或者plt全部的数据内容,大约5-10M大小,为了兼容小内存手机。
        //所以放在堆里面,而不是栈,防止小内存手机栈指针溢出。
        auto buffer = (void *) calloc(1, memSize[i] * sizeof(uint8_t));
        if (buffer == nullptr) {
            free(buffer);
            return section;
        }
        my_read(fd, buffer, memSize[i]);
        section.offset[i] = offset[i];
        section.memsize[i] = memSize[i];
        section.checksum[i] = checksum(buffer, memSize[i]);
 
//        LOGE("fetch_checksum_of_library %s ExecSection:[%d][%ld][%ld][%ld]",
//             filePath, i, offset[i], memSize[i], section->checksum[i])
        free(buffer);
    }
    section.isSuccess = true;
    my_close(fd);
    return section;
}

```

然后和本地的指令去计算 。计算本地的指令方法就是对 maps 进行遍历，只遍历 text 和 plt 段 ，计算累加值和本地进行判断 。

```
/**
 * 检测问的check sum
 * 检测到check未修改返回0
 * 检测已修改返回1
 * 检测失败返回-1
 */
int detect_elf_checksum(const char *soPath, execSection *pSection) {
    if (pSection == nullptr) {
        LOGI("detect_elf_checksum execSection == null  ");
        return -1;
    }
 
    char map[MAX_LINE];
    const char *maps_path = string("proc/").append(to_string(getpid())).append("/maps").c_str();
 
    int fd = my_openat(AT_FDCWD, maps_path, O_RDONLY, 0);
 
    if (fd <= 0) {
        LOGE("detect_elf_checksum open %s fail ", PROC_MAPS);
        return -1;
    }
    int checkSum = 0;
    while ((read_one_line(fd, map, MAX_LINE)) > 0) {
        if (my_strstr(map, soPath) != nullptr) {
            checkSum = scan_executable_segments(map, pSection, soPath);
            if (checkSum == 1) {
                break;
            }
        }
    }
 
    my_close(fd);
    return checkSum;
}
 
/**
 * 检测问的check sum
 * 检测到check未修改返回0
 * 检测已修改返回1
 * 检测失败返回-1
 */
int scan_executable_segments(
        char *mapItem,
        execSection *pElfSectArr,
        const char *libraryName) {
    unsigned long start, end;
    char buf[MAX_LINE] = "";
    char path[MAX_LENGTH] = "";
    char tmp[100] = "";
 
    sscanf(mapItem, "%lx-%lx %s %s %s %s %s", &start, &end, buf, tmp, tmp, tmp, path);
 
    if (buf[2] == 'x') {
        if (buf[0] == 'r') {
            uint8_t *buffer;
 
            buffer = (uint8_t *) start;
            for (int i = 0; i < pElfSectArr->execSectionCount; i++) {
                if (start + pElfSectArr->offset[i] + pElfSectArr->memsize[i] > end) {
                    if (pElfSectArr->startAddrinMem != 0) {
                        buffer = (uint8_t *) pElfSectArr->startAddrinMem;
                        pElfSectArr->startAddrinMem = 0;
                        break;
                    }
                }
            }
            for (int i = 0; i < pElfSectArr->execSectionCount; i++) {
                auto begin = (void *) (buffer + pElfSectArr->offset[i]);
                unsigned long size = pElfSectArr->memsize[i];
                LOGI("%s [%p] size ->[%lu]", libraryName, begin, size);
                //MPROTECT((size_t)begin, size, MEMORY_R);
                unsigned long output = checksum(begin, size);
                LOGI("%s checksum:[%ld][%ld]", libraryName, output, pElfSectArr->checksum[i])
                if (output != pElfSectArr->checksum[i]) {
                    //和本地的So Checksum 对不上
                    return 1;
                }
            }
        }
        return 0;
    } else {
        if (buf[0] == 'r') {
            pElfSectArr->startAddrinMem = start;
        }
    }
    return 0;
}

```

当然这块就一定是比较完善的检测方案了么？其实有很多对抗思路 。比如把 maps 里面的内存进行隐藏，变成匿名内存，这样他在扫描 maps 的就找不到，对应的比如上面说的 libc 的 item ，

 

这块可以参考 riru 里面的 map_hide ，代码路径如下 ：

 

https://github.com/RikkaApps/Riru/blob/master/riru/src/main/cpp/hide_utils.cpp

 

当然这种方式一定是安全的么？

> 其实，不是的，因为在他的这个代码里面我发现存在一个遗漏点，可以作为检测入手点 ，某个加固厂商，不仅仅会对 maps item 进行遍历 ，还会对里面的匿名内存进行遍历 ，检测匿名内存里面的 magic ，比如 so 文件的 magic 是 elf ，如果 magic 匹配上一样当 maps 去解析 去遍历。
> 
> 所以可以在原有的基础上改改，在将内存变成匿名内存以后，把 elf 的前四个字节抹掉，也就是 magic 的 内容，抹掉以后记得把权限修改成和之前的一样 。防止内存检测 。

### [](#检测沙箱：)检测沙箱：

这块检测核心逻辑全部放在 ISO 线程检测 。可以配置一个服务，然后服务里使用如下变量即可 。

这个服务非常恶心，一般沙箱对这个线程都会进行跳过 。

 

这块有人可能会问什么是 iso 线程？可以理解成一个独立的安全的线程，只能通过和外部 IPC 交互的方式进行通讯 。useAppZygote 相当于让这个进程运行在 Zygote 中 。这个时候时机特别早，早到什么程度呢？就连 libart.so 都没加载，所以这个检测进程只能调用一些原始的 libc 方法，不能调用任何 Art 相关的函数 。

 

再多的我也不知道了 ，这块具体什么是 ISO 线程，让 GTP 回答一下吧，回答如下 ：

```
我:
请你分别介绍一下这两个标签的详细含义 .
android:isolatedProcess="true"
android:useAppZygote="true" 
以及他们的使用场景 ?
 
 
ChatGPT:
这两个属性都属于Android清单文件（Android Manifest）中标签的属性。它们控制你的应用服务（Service）如何运行。
 
android:isolatedProcess="true"：
 
android:isolatedProcess属性控制服务是否在一个特殊的孤立进程（isolated process）中运行。如果此属性设置为 "true"，那么服务将运行在一个没有任何权限的孤立进程中。这个进程是应用的一个私有进程，其不与应用的其他部分共享同一Linux用户ID和权限。由于孤立进程不具备任何权限，因此它无法访问设备的大部分资源，除非通过IPC机制向其他进程请求。这种方式可以提高安全性，因为即使孤立进程被攻破，攻击者也无法直接访问应用的其他资源。一般来说，如果你的服务需要处理敏感数据或执行敏感操作，可以考虑使用孤立进程。
 
android:useAppZygote="true"：
 
android:useAppZygote这个属性指定了应用的服务是否应该在一个专用的AppZygote进程中运行。AppZygote进程是一个预加载了应用共享资源和一些类的轻量级进程，可以更快地启动服务。如果此属性设置为"true"，那么服务将在一个AppZygote进程中运行。这种方式可以提高服务启动速度，但可能会导致内存消耗增加。一般来说，如果你的服务需要频繁启动，而且启动速度很关键，你可以考虑使用AppZygote进程。
 
请注意，这两个属性的具体行为可能取决于Android的版本和设备制造商的实现，你可能需要在实际设备上进行测试以确认它们的行为。 
```

下面也是主要介绍一下 hunter 的检测思路 。

#### 检测多余线程 PID:

主要实现思路就是去检测 proc 下面是否有除了 main 进程以外的其他 pid , 因为正常启动的话，肯定是只有一个 main 进程 。

 

但是沙箱的话会在启动之前去启动别的进程 ，所以这块可以进行 bypass 。后面我会统一说这块应该如何对抗 ，包括如何绕过 。

 

这块先介绍检测思路 ，和检测原理 。这块里面有一个 replaceSecInsns 是我自己封装的一个函数，我担心 opendir 被 Hook 了，所以每次执行都去把指令替换成

 

本地文件的指令，而不是去执行内存里面的指令 。

```
auto orig_opendir = reinterpret_cast

(replaceSecInsns(getlibcPath(), "opendir"));
    DIR *pdr = orig_opendir("/proc");
    if (pdr == nullptr) {
        return getItemData(env, "程序出错请放弃修改后重试",
                           nullptr, true, RISK_LEAVE_DEADLY, TAG_SANDBOX);
    }
    auto orig_ator = reinterpret_cast (replaceSecInsns(getlibcPath(), "readdir"));
    dirent *read_ptr;
    //在app启动之前检测当前app所有的进程,判断是否存在和main不一样的进程
    while ((read_ptr = orig_ator(pdr)) != nullptr) {
        long procPid = strtol(read_ptr->d_name, nullptr, 10);
        //LOG(INFO) << "find /proc/ child dir  " << procPid;
        //打开成功&&发现一条不等于主进程id的pid
        if (procPid && procPid != getpid()) {
            auto title = string("检测到APK存在沙箱内部");
            char buff[200];
            getNameByPid((pid_t) procPid, buff);
            LOG(ERROR) << ">>>>>  FIND OTHER THREAD SANDBOX " << procPid << "  " << buff;
            auto &data =
                    string("当前线程主进程PID(").append(to_string(getpid())).append(")").append("\n").
                            append("异常pid -> ").append(to_string(procPid)).
                            append("(").append("pid name: ").append(buff).append(")");
            //可能存在多个异常pid而非一个
            getItemData(env, title, data.c_str(), false, RISK_LEAVE_WARN, TAG_SANDBOX);
        }
    }
    closedir(pdr); 


```

##### [](#对抗：)对抗：

这块想要绕过也很简单，opendir 底层其实就是 getdents64，getdents 如果遇到你想隐藏的文件直接对文件进行 bypass 就可以了，直接指向下一个指针 ，

 

代码来自 proot

```
        case SC_getdents64:
        case SC_getdents: {
            /* get the result of the syscall, which is the number of bytes read by getdents
             * 获取系统调用的结果，即getdents读取的字节数
             * */
            unsigned int res = peek_reg(tracee, CURRENT, SYSARG_RESULT);
            if (res <= 0) {
                break;
            }
 
            /* get the system call arguments */
            word_t orig_start = peek_reg(tracee, CURRENT, SYSARG_2);
            unsigned int count = peek_reg(tracee, CURRENT, SYSARG_3);
            char orig[count];
 
            char path[PATH_MAX];
            int status = readlink_proc_pid_fd(tracee->pid,
                                              (int) peek_reg(tracee, ORIGINAL, SYSARG_1), path);
            if (status < 0) {
                break;
            }
            //不属于绑定路径则不处理,这块应该加个判断。如果这个path没有处理的路径应该返回 。
            //目前需要处理的只有proc,用于隐藏调试线程。
//            if(!belongs_to_guestfs(tracee, path))
//                return 0;
            if (!StringUtils::containsInsensitive(path,"proc")){
                break;
            }
            /* retrieve the data from getdents
             * 从getdents检索数据
             * */
            status = read_data(tracee, orig, orig_start, res);
            if (status < 0) {
                break;
            }
 
            /* allocate a space for the copy of the data we want
             * 为我们想要的数据的副本分配一个空间
             * */
            char copy[count];
            /* curr will hold the current struct we're examining
             * curr将保存我们正在检查的当前结构
             * */
            struct linux_dirent64 *curr64;
            struct linux_dirent *curr32;
            /* pos keeps track of where in memory the copy is
             * pos跟踪副本在内存中的位置
             * */
            char *pos = copy;
            /* ptr keeps track of where in memory the original is
             * ptr跟踪原始文件在内存中的位置
             * */
            char *ptr = orig;
            /* nleft keeps track of how many bytes we've saved
             * nleft跟踪我们保存了多少字节
             * */
            unsigned int nleft = 0;
 
            /* while we're still within the memory allowed
             * 当我们还在允许的memory范围内时
             * */
            if (get_sysnum(tracee, ORIGINAL) == SC_getdents64) {
                while (ptr < orig + res) {
 
                    /* get the current struct
                     * 获取当前结构
                     * */
                    curr64 = (struct linux_dirent64 *) ptr;
 
                    /* if the name does not matche a given prefix
                     * 如果名称与给定前缀不匹配
                     *
                     * 如果这个目录项的名称不以HIDDEN_PREFIX开始，
                     * 也就是hasprefix(HIDDEN_PREFIX, curr64->d_name)返回false，
                     * 那么这个目录项会被保留在结果中，否则，这个目录项会被忽略，也就是隐藏。
                     * */
                    //if (!hasprefix(HIDDEN_PREFIX, curr64->d_name)) {
                    if (!isRuntimeHideDir(tracee->pid,curr64->d_name, tracer_pc(tracee), tracer_lr(tracee))) {
                        /* copy the information
                         * 复制信息
                         * */
                        mybcopy(ptr, pos, curr64->d_reclen);
 
                        /* move the pos and nleft */
                        pos += curr64->d_reclen;
                        nleft += curr64->d_reclen;
                    }
                    /* move to the next linux_dirent
                     * 隐藏这个目录,指向下一个文件
                     * */
                    ptr += curr64->d_reclen;
                }
            } else {
                while (ptr < orig + res) {
 
                    /* get the current struct */
                    curr32 = (struct linux_dirent *) ptr;
 
                    /* if the name does not matche a given prefix */
//                    if (!hasprefix(HIDDEN_PREFIX, curr64->d_name)) {
                    if (!isRuntimeHideDir(tracee->pid,curr32->d_name, tracer_pc(tracee), tracer_lr(tracee))) {
 
                        /* copy the information */
                        mybcopy(ptr, pos, curr32->d_reclen);
 
                        /* move the pos and nleft */
                        pos += curr32->d_reclen;
                        nleft += curr32->d_reclen;
                    }
                    /* move to the next linux_dirent */
                    ptr += curr32->d_reclen;
                }
            }
            /* If there is nothing left
             * 这个文件夹里面本身没有东西,暂不处理
             * */
            if (!nleft) {
//                /* call getdents again */
//                if (get_sysnum(tracee, ORIGINAL) == PR_getdents64)
//                    register_chained_syscall(tracee, PR_getdents64, peek_reg(tracee, ORIGINAL, SYSARG_1), orig_start, count, 0, 0, 0);
//                else
//                    register_chained_syscall(tracee, PR_getdents, peek_reg(tracee, ORIGINAL, SYSARG_1), orig_start, count, 0, 0, 0);
            } else {
                /* copy the data back into the register */
                status = write_data(tracee, orig_start, copy, nleft);
                if (status < 0) {
                    break;
                }
                /* update the return value to match the data */
                poke_reg(tracee, SYSARG_RESULT, nleft);
            }
            break;
        }

```

#### [](#执行ps命令：)执行 ps 命令：

这块的思路就是检测是否存在其他进程 ，通过执行 ps -ef 命令 。

```
if (get_sdk_level() > ANDROID_Q) {
    //check process
    auto orig_popen = reinterpret_cast(replaceSecInsns(getlibcPath(),
                                                                                "popen"));
    FILE *file = orig_popen("ps -ef", "r");
    if (file == nullptr) {
        return getItemData(env, "程序出错请放弃修改后重试",
                           "ps error", true, 3, TAG_SANDBOX);
    }
    char buf[0x1000];
    string buffStr;
    uint size = 0;
    // get process count size
    while (fgets(buf, sizeof(buf), file)) {
        buffStr.append(buf).append("\n");
        //不包含++
        if(!StringUtils::contains(buf,"hunter")){
            size++;
            LOG(INFO) << "ps -ef match -> %s " << buf;
        }
    }
    if (size > 2) {
        //UID             PID   PPID C STIME TTY          TIME CMD
        //u0_a531        6187    885 72 10:27:53 ?    00:00:00 com.zhenxi.hunter
        //u0_a531        6236   6187 0 10:27:53 ?     00:00:00 ps -ef
        pclose(file);
 
        return getItemData(env, "检测到APK存在沙箱内部(异常线程)",
                           buffStr.c_str(), false, RISK_LEAVE_DEADLY, TAG_SANDBOX);
    }
    LOG(ERROR) << ">>>>> NOT FIND SANDBOX ";
    pclose(file);
} 
```

##### [](#对抗：)对抗：

可以先伪造一个正常的文件，在执行到 ps -ef 的时候把命令换成 cat 你自己的文件 即可 。

#### 内存 Choose:

这块的实现思路就是，检测内存里面的 Activity 或者 Application 的个数 。正常我们的 apk 启动只会有我们自己的

 

Application，沙箱因为预启动的关系，会存在其他的 Application ，这块也是一个很好的检测思路 。

```
public static ListItemBean checkSandbox() {
    ArrayList choose;
    try {
        choose = ChooseUtils.choose(Activity.class, true);
    } catch (Throwable e) {
        return null;
    }
    if (choose != null) {
        ArrayList list = new ArrayList<>();
        //移除我们的Activity,把其他的activity实例加到List里面
        for(Object activty:choose){
            String name = activty.getClass().getName();
            if(!name.equals(MainActivity.class.getName())&&
                    !name.equals(ShareActivity.class.getName())&&
                        !name.equals(FeedbackActivity.class.getName())
                ){
                list.add(activty);
            }
        }
        //判断list数量是否大于1
        if (list.size() >= 1) {
            ListItemBean item = new ListItemBean(
                    "检测到APK存在沙箱内部",
                    ListItemBean.RiskLeave.Deadly);
            for (Object obj : list) {
                item.putData(obj.getClass().getName());
            }
            return item;
        }
    }
    return null;
} 
```

### 检测 Google&lineageos

因为在国内手机里基本 Google 和 lineageos 都是黑产的标配 ，大部分都是自己刷的 ROM 。

 

这种手机理论上在国内 不应该出现，如果出现也会被打上标签，被认定为黑产 。可以直接 Build.MODEL 获取厂商 。

 

另外这块还有一个细节点 ，就是 lineageos 一个特有文件 / system/addon.d 。具体如下 。

```
private static void isLineageOs(ArrayList items) {
    //lineage检测
    String display = NativeEngine.getZhenxiInfoH("ro.build.display.id");
    if (display.toLowerCase(Locale.ROOT).contains("lineage")) {
        items.add(
                new ListItemBean("当前手机为Lineage系统",
                        ListItemBean.RiskLeave.Warn,
                        "可能存在自定义ROM,当前设备不可信！"
                ));
        return;
    }
    //lineage 特有文件
    File file = new File("/system/addon.d");
    if (file.exists()) {
        items.add(
                new ListItemBean("当前手机为Lineage系统",
                        ListItemBean.RiskLeave.Warn,
                        "可能存在自定义ROM,当前设备不可信！"
                ));
        return;
    }
} 
```

当然这块有一个细节点：

 

直接获取 ro.build.display.id 可能会被 Hook ，所以这块我的建议是直接去解析配置文件 。

 

核心解析（/dev/**properties**/u:object_r: ）文件方法如下 ：

```
    public PropArea(String area) throws IOException {
        area = "/dev/__properties__/u:object_r:" + area + ":s0";
        File file = new File(area);
        if (!file.isFile()) throw new FileNotFoundException("Not a file: " + area);
        long size = file.length();
        if (size <= 0 || size >= 0x7fffffffL) throw new IllegalArgumentException("invalid file size " + size);
 
        try (FileChannel channel = new FileInputStream(area).getChannel()) {
            data = channel.map(FileChannel.MapMode.READ_ONLY, 0, size).order(ByteOrder.nativeOrder());
        }
 
        byteUsed = data.getInt();
        data.getInt(); // serial
        int magic = data.getInt();
        if (magic != PROP_AREA_MAGIC) throw new IllegalArgumentException("Bad file magic: " + magic);
        int version = data.getInt();
        if (version != PROP_AREA_VERSION) throw new IllegalArgumentException("Bad area versin: " + version);
        data.position(data.position() + 28); // reserved
    }
 
public List findPossibleValues(String name) {
    List values ;
    try {
        //  atomic_uint_least32_t serial;
        //  union {
        //    char value[PROP_VALUE_MAX];
        //    struct {
        //      char error_message[kLongLegacyErrorBufferSize];
        //      uint32_t offset;
        //    } long_property;
        //  };
        final int LONG_PROP_FLAG = 1 << 16;
        final int PROP_VALUE_MAX = 92;
        final int VALUE_OFFSET = 4;
        final int NAME_OFFSET = VALUE_OFFSET + 92;
        values = new ArrayList<>(2);
        findFromBuffer(data.slice(), name.getBytes(StandardCharsets.UTF_8), (buffer, offset) -> {
            if (offset < NAME_OFFSET) return;
            int base = offset - NAME_OFFSET;
            int serial = buffer.getInt(base);
            if ((serial & LONG_PROP_FLAG) != 0) return; // Long properties are not supported
            values.add(toString(buffer, base + VALUE_OFFSET, PROP_VALUE_MAX));
        });
        CLog.i("Found " + name + "=" + values);
        return values;
    } catch (Throwable e) {
        CLog.e("findPossibleValues get error "+ name+" "+e);
        return null;
    }
} 
```

### [](#检测ida和反调试：)检测 IDA 和反调试：

这块检测方法太多了，我知道的就不下于 10 种，比如核心 A 和 B 方法里面加个时间搓，如果 A 执行到 B 大于 5 秒就可以认为被调试 。

 

检测调试状态也是很不错的选择，不过这块可以利用 ISO 线程去检测调试状态，防止被 ptrace

#### service list

也可以执行 "service list" 获取服务列表 ，判断是否包含 frida 关键字 。

```
public static CheckServerRet checkServer(Context context, String[] fit) {
    CheckServerRet ret = new CheckServerRet();
    ArrayList list = ret.list;
    BufferedReader reader = null;
    Process process;
    try {
        // 执行 service list 命令
        process = Runtime.getRuntime().exec("service list");
 
        // 读取命令输出结果
        reader = new BufferedReader(new InputStreamReader(process.getInputStream()));
        String line;
        while ((line = reader.readLine()) != null) {
            if (line.length() > 1) {
                for (String fitItem : fit) {
                    //比较不区分大小写
                    if (line.toLowerCase().contains(fitItem.toLowerCase())) {
                        list.add(line);
                        //CLog.e("checkServer find item is match " + Arrays.toString(fit) + " " + line);
                        break;
                    } else {
                        //走到这里说明能拿到。返回的不是空
                        ret.isSuccess = true;
                    }
                }
            }
        }
 
        // 关闭流
        reader.close();
 
        // 等待命令执行完毕
        int exitValue = process.waitFor();
        CLog.e("checkServer ret mark " + Arrays.toString(fit) + " " + ret);
        return ret;
    } catch (IOException | InterruptedException e) {
        e.printStackTrace();
        if (reader != null) {
            try {
                reader.close();
            } catch (IOException ignored) {
 
            }
        }
    }
    return ret;
} 
```

[](#后语：)后语：
-----------

如果你使用了这个帖子里面的代码，请备注出处 。感谢理解 ！

  

[[2023 春季班]2023，新的征程，脱壳机更新、iOS/eBPF、赠送云手机套装！一块裸板虚拟化五个容器云手机！3 月 25 日起同时上调价格并赠送新设备！](https://bbs.kanxue.com/thread-276361.htm)

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#NDK 分析](forum-161-1-119.htm) [#协议分析](forum-161-1-120.htm) [#混淆加固](forum-161-1-121.htm) [#脱壳反混淆](forum-161-1-122.htm) [#漏洞相关](forum-161-1-123.htm)
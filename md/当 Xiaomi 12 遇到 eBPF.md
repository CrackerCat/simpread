> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1652946-1-1.html)

> [md]# 当 Xiaomi 12 遇到 eBPF 最近有大佬在 android 上实践 ebpf 成功 前有 evilpan 大佬：https://bbs.pediy.com/thread-271043......

![](https://avatar.52pojie.cn/data/avatar/000/97/63/56_avatar_middle.jpg)huaerxiela _ 本帖最后由 huaerxiela 于 2022-6-23 14:49 编辑_  

当 Xiaomi 12 遇到 eBPF
===================

最近有大佬在 android 上实践 ebpf 成功 <br>  
前有 evilpan 大佬：[https://bbs.pediy.com/thread-271043.htm](https://bbs.pediy.com/thread-271043.htm) <br>  
后有 weishu 大佬：[https://mp.weixin.qq.com/s/mul4n5D3nXThjxuHV7GpMA](https://mp.weixin.qq.com/s/mul4n5D3nXThjxuHV7GpMA) <br>  
当然还有其他隐藏的大佬啦，就不一一列举啦 <br>  
遂 android-ebpf 大火 <br>  
两位大佬的方案也很有代表性，一个是 androdeb + 自编内核 + 内核移植 + 内核 4.19（文章中看的），一个是 androdeb + 内核 5.10(pixel 6)<br>  
目前来看，androdeb + 高版本内核 方案可以更快上手，花钱投资个新设备就好了，而且 weishu 大佬也已经手把手把工具都准备好了 <br>  
故本次就是对 weishu 大佬视频号直播的 "搭建 Android eBPF 环境" 的文字实践 + 反调试样本测试 <br>

eBPF 是啥
-------

> 来自官方的说法：[https://ebpf.io/what-is-ebpf](https://ebpf.io/what-is-ebpf)  
> 来自大佬的总结：[https://mp.weixin.qq.com/s/eI61wqcWh-_x5FYkeN3BOw](https://mp.weixin.qq.com/s/eI61wqcWh-_x5FYkeN3BOw)

失败尝试
----

> 魅族 18 内核版本 5.4 <br>  
> 虽说环境编译成功了，但体验脑壳疼  <br>  
> opensnoop 没有 path  <br>  
> execsnoop pwd 命令监控不到，长命令被截断  <br>

环境准备
----

> PC 环境：macOS  <br>  
> 小米 12 内核版本 5.10.43 <br>  
> magisk 提供 root  <br>  
> androdeb 连接方式选取的也是 ssh 方式，故安装 SSH for Magisk 模块提供 ssh 功能  <br>  
> 手机最好也科学上网一下吧，要 git 拉一些东西  <br>

环境准备 over，开干
------------

> #### 确保手机 ssh 已开启，先去 adb shell 中 ps 一下
> 
> ps -ef|grep sshd
> 
> #### 没问题的话，就查看下 PC 上的 ssh key
> 
> cat ~/.ssh/id_rsa.pub
> 
> #### 然后把 key 粘贴到手机 authorized_keys 文件中，再改下权限
> 
> su <br>  
> cd /data/ssh/root/.ssh/ <br>  
> /data/adb/magisk/busybox vi authorized_keys <br>  
> chmod 600 authorized_keys <br>
> 
> #### 再看下手机 ip（因为是 ssh 连接，故手机和 PC 在同一局域网下）
> 
> ifconfig |grep addr
> 
> #### 在 PC 上测试下 ssh 是否可以成功连接
> 
> ssh root@手机 ip
> 
> #### 没问题的话，直接开搞准备好的 androdeb 环境了（weishu 大佬用 rust 重写了叫 eadb）
> 
> sudo chmod 777 ./eadb-darwin <br>  
> ./eadb-darwin --ssh root@手机 ip  prepare -a androdeb-fs.tgz
> 
> #### 等待完成后，进 androdeb shell, 开始编译 bcc
> 
> ./eadb-darwin --ssh root@手机 ip  shell <br>  
> git clone [https://github.com/tiann/bcc.git](https://github.com/tiann/bcc.git) --depth=1 <br>  
> cd bcc && mkdir build && cd build <br>  
> cmake ..  
> make -j8 && make install
> 
> #### 等待成功后，就有各种工具可以用了
> 
> ```
> root@localhost:/usr/share/bcc/tools# ls
> argdist       btrfsdist     dbslower        exitsnoop     gethostlatency  klockstat       nfsdist           perlflow        pythonstat   runqslower   syncsnoop   tcpdrop           tplist      zfsslower
> bashreadline  btrfsslower   dbstat        ext4dist      hardirqs              kvmexit              nfsslower    perlstat        readahead    shmsnoop          syscount    tcplife           trace
> bindsnoop     cachestat     dcsnoop        ext4slower    inject              lib              nodegc           phpcalls        reset-trace  slabratetop  tclcalls    tcpretrans   ttysnoop
> biolatency    cachetop            dcstat        filelife      javacalls       llcstat              nodestat           phpflow        rubycalls    sofdsnoop          tclflow     tcprtt           vfscount
> biolatpcts    capable            deadlock        fileslower    javaflow              mdflush              offcputime   phpstat        rubyflow     softirqs          tclobjnew   tcpstates    vfsstat
> biopattern    cobjnew            deadlock.c        filetop       javagc              memleak              offwaketime  pidpersec        rubygc             solisten          tclstat     tcpsubnet    virtiostat
> biosnoop      compactsnoop  dirtop        funccount     javaobjnew      mountsnoop      old           profile        rubyobjnew   sslsniff          tcpaccept   tcpsynbl           wakeuptime
> biotop              cpudist            doc                funcinterval  javastat              mysqld_qslower  oomkill           pythoncalls        rubystat     stackcount   tcpcong     tcptop           xfsdist
> bitesize      cpuunclaimed  drsnoop        funclatency   javathreads     netqtop              opensnoop    pythonflow        runqlat      statsnoop          tcpconnect  tcptracer    xfsslower
> bpflist       criticalstat  execsnoop        funcslower    killsnoop       netqtop.c       perlcalls    pythongc        runqlen      swapin          tcpconnlat  threadsnoop  zfsdist
> ```
> 
> #### 👆👆👆得益于 weishu 大佬的手把手环境工具包，androdeb + 内核 5.10 的 eBPF 环境搭建起来就是这么简单

反调试样本实操
-------

> DetectFrida.apk 核心逻辑: [https://github.com/kumar-rahul/detectfridalib/blob/HEAD/app/src/main/c/native-lib.c](https://github.com/kumar-rahul/detectfridalib/blob/HEAD/app/src/main/c/native-lib.c)
> 
> #### 哎😆，这里我直接就拿山佬的实践来说，至于为啥后面再说
> 
> ![](https://user-images.githubusercontent.com/30793389/174941184-ba9e041d-87fa-4720-9964-e1ff45684114.png)  
> ![](https://user-images.githubusercontent.com/30793389/174941257-ff9a79da-615f-417a-bd9f-2177d2bd35b8.png)
> 
> #### 还少了一个关键的
> 
> ![](https://user-images.githubusercontent.com/30793389/174941406-69af8111-8a68-49bc-85e5-b2a50194c0af.png)
> 
> #### 手写 trace 干它
> 
> trace 'do_readlinkat"%s", arg2@user' --uid 10229 <br>
> 
> #### 再来一次
> 
> ![](https://user-images.githubusercontent.com/30793389/174941534-6e03e98b-90d3-4a15-8f65-7dc49ea79c5a.png)
> 
> #### 👆👆👆可以了，差不多了，这样分析已经为后续对抗 bypass 提供了很大的帮助
> 
> #### 当然了，上述只是最基础的操作，后续还得继续深入探索学习，解锁更多顶级玩法
> 
> #### 还有就是，其实我的 Xiaomi 12 还没搞好，在等解 BL 锁，至于秒解，我不想花钱，所以就拿山佬的实践来借花献佛，真是个好主意啊，哈哈😄

总结
--

基于内核级别的监控，让应用中所有的加固 / 隐藏 / 内联汇编等防御措施形同虚设，而且可以在应用启动的初期进行观察，让应用的一切行为在我们眼中无所遁形 <br>  
这是真真正正的降维打击，内核级的探测能力提供了无限可能，堪称：屠龙技 <br>

最后
--

文中用的工具和软件，我已经打包整理好了 <br>  
聊天界面不用回复 "ebpf" 即可 <br>

我通过百度网盘分享的文件：ebpf  
链接:[https://pan.baidu.com/s/17HsadIwAFhjrYMTrd33rng?fm=lk0](https://pan.baidu.com/s/17HsadIwAFhjrYMTrd33rng?fm=lk0) 提取码: 7t85  
复制这段内容打开「百度网盘 APP 即可获取」

再次感谢先行者大佬们的无私奉献，和为技术发展所做的贡献🎉🎉🎉 <br>![](https://avatar.52pojie.cn/data/avatar/001/53/22/67_avatar_middle.jpg)aonima 知识又增加了 ![](https://avatar.52pojie.cn/data/avatar/001/28/31/86_avatar_middle.jpg) yinpeiqi 知识又增加了![](https://avatar.52pojie.cn/data/avatar/000/95/45/41_avatar_middle.jpg)小蓝人 知识又增加了 ![](https://avatar.52pojie.cn/data/avatar/001/56/35/07_avatar_middle.jpg) Piz.liu 今天百度了解了一个新技术 eBPF![](https://static.52pojie.cn/static/image/smiley/default/46.gif)![](https://avatar.52pojie.cn/data/avatar/001/32/60/92_avatar_middle.jpg)maguoli123 小白的我就爱看这种文章 ![](https://avatar.52pojie.cn/data/avatar/001/34/08/56_avatar_middle.jpg) x179 虽然看不懂![](https://static.52pojie.cn/static/image/smiley/default/2.gif)但感觉很厉害 ![](https://avatar.52pojie.cn/data/avatar/000/79/20/78_avatar_middle.jpg) westerman 今天百度了解了一个新技术 eBPF![](https://avatar.52pojie.cn/data/avatar/001/79/11/54_avatar_middle.jpg)kkh123 身为小白的我不知道有什么用![](https://avatar.52pojie.cn/data/avatar/001/33/48/43_avatar_middle.jpg)林淮想当小白  
身为小白的我不知道有什么用
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [ruan777.github.io](https://ruan777.github.io/2020/08/25/mips64%E8%B0%83%E8%AF%95%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/)

[2020-08-25](https://ruan777.github.io/2020/08/25/mips64%E8%B0%83%E8%AF%95%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/)

这里要感谢下 Xpoint 的大佬，超级感谢！大佬的博客：[http://matshao.com/archives/](http://matshao.com/archives/)

[](#前言 "前言")前言
--------------

主要就是今年强网杯的一题`Mipsgame`，程序开了 PIE，直接用`qemu-mips64`起的话，程序没有 pie，栈也不是随机的，gdb 调试的时候还不能`crtl+c`，体验及其的不好，赛后 Xpoint 的大佬一步步的教了我怎么搭环境，搭完环境之后调试起来 mips64 的程序，感觉舒服多了，效果：

原来：

![](https://s1.ax1x.com/2020/08/25/dgUesO.png)

现在：

![](https://s1.ax1x.com/2020/08/25/dgaUc6.png)

支持`crtl+c`，`vmmap`也能看到程序的地址，而且每次启动都是地址都是随机的，体验不止上升了一点点，在配合 tmux，体验极佳

![](https://s1.ax1x.com/2020/08/25/dgaqg0.png)

[](#安装 "安装")安装
--------------

*   这里要用到的是 [buildroot](https://buildroot.org/) 这个神器，真的和它官网首页说的那样：**Making Embedded Linux Easy**，在官网的 [download](https://buildroot.org/download.html) 页面下载压缩包下来，解压就行
    
*   接着就是切到 buildroot 的目录下，想好你要编译的架构，这里我们是`mips64`，然后`make qemu_mips64_malta_defconfig`，怎么知道 make 后面的参数呢，可以`ls configs`下：
    
    ![](https://s1.ax1x.com/2020/08/25/dgw760.png)
    
    我们可以看到第四列和第五列是 qemu 开头的，用这些`config`可以生成 qemu 用的 vmlinux 还有 rootfs，很是方便
    
*   接着就是`make menuconfig`，编译过内核的应该对这个挺熟悉的，在 menuconfig 里，我们可以选择要安装`package`，这里就是最舒服的地方了，这里我们就选 ncat，还有 gdb，因为我们只是为了调试而已，当然你要玩的话，也可以安装 gcc，g++，git 之类的，这是我当时的选项：
    
    ![](https://s1.ax1x.com/2020/08/25/dgDPoD.png)
    

下面还有一个 strace 也可以勾上，ncat 在`networking applications`里：

![](https://s1.ax1x.com/2020/08/25/dgDGSs.png)

如果你没看见这些包，可以用搜索功能，里面会有这个包的依赖，把依赖勾上，就会出现，`Target packages->Show packages that are also provided by busybox`这个也勾上，这里是我当时 toolchain 的选项：

![](https://s1.ax1x.com/2020/08/25/dgD2m6.png)

这些都搞好了之后，save 下退出

*   现在可以执行 make 了，大概几十分钟之后，buildroot 会帮你准备好一切，你可以在 output 下的 images 目录看到输出：
    
    ```
    ruan@ubuntu  ~/buildroot-2020.05.1/output/images  ls        
    rootfs.ext2  start-qemu.sh  vmlinux
    ```
    

*   `./start-qemu.sh`可以直接启动`qemu-system`，但是为了我们能调试，我们得加个端口转发：
    
    ```
    ruan@ubuntu  ~/buildroot-2020.05.1/output/images  cat start-qemu.sh 
    #!/bin/sh
    IMAGE_DIR="${0%/*}/"
    
    if [ "${1}" = "serial-only" ]; then
        EXTRA_ARGS='-nographic'
    else
        EXTRA_ARGS='-serial stdio'
    fi
    
    export PATH="/home/ruan/buildroot-2020.05.1/output/host/bin:${PATH}"
    exec  qemu-system-mips64 -M malta -m 1024 -kernel ${IMAGE_DIR}/vmlinux  -drive file=${IMAGE_DIR}/rootfs.ext2,format=raw -append "rootwait root=/dev/hda"  ${EXTRA_ARGS} -nic user,hostfwd=tcp::3333-:3333,hostfwd=tcp::5555-:5555
    ```
    
    其实就是在后面加了`-nic user,hostfwd=tcp::3333-:3333,hostfwd=tcp::5555-:5555`，也是大佬教的
    
*   `./start-qemu.sh`启动内核，输入 root 登入进去，然后执行`ncat -vc "gdbserver 0.0.0.0:5555 ./httpd" -kl 0.0.0.0 3333`，这时候我们在 qemu 外面用 nc 连 3333 端口，这时候 qemu 里面就会执行`gdbserver 0.0.0.0:5555 ./httpd`，我们外面在用`gdb-multiarch` 去连这个 5555 端口：
    

![](https://s1.ax1x.com/2020/08/25/dgsMGQ.png)

现在可以在 gdb 里下断点了：

![](https://s1.ax1x.com/2020/08/25/dgsDMR.png)

往 3333 端口发数据，gdb 里就会断在相应的函数里，意思就是我们可以用 pwntools 里的 io 模块来连 3333 端口，然后就像正常的那样收发数据就行

[](#坑 "坑")坑
-----------

我们不仅要把程序`httpd`拷贝进去，还要把题目给的`ld`还有`libc`也一起拷贝进去，修改 lib 下的符号连接，不然程序跑不起来：

![](https://s1.ax1x.com/2020/08/25/dgyVSJ.png)

拷贝程序的话，我们可以先挂载`rootfs.ext2`，比如：`sudo mount -t ext2 ./rootfs.ext2 /tmp/rootfs`，挂载到`/tmp/rootfs`，然后把程序，libc，ld 直接拷贝进去，最后`umount`就行

[](#后记 "后记")后记
--------------

这个 buildroot 不仅可以搭建 mips64 的，其它的架构也是可以的，我觉得可以提前编译几个，做好准备，免得打比赛的时候还在搭环境，嘻嘻，这里在感谢下 [matshao](http://matshao.com/archives/) 大佬。

[

强网杯和钓鱼城 ctf 部分题解

](https://ruan777.github.io/2020/09/01/%E5%BC%BA%E7%BD%91%E6%9D%AF%E5%92%8C%E9%92%93%E9%B1%BC%E5%9F%8Ectf%E9%83%A8%E5%88%86%E9%A2%98%E8%A7%A3/)[

TCTF/0CTF2020 初赛 simple_echoserver

](https://ruan777.github.io/2020/07/26/TCTF2020%E5%88%9D%E8%B5%9Bsimple_echoserver/)
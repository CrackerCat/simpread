> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-261261.htm)

> [分享] 关于某个加固的第二个 SO 内存加载信息获取

[分享] 关于某个加固的第二个 SO 内存加载信息获取

2020-8-5 18:23 5867

### [分享] 关于某个加固的第二个 SO 内存加载信息获取

 [![](http://passport.kanxue.com/upload/avatar/981/981.png?1)](user-home-981.htm) [fxyang](user-home-981.htm) ![](https://bbs.kanxue.com/view/img/rank/7.png) 18  ![](http://passport.kanxue.com/pc/view/img/sun.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 2020-8-5 18:23  5867

最近看到不少人在问某个加固的第二个 SO 如何得到基地址和内存 dump 的问题。提供一个废弃的脚本，大家可以试试。个人只是好奇，不深究。

```
Java.perform(function () {
    Interceptor.attach(Module.getExportByName('libz.so', 'uncompress'), {
        onEnter: function (args) {
                if (args[2] != null) {
                    var memcpy_add = Module.findExportByName("libc.so", "memcpy");
                    console.log("memcpy:" + memcpy_add);
                    Interceptor.attach(memcpy_add, {
                        onEnter: function (args) {
                            console.log("begin:memcpy,len:" + args[2]);
                            console.log(hexdump(args[1]))
 
                            // if (args[2] == 0x100) {
                            //     pht = args[0];
                            //     console.log(hexdump(args[1]))
                            // }
 
                            // if (args[2] == 0x948) {
                            //     jmprel = args[0];
                            //     console.log(hexdump(args[1]))
                            // }
                            //
                            // if (args[2] == 0x4a58) {
                            //     rel = args[0];
                            //     console.log(hexdump(args[1]))
                            // }
                            //
                            // if (args[2] == 0xd8) {
                            //     dyn = args[0];
                            //     console.log(hexdump(args[1]))
                            // }
                            //
                            // if (args[2] == 0xbbff4) {
                            //     console.log(hexdump(args[1]));
                            //     var new_so_base = args[0];
                            //     console.log("newso_base:"+args[0])
                            // }
                            // if (args[2] == 0x38fc) {
                            //     console.log(hexdump(args[1]))
                            // }
                            // if (args[2] == 0xb4) {
                            //     console.log(hexdump(args[1]))
                            //     差不多可以开搞了
                            // }
                        }
                    })
                }
            },
 
        onLeave: function (retval) {
 
        }
    })
 
})

```

上面的脚本会得到下面的一些记录：  
begin:memcpy,len:0x4009  
0 1 2 3 4 5 6 7 8 9 A B C D E F 0123456789ABCDEF  
c9188a7b 83 5b 8b 2c 4b 2e 16 ff 34 7d 4f 4e ad 71 fd 75 .[.,K...4}ON.q.u  
c9188a8b 36 30 74 13 7c ca f9 cd 68 3e de b2 10 96 15 93 60t.|...h>......

 

begin:memcpy,len:0x2ec  
0 1 2 3 4 5 6 7 8 9 A B C D E F 0123456789ABCDEF  
e6200704 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 ................

 

begin:memcpy,len:0x4  
0 1 2 3 4 5 6 7 8 9 A B C D E F 0123456789ABCDEF  
c9080001 00 01 00 00

 

begin:memcpy,len:0x100  
0 1 2 3 4 5 6 7 8 9 A B C D E F 0123456789ABCDEF  
c9080005 d8 de de de ea de de de ea de de de ea de de de ................  
c9080015 de df de de de df de de da de de de da de de de ................  
c9080025 df de de de de de de de de de de de de de de de ................

 

begin:memcpy,len:0x4  
0 1 2 3 4 5 6 7 8 9 A B C D E F 0123456789ABCDEF  
c9080105 48 09 00 00

 

begin:memcpy,len:0x948  
0 1 2 3 4 5 6 7 8 9 A B C D E F 0123456789ABCDEF  
c9080109 82 35 d5 de c8 df de de be 35 d5 de c8 dd de de .5.......5......  
c9080119 ba 35 d5 de c8 da de de b6 35 d5 de c8 db de de .5.......5......  
c9080129 b2 35 d5 de c8 d9 de de ae 35 d5 de c8 d6 de de .5.......5......

 

begin:memcpy,len:0x4  
0 1 2 3 4 5 6 7 8 9 A B C D E F 0123456789ABCDEF  
c9080a51 58 4a 00 00

 

begin:memcpy,len:0x4a58  
0 1 2 3 4 5 6 7 8 9 A B C D E F 0123456789ABCDEF  
c9080a55 a6 04 d5 de c9 de de de a2 04 d5 de c9 de de de ................  
c9080a65 5e 04 d5 de c9 de de de 4e 04 d5 de c9 de de de ^.......N.......  
c9080a75 4a 04 d5 de c9 de de de 42 04 d5 de c9 de de de J.......B.......

 

begin:memcpy,len:0x4  
0 1 2 3 4 5 6 7 8 9 A B C D E F 0123456789ABCDEF  
c90854ad d8 00 00 00

 

begin:memcpy,len:0xd8  
0 1 2 3 4 5 6 7 8 9 A B C D E F 0123456789ABCDEF  
c90854b1 dd de de de 8e 35 d5 de dc de de de 96 d7 de de .....5..........  
c90854c1 c9 de de de 22 1a de de ca de de de cf de de de ...."...........  
c90854d1 cf de de de 7a a4 de de cc de de de 86 94 de de ....z...........

 

begin:memcpy,len:0xbbff4  
0 1 2 3 4 5 6 7 8 9 A B C D E F 0123456789ABCDEF  
c9085589 7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00 .ELF............  
c9085599 03 00 28 00 01 00 00 00 00 00 00 00 34 00 00 00 ..(.........4...  
c90855a9 bc fa 0b 00 00 00 00 05 34 00 20 00 08 00 28 00 ........4. ...(.  
c90855b9 19 00 18 00 11 a6 dd 35 da cf 22 1a 71 b7 8b 08 .......5..".q...

 

begin:memcpy,len:0x38fc  
0 1 2 3 4 5 6 7 8 9 A B C D E F 0123456789ABCDEF  
c9141589 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 ................  
c9141599 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 ................  
c91415a9 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 ................

 

begin:memcpy,len:0xb4  
0 1 2 3 4 5 6 7 8 9 A B C D E F 0123456789ABCDEF  
e61b4340 28 97 29 c9 15 63 00 00 00 00 00 00 00 00 00 00 (.)..c..........  
e61b4350 02 00 00 00 6c 69 62 73 74 6c 5f 63 6f 6d 70 69 ....libstl_compi  
e61b4360 6c 65 72 2e 73 6f 00 00 00 00 00 00 00 00 00 00 ler.so..........  
e61b4370 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 ................

 

看到这个 ELF 了吗：  
begin:memcpy,len:0xbbff4  
0 1 2 3 4 5 6 7 8 9 A B C D E F 0123456789ABCDEF  
c9085589 7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00 .ELF............

 

关闭这个脚本，然后把上面的 len 值填到注释中的脚本，然后去掉注释再跑一次。因为每个得到的 len 长度不一样，需要根据自己的修改。  
然后就各种痛苦开始了 ^_^

  

[[CTF 入门培训] 顶尖高校博士及硕士团队亲授《30 小时教你玩转 CTF》，视频 + 靶场 + 题目！助力进入 CTF 世界](http://www.kanxue.com/book-brief-170.htm#h3a6WRhDT9Q_3D)

最后于 2020-8-6 11:24 被 fxyang 编辑 ，原因：
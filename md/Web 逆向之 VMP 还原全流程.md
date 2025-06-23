> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2040789-1-1.html)

> 某歌邮箱注册参数 f.req 还原过程。trace 不说了，试试自实现反编译器还原 VMP 吧！注册参数如下：返回位置如下：虚拟机解释器如下：开始分析 VMP 结构：一、V ...

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)lichuntian00 某歌邮箱注册参数 f.req 还原过程。  
trace 不说了，试试自实现反编译器还原 VMP 吧！  
注册参数如下：  
![](https://attach.52pojie.cn/forum/202506/22/180416awbkplvpnn4wlzv4.png)

**image.png** _(342.1 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU1NHw3ZTMyYmY2YXwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:04 上传

  
返回位置如下：  
![](https://attach.52pojie.cn/forum/202506/22/180619q8ccpcddhp7sy9d7.png)

**image.png** _(362.5 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU1NXw5ODI5OWUzM3wxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:06 上传

  
[虚拟机](https://www.52pojie.cn/thread-661779-1-1.html)解释器如下：  
![](https://attach.52pojie.cn/forum/202506/22/213945nz9et93cmp22kde2.png)

**image.png** _(135.17 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDYxMHwxYWVhYjcyY3wxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

虚拟机引擎

2025-6-22 21:39 上传

  
开始分析 VMP 结构：  
**一、VMP 指令集**  
![](https://attach.52pojie.cn/forum/202506/22/180911xoztkdgbmlolzdmj.png)

**image.png** _(18.18 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU1N3w5YjBjNzEzMXwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:09 上传

  
请求首页，302 跳转下发 js-challenge 页面，有加密配置参数，里面包含指令集字符串（图片所示），很好拿到，后续调用 atob 解密，并 &255 保留低 8 位，得到字节数组形式的指令集  
**二、虚拟机上下文分析**  
![](https://attach.52pojie.cn/forum/202506/22/185500hlz7pse4shoph46p.png)

**image.png** _(67.38 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU4Nnw4MWRhNGM4N3wxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

虚拟机上下文

2025-6-22 18:55 上传

  
这是分析 vmp 之后提取出来的虚拟机上下文，Z 是虚拟寄存器集合，一共 512 个  
其中关键的几个寄存器有，  
36 号 PC 寄存器，  
336 号内层虚拟机 PC 寄存器，  
184 号轮密钥种子寄存器  
280  115 509  234 147  44 136 511 210 471 这些寄存器存放生成加密参数所需上下文  
关键虚拟机函数有：  
![](https://attach.52pojie.cn/forum/202506/22/190948aqnz2ll9o802nlwn.png)

**image.png** _(28.56 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU5MnxiMmE4Mzc3MXwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

操作寄存器函数

2025-6-22 19:09 上传

  
N 函数，写入寄存器函数  
![](https://attach.52pojie.cn/forum/202506/22/183450qqh7kic8niiciq8o.png)

**image.png** _(16.6 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU3MnxmNzNhYjkzM3wxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:34 上传

  
y 函数 ，读取寄存器值  
![](https://attach.52pojie.cn/forum/202506/22/184516a9puki7wep7lddl6.png)

**image.png** _(82.01 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU3OXw3ODZkOTkzY3wxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:45 上传

  
C 函数，从虚拟机字节码中读取指令并解码复杂指令集为不同的指令类型，分发给 B8 解密指令  
![](https://attach.52pojie.cn/forum/202506/22/182408tr4xopg3lxra9oo3.png)

**image.png** _(114.78 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU2NXxkMGYyNmM4YnwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:24 上传

  
B8 函数，读取指令并根据参数判断是否密钥变换表，使用当前字节块的轮密钥对指令进行解密  
![](https://attach.52pojie.cn/forum/202506/22/182813b7bjpj7vr5xnjxtu.png)

**image.png** _(37.3 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU2NnwyMmVjMTMxNXwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:28 上传

  
hJ 函数，生成 8 字节解密表，用于取指之后，对指令进行异或解密  
**三、虚拟机执行逻辑分析**  
1. 先从 PC 寄存器取字节码索引，然后同步到内层虚拟机的 PC 寄存器，然后调用 C 函数解码  
![](https://attach.52pojie.cn/forum/202506/22/183006cim066m3m6p6622z.png)

**image.png** _(93.75 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU2N3xiNjI2Y2NlYnwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:30 上传

  
2.C 函数将指令解码为长短指令等复杂指令集，函数体一共有 5 个分支，分别对应  
1. [backcolor=var(--ds-md-inline-code-color,#ececec)][size=0.875em](w | 48) == w 分支 往寄存器中存值  
[backcolor=var(--ds-md-inline-code-color,#ececec)][size=0.875em]2.[backcolor=var(--ds-md-inline-code-color,#ececec)][size=0.875em](w - 5 | 5) >= w && (w + 4 & 43) < w 分支  生成整数值  
[backcolor=var(--ds-md-inline-code-color,#ececec)][size=0.875em]3.[backcolor=var(--ds-md-inline-code-color,#ececec)][size=0.875em](w >> 1 & 12) < 12 && w + 6 >> 4 >= 3 分支  处理变长操作码  先读取 1 字节，判断高位是否为 1，扩展为长指令  
[backcolor=var(--ds-md-inline-code-color,#ececec)][size=0.875em]4.[backcolor=var(--ds-md-inline-code-color,#ececec)][size=0.875em]w + 5 >> 3 == 1 分支 将寄存器的值变为闭包变量  
![](https://attach.52pojie.cn/forum/202506/22/210607sfg2s7fgmllljlz6.png)

**image.png** _(48.71 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDYwM3w3YjAzYTI0NHwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

第 5 分支

2025-6-22 21:06 上传

[backcolor=var(--ds-md-inline-code-color,#ececec)][size=0.875em]  
[backcolor=var(--ds-md-inline-code-color,#ececec)][size=0.875em]5.[backcolor=var(--ds-md-inline-code-color,#ececec)][size=0.875em](w + 7 ^ 27) < w && (w - 6 ^ 9) >= w 分支  操作码读取  读取 8 位并开启密钥变换，或者读取低 7 位，左移 2 位组成 9 位操作码  
3.C 函数解码完成之后，下发给 B8 函数，按照 C 的解码规则来读取指令，详情如下：  
![](https://attach.52pojie.cn/forum/202506/22/211442yyii3c5i7oioz55g.png)

**image.png** _(119.81 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDYwNXw3NTdkMWMxMnwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

B8

2025-6-22 21:14 上传

  
首先从 36 PC 寄存器取出指定位数，同时判断是否超出索引  
判断现在解密的位数属于第几组解密块，如果是新的解密块，需要调用 hj 更新轮密钥变换表  
对取到的字节和当前轮的轮密钥进行异或解密  
计算已经解密的字节，并向前推进对应位数，对齐到字节最低位，提取结果，并更新 PC 寄存器值  
B8 返回操作码，并根据高位判断是否为长指令，如果是长指令需要扩展为对应的操作码  
![](https://attach.52pojie.cn/forum/202506/22/214831ix88kkyi7327q3za.png)

**image.png** _(51.69 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDYxMnxjMjU4MzQwNHwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

取 handler

2025-6-22 21:48 上传

  
4.C 返回对应的操作码之后，调用 y 函数从寄存器取操作码对应的 handler，  
不同的操作码对应不同的 handler，利用 handler 执行操作码，操作数是基于操作码计算得到的  
5. 每执行完一轮操作码之后，进行检测。然后跳出循环执行下一个取指解码操作  
完整流程是：取索引下标（利用 PC）————解码（C 函数）————取指（B8）————执行（跳转到对应 handler）————检测（js 文件被我处理过，移除了反调试和检测函数）  
**四、操作码分析**  
1. 生成数值型操作码  
![](https://attach.52pojie.cn/forum/202506/22/184000e6d14a964gg9lg2t.png)

**image.png** _(1.74 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU3NHxjN2NlYzcwM3wxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:40 上传

  
操作码 24  生成 4 字节数值  
![](https://attach.52pojie.cn/forum/202506/22/215930ch8zaazypaa27844.png)

**image.png** _(1.57 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDYxNXxhZTI2MDkyZHwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

操作码 66

2025-6-22 21:59 上传

  
操作码 66，生成 1 字节数值  
![](https://attach.52pojie.cn/forum/202506/22/220019b15n1ozedyz9tcsn.png)

**image.png** _(1.63 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDYxNnwxOGNkZmYyNnwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 22:00 上传

  
操作码 105 ，生成双字节数值  
不一一列举  
这些操作码的作用都是生成对应字节数值，并存储到寄存器  
2. 生成字符串型操作码  
![](https://attach.52pojie.cn/forum/202506/22/220250laxmfaghrf937zm0.png)

**image.png** _(75.77 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDYxN3wyMmY2NGQ4NHwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

操作码 304

2025-6-22 22:02 上传

  
操作码 304 生成字符串  （调试的时候忘了截图！！）  
可以看出来操作数是基于操作码计算得到的    
3. 函数调用型操作码  
![](https://attach.52pojie.cn/forum/202506/22/220549hwl1l2vw261dq4bn.png)

**image.png** _(84.5 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDYyN3xjOGNmMWJmZnwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

操作码 302

2025-6-22 22:05 上传

  
操作码 302  进行函数调用  
4. 属性访问型操作码  
![](https://attach.52pojie.cn/forum/202506/22/220805h9u1zzcu7qq9ea9p.png)

**image.png** _(16.57 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDYyOHwyZjNhM2FkM3wxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

操作码 495

2025-6-22 22:08 上传

  
操作码 495 访问浏览器对象下的属性  
另外，  
- ** 操作码 122**：生成 `eval` 函数调用。  
- ** 操作码 149、220、446**：处理数组操作（增大数组 `register_280`）。  
- ** 操作码 265、495**：对象属性赋值。  
- ** 操作码 319**：生成数组。  
- ** 操作码 477**：创建新对象  
**五、编写反编译器**  
分析完了，开始写反编译器，流程如下：  
1. 初始化节点  
![](https://attach.52pojie.cn/forum/202506/22/185343bmicf8iy6mcmg6zf.png)

**image.png** _(28.55 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU4NHxhMjgwY2I3ZXwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:53 上传

  
2. 创建空 AST  
![](https://attach.52pojie.cn/forum/202506/22/185426lxs7kyj1lzmxskjx.png)

**image.png** _(4.71 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU4NXwyYjg1NjBiZnwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:54 上传

  
3. 模拟虚拟机执行环境  
4. 加载 base64 解码后的指令  
![](https://attach.52pojie.cn/forum/202506/22/185538ovy93f3ih0tqyzxf.png)

**image.png** _(28.86 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU4N3wxYzZmMTUzOXwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:55 上传

  
5. 指令处理循环  读取操作码  处理不同的操作码    解码操作码语义  
![](https://attach.52pojie.cn/forum/202506/22/221129byb84e8ydpqbnt4t.png)

**image.png** _(77.73 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDYyOXw0MzM3NzhhYXwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

解码操作码语义

2025-6-22 22:11 上传

  
6. 类型推断生成 ast 生成不同类型节点  根据值类型生成对应 AST 节点  
![](https://attach.52pojie.cn/forum/202506/22/185711cu6abuqux8gumz5q.png)

**image.png** _(41.56 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU5MHwxODAwMWQ1NXwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:57 上传

  
7.ast 生成  根据操作类型创建对应的 AST 节点   将节点添加到程序体    
![](https://attach.52pojie.cn/forum/202506/22/185633qerzpo2lsgvsp2n7.png)

**image.png** _(23.3 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU4OXxjMDNiYmZhNXwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:56 上传

  
**六、输出虚拟化之前的 js 代码**  
![](https://attach.52pojie.cn/forum/202506/22/185256avedtdjyvhd0p3kh.png)

**image.png** _(123.73 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU4M3wyN2NkMWY5ZnwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:52 上传

  
一个变种白盒 AES，一个变异的 base64  
web 端协议参数生成  
![](https://attach.52pojie.cn/forum/202506/22/221714emv3yrh2ooyd3oli.png)

**image.png** _(325.81 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDYzMHw4ZjE5Mzc2NHwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

协议

2025-6-22 22:17 上传

  
app 端 webview 参数生成  
![](https://attach.52pojie.cn/forum/202506/22/221923el9zxmzd2dmu9r0m.png)

**image.png** _(198.22 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDYzMXxkOTVlMzRkM3wxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

app

2025-6-22 22:19 上传

  
[image.png](forum.php?mod=attachment&aid=Mjc4NDU1NnxjMzMxZWIwZXwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes) _(134.39 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU1NnxjMzMxZWIwZXwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:07 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202506/22/180719nmqszzcfe1ycmaem.png) [image.png](forum.php?mod=attachment&aid=Mjc4NDU1OHxiNzJkNGMxNXwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes) _(28.56 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU1OHxiNzJkNGMxNXwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:10 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202506/22/181037yknn1up8y07cffcc.png) [image.png](forum.php?mod=attachment&aid=Mjc4NDU1OXxmYWI0OWY4N3wxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes) _(134.39 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU1OXxmYWI0OWY4N3wxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:13 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202506/22/181337jh84wzz834hh5e3t.png) [image.png](forum.php?mod=attachment&aid=Mjc4NDU2M3w0MmRkM2I0YnwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes) _(103.44 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU2M3w0MmRkM2I0YnwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:20 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202506/22/182042kzxf0f0sfzigjqfh.png) [image.png](forum.php?mod=attachment&aid=Mjc4NDU3M3w5MDU3NzBiYXwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes) _(29.57 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU3M3w5MDU3NzBiYXwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:35 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202506/22/183526gplzrzp3bjbopywp.png) [image.png](forum.php?mod=attachment&aid=Mjc4NDU3NXxhODVjYjg4Y3wxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes) _(51.07 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU3NXxhODVjYjg4Y3wxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:41 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202506/22/184123uv8ynjs3oyryqsjn.png) [image.png](forum.php?mod=attachment&aid=Mjc4NDU3Nnw4MTc1ZjAxY3wxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes) _(81.76 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU3Nnw4MTc1ZjAxY3wxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:42 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202506/22/184203xv6akh77zvd1lrhp.png) [image.png](forum.php?mod=attachment&aid=Mjc4NDU3N3xjZGEyNzJmNHwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes) _(10.26 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU3N3xjZGEyNzJmNHwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:42 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202506/22/184234hp7u39qptxqzylf8.png) [image.png](forum.php?mod=attachment&aid=Mjc4NDU4MXw1ZTQxYWQ0ZHwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes) _(76.28 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU4MXw1ZTQxYWQ0ZHwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:51 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202506/22/185146vubaccfuybsy8bs6.png) [image.png](forum.php?mod=attachment&aid=Mjc4NDU4OHxlN2UxMzExMXwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes) _(76.28 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDU4OHxlN2UxMzExMXwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 18:55 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

![](https://attach.52pojie.cn/forum/202506/22/185557ecllmxma3klx3k1x.png) 

[image.png](forum.php?mod=attachment&aid=Mjc4NDYxM3w2NDE2YzRkMHwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes) _(30.11 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=Mjc4NDYxM3w2NDE2YzRkMHwxNzUwNjYwODM2fDIxMzQzMXwyMDQwNzg5&nothumb=yes)

2025-6-22 21:50 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

检测函数

![](https://attach.52pojie.cn/forum/202506/22/215027a0w55e86qq10llzl.png)![](https://avatar.52pojie.cn/data/avatar/001/39/82/29_avatar_middle.jpg)qqycra 这文章精彩，把整个过程都写出来了。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)YIUA 大佬，可以加好友带带我吗![](https://static.52pojie.cn/static/image/smiley/default/44.gif) ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) latecomer

> [YIUA 发表于 2025-6-23 08:38](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=53294813&ptid=2040789)  
> 大佬，可以加好友带带我吗

顶楼上，同求 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) xiaoweigege 反编译这块的逻辑有点一笔带过，希望博主可以具体讲讲怎么利用 ast 编写反编译的代码。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)FlyPan 大佬牛波一 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) mscsky 这么硬核的吗![](https://avatar.52pojie.cn/images/noavatar_middle.gif)外酥内嫩 大佬太强了，还原它肯定研究了很久吧 ![](https://avatar.52pojie.cn/data/avatar/000/00/00/01_avatar_middle.jpg) Hmily [@lichuntian00](https://www.52pojie.cn/home.php?mod=space&uid=2086389) 底部多了几张图是不是文章插丢了？
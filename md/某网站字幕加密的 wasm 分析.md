> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1461335-1-1.html)

> [md]@[TOC](某网站字幕加密的 wasm 分析)## js 层动态分析网站地址：aHR0cHM6Ly93d3cuaXEuY29tL3BsYXkvMmZhcWJkMTV1YWM=（需要【台】的 ip）首先......

 ![](https://avatar.52pojie.cn/data/avatar/000/55/71/95_avatar_middle.jpg) 漁滒

@[TOC](某网站字幕加密的wasm分析)

js 层动态分析
--------

网站地址：aHR0cHM6Ly93d3cuaXEuY29tL3BsYXkvMmZhcWJkMTV1YWM=（需要【台】的 ip）

首先打开网址，f12 抓包开起来，播放视频后在过滤器中搜索【.xml】

![](https://img-blog.csdnimg.cn/20210615181141124.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

可以看到 sub 标签里面的字幕内容被加密了，先给字幕下一个 xhr 断点

![](https://img-blog.csdnimg.cn/20210615182457420.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

这里可以看到当字幕文件加载完成后，会执行 t 函数，跟进去

![](https://img-blog.csdnimg.cn/2021061518261759.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)  
这里可以看到对字幕文件进行简单的解析，还没开始进行解密，当解析完成后会执行 changeSuccess 函数，继续跟下去

![](https://img-blog.csdnimg.cn/20210615220859443.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

这里继续往下，p 参数这一行有一个分之，查看 p 的值是 1，说走前面的代码。然后有一个 a._encrypt.show() 引起了注意，从这里继续跟进去

![](https://img-blog.csdnimg.cn/20210615221214929.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

这里跟着执行 setStyle 函数，继续跟下去

![](https://img-blog.csdnimg.cn/20210615221458713.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

前面进行了一大段的计算，和函数名相符，这里设置了一些字幕的样式，但是还没有涉及字幕本身的解密，最后的 setText 函数，看起来就和文本有关，有可能解密就在里面完成，那就继续跟进去看看

![](https://img-blog.csdnimg.cn/20210615221940149.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

断点断下后就可以看到字幕的密文了，d 参数就是需要解密的内容，复制去验证一下

![](https://img-blog.csdnimg.cn/20210615222235402.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

确实可以在 xml 文件中找到这一段密文，接着就是看看调用到这段密文的函数

![](https://img-blog.csdnimg.cn/20210615222453825.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

这里可以看到，调用了一个 s 函数后返回一个 f，然后这个 f 就用来设置字幕的宽度。既然这个 f 能用来计算字幕的宽度，说明了 s 函数内部已经解密的明文，那么才能计算宽度的，说明要继续跟进 s 函数里面

![](https://img-blog.csdnimg.cn/20210615223146858.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

这次跟进去后，发现并没有那么顺利，直接来到了 call 的函数，这是 js 层调用了 wasm 的一个函数的特征，如果需要继续分析的话，那么就得对 wasm 进行分析了。过滤器中搜索 wasm 文件下载下来。

wasm 初步处理
---------

根据这篇文件的介绍[【一种 Wasm 逆向静态分析方法】](https://www.52pojie.cn/thread-962068-1-1.html)，可以使用 wabt 工具【项目地址：[wabt](https://github.com/WebAssembly/wabt)】中的 wasm2c，将 wasm 的二进制文件转换为 c 文件

```
wasm2c wasm.wasm -o wasm.c

```

此时可以得到 wasm.c 和 wasm.h，然后将 wabt 项目内的 wasm-rt.h，wasm-rt-impl.c，wasm-rt-impl.h 三个文件放到同一个文件夹，通过 gcc 得到编译的 o 文件

```
gcc -c wasm.c -o wasm.o

```

此时的 o 文件就可以放进 IDA 进行反汇编分析了。如果觉得上面的步骤繁琐的话，可以使用逍遥一仙大佬封装的一键工具。可以直接将 wasm 得到 o 文件

[wasm 一键转 c](https://www.52pojie.cn/thread-1438499-1-1.html)

![](https://img-blog.csdnimg.cn/20210615225021177.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)  
将最后得到的 o 文件加载到 IDA 中

![](https://img-blog.csdnimg.cn/2021061522560153.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

加载完发现有两百多个函数，肯定不可能一个一个函数去分析。首先肯定是要先找到 js 层调用的是哪个函数，然后再重点去分析对应的函数

浏览器动态分析与 IDA 静态分析合作
-------------------

回到浏览器，call 函数的第一个参数就是指明需要调用 wasm 中的哪个函数

![](https://img-blog.csdnimg.cn/20210616220247934.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)  
可以看到函数名是 monalisa_get_line_number，但是在 wasm 中的函数名称窗口搜索，却搜索不到这个函数，因为这个只是 js 层的函数名，还要看是绑定在 wasm 中的哪个导出函数，在同一个 js 文件中搜索这个函数名

![](https://img-blog.csdnimg.cn/20210616220511954.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

这时就可以清楚的看到是 wasm 中的 v 函数

![](https://img-blog.csdnimg.cn/20210616220650571.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

有了函数名，现在还需要知道分别传入的参数是什么，这里就要回到前面的 s 函数了  
![](https://img-blog.csdnimg.cn/2021061622082327.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)  
第一个参数 n 是前面获取的一个上下文，搜索一下这个_ctx

![](https://img-blog.csdnimg.cn/20210616220948974.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

可以看到 ctx 是通过 moAlloc 函数获取的，相当于 monalisa_context_alloc，看到 alloc 可以确定这是一个申请内存的 c 库函数，所以不需要继续分析，简单可以理解成申请一段内存，返回的是这段内存起始的指针

第二个参数就是字幕的密文字符串，第三个参数就是字符串的长度，第四个参数就是一段固定的字符串，那么这时可以将 IDA 的变量名改一下

![](https://img-blog.csdnimg.cn/20210616221514961.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

字幕内容只有传入到 w 函数，其他都没有用到，那么就可以进入到 w 函数分析。回到浏览器，在 w 函数前面下一个断点断下来

![](https://img-blog.csdnimg.cn/20210616221810416.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)  
这里可以看到这个函数传入了 5 个 i32 类型的参数，但是 IDA 识别的不正确

![](https://img-blog.csdnimg.cn/20210616221905544.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

这时可以在函数名邮件，设置项目类型，修改成正确的，顺便将变量名修改一下

![](https://img-blog.csdnimg.cn/20210616222218810.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)  
![](https://img-blog.csdnimg.cn/20210616225137503.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)  
跟着就来到了 w2c_f24 函数  
![](https://img-blog.csdnimg.cn/20210616225701970.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)  
![](https://img-blog.csdnimg.cn/20210616225947742.jpg#pic_center)

这里可以看到返回值就是 w2c_J，从 js 中可以查看到，又是一个申请内存的函数，然后就是 w2c_f93 函数

![](https://img-blog.csdnimg.cn/20210616231421259.jpg#pic_center)  
第一个参数就是前面刚刚申请的内存，说明极大可能是用来放函数的返回值，并且将密文传了进去，说明这个函数肯定是一个关键点，那么来看看返回值是什么，首先单步进入函数

![](https://img-blog.csdnimg.cn/20210616232435614.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

第一个参数是一个指针，地址是 6066376，然后直接结束这个函数，然后去查看这个地址  
![](https://img-blog.csdnimg.cn/20210616232627836.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

可以看到一段 16 字节的内容，仔细观察一下其实可以发现，这个函数实际就是把密文进行了 base64 解码

![](https://img-blog.csdnimg.cn/20210616232937882.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)  
但是接下来静态分析并不能知道 v11 的值，所以继续在浏览器单步运行

![](https://img-blog.csdnimg.cn/20210616233218897.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

可以看到浏览器跟着运行的是 call $func71，继续跟进去。w2c_f24 是申请内存，前面已经分析过了，然后是 w2c_f23

![](https://img-blog.csdnimg.cn/20210616233942126.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

直接去到函数的结尾，函数的返回值就是第一个参数，实际上这个函数就是在做内存的复制

![](https://img-blog.csdnimg.cn/20210616234052885.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)  
复制后的内存就只有 w2c_f95 用到，那么就肯定要跟进去这个函数

![](https://img-blog.csdnimg.cn/20210616234216276.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)  
进到 w2c_f95 后发现并没有那么顺利，里面只有 w2c_f94 和 w2c_f149，里面的运算都比较复杂，这时我卡壳了。

c 层 aes 算法特征分析
--------------

解密的话常见的就三种情况，异或加位运算、对称加密以及非对称加密。这个时有个地方引起了我的注意

![](https://img-blog.csdnimg.cn/20210618205207154.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

128、192、256 这三个数字不就是 aes 算法的三种密钥长度，如果推论为加密用的是 aes 的话，那么里面的 a4a 与 v40 又恰好可以认为是密文的密钥长度和轮换次数。再去 xml 里面看一下密文，果然密文全部都是 16 字节的倍数，那么就已经可以确定是用的 aes 算法了

![](https://img-blog.csdnimg.cn/20210618213048921.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

知道了算法以后，还需要知道几个关键的值，分别是密钥及其长度，算法模式，偏移。密钥长度可以看到就是 a5，它分别和 128、192、256 进行对比，a5 是传进去来参数，是固定的 128，那么现在还剩下密钥、算法模式、偏移

接下来需要寻找密钥，根据文章【[常见加密算法](http://www.codinganswer.com/?yohytk=skmhl1&ektovs=8azel2)】中的讲述，aes 算法首先通过的是 initial_round，然后是 9 个 rounds，接着最后一个 round 少一个步骤

![](https://img-blog.csdnimg.cn/20210618212323556.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)  
进入 w2c_f149，其中的 a4 就是前面传进来的轮换次数 10

![](https://img-blog.csdnimg.cn/20210618212716810.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

可以看到每次循环自减 1，一共轮换 9 次，剩下的是第十次，这与算法完全吻合，但是 key 应该在哪里获取呢，可以看到 initial_round 步骤就用到了 key，这时自然就想到了 w2c_f149 前面的 w2c_f94

![](https://img-blog.csdnimg.cn/20210618213747100.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

![](https://img-blog.csdnimg.cn/20210618213823595.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

从高级加密标准 AES-FIPS197 中可以知道，initial_round 执行的就是轮密钥加 (AddRoundKey()) 变换，这是需要与密钥进行异或，那么就肯定要先把密钥取出来，自然想到一开始的循环就是将密钥取出来，然后进行异或，这里的 a1 就是前面传进来的值，先在浏览器看看密钥是什么，也就是 w2c_f94 的第一个参数

![](https://img-blog.csdnimg.cn/20210618215002685.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)

不知道哪里来的一段 16 字节，不管那么多，先试试能不能解密，现在还没有 iv，所以先用 ECB 的模式试试

```
def decrypt_zimu():
    enc_text = 'by5JecM7CKHaHHUd0C2wupB2A/X+CE2JRSbc8LK9p/U='
    crypto = AES.new(key=bytes([29,210,139,12,180,186,89,38,237,117,185,130,29,2,53,180]), mode=AES.MODE_ECB)
    print(crypto.decrypt(base64.b64decode(enc_text.encode())).decode(errors='ignore'))
    # 怎麼樣啊 醫t6x 

```

可以看到前 16 字节可以解密，但是后面的无法解密，说明模式错了，应该是 CBC，那么这时还需要一个正确的 iv，不然前 16 字节是错误的。CBC 的 iv 是在最后进行异或的，自然想到了最后的一段函数

![](https://img-blog.csdnimg.cn/20210618215941837.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)  
那么这里的 a2 就是偏移，和密钥一样去浏览器获取，可以知道 16 字节都是 0，加上 iv 重新解密

```
def decrypt_zimu():
    enc_text = 'by5JecM7CKHaHHUd0C2wupB2A/X+CE2JRSbc8LK9p/U='
    crypto = AES.new(key=bytes([29,210,139,12,180,186,89,38,237,117,185,130,29,2,53,180]), mode=AES.MODE_CBC, iv=bytes([0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]))
    print(unpad(crypto.decrypt(base64.b64decode(enc_text.encode())), AES.block_size).decode())
    # 怎麼樣啊 醫生

```

这时就完全解密出字幕了，接下来如果能知道 key 怎么来的，那么就大功告成了。

密钥的参数一直往外追，实际是 v 函数的第一个参数偏移 4，也就是一开始的 ctx 偏移 4，那么就是说在执行解密之前，执行了其他函数来设置了密钥，因为不可能申请内存里面就有密钥了。这时可以将除了内存处理之外的所有导出函数都下一个断点，在这个 wasm 中就是所有 monalisa 开头的导出函数，然后刷新，会在_monalisa_set_license 的地方断下

![](https://img-blog.csdnimg.cn/20210618222109328.jpg#pic_center)

这个函数的第一个参数也传入了 ctx，这里的参数可以在 dash 接口里面找到

![](https://img-blog.csdnimg.cn/20210618222255292.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)  
继续让这个函数运行，接着就在前面的函数断下了，那就充分说明这个就是获取密钥的函数，接下来也按照前面的方法，找函数，一步一步分析，那么理论上就可以获取到密钥了，但是实际并没有那么简单，中间分析的过程就忽略了，因为和上面是大同小异。当我分析到 w2c_f129 的时候，这个函数结束，密钥生生成了，但是这个函数超长，仅仅定义变量就有 1000 多个，这明显加了混淆了。

既然不能直接分析出算法，那么能不能用魔法来打败魔法呢？nodejs 可以加载 wasm 运行，如果可以调用 nodejs 来得到密钥，那不就可以省下很多功夫了。

wasm 调用代码扣取与异步加载处理
------------------

一般网页加载 wasm 的话，都有一个对应名称的 js，把与 wasm 同名的 js 下载下来，并且与 wasm 放在同一文件夹内

![](https://img-blog.csdnimg.cn/20210618223406890.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pqcTU5Mjc2NzgwOQ==,size_16,color_FFFFFF,t_70#pic_center)  
下载格式化后发现，这个 Module 被放到一个自执行函数的里面，那么外部就无法调用，那么就需要将这个自执行函数的代码放到外面，让全局中可以找到 Module 这个变量，注释头尾部分内容，就可以获取到 Module

```
// var Monalisa = (function() {
//     var _scriptDir = typeof document !== 'undefined' && document.currentScript ? document.currentScript.src : undefined;
//     if (typeof __filename !== 'undefined')
//         _scriptDir = _scriptDir || __filename;
//     return (function(Monalisa) {
//         Monalisa = Monalisa || {};
        Monalisa = {};

        var Module = typeof Monalisa !== "undefined" ? Monalisa : {};
        var readyPromiseResolve, readyPromiseReject;
        Module["ready"] = new Promise(function(resolve, reject) {
            readyPromiseResolve = resolve;
            readyPromiseReject = reject
        }
        );
        var moduleOverrides = {};
        var key;
        for (key in Module) {
            if (Module.hasOwnProperty(key)) {
                moduleOverrides[key] = Module[key]
            }
        }
        /*
        中间省略几千行     
        */
        Module["run"] = run;
        if (Module["preInit"]) {
            if (typeof Module["preInit"] == "function")
                Module["preInit"] = [Module["preInit"]];
            while (Module["preInit"].length > 0) {
                Module["preInit"].pop()()
            }
        }
        noExitRuntime = true;
        run();

        // return Monalisa.ready
    // }
    // );
// }
// )();
// if (typeof exports === 'object' && typeof module === 'object')
//     module.exports = Monalisa;
// else if (typeof define === 'function' && define['amd'])
//     define([], function() {
//         return Monalisa;
//     });
// else if (typeof exports === 'object')
//     exports["Monalisa"] = Monalisa;
console.log(Module);

```

接下来就是尝试调用_monalisa_set_license 方法来获取密钥了，安装 js 的方法，首先获取一个 ctx，前面有说过，然后是调用_monalisa_set_license 方法

```
console.log(Module);
function decrypt() {
    var ctx = Module["cwrap"]("monalisa_context_alloc", "number", [])();
    var License = "AA4ACgMAAAAAAAAAAAQCDwACATADEAAnAgAgeyWysVa0GpbmCNvd+S1tsL6yp/j2tbA14sqW1ppgepYCAAAAAxEANwEAMDCtrqLHyZQ7p8RX3ih4NIqLWR1zCfu3mMFlxC2kiPgHmxZY7I/KYq4pMkH3rZQsqgEAAgD/EgAkAQAAIGSSUL7C0qWJp/LIkKoS12QYws1e0z/CewNJaaqktC3z";
    Module["cwrap"]("monalisa_set_license", "number", ["number", "string", "number", "string"])(ctx, License, License.length, "0");

    console.log(new Buffer.from(Module.HEAPU8.slice(ctx+4, ctx+4+16)).toString('hex'))
}
decrypt();

```

但是出现报错了

```
                    throw ex
                    ^

TypeError: Cannot read property 'D' of undefined

```

说 js 中的 D 函数还没有定义，实际这是一个异步加载的 wasm，当我们运行到解密函数的时候，实际上 wasm 还没有加载完。对于异步加载的 wasm，有两个重要的参数 runtimeInitialized 和 runtimeExited，一个代表 wasm 的加载时机，完成加载则会变成 true；runtimeExited 是 wasm 的卸载时机，完成卸载则会变成 true。

既然后异步加载的，那么就可以设置一个定时器来监控 runtimeInitialized 的值，当期变为 true 时，再执行解密函数

```
console.log(Module);
function decrypt() {
    var ctx = Module["cwrap"]("monalisa_context_alloc", "number", [])();
    var License = "AA4ACgMAAAAAAAAAAAQCDwACATADEAAnAgAgeyWysVa0GpbmCNvd+S1tsL6yp/j2tbA14sqW1ppgepYCAAAAAxEANwEAMDCtrqLHyZQ7p8RX3ih4NIqLWR1zCfu3mMFlxC2kiPgHmxZY7I/KYq4pMkH3rZQsqgEAAgD/EgAkAQAAIGSSUL7C0qWJp/LIkKoS12QYws1e0z/CewNJaaqktC3z";
    Module["cwrap"]("monalisa_set_license", "number", ["number", "string", "number", "string"])(ctx, License, License.length, "0");

    console.log(new Buffer.from(Module.HEAPU8.slice(ctx+4, ctx+4+16)).toString('hex'))
}

var timer = setInterval(c, 1);
function c() {
    if (runtimeInitialized){
        clearInterval(timer);
        decrypt()
    }
}

```

这时就可以正确获取到密钥，这时只要将 License 修改为 process.argv[2]，就可以在命令行调用了

```
function decrypt() {
    var ctx = Module["cwrap"]("monalisa_context_alloc", "number", [])();
    var License = process.argv[2];
    Module["cwrap"]("monalisa_set_license", "number", ["number", "string", "number", "string"])(ctx, License, License.length, "0");

    console.log(new Buffer.from(Module.HEAPU8.slice(ctx+4, ctx+4+16)).toString('hex'))
}

var timer = setInterval(c, 1);
function c() {
    if (runtimeInitialized){
        clearInterval(timer);
        decrypt()
    }
}

```

```
def decrypt_zimu():
    License = "AA4ACgMAAAAAAAAAAAQCDwACATADEAAnAgAgeyWysVa0GpbmCNvd+S1tsL6yp/j2tbA14sqW1ppgepYCAAAAAxEANwEAMDCtrqLHyZQ7p8RX3ih4NIqLWR1zCfu3mMFlxC2kiPgHmxZY7I/KYq4pMkH3rZQsqgEAAgD/EgAkAQAAIGSSUL7C0qWJp/LIkKoS12QYws1e0z/CewNJaaqktC3z";
    nodejs = os.popen('node libmonalisa-v3.0.6-browser '+License)
    key = nodejs.read().replace('\n', '')
    nodejs.close()
    enc_text = 'by5JecM7CKHaHHUd0C2wupB2A/X+CE2JRSbc8LK9p/U='
    crypto = AES.new(key=bytes.fromhex(key), mode=AES.MODE_CBC, iv=bytes([0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]))
    print(unpad(crypto.decrypt(base64.b64decode(enc_text.encode())), AES.block_size).decode())
    # 怎麼樣啊 醫生

```

完美，正确解密出结果，同一个字幕文件，所使用的所有 key 都是一样的，也就是说调用一次获取密钥，就可以解密出一整个字幕文件了

完结，散花

参考文献
----

1.XXX 视频 cKey9.1 的生成分析和实现：[https://www.52pojie.cn/thread-948353-1-1.html](https://www.52pojie.cn/thread-948353-1-1.html)  
2. 一种 Wasm 逆向静态分析方法：[https://www.52pojie.cn/thread-962068-1-1.html](https://www.52pojie.cn/thread-962068-1-1.html)  
3.wasm 一键转 c：[https://www.52pojie.cn/thread-1438499-1-1.html](https://www.52pojie.cn/thread-1438499-1-1.html)  
4. 高级加密标准 AES-FIPS197：[https://wenku.baidu.com/view/2ce7a11b10a6f524ccbf8514.html](https://wenku.baidu.com/view/2ce7a11b10a6f524ccbf8514.html)  
5. 常见加密算法：[http://www.codinganswer.com/?yohytk=skmhl1](http://www.codinganswer.com/?yohytk=skmhl1)![](https://avatar.52pojie.cn/data/avatar/000/69/36/16_avatar_middle.jpg)ofo 这长好的教程，wasm 相关的书买了好几本了，就是看不进去![](https://avatar.52pojie.cn/data/avatar/000/83/54/29_avatar_middle.jpg)侃遍天下无二人

> [侃遍天下无二人 发表于 2021-6-19 00:13](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=38965568&ptid=1461335)  
> 这篇应该上精华，想不到爱奇艺 tw 也开始上 wasm 了

果然上了 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) songxp03 wasm 的视频教程买了好多，以为这种编译好的字节码破解不了呢![](https://avatar.52pojie.cn/data/avatar/000/83/54/29_avatar_middle.jpg)侃遍天下无二人 这篇应该上精华，想不到爱奇艺 tw 也开始上 wasm 了 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) xuepojie 很不错的分享，谢谢楼主！![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)gjwq258 得好好学习一下的说，谢谢 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) nanaqilin 太复杂了，看不懂啊，不过还是谢谢楼主无私的分享![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)无言 Y 学习一下 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) orb001 牛掰格拉斯
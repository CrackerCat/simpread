> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-276600.htm)

> [原创]GoJni 协议加解密分析

[原创]GoJni 协议加解密分析

16 小时前 2513

### [原创]GoJni 协议加解密分析

 [![](http://passport.kanxue.com/upload/avatar/604/918604.png?1616807717)](user-home-918604.htm) [.KK](user-home-918604.htm) ![](https://bbs.kanxue.com/view/img/rank/7.png)  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 16 小时前  2513

GoJni 协议加解密分析
=============

前言
==

*   应用获取 ▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇▇
*   该案例更新应用不是覆盖安装而是更换包名重新安装了一个 App，我还捣鼓了几分钟发现更新完了还要我更新，后面发现是因为 Pixel 安装 App 默认是在抽屉里没发现这个问题点

前置知识
----

*   我的理解是逆向分析永远是站在开发角度去思考问题，所以这里需要先熟悉一下 Go 的一些底层结构

### GO 中的字符串

*   `string` 看起来是一个整体，但是本质上是一片连续的内存空间，我们也可以将它理解成一个由字符组成的数组，相比于切片仅仅少了一个 `Cap` 属性
    
    ```
    type string string
    
    ```
    
    ```
    // from: src\reflect\value.go
    type StringHeader struct {
        Data uintptr
        Len  int
    }
    // Data 字段是一个 uintptr 类型的指针，指向实际存储字符串数据的内存地址；Len 字段表示字符串的长度，即其中包含的字符数（而不是字节数）。
     
    需要注意的是，由于 Go 语言中字符串是不可变的，因此字符串的底层结构是只读的。在对字符串进行修改时，会创建一个新的字符串对象来保存修改后的结果。
     
    // 切片
    type SliceHeader struct {
        Data uintptr
        Len  int
        Cap  int
    }
    
    ```
    
*   字符串：底层结构是一个包含指向底层数据的指针和长度信息的结构体，定义如下：
    
    *   `Data` 字段是一个 `uintptr` 类型的指针，指向实际存储字符串数据的内存地址
    *   `Len` 字段表示字符串的长度，即其中包含的字符数（而不是字节数）
    *   需要注意的是，由于 Go 语言中字符串是不可变的，因此字符串的底层结构是只读的
    *   在对字符串进行修改时，会创建一个新的字符串对象来保存修改后的结果
*   切片：带有容量信息（Capacity）的字符串底层结构是一个包含指向底层数据的指针、长度信息和容量信息的结构体，定义如下：
    *   `Data` 字段和 `Len` 字段的含义与普通字符串相同，分别表示底层数据的地址和字符串的长度
    *   不同之处在于，`Cap` 字段表示字符串底层数据的容量，即该字符串的底层数据所占用的内存空间大小
    *   需要注意的是，虽然字符串底层数据的容量可以手动设置或者调整，但在实际使用时一般不会直接操作字符串的底层结构，而是通过字符串提供的方法进行操作，例如使用 `append()` 函数来追加字符串内容，由编译器自动处理扩容等操作
*   `Runtime\string.go` 里存在一些常见的字符串处理：`slicebytetostring` `concatstrings` 等等函数会在反编译代码中出现的
*   这里记住 Go 的字符串是，字符串 + 字符串长度 或 字符串 + 字符串长度 + 容量 组成的即可
    
    ![](https://bbs.kanxue.com/upload/attach/202303/918604_VADT7PPUMBKSW6S.png)
    

### Go 返回值

*   在 Go 中是支持多返回值的，且调用约定基本都是 `usercall`，那返回值应该在栈上
*   举个栗子：`sp+4` `sp+8` `sp+12` 是传入参数，返回值就应该在参数后 `sp+16` `sp+20` `sp+24 ...`
*   ARM 中常用的栈是 `sp < bp` 的，也就是递减的（向下增长、连续的内存区域，通常被称为 “向下堆栈” 或“逆序堆栈”），`临时变量 < sp`，`可用堆栈 > sp`
    
    补充：SP 与 BP 都是栈指针，用于管理栈的位置和操作。它们的使用方法和作用有所不同，但都是必不可少的
    
    1.  SP 指向栈顶，BP 指向栈底或当前函数的栈帧底部。在函数调用过程中，SP 和 BP 都会发生变化，但是它们的相对位置保持不变，因此 BP 可以用来访问当前函数的局部变量和参数，而 SP 则用于处理栈的压入和弹出操作
    2.  在 ARM 架构中，BP 并不是必须的，因为可以使用 SP 来访问局部变量和参数。但是在某些情况下，BP 可以提高代码的可读性和可维护性，特别是在调试时。此外，BP 还可以用于保存上一个函数的栈帧指针，以便在返回时恢复上一个函数的状态
        
        ![](https://bbs.kanxue.com/upload/attach/202303/918604_6DHJEZW4JH2QS84.png)
        

抓包
--

### 抓包数据

*   大概分析了下请求都有加密且格式固定，如下图 data
    
    ![](https://bbs.kanxue.com/upload/attach/202303/918604_FWSCNGQSYSV8QX4.png)
    

自吐 hook
-------

```
// 请求数据
data    String    4b797a6b79724b6a784f4e5063555949c7102e99bdf73b4b11732ca323bc6ec9d1bd741a879d5675286139db959b8b5f7e928e412007f5270161a89213628b4f2d541a83c9a3a504d18fc62380dc8bdab4a756ecba21a00377a3d21779ce5c5def79b933e1238237a567405ede8d609a051a9960b668bb7beca1a6910d36014bc6c99dc6b063bbc6afad592ffe6cea1a1a667723e7d97f54e43369e5fdedf38fd9649715200d21f756cf3ae294405f057dd16d0be6c9b85e19a11f64a02d8c62a0ceedc8875703a7c1034b5872822143c4b1e0de826bd02576f88dc0f94586b70225363e99ae9dd86991b66c23bb6ea3750e1d9e403634babca10d4853446a852fcb3a93a7e0119b96e3d4d157395d4f1b1a033d4ab62f52fcdac519fe10f9d4ece5b0ba4030adea08b0eb6b9dd2f63c8e2332ea04e1a2b23432b5137219a0f780f5301955dc48418a230b59636f1281a954986ca3cf33fda43b07439558e41cbb6e34592db8d2bba90ddb9a923f93c7b9b6f5810fcd036cc6b2cf5aff30b6b12c273a1a07fbdb7dcb36766b03fc962fc43556a19a360c2a9de8f8728d396816039959abfb36e01b01ce7661ad07d5b2400a12cc0a43d5538c82b92351ef9ebf1cba1dafec962cfcbb0ba5d2dda2923b2438580dcb1b4e1bc0589f877ec7e1ccbec214f2849140c1103a4deaea4432f5ea5cfa03df281dab9f2017dac99a5f7dce1e7560ca130d5d31e8a8d30cb053d4cf67d959ad32337994ff49646a34994b5d05bc9613eb7f619988714ab820ad0c1d668af5d734d5ee29658a9b14086c5bdee3cd9a55b7522e6b18cfdeb420e550bf8344a3e3c19aa79dbb3f4918c6b5fb082ea5b0b32752defe446fd418b372cf672b185a7d662ac0e00bb75b9c6b62b854bf0eb4c1599e4ba2b4f4c0c81a94c9cda58b8b4d4f6a5c
_ver    String    v1
sign    String    ceaac2487fa17e7019b05ab4cf41ebd0
timestamp    String    1678025314

```

```
java.lang.Exception
    at java.security.MessageDigest.update(Native Method)
    at com.szcx.lib.encrypt.e.a.e(SourceFile:2)
    at com.szcx.lib.encrypt.c.k(SourceFile:10)
    at com.tencent.mm.network.d.o2(SourceFile:1)
    at com.tencent.mm.network.d.q2(SourceFile:6)
    at com.tencent.mm.network.d.a4(SourceFile:3)
    at com.tencent.mm.ui.fragment.main.MineFragment.r3(SourceFile:1)
    at com.tencent.mm.ui.fragment.main.MineFragment.s4(SourceFile:1)
    at com.scwang.smartrefresh.layout.SmartRefreshLayout$l.onAnimationEnd(SourceFile:4)
    at android.animation.Animator$AnimatorListener.onAnimationEnd(Animator.java:554)
    at android.animation.ValueAnimator.endAnimation(ValueAnimator.java:1242)
    at android.animation.ValueAnimator.doAnimationFrame(ValueAnimator.java:1484)
    at android.animation.AnimationHandler.doAnimationFrame(AnimationHandler.java:146)
    at android.animation.AnimationHandler.access$100(AnimationHandler.java:37)
    at android.animation.AnimationHandler$1.doFrame(AnimationHandler.java:54)
    at android.view.Choreographer$CallbackRecord.run(Choreographer.java:964)
    at android.view.Choreographer.doCallbacks(Choreographer.java:790)
    at android.view.Choreographer.doFrame(Choreographer.java:721)
    at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:951)
    at android.os.Handler.handleCallback(Handler.java:883)
    at android.os.Handler.dispatchMessage(Handler.java:100)
    at android.os.Looper.loop(Looper.java:214)
    at android.app.ActivityThread.main(ActivityThread.java:7356)
    at java.lang.reflect.Method.invoke(Native Method)
    at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:492)
    at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:930)
 
[*]    SHA-256 | update | Utf8: _ver=v1&data=4b797a6b79724b6a784f4e5063555949c7102e99bdf73b4b11732ca323bc6ec9d1bd741a879d5675286139db959b8b5f7e928e412007f5270161a89213628b4f2d541a83c9a3a504d18fc62380dc8bdab4a756ecba21a00377a3d21779ce5c5def79b933e1238237a567405ede8d609a051a9960b668bb7beca1a6910d36014bc6c99dc6b063bbc6afad592ffe6cea1a1a667723e7d97f54e43369e5fdedf38fd9649715200d21f756cf3ae294405f057dd16d0be6c9b85e19a11f64a02d8c62a0ceedc8875703a7c1034b5872822143c4b1e0de826bd02576f88dc0f94586b70225363e99ae9dd86991b66c23bb6ea3750e1d9e403634babca10d4853446a852fcb3a93a7e0119b96e3d4d157395d4f1b1a033d4ab62f52fcdac519fe10f9d4ece5b0ba4030adea08b0eb6b9dd2f63c8e2332ea04e1a2b23432b5137219a0f780f5301955dc48418a230b59636f1281a954986ca3cf33fda43b07439558e41cbb6e34592db8d2bba90ddb9a923f93c7b9b6f5810fcd036cc6b2cf5aff30b6b12c273a1a07fbdb7dcb36766b03fc962fc43556a19a360c2a9de8f8728d396816039959abfb36e01b01ce7661ad07d5b2400a12cc0a43d5538c82b92351ef9ebf1cba1dafec962cfcbb0ba5d2dda2923b2438580dcb1b4e1bc0589f877ec7e1ccbec214f2849140c1103a4deaea4432f5ea5cfa03df281dab9f2017dac99a5f7dce1e7560ca130d5d31e8a8d30cb053d4cf67d959ad32337994ff49646a34994b5d05bc9613eb7f619988714ab820ad0c1d668af5d734d5ee29658a9b14086c5bdee3cd9a55b7522e6b18cfdeb420e550bf8344a3e3c19aa79dbb3f4918c6b5fb082ea5b0b32752defe446fd418b372cf672b185a7d662ac0e00bb75b9c6b62b854bf0eb4c1599e4ba2b4f4c0c81a94c9cda58b8b4d4f6a5c×tamp=167802531481d7beac44a86f4337f534ec93328370
[*]    SHA-256 | digest | Hex: 596d1c38df70c52a5c4834a970f78774e0213c95cb3b852ac96cbc1dacf08cf4
 
================================================== ⚡ ==================================================
 
java.lang.Exception
    at java.security.MessageDigest.update(Native Method)
    at java.security.MessageDigest.digest(MessageDigest.java:447)
    at com.szcx.lib.encrypt.e.c.b(SourceFile:3)
    at com.szcx.lib.encrypt.c.j(SourceFile:3)
    at com.szcx.lib.encrypt.c.k(SourceFile:10)
    at com.tencent.mm.network.d.o2(SourceFile:1)
    at com.tencent.mm.network.d.q2(SourceFile:6)
    at com.tencent.mm.network.d.a4(SourceFile:3)
    at com.tencent.mm.ui.fragment.main.MineFragment.r3(SourceFile:1)
    at com.tencent.mm.ui.fragment.main.MineFragment.s4(SourceFile:1)
    at com.scwang.smartrefresh.layout.SmartRefreshLayout$l.onAnimationEnd(SourceFile:4)
    at android.animation.Animator$AnimatorListener.onAnimationEnd(Animator.java:554)
    at android.animation.ValueAnimator.endAnimation(ValueAnimator.java:1242)
    at android.animation.ValueAnimator.doAnimationFrame(ValueAnimator.java:1484)
    at android.animation.AnimationHandler.doAnimationFrame(AnimationHandler.java:146)
    at android.animation.AnimationHandler.access$100(AnimationHandler.java:37)
    at android.animation.AnimationHandler$1.doFrame(AnimationHandler.java:54)
    at android.view.Choreographer$CallbackRecord.run(Choreographer.java:964)
    at android.view.Choreographer.doCallbacks(Choreographer.java:790)
    at android.view.Choreographer.doFrame(Choreographer.java:721)
    at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:951)
    at android.os.Handler.handleCallback(Handler.java:883)
    at android.os.Handler.dispatchMessage(Handler.java:100)
    at android.os.Looper.loop(Looper.java:214)
    at android.app.ActivityThread.main(ActivityThread.java:7356)
    at java.lang.reflect.Method.invoke(Native Method)
    at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:492)
    at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:930)
 
[*]    MD5 | update | Utf8: 596d1c38df70c52a5c4834a970f78774e0213c95cb3b852ac96cbc1dacf08cf4
[*]    MD5 | digest | Hex: ceaac2487fa17e7019b05ab4cf41ebd0
 
================================================== ⚡ ==================================================

```

sign 解析
-------

*   通过自吐 hook 可知 `sign = MD5( SHA256(requestData))`
*   所以这里核心就是请求参数 Data 解析
*   根据一些关键词进行快速定位，`k` 方法一看就很像
    
    ![](https://bbs.kanxue.com/upload/attach/202303/918604_6M5VSRNWW56MDKE.png)
    
*   跟了一下 `String e2 = e(encryptData);` 逻辑最终到了 native 层，hook 一下
    
    ![](https://bbs.kanxue.com/upload/attach/202303/918604_HJ8MEG55CKXTPZR.png)
    

encrypt hook
------------

*   拿了 token + 一堆设备信息进行加密，这里先不追 token 怎么来的（我猜是初始化 App 得到的！
    
    ```
    // 抓包数据
    data    String    627a50674a6d776d73587454705468521edbeb72a11ac72b90b74764a0580a0ef17d65423ba7bca562050f72a3c518aea1bf9c6cc3427cb1aff14212e40f2c30c44494b628355e0a5911c066595a2da3f265b76de6c4be7d6cae4277bba320f2cd730a40aa8558644176567f2b94c006513bf19ef1f7724566c843c8bf8dd6a9f06fad04362847472c91d1f63f8f1a11b48d9aa7a330535fd558ef2f87b15bbe233797659bcf01d4a48089c07cce73644da19f6a5f0fb54cf014a5212843aa03a28b3f5a616d2a34115d03b7dec62ea4f1b9da0b6dea710587c4d758e47fbd824fc38c5b6e49491e0aead6fd2134b4f6ca0ce7a5bb4c1fc8e77be276b0523f5ddee057a1007a6f6abbbba042f78a4afd6be2b41d10988e1d470c1cf003a2642a60112e127cfcb585a835989146da5bfae44adab85c01d8c3ea2f49c213aad8a12e9fda7f876c401c21af65bda3c5212147c1a71cb583988002ac631c13d2fed6f30bb2c48b11e34fadd1f7c827fb40d8d02065b564c2304db9ebae729db919ffd0080ab3bdca10a740b89d0da0f3885f7db85a4a13665258eff2b2be7b30895606a9820a8a2dd71a4cb5c3c594077176d49deaf08dc77b98b69729a87cd82f48e6656b179f69e1942fe68932eec1c7da4393b592cd41ffc68de9f1721a44c62ae2c645f3be198e64173d5151a329329cdc2ddabeadb82032e7be0ddaff534344e963984f80e245820c48e1fee944b8814b8d7f80095dd0a983270292c73aeed987e237405b86499098ca1fbc688dc986c17653be283ae69f9c371749034de089012c26497cffc9911b4e0365ceb489086f2b3c987c3073a334996d444686047f47651789c8c4e5d1f1864f54c384aad28b1d6856005c8ed4973e62fee5ec8966e462d18f74cb81407067ddf7b3c07a4cbcff164331a2c13c
    _ver    String    v1
    sign    String    dbda8ee96f5dcd2de4b85a66e5fb10ff
    timestamp    String    1678030550
    
    ```
    
*   经过多次调用相同入参，密文不同（猜测可能对称加密里不同的加密分组模式， `pwd` 可能就是初始化向量的 `IV` ，这里仅仅是一个猜测
    
    ```
    [DEBUG][03/05/2023, -1:35:50 PM][PID:28955][main][28955][showStacks]  java.lang.Exception
            at com.qq.lib.EncryptUtil.encrypt(Native Method)                                                    
            at com.szcx.lib.encrypt.c.f(SourceFile:1)                                                           
            at com.szcx.lib.encrypt.c.e(SourceFile:1)                                                           
            at com.szcx.lib.encrypt.c.k(SourceFile:2)                                                           
            at com.tencent.mm.network.d.o2(SourceFile:1)                                                        
            at com.tencent.mm.network.d.q2(SourceFile:6)                                                        
            at com.tencent.mm.network.d.a4(SourceFile:3)                                                        
            at com.tencent.mm.ui.fragment.main.MineFragment.r3(SourceFile:1)                                    
            at com.tencent.mm.ui.fragment.main.MineFragment.s4(SourceFile:1)                                    
            at com.scwang.smartrefresh.layout.SmartRefreshLayout$l.onAnimationEnd(SourceFile:4)                 
            at android.animation.Animator$AnimatorListener.onAnimationEnd(Animator.java:554)                    
            at android.animation.ValueAnimator.endAnimation(ValueAnimator.java:1242)                            
            at android.animation.ValueAnimator.doAnimationFrame(ValueAnimator.java:1484)                        
            at android.animation.AnimationHandler.doAnimationFrame(AnimationHandler.java:146)                   
            at android.animation.AnimationHandler.access$100(AnimationHandler.java:37)                          
            at android.animation.AnimationHandler$1.doFrame(AnimationHandler.java:54)                           
            at android.view.Choreographer$CallbackRecord.run(Choreographer.java:964)                            
            at android.view.Choreographer.doCallbacks(Choreographer.java:790)                                   
            at android.view.Choreographer.doFrame(Choreographer.java:721)                                       
            at android.view.Choreographer$FrameDisplayEventReceiver.run(Choreographer.java:951)                 
            at android.os.Handler.handleCallback(Handler.java:883)                                              
            at android.os.Handler.dispatchMessage(Handler.java:100)
            at android.os.Looper.loop(Looper.java:214)
            at android.app.ActivityThread.main(ActivityThread.java:7356)
            at java.lang.reflect.Method.invoke(Native Method)
            at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:492)
            at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:930)
     
    [>>>] com.qq.lib.EncryptUtil.encrypt
    [ + ] encrypt_arg[0]       :=>  {"system_build_id":"a1000","system_iid":"923868eb5f543afa55f1f33cfac37d35","app_status":"9A5C6BDC62AD1CFE45A6578F84E858F9CA1A4F76:2","system_version":"5.7.1","system_build_aff":"","bundle_id":"tv.iytqy.cvhaca","system_app_type":"local","new_player":"fx","system_oauth_id":"02274477773a9ac8a2cf3605400ece4b","system_oauth_type":"android","system_token":"023FAC3AFE2285DC98E50CA4C638E0845277DC8CB62CCCC088DA1E3CB679180C97F2688C0DBE197F03950D84900C2A6B050334216C4CC604021DCC624543AFDDEF8D19363E43F9C0B8EEC79BBF269E7DE77B0304FF1034FED885ACA4C74DAC768D26D9FADD"}
    [ + ] encrypt_arg[1]       :=>  BwcnBzRjN2U/MmZhYjRmND4xPjI+NWQwZWU0YmI2MWQ3YjAzKw8cEywsIS4BIg==
    [<<<] encrypt_result       :=>  627a50674a6d776d73587454705468521edbeb72a11ac72b90b74764a0580a0ef17d65423ba7bca562050f72a3c518aea1bf9c6cc3427cb1aff14212e40f2c30c44494b628355e0a5911c066595a2da3f265b76de6c4be7d6cae4277bba320f2cd730a40aa8558644176567f2b94c006513bf19ef1f7724566c843c8bf8dd6a9f06fad04362847472c91d1f63f8f1a11b48d9aa7a330535fd558ef2f87b15bbe233797659bcf01d4a48089c07cce73644da19f6a5f0fb54cf014a5212843aa03a28b3f5a616d2a34115d03b7dec62ea4f1b9da0b6dea710587c4d758e47fbd824fc38c5b6e49491e0aead6fd2134b4f6ca0ce7a5bb4c1fc8e77be276b0523f5ddee057a1007a6f6abbbba042f78a4afd6be2b41d10988e1d470c1cf003a2642a60112e127cfcb585a835989146da5bfae44adab85c01d8c3ea2f49c213aad8a12e9fda7f876c401c21af65bda3c5212147c1a71cb583988002ac631c13d2fed6f30bb2c48b11e34fadd1f7c827fb40d8d02065b564c2304db9ebae729db919ffd0080ab3bdca10a740b89d0da0f3885f7db85a4a13665258eff2b2be7b30895606a9820a8a2dd71a4cb5c3c594077176d49deaf08dc77b98b69729a87cd82f48e6656b179f69e1942fe68932eec1c7da4393b592cd41ffc68de9f1721a44c62ae2c645f3be198e64173d5151a329329cdc2ddabeadb82032e7be0ddaff534344e963984f80e245820c48e1fee944b8814b8d7f80095dd0a983270292c73aeed987e237405b86499098ca1fbc688dc986c17653be283ae69f9c371749034de089012c26497cffc9911b4e0365ceb489086f2b3c987c3073a334996d444686047f47651789c8c4e5d1f1864f54c384aad28b1d6856005c8ed4973e62fee5ec8966e462d18f74cb81407067ddf7b3c07a4cbcff164331a2c13c
    
    ```
    

sojm 分析
-------

*   其实打开 IDA 看到这我是懵逼的
    
    ![](https://bbs.kanxue.com/upload/attach/202303/918604_9F5R4QCBHS93XDP.png)
    
*   通过 ChatGPT 分析可知，这是 Go 写的 jni 程序（利用 ChatGPI 辅助分析这套组合拳是真的不错！！  
    ![](https://bbs.kanxue.com/upload/attach/202303/918604_D44JG2EHFVFJRXQ.png)
    
    ![](https://bbs.kanxue.com/upload/attach/202303/918604_XD8DX99QRFYG6PR.png)
    

*   这里其实可以简单理解是 Go 与 C/C++ 的交互

Java_com_qq_lib_EncryptUtil_encrypt 分析
--------------------------------------

*   汇编解释
    
    ```
    int __usercall Java_com_qq_lib_EncryptUtil_encrypt@()
    {
      cgo_wait_runtime_init_done();  // 初始化上下文
      crosscall2();
      cgo_release_context();  // 释放上下文
      return 0;
    } 
    ```
    
    ```
    ; Attributes: bp-based frame
     
    ; int __usercall Java_com_qq_lib_EncryptUtil_encrypt@()
    EXPORT Java_com_qq_lib_EncryptUtil_encrypt
    Java_com_qq_lib_EncryptUtil_encrypt
     
    var_2C= -0x2C
    var_28= -0x28
    var_24= -0x24
    var_20= -0x20
    var_1C= -0x1C
     
    PUSH            {R4-R8,R10,R11,LR}              ; 将寄存器 R4-R8、R10、R11 和 LR 压入栈中
    ADD             R11, SP, #0x18                  ; 建立栈帧:  将 R11 设为当前栈顶地址加上 24 的值
    SUB             SP, SP, #0x18                   ; 分配栈空间
    MOV             R8, R3                          ; 将寄存器 R3 中的值复制到 R8 中，这是第 4 个参数⭐pwd
    MOV             R5, R2                          ; 将寄存器 R2 中的值复制到 R5 中，这是第 3 个参数⭐src
    MOV             R6, R1                          ; 将寄存器 R1 中的值复制到 R6 中，这是第 2 个参数⭐class
    MOV             R7, R0                          ; 将寄存器 R0 中的值复制到 R7 中，这是第 1 个参数⭐env
    BL              _cgo_wait_runtime_init_done     ; 调用_cgo_wait_runtime_init_done函数
    MOV             R4, R0                          ; 将 _cgo_wait_runtime_init_done 函数的返回值存储到 R4 中
    MOV             R0, #0                          ; 将 0 存储到 R0 中
    STR             R8, [SP,#0x30+var_20]           ; 将 R8 寄存器的值保存到栈空间⭐pwd
    ADD             R1, SP, #0x30+var_2C            ; 计算第一个参数的地址⭐将指针变量 var_2C 的地址存储到 R1 中⭐args 参数结构体 栈起始地址
    STR             R5, [SP,#0x30+var_24]           ; 将 R5 寄存器的值保存到栈空间⭐src Java传过来的参数
    MOV             R2, #0x14                       ; 将 20 保存到 R2 寄存器，也就是参数占用的字节数⭐args_len 参数长度
    STR             R6, [SP,#0x30+var_28]           ; 将R6寄存器的值保存到栈空间⭐class
    MOV             R3, R4                          ; 将 R4 寄存器的值保存到 R3 寄存器⭐cgo_context
    STR             R7, [SP,#0x30+var_2C]           ; 将 R7 寄存器的值保存到栈空间
    STR             R0, [SP,#0x30+var_1C]           ; 将 0 保存到栈空间
    LDR             R0, =(_cgoexp_17c794619cba_Java_com_qq_lib_EncryptUtil_encrypt_ptr - 0x103EA0) ; 计算 _cgoexp_17c794619cba_Java_com_qq_lib_EncryptUtil_encrypt_ptr 函数的地址到 R0 中
    LDR             R0, [PC,R0]                     ; 将 _cgoexp_17c794619cba_Java_com_qq_lib_EncryptUtil_encrypt 的地址加载到 R0 中
    BL              crosscall2                      ; 调用 crosscall2 函数
    MOV             R0, R4                          ; 将 R4 寄存器的值保存到 R0 寄存器
    BL              _cgo_release_context            ; 调用 _cgo_release_context 函数
    LDR             R0, [SP,#0x30+var_1C]           ; 加载栈空间中的值
    SUB             SP, R11, #0x18                  ; 设置栈顶为 R11，即栈底指针，也就是恢复原来的栈空间
    POP             {R4-R8,R10,R11,PC}              ; 弹出 R4-R8、R10、R11 和返回地址 PC 的值，返回跳转到 PC 指向的地址
    ; End of function Java_com_qq_lib_EncryptUtil_encrypt 
    ```
    
*   参考文档 [内部机制 - Go 语言高级编程](https://chai2010.cn/advanced-go-programming-book/ch2-cgo/ch2-05-internal.html)
    
*   这段汇编码中，通过调用 **`_cgo_wait_runtime_init_done`** 函数等待 Go 运行时初始化完成，然后通过 **`crosscall2`** 函数用 C 语言传输参数调用 **`Java_com_qq_lib_EncryptUtil_encrypt`** 函数
*   最后，通过调用 **`_cgo_release_context`** 函数释放上下文，结束函数的执行
*   所以伪代码 `crosscall2` 函数调用解析可能出错了， R0 就是调用的函数，这里应该解析为 `crosscall2(中间代理函数指针, 调用参数与返回值的结构体指针, 参数结构体大小, _cgo_ctxt)`
*   `_cgo_ctxt` 是 `_cgo_wait_runtime_init_done` 的返回值
*   所以这里我们也可以对 `crosscall2` 的调用参数进行修改，使用快捷键 `Y` ，修改完过后流程就清晰多了 `void __fastcall crosscall2(int funPtr, int args, int args_len, int cgo_context)` 可以发现代码清晰许多
*   `Java_com_qq_lib_EncryptUtil_encrypt` 这里应该遵循 JNI 调用，所以参数 1 到参数 4 应该是 `env, clz, src, pwd`
    
    ![](https://bbs.kanxue.com/upload/attach/202303/918604_FE4YUVNUBPU2BQ7.png)
    
    ```
    int __usercall Java_com_qq_lib_EncryptUtil_encrypt@(int env, int clz, int src, int pwd)
    {
      int inited; // r4
      args v14; // [sp+4h] [bp-2Ch] BYREF
      int v15; // [sp+14h] [bp-1Ch]
     
      inited = cgo_wait_runtime_init_done();
      v14.pwd = pwd;
      v14.src = src;
      v14.clz = clz;
      v14.env = env;
      v15 = 0;
      crosscall2((int)cgoexp_17c794619cba_Java_com_qq_lib_EncryptUtil_encrypt, (int)&v14, 20, inited);
      cgo_release_context();
      return v15;
    } 
    ```
    

crosscall2
----------

*   `crosscall2` 函数将参数结构体传给 `cgoexp_17c794619cba_Java_com_qq_lib_EncryptUtil_encrypt` 函数执行，所以直接分析 `cgoexp_17c794619cba_Java_com_qq_lib_EncryptUtil_encrypt` 的逻辑即可
*   其实这里还可以使用 JEB 分析 So（下图为 JEB 分析 Native 函数的结果
    
    ![](https://bbs.kanxue.com/upload/attach/202303/918604_XES9JEP6BG559BD.png)
    
*   进入 `cgoexp_17c794619cba_Java_com_qq_lib_EncryptUtil_encrypt` 分析逻辑可以发现不管是 JEB 还是 IDA，区别还是挺大的，但我对它俩的反编译结果都不是很满意
    
    ![](https://bbs.kanxue.com/upload/attach/202303/918604_3KJJKDQ35VGD937.png)
    
*   大概分析了下，看了几个函数调用觉得 `sub_FFD0C` 函数有点像加密处理相关的东西
    
    ![](https://bbs.kanxue.com/upload/attach/202303/918604_PC5N23TY5XGXQVE.png)
    

修复 Go 符号
--------

*   后面逆 Go 的应用实在痛苦，发现了一个 IDA Go 的辅助分析插件 [go_parser: Yet Another Golang binary parser for IDAPro](https://github.com/0xjiayu/go_parser)
*   ALT + F7 运行 `go_parser/go_parser.py` 加载脚本文件即可恢复符号，不过这里脚本似乎跑到提示 `Standard types building finished. Total types count: 718` 应该就可以中断脚本了（估计脚本还存在 Bug，不过对于我们来说够用了
    
    ![](https://bbs.kanxue.com/upload/attach/202303/918604_DZ6MRHR6HP33T95.png)
    
*   与没修复的 `cgoexp_17c794619cba_Java_com_qq_lib_EncryptUtil_encrypt` 函数对比，可以发现清晰许多
    
    ```
    // IDA源反编译源代码
    int __usercall cgoexp_17c794619cba_Java_com_qq_lib_EncryptUtil_encrypt@(int a1, int *a2)
    {
      int v2; // r10
      int v4; // [sp+14h] [bp-4h]
     
      while ( (unsigned int)&a1 <= *(_DWORD *)(v2 + 8) )
        sub_9FD10();
      v4 = sub_103658(*a2, a2[1], a2[2], a2[3]);
      a2[4] = v4;
      sub_40DF8(v4);
      return sub_3ADAC();
    }
     
    // 经过脚本修复
    int __usercall cgoexp_17c794619cba_Java_com_qq_lib_EncryptUtil_encrypt@(int a1, int *a2)
    {
      int v2; // r10
      int v4; // [sp+14h] [bp-4h]
     
      while ( (unsigned int)&a1 <= *(_DWORD *)(v2 + 8) )
        runtime_morestack_noctxt();
      v4 = main_Java_com_qq_lib_EncryptUtil_encrypt(*a2, a2[1], a2[2], a2[3]);
      a2[4] = v4;
      runtime_convT32(v4);
      return runtime_cgoCheckResult();
    } 
    ```
    
*   `sub_FFD0C` 函数对应 `main__libso_encrypt` ，是不是清晰许多
    
    ![](https://bbs.kanxue.com/upload/attach/202303/918604_HBSADTW8DVJTRK8.png)
    

main__libso_encrypt 分析
----------------------

*   通过 hook 可以发现这里主要传入了待加密的明文，跟 `pwd`，其他参数跟 Go 的字符串结构有关，开头有提到过，不过这个参数是真的乱
*   这里还写了一个 `Call` 去调用，方便我们调试，也可以选择传入不是 JSON 格式的数据，因为后面有对这个明文进行处理，如果不是 JSON 格式就不会执行那个流程，但是也会出密文
    
    ```
    function hook_main__libso_encrypt() {
        let base = Module.findBaseAddress("libsojm.so");
     
        Interceptor.attach(base.add(0xFFD0C), {
            onEnter: function (args) {
                this.arg0_len = findRangeByAddress(args[1])
                if (this.arg0_len == 64) {  // 过滤输出, 感觉其他值都是无用的
                    console.log(`onEnter encrypt arg0:${args[0]} arg1:${args[1]} arg2:${args[2]} arg3:${args[3]} arg4:${args[4]} arg5:${args[5]} arg6:${args[6]} arg7:${args[7]} arg8:${args[8]} arg9:${args[9]}`);
                    console.error(`[ * ] libso_encrypt.args[${0}] onEnter :=> ${args[0].readCString()}`)  // BwcnBzRjN2U/MmZhYjRmND4xPjI+NWQwZWU0YmI2MWQ3YjAzKw8cEywsIS4BIg==
                    console.error(`[ * ] libso_encrypt.args[${1}] onEnter :=> ${findRangeByAddress(args[1])}`)  // arg1 是 args[0] 字符串的长度
                    // console.error(`[ * ] libso_encrypt.args[${1}] onEnter :=> ${findRangeByAddress(args[1])}`)  // arg1 是 args[0] 字符串的长度
                    // console.error(`[ * ] libso_encrypt.args[${2}] onEnter :=> ${findRangeByAddress(args[2].readPointer())}`)  // arg2 感觉也是字符串长度 go 的字符串可能是字符串+长度 | slice 是长度+容量
                    // arg3 似乎是函数执行次数
                    // console.error(`[ * ] libso_encrypt.args[${4}] onEnter :=> ${findRangeByAddress(args[4].readPointer().readPointer())}`)  // 867740 固定值
                    // console.error(`[ * ] libso_encrypt.args[${5}] onEnter :=> ${findRangeByAddress(args[5].readPointer().readPointer())}`)  // 16个0
                    console.error(`[ * ] libso_encrypt.args[${5}] onEnter :=> ${ab2Hex(args[5].readPointer().readPointer().readByteArray(16))}`)  // 16个0
                    console.error(`[ * ] libso_encrypt.args[${6}] onEnter :=> ${args[6].readCString()}`)         // data 明文数据⭐
                    console.error(`[ * ] libso_encrypt.args[${7}] onEnter :=> ${findRangeByAddress(args[7])}`)   // 明文数据长度
                    console.error(`[ * ] libso_encrypt.args[${8}] onEnter :=> ${findRangeByAddress(args[8])}`)   // 也是明文数据长度
                    console.error(`[ * ] libso_encrypt.args[${9}] onEnter :=> ${args[9].readCString()}`)         // 又是key
                    console.error(`[ * ] libso_encrypt.args[${10}] onEnter :=> ${findRangeByAddress(args[10])}`) // key 长度
                    console.error(`[ * ] libso_encrypt.args[${11}] onEnter :=> ${findRangeByAddress(args[11])}`) // key 长度
                    console.error("------------------------------------------------------------");
                }
     
            }, onLeave: function (retval) {
                // console.log(`onLeave encrypt ${retval}`);
                // console.error("------------------------------------------------------------");
            }
        });
    }
     
    function call() {
        Java.perform(() => {
            let ret = Java.use("com.qq.lib.EncryptUtil").encrypt(`{"system_build_id":"a1000","system_iid":"923868eb5f543afa55f1f33cfac37d35","app_status":"9A5C6BDC62AD1CFE45A6578F84E858F9CA1A4F76:2","system_version":"5.7.1","system_build_aff":"","bundle_id":"tv.iytqy.cvhaca","system_app_type":"local","new_player":"fx","system_oauth_id":"02274477773a9ac8a2cf3605400ece4b","system_oauth_type":"android","system_token":"023FAC3AFE2285DC98E50CA4C638E0845277DC8CB62CCCC088DA1E3CB679180C97F2688C0DBE197F03950D84900C2A6B050334216C4CC604021DCC624543AFDDEF8D19363E43F9C0B8EEC79BBF269E7DE77B0304FF1034FED885ACA4C74DAC768D26D9FADD"}`, "BwcnBzRjN2U/MmZhYjRmND4xPjI+NWQwZWU0YmI2MWQ3YjAzKw8cEywsIS4BIg==")
            console.warn(`encrypt("abcdef0123456789", "BwcnBzRjN2U/MmZhYjRmND4xPjI+NWQwZWU0YmI2MWQ3YjAzKw8cEywsIS4BIg==") :=> ${ret}`)
        })
    }
     
    function hook_dlopen(addr, soName, callback) {
        Interceptor.attach(addr, {
            onEnter: function (args) {
                const name = args[0].readCString();  // 输出so路径
                if (name.indexOf(soName) !== -1) this.hook = true;
            }, onLeave: function (retval) {
                if (this.hook) callback();
            }
        })
    }
     
    const android_dlopen_ext = Module.findExportByName(null, "android_dlopen_ext");
    hook_dlopen(android_dlopen_ext, "libsojm.so", so);
     
    so()
    
    ```
    
    ```
    // 输出日志
    onEnter encrypt arg0:0x86902940 arg1:0x40 arg2:0x40 arg3:0x22d arg4:0xc6a44ba0 arg5:0x86965f28 arg6:0x86944b40 arg7:0x22d arg8:0x22d arg9:0x86902940
    [ * ] libso_encrypt.args[0] onEnter :=> BwcnBzRjN2U/MmZhYjRmND4xPjI+NWQwZWU0YmI2MWQ3YjAzKw8cEywsIS4BIg==
    [ * ] libso_encrypt.args[1] onEnter :=> 64
    [ * ] libso_encrypt.args[5] onEnter :=> [00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00]
    [ * ] libso_encrypt.args[6] onEnter :=> {"system_build_id":"a1000","system_iid":"923868eb5f543afa55f1f33cfac37d35","app_status":"9A5C6BDC62AD1CFE45A6578F84E858F9CA1A4F76:2","system_version":"5.7.1","system_build_aff":"","bundle_id":"tv.iytqy.cvhaca","system_app_type":"local","new_player":"fx","system_oauth_id":"02274477773a9ac8a2cf3605400ece4b","system_oauth_type":"android","system_token":"023FAC3AFE2285DC98E50CA4C638E0845277DC8CB62CCCC088DA1E3CB679180C97F2688C0DBE197F03950D84900C2A6B050334216C4CC604021DCC624543AFDDEF8D19363E43F9C0B8EEC79BBF269E7DE77B0304FF1034FED885ACA4C74DAC768D26D9FADD"}
    [ * ] libso_encrypt.args[7] onEnter :=> 557
    [ * ] libso_encrypt.args[8] onEnter :=> 557
    [ * ] libso_encrypt.args[9] onEnter :=> BwcnBzRjN2U/MmZhYjRmND4xPjI+NWQwZWU0YmI2MWQ3YjAzKw8cEywsIS4BIg==
    [ * ] libso_encrypt.args[10] onEnter :=> 64
    [ * ] libso_encrypt.args[11] onEnter :=> 64
    ------------------------------------------------------------
    encrypt("abcdef0123456789", "BwcnBzRjN2U/MmZhYjRmND4xPjI+NWQwZWU0YmI2MWQ3YjAzKw8cEywsIS4BIg==") :=> 6841646c536444435553467a65706946f933c2cae8a2a199fb1e46d1d60a1da52bd51cf0bdf4cae032f95454abdbfed529ccfcf252fa7581ef7504c5cd3c3465940bcbc59c629a029459dc10585f7426ab652646f2d2e4fc5861572e9b6ee65320ac4190fc6ffe3d41487fbb365e98dd906f9c3b13cbc46e422e75b0a28b947ede0dfe4acc98a851ec029f3d74f556f31cc2b82fafe70669fd0177ab1e2194e919c1c86789978beb1a46c2b4b14fb199596b46050fa85bf667d599fb31fd56e00de19b90b154c73a418c0ca7f4b9a2fadb3d5f912551c255681893db2e4260e3982743c8dfa5f3ba1010b58ae876c3361348f45cdfbd11be8a65e036e7c638834aa787c08417102f84c612020587e5518db749cd5616394e2eee544ca4febfa386e9befb9e8634ef17f34cb5afb8f1cbe45bb00706ef2547a8009b8faa7e7c4702dc9061e0143478d413b4d0477720c4c9779cc7b4f2ae60cc6b3fe5191e3c8a4948a344c92fdff1bd291b79d7148f933771199b6383bfa05c51f73d059673e5a1459213d2ef2bb858c134093aa44bb58f139a6870a853e31b9590c216f70e739fcbaef17d95550e61d8c3ca5ffa257f7f2534e6ed069e51f355a2b03e9f786fd33060b39f17721acb449bec0fce25cb9fdfce67be2475b06eba9ab0005a68bace1918ce4a2e61a726f286f775963f424c22dbcffaefef7d575fa868fb7a5111f36967a3537fd3d0c24e8e86196df7885624620fed4f085a07d205f19d201536391a1687fd7781491a60fe27f11d3ff3b03bba1ec20175ba4533ac8e2d3fd3a13dd20721706da02e5107745436ebd97425b1b7550f68e55d65a021e3467d1fa48a44912669286f4915ed84dbe276e53059f632d98cf2154831857d23c7d88f3f54f5050d7e97290b5f5a91a976c2ff4aec897c84ccc6ebad
    
    ```
    

### **main__libso_encrypt 伪代码**

```
void __golang main__libso_encrypt(
        int a1,
        int a2,
        int key1,
        int a4,
        int a5,
        int a6,
        int a7,
        int a8,
        int data,
        int data_len,
        int a11,
        int key2)
{
// .. 省略一些参数命名
 
  while ( (unsigned int)&a1 <= *(_DWORD *)(v12 + 8) )
    runtime_morestack_noctxt();
  data = 0;
  data_len = 0;
  v96 = &off_11ED34;
  main_logger(v13, (int)"in encrypt", 10);      // 应该是个C的打印函数
  main_parsePassphrase(v24, a6, a7, a8);        // 将 base64 编码的字符串解码并进行格式校验
                                                // 1.检查栈空间是否足够。
                                                //
                                                // 2.将传入的 base64 编码的密码字符串解码为二进制数据。
                                                //
                                                // 3.如果解码出错或者解码后的二进制数据长度小于 30，则返回相应的错误信息。
                                                //
                                                // 4.根据特定规则对解密后的二进制数据进行处理（异或操作等）。
                                                //
                                                // 5.将处理后的数据切片成若干个固定长度的子切片，并分别记录日志。
                                                //
                                                // 6.对其中一个子切片进行进一步的处理，并检查其格式是否符合特定要求。
                                                //
                                                // 7.如果格式不符合要求，则返回相应的错误信息；否则返回空指针表示处理成功。
  v14 = v58;
  if ( v83 )
  {
    data = 0;
    data_len = 0;
    main__libso_encrypt_func1(v37, v41);        // 异常捕获的函数,用于恢复发生在goroutine中的panic
  }
  else
  {
    v92 = v69;
    v87 = v75;
    v88 = v78;
    v93 = v79;
    runtime_newobject(v25, (int)&map_string_interface_, v41);// malloc 分配一段内存并返回指向该内存地址的指针
    v95 = v42;
    runtime_makemap_small(v26, v38);
    if ( dword_1B9460 )
    {
      runtime_gcWriteBarrier();                 // 垃圾回收
      v15 = v16;
    }
    else
    {
      v15 = v95;
      *(_DWORD *)v95 = v39;
    }
    encoding_json_Unmarshal(v27, key1, a4, a5, (int)&map_string_interface__ptr, v15, v75, v78);// 将 JSON 数据解析为 Go 的结构体
    if ( v76 )
    {
      v17 = a5;
      v18 = a4;
      v19 = key1;
    }
    else
    {
      v20 = v89;
      do
        *v20++ = 0;                             // 内存清零
      while ( (int)v20 <= (int)&v89[15] );
      qmemcpy(v89, "__package_name__", sizeof(v89));// 字符串拷贝
      runtime_slicebytetostring(v28, 0, (int)v89, 16, v59, v70);// 将字节切片转换为字符串
      v91 = v60;
      v86 = v71;
      v21 = v89;
      do
        *v21++ = 0;                             // 内存清零
      while ( (int)v21 <= (int)&v89[15] );
      qmemcpy(v89, "__package_hash__", sizeof(v89));// 字符串拷贝
      runtime_slicebytetostring(v29, 0, (int)v89, 16, v60, v71);// 将字节切片转换为字符串
      v90 = v61;
      v85 = v72;
      main__libso_getPackageName(v30, a2, v44, v48, v61);// 反射调用 Java 层方法获取包名
      runtime_slicebytetostring(v31, 0, v45, v49, v62, v72);// 将字节切片转换为字符串
      runtime_convTstring(v32, v63, (int)v73, v50);// 将指向任意类型的指针转换为字符串
      v22 = *(_DWORD *)v95;
      v94 = v51;
      runtime_mapassign_faststr((int)&map_string_interface_, v22, v91, v86);// 将指针指向的数据转为字符串返回
      *v73 = &string;
      if ( dword_1B9460 )
        runtime_gcWriteBarrier();               // 垃圾回收
      else
        v73[1] = v94;
      main__libso_getPackageCodePath(v33, a2, v46, v52, v64);// 获取应用路径
      runtime_slicebytetostring(v34, 0, v47, v53, v65, (int)v73);// 将字节切片转换为字符串
      main_md5File(v35, v66, (int)v74, v54, v66);
      runtime_convTstring(v36, (int)v55, v67, v55);// 将指向任意类型的指针转换为字符串
      v23 = *(_DWORD *)v95;
      v94 = v56;
      v77 = runtime_mapassign_faststr((int)&map_string_interface_, v23, v90, v85);// 将指针指向的数据转为字符串返回
      *v74 = &string;
      if ( dword_1B9460 )
        runtime_gcWriteBarrier();               // 垃圾回收
      else
        v74[1] = v94;
      v68 = encoding_json_Marshal((int)&map_string_interface_, *(_DWORD *)v95);// 转json数据
      v19 = v57;
      v18 = v68;
      v17 = (int)v74;
      if ( v77 )
      {
        v17 = a5;
        v18 = a4;
        v19 = key1;
      }
    }
    if ( v14 )
    {
      data = 0;
      data_len = 0;
    }
    else
    {
      main_oldEncrypt(v28, v19, v18, v17, v93, v80, v81, v92, v87, v88, v81, 0);// 开始加密
      data = v82;
      data_len = v84;
    }
    main__libso_encrypt_func1(v40, v43);
  }
}

```

### main_parsePassphrase

*   该函数主要做的事：将 base64 编码的字符串解码并进行格式校验
    
    1.  检查栈空间是否足够 `runtime_morestack_noctxt`
    2.  将传入的 base64 编码的密码字符串解码为二进制数据
    3.  如果解码出错或者解码后的二进制数据长度小于 30，则返回相应的错误信息
    4.  根据特定规则对解密后的二进制数据进行处理（异或操作等）
    5.  将处理后的数据切片成若干个固定长度的子切片，并分别记录日志
    6.  对其中一个子切片进行进一步的处理，并检查其格式是否符合特定要求
    7.  如果格式不符合要求，则返回相应的错误信息；否则返回空指针表示处理成功
        
        ```
        function hook_main_parsePassphrase_ret() {
         let base = Module.findBaseAddress("libsojm.so");
         Interceptor.attach(base.add(0xFFD9C), {
             onEnter(args) {
                 console.log(`call 0xFFD9C ${JSON.stringify(this.context)}`);
                 console.log(`call 0xFFD9C ${findRangeByAddress(this.context.r0)}`);
                 console.log(`call 0xFFD9C ${findRangeByAddress(this.context.r1)}`);
             }
         });
        }
         
        // BwcnBzRjN2U/MmZhYjRmND4xPjI+NWQwZWU0YmI2MWQ3YjAzKw8cEywsIS4BIg==
        // ..'.4c7e?2fab4f4>1>2>5d0ee4bb61d7b03+...,,!.." 这是直接解码的字符串
         
        // 得到了两个字符串（其实这里就是对我们传入的 PWD 解码对后半部分进行了一些类的运算
        8692ece4  6d 49 5a 55 6a 6a 67 68 47 64 00 00 00 00 00 00  mIZUjjghGd......
         
        8692ecc4  34 63 37 65 3f 32 66 61 62 34 66 34 3e 31 3e 32  4c7e?2fab4f4>1>2
        8692ecd4  3e 35 64 30 65 65 34 62 62 36 31 64 37 62 30 33  >5d0ee4bb61d7b03
         
        // 关于这里为什么是从栈上取的，是因为它的调用约定并不是 fastcall，分析汇编可知都是栈传递参数跟接受返回值
        .text:000FFD64 5B 0D 00 EB                   BL              main_logger
        .text:000FFD64
        .text:000FFD68 90 00 9D E5                   LDR             R0, [SP,#0x7C+arg_14]
        .text:000FFD6C 04 00 8D E5                   STR             R0, [SP,#0x7C+var_78]   ; int
        .text:000FFD70 94 00 9D E5                   LDR             R0, [SP,#0x7C+data]
        .text:000FFD74 08 00 8D E5                   STR             R0, [SP,#0x7C+var_74]   ; int
        .text:000FFD78 98 00 9D E5                   LDR             R0, [SP,#0x7C+data_len]
        .text:000FFD7C 0C 00 8D E5                   STR             R0, [SP,#0x7C+var_70]   ; int
        .text:000FFD80 1C 05 00 EB                   BL              main_parsePassphrase
        .text:000FFD80
        .text:000FFD84 14 00 9D E5                   LDR             R0, [SP,#0x7C+var_68]
        .text:000FFD88 20 10 9D E5                   LDR             R1, [SP,#0x7C+var_5C]
        .text:000FFD8C 18 20 9D E5                   LDR             R2, [SP,#0x7C+var_64]
        .text:000FFD90 1C 30 9D E5                   LDR             R3, [SP,#0x7C+var_60]
        .text:000FFD94 24 40 9D E5                   LDR             R4, [SP,#0x7C+var_58]
        .text:000FFD98 28 50 9D E5                   LDR             R5, [SP,#0x7C+var_54]
        .text:000FFD9C 2C 60 9D E5                   LDR             R6, [SP,#0x7C+var_50]
        .text:000FFDA0 10 70 DD E5                   LDRB            R7, [SP,#0x7C+var_6C]
        .text:000FFDA4 11 80 DD E5                   LDRB            R8, [SP,#0x7C+var_6C+1]
        .text:000FFDA8 00 00 56 E3                   CMP             R6, #0
        .text:000FFDAC 32 01 00 1A                   BNE             loc_10027C
        
        ```
        

### main_md5File

*   这里就是传入了一个文件路径，得到了一个文件的 MD5 值
    
    ```
    function hook_main_md5File() {
        let base = Module.findBaseAddress("libsojm.so");
     
        Interceptor.attach(base.add(0xFF94C), {
            onEnter: function (args) {
                this.arg1 = args[1]
                if (this.arg1 == 0x3b) {
                    console.log(`onEnter main_md5File arg0:${args[0]} arg1:${args[1]} arg2:${args[2]} arg3:${args[3]} arg4:${args[4]}`);
                    console.log(`onEnter main_md5File arg0:${args[0].readCString(0x3b)}`);  // 包路径 /data/app/tv.iytqy.cvhaca-gmGCryB_O0HztLC7rqkhRQ==/base.apk
                    // console.log(`onEnter main_md5File arg1:${findRangeByAddress(args[1])}`);// arg1 参数0 长度
                    // console.log(`onEnter main_md5File arg2:${findRangeByAddress(args[2])}`); // 地址内容为0
                    // console.log(`onEnter main_md5File arg3:${findRangeByAddress(args[3])}`);  // 地址内容为0
                    // console.log(`onEnter main_md5File arg4:${findRangeByAddress(args[4])}`);  // 不知道这个是啥
                }
     
            }, onLeave: function (retval) {
     
                if (this.arg1 == 0x3b) {
                    console.log(`onLeave main_md5File ${retval.readCString(32)}`);  // 678c7f3bf3584a2079295d8834928146
                }
            }
        });
    }
    
    ```
    
    ```
    onEnter main_md5File arg0:0x86844940 arg1:0x3b arg2:0x304f5f42 arg3:0x4c747a48 arg4:0xe6f73709
    onEnter main_md5File arg0:/data/app/tv.iytqy.cvhaca-gmGCryB_O0HztLC7rqkhRQ==/base.apk
    onLeave main_md5File 678c7f3bf3584a2079295d8834928146
    
    ```
    
*   写个了 Python 代码进行验证，发现结果是对的
    
    ```
    import hashlib
     
    filename = '50ash_5.7.1_230305_3.apk'
    hasher = hashlib.md5()
     
    with open(filename, 'rb') as f:
        buf = f.read()
        hasher.update(buf)
     
    md5hash = hasher.hexdigest()
    print(md5hash)
     
    # [Running] python -u "c:\Users\Administrator\Desktop\50度灰\getMD5File.py"
    # 678c7f3bf3584a2079295d8834928146
    
    ```
    

### encoding_json_Marshal

*   这里就是把我们之前的字符串再次序列化
    
    ```
    v68 = encoding_json_Marshal((int)&map_string_interface_, *(_DWORD *)v95);  // 转json数据
    v18 = v68;
    main_oldEncrypt(v28, v19, v18, v17, v93, v80, v81, v92, v87, v88, v81, 0);  // 开始加密
     
    // 所以这里我们直接 hook main_oldEncrypt 就可以得到加密前的明文了
    
    ```
    

### main_oldEncrypt

*   hook 代码
    
    ```
    function hook_main_oldEncrypt() {
        let base = Module.findBaseAddress("libsojm.so");
     
        Interceptor.attach(base.add(0xFF5EC), {
            onEnter: function (args) {
                console.log(`onEnter main_oldEncrypt arg0:${args[0]} arg1:${args[1]} arg2:${args[2]} arg3:${args[3]} arg4:${args[4]} arg5:${args[5]} arg6:${args[6]} arg7:${args[7]} arg8:${args[8]} arg9:${args[9]} arg10:${args[10]} arg11:${args[11]}`);
                console.warn(`onEnter main_oldEncrypt arg1:${args[1].readCString()}`);
                // console.log(`onEnter main_oldEncrypt arg4:${findRangeByAddress(args[4])}`);
                console.log(`onEnter main_oldEncrypt arg8:${args[8].readCString()}`);  // key1
                console.log(`onEnter main_oldEncrypt arg11:${args[11].readCString()}`);  // key2
            }, onLeave: function (retval) {
                // console.log(`onLeave main_oldEncrypt ${findRangeByAddress(retval)}`);
                console.log(`onLeave main_oldEncrypt ${retval.readCString()}`);
                console.error("------------------------------------------------------------");
            }
        });
    }
    
    ```
    
    ```
    [Pixel::XXXXX]-> call()
    onEnter main_oldEncrypt arg0:0xc arg1:0x869d0580 arg2:0x288 arg3:0x2c0 arg4:0xc96436f0 arg5:0x869d0580 arg6:0x288 arg7:0x2c0 arg8:0x868a4c34 arg9:0x20 arg10:0x2c arg11:0x868a4c54
    onEnter main_oldEncrypt arg1:{"__package_hash__":"678c7f3bf3584a2079295d8834928146","__package_name__":"tv.iytqy.cvhaca","app_status":"9A5C6BDC62AD1CFE45A6578F84E858F9CA1A4F76:2","bundle_id":"tv.iytqy.cvhaca","new_player":"fx","system_app_type":"local","system_build_aff":"","system_build_id":"a1000","system_iid":"923868eb5f543afa55f1f33cfac37d35","system_oauth_id":"02274477773a9ac8a2cf3605400ece4b","system_oauth_type":"android","system_token":"023FAC3AFE2285DC98E50CA4C638E0845277DC8CB62CCCC088DA1E3CB679180C97F2688C0DBE197F03950D84900C2A6B050334216C4CC604021DCC624543AFDDEF8D19363E43F9C0B8EEC79BBF269E7DE77B0304FF1034FED885ACA4C74DAC768D26D9FADD","system_version":"5.7.1"}
    onEnter main_oldEncrypt arg8:4c7e?2fab4f4>1>2>5d0ee4bb61d7b03mIZUjjghGd
    onEnter main_oldEncrypt arg11:mIZUjjghGd
    onLeave main_oldEncrypt 436a4b4d6961565043615063737077526c2e5dd70001b006397ce672c6fe1948f74ac72436bccb766585953419b0a7d4643fd7b98319a6696558f3e129bd7865383a4311256cdc126a00d49126379d5cc3f5a1a0fcd8a6fb6f6293d0ad6e628451cd26efecfaa5021099af5b428da2081533953dcf9a1e3a6f46a560471e6cef6e591b4937c36cd5ccb0de0364279eecf0970e101286faa0d25476e49a2e8b5a24f5d4d2e2c1afb67785f96511f48290864e6fdac548a325f9b30b0c7a99dd2c909518ccac45d9efa2c9973c5f0de468d094e6573afc44d2ffe73d6d79ee76d45fd8fcdf7d45c8a47fca7e85769239da6bfd9e35eaaa59bf3a6bb7741258a42bca7272e611bb8ddd93e807d126c3898f0f1aa9f37af485c140b706fe7a3f0d684eb3132ecd208cf52338a617095e7bb2b2b3eb749c7f7f01d8bfc7d386a2799ae6f015a87ba1519109aa92c4ec2e9d15650a46d239616ec515605d56b7ef3a954f2edbbb85f8c86e2f19c19e6422cc7dcf2277ebd968cf9751f5548f5ae77387985e0dfb7262787b13416138cb5c5952d4d3305a7688910984c75adc0a7f85347917bef0d909ad5a024f2275a5609e2d813f4c43d9bd00b4e82479725b78b197032bf78a163db5f1cbbbd1d09df698cbcb0024646d58d848dd08ad6168413842aa3925a5dd782c6b7d4d58d38a24d3dfe17e8a5e7be5dbeb0ade63d730e10015fd421d1f78f8744322f38f2f2764584dd4f6b36fc3cd0ccd22a9aaaacd92174882fccabc91712364d2afbe83bb6c2ad3d5cafcfd2f09e6e94d79d9dea18c9f7a599640177893b37bdd3e6ed388339d9c1ef9f5d9c197cff02555488bbce4b8ac7366b623b9dadcf3813a6451290094773076b616bbf9f7079c5c39c1ed3c3e211648f862d47b34d644d4c0ae05e0d5fb73d8b046151d2677
     
    // 分析
    // onLeave 返回值就是我们最终加密的密文
    // arg1 是我们加密前的明文
    // 传入了两个key 4c7e?2fab4f4>1>2>5d0ee4bb61d7b03mIZUjjghGd 与 mIZUjjghGd
    // 其实这里应该是 4c7e?2fab4f4>1>2>5d0ee4bb61d7b03 与 mIZUjjghGd，他们挨在一起的，所以是一起打印的
    
    ```
    

**main_oldEncrypt** 详细分析
------------------------

```
int __usercall main_oldEncrypt@(
        int a1,
        int data,
        int data_len,
        int data_cap,
        int a5,
        int data_,
        int data_len_,
        int data_cap_,
        int key1,
        int key1_len,
        int key1_cap,
        int key2)
{
  // 省略一些参数命名
 
  while ( (unsigned int)&a1 <= *(_DWORD *)(v12 + 8) )
    runtime_morestack_noctxt();
  for ( i = 0; data_ > (int)i; ++i )            // 对key做第一次处理
  {
    for ( j = 0; key1 > j; ++j )
    {
      if ( data_ <= i )
        runtime_panicIndex();
      *(_BYTE *)(a5 + i) ^= *(_BYTE *)(data_cap_ + j);
    }
  }
  v44 = main_swapByteLocation(a5, data_, data_len_, data_cap_, key1, key1_len);// 对key做第二次处理
  ((void (*)(void))main_opensslRandomPseudoBytes)();// 生成指定长度的随机IV
  v41 = v29;
  v45 = v28;
  v42 = v30;
  v37 = main_evpBytesToKey(v44, v38, v39, 32, 16);// 对key做第三次处理
  v31 = crypto_aes_NewCipher(v37, v44, v38);    // 初始化aes
  if ( v37 )
  {
    main_logger(v26, (int)"new cipher error", 16);
    key1_cap = 0;
    result = 0;
    key2 = 0;
  }
  else
  {
    runtime_makeslice((int)&uint8, data_len, data_len);// slice 扩容 | 申请内存空间
    v19 = *(void (**)(void))(crypto_cipher_newCFB(v31, v34, v45, v41, v42, 0) + 16);// AES_CFB 初始化
    v35 = data;
    v19();                                      // ⭐开始加密  偏移CC0D8 crypto_cipher__cfb_XORKeyStream | XORKeyStream(&密文, 明文)
    v40 = v41 + data_len;
    if ( v42 < v41 + data_len )
      v20 = runtime_growslice((int)&uint8, v45, v41, v42, v41 + data_len);
    else
      v20 = v45;
    v43 = v20;
    runtime_memmove();                          // 连接初始化向量跟密文
    v32 = runtime_makeslice((int)&uint8, 2 * v40, 2 * v40);// 声明 slick 类似这样 make([]int,5,10)，两倍密文长度
    v21 = 0;
    v22 = 0;
    while ( v21 < v40 )                         // 循环对密文进行映射，也就是说对密文进行二次处理，得到一个新的密文
    {
      v24 = *(_BYTE *)(v43 + v21);
      if ( v22 >= 2 * v40 )
        runtime_panicIndex();
      *(_BYTE *)(v32 + v22) = a0123456789abcd_0[v24 >> 4];// a0123456789abcd_0 => "0123456789abcdef" 字符串映射
      v25 = a0123456789abcd_0[v24 & 0xF];
      if ( v22 + 1 >= 2 * v40 )
        runtime_panicIndex();
      *(_BYTE *)(v32 + v22 + 1) = v25;
      ++v21;
      v22 += 2;
    }
    runtime_slicebytetostring(v27, 0, v32, 2 * v40, v32, v35); // 密文转 hex 字符串,是我们最终结果
    result = v33;
    key1_cap = v33;
    key2 = v36;
  }
  return result;
} 
```

### key & IV

*   这里是 for 循环得到一个 key3，在调用函数 `main_swapByteLocation` 得到 key4，因为这里的值是固定的，只要 pwd 与前面对 pwd 的算法没有改变，这里就可以直接拿最后计算的结果即可
    
    ```
    for ( i = 0; data_ > i; ++i )
    {
      for ( j = 0; key1 > j; ++j )
      {
        if ( data_ <= i )
          runtime_panicIndex();
        *(a5 + i) ^= *(data_cap_ + j);
      }
    }
    v53 = main_swapByteLocation(a5, data_, data_len_, data_cap_, key1, key1_len);
    main_opensslRandomPseudoBytes(v27, 0x10u, v30, v32, v34); // 生成指定长度的随机IV
    ...
    main_evpBytesToKey(v29, v53, v47, v48, 32, 16);  // 这里还对 key 做了一次处理
     
    // EvpBytesToKey 函数是 Go 语言中的一个加密函数，用于将一组字节数组转换为一个密钥和初始化向量（IV）。该函数通常用于将一个密码字符串转换为一个加密密钥。
    // 该函数的实现基于 OpenSSL 中的 EVP_BytesToKey 函数，它使用一个称为 EVP 的加密算法来生成密钥和 IV。该算法使用一个称为“密钥派生函数”的过程，将输入的字节数组转换为一个加密密钥和一个随机的初始化向量。
     
    // 在 Go 中，EvpBytesToKey 函数的签名如下：
    // func EvpBytesToKey(password []byte, keyLen, ivLen int) ([]byte, []byte)
    // 其中，password 表示输入的密码字节数组，keyLen 表示生成的密钥长度，ivLen 表示生成的初始化向量长度。该函数返回两个字节数组，分别表示生成的密钥和初始化向量。
    // 需要注意的是，EvpBytesToKey 函数并不是一个标准的加密函数，它只是一个基于 OpenSSL 的实现。因此，它可能不适用于所有的加密场景，需要谨慎使用。
    
    ```
    
    ```
    // 将原始的密码和盐值转换成密钥和初始向量
     
    // The EVP_BytesToKey function is a key derivation function that uses a
    // hash function to derive a key and initialization vector (IV) from a
    // password and salt. It is used in the OpenSSL library to derive keys for
    // symmetric encryption algorithms.
    func EvpBytesToKey(password, salt []byte, keyLen, ivLen int, md func() hash.Hash) (key, iv []byte) {
        // The key and IV are derived from the password and salt using a
        // hash function. The OpenSSL library uses the MD5 hash function by default,
        // but other hash functions can be used by passing a different hash.Hash
        // implementation to the function.
        var (
            keyBuf = make([]byte, keyLen)
            ivBuf  = make([]byte, ivLen)
            prev   []byte
        )
        for i := 0; i < keyLen+ivLen; {
            h := md()
            if len(prev) > 0 {
                h.Write(prev)
            }
            h.Write(password)
            h.Write(salt)
            sum := h.Sum(nil)
            copy(sum, keyBuf[i:])
            i += len(sum)
            if i < keyLen+ivLen {
                prev = sum
            }
        }
        copy(key, keyBuf)
        copy(iv, ivBuf)
        return keyBuf, ivBuf
    }
     
    // 该函数的实现过程如下：
     
    // 1.创建一个长度为 keyLen+ivLen 的缓冲区 keyBuf，并创建一个长度为 ivLen 的缓冲区 ivBuf，用于存储密钥和初始向量。
    // 2.创建一个空的前一个哈希值 prev，用于存储前一个哈希值的结果。
    // 3.循环执行以下步骤，直到生成了足够的密钥和初始向量：
    // 3.1创建一个新的哈希函数实例 h。
    // 3.2如果 prev 不为空，则将 prev 添加到哈希函数中。
    // 3.3将密码和盐值添加到哈希函数中。
    // 3.4计算哈希函数的结果，并将其复制到 keyBuf 的相应位置。
    // 3.5如果生成的密钥和初始向量长度还不够，则将结果存储到 prev 中，以便下次循环使用。
    // 4.将 keyBuf 和 ivBuf 复制到 key 和 iv 中，并返回 key 和 iv。
    // 总体来说，EvpBytesToKey 函数的实现过程比较简单，主要是通过哈希函数对密码和盐值进行迭代计算，最终生成密钥和初始向量。
    
    ```
    
*   关于 Key 的处理
    
    ```
    function hook_main_swapByteLocation() {
        let base = Module.findBaseAddress("libsojm.so");
     
        Interceptor.attach(base.add(0x10170C), {
            onLeave: function (retval) {
                console.warn(`key3: ${retval.readCString()}`);  // fc9e9a75e633ee4d317b08520b6c3ba9
            }
        });
    }
     
    function hook_main_evpBytesToKey() {
        let base = Module.findBaseAddress("libsojm.so");
     
        Interceptor.attach(base.add(0x100C38), {
            onEnter: function (args) {
                console.error(`evpBytesToKey.init.IV: ${args[1].readCString()}`);  // IV: iGSkJXWcEwNYZnvt 这个是随机生成的
            }, onLeave: function (retval) {
                console.error(`evpBytesToKey.init.Key ${ab2Hex(retval.readByteArray(32))}`);  // 最终 key
            }
        });
    }
     
    // 日志
    key3: fc9e9a75e633ee4d317b08520b6c3ba9
    evpBytesToKey.init.IV: tOVJKhoOojtjnqxB
    evpBytesToKey.init.Key [A7 91 4B 68 21 10 16 5D DB 0B A7 CF 4E AC BE FC 1C C2 1F 56 F3 41 B8 2A 89 0A CA ED 2C C3 4A 75]
    
    ```
    
*   关于 IV 的处理
    
    *   生成随机 IV `main_opensslRandomPseudoBytes` 生成指定长度的伪随机字节序列
    *   main_evpBytesToKey 初始化向量 IV

### crypto_cipher_newCFB

*   还原符号就是爽！！！，这里可以直接看到算法名以及使用的模式
    
    ```
    // 这里有些参数没反编译出来，要么手动识别识别一下（进入函数在返回，F5），要么看汇编代码
    v19 = *(crypto_cipher_newCFB(v35, v36, v46, v42, v43, 0) + 16);  // AES_CFB 初始化，返回一个 Stream 使用它进行加密
    v37 = a2;
    v19();  // ⭐开始加密  偏移CC0D8 crypto_cipher__cfb_XORKeyStream | XORKeyStream(&密文, 明文)
     
    // Go 中原型
    // stream := cipher.NewCFBEncrypter(block, iv)
    // stream.XORKeyStream(ciphertext[aes.BlockSize:], plaintext)
    
    ```
    
    ```
    .text:000FF744 45 33 FF EB                   BL              crypto_cipher_newCFB    ; 这里调用完方法后返回
    .text:000FF744
    .text:000FF748 1C 00 9D E5                   LDR             R0, [SP,#0x58+var_3C]
    .text:000FF74C 10 00 90 E5                   LDR             R0, [R0,#0x10]          ; 可以看到这里对 R0 做了一些处理，下面就是一些参数的准备
    .text:000FF750 20 10 9D E5                   LDR             R1, [SP,#0x58+var_38]
    .text:000FF754 04 10 8D E5                   STR             R1, [SP,#0x58+var_54]
    .text:000FF758 50 10 9D E5                   LDR             R1, [SP,#0x58+var_8]
    .text:000FF75C 08 10 8D E5                   STR             R1, [SP,#0x58+var_50]
    .text:000FF760 60 20 9D E5                   LDR             R2, [SP,#0x58+data_len]
    .text:000FF764 0C 20 8D E5                   STR             R2, [SP,#0x58+var_4C]
    .text:000FF768 10 20 8D E5                   STR             R2, [SP,#0x58+var_48]
    .text:000FF76C 5C 30 9D E5                   LDR             R3, [SP,#0x58+data]
    .text:000FF770 14 30 8D E5                   STR             R3, [SP,#0x58+var_44]
    .text:000FF774 18 20 8D E5                   STR             R2, [SP,#0x58+var_40]
    .text:000FF778 64 30 9D E5                   LDR             R3, [SP,#0x58+data_cap]
    .text:000FF77C 1C 30 8D E5                   STR             R3, [SP,#0x58+var_3C]
    .text:000FF780 30 FF 2F E1                   BLX             R0                       ; 这里是 v19() 调用函数，所以 hook 这里得到 R0 地址减去基址就是我们的偏移
    
    ```
    
    ```
    function hook_0xFF780() {
        let base = Module.findBaseAddress("libsojm.so");
     
        Interceptor.attach(base.add(0xFF780), {
            onEnter(args) {
                console.log(`call 加密函数地址偏移：${ptr(this.context.r0 - base)}`);  // 容易崩溃，hook 一次崩溃一次，但是能拿到我们要的值
            }
        });
    }
     
    // call 加密函数地址偏移：0xcc0d8
    
    ```
    
    ![](https://bbs.kanxue.com/upload/attach/202303/918604_WUK7UBHPCKNJDMZ.png)
    

### cfb_XORKeyStream

```
// stream.XORKeyStream(ciphertext[aes.BlockSize:], plaintext) 这是 Go 的原型
// 根据汇编分析这里其实传了四个参数，第一个参数结束后才是我们的密文
 
function hook_crypto_cipher__cfb_XORKeyStream() {
    let base = Module.findBaseAddress("libsojm.so");
 
    Interceptor.attach(base.add(0xCC0D8), {
        onEnter: function (args) {
            this.arg1 = args[1]  // 密文
            this.arg2 = args[2]  // 密文长度
        }, onLeave: function (retval) {
            console.error(`cfb_XORKeyStream 密文 ${ab2Hex(this.arg1.readByteArray(this.arg2.toInt32()))}`);
        }
    });
}
 
// evpBytesToKey.init.IV: BGrtHIdEKMvPYAEt
// evpBytesToKey.init.Key [A7 91 4B 68 21 10 16 5D DB 0B A7 CF 4E AC BE FC 1C C2 1F 56 F3 41 B8 2A 89 0A CA ED 2C C3 4A 75]
// cfb_XORKeyStream 密文 [14 99 4C 88 AE 68 4F 59 0C 09 5B 26 14 AD 8B 72]

```

写了一个 Go 的 AES_CFB 进行验证，验证结果是一致的，这里还剩下最后一步

```
package main
 
import (
    "crypto/aes"
    "crypto/cipher"
    "encoding/hex"
    "fmt"
)
 
func main() {
    data := []byte("abcdef0123456789")
    data_len := len(data)
    enc_text := make([]byte, data_len)
 
    iv := []byte("BGrtHIdEKMvPYAEt")
    key, _ := hex.DecodeString("a7914b682110165ddb0ba7cf4eacbefc1cc21f56f341b82a890acaed2cc34a75")
    newCipher, err := aes.NewCipher(key)
    if err != nil {
        fmt.Println(err)
        return
    }
 
    cfbEncrypter := cipher.NewCFBEncrypter(newCipher, iv)
    cfbEncrypter.XORKeyStream(enc_text, data)
    fmt.Println(hex.EncodeToString(enc_text)) // 14994c88ae684f590c095b2614ad8b72
}

```

### 密文映射

```
while ( v22 < v41 )  // 循环对密文进行映射，也就是说对密文进行二次处理
{
  v25 = *(v44 + v22);
  if ( v23 >= 2 * v41 )
    runtime_panicIndex();
  *(v21 + v23) = a0123456789abcd_0[v25 >> 4];  // a0123456789abcd_0 => "0123456789abcdef" 字符串映射
  v26 = a0123456789abcd_0[v25 & 0xF];
  if ( v23 + 1 >= 2 * v41 )
    runtime_panicIndex();
  *(v21 + v23 + 1) = v26;
  ++v22;
  v23 += 2;
}
 
------------------------------------------------------------------------------------
 
var str = "0123456789abcdef"
for (let i = 0; i < cipher.len; i++) {
    var tmp = cipher[i]
    cipher[i << 1] = str[tmp >> 4]
    cipher[i << 1 ^ 1] = str[tmp & 0xf]
    // cipher[i * 2 ^ 1] = str[tmp & f]  // 等同上一句代码
}

```

### Golang 明文测试 abcdef0123456789

```
package main
 
import (
    "crypto/aes"
    "crypto/cipher"
    "encoding/hex"
    "fmt"
)
 
func main() {
    data := []byte("abcdef0123456789")
    dataLen := len(data)
    ciphertextBytes := make([]byte, dataLen)
 
    iv := []byte("eoyEeRxfiXqDImeQ")
    key, _ := hex.DecodeString("a7914b682110165ddb0ba7cf4eacbefc1cc21f56f341b82a890acaed2cc34a75")
    newCipher, err := aes.NewCipher(key)
    if err != nil {
        fmt.Println(err)
        return
    }
 
    cfbEncrypter := cipher.NewCFBEncrypter(newCipher, iv)
    cfbEncrypter.XORKeyStream(ciphertextBytes, data)
    ciphertextStr := hex.EncodeToString(ciphertextBytes)
    fmt.Println("AES_CFB 密文:", ciphertextStr)
 
    newCiphertextBytes := make([]byte, len(ciphertextBytes)*2)
    mapStr := "0123456789abcdef"
    for i := 0; i < len(iv); i++ {
        tmp := iv[i]
        newCiphertextBytes[i<<1] = mapStr[tmp>>4]
        newCiphertextBytes[i<<1^1] = mapStr[tmp&0xf]
    }
    fmt.Println("最终密文结果:", string(newCiphertextBytes)+ciphertextStr)
    fmt.Println("最终密文结果:", hex.EncodeToString(iv)+ciphertextStr)
    // AES_CFB 密文: 9368e37e4c81bf11ec0b3c14e5d16f45
    // 最终密文结果: 656f79456552786669587144496d65519368e37e4c81bf11ec0b3c14e5d16f45
    // 最终密文结果: 656f79456552786669587144496d65519368e37e4c81bf11ec0b3c14e5d16f45
 
 
    // hook 输出结果
    // onEnter main_oldEncrypt arg0:0xc arg1:0x86990280 arg2:0x10 arg3:0x10 arg4:0xcfc046f0 arg5:0x86990280 arg6:0x10 arg7:0x10 arg8:0x8698e8d4 arg9:0x20 arg10:0x2c arg11:0x8698e8f4
    // onEnter main_oldEncrypt arg1:abcdef0123456789'a'
    // onEnter main_oldEncrypt arg8:4c7e?2fab4f4>1>2>5d0ee4bb61d7b03mIZUjjghGd
    // onEnter main_oldEncrypt arg11:mIZUjjghGd
    // onEnter main_oldEncrypt arg0:0x0 arg1:0x868023dc arg2:0x86918000 arg3:0x57 arg4:0xcfc046f0 arg5:0x86990280 arg6:0x10 arg7:0x10 arg8:0x8698e8d4 arg9:0x20 arg10:0x2c arg11:0x8698e8f4
    // onEnter main_oldEncrypt arg1:
    // onEnter main_oldEncrypt arg8:4c7e?2fab4f4>1>2>5d0ee4bb61d7b03mIZUjjghGd
    // onEnter main_oldEncrypt arg11:mIZUjjghGd
    // key3: fc9e9a75e633ee4d317b08520b6c3ba9
    // evpBytesToKey.init.IV: eoyEeRxfiXqDImeQ
    // evpBytesToKey.init.Key [A7 91 4B 68 21 10 16 5D DB 0B A7 CF 4E AC BE FC 1C C2 1F 56 F3 41 B8 2A 89 0A CA ED 2C C3 4A 75]
    // cfb_XORKeyStream 密文 [93 68 E3 7E 4C 81 BF 11 EC 0B 3C 14 E5 D1 6F 45]
    // onLeave main_oldEncrypt 656f79456552786669587144496d65519368e37e4c81bf11ec0b3c14e5d16f45
    // ------------------------------------------------------------
    // onLeave main_oldEncrypt 656f79456552786669587144496d65519368e37e4c81bf11ec0b3c14e5d16f45
    // ------------------------------------------------------------
    // encrypt(abcdef0123456789, "BwcnBzRjN2U/MmZhYjRmND4xPjI+NWQwZWU0YmI2MWQ3YjAzKw8cEywsIS4BIg==") :=> 656f79456552786669587144496d65519368e37e4c81bf11ec0b3c14e5d16f45
}

```

### Golang 明文测试 JSON 数据

```
package main
 
import (
    "crypto/aes"
    "crypto/cipher"
    "encoding/hex"
    "fmt"
)
 
func main() {
    data := []byte("{\"__package_hash__\":\"678c7f3bf3584a2079295d8834928146\",\"__package_name__\":\"tv.iytqy.cvhaca\",\"app_status\":\"9A5C6BDC62AD1CFE45A6578F84E858F9CA1A4F76:2\",\"bundle_id\":\"tv.iytqy.cvhaca\",\"new_player\":\"fx\",\"system_app_type\":\"local\",\"system_build_aff\":\"\",\"system_build_id\":\"a1000\",\"system_iid\":\"923868eb5f543afa55f1f33cfac37d35\",\"system_oauth_id\":\"02274477773a9ac8a2cf3605400ece4b\",\"system_oauth_type\":\"android\",\"system_token\":\"023FAC3AFE2285DC98E50CA4C638E0845277DC8CB62CCCC088DA1E3CB679180C97F2688C0DBE197F03950D84900C2A6B050334216C4CC604021DCC624543AFDDEF8D19363E43F9C0B8EEC79BBF269E7DE77B0304FF1034FED885ACA4C74DAC768D26D9FADD\",\"system_version\":\"5.7.1\"}")
    dataLen := len(data)
    ciphertextBytes := make([]byte, dataLen)
 
    iv := []byte("oLePKyUruTFvtvcL")
    key, _ := hex.DecodeString("a7914b682110165ddb0ba7cf4eacbefc1cc21f56f341b82a890acaed2cc34a75")
    newCipher, err := aes.NewCipher(key)
    if err != nil {
        fmt.Println(err)
        return
    }
 
    cfbEncrypter := cipher.NewCFBEncrypter(newCipher, iv)
    cfbEncrypter.XORKeyStream(ciphertextBytes, data)
    ciphertextStr := hex.EncodeToString(ciphertextBytes)
    fmt.Println("AES_CFB 密文:", ciphertextStr)
 
    newCiphertextBytes := make([]byte, len(ciphertextBytes)*2)
    mapStr := "0123456789abcdef"
    for i := 0; i < len(iv); i++ {
        tmp := iv[i]
        newCiphertextBytes[i<<1] = mapStr[tmp>>4]
        newCiphertextBytes[i<<1^1] = mapStr[tmp&0xf]
    }
    fmt.Println("最终密文结果:", string(newCiphertextBytes)+ciphertextStr)
    fmt.Println("最终密文结果:", hex.EncodeToString(iv)+ciphertextStr)
    fmt.Printf("%x", newCiphertextBytes)
    // AES_CFB 密文: 912ed4bfa0ff60018fc53d6307c81ca479d73d95f5e6439e8dfdfb3182f8f14715095a596a187278ecdf0ff2efa6334a8d557d20375750cf1787b7f2f677f8359f38e4ff1b9c89c448434a4dc2c5af4b7eca3a43a088665e45cb93317f9022fee0771937ce5c922007764f8c30c921e2013f286af14c455a986653221f6a9ec6d1ab7aa0e1030c7d238fba47956772311abb99cf278c06e6b93b9f78b4bfa06233eb8a3f339e89c7b1b698921aed21383820af0ed0805cf357c78da8a4f802f7cefe09444c926ee747fa10acca3758f3a8d420a46c8f0397f730053aeb02d4cf31b2d2f600f719483e57252967bc341952319308fa409b23e993743cb3acd1e73c37dd169f0a11198b5fb01fa09106f9059b18a8293ce67f95a9b4a3caeaf37b037cd3771d02c8b86de7e443f70304c48c774fa48f6f21c3fb9fa94a4624cc101d25d79719bbf31fa04f3f914e95d027e1bd31980dbb9cc08f7421a4c75ab150857ef7d75c7807224ec3d9284aa15aa24479090e6710d1f50a8f1aaad1c4b5b7b2370dc483ed9d149df218ad2c6017c1fddca45904897d87d59393afe3d629e305c215f77add26eb121610cb778fdadfe7d298bb65ca72cdfe04299cf962fd955d734fe4d75299e755a5234b2037cf259b7aadcd9e8aabd89a52600ed8cd45e338b2a915086bdb97fcbee80ab747ba3a9efaefa0a7cbcc1da0cd2ff58509a0dd1a33c1148495b8f8a18ebf28d33bf1c919f91f79c6a4ae8eb1798ed66312401b7970d95b202b4d2c7cf22e8148417598522dd896c6e05b235f6e6a9e0ebcdac1551d4c28fd7fa371bf75f4705d32b50a54702e73dca71160b354e46151f4545f0acd811d0687e8bc94e0f489cc1ce16dd8bf33cd36c51d1a095411a1383bfbb5cce26ae24c0c4c73
    //最终密文结果: 6f4c65504b795572755446767476634c                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                912ed4bfa0ff60018fc53d6307c81ca479d73d95f5e6439e8dfdfb3182f8f14715095a596a187278ecdf0ff2efa6334a8d557d20375750cf1787b7f2f677f8359f38e4ff1b9c89c448434a4dc2c5af4b7eca3a43a088665e45cb93317f9022fee0771937ce5c922007764f8c30c921e2013f286af14c455a986653221f6a9ec6d1ab7aa0e1030c7d238fba47956772311abb99cf278c06e6b93b9f78b4bfa06233eb8a3f339e89c7b1b698921aed21383820af0ed0805cf357c78da8a4f802f7cefe09444c926ee747fa10acca3758f3a8d420a46c8f0397f730053aeb02d4cf31b2d2f600f719483e57252967bc341952319308fa409b23e993743cb3acd1e73c37dd169f0a11198b5fb01fa09106f9059b18a8293ce67f95a9b4a3caeaf37b037cd3771d02c8b86de7e443f70304c48c774fa48f6f21c3fb9fa94a4624cc101d25d79719bbf31fa04f3f914e95d027e1bd31980dbb9cc08f7421a4c75ab150857ef7d75c7807224ec3d9284aa15aa24479090e6710d1f50a8f1aaad1c4b5b7b2370dc483ed9d149df218ad2c6017c1fddca45904897d87d59393afe3d629e305c215f77add26eb121610cb778fdadfe7d298bb65ca72cdfe04299cf962fd955d734fe4d75299e755a5234b2037cf259b7aadcd9e8aabd89a52600ed8cd45e338b2a915086bdb97fcbee80ab747ba3a9efaefa0a7cbcc1da0cd2ff58509a0dd1a33c1148495b8f8a18ebf28d33bf1c919f91f79c6a4ae8eb1798ed66312401b7970d95b202b4d2c7cf22e8148417598522dd896c6e05b235f6e6a9e0ebcdac1551d4c28fd7fa371bf75f4705d32b50a54702e73dca71160b354e46151f4545f0acd811d0687e8bc94e0f489cc1ce16dd8bf33cd36c51d1a095411a1383bfbb5cce26ae24c0c4c73
    //最终密文结果: 6f4c65504b795572755446767476634c912ed4bfa0ff60018fc53d6307c81ca479d73d95f5e6439e8dfdfb3182f8f14715095a596a187278ecdf0ff2efa6334a8d557d20375750cf1787b7f2f677f8359f38e4ff1b9c89c448434a4dc2c5af4b7eca3a43a088665e45cb93317f9022fee0771937ce5c922007764f8c30c921e2013f286af14c455a986653221f6a9ec6d1ab7aa0e1030c7d238fba47956772311abb99cf278c06e6b93b9f78b4bfa06233eb8a3f339e89c7b1b698921aed21383820af0ed0805cf357c78da8a4f802f7cefe09444c926ee747fa10acca3758f3a8d420a46c8f0397f730053aeb02d4cf31b2d2f600f719483e57252967bc341952319308fa409b23e993743cb3acd1e73c37dd169f0a11198b5fb01fa09106f9059b18a8293ce67f95a9b4a3caeaf37b037cd3771d02c8b86de7e443f70304c48c774fa48f6f21c3fb9fa94a4624cc101d25d79719bbf31fa04f3f914e95d027e1bd31980dbb9cc08f7421a4c75ab150857ef7d75c7807224ec3d9284aa15aa24479090e6710d1f50a8f1aaad1c4b5b7b2370dc483ed9d149df218ad2c6017c1fddca45904897d87d59393afe3d629e305c215f77add26eb121610cb778fdadfe7d298bb65ca72cdfe04299cf962fd955d734fe4d75299e755a5234b2037cf259b7aadcd9e8aabd89a52600ed8cd45e338b2a915086bdb97fcbee80ab747ba3a9efaefa0a7cbcc1da0cd2ff58509a0dd1a33c1148495b8f8a18ebf28d33bf1c919f91f79c6a4ae8eb1798ed66312401b7970d95b202b4d2c7cf22e8148417598522dd896c6e05b235f6e6a9e0ebcdac1551d4c28fd7fa371bf75f4705d32b50a54702e73dca71160b354e46151f4545f0acd811d0687e8bc94e0f489cc1ce16dd8bf33cd36c51d1a095411a1383bfbb5cce26ae24c0c4c73
 
    // hook 输出结果
    // onEnter main_oldEncrypt arg0:0xc arg1:0x8697f600 arg2:0x288 arg3:0x2c0 arg4:0xcfc046f0 arg5:0x8697f600 arg6:0x288 arg7:0x2c0 arg8:0x8698e9f4 arg9:0x20 arg10:0x2c arg11:0x8698ea14
    // onEnter main_oldEncrypt arg1:{"__package_hash__":"678c7f3bf3584a2079295d8834928146","__package_name__":"tv.iytqy.cvhaca","app_status":"9A5C6BDC62AD1CFE45A6578F84E858F9CA1A4F76:2","bundle_id":"tv.iytqy.cvhaca","new_player":"fx","system_app_type":"local","system_build_aff":"","system_build_id":"a1000","system_iid":"923868eb5f543afa55f1f33cfac37d35","system_oauth_id":"02274477773a9ac8a2cf3605400ece4b","system_oauth_type":"android","system_token":"023FAC3AFE2285DC98E50CA4C638E0845277DC8CB62CCCC088DA1E3CB679180C97F2688C0DBE197F03950D84900C2A6B050334216C4CC604021DCC624543AFDDEF8D19363E43F9C0B8EEC79BBF269E7DE77B0304FF1034FED885ACA4C74DAC768D26D9FADD","system_version":"5.7.1"}
    // onEnter main_oldEncrypt arg8:4c7e?2fab4f4>1>2>5d0ee4bb61d7b03mIZUjjghGd
    // onEnter main_oldEncrypt arg11:mIZUjjghGd
    // key3: fc9e9a75e633ee4d317b08520b6c3ba9
    // evpBytesToKey.init.IV: oLePKyUruTFvtvcL
    // evpBytesToKey.init.Key [A7 91 4B 68 21 10 16 5D DB 0B A7 CF 4E AC BE FC 1C C2 1F 56 F3 41 B8 2A 89 0A CA ED 2C C3 4A 75]
    // cfb_XORKeyStream 密文 [91 2E D4 BF A0 FF 60 01 8F C5 3D 63 07 C8 1C A4 79 D7 3D 95 F5 E6 43 9E 8D FD FB 31 82 F8 F1 47 15 09 5A 59 6A 18 72 78 EC DF 0F F2 EF A6 33 4A 8D 55 7D 20 37 57 50 CF 17 87 B7 F2 F6 77 F8 35 9F 38 E4 FF 1B 9C 89 C4 48 43 4A 4D C2 C5 AF 4B 7E CA 3A 43 A0 88 66 5E 45 CB 93 31 7F 90 22 FE E0 77 19 37 CE 5C 92 20 07 76 4F 8C 30 C9 21 E2 01 3F 28 6A F1 4C 45 5A 98 66 53 22 1F 6A 9E C6 D1 AB 7A A0 E1 03 0C 7D 23 8F BA 47 95 67 72 31 1A BB 99 CF 27 8C 06 E6 B9 3B 9F 78 B4 BF A0 62 33 EB 8A 3F 33 9E 89 C7 B1 B6 98 92 1A ED 21 38 38 20 AF 0E D0 80 5C F3 57 C7 8D A8 A4 F8 02 F7 CE FE 09 44 4C 92 6E E7 47 FA 10 AC CA 37 58 F3 A8 D4 20 A4 6C 8F 03 97 F7 30 05 3A EB 02 D4 CF 31 B2 D2 F6 00 F7 19 48 3E 57 25 29 67 BC 34 19 52 31 93 08 FA 40 9B 23 E9 93 74 3C B3 AC D1 E7 3C 37 DD 16 9F 0A 11 19 8B 5F B0 1F A0 91 06 F9 05 9B 18 A8 29 3C E6 7F 95 A9 B4 A3 CA EA F3 7B 03 7C D3 77 1D 02 C8 B8 6D E7 E4 43 F7 03 04 C4 8C 77 4F A4 8F 6F 21 C3 FB 9F A9 4A 46 24 CC 10 1D 25 D7 97 19 BB F3 1F A0 4F 3F 91 4E 95 D0 27 E1 BD 31 98 0D BB 9C C0 8F 74 21 A4 C7 5A B1 50 85 7E F7 D7 5C 78 07 22 4E C3 D9 28 4A A1 5A A2 44 79 09 0E 67 10 D1 F5 0A 8F 1A AA D1 C4 B5 B7 B2 37 0D C4 83 ED 9D 14 9D F2 18 AD 2C 60 17 C1 FD DC A4 59 04 89 7D 87 D5 93 93 AF E3 D6 29 E3 05 C2 15 F7 7A DD 26 EB 12 16 10 CB 77 8F DA DF E7 D2 98 BB 65 CA 72 CD FE 04 29 9C F9 62 FD 95 5D 73 4F E4 D7 52 99 E7 55 A5 23 4B 20 37 CF 25 9B 7A AD CD 9E 8A AB D8 9A 52 60 0E D8 CD 45 E3 38 B2 A9 15 08 6B DB 97 FC BE E8 0A B7 47 BA 3A 9E FA EF A0 A7 CB CC 1D A0 CD 2F F5 85 09 A0 DD 1A 33 C1 14 84 95 B8 F8 A1 8E BF 28 D3 3B F1 C9 19 F9 1F 79 C6 A4 AE 8E B1 79 8E D6 63 12 40 1B 79 70 D9 5B 20 2B 4D 2C 7C F2 2E 81 48 41 75 98 52 2D D8 96 C6 E0 5B 23 5F 6E 6A 9E 0E BC DA C1 55 1D 4C 28 FD 7F A3 71 BF 75 F4 70 5D 32 B5 0A 54 70 2E 73 DC A7 11 60 B3 54 E4 61 51 F4 54 5F 0A CD 81 1D 06 87 E8 BC 94 E0 F4 89 CC 1C E1 6D D8 BF 33 CD 36 C5 1D 1A 09 54 11 A1 38 3B FB B5 CC E2 6A E2 4C 0C 4C 73]
    // onLeave main_oldEncrypt 6f4c65504b795572755446767476634c912ed4bfa0ff60018fc53d6307c81ca479d73d95f5e6439e8dfdfb3182f8f14715095a596a187278ecdf0ff2efa6334a8d557d20375750cf1787b7f2f677f8359f38e4ff1b9c89c448434a4dc2c5af4b7eca3a43a088665e45cb93317f9022fee0771937ce5c922007764f8c30c921e2013f286af14c455a986653221f6a9ec6d1ab7aa0e1030c7d238fba47956772311abb99cf278c06e6b93b9f78b4bfa06233eb8a3f339e89c7b1b698921aed21383820af0ed0805cf357c78da8a4f802f7cefe09444c926ee747fa10acca3758f3a8d420a46c8f0397f730053aeb02d4cf31b2d2f600f719483e57252967bc341952319308fa409b23e993743cb3acd1e73c37dd169f0a11198b5fb01fa09106f9059b18a8293ce67f95a9b4a3caeaf37b037cd3771d02c8b86de7e443f70304c48c774fa48f6f21c3fb9fa94a4624cc101d25d79719bbf31fa04f3f914e95d027e1bd31980dbb9cc08f7421a4c75ab150857ef7d75c7807224ec3d9284aa15aa24479090e6710d1f50a8f1aaad1c4b5b7b2370dc483ed9d149df218ad2c6017c1fddca45904897d87d59393afe3d629e305c215f77add26eb121610cb778fdadfe7d298bb65ca72cdfe04299cf962fd955d734fe4d75299e755a5234b2037cf259b7aadcd9e8aabd89a52600ed8cd45e338b2a915086bdb97fcbee80ab747ba3a9efaefa0a7cbcc1da0cd2ff58509a0dd1a33c1148495b8f8a18ebf28d33bf1c919f91f79c6a4ae8eb1798ed66312401b7970d95b202b4d2c7cf22e8148417598522dd896c6e05b235f6e6a9e0ebcdac1551d4c28fd7fa371bf75f4705d32b50a54702e73dca71160b354e46151f4545f0acd811d0687e8bc94e0f489cc1ce16dd8bf33cd36c51d1a095411a1383bfbb5cce26ae24c0c4c73
    // ------------------------------------------------------------
    // encrypt({"system_build_id":"a1000","system_iid":"923868eb5f543afa55f1f33cfac37d35","app_status":"9A5C6BDC62AD1CFE45A6578F84E858F9CA1A4F76:2","system_version":"5.7.1","system_build_aff":"","bundle_id":"tv.iytqy.cvhaca","system_app_type":"local","new_player":"fx","system_oauth_id":"02274477773a9ac8a2cf3605400ece4b","system_oauth_type":"android","system_token":"023FAC3AFE2285DC98E50CA4C638E0845277DC8CB62CCCC088DA1E3CB679180C97F2688C0DBE197F03950D84900C2A6B050334216C4CC604021DCC624543AFDDEF8D19363E43F9C0B8EEC79BBF269E7DE77B0304FF1034FED885ACA4C74DAC768D26D9FADD"}, "BwcnBzRjN2U/MmZhYjRmND4xPjI+NWQwZWU0YmI2MWQ3YjAzKw8cEywsIS4BIg==") :=> 6f4c65504b795572755446767476634c912ed4bfa0ff60018fc53d6307c81ca479d73d95f5e6439e8dfdfb3182f8f14715095a596a187278ecdf0ff2efa6334a8d557d20375750cf1787b7f2f677f8359f38e4ff1b9c89c448434a4dc2c5af4b7eca3a43a088665e45cb93317f9022fee0771937ce5c922007764f8c30c921e2013f286af14c455a986653221f6a9ec6d1ab7aa0e1030c7d238fba47956772311abb99cf278c06e6b93b9f78b4bfa06233eb8a3f339e89c7b1b698921aed21383820af0ed0805cf357c78da8a4f802f7cefe09444c926ee747fa10acca3758f3a8d420a46c8f0397f730053aeb02d4cf31b2d2f600f719483e57252967bc341952319308fa409b23e993743cb3acd1e73c37dd169f0a11198b5fb01fa09106f9059b18a8293ce67f95a9b4a3caeaf37b037cd3771d02c8b86de7e443f70304c48c774fa48f6f21c3fb9fa94a4624cc101d25d79719bbf31fa04f3f914e95d027e1bd31980dbb9cc08f7421a4c75ab150857ef7d75c7807224ec3d9284aa15aa24479090e6710d1f50a8f1aaad1c4b5b7b2370dc483ed9d149df218ad2c6017c1fddca45904897d87d59393afe3d629e305c215f77add26eb121610cb778fdadfe7d298bb65ca72cdfe04299cf962fd955d734fe4d75299e755a5234b2037cf259b7aadcd9e8aabd89a52600ed8cd45e338b2a915086bdb97fcbee80ab747ba3a9efaefa0a7cbcc1da0cd2ff58509a0dd1a33c1148495b8f8a18ebf28d33bf1c919f91f79c6a4ae8eb1798ed66312401b7970d95b202b4d2c7cf22e8148417598522dd896c6e05b235f6e6a9e0ebcdac1551d4c28fd7fa371bf75f4705d32b50a54702e73dca71160b354e46151f4545f0acd811d0687e8bc94e0f489cc1ce16dd8bf33cd36c51d1a095411a1383bfbb5cce26ae24c0c4c73
}

```

### 小结

*   做完 Go 的算法测试才发现最后那段 for 循环就是把 key 转为 hexStr，然后拼接到密文提交
*   因为我们是随机生成的 IV，key 是固定处理的，如果服务器想要解密我们的数据，客户端生成的随机 IV 是必定要提交给服务端的，所以这里的情况也是比较合理的
*   所以在 Go 里，`hex.EncodeToString(iv)` 与最后那段 for 的处理循环效果是一致的

流程梳理
----

1.  传入明文数据 json_data 与 base64_key
2.  对明文进行处理添加 **`package_name`** 与 **`package_hash`（**`"package_hash":"678c7f3bf3584a2079295d8834928146","package_name":"tv.iytqy.cvhaca"` Hash APK Filse MD5
3.  对 key 解码进行处理
    
    ```
    # 源         ：BwcnBzRjN2U/MmZhYjRmND4xPjI+NWQwZWU0YmI2MWQ3YjAzKw8cEywsIS4BIg==
    # 解码       ：␇␇'␇4c7e?2fab4f4>1>2>5d0ee4bb61d7b03+␏␜␓,,!.␁"
    # 第一次处理 : 4c7e?2fab4f4>1>2>5d0ee4bb61d7b03mIZUjjghGd / mIZUjjghGd（分成两个字符串 main_parsePassphrase offset：0x1011F8
    # 第二次处理 : （在函数开头的 for 循环，一顿异或操作 main_oldEncrypt offset：0xFF5EC
    # 第三次处理 : fc9e9a75e633ee4d317b08520b6c3ba9 （交互字节 main_swapByteLocation offset：0xFF648
    # 第四次处理 : [A7 91 4B 68 21 10 16 5D DB 0B A7 CF 4E AC BE FC 1C C2 1F 56 F3 41 B8 2A 89 0A CA ED 2C C3 4A 75]（main_evpBytesToKey offset：0xFF6B0
     
    因为处理中间都没有涉及到随机值以及其他相关数据，只要这里传入的 PWD 也就是 base64_key 不变，直接拿第四次处理后的值即可
    
    ```
    
4.  生成 16 位随机 IV `opensslRandomPseudoBytes`
    
5.  使用 Go 中的 AES_CFB 进行加密，似乎没看到填充环节
6.  把 IV 转 HexStr 拼接到 AES_CFB 密文前，这样提交给服务器，服务器就可以拿前 32 位当解密 IV 进行数据解密

Python 算法复现
-----------

```
from Crypto.Cipher import AES
import binascii
 
def aes_encrypt(data):
    iv = 'eoyEeRxfiXqDImeQ'.encode()
    key = bytes.fromhex('a7914b682110165ddb0ba7cf4eacbefc1cc21f56f341b82a890acaed2cc34a75')
    cipher = AES.new(key, AES.MODE_CFB, iv, segment_size=AES.block_size * 8)  # Python 实现多加一个 segment_size
    ciphertext_bytes = cipher.encrypt(data)
    ciphertext_hexStr = bytes.hex(ciphertext_bytes)
    print("AES_CFB 密文:", ciphertext_hexStr)
    ret = bytes.hex(iv) + ciphertext_hexStr
    print("提交结果:", ret)
    return ret
 
def aes_decrypt(ciphertext):
    iv = bytes.fromhex(ciphertext[:32])
    ciphertext = bytes.fromhex(ciphertext[32:])
    print("IV:", iv.decode())
    print("待解密密文:", ciphertext)
    key = bytes.fromhex('a7914b682110165ddb0ba7cf4eacbefc1cc21f56f341b82a890acaed2cc34a75')
    cipher = AES.new(key, AES.MODE_CFB, iv, segment_size=AES.block_size * 8)  # Python 实现多加一个 segment_size
    plaintext = cipher.decrypt(ciphertext).decode()
    print("最终明文:", plaintext)
 
if __name__ == '__main__':
    data = b'abcdef0123456789'
    aes_decrypt(aes_encrypt(data))
 
-----------------------------------------------------------------------------------------------
# log日志
# AES_CFB 密文: 9368e37e4c81bf11ec0b3c14e5d16f45
# 提交结果: 656f79456552786669587144496d65519368e37e4c81bf11ec0b3c14e5d16f45
# IV: eoyEeRxfiXqDImeQ
# 待解密密文: b'\x93h\xe3~L\x81\xbf\x11\xec\x0b<\x14\xe5\xd1oE'
# 最终明文: abcdef0123456789

```

抓包解密数据尝试
--------

![](https://bbs.kanxue.com/upload/attach/202303/918604_B73T63MR9RRX9PS.png)

Refers
------

1.  [深度解密 Go 语言中字符串的使用](https://www.leyeah.com/article/deeply-decrypt-use-strings-go-language-696034)
2.  [go_parser: Yet Another Golang binary parser for IDAPro](https://github.com/0xjiayu/go_parser)⭐
3.  [xx 度灰 app 加密算法分析还原](https://www.52pojie.cn/thread-1691013-1-1.html)⭐
4.  [内部机制 - Go 语言高级编程](https://chai2010.cn/advanced-go-programming-book/ch2-cgo/ch2-05-internal.html)⭐
5.  [cgocall.go - Go](https://cs.opensource.google/go/go/+/master:src/runtime/cgocall.go;l=124)
6.  [frida 详细分析某 gojni 协议](https://www.52pojie.cn/forum.php?mod=viewthread&tid=1698434&extra=&page=1)
7.  [聊一聊 goroutine stack](https://zhuanlan.zhihu.com/p/28409657) `morestack_noctxt`
8.  [分析 Golang 可执行文件 – JEB 的实际应用](https://www.pnfsoftware.com/blog/analyzing-golang-executables/)
9.  [GO 恶意样本实例分析](https://bbs.kanxue.com/thread-264344.htm) ⭐
10.  [无符号 Golang 程序逆向方法解析](https://www.anquanke.com/post/id/170332)⭐
11.  [Reversing GO binaries like a pro](https://rednaga.io/2016/09/21/reversing_go_binaries_like_a_pro/)
12.  [Go 二进制文件逆向分析从基础到进阶——综述](https://www.anquanke.com/post/id/214940)
13.  [Go 二进制文件逆向分析从基础到进阶——MetaInfo、函数符号和源码文件路径列表](https://www.anquanke.com/post/id/215419)
14.  [Go 二进制文件逆向分析从基础到进阶——数据类型](https://www.anquanke.com/post/id/215820)
15.  [Go 二进制文件逆向分析从基础到进阶——itab 与 strings](https://www.anquanke.com/post/id/218377)
16.  [Go 二进制文件逆向分析从基础到进阶——Tips 与实战案例](https://www.anquanke.com/post/id/218674)

总结
==

1.  Go 中的字符串都是放在一起的，所以偏移量和长度是很关键的，IDA 或 Jeb 某些地方识别也挺有问题的
2.  以后遇见这种 So 如果不能还原符号最好的方式就是动态调试比 FRIDA hook 可能更直观一些
3.  符号还原除了可以使用 IDA 脚本，还可以利用 **`bindiff / diaphora`** 进行对比（使用姿势可以参考 Refers 第 10 条
4.  关于调用约定这里就不多说了，基本都是 `usercall` 通过栈传递的，看汇编就知道啦~
5.  利用 ChatGPT 进行逆向辅助分析效率更高!!
6.  光看文章的话还是挺迷糊的，自己动手操作一遍影响会深刻一些

已在知识星球 "10 亿级应用的逆向分享" 原创首发

  

[冰蝎，蚁剑 Java 内存马查杀防御技术](https://www.kanxue.com/book-section_list-155.htm)

最后于 16 小时前 被. KK 编辑 ，原因： 添加标题 [#逆向分析](forum-161-1-118.htm) [#协议分析](forum-161-1-120.htm)

上传的附件：

*   [分析. 7z](javascript:void(0)) （3.17MB，4 次下载）
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-287015.htm)

> [原创] 某短视频网站滑块分析

某短视频网站滑块分析
==========

目标网站：aHR0cHM6Ly9jYXB0Y2hhLnp0Lmt1YWlzaG91LmNvbS9pZnJhbWUvaW5kZXguaHRtbA==

前言
--

最近刚好闲着没事，本来有工作要研究这个滑块，但是一搜基本都是补环境的，觉得不够优雅，就尝试手撕一下 JS VMP，也是好久没撕过了，就当练手了

加密分析
----

### 提交接口抓包

当滑块松手后会调用接口 kSecretApiVerify，传入 verifyParam 参数，这个参数是一个很长的 base64，这里我初步猜测是对称加密

### 加密定位

直接搜索 verifyParam 就可以定位到加密，这代码里面用了很多异步，所以看起来非常的恶心  
![](https://bbs.kanxue.com/upload/attach/202505/943408_BTGNHJBQUT53283.webp)  
最终我们可以定位到加密函数  
![](https://bbs.kanxue.com/upload/attach/202505/943408_KS23VFD2K824TQ3.webp)  
我解释一下这里，`i = a.a[r("0x34")](c)`是将 c 这个 object 进行 json -> urlformencode 的转换的，`o = s[r("0x2d")](h, i)`这一步是为了将 i 转为 Uint8Array，方便下一步进行加密，`s[r("0x2d")](x, o)`这里就是执行加密的函数了。这一步往里面跟，发现调用了一个所谓的 Jose 的库，这个就是加密的 VMP，这行代码还原后就是`Jose.$encrypt([uint8ArrayData, fixedUuid])`  
![](https://bbs.kanxue.com/upload/attach/202505/943408_V4SY94MQ7B8SY6E.webp)  
这里的 uint8ArrayData 就是 urlformencode 后的数据，fixedUuid 是个固定值，后面我们会讲它的作用，这里我们直接看`Jose.$encrypt`的实现

### JSVMP 分析

往里继续跟就会到一个`encrypt.js`文件中, 一眼就看出是 VMP 了，VM 的变量名没有被混淆  
![](https://bbs.kanxue.com/upload/attach/202505/943408_SFE2RQFQMHVQVFG.webp)  
看到这里我们可以直接祭出 Trace 了，找到一些关键的操作我们可以直接插桩

```
c.prototype.lnot = function(t) {
    return !t
}
,
c.prototype.not = function(t) {
    return ~t
}
,
c.prototype.inc = function(t) {
    return t + 1
}
,
c.prototype.dec = function(t) {
    return t - 1
}
,
c.prototype.add = function(t, n) {
    return n + t
}
,
c.prototype.sub = function(t, n) {
    return n - t
}
,
c.prototype.mul = function(t, n) {
    return n * t
}
,
c.prototype.div = function(t, n) {
    return n / t
}
,
c.prototype.mod = function(t, n) {
    return n % t
}
,
c.prototype.shl = function(t, n) {
    return n << t
}
,
c.prototype.sar = function(t, n) {
    return n >> t
}
,
c.prototype.shr = function(t, n) {
    return n >>> t
}
,
c.prototype.or = function(t, n) {
    return n | t
}
,
c.prototype.and = function(t, n) {
    return n & t
}
,
c.prototype.xor = function(t, n) {
    return n ^ t
}
,
c.prototype.ceq = function(t, n) {
    return n == t
}
,
```

向以上这些都是一些运算指令，因为我们要分析加密算法，所以将位操作全部插桩

```
new e("",(function(t, n, e, r) {
    var o = n.pop()
        , i = n.pop();
    return null == o ? g(t, new Error("Cannot read property '" + i + "' of " + o)) : n.push(r.get(o, i))
}
)), new e("",(function(t, n, e, r) {
    var o = n.pop()
        , i = n.pop()
        , u = n.pop();
    return null == o ? g(t, new Error("Cannot set property '" + i + "' of " + o)) : n.push(r.set(o, i, u))
}
```

以上是访问 object 的对象的指令，我们也进行插桩

```
function(t, n, e, r, o) {
    var i = t.evalStack
        , u = t.realm;
    if (null == r)
        return g(t, new Error("Cannot call method '" + e + "' of " + (void 0 === r ? "undefined" : "null")));
    var p = r.constructor.name || "Object";
    u = u.get(r, e);
    return u instanceof Function ? l(t, n, u, r, o) : null == u ? (i.pop(),
    g(t, new Error("Object #<" + p + "> has no method '" + e + "'"))) : (i.pop(),
    g(t, new Error("Property '" + e + "' of object #<" + p + "> is not a function")))
}
), l = function(t, n, e, r, o, i) {
    if ("function" != typeof e)
        return g(t, new Error("object is not a function"));
    for (var u = t.evalStack, p = t.fiber, s = t.realm, c = {
        length: n,
        callee: e
    }; n; )
        c[--n] = u.pop();
    r = r || s.global,
    c = Array.prototype.slice.call(c);
    try {
        var a = i ? m(e, c) : e.apply(r, c);
        if (!p.paused)
            return u.push(a)
    } catch (h) {
        g(t, h)
    }
}
```

以上是函数调用的指令，也可以进行插桩，方便我们分析，这里有个坑，我插桩了运算后就会导致浏览器卡死，所以我直接将代码抠到 NodeJS 里直接运行然后跑日志了

### 分析 Trace 日志

因为我在分析之前就查阅了一些文章，这个滑块上一个版本用的是 AES 的加密，所以我尝试在 tracelog 中搜索了很多 sbox 的魔数以及 rcon 的操作，想直接把密钥还原出来，但是都没找到，这里浪费了很多时间

#### 转换 verifyParam

这里我们可以直接将 verifyParam 转换为 Uint8Array, 找到日志中生成的部分。最后发现这个 verifyParam 的结构通过 concat 是拼接出来的，非常像一个私有协议的实现，前 64 个 byte 都是头数据

![](https://bbs.kanxue.com/upload/attach/202505/943408_4N5RUHMGK6ZW575.webp)

```
[加密算法] 获取属性操作 - 参数: { object: [Arguments] { '0': '00000000000000000003' }, property: 0 }
[加密算法] 获取属性操作 - 参数: { object: '00000000000000000003', property: 'length' }
[加密算法] 获取属性操作 - 参数: { object: '00000000000000000003', property: 'substr' }
[加密算法] 函数调用 - 参数: {
  function: [Function: substr],
  context: '00000000000000000003',
  args: [ 0, 2 ]
}
 
[加密算法] 函数调用结果: 3
[加密算法] 获取属性操作 - 参数: { object: '03', property: 'length' }
[加密算法] 获取属性操作 - 参数: { object: Object [Math] {}, property: 'pow' }
[加密算法] 函数调用 - 参数: {
  function: [Function: pow],
  context: Object [Math] {},
  args: [ 2, 8 ]
}
[加密算法] 函数调用结果: 256
[加密算法] 函数调用结果: 3
[加密算法] 获取属性操作 - 参数: {
  object: [
    0, 0, 0, 0, 0,
    0, 0, 0, 0
  ],
  property: 'push'
}
[加密算法] 函数调用 - 参数: {
  function: [Function: push],
  context: [
    0, 0, 0, 0, 0,
    0, 0, 0, 0
  ],
  args: [ 3 ]
}
[加密算法] 函数调用结果: 10
[加密算法] 获取属性操作 - 参数: { object: '00000000000000000003', property: 'length' }
[加密算法] 函数调用结果: [
  0, 0, 0, 0, 0,
  0, 0, 0, 0, 3
]
[加密算法] 获取属性操作 - 参数: { object: [ 222, 192, 173, 222, 32, 0 ], property: 'concat' }
[加密算法] 函数调用 - 参数: {
  function: [Function: concat],
  context: [ 222, 192, 173, 222, 32, 0 ],
  args: [
    [
      0, 0, 0, 0, 0,
      0, 0, 0, 0, 3
    ]
  ]
}
[加密算法] 函数调用结果: [
  222, 192, 173, 222, 32, 0,
    0,   0,   0,   0,  0, 0,
    0,   0,   0,   3
]
```

以上只是头部部分拼接的日志，感兴趣可以自己去分析，这其中还包含了 CRC32 对数据的完整性校验，具体的包头的结构如下

```
hexToLittleEndian('deadc0de') // deadcode魔数
    + '20 00' // 0x20好像是固定值 00 = (((0x20 >>> 0) && ff) >>> 8) & ff
    + '00 00 00 00 00 00 00 00 00 03' // 固定字符串 以上日志展示过
    + '01 10' // 4097固定值 (4097 >>> 0) && 0xff), (4097 >>> 8) && 0xff)
    + '06' // 固定值
    + hexToLittleEndian(crc32(Buffer.from('c7b645db-65e8-401f-b38c-4c07c5fff247')).toString(16).padStart(8, '0')) // 小端序 crc32 c7b645db-65e8-401f-b38c-4c07c5fff247 这个就是前面的fixedUuid
    + '02' // 可能是版本号，也是个固定值
    + hexToLittleEndian(crc32(Buffer.from(encryptData, 'hex')).toString(16).padStart(8, '0')) // 小端序 数据crc32
    + hexToLittleEndian((encryptData.length / 2).toString(16).padStart(8, '0')) // 4字节数据长度， 小端序
```

### 包体加密

后续的拼接就是包体的加密了，这里也是分析很久，因为 ks 用了一个非常少见的算法，我甚至都是第一次听说这个算法

```
这里的100就是原数据的uint8
[加密算法] 获取属性操作 - 参数: { object: [Arguments] { '0': 100 }, property: 0 }
[加密算法] 执行按位与操作, 输入: d9183ab9 & 1 输出: 1
[加密算法] 执行按位与操作, 输入: d530d1b & 1 输出: 1
[加密算法] 执行按位与操作, 输入: 50e74534 & 1 输出: 0
[加密算法] 执行逻辑右移操作, 输入: 50e74534 >>> 1 输出: 2873a29a
[加密算法] 执行按位与操作, 输入: 2873a29a & 7fffffff 输出: 2873a29a
[加密算法] 执行按位与操作, 输入: d530d1b & 1 输出: 1
[加密算法] 执行逻辑右移操作, 输入: 10000002 >>> 1 输出: 8000001
[加密算法] 执行按位异或操作, 输入: d530d1b ^ 8000001 输出: 5530d1a
[加密算法] 执行按位或操作, 输入: 5530d1a | f0000000 输出: f5530d1a
[加密算法] 执行左移操作, 输入: 0 << 1 输出: 0
[加密算法] 执行按位异或操作, 输入: 1 ^ 1 输出: 0
[加密算法] 执行按位或操作, 输入: 0 | 0 输出: 0
[加密算法] 执行按位与操作, 输入: 2873a29a & 1 输出: 0
[加密算法] 执行逻辑右移操作, 输入: 2873a29a >>> 1 输出: 1439d14d
[加密算法] 执行按位与操作, 输入: 1439d14d & 7fffffff 输出: 1439d14d
[加密算法] 执行按位与操作, 输入: f5530d1a & 1 输出: 0
[加密算法] 执行逻辑右移操作, 输入: f5530d1a >>> 1 输出: 7aa9868d
[加密算法] 执行按位与操作, 输入: 7aa9868d & fffffff 输出: aa9868d
[加密算法] 执行左移操作, 输入: 0 << 1 输出: 0
[加密算法] 执行按位异或操作, 输入: 1 ^ 0 输出: 1
[加密算法] 执行按位或操作, 输入: 0 | 1 输出: 1
[加密算法] 执行按位与操作, 输入: 1439d14d & 1 输出: 1
[加密算法] 执行逻辑右移操作, 输入: 80000062 >>> 1 输出: 40000031
[加密算法] 执行按位异或操作, 输入: 1439d14d ^ 40000031 输出: 5439d17c
[加密算法] 执行按位或操作, 输入: 5439d17c | 80000000 输出: d439d17c
[加密算法] 执行按位与操作, 输入: d9183ab9 & 1 输出: 1
[加密算法] 执行逻辑右移操作, 输入: 40000020 >>> 1 输出: 20000010
[加密算法] 执行按位异或操作, 输入: d9183ab9 ^ 20000010 输出: f9183aa9
[加密算法] 执行按位或操作, 输入: f9183aa9 | c0000000 输出: f9183aa9
[加密算法] 执行左移操作, 输入: 1 << 1 输出: 2
[加密算法] 执行按位异或操作, 输入: 1 ^ 0 输出: 1
[加密算法] 执行按位或操作, 输入: 2 | 1 输出: 3
[加密算法] 执行按位与操作, 输入: d439d17c & 1 输出: 0
[加密算法] 执行逻辑右移操作, 输入: d439d17c >>> 1 输出: 6a1ce8be
[加密算法] 执行按位与操作, 输入: 6a1ce8be & 7fffffff 输出: 6a1ce8be
[加密算法] 执行按位与操作, 输入: aa9868d & 1 输出: 1
[加密算法] 执行逻辑右移操作, 输入: 10000002 >>> 1 输出: 8000001
[加密算法] 执行按位异或操作, 输入: aa9868d ^ 8000001 输出: 2a9868c
[加密算法] 执行按位或操作, 输入: 2a9868c | f0000000 输出: f2a9868c
[加密算法] 执行左移操作, 输入: 3 << 1 输出: 6
[加密算法] 执行按位异或操作, 输入: 1 ^ 1 输出: 0
[加密算法] 执行按位或操作, 输入: 6 | 0 输出: 6
[加密算法] 执行按位与操作, 输入: 6a1ce8be & 1 输出: 0
[加密算法] 执行逻辑右移操作, 输入: 6a1ce8be >>> 1 输出: 350e745f
[加密算法] 执行按位与操作, 输入: 350e745f & 7fffffff 输出: 350e745f
[加密算法] 执行按位与操作, 输入: f2a9868c & 1 输出: 0
[加密算法] 执行逻辑右移操作, 输入: f2a9868c >>> 1 输出: 7954c346
[加密算法] 执行按位与操作, 输入: 7954c346 & fffffff 输出: 954c346
[加密算法] 执行左移操作, 输入: 6 << 1 输出: c
[加密算法] 执行按位异或操作, 输入: 1 ^ 0 输出: 1
[加密算法] 执行按位或操作, 输入: c | 1 输出: d
[加密算法] 执行按位与操作, 输入: 350e745f & 1 输出: 1
[加密算法] 执行逻辑右移操作, 输入: 80000062 >>> 1 输出: 40000031
[加密算法] 执行按位异或操作, 输入: 350e745f ^ 40000031 输出: 750e746e
[加密算法] 执行按位或操作, 输入: 750e746e | 80000000 输出: f50e746e
[加密算法] 执行按位与操作, 输入: f9183aa9 & 1 输出: 1
[加密算法] 执行逻辑右移操作, 输入: 40000020 >>> 1 输出: 20000010
[加密算法] 执行按位异或操作, 输入: f9183aa9 ^ 20000010 输出: d9183ab9
[加密算法] 执行按位或操作, 输入: d9183ab9 | c0000000 输出: d9183ab9
[加密算法] 执行左移操作, 输入: d << 1 输出: 1a
[加密算法] 执行按位异或操作, 输入: 1 ^ 0 输出: 1
[加密算法] 执行按位或操作, 输入: 1a | 1 输出: 1b
[加密算法] 执行按位与操作, 输入: f50e746e & 1 输出: 0
[加密算法] 执行逻辑右移操作, 输入: f50e746e >>> 1 输出: 7a873a37
[加密算法] 执行按位与操作, 输入: 7a873a37 & 7fffffff 输出: 7a873a37
[加密算法] 执行按位与操作, 输入: 954c346 & 1 输出: 0
[加密算法] 执行逻辑右移操作, 输入: 954c346 >>> 1 输出: 4aa61a3
[加密算法] 执行按位与操作, 输入: 4aa61a3 & fffffff 输出: 4aa61a3
[加密算法] 执行左移操作, 输入: 1b << 1 输出: 36
[加密算法] 执行按位异或操作, 输入: 1 ^ 0 输出: 1
[加密算法] 执行按位或操作, 输入: 36 | 1 输出: 37
[加密算法] 执行按位与操作, 输入: 7a873a37 & 1 输出: 1
[加密算法] 执行逻辑右移操作, 输入: 80000062 >>> 1 输出: 40000031
[加密算法] 执行按位异或操作, 输入: 7a873a37 ^ 40000031 输出: 3a873a06
[加密算法] 执行按位或操作, 输入: 3a873a06 | 80000000 输出: ba873a06
[加密算法] 执行按位与操作, 输入: d9183ab9 & 1 输出: 1
[加密算法] 执行逻辑右移操作, 输入: 40000020 >>> 1 输出: 20000010
[加密算法] 执行按位异或操作, 输入: d9183ab9 ^ 20000010 输出: f9183aa9
[加密算法] 执行按位或操作, 输入: f9183aa9 | c0000000 输出: f9183aa9
[加密算法] 执行左移操作, 输入: 37 << 1 输出: 6e
[加密算法] 执行按位异或操作, 输入: 1 ^ 0 输出: 1
[加密算法] 执行按位或操作, 输入: 6e | 1 输出: 6f
[加密算法] 执行按位异或操作, 输入: 64 ^ 6f 输出: b
// 这个b就是对100这个原字节进行加密后的结果
```

这里我就发现这个加密好像是逐字节加密的，很明显的流加密算法，我以为是 RC4，结果分析了以后发现完全不一样，我就一直找到第一个字节加密的 trace，发现有一个初始出来的魔数，然后通过这个魔数继续往上翻，找到了这个 byte 初始化的位置

```
[加密算法] 获取属性操作 - 参数: { object: 'BvWTr0uRBGH366Yb', property: 12 }
[加密算法] 获取属性操作 - 参数: { object: '6', property: 'charCodeAt' }
[加密算法] 函数调用 - 参数: { function: [Function: charCodeAt], context: '6', args: [ 0 ] }
[加密算法] 函数调用结果: 54
[加密算法] 设置属性操作 - 参数: {
  object: [
    66,
    118,
    87,
    84,
    114,
    48,
    117,
    82,
    66,
    71,
    72,
    51,
    <4 empty items>
  ],
  property: 12,
  value: 54
}
[加密算法] 获取属性操作 - 参数: { object: 'BvWTr0uRBGH366Yb', property: 13 }
[加密算法] 获取属性操作 - 参数: { object: '6', property: 'charCodeAt' }
[加密算法] 函数调用 - 参数: { function: [Function: charCodeAt], context: '6', args: [ 0 ] }
[加密算法] 函数调用结果: 54
[加密算法] 设置属性操作 - 参数: {
  object: [
    66,
    118,
    87,
    84,
    114,
    48,
    117,
    82,
    66,
    71,
    72,
    51,
    54,
    <3 empty items>
  ],
  property: 13,
  value: 54
}
[加密算法] 获取属性操作 - 参数: { object: 'BvWTr0uRBGH366Yb', property: 14 }
[加密算法] 获取属性操作 - 参数: { object: 'Y', property: 'charCodeAt' }
[加密算法] 函数调用 - 参数: { function: [Function: charCodeAt], context: 'Y', args: [ 0 ] }
[加密算法] 函数调用结果: 89
[加密算法] 设置属性操作 - 参数: {
  object: [
    66,
    118,
    87,
    84,
    114,
    48,
    117,
    82,
    66,
    71,
    72,
    51,
    54,
    54,
    <2 empty items>
  ],
  property: 14,
  value: 89
}
[加密算法] 获取属性操作 - 参数: { object: 'BvWTr0uRBGH366Yb', property: 15 }
[加密算法] 获取属性操作 - 参数: { object: 'b', property: 'charCodeAt' }
[加密算法] 函数调用 - 参数: { function: [Function: charCodeAt], context: 'b', args: [ 0 ] }
[加密算法] 函数调用结果: 98
[加密算法] 设置属性操作 - 参数: {
  object: [
    66,
    118,
    87,
    84,
    114,
    48,
    117,
    82,
    66,
    71,
    72,
    51,
    54,
    54,
    89,
    <1 empty item>
  ],
  property: 15,
  value: 98
}
[加密算法] 执行左移操作, 输入: 13579bdf << 8 输出: 579bdf00
[加密算法] 获取属性操作 - 参数: {
  object: [
     66, 118, 87, 84, 114, 48,
    117,  82, 66, 71,  72, 51,
     54,  54, 89, 98
  ],
  property: 4
}
[加密算法] 执行按位或操作, 输入: 579bdf00 | 72 输出: 579bdf72 // key1  第一轮
[加密算法] 执行左移操作, 输入: 2468ace0 << 8 输出: 68ace000
[加密算法] 获取属性操作 - 参数: {
  object: [
     66, 118, 87, 84, 114, 48,
    117,  82, 66, 71,  72, 51,
     54,  54, 89, 98
  ],
  property: 4
}
[加密算法] 执行按位或操作, 输入: 68ace000 | 72 输出: 68ace072 // key2 第一轮
[加密算法] 执行左移操作, 输入: fdb97531 << 8 输出: b9753100
[加密算法] 获取属性操作 - 参数: {
  object: [
     66, 118, 87, 84, 114, 48,
    117,  82, 66, 71,  72, 51,
     54,  54, 89, 98
  ],
  property: 4
}
[加密算法] 执行按位或操作, 输入: b9753100 | 72 输出: b9753172
[加密算法] 执行左移操作, 输入: 579bdf72 << 8 输出: 9bdf7200
[加密算法] 获取属性操作 - 参数: {
  object: [
     66, 118, 87, 84, 114, 48,
    117,  82, 66, 71,  72, 51,
     54,  54, 89, 98
  ],
  property: 5
}
[加密算法] 执行按位或操作, 输入: 9bdf7200 | 30 输出: 9bdf7230
[加密算法] 执行左移操作, 输入: 68ace072 << 8 输出: ace07200
[加密算法] 获取属性操作 - 参数: {
  object: [
     66, 118, 87, 84, 114, 48,
    117,  82, 66, 71,  72, 51,
     54,  54, 89, 98
  ],
  property: 5
}
[加密算法] 执行按位或操作, 输入: ace07200 | 30 输出: ace07230
[加密算法] 执行左移操作, 输入: b9753172 << 8 输出: 75317200
[加密算法] 获取属性操作 - 参数: {
  object: [
     66, 118, 87, 84, 114, 48,
    117,  82, 66, 71,  72, 51,
     54,  54, 89, 98
  ],
  property: 5
}
[加密算法] 执行按位或操作, 输入: 75317200 | 30 输出: 75317230
[加密算法] 执行左移操作, 输入: 9bdf7230 << 8 输出: df723000
[加密算法] 获取属性操作 - 参数: {
  object: [
     66, 118, 87, 84, 114, 48,
    117,  82, 66, 71,  72, 51,
     54,  54, 89, 98
  ],
  property: 6
}
[加密算法] 执行按位或操作, 输入: df723000 | 75 输出: df723075
[加密算法] 执行左移操作, 输入: ace07230 << 8 输出: e0723000
[加密算法] 获取属性操作 - 参数: {
  object: [
     66, 118, 87, 84, 114, 48,
    117,  82, 66, 71,  72, 51,
     54,  54, 89, 98
  ],
  property: 6
}
[加密算法] 执行按位或操作, 输入: e0723000 | 75 输出: e0723075
[加密算法] 执行左移操作, 输入: 75317230 << 8 输出: 31723000
[加密算法] 获取属性操作 - 参数: {
  object: [
     66, 118, 87, 84, 114, 48,
    117,  82, 66, 71,  72, 51,
     54,  54, 89, 98
  ],
  property: 6
}
[加密算法] 执行按位或操作, 输入: 31723000 | 75 输出: 31723075
[加密算法] 执行左移操作, 输入: df723075 << 8 输出: 72307500
[加密算法] 获取属性操作 - 参数: {
  object: [
     66, 118, 87, 84, 114, 48,
    117,  82, 66, 71,  72, 51,
     54,  54, 89, 98
  ],
  property: 7
}
[加密算法] 执行按位或操作, 输入: 72307500 | 52 输出: 72307552
[加密算法] 执行左移操作, 输入: e0723075 << 8 输出: 72307500
[加密算法] 获取属性操作 - 参数: {
  object: [
     66, 118, 87, 84, 114, 48,
    117,  82, 66, 71,  72, 51,
     54,  54, 89, 98
  ],
  property: 7
}
[加密算法] 执行按位或操作, 输入: 72307500 | 52 输出: 72307552
[加密算法] 执行左移操作, 输入: 31723075 << 8 输出: 72307500
[加密算法] 获取属性操作 - 参数: {
  object: [
     66, 118, 87, 84, 114, 48,
    117,  82, 66, 71,  72, 51,
     54,  54, 89, 98
  ],
  property: 7
}
[加密算法] 执行按位或操作, 输入: 72307500 | 52 输出: 72307552
```

这里可以看到，会通过一个种子和三个魔数（0x13579bdf，0x2468ace0，0xfdb97531）来计算一个 key（0x72307552），然后通过这个 key 来加密原数据，于是我找了很多资料去查这三个魔数，最后确定了是 cfmxCompat 的算法，很多年就已经过气的算法了，这中间踩了很多坑，因为关于这个算法的文章非常少，实现也少，甚至还有很多变体，只能一点点去对这 Trace 比结果，这个算法应该是 ks 自己去实现的，我这里就不展开了，有兴趣的可以自己去查资料，还原了这个算法整个加密就出来了

加密 / 解密算法还原
-----------

```
class CfmxCompat {
    constructor() {
        this.m_LFSR_A = 0x13579bdf;
        this.m_LFSR_B = 0x2468ace0;
        this.m_LFSR_C = 0xfdb97531;
 
        this.m_MASK_A = 0x80000062;
        this.m_MASK_B = 0x40000020;
        this.m_MASK_C = 0x10000002;
        this.m_ROT0_A = 0x7fffffff;
        this.m_ROT0_B = 0x3fffffff;
        this.m_ROT0_C = 0xfffffff;
        this.m_ROT1_A = 0x80000000;
        this.m_ROT1_B = 0xc0000000;
        this.m_ROT1_C = 0xf0000000;
    }
 
    /**
     * Encrypt a value
     *
     * @param {String} string - String to encrypt.
     * @param {String} key - Key or seed used to encrypt the string.
     * @param {String} [encoding=hex] - The binary encoding in which to represent the data as a string. Can be either 'hex' or 'base64'.
     * @returns {String} The encrypted value.
     */
    encrypt(inString, key, encoding = 'hex') {
        let inBytes = Buffer.from(inString);
        let outString = this.transformString(inBytes, key);
 
        if (encoding === 'base64') {
            outString = outString.toString('base64');
        } else if (encoding === 'hex') {
            outString = outString.toString('hex').toUpperCase();
        } else {
            throw new Error('Invalid encoding');
        }
 
        return outString;
    }
 
    /**
     * Decrypt a value
     *
     * @param {String} string - String to decrypt.
     * @param {String} key - The key or seed that was used to encrypt the string.
     * @param {String} [encoding=hex] - The binary encoding in which to represent the data as a string. Must be the same as the algorithm used to encrypt the string.
     * @returns {Buffer} The decrypted value.
     */
    decrypt(inString, key, encoding = 'hex') {
        let inBytes;
 
        if (encoding === 'base64') {
            inBytes = Buffer.from(inString, 'base64');
        } else if (encoding === 'hex') {
            inBytes = Buffer.from(inString, 'hex');
        } else {
            throw new Error('Invalid encoding');
        }
 
        let outBytes = this.transformString(inBytes, key);
        return outBytes.toString();
    }
 
    transformString(inBytes, key) {
        this.m_LFSR_A = 0x13579bdf;
        this.m_LFSR_B = 0x2468ace0;
        this.m_LFSR_C = 0xfdb97531;
 
        this.setKey(key);
 
        let length = inBytes.length;
        let outBytes = Buffer.alloc(length);
 
        for (let i = 0; i < length; ++i) {
            let byte = inBytes.readInt8(i, true);
            outBytes[i] = this.transformByte(byte);
        }
 
        return outBytes;
    }
 
    transformByte(target) {
        let crypto = 0;
        let b = this.m_LFSR_B & 0x1;
        let c = this.m_LFSR_C & 0x1;
 
        for (let i = 0; i < 8; ++i) {
            if (0 !== (this.m_LFSR_A & 0x1)) {
                this.m_LFSR_A = this.m_LFSR_A ^ this.m_MASK_A >>> 1 | this.m_ROT1_A;
 
                if (0 !== (this.m_LFSR_B & 0x1)) {
                    this.m_LFSR_B = this.m_LFSR_B ^ this.m_MASK_B >>> 1 | this.m_ROT1_B;
                    b = 1;
                } else {
                    this.m_LFSR_B = this.m_LFSR_B >>> 1 & this.m_ROT0_B;
                    b = 0;
                }
            } else {
                this.m_LFSR_A = this.m_LFSR_A >>> 1 & this.m_ROT0_A;
 
                if (0 !== (this.m_LFSR_C & 0x1)) {
                    this.m_LFSR_C = this.m_LFSR_C ^ this.m_MASK_C >>> 1 | this.m_ROT1_C;
                    c = 1;
                } else {
                    this.m_LFSR_C = this.m_LFSR_C >>> 1 & this.m_ROT0_C;
                    c = 0;
                }
            }
 
            crypto = crypto << 1 | b ^ c;
        }
 
        target = target ^ crypto;
        return target;
    }
 
    setKey(key) {
        key = key || 'Default Seed';
 
        let i;
        let keyLength = key.length;
        let seed = new Array(Math.max(12, keyLength));
 
        for (i = 0; i < keyLength; i++) {
            seed[i] = key[i].charCodeAt(0);
        }
 
        for (i = 0; keyLength + i < 12; ++i) {
            seed[keyLength + i] = seed[i];
        }
 
        for (i = 0; i < 4; i++) {
            this.m_LFSR_A = (this.m_LFSR_A << 8) | seed[i + 4];
            this.m_LFSR_B = (this.m_LFSR_B << 8) | seed[i + 4];
            this.m_LFSR_C = (this.m_LFSR_C << 8) | seed[i + 4];
        }
 
        if (0 === this.m_LFSR_A) this.m_LFSR_A = 0x13579bdf;
        if (0 === this.m_LFSR_B) this.m_LFSR_B = 0x2468ace0;
        if (0 === this.m_LFSR_C) this.m_LFSR_C = 0xfdb97531;
    }
}
 
new CfmxCompat().encrypt("will encrypt data", 'BvWTr0uRBGH366Yb', 'hex') // 加密
new CfmxCompat().decrypt("will decrypt data", 'BvWTr0uRBGH366Yb', 'hex') // 解密
```

尾声
==

到这里 verifyParam 的加密就还原完了，但是这里面还有个轨迹需要补，不过在 52 那边有人分享过这个轨迹的生成，虽然有一点变动，但是大体上还是一样的，这里我就不展开了，如果有时间我会补上～

[[注意] 看雪招聘，专注安全领域的专业人才平台！](https://job.kanxue.com/)

最后于 1 天前 被 Mysterious77 编辑 ，原因：

[#其他](forum-151-1-212.htm)
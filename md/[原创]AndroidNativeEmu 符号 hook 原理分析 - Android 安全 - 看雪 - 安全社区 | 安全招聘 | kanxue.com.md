> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-282571.htm)

> [原创]AndroidNativeEmu 符号 hook 原理分析

[原创]AndroidNativeEmu 符号 hook 原理分析

2 天前 786

### [原创]AndroidNativeEmu 符号 hook 原理分析

![](https://passport.kanxue.com/pc/view/img/moon.gif)![](https://passport.kanxue.com/pc/view/img/star.gif)

* * *

AndroidNativeEmu 支持某个函数符号进行 hook，代码实例如下：

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p></td><td><p><code>@native_method</code></p><p><code>def</code> <code>sprintf(mu, buffer_addr, format_addr, arg1, arg2):</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>print</code><code>(</code><code>"sprintf(%x,%x,%x,%x)"</code> <code>%</code> <code>(buffer_addr, format_addr, arg1, arg2))</code></p><p><code>emulator.modules.add_symbol_hook(</code><code>"sprintf"</code><code>, emulator.hooker.write_function(sprintf) </code><code>+</code> <code>1</code><code>)</code></p></td></tr></tbody></table>

他是怎么实现的呢？我们从其源代码来分析。

考虑如下问题：

1.  如何实现的函数符号的 hook？
    
    初步的猜想是将符号表的对应地址值填为指定函数的地址，后续解析符号的时候，就将该符号的地址填充过去。但是这样存在问题，因为我们的函数是 python 函数，没有地址，所以无法直接跳转到我们的 python 函数。
    
2.  根据问题 1，是如何将 hook 的地址和 python 函数关联的？
    
    unicorn 本身的 hook 是只支持对某个地址区域进行 hook 的，只要执行到这个区域的代码，就会调用对应的 python 函数。但是这里是符号 hook，而不是地址区域 hook。
    

#### emulator.hooker.write_function(sprintf) 分析

首先从`emulator.hooker.write_function(sprintf)`开始分析，进去看看他干了什么。

###### hooker

先看看 emulator 内部的 hooker 对象是啥，进入到 emulator 的构造函数中：

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p></td><td><p><code>HOOK_MEMORY_BASE </code><code>=</code> <code>0x20000000</code></p><p><code>HOOK_MEMORY_SIZE </code><code>=</code> <code>0x00200000</code></p><p><code>self</code><code>.hooker </code><code>=</code> <code>Hooker(</code><code>self</code><code>, HOOK_MEMORY_BASE, HOOK_MEMORY_SIZE)</code></p></td></tr></tbody></table>

可以看到他是 Hooker 的实例，再进入 Hooker 类，其构造函数部分的代码如下：

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p><p>11</p><p>12</p><p>13</p><p>14</p><p>15</p><p>16</p><p>17</p><p>18</p><p>19</p><p>20</p><p>21</p><p>22</p></td><td><p><code>from</code> <code>keystone </code><code>import</code> <code>Ks, KS_ARCH_ARM, KS_MODE_THUMB</code></p><p><code>from</code> <code>unicorn </code><code>import</code> <code>*</code></p><p><code>from</code> <code>unicorn.arm_const </code><code>import</code> <code>*</code></p><p><code>STACK_OFFSET </code><code>=</code> <code>8</code></p><p><code>class</code> <code>Hooker:</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>def</code> <code>__init__(</code><code>self</code><code>, emu, base_addr, size):</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>self</code><code>._emu </code><code>=</code> <code>emu</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>self</code><code>._keystone </code><code>=</code> <code>Ks(KS_ARCH_ARM, KS_MODE_THUMB)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>self</code><code>._size </code><code>=</code> <code>size</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>self</code><code>._current_id </code><code>=</code> <code>0xFF00</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>self</code><code>._hooks </code><code>=</code> <code>dict</code><code>()</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>self</code><code>._hook_magic </code><code>=</code> <code>base_addr</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>self</code><code>._hook_start </code><code>=</code> <code>base_addr </code><code>+</code> <code>4</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>self</code><code>._hook_current </code><code>=</code> <code>self</code><code>._hook_start</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>self</code><code>._emu.uc.hook_add(UC_HOOK_CODE, </code><code>self</code><code>._hook, </code><code>None</code><code>, </code><code>self</code><code>._hook_start, </code><code>self</code><code>._hook_start </code><code>+</code> <code>size)</code></p></td></tr></tbody></table>

到这里，hooker 对象就创建完成了。还是比较简单，就是抽象了一个专门的地址区域用于 hook，同时在 unicorn 中给这段地址区域设置了 UC_HOOK_CODE。

###### write_function

接下来是`hook.write_function`：

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p><p>11</p><p>12</p><p>13</p><p>14</p><p>15</p><p>16</p><p>17</p><p>18</p><p>19</p><p>20</p><p>21</p><p>22</p><p>23</p><p>24</p><p>25</p><p>26</p></td><td><p><code>def</code> <code>write_function(</code><code>self</code><code>, func):</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>hook_id </code><code>=</code> <code>self</code><code>._get_next_id()</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>hook_addr </code><code>=</code> <code>self</code><code>._hook_current</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>asm </code><code>=</code> <code>"PUSH {R4,LR}\n"</code> <code>\</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>"MOV R4, #"</code> <code>+</code> <code>hex</code><code>(hook_id) </code><code>+</code> <code>"\n"</code> <code>\</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>"MOV R4, R4\n"</code> <code>\</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>"POP {R4,PC}"</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>asm_bytes_list, asm_count </code><code>=</code> <code>self</code><code>._keystone.asm(bytes(asm, encoding</code><code>=</code><code>'ascii'</code><code>))</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>asm_count !</code><code>=</code> <code>4</code><code>:</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>raise</code> <code>ValueError(</code><code>"Expected asm_count to be 4 instead of %u."</code> <code>%</code> <code>asm_count)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>self</code><code>._emu.uc.mem_write(hook_addr, bytes(asm_bytes_list))</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>self</code><code>._hook_current </code><code>+</code><code>=</code> <code>len</code><code>(asm_bytes_list)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>self</code><code>._hooks[hook_id] </code><code>=</code> <code>func</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code> <code>hook_addr</code></p></td></tr></tbody></table>

简单来说，就是创建了一段特殊的代码区域，返回了这个区域的地址。这个代码区域有特定的 id 标识`（为了区分不同的函数hook）`，且这段内存区域在 unicorn 中被设置了回调函数`（hooker的构造函数中）`。

#### emulator.modules.add_symbol_hook("sprintf", emulator.hooker.write_function(sprintf) + 1)

这个函数实现非常简单：

<table><tbody><tr><td><p>1</p><p>2</p></td><td><p><code>def</code> <code>add_symbol_hook(</code><code>self</code><code>, symbol_name, addr):</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>self</code><code>.symbol_hooks[symbol_name] </code><code>=</code> <code>addr</code></p></td></tr></tbody></table>

就是将`write_function`返回的地址写入到符号表中，后续其他依赖这个函数的库会解析为对应的地址。也就是说，后续执行这个符号对应的函数的时候，就会跳转到`write_function`中生成的代码区域。同时要注意，这个区域是被 hook 了的。所以，我们要去看看 Hooker 类中的`_hook`方法是如何实现的，看看他做了什么处理

#### _hook 方法

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p><p>11</p><p>12</p><p>13</p><p>14</p><p>15</p><p>16</p><p>17</p><p>18</p></td><td><p><code>def</code> <code>_hook(</code><code>self</code><code>, uc, address, size, user_data):</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>size !</code><code>=</code> <code>2</code> <code>or</code> <code>self</code><code>._emu.uc.mem_read(address, size) !</code><code>=</code> <code>b</code><code>"\x24\x46"</code><code>:</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>hook_id </code><code>=</code> <code>self</code><code>._emu.uc.reg_read(UC_ARM_REG_R4)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>hook_func </code><code>=</code> <code>self</code><code>._hooks[hook_id]</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>try</code><code>:</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>hook_func(</code><code>self</code><code>._emu)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>except</code><code>:</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>uc.emu_stop()</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>raise</code></p></td></tr></tbody></table>

#### 总结

回答之前我们的提问：

1.  如何实现的符号 hook？
    
    对于符号的 hook 确实是依赖于对符号表的 hook 实现的，但是其地址不能是 python 函数的地址。框架的作者专门为每个被 hook 的函数开辟了一块内存空间，放入一些特殊的汇编代码和一个函数的 id，通过这个 id 能找到对应的 python 函数。同时这个内存区域是被 unicorn 的原生 hook hook 了的，会调用框架内部的处理函数。在该处理函数中，会从这个内存区域中去找到函数 id，从而得到该符号对应的 python 函数进行调用。
    

通过前面的分析，我们已经知道了如何将符号和我们对应的 python 函数关联上。但是还存在一个问题，就是函数参数的问题，如何将函数参数传递给对应的 python 参数？这就要用到框架提供的`@native_method`装饰器

装饰器本质上是一个可调用对象，它接收一个函数作为参数，并返回一个新函数或可调用对象。装饰器通常用于横切关注点（cross-cutting concerns），如日志记录、权限检查、缓存等。

实例：

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p><p>11</p><p>12</p><p>13</p></td><td><p><code>def</code> <code>my_decorator(func):</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>def</code> <code>wrapper(</code><code>*</code><code>args, </code><code>*</code><code>*</code><code>kwargs):</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>print</code><code>(</code><code>"Something is happening before the function is called."</code><code>)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>result </code><code>=</code> <code>func(</code><code>*</code><code>args, </code><code>*</code><code>*</code><code>kwargs)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>print</code><code>(</code><code>"Something is happening after the function is called."</code><code>)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code> <code>result</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code> <code>wrapper</code></p><p><code>@my_decorator</code></p><p><code>def</code> <code>say_hello(name):</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>print</code><code>(f</code><code>"Hello, {name}!"</code><code>)</code></p><p><code>say_hello(</code><code>"Alice"</code><code>)</code></p></td></tr></tbody></table>

在这个示例中，`@my_decorator` 应用于 `say_hello` 函数，等价于 `say_hello = my_decorator(say_hello)`。当你调用 `say_hello("Alice")` 时，实际执行的是 `wrapper` 函数。

源代码如下：

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p><p>7</p><p>8</p><p>9</p><p>10</p><p>11</p><p>12</p><p>13</p><p>14</p><p>15</p><p>16</p><p>17</p><p>18</p><p>19</p><p>20</p><p>21</p><p>22</p><p>23</p><p>24</p><p>25</p><p>26</p><p>27</p><p>28</p><p>29</p><p>30</p></td><td><p><code>def</code> <code>native_method(func):</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>def</code> <code>native_method_wrapper(</code><code>*</code><code>argv):</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>emu </code><code>=</code> <code>argv[</code><code>1</code><code>] </code><code>if</code> <code>len</code><code>(argv) </code><code>=</code><code>=</code> <code>2</code> <code>else</code> <code>argv[</code><code>0</code><code>]</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>uc </code><code>=</code> <code>emu.uc</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>args </code><code>=</code> <code>inspect.getfullargspec(func).args</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>args_count </code><code>=</code> <code>len</code><code>(args) </code><code>-</code> <code>(</code><code>2</code> <code>if</code> <code>'self'</code> <code>in</code> <code>args </code><code>else</code> <code>1</code><code>)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>args_count &lt; </code><code>0</code><code>:</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>raise</code> <code>RuntimeError(</code><code>"NativeMethod accept at least (self, uc) or (uc)."</code><code>)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>native_args </code><code>=</code> <code>native_read_args(uc, args_count)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>len</code><code>(argv) </code><code>=</code><code>=</code> <code>1</code><code>:</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>result </code><code>=</code> <code>func(uc, </code><code>*</code><code>native_args)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>else</code><code>:</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>result </code><code>=</code> <code>func(argv[</code><code>0</code><code>], uc, </code><code>*</code><code>native_args)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>if</code> <code>result </code><code>is</code> <code>not</code> <code>None</code><code>:</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>native_write_arg_register(emu, UC_ARM_REG_R0, result)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>else</code><code>:</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</code><code>uc.reg_write(UC_ARM_REG_R0, JNI_ERR)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>return</code> <code>native_method_wrapper</code></p></td></tr></tbody></table>

所以，在我们的代码中：

<table><tbody><tr><td><p>1</p><p>2</p><p>3</p><p>4</p><p>5</p><p>6</p></td><td><p><code>@native_method</code></p><p><code>def</code> <code>sprintf(mu, buffer_addr, format_addr, arg1, arg2):</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>print</code><code>(</code><code>"sprintf(%x,%x,%x,%x)"</code> <code>%</code> <code>(buffer_addr, format_addr, arg1, arg2))</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>format</code> <code>=</code> <code>memory_helpers.read_utf8(mu, format_addr)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>result </code><code>=</code> <code>format</code> <code>%</code> <code>(memory_helpers.read_utf8(mu, arg1), arg2)</code></p><p><code>&nbsp;&nbsp;&nbsp;&nbsp;</code><code>mu.mem_write(buffer_addr, result.encode() </code><code>+</code> <code>b</code><code>'\x00'</code><code>)</code></p></td></tr></tbody></table>

调用 sprintf 之前，实际上调用的是 native_method 方法，该方法对 sprintf 进行了装饰。

而其 native_method_wrapper 装饰的实现就是在获取函数参数的个数，然后从寄存器中获取对应的参数，传递给 python。

##### 扩展思考

这个框架中的 @native_method 装饰器的作用是解析函数参数，从寄存器或者内存中读取参数再传递给 python，但是他的实现是比较简陋的。本质上是利用 inspect 获取 python 的函数原型再决定读取几个参数。

比如，如果我们要完整的实现 sprintf 的 hook 就不太行，因为 sprintf 的参数是不定的，函数原型中不能知道具体有几个参数。只能解析 spriintf 的 format 来确定参数数量，再读取参数。当然实现起来有点麻烦，后续尝试写一下。

[[培训] 内核驱动高级班，冲击 BAT 一流互联网大厂工作，每周日 13:00-18:00 直播授课](https://www.kanxue.com/book-section_list-174.htm)
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1485513-1-1.html)

> [md]#### 由于在工作的需要，在逆向分析中，常遇到强混淆，以及 vm 虚拟化加固方案。

![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)yangyss _ 本帖最后由 涛之雨 于 2021-8-3 00:05 编辑_  

#### 由于在工作的需要，在逆向分析中，常遇到强混淆，以及 vm 虚拟化加固方案。在分析的过程中，痛不欲生。为了在逆向分析过程中，能安心的喝口咖啡，同时还能还原出高度混淆 / vm 虚拟化的代码，参考函数追踪的原理，弄了个 指令级的 lldb-trace 脚本。

**_[前瞻]_**

> 平时分析中，难免遇到 c/c++ 函数，objc 函数。而 objc 函数中，又包含类如 release, 引用计数函数... 等等。故而，在 trace 过程中，objc 中的 release 等引用计数等函数，系统函数，都直接过滤。而 objc_msgsend 函数，可以做重点分析：取决于你要不要分析此次对应的函数的实现。当然，此次方案中，我是不需要的，所以，我只对 objc_msgsend 函数的函数名做了 trace，而对应的函数实现，我就没 trace 了。

**_[框架分析]_**

> 1，对不同的平台，做不同的配置处理  
> 2，忽略掉的函数，都放到 忽略函数列表中  
> 3，objc_msgsend 函数需做特殊处理，所以放在受保护的函数列表中  
> 4，在 trace 中，需要读取上一条指令中用到寄存器，故用正则匹配出其所有的寄存器。

### 更新：

> 1，trace 参数优化，绝大部分参数都为默认参数，方便使用。  
> 2，结束地址可以有多个（在某些混淆情况下，不确定结束地址到哪里，可以多设置几个结束地址，用 ";" 分开）。  
> 3，增加了 暂停其他线程 的 可选参数。  
> 4，增加了 只 trace 本模块的 可选参数。  
> 5，增加了 进度 信息（防止以为脚本卡死… 等的不耐心… 从而关闭了 lldb。此处用到了 objc 脚本，读取 ASLR，如果平台不同，可以忽略掉 ASLR）。  
> 6，增加了检测 逆向还原的算法 的脚本 (用脚本化，去验证你分析的算法是否 ok，懒人做法~_~)。  
> 7，对 msg_send 函数的参数解析开发中…

**怎么使用该脚本:**

> 1，在你准备追踪的地方下断点：（我的断点，从 breakpoint 函数 si 进入到 a 函数的 第一行）  
> ![](https://attach.52pojie.cn/forum/202107/30/142054xdzddxuj29dqj7p2.png)
> 
> **1111.png** _(145.51 KB, 下载次数: 0)_
> 
> [下载附件](forum.php?mod=attachment&aid=MjMxNzcwOXxkNWI5ZWYxMHwxNjI4MDY1MDE2fDIxMzQzMXwxNDg1NTEz&nothumb=yes)
> 
> 1111
> 
> 2021-7-30 14:20 上传
> 
> "alt="111" />  
> 2，导入 lldbTrace.py 脚本。（你可以设置，默认的 log 文件路径。如果不设置，默认和脚本同位置）  
> ![](https://attach.52pojie.cn/forum/202107/30/142154uvvvtpppr407gqst.png)
> 
> **2222.png** _(56.11 KB, 下载次数: 0)_
> 
> [下载附件](forum.php?mod=attachment&aid=MjMxNzcxMHxmMWRiMDFjZnwxNjI4MDY1MDE2fDIxMzQzMXwxNDg1NTEz&nothumb=yes)
> 
> 2021-7-30 14:21 上传
> 
> "alt="222" />  
> 3，设置一个停止追踪的地址：（当前 a 函数，我把结束地址设为 最后地址，和 ret 地址。为了查看 debug 信息，我把 log 类型设置成了 debug）  
> ![](https://attach.52pojie.cn/forum/202107/30/142201uwvak3b3i38sijs3.png)
> 
> **3333.png** _(249.32 KB, 下载次数: 0)_
> 
> [下载附件](forum.php?mod=attachment&aid=MjMxNzcxMXw4NjZlNWQwZHwxNjI4MDY1MDE2fDIxMzQzMXwxNDg1NTEz&nothumb=yes)
> 
> 333
> 
> 2021-7-30 14:22 上传
> 
> "alt="333" />  
> 4，设置好，直接回车，结果如下：  
> ![](https://attach.52pojie.cn/forum/202107/30/142208li84xx64vzh0i4pu.png)
> 
> **4444.png** _(390.06 KB, 下载次数: 0)_
> 
> [下载附件](forum.php?mod=attachment&aid=MjMxNzcxMnw3ZDhmNTIzNXwxNjI4MDY1MDE2fDIxMzQzMXwxNDg1NTEz&nothumb=yes)
> 
> 2021-7-30 14:22 上传
> 
> "alt="444" />

**脚本在 git 上:**[lldb-trace 脚本仓库](https://github.com/yangyss/lldb-trace)

> 脚本还有很多不完善的地方，需要慢慢优化。  
> 不过利用 trace 结果，能还原 手写的算法，以及强混淆 或者 某些 vm 虚拟机。![](https://avatar.52pojie.cn/data/avatar/000/87/90/80_avatar_middle.jpg)涛之雨 图片的插入有点问题，已经帮你编辑好了，  
具体参见论坛的帮助  
[https://www.52pojie.cn/misc.php? ... 29&messageid=36](https://www.52pojie.cn/misc.php?mod=faq&action=faq&id=29&messageid=36) ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) _ 本帖最后由 yangyss 于 2021-8-3 07:04 编辑_  

### * 实在有点尴尬，不太会用，谢谢版主大大的修改~_~，一直没编辑过，预览也没达到自己的目标，后面内容，我在回帖中，补充吧。

#### --------------------------------------------------

还原的算法 的验证脚本:
------------

### 1, 脚本开发背景：

> 在逆向过程中，我们对照汇编代码 / ida 伪代码 等翻译了算法。然后需要知道这个算法是不是正确的。平常的做法，就是一边继续调试一遍验证。对于代码比较少的，调试几次，就能搞定最后搞定；而对于算法复杂点，代码多点的，调试一遍，就需要花费不少经历，何况需要调试多次，才能验证整个算法的 ok。工作量很大的。。。

### 2，脚本原理:

> 1，不论是原算法，还是翻译的算法，其数据流必须一样。(所以，借助 lldb-trace 的结果，拿到数据流... 也就是拿到 trace 的 log 信息。。。)  
> 2，由于 python sys 模块中，有 settrace 相关的函数。如果设置好 sys.settrace(dest_fun())，在 dest_fun() 中，我们就能监控到相关的信息。原理就是 用 python 调试器，调试你的 python 代码.... 大概就是这样。（别的语言，不太熟悉，恰好 python，简单方便，所以就选他了..）  
> 3，由于 test_fun() 监控的是所有的 python 代码，而我们只用监测某些变量，所以引入一个 api，check_value(value,flag) 表示监控此 value，同时，让注册的回调函数，监控他。  
> 4，根据 flag，解析 lldb-trace 的 log 信息，拿到所有的数据流...  
> 5，然后判断数据流，在算法中，是否一一对应。。。

### 3，怎么使用该脚本:

> 1，在 ida 中，找到需要还原的算法 (可以是汇编，也可以是伪代码)，翻译成对应的 python 代码  
> ![](https://attach.52pojie.cn//forum/202108/03/065609mmdvy9zzfp198dv9.png?l)  
> 2，利用 lldbTrace.py 脚本，把需要翻译的函数 trace 一哈  
> 3，在 ida 伪代码中，切换到对应的汇编，对照 trace 结果，确定具体检测的地址。因为 trace，当前打印的是上一句代码执行后的寄存器值。所以，我们把 check 的地址，定义为 0x10000393c，寄存器为 w8.  
> ![](https://attach.52pojie.cn//forum/202108/03/065813ljpxuu4940crrxyc.png?l)  
> 4，定义检测的 变量 Check_0x10000393c_w8 = ‘Check_0x10000393c_w8’ 。在翻译的代码中，添加检测函数 check_value(ret,Check_0x10000393c_w8) 。在解析 tracelog 文件之前，设置 check 的相关信息 set_trace_data(Check_0x10000393c_w8)  
> 5，用 python 调用此脚本，并传入 tracelog 文件的路径。结果如下:  
> ![](https://attach.52pojie.cn//forum/202108/03/065905gzgvxn98j53h8of8.png?l)

### 4，具体代码:

```
#!/usr/bin/python3
# -*- encoding: utf-8 -*-
#home.php?mod=space&uid=267492    :   wtpytracer.py
#home.php?mod=space&uid=238618    :   2021/07/27 18:17:18
#home.php?mod=space&uid=686208  :   wt

import re
import sys
import inspect
from collections import OrderedDict

class TracebackFancy:

    def __init__(self, traceback):
        self.t = traceback

    def getFrame(self):
        return FrameFancy(self.t.tb_frame)

    def getLineNumber(self):
        return self.t.tb_lineno if self.t is not None else None

    def getNext(self):
        return TracebackFancy(self.t.tb_next)

    def __str__(self):
        if self.t is None:
            return ""
        str_self = "%s home.php?mod=space&uid=402414 %s" % (
            self.getFrame().getName(), self.getLineNumber())
        return str_self + "\n" + self.getNext().__str__()

class ExceptionFancy:

    def __init__(self, frame):
        self.etraceback = frame.f_exc_traceback
        self.etype = frame.exc_type
        self.evalue = frame.f_exc_value

    def __init__(self, tb, ty, va):
        self.etraceback = tb
        self.etype = ty
        self.evalue = va

    def getTraceback(self):
        return TracebackFancy(self.etraceback)

    def __nonzero__(self):
        return self.etraceback is not None or self.etype is not None or self.evalue is not None

    def getType(self):
        return str(self.etype)

    def getValue(self):
        return self.evalue

class CodeFancy:

    def __init__(self, code):
        self.c = code

    def getArgCount(self):
        return self.c.co_argcount if self.c is not None else 0

    def getFilename(self):
        return self.c.co_filename if self.c is not None else ""

    def getVariables(self):
        return self.c.co_varnames if self.c is not None else []

    def getName(self):
        return self.c.co_name if self.c is not None else ""

    def getFileName(self):
        return self.c.co_filename if self.c is not None else ""

class ArgsFancy:

    def __init__(self, frame, arginfo):
        self.f = frame
        self.a = arginfo

    def __str__(self):
        args, varargs, kwargs = self.getArgs(), self.getVarArgs(), self.getKWArgs()
        ret = ""
        count = 0
        size = len(args)
        for arg in args:
            ret = ret + ("%s = %s" % (arg, args[arg]))
            count = count + 1
            if count < size:
                ret = ret + ", "
        if varargs:
            if size > 0:
                ret = ret + " "
            ret = ret + "varargs are " + str(varargs)
        if kwargs:
            if size > 0:
                ret = ret + " "
            ret = ret + "kwargs are " + str(kwargs)
        return ret

    def getNumArgs(wantVarargs=False, wantKWArgs=False):
        args, varargs, keywords, values = self.a
        size = len(args)
        if varargs and wantVarargs:
            size = size + len(self.getVarArgs())
        if keywords and wantKWArgs:
            size = size + len(self.getKWArgs())
        return size

    def getArgs(self):
        args, _, _, values = self.a
        argWValues = OrderedDict()
        for arg in args:
            argWValues[arg] = values[arg]
        return argWValues

    def getVarArgs(self):
        _, vargs, _, _ = self.a
        if vargs:
            return self.f.f_locals[vargs]
        return ()

    def getKWArgs(self):
        _, _, kwargs, _ = self.a
        if kwargs:
            return self.f.f_locals[kwargs]
        return {}

class FrameFancy:

    def __init__(self, frame):
        self.f = frame

    def getCaller(self):
        return FrameFancy(self.f.f_back)

    def getLineNumber(self):
        return self.f.f_lineno if self.f is not None else 0

    def getCodeInformation(self):
        return CodeFancy(self.f.f_code) if self.f is not None else None

    def getExceptionInfo(self):
        return ExceptionFancy(self.f) if self.f is not None else None

    def getName(self):
        return self.getCodeInformation().getName() if self.f is not None else ""

    def getFileName(self):
        return self.getCodeInformation().getFileName() if self.f is not None else ""

    def getLocals(self):
        return self.f.f_locals if self.f is not None else {}

    def getArgumentInfo(self):
        return ArgsFancy(
            self.f, inspect.getargvalues(
                self.f)) if self.f is not None else None

class TracerClass:

    def callEvent(self, frame):
        pass

    def lineEvent(self, frame):
        pass

    def returnEvent(self, frame, retval):
        pass

    def exceptionEvent(self, frame, exception, value, traceback):
        pass

    def cCallEvent(self, frame, cfunct):
        pass

    def cReturnEvent(self, frame, cfunct):
        pass

    def cExceptionEvent(self, frame, cfunct):
        pass

tracer_impl = TracerClass()
data_dic = {}
old_trace_func = None

def parser_flag(flag):
    import re
    aa = re.split(r'_',flag)
    if len(aa) != 3 :
        return None,None
    return aa[1],aa[2]

class CheckFunctionTracer():
    def callEvent(self, frame):
        if 'check_value' == frame.getName():
            flag = frame.getArgumentInfo().getArgs()['check_flag']
            value = frame.getArgumentInfo().getArgs()['value']
            addr,register = parser_flag(flag)
            if addr in data_dic and register in data_dic[addr]:
                run_index = data_dic[addr][register]['run_index']
                data_len = len(data_dic[addr][register]['data'])
                if run_index >= data_len:
                    print('*** err : at address : {} . run_index : {} out of rang'.format(addr,run_index))
                    return
                if value == data_dic[addr][register]['data']['{}'.format(run_index + 1)] :
                    print('check : {} at {} times,match.'.format(addr,run_index + 1))
                    data_dic[addr][register]['run_index'] = run_index + 1

            # print("->>LoggingTracer : call " + frame.getName() + " from " + frame.getCaller().getName() + " @ " + str(frame.getCaller().getLineNumber()) + " args are " + str(frame.getArgumentInfo()))

# @ check_flag 为 携带了地址，和寄存器名称
# @ value 为当前需要 check 的值
# 在 sys.settracer设置的回调中，只接管此函数
def check_value(value,check_flag):
    pass

def set_trace_data(check_flag):
    global data_dic
    addr,register = parser_flag(check_flag)
    if not addr or not register :
        print('err : check_flag is wrong.')
        return

    if addr in data_dic:
        data_dic[addr][register] = {
            'data':{},
            'run_index':0  
        }
    else:
        addr_dic = {
            register:{
                'data':{},
                'run_index':0
            }
        }
        data_dic[addr] = addr_dic

def add_data_in_data_dic(addr,register,value):
    global data_dic
    cur_reg_dic = data_dic[addr][register]
    data_len = len(cur_reg_dic['data'])
    data_dic[addr][register]['data']['{}'.format(data_len + 1)] = value

def parser_trace_log_file(fileName):
    global data_dic
    file = open(fileName)
    while True:
        lines = file.readlines(100000)
        if not lines:
            break
        for line in lines:
            matchObj = re.match(r'\s*(\S+)\s+',line,re.M|re.I)
            if matchObj:
                addr = str(matchObj.group()).replace(' ','')
                if addr in data_dic:
                    reg = data_dic[addr]
                    for register in data_dic[addr].keys():
                        register_out = re.findall(register +r'  : (\S+)',line)
                        if register_out:
                            register_value = int(register_out[0],16)
                            add_data_in_data_dic(addr,register,register_value)

    file.close()
    # {'1234':{'1':0,"2":1}}  # flag : {...}  address:{'x0':{data:{},run_index:0},'x1':{data:{},run_index:0}}

def the_tracer_check_data(frame, event, args = None):
    global data_dic
    global tracer_impl

    code = frame.f_code

    func_name = code.co_name

    line_no = frame.f_lineno
    if tracer_impl is None:
        print('@@@ tracer_impl : None.')
        return None

    if event == 'call':
        tracer_impl.callEvent(FrameFancy(frame))

    return the_tracer_check_data

def enable(tracer_implementation=None):
    global tracer_impl,old_trace_func
    if tracer_implementation:
        tracer_impl = tracer_implementation  # 传递 工厂实力的对象
    old_trace_func = sys.gettrace()
    sys.settrace(the_tracer_check_data) # 注册回调到系统中

def check_run_ok():
    global data_dic
    for addr,addr_dic in data_dic.items():
        for _,reg_dic in addr_dic.items():
            if reg_dic['run_index'] == len(reg_dic['data']):
                print('->>> at {} check value is perfect.'.format(addr))
            else:
                print('*** err : at {} check {} times.'.format(addr,reg_dic['run_index']))

def disable():
    check_run_ok()
    global old_trace_func
    sys.settrace(old_trace_func)

```

> demo，如下:

```
#!/usr/bin/python3
# -*- encoding: utf-8 -*-
#@File    :   test.py
#@Time    :   2021/07/29 15:15:38
#@Author  :   wt

from wtpytracer import *

####################
## command :
##      python3 test.py ~/Desktop/1627533881instrace.log

## 定义需要检测的 变量 : flag + '_' + 地址 + '_' + 寄存器
Check_0x10000393c_w8 = 'Check_0x10000393c_w8'

### 翻译的测试代码
def f(x):
    ret = 0
    for index in range(x):
        ret = ret + index
        check_value(ret,Check_0x10000393c_w8) # check ret 和 0x10000393c 的 w8 的寄存器值
    return ret + x

if __name__ == '__main__':
    import sys
    args_list = sys.argv
    if len(args_list) != 2 :
        exit()
    file_name = args_list[1]

    try:
        set_trace_data(Check_0x10000393c_w8)
        parser_trace_log_file(file_name)
        enable(CheckFunctionTracer())
        f(5)
    finally:
        disable()

```

### 代码，在 lldb-trace 项目 / tools/verifyAlgorithm / 中, 这里就不继续写 git 地址了。

[check_com.png](forum.php?mod=attachment&aid=MjMxODkzNHwyMWJjNjRlM3wxNjI4MDY1MDE2fDIxMzQzMXwxNDg1NTEz&nothumb=yes) _(135.61 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxODkzNHwyMWJjNjRlM3wxNjI4MDY1MDE2fDIxMzQzMXwxNDg1NTEz&nothumb=yes)

2021-8-3 06:58 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

222

![](https://attach.52pojie.cn/forum/202108/03/065813ljpxuu4940crrxyc.png)

[ida_fun.png](forum.php?mod=attachment&aid=MjMxODkzM3w1ZjQ3MWIyZnwxNjI4MDY1MDE2fDIxMzQzMXwxNDg1NTEz&nothumb=yes) _(43.28 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxODkzM3w1ZjQ3MWIyZnwxNjI4MDY1MDE2fDIxMzQzMXwxNDg1NTEz&nothumb=yes)

2021-8-3 06:56 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

111

![](https://attach.52pojie.cn/forum/202108/03/065609mmdvy9zzfp198dv9.png)

[check_pintout.png](forum.php?mod=attachment&aid=MjMxODkzNXxjOTEyODBhMHwxNjI4MDY1MDE2fDIxMzQzMXwxNDg1NTEz&nothumb=yes) _(66.84 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjMxODkzNXxjOTEyODBhMHwxNjI4MDY1MDE2fDIxMzQzMXwxNDg1NTEz&nothumb=yes)

2021-8-3 06:59 上传 [![](https://static.52pojie.cn/static/image/common/rleft.gif)](javascript:;) [![](https://static.52pojie.cn/static/image/common/rright.gif)](javascript:;)

333

![](https://attach.52pojie.cn/forum/202108/03/065905gzgvxn98j53h8of8.png)![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)yangyss 大神的聚集地，又长见识了
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268633.htm)

> 基于 lldb 的 trace 脚本 <辅助算法分析，算法还原，以及算法验证>

由于工作的原因，在逆向分析中，经常遇到强混淆，vm 虚拟化等加固方案。在分析中，痛不欲生。为了能在逆向过程中，还能安心的喝口咖啡，同时还能分析 / 还原各种高混淆 / vm 虚拟化的代码。参考函数追踪的原理，弄了个指令级的 lldb-trace 脚本。
------------------------------------------------------------------------------------------------------------------------------

> **[前瞻]：**  
> 平时分析过程中，难免遇到 c/c++ 函数，以及 objc 函数。而 objc 函数中，又包含了 retain，release... 等函数。故而，在 trace 过程中，objc 函数形如，retain，release.. 等等都直接忽略。而对于 c++ 函数，类的构造函数，析构函数，系统函数等，也可以做过滤处理。对于 objc 函数 objc_msgsend，我们可以做重点分析：在于你需不需要分析这个函数。而我不需要，所以，我只需要知道 objc_msgsend 函数的函数即可，而对应的函数实现，我就不 trace 了。

### 框架分析:

> 1，对不同的平台而言，做不同的配置  
> 2，忽略的函数，都放在忽略的函数列表中  
> 3，objc_msgsend 函数需要特殊处理，故放在受保护的函数列表中  
> 4，trace 过程中，需要读取寄存器的值，所以，用正则匹配出上一条指令所有的寄存器

### 更新:

> 1，trace 指令 参数的优化，绝大部分参数，都有默认值  
> 2，tracing 中，结束地址可以有多个（在某些混淆情况下，不确定结束地址在哪，可以多设置几个结束地址，用 ";" 分割）  
> 3，增加了暂停其他线程的 可选参数  
> 4，增加了只 trace 本模块的 可选参数  
> 5，增加了 进度 信息（防止以为脚本卡死… 等的不耐心… 从而关闭了 lldb  
> 6，对 msg_send 函数的参数解开发中…  
> 7，增加了 对还原的算法检测脚本

脚本怎么使用:
-------

> 1, 在你准备追踪的地方下断点：（我的断点，从 breakpoint 函数 si 进入到 a 函数的 第一行）  
> ![](https://bbs.pediy.com/upload/attach/202107/748611_8JXMMAMNGHUY3RQ.png)  
> 2，导入 lldbTrace.py 脚本。（你可以设置，默认的 log 文件路径。如果不设置，默认和脚本同位置）  
> ![](https://bbs.pediy.com/upload/attach/202107/748611_CWY5Y29QTD7T65B.png)
> 
> 3，设置一个停止追踪的地址：（当前 a 函数，我把结束地址设为 最后地址，和 ret 地址。为了查看 debug 信息，我把 log 类型设置成了 debug）  
> ![](https://bbs.pediy.com/upload/attach/202107/748611_9VNXU5JG99Q5HK9.png)
> 
> 4，设置好，直接回车，结果如下：  
> ![](https://bbs.pediy.com/upload/attach/202107/748611_W4NWEK25E95S2P7.png)

### **脚本在 git 上:**

[基于 lldb 的汇编指令级 trace 脚本](https://github.com/yangyss/lldb-trace)

> 脚本还有很多不完善的地方，需要慢慢优化。  
> 不过利用 trace 结果，能还原 手写的算法，以及强混淆 或者 某些 vm 虚拟机。

### 某手撸 aes 算法的 trace 结果：

![](https://bbs.pediy.com/upload/attach/202107/748611_RZNWTHPYTY83E48.png)

基于 trace 脚本的基础上，对 trace 结果，以及实际分析中，做了一个 自动检测还原的函数 的脚本。利用脚本，不用每次都去动态调试，对照比较结果。
-----------------------------------------------------------------------------

### 算法还原检测脚本 简介:

```
#!/usr/bin/python3
# -*- encoding: utf-8 -*-
#@File    :   test.py
#@Time    :   2021/07/29 15:15:38
#@Author  :   wt
 
from wtpytracer import *
 
####################
## command :
##      python3 test.py ~/Desktop/1627533881instrace.log
 
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

### **具体操作如下:**

> **<a>**，, 在 ida 中，找到需要还原的算法 (可以是汇编，也可以是伪代码)，翻译成对应的 python 代码  
> ![](https://bbs.pediy.com/upload/attach/202107/748611_R5MCHG82KJE8KZ5.png)  
> **<b>**，利用 lldbTrace.py 脚本，把需要翻译的函数 trace 一哈。结果如下：  
> ![](https://bbs.pediy.com/upload/attach/202107/748611_8TCBSVU8FJF5EG4.png)  
> **<c>**，在 ida 伪代码中，切换到对应的汇编，对照 trace 结果，确定具体检测的地址。因为 trace，当前打印的是上一句代码执行后的寄存器值。所以，我们把 check 的地址，定义为 0x10000393c，寄存器为 w8  
> ![](https://bbs.pediy.com/upload/attach/202107/748611_AZEE9BVXST3KVXF.png)  
> **<d>**，, 定义检测的 变量 Check_0x10000393c_w8 = ‘Check_0x10000393c_w8’ 。在翻译的代码中，添加检测函数 check_value(ret,Check_0x10000393c_w8) 。在解析 tracelog 文件之前，设置 check 的相关信息 set_trace_data(Check_0x10000393c_w8)
> 
> **<e>**，用 python 调用此脚本，并传入 tracelog 文件的路径。结果如下:  
> ![](https://bbs.pediy.com/upload/attach/202107/748611_2GXP7CR349V2WJC.png)

### [](#模块代码：)模块代码：

```
#!/usr/bin/python3
# -*- encoding: utf-8 -*-
#@File    :   wtpytracer.py
#@Time    :   2021/07/27 18:17:18
#@Author  :   wt
 
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
        str_self = "%s @ %s" % (
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
                        register_out = re.findall(register +r'  : (\S+)',line)
                        if register_out:
                            register_value = int(register_out[0],16)
                            add_data_in_data_dic(addr,register,register_value)
 
    file.close()
    # {'1234':{'1':0,"2":1}}  # flag : {...}  address:{'x0':{data:{},run_index:0},'x1':{data:{},run_index:0}}
 
 
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
        tracer_impl = tracer_implementation  # 传递 工厂实力的对象
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

[[注意] 招人！base 上海，课程运营、市场多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)

[#逆向分析](forum-166-1-189.htm) [#工具脚本](forum-166-1-195.htm)
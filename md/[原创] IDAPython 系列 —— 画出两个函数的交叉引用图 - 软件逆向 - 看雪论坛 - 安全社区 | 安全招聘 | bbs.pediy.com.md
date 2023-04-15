> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-276867.htm)

> [原创] IDAPython 系列 —— 画出两个函数的交叉引用图

[原创] IDAPython 系列 —— 画出两个函数的交叉引用图

6 小时前 308

[举报](javascript:void(0);)

### [原创] IDAPython 系列 —— 画出两个函数的交叉引用图

 [![](https://bbs.kanxue.com/view/img/avatar.png)](user-home-669610.htm) [singhost](user-home-669610.htm) ![](https://bbs.kanxue.com/view/img/rank/4.png)  ![](http://passport.kanxue.com/pc/view/img/star_0.gif) [ 举报](javascript:void(0);) 6 小时前  308

第一次在看雪平台投稿，还是有点忐忑的，一直想分享些 idapython 的内容，网上现在的内容太少，都只是教了一些基础的用法。官方的文档和例子又比较散。希望我的分享能让大家更加快乐的逆向吧，哈哈！

 

这是这个系列的第一篇，分享一下如何用 IDAPython 画出两个函数之间的交叉引用图，可视化一直是我很喜欢的一个东西，逆向的目的就是理解代码嘛，可视化是理解代码的一个很重要的工具。

[](#开发环境：)开发环境：
===============

*   Python3.8
*   IDA Pro 7.7+

此外，本教程的实现还依赖了几个 Python 包：

*   sark
*   networkx
*   matplotlib

[](#本教程安排如下：)本教程安排如下：
=====================

1.  定义递归函数 find_cross_refs
2.  IDAPython Action 概念介绍
3.  添加我们的 Action
4.  最终效果

一、定义递归函数 find_cross_refs
========================

在这里，我们定义了一个名为 find_cross_refs 的递归函数，用于查找两个函数之间的交叉引用关系。该函数包含以下参数：

*   func: sark.Function 类型，表示要查找交叉引用关系的起始函数对象。
*   target_func: sark.Function 类型，表示要查找交叉引用关系的目标函数对象。
*   G: networkx.DiGraph 类型，表示用于存储函数之间引用关系的有向图对象。
*   max_depth: int 类型，表示查找引用关系的最大深度。
*   include_data_xref: bool 类型，表示在查找引用关系时是否要包含数据引用关系。

函数中主要分为以下几个步骤：

*   首先判断查找引用关系的深度是否达到最大深度，如果达到最大深度就直接返回 False。
*   然后根据 include_data_xref 的设置，获取该函数中所有的引用 refes。如果 include_data_xref 为 True，则获取所有数据引用关系，否则获取所有函数调用引用关系。
*   遍历函数的所有引用 ref，如果该引用 ref 指向目标函数，则在有向图 G 中通过 add_edge 函数添加一条从当前函数到目标函数的边，并返回 True。
*   如果引用指向另一个函数，则递归调用 find_cross_refs 函数查找两个函数之间的交叉引用关系。
*   如果所有引用遍历完，仍然没有找到交叉引用，则返回 False。

下面是 find_cross_refs 函数的详细实现代码：

```
MAX_SEARCH_DEPTH = 10
# 定义递归函数，用于查找两个函数之间的交叉引用关系
def find_cross_refs(func: sark.Function, target_func, G, max_depth, include_data_xref):
    if max_depth == 0:
        return False
    max_depth -= 1
 
    # 获取函数的所有引用
    if include_data_xref:
        refs = list(func.xrefs_from)
    else:
        refs = list(func.calls_from)
    # 遍历函数的每一个引用
    for ref in refs:
        # 获取引用的地址
        ref_addr = ref.to  # type: ignore
 
        # 判断引用是否指向目标函数
        if ref_addr == target_func.start_ea:
            # 如果指向目标函数，则找到了交叉引用关系
            # 在图中添加一条边表示函数之间的引用关系
            # G.add_node(func, name=func.demangled)
            # G.add_node(target_func, name=target_func.demangled)
            G.add_edge(func.ea, target_func.ea)
            return True
 
        # 如果引用指向另一个函数，则递归调用find_cross_refs函数，查找两个函数之间的交叉引用关系
 
        seg = sark.Segment(ea=ref_addr)
        if seg.name != "__text":
            #是 external Function
            continue
        else:
            try:
                ref_func = sark.Function(ea=ref_addr)
            except Exception as e:
                print(e)
                continue
        if find_cross_refs(ref_func, target_func, G, max_depth, include_data_xref):
            # 如果找到了交叉引用关系，则在图中添加一条边表示函数之间的引用关系
            # G.add_node(func, name=func.demangled)
            # G.add_node(ref_func, name=ref_func.demangled)
            G.add_edge(func.ea, ref_func.ea)
            return True
 
    # 如果遍历完所有引用后仍然没有找到交叉引用关系，则返回False
    return False

```

二、IDApython Action 概念介绍
=======================

第二步，我们要向 IDA 中添加 action，通过 action 来执行我们的 find_cross_refs 函数，首先我们了解一下 IDA 中 action 的概念：

1.  action 首先需要被注册。一旦被注册，action 可以用快捷键触发（如果指定了的话），但是 action 还没法在 UI 中看到
2.  action 被注册之后，我们可以将 action attach 到以下三个地方
    1.  主菜单
    2.  Toolbar
    3.  上下文菜单
3.  同一个 action 可以在不同的地方被使用
4.  一个 action 有一个 handler，是一个结构体带有以下两个 callback
    1.  activate callback，当 action 触发时调用
    2.  update callback，声明 action 是否 enabled

给一段代码，看一下如何注册一个 Action

```
# 1) Create the handler class
    class MyHandler(idaapi.action_handler_t):
        def __init__(self):
            idaapi.action_handler_t.__init__(self)
        # Say hello when invoked.
        def activate(self, ctx):
            print "Hello!"
            return 1
        # This action is always available.
        def update(self, ctx):
            return idaapi.AST_ENABLE_ALWAYS
    # 2) Describe the action
    action_desc = idaapi.action_desc_t(
        'my:action',   # The action name. This acts like an ID and must be unique
        'Say hello!',  # The action text.
        MyHandler(),   # The action handler.
        'Ctrl+H',      # Optional: the action shortcut
        'Says hello',  # Optional: the action tooltip (available in menus/toolbar)
        199)           # Optional: the action icon (shows when in menus/toolbars)
    # 3) Register the action
    idaapi.register_action(action_desc)

```

然后是将 action 放到 UI 上的代码

```
# 将 action 放到菜单中
idaapi.attach_action_to_menu(
        'Edit/Other/Manual instruction...', # The relative path of where to add the action
        'my:action',                        # The action ID (see above)
        idaapi.SETMENU_APP)                 # We want to append the action after the 'Manual instruction...'
# 将 action 放到 toolbar 中
idaapi.attach_action_to_toolbar(
        "AnalysisToolBar",  # The toolbar name
        'my:action')        # The action ID
 
# 将 action 永久的放到某个 widget 的上下文菜单中
# Create a widget, or retrieve a pointer to it.
    form = idaapi.get_current_tform()
    idaapi.attach_action_to_popup(form, None, "my:action", None)

```

了解完这些我们就可以写自己的 Action 了，为了方便我对官方的 Action api 做了一些封装, 下面是我自己的 action.py 文件

```
import idaapi
 
from .hookers import global_hooker_manager
from typing import Optional
 
class ActionManager(object):
    def __init__(self):
        self.__actions = []
 
    def register(self, action):
        self.__actions.append(action)
        idaapi.register_action(
                idaapi.action_desc_t(action.name, action.description, action, action.hotkey)
            )
        if isinstance(action, HexRaysPopupAction):
            global_hooker_manager.register(HexRaysPopupRequestHandler(action))
 
    def initialize(self):
        pass
 
    def finalize(self):
        for action in self.__actions:
            idaapi.unregister_action(action.name)
 
 
action_manager = ActionManager()
 
class Action(idaapi.action_handler_t):
    """
    Convenience wrapper with name property allowing to be registered in IDA using ActionManager
    """
    description: Optional[str] = None
    hotkey: Optional[str] = None
 
    def __init__(self):
        super(Action, self).__init__()
 
    @property
    def name(self):
        return "HexRaysPyTools:" + type(self).__name__
 
    def activate(self, ctx: idaapi.action_ctx_base_t):
        raise NotImplementedError
 
    def update(self, ctx: idaapi.action_ctx_base_t):
        # return idaapi.AST_XXX
        # 通常会根据  ctx.widget_type == idaapi.BWN_XXX 来判断当前是在哪个窗口
        raise NotImplementedError
 
 
class HexRaysPopupAction(Action):
    """
    Wrapper around Action. Represents Action which can be added to menu after right-clicking in Decompile window.
    Has `check` method that should tell whether Action should be added to popup menu when different items
    are right-clicked.
    Children of this class can also be fired by hot-key without right-clicking if one provided in `hotkey`
    static member.
    """
 
    def __init__(self):
        super(HexRaysPopupAction, self).__init__()
 
    def activate(self, ctx: idaapi.action_ctx_base_t):
        raise NotImplementedError
 
    def check(self, hx_view):
        # type: (idaapi.vdui_t) -> bool
        raise NotImplementedError
 
    def update(self, ctx: idaapi.action_ctx_base_t):
        if ctx.widget_type == idaapi.BWN_PSEUDOCODE:
            return idaapi.AST_ENABLE_FOR_WIDGET
        return idaapi.AST_DISABLE_FOR_WIDGET
 
 
class HexRaysPopupRequestHandler(idaapi.Hexrays_Hooks):
    def __init__(self, action):
        super().__init__()
        self.__action = action
 
    def populating_popup(self, widget, popup_handle, vu):
        if self.__action.check(vu):
            idaapi.attach_action_to_popup(widget, popup_handle, self.__action.name, None)
        return 0

```

三、添加我们的 Action
==============

然后我们开始定义显示交叉引用图的 Action，我把它叫做 XrefRoadMapAction

 

具体来说，该类包含以下方法：

*   check 方法：返回 True 表示该操作可用。
*   activate 方法：执行实际的操作，该方法获取当前函数地址，获取用户输入的目标函数地址，并创建一个有向图进行处理。
*   update 方法：更新当前操作的 UI（例如菜单项）的图形状态。

可以看出，该类实现的核心是 activate 方法。其主要流程如下：

*   获取当前光标所在的函数对象；
*   获取用户输入的目标函数地址；
*   创建空的有向图对象 G；
*   根据用户设置调用 find_cross_refs 函数搜索从当前函数到目标函数的交叉引用路径；
*   如果在搜索结果中找到了交叉引用路径，则在 G 中添加表示函数之间引用关系的边，然后使用 sark.ui.NXGraph 绘制有向图；
*   如果没有找到路径，则显示一个警告信息。

下面是 XrefRoadMapAction 类的详细实现代码：

```
class XrefRoadMapAction(HexRaysPopupAction):
    """
    Convenience wrapper with name property allowing to be registered in IDA using ActionManager
    """
    description: Optional[str] = "Find Xref Road map between A and B"
    hotkey: Optional[str] = "Alt-F3"
 
    def __init__(self):
        super(Action, self).__init__()
 
    def check(self, hx_view: idaapi.vdui_t) -> bool:
        return True
 
    def activate(self, ctx: idaapi.action_ctx_base_t):
        # 获取光标所在的函数地址
        cur_ea = idaapi.get_screen_ea()
        try:
            cur_func = sark.Function(ea=cur_ea)
        except Exception as e:
            traceback.print_exc()
            # 如果光标不在函数中，则提示错误
            print("Error: cursor is not in a function!")
            return
 
        # 获取用户输入的目标函数的地址
        target_func_addr = idaapi.ask_addr(defval=0, format="Enter target function address:")
        if target_func_addr == idaapi.BADADDR or target_func_addr is None:
            # 如果用户输入的是无效的地址，则提示错误
            print("Error: invalid target function address!")
            return
 
        # 获取目标函数的函数对象
        try:
            s = sark.Segment(ea=target_func_addr)
            if s.name == "UNDEF":
                target_func = sark.ExternFunction(ea=target_func_addr)
            else:
                target_func = sark.Function(ea=target_func_addr)  # type: ignore
        except Exception as e:
            # 如果目标函数不存在，则提示错误
            traceback.print_exc()
            print("Error: target function does not exist!")
            return
 
 
        import networkx as nx
        # 创建有向图
        G = nx.DiGraph()
 
        # 在这里实现查找两个函数之间交叉引用关系的代码...
        print(f"find a path between func {cur_func.demangled} and {target_func.demangled}, max search depth is {MAX_SEARCH_DEPTH}")
        btn_selected = idaapi.ask_yn(idaapi.ASKBTN_NO, "是否包括 data xrefs?")
        if btn_selected == idaapi.ASKBTN_CANCEL:
            return
        ret = find_cross_refs(cur_func, target_func, G, MAX_SEARCH_DEPTH, include_data_xref=True if btn_selected == idaapi.ASKBTN_YES else False)
        if ret == False:
            assert len(G.nodes()) == 0, "find_cross_refs 返回 False，但是 G 中有节点"
 
        if len(G.nodes()) > 0:
            # 绘制图
 
            G.nodes[cur_func.ea][sark.ui.NXGraph.BG_COLOR] = 0x80
 
            title = f"Path from `{cur_func.demangled}` to `{target_func.demangled}`"
 
            # Create an NXGraph viewer
            viewer = sark.ui.NXGraph(G, handler=sark.ui.AddressNodeHandler(), title=title)
 
            # Show the graph
            viewer.Show()
        else:
            idaapi.warning("Cannot find a path!!!")
 
    def update(self, ctx: idaapi.action_ctx_base_t):
 
        # 获取当前窗口的类型
        widget_type = ctx.widget_type
 
        # 如果当前窗口是反汇编或者反编译窗口，则菜单项可用
        if widget_type == idaapi.BWN_DISASM or widget_type == idaapi.BWN_PSEUDOCODE:
            return idaapi.AST_ENABLE_FOR_WIDGET
        else:
            return idaapi.AST_DISABLE_FOR_WIDGET

```

代码里面其实还用了很多 sark 库的函数，大家可以自己学习其用法，这个库是专门对 idapython 的 api 做封装的，比较 pythonic，非常好用，我自己也给这个库添加了很多新的功能。

[](#四、最终效果)四、最终效果
=================

如下所示：  
![](https://bbs.kanxue.com/upload/attach/202304/669610_93AQNH3S98TTNRX.png)  
![](https://bbs.kanxue.com/upload/attach/202304/669610_4SN3EMGD4ZH8Q46.png)  
![](https://bbs.kanxue.com/upload/attach/202304/669610_45GDJ7XFU9MRWMB.png)

  

[议题征集启动！看雪 · 第七届安全开发者峰会](https://bbs.kanxue.com/thread-276136.htm)

[#调试逆向](forum-4-1-1.htm)
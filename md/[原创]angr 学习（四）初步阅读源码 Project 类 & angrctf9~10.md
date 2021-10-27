> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269990.htm)

> [原创]angr 学习（四）初步阅读源码 Project 类 & angrctf9~10

> *   源码版本：9.0.10281

 

一般 angr 的利用脚本首先都是创建一个 Project 对象，如下所示。Project 类是 angr 模块的主类，它包含了一组二进制文件及它们之间的关系，并对它们执行分析。

```
>>> project = angr.Project("/home/lzx/tools/angr_ctf/solutions/00_angr_find/00_angr_find")

```

1 __init__ （构造函数）
-----------------

Project 类的__init__函数调用图。  
![](https://l-tuchuang.oss-cn-beijing.aliyuncs.com/img/20211027122437.png)

### 1.1 参数

```
class Project:
    """
    This is the main class of the angr module. It is meant to contain a set of binaries and the relationships between
    them, and perform analyses on them. 这是angr模块的主类。它旨在包含一系列二进制文件和它们之间的关系，并基于它们进行分析
 
    :param thing:                       The path to the main executable object to analyze, or a CLE Loader object. 要分析的主要的可执行对象的路径，或CLE Loader对象，这个是必须指定的参数
 
    The following parameters are optional. 以下参数是可选的
 
    :param default_analysis_mode:       The mode of analysis to use by default. Defaults to 'symbolic'. 默认使用的分析模式， 默认为 ‘symbolic’
 
    :param ignore_functions:            A list of function names that, when imported from shared libraries, should
                                        never be stepped into in analysis (calls will return an unconstrained value).
                                        是一个函数名称列表，当列表里的这些函数从共享库导入后，不会被进入分析（调用将返回一个非约束值），简单来说就是传入一个要忽略的函数列表
    :param use_sim_procedures:          Whether to replace resolved dependencies for which simprocedures are
                                        available with said simprocedures.
                                        是否使用符号摘要替换库函数，默认是开启的，但是因为sim是模拟库函数可能存在精确度和准确性问题
    :param exclude_sim_procedures_func: A function that, when passed a function name, returns whether or not to wrap
                                        it with a simprocedure.
                                        不需要被替换的库函数
    :param exclude_sim_procedures_list: A list of functions to *not* wrap with simprocedures.
                                        不用sim替换的函数列表
    :param arch:                        The target architecture (auto-detected otherwise). 目标架构
    :param simos:                       a SimOS class to use for this project.
                                        确定 guest OS。创建了一个 angr.SimOS 或者其子类实例有以下定义：SimLinux、SimWindows、SimCGC、SimJavaVM、SimUserland。
 
    :param engine:                      The SimEngine class to use for this project.
                                        指定要使用的SimEngine引擎类型：failure engine、syscall engine、hook engine、unicorn engine、VEX engine。
                                        angr使用一系列引擎（SimEngine的子类）来模拟被执行代码对输入状态产生的影响。源码位于 angr/engines 目录下。
    :param bool translation_cache:      If True, cache translated basic blocks rather than re-translating them.
                                        布尔变量，如果为True，则缓存已转化的基本块，而不是重新转化它们，简单来说就是否开启对于基本块的缓存
    :param support_selfmodifying_code:  Whether we aggressively support self-modifying code. When enabled, emulation
                                        will try to read code from the current state instead of the original memory,
                                        regardless of the current memory protections.
    :type support_selfmodifying_code:   bool
                                        布尔变量，设定是否支持自修改代码。启用后，无论当前的内存保护如何，仿真都会尝试从当前状态而不是原始内存中读取代码
    :param store_function:              A function that defines how the Project should be stored. Default to pickling.
                                        一个函数，定义如何存储Project，默认为pickling方式（Python中的pickle，序列化对象并保存到磁盘中）
    :param load_function:               A function that defines how the Project should be loaded. Default to unpickling.
                                        一个函数，定义如何加载Project，默认是unpicklink方式（也就是反序列化）
    :param analyses_preset:             The plugin preset for the analyses provider (i.e. Analyses instance).
    :type analyses_preset:              angr.misc.PluginPreset
                                        设置project的插件，定义在angr.misc.PluginPreset
 
    Any additional keyword arguments passed will be passed onto ``cle.Loader``. 任何额外的关键参数都会传递给cle.Loader,angr 中的 CLE 模块用于将二进制文件载入虚拟地址空间，而CLE 最主要的接口就是 loader 类
 
    :ivar analyses:     The available analyses. analyses模块
    :type analyses:     angr.analysis.Analyses
    :ivar entry:        The program entrypoint. 程序入口点
    :ivar factory:      Provides access to important analysis elements such as path groups and symbolic execution results. 提供对重要分析元素的访问
    :type factory:      AngrObjectFactory
    :ivar filename:     The filename of the executable. 可执行文件的文件名
    :ivar loader:       The program loader. 程序加载器
    :type loader:       cle.Loader
    :ivar storage:      Dictionary of things that should be loaded/stored with the Project.
    :type storage:      defaultdict(list)
    """
 
    def __init__(self, thing,
                 default_analysis_mode=None,
                 ignore_functions=None,
                 use_sim_procedures=True,
                 exclude_sim_procedures_func=None,
                 exclude_sim_procedures_list=(),
                 arch=None, simos=None,
                 engine=None,
                 load_options: Dict[str, Any]=None,
                 translation_cache=True,
                 support_selfmodifying_code=False,
                 store_function=None,
                 load_function=None,
                 analyses_preset=None,
                 concrete_target=None,
                 **kwargs):

```

### 1.2 主体

![](https://l-tuchuang.oss-cn-beijing.aliyuncs.com/img/20211027125337.png)

### 1.3 细节

#### Step1：加载二进制文件，获得一个 cle.Loader 实例。

```
# Step 1: Load the binary 加载二进制文件，对输入的二进制文件进行初始化处理，最终目的是获得一个 cle.Loader 实例
 
# 二进制的装载组建是CLE（CLE Load Everything)，它负责装载二进制对象以及它所依赖的库，将自身无法执行的操作转移给angr的其它组件，最后生成地址空间，表示该程序已加载并可以准备运行。
# cle.loader代表着将整个程序映射到某个地址空间，而地址空间的每个对象都可以由一个加载器后端加载
 
if load_options is None: load_options = {}
 
load_options.update(kwargs) # 更新load_options
if arch is not None: # 如果设置了架构，则将架构加入到load_options
    load_options.update({'arch': arch})
if isinstance(thing, cle.Loader): # 对加载的 thing 做判断，如果是一个 cle.Loader 类实例，则将其设置为 self.loader 成员变量的值
    if load_options:
        l.warning("You provided CLE options to angr but you also provided a completed cle.Loader object!")
    self.loader = thing
    self.filename = self.loader.main_object.binary
elif isinstance(thing, cle.Backend): #
    self.filename = thing.binary
    self.loader = cle.Loader(thing, **load_options)
elif hasattr(thing, 'read') and hasattr(thing, 'seek'): # 否则如果thing是一个流，则创建一个新的 cle.Loader
    l.info("Loading binary from stream")
    self.filename = None
    self.loader = cle.Loader(thing, **load_options)
elif not isinstance(thing, (str, Path)) or not os.path.exists(thing) or not os.path.isfile(thing): # 否则如果thing是一个无效二进制文件，则打印错误信息
    raise Exception("Not a valid binary file: %s" % repr(thing))
else: # 否则如果thing是一个有效的二进制文件，则创建一个新的 cle.Loader
    # use angr's loader, provided by cle
    l.info("Loading binary %s", thing)
    self.filename = str(thing)
    self.loader = cle.Loader(self.filename, concrete_target=concrete_target, **load_options)

```

`cle.loader`代表着将整个程序映射到某个地址空间，而地址空间的每个对象（比如，下方的 main_object 是 00_angr_find）都可以由一个加载器后端加载。通过 loader 可查看二进制文件加载的共享库，以及执行对加载地址空间相关的基本查询。

```
>>> project = angr.Project("/home/lzx/tools/angr_ctf/solutions/00_angr_find/00_angr_find")
 
>>> project.loader
 >>> hex(project.loader.max_addr)
'0x8607fff'
 
>>> hex(project.loader.min_addr)
'0x8048000'
 
>>> project.loader.shared_objects
OrderedDict([('00_angr_find', ), ('libc.so.6', ), ('ld-linux.so.2', ), ('extern-address space', ), ('cle##tls', )])
 
>>> project.loader.all_objects
[, , , , , ]
 
 
>>> project.loader.main_object
 >>> project.loader.main_object.execstack
False
 
>>> project.loader.main_object.pic
False 
```

#### [](#step2：判断二进制文件的cpu架构。)Step2：判断二进制文件的 CPU 架构。

```
# Step 2: determine its CPU architecture, ideally falling back to CLE's guess 判断二进制文件的 CPU 架构 。
 
# archinfo是一个Python的第三方库用来判断二进制文件的目标架构
 
if isinstance(arch, str): # 如果输入的arch是字符串类型，则从 archinfo 里匹配后，再赋值给成员变量self.arch
    self.arch = archinfo.arch_from_id(arch)  # may raise ArchError, let the user see this
elif isinstance(arch, archinfo.Arch): # 如果输入的 arch 是 archinfo.Arch 类型，则直接赋值给成员变量self.arch
    self.arch = arch # type: archinfo.Arch
elif arch is None: # 如果没有输入arch，则从 self.loader 获取后赋值给成员变量self.arch
    self.arch = self.loader.main_object.arch
else: # 否则，异常
    raise ValueError("Invalid arch specification.")

```

```
>>> project.loader.main_object.arch
 >>> project.arch 
```

#### Step3 : 设置部分默认的属性，以及设置 public 和 private 的属性。

```
# Step 3: Set some defaults and set the public and private properties 对相关的默认、公共和私有属性进行设置
if not default_analysis_mode: # 首先就是对默认使用的分析模式
    default_analysis_mode = 'symbolic'
if not ignore_functions: # 需要忽略替换的函数列表
    ignore_functions = []
 
if isinstance(exclude_sim_procedures_func, types.LambdaType): # exclude_sim_procedures_func-不需替换的库函数
    l.warning("Passing a lambda type as the exclude_sim_procedures_func argument to "
              "Project causes the resulting object to be un-serializable.")  # 如果传递lambda类型的exclude_sim_procedures_func参数给Project，则会导致生成的对象不可序列化
 
self._sim_procedures = {}
 
self.concrete_target = concrete_target
 
# It doesn't make any sense to have auto_load_libs 参数auto_load_libs没有意义
# if you have the concrete target, let's warn the user about this. 如果有具体的目标，则就这点警告用户
if self.concrete_target and load_options.get('auto_load_libs', None):
 
    l.critical("Incompatible options selected for this project, please disable auto_load_libs if "
               "you want to use a concrete target.")  # critical 当发生严重错误，导致应用程序不能继续运行时记录的信息
    raise Exception("Incompatible options for the project")
 
if self.concrete_target and self.arch.name not in ['X86', 'AMD64', 'ARMHF']: # 如果有具体的目标，但是cpu架构不是【x86、amd64、armhf】
    l.critical("Concrete execution does not support yet the selected architecture. Aborting.")
    raise Exception("Incompatible options for the project")
 
self._default_analysis_mode = default_analysis_mode  # 默认分析模式
self._exclude_sim_procedures_func = exclude_sim_procedures_func # 不需被SimProcedure中符号摘要替换的库函数
self._exclude_sim_procedures_list = exclude_sim_procedures_list # 不需被SimProcedure中符号摘要替换的库函数列表
self.use_sim_procedures = use_sim_procedures # 是否使用符号摘要替换库函数
self._ignore_functions = ignore_functions # 要忽略的函数列表
self._support_selfmodifying_code = support_selfmodifying_code # 是否支持自修改代码
self._translation_cache = translation_cache # 是否开启对于基本块的缓存
self._executing = False # this is a flag for the convenience API, exec() and terminate_execution() below
 
if self._support_selfmodifying_code: # 如果支持自修改代码，那么确保关闭对基本块的缓存
    if self._translation_cache is True:
        self._translation_cache = False
        l.warning("Disabling IRSB translation cache because support for self-modifying code is enabled.")
 
self.entry = self.loader.main_object.entry # 程序入口点
self.storage = defaultdict(list)
self.store_function = store_function or self._store
self.load_function = load_function or self._load

```

auto_load_libs 没有意义。如果同时传入 concrete_target 和 auto_load_libs，那就会警告用户选项 auto_load_libs 无效。  
![](https://l-tuchuang.oss-cn-beijing.aliyuncs.com/img/20211027122514.png)

#### Step4 : 设置 Project 的各种插件。

```
# Step 4: Set up the project's hubs 设置Project的各种插件，CLE模块会将自身无法执行的操作转移给angr的其它组件，这里就是对于CLE分析的一些组件的初始化
# Step 4.1 Factory
# 第一步设置的就是angr中最重要的Factory组件，factory有几个方便的构造函数，用于经常使用的常见对象。这一步从参数、loader、arch 或者默认值中获取预设的引擎，创建了一个 angr.EngineHub 类实例
self.factory = AngrObjectFactory(self, default_engine=engine)
 
# Step 4.2: Analyses
# 第二步主要是从参数或者默认值中获取预设的分析。创建了一个 angr.AnalysesHub 类实例，angr 内置了一些分析方法，用于提取程序信息，接口位于 proj.analyses. 中
self._analyses_preset = analyses_preset
self.analyses = None
self._initialize_analyses_hub()
 
# Step 4.3: ...etc
self.kb = KnowledgeBase(self, )

```

*   AngrObjectFactory

```
class AngrObjectFactory:
    """
    This factory provides access to important analysis elements.
    该工厂提供对重要分析元素的访问
    """
    def __init__(self, project, default_engine=None):
        if default_engine is None:
            default_engine = UberEngine
 
        self.project = project
        self._default_cc = DEFAULT_CC.get(project.arch.name, SimCCUnknown)
        self.default_engine = default_engine(project)
        self.procedure_engine = ProcedureEngine(project)
 
        if project.concrete_target:
            self.concrete_engine = SimEngineConcrete(project)
        else:
            self.concrete_engine = None

```

```
>>> project.factory
 >>> project.factory.project
 >>> project.factory._default_cc
 >>> project.factory.default_engine
 >>> project.factory.procedure_engine 
```

*   AnalysesHub

```
def _initialize_analyses_hub(self):
     """
     Initializes self.analyses using a given preset.
     """
     self.analyses = AnalysesHub(self)
     self.analyses.use_plugin_preset(self._analyses_preset if self._analyses_preset is not None else 'default')

```

```
class AnalysesHub(PluginVendor):
    """
    This class contains functions for all the registered and runnable analyses,
    """
    def __init__(self, project):
        super(AnalysesHub, self).__init__()
        self.project = project

```

*   KnowledgeBase

```
class KnowledgeBase:
    """Represents a "model" of knowledge about an artifact.
 
    Contains things like a CFG, data references, etc.
    """
    functions: 'FunctionManager'
    variables: 'VariableManager'
    structured_code: 'StructuredCodeManager'
    defs: 'KeyDefinitionManager'
    cfgs: 'CFGManager'
    _project: 'Project'
    types: 'TypesStore'
 
    def __init__(self, project, obj=None, name=None):
        if obj is not None:
            l.warning("The obj parameter in KnowledgeBase.__init__() has been deprecated.")
        object.__setattr__(self, '_project', project)
        object.__setattr__(self, '_plugins', {})
 
        self.name = name if name else ("kb_%d" % next(kb_ctr))

```

#### Step5：确定 guest OS

```
# Step 5: determine the guest OS  确定 guest OS。创建了一个 angr.SimOS 或者其子类实例
if isinstance(simos, type) and issubclass(simos, SimOS):
    self.simos = simos(self) #pylint:disable=invalid-name
elif isinstance(simos, str):
    self.simos = os_mapping[simos](self)
elif simos is None:
    self.simos = os_mapping[self.loader.main_object.os](self)
else:
    raise ValueError("Invalid OS specification or non-matching architecture.")
 
self.is_java_project = isinstance(self.arch, ArchSoot)
self.is_java_jni_project = isinstance(self.arch, ArchSoot) and self.simos.is_javavm_with_jni_support

```

```
>>> project.simos
 
```

#### [](#step6：根据库函数注册合适的simprocedures)Step6：根据库函数注册合适的 simprocedures

为了缓解路径爆炸以及使符号执行的实践性更强，angr 用 Python 写的函数摘要（summary）来替换库函数，高效地模拟库函数对状态的影响。而 Simprocedures 对象里包含了函数摘要。比如 angr 用 <SimProcedure strcmp> 对象替换库函数 strcmp。当然，Simprocedures 不仅可用于替换库函数，也可替换用户自定义函数。也就是说，Simprocedures 用于 hook。  
Simprocedures 可以缓解路径爆炸，反过来也可能引入路径爆炸。比如，strlen 的参数是符号字符串。​

```
# Step 6: Register simprocedures as appropriate for library functions 根据库函数适当地注册 simprocedures
if isinstance(self.arch, ArchSoot) and self.simos.is_javavm_with_jni_support:
    # If we execute a Java archive that includes native JNI libraries,
    # we need to use the arch of the native simos for all (native) sim
    # procedures.
    sim_proc_arch = self.simos.native_arch
else:
    sim_proc_arch = self.arch
for obj in self.loader.initial_load_objects:
    self._register_object(obj, sim_proc_arch) # 调用了内部函数 _register_object，这个函数将尽可能地用angr库中的实现的符号摘要将程序中的库函数与替换掉

```

```
>>> project.loader.initial_load_objects
[, , ]
 
>>> project.loader.initial_load_objects[0].imports.values()
dict_values([, , , , , , , ]) 
```

```
def _register_object(self, obj, sim_proc_arch):
    """
    This scans through an objects imports and hooks them with simprocedures from our library whenever possible
    扫描一个对象的导入并尽可能地使用 simprocedures hook他们
    """
 
    # Step 1: get the set of libraries we are allowed to use to resolve unresolved symbols 获取可以用来解析未解析符号的库集，即获取angr已经实现符号摘要的库函数
    missing_libs = []
    for lib_name in self.loader.missing_dependencies:
        try:
            missing_libs.append(SIM_LIBRARIES[lib_name])
        except KeyError:
            l.info("There are no simprocedures for missing library %s :(", lib_name)
    # additionally provide libraries we _have_ loaded as a fallback fallback
    # this helps in the case that e.g. CLE picked up a linux arm libc to satisfy an android arm binary
    for lib in self.loader.all_objects:
        if lib.provides in SIM_LIBRARIES:
            simlib = SIM_LIBRARIES[lib.provides]
            if simlib not in missing_libs:
                missing_libs.append(simlib)
 
    # Step 2: Categorize every "import" symbol in each object. 给每个object中的重要符号 分类，即分析程序中的导入函数
    # If it's IGNORED, mark it for stubbing 如果某个符号是忽略函数列表中的某个，标记为stubbing，不替换
    # If it's blacklisted, don't process it 如果某个符号处在黑名单，不处理
    # If it matches a simprocedure we have, replace it 如果某个符号匹配某个simprocedure，替换
    for reloc in obj.imports.values():
        # Step 2.1: Quick filter on symbols we really don't care about  快速过滤掉不关心的符号
        func = reloc.symbol
        if func is None: # None
            continue
        if not func.is_function and func.type != cle.backends.symbol.SymbolType.TYPE_NONE: # 符号不是函数且其类型不等于SymbolType.TYPE_NONE
            continue
        if func.resolvedby is None:
            # I don't understand the binary which made me add this case. If you are debugging and see this comment,
            # good luck.
            # ref: https://github.com/angr/angr/issues/1782
            # (I also don't know why the TYPE_NONE check in the previous clause is there but I can't find a ref for
            # that. they are probably related.)
            continue
        if not reloc.resolved: # 如果没有解析，就猜测simprocedure
            # This is a hack, effectively to support Binary Ninja, which doesn't provide access to dependency
            # library names. The backend creates the Relocation objects, but leaves them unresolved so that
            # we can try to guess them here. Once the Binary Ninja API starts supplying the dependencies,
            # The if/else, along with Project._guess_simprocedure() can be removed if it has no other utility,
            # just leave behind the 'unresolved' debug statement from the else clause.
 
            # Binary Ninja 提供了一个可供操作二进制文件的平台，甚至还可以在平台的基础上基于API来编写更方便的脚本和插件
            if reloc.owner.guess_simprocs:
                l.debug("Looking for matching SimProcedure for unresolved %s from %s with hint %s",
                        func.name, reloc.owner, reloc.owner.guess_simprocs_hint)
                self._guess_simprocedure(func, reloc.owner.guess_simprocs_hint)
            else:
                l.debug("Ignoring unresolved import '%s' from %s ...?", func.name, reloc.owner)
            continue
        export = reloc.resolvedby
        if self.is_hooked(export.rebased_addr): # 判断该函数是否已经被hook，如果已经被hook就不进行下面的操作，换下一个函数
            l.debug("Already hooked %s (%s)", export.name, export.owner)
            continue
 
        # Step 2.2: If this function has been resolved by a static dependency,
        # check if we actually can and want to replace it with a SimProcedure.
        # We opt out of this step if it is blacklisted by ignore_functions, which
        # will cause it to be replaced by ReturnUnconstrained later.
        # 如果此函数已经由静态依赖项解析，检查是否真的可以并想用SimProcedure替换它，如果
        if export.owner is not self.loader._extern_object and \
                export.name not in self._ignore_functions:
            if self._check_user_blacklists(export.name): # 如果该函数在黑名单中，结束操作
                continue
            owner_name = export.owner.provides
            if isinstance(self.loader.main_object, cle.backends.pe.PE): # 如果是PE文件，owner_name变小写
                owner_name = owner_name.lower()
            if owner_name not in SIM_LIBRARIES:
                continue
            sim_lib = SIM_LIBRARIES[owner_name]
            if not sim_lib.has_implementation(export.name):
                continue
            l.info("Using builtin SimProcedure for %s from %s", export.name, sim_lib.name)
            self.hook_symbol(export.rebased_addr, sim_lib.get(export.name, sim_proc_arch))
 
        # Step 2.3: If 2.2 didn't work, check if the symbol wants to be resolved
        # by a library we already know something about. Resolve it appropriately.
        # Note that _check_user_blacklists also includes _ignore_functions.
        # An important consideration is that even if we're stubbing a function out,
        # we still want to try as hard as we can to figure out where it comes from
        # so we can get the calling convention as close to right as possible.
        elif reloc.resolvewith is not None and reloc.resolvewith in SIM_LIBRARIES:
            sim_lib = SIM_LIBRARIES[reloc.resolvewith]
            if self._check_user_blacklists(export.name):
                if not func.is_weak:
                    l.info("Using stub SimProcedure for unresolved %s from %s", func.name, sim_lib.name)
                    self.hook_symbol(export.rebased_addr, sim_lib.get_stub(export.name, sim_proc_arch))
            else:
                l.info("Using builtin SimProcedure for unresolved %s from %s", export.name, sim_lib.name)
                self.hook_symbol(export.rebased_addr, sim_lib.get(export.name, sim_proc_arch))
 
        # Step 2.4: If 2.3 didn't work (the symbol didn't request a provider we know of), try
        # looking through each of the SimLibraries we're using to resolve unresolved
        # functions. If any of them know anything specifically about this function,
        # resolve it with that. As a final fallback, just ask any old SimLibrary
        # to resolve it.
        elif missing_libs:
            for sim_lib in missing_libs:
                if sim_lib.has_metadata(export.name):
                    if self._check_user_blacklists(export.name):
                        if not func.is_weak:
                            l.info("Using stub SimProcedure for unresolved %s from %s", export.name, sim_lib.name)
                            self.hook_symbol(export.rebased_addr, sim_lib.get_stub(export.name, sim_proc_arch))
                    else:
                        l.info("Using builtin SimProcedure for unresolved %s from %s", export.name, sim_lib.name)
                        self.hook_symbol(export.rebased_addr, sim_lib.get(export.name, sim_proc_arch))
                    break
            else:
                if not func.is_weak:
                    l.info("Using stub SimProcedure for unresolved %s", export.name)
                    the_lib = missing_libs[0]
                    if export.name and export.name.startswith("_Z"):
                        # GNU C++ name. Use a C++ library to create the stub
                        if 'libstdc++.so' in SIM_LIBRARIES:
                            the_lib = SIM_LIBRARIES['libstdc++.so']
                        else:
                            l.critical("Does not find any C++ library in SIM_LIBRARIES. We may not correctly "
                                       "create the stub or resolve the function prototype for name %s.", export.name)
 
                    self.hook_symbol(export.rebased_addr, the_lib.get(export.name, sim_proc_arch))
 
        # Step 2.5: If 2.4 didn't work (we have NO SimLibraries to work with), just
        # use the vanilla ReturnUnconstrained, assuming that this isn't a weak func
        elif not func.is_weak:
            l.info("Using stub SimProcedure for unresolved %s", export.name)
            self.hook_symbol(export.rebased_addr, SIM_PROCEDURES['stubs']['ReturnUnconstrained'](display_name=export.name, is_stub=True))

```

#### [](#step7：执行os特定的配置)Step7：执行 OS 特定的配置

```
# Step 7: Run OS-specific configuration 执行 OS 特定的配置，函数原型在 ./simos/simos.py
self.simos.configure_project()

```

2 Hook 相关函数
-----------

### 2.1 hook（通过地址 hook 函数）

```
def hook(self, addr, hook=None, length=0, kwargs=None, replace=False):
    """
    Hook a section of code with a custom function. This is used internally to provide symbolic
    summaries of library functions, and can be used to instrument execution or to modify
    control flow.
    使用自定义函数挂钩一段代码。 这在内部用于提供库函数的符号摘要，并可用于检测执行或修改控制流。
 
    When hook is not specified, it returns a function decorator that allows easy hooking.
    如果没有指定hook参数，则假设函数作为装饰器使用
    Usage::
 
        # Assuming proj is an instance of angr.Project, we will add a custom hook at the entry
        # point of the project.
        @proj.hook(proj.entry)
        def my_hook(state):
            print("Welcome to execution!")
 
    :param addr:        The address to hook. 要hook的地址
    :param hook:        A :class:`angr.project.Hook` describing a procedure to run at the   
                        given address. You may also pass in a SimProcedure class or a function
                        directly and it will be wrapped in a Hook object for you.
                        Hook类描述了在给定地址上运行SimProcedure，也可以直接传递SimProcedure类或者函数，将包装在Hook对象中
 
    :param length:      If you provide a function for the hook, this is the number of bytes 
                        that will be skipped by executing the hook by default.
                        如果为hook提供替换函数，则默认情况下执行hook代码时将跳过的字节数。比如hook了0x08048485处的指令（xor eax, eax），该指令长度为2，那么length设置为2，执行完hook函数之后，将不会再执行该指令
 
    :param kwargs:      If you provide a SimProcedure for the hook, these are the keyword 
                        arguments that will be passed to the procedure's `run` method
                        eventually.
                        如果为hook提供SimProcedure，这些关键字参数最终将被传递给procedure的run方法
 
    :param replace:     Control the behavior on finding that the address is already hooked. If 
                        true, silently replace the hook. If false (default), warn and do not
                        replace the hook. If none, warn and replace the hook.
                        如果发现地址已经被hook，控制函数的行为。如果为真，则不提示信息直接替换hook。如果为false(默认)，则发出警告且不替换hook。如果为None，警告并更换hook。
 
    """
    if hook is None:
        # if we haven't been passed a thing to hook with, assume we're being used as a decorator
        # 如果没有传递一个需要被hook的东西，则假设该函数将被用作装饰器
        return self._hook_decorator(addr, length=length, kwargs=kwargs)
 
    if kwargs is None: kwargs = {}
 
    l.debug('hooking %s with %s', self._addr_to_str(addr), str(hook))
 
    if self.is_hooked(addr): # 如果已经被hook
        if replace is True: # 且如果replace为True，pass
            pass
        elif replace is False: # 如果replace为False，发出警告，不重新hook
            l.warning("Address is already hooked, during hook(%s, %s). Not re-hooking.", self._addr_to_str(addr), hook)
            return
        else:  # 如果replace为None，发出警告，但重新hook
            l.warning("Address is already hooked, during hook(%s, %s). Re-hooking.", self._addr_to_str(addr), hook)
 
    if isinstance(hook, type): # 通过判断hook变量是否是type类型，来判断hook是否实例化为Simprocedure
        raise TypeError("Please instanciate your SimProcedure before hooking with it")
 
    if callable(hook): # 检查对象hook是否是可调用的，如果返回true，hook对象可能调用失败；但如果返回false，hook对象绝对不会被调用成功
        hook = SIM_PROCEDURES['stubs']['UserHook'](user_func=hook, length=length, **kwargs)
 
    self._sim_procedures[addr] = hook # 被 hook 的地址及实例 hook 被放入字典 self._sim_procedures

```

### 2.2 is_hooked（判断某个地址是否被 hook）

```
def is_hooked(self, addr):
    """
    Returns True if `addr` is hooked. 通过判断地址是否在_sim_procedures这个字典来判断是否被hook
 
    :param addr: An address.
    :returns:    True if addr is hooked, False otherwise.
    """
    return addr in self._sim_procedures

```

```
>>> project._sim_procedures
{135783856: , 135599152: , 136354288: , 135691424: , 135464112: , 135368240: , 135675152: , 137438448: , 138412032: , 138412040: , 138412048: , 138412056: , 138412064: , 138412072: , 137438512: , 137436736: , 138412088: , 138412096: , 138412104: } 
```

### 2.3 hooked_by （返回地址 addr 对应的 SimProcedure 对象）

```
def hooked_by(self, addr): # 返回地址addr对应的SimProcedure对象
    """
    Returns the current hook for `addr`.
 
    :param addr: An address. 地址
 
    :returns:    None if the address is not hooked.
    """
    if not self.is_hooked(addr):
        l.warning("Address %s is not hooked", self._addr_to_str(addr))
        return None
 
    return self._sim_procedures[addr]

```

```
>>> project._sim_procedures[135783856]
 
```

### 2.4 unhook（解除对某个地址的 hook）

```
def unhook(self, addr):
        """
        Remove a hook. 通过删除字典_sim_procedures中addr键值对来解除hook
 
        :param addr:    The address of the hook. 被hook函数的地址
        """
        if not self.is_hooked(addr):
            l.warning("Address %s not hooked", self._addr_to_str(addr))
            return
 
        del self._sim_procedures[addr]

```

### 2.5 angrctf-09_angr_hooks

#### IDA 查看

![](https://l-tuchuang.oss-cn-beijing.aliyuncs.com/img/20211027122538.png)  
利用 hook 改写 check_equals_xxx 函数为自己的函数。  
![](https://l-tuchuang.oss-cn-beijing.aliyuncs.com/img/20211027122546.png)

#### 脚本

```
import angr
import sys
import claripy
def Go():
    path_to_binary = "./09_angr_hooks"
    project = angr.Project(path_to_binary, auto_load_libs=False)
    initial_state = project.factory.entry_state() # Angr可以处理对scanf的初始调用
 
    check_equals_called_address = 0x80486B8 # 找到需要hook的函数地址
    instruction_to_skip_length = 5 # 指定完成hook后应跳过多少字节，具体多少字节由hook处地址的指令长度确定
 
    # 当hook函数没有hook参数时，project.hook(addr) 作为函数装饰器，编写自己的 hook 函数。
    # 第一个参数是需要Hook的调用函数的地址，第二个参数length指定执行引擎在完成hook后应跳过多少字节。
    @project.hook(check_equals_called_address, length=instruction_to_skip_length)
    def skip_check_equals_(state):
        user_input_buffer_address = 0x804A054
        user_input_buffer_length = 16
 
        user_input_string = state.memory.load( # 使用 state.memory 的 .load(addr, size)接口读出buffer处的内存数据，与答案进行比较
            user_input_buffer_address,
            user_input_buffer_length
        )
 
        check_against_string = 'XKSPZSJKJYQCQXZV'
 
        # 模拟一个函数就是把它视作一个黑盒，能成功模拟输入相对应的输出即可，所以需要处理check函数的返回值
        # 函数check_equals_xxx 是利用EAX寄存器作为返回值，然后成功则返回1，不成功则返回0，还需要注意在构建符号位向量的时候EAX寄存器是32位寄存器
        register_size_bit = 32
        state.regs.eax = claripy.If(
            user_input_string == check_against_string,
            claripy.BVV(1, register_size_bit),
            claripy.BVV(0, register_size_bit)
        )
 
    # 或者把上面的装饰器注释掉，调用project对象的hook函数，并将skip_check_equals_函数作为参数hook进行传递
    # project.hook(addr=check_equals_called_address,hook=skip_check_equals_,length=instruction_to_skip_length)
 
    simulation = project.factory.simgr(initial_state)
 
    def is_successful(state):
        stdout_output = state.posix.dumps(1)
        if b'Good Job.\n' in stdout_output:
            return True
        else:
            return False
 
    def should_abort(state):
        stdout_output = state.posix.dumps(1)
        if b'Try again.\n' in  stdout_output:
            return True
        else:
            return False
 
    simulation.explore(find=is_successful, avoid=should_abort)
 
    if simulation.found:
        for i in simulation.found:
            solution_state = i
            solution = solution_state.posix.dumps(0)
            print("[+] Success! Solution is: {0}".format(solution.decode('utf-8')))
            #print(solution0)
    else:
        raise Exception('Could not find the solution')
 
if __name__ == "__main__":
    Go()

```

> 结果为：ZJOIPFTRNZOXIMLEWGLFMCQOKWLUFJIB

### 2.6 hook_symbol（通过函数名来 hook）

```
def hook_symbol(self, symbol_name, simproc, kwargs=None, replace=None): # 通过函数名（符号）来hook
    """
    找到所给符号对应的地址，然后hook地址。如果符号在加载的库中不可用，则该地址可能由 CLE externs object提供。emmm，不可用，怎么个不可用法？还有CLE externs object？
    Resolve a dependency in a binary. Looks up the address of the given symbol, and then hooks that
    address. If the symbol was not available in the loaded libraries, this address may be provided
    by the CLE externs object.
 
    如果传入的不是符号，而是地址，一些秘密功能将会启用？？？
    Additionally, if instead of a symbol name you provide an address, some secret functionality will
    kick in and you will probably just hook that address, UNLESS you're on powerpc64 ABIv1 or some
    yet-unknown scary ABI that has its function pointers point to something other than the actual
    functions, in which case it'll do the right thing.
 
    :param symbol_name: The name of the dependency to resolve. 需要解析的函数名称
    :param simproc:     The SimProcedure instance (or function) with which to hook the symbol
                        用于hook的SimProcedure实例（或函数）
 
    :param kwargs:      If you provide a SimProcedure for the hook, these are the keyword
                        arguments that will be passed to the procedure's `run` method
                        eventually.
                        如果为hook提供的是SimProcedure对象，那么这些参数将传递给procedure的run方法
 
    :param replace:     Control the behavior on finding that the address is already hooked. If
                        true, silently replace the hook. If false, warn and do not replace the
                        hook. If none (default), warn and replace the hook.
                        控制函数的行为。如果为真，则不提示信息且直接替换hook。如果为false，则发出警告且不替换hook。如果为None（默认），警告并更换hook。
 
    :returns:           The address of the new symbol. 返回值是解析完毕的函数的地址
    :rtype:             int 返回值的类型是int
    """
    if type(symbol_name) is not int: # 如果传入的symbol_name不是地址
        sym = self.loader.find_symbol(symbol_name) # emmm
        if sym is None:
            # it could be a previously unresolved weak symbol..?
            new_sym = None
            for reloc in self.loader.find_relevant_relocations(symbol_name):
                if not reloc.symbol.is_weak:
                    raise Exception("Symbol is strong but we couldn't find its resolution? Report to @rhelmot.")
                if new_sym is None:
                    new_sym = self.loader.extern_object.make_extern(symbol_name)
                reloc.resolve(new_sym)
                reloc.relocate([])
 
            if new_sym is None:
                l.error("Could not find symbol %s", symbol_name)
                return None
            sym = new_sym
 
        basic_addr = sym.rebased_addr
    else: # 如果传入的symbol_name是地址
        basic_addr = symbol_name
        symbol_name = None
 
    hook_addr, _ = self.simos.prepare_function_symbol(symbol_name, basic_addr=basic_addr) # emmm
 
    self.hook(hook_addr, simproc, kwargs=kwargs, replace=replace) # 调用hook方法
    return hook_addr # 返回被hook函数解析出来的地址

```

```
>>> project.loader.find_symbol('strcmp')
 
```

### 2.7 is_symbol_hooked（判断某个符号是否被 hook）

```
def is_symbol_hooked(self, symbol_name):
    """
    Check if a symbol is already hooked. 判断一个符号是否已经被hook
 
    :param str symbol_name: Name of the symbol.
    :return: True if the symbol can be resolved and is hooked, False otherwise.
    :rtype: bool
    """
    sym = self.loader.find_symbol(symbol_name)
    if sym is None: # 如果找不到符号，警告，返回false
        l.warning("Could not find symbol %s", symbol_name)
        return False
    hook_addr, _ = self.simos.prepare_function_symbol(symbol_name, basic_addr=sym.rebased_addr) # 解析得到符号对应的地址
    return self.is_hooked(hook_addr) # 调用is_hook()判断该符号是否被hook

```

### 2.8 unhook_symbol（解除对某个符号的 hook）

```
def unhook_symbol(self, symbol_name):
     """
     Remove the hook on a symbol. 解除对某个符号的hook
     This function will fail if the symbol is provided by the extern object, as that would result in a state where
     analysis would be unable to cope with a call to this symbol.
     """
     sym = self.loader.find_symbol(symbol_name)
     if sym is None: # 如果找不到符号，警告，返回false
         l.warning("Could not find symbol %s", symbol_name)
         return False
     if sym.owner is self.loader._extern_object: # 如果符号属于extern object
         l.warning("Refusing to unhook external symbol %s, replace it with another hook if you want to change it",
                   symbol_name)
         return False
 
     hook_addr, _ = self.simos.prepare_function_symbol(symbol_name, basic_addr=sym.rebased_addr) # 解析得到符号对应的地址
     self.unhook(hook_addr) # 调用unhook解除对符号的hook
     return True

```

### 2.9 rehook_symbol（将对某个符号的 hook 移动到一个特定的地址）

```
def rehook_symbol(self, new_address, symbol_name, stubs_on_sync):
    """
    Move the hook for a symbol to a specific address 将对某个符号的hook移动到一个特定的地址，待具体分析
    :param new_address: the new address that will trigger the SimProc execution
    :param symbol_name: the name of the symbol (f.i. strcmp )
    :return: None
    """
    new_sim_procedures = {}
    for key_address, simproc_obj in self._sim_procedures.items():
 
        # if we don't want stubs during the sync let's skip those, we will execute the real function.
        if not stubs_on_sync and simproc_obj.is_stub:
            continue
 
        if simproc_obj.display_name == symbol_name:
            new_sim_procedures[new_address] = simproc_obj
        else:
            new_sim_procedures[key_address] = simproc_obj
 
    self._sim_procedures = new_sim_procedures

```

### 2.10 angrctf-10_angr_simprocedures

#### IDA 查看

![](https://l-tuchuang.oss-cn-beijing.aliyuncs.com/img/20211027122600.png)  
不同于第 9 题，本题中 check_equals_xxx 函数的第一个参数是局部变量，而不是全局变量，无法确定其地址。而且虽然上图中的伪代码只调用了一个 check_equals_xxx 函数，但其实汇编代码中有很多个分支，如下图所示，单看汇编代码是很难确定该函数的地址的，所以这题需要通过函数名（符号）hook 所有对该函数的调用。  
![](https://l-tuchuang.oss-cn-beijing.aliyuncs.com/img/20211027122619.png)

#### 脚本

```
import angr
import claripy
import sys
 
def Go():
    path_to_binary = "./10_angr_simprocedures"
    project = angr.Project(path_to_binary, auto_load_libs=False)
    initial_state = project.factory.entry_state()
 
    class ReplacementCheckEquals(angr.SimProcedure): # 定义一个继承angr.SimProcedure的类
        def run(self, to_check, length): # SimProcedure用Python编写的自己的函数代替了原来函数。 除了用Python编写之外，该函数的行为与用C编写的任何函数基本相同。
                                        # self之后的任何参数都将被视为要替换的函数的参数， 参数将是符号位向量。
                                        # 另外，Python可以以常用的Python方式返回，Angr将以与原来函数相同的方式对待它
 
            user_input_buffer_address = to_check # #即第一个参数
            user_input_buffer_length = length # #即第二个参数
            user_input_string = self.state.memory.load( # 使用self.state在SimProcedure中查找系统状态，从该状态的内存中提取出数据
                user_input_buffer_address,
                user_input_buffer_length
            )
            check_against_string = 'WQNDNKKWAWOLXBAC'  # 如果符合条件则返回输入的符号位向量
            return claripy.If(
                user_input_string == check_against_string,
                claripy.BVV(1, 32),
                claripy.BVV(0, 32)
            )
 
    check_equals_symbol = 'check_equals_WQNDNKKWAWOLXBAC'
    project.hook_symbol(check_equals_symbol, ReplacementCheckEquals()) # Hook上check_equals函数， angr会自动查找与该函数符号关联的地址
 
    simulation = project.factory.simgr(initial_state)
 
    def is_successful(state):
        stdout_output = state.posix.dumps(1)
        if b'Good Job.\n' in stdout_output:
            return True
        else:
            return False
 
    def should_abort(state):
        stdout_output = state.posix.dumps(1)
        if b'Try again.\n' in  stdout_output:
            return True
        else:
            return False
 
    simulation.explore(find=is_successful, avoid=should_abort)
 
    if simulation.found:
        for i in simulation.found:
            solution_state = i
            solution = solution_state.posix.dumps(0)
            print("[+] Success! Solution is: {0}".format(solution.decode('utf-8')))
            #print(solution0)
    else:
        raise Exception('Could not find the solution')
 
if __name__ == "__main__":
    Go()

```

> 结果为：URRKXXAPWVQQFMOT

### 2.11_hook_decorator（返回一个允许简单 hook 的函数装饰器）

上面和 hook 相关的函数都是 public 的，而这个函数是 private 的。

```
def _hook_decorator(self, addr, length=0, kwargs=None):
    """
    Return a function decorator that allows easy hooking. Please refer to hook() for its usage.
    返回一个允许简单hook的函数装饰器
 
    :return: The function decorator.
    """
 
    def hook_decorator(func):
        self.hook(addr, func, length=length, kwargs=kwargs)
        return func
 
    return hook_decorator

```

3 Convenience API
-----------------

```
#
# A convenience API (in the style of triton and manticore) for symbolic execution.
# 用于符号执行的方便的API（triton 和 manticore风格）
# triton 是一个包含动态符号执行工具的动态二进制分析平台，https://triton.quarkslab.com/
# manticore是 也是一个针对二进制文件（x86, x86_64 and ARMV7）的符号执行工具，https://github.com/trailofbits/manticore
#

```

### 3.1 execute（开始符号执行）

```
def execute(self, *args, **kwargs):
    """
    This function is a symbolic execution helper in the simple style
    supported by triton and manticore. It designed to be run after
    setting up hooks (see Project.hook), in which the symbolic state
    can be checked.
    该函数是一个由triton和manticore支持的简单风格的符号执行助手。被设计为在设置hook之后运行，使得可以在hook之后检查符号状态
 
    This function can be run in three different ways: 该函数可以用三种不同方式运行
 
        - When run with no parameters, this function begins symbolic execution
          from the entrypoint.
          无参数，该函数从入口点开始符号执行
        - It can also be run with a "state" parameter specifying a SimState to
          begin symbolic execution from.
          带state参数，从指定的state开始符号执行
        - Finally, it can accept any arbitrary keyword arguments, which are all
          passed to project.factory.full_init_state.
          任意参数，这些参数将被传递到project.factory.full_init_state
 
    If symbolic execution finishes, this function returns the resulting
    simulation manager.  如果符号执行结束，将把执行结果返回给simulation manager
    """
 
    if args: # 如果有参数，第一个参数给state
        state = args[0]
    else: #
        state = self.factory.full_init_state(**kwargs)
 
    pg = self.factory.simulation_manager(state) # 设置simulation manager
    self._executing = True # _executing设置为True，该变量标志开始或终止
    return pg.run(until=lambda lpg: not self._executing) # run函数待分析

```

### 3.2 terminate_execution（结束符号执行）

```
def terminate_execution(self):
    """
    Terminates a symbolic execution that was started with Project.execute().
    终止一个由execute开始的符号执行
    """
    self._executing = False

```

4 load_shellcode
----------------

```
def load_shellcode(shellcode, arch, start_offset=0, load_address=0, thumb=False, **kwargs):
    """
    Load a new project based on a snippet of assembly or bytecode.
    基于一串汇编代码或字节码加载一个新的Project
 
    :param shellcode:       The data to load, as either a bytestring of instructions or a string of assembly text 要加载的数据，可以是指令的字节串或者是汇编文本的字符串
    :param arch:            The name of the arch to use, or an archinfo class 要用的arch的名字，或者是一个archinfo类
    :param start_offset:    The offset into the data to start analysis (default 0) 开始分析的数据的偏移，默认是0
    :param load_address:    The address to place the data in memory (default 0) 在内存中放置数据的地址，默认是0
    :param thumb:           Whether this is ARM Thumb shellcode 是否是ARM架构Thumb模式的shellcode
    """
    if not isinstance(arch, archinfo.Arch): # 如果传入的arch不是archinfo.Arch类，就把它转换成它
        arch = archinfo.arch_from_id(arch)
    if type(shellcode) is str: # 如果shellcode是字符串，就对其进行汇编，得到对应的机器码
        shellcode = arch.asm(shellcode, load_address, thumb=thumb)
    if thumb: # 如果是thumb模式
        start_offset |= 1 # 数据偏移的最低位变成1
 
    return Project( # 返回一个新的 Project
            BytesIO(shellcode), # 流
            main_opts={
                'backend': 'blob', #后端使用的加载器（“elf”，“pe”，“mach-o”，“ida”，“blob”）
                'arch': arch, # 架构
                'entry_point': start_offset, # 入口
                'base_addr': load_address, # 基址
            },
        **kwargs
        )

```

5 总结
----

还有很多细节没有搞懂，比如 cle.Loader，很多地方用到了 loader 里面的东西，但目前只是通过在解释器里运行代码看一看这些结构长什么样。还有 SimProcedures 等，另外还有 Pickling 序列化那些函数也没看出个所以然，后面再分析吧。

*   Project 涉及
    *   cle：二进制加载
    *   engine：模拟执行引擎
    *   analyses：CFG 分析模块
    *   factory：工厂
    *   simprocedure：hook
    *   simos：io 交互

> 明天华为签约，希望能开出个比我拒掉 offer 更高的价钱吧。难受，秋招不理想，归根到底还是自己菜。立个 flag，一周看完 angr 的重要部分源码。

参考文献
----

*   [https://www.anquanke.com/post/id/231460](https://www.anquanke.com/post/id/231460)
*   [http://angr.io/api-doc/angr.html](http://angr.io/api-doc/angr.html)
*   [https://github.com/another1024/angr-analysis](https://github.com/another1024/angr-analysis)
*   [https://www.anquanke.com/post/id/214288#h2-7](https://www.anquanke.com/post/id/214288#h2-7)
*   [https://www.wangan.com/p/7fygfgbcefb38cfc](

[[培训] 优秀毕业生寄语：恭喜 id 咸鱼炒白菜拿到远超 3W 月薪的 offer，《安卓高级研修班》火热招生！！！](https://zhuanlan.kanxue.com/article-16096.htm)

最后于 8 小时前 被直木编辑 ，原因：

[#其他内容](forum-4-1-10.htm)
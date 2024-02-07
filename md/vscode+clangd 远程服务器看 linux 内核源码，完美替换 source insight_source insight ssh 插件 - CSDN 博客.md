> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_38384263/article/details/129186368?spm=1001.2014.3001.5502)

#### vscode+clangd 远程服务器看 linux 内核源码，完美替换 source insight

*   [vscode 插件需求](#vscode_2)
*   *   [插件设置](#_12)
*   [工具需求](#_44)
*   *   [bear 编译](#bear_49)
    *   [clangd 编译](#clangd_63)
*   [内核索引](#_74)
*   [设置代码风格及静态检查项](#_81)
*   [配置 clangd 工程代码风格](#clangd_292)

vscode 插件需求
-----------

插件共有以下几个：

1.  C/C++
2.  clangd
3.  Remote - SSH

### 插件设置

**C/C++ 1** 插件在这里只需要提供基础的 C/C++ 服务即可，不需要语法解析，自动跳转和补全功能。所以需要关闭 C/C++，在 vscode 设置里搜索`C_Cpp: Intelli Sense Engine`，选择`disabled`。

**Remote - SSH** 插件用于[远程连接服务器](https://so.csdn.net/so/search?q=%E8%BF%9C%E7%A8%8B%E8%BF%9E%E6%8E%A5%E6%9C%8D%E5%8A%A1%E5%99%A8&spm=1001.2101.3001.7020)或者是虚拟机。配置一下 IP 及端口即可。`ctrl+shift+p`：搜索`Remote-SSH:Open SSH Configuration File`。输入自己的服务器 IP 及端口，例：

```
Host xxxx
    HostName 192.168.1.111
    User xxx
    Port xx

```

**clangd** 插件，[远程连接](https://so.csdn.net/so/search?q=%E8%BF%9C%E7%A8%8B%E8%BF%9E%E6%8E%A5&spm=1001.2101.3001.7020)到服务器后在内核工程目录创建`.vscode`文件夹，里面新建`settings.json`，内容如下：

```
{
    "clangd.arguments": [
        "--all-scopes-completion",
        "--clang-tidy",
        "--completion-parse=always",
        "--header-insertion=never",
        "--completion-style=detailed",
        "--query-driver=***/***/***gcc",//这里是自己的交叉编译路径
        "--function-arg-placeholders=false",
        "--compile-commands-dir=${workspaceFolder}/",
        "-log=info",
        "-j=32"
    ],
    "clangd.checkUpdates": true,
}

```

工具需求
----

1.  bear
2.  clangd

### bear 编译

`bear`用于创建`compile_commands.json`文件，该文件详细描述各个代码文件的编译命令，便于`clangd`建立代码工程。

下载`bear`源码：[bear 下载链接](https://github.com/rizsotto/Bear/releases)。推荐下载 2.4.3 的版本。

```
tar -xvf Bear-2.4.3.tar.gz
cd Bear-2.4.3
camke ./ -DCMAKE_INSTALL_PREFIX=${install_dir}
make all -j32
make install

```

将`${install_dir}/bin`添加到`$PATH`里，将`${install_dir/lib64/bear}`添加到`$LD_LIBRARY_PATH`。

### clangd 编译

`clangd`可以根据`compile_commands.json`文件建立索引数据库。从而达到索引各种符号的目的。  
`clangd`是在`llvm`工程里的。所以下载 llvm 的工程源码 [llvm 源码](https://github.com/llvm/llvm-project/releases)，下载`llvm-project-xx.x.x.src.tar.xz`文件，这个才是工程源码。

```
cmake -G "Unix Makefiles" -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra" -DCMAKE_INSTALL_PREFIX=~/${install_dir} -DCMAKE_BUILD_TYPE=Release ../llvm -DCMAKE_CXX_COMPILER=${g++_dir}/g++ -DCMAKE_C_COMPILER=${gcc_dir}/gcc
make -j32
make install

```

将`${install_dir}/bin`添加到`$PATH`里，将`${install_dir/lib}`添加到`$LD_LIBRARY_PATH`。

内核索引
----

```
bear make ARCH=arm64 CROSS_COMPILE=xxx

```

在编译命令前加上`bear`就可以生成`compile_commands.json`文件了。之后就可以愉快的索引了。

设置代码风格及静态检查项
------------

代码工程目录下可以新建. clangd 文件，该文件描述了要启用的静态检查项，也可以去除 clang 无法识别的一些编译指令，这里给个例子，可以参考我这个配置，也可以参考这个网站 [clang analyzer](https://clang.llvm.org/docs/analyzer/checkers.html)。

```
Diagnostics:
  ClangTidy:
    Add:#添加的代码检查项
    [
      clang-analyzer-*,
      modernize-*,
      bugprone-*,
      performance-*,
      portability-*,
      readability-*,
      cppcoreguidelines-no-malloc,
      cppcoreguidelines-macro-usage,
      cppcoreguidelines-pro-bounds-pointer-arithmetic,
      bugprone-infinite-loop,
      bugprone-argument-comment,
      bugprone-assert-side-effect,
      bugprone-bad-signal-to-kill-thread,
      bugprone-branch-clone,
      bugprone-copy-constructor-init,
      bugprone-dangling-handle,
      bugprone-dynamic-static-initializers,
      bugprone-fold-init-type,
      bugprone-forward-declaration-namespace,
      bugprone-forwarding-reference-overload,
      bugprone-inaccurate-erase,
      bugprone-incorrect-roundings,
      bugprone-integer-division,
      bugprone-lambda-function-name,
      bugprone-macro-parentheses,
      bugprone-macro-repeated-side-effects,
      bugprone-misplaced-operator-in-strlen-in-alloc,
      bugprone-misplaced-pointer-arithmetic-in-alloc,
      bugprone-misplaced-widening-cast,
      bugprone-move-forwarding-reference,
      bugprone-multiple-statement-macro,
      bugprone-no-escape,
      bugprone-not-null-terminated-result,
      bugprone-parent-virtual-call,
      bugprone-posix-return,
      bugprone-reserved-identifier,
      bugprone-sizeof-container,
      bugprone-sizeof-expression,
      bugprone-spuriously-wake-up-functions,
      bugprone-string-constructor,
      bugprone-string-integer-assignment,
      bugprone-string-literal-with-embedded-nul,
      bugprone-suspicious-enum-usage,
      bugprone-suspicious-include,
      bugprone-suspicious-memory-comparison,
      bugprone-suspicious-memset-usage,
      bugprone-suspicious-missing-comma,
      bugprone-suspicious-semicolon,
      bugprone-suspicious-string-compare,
      bugprone-swapped-arguments,
      bugprone-terminating-continue,
      bugprone-throw-keyword-missing,
      bugprone-too-small-loop-variable,
      bugprone-undefined-memory-manipulation,
      bugprone-undelegated-constructor,
      bugprone-unhandled-self-assignment,
      bugprone-unused-raii,
      bugprone-unused-return-value,
      bugprone-use-after-move,
      bugprone-virtual-near-miss,
            cert-dcl21-cpp,
      cert-dcl58-cpp,
      cert-err34-c,
      cert-err52-cpp,
      cert-err58-cpp,
      cert-err60-cpp,
      cert-flp30-c,
      cert-msc50-cpp,
      cert-msc51-cpp,
      cert-str34-c,
      cppcoreguidelines-interfaces-global-init,
      cppcoreguidelines-narrowing-conversions,
      cppcoreguidelines-pro-type-member-init,
      cppcoreguidelines-pro-type-static-cast-downcast,
      cppcoreguidelines-slicing,
      google-default-arguments,
      google-explicit-constructor,
      google-runtime-operator,
      hicpp-exception-baseclass,
      hicpp-multiway-paths-covered,
      misc-misplaced-const,
      misc-new-delete-overloads,
      misc-no-recursion,
      misc-non-copyable-objects,
      misc-throw-by-value-catch-by-reference,
      misc-unconventional-assign-operator,
      misc-uniqueptr-reset-release,
      modernize-avoid-bind,
      modernize-concat-nested-namespaces,
      modernize-deprecated-headers,
      modernize-deprecated-ios-base-aliases,
      modernize-loop-convert,
      modernize-make-shared,
      modernize-make-unique,
      modernize-pass-by-value,
      modernize-raw-string-literal,
      modernize-redundant-void-arg,
      modernize-replace-auto-ptr,
      modernize-replace-disallow-copy-and-assign-macro,
      modernize-replace-random-shuffle,
      modernize-return-braced-init-list,
      modernize-shrink-to-fit,
      modernize-unary-static-assert,
      modernize-use-auto,
      modernize-use-bool-literals,
      modernize-use-emplace,
      modernize-use-equals-default,
      modernize-use-equals-delete,
      modernize-use-nodiscard,
      modernize-use-noexcept,
      modernize-use-nullptr,
      modernize-use-override,
      modernize-use-transparent-functors,
      modernize-use-uncaught-exceptions,
      openmp-use-default-none,
      performance-faster-string-find,
      performance-for-range-copy,
      performance-implicit-conversion-in-loop,
      performance-inefficient-algorithm,
      performance-inefficient-string-concatenation,
      performance-inefficient-vector-operation,
      performance-move-const-arg,
      performance-move-constructor-init,
      performance-no-automatic-move,
      performance-noexcept-move-constructor,
      performance-trivially-destructible,
      performance-type-promotion-in-math-fn,
      performance-unnecessary-copy-initialization,
      performance-unnecessary-value-param,
      portability-simd-intrinsics,
      readability-avoid-const-params-in-decls,
      readability-const-return-type,
      readability-container-size-empty,
      readability-convert-member-functions-to-static,
      readability-delete-null-pointer,
      readability-inconsistent-declaration-parameter-name,
      readability-make-member-function-const,
      readability-misleading-indentation,
      readability-misplaced-array-index,
      readability-redundant-control-flow,
      readability-redundant-declaration,
      readability-redundant-function-ptr-dereference,
      readability-redundant-smartptr-get,
      readability-redundant-string-cstr,
      readability-redundant-string-init,
      readability-simplify-subscript-expr,
      readability-static-accessed-through-instance,
      readability-static-definition-in-anonymous-namespace,
      readability-string-compare,
      readability-uniqueptr-delete-release,
      readability-use-anyofallof,
      
    ]
    Remove:#忽略的代码检查项
    [
      readability-function-cognitive-complexity,
      readability-identifier-length,
      readability-magic-numbers,
      readability-non-const-parameter,
      bugprone-easily-swappable-parameters,
      readability-misleading-indentation,
      readability-isolate-declaration,
      readability-braces-around-statements,
    ]
CompileFlags:                             
  Add: #添加的编译标志
    [
      -W,
      -Wall,
      -Wshadow,
      -Wtype-limits,
      -Wasm,
      -Wchkp,
      -Warray-parameter,
      -Wthread-safety,
      -Wswitch-default,
      -Wuninitialized,
      -Wunused-label,
      -Wunused-lambda-capture,
      -Wno-error=unused-command-line-argument-hard-error-in-future,
      -Wno-sign-compare,
      -Wno-void-pointer-to-int-cast,
      -Wno-int-to-pointer-cast,
      -Wno-asm_invalid_global_var_reg,
      -Wno-format,
      --target=aarch64-linux-gnu,
    ]
  Remove:#忽略的编译标志
    [
      -mabi=lp64,
      -fno-var-tracking-assignments,
      -fconserve-stack,
    ]
    
      
InlayHints:#嵌入提示
  Enabled: Yes
  ParameterNames: Yes
  DeducedTypes: Yes

Hover:
  ShowAKA: Yes

```

配置 clangd 工程代码风格
----------------

`clangd`支持一键格式化代码，索引，空格，换行，变量命名规则等都可以设置，在工程下新建. clang-format 文件，具体怎么设置可以参考我的工程配置和这个网站 [clang format](https://clang.llvm.org/docs/ClangFormatStyleOptions.html)。

```
# Generated from CLion C/C++ Code Style settings

BasedOnStyle: Microsoft

#头文件排序
SortIncludes: Never

#转义换行符对齐
AlignEscapedNewlines: Left

#对齐注释
AlignTrailingComments: true

# 连续赋值时，对齐所有等号
AlignConsecutiveAssignments: 
  Enabled: true
  #跨空行对齐
  AcrossEmptyLines: true
  #跨注释对齐
  AcrossComments: false
  #填充空格
  PadOperators: true
  AlignCompound: true
 
# 连续声明时，对齐所有声明的变量名
AlignConsecutiveDeclarations: 
  Enabled: true
  AcrossEmptyLines: true
  AcrossComments: false
  PadOperators: true
  AlignCompound: true

#对齐宏
AlignConsecutiveMacros:
  Enabled: true
  AcrossEmptyLines: true
  AcrossComments: false
  PadOperators: true

#访问修饰符的额外缩进或升级，例如 .public:
AccessModifierOffset: -4

#将参数水平对齐在左括号之后
AlignAfterOpenBracket: Align

#对结构数组使用初始化时，会将字段对齐到列中
AlignArrayOfStructures: Left

#对齐连续位字段的样式
AlignConsecutiveBitFields: false

#水平对齐二元表达式和三元表达式的操作数
AlignOperands: Align

#如果函数调用或带大括号的初始值设定项列表不适合某一行，则允许将所有参数放在下一行
AllowAllArgumentsOnNextLine: false

#如果函数声明不适合某一行，则允许将函数声明的所有参数放在下一行
AllowAllParametersOfDeclarationOnNextLine: false

#根据值，可以放在一行上
AllowShortBlocksOnASingleLine: Never

#短的case将被收缩到一行
AllowShortCaseLabelsOnASingleLine: false

#允许在一行上使用短枚举
AllowShortEnumsOnASingleLine: false

#短的函数可以放在一行 None:永远不放在一行
AllowShortFunctionsOnASingleLine: None

#根据值，if可以放在一行上
AllowShortIfStatementsOnASingleLine: Never

#根据值，可以放在一行上
AllowShortLambdasOnASingleLine: None

#如果 ，可以放在一行上
AllowShortLoopsOnASingleLine: false

#函数声明返回要使用的类型中断样式
AlwaysBreakAfterReturnType: None

#要使用的模板声明中断样式
AlwaysBreakTemplateDeclarations: Yes

#如果 ，则函数调用的参数要么全部在同一行上，要么每个参数都有一行
BinPackArguments: true


BreakBeforeBraces: Custom
BraceWrapping:
  AfterCaseLabel: true
  AfterClass: true
  AfterControlStatement: Always
  AfterEnum: true
  AfterStruct: true
  AfterFunction: true
  AfterNamespace: true
  AfterUnion: true
  AfterExternBlock: true
  BeforeCatch: true
  BeforeElse: true
  IndentBraces: false
  SplitEmptyFunction: true
  SplitEmptyRecord: true
BreakBeforeBinaryOperators: None
BreakBeforeTernaryOperators: true
BreakConstructorInitializers: BeforeColon
BreakInheritanceList: BeforeColon
ColumnLimit: 80
CompactNamespaces: false
ContinuationIndentWidth: 8
IndentCaseLabels: true
IndentPPDirectives: None
IndentWidth: 4
KeepEmptyLinesAtTheStartOfBlocks: true
MaxEmptyLinesToKeep: 2
NamespaceIndentation: All
ObjCSpaceAfterProperty: false
ObjCSpaceBeforeProtocolList: true
PointerAlignment: Right
ReflowComments: false
SpaceAfterCStyleCast: false
SpaceAfterLogicalNot: false
SpaceAfterTemplateKeyword: false
SpaceBeforeAssignmentOperators: true
SpaceBeforeCpp11BracedList: false
SpaceBeforeCtorInitializerColon: true
SpaceBeforeInheritanceColon: true
SpaceBeforeParens: ControlStatements
SpaceBeforeRangeBasedForLoopColon: false
SpaceInEmptyParentheses: false
SpacesBeforeTrailingComments: 0
SpacesInAngles: false
SpacesInCStyleCastParentheses: false
SpacesInContainerLiterals: false
SpacesInParentheses: false
SpacesInSquareBrackets: false
# TabWidth: 4
UseTab: Never


```
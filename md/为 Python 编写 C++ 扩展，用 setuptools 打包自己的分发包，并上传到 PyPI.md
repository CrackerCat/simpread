> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1923720-1-1.html)

> [md]## 引言最近手痒想写 Python 和 C++ 代码，于是有了这篇~~ 精华~~ 水 blog。

![](https://avatar.52pojie.cn/data/avatar/001/90/61/77_avatar_middle.jpg)hans7 _ 本帖最后由 hans7 于 2024-5-14 17:08 编辑_  

引言
--

最近手痒想写 Python 和 C++ 代码，于是有了这篇~精华~水 blog。这次打算用 C++ 写些数据结构，打包后给 Python 调用。

[项目 GitHub 传送门](https://github.com/Hans774882968/data-structure-demos-hans)

**作者：[hans774882968](https://blog.csdn.net/hans774882968) 以及 [hans774882968](https://juejin.cn/user/1464964842528888) 以及 [hans774882968](https://www.52pojie.cn/home.php?mod=space&uid=1906177)**

本文 52pojie：[https://www.52pojie.cn/thread-1923720-1-1.html](https://www.52pojie.cn/thread-1923720-1-1.html)

本文 juejin：[https://juejin.cn/post/7368319486778703898](https://juejin.cn/post/7368319486778703898)

本文 CSDN：[https://blog.csdn.net/hans774882968/article/details/138812416](https://blog.csdn.net/hans774882968/article/details/138812416)

环境
--

*   Windows10 VSCode
*   Python 3.7.6 pytest 7.4.4 setuptools 68.0.0
*   `g++.exe (x86_64-win32-seh-rev1, Built by MinGW-Builds project) 13.1.0` from [here](https://whitgit.whitworth.edu/tutorials/installing_mingw_64)

setuptools 官方文档建议舍弃`setup.py`（setup 脚本）的写法，转而使用一个叫 [build](https://build.pypa.io/en/latest/installation.html) 的命令行工具。但该工具只支持到 Python3.8，所以本文依旧使用`setup.py`。

Usage of data-structure-demos-hans
----------------------------------

Install: `pip install data-structure-demos-hans`

binary indexed tree:

```
from binary_indexed_tree.bit_cpp_extension import BIT

def bit_cpp(a: List[int]):
    b1 = BIT(len(a))
    for i, v in enumerate(a):
        b1.add(i + 1, v)
    for i in range(1, len(a) + 1):
        print(b1.sum(i))
    b1.add(9, 113964)
    print(b1.sum(len(a)))

bit_cpp([i * 10 + 10 for i in range(10)])

```

bisect_decr:

```
from bisect_decr.bisect_decr_cpp import dec_lower_bound, dec_upper_bound

def test_dec_lower_bound_unique():
    a = [100, 90, 80, 70, 60, 50, 40, 30, 20, 10]
    test_arr = [101, 61, 9, 100, 60, 10]
    res = [dec_lower_bound(a, v) for v in test_arr]
    assert res == [0, 4, 10, 0, 4, 9]

def test_comparable_object_array():
    class Person():
        def __init__(self, age: int) -> None:
            self.age = age

        def __ge__(self, other):
            return self.age >= other.age

        def __gt__(self, other):
            return self.age > other.age

    persons = [Person(60), Person(25), Person(18), Person(18), Person(6)]
    test_arr = [Person(100), Person(60), Person(33), Person(25), Person(23), Person(18), Person(12), Person(6), Person(4)]
    res_l = [dec_lower_bound(persons, v) for v in test_arr]
    assert res_l == [0, 0, 1, 1, 2, 2, 4, 4, 5]
    res_u = [dec_upper_bound(persons, v) for v in test_arr]
    assert res_u == [0, 1, 1, 2, 2, 4, 4, 5, 5]

```

制作源码包
-----

顾名思义，源码包就是不在打包阶段进行编译工作，到`pip install`阶段再执行。根据[参考链接 1](https://blog.csdn.net/hxxjxw/article/details/124298543)，我们可以用`setuptools`制作源码包。首先我们新建一个项目，目录如下：

```
.
│  setup.py 调用 setuptools.setup
├─binary_indexed_tree
      bit.py 最简单的命令行程序
      __init__.py 留空就行。为了让 setuptools 扫描到这个包，必须要有

```

`setup.py`可以参考 [setup 脚本文档传送门](https://setuptools.pypa.io/en/latest/deprecated/distutils/setupscript.html)来写：

```
from setuptools import setup, find_packages

setup(
    name='data-structure-demos-hans',
    version='0.1',
    description='provide some data structures for competitive programming',
    author='<author name>',
    author_email='<your email>',
    url='https://github.com/Hans774882968/data-structure-demos-hans',
    packages=find_packages('.'),
    license='MIT',
)

```

`find_packages`用来查找当前项目有哪些包，[文档](https://setuptools.pypa.io/en/latest/userguide/package_discovery.html#custom-discovery)。`find_packages('.')`只指定了`where='.'`，表示从当前目录开始扫描，把所有含有`__init__.py`的目录都当成包。

制作源码包命令：`python setup.py sdist`。

```
running sdist
running egg_info
creating data_structure_demos_hans.egg-info
writing data_structure_demos_hans.egg-info\PKG-INFO
writing dependency_links to data_structure_demos_hans.egg-info\dependency_links.txt
writing top-level names to data_structure_demos_hans.egg-info\top_level.txt
writing manifest file 'data_structure_demos_hans.egg-info\SOURCES.txt'
reading manifest file 'data_structure_demos_hans.egg-info\SOURCES.txt'
writing manifest file 'data_structure_demos_hans.egg-info\SOURCES.txt'
running check
creating data-structure-demos-hans-0.1
creating data-structure-demos-hans-0.1\binary_indexed_tree
creating data-structure-demos-hans-0.1\data_structure_demos_hans.egg-info
copying files to data-structure-demos-hans-0.1...
copying README.md -> data-structure-demos-hans-0.1
copying setup.py -> data-structure-demos-hans-0.1
copying binary_indexed_tree\__init__.py -> data-structure-demos-hans-0.1\binary_indexed_tree
copying binary_indexed_tree\bit.py -> data-structure-demos-hans-0.1\binary_indexed_tree 
copying data_structure_demos_hans.egg-info\PKG-INFO -> data-structure-demos-hans-0.1\data_structure_demos_hans.egg-info
copying data_structure_demos_hans.egg-info\SOURCES.txt -> data-structure-demos-hans-0.1\data_structure_demos_hans.egg-info
copying data_structure_demos_hans.egg-info\dependency_links.txt -> data-structure-demos-hans-0.1\data_structure_demos_hans.egg-info
copying data_structure_demos_hans.egg-info\top_level.txt -> data-structure-demos-hans-0.1\data_structure_demos_hans.egg-info
Writing data-structure-demos-hans-0.1\setup.cfg
creating dist
Creating tar archive
removing 'data-structure-demos-hans-0.1' (and everything under it)

```

运行完毕后生成`dist`目录和`data_structure_demos_hans.egg-info`目录。后者暂时不用理会，而`dist`目录下有`data-structure-demos-hans-0.1.tar.gz`，这就是我们的源码包。

接下来运行`pip install data-structure-demos-hans-0.1.tar.gz`安装我们刚刚打好的包：

```
Looking in indexes: https://mirrors.aliyun.com/pypi/simple
Processing <project dir>\dist\data-structure-demos-hans-0.1.tar.gz
  Preparing metadata (setup.py) ... done
Building wheels for collected packages: data-structure-demos-hans
  Building wheel for data-structure-demos-hans (setup.py) ... done
  Created wheel for data-structure-demos-hans: filename=data_structure_demos_hans-0.1-py3-none-any.whl size=1943 sha256=14aa4f5dff5de02c7d2e6125ef3c0127ec6466a97929573b355545fea9800b3f
  Stored in directory: <%homepath%>\appdata\local\pip\cache\wheels\47\64\58\02429501ec651b154a43b87f2630ad2d34b69df5fb31ee1d49
Successfully built data-structure-demos-hans
Installing collected packages: data-structure-demos-hans
Successfully installed data-structure-demos-hans-0.1

```

运行`pip list`可看到这一行`data-structure-demos-hans 0.1`已经出现。此时进入`<python install dir>\Lib\site-packages`可以看到`binary_indexed_tree`文件夹和`data_structure_demos_hans-0.1.dist-info`文件夹。因此，调用树状数组的代码可以这么写：

```
from binary_indexed_tree.bit import BITPy

def main():
    a1 = [10, 20, 30, 40, 50, 60, 70, 80, 90, 100]
    b1 = BITPy(len(a1))
    for i, v in enumerate(a1):
        b1.add(i + 1, v)
    for i in range(1, len(a1) + 1):
        print(b1.sum(i))

if __name__ == '__main__':
    main()

```

制作 wheel 包（二进制包）
----------------

代码不需要变，执行`python setup.py bdist_wheel`即可编译出二进制包。同理，执行`python setup.py sdist bdist_wheel`可以同时编译出源码包和二进制包。

编译出二进制包的同时还会生成一个 build 文件夹，这个文件夹的内容可以无视。`dist`文件夹下会生成`data_structure_demos_hans-0.1-cp37-cp37m-win_amd64.whl`。`.whl`文件是`zip`压缩包，如果你好奇里面的内容，可以改文件名后缀直接查看。

安装 whl 包命令：`pip install --force-reinstall data_structure_demos_hans-0.1-cp37-cp37m-win_amd64.whl`。

C/C++ 扩展 Hello World：实现一些函数给 Python 层调用
---------------------------------------

新增文件夹`cpp_extension_hello`：

```
cpp_extension_hello
    hello.cpp
    __init__.py 空文件

```

和上一章类似，我们只需要修改`setup.py`和`hello.cpp`。先看`setup.py`的改动：

```
from setuptools import setup, find_packages, Extension

extensions = [
    Extension(
        'cpp_extension_hello.hello_cpp_extension',
        [
            'cpp_extension_hello/hello.cpp',
        ],
        language='c++'
    ),
]

setup(
    # omit other params
    ext_modules=extensions,
)

```

`Extension`类代表一个 C/C++ 模块。这里指定了模块名`hello_cpp_extension`，那么链接时就会去找`PyInit_hello_cpp_extension`这个函数，这个函数用于模块初始化。如果打算写 C 模块，可以省略`language='c++'`。

在写 C++ 代码前，最好先让 VSCode 能够找到`Python.h`。这次我们先不用 cmake，而是选择一个更轻量的方案：配置`.vscode\c_cpp_properties.json`。`Ctrl+Shift+P`选择`C/C++ Edit Configurations`，会自动生成上述文件。

```
{
    "configurations": [
        {
            // ...
            "includePath": [
                "${workspaceFolder}/**",
                "<python install path>\\include" // Python.h 所在的目录
            ],
            "intelliSenseMode": "windows-gcc-x64",
            "compilerPath": "<your mingw64 path>\\bin\\g++"
        }
    ],
    // ...
}

```

接下来看`hello.cpp`：

```
#define PY_SSIZE_T_CLEAN
#include <Python.h>
#include <iostream>
using std::cout;
using std::endl;

static PyObject* cpp_extension_hello(PyObject* self, PyObject* args) {
  const char* name = nullptr;
  if (!PyArg_ParseTuple(args, "s", &name)) {
    return nullptr;
  }

  cout << "[cpp_extension_hello] hello, " << name << endl;

  PyObject* res_dict = PyDict_New();
  PyObject* name_py = Py_BuildValue("s", name);
  PyObject* age_py = Py_BuildValue("i", 18);
  PyDict_SetItemString(res_dict, "name", name_py);
  PyDict_SetItemString(res_dict, "age", age_py);

  return res_dict;
}

static PyMethodDef cpp_extension_methods[] = {
    {"hello_from_cpp_extension", cpp_extension_hello, METH_VARARGS,
     "Print a hello message"},
    {NULL, NULL, 0, NULL}  // Sentinel is required
};

static struct PyModuleDef cpp_extension_module = {
    PyModuleDef_HEAD_INIT, "hello_cpp_extension", NULL, -1,
    cpp_extension_methods  // comment to regulate the behavior of clang-format
};

PyMODINIT_FUNC PyInit_hello_cpp_extension() {
  return PyModule_Create(&cpp_extension_module);
}

```

注意点：

1.  这个文件格式有一些要求，我模仿着[参考链接 3](https://codedamn.com/news/python/implementing-custom-python-c-extensions-step-by-step-guide) 写的。
2.  `static struct PyModuleDef cpp_extension_module`的`m_name`应该要和`setup.py`指定的一致？TODO
3.  `cpp_extension_methods`的哨兵是必要的，否则运行时会报错`module functions cannot set METH_CLASS or METH_STATIC`。
4.  `PyMethodDef.ml_name`指定了模块调用者使用的方法名为`hello_from_cpp_extension`。
5.  如果函数要返回 None，就使用`Py_RETURN_NONE;`。这里函数返回一个`dict`，意在简单体验一下 Python 层和 C++ 层通信的 API。

在此我希望用我本地的 g++ 来编译 cpp，所以需要建一个`setup.cfg`：

```
[build]
compiler=mingw32

```

如何查看支持的编译器：根据[参考链接 6](https://setuptools.pypa.io/en/latest/deprecated/distutils/configfile.html)，输入以下命令即可。

```
python setup.py build_ext --help-compiler

```

输出示例：

```
List of available compilers:
  --compiler=bcpp     Borland C++ Compiler
  --compiler=cygwin   Cygwin port of GNU C Compiler for Win32
  --compiler=mingw32  Mingw32 port of GNU C Compiler for Win32
  --compiler=msvc     Microsoft Visual C++
  --compiler=unix     standard UNIX-style compiler

```

之后，我们执行`python setup.py sdist`并不会编译 cpp，因为这个编译过程发生在`pip install data-structure-demos-hans-0.1.tar.gz`阶段。接下来如果不出意外的话，就要出意外了 TAT。`pip install`报错：

```
        File "<python install dir>\lib\distutils\cygwinccompiler.py", line 86, in get_msvcr
          raise ValueError("Unknown MS Compiler version %s " % msc_ver)
      ValueError: Unknown MS Compiler version 1916

```

幸好，经过一番搜索，找到了[参考链接 2](https://blog.51cto.com/lang13002/6723721)。按其指示，我们新增一小段代码就 OK：

```
        elif msc_ver == '1916':
            # manually add because "Unknown MS Compiler version 1916". VS2015
            return ['msvcr120']

```

`pip install`成功情况下的输出如下：

```
Looking in indexes: https://mirrors.aliyun.com/pypi/simple
Processing <project dir>\dist\data-structure-demos-hans-0.1.tar.gz
  Preparing metadata (setup.py) ... done
Building wheels for collected packages: data-structure-demos-hans
  Building wheel for data-structure-demos-hans (setup.py) ... done
  Created wheel for data-structure-demos-hans: filename=data_structure_demos_hans-0.1-cp37-cp37m-win_amd64.whl size=23786 sha256=96cfef2a169f3edda75a86d8303fee8724b7b32bcb496328d7161404807cce85
  Stored in directory: <%homepath%>\appdata\local\pip\cache\wheels\47\64\58\02429501ec651b154a43b87f2630ad2d34b69df5fb31ee1d49
Successfully built data-structure-demos-hans
Installing collected packages: data-structure-demos-hans
  Attempting uninstall: data-structure-demos-hans
    Found existing installation: data-structure-demos-hans 0.1
    Uninstalling data-structure-demos-hans-0.1:
      Successfully uninstalled data-structure-demos-hans-0.1
Successfully installed data-structure-demos-hans-0.1

```

安装成功后，可以找到一个 pyd 文件`<python install path>\Lib\site-packages\cpp_extension_hello\hello_cpp_extension.cp37-win_amd64.pyd`。最后写段代码调用一下：

```
from cpp_extension_hello.hello_cpp_extension import hello_from_cpp_extension

def main():
    res_dict = hello_from_cpp_extension('hans')
    print(res_dict)

if __name__ == '__main__':
    main()

```

### 逆向：查看 pyd 文件的内容并调用之

pyd 文件是 dll 文件，可以直接用 IDA 打开并分析。用 IDA 打开后，熟悉的`PyInit_hello_cpp_extension`就映入眼帘了。

```
__int64 __fastcall PyInit_hello_cpp_extension(__int64 a1, __int64 a2) {
  return PyModule_Create2(a1, a2, 1013LL, &unk_3B4BF3020);
}

```

而我们导出的`hello_from_cpp_extension`函数就在它隔壁。

```
__int64 __fastcall sub_3B4BF1370(const char *a1, __int64 a2, __int64 a3)
{
  size_t v3; // rax
  __int64 v4; // rdx
  _BYTE *v5; // rbx
  char v6; // dl
  __int64 v7; // rax
  __int64 v8; // rdx
  __int64 v9; // rbx
  __int64 v10; // rdi
  __int64 v11; // rsi
  __int64 v13[4]; // [rsp+28h] [rbp-20h] BYREF

  v13[0] = 0LL;
  if ( !(unsigned int)PyArg_ParseTuple_SizeT(a1, a2, &unk_3B4BF4000, a3, v13) )
    return 0LL;
  ZSt16__ostream_insertIcSt11char_traitsIcEERSt13basic_ostreamIT_T0_ES6_PKS3_x(
    a1,
    a2,
    "[cpp_extension_hello] hello, ",
    &ZSt4cout,
    29LL);
  if ( v13[0] )
  {
    v3 = strlen(a1);
    ZSt16__ostream_insertIcSt11char_traitsIcEERSt13basic_ostreamIT_T0_ES6_PKS3_x(a1, a2, v13[0], &ZSt4cout, v3);
  }
  else
  {
    ZNSt9basic_iosIcSt11char_traitsIcEE5clearESt12_Ios_Iostate(
      a1,
      a2,
      *(_DWORD *)((char *)&ZSt4cout + *((_QWORD *)&ZSt4cout - 3) + 32) | 1u);
  }
  v4 = *((_QWORD *)&ZSt4cout - 3);
  v5 = *(_BYTE **)((char *)&ZSt4cout + v4 + 240);
  if ( !v5 )
    ZSt16__throw_bad_castv();
  if ( v5[56] )
  {
    v6 = v5[67];
  }
  else
  {
    ZNKSt5ctypeIcE13_M_widen_initEv(a1, a2, v4, *(_QWORD *)((char *)&ZSt4cout + v4 + 240));
    v6 = (*(__int64 (__fastcall **)(const char *, __int64, __int64, _BYTE *))(*(_QWORD *)v5 + 48LL))(a1, a2, 10LL, v5);
  }
  v7 = ZNSo3putEc(a1, a2, (unsigned int)v6, &ZSt4cout);
  ZNSo5flushEv(a1, a2, v8, v7);
  v9 = PyDict_New();
  v10 = Py_BuildValue_SizeT(a1, Py_BuildValue_SizeT, v13[0], &unk_3B4BF4000);
  v11 = Py_BuildValue_SizeT(v10, Py_BuildValue_SizeT, 18LL, "i");
  PyDict_SetItemString(PyDict_SetItemString, v11, "name", v9, v10);
  PyDict_SetItemString(PyDict_SetItemString, v11, "age", v9, v11);
  return v9;
}

```

我们将文件名改为`pyd_demo.cp37-win_amd64.pyd`或`pyd_demo.pyd`，并用 PETools 修改其导出表的 PyInit 函数名为`PyInit_pyd_demo`，然后就可以在同一文件夹下写 Python 代码调用了。

```
from pyd_demo import hello_from_cpp_extension

res_dict = hello_from_cpp_extension('hans')
print(res_dict)

```

### Optional：类型提示支持

虽然调用成功了，但是没有类型提示，不太友好。首先我们需要加一个`.pyi`文件。`.pyi`文件的名字要和 import 的文件名一致，所以文件名是`hello_cpp_extension`。`hello_cpp_extension.pyi`：

```
def hello_from_cpp_extension(name: str) -> dict:
    pass

```

接下来我们要把`.pyi`文件打包进去，但默认情况下它不会被打包进去。根据[参考链接 4](https://github.com/pypa/setuptools/issues/3136)，如果你的`setuptools`版本大于等于`69.0.0`，那么可以比较容易地做到这件事。但我的是`68.0.0`，因此需要：

1.  传递选项`include_package_data=True`。
2.  新增`MANIFEST.in`：

```
recursive-include * *.pyi

```

之后再打包，就能看到`.pyi`文件被打包进去了。

Q：不需要加一个空的`py.typed`文件？A：是的。本项目场景比较简单，暂时不需要理会 PEP-561 规范。

在 C++ 扩展中定义 Python 数据结构实战：用 C++ 扩展实现树状数组
----------------------------------------

`binary_indexed_tree`目录新增两个文件`bit.cpp`和`bit_cpp_extension.pyi`：

```
binary_indexed_tree
    bit.cpp
    bit.py
    bit_cpp_extension.pyi
    __init__.py

```

`setup.py`例行更改`extensions`数组：

```
extensions = [
    Extension(
        'binary_indexed_tree.bit_cpp_extension',
        [
            'binary_indexed_tree/bit.cpp',
        ],
        language='c++'
    ),
    # ...
]

```

在 C++ 扩展中定义一个 Python 数据结构至少需要定义两个数据结构，类类型数据结构和对象类型数据结构。

对象类型数据结构用来装你需要的数据，但是必须包含一个`PyObject_HEAD`头部。本章新增的对象类型数据结构如下：

```
struct BITObject {
  PyObject_HEAD int n;
  int* a;
};

```

类类型数据结构是一个`PyTypeObject`类型的实例。这家伙有相当多的字段，但在本例中我们需要提供的很少。结合一个叫`pysegmenttree`的项目（[具体参考的代码传送门](https://github.com/greshilov/pysegmenttree/blob/master/pysegmenttree/_extensions/intsegmenttree.h)）和[参考链接 5](https://zhuanlan.zhihu.com/p/106773873)，和上面定义的对象类型数据结构，本章新增的类类型数据结构**主要**需要提供的字段只有`tp_dealloc`、`tp_flags`、`tp_doc`、`tp_methods`、`tp_new = bit_new`。其他字段都保持为 nullptr 即可。

```
static PyTypeObject BitType = {
    PyVarObject_HEAD_INIT(NULL, 0) "binary_indexed_tree.bit_cpp_extension.BIT",
    sizeof(BITObject),
    .tp_dealloc = (destructor)bit_dealloc,
    .tp_flags = Py_TPFLAGS_DEFAULT | Py_TPFLAGS_BASETYPE,
    .tp_doc = "Binary Indexed Tree",
    .tp_methods = bit_methods,
    .tp_new = bit_new,
};

```

1.  `PyVarObject_HEAD_INIT(NULL, 0)`是固定写法。
2.  根据参考链接 5，`tp_name = binary_indexed_tree.bit_cpp_extension.BIT`不是随意定义的字符串。它应对应 Python 调用者的 import 方式`from binary_indexed_tree.bit_cpp_extension import BIT`。
3.  `tp_basicsize`指定为`sizeof(对象类型数据结构)`。
4.  `tp_doc`随意写即可。

接下来咱把目光放到模块初始化函数。相比于上一章的一句话`return PyModule_Create(&cpp_extension_module);`，这次的代码量要大得多了，但也是看着参考链接 5 和`pysegmenttree`项目，按格式来即可。

```
PyMODINIT_FUNC PyInit_bit_cpp_extension() {
  PyObject* m;
  if (PyType_Ready(&BitType) < 0) {
    return nullptr;
  }
  m = PyModule_Create(&cpp_extension_module);
  if (m == nullptr) {
    return nullptr;
  }
  Py_INCREF(&BitType);
  if (PyModule_AddObject(m, "BIT", (PyObject*)&BitType) < 0) {
    Py_DECREF(&BitType);
    Py_DECREF(m);
    return nullptr;
  }
  return m;
}

```

我们接着看类类型数据结构涉及的几个对象和方法。

`bit_methods`：我们希望提供的类的成员方法。

```
static PyMethodDef bit_methods[] = {
    {"sum", (PyCFunction)bit_sum, METH_VARARGS | METH_KEYWORDS,
     "Queries prefix sum"},
    {"add", (PyCFunction)bit_add, METH_VARARGS | METH_KEYWORDS,
     "Performs the add operation"},
    {nullptr},  // Sentinel is required
};

```

`bit_sum`、`bit_add`的函数签名模仿着参考链接 5 写就行。`bit_sum`、`bit_add`和它们共用的入参校验逻辑`jdg_idx`如下：

```
static bool jdg_idx(BITObject* self, int idx) {
  if (idx < 0) {
    PyErr_SetString(PyExc_ValueError,
                    "idx should be greater than or equal to 0");
    return false;
  }
  if (idx > self->n) {
    PyErr_SetString(PyExc_ValueError,
                    "idx should be less than or equal to array size");
    return false;
  }
  return true;
}

static PyObject* bit_add(BITObject* self, PyObject* args, PyObject* kwds) {
  int idx = 0, v = 0;
  if (!PyArg_ParseTuple(args, "ii", &idx, &v)) {
    return nullptr;
  }

  if (!jdg_idx(self, idx)) {
    return nullptr;
  }

  for (; idx <= self->n; idx += idx & -idx) self->a[idx] += v;

  Py_RETURN_NONE;
}

static PyObject* bit_sum(BITObject* self, PyObject* args, PyObject* kwds) {
  int idx = 0;
  if (!PyArg_ParseTuple(args, "i", &idx)) {
    return nullptr;
  }

  if (!jdg_idx(self, idx)) {
    return nullptr;
  }

  int res = 0;
  for (; idx; idx -= idx & -idx) res += self->a[idx];
  PyObject* res_py = PyLong_FromLong(res);
  return res_py;
}

```

这里用到了几个 API：

1.  `PyArg_ParseTuple`用来拿入参。`ii`表示取两个 int。
2.  `PyErr_SetString`用来抛异常。这里我抛出了`ValueError`。
3.  因为几个方法返回值都得是`PyObject*`，所以需要用`PyLong_FromLong`把 C++ 的 int 转为`PyObject*`。

抽象出`jdg_idx`的过程：原本的写法是：

```
  if (idx < 0) {
    PyErr_SetString(PyExc_ValueError,
                    "idx should be greater than or equal to 0");
    return nullptr;
  }

```

那么我们可以将其变为`PyErr_SetString; return false/true; return nullptr;`，于是可以抽象出一个返回`bool`的函数，然后这么调用：

```
  if (!jdg_idx(self, idx)) {
    return nullptr;
  }

```

`bit_new`：进行对象的初始化工作。除了`type->tp_alloc(type, 0)`是没见过的固定写法外，其他内容我们都比较熟悉了。`tp_alloc`顾名思义就是给对象类型数据结构分配内存。

```
static PyObject* bit_new(PyTypeObject* type, PyObject* args, PyObject* kwds) {
  BITObject* self = (BITObject*)type->tp_alloc(type, 0);
  int n = 0;
  if (!PyArg_ParseTuple(args, "i", &n)) {
    Py_DECREF(self);
    return nullptr;
  }
  self->n = n;
  self->a = new int[n + 1]();
  return (PyObject*)self;
}

```

`bit_dealloc`：进行堆内存释放工作，依据参考链接 5，不写也能跑，但会造成内存泄露。

```
static void bit_dealloc(BITObject* self) {
  delete[] self->a;
  Py_TYPE(self)->tp_free(self);
}

```

即使我们不提供该函数，Python 解释器也会调用`Py_TYPE(self)->tp_free(self)`。但我们提供该函数后，就不得不手动加上这句话了。

最后按惯例，展示一下我们刚刚实现的树状数组的用法：

```
from binary_indexed_tree.bit_cpp_extension import BIT

def bit_cpp(a: List[int]):
    b1 = BIT(len(a))
    for i, v in enumerate(a):
        b1.add(i + 1, v)
    for i in range(1, len(a) + 1):
        print(b1.sum(i))
    b1.add(9, 113964)
    print(b1.sum(len(a)))
    b2 = BIT(20)
    try:
        b2.add(-2, 10)
    except ValueError as e:
        print(e)
    try:
        b2.sum(-1)
    except ValueError as e:
        print(e)
    try:
        b2.sum(22)
    except ValueError as e:
        print(e)
    try:
        b2.add(21, 10)
    except ValueError as e:
        print(e)
    b2.add(2, 20)
    b2.add(4, 30)
    print(b2.sum(5))  # 50

```

餐后甜点：用 C++ 扩展实现单调递减数组的二分查找函数
----------------------------

本节我们模仿`bisect`包的 API，用 C++ 扩展实现单调递减数组的二分查找函数。

### 支持 int 数组的二分查找

新增模块`bisect_decr_cpp`，提供的方法如下：

```
const char* lower_bound_intro =
    "The return value i is such that all e in a[:i] have e > x, and all e in "
    "a[i:] have e <= x";
const char* upper_bound_intro =
    "The return value i is such that all e in a[:i] have e >= x, and all e in "
    "a[i:] have e < x";

static PyMethodDef cpp_extension_methods[] = {
    {"dec_lower_bound", dec_lower_bound, METH_VARARGS, lower_bound_intro},
    {"dec_upper_bound", dec_upper_bound, METH_VARARGS, upper_bound_intro},
    {NULL, NULL, 0, NULL}  // Sentinel is required
};

```

`dec_lower_bound`和`dec_upper_bound`几乎完全一致，只有要用到的比较运算符不一样，所以我抽象出了`dec_binary_search`方法，并用`op`区分是哪个方法，`op == Py_GT`的为`dec_lower_bound`，`op == Py_GE`的为`dec_upper_bound`。这里选用`Python.h`提供的`Py_GT, Py_GE`宏是在给下一节实现任意可比较对象的数组的二分查找做铺垫。

```
static bool jdg_arr(PyObject* a_py) {
  if (!PyList_Check(a_py)) {
    PyErr_SetString(PyExc_ValueError,
                    "Plz pass an integer array and an integer");
    return false;
  }
  return true;
}

static PyObject* dec_binary_search(PyObject* self, PyObject* args, int op) {
  PyObject* a_py = nullptr;
  int x = 0;
  if (!PyArg_ParseTuple(args, "Oi", &a_py, &x)) {
    PyErr_SetString(PyExc_ValueError,
                    "Plz pass an integer array and an integer");
    return nullptr;
  }
  if (!jdg_arr(a_py)) {
    return nullptr;
  }

  int n = PyList_GET_SIZE(a_py);
  int l = 0, r = n;
  while (l < r) {
    int mid = (l + r) >> 1;
    PyObject* num_py = PyList_GET_ITEM(a_py, mid);
    if (!PyLong_Check(num_py)) {
      PyErr_SetString(PyExc_ValueError,
                      "Every elements of the array should be integer");
      return nullptr;
    }
    int num = PyLong_AS_LONG(num_py);
    bool cmp_res = false;
    if (op == Py_GT) {
      cmp_res = num > x;
    } else {
      cmp_res = num >= x;
    }
    if (cmp_res) {
      l = mid + 1;
    } else {
      r = mid;
    }
  }

  PyObject* res_py = PyLong_FromLong(l);
  return res_py;
}

static PyObject* dec_lower_bound(PyObject* self, PyObject* args) {
  return dec_binary_search(self, args, Py_GT);
}

static PyObject* dec_upper_bound(PyObject* self, PyObject* args) {
  return dec_binary_search(self, args, Py_GE);
}

```

在此我们看到一个具有普适性的规律：C++ 代码和 Python 代码属于两个抽象层，因此我们的工作流程一般如下：先把 Python 抽象层的语言翻译为 C++ 抽象层的语言，接着判断参数的合法性，然后做些计算，最后再翻译回 Python 抽象层的语言。

1.  抽象出`jdg_arr`的思路同上一节。
2.  `PyList_Check, PyLong_Check`用于校验参数类型，`PyList_GET_SIZE`拿列表长度，`PyList_GET_ITEM`根据下标拿列表元素。
3.  为了保证复杂度为`O(logn)`，不能遍历整个数组，于是检验单个元素的类型的逻辑不得不和二分查找的逻辑耦合在一起。

最后按惯例，展示一下其用法：

```
from bisect_decr.bisect_decr_cpp import dec_lower_bound, dec_upper_bound

def case1():
    a = [100, 90, 80, 70, 60, 50, 40, 30, 20, 10]
    l1 = dec_lower_bound(a, 101)
    l2 = dec_lower_bound(a, 61)
    l3 = dec_lower_bound(a, 9)
    l4 = dec_lower_bound(a, 100)
    l5 = dec_lower_bound(a, 60)
    l6 = dec_lower_bound(a, 10)
    u1 = dec_upper_bound(a, 101)
    u2 = dec_upper_bound(a, 61)
    u3 = dec_upper_bound(a, 9)
    u4 = dec_upper_bound(a, 100)
    u5 = dec_upper_bound(a, 60)
    u6 = dec_upper_bound(a, 10)
    print(l1, l2, l3, l4, l5, l6)  # 0 4 10 0 4 9
    print(u1, u2, u3, u4, u5, u6)  # 0 4 10 1 5 10

def case2():
    a = [90, 90, 90, 80, 80, 70]
    l1 = dec_lower_bound(a, 91)
    l2 = dec_lower_bound(a, 90)
    l3 = dec_lower_bound(a, 85)
    l4 = dec_lower_bound(a, 80)
    l5 = dec_lower_bound(a, 75)
    l6 = dec_lower_bound(a, 70)
    l7 = dec_upper_bound(a, 60)
    u1 = dec_upper_bound(a, 91)
    u2 = dec_upper_bound(a, 90)
    u3 = dec_upper_bound(a, 85)
    u4 = dec_upper_bound(a, 80)
    u5 = dec_upper_bound(a, 75)
    u6 = dec_upper_bound(a, 70)
    u7 = dec_upper_bound(a, 60)
    print(l1, l2, l3, l4, l5, l6, l7)  # 0 0 3 3 5 5 6
    print(u1, u2, u3, u4, u5, u6, u7)  # 0 3 3 5 5 6 6

def main():
    case1()
    case2()

if __name__ == '__main__':
    main()

```

### 支持任意可比较对象的数组的二分查找

首先，原本的`int x = 0`参数要改为`PyObject*`：

```
  PyObject* a_py = nullptr;
  PyObject* x = nullptr;
  if (!PyArg_ParseTuple(args, "OO", &a_py, &x)) {
    PyErr_SetString(
        PyExc_ValueError,
        "Plz pass a comparable object array and a comparable object");
    return nullptr;
  }
  // ...

```

接着微调参数校验逻辑，不是很重要，在此不赘述。然后我们来看本章的关键 API：`int PyObject_RichCompareBool(PyObject *, PyObject *, int)`。第三个参数是运算符，`Py_GT, Py_GE`分别表示`>, >=`。返回值为`1, 0, -1`，0 和 1 就相当于 bool 结果，-1 表示由对象没有重载相关运算符等原因导致比较失败。如果不处理 - 1 这个返回值，会导致 Python 解释器抛出`SystemError`，取`e.__cause__`可以拿到`TypeError`，比如：`TypeError: '>' not supported between instances of 'Person' and 'Person'`。我的代码处理了这种情况，因此仍然可以返回`ValueError`，比如：`ValueError: Comparison error. Index: 0. Operator: ">"`。

改动到的核心代码：

```
  auto get_cmp_error_info = [op](int idx) {
    string s = "Comparison error. Index: ";
    s += to_string(idx);
    s += ". Operator: \"";
    if (op == Py_GT)
      s += ">";
    else
      s += ">=";
    s += "\"";
    return s;
  };

  int n = PyList_GET_SIZE(a_py);
  int l = 0, r = n;
  while (l < r) {
    int mid = (l + r) >> 1;
    PyObject* num_py = PyList_GET_ITEM(a_py, mid);
    int cmp_res = PyObject_RichCompareBool(num_py, x, op);
    if (cmp_res == -1) {
      string cmp_err_info = get_cmp_error_info(mid);
      PyErr_SetString(PyExc_ValueError, cmp_err_info.c_str());
      return nullptr;
    }
    if (cmp_res) {
      l = mid + 1;
    } else {
      r = mid;
    }
  }

```

使用：

```
from bisect_decr.bisect_decr_cpp import dec_lower_bound, dec_upper_bound

def test_comparable_object_array():
    class Person():
        def __init__(self, age: int) -> None:
            self.age = age

        def __ge__(self, other):
            return self.age >= other.age

        def __gt__(self, other):
            return self.age > other.age

    persons = [Person(60), Person(25), Person(18), Person(18), Person(6)]
    test_arr = [Person(100), Person(60), Person(33), Person(25), Person(23), Person(18), Person(12), Person(6), Person(4)]
    res_l = [dec_lower_bound(persons, v) for v in test_arr]
    assert res_l == [0, 0, 1, 1, 2, 2, 4, 4, 5]

```

单测：pytest
---------

```
pip install -U pytest
pip install -U pytest-html
# 测试安装是否成功
pytest --version

```

常规的断言使用`assert`关键字就行。异常断言的写法：`with pytest.raises(ValueError) as e_info: # write code here that might throw an exception`。举例：

```
    b2 = BIT(20)
    with pytest.raises(ValueError) as e_info:
        b2.add(-2, 10)
    assert 'idx should be greater than or equal to 0' in str(e_info.value)

```

运行单测命令：

```
pytest --html=coverage/report.html

```

上传到 PyPI
--------

接下来开始污染 PyPI！PyPI 为了便于大家练习上传过程，所以还同步提供了功能与 PyPi 完全一样但相互隔离的测试环境：[https://test.pypi.org/](https://test.pypi.org/)。

1.  注册账号、激活 Email。
2.  现在 PyPI 也开始逼你激活 Two factor authentication (2FA) 了。第一步，网站会给你一些 16 位的 Recovery Code，你要找个地方保存它们。第二步是输入其中一个 Recovery Code。第三步是点击`Add 2FA with authentication application`按钮，用 Authenticator APP（IPhone）扫描二维码，然后输入验证码。
3.  生成 API Token。[入口](https://test.pypi.org/manage/account/#api-tokens)。Token 名随意，Scope 选 All Projects。

安装 twine：`pip install twine`。验证 twine 已经安装成功：`twine --version`。

用 twine 将包上传到测试环境：

```
twine upload --repository testpypi dist/*

```

运行以上命令后，如果你的电脑没创建`<%homepath%>/.pypirc`，会出现 Enter your username 和 Enter your password 提示，其中 username 输入`__token__`（就是这个字符串，不是 Token 名或其他），password 输入 token 值，然后按 Enter。根据[帮助文档](https://test.pypi.org/help#apitoken)，也可以创建`<%homepath%>/.pypirc`：

```
[testpypi]
  username = __token__
  password = <api token>

```

上传到正式环境：

```
twine upload dist/*

```

上传成功后就能用 pip 安装了。

注意点：

1.  根据帮助文档，你想再次上传一个更改过或没更改过的包但不修改版本号都是不可能的。
2.  `The author of this package has not provided a project description`指的是你没有指定`long_description`。

```
from setuptools import setup, find_packages, Extension

def get_long_description():
    with open('README.md', 'r', encoding='utf-8') as f:
        content = f.read()
        return content

long_description = get_long_description()

setup(
    # ...
    long_description=long_description,
    long_description_content_type='text/markdown',
)

```

参考资料
----

1.  Python 使用 setuptools 打包自己的分发包并使用举例：[https://blog.csdn.net/hxxjxw/article/details/124298543](https://blog.csdn.net/hxxjxw/article/details/124298543)
2.  Windows10 下用 mingw 编译 python 扩展库：[https://blog.51cto.com/lang13002/6723721](https://blog.51cto.com/lang13002/6723721)
3.  [https://codedamn.com/news/python/implementing-custom-python-c-extensions-step-by-step-guide](https://codedamn.com/news/python/implementing-custom-python-c-extensions-step-by-step-guide)
4.  Include type information by default (`*.pyi`, `py.typed`)：[https://github.com/pypa/setuptools/issues/3136](https://github.com/pypa/setuptools/issues/3136)
5.  使用 c/c++ 编写 python 扩展（三）：自定义 Python 内置类型：[https://zhuanlan.zhihu.com/p/106773873](https://zhuanlan.zhihu.com/p/106773873)
6.  [https://setuptools.pypa.io/en/latest/deprecated/distutils/configfile.html](https://setuptools.pypa.io/en/latest/deprecated/distutils/configfile.html)![](https://avatar.52pojie.cn/images/noavatar_middle.gif)FDL 看起来可以用 C++ 实现一些底层的东西，然后给 Python 调用![](https://static.52pojie.cn/static/image/smiley/default/4.gif) ![](https://avatar.52pojie.cn/data/avatar/001/90/61/77_avatar_middle.jpg) hans7

> [FDL 发表于 2024-5-13 17:44](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=50353140&ptid=1923720)  
> 看起来可以用 C++ 实现一些底层的东西，然后给 Python 调用

是的。下一个 pytorch 和 tensorflow 等你来实现~![](https://avatar.52pojie.cn/images/noavatar_middle.gif)xiaopo 不是 c++ 写好的插件给 python 调用吗 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) sev1n 膜拜一下 ![](https://avatar.52pojie.cn/data/avatar/001/90/61/77_avatar_middle.jpg) hans7

> [xiaopo 发表于 2024-5-13 19:01](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=50353532&ptid=1923720)  
> 不是 c++ 写好的插件给 python 调用吗

是啊，同一个东西的不同说法嘛 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) GVOPwZWhL2IFcIZ 收藏一下，正好这几天要用 C++ 也有这想法。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)zd53011 没看明白 ![](https://avatar.52pojie.cn/data/avatar/001/90/61/77_avatar_middle.jpg) hans7

> [zd53011 发表于 2024-5-14 14:29](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=50360233&ptid=1923720)  
> 没看明白

明明这么通俗易懂… 哪块没懂？![](https://avatar.52pojie.cn/images/noavatar_middle.gif)wojiushiliu

> [hans7 发表于 2024-5-14 16:45](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=50361722&ptid=1923720)  
> 明明这么通俗易懂… 哪块没懂？

对于外行确实不好懂
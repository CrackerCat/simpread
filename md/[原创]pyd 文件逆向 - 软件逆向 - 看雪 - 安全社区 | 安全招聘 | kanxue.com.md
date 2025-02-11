> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-285496.htm)

> [原创]pyd 文件逆向

官方文档:
=====

[https://cython.readthedocs.io/en/latest/src/quickstart/build.html](https://bbs.kanxue.com/target-hRKU2djoRKaZYBS_2BgZknVfdfefztKT26BVS1m2KR3G1EH0UmHYcqXIyZmHwzDv4BXPGiNFS8vp3Uno_2F6YurUb_2BUlNg_2BQ0FPO.htm)

[https://cython.org/](https://bbs.kanxue.com/target-J_2BWutLQli0xc1NkfDK7bw16jqx1rQZx7.htm)

编译
==

手动编译
----

按照官方文档编译一个 pyd 文件:

python pip 安装模块 cython

```
pip install Cython
```

编写一个 hello.pyx 脚本, 里面随便写一个函数

```
def say_hello_to(name):
    print(f"Hello {name}!")
```

在同目录编写一个 setup.py 脚本，里面的文件名为固定的内容，文件名为刚刚写的 py 脚本完整名称

```
from setuptools import setup
from Cython.Build import cythonize
 
setup(
    name='Hello world app',
    ext_modules=cythonize("hello.pyx"),
)
```

在命令行执行命令, 来编译这个文件

```
python setup.py build_ext --inplace
```

编译成果后会在同级目录下生成很多东西:

build 目录 存放一些编译生成的中间文件

xxxx.c py 文件到 c 源码文件，中间包含了完整的 c 代码

xxxxx.cpyyyy-win_amd64.pyd 编译好的文件 xxxx 为 py 文件的命名 yyyy 为 python 的版本号 ，这个文件可只保留原始文件名和后缀. pyd 例如 bbbbb.cp312-win_amd64.pyd -> bbbbb.pyd

> 如果两个文件在同时在同级目录下，python 会优先选择带版本号的，如果没有再选择不带版本号的文件

如何调用?

在同级目录下，直接打开 python 使用 import xxxx 即可调用

```
import hello
from hello import say_hello_to
```

查看 xxxx.c

里面有非常多的结构，并且有 python 代码转换的 c 源代码，我们可以在这里看到 cython 的转换规则，和大体的逻辑

```
static PyObject *__pyx_pf_5bbbbb_say_hello_to(CYTHON_UNUSED PyObject *__pyx_self, PyObject *__pyx_v_name) {
  PyObject *__pyx_r = NULL;
  __Pyx_RefNannyDeclarations
  PyObject *__pyx_t_1 = NULL;
  PyObject *__pyx_t_2 = NULL;
  int __pyx_lineno = 0;
  const char *__pyx_filename = NULL;
  int __pyx_clineno = 0;
  __Pyx_RefNannySetupContext("say_hello_to", 1);
 
  /* "bbbbb.pyx":4
 *
 * def say_hello_to(name):
 *  print(f"Hello {name}")             # <<<<<<<<<<<<<<
 */
  __pyx_t_1 = __Pyx_PyObject_FormatSimple(__pyx_v_name, __pyx_empty_unicode); if (unlikely(!__pyx_t_1)) __PYX_ERR(0, 4, __pyx_L1_error)
  __Pyx_GOTREF(__pyx_t_1);
  __pyx_t_2 = __Pyx_PyUnicode_Concat(__pyx_kp_u_Hello, __pyx_t_1); if (unlikely(!__pyx_t_2)) __PYX_ERR(0, 4, __pyx_L1_error)
  __Pyx_GOTREF(__pyx_t_2);
  __Pyx_DECREF(__pyx_t_1); __pyx_t_1 = 0;
  __pyx_t_1 = __Pyx_PyObject_CallOneArg(__pyx_builtin_print, __pyx_t_2); if (unlikely(!__pyx_t_1)) __PYX_ERR(0, 4, __pyx_L1_error)
  __Pyx_GOTREF(__pyx_t_1);
  __Pyx_DECREF(__pyx_t_2); __pyx_t_2 = 0;
  __Pyx_DECREF(__pyx_t_1); __pyx_t_1 = 0;
 
  /* "bbbbb.pyx":3
 *
 *
 * def say_hello_to(name):             # <<<<<<<<<<<<<<
 *  print(f"Hello {name}")
 */
 
  /* function exit code */
  __pyx_r = Py_None; __Pyx_INCREF(Py_None);
  goto __pyx_L0;
  __pyx_L1_error:;
  __Pyx_XDECREF(__pyx_t_1);
  __Pyx_XDECREF(__pyx_t_2);
  __Pyx_AddTraceback("bbbbb.say_hello_to", __pyx_clineno, __pyx_lineno, __pyx_filename);
  __pyx_r = NULL;
  __pyx_L0:;
  __Pyx_XGIVEREF(__pyx_r);
  __Pyx_RefNannyFinishContext();
  return __pyx_r;
}
```

这里编译的没有带符号，所以逆向的话，ida 载入，很多函数都是没有函数名称的

手动编译 1 带符号
----------

在 Linux 下默认带有符号，Windows 下默认不带符号，需要增加参数来生成详细的 PDB，并 ida 加载 PDB 才能看到更详细内容

我们可以通过修改 setup.py 来编译一个带符号的

```
from setuptools import setup, Extension
from Cython.Build import cythonize
 
ext_module = [
    Extension(
        ,
        sources=["http://www.py"],
        extra_compile_args=["/Zi"],
        extra_link_args=["/DEBUG"]
    )
]
 
setup(
    name = "www",
    ext_modules = cythonize(ext_module,annotate=True)
)
```

![](https://bbs.kanxue.com/upload/attach/202502/880848_8X64S6FJXW49F5Q.png)

也可以在命令行编译时，增加 --debug 参数，但需要在安装 Python 时勾选 Download debug binaries ，否则会提示 104: 无法打开文件 “python312_d.lib”

各种表达式解析
=======

函数
--

默认情况下, 对外函数，可以搜索名称, 因为是全局变量，所以，变量都会放在一起

![](https://bbs.kanxue.com/upload/attach/202502/880848_EN82U3C4UY7HP6X.png)

查找引用，可以发现结构体:

![](https://bbs.kanxue.com/upload/attach/202502/880848_TCGXUEWKSAE2E4M.png)

![](https://bbs.kanxue.com/upload/attach/202502/880848_NYUVYHY2442N2EB.png)

windows 下会多出一层调用 pw -> pf ， pf 是真正的地址

![](https://bbs.kanxue.com/upload/attach/202502/880848_M76W2WEVMRPE5FJ.png)

Windows Release 版本也是一致: 就是符号没有这么明显，要稍微熟悉一点 pyd 的结构体。

![](https://bbs.kanxue.com/upload/attach/202502/880848_KNQ89KJDQ8JABRW.png)

Linux 默认情况下没有，可能被编译器优化掉 了，但 c 语言源码上基本一致，pw->pf 函数名

![](https://bbs.kanxue.com/upload/attach/202502/880848_AJZXVWXKJS7MEY9.png)

### 模块内函数 / 变量引用

函数的结构体大多都放在一起的，可以上下查找有没有其他的函数，直接搜索函数名，有时候可能会搜不到

![](https://bbs.kanxue.com/upload/attach/202502/880848_H8BHU98KRVGQ7GZ.png)

```
def get_list():
    return [1,2,3,4,991,776,229, 12.22]
 
def get_list1(index,end):
    a= get_list()
        return a
```

在模块中调用模块的内容，会先调用 PyDict_GetItem_KnownHash 中获取，

![](https://bbs.kanxue.com/upload/attach/202502/880848_ZFQZQ7WGUQUG83K.png)

寻找引用，一般是在 pymod_exec 完成初始化, xxxx+38) + 24

![](https://bbs.kanxue.com/upload/attach/202502/880848_FW7RD9UU6MHHKHH.png)

可以看到确实是 get_list 这个函数

![](https://bbs.kanxue.com/upload/attach/202502/880848_J8Z5BABWHB7HCRN.png)

对比有符号的发现，+24 是 ob_type v2 是函数地址

模块调用
----

```
import socket
import subprocess
 
socket.socket(socket.AF_INET,socket.SOCK_STREAM)
```

![](https://bbs.kanxue.com/upload/attach/202502/880848_CH3WB9JDZ3B5EFK.png)

调用后的模块地址在 v6, 然后又传给 v10， 就可以跟着 v10，看看有没有发生其他动作 一般来说，会使用 PyObject_GetAttr 来获取属性

![](https://bbs.kanxue.com/upload/attach/202502/880848_3TYH4GKJARGDTFE.png)

这种模块调用会有引用计数的特性, 可以看到 v17 为属性, 其中经过了 ob_refcnt ，比较经典。

![](https://bbs.kanxue.com/upload/attach/202502/880848_642TFU5CPBHRBF9.png)

再经过 Pyx_PyObject_FastCallDict 来调用，参数为 v21 v26

![](https://bbs.kanxue.com/upload/attach/202502/880848_3UYZ6SR6SUXNPDT.png)

有时候 ida 可能识别的参数不对，数组大小不对，需要手动判别，更改。

![](https://bbs.kanxue.com/upload/attach/202502/880848_3EVGTPKGQF23GG8.png)

可以看到 v21 的具体内容为 AF_INET

![](https://bbs.kanxue.com/upload/attach/202502/880848_WFH29FBSGQZGWZ6.png)

v26 的内容为 SOCK_STREAM

![](https://bbs.kanxue.com/upload/attach/202502/880848_X8J5AYDGJCF3Z8M.png)

基本上可以通过静态大致识别出一些关键代码

运算
--

### 加减乘除移位左移右移 xor 与或非

编写测试函数

```
def test_calc(a):
    res = a + 1
    res = res -10
    res = res * 998
    res = res / 100
    res = res * 200
    res = res // 30
    res = res ** 3
    res = res << 3
    res = res ^ a
    res = hex(res)[2:]
    res = res + '00'
    res = int(res, 16)
    res = res >> 4
    res = res | 0xf
    res = res & 0x1f
    res = ~rsa
    return res
    
```

```
_typeobject *__fastcall _pyx_pf_5bbbbb_10test_calc(_object *__pyx_v_a, _object *__pyx_self, __int64 a3)
{
  _typeobject *v3; // r12
  _typeobject *v5; // rbx
  _object *pyx_int_1; // r9
  _typeobject *ob_type; // rax
  unsigned __int64 ob_refcnt; // r8
  __int64 v9; // rax
  unsigned __int64 v10; // r8
  const char *v11; // rsi
  int v12; // edi
  int v13; // ebp
  _object *p_ob_base; // rax
  unsigned __int64 v15; // rdx
  __int64 v16; // rax
  __int64 v17; // rdx
  __int64 v18; // r8
  __int64 v19; // r9
  unsigned __int64 v20; // rdx
  _object *v21; // rdi
  int *v22; // rcx
  bool v23; // zf
  _object *v24; // rax
  __int64 v25; // rdx
  __int64 v26; // r8
  __int64 v27; // r9
  _object *v28; // rdi
  int *v29; // rcx
  _object *pyx_int_100; // r8
  _object *v31; // rax
  __int64 v32; // rax
  __int64 v33; // rdx
  __int64 v34; // r8
  __int64 v35; // r9
  _object *v36; // rdi
  int *v37; // rcx
  _object *v38; // rax
  __int64 v39; // rdx
  __int64 v40; // r8
  __int64 v41; // r9
  __int64 v42; // rdi
  int *v43; // rcx
  _object *v44; // r8
  _object *pyx_int_30; // r9
  unsigned __int64 v46; // rdx
  int *v47; // rcx
  __int64 v48; // rax
  __int64 v49; // rdx
  __int64 v50; // r8
  __int64 v51; // r9
  __int64 v52; // rdi
  int v53; // r8d
  unsigned int v54; // ecx
  int v55; // edx
  unsigned int v56; // r8d
  __int64 v57; // rax
  signed __int64 v58; // r8
  unsigned __int64 v59; // rcx
  __int64 v60; // rdx
  unsigned __int64 v61; // r8
  int *v62; // rcx
  _object *v63; // r8
  _object *pyx_int_3; // r9
  unsigned __int64 v65; // rdx
  int *v66; // rcx
  __int64 v67; // rax
  __int64 v68; // rdx
  __int64 v69; // r8
  _object *v70; // r9
  _object *v71; // rdi
  unsigned __int64 v72; // rcx
  __int64 v73; // rax
  int *v74; // rcx
  _object *v75; // rax
  _object *v76; // r14
  _typeobject *v77; // r8
  PyMappingMethods *tp_as_mapping; // r15
  __int64 v79; // rdx
  _typeobject *v80; // rdi
  __int64 v81; // r8
  __int64 v82; // r9
  __int64 v83; // rax
  int *v84; // rdi
  int *v85; // rsi
  int *v86; // rcx
  __int64 v87; // rax
  __int64 v88; // rdx
  __int64 v89; // r8
  __int64 v90; // r9
  _object *v91; // r8
  __pyx_mstate *v92; // rdx
  _object *pyx_int_16; // rcx
  _object *v94; // rdi
  int *v95; // rcx
  _object *pyx_int_4; // r9
  unsigned __int64 v97; // rdx
  int *v98; // rcx
  _object *pyx_int_15; // rsi
  unsigned __int64 v100; // rcx
  __int64 v101; // rax
  unsigned __int64 v102; // rdx
  __int64 v103; // rax
  int v104; // ecx
  __int64 v105; // rax
  __int64 v106; // rdx
  __int64 v107; // r8
  __int64 v108; // r9
  int *v109; // rdi
  __int64 v110; // rdx
  __int64 v111; // r8
  __int64 v112; // r9
  _typeobject *v113; // r14
  _object *args; // [rsp+38h] [rbp-30h] BYREF
 
  v3 = 0i64;
  v5 = 0i64;
  pyx_int_1 = _pyx_mstate_global->__pyx_int_1;
  ob_type = __pyx_self->ob_type;
  if ( ob_type != (_typeobject *)PyLong_Type )
  {
    if ( ob_type == (_typeobject *)PyFloat_Type )
      v9 = PyFloat_FromDouble(__pyx_v_a, __pyx_self, a3, pyx_int_1);
    else
      v9 = PyNumber_Add(__pyx_self, _pyx_mstate_global->__pyx_int_1);
LABEL_14:
    pyx_int_1 = (_object *)v9;
    goto LABEL_15;
  }
  ob_refcnt = __pyx_self[1].ob_refcnt;
  if ( (ob_refcnt & 1) == 0 )
  {
    if ( ob_refcnt >= 0x10 )
    {
      v10 = ob_refcnt >> 3;
      switch ( v10 * (1 - (__pyx_self[1].ob_refcnt & 3)) )
      {
        case 0xFFFFFFFFFFFFFFFEui64:
          v9 = PyLong_FromLongLong(
                 1 - (LODWORD(__pyx_self[1].ob_type) | ((unsigned __int64)HIDWORD(__pyx_self[1].ob_type) << 30)),
                 __pyx_self,
                 v10,
                 pyx_int_1);
          break;
        case 2ui64:
          v9 = PyLong_FromLongLong(
                 (LODWORD(__pyx_self[1].ob_type) | ((unsigned __int64)HIDWORD(__pyx_self[1].ob_type) << 30)) + 1,
                 __pyx_self,
                 v10,
                 pyx_int_1);
          break;
        default:
          v9 = (**((__int64 (__fastcall ***)(_object *, _object *))&PyLong_Type + 12))(
                 __pyx_self,
                 _pyx_mstate_global->__pyx_int_1);
          break;
      }
    }
    else
    {
      v9 = PyLong_FromLong(LODWORD(__pyx_self[1].ob_type) * (1 - (unsigned int)(ob_refcnt & 3)) + 1);
    }
    goto LABEL_14;
  }
  if ( pyx_int_1->ob_refcnt_split[0] != -1 )
    ++pyx_int_1->ob_refcnt_split[0];
LABEL_15:
  if ( !pyx_int_1 )
  {
    v11 = _pyx_f[0];
    v12 = 42;
    v13 = 3937;
    goto LABEL_204;
  }
  v5 = (_typeobject *)pyx_int_1;
  p_ob_base = &pyx_int_1->ob_type->ob_base.ob_base;
  if ( p_ob_base == PyLong_Type )
  {
    v15 = pyx_int_1[1].ob_refcnt;
    if ( (v15 & 1) != 0 )
    {
      v16 = PyLong_FromLong(0xFFFFFFF6i64);
    }
    else if ( v15 >= 0x10 )
    {
      v20 = v15 >> 3;
      switch ( v20 * (1 - (pyx_int_1[1].ob_refcnt & 3)) )
      {
        case 0xFFFFFFFFFFFFFFFEui64:
          v16 = PyLong_FromLongLong(
                  -(__int64)(LODWORD(pyx_int_1[1].ob_type) | ((unsigned __int64)HIDWORD(pyx_int_1[1].ob_type) << 30))
                - 10,
                  v20,
                  ob_refcnt,
                  pyx_int_1);
          break;
        case 2ui64:
          v16 = PyLong_FromLongLong(
                  (LODWORD(pyx_int_1[1].ob_type) | ((unsigned __int64)HIDWORD(pyx_int_1[1].ob_type) << 30)) - 10,
                  v20,
                  ob_refcnt,
                  pyx_int_1);
          break;
        default:
          v16 = (*(__int64 (__fastcall **)(_object *, _object *))(*((_QWORD *)&PyLong_Type + 12) + 8i64))(
                  pyx_int_1,
                  _pyx_mstate_global->__pyx_int_10);
          break;
      }
    }
    else
    {
      v16 = PyLong_FromLong(LODWORD(pyx_int_1[1].ob_type) * (1 - (unsigned int)(v15 & 3)) - 10);
    }
  }
  else if ( p_ob_base == (_object *)PyFloat_Type )
  {
    v16 = PyFloat_FromDouble(__pyx_v_a, __pyx_self, ob_refcnt, pyx_int_1);
  }
  else
  {
    v16 = PyNumber_Subtract(pyx_int_1, _pyx_mstate_global->__pyx_int_10);
  }
  v21 = (_object *)v16;
  if ( !v16 )
  {
    v11 = _pyx_f[0];
    v12 = 43;
    v13 = 3949;
    goto LABEL_204;
  }
  v22 = (int *)v5;
  v5 = (_typeobject *)v16;
  if ( *v22 >= 0 )
  {
    v23 = (*(_QWORD *)v22)-- == 1i64;
    if ( v23 )
      _Py_Dealloc(v22, v17, v18, v19);
  }
  v24 = _Pyx_PyInt_MultiplyObjC(v21, _pyx_mstate_global->__pyx_int_998, 998i64, v19);
  v28 = v24;
  if ( !v24 )
  {
    v11 = _pyx_f[0];
    v12 = 44;
    v13 = 3961;
    goto LABEL_204;
  }
  v29 = (int *)v5;
  v5 = (_typeobject *)v24;
  if ( *v29 >= 0 )
  {
    v23 = (*(_QWORD *)v29)-- == 1i64;
    if ( v23 )
      _Py_Dealloc(v29, v25, v26, v27);
  }
  pyx_int_100 = _pyx_mstate_global->__pyx_int_100;
  v31 = &v28->ob_type->ob_base.ob_base;
  if ( v31 == PyLong_Type )
  {
    if ( v28[1].ob_refcnt >= 0x10ui64 )
      v32 = (*(__int64 (__fastcall **)(_object *, _object *))(*((_QWORD *)&PyLong_Type + 12) + 240i64))(
              v28,
              _pyx_mstate_global->__pyx_int_100);
    else
      v32 = PyFloat_FromDouble(v28[1].ob_refcnt & 3, PyLong_Type, pyx_int_100, v27);
  }
  else if ( v31 == (_object *)PyFloat_Type )
  {
    v32 = PyFloat_FromDouble(v29, PyLong_Type, pyx_int_100, v27);
  }
  else
  {
    v32 = PyNumber_TrueDivide(v28, _pyx_mstate_global->__pyx_int_100);
  }
  v36 = (_object *)v32;
  if ( !v32 )
  {
    v11 = _pyx_f[0];
    v12 = 45;
    v13 = 3973;
    goto LABEL_204;
  }
  v37 = (int *)v5;
  v5 = (_typeobject *)v32;
  if ( *v37 >= 0 )
  {
    v23 = (*(_QWORD *)v37)-- == 1i64;
    if ( v23 )
      _Py_Dealloc(v37, v33, v34, v35);
  }
  v38 = _Pyx_PyInt_MultiplyObjC(v36, _pyx_mstate_global->__pyx_int_200, 200i64, v35);
  v42 = (__int64)v38;
  if ( !v38 )
  {
    v11 = _pyx_f[0];
    v12 = 46;
    v13 = 3985;
    goto LABEL_204;
  }
  v43 = (int *)v5;
  v5 = (_typeobject *)v38;
  if ( *v43 >= 0 )
  {
    v23 = (*(_QWORD *)v43)-- == 1i64;
    if ( v23 )
      _Py_Dealloc(v43, v39, v40, v41);
  }
  v44 = PyLong_Type;
  pyx_int_30 = _pyx_mstate_global->__pyx_int_30;
  if ( *(_object *const *)(v42 + 8) == PyLong_Type )
  {
    v46 = *(_QWORD *)(v42 + 16);
    if ( (v46 & 1) != 0 )
    {
      if ( *(_DWORD *)v42 != -1 )
        ++*(_DWORD *)v42;
      goto LABEL_60;
    }
    if ( v46 >= 0x10 )
    {
      switch ( (*(_QWORD *)(v42 + 16) >> 3) * (1 - (*(_QWORD *)(v42 + 16) & 3ui64)) )
      {
        case 0xFFFFFFFFFFFFFFFEui64:
          v58 = -(__int64)(*(unsigned int *)(v42 + 24) | ((unsigned __int64)*(unsigned int *)(v42 + 28) << 30));
          goto LABEL_72;
        case 2ui64:
          v58 = *(unsigned int *)(v42 + 24) | ((unsigned __int64)*(unsigned int *)(v42 + 28) << 30);
LABEL_72:
          v59 = 0i64;
          v60 = v58 / 30;
          v61 = v58 % 30;
          if ( v61 )
            v59 = v61 >> 63;
          v57 = PyLong_FromLongLong(v60 - v59, v60 - v59, v61, pyx_int_30);
          break;
        default:
          v57 = (*(__int64 (__fastcall **)(__int64, _object *))(*((_QWORD *)&PyLong_Type + 12) + 232i64))(
                  v42,
                  _pyx_mstate_global->__pyx_int_30);
          break;
      }
    }
    else
    {
      v53 = *(_DWORD *)(v42 + 24) * (1 - (v46 & 3));
      v54 = 0;
      v55 = v53 / 30;
      v56 = v53 % 30;
      if ( v56 )
        v54 = v56 >> 31;
      v57 = PyLong_FromLong(v55 - v54);
    }
  }
  else
  {
    v57 = PyNumber_FloorDivide(v42, _pyx_mstate_global->__pyx_int_30);
  }
  v42 = v57;
  if ( !v57 )
  {
    v11 = _pyx_f[0];
    v12 = 47;
    v13 = 3997;
    goto LABEL_204;
  }
LABEL_60:
  v47 = (int *)v5;
  v5 = (_typeobject *)v42;
  if ( *v47 >= 0 )
  {
    v23 = (*(_QWORD *)v47)-- == 1i64;
    if ( v23 )
      _Py_Dealloc(v47, v46, v44, pyx_int_30);
  }
  v48 = PyNumber_Power(v42, _pyx_mstate_global->__pyx_int_3, _Py_NoneStruct, pyx_int_30);
  v52 = v48;
  if ( !v48 )
  {
    v11 = _pyx_f[0];
    v12 = 48;
    v13 = 4009;
    goto LABEL_204;
  }
  v62 = (int *)v5;
  v5 = (_typeobject *)v48;
  if ( *v62 >= 0 )
  {
    v23 = (*(_QWORD *)v62)-- == 1i64;
    if ( v23 )
      _Py_Dealloc(v62, v49, v50, v51);
  }
  v63 = PyLong_Type;
  pyx_int_3 = _pyx_mstate_global->__pyx_int_3;
  if ( *(_object *const *)(v52 + 8) != PyLong_Type )
  {
LABEL_102:
    v73 = PyNumber_Lshift(v52, _pyx_mstate_global->__pyx_int_3);
    goto LABEL_103;
  }
  v65 = *(_QWORD *)(v52 + 16);
  if ( (v65 & 1) != 0 )
  {
    if ( *(_DWORD *)v52 != -1 )
      ++*(_DWORD *)v52;
    goto LABEL_86;
  }
  if ( v65 < 0x10 )
  {
    LODWORD(v72) = *(_DWORD *)(v52 + 24) * (1 - (v65 & 3));
    if ( (_DWORD)v72 == (8 * (int)v72) >> 3 || !(_DWORD)v72 )
    {
      v73 = PyLong_FromLong((unsigned int)(8 * v72));
      goto LABEL_103;
    }
    v72 = (int)v72;
LABEL_99:
    if ( v72 == (__int64)(8 * v72) >> 3 )
    {
      v73 = PyLong_FromLongLong(8 * v72, 8 * v72, PyLong_Type, pyx_int_3);
      goto LABEL_103;
    }
    goto LABEL_102;
  }
  switch ( (*(_QWORD *)(v52 + 16) >> 3) * (1 - (*(_QWORD *)(v52 + 16) & 3ui64)) )
  {
    case 0xFFFFFFFFFFFFFFFEui64:
      v72 = -(__int64)(*(unsigned int *)(v52 + 24) | ((unsigned __int64)*(unsigned int *)(v52 + 28) << 30));
      goto LABEL_99;
    case 2ui64:
      v72 = *(unsigned int *)(v52 + 24) | ((unsigned __int64)*(unsigned int *)(v52 + 28) << 30);
      goto LABEL_99;
    default:
      v73 = (*(__int64 (__fastcall **)(__int64, _object *))(*((_QWORD *)&PyLong_Type + 12) + 88i64))(
              v52,
              _pyx_mstate_global->__pyx_int_3);
      break;
  }
LABEL_103:
  v52 = v73;
  if ( !v73 )
  {
    v11 = _pyx_f[0];
    v12 = 49;
    v13 = 4021;
    goto LABEL_204;
  }
LABEL_86:
  v66 = (int *)v5;
  v5 = (_typeobject *)v52;
  if ( *v66 >= 0 )
  {
    v23 = (*(_QWORD *)v66)-- == 1i64;
    if ( v23 )
      _Py_Dealloc(v66, v65, v63, pyx_int_3);
  }
  v67 = PyNumber_Xor(v52, __pyx_self, v63, pyx_int_3);
  v71 = (_object *)v67;
  if ( !v67 )
  {
    v11 = _pyx_f[0];
    v12 = 50;
    v13 = 4033;
    goto LABEL_204;
  }
  v74 = (int *)v5;
  v5 = (_typeobject *)v67;
  if ( *v74 >= 0 )
  {
    v23 = (*(_QWORD *)v74)-- == 1i64;
    if ( v23 )
      _Py_Dealloc(v74, v68, v69, v70);
  }
  args = v71;
  v75 = _Pyx_PyObject_FastCallDict(_pyx_builtin_hex, &args, 0x8000000000000001ui64, v70);
  v76 = v75;
  if ( !v75 )
  {
    v11 = _pyx_f[0];
    v12 = 51;
    v13 = 4045;
    goto LABEL_204;
  }
  v77 = v75->ob_type;
  tp_as_mapping = v77->tp_as_mapping;
  if ( !tp_as_mapping || !tp_as_mapping->mp_subscript )
  {
    PyErr_Format(PyExc_TypeError, "'%.200s' object is unsliceable", v77->tp_name);
    goto LABEL_200;
  }
  if ( _pyx_mstate_global == (__pyx_mstate *)-656i64 )
  {
    v83 = PyLong_FromSsize_t(2i64);
    v84 = (int *)v83;
    if ( !v83 )
      goto LABEL_200;
    v85 = (int *)PySlice_New(v83, _Py_NoneStruct, _Py_NoneStruct);
    if ( *v84 >= 0 )
    {
      v23 = (*(_QWORD *)v84)-- == 1i64;
      if ( v23 )
        _Py_Dealloc(v84, v79, v81, v82);
    }
    if ( !v85 )
      goto LABEL_200;
    v80 = (_typeobject *)tp_as_mapping->mp_subscript(v76, (_object *)v85);
    if ( *v85 >= 0 )
    {
      v23 = (*(_QWORD *)v85)-- == 1i64;
      if ( v23 )
        _Py_Dealloc(v85, v79, v81, v82);
    }
  }
  else
  {
    v80 = (_typeobject *)tp_as_mapping->mp_subscript(v75, _pyx_mstate_global->__pyx_slice__4);
  }
  if ( !v80 )
  {
LABEL_200:
    v12 = 51;
    v13 = 4047;
    goto $__pyx_L1_error_2;
  }
  if ( (v76->ob_refcnt_split[0] & 0x80000000) == 0 )
  {
    v23 = v76->ob_refcnt-- == 1;
    if ( v23 )
      _Py_Dealloc(v76, v79, v81, v82);
  }
  v86 = (int *)v5;
  v5 = v80;
  if ( *v86 >= 0 )
  {
    v23 = (*(_QWORD *)v86)-- == 1i64;
    if ( v23 )
      _Py_Dealloc(v86, v79, v81, v82);
  }
  v87 = PyNumber_Add(v80, _pyx_mstate_global->__pyx_kp_s_00);
  if ( !v87 )
  {
    v11 = _pyx_f[0];
    v12 = 52;
    v13 = 4060;
    goto LABEL_204;
  }
  v5 = (_typeobject *)v87;
  if ( (v80->ob_base.ob_base.ob_refcnt_split[0] & 0x80000000) == 0 )
  {
    v23 = v80->ob_base.ob_base.ob_refcnt-- == 1;
    if ( v23 )
      _Py_Dealloc(v80, v88, v89, v90);
  }
  v76 = (_object *)PyTuple_New(2i64);
  if ( !v76 )
  {
    v11 = _pyx_f[0];
    v12 = 53;
    v13 = 4072;
    goto LABEL_204;
  }
  if ( v5->ob_base.ob_base.ob_refcnt_split[0] != -1 )
    ++v5->ob_base.ob_base.ob_refcnt_split[0];
  v92 = _pyx_mstate_global;
  v76[1].ob_type = v5;
  pyx_int_16 = v92->__pyx_int_16;
  if ( pyx_int_16->ob_refcnt_split[0] != -1 )
    ++pyx_int_16->ob_refcnt_split[0];
  v76[2].ob_refcnt = (__int64)v92->__pyx_int_16;
  v94 = _Pyx_PyObject_Call(PyLong_Type, v76, v91);
  if ( !v94 )
  {
    v12 = 53;
    v13 = 4080;
$__pyx_L1_error_2:
    v11 = _pyx_f[0];
    if ( (v76->ob_refcnt_split[0] & 0x80000000) == 0 )
    {
      v23 = v76->ob_refcnt-- == 1;
      if ( v23 )
        _Py_Dealloc(v76, v79, v81, v82);
    }
LABEL_204:
    v113 = v5;
    _Pyx_AddTraceback("bbbbb.test_calc", v13, v12, v11);
    if ( !v5 )
      return v3;
    goto LABEL_205;
  }
  if ( (v76->ob_refcnt_split[0] & 0x80000000) == 0 )
  {
    v23 = v76->ob_refcnt-- == 1;
    if ( v23 )
      _Py_Dealloc(v76, v79, v81, v82);
  }
  v95 = (int *)v5;
  v5 = (_typeobject *)v94;
  if ( *v95 >= 0 )
  {
    v23 = (*(_QWORD *)v95)-- == 1i64;
    if ( v23 )
      _Py_Dealloc(v95, v79, v81, v82);
  }
  pyx_int_4 = _pyx_mstate_global->__pyx_int_4;
  if ( (_object *const)v94->ob_type == PyLong_Type )
  {
    v97 = v94[1].ob_refcnt;
    if ( (v97 & 1) != 0 )
    {
      if ( v94->ob_refcnt_split[0] != -1 )
        ++v94->ob_refcnt_split[0];
      goto LABEL_152;
    }
    if ( v97 >= 0x10 )
    {
      v102 = v97 >> 3;
      switch ( v102 * (1 - (v94[1].ob_refcnt & 3)) )
      {
        case 0xFFFFFFFFFFFFFFFEui64:
          v101 = PyLong_FromLongLong(
                   -(__int64)(LODWORD(v94[1].ob_type) | ((unsigned __int64)HIDWORD(v94[1].ob_type) << 30)) >> 4,
                   v102,
                   v81,
                   pyx_int_4);
          break;
        case 2ui64:
          v101 = PyLong_FromLongLong(
                   (__int64)(LODWORD(v94[1].ob_type) | ((unsigned __int64)HIDWORD(v94[1].ob_type) << 30)) >> 4,
                   v102,
                   v81,
                   pyx_int_4);
          break;
        default:
          v101 = (*(__int64 (__fastcall **)(_object *, _object *))(*((_QWORD *)&PyLong_Type + 12) + 96i64))(
                   v94,
                   _pyx_mstate_global->__pyx_int_4);
          break;
      }
    }
    else
    {
      v101 = PyLong_FromLong((unsigned int)((int)(LODWORD(v94[1].ob_type) * (1 - (v97 & 3))) >> 4));
    }
  }
  else
  {
    v101 = PyNumber_Rshift(v94, _pyx_mstate_global->__pyx_int_4);
  }
  v94 = (_object *)v101;
  if ( !v101 )
  {
    v11 = _pyx_f[0];
    v12 = 54;
    v13 = 4093;
    goto LABEL_204;
  }
LABEL_152:
  v98 = (int *)v5;
  v5 = (_typeobject *)v94;
  if ( *v98 >= 0 )
  {
    v23 = (*(_QWORD *)v98)-- == 1i64;
    if ( v23 )
      _Py_Dealloc(v98, v97, v81, pyx_int_4);
  }
  pyx_int_15 = _pyx_mstate_global->__pyx_int_15;
  if ( (_object *const)v94->ob_type != PyLong_Type )
  {
    v103 = PyNumber_Or(v94, _pyx_mstate_global->__pyx_int_15, v81, pyx_int_4);
LABEL_175:
    pyx_int_15 = (_object *)v103;
    goto LABEL_176;
  }
  v100 = v94[1].ob_refcnt;
  if ( (v100 & 1) == 0 )
  {
    if ( v100 >= 0x10 )
    {
      switch ( (v100 >> 3) * (1 - (v94[1].ob_refcnt & 3)) )
      {
        case 0xFFFFFFFFFFFFFFFEui64:
          v103 = PyLong_FromLongLong(
                   -(__int64)(LODWORD(v94[1].ob_type) | ((unsigned __int64)HIDWORD(v94[1].ob_type) << 30)) | 0xF,
                   v97,
                   v81,
                   pyx_int_4);
          break;
        case 2ui64:
          v103 = PyLong_FromLongLong(
                   LODWORD(v94[1].ob_type) | ((unsigned __int64)HIDWORD(v94[1].ob_type) << 30) | 0xF,
                   v97,
                   v81,
                   pyx_int_4);
          break;
        default:
          v103 = (*(__int64 (__fastcall **)(_object *, _object *, _QWORD, _object *))(*((_QWORD *)&PyLong_Type + 12)
                                                                                    + 120i64))(
                   v94,
                   _pyx_mstate_global->__pyx_int_15,
                   *((_QWORD *)&PyLong_Type + 12),
                   pyx_int_4);
          break;
      }
    }
    else
    {
      v103 = PyLong_FromLong((LODWORD(v94[1].ob_type) * (1 - (v100 & 3))) | 0xF);
    }
    goto LABEL_175;
  }
  if ( pyx_int_15->ob_refcnt_split[0] != -1 )
    ++pyx_int_15->ob_refcnt_split[0];
LABEL_176:
  if ( !pyx_int_15 )
  {
    v11 = _pyx_f[0];
    v12 = 55;
    v13 = 4105;
    goto LABEL_204;
  }
  v5 = (_typeobject *)pyx_int_15;
  if ( (v94->ob_refcnt_split[0] & 0x80000000) == 0 )
  {
    v23 = v94->ob_refcnt-- == 1;
    if ( v23 )
      _Py_Dealloc(v94, v97, v81, pyx_int_4);
  }
  if ( (_object *const)pyx_int_15->ob_type == PyLong_Type )
  {
    v104 = 0x40000000 - LODWORD(pyx_int_15[1].ob_type);
    if ( (pyx_int_15[1].ob_refcnt & 3) == 0 )
      v104 = (int)pyx_int_15[1].ob_type;
    v105 = PyLong_FromLong(v104 & 0x1F);
  }
  else
  {
    v105 = PyNumber_And(pyx_int_15, _pyx_mstate_global->__pyx_int_31, v81, pyx_int_4);
  }
  v109 = (int *)v105;
  if ( !v105 )
  {
    v11 = _pyx_f[0];
    v12 = 56;
    v13 = 4117;
    goto LABEL_204;
  }
  v5 = (_typeobject *)v105;
  if ( (pyx_int_15->ob_refcnt_split[0] & 0x80000000) == 0 )
  {
    v23 = pyx_int_15->ob_refcnt-- == 1;
    if ( v23 )
      _Py_Dealloc(pyx_int_15, v106, v107, v108);
  }
  v113 = (_typeobject *)PyNumber_Invert(v109);
  if ( !v113 )
  {
    v11 = _pyx_f[0];
    v12 = 57;
    v13 = 4129;
    goto LABEL_204;
  }
  if ( *v109 >= 0 )
  {
    v23 = (*(_QWORD *)v109)-- == 1i64;
    if ( v23 )
      _Py_Dealloc(v109, v110, v111, v112);
  }
  if ( v113->ob_base.ob_base.ob_refcnt_split[0] != -1 )
    ++v113->ob_base.ob_base.ob_refcnt_split[0];
  v3 = v113;
LABEL_205:
  if ( (v113->ob_base.ob_base.ob_refcnt_split[0] & 0x80000000) == 0 )
  {
    v23 = v113->ob_base.ob_base.ob_refcnt-- == 1;
    if ( v23 )
      _Py_Dealloc(v113, v110, v111, v112);
  }
  return v3;
}
```

ida 打开，找到相关函数

> 说明，紫色的变量 / 函数名为从别处导入的内容，在抹除符号情况下也能被识别出来

加法 +1 PyNumber_Add(__pyx_self, _pyx_mstate_global->__pyx_int_1); 如果是原地的话， 即 al += 1; 则可能会调用到 PyNumber_InPlaceAdd

![](https://bbs.kanxue.com/upload/attach/202502/880848_2KWN3VJRTZZRGVG.png)

-10 PyNumber_Subtract(pyx_int_1, _pyx_mstate_global->__pyx_int_10);

![](https://bbs.kanxue.com/upload/attach/202502/880848_ZS4GX6JASFWRY9F.png)

乘以 998 _Pyx_PyInt_MultiplyObjC(v21, _pyx_mstate_global->__pyx_int_998, 998i64, v19);

![](https://bbs.kanxue.com/upload/attach/202502/880848_A3T44G56NQHAWCE.png)

这里的乘法是经过了一层封装的，无符号情况下需要进入到里面才能看到实际的从导入表中调用的函数 PyNumber_Multiply(op1, op2, intval); 这里其他的运算函数也可能会这么封装，注意用此方法识别。

![](https://bbs.kanxue.com/upload/attach/202502/880848_H5AXF9UQJY4DUB6.png)

除以 100 不取整 真除 PyNumber_TrueDivide(v28, _pyx_mstate_global->__pyx_int_100);

![](https://bbs.kanxue.com/upload/attach/202502/880848_QAN3P37Y8NSWEK7.png)

乘以 200 _Pyx_PyInt_MultiplyObjC(v36, _pyx_mstate_global->__pyx_int_200, 200i64, v35);

![](https://bbs.kanxue.com/upload/attach/202502/880848_DVV8HNU48YBV7FB.png)

地板除 30 即 整除 30 PyNumber_FloorDivide(v42, _pyx_mstate_global->__pyx_int_30);

![](https://bbs.kanxue.com/upload/attach/202502/880848_US4QV76SPP97K8R.png)

3 次方 PyNumber_Power(v42, _pyx_mstate_global->__pyx_int_3, _Py_NoneStruct, pyx_int_30);

![](https://bbs.kanxue.com/upload/attach/202502/880848_WM3RDEBXN8WSY4W.png)

左移 3 PyNumber_Lshift(v52, _pyx_mstate_global->__pyx_int_3);

![](https://bbs.kanxue.com/upload/attach/202502/880848_3N422YX83H6CNW6.png)

xor PyNumber_Xor(v52, __pyx_self, v63, pyx_int_3);

![](https://bbs.kanxue.com/upload/attach/202502/880848_B9KSVNQS8UTERXE.png)

FastCall 调用 内建函数 hex _Pyx_PyObject_FastCallDict(_pyx_builtin_hex, &args, 0x8000000000000001ui64, v70);

![](https://bbs.kanxue.com/upload/attach/202502/880848_CCWPFE9QUBEY6FG.png)

FastCall 不是导入函数，进入到该函数内部可以看到调用了 PyObject_VectorcallDict ，可以依此为特征

![](https://bbs.kanxue.com/upload/attach/202502/880848_A7QBYKSXZ2TBTR8.png)

切片操作 从 2 开始 PySlice_New(v83, _Py_NoneStruct, _Py_NoneStruct);

![](https://bbs.kanxue.com/upload/attach/202502/880848_RNGZJZTAWZHAD4N.png)

字符串加法，加 "00" PyNumber_Add(v80, _pyx_mstate_global->__pyx_kp_s_00);

![](https://bbs.kanxue.com/upload/attach/202502/880848_559BFSQETRS9EB7.png)

强转类型, _Pyx_PyObject_Call(PyLong_Type, v76, v91); 第一个参数为类型 PyLong_Type 整型 int。

![](https://bbs.kanxue.com/upload/attach/202502/880848_86V45QK3PUANKPJ.png)

类型强转函数在无符号情况下，没有符号，需要进入到里面识别一下, 特征也是非常明显，传递的第一个参数是类型名，是一个紫色的被导入的函数 / 变量，函数内部也会有一些字符串来辅助识别，最后也是调用 PyObject_Call

![](https://bbs.kanxue.com/upload/attach/202502/880848_77TT399GX8BWFFU.png)

右移 4 PyNumber_Rshift(v94, _pyx_mstate_global->__pyx_int_4);

![](https://bbs.kanxue.com/upload/attach/202502/880848_HB3HYFQVHDP59F6.png)

或操作 PyNumber_Or(v94, _pyx_mstate_global->__pyx_int_15, v81, pyx_int_4);

![](https://bbs.kanxue.com/upload/attach/202502/880848_PGKMUXYVV79JUDU.png)

与操作， 与 31 PyNumber_And(pyx_int_15, _pyx_mstate_global->__pyx_int_31, v81, pyx_int_4);

![](https://bbs.kanxue.com/upload/attach/202502/880848_XTHTQMUPGYTZDBP.png)

取反 (_typeobject *)PyNumber_Invert(v109);

![](https://bbs.kanxue.com/upload/attach/202502/880848_KADZBN29Y3FX3CU.png)

技巧: 有些会运算先执行一遍，做一些判断？最终的运算还是调用导入的函数。所以需要仔细观察导入的函数和被调用的函数。

![](https://bbs.kanxue.com/upload/attach/202502/880848_KNJD49UAKWZUTAZ.png)

其中还会穿插一些引用计数的内容。

在无符号情况下，_pyx_mstate_global 结构体是不会识别出来的，需要手动从数据初始化部分来识别。

### 幂运算 pow

rsa 相关

```
def rsa_enc(e,n,c):
    a = pow(c, 0x10001, 10403)
    b = c ** 0x10001, 10403
    d = pow(c, e, n)
    return a,b,d
```

```
_object *__fastcall _pyx_pf_5bbbbb_8rsa_enc(
        _object *__pyx_v_e,
        _object *__pyx_v_n,
        _object *__pyx_v_c,
        _object *__pyx_self)
{
  _QWORD *v4; // rbp
  _DWORD *v5; // rdi
  int *v6; // rsi
  _DWORD *v9; // r14
  __int64 v10; // rax
  int v11; // r15d
  int v12; // r13d
  __int64 v13; // rbx
  unsigned __int64 v14; // rcx
  int *v15; // r15
  bool v16; // zf
  __int64 v17; // rax
  unsigned int v18; // r8d
  unsigned int v19; // eax
  __int64 v20; // rax
  signed __int64 v21; // r8
  unsigned __int64 v22; // r8
  unsigned __int64 v23; // rax
  const char *v24; // r12
  _QWORD *v25; // rax
 
  v4 = 0i64;
  v5 = 0i64;
  v6 = 0i64;
  v9 = 0i64;
  v10 = PyNumber_Power(__pyx_self, _pyx_mstate_global->__pyx_int_65537, _pyx_mstate_global->__pyx_int_10403);
  if ( !v10 )
  {
    v11 = 33;
    v12 = 3744;
LABEL_33:
    v24 = _pyx_f[0];
    goto LABEL_34;
  }
  v9 = (_DWORD *)v10;
  v13 = PyNumber_Power(__pyx_self, _pyx_mstate_global->__pyx_int_65537, _Py_NoneStruct);
  if ( !v13 )
  {
    v11 = 34;
    v12 = 3756;
    goto LABEL_33;
  }
  if ( *(_object *const *)(v13 + 8) == PyLong_Type )
  {
    v14 = *(_QWORD *)(v13 + 16);
    if ( (v14 & 1) != 0 )
    {
      if ( *(_DWORD *)v13 != -1 )
        ++*(_DWORD *)v13;
      v15 = (int *)v13;
      goto LABEL_10;
    }
    if ( v14 >= 0x10 )
    {
      switch ( (v14 >> 3) * (1 - (*(_QWORD *)(v13 + 16) & 3i64)) )
      {
        case 0xFFFFFFFFFFFFFFFEui64:
          v21 = -(__int64)(*(unsigned int *)(v13 + 24) | ((unsigned __int64)*(unsigned int *)(v13 + 28) << 30));
          goto LABEL_22;
        case 2ui64:
          v21 = *(unsigned int *)(v13 + 24) | ((unsigned __int64)*(unsigned int *)(v13 + 28) << 30);
LABEL_22:
          v22 = v21 % 10455;
          v23 = 0i64;
          if ( v22 )
            v23 = v22 >> 63;
          v20 = PyLong_FromLongLong(v22 + 10455 * v23);
          break;
        default:
          v20 = (*(__int64 (__fastcall **)(__int64, _object *))(*((_QWORD *)&PyLong_Type + 12) + 24i64))(
                  v13,
                  _pyx_mstate_global->__pyx_int_10455);
          break;
      }
    }
    else
    {
      v18 = (int)(*(_DWORD *)(v13 + 24) * (1 - (v14 & 3))) % 10455;
      v19 = 0;
      if ( v18 )
        v19 = v18 >> 31;
      v20 = PyLong_FromLong(v18 + 10455 * v19);
    }
  }
  else
  {
    v20 = PyNumber_Remainder(v13, _pyx_mstate_global->__pyx_int_10455);
  }
  v15 = (int *)v20;
  if ( !v20 )
  {
    v24 = _pyx_f[0];
    v11 = 34;
    v12 = 3758;
    if ( *(int *)v13 >= 0 )
    {
      v16 = (*(_QWORD *)v13)-- == 1i64;
      if ( v16 )
        _Py_Dealloc(v13);
    }
LABEL_34:
    _Pyx_AddTraceback("bbbbb.rsa_enc", v12, v11, v24);
    if ( !v9 )
      goto LABEL_46;
    goto LABEL_43;
  }
LABEL_10:
  if ( *(int *)v13 >= 0 )
  {
    v16 = (*(_QWORD *)v13)-- == 1i64;
    if ( v16 )
      _Py_Dealloc(v13);
  }
  v6 = v15;
  v17 = PyNumber_Power(__pyx_self, __pyx_v_n, __pyx_v_c);
  if ( !v17 )
  {
    v11 = 35;
    v12 = 3771;
    goto LABEL_33;
  }
  v5 = (_DWORD *)v17;
  v25 = (_QWORD *)PyTuple_New(3i64);
  if ( !v25 )
  {
    v11 = 36;
    v12 = 3784;
    goto LABEL_33;
  }
  if ( *v9 != -1 )
    ++*v9;
  v25[3] = v9;
  if ( *v15 != -1 )
    ++*v15;
  v25[4] = v15;
  if ( *v5 != -1 )
    ++*v5;
  v25[5] = v5;
  v4 = v25;
LABEL_43:
  if ( (int)*v9 >= 0 )
  {
    v16 = (*(_QWORD *)v9)-- == 1i64;
    if ( v16 )
      _Py_Dealloc(v9);
  }
LABEL_46:
  if ( v6 )
  {
    if ( *v6 >= 0 )
    {
      v16 = (*(_QWORD *)v6)-- == 1i64;
      if ( v16 )
        _Py_Dealloc(v6);
    }
  }
  if ( v5 )
  {
    if ( (int)*v5 >= 0 )
    {
      v16 = (*(_QWORD *)v5)-- == 1i64;
      if ( v16 )
        _Py_Dealloc(v5);
    }
  }
  return (_object *)v4;
}
```

PyNumber_Power(__pyx_self, _pyx_mstate_global->__pyx_int_65537, _pyx_mstate_global->__pyx_int_10403, __pyx_self); pow 参数 第一个原始数，第二个 e 次方，第三个 mod 数

![](https://bbs.kanxue.com/upload/attach/202502/880848_SVQQY88JAKNPZ9S.png)

第二个运算形式是先有一个 pow 取次幂，再有一个取模

v13 = PyNumber_Power(__pyx_self, _pyx_mstate_global->__pyx_int_65537, _Py_NoneStruct);

PyNumber_Remainder(v13, _pyx_mstate_global->__pyx_int_10455);

![](https://bbs.kanxue.com/upload/attach/202502/880848_Y3PPBJAU4BFBK8G.png)

![](https://bbs.kanxue.com/upload/attach/202502/880848_8RERARX64NEW6D5.png)

第三个就是取参数

![](https://bbs.kanxue.com/upload/attach/202502/880848_AHC3896MCNAMU3W.png)

数据类型及强转
-------

### 元组 列表

取上面函数的例子:

```
def rsa_enc(e,n,c):
    a = pow(c, 0x10001, 10403)
    b = c ** 0x10001, 10403
    d = pow(c, e, n)
    return a,b,d
```

部分汇编代码，三个变量的由来，最后的组成

```
  v9 = 0i64;
  v10 = PyNumber_Power(__pyx_self, _pyx_mstate_global->__pyx_int_65537, _pyx_mstate_global->__pyx_int_10403);
  if ( !v10 )
  {
    v11 = 33;
    v12 = 3744;
LABEL_33:
    v24 = _pyx_f[0];
    goto LABEL_34;
  }
  v9 = (_DWORD *)v10;
 
  .......
  ........
   
    v20 = PyNumber_Remainder(v13, _pyx_mstate_global->__pyx_int_10455);
  }
  v15 = (int *)v20;
   
  .......
  ........
  v5 = (_DWORD *)v17;
  v25 = (_QWORD *)PyTuple_New(3i64);
  if ( !v25 )
  {
    v11 = 36;
    v12 = 3784;
    goto LABEL_33;
  }
  if ( *v9 != -1 )
    ++*v9;
  v25[3] = v9;
  if ( *v15 != -1 )
    ++*v15;
  v25[4] = v15;
  if ( *v5 != -1 )
    ++*v5;
  v25[5] = v5;
  v4 = v25;
```

可以看到，元组的第三个 qword 才是真实的数据存储， 初始化时会调用 PyTuple_New 函数

列表初始化, 基本上也差不多，很有规律

```
def get_list():
    return [1,2,3,4,991,776,229,"aaa","str", 12.22]
```

![](https://bbs.kanxue.com/upload/attach/202502/880848_GXAGS3BEGM3VECZ.png)

切片，上面的运算里面稍微提到了一点

```
def get_list():
    return [1,2,3,4,991,776,229,"aaa","str", 12.22]
 
def get_list1(index,end):
    a= get_list()
    return  a[index:end]
```

PySlice_New(从哪开始，到哪结束, 步进)

_Py_NoneStruct 参数表示留空

例如 l[2:] PySlice_New(2, _Py_NoneStruct,_Py_NoneStruct)

![](https://bbs.kanxue.com/upload/attach/202502/880848_7YT4H7REXW2EQC5.png)

![](https://bbs.kanxue.com/upload/attach/202502/880848_CATBDCEZGWSD73W.png)

### 集合 / 字典

```
def test_set_dict():
    d = {1:'12',2:'4'}
    s = set([1,8,4,6])
 
    res = d.items(),d.values()
    return res, d,s[1:8]
```

字典初始化, 使用 PyDict_SetItem 函数， 第一个参数为字典的地址，第二个为 key, 第三个为值

![](https://bbs.kanxue.com/upload/attach/202502/880848_NP62D9E4UDBKSB6.png)

集合初始化， 使用 PySet_Add 新增

![](https://bbs.kanxue.com/upload/attach/202502/880848_W693W5QZYMYAAEG.png)

字典的函数

items

![](https://bbs.kanxue.com/upload/attach/202502/880848_NJE3HGCR3W7CYHP.png)

values

![](https://bbs.kanxue.com/upload/attach/202502/880848_C6VYN8J4QRFXSAV.png)

在生成的源码里发现，有两种调用形式，大版本大于 3 和不大于 3

![](https://bbs.kanxue.com/upload/attach/202502/880848_9MHEP3MHJ7KXYV7.png)

![](https://bbs.kanxue.com/upload/attach/202502/880848_T84TCB5BW5D6A9A.png)

这个无符号的话没办法，只能靠硬猜，通过交叉引用，可以获得变量类型

无符号的调用展示:

![](https://bbs.kanxue.com/upload/attach/202502/880848_QZV6FKFWF92QBEX.png)

向上寻找 qword_180012208 的交叉引用

![](https://bbs.kanxue.com/upload/attach/202502/880848_XCWPBP6D7J3V3T6.png)

发现 qword_180012208 被写入的内容为 PyDict_Type

![](https://bbs.kanxue.com/upload/attach/202502/880848_9NQ8UGUXUTTJX78.png)

循环选择结构
------

循环结构，一般，就是因为代码过长，直接 f5 可能会忽略掉 循环结构，可以先看流程图，观察大致结构，再对功能做简单分析

![](https://bbs.kanxue.com/upload/attach/202502/880848_CHHXVBP8EEH2XSF.png)

```
def get_list():
    return [1,2,3,4,991,776,229, 12.22]
 
def get_list1(index,end):
    a= get_list()
 
    sum = 0
    for i in a:
        sum += i
 
    return sum
```

直接遍历列表使用 PyObject_GetIter(obj);

![](https://bbs.kanxue.com/upload/attach/202502/880848_3XJJZGSM4Q3Y9DD.png)

### range

```
def get_list():
    return [1,2,3,4,991,776,229, 12.22]
 
def get_list1(index,end):
    a= get_list()
 
 
    sum1 = 0
    for i in range(len(a)):
        sum1 *= a[i]
 
    return sum1
```

range 是一个内建函数，以这个 FastCall 的方式来进行调用， len 则使用 PyLong_FromSsize_t 来进行获取，当然，不一样的 len 可能会有不一样的方式。

![](https://bbs.kanxue.com/upload/attach/202502/880848_B9NM7YG9K4FHQZT.png)

文件操作
----

```
def test_file_(filename):
    f = open(filename)
    content1 = f.read()
    f.close()
 
 
    return content1
```

基本没区别，调用内置 open， 获取 属性 read 然后 Fastcall 执行

![](https://bbs.kanxue.com/upload/attach/202502/880848_PPSRBDVE3YHEKNP.png)

数据初始化
=====

pyd 的初始化主要分为三个部分，分别为 常量 / 整数的初始化 Pyx_InitConstants ， 字符串的初始化 Pyx_CreateStringTabAndInitStrings ，其他的初始化 pyx_pymod_exec

其中 Pyx_InitConstants 的开头包含了字符串的初始化 Pyx_CreateStringTabAndInitStrings（如果没有代码比较简单，没有常量，则不会有 Pyx_InitConstants 生成）

![](https://bbs.kanxue.com/upload/attach/202502/880848_PVXNUQ8B4UECFCA.png)

pyx_mod_exec 也有可能会调用到 Pyx_CreateStringTabAndInitStrings 函数，但不在开头

并且他们会围绕着一个变量走: pyx_mstate_global

注意， pyx_mstate_global 每一个变量的长度都为一个 QWORD，如果被 ida 识别成了 void* 则需要手动更改。

每一个用户自写函数，都会在前几行代码中生成引用 pyx_mstate_global 的代码

![](https://bbs.kanxue.com/upload/attach/202502/880848_3UWMAZ4UMDMAQ8K.png)

并且调用最多的也就是 用户的自写函数 / 代码

![](https://bbs.kanxue.com/upload/attach/202502/880848_7JSUX9X8K2WJ9FJ.png)

我们观察生成的源码中 pymod_exec 的函数代码

会将我们定义的函数 设置为 __pyx_d 字典的键值对，就是说会调用用户定义的函数的首地址。

![](https://bbs.kanxue.com/upload/attach/202502/880848_83HX5D8ZEJ3BZXF.png)

__Pyx_InitConstants __Pyx_InitGlobals 初始化常量，初始化全局变量

![](https://bbs.kanxue.com/upload/attach/202502/880848_KEYMR8MBV96F6KD.png)

__Pyx_CreateStringTabAndInitStrings 被 __Pyx_InitConstants 调用

![](https://bbs.kanxue.com/upload/attach/202502/880848_V29H9K4USFNVM4C.png)

当然还有一种情况，pyd 模块不定义任何函数，那么就不会有函数导出，代码会在 pymod_exec 的后面执行，望周知。

无符号技巧
=====

首先我们根据 pyd 文件生成的特点，先确定 pyx_pymod_exec ，常量初始化函数 __Pyx_InitConstants 和 字符串初始化函数 __Pyx_CreateStringTabAndInitStrings

pyx_pymod_exec 确定的方法:

1.  函数中有一些字符串可助力快速定位
    
2.  用户定义的函数结构体会被 pymod_exec 引用一次
    

![](https://bbs.kanxue.com/upload/attach/202502/880848_ZUXSAKTWN5N6U7G.png)

![](https://bbs.kanxue.com/upload/attach/202502/880848_CGMW5D86BBE98M5.png)

__Pyx_InitConstants 确定方法:

PyLong_FromLong 会被多次调用，在有很多常量的情况下

![](https://bbs.kanxue.com/upload/attach/202502/880848_WXR25QNXXEY3247.png)

__Pyx_CreateStringTabAndInitStrings 确定方法:

字符串会被引用, 比如说 **main** 之类的一些 内容，还有一些代码残片。

![](https://bbs.kanxue.com/upload/attach/202502/880848_GSBK7SS4YMMD32N.png)

![](https://bbs.kanxue.com/upload/attach/202502/880848_29W5VJP2W9299H8.png)

根据这些函数解析 _pyx_mstate_global 的结构

然后再开始搜索函数名字符串，寻找函数的真正地址， 配合 pyx_mstate_global 来对代码进行解释

根据上面的技巧，识别原始的 python 代码，来完成逆向。

动态调试
====

参考前辈: [https://bbs.kanxue.com/thread-285349.htm](https://bbs.kanxue.com/thread-285349.htm) 提供的方法

调用前，先使用 input 暂停 python 进程，然后下断点附加 python 进程，开始调试。

linux 和 windows 上的方法基本一致，但要注意 ida 和实际加载的模块名称必须一致，否则不能顺利断下。

样题演示
====

18 ciscn&ccb 线上 rand0m
----------------------

ida 打开查看字符串，发现有两个 rand0m 和 check 猜测为两个函数

![](https://bbs.kanxue.com/upload/attach/202502/880848_A2A9K2PPS9442RH.png)

查找真实的函数地址:

![](https://bbs.kanxue.com/upload/attach/202502/880848_CJGVAUED243G7RQ.png)

![](https://bbs.kanxue.com/upload/attach/202502/880848_ZPAYFXRSWP3K2M7.png)

观察函数 sub_1800017E0 确定 off_18000B688 为 __pyx_mstate

![](https://bbs.kanxue.com/upload/attach/202502/880848_FMN9W2F5EA6AZ7Q.png)

重命名，check2 为真实函数

![](https://bbs.kanxue.com/upload/attach/202502/880848_YCZB728ABZHDD5A.png)

先确定 pymod_exec 和一些初始化函数位置，上文分析时，我们也可以通过函数结构体来确定 pymod_exec 函数，因为 pymod_exec 会初始化这些函数

![](https://bbs.kanxue.com/upload/attach/202502/880848_6B2M4ZXF8BPYT2J.png)

PyLong_FromLong 函数调用最多的就是 init_constra （当然不完全准确，有时候并不存在这个函数）

![](https://bbs.kanxue.com/upload/attach/202502/880848_FDQPJHDN2KQ3FV7.png)

![](https://bbs.kanxue.com/upload/attach/202502/880848_WASUN955U9R4XJE.png)

将这些函数重命名，我们就能拿到大部分的常量了。

```
_pyx_mstate
  
  
pymod_exec...:
19: random.rand0m
 
init_con...:
     
29: 0
30: 1
31: 2
32: 4
33: 5
34: 8
35: 0xb
36: 0x10
37: 0x17
38: 0x10001
39: 0x23A1268
40: 0x12287F38
41: 1244723021
42: 2282784775
43: 2563918650
44: 2654435769
45: 2918417411
46: 3628702646
47: 3773946743
48: 4198170623
49: 4294967293
```

pymod_exec 的 19, rand0m

![](https://bbs.kanxue.com/upload/attach/202502/880848_DQBY667M3KNKYCE.png)

![](https://bbs.kanxue.com/upload/attach/202502/880848_69H2SWTCZZ4XCQP.png)

可以 f5 还原的更快一点

![](https://bbs.kanxue.com/upload/attach/202502/880848_E8KXHMPZ26R82EM.png)

再分析 check 函数 创建 8 长度的列表

![](https://bbs.kanxue.com/upload/attach/202502/880848_SNWTSDRGWDH9MCH.png)

分别用到了_pyx_mstate 的 [40,43,41,47,39,45,42,46]

向下走，看到了一个 乘以 8 , 观察 mstate 34 也是 8

![](https://bbs.kanxue.com/upload/attach/202502/880848_NE54U5VB9V6HKNE.png)

然后有一个 +1

![](https://bbs.kanxue.com/upload/attach/202502/880848_N8985PTQCR552ZK.png)

乘 8

![](https://bbs.kanxue.com/upload/attach/202502/880848_J9APGJWWE845U7T.png)

分割 段为: (?*8:(?+1)*8)

![](https://bbs.kanxue.com/upload/attach/202502/880848_5JF5J2TK7ZX3S45.png)

然后获取了 _pyx_mstate 第 19 个的变量，确定是 rand0m 函数

![](https://bbs.kanxue.com/upload/attach/202502/880848_CEBZQWN8KKYFN6H.png)

向下走，调用了 sub_180004660，点进去发现存在 PyObject_VectorcallDict， 确定为 FastCall

![](https://bbs.kanxue.com/upload/attach/202502/880848_6R92HY6UWSRA553.png)

结合上下文判断 sub_180004790(v52, 1i64, v53, 0); 为 获取元组中的某个元素，下面还有一个调用，参数为 0

![](https://bbs.kanxue.com/upload/attach/202502/880848_B38VJQ4PG6NWT3T.png)

乘以 2

![](https://bbs.kanxue.com/upload/attach/202502/880848_BUGCPWAB7VTMG9B.png)

根据索引获取值，并比较 v34 来自 sub_180004790 的值，它的参数是调用了模块内函数返回的值

![](https://bbs.kanxue.com/upload/attach/202502/880848_XC4828ZFSPWPSZJ.png)

判断是否为真

![](https://bbs.kanxue.com/upload/attach/202502/880848_GCN83U3RDMWNWQM.png)

再调用 v39 = sub_180004790(v55, 0i64, v63, 0);

![](https://bbs.kanxue.com/upload/attach/202502/880848_CU57USATS8AY9EN.png)

乘以 2

![](https://bbs.kanxue.com/upload/attach/202502/880848_JGXBPDQRZ53F4SN.png)

sub_1800044A0 分析出来是加法， +1

![](https://bbs.kanxue.com/upload/attach/202502/880848_X39QUXR4RN39WMD.png)

sub_1800044A0 内部为 add

![](https://bbs.kanxue.com/upload/attach/202502/880848_G7Z878KWQ5VJC5S.png)

getValueWithIndex 传入 v82 和 v34, v34 是上面计算出来的值， v82 是一开始的那个 List 列表

![](https://bbs.kanxue.com/upload/attach/202502/880848_GKNS8FEWQADUWY8.png)

判断是否成功比较

![](https://bbs.kanxue.com/upload/attach/202502/880848_GXTSZ396UJMTNAY.png)

再有一个加法 _pyx_mstate[30] 即为 1, 注意 这里判断 v74 ，再进行加法的

![](https://bbs.kanxue.com/upload/attach/202502/880848_SR7HXV2NMC7HUFE.png)

大于 4 就 break 退出循环

![](https://bbs.kanxue.com/upload/attach/202502/880848_T9VTVJ6FX2B6XA5.png)

这里 check 函数的一个循环逻辑走完了, 趁热打铁把代码还原一下

```
input = parm[0]
#l = [[[40,43,41,47,39,45,42,46]]]
l = [0x12287F38, 2563918650,1244723021,3773946743,0x23A1268,2918417411,2282784775,3628702646]
 
i = 0
n_count = 0
while i<= 4:
    cur_s = input[i*8:(i+1)*8]
    cur_res = rand0m(cur_s)
     
    if l[i*2] == cur_res[1] and l[i*2+1] == cur_res[0]:
        n_count = n_count + 1
    
```

继续向下， 判断 v77 是否等于 v19 , v77 为 pyx_mstate[32]

![](https://bbs.kanxue.com/upload/attach/202502/880848_WFM7QSG52JX7Y75.png)

最后返回 v78

![](https://bbs.kanxue.com/upload/attach/202502/880848_TCXX39RA37J4V58.png)

继续完善代码：

```
input = parm[0]
#l = [[[40,43,41,47,39,45,42,46]]]
l = [0x12287F38, 2563918650,1244723021,3773946743,0x23A1268,2918417411,2282784775,3628702646]
 
i = 0
n_count = 0
while i<= 4:
    cur_s = input[i*8:(i+1)*8]
    cur_res = rand0m(cur_s)
     
    if l[i*2] == cur_res[1] and l[i*2+1] == cur_res[0]:
        n_count = n_count + 1
     
if n_count == 4:
    return True
else:
    return False
    
```

现在找到 rand0m 函数的逻辑逆向出来

也是和上面一样的方法搜索字符串寻找该函数

创建一个元组

![](https://bbs.kanxue.com/upload/attach/202502/880848_KZ3QKW3FZNFTUJ5.png)

传入的参数做类型转换 int(x,0x10)

![](https://bbs.kanxue.com/upload/attach/202502/880848_AVFV6FXDAR6HRN3.png)

做 xor 2654435769 和 右移 5

v6 = v14 = PyNumber_Xor(v12, _pyx_mstate[44])

v3 = v15 = Pyx_Rshift(v12, _pyx_mstate[33], 5, 0)

![](https://bbs.kanxue.com/upload/attach/202502/880848_3GP44FQGDQYQVYB.png)

左移 4

v8 = v12 = v21 = PyNumber_Lshift(v12, _pyx_mstate[32]);

![](https://bbs.kanxue.com/upload/attach/202502/880848_T5U9MD55JR9HV5A.png)

与运算 4198170623

v22 = (int *)PyNumber_And(v12, v16[48], v17, v18);// & 4198170623

![](https://bbs.kanxue.com/upload/attach/202502/880848_BUYSZM9GNHEYAVN.png)

右移运算 加法运算

v24 = Pyx_Rshift((__int64)v3, _pyx_mstate[37], 23, 0);

v3 = v25 = PyNumber_Add(v22, v24); // v22 + v24

![](https://bbs.kanxue.com/upload/attach/202502/880848_7UA6SVU2QR2SPGJ.png)

右移 11， 次方 0x10001

v6 = v28 = v27 = Pyx_Rshift((__int64)v6, _pyx_mstate[35], 11, 1);

v8 = v30 = PyNumber_Power(v28, _pyx_mstate[38], Py_NoneStruct);

![](https://bbs.kanxue.com/upload/attach/202502/880848_4KG9CC58N84VHWM.png)

模 4294967293

v31 = (int *)PyNumber_Remainder(v30, _pyx_mstate[49]);

![](https://bbs.kanxue.com/upload/attach/202502/880848_S7WX5Y2VBX9QUNE.png)

再创建一个元组

(v31, v3)

![](https://bbs.kanxue.com/upload/attach/202502/880848_EYVAT9RCYKMBAAK.png)

返回该元组

![](https://bbs.kanxue.com/upload/attach/202502/880848_6KKMD2Y35TBEK3A.png)

现在逻辑基本上清晰

```
v12 = int(a1, 0x10)
v6 = v14 = PyNumber_Xor(v12, _pyx_mstate[44])
v3 = v15 = Pyx_Rshift(v12, _pyx_mstate[33], 5, 0)
v8 = v12 = v21 = PyNumber_Lshift(v12, _pyx_mstate[32]);
v22 = (int *)PyNumber_And(v12, v16[48], v17, v18);// & 4198170623
v24 = Pyx_Rshift((__int64)v3, _pyx_mstate[37], 23, 0);
v3 = v25 = PyNumber_Add(v22, v24); 
v6 = v28 = v27 = Pyx_Rshift((__int64)v6, _pyx_mstate[35], 11, 1);
v8 = v30 = PyNumber_Power(v28, _pyx_mstate[38], Py_NoneStruct);
v31 = (int *)PyNumber_Remainder(v30, _pyx_mstate[49]);
return (v31, v3)
```

整理:

```
def rand0m(a1):
    v12 = int(a1, 0x10)
    v3 = (v12 & 4198170623) + (v12 >> (5 + 0x17))
    v31 = pow(((v12 ^ 2654435769) >> 0xb), 0x10001, 4294967293)
    return v31, v3
```

解法

这里 v31 是 rsa 可以求出来 ，但缺失 数据 0xb 位， v3 刚好可以补上缺失的位, 但要注意结合 check 函数，他们比较的时候，是先源 [0] == res[1] 源 [1] == res[0]

```
import libnum
e = 0x10001
p = 9241
q = 464773
d = libnum.invmod(e, (p-1) * (q-1))
 
 
l = [0x12287F38, 2563918650, 1244723021, 3773946743, 0x23A1268, 2918417411, 2282784775, 3628702646]
l = [l[i * 2:i * 2 + 2] for i in range(4)]
 
for i in l:
    # print(i)
    cur_val = (i[0] & 0xf)
    b = (i[0] >> 4) & 0xfff
    b = hex(b)[2:]
    res = pow(i[1], d, p * q)
 
    res = res << 11
    res = res ^ 2654435769
 
    a = hex(res)[2:]
    a = a[:5]
    c = a + b
    print(c, end="")
 
 
 
# 813a97f3d4b34f74802ba12678950880
```

总结
==

文章讲解了 pyd 的一些特性，希望能够助力师傅们在 pyd 逆向中如鱼得水。当然作者水平有限，可能没办法讲解更深的内容，文章也可能存在错误及不足之处，感谢理解。<抱拳>

[[注意] 看雪招聘，专注安全领域的专业人才平台！](https://job.kanxue.com/)

最后于 9 小时前 被 mingyuexc 编辑 ，原因：

[#调试逆向](forum-4-1-1.htm) [#其他内容](forum-4-1-10.htm)

上传的附件：

*   [rand0m_84de0f4fbc09d405e40850e97a05765f.zip](javascript:void(0);) （20.34kb，2 次下载）
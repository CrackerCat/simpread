> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [github.com](https://github.com/wuhx/AppInspect/wiki/%E4%BB%8EFrida%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0ArtHook%EF%BC%88%E4%BA%8C%EF%BC%89)

> codeless Android hook (experimental). Contribute to wuhx/AppInspect development by creating an accoun......

[上一篇](https://github.com/wuhx/AppInspect/wiki/%E4%BB%8EFrida%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0ArtHook%EF%BC%88%E4%B8%80%EF%BC%89)描述了获取 libart.so 的底层操作，这里继续分析 Frida 的 js 层如何利用这些接口来实现 ArtHook。

Frida 把需要使用的 Api，弄了一张[列表](https://github.com/frida/frida-java-bridge/blob/7ab7135ae7a9ae8a8946b2831f1fb0eed2810367/lib/android.js#L138)

表的 key 是需要查找的 symbol 名，表的 value 有几种类型：

##### [](#1%E5%8F%AF%E4%BB%A5%E7%9B%B4%E6%8E%A5%E8%B0%83%E7%94%A8%E7%9A%84symbolvalue%E4%B8%BAapi%E5%90%8D%E7%A7%B0symbol%E8%BF%94%E5%9B%9E%E5%80%BC%E7%B1%BB%E5%9E%8B%E5%92%8C%E5%8F%82%E6%95%B0%E7%B1%BB%E5%9E%8B%E5%88%97%E8%A1%A8)1）可以直接调用的 symbol，value 为 Api 名称 + symbol 返回值类型和参数类型列表。

如:

```
// Android >= 7 原型void ClassLinker::VisitClasses(ClassVisitor* visitor)
_ZN3art11ClassLinker12VisitClassesEPNS_12ClassVisitorE: ['art::ClassLinker::VisitClasses', 'void', ['pointer', 'pointer']]

```

`_ZN3art11ClassLinker12VisitClassesEPNS_12ClassVisitorE` 为要查找的符号。

`art::ClassLinker::VisitClasses` 调用时 Api 的名称。

`void` api 返回值

`['pointer', 'pointer']` api 参数列表

##### [](#2api%E9%9C%80%E8%A6%81%E9%80%9A%E8%BF%87%E8%B0%83%E7%94%A8%E5%85%B6%E4%BB%96%E5%87%BD%E6%95%B0%E9%97%B4%E6%8E%A5%E5%AE%9E%E7%8E%B0)2）Api 需要通过调用其他函数间接实现

如：

```
// Android < 7 原型 void ClassLinker::VisitClasses(ClassVisitor* visitor, void* arg)
_ZN4art11ClassLinker12VisitClassesEPFbPNS_6mirror5ClassEPvES4_: function (address) {
  const visitClasses = new NativeFunction(address, 'void', ['pointer', 'pointer', 'pointer'], nativeFunctionOptions);
  this['art::ClassLinker::VisitClasses'] = function (classLinker, visitor) {
    visitClasses(classLinker, visitor, NULL);
  };
},

```

这种情况下

Api`art::ClassLinker::VisitClasses` 封装了`void ClassLinker::VisitClasses(ClassVisitor* visitor, void* arg)`的最后一个参数（固定为 NULL），从而统一了 Android 7 和之前版本的接口。

##### [](#%E5%88%9D%E5%A7%8B%E5%8C%96api)初始化 Api

```
    const exportByName = Module
      .enumerateExports(api.module)
      .reduce(function (result, exp) {
        result[exp.name] = exp;
        return result;
      }, {});

   //遍历Api表
    Object.keys(functions)
      .forEach(function (name) {
        const exp = exportByName[name];
        if (exp !== undefined && exp.type === 'function') {
          const signature = functions[name];
          if (typeof signature === 'function') {
            //第二种情况间接调用 ，temporaryApi作为this，在函数内给api地址赋值
            signature.call(temporaryApi, exp.address); 
          } else {
            //第一种情况，根据signature转成对应的函数直接赋值
            temporaryApi[signature[0]] = new NativeFunction(exp.address, signature[1], signature[2], nativeFunctionOptions);
           
          }
        } else {
          if (!optionals.has(name)) {
            missing.push(name);
          }
        }
      });

```

##### [](#_enumerateloadedclassesart%E8%B0%83%E7%94%A8api%E8%8E%B7%E5%8F%96%E6%89%80%E6%9C%89%E5%8A%A0%E8%BD%BD%E7%B1%BB)_enumerateLoadedClassesArt 调用 Api 获取所有加载类

```
 _enumerateLoadedClassesArt (callbacks) {
   //略....
      const collectClassHandles = makeArtClassVisitor(klass => {
        classHandles.push(addGlobalReference(vmHandle, thread, klass));
        return true;
      });

      api['art::ClassLinker::VisitClasses'](api.artClassLinker.address, collectClassHandles);
	//略....
}


```

先看看 AOSP 中 [VisitClasses 函数原型](http://aospxref.com/android-11.0.0_r21/xref/art/runtime/class_linker.cc?fi=VisitClasses#2373)

```
void ClassLinker::VisitClasses(ClassVisitor* visitor);

class ClassVisitor {
 public:
  virtual ~ClassVisitor() {}
  // Return true to continue visiting.
  virtual bool operator()(ObjPtr<mirror::Class> klass) = 0;
};

```

完整的看一下 [AOSP 中如何调用 VisitClasses 函数](http://aospxref.com/android-11.0.0_r21/xref/art/openjdkjvmti/ti_redefine.cc#485):

```
art::ClassFuncVisitor cfv(
  [&](art::ObjPtr<art::mirror::Class> k) REQUIRES_SHARED(art::Locks::mutator_lock_) {
    // if there is any class 'K' that is a subtype (i.e. extends) klass and has pointer-jni-ids
    // we cannot structurally redefine the class 'k' since we would structurally redefine the
    // subtype.
    if (k->IsLoaded() && klass->IsAssignableFrom(k) && has_pointer_marker(k)) {
      *error_msg = StringPrintf(
          "%s has active pointer jni-ids from subtype %s and cannot be redefined structurally",
          klass->PrettyClass().c_str(),
          k->PrettyClass().c_str());
      res = ERR(UNMODIFIABLE_CLASS);
      return false;
    }
    return true;
  });
art::Runtime::Current()->GetClassLinker()->VisitClasses(&cfv)

```

ClassFuncVisitor 是一个实现了 ClassVisitor 中虚函数的子类

```
template <typename Func>
class ClassFuncVisitor final : public ClassVisitor {
 public:
  explicit ClassFuncVisitor(Func func) : func_(func) {}
  bool operator()(ObjPtr<mirror::Class> klass) override REQUIRES_SHARED(Locks::mutator_lock_) {
    return func_(klass);
  }

 private:
  Func func_;
};

```

看起来相当复杂，然而 Frida 中创建这个 ClassVisitor * 参数异常简单，调用 [makeArtClassVisitor](https://github.com/frida/frida-java-bridge/blob/master/lib/android.js#L1237) 创建了一个 NativeCallback 就完事了，这么神奇吗？

```
function makeArtClassVisitor (visit) {
  //忽略Android < 7.0代码
  return new NativeCallback(klass => {
    return visit(klass) === true ? 1 : 0;
  }, 'bool', ['pointer', 'pointer']);
}  

```

看一下 [NativeCallback](https://frida.re/docs/javascript-api/#nativecallback) 的定义：

```
create a new NativeCallback implemented by the JavaScript function func, where returnType specifies the return type, and the argTypes array specifies the argument types. 


```

只是定义了一个函数，转成指针怎么能和 ClassVisitor 类实例指针兼容的呢？

让我们冷静下来仔细想想这个问题：

##### [](#%E5%A6%82%E4%BD%95%E5%9C%A8%E6%B2%A1%E6%9C%89%E6%BA%90%E7%A0%81%E7%9A%84%E6%83%85%E5%86%B5%E4%B8%8B%E5%88%9B%E5%BB%BA%E4%B8%80%E4%B8%AAc%E7%B1%BB)如何在没有源码的情况下创建一个 C++ 类。

理论上有两种方法：

1.  逆向分析原 C++ 类的函数成员变量，写一个类似的类，并创建出实例。对于 ClassVisitor 只需参考 AOSP，更简单，但 Frida 是纯 js 实现，没法这么搞。
2.  模拟真正实例的内存分布，用代码分配一块内存，并写入相同结构的数据。对于 ClassVisitor 这种只有一个虚函数的类，可能它的内存分布就是一个回调函数。

为了验证这个想法，来做个实验：

```
#include <iostream>

class A
{
public:
  //模拟ClassVisitor只有一个虚函数
    virtual int method(void *arg) 
    {
        printf("call method in A");
        return 0;
    }
};

class B : public A
{
public:
    int method(void *arg)
    {
      //B继承A并实现这个虚函数
        printf("call method in B");
        return 1;
    }
};
//调试打印内存
void Dump( const void * mem, unsigned int n ) {
  const unsigned char * p = reinterpret_cast< const unsigned char*>( mem );
  for ( unsigned int i = 0; i < n; i++ ) {
    std::cout << std::hex  << int(p[i]) << " ";
  }
  std::cout << std::endl;
}
//调用类B实例
int call(B *b)
{
    return b->method(NULL);
}

int main(int argc, char *argv[])
{
  B* b = new B();
  call(b); //打印call method in B
  Dump(b, 32); //打印 50 80 eb 2 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
}


```

B 的内容似乎就是一个指针。

##### [](#%E7%BB%A7%E7%BB%AD%E6%B5%8B%E8%AF%95%E7%94%A8%E5%87%BD%E6%95%B0%E6%8C%87%E9%92%88%E4%BB%A3%E6%9B%BF%E7%B1%BB%E5%AE%9E%E4%BE%8B)继续测试：用函数指针代替类实例

```
int fn_method(void *o)
{
    printf("call fn_method");
    return 3;
}

int main(int argc, char *argv[])
{
  //B b;
  //call(&b); //打印call method in B
  //Dump(&b, sizeof(B)); //打印 50 80 b1 4 1 0 0 0  

  size_t *obj = (size_t*)malloc(8);
  *obj = (size_t)&fn_method;
  call((B*)&obj); //打印call fn_method
}

```

至此确认了 Frida 用一个 NativeCallback 替代 ClassVisitor 实例的黑魔法。

背后的原理是：C++ 类实例起始地址上放的是保存所有虚函数地址的 VTable 指针，Frida 的做法相当于手动创建了这张 VTable，

另外 C++ 类函数的第一个参数是实例的 this 指针，所以 Frida 用的回调的 signature 是`'bool', ['pointer', 'pointer']`，上面例子对应也应改成：`int fn_method(void *this, void* arg)`

##### [](#%E5%8F%82%E8%80%83)参考：

[Fast Virtual Functions: Hacking the VTable for Fun and Profit](https://medium.com/@calebleak/fast-virtual-functions-hacking-the-vtable-for-fun-and-profit-25c36409c5e0)
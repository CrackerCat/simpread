> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/forum.php?mod=viewthread&tid=497018) ![](https://avatar.52pojie.cn/data/avatar/000/16/94/61_avatar_middle.jpg)coldnight  _ 本帖最后由 coldnight 于 2016-7-24 22:49 编辑_  
**Qt5 程序初步逆向分析 + 解析脚本**  
本文参考了以下文章：  

> > Qt Internals & Reversing  
> > 【翻译】Qt 内部机制及逆向 QT 的信号与槽机制介绍

本文是参考以上文章作出的，但是文章对象是 Qt4 的，其解析脚本已不适用于 Qt5，本人重新分析了 Qt5 程序的元数据结构，并给出了解析脚本，方便 Qt5 程序的逆向[破解](https://www.52pojie.cn)。  
第一次发贴，有任何疑问请回贴，谢谢。  
**Qt 的信号 / 槽机制**  
Qt 是一个跨平台的 C++ 图形用户界面应用程序框架。它提供给开发者建立图形用户界面所需的功能，广泛用于开发 GUI 程序，也可用于开发非 GUI 程序。  
Qt 使用信号 (Signal) 和槽（Slot）机制用于对象间的通信。可以将信号和槽通过 QObject 对象的 connet 函数关联起来。我们可以使用 emit（Qt 定义的语句）发出某个信号，与该信号关联的槽就会接受到信号进行处理。  
下面是一个简单的 Qt5 代码：  
[C++] _纯文本查看_ _复制代码_

```
// tsignal.h
#include #include // 必须继承QObject才能使用信号和槽
class TsignalApp:public QMainWindow
 {
public:
    TsignalApp();
    void slotFileNew();
     Q_OBJECT
     // 信号声明区
     signals:
         // 声明信号 mySignal()
         void mySignal();
         // 声明信号 mySignal(int)
         void mySignal(int x);
         // 声明信号 mySignalParam(int,int)
         void mySignalParam(int x,int y);
     // 槽声明区
     public slots:
         // 声明槽函数 mySlot()
         void mySlot();
         // 声明槽函数 mySlot(int)
         void mySlot(int x);
         // 声明槽函数 mySignalParam (int，int)
         void mySlotParam(int x,int y);
         TsignalApp* mySlot2();
 };
 
 
tsignal.cpp
#include "tsignal.h"
#include TsignalApp::TsignalApp()
{
    // 将信号 mySignal() 与槽 mySlot() 相关联
    connect(this,SIGNAL(mySignal()),SLOT(mySlot()));
    // 将信号 mySignal(int) 与槽 mySlot(int) 相关联
    connect(this,SIGNAL(mySignal(int)),SLOT(mySlot(int)));
    // 将信号 mySignalParam(int,int) 与槽 mySlotParam(int,int) 相关联
    connect(this,SIGNAL(mySignalParam(int,int)),SLOT(mySlotParam(int,int)));
}
// 定义槽函数 mySlot()
void TsignalApp::mySlot()
{
    QMessageBox::about(this,"Tsignal", "This is a signal/slot sample withoutparameter.");
}
// 定义槽函数 mySlot(int)
void TsignalApp::mySlot(int x)
{
    QMessageBox::about(this,"Tsignal", "This is a signal/slot sample with oneparameter.");
}
// 定义槽函数 mySlotParam(int,int)
void TsignalApp::mySlotParam(int x,int y)
{
    char s[256];
    sprintf(s,"x:%d y:%d",x,y);
    QMessageBox::about(this,"Tsignal", s);
}
void TsignalApp::slotFileNew()
{
    // 发射信号 mySignal()
    emit mySignal();
    // 发射信号 mySignal(int)
    emit mySignal(5);
    // 发射信号 mySignalParam(5，100)
    emit mySignalParam(5,100);
}
 
# main.cpp
#include "tsignal.h"
#include int main(int argc, char *argv[])
{
    QApplication a(argc, argv);
    TsignalApp w;
    w.slotFileNew();
    return a.exec();
} 
```

上面的代码编译后运行，会弹出依次弹出，"This is a signal/slot sample withoutparameter."、"This is a signal/slot sample with oneparameter."、"Tsignal x: y:" 的窗口。  
从上面的代码可以看出，Qt 定义 signals、slots、emit 等关键字用于简化 Qt 程序的开发。Qt 使用元对象编译器 moc（meta object compiler）将这些关键字翻译成正常的 C++ 代码，以便 gcc 或者 vc 编译。以上面的代码为例，tsignal.cpp 中的 qt 关键字将被编译成如下代码：  
[C++] _纯文本查看_ _复制代码_

```
// moc_tsignal.cpp
#include #include #if !defined(Q_MOC_OUTPUT_REVISION)
#error "The header file 'tsignal.h' doesn't include ."
#elif Q_MOC_OUTPUT_REVISION != 67
#error "This file was generated using the moc from 5.4.1. It"
#error "cannot be used with the include files from this version of Qt."
#error "(The moc has changed too much.)"
#endif
 
QT_BEGIN_MOC_NAMESPACE
struct qt_meta_stringdata_TsignalApp_t {
    QByteArrayData data[10];
    char stringdata[78];
};
#define QT_MOC_LITERAL(idx, ofs, len) \
    Q_STATIC_BYTE_ARRAY_DATA_HEADER_INITIALIZER_WITH_OFFSET(len, \
    qptrdiff(offsetof(qt_meta_stringdata_TsignalApp_t, stringdata) + ofs \
        - idx * sizeof(QByteArrayData)) \
    )
static const qt_meta_stringdata_TsignalApp_t qt_meta_stringdata_TsignalApp = {
    {
QT_MOC_LITERAL(0, 0, 10), // "TsignalApp"
QT_MOC_LITERAL(1, 11, 8), // "mySignal"
QT_MOC_LITERAL(2, 20, 0), // ""
QT_MOC_LITERAL(3, 21, 1), // "x"
QT_MOC_LITERAL(4, 23, 13), // "mySignalParam"
QT_MOC_LITERAL(5, 37, 1), // "y"
QT_MOC_LITERAL(6, 39, 6), // "mySlot"
QT_MOC_LITERAL(7, 46, 11), // "mySlotParam"
QT_MOC_LITERAL(8, 58, 7), // "mySlot2"
QT_MOC_LITERAL(9, 66, 11) // "TsignalApp*"
 
    },
    "TsignalApp\0mySignal\0\0x\0mySignalParam\0"
    "y\0mySlot\0mySlotParam\0mySlot2\0TsignalApp*"
};
#undef QT_MOC_LITERAL
 
static const uint qt_meta_data_TsignalApp[] = {
 
 // content:
       7,       // revision
       0,       // classname
       0,    0, // classinfo
       7,   14, // methods
       0,    0, // properties
       0,    0, // enums/sets
       0,    0, // constructors
       0,       // flags
       3,       // signalCount
 
 // signals: name, argc, parameters, tag, flags
       1,    0,   49,    2, 0x06 /* Public */,
       1,    1,   50,    2, 0x06 /* Public */,
       4,    2,   53,    2, 0x06 /* Public */,
 
 // slots: name, argc, parameters, tag, flags
       6,    0,   58,    2, 0x0a /* Public */,
       6,    1,   59,    2, 0x0a /* Public */,
       7,    2,   62,    2, 0x0a /* Public */,
       8,    0,   67,    2, 0x0a /* Public */,
 
 // signals: parameters
    QMetaType::Void,
    QMetaType::Void, QMetaType::Int,    3,
    QMetaType::Void, QMetaType::Int, QMetaType::Int,    3,    5,
 
 // slots: parameters
    QMetaType::Void,
    QMetaType::Void, QMetaType::Int,    3,
    QMetaType::Void, QMetaType::Int, QMetaType::Int,    3,    5,
    0x80000000 | 9,
 
       0        // eod
};
 
void TsignalApp::qt_static_metacall(QObject *_o, QMetaObject::Call _c, int _id, void **_a)
{
    if (_c == QMetaObject::InvokeMetaMethod) {
        TsignalApp *_t = static_cast(_o);
        switch (_id) {
        case 0: _t->mySignal(); break;
        case 1: _t->mySignal((*reinterpret_cast< int(*)>(_a[1]))); break;
        case 2: _t->mySignalParam((*reinterpret_cast< int(*)>(_a[1])),(*reinterpret_cast< int(*)>(_a[2]))); break;
        case 3: _t->mySlot(); break;
        case 4: _t->mySlot((*reinterpret_cast< int(*)>(_a[1]))); break;
        case 5: _t->mySlotParam((*reinterpret_cast< int(*)>(_a[1])),(*reinterpret_cast< int(*)>(_a[2]))); break;
        case 6: { TsignalApp* _r = _t->mySlot2();
            if (_a[0]) *reinterpret_cast< TsignalApp**>(_a[0]) = _r; }  break;
        default: ;
        }
    } else if (_c == QMetaObject::IndexOfMethod) {
        int *result = reinterpret_cast(_a[0]);
        void **func = reinterpret_cast(_a[1]);
        {
            typedef void (TsignalApp::*_t)();
            if (*reinterpret_cast<_t *>(func) == static_cast<_t>(&TsignalApp::mySignal)) {
                *result = 0;
            }
        }
        {
            typedef void (TsignalApp::*_t)(int );
            if (*reinterpret_cast<_t *>(func) == static_cast<_t>(&TsignalApp::mySignal)) {
                *result = 1;
            }
        }
        {
            typedef void (TsignalApp::*_t)(int , int );
            if (*reinterpret_cast<_t *>(func) == static_cast<_t>(&TsignalApp::mySignalParam)) {
                *result = 2;
            }
        }
    }
}
 
const QMetaObject TsignalApp::staticMetaObject = {
    { &QMainWindow::staticMetaObject, qt_meta_stringdata_TsignalApp.data,
      qt_meta_data_TsignalApp,  qt_static_metacall, Q_NULLPTR, Q_NULLPTR}
};
 
 
const QMetaObject *TsignalApp::metaObject() const
{
    return QObject::d_ptr->metaObject ? QObject::d_ptr->dynamicMetaObject() : &staticMetaObject;
}
 
void *TsignalApp::qt_metacast(const char *_clname)
{
    if (!_clname) return Q_NULLPTR;
    if (!strcmp(_clname, qt_meta_stringdata_TsignalApp.stringdata))
        return static_cast(const_cast< TsignalApp*>(this));
    return QMainWindow::qt_metacast(_clname);
}
 
int TsignalApp::qt_metacall(QMetaObject::Call _c, int _id, void **_a)
{
    _id = QMainWindow::qt_metacall(_c, _id, _a);
    if (_id < 0)
        return _id;
    if (_c == QMetaObject::InvokeMetaMethod) {
        if (_id < 7)
            qt_static_metacall(this, _c, _id, _a);
        _id -= 7;
    } else if (_c == QMetaObject::RegisterMethodArgumentMetaType) {
        if (_id < 7)
            *reinterpret_cast(_a[0]) = -1;
        _id -= 7;
    }
    return _id;
}
 
// SIGNAL 0
void TsignalApp::mySignal()
{
    QMetaObject::activate(this, &staticMetaObject, 0, Q_NULLPTR);
}
 
// SIGNAL 1
void TsignalApp::mySignal(int _t1)
{
    void *_a[] = { Q_NULLPTR, const_cast(reinterpret_cast(&_t1)) };
    QMetaObject::activate(this, &staticMetaObject, 1, _a);
}
 
// SIGNAL 2
void TsignalApp::mySignalParam(int _t1, int _t2)
{
    void *_a[] = { Q_NULLPTR, const_cast(reinterpret_cast(&_t1)), const_cast(reinterpret_cast(&_t2)) };
    QMetaObject::activate(this, &staticMetaObject, 2, _a);
}
 
QT_END_MOC_NAMESPACE 
```

可以看到，mySignal 信号被翻译成如下形式  
[C++] _纯文本查看_ _复制代码_

```
void TsignalApp::mySignal() {
    QMetaObject::activate(this, &staticMetaObject, 0, Q_NULLPTR);
}

```

如果信号带参数，那么会多一次类型形转换的过程：  
[C++] _纯文本查看_ _复制代码_

```
void TsignalApp::mySignalParam(int _t1, int _t2) {
    void *_a[] = { Q_NULLPTR, const_cast(reinterpret_cast(&_t1)), const_cast(reinterpret_cast(&_t2)) };
    QMetaObject::activate(this, &staticMetaObject, 2, _a);
} 
```

无论带不带参数，发出信号都会调用的 QMetaObject::activate 函数。  
**Qt 元数据结构**  
QMetaObject::activate 使得 Qt 可以动态调用信号关联的槽，但这给也我们破解 Qt 程序带来了困难。  
假设有这样一个程序，在接收到注册码调用后进行检查，如果正确，则触发触发注册成功的信号，否则触发注册失败信号。注册码破解的直接思路是通过注册失败的信息查找注册函数，但是由上面我们可以知道发送信号的时候是通过 QMetaObject::activate 触发信号，这不是一个直接的函数调用。即使我们在出错信息上下断点，此时栈上是大量的 Qt 核心库调用信息，触发信号的函数查找起来十分麻烦。  
Qt 是通过 connect 将信号与槽关联。但是 connet 的时候，我们无法直接知道与信号相关联的槽函数的位置。例如：  
[C++] _纯文本查看_ _复制代码_

```
connect(this,SIGNAL(mySignal()),SLOT(mySlot()));

```

在编译后使用 IDA 反编译如下：  
![](http://i.imgur.com/G4Xa382.png)  
没有出现 mySlot 的地址，保留的只有 mySlot 的名称。  
但是 Qt 是可以正确调用相应的函数的，原因是 Qt 会将信号和槽的元数据保存，并在运行的时候动态查找相应的方法。在 moc_tsignal.cpp 中，元数据如下：  

> [C++] _纯文本查看_ _复制代码_
> 
> ```
> static const qt_meta_stringdata_TsignalApp_t qt_meta_stringdata_TsignalApp = {
>     {
> QT_MOC_LITERAL(0, 0, 10), // "TsignalApp"
> QT_MOC_LITERAL(1, 11, 8), // "mySignal"
> QT_MOC_LITERAL(2, 20, 0), // ""
> QT_MOC_LITERAL(3, 21, 1), // "x"
> QT_MOC_LITERAL(4, 23, 13), // "mySignalParam"
> QT_MOC_LITERAL(5, 37, 1), // "y"
> QT_MOC_LITERAL(6, 39, 6), // "mySlot"
> QT_MOC_LITERAL(7, 46, 11), // "mySlotParam"
> QT_MOC_LITERAL(8, 58, 7), // "mySlot2"
> QT_MOC_LITERAL(9, 66, 11) // "TsignalApp*"
>  
>     },
>     "TsignalApp\0mySignal\0\0x\0mySignalParam\0"
>     "y\0mySlot\0mySlotParam\0mySlot2\0TsignalApp*"
> };
> #undef QT_MOC_LITERAL
>  
> static const uint qt_meta_data_TsignalApp[] = {
>  
>  // content:
>        7,       // revision
>        0,       // classname
>        0,    0, // classinfo
>        7,   14, // methods
>        0,    0, // properties
>        0,    0, // enums/sets
>        0,    0, // constructors
>        0,       // flags
>        3,       // signalCount
>  
>  // signals: name, argc, parameters, tag, flags
>        1,    0,   49,    2, 0x06 /* Public */,
>        1,    1,   50,    2, 0x06 /* Public */,
>        4,    2,   53,    2, 0x06 /* Public */,
>  
>  // slots: name, argc, parameters, tag, flags
>        6,    0,   58,    2, 0x0a /* Public */,
>        6,    1,   59,    2, 0x0a /* Public */,
>        7,    2,   62,    2, 0x0a /* Public */,
>        8,    0,   67,    2, 0x0a /* Public */,
>  
>  // signals: parameters
>     QMetaType::Void,
>     QMetaType::Void, QMetaType::Int,    3,
>     QMetaType::Void, QMetaType::Int, QMetaType::Int,    3,    5,
>  
>  // slots: parameters
>     QMetaType::Void,
>     QMetaType::Void, QMetaType::Int,    3,
>     QMetaType::Void, QMetaType::Int, QMetaType::Int,    3,    5,
>     0x80000000 | 9,
>  
>        0        // eod
> };
> 
> ```
> 
> 此外，每个 Qt 对象都有一个 qt___static___metacall 方法用于确定调用的函数：  

[C++] _纯文本查看_ _复制代码_

```
void TsignalApp::qt_static_metacall(QObject *_o, QMetaObject::Call _c, int _id, void **_a)
{
    if (_c == QMetaObject::InvokeMetaMethod) {
        TsignalApp *_t = static_cast(_o);
        switch (_id) {
        case 0: _t->mySignal(); break;
        case 1: _t->mySignal((*reinterpret_cast< int(*)>(_a[1]))); break;
        case 2: _t->mySignalParam((*reinterpret_cast< int(*)>(_a[1])),(*reinterpret_cast< int(*)>(_a[2]))); break;
        case 3: _t->mySlot(); break;
        case 4: _t->mySlot((*reinterpret_cast< int(*)>(_a[1]))); break;
        case 5: _t->mySlotParam((*reinterpret_cast< int(*)>(_a[1])),(*reinterpret_cast< int(*)>(_a[2]))); break;
        case 6: { TsignalApp* _r = _t->mySlot2();
            if (_a[0]) *reinterpret_cast< TsignalApp**>(_a[0]) = _r; }  break;
        default: ;
        }
    } else if (_c == QMetaObject::IndexOfMethod) {..
    }
} 
```

其中的_id 是相应 Qt 方法（信号 / 槽等）的索引，将索引与方法对应十分简单，之后会说。  
实际上，在编译后 Qt 程序中 qt_meta_stringdata_TsignalApp 和 qt_meta_data_TsignalApp 会被封装在另一个数据结构中，由 Qt 的源代码分析得，该结构为 QMetaObject 内部的结构体 d：  

> [C++] _纯文本查看_ _复制代码_
> 
> ```
> // qobjectdefs.h
> struct QMetaObject {
>     struct { // private data
>     const QMetaObject *superdata;
>     const QByteArrayData *stringdata;
>     const uint *data;
>     typedef void (*StaticMetacallFunction)(QObject *, QMetaObject::Call, int, void **);
>     StaticMetacallFunction static_metacall;
>     const QMetaObject * const *relatedMetaObjects;
>     void *extradata; //reserved for future use
>     } d;
> };
> 
> ```

在结构体由索引（index）得到对应的方法代码如下：  

> [C++] _纯文本查看_ _复制代码_
> 
> ```
> // qmetaobject.cpp
> QMetaMethod QMetaObject::method(int index) const
> {
>     int i = index;
>     i -= methodOffset();
>     if (i < 0 && d.superdata)
>         return d.superdata->method(index);
>  
>     QMetaMethod result;
>     if (i >= 0 && i < priv(d.data)->methodCount) {
>         result.mobj = this;
>         result.handle = priv(d.data)->methodData + 5*i;
>     }
>     return result;
> }
> 
> ```
> 
> 其中 priv(d.data) 的代码如下：  
> [C++] _纯文本查看_ _复制代码_
> 
> ```
> static inline const QMetaObjectPrivate *priv(const uint* data)
> { return reinterpret_cast(data); } 
> ```

即 d.data 指向一个 QMetaObjectPrivate 对象。  
QMetaObjectPrivate 的声明如下：  

> [C++] _纯文本查看_ _复制代码_
> 
> ```
> // qmetaobject_p
> struct QMetaObjectPrivate {
>     enum { OutputRevision = 7 }; // Used by moc, qmetaobjectbuilder and qdbus
>  
>     int revision;
>     int className;
>     int classInfoCount, classInfoData;
>     int methodCount, methodData;
>     int propertyCount, propertyData;
>     int enumeratorCount, enumeratorData;
>     int constructorCount, constructorData; //since revision 2
>     int flags; //since revision 3
>     int signalCount; //since revision 4
> }
> 
> ```

实际上，这就是 moc_tsignal.cpp 中的相应内容  

> [C++] _纯文本查看_ _复制代码_
> 
> ```
> // content:
>       7,       // revision
>       0,       // classname
>       0,    0, // classinfo
>       7,   14, // methods
>       0,    0, // properties
>       0,    0, // enums/sets
>       0,    0, // constructors
>       0,       // flags
>       3,       // signalCount
> 
> ```

由 QMetaObjectPrivate.methodData 成员，可以确定 QMetaMethod 在 d.data 中的偏移，不考虑父类的情况下，索引为 id 的 QMetaMethod 在 d.data 的偏移为 QMetaObjectPrivate.methodData + id * 5  
**  
QMetaMethod  
**QMetaMethod 在 Qt 源代码中只保留 QObject 的指针及偏移量计算的代码，没有定义真正的结构体，为了方便，将结构体整理如下：  
[C++] _纯文本查看_ _复制代码_

```
struct QMetaMethod {
    int name;
    int parameterCount;  // 参数个数
    int typesDataIndex;
    int tag;
    int flag;
}

```

**  
获得方法名  
**  
QMetaMethod 其中的 name 和 typesDataIndex 都是其在 d.stringdata[] 的索引。在 moc 中，由 d.stringdata 由如下宏生成：  

> [C++] _纯文本查看_ _复制代码_
> 
> ```
> QT_MOC_LITERAL(0, 0, 10), // "TsignalApp"
> 
> ```

在宏之后紧跟着 C 语言的字符串数据：  

> [C++] _纯文本查看_ _复制代码_
> 
> ```
> "TsignalApp\0mySignal\0\0x\0mySignalParam\0"
> "y\0mySlot\0mySlotParam\0mySlot2\0TsignalApp*"
> 
> ```

d.stringdata 指向的类型为 QByteArrayData，其定义为：  

> [C++] _纯文本查看_ _复制代码_
> 
> ```
> typedef QArrayData QByteArrayData;
>  
> // qarraydata.h
> struct QTypedArrayData
>     : QArrayData
>  
> struct QArrayData
> {
>     QtPrivate::RefCount ref;
>     int size;
>     uint alloc : 31;
>     uint capacityReserved : 1;
>  
>     qptrdiff offset; // 相对于QArrayData所在地址的偏移
> };
> 
> ```

由 QArrayData.offset，我们可以计算 C 字符串的偏移，由 QArrayData.size 可以确定字符串的长度。  
至此，我们可以得到 QMetaMethod 的方法名。  
例如：设 mth 为 QMetaMethod 对象，通过 d.stringdata[mth.name] 获得 QArrayData 对象 arr，则 & arr + arr.offset 处即为 mth 的方法名字符串。  
**获得方法类型数据**  
QMetaMethod.typesDataIndex 是方法的类型数据偏移，其计算方式如下：  
偏移 = d.data + QMetaMethod.typesDataIndex * 4  
知道类型数据偏移后，我们就可以得到 QMetaMethod 的类型，以上面例子的 mySignalParam 为例，其原型为：  

> `void mySlotParam(int x,int y);  

其编译后的类型信息为：  

> QMetaType::Void, QMetaType::Int, QMetaType::Int,    3,    5,  
> 其中 QMetaType::Void、QMetaType::Int 都是定义在 qmetatype.h 的枚举。上面对应于：  

[C++] _纯文本查看_ _复制代码_

```
返回值类型, 第一个参数类型,第二个参数类型,第一个参数名称,第二个参数名称

```

参数名称和 QMetaMethod.name 一样，是在 stringdata 数组中的索引。  
如果类型没有在 QMetaType 中定义，如 TSingapp * 这样的自定义类型，其类型将以  
[C++] _纯文本查看_ _复制代码_

```
0x80000000 | 类型名索引

```

的形式定义，如例子中的 TsignalApp* mySlot2(); 被编译为：  
[C++] _纯文本查看_ _复制代码_

```
0x80000000 | 9,

```

9 即为  
QT_MOC_LITERAL(9, 66, 11) // "TsignalApp*"  
QMetaMethod.flag 保存相应方法的属性，如 Public 等，这里略过。  
至此，我们已经可以得到完整的 QMetaMethod 的函数签名信息了。  
**确定 QMetaObject.d 位置**  
以上对方法的解析前提是有 QMetaObject.d 的信息，下面来确定该位置。 实际上十分简单，Qt 的元数据类型的签名为 static const，也就是说只要在程序的. rdata 段或者. data 段查找如下形式的结构体即可：  
![](http://i.imgur.com/xOI0n7n.png)  
QMetaObject.d 的主要特征是成员都是指什，superdata 可以为 null，因此找到连续三个的 offset，且第 4 个成员为一函数指针，可以考虑为 QMetaObject.d。  
**破解 Qt5 程序的思路**

*   在错误信息上下断点，其所在函数 err_func 通常为槽。
*   断下后跟踪到对应的 static_metacall，确定 err_func 的索引 idx
*   解析 QMetaObject.d，在 d 的 data 中查找 idx 对应的方法，解析方法的名称 name
*   在字符串引用中查找 name（可能要加上 SLOT、SINGAL 前缀）的引用，确定 Qt 进行 connect 的信号
*   查到对信号的引用，逆向完成破解  
    

**  
总结**  
Qt 通过信号 / 槽机制实现事件通信，可以在运行时动态 connet。Qt 在生成的可执行文件中保存了 Qt 对象（QObject）的元数据 (QMetaObject.d 结构），所有 Qt 对象均有一个 static_metacall 函数，根据索引调用相应的方法。我们可以根据 QMetaObject.d->data 获得方法的名称、返回值、参数、索引等信息。  
本人写了一个 IDA 脚本，功能如下：  

*   自动完成方法的名称、返回值、参数、索引的解析；
*   Qt 结构体（QMetaObject.d、QArrayData、QMethod）的添加和标记；
*   static_metacall 函数的重命名；
*   支持 32 位和 64 位的 Qt5 程序  
    

  
使用方法：  
将光标置于 QMetaObject.d 的起始处，Alt+F7 调用本脚本即可。  
效果如下：  
![](https://raw.githubusercontent.com/xzefeng/qtmetaparser/master/img/qtmetaobject.png)  
运行脚本后：  
![](https://raw.githubusercontent.com/xzefeng/qtmetaparser/master/img/qtmetaobject_parsed.png)  
QMetaObject.d.data 的效果如下：  
![](https://raw.githubusercontent.com/xzefeng/qtmetaparser/master/img/qtmetaobjectprivate_parsed.png)  
方法的名称和参数都还原了。  
![](https://avatar.52pojie.cn/images/noavatar_middle.gif)lacoucou  事实上还有动态初始化的。。。。。。。  

> .data:0094E764 ?staticMetaObject@testDlg@@2UQMetaObject@@B dd 0  
> .data:0094E764                                         ; DATA XREF: testDlg::metaObject(void):loc_4021B2o  
> .data:0094E764                                         ; _dynamic_initializer_for__testDlg__staticMetaObject__+8w  
> .data:0094E768 dword_94E768    dd 0                    ; DATA XREF: _dynamic_initializer_for__testDlg__staticMetaObject__+Dw  
> .data:0094E76C dword_94E76C    dd 0                    ; DATA XREF: _dynamic_initializer_for__testDlg__staticMetaObject__+17w  
> .data:0094E770 dword_94E770    dd 0                    ; DATA XREF: _dynamic_initializer_for__testDlg__staticMetaObject__+21w  
> .data:0094E774 dword_94E774    dd 0                    ; DATA XREF: _dynamic_initializer_for__testDlg__staticMetaObject__+2Bw  
> .data:0094E778 dword_94E778    dd 0                    ; DATA XREF: _dynamic_initializer_for__testDlg__staticMetaObject__+35w

  
[Asm] _纯文本查看_ _复制代码_

```
.text:00417500 _dynamic_initializer_for__testDlg__staticMetaObject__ proc near
.text:00417500                                        ; DATA XREF: .rdata:testDlg__staticMetaObject$initializer$o
.text:00417500                 push    ebp
.text:00417501                 mov     ebp, esp
.text:00417503                 mov     eax, ds:__imp_?staticMetaObject@QDialog@@2UQMetaObject@@B ; QMetaObject const QDialog::staticMetaObject
.text:00417508                 mov     ?staticMetaObject@testDlg@@2UQMetaObject@@B, eax ; QMetaObject const testDlg::staticMetaObject
.text:0041750D                 mov     dword_94E768, offset qt_meta_stringdata_testDlg
.text:00417517                 mov     dword_94E76C, offset qt_meta_data_testDlg
.text:00417521                 mov     dword_94E770, offset j_?qt_static_metacall@testDlg@@CAXPAVQObject@@W4Call@QMetaObject@@HPAPAX@Z ; testDlg::qt_static_metacall(QObject *,QMetaObject::Call,int,void * *)
.text:0041752B                 mov     dword_94E774, 0
.text:00417535                 mov     dword_94E778, 0
.text:0041753F                 pop     ebp
.text:00417540                 retn
.text:00417540 _dynamic_initializer_for__testDlg__staticMetaObject__ endp

```

以上 Qt5.5 +vs2010  debug 版用 PDB 发现的。debug 版上边的方法就不太好用了，不过很少有 debug 版的程序。  
Release 版倒是符合上述规律。  
获取可以加一个自动搜索 QMetaObject 结构体。  
感谢楼主的精彩分享。![](https://static.52pojie.cn/static/image/smiley/default/42.gif)  
![](https://avatar.52pojie.cn/images/noavatar_middle.gif)IWayne  目测这帖子是精华。。![](https://avatar.52pojie.cn/data/avatar/000/18/15/45_avatar_middle.jpg)zt185  学习了，教程很好！![](https://avatar.52pojie.cn/data/avatar/000/50/23/14_avatar_middle.jpg)雪莱鸟  厉害！ Qt 有些过于庞大，不然还继续用它 ![](https://avatar.52pojie.cn/data/avatar/000/48/13/40_avatar_middle.jpg) a2523188267 感谢分享，学习一下 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) Linkasyan 论坛 Qt 相关内容较少  感谢楼主分享 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) love5252 楼主有心了 不错 ![](https://avatar.52pojie.cn/data/avatar/000/16/85/30_avatar_middle.jpg) yhxing 牛.. 这东西应该必须收藏了  谢谢楼主的分享 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 035268 好贴，努力学习吧 ![](https://avatar.52pojie.cn/data/avatar/000/44/86/18_avatar_middle.jpg) wangqiustc 来学习学习了
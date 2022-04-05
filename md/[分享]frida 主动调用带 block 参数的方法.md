> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-272174.htm)

> [分享]frida 主动调用带 block 参数的方法

```
var handler = new ObjC.Block({
    retType: 'void',
    argTypes: ['object','object'],
    implementation: function (age,age2) {
        console.log('test')
        console.log(age)
        console.log(age2)
    }
  });
 
let lClass
let _methodName = "- generateTokenWithCompletionHandler:"
 
let dcclass = ObjC.classes[lClassName]['+ currentDevice']()
console.log(dcclass['- isSupported']())
dcclass[_methodName](handler)

```

[【公告】 讲师招募 | 全新 “预付费” 模式，不想来试试吗？](https://bbs.pediy.com/thread-271621.htm)

最后于 9 小时前 被 KeiVi233 编辑 ，原因：

[#HOOK 注入](forum-166-1-192.htm)
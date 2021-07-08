> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1471146-1-1.html)

> js 源代码：https://tak.jd.com/a/tr.js?_t=2704236 通过结构可以看出是 ob 混淆。

![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)beattortoise _ 本帖最后由 beattortoise 于 2021-7-6 10:38 编辑_  
js 源代码：https://tak.jd.com/a/tr.js?_t=2704236  
通过结构可以看出是 ob 混淆。（开头定义了一个大数组, 然后对这个大数组里的内容进行位移, 再定义一个解密函数。后面大部分的值都调用了这个解密函数, 以达到混淆的效果。这种代码即为 ob 混淆）  
  
[JavaScript] _纯文本查看_ _复制代码_

```
var a = ['RvXmb', 'cSsHp', 'WahbH', 'zuTaJ', 'ODElm', 'firstChild', 'uGvdZ', 'dXMBT', 'CftlR', 'illidan', '154463rpvXqY', 'MQuTx', 'HXDJt', 'wKhrk', 'hoHvR', 'rJuYL', 'jqfFx', 'async', 'pqpAE', 'Rhfgq', '\x20=\x20{};', 'imUlu', 'isPrototyp', 'AESgo', 'oawoZ', 'AeaHV', 'WzuzZ', 'IE7.0', 'gZUht', 'abcdefghij', 'WTMSe', 'OTHER', 'OfuXw', 'kjCTH', 'YhUyZ', 'SrOBF', '7|3|6', 'src', 'xplorer', 'vIkRh', 'uABQI', 'bhhXQ', 'CQoFf', 'Cbvxl', 'Tlgcs', 'gEvkv', 'fERop', 'VHYNu', 'gJcSo', 'yVmxl', 'NFjdG', 'tpjCD', 'GMJtT', 'n/x-www-fo', 'MpJQX', 'XavQm', 'daJOJ', '191490RkjscV', 'cAVRC', 'bnaKI', 'applicatio', 'IE9', 'MSIE7.0', 'Vlobn', 'dxbel', 'BhrXu', 'puKUP', 'XFsqL', '41061GCagrF', 'XquUH', 'IVZKD', 'MSIE8.0', 'VtQOM', 'XYzBC', 'yXUuj', 'tPERa', 'wYZNM', 'GET', 'toString', 'dyrQs', '1|3|0|4|7|', 'WXlXp', 'oHlDt', 'MtiFN', 'oVUIa', '_tak.callb', 'ioIgU', 'hwUej', 'EjPKi', 'LdWYK', 'object', 'createElem', 'WJmSa', 'replace', 'tlOjZ', 'dnUGW', 'length', 'CPAcc', 'HtLHw', 'eOf', '50483RiJcwG', 'fKooK', 'number', 'sYcXK', 'MFVWY', 'VzAYV', 'SAmxc', 'tdDpF', 'HoFeQ', 'TZinu', 'pDhAJ', 'benYW', 'vhbob', 'VvHos', 'gmi', 'feFfk', 'wQrgx', 'gbKuP', 'qtgTL', 'SpQhh', 'IE9.0', 'NWFDK', 'HiusG', '213639PnqezU', 'cwgid', 'new\x20', 'random', 'vMlKF', 'kctLz', 'Microsoft\x20', 'Oiads', 'FPntP', 'kfcrr', 'brexG', 'BtlFX', 'yLiAb', 'FJJxV', 'xdXXJ', '2|4|1|0|5|', 'type', 'xTsUF', 'KQNUJ', 'removeChil', 'getTakId', 'XMLHttpReq', '2uCfmMS', 'substr', 'cript', 'Szmpf', 'xVXDW', 'GpdbS', 'BtBiE', 'ukywW', 'Rixvz', 'klmnopqrsd', 'FdIoK', '/41CD2', 'aqkUv', 'IE8', 'EXGvw', 'NGOyM', 'tfSEj', 'dDvmJ', 'KjWKU', 'EFGHIJKLMN', 'd11qaz', 'xJTyX', 'iJhBT', 'jAWJQ', 'IE6', 'split', 'JHPVJ', 'aTFFX', 'eclOP', 'ZnLgO', 'Thdeu', 'jAlUf', 'Content-Ty', 'NjnDJ', 'QawdZ', 'dEGce', 'OEDlw', 'VBdRO', 'mExNU', 'jINjZ', 'timeout', 'getTime', 'k.jd.com/t', 'QmPmh', 'zLbir', 'gwpfX', 'getElement', 'cca', 'dykhM', 'MSIE6.0', 'WPLkl', 'UAmvY', 'IE7', '1kZbkDp', 'aOYVx', 'nujDB', 'SBIrF', 'push', 'open', '&callback=', 'jwuPA', 'gVBoi', 'RWiSk', 'parse', 'OXiXD', 'substring', 'skRZQ', 'fYgxD', 'hdaCG', 'VUeyA', 'Tk01qaz', '1|0|8|7', 'IXRjd', 'OFNPw', 'YDhBm', 'beforeSend', 'zbYOA', 'head', 'RyhPf', 'bUYDh', 'https://ta', 'neOct', 'MdAym', 'hCDBY', 'gUCtv', 'ydnaS', 'cWauJ', '26267LGVCWM', 'sAueq', 'Header', 'indexOf', 'RaMHq', 'ttgjF', 'KQQXN', 'OPQRSTUVWX', 'sKvNc', 'ycybF', 'responseTe', 'ZMXkK', '121259Sbaora', 'ovSPN', 'QSGUp', 'hWocX', 'djNeJ', 'YyVMi', 'tKrEf', 'eYCxh', 'lmDEo', 'JjkSH', '618qaz', 'fZfUh', 'parentNode', 'IE8.0', 'WdjNm', 'MNMqe', 'KVEhy', 'aRogw', 'VFnvP', 'yprDA', 'skyUR', 'slice', 'toLowerCas', 'iJvMh', 'ICcil', 'aXxOM', 'flTQE', 'Gwbdj', 'onreadysta', 'PqZmm', 'uest()', '11JHuOQn', 'Internet\x20E', 'ack', 'url', 'PfbEc', 'acLsR', 'dkjnA', 'PLwhf', 'VFCSY', 'kbOMB', 'charAt', 'IsdiX', 'FbSCx', 'unduK', 'wMaJw', 'cWKWW', 'JVBYT', 'qBAty', 'tials', 'mlBgU', 'nvdOv', 'urODW', 'sLylu', 'GiEVL', 'error', 'yqncp', 'RsmYE', 'Yoxge', 'RSurs', 'eSLwz', 'bNnIc', 'BmBaH', 'RkXvF', 'aYlEO', 'data', 'zZxTb', 'lvwPv', '8fHzvJA', 'fNewV', 'wcCpn', 'apply', 'KsSpD', 'CSnFR', 'OsPUT', 'CXGCX', 'callback', 'BPwUN', '0123456789', 'kmBra', 'xpUTP', 'text/javas', 'ntKft', 'gzPqx', 'appVersion', 'TNxjk', 'XFMrG', 'tiOTP', 'JhDkA', 'CnzDD', 'success', 'ikTFd', 'function\x20', 'EJTjg', 'lssfb', 'prototype', 'kQJrf', 'JbEdg', 'other', 'bmeLP', 'ent', 'script', 'insertBefo', 'oHpLT', 'Zfbje', '?_t=', 'drFYw', 'hzkKD', 'IcdjI', 'iJcXr', 'wUTdF', 'floor', 'appName', 'VWsuc', 'ieoWQ', 'VSlZK', 'Math', 'ZZHQA', 'window', 'snXee', 'vvLok', 'IVnjj', 'tdjiS', 'DYJgJ', 'irSDI', 'UQFef', 'xSRhb', 'aBipU', 'VTtSg', 'sByTagName', 'uYJBI', 'qWSmj', 'iJKoq', 'GzcWa', 'dandF', 'toUpperCas', 'replaceAll', 'tuYlG', 'abort', 'svVZw', 'Tlnmf', 'IE6.0', 'DXnqA', 'rm-urlenco', 'ded', 'qzysT', 'uduep', 'vBXyx', 'cbc', 'hyRvY', 'AgMUO', 'QjSsJ', 'nOUud', 'uvwxyzABCD', 'text'];
var b = function (c, d) {
    c = c - 0xa3;
    var e = a[c];
    return e;
};
var a4 = b;
(function (c, d) {
    var a3 = b;
    while (!![]) {
        try {
            var e = -parseInt(a3(0x13a)) * parseInt(a3(0xea)) + -parseInt(a3(0xc8)) * parseInt(a3(0x19b)) + parseInt(a3(0x1d5)) * parseInt(a3(0x222)) + parseInt(a3(0x1ca)) + -parseInt(a3(0x20c)) + -parseInt(a3(0xf6)) + -parseInt(a3(0x115)) * -parseInt(a3(0x1f5));
            if (e === d) break;
            else c['push'](c['shift']());
        } catch (f) {
            c['push'](c['shift']());
        }
    }
}(a, 0x1f994));

```

ob 混淆这块是解码区域。通过基本的 ast 树分析，可以将上面这块做为 function b 是解码函数。  
代码下面是 try{}catch{}, 里面大量用到 a4,a5,a8,b3,b4 的函数，通过上面定义的函数对大数组进行取值。  
第一步：对所有使用 function b 函数处理成 str 结果输出来。funToStr  
[JavaScript] _纯文本查看_ _复制代码_

```
String['prototype']["replaceAll"] = function (c, d, e) {
  var a5 = b,
      f = {
    'hWocX': function (g, h) {
      return g + h;
    },
    'NjnDJ': function (g, h) {
      return g !== h;
    },
    'EJTjg': 'GjCQo',
    'VSlZK': function (g, h) {
      return g !== h;
    },
    'lwSYh': "gEvkv",
    'oNezC': "gmi"
  };
 
  if (RegExp["prototype"]["isPrototyp" + "eOf"](c)) {
    if (f["NjnDJ"](f['EJTjg'], f["EJTjg"])) {
      function g() {
        var a6 = a5;
        NMZMaE["hWocX"](e['slice'](0xe, 0x13)['toLowerCas' + 'e'](), f['substring'](0x5, 0xf)["toUpperCas" + 'e']());
      }
    } else return this["replace"](c, d);
  } else {
    if (f["VSlZK"](f['lwSYh'], f['lwSYh'])) {
      function h() {
        var a7 = a5;
        NMZMaE["hWocX"](e["toUpperCas" + 'e'](), f["substring"](0x6, 0xa)['toUpperCas' + 'e']());
      }
    } else return this['replace'](new RegExp(c, e ? f['oNezC'] : 'gm'), d);
  }
},

```

第二步： 观察 if (f["NjnDJ"](f['EJTjg'], f["EJTjg"])) 将所有此类对象执行结果输出 str if ("gEvkv" !== "gEvkv") 。 callToStr  
  
以下就是通过上面两部做出的简单处理还原：  
[JavaScript] _纯文本查看_ _复制代码_

```
const parser = require("@babel/parser");
const template = require("@babel/template").default;
const traverse = require("@babel/traverse").default;
const t = require("@babel/types");
const generator = require("@babel/generator").default;
const path = require('path');
const fs = require('fs');
 
const {
    decryptStr,
    decryptStrFnName
} = require('./module');
 
fs.readFile(path.resolve(__dirname, './ob.js'), {
    "encoding": 'utf-8'
}, function (err, data) {
    const ast = parser.parse(data);
    step1(ast);
     
   step2(ast);
    // 将ast 转为 js
    let {
        code
    } = generator(ast);
    // code = code.replace(/!!\[\]/g, 'true').replace(/!\[\]/g, 'false');
  //  console.log(code);
 
    fs.writeFile('./try4.js', code, function (err) {
        if (err) {
            throw err;
        }
        // 写入成功后读取测试
        fs.readFile('./try4.js', 'utf-8', function (err, data) {
            if (err) {
                throw err;
            }
        });
    });
});
 
 
 
function step1(ast) {
    traverse(ast, {
        CallExpression: funToStr
    })
}
  
 
 
function step2(ast) {
    traverse(ast, {
        VariableDeclarator: callToStr 
    })
}
 
 
function funToStr(path) {
    var curNode = path.node; 
    var name = curNode.callee.name; 
 
    if ((name && name.length==2 && name.substr(0,1) =='a') ||(name && name.length==2 && name.substr(0,1) =='b')  && curNode.arguments.length === 1) {  //观察可知基本是a1-aa之类的，b1-bb之类的两位字符串，a或者b开头
        var strC = decryptStr(curNode.arguments[0].value);  
        path.replaceWith(t.stringLiteral(strC))
 
    }
}
  
 
function callToStr(path) {
    var node = path.node; 
 
    if (!t.isObjectExpression(node.init))
        return;
 
    var objPropertiesList = node.init.properties;  
 
    if (objPropertiesList.length==0)
        return;
 
    var objName = node.id.name; 
 
    objPropertiesList.forEach(prop => {
        var key = prop.key.value;
    
        if(!t.isStringLiteral(prop.value))   // 处理属性值类似是a=function
        {
  
            if(prop.value.body){
            var retStmt = prop.value.body.body[0]; 
            // 该path的最近父节点
            var fnPath = path.getFunctionParent(); 
            fnPath.traverse({
                CallExpression: function (_path) {
                    if (!t.isMemberExpression(_path.node.callee))
                        return;
 
                    var _node = _path.node.callee;
                    if (!t.isIdentifier(_node.object) || _node.object.name !== objName)
                        return;
                    if (!t.isStringLiteral(_node.property) || _node.property.value != key)
                        return;
 
                    var args = _path.node.arguments; 
 
                    // 二元运算
                    if (t.isBinaryExpression(retStmt.argument) && args.length===2)
                    { 
                        _path.replaceWith(t.binaryExpression(retStmt.argument.operator, args[0], args[1]));
                    }
                    // 逻辑运算
                    else if(t.isLogicalExpression(retStmt.argument) && args.length==2)
                    {
                        _path.replaceWith(t.logicalExpression(retStmt.argument.operator, args[0], args[1]));
                    }
                    // 函数调用
                    else if(t.isCallExpression(retStmt.argument) && t.isIdentifier(retStmt.argument.callee))
                    { 
                        _path.replaceWith(t.callExpression(args[0], args.slice(1)))
                    }
                } 
               })
              }
        }
        else{  // 处理属性值类似是a=1
             
            var retStmt = prop.value.value;
 
            // 该path的最近父节点
            var fnPath = path.getFunctionParent();
            fnPath.traverse({
                MemberExpression:function (_path) {
                    var _node = _path.node;
                    if (!t.isIdentifier(_node.object) || _node.object.name !== objName)
                        return;
                    if (!t.isStringLiteral(_node.property) || _node.property.value != key)
                        return;
 
                    _path.replaceWith(t.stringLiteral(retStmt))
                }
            })
 
        }
 
    });
 
    path.remove();
}

```

解码文件  module.js  
[JavaScript] _纯文本查看_ _复制代码_

```
var a = ['RvXmb', 'cSsHp', 'WahbH', 'zuTaJ', 'ODElm', 'firstChild', 'uGvdZ', 'dXMBT', 'CftlR', 'illidan', '154463rpvXqY', 'MQuTx', 'HXDJt', 'wKhrk', 'hoHvR', 'rJuYL', 'jqfFx', 'async', 'pqpAE', 'Rhfgq', '\x20=\x20{};', 'imUlu', 'isPrototyp', 'AESgo', 'oawoZ', 'AeaHV', 'WzuzZ', 'IE7.0', 'gZUht', 'abcdefghij', 'WTMSe', 'OTHER', 'OfuXw', 'kjCTH', 'YhUyZ', 'SrOBF', '7|3|6', 'src', 'xplorer', 'vIkRh', 'uABQI', 'bhhXQ', 'CQoFf', 'Cbvxl', 'Tlgcs', 'gEvkv', 'fERop', 'VHYNu', 'gJcSo', 'yVmxl', 'NFjdG', 'tpjCD', 'GMJtT', 'n/x-www-fo', 'MpJQX', 'XavQm', 'daJOJ', '191490RkjscV', 'cAVRC', 'bnaKI', 'applicatio', 'IE9', 'MSIE7.0', 'Vlobn', 'dxbel', 'BhrXu', 'puKUP', 'XFsqL', '41061GCagrF', 'XquUH', 'IVZKD', 'MSIE8.0', 'VtQOM', 'XYzBC', 'yXUuj', 'tPERa', 'wYZNM', 'GET', 'toString', 'dyrQs', '1|3|0|4|7|', 'WXlXp', 'oHlDt', 'MtiFN', 'oVUIa', '_tak.callb', 'ioIgU', 'hwUej', 'EjPKi', 'LdWYK', 'object', 'createElem', 'WJmSa', 'replace', 'tlOjZ', 'dnUGW', 'length', 'CPAcc', 'HtLHw', 'eOf', '50483RiJcwG', 'fKooK', 'number', 'sYcXK', 'MFVWY', 'VzAYV', 'SAmxc', 'tdDpF', 'HoFeQ', 'TZinu', 'pDhAJ', 'benYW', 'vhbob', 'VvHos', 'gmi', 'feFfk', 'wQrgx', 'gbKuP', 'qtgTL', 'SpQhh', 'IE9.0', 'NWFDK', 'HiusG', '213639PnqezU', 'cwgid', 'new\x20', 'random', 'vMlKF', 'kctLz', 'Microsoft\x20', 'Oiads', 'FPntP', 'kfcrr', 'brexG', 'BtlFX', 'yLiAb', 'FJJxV', 'xdXXJ', '2|4|1|0|5|', 'type', 'xTsUF', 'KQNUJ', 'removeChil', 'getTakId', 'XMLHttpReq', '2uCfmMS', 'substr', 'cript', 'Szmpf', 'xVXDW', 'GpdbS', 'BtBiE', 'ukywW', 'Rixvz', 'klmnopqrsd', 'FdIoK', '/41CD2', 'aqkUv', 'IE8', 'EXGvw', 'NGOyM', 'tfSEj', 'dDvmJ', 'KjWKU', 'EFGHIJKLMN', 'd11qaz', 'xJTyX', 'iJhBT', 'jAWJQ', 'IE6', 'split', 'JHPVJ', 'aTFFX', 'eclOP', 'ZnLgO', 'Thdeu', 'jAlUf', 'Content-Ty', 'NjnDJ', 'QawdZ', 'dEGce', 'OEDlw', 'VBdRO', 'mExNU', 'jINjZ', 'timeout', 'getTime', 'k.jd.com/t', 'QmPmh', 'zLbir', 'gwpfX', 'getElement', 'cca', 'dykhM', 'MSIE6.0', 'WPLkl', 'UAmvY', 'IE7', '1kZbkDp', 'aOYVx', 'nujDB', 'SBIrF', 'push', 'open', '&callback=', 'jwuPA', 'gVBoi', 'RWiSk', 'parse', 'OXiXD', 'substring', 'skRZQ', 'fYgxD', 'hdaCG', 'VUeyA', 'Tk01qaz', '1|0|8|7', 'IXRjd', 'OFNPw', 'YDhBm', 'beforeSend', 'zbYOA', 'head', 'RyhPf', 'bUYDh', 'https://ta', 'neOct', 'MdAym', 'hCDBY', 'gUCtv', 'ydnaS', 'cWauJ', '26267LGVCWM', 'sAueq', 'Header', 'indexOf', 'RaMHq', 'ttgjF', 'KQQXN', 'OPQRSTUVWX', 'sKvNc', 'ycybF', 'responseTe', 'ZMXkK', '121259Sbaora', 'ovSPN', 'QSGUp', 'hWocX', 'djNeJ', 'YyVMi', 'tKrEf', 'eYCxh', 'lmDEo', 'JjkSH', '618qaz', 'fZfUh', 'parentNode', 'IE8.0', 'WdjNm', 'MNMqe', 'KVEhy', 'aRogw', 'VFnvP', 'yprDA', 'skyUR', 'slice', 'toLowerCas', 'iJvMh', 'ICcil', 'aXxOM', 'flTQE', 'Gwbdj', 'onreadysta', 'PqZmm', 'uest()', '11JHuOQn', 'Internet\x20E', 'ack', 'url', 'PfbEc', 'acLsR', 'dkjnA', 'PLwhf', 'VFCSY', 'kbOMB', 'charAt', 'IsdiX', 'FbSCx', 'unduK', 'wMaJw', 'cWKWW', 'JVBYT', 'qBAty', 'tials', 'mlBgU', 'nvdOv', 'urODW', 'sLylu', 'GiEVL', 'error', 'yqncp', 'RsmYE', 'Yoxge', 'RSurs', 'eSLwz', 'bNnIc', 'BmBaH', 'RkXvF', 'aYlEO', 'data', 'zZxTb', 'lvwPv', '8fHzvJA', 'fNewV', 'wcCpn', 'apply', 'KsSpD', 'CSnFR', 'OsPUT', 'CXGCX', 'callback', 'BPwUN', '0123456789', 'kmBra', 'xpUTP', 'text/javas', 'ntKft', 'gzPqx', 'appVersion', 'TNxjk', 'XFMrG', 'tiOTP', 'JhDkA', 'CnzDD', 'success', 'ikTFd', 'function\x20', 'EJTjg', 'lssfb', 'prototype', 'kQJrf', 'JbEdg', 'other', 'bmeLP', 'ent', 'script', 'insertBefo', 'oHpLT', 'Zfbje', '?_t=', 'drFYw', 'hzkKD', 'IcdjI', 'iJcXr', 'wUTdF', 'floor', 'appName', 'VWsuc', 'ieoWQ', 'VSlZK', 'Math', 'ZZHQA', 'window', 'snXee', 'vvLok', 'IVnjj', 'tdjiS', 'DYJgJ', 'irSDI', 'UQFef', 'xSRhb', 'aBipU', 'VTtSg', 'sByTagName', 'uYJBI', 'qWSmj', 'iJKoq', 'GzcWa', 'dandF', 'toUpperCas', 'replaceAll', 'tuYlG', 'abort', 'svVZw', 'Tlnmf', 'IE6.0', 'DXnqA', 'rm-urlenco', 'ded', 'qzysT', 'uduep', 'vBXyx', 'cbc', 'hyRvY', 'AgMUO', 'QjSsJ', 'nOUud', 'uvwxyzABCD', 'text'];
var b = function (c, d) {
    c = c - 0xa3;
    var e = a[c];
    return e;
};
 
(function (c, d) {
 
    while (!![]) {
        try {
            var e = -parseInt(b(0x13a)) * parseInt(b(0xea)) + -parseInt(b(0xc8)) * parseInt(b(0x19b)) + parseInt(b(0x1d5)) * parseInt(b(0x222)) + parseInt(b(0x1ca)) + -parseInt(b(0x20c)) + -parseInt(b(0xf6)) + -parseInt(b(0x115)) * -parseInt(b(0x1f5));
            if (e === d) break;
            else c['push'](c['shift']());
        } catch (f) {
            c['push'](c['shift']());
        }
    }
}(a, 0x1f994)); 
exports.decryptStr = b;

```

最终输出结果是：try4.js  
[JavaScript] _纯文本查看_ _复制代码_

```
String['prototype']["replaceAll"] = function (c, d, e) {
  var a5 = b;
 
  if (RegExp["prototype"]["isPrototyp" + "eOf"](c)) {
    if ("GjCQo" !== "GjCQo") {
      function g() {
        var a6 = a5;
        NMZMaE["hWocX"](e['slice'](0xe, 0x13)['toLowerCas' + 'e'](), f['substring'](0x5, 0xf)["toUpperCas" + 'e']());
      }
    } else return this["replace"](c, d);
  } else {
    if ("gEvkv" !== "gEvkv") {
      function h() {
        var a7 = a5;
        NMZMaE["hWocX"](e["toUpperCas" + 'e'](), f["substring"](0x6, 0xa)['toUpperCas' + 'e']());
      }
    } else return this['replace'](new RegExp(c, e ? "gmi" : 'gm'), d);
  }
}, function () {
  var a8 = b;
  _tak = _tak || {};
 
  var e = "https://ta" + "k.jd.com/t" + "/41CD2",
      f = eval,
      g = f('setTimeout'),
      h = f("window"),
      i = eval('var a9 = a8, L = {\n        \'hyRvY\': function (M, N) {\n            return c[\'wNwYZ\'](M, N);\n        }\n    };if (\'eRikm\' !== a9(352))\n    clearTimeout;\nelse {\n    function M() {\n        var aa = a9;\n        dClqiO[aa(395)](e[\'toLowerCas\' + \'e\']()[aa(212)](6, 19), f[\'substring\'](5, 11));\n    }\n}'),
      j = f("Math"),
      k = function (L) {
    var ab = a8;
 
    if ("MgzBH" === "VHYNu") {
      function N() {
        var ac = ab;
        e(f["url"] + ("&callback=" + "_tak.callb" + 'ack'));
      }
    } else {
      if (typeof L == "number") {
        if ("xVXDW" === "xVXDW") return L;else {
          function O() {
            var ad = ab;
            return this["replace"](new g(h, i ? "gmi" : 'gm'), j);
          }
        }
      }
 
      throw new Error('error');
    }
  };
 
  function m() {
    var ae = a8;
 
    if ("SrOBF" !== "SrOBF") {
      function M() {
        var af = ae,
            N = E["toString"]();
        return N = N['substr']("function "["length"]), N = N["substr"](0x0, N["indexOf"]('(')), N;
      }
    } else {
      if (navigator["appName"] == c["dandF"] && navigator["appVersion"]["split"](';')[0x1]["replace"](/[ ]/g, '') == "MSIE6.0") {
        if ("WahbH" !== "RHDXy") return "IE6.0";else {
          function N() {
            var ag = ae;
            return e('new\x20' + f);
          }
        }
      } else {
        if (navigator["appName"] == "Microsoft " + 'Internet\x20E' + "xplorer" && navigator['appVersion']["split"](';')[0x1]['replace'](/[ ]/g, '') == "MSIE7.0") {
          if ("YhUyZ" !== "snXee") return "IE7.0";else {
            function O() {
              var ah = ae;
              e["substring"](0x5, 0x8) + f['replace'](/a/gi, 'c');
            }
          }
        } else {
          if (navigator["appName"] == "Microsoft " + "Internet E" + "xplorer" && navigator['appVersion']["split"](';')[0x1]["replace"](/[ ]/g, '') == "MSIE8.0") {
            if ("tpjCD" !== "tpjCD") {
              function P() {
                if (typeof f == "number") return i;
                throw new h('error');
              }
            } else return "IE8.0";
          } else {
            if (navigator['appName'] == c["dandF"] && navigator['appVersion']["split"](';')[0x1]["replace"](/[ ]/g, '') == "MSIE9.0") {
              if ("JjkSH" === "BmBaH") {
                function Q() {
                  var ai = ae,
                      R = [];
 
                  for (var S in i["data"]) {
                    R["push"](o(p["data"][S]));
                  }
 
                  l = m(n['apply'](this, R));
                }
              } else return "IE9.0";
            }
          }
        }
      }
 
      return "other";
    }
  }
 
  var n = m();
 
  function o() {
    var aj = a8;
 
    if ('qtgTL' !== "qtgTL") {
      function L() {
        var ak = aj;
        p = q(r);
 
        if (s(t) && u(v['data'])) {
          var M = [];
 
          for (var N in C["data"]) {
            M["push"](I(J['data'][N]));
          }
 
          F = G(H["apply"](this, M));
        }
      }
    } else return n == "IE8.0" || n == "IE9.0";
  }
 
  function q(L) {
    var am = a8;
 
    if ("jAWJQ" === "EXGvw") {
      function O() {
        var al = b;
        vDtEid["ikTFd"](e["substring"](0xa, 0x12), f['toLowerCas' + 'e']()["substring"](0x2, 0xd));
      }
    } else {
      var M = {};
 
      for (var N in p) {
        M[N] = L[N] == undefined ? p[N] : L[N];
      }
 
      return M;
    }
  }
 
  var r = '';
 
  function s(L) {
    var an = a8,
        M = c['lzTMZ']['split']('|'),
        N = 0x0;
 
    while (!![]) {
      switch (M[N++]) {
        case '0':
          O["src"] = L + '';
          continue;
 
        case '1':
          O['async'] = !![];
          continue;
 
        case '2':
          var O = document['createElem' + "ent"]("script");
          continue;
 
        case '3':
          P["parentNode"]["insertBefo" + 're'](O, P);
          continue;
 
        case '4':
          O['type'] = c["aqkUv"];
          continue;
 
        case '5':
          var P = document["getElement" + 'sByTagName']("script")[0x0];
          continue;
      }
 
      break;
    }
  }
 
  _tak["callback"] = function () {
    var ap = a8,
        M = arguments[0x0];
 
    try {
      if ("bnaKI" !== "aPSAi") {
        if (I(M)) {
          if ("SpQhh" !== "eYCxh") {
            var N = [];
 
            for (var O in M) {
              N["push"](I(M[O]));
            }
 
            r = I(F['apply'](this, N));
          } else {
            function P() {
              var as = ap;
 
              if (L["unduK"](typeof g, "object")) {
                var Q = '';
 
                for (var R in k) {
                  Q += L["skyUR"](L["nujDB"](L['nujDB'](R, '='), m[R]), '&');
                }
 
                return Q = Q["substring"](0x0, Q["length"] - 0x1), Q;
              } else return n;
            }
          }
        }
      } else {
        function Q() {
          var R = {};
 
          for (var S in h) {
            R[S] = m[S] == n ? o[S] : p[S];
          }
 
          return R;
        }
      }
    } catch (R) {}
  };
 
  function t(L) {
    var at = a8;
 
    if ("tKrEf" === "tKrEf") {
      var M = L["toString"]();
      return M = M["substr"]("function "["length"]), M = M["substr"](0x0, M["indexOf"]('(')), M;
    } else {
      function N() {
        var au = at;
        vDtEid["Rhfgq"](e["substring"](0x5, 0xe), f['substring'](0x2, 0xd)['toUpperCas' + 'e']());
      }
    }
  }
 
  function u() {
    var ay = a8,
        M = arguments[0x0],
        N = q(M);
    N["beforeSend"]();
    var O = y();
 
    if (o()) {
      if ("uYJBI" === "uYJBI") g(function () {
        var az = ay;
 
        if ("OXiXD" === "daJOJ") {
          function Q() {
            e['getTakId'] = f;
          }
        } else s(N["url"] + c['vpsMG']);
      });else {
        function Q() {
          var aA = ay;
          return h(function () {
            L['YGpxE'](m);
          }, 0x64), j ? k : L['YGpxE'](l);
        }
      }
    } else {
      if ("qUAFJ" === "qUAFJ") {
        O["open"](N["type"], N['url'], N["async"]), O['withCreden' + "tials"] = !![], O['setRequest' + "Header"](c["ZMXkK"], N['contentTyp' + 'e']);
        var P = null;
 
        if (!N['async'] && N["timeout"] > 0x0) {
          if ("QawdZ" === "kkTYY") {
            function R() {
              var aC = ay;
              h = i(function () {
                var aB = b;
                m["abort"](), n["error"]();
              }, l["timeout"]);
            }
          } else P = g(function () {
            var aD = ay;
            if ("RSurs" !== "TNxjk") O["abort"](), N['error']();else {
              function S() {
                var aE = aD;
                return this["replace"](e, f);
              }
            }
          }, N["timeout"]);
        }
 
        O["onreadysta" + 'techange'] = function () {
          var aG = ay;
 
          if ("pFMKw" === "pFMKw") {
            if (O['readyState'] == 0x4) {
              if ("IcdjI" === "vDwus") {
                function T() {
                  return E;
                }
              } else {
                try {
                  if ("Cbvxl" !== "OfuXw") {
                    if (P) {
                      if ("pDhAJ" === "tiOTP") {
                        function U() {
                          var aH = aG;
                          g['push'](S["dykhM"](h, i[j]));
                        }
                      } else i(P), P = null;
                    }
                  } else {
                    function V() {
                      var aI = aG;
                      return f["floor"](g["random"]() * h);
                    }
                  }
                } catch (W) {}
 
                if (O['status'] == 0xc8) {
                  if ("MFVWY" !== "MFVWY") {
                    function X() {
                      var aJ = aG;
                      e['toUpperCas' + 'e']()['substring'](0x3, 0xd) + f["toLowerCas" + 'e']()["substring"](0xa, 0x13);
                    }
                  } else N["success"](O["responseTe" + 'xt']);
                } else {
                  if ("pbuaB" === "TZinu") {
                    function Y() {
                      var aK = aG;
                      return L["aTFFX"](h, i), j ? k : L["aTFFX"](l, "Tk01qaz");
                    }
                  } else N["error"]();
                }
              }
            }
          } else {
            function Z() {
              var aL = aG,
                  a0 = [];
 
              for (var a1 in i) {
                a0["push"](o(p[a1]));
              }
 
              l = L['FZZtm'](m, n['apply'](this, a0));
            }
          }
        }, O['send'](z(N["data"]));
      } else {
        function S() {
          var aM = ay;
          E["error"]();
        }
      }
    }
  }
 
  function v(L) {
    var aN = a8;
    if ("oHlDt" !== "tuYlG") return f(L) != undefined;else {
      function M() {
        var aO = aN;
 
        try {
          return g["parse"](h);
        } catch (N) {}
 
        return {};
      }
    }
  }
 
  function x(L) {
    var aP = a8;
 
    if ("gZUht" === "FdIoK") {
      function M() {
        var aQ = aP;
        return g == h["IE8"] || i == j["IE9"];
      }
    } else return f("new " + L);
  }
 
  function y() {
    var aR = a8;
    if ("sKvNc" !== "fZfUh") return x(c["QjSsJ"]);else {
      function L() {
        var aS = aR;
 
        if (k(l)) {
          var M = [];
 
          for (var N in s) {
            M["push"](y(z[N]));
          }
 
          v = D(x["apply"](this, M));
        }
      }
    }
  }
 
  function z(L) {
    var aT = a8;
 
    if ("bNnIc" === "oAYcQ") {
      function O() {
        var aU = aT;
        g && (k(l), m = null);
      }
    } else {
      if (typeof L === "object") {
        if ("IsdiX" === "IsdiX") {
          var M = '';
 
          for (var N in L) {
            if ("FJJxV" === "FJJxV") M += N + '=' + L[N] + '&';else {
              function P() {
                var aV = aT;
                g["push"](h(i["data"][j]));
              }
            }
          }
 
          return M = M['substring'](0x0, M['length'] - 0x1), M;
        } else {
          function Q() {
            var aW = aT;
            e["abort"](), f["error"]();
          }
        }
      } else {
        if ('hzkKD' !== "hzkKD") {
          function R() {
            var aX = aT;
            E();
          }
        } else return L;
      }
    }
  }
 
  var A = c["WzuzZ"];
 
  function B(L) {
    var aY = a8;
    if ("AgMUO" !== "NFjdG") return j["floor"](j['random']() * L);else {
      function M() {
        var aZ = aY;
 
        try {
          if (typeof k !== "object") return ![];
          var N = "illidan" + (new l() - 0x0),
              O = O["createElem" + "ent"]("script"),
              P = O['getElement' + "sByTagName"]("head")[0x0];
          return P["insertBefo" + 're'](O, P["firstChild"]), O['text'] = N + " = {};", P["removeChil" + 'd'](O), m[N] === n[N];
        } catch (Q) {
          return ![];
        }
      }
    }
  }
 
  function C() {
    var b0 = a8;
 
    if ('HtIit' === "SAmxc") {
      function O() {
        var b1 = b0;
        return E["IE7"];
      }
    } else {
      var L = '';
 
      for (var M = 0x0; M < 0x7; M++) {
        if ("uxihA" === 'uxihA') {
          var N = B(A["length"]);
          L = L + A["charAt"](N) + N;
        } else {
          function P() {
            return e['parse'](f);
          }
        }
      }
 
      return L;
    }
  }
 
  var D = window,
      E = document;
 
  function F() {
    var b3 = a8;
 
    if ("wKhrk" !== "wKhrk") {
      function U() {
        var b6 = b3;
        return E["IE9"];
      }
    } else try {
      if ("vvLok" !== "vvLok") {
        function V() {
          var b7 = b3;
          f(function () {
            var b8 = b7;
            i(j["url"] + W['IWTxg']);
          });
        }
      } else {
        var M = arguments,
            N = M[0x0],
            O = M[0x1],
            P = M[0x2],
            Q = M[0x3],
            R = M[0x4],
            S = M[0x5],
            T = '';
        if (N == "cca") G(D) ? T = eval('var bb = b3, W = {\n        \'xSRhb\': function (X, Y) {\n            return X < Y;\n        },\n        \'Qbewl\': function (X, Y) {\n            var b9 = b;\n            return L[b9(489)](X, Y);\n        },\n        \'ZnLgO\': function (X, Y) {\n            return X + Y;\n        },\n        \'nOUud\': function (X, Y) {\n            var ba = b;\n            return L[ba(226)](X, Y);\n        }\n    };if (L[bb(534)](L[\'ovSPN\'], L[bb(247)])) {\n    function X() {\n        var bc = bb, Y = \'\';\n        for (var Z = 0; W[bc(372)](Z, 7); Z++) {\n            var a0 = W[\'Qbewl\'](i, j[bc(497)]);\n            Y = W[bc(176)](W[bc(398)](Y, k[bc(287)](a0)), a0);\n        }\n        return Y;\n    }\n} else\n    L[bb(475)](P[bb(267)](14, 19)[\'toLowerCas\' + \'e\'](), O[\'substring\'](5, 15)[bb(381) + \'e\']());') : '';
        if (N == 'ab') G(D) ? T = eval('var bd = b3;c[bd(201)](R[\'substring\'](10, 18), S[bd(268) + \'e\']()[\'substring\'](2, 13));') : '';
        if (N == 'ch') G(D) ? T = eval('var be = b3;if (c[be(313)] === c[be(350)]) {\n    function W() {\n        var bf = be;\n        return k[\'prototype\'][bf(423) + \'eOf\'](l) ? this[bf(494)](s, N) : this[\'replace\'](new u(v, D ? L[bf(473)] : \'gm\'), x);\n    }\n} else\n    Q[\'toUpperCas\' + \'e\']() + R[\'substring\'](6, 10)[be(381) + \'e\']();') : '';
        if (N == "cbc") G(D) ? T = eval('var bg = b3;if (c[bg(541)](c[bg(369)], c[bg(476)]))\n    c[bg(254)](Q[\'toUpperCas\' + \'e\']()[bg(212)](3, 13), P[bg(268) + \'e\']()[bg(212)](10, 19));\nelse {\n    function W() {\n        var bh = bg;\n        e[bh(336)](f[bh(244) + \'xt\']);\n    }\n}') : '';
        if (N == 'by') G(D) ? T = eval('var bi = b3;if (L[\'brexG\'](L[bi(401)], L[bi(506)]))\n    O[bi(212)](5, 8) + P[\'replace\'](/a/gi, \'c\');\nelse {\n    function W() {\n        var bj = bi;\n        return E[bj(171)];\n    }\n}') : '';
        if (N == 'xa') G(D) ? T = eval('var bl = b3, W = {\n        \'GMJtT\': function (X, Y) {\n            return X + Y;\n        },\n        \'cWKWW\': function (X, Y) {\n            return c[\'xmZfy\'](X, Y);\n        },\n        \'kqbCQ\': function (X, Y) {\n            var bk = b;\n            return c[bk(359)](X, Y);\n        }\n    };if (c[bl(178)] === c[\'jAlUf\'])\n    c[bl(408)](O[\'substring\'](1, 16), S[\'slice\'](4, 10));\nelse {\n    function X() {\n        var bm = bl, Y = \'\';\n        for (var Z in e) {\n            Y += W[bm(453)](W[bm(453)](W[bm(292)](Z, \'=\'), g[Z]), \'&\');\n        }\n        return Y = Y[bm(212)](0, W[\'kqbCQ\'](Y[bm(497)], 1)), Y;\n    }\n}') : '';
        if (N == 'cza') G(D) ? T = eval('var bo = b3, W = {\n        \'Yoxge\': function (X, Y) {\n            var bn = b;\n            return L[bn(534)](X, Y);\n        },\n        \'qBAty\': L[bo(415)],\n        \'HoFeQ\': function (X, Y) {\n            return X(Y);\n        },\n        \'fERop\': L[bo(532)],\n        \'oNoDS\': function (X, Y) {\n            var bp = bo;\n            return L[bp(475)](X, Y);\n        },\n        \'XquUH\': L[bo(342)],\n        \'HKVSt\': function (X, Y) {\n            return X - Y;\n        },\n        \'oVUIa\': L[bo(173)],\n        \'bhhXQ\': L[bo(431)],\n        \'xdXXJ\': function (X, Y) {\n            var bq = bo;\n            return L[bq(392)](X, Y);\n        },\n        \'vkqBC\': L[bo(228)],\n        \'svVZw\': function (X, Y) {\n            var br = bo;\n            return L[br(233)](X, Y);\n        },\n        \'nQRQO\': L[bo(518)]\n    };if (L[bo(169)] === bo(282))\n    Q[bo(268) + \'e\']()[bo(212)](6, 19) + S[bo(212)](5, 11);\nelse {\n    function X() {\n        var bs = bo;\n        if (W[bs(304)](typeof i, W[bs(294)]))\n            return W[bs(509)](j, W[bs(447)]);\n        var Y = W[\'oNoDS\'](W[bs(470)], W[\'HKVSt\'](new k(), 0)), Z = Z[bs(492) + \'ent\'](W[bs(485)]), a0 = Z[bs(193) + bs(375)](W[bs(442)])[0];\n        a0[bs(348) + \'re\'](Z, a0[\'firstChild\']), Z[bs(400)] = W[bs(538)](Y, W[\'vkqBC\']), a0[\'removeChil\' + \'d\'](Z);\n        if (W[bs(385)](l[Y], m[Y]))\n            return;\n        return W[\'HoFeQ\'](n, W[\'nQRQO\']);\n    }\n}') : '';
        if (N == 'cb') G(D) ? T = eval('var bt = b3;if (c[\'wweeQ\'](c[bt(456)], c[\'XavQm\'])) {\n    function W() {\n        var bu = bt;\n        e[bu(212)](1, 16) + f[bu(267)](4, 10);\n    }\n} else\n    c[\'dXMBT\'](S[bt(212)](5, 14), P[bt(212)](2, 13)[bt(381) + \'e\']());') : '';
        return T;
      }
    } catch (W) {
      if ("wUTdF" === "gzPqx") {
        function X() {
          d;
        }
      } else return T;
    }
  }
 
  function G(L) {
    var bw = a8;
    if ('iCXja' !== "LdWYK") try {
      if ("flTQE" === "cwgid") {
        function S() {
          var bx = bw;
          return E["IE8"];
        }
      } else {
        var N = ("1|3|0|4|7|" + '6|2|5')["split"]('|'),
            O = 0x0;
 
        while (!![]) {
          switch (N[O++]) {
            case '0':
              var P = E['createElem' + "ent"]("script");
              continue;
 
            case '1':
              if (typeof L !== "object") return ![];
              continue;
 
            case '2':
              R['removeChil' + 'd'](P);
              continue;
 
            case '3':
              var Q = 'illidan' + (new Date() - 0x0);
              continue;
 
            case '4':
              var R = E['getElement' + "sByTagName"]("head")[0x0];
              continue;
 
            case '5':
              return D[Q] === L[Q];
 
            case '6':
              P["text"] = Q + " = {};";
              continue;
 
            case '7':
              R["insertBefo" + 're'](P, R["firstChild"]);
              continue;
          }
 
          break;
        }
      }
    } catch (T) {
      if ("XFMrG" === "XFMrG") return ![];else {
        function U() {
          var by = bw;
          return M['ghrEK'](E, "XMLHttpReq" + "uest()");
        }
      }
    } else {
      function V() {
        return E;
      }
    }
  }
 
  function H(L) {
    var bA = a8;
 
    if ("irSDI" === "irSDI") {
      try {
        if ("skRZQ" === "skRZQ") return JSON['parse'](L);else {
          function N() {
            var bC = bA,
                O = f["createElem" + "ent"]("script");
            O["type"] = "text/javas" + "cript", O["async"] = !![], O["src"] = M['gwpfX'](g, '');
            var P = h["getElement" + 'sByTagName']("script")[0x0];
            P["parentNode"]["insertBefo" + 're'](O, P);
          }
        }
      } catch (O) {}
 
      return {};
    } else {
      function P() {
        var bD = bA,
            Q = M['QIlwE']['split']('|'),
            R = 0x0;
 
        while (!![]) {
          switch (Q[R++]) {
            case '0':
              var S = U["getElement" + "sByTagName"]("head")[0x0];
              continue;
 
            case '1':
              var T = M["gwpfX"]("illidan", new h() - 0x0);
              continue;
 
            case '2':
              var U = U["createElem" + "ent"]("script");
              continue;
 
            case '3':
              S["removeChil" + 'd'](U);
              continue;
 
            case '4':
              S["insertBefo" + 're'](U, S['firstChild']);
              continue;
 
            case '5':
              if (M["VBdRO"](typeof g, "object")) return ![];
              continue;
 
            case '6':
              return i[T] === j[T];
 
            case '7':
              U["text"] = T + " = {};";
              continue;
          }
 
          break;
        }
      }
    }
  }
 
  function I(L) {
    var bE = a8;
 
    if ("dnUGW" !== "dnUGW") {
      function M() {
        var bF = bE;
        l[m] = n[o] == p ? q[r] : s[t];
      }
    } else return J(h), L ? L : k("Tk01qaz");
  }
 
  function J(L) {
    var bH = a8;
 
    if ("PfbEc" === "cSsHp") {
      function S() {
        var bI = bH;
        j(), k["getTakId"] = function () {
          var bJ = bI;
          return q(function () {
            var bK = bJ;
            v();
          }, 0x64), s ? t : M["NWFDK"](u);
        };
      }
    } else {
      var N = c["PLwhf"]["split"]('|'),
          O = 0x0;
 
      while (!![]) {
        switch (N[O++]) {
          case '0':
            Q["removeChil" + 'd'](P);
            continue;
 
          case '1':
            P['text'] = R + " = {};";
            continue;
 
          case '2':
            Q['insertBefo' + 're'](P, Q["firstChild"]);
            continue;
 
          case '3':
            if (typeof L !== "object") return k("618qaz");
            continue;
 
          case '4':
            var P = E["createElem" + 'ent']("script");
            continue;
 
          case '5':
            var Q = E['getElement' + "sByTagName"]("head")[0x0];
            continue;
 
          case '6':
            var R = "illidan" + (new Date() - 0x0);
            continue;
 
          case '7':
            return k("d11qaz");
 
          case '8':
            if (D[R] === L[R]) return;
            continue;
        }
 
        break;
      }
    }
  }
 
  var K = function () {
    var bN = a8;
 
    if ("Oiads" === "Oiads") {
      var M = '';
 
      try {
        var N = {};
        N["url"] = I(e) + "?_t=" + new Date()["getTime"](), N["async"] = ![], N["timeout"] = 0x12c, N["success"] = function (O) {
          var bQ = bN;
          if ("MNMqe" === "MNMqe") try {
            if ("MtiFN" === "tfSEj") {
              function R() {
                return E;
              }
            } else {
              O = L["UQFef"](H, O);
 
              if (I(O) && L['VVkJk'](I, O["data"])) {
                var P = [];
 
                for (var Q in O['data']) {
                  if ("Szmpf" !== "MQuTx") P['push'](L["dEGce"](I, O["data"][Q]));else {
                    function S() {
                      var bR = bQ,
                          T = L["UQFef"](h, i["length"]);
                      j = L["AeaHV"](L["AeaHV"](k, l["charAt"](T)), T);
                    }
                  }
                }
 
                M = L['dEGce'](I, F["apply"](this, P));
              }
            }
          } catch (T) {} else {
            function U() {
              f(g), h = null;
            }
          }
        }, u(I(N));
      } catch (O) {}
 
      if (!M) {
        if ("QHBzj" === "QHBzj") M = C();else {
          function P() {
            var bS = bN;
            return f(g) != h;
          }
        }
      }
 
      return I(M);
    } else {
      function Q() {
        var bT = bN;
        g += L["AeaHV"](h + '=', i[j]) + '&';
      }
    }
  };
 
  if (o()) K(), _tak['getTakId'] = function () {
    var bU = a8;
    return g(function () {
      var bV = bU;
      if ("GpdbS" === "GpdbS") K();else {
        function L() {
          e = f();
        }
      }
    }, 0x64), r ? r : C();
  };else {
    if ("FbSCx" !== "vbVPm") _tak["getTakId"] = K;else {
      function L() {
        return ![];
      }
    }
  }
}();

```

可以明显看出 tk 是怎么生成的  
[JavaScript] _纯文本查看_ _复制代码_

```
if (N == "cca") G(D) ? T = eval('var bb = b3, W = {\n \'xSRhb\': function (X, Y) {\n return X < Y;\n },\n \'Qbewl\': function (X, Y) {\n var b9 = b;\n return L[b9(489)](X, Y);\n },\n \'ZnLgO\': function (X, Y) {\n return X + Y;\n },\n \'nOUud\': function (X, Y) {\n var ba = b;\n return L[ba(226)](X, Y);\n }\n };if (L[bb(534)](L[\'ovSPN\'], L[bb(247)])) {\n function X() {\n var bc = bb, Y = \'\';\n for (var Z = 0; W[bc(372)](Z, 7); Z++) {\n var a0 = W[\'Qbewl\'](i, j[bc(497)]);\n Y = W[bc(176)](W[bc(398)](Y, k[bc(287)](a0)), a0);\n }\n return Y;\n }\n} else\n L[bb(475)](P[bb(267)](14, 19)[\'toLowerCas\' + \'e\'](), O[\'substring\'](5, 15)[bb(381) + \'e\']());') : '';
if (N == 'ab') G(D) ? T = eval('var bd = b3;c[bd(201)](R[\'substring\'](10, 18), S[bd(268) + \'e\']()[\'substring\'](2, 13));') : '';
if (N == 'ch') G(D) ? T = eval('var be = b3;if (c[be(313)] === c[be(350)]) {\n function W() {\n var bf = be;\n return k[\'prototype\'][bf(423) + \'eOf\'](l) ? this[bf(494)](s, N) : this[\'replace\'](new u(v, D ? L[bf(473)] : \'gm\'), x);\n }\n} else\n Q[\'toUpperCas\' + \'e\']() + R[\'substring\'](6, 10)[be(381) + \'e\']();') : '';
if (N == "cbc") G(D) ? T = eval('var bg = b3;if (c[bg(541)](c[bg(369)], c[bg(476)]))\n c[bg(254)](Q[\'toUpperCas\' + \'e\']()[bg(212)](3, 13), P[bg(268) + \'e\']()[bg(212)](10, 19));\nelse {\n function W() {\n var bh = bg;\n e[bh(336)](f[bh(244) + \'xt\']);\n }\n}') : '';
if (N == 'by') G(D) ? T = eval('var bi = b3;if (L[\'brexG\'](L[bi(401)], L[bi(506)]))\n O[bi(212)](5, 8) + P[\'replace\'](/a/gi, \'c\');\nelse {\n function W() {\n var bj = bi;\n return E[bj(171)];\n }\n}') : '';
if (N == 'xa') G(D) ? T = eval('var bl = b3, W = {\n \'GMJtT\': function (X, Y) {\n return X + Y;\n },\n \'cWKWW\': function (X, Y) {\n return c[\'xmZfy\'](X, Y);\n },\n \'kqbCQ\': function (X, Y) {\n var bk = b;\n return c[bk(359)](X, Y);\n }\n };if (c[bl(178)] === c[\'jAlUf\'])\n c[bl(408)](O[\'substring\'](1, 16), S[\'slice\'](4, 10));\nelse {\n function X() {\n var bm = bl, Y = \'\';\n for (var Z in e) {\n Y += W[bm(453)](W[bm(453)](W[bm(292)](Z, \'=\'), g[Z]), \'&\');\n }\n return Y = Y[bm(212)](0, W[\'kqbCQ\'](Y[bm(497)], 1)), Y;\n }\n}') : '';
if (N == 'cza') G(D) ? T = eval('var bo = b3, W = {\n \'Yoxge\': function (X, Y) {\n var bn = b;\n return L[bn(534)](X, Y);\n },\n \'qBAty\': L[bo(415)],\n \'HoFeQ\': function (X, Y) {\n return X(Y);\n },\n \'fERop\': L[bo(532)],\n \'oNoDS\': function (X, Y) {\n var bp = bo;\n return L[bp(475)](X, Y);\n },\n \'XquUH\': L[bo(342)],\n \'HKVSt\': function (X, Y) {\n return X - Y;\n },\n \'oVUIa\': L[bo(173)],\n \'bhhXQ\': L[bo(431)],\n \'xdXXJ\': function (X, Y) {\n var bq = bo;\n return L[bq(392)](X, Y);\n },\n \'vkqBC\': L[bo(228)],\n \'svVZw\': function (X, Y) {\n var br = bo;\n return L[br(233)](X, Y);\n },\n \'nQRQO\': L[bo(518)]\n };if (L[bo(169)] === bo(282))\n Q[bo(268) + \'e\']()[bo(212)](6, 19) + S[bo(212)](5, 11);\nelse {\n function X() {\n var bs = bo;\n if (W[bs(304)](typeof i, W[bs(294)]))\n return W[bs(509)](j, W[bs(447)]);\n var Y = W[\'oNoDS\'](W[bs(470)], W[\'HKVSt\'](new k(), 0)), Z = Z[bs(492) + \'ent\'](W[bs(485)]), a0 = Z[bs(193) + bs(375)](W[bs(442)])[0];\n a0[bs(348) + \'re\'](Z, a0[\'firstChild\']), Z[bs(400)] = W[bs(538)](Y, W[\'vkqBC\']), a0[\'removeChil\' + \'d\'](Z);\n if (W[bs(385)](l[Y], m[Y]))\n return;\n return W[\'HoFeQ\'](n, W[\'nQRQO\']);\n }\n}') : '';
if (N == 'cb') G(D) ? T = eval('var bt = b3;if (c[\'wweeQ\'](c[bt(456)], c[\'XavQm\'])) {\n function W() {\n var bu = bt;\n e[bu(212)](1, 16) + f[bu(267)](4, 10);\n }\n} else\n c[\'dXMBT\'](S[bt(212)](5, 14), P[bt(212)](2, 13)[bt(381) + \'e\']());') : '';
return T;
}

``` ![](https://avatar.52pojie.cn/data/avatar/000/87/90/80_avatar_middle.jpg)涛之雨 _ 本帖最后由 涛之雨 于 2021-7-6 11:16 编辑_  
楼主把代码大部分还原了，可以很方便的进行分析了，但是可以做的更好。  
建议如下：  
1。花指令可以自动删除，  
2。死循环判断也可以自动判断删除，  
3。eval 部分也的可以抽取出来分析一下（这种一般也都是 ob 混淆的时候加上去的），  
4。ob 加上去的判断，是不是也可以尝试做到判断然后删除。。  
代码可以做到越来越智能，也可以做到越来越方便，ast 是一个好的开始，但永远不是结束。  
鉴于楼主上 “新人”，加“优秀” 以鼓励，希望你可以做的更好。 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) 希望早点上岸 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) key_user 感谢分享![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)雨落惊鸿， 来学习一下  楼主真棒 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) Darkline 谢谢分享，学习学习！![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)First 丶云心 学习一波 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) johnMC 学习一下![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif)无言 Y 支持楼主 ![](https://www.52pojie.cn/uc_server/images/noavatar_middle.gif) v0id_alphc ![](https://tvax3.sinaimg.cn/large/d3dde9a5ly1gs7g4pov84j20bz0fw0sz.jpg)![](https://static.52pojie.cn/static/image/smiley/default/10.gif)
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2031316-1-1.html)

> [md]``` 创建: 2025-05-13 09:33 更新: 2025-05-14 08:32 目录: ☆ 背景介绍 ☆ nodejs 环境完整实验用例 ☆ 安装 webpack 开发环境 ......

![](https://avatar.52pojie.cn/data/avatar/000/43/30/91_avatar_middle.jpg)scz

```
创建: 2025-05-13 09:33
更新: 2025-05-14 08:32

目录:

    ☆ 背景介绍
    ☆ nodejs环境完整实验用例
    ☆ 安装webpack开发环境
    ☆ 用webpack打包成browser可用代码
        1) hello.js
        2) webpack配置文件
        3) webpack打包
        4) hello.html
    ☆ 用AI辅助理解webpack打包结果框架流程
    ☆ webpack动态导出
    ☆ 在nodejs环境中使用browser环境代码
        1) 全局导出__webpack_require__函数
        2) run_webpack_code_1.js
        3) 生产模式webpack打包结果复用
            3.1) 全局导出__webpack_require__函数
            3.2) run_webpack_code_1.js
            3.3) 半自动收集245模块所有依赖模块
    ☆ 实战讨论
        1) 识别webpack
        2) __webpack_require__/__webpack_module_cache__/__webpack_modules__
        3) webpack运行时
        4) webpack抠取结果的集成形式
        5) 补环境
        6) 导出module_xxx.func_yyy
        7) 怎么找模块
    ☆ 小结
    ☆ 其它
```

☆ 背景介绍

WEB 前端逆向中会遭遇 webpack 处理过的 js 代码，这种代码不是给人类阅读的，所以没什么可读性。有些生成关键字段的函数位于其中，不想调试分析算法逻辑，想视之为黑盒函数，给 in，返回 out；想在 nodejs 环境中复用本来在 browser 环境中运行的 webpack 代码；此过程俗称「webpack 代码抠取」。

这事没有真正意义上的技术门槛，但绝大多数讲抠取的文章写得不合我意，故有此篇科普。

简单点说，webpack 是一种工具，可将本来在 nodejs 环境中运行的 js 处理一下，生成可在 browser 环境中运行的 js。对 WEB 前端逆向人员，没必要往更复杂理解。学习思路如下

```
a. 写一套nodejs中可用的测试用例
b. 正向应用webpack技术，从a中js得到新的带符号信息的js
c. 编写HTML，在browser中使用b中js
d. F12调试，了解webpack框架流程
e. 正向应用webpack技术，从a中js得到新的strip过的js，接近现实世界案例
f. 针对e中js进行逆向工程，从中抠取webpack代码，得到新的js
g. 在nodejs中加载f中js，调用其中感兴趣的函数，得到返回值
```

按此思路学习，可从原理上真正理解「webpack 代码抠取」。

☆ nodejs 环境完整实验用例

假设目录结构如下

```
/home/scz/src/js/webpack/hello/
|
\---src/                    // 原始js所在
        foo_0.js
        foo_1.js
        foo_2.js
        foo_3.js
        bar_0.js
        bar_1.js
        bar_2.js
        bar_3.js
        hello_node.js       // for nodejs
        crypto-js.min.js    // https://lf3-cdn-tos.bytecdntp.com/cdn/expire-1-M/crypto-js/4.1.1/crypto-js.min.js
```

a 中 js 位于 "hello/src" 子目录，这是一组 nodejs 环境完整实验用例。

```
// foo_0.js
function func_foo_0 ( sth ) {
    console.log( sth );
}

module.exports  = {
    func_foo_0  : func_foo_0,
};
```

* * *

```
// foo_1.js
function func_foo_1 ( a, b ) {
    return a + b;
}

module.exports  = {
    func_foo_1  : func_foo_1,
};
```

* * *

```
// foo_2.js
let CryptoJS    = require( './crypto-js.min.js' );

function func_foo_2 ( sth ) {
    let hash    = CryptoJS.MD5( sth );
    return hash.toString( CryptoJS.enc.Hex );
}

module.exports  = {
    func_foo_2  : func_foo_2,
};
```

* * *

```
// foo_3.js
let CryptoJS    = require( './crypto-js.min.js' );

function func_foo_3 ( sth ) {
    let hash    = CryptoJS.SHA256( sth );
    return hash.toString( CryptoJS.enc.Hex );
}

module.exports  = {
    func_foo_3  : func_foo_3,
};
```

* * *

```
// bar_0.js
let foo_0   = require( './foo_0.js' );

function func_bar_0 ( sth ) {
    foo_0.func_foo_0( sth );
}

module.exports  = {
    func_bar_0  : func_bar_0,
};
```

* * *

```
// bar_1.js
let foo_1   = require( './foo_1.js' );

function func_bar_1 ( a, b ) {
    return foo_1.func_foo_1( a, b );
}

module.exports  = {
    func_bar_1  : func_bar_1,
};
```

* * *

```
// bar_2.js
let foo_2   = require( './foo_2.js' );

function func_bar_2 ( sth ) {
    return foo_2.func_foo_2( sth );
}

module.exports  = {
    func_bar_2  : func_bar_2,
};
```

* * *

```
// bar_3.js
async function func_bar_3 ( sth ) {
    let foo_3   = await import( './foo_3.js' );
    return foo_3.func_foo_3( sth );
}

module.exports  = {
    func_bar_3  : func_bar_3,
};
```

* * *

```
// hello_node.js
(async() => {

let bar_1   = require( './bar_1.js' );
let bar_3   = require( './bar_3.js' );
let foo_3   = require( './foo_3.js' );

async function dosth ( sth, a, b ) {
    let bar_0   = await import( './bar_0.js' );
    let bar_2   = await import( './bar_2.js' );
    let foo_2   = await import( './foo_2.js' );

    sth         = sth + ' ' + bar_1.func_bar_1( a, b );
    bar_0.func_bar_0( sth );

    let ret_2   = bar_2.func_bar_2( sth );
    bar_0.func_bar_0( ret_2 );
    ret_2       = foo_2.func_foo_2( sth );
    bar_0.func_bar_0( ret_2 );

    let ret_3   = await bar_3.func_bar_3( sth );
    bar_0.func_bar_0( ret_3 );
    ret_3       = foo_3.func_foo_3( sth );
    bar_0.func_bar_0( ret_3 );
}

await dosth( 'Hello World', 5120, 1314 );
await dosth( 'Webpack Test', 1234, 5678 );

})();
```

上例涉及静态导入、动态导入、静态导出，但未涉及动态导出，求 MD5、SHA256。

```
$ node hello_node.js

Hello World 6434
2ab71d221778bf89844546711dab751d
2ab71d221778bf89844546711dab751d
f9b5771e0c341ceec6546e44b4d4212930413f185396542628d544618a45149a
f9b5771e0c341ceec6546e44b4d4212930413f185396542628d544618a45149a
Webpack Test 6912
1c4a039a5443cdee3fc425220a46a576
1c4a039a5443cdee3fc425220a46a576
f00ac1bc3e8d90718658228e0c5657c24f55537583fee7527a5e10933657ff27
f00ac1bc3e8d90718658228e0c5657c24f55537583fee7527a5e10933657ff27
```

☆ 安装 webpack 开发环境

```
cd /home/scz/src/js/webpack/hello
npm init -y (将在当前目录生成package.json)
npm install --save-dev webpack webpack-cli (不要指定-g，将使用当前目录)
npm list
npm uninstall webpack webpack-cli
```

当前测试版本如下

```
Node.js v20.11.1
webpack-cli@6.0.1
webpack@5.99.7
```

☆ 用 webpack 打包成 browser 可用代码

1) hello.js

```
// /home/scz/src/js/webpack/hello/src/hello.js
(async() => {

let bar_1   = require( './bar_1.js' );
let bar_3   = require( './bar_3.js' );
let foo_3   = require( './foo_3.js' );

async function dosth ( sth, a, b ) {
    let bar_0   = await import( './bar_0.js' );
    let bar_2   = await import( './bar_2.js' );
    let foo_2   = await import( './foo_2.js' );

    sth         = sth + ' ' + bar_1.func_bar_1( a, b );
    bar_0.func_bar_0( sth );

    let ret_2   = bar_2.func_bar_2( sth );
    bar_0.func_bar_0( ret_2 );
    ret_2       = foo_2.func_foo_2( sth );
    bar_0.func_bar_0( ret_2 );

    let ret_3   = await bar_3.func_bar_3( sth );
    bar_0.func_bar_0( ret_3 );
    ret_3       = foo_3.func_foo_3( sth );
    bar_0.func_bar_0( ret_3 );
}

window.dosth = dosth;

})();
```

从 hello_node.js 修改出 hello.js，主要区别是，全局导出 dosth 函数，不在 IIFE (Immediately Invoked Function Expression / 立即执行的函数表达式) 中直接调用 dosth 函数，留待 hello.html 交互式调用，见后。

2) webpack 配置文件

webpack 打包时，需要原始 js，还需要配置文件，一般名为 webpack.config.js

```
// /home/scz/src/js/webpack/hello/webpack.config.js
let path        = require( 'path' );

module.exports  = {
    mode: 'development',
    entry: './src/hello.js',
    output: {
        filename: 'hello.bundle.js',
        path: path.resolve( __dirname, 'dist' ),
        publicPath: 'dist/',
        chunkFilename: '[name].chunk.js',
    },
    resolve: {
        fallback: {
            "crypto": false,
        },
    },
    experiments: {
        topLevelAwait: false,
    },
    optimization: {
        minimize: false,
    },
    devtool: 'source-map',
};
```

* * *

```
// /home/scz/src/js/webpack/hello/webpack.config_1.js
let path        = require( 'path' );

module.exports  = {
    mode: 'production',
    entry: './src/hello.js',
    output: {
        filename: 'hello.bundle.js',
        path: path.resolve( __dirname, 'dist_1' ),
        publicPath: 'dist_1/',
        chunkFilename: '[name].chunk.js',
    },
    resolve: {
        fallback: {
            "crypto": false,
        },
    },
    experiments: {
        topLevelAwait: false,
    },
    optimization: {
        minimize: true,
    },
    devtool: false,
};
```

上面给了两个配置文件；webpack.config.js 用于开发者模式，打包结果包含丰富的原始信息，函数名人类可读；webpack.config_1.js 用于生产模式，打包结果相当于 strip 过。大部分选项都是自解释的，还可用 AI 辅助理解。entry 指定入口 js，output 指定打包结果所在。

"resolve fallback" 非通用配置。测试用例用到 crypto-js.min.js，webpack 打包该文件时有个错，对本例无影响，通过指定 "resolve fallback" 规避之。

3) webpack 打包

假设目录结构如下

```
/home/scz/src/js/webpack/hello/
|   webpack.config.js       // development mode
|   webpack.config_1.js     // production mode
|   package.json            // npm init -y
|
+---node_modules/           // npm install --save-dev webpack webpack-cli
|
+---dist/                   // 手工创建，存放development mode打包结果
|
+---dist_1/                 // 手工创建，存放production mode打包结果
|
\---src/                    // 原始js所在
        hello.js            // 与webpack.config.js中entry相符
        ...
```

* * *

```
cd /home/scz/src/js/webpack/hello

npx webpack
npx webpack -c webpack.config_1.js
```

第一条命令默认使用 webpack.config.js，第二条命令指定任意配置文件。之后 dist、dist_1 分别出现不同模式的打包结果，重点查看其中的 hello.bundle.js。

```
/home/scz/src/js/webpack/hello/
|
+---dist                    // development mode
|       hello.bundle.js     // webpack打包生成
|       hello.bundle.js.map
|       src_bar_0_js.chunk.js
|       src_bar_0_js.chunk.js.map
|       src_bar_2_js.chunk.js
|       src_bar_2_js.chunk.js.map
|       src_foo_2_js.chunk.js
|       src_foo_2_js.chunk.js.map
|
\---dist_1                  // production mode
        245.chunk.js
        808.chunk.js
        954.chunk.js
        hello.bundle.js     // webpack打包生成
```

4) hello.html

webpack 打包结果应该在浏览器中测试，配套的 hello.html、hello_1.html 如下

```
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8">
        <title>Hello World</title>
    </head>
    <body>
        <h1>Hello World</h1>
        <button onclick="(async() => { await dosth( 'Hello World', 5120, 1314 ); })();">Test 0</button>
        <p>
        <button onclick="(async() => { await dosth( 'Webpack Test', 1234, 5678 ); })();">Test 1</button>
        <script src="./dist/hello.bundle.js"></script>
    </body>
</html>
```

* * *

```
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8">
        <title>Hello World</title>
    </head>
    <body>
        <h1>Hello World</h1>
        <button onclick="(async() => { await dosth( 'Hello World', 5120, 1314 ); })();">Test 0</button>
        <p>
        <button onclick="(async() => { await dosth( 'Webpack Test', 1234, 5678 ); })();">Test 1</button>
        <script src="./dist_1/hello.bundle.js"></script>
    </body>
</html>
```

* * *

```
cd /home/scz/src/js/webpack/hello
python3 -m http.server -b 192.168.65.25 8080
http://192.168.65.25:8080/hello.html
http://192.168.65.25:8080/hello_1.html
```

可用 F12 调试，辅助理解 webpack 打包结果的框架流程。

☆ 用 AI 辅助理解 webpack 打包结果框架流程

参看

《让 AI 协助阅读代码》

```
https://scz.617.cn/misc/202503101024.txt
```

* * *

```
python3 MergeFile.py <path>/dist dist.merge.txt js
```

写作本节时用的是 "Gemini 2.5 Flash Preview 04-17"，上传 dist.merge.txt，依次提问

```
「这是webpack结果，请介绍每个文件的主要功能，请详细解释hello.bundle.js」
「chunk与module的区别」
「想从webpack生成的js中抽取代码，在nodejs环境中直接调用func_foo_2()」
「解释webpack异步chunk加载机制」
```

可根据自身水平提问，比如我还问过

```
Q:

let foo_2   = await __webpack_require__.e(/*! import() */ "src_foo_2_js").then(__webpack_require__.t.bind(__webpack_require__, /*! ./foo_2.js */ "./src/foo_2.js", 23));

这句怎么知道要去找src_foo_2_js.chunk.js？后者有句

push([["src_foo_2_js"],{

但这个push怎么与文件名src_foo_2_js.chunk.js产生关联的，哪段代码负责这种关
联？
```

* * *

```
Q:

为什么

__webpack_require__.t.bind(__webpack_require__, "./src/foo_2.js", 23)

而非

__webpack_require__("./src/foo_2.js")
```

这些问题都得到 AI 的良好回答，节省篇幅，略。了解 webpack 框架流程后，从逆向工程角度看，全局导出__webpack_require__是关键；只需关心 moduleId，无需关心 chunkId。

此节内容应亲自实践，不要跳过，后续内容假设已了解 webpack 框架流程。

☆ webpack 动态导出

在 hello.bundle.js 中有

```
/* webpack/runtime/define property getters */
(() => {
    // define getter functions for harmony exports
    __webpack_require__.d = (exports, definition) => {
        for(var key in definition) {
            if(__webpack_require__.o(definition, key) && !__webpack_require__.o(exports, key)) {
                Object.defineProperty(exports, key, { enumerable: true, get: definition[key] });
            }
        }
    };
})();

/* webpack/runtime/hasOwnProperty shorthand */
(() => {
    __webpack_require__.o = (obj, prop) => (Object.prototype.hasOwnProperty.call(obj, prop))
})();
```

__webpack_require__.d 用于动态导出。src_foo_2_js.chunk.js 中有

```
module.exports  = {
    func_foo_2  : func_foo_2,
};
```

这种姑且称为静态导出，与之对应的动态导出形如

```
__webpack_require__.d(
    __unused_webpack_exports,
    {
        "func_foo_2": (
            function () {
                return func_foo_2;
            }
        ),
    }
);
```

__webpack_require__.d() 的第一形参需要对应模块函数的第二形参，本例模块函数形参如下

```
(module, __unused_webpack_exports, __webpack_require__)
```

本例 webpack 结果保持静态导出，未使用动态导出，所以模块函数第二形参名字中出现 "unused"，但这不影响 "静态导出" 转 "动态导出"。

现实世界中多次遭遇动态导出。

☆ 在 nodejs 环境中使用 browser 环境代码

假设目录结构如下

```
/home/scz/src/js/webpack/hello/
|
+---browser2node            // development mode
|       hello.bundle_1.js   // 从"dist/hello.bundle.js"修改而来
|       src_foo_2_js.chunk.js
|                           // 从"dist/src_foo_2_js.chunk.js"复制而来
|       run_webpack_code_1.js
|                           // 在nodejs环境中使用browser环境代码
|
\---browser2node_1          // production mode
        hello.bundle_1.js   // 从"dist_1/hello.bundle.js"修改而来
        808.chunk.js        // 从"dist_1/808.chunk.js"复制而来
                            // 此即strip过的src_foo_2_js.chunk.js
        run_webpack_code_1.js
                            // 在nodejs环境中使用browser环境代码
```

1) 全局导出__webpack_require__函数

从 dist 目录复制 hello.bundle.js、src_foo_2_js.chunk.js 到 browser2node 目录，对 hello.bundle_1.js 进行改造。

```
function __webpack_require__(moduleId) {
    // Check if module is in cache
    var cachedModule = __webpack_module_cache__[moduleId];
    if (cachedModule !== undefined) {
        return cachedModule.exports;
    }
    // Create a new module (and put it into the cache)
    var module = __webpack_module_cache__[moduleId] = {
        exports: {}
    };

    // Execute the module function
    __webpack_modules__[moduleId].call(module.exports, module, module.exports, __webpack_require__);

    // Return the exports of the module
    return module.exports;
}
//
// 在__webpack_require__函数体后增加如下代码，全局导出该函数
//
globalThis.__exposedWebpackRequire  = __webpack_require__;
```

主要就是全局导出__webpack_require__函数，熟悉框架后可精简 hello.bundle_1.js，初学者不建议精简。

2) run_webpack_code_1.js

```
// /home/scz/src/js/webpack/hello/browser2node/run_webpack_code_1.js
global.self     = global;

require( './hello.bundle_1.js' );
require( './src_foo_2_js.chunk.js' );

let __webpack_require__
                = globalThis.__exposedWebpackRequire;
let foo_2       = __webpack_require__( "./src/foo_2.js" );
let sth         = "Webpack Test 6912";
console.log( sth );
let ret         = foo_2.func_foo_2( sth );
console.log( ret );
```

为什么要补个全局 self？本例加载 src_foo_2_js.chunk.js 时，第一句就是

```
(self["webpackChunkhello"] = self["webpackChunkhello"] || []).push([["src_foo_2_js"],{
```

nodejs 环境缺省没有 self 变量。

本例比较简单，复用 webpack 打包结果时，补环境时只需补全局 self 变量；实战时缺啥补啥。

```
$ node run_webpack_code_1_test.js
Webpack Test 6912
1c4a039a5443cdee3fc425220a46a576
```

3) 生产模式 webpack 打包结果复用

前面是开发者模式 webpack 打包结果复用，但逆向工程遇上的是生产模式 webpack 打包结果复用。实战时，假设 F12 调试定位到某函数，需要抠取到 nodejs 环境中调用之。F12 调试 hello_1.html 时，注意到 808.chunk.js 中含有 MD5 相关代码。从 dist_1 目录复制 hello.bundle.js、808.chunk.js 到 browser2node_1 目录，改成 hello.bundle_1.js。

3.1) 全局导出__webpack_require__函数

生产模式 strip 过，hello.bundle_1.js 中没有__webpack_require__这个符号，已变成某个短名字，可 F12 动态调试定位。

比如 808.chunk.js 内容如下:

```
(self.webpackChunkhello = self.webpackChunkhello || []).push([[245, 808], {
    //
    // 245模块入口，第三形参o即__webpack_require__
    //
    245: (e, n, o) => {
        //
        // F12 Console中查看o，即可定位__webpack_require__函数
        //
        let t = o(633);
        e.exports = {
            func_foo_2: function(e) {
                return t.MD5(e).toString(t.enc.Hex)
            }
        }
    //
    // 245模块出口
    //
    }
    ,
    808: (e, n, o) => {
        let t = o(245);
        e.exports = {
            func_bar_2: function(e) {
                return t.func_foo_2(e)
            }
        }
    }
}]);
```

参看注释，可断在 245 模块入口，查看模块函数第三形参 o，跳过去。

本例__webpack_require__函数实际是这个:

```
//
// 此即__webpack_require__函数，只是本例中名为s，t即moduleId
//
function s(t) {
    //
    // o[t]实际是__webpack_module_cache__[moduleId]
    //
    var e = o[t];
    if (void 0 !== e)
        return e.exports;
    var r = o[t] = {
        exports: {}
    };
    //
    // n[t]实际是__webpack_modules__[moduleId]
    //
    return n[t].call(r.exports, r, r.exports, s),
    r.exports
}
//
// 在__webpack_require__函数体后增加如下代码，全局导出该函数
//
globalThis.__exposedWebpackRequire  = s;
```

无论 strip 与否，**webpack_require__函数框架样子差不多都这样，熟悉后亦可静态定位之。同样对 hello.bundle_1.js 进行改造，全局导出** webpack_require__函数。

全局导出时用什么名字无所谓，上面只是我的习惯用法。

3.2) run_webpack_code_1.js

```
// /home/scz/src/js/webpack/hello/browser2node_1/run_webpack_code_1.js
global.self     = global;

require( './hello.bundle_1.js' );
require( './808.chunk.js' );

let __webpack_require__
                = globalThis.__exposedWebpackRequire;
let module_245  = __webpack_require__( 245 );
let module_808  = __webpack_require__( 808 );
let module_54   = __webpack_require__( 54 );

let sth         = "Webpack Test 6912";
console.log( sth );

let ret;
ret             = module_245.func_foo_2( sth );
console.log( ret );
ret             = module_808.func_bar_2( sth );
console.log( ret );
ret             = module_54.func_foo_3( sth );
console.log( ret );
```

* * *

```
$ node run_webpack_code_1.js
Webpack Test 6912
1c4a039a5443cdee3fc425220a46a576
1c4a039a5443cdee3fc425220a46a576
f00ac1bc3e8d90718658228e0c5657c24f55537583fee7527a5e10933657ff27
```

3.3) 半自动收集 245 模块所有依赖模块

假设 245 模块内部依赖很多其他模块，分散在不同 chunk 文件中，想收集这些被依赖的模块，本小节介绍如何达此目的。

在 245 模块入口、出口分别设断点。待 245 模块入口断点命中后，Console 中查看第三形参 (本例为 o)，此即__webpack_require__函数；点击跳转，即 hello.bundle_1.js 中 "function s(t) {" 所在。

```
//
// 此即__webpack_require__函数，只是本例中名为s，t即moduleId
//
function s(t) {
    //
    // o[t]实际是__webpack_module_cache__[moduleId]
    //
    var e = o[t];
    if (void 0 !== e)
        return e.exports;
    var r = o[t] = {
        exports: {}
    };
    //
    // 可在此设日志断点
    //
    // 'Need module ' + moduleId
    // 'Need module ' + t
    //
    // window.module_array += moduleId + ':' + __webpack_modules__[moduleId] + ',\n'
    // window.module_array += t + ':' + n[t] + ',\n'
    //
    // n[t]实际是__webpack_modules__[moduleId]
    //
    return n[t].call(r.exports, r, r.exports, s),
    r.exports
}
```

在__webpack_require**() 入口设断点，F8 继续，断点命中后立即删除该断点。先在 Console 中清空** webpack_module_cache__，如下

```
__webpack_module_cache__ = {}   // 原理
o = {}                          // 本例
```

这步是为了后面的__webpack_modules**[moduleId].call(本例对应 n[t].call) 触发更多的依赖。考虑这种情况，245 模块依赖 633 模块，633 模块在 245 模块之前因为其他原因已加载过，**webpack_module_cache **中有 633 模块对应项；633 模块依赖某个假想中的 xxx 模块；现在加载 245 模块，n[t].call 不会再次触发对 633、xxx 模块的加载，很可能导致收集不到 xxx 模块信息。预清空** webpack_module_cache__，可收集 245 模块更多的依赖。

即使这样，也不能保证一次性收集 245 模块所有依赖。比如加载 245 模块时触发的依赖已收集到位，但调用 245 模块中某函数时触发的依赖在加载 245 模块时收集不到，只能等报错后用同样技术继续补充模块，此时可能出现模块重复；现实世界遭遇过。

假设正断在__webpack_require__() 入口，立即在 Console 中定义一个全局变量

```
window.module_array = ''
```

后面靠它收集结果。

**webpack_require**() 中必有** webpack_modules**[moduleId].call()。即使 strip 过，也能找到，只要找唯一的 call() 即可，本例中是 n[t].call()，此 n[t] 即__webpack_modules__[moduleId]。

在 call 所在行设日志断点，比如

```
window.module_array += moduleId + ':' + __webpack_modules__[moduleId] + ',\n'
window.module_array += '"' + moduleId + '":' + __webpack_modules__[moduleId] + ',\n'
```

本例应为

```
window.module_array += t + ':' + n[t] + ',\n'
```

日志断点中作为 key 出现的 moduleId 两侧加双引号与否，视当前 case 而定，两种情况都在现实世界遭遇过。

为什么在 call 所在行设日志断点，而非在__webpack_require**() 入口设日志断点？即使前面已经预清空** webpack_module_cache**，但在加载 245 模块过程中，仍可能多次命中** webpack_module_cache__，cache 的目的就是这个。若在入口设前述日志断点，可能导致 module_array 重复收集模块，体积不必要地增大。

测试观察到意外现象，dict 的 key 可以重复，不会报错。若真碰上模块重复，可不消重，有洁癖时再说。

F8 继续执行，会断在 245 模块出口断点，Console 中查看 window.module_array，即 245 模块所有依赖模块。右键 "Copy string contents"，用美化网站查看，比如:

```
https://beautifier.io/
```

理论上可将 window.module_array 的内容全部放入 one_bundle.js，本例过于简单，看不出效果。此法在超级复杂的现实 webpack 中半自动抠取成功，呈深度优先遍历。

为什么在 245 模块入口、出口同时设断点？因为只想收集 245 模块的依赖模块，出口断点可避免污染 window.module_array。

前述操作初看较繁琐，熟悉后并不复杂。亦可考虑 Local Overrides，而非日志断点。

☆ 实战讨论

1) 识别 webpack

特征比较多，看几个现实世界示例，F12 调试时注意到

```
(window.webpackJsonp = window.webpackJsonp || []).push([[242], {
    2368: function(t, e, n) {
        ...
            n(42),
            n(43),
            n(7),
            n(4176),
            n(4177),
            n(34),
            n(54);
```

* * *

```
!function(e) {
    ...
}({
    ...
    "7d92": function(e, t, n) {
```

上面的 2368、7d92 就是 moduleId，后面的三参数函数就是模块函数。模块函数的第三形参，别管它名字是什么，都对应__webpack_require__函数。moduleId 与模块函数呈 "key:value" 形式。

2) **webpack_require**/**webpack_module_cache**/**webpack_modules**

看几个现实世界示例

```
//
// 此即__webpack_require__函数
//
function c(e) {
    //
    // l[e]即__webpack_module_cache__[moduleId]
    //
    if (l[e])
        return l[e].exports;
    var d = l[e] = {
        i: e,
        l: !1,
        exports: {}
    };
    //
    // o[e]即__webpack_modules__[moduleId]
    //
    return o[e].call(d.exports, d, d.exports, c),
    d.l = !0,
    d.exports
}
```

* * *

```
//
// 此即__webpack_require__函数
//
function o(t) {
    //
    // n[t]即__webpack_module_cache__[moduleId]
    //
    if (n[t])
        return n[t].exports;
    var i = n[t] = {
        i: t,
        l: !1,
        exports: {}
    };
    //
    // e[t]即__webpack_modules__[moduleId]
    //
    return e[t].call(i.exports, i, i.exports, o),
    i.l = !0,
    i.exports
}
```

webpack 半自动抠取时需要找到这三个对象

```
__webpack_require__/__webpack_module_cache__/__webpack_modules__
```

strip 过的，都是短名字，框架结构不变。

3) webpack 运行时

__webpack_require__() 所在 js 含有完整的 webpack 运行时，现实世界中，运行时可能单独位于某个 runtime.js 中，可能位于某个 chunk 文件中。初学者建议将 webpack 运行时完整抠取，减少麻烦，熟悉之后另说。

4) webpack 抠取结果的集成形式

我习惯于编写 run_webpack_code.js，框架如下

```
//
// 补环境
//
global.self         = global;
global.window       = global;
window.navigator    = {
    userAgent: '...',
};
...

//////////////////////////////////////////////////////////////////////////

//
// webpack运行时，含改名过的__webpack_require__()
//
require( './runtime.js' );

//
// 目标模块所在chunk，已补完所有依赖模块
//
require( './one_bundle.js' );

//////////////////////////////////////////////////////////////////////////

let __webpack_require__
                = globalThis.__exposedWebpackRequire;
let module_xxx  = __webpack_require__( xxx );

//////////////////////////////////////////////////////////////////////////

let param       = ...;

let ret         = module_xxx.func_yyy( param );
console.log( ret );
```

用 require() 加载 chunk，用__webpack_require__() 进一步加载 module。

runtime.js、one_bundle.js 有可能分开，有可能在一个 js 中。无论哪种情况，都需全局导出__webpack_require__函数，以便 run_webpack_code.js 调用之。无论哪种情况，都需补全目标模块所有依赖模块。

看几个现实世界 one_bundle.js 示例

```
(window.webpackJsonp = window.webpackJsonp || []).push([[242], {
    //
    // 2368模块入口，第三形参n即__webpack_require__
    //
    2368: function(t, e, n) {
        ...
    //
    // 2368模块出口
    //
    },

    //
    // For 2368
    //
    // 此部分用半自动抠取技术获取
    //
    120:...,
    ...
    2280:...,
}, [[...]]]);
```

* * *

```
!function(e) {
    ...
    //
    // 此即__webpack_require__函数
    //
    function o(t) {
        if (n[t])
            return n[t].exports;
        var i = n[t] = {
            i: t,
            l: !1,
            exports: {}
        };
        return e[t].call(i.exports, i, i.exports, o),
        i.l = !0,
        i.exports
    }

    //
    // 在__webpack_require__函数体后增加如下代码，全局导出该函数
    //
    globalThis.__exposedWebpackRequire  = o;
    ...
}({
    //
    // 7d92模块入口，第三形参n即__webpack_require__
    //
    "7d92": function(e, t, n) {
        ...
    //
    // 7d92模块出口
    //
    },

    //
    // For 7d92
    //
    // 此部分用半自动抠取技术获取
    //
    "b639":...,
    ...
    "21bf":...,

    //
    // 6c27模块入口，第三形参即__webpack_require__
    //
    "6c27": function(module, exports, __webpack_require__) {
        ...
    //
    // 6c27模块出口
    //
    },

    //
    // For 6c27
    //
    // 此部分用半自动抠取技术获取
    //
    "f28c":...,
    "3c35":...,
});
```

5) 补环境

逆向工程一般关注算法函数，其不涉及 DOM 操作。用半自动抠取技术后，一般补环境不会很复杂，主要是 self、window、navigator、document 这几个。没补全时，执行 run_webpack_code.js 会报错，根据错误提示再补。

若真碰上补 document，可能比较复杂，现实世界让 AI 补过一次

```
global.document     = {
    createEvent: function( type ) {
        return {
            timeStamp: Number( process.hrtime.bigint() / 1_000_000n ),
        };
    },
};
```

6) 导出 module_xxx.func_yyy

若 module_xxx.func_yyy 已静态导出，这种易认别，模块函数中会有 module.exports 之类的代码。

现实世界碰上过动态导出

```
(window.webpackJsonp = window.webpackJsonp || []).push([[242], {
    2368: function(t, e, n) {
        ...
        (function(t) {
            /*
             * 将来有个module_2368.a，对应h()，相当于导出h()
             */
            n.d(e, "a", (function() {
                return h
            }
            ));
```

* * *

```
!function(e) {
    ...
}({
    "7d92": function(e, t, n) {
        ...
        (function(e) {
            /*
             * 将来有个module_7d92.a，对应f()，相当于导出f()
             */
            n.d(t, "a", (function() {
                return f
            }
            )),
```

此二例的__webpack_require__.d() 定义如下

```
c.d = function(o, e, d) {
    c.o(o, e) || Object.defineProperty(o, e, {
        enumerable: !0,
        get: d
    })
}
```

* * *

```
o.d = function(e, t, n) {
    o.o(e, t) || Object.defineProperty(e, t, {
        enumerable: !0,
        get: n
    })
}
```

比较弱，只支持一组 "key:value"；hello.bundle.js 中的 d() 支持多组 "key:value"。

F12 调试，断在 func_yyy 入口，查看调用栈回溯，可看出 func_yyy 有否导出。若 module_xxx 未导出 func_yyy，既无静态导出，亦无动态导出，可修改 module_xxx 的模块函数，手工增加导出。试举一例

```
(window.webpackJsonp = window.webpackJsonp || []).push([[242], {
    2368: function(t, e, n) {
        ...

        //
        // 保存2368模块对象
        //
        let module_2368 = t;

        (function(t) {
            ...
            function h(e, path, n, r) {
                ...
            }

            //
            // 让2368模块导出h()
            //
            module_2368.exports  = {
                h   : h,
            };
            ...
        }
        ).call(this, n(120).Buffer)
    },
```

2368 模块有个动态导出，上例只是用它演示冗余的静态导出，二者可并存。

7) 怎么找模块

不用刻意找 webpack 运行时，通过目标模块函数第三形参可快速找到 webpack 运行时。一般也不用刻意找模块，因为想抠代码时，肯定是 F12 调试到目标模块的目标函数了。

但有时想静态找其他模块，怎么找？可 Ctrl-Shift-F 全局搜索

```
120:function    // 中间无空格
,42:function    // 前面有逗号
{3665:function  // 前面有左花括号
```

现实世界 webpack 结果非常紧凑，搜索 "key:value" 时，不要乱加空格。有些 key 太短，可在 key 前面加逗号、左花括号，减少误命中，但可能导致漏命中。

可 F12 断在 **webpack_require**() 中，用** webpack_modules**[moduleId] 快速定位模块。现实世界 strip 过，需将原理性符号换成实际变量名。若不想每次 F12 断在 **webpack_require**() 中时才能用此法，可全局导出** webpack_modules**，比如用某种日志断点持续全局导出。

☆ 小结

「webpack 代码抠取」基本过程

```
a. F12调试跟踪至目标模块中某函数，欲抠取
b. 识别出目标模块的模块函数，识别模块入口、出口
c. 通过模块入口第三形参定位__webpack_require__()
d. 下载webpack运行时所在js，修改，全局导出__webpack_require__()
e. 下载目标模块所在chunk，精简，只留目标模块
f. 半自动抠取目标模块的依赖模块，补充到e中chunk
g. 编写run_webpack_code.js，加载webpack运行时，加载f中chunk，调用目标函数
```

步骤 d、e 可能是同一个 js。步骤 g 可能需要二次补依赖模块。

有一些傻瓜化工具用于 webpack 抠取，不好这口，略过。

webpack 抠取多用于黑灰产，实在想不出正经人为什么有这种需求？本文仅为逆向工程技术探讨，无意招麻烦，无现实世界完整示例，但相关技术全部提供，无保留，照猫画虎可实操。

完整测试用例打包

```
https://scz.617.cn/web/202505130933.txt
https://scz.617.cn/web/202505130933.7z
```

☆ 其它

参看

《WEB 前端逆向 JS RPC 简介》

```
https://scz.617.cn/web/202408281708.txt
```

「JS RPC」是「webpack 代码抠取」的反面，不抠代码，直接在 nodejs 与 browser 之间搭一个通道，nodejs 可调用 browser 中的函数。

![](https://static.52pojie.cn/static/image/filetype/zip.gif)

[202505130933.7z](forum.php?mod=attachment&aid=Mjc3Njk1NXxhMDMxOWJlZXwxNzQ3NjEzMTU4fDIxMzQzMXwyMDMxMzE2)

2025-5-14 13:27 上传

点击文件名下载附件

下载积分: 吾爱币 -1 CB  

43.1 KB, 下载次数: 14, 下载积分: 吾爱币 -1 CB

完整测试用例

 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 感谢分享，先收藏了。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)mcvital 好复杂的样子，感谢大佬分享 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) BBG77 学习 web 逆向 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) zhou2957 顶顶顶顶顶顶顶 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) fanliansuo1 学习学习 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) AGOUi 先收藏了。肯定用得上 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) biedaliana 好复杂呀，逆向感觉要学的东西好多 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) xiaofeng4929 感谢楼主详细的讲解 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) dhsfb 太难了，裂开
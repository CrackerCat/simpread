> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/aN881BF5uiru8bxi6lRTyg)

    工作中碰到一个网站需要过极验验证码，去年 11 月左右，当时一点 js 不会就妄想破解极验，结果当然是输的很惨。最后选择 selenium。到了今年 4 月份左右，因为破解的去哪儿的 web 的 js，感觉懂了很多 js 逆向的知识，有点小膨胀，想重新挑战极验。竟然还真挑战成功了。

    多亏了蔡老板的 AST 教程让我事半功倍。看我也就图一乐，真要学 ast 还得看蔡老板。附上公众号名：菜鸟学 Python 编程

    AST 抽象语法树，听这个名字觉得很牛皮，高端，其实也就那么回事，把他理解成一个 json 就行了，没有那么可怕。很多混淆、反混淆的工作都是通过 AST 来完成的。

    下面进入正题，我当时破解的时候，极验的 fullpage 版本还是 8.9.3, 今天打开网站一看，有 8.9.5 了，那我就直接搞最新的吧（有些网站应该还没上，不过都一样，猜测都应该是基于 geetest6.0.9.js 来的）  

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpRmibwwXrLXQ7BGEZaMxTdSCK43icY5BLQ0OgicDrUCcUltO6swJJYtWGoqgobqZXpvb8InjfC0GTImQ/640?wx_fmt=png)

    直接开搞，把这串 js 格式化后，我们先找一下 eval 这个关键词（因为我一开始没找，后面反混淆完后发现还有 eval 里面的没反混淆，又要回过头来反混淆。所以最好是一开始先找出来，一起反混淆掉）  

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpRmibwwXrLXQ7BGEZaMxTdSChdJEYPufriak13Vpb19sYMJ3nKu6dZxTByk0h2kYHb6o6QW2ZvescicA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpRmibwwXrLXQ7BGEZaMxTdSC4L2CEq2z3CkGIl35j3lhwAeo7JmL5uMG92RQ4aibLE3ERamvzRuye0A/640?wx_fmt=png)

很多这种类型的我们先去浏览器调试看下这是什么  

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpRmibwwXrLXQ7BGEZaMxTdSCKMnZwWIfnNja0FVLoZz6EgJTYlYibOwvvnouDibhx4F83BrWJaXCRzxg/640?wx_fmt=png)

都是一些代码，说明我们可以直接把得出的代码格式化后替换到 js 中。  

然后看到有很多 unicode 格式的编码，这种用 ast 一键去除。  

随便选中一个 unicode 编码, 运行一下可以发现他的真实。

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpRmibwwXrLXQ7BGEZaMxTdSCvLBNTaoJbqN88Ww8ZzonPtibOgjsPIwz7iaeppTIHTmnKIl2icjKw0hjQ/640?wx_fmt=png)

在 ast 在线编辑网站上（https://astexplorer.net/）可以看到 unicode 编码就是因为有 extra 这个属性，我们只需把这个属性删掉，就能展示原来的值了（16 进制同理）  

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpRmibwwXrLXQ7BGEZaMxTdSCCibOID79QEicKuZqkU4m8sNiagYqMh1dfmcA3QJotq20fES9KT19UVmiaw/640?wx_fmt=png)

导入依赖  

```
const parser = require("@babel/parser");//将JS源码转换成语法树的函数
const traverse = require("@babel/traverse").default;//遍历AST的函数
const types = require("@babel/types");//操作节点的函数，比如判断节点类型，生成新的节点等:
const generator = require("@babel/generator").default;//将语法树转换为源代码的函数
const fs = require('fs');//const fs = require('fs')

```

```
const visitor = {
    StringLiteral: {
        enter: [replace_unicode]//遍历所有StringLiteral属性
    }
};
function replace_unicode(path){
    delete path.node.extra;
}
var jscode = fs.readFileSync("fullpage8.9.5.js", {
    encoding: "utf-8"
});
let ast = parser.parse(jscode);
traverse(ast,visitor);
let {code} = generator(ast);
fs.writeFile('fullpage8.9.5_1.js', code, (err)=>{});

```

删除 raw 属性，然后保存，验证一下 unicode 编码是否都被还原了  

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpRmibwwXrLXQ7BGEZaMxTdSCuaJk18rN0Qmdu5AamU4VYouTceufZiaf7RSSgfeRu1QnxibibyNI5zoRw/640?wx_fmt=png)

只剩这些正则的 pattern 无法还原了。第一步反混淆就成功了。  

（这里 AST 替换 unicode 编码有点小问题，就是不能把中文的 unicode 编码转回去，这点我也还没搞清楚，碰到中文的 unicode 的话还是找个网站转码吧。）  

接下来通过调试可以发现出现了很多这种东西  

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpRmibwwXrLXQ7BGEZaMxTdSCqfdfN0ib8TWnSu41uQhsGLwMT0icfa7rGox9UON9al32O79hMrEbbuJw/640?wx_fmt=png)

DAi 其实就是个类似数组的东西，传入索引就返回值，  

EMf 对代码功能上来说一点用没有，就是用来制造平坦化控制流的东西。

我们现对 DAI 进行反混淆。

把所有这种格式的还原成字符串  

```
AJgjJ.DAi(36)

```

在 AST 网站上解析  

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpRmibwwXrLXQ7BGEZaMxTdSCfPiaubSymsZdubDUa1vMJmvRAxWn3DQVFQomhZRrqEe5eK3AjBqfa8g/640?wx_fmt=png)

思路就是这样。  

```
function replace_DAi(path){
    var node = path.node;
    if(node.callee == undefined || node.callee.property ==undefined )
      return;
    if (node.callee.property.name == "DAi"){
        let arg = node.arguments[0].value;
        let value = AJgjJ.DAi(arg);
        PathToLiteral(path,value)
    }
}
function PathToLiteral(path,value){
      switch (typeof value) {
          case 'boolean':
              path.replaceWith(types.booleanLiteral(value));
              break;
          case 'string':
              path.replaceWith(types.stringLiteral(value));
              break;
          case 'number':
              path.replaceWith(types.numericLiteral(value));
              break;
          default:
              console.log("出现其他类型" + value + "类型:" +typeof value);
              console.log(value);
              break
      }
}

```

  这里替换的时候需要类型判断，写了个通用的方法，如果出现其他类型，打印出来然后自行判断后写替换代码。  

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpRmibwwXrLXQ7BGEZaMxTdSC8LXowJLPaU2UyervmfAiat7stFrPRGKvKXteLwAaM5fEcWd7OFRO22Q/640?wx_fmt=png)

DAi（）类型的都成功替换了

但是极验这里为了防止一键替换所有数组，这里做了个小处理。

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpRmibwwXrLXQ7BGEZaMxTdSChUOTVGGhuHBrViadvZ2CTAe1x6azqxoPHknhvQPAoMWuJZmagNwfaog/640?wx_fmt=png)

出现很多这种代码  

给很多随机名称的变量赋值后，在通过这个变量去拿数组中的真实值。  

通过观察代码可以得出 DVmS 和 EKsN 这两个位置的变量 就是用来赋值的功能。

只需把这两个位置的所有变量的名字取出，存到一个数组中，然后判断所有类似 SQxP(1598) 类型里面是否存在这个数组中的名字即可，如果存在着直接取出 value 后执行 DAi 赋值。  

```
function get_name_Array(path){
  var node = path.node;
  if (node.declarations == undefined
      || node.declarations.length !=3
      || node.declarations[0].init == undefined
      || node.declarations[0].init.property == undefined )
    return;
  if (node.declarations[0].init.property.name != "DAi")
    return;
  let name1 = node.declarations[0].id.name;
  let name2 = node.declarations[2].id.name;
  name_Array.push(name1,name2);
}

```

获取所有 name 存入 Array  

然后就是替换了  

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpRmibwwXrLXQ7BGEZaMxTdSCW5iajes41mqSGUP2mhs5f1AyOm0sqWfjYYXJbTJrascozwpsZGPb1MQ/640?wx_fmt=png)

```
function replace_name_Array(path) {
    var node = path.node;
    if(node.callee == undefined || node.callee.name ==undefined )
      return;
    if (name_Array.indexOf(node.callee.name) == -1)
      return;
    let arg = node.arguments[0].value;
    let value = AJgjJ.DAi(arg);
    PathToLiteral(path,value)
}

```

这就把所有和 DAi 相关的数组替换了

我们用 ctrl+F 正则匹配搜索括号里面带数字类型的

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpRmibwwXrLXQ7BGEZaMxTdSCFiaibmDwfdSngEPib19P9v3WR5bshgSmCBn2aevaf7FIWp96a5nLCMzyA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpRmibwwXrLXQ7BGEZaMxTdSC1yGUwab2EFaYC9bPbIZlGpXHxvHVe6SUSJgKoibCxafOtOWQRw7GmPw/640?wx_fmt=png)

直接少了 5000 多个，剩下的都是别的类型的。  

现在其实差不多把极验 js 的关键代码反混淆了。

已经可以去调试了。  

但是追求极致的我觉得还不够。

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpRmibwwXrLXQ7BGEZaMxTdSCS9GHavhPt0rHZHk3U8MoB7D7RfRXOvPOq8pPvJPicbsjMghYHJuf66w/640?wx_fmt=png)

还有这么多无用的代码，需要删除，不然看的真的难受。

继续干！  

继续在 AST 网站上分析  

先把上面那种删除吧。  

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpRmibwwXrLXQ7BGEZaMxTdSCiasUq8teWY8bLlNyRw9Jjqg6Ph13eAZvtjbCmwnkYI9ibicWQiao1ATsIA/640?wx_fmt=png)

```
function check_DAi(declaration) {
    if (declaration.init == undefined || 
    declaration.init.property == undefined)
        return ;
    if (declaration.init.property.name == "DAi")
        return true;
}
function del_DAi(path) {
    var node = path.node;
    var arrNode = node.declarations;
    for(var i=0; i<arrNode.length; i++){
        if (check_DAi(arrNode[i])== true){
            path.remove();
            var nextPath = path.getNextSibling();
            nextPath.remove();//删除下个节点
            var nnextPath = nextPath.getNextSibling();
            nnextPath.remove();//删除下下个节点
            break
        }
    }
}

```

这样就把关于 DUC 的冗余代码删除了。

接下来就是比较难的平坦化控制流了。  

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSYib77MT7eVxkmwgaRLhXByjCfaXRzdic430fX4P3SQep3M4BkjIa34UGouiaxCmIzqrXZ9aS9klz6g/640?wx_fmt=png)

思路是这样，可能讲的有点笼统，直接看代码的注释吧。  

```
function replaceForStatement(path) {
    var node = path.node;
    //上一个节点
    var prevSiblingPath = path.getPrevSibling();
    //判断有无这些属性，不引起报错
    if (prevSiblingPath.container ==undefined ||
        prevSiblingPath.container[0].declarations ==undefined ||
        prevSiblingPath.container[0].declarations[0].init ==undefined ||
        prevSiblingPath.container[0].declarations[0].init.object ==undefined ||
        prevSiblingPath.container[0].declarations[0].init.object.object ==undefined
    )
      return;
    //判断上一个节点是否是EMF这种类型
    if (prevSiblingPath.container[0].declarations[0].init.object.object.callee.property.name != "EMf")
        return;
    var body = node.body.body;
    //判断当前节点的body[0]属性 和 body[0]/discriminant是否存在
    if(!types.isSwitchStatement(body[0] ||
        !types.isIdentifier(body[0].discriminant)))
        return;
    var swithStm = body[0];
    //提取上个节点中的value作为初始参数
    var arg = prevSiblingPath.container[0].declarations[0].init;
    let init_arg_f = arg.object.property.value;
    let init_arg_s = arg.property.value;
    var init_arg = AJgjJ.EMf()[init_arg_f][init_arg_s];
    //提取for节点中的if判断参数的value作为判断参数
    let break_arg_f = node.test.right.object.property.value;
    let break_arg_s = node.test.right.property.value;
    var break_init = AJgjJ.EMf()[break_arg_f][break_arg_s];
    //拿到swith所有的cases
    var caseList = swithStm.cases;
    var resultBody =[];
    //按照他的逻辑写for循环
    for (var i=0;i<caseList.length;i++){
        for (;init_arg !== break_init;)
            {
                var case_arg_f = caseList[i].test.object.property.value;
                var case_arg_s = caseList[i].test.property.value;
                var case_init = AJgjJ.EMf()[case_arg_f][case_arg_s];
                if (init_arg == case_init){
                    var targetBody = caseList[i].consequent;
                    //删除break 和 SKD = AJgjJ.EMf()[4][16]; 这种无用代码
                    if (types.isBreakStatement(targetBody[targetBody.length - 1]) &&
                        types.isExpressionStatement(targetBody[targetBody.length - 2]) &&
                        targetBody[targetBody.length - 2].expression.right.object.object.callee.object.name == "AJgjJ"){
                        var change_arg_f = targetBody[targetBody.length - 2].expression.right.object.property.value;
                        var change_arg_s = targetBody[targetBody.length - 2].expression.right.property.value;
                        init_arg = AJgjJ.EMf()[change_arg_f][change_arg_s];
                        targetBody.pop();
                        targetBody.pop();
                    }
                    //删除break
                    else if (types.isBreakStatement(targetBody[targetBody.length - 1])){
                        targetBody.pop();
                    }
                    resultBody = resultBody.concat(targetBody);
                    break;
                }else{
                    break;
                }
            }
    }
    //替换for节点，多个节点替换一个节点用replaceWithMultiple
  path.replaceWithMultiple(resultBody);
  //删除上一个节点
  prevSiblingPath.remove();
}

```

这样控制流平坦化也去除了  

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSYib77MT7eVxkmwgaRLhXByYzXOEeCDkMV0pa2Q0FahbFdVV1U65m7xMeD2k5WgMUcTvfEHLzED0w/640?wx_fmt=png)

其实也是耐心分析他的结构，多试试就能搞出来了，搞完后发现也不是非常的难。  

操作完后 就剩 8800 多行代码了，可以很清晰的调试了，我们还可以验证下我们替换是不是正确的

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSYib77MT7eVxkmwgaRLhXBys3n4cFkXicoqR6AdHp929IHgXsg3BgGTAcibu7fJDsqicy3YyNtXJMOog/640?wx_fmt=png)

找到需要 geetest 的网站，用 reres 直接替换成本地的 js

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSYib77MT7eVxkmwgaRLhXByoxLLyEVYAbyS9NowHDicXNl8a6eGkEB0xib4WWW6n4Evg228mtGQURBQ/640?wx_fmt=png)

这就替换成功了  

然后我们试试验证码功能是否正常

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSYib77MT7eVxkmwgaRLhXBypz4jX0eiakZGTBRjGxJpibr5AZ1Z6MMhhyWGJZiaKPb1icDbhuwkFa3gdA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/J7WeDiaX8bpSYib77MT7eVxkmwgaRLhXByyQjmaYMhMY9Zm4w9GTibY52CG1LLPVV97cF7IibiaFMJdd2GcOLhOKKPw/640?wx_fmt=png)

正常的发送验证码说明我们的反混淆就成功了！（电话号码我随便填的。。）

接下来要调试的话就很简单了。

就不张开叙述了，请自行操作。

马上要答辩了！毕业设计搞完前就不更新文章了!

做了个只需修改 fullpagex.x.x.js 的里面数组方法名的脚本，即可一键反混淆所有类似的 fullpage

完整代码请在公众号回复 geetest 获取
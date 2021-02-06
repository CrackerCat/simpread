> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/UDuFCS6WBrSaDCXxh4YRrw)

一、测试 gps 定位功能的方法探讨
------------------

     在安卓 App gps 定位开发过程中，往往需要进行 GPS 定位的测试。由于室内 gps 信号被挡住了，所以如果需要测试 gps 定位的代码，要么跑到室外去等待 gps 定位更新或者使用安卓 sdk 中提供的 "MOCK_LOCATION" 功能编写额外的测试代码来模拟 gps 位置更新。如果从系统定制的角度出发, 也可以根据 gps 上报的流程来模拟 gps 上报的逻辑实现模拟 gps 的定位数据。

二、修改系统实现模拟 gps 数据上报
-------------------

### 2.1 安卓系统上报 gps 数据流程总结

      在之前的文章[安卓 10 源码开发定制 (21)GPS 定位研究 (2)gps 位置改变监听源码分析](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247484306&idx=1&sn=a3303165ca78ce9e7b633d4ea1f8ee58&scene=21#wechat_redirect)中已经分析安卓系统中 gps 数据上报的大概流程:

    1. 当 gps 位置改变更新,com_android_server_location_GnssLocationProvider.cpp 中通过反射调用 GnssLocationProvider.java 中的 reportLocation 方法将 gps 数据上报到安卓系统 java 层。

   2. GnssLocationProvider.java 中使用 LocationProvider.java 提供的 onReportLocation 方法将 gps 数据上报到 GPS 位置管理器。

 3.GPS 位置管理器 LocationProvider.java 调用 LocationManagerService.java 提供的方法 handleLocationChangedLocked 将 gps 数据传到 LocationManagerService.java 中。

    4. LocationManagerService.java 中通过 GPS_PROVIDER 关键字获取缓存的注册监听器 Receiver 列表，然后遍历每一个监听器并调用监听的 callLocationChangedLocked 方法来通知客户端位置更新。

### 2.2 修改 GPS 数据的方案构想

    参考以上 gps 上报的流程，可以在 LocationManagerService 中使用定时器。定时获取设置的 gps 坐标，然后通过名称 GPS_PROVIDER 获取缓存的 GPS 位置提供管理器对象，然后调用 onReportLocation 方法达到模拟上报的目的。

### 2.3 实现参考代码

    基于以上构想，通过修改系统测试高德地图验证成功。以下是验证的核心参考代码:

```
 ///ADD START
     //LocationManagerService中添加模拟上报位置的接口
    //主动上报GPS位置信息
    private void  reportLocation()
    {

        Timer time=new Timer();
        TimerTask timerTask=new TimerTask() {
            @Override
            public void run() {
                Location location=new Location(LocationManager.GPS_PROVIDER);
                location.setLongitude(117.27127064678953);
                location.setLatitude(39.138059417665445);
				
                location.setElapsedRealtimeNanos(System.nanoTime());
				location.makeComplete();
				LocationProvider mylocProvider=getLocationProviderLocked(LocationManager.GPS_PROVIDER);
				if(mylocProvider!=null)
				{
				    mylocProvider.onReportLocation(location);
					XLog.d("LocationManagerService","onReportLocation success");
				}else{
                    XLog.d("LocationManagerService","mylocProvider is NULL");
				}
				
            }
        };
        time.schedule(timerTask,10*1000,5*1000);
    }
    ///ADD END


```

本文的目的只有一个就是学习更多的安卓系统方面的知识，如果有人利用本文技术去进行非法商业获取利益带来的法律责任都是操作者自己承担，和本文以及作者没关系。

安卓系统、安卓 ndk 开发、安卓应用安全和逆向分析相关等 IT 知识分享，系统定制、frida、xposed(sandhook、edxposed)、加固刷机交流等等。微信搜索公众号 "QDOIRD88888" 或者扫描以下二维码关注公众号。第一时间接收文章更新。

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432sjSMO5rOyZfpibCSD6Kv3YVaOaEftn2nibNSSmd26LrZRa2JtibvzrRIdticrOn3cs3I1P7WriaGlBpg/640?wx_fmt=png)
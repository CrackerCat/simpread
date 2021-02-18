> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/0IkD95SuBkX_PZYtIzlkRg)

**一、添加 IGetWifiMacInterface.aidl 和 GetWifiMacServiceManager**

   在如下目录中创建目录 **wifiex** 目录:

```
frameworks\base\core\java\android

```

   将上文中 Android Studio 创建好的 **IGetWifiMacInterface.aidl** 和 **GetWifiMacServiceManager.java** 文件复制到 wifiex 目录。如下图所示:  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432k0AkqrG9E5icShT0ANM4qKBWOticl98VqhRqJzxmNzAGmeRHrYOAUkG1NT6ofbvQavD5XzI2gljqg/640?wx_fmt=png)

   在文件 **"frameworks\base\Android.bp"** 中将 aidl 文件加入编译链。添加之后如下:  

```
        //...省略
        "core/java/android/app/IAlarmListener.aidl",
        "core/java/android/app/IAlarmManager.aidl",
        ///ADD START
        "core/java/android/wifiex/IGetWifiMacInterface.aidl",
        ///ADD END
        "core/java/android/app/IAppTask.aidl",
        "core/java/android/app/IApplicationThread.aidl",
        //...省略

```

   在 "**Context.java**" 文件中仿照其他服务的名称添加 **GetWifiMacService** 对应的名称。**Context.java** 文件路径位于:  

> **frameworks\base\core\java\android\content\Context.java**

**Context.java** 文件添加之后的部分代码如下:

```
     //...省略
     /**
     * Use with {@link #getSystemService(String)} to retrieve an
     * {@link android.os.image.DynamicSystemManager}.
     * @hide
     */
    public static final String DYNAMIC_SYSTEM_SERVICE = "dynamic_system";
    ///ADD START
    public static final String GET_WIFI_MAC_SERVICE="get_wifi_mac";
    ///ADD END
    //...省略

```

**二、创建 GetWifiMacService**

 在安卓系统源码目录 "frameworks\base\services\core\java\com\android\server" 中添加目录 wifiex。

   将 Android Studio 创建好的 **GetWifiMacService** 文件复制到 wifiex 目录下面。如下图所示:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432k0AkqrG9E5icShT0ANM4qKJZxoduS9iawib55E3KGxlagVgNzHIJoZXXZuRRUhqHttSibHCo8zlPdMg/640?wx_fmt=png)

**三、在 SystemServer 中启动 GetWifiMacService**

    **SystemServer** 类文件路径如下:

> **frameworks\base\services\java\com\android\server\SystemServer.java**

   在 SystemServer 类中的 startOtherServices 函数添加以下注册 GetWifiMacService 的逻辑代码。如下:

```
//...省略
private void startOtherServices() {
//...省略
   ///ADD START
  traceBeginAndSlog("StartGetWifiMacService");
  GetWifiMacService getWifiMacService = new GetWifiMacService();
  ServiceManager.addService(Context.GET_WIFI_MAC_SERVICE, getWifiMacService);
  traceEnd();
  ///ADD END
//...省略
}
//...省略

```

**四、绑定 GetWifiMacServiceManager 和 GetWifiMacService**

     将 **GetWifiMacServiceManager** 和 **GetWifiMacService** 绑定的目的主要是想想让 **app** 通过以下代码访问 **GetWifiMacService** 的接口。访问的代码形式:

```
GetWifiMacServiceManager manager=context.getSystemService(Context.GET_WIFI_MAC_SERVICE);
manager.getWifiMac();

```

   参考之前分析的 LocationManager 的注册流程, 注册 GetWifiMacServiceManager 。LocationManager 的初始流程参考文章: [安卓 10 源码开发定制 (21)GPS 定位研究 (1)LocationManager 对象获取流程](http://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247484289&idx=1&sn=75b888e61363e48c0721733771d92a04&chksm=ce0752c4f970dbd22c57c62ebe62ec0cbd941c12b62f4bc95b7fffca90911fa0653d9c757050&scene=21#wechat_redirect)。

    安卓系统中绝大部分的系统服务的 Manager 的注册都在 SystemServiceRegistry 类中实现。SystemServiceRegistry 类源码中路径如下:

> **frameworks/base/core/java/android/app/SystemServiceRegistry.java**

   在该类中添加如下的代码完成 GetWifiMacServiceManager 的注册。核心代码如下:

```
 //...省略
 registerService(Context.DYNAMIC_SYSTEM_SERVICE, DynamicSystemManager.class,
                new CachedServiceFetcher<DynamicSystemManager>() {
                    @Override
                    public DynamicSystemManager createService(ContextImpl ctx)
                            throws ServiceNotFoundException {
                        IBinder b = ServiceManager.getServiceOrThrow(
                                Context.DYNAMIC_SYSTEM_SERVICE);
                        return new DynamicSystemManager(
                                IDynamicSystemService.Stub.asInterface(b));
                    }});
        ///ADD START
         registerService(Context.GET_WIFI_MAC_SERVICE, GetWifiMacServiceManager.class,
                new CachedServiceFetcher<GetWifiMacServiceManager>() {
                    @Override
                    public GetWifiMacServiceManager createService(ContextImpl ctx)
                            throws ServiceNotFoundException {
                           IBinder b = ServiceManager.getService(Context.GET_WIFI_MAC_SERVICE);
                           return b == null ? null : new GetWifiMacServiceManager(ctx,IGetWifiMacInterface.Stub.asInterface(b));
                    }});
        ///ADD END
 //...省略

```

**五、为新增的 service 配置 seandroid 策略**

    默认新增的 **GetWifiMacService** 服务的 **seandroid** 类型为 **default_android_service**。如果不配置第三方 App 访问不了。具体配置如下:

1.  **在文件 system\sepolicy\private\service_contexts 中添加如下内容**
    
    ```
    # ///ADD START
    get_wifi_mac                                  u:object_r:get_wifi_mac_service:s0
    # ///ADD END
    
    ```
    
2.  **在文件** system\sepolicy\prebuilts\api\29.0\private\service_contexts  
    
    **中添加如下内容**
    
    ```
    # ///ADD START
    get_wifi_mac                                  u:object_r:get_wifi_mac_service:s0
    # ///ADD END
    
    ```
    
3.  **在文件 system\sepolicy\public\service.te 中添加如下内容**
    
    ```
    # ///ADD START
    type get_wifi_mac_service, app_api_service, ephemeral_app_api_service, system_server_service, service_manager_type;
    # ///ADD END
    
    ```
    
4.  **在文件 system\sepolicy\prebuilts\api\29.0\public\service.te 中添加如下内容**  
    

```
# ///ADD START
type get_wifi_mac_service, app_api_service, ephemeral_app_api_service, system_server_service, service_manager_type;
# ///ADD END

```

**六、编译刷机**

 执行如下命令进行编译刷机包，然后进行刷机操作。

```
qiang@ubuntu:~/lineageOs$ source build/envsetup.sh 
qiang@ubuntu:~/lineageOs$ breakfast oneplus3
qiang@ubuntu:~/lineageOs$ make update-api
qiang@ubuntu:~/lineageOs$ brunch oneplus3

```

以下是终端命令查看是否添加服务成功。如下所示:

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432KKrqLJGdkeviaCU3MOia180ibH1b0bvlhoQYkttd1SK9jHphKgCIjfRltlpSKEIYGBTmjdAdtBLdcQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjguCKrYZQfRXxK6hibNjOh10JibAdHj553dxk3PmoyUibjDCGcNdq3IQBKA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgibOWZXyrOLic5KPJ2y9A1gznt4xUa1H7MEhlgmcQgnE3IJvphZfOezfA/640?wx_fmt=png)  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgxGibv8NMwbmJuQo55Ry33RkQj6WTGwwyXgrcduXPL3xnUWeLUa3cDvA/640?wx_fmt=png)
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/0iuPZFR-58t6gTF3rJT97w)

  

一、App 中使用 LocationManager 获取 gps 定位信息
=====================================

安卓 app 中使用如下代码获取当前设备的 GPS 定位信息，参考代码如下:

```
public class MainActivity extends Activity
{
    //声明LocationManager
    LocationManager locationManager=null;
    private String provider=null;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        locationManager=(LocationManager)getSystemService(Context.LOCATION_SERVICE);
        //获取所有可用的位置提供器
        List<String>providerList=locationManager.getProviders(true);
        if(providerList.contains(LocationManager.GPS_PROVIDER)){
            provider=LocationManager.GPS_PROVIDER;
        }
        else if(providerList.contains(LocationManager.NETWORK_PROVIDER)){
            provider=LocationManager.NETWORK_PROVIDER;
        }
        else{
            //当没有可用的位置提供器时，弹出消息通知用户
            Toast.makeText(this, "未找到可用的手机位置提供器", Toast.LENGTH_SHORT).show();
            return;
        }
        //获取当前设备最后一次刷新的gps位置信息
        Location location=locationManager.getLastKnownLocation(provider);
        if(location!=null){
            //显示当前设备的位置信息
            Log.d(TAG, "current location ("+location.getLatitude()+","+location.getLongitude()+")");
            
        }
        locationManager.requestLocationUpdates(provider, 1000, 1, locationListener);
    }
    
    protected void onDestroy(){
        super.onDestroy();
        if(locationManager!=null){
            //关闭程序时将监听移除
            locationManager.removeUpdates(locationListener);
        }
    }
    //创建位置改变监听器,gps位置信息改变的时候作为回调通知
    LocationListener locationListener=new LocationListener(){

        @Override
        public void onLocationChanged(Location location) {
            // TODO Auto-generated method stub
            //更新当前设备的位置信息
            Log.d(TAG, "current location ("+location.getLatitude()+","+location.getLongitude()+")");
        }

        @Override
        public void onProviderDisabled(String provider) {
            // TODO Auto-generated method stub
            
        }

        @Override
        public void onProviderEnabled(String provider) {
            // TODO Auto-generated method stub
            
        }

        @Override
        public void onStatusChanged(String provider, int status, Bundle extras) {
            // TODO Auto-generated method stub
            
        }
        
    };
}


```

由以上代码可知，**LocationManager** 是通过当前 **Activity** 提供的方法 **getSystemService(Context.LOCATION_SERVICE)** 来获取位置管理器。

二、LocatoinManager 的初始流程追踪
=========================

1.Activity 中 getSystemService 追踪
--------------------------------

安卓 10 源码中 **Activity** 类文件路径位于:**frameworks/base/core/java/android/app/Activity.java** 。在该文件中可以搜索到 **getSystemService** 的实现，如下所示:

```
///...省略
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback, WindowControllerCallback,
        AutofillManager.AutofillClient, ContentCaptureManager.ContentCaptureClient {
      ///...省略
      
      //此处实现了getSystemService，该方法会根据name来判断执行的路径。由于
      //LocationManager是通过传入Context.LOCATION_SERVICE进行初始化,所以该方法       //中将执行super.getSystemService(name);
       @Override
    public Object getSystemService(@ServiceName @NonNull String name) {
        if (getBaseContext() == null) {
            throw new IllegalStateException(
                    "System services not available to Activities before onCreate()");
        }

        if (WINDOW_SERVICE.equals(name)) {
            return mWindowManager;
        } else if (SEARCH_SERVICE.equals(name)) {
            ensureSearchManager();
            return mSearchManager;
        }
        //调用父类的getSystemService
        return super.getSystemService(name);
    }
      ///...省略
      
}


```

通过以上代码分析可以知道获取 **LocationManager** 的时候执行的是 **super.getSystemService(name);** , 说明 **Activity** 中调用的是父类的 **getSystemService** 方法。由于 **Activity** 继承 **ContextThemeWrapper**，所以下面将分析 **ContextThemeWrapper** 中 **super.getSystemService(name)** 方法。

2.ContextThemeWrapper 中 getSystemService 追踪
-------------------------------------------

**ContextThemeWrapper** 类在安卓源码中的文件路径位于:**frameworks\base\core\java\android\view\ContextThemeWrapper.java**。该类中 getSystemService 方法实现如下:

```
///...省略
public class ContextThemeWrapper extends ContextWrapper {
   
///...省略
 
    @Override
    public Object getSystemService(String name) {
        if (LAYOUT_INFLATER_SERVICE.equals(name)) {
            if (mInflater == null) {
                mInflater = LayoutInflater.from(getBaseContext()).cloneInContext(this);
            }
            return mInflater;
        }
        //调用getBaseContext返回的对象再调用getSystemService
        return getBaseContext().getSystemService(name);
    }


///...省略

}


```

**getBaseContext** 函数在 **ContextThemeWrapper** 类中并未搜索到。跳转到他的父类 "**ContextWrapper** " 中搜索到 **getBaseContext** 方法，实现如下:

```
///...省略
 public class ContextWrapper extends Context {
    @UnsupportedAppUsage
    Context mBase;

    public ContextWrapper(Context base) {
        mBase = base;
    }
    
    /**
     * Set the base context for this ContextWrapper.  All calls will then be
     * delegated to the base context.  Throws
     * IllegalStateException if a base context has already been set.
     * 
     * @param base The new base context for this wrapper.
     */
    protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }

    /**
     * @return the base context as set by the constructor or setBaseContext
     */
    public Context getBaseContext() {
        return mBase;
    }
   ///...省略    
}
    


```

从以上代码分析可知最终调用的是 **mBase.getSystemService** 。所以目前关键要找到 **mBase** 真正对应的实现类。**ContextWrapper** 中存在两种方法初始化 **mBase** ，一种是构造 **ContextWrapper** 对象的时候传入参数; 另一种是调用 **attachBaseContext** 方法进行初始化 **mBase**。在源码中搜索了一下没找到突破点，下面换点思路来获取 **mBase** 的真正实现类。由于 **Activity** 中可以调用 **getBaseContext** 方法获取 mBase 对象，所有我们下面写一个测试的 App，在 **Activity** 中调用 **getBaseContext** 获取对象打印类名类获取 mBase 真正的实现类类。

3、mBase 真正实现类追踪
---------------

在 App 的 Activity 中，使用如下代码获取 mBase 的真正实现类:

```
 String className=getBaseContext().getClass().getName();
Log.d(TAG,"class+className);


```

运行测试 App 之后，打印日志如下:

```
2021-01-24 22:48:47.267 7855-7855/com.android.helloworld001 D/MainActivity: className=======>android.app.ContextImpl


```

从日志中可以知道 **mBase** 的真正实现类为 "**android.app.ContextImpl**"。

4、ContextImpl 分析
----------------

安卓源码中 **ContextImpl** 源码路径:**frameworks/base/core/java/android/app/ContextImpl.java**。在该文件中 **getSystemService** 的实现如下所示:

```
///...省略
class ContextImpl extends Context {

  ///...省略
  //可以看到追踪调用的是SystemServiceRegistry.getSystemService(this, name)
   @Override
    public Object getSystemService(String name) {
        return SystemServiceRegistry.getSystemService(this, name);
    }
  ///...省略

}
///...省略


```

由以上代码可知 **getSystemService** 追踪调用的是 **SystemServiceRegistry.getSystemService** 方法来获取。下面将分析 **SystemServiceRegistry** 类。

5、SystemServiceRegistry 分析
--------------------------

安卓源码中 **SystemServiceRegistry 分析**源码路径:**frameworks/base/core/java/android/app/SystemServiceRegistry.java**。在该文件中 **getSystemService** 的实现如下所示:

```
///...省略
final class SystemServiceRegistry { 

  ///...省略
  //可以看到追踪调用的是SYSTEM_SERVICE_FETCHERS
    /**
     * Gets a system service from a given context.
     */
    public static Object getSystemService(ContextImpl ctx, String name) {
        ServiceFetcher<?> fetcher = SYSTEM_SERVICE_FETCHERS.get(name);
        return fetcher != null ? fetcher.getService(ctx) : null;
    }
  ///...省略

}
///...省略


```

以上代码中可以分析可知最终是 SYSTEM_SERVICE_FETCHERS 根据 name 缓存的对象来获取服务对象, 以下我将 getSystemService 中相关调用的代码抽取如下。

```
 //此处定义SYSTEM_SERVICE_FETCHERS作为缓存注册的服务对象。
 private static final Map<String, ServiceFetcher<?>> SYSTEM_SERVICE_FETCHERS =
            new ArrayMap<String, ServiceFetcher<?>>();
            
            
 
 
 //SystemServiceRegistry中的静态代码区，注册了许多的Manager对象比如LocationManager。平时我们使用的各种Manager基本都在这个地方注册了
 static {
        //...省略
            registerService(Context.LOCATION_SERVICE, LocationManager.class,
                new CachedServiceFetcher<LocationManager>() {
            @Override
            public LocationManager createService(ContextImpl ctx) throws ServiceNotFoundException {
                IBinder b = ServiceManager.getServiceOrThrow(Context.LOCATION_SERVICE);
                return new LocationManager(ctx, ILocationManager.Stub.asInterface(b));
            }});
            //...省略
 }
 
    //registerService 实现代码
    private static <T> void registerService(String serviceName, Class<T> serviceClass,
            ServiceFetcher<T> serviceFetcher) {
        SYSTEM_SERVICE_NAMES.put(serviceClass, serviceName);
        SYSTEM_SERVICE_FETCHERS.put(serviceName, serviceFetcher);
    }

    /**
     * Base interface for classes that fetch services.
     * These objects must only be created during static initialization.
     */
    static abstract interface ServiceFetcher<T> {
        T getService(ContextImpl ctx);
    }
 


```

以上就是 LocationManager 的获取整个流程。如果想看其他 Manager 的获取流程, 其实都是一样的流程。

三、总结
====

LocationManager 的获取可以概括为如下调用序列:

```
1.Activity.getSystemService(Context.LOCATION_SERVICE)->
2.ContextThemeWrapper->getSystemService(Context.LOCATION_SERVICE)->
3.ContextWrapper.getSystemService(Context.LOCATION_SERVICE)->
4.ContextImpl.getSystemService(Context.LOCATION_SERVICE)->
5.SYSTEM_SERVICE_FETCHERS.get(Context.LOCATION_SERVICE)->
6.ServiceFetcher.getService()


```

[[上一篇] 玩转安卓 10 源码开发定制 (20)libc 中添加日志输出接口](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247484270&idx=1&sn=cd3456a66a3720b42732d76b4dd5d549&scene=21#wechat_redirect)

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew54322cfAhHjqkULYArSo8PGicfqANQutje4aAFKNMl021geqymUxruiavdbGaTzpSiaLEGqibGSZfNMsnxA/640?wx_fmt=png)

**安卓系统、安卓 ndk 开发、安卓应用安全和逆向分析相关等 IT 知识分享，系统定制、frida、xposed(sandhook、edxposed) 系统集成、加固、脱壳、刷机交流等等。微信搜索公众号 "QDOIRD88888" 或者扫描以下二维码关注公众号。第一时间接收文章更新。**

![](https://mmbiz.qpic.cn/mmbiz_jpg/9vkUcew5430HpkFIRvrbTB68PwHwicZh5YG5aXIeibCxz29DDYLdQrf3ibjZxrCHST9r0zicRIsBYJ8HasrIwJU55Q/640?wx_fmt=jpeg)

扫一扫关注公众号
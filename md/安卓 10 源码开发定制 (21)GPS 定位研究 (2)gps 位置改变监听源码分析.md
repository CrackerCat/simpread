> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/PpDjGU0cJFKRUhbDqxBnCw)

一、安卓中 GPS 定位更新监控
----------------

在安卓很多提供定位服务的 App 应用程序中，不仅需要获取当前的位置信息，还需要监视位置的变化，在位置改变时获取到当前的位置信息。gps 定位中 LocationManager 提供了一种便捷、高效的位置监视方法 requestLocationUpdates(String provider, long minTime, float minDistance,LocationListener listener)，可以根据位置的距离变化和时间间隔设定，产生位置改变事件的条件，这样可以避免因微小的距离变化而产生大量的位置改变事件 ，LocationManager 中设定监听位置变化的参考代码如下:

```
    locationManager.requestLocationUpdates(LocationManager.GPS_PROVIDER, 0, 0, locationListener);


```

以下将分析 LocationManager 中 requestLocationUpdates 到 LocationManagerService 中 requestLocationUpdates 的流程来分析 GPS 定位中的交互。

二、requestLocationUpdates 调用流程追踪
-------------------------------

### 2.1 LocationManager 中 requestLocationUpdates 调用分析

LocationManager 源码文件位于:

> frameworks\base\location\java\android\location\LocationManager.java

LocationManager 中对外提供了很多 API 接口供 App 调用来实现 gps 定位。其中位置改变监控的接口的实现代码如下:

```
   //从代码中可以看到调用了 requestLocationUpdates(LocationRequest request, LocationListener listener,Looper looper, PendingIntent intent)
    @RequiresPermission(anyOf = {ACCESS_COARSE_LOCATION, ACCESS_FINE_LOCATION})
    public void requestLocationUpdates(@NonNull String provider, long minTime, float minDistance,
            @NonNull LocationListener listener) {
        checkProvider(provider);
        checkListener(listener);

        LocationRequest request = LocationRequest.createFromDeprecatedProvider(
                provider, minTime, minDistance, false);
        requestLocationUpdates(request, listener, null, null);
    }
 
    // 该方法最终调用了mService.requestLocationUpdates和LocationManagerService进行binder通信。
    //所以requestLocationUpdates的真正实现存在于LocationManagerService
      @UnsupportedAppUsage
    private void requestLocationUpdates(LocationRequest request, LocationListener listener,
            Looper looper, PendingIntent intent) {

        String packageName = mContext.getPackageName();

        // wrap the listener class
        ListenerTransport transport = wrapListener(listener, looper);

        try {
            mService.requestLocationUpdates(request, transport, intent, packageName);
       } catch (RemoteException e) {
           throw e.rethrowFromSystemServer();
       }
    }


```

在 LocationManager 中 mService 定义如下:

```
private final ILocationManager mService;


```

从定义可以看出类型为 ILocationManager。在源码中没有搜索到 ILocationManager.java 类, 存在 ILocationManager.aidl，aidl 文件在源码编译期间会生成对应 java 文件。源码中 aidl 具体的 java 代码如何查看可以参考: [安卓 10 源码开发定制 (22) 使用 jadx 反编译工具查看 aidl 文件实现代码](https://mp.weixin.qq.com/s?__biz=Mzg2MjU1NDE1NA==&mid=2247484298&idx=1&sn=f3ddd14a279c9c010c4e49851217f325&scene=21#wechat_redirect)。

### 2.2 LocationManagerService 中 requestLocationUpdates 调用分析

LocationManagerService 源码文件位于:

> frameworks\base\services\core\java\com\android\server\LocationManagerService.java

该文件中 requestLocationUpdates 方法的 关键实现如下:

```
 @Override
    public void requestLocationUpdates(LocationRequest request, ILocationListener listener,PendingIntent intent, String packageName) {
               //...省略
                Receiver receiver;
                if (intent != null) {
                    receiver = getReceiverLocked(intent, pid, uid, packageName, workSource,
                            hideFromAppOps);
                } else {
                    //LocationManager调用requestLocationUpdates的时候，intent=null。所以会执行如下的代码分支,getReceiverLocked会创建一个Receiver对象来管理app注册的监听。
                    receiver = getReceiverLocked(listener, pid, uid, packageName, workSource,
                            hideFromAppOps);
                }
                //该方法更新receiver的一些参数，方便后续收到位置更新之后进行遍历调用
                requestLocationUpdatesLocked(sanitizedRequest, receiver, uid, packageName);
            //...省略
        }
    }

    @GuardedBy("mLock")
    private void requestLocationUpdatesLocked(LocationRequest request, Receiver receiver,
            int uid, String packageName) {
         //...省略
        UpdateRecord record = new UpdateRecord(name, request, receiver);
        if (D) {
            Log.d(TAG, "request " + Integer.toHexString(System.identityHashCode(receiver))
                    + " " + name + " " + request + " from " + packageName + "(" + uid + " "
                    + (record.mIsForegroundUid ? "foreground" : "background")
                    + (isThrottlingExemptLocked(receiver.mCallerIdentity)
                    ? " [whitelisted]" : "") + ")");
        }

        UpdateRecord oldRecord = receiver.mUpdateRecords.put(name, record);
        if (oldRecord != null) {
            oldRecord.disposeLocked(false);
        }

        if (!provider.isUseableLocked() && !isSettingsExemptLocked(record)) {
            // Notify the listener that updates are currently disabled - but only if the request
            // does not ignore location settings
            receiver.callProviderEnabledLocked(name, false);
        }
         
       receiver.updateMonitoring(true);
    }

 //getReceiverLocked方法会缓存LocationManager传过来的位置监听器，后续有位置改变就会遍历mReceivers中的缓存对象，然后通知通知位置改变
 @GuardedBy("mLock")
    private Receiver getReceiverLocked(ILocationListener listener, int pid, int uid,String packageName, WorkSource workSource, boolean hideFromAppOps) {
        IBinder binder = listener.asBinder();
        Receiver receiver = mReceivers.get(binder);
        if (receiver == null) {
            //创建接收对象
            receiver = new Receiver(listener, null, pid, uid, packageName, workSource,
                    hideFromAppOps);
            if (!linkToListenerDeathNotificationLocked(receiver.getListener().asBinder(),
                    receiver)) {
                return null;
            }
            //缓存接收对象
            mReceivers.put(binder, receiver);
        }
        return receiver;
    }


```

在以上分析中，位置监听 api 调用之后追踪会创建 Receiver 对象进行包装。Receiver 类定义在 LocationManagerService.java 中，以下是该类中和位置变化监听相关的部分代码:

```
 private final class Receiver extends LinkedListenerBase implements PendingIntent.OnFinished {
        //...省略
        //缓存位置改变的监听回调
        private final ILocationListener mListener;
        //...省略
        //唯一标识
        private final Object mKey;

        final HashMap<String, UpdateRecord> mUpdateRecords = new HashMap<>();
       //...省略

        private Receiver(ILocationListener listener, PendingIntent intent, int pid, int uid,String packageName, WorkSource workSource, boolean hideFromAppOps) {
            super(new CallerIdentity(uid, pid, packageName), "LocationListener");
            mListener = listener;
            mPendingIntent = intent;
            if (listener != null) {
                mKey = listener.asBinder();
            } else {
                mKey = intent;
            }
           //省略
        }

       //...省略
       //位置改变调用接口，最终会调用到mListener.onLocationChanged
        public boolean callLocationChangedLocked(Location location) {
            if (mListener != null) {
                try {
                    mListener.onLocationChanged(new Location(location));
                    // call this after broadcasting so we do not increment
                    // if we throw an exception.
                    incrementPendingBroadcastsLocked();
                } catch (RemoteException e) {
                    return false;
                }
            }
            //...省略
        }
        //...省略
    }


```

由以上 Receiver 的分析可以知道，如果有位置变化会调用 Receiver.callLocationChangedLocked(location)。

在 LocationManagerService 中通过 "callLocationChangedLocked" 定位到被调用的地方如下:

```
//LocationManagerService.java中的方法
 private void handleLocationChangedLocked(Location location, LocationProvider provider) {
       //...省略
                    if (!receiver.callLocationChangedLocked(notifyLocation)) {
                        Slog.w(TAG, "RemoteException calling onLocationChanged on "
                                + receiver);
                        receiverDead = true;
                    }
       //...省略   
    }


```

在 LocationManagerService 中通过 "handleLocationChangedLocked" 定位到被调用的地方在 LocationProvider 类中，该类定义在 LocationManagerService 中。核心关键代码如下:

```
 private class LocationProvider implements AbstractLocationProvider.LocationProviderManager {

        
        //...省略
        //根据传入的gps提供商名称创建位置提供者
       private LocationProvider(String name) {
            this(name, false);
        }
       //根据传入的gps提供商名称创建位置提供者
        private LocationProvider(String name, boolean isManagedBySettings) {
            mName = name;
            mIsManagedBySettings = isManagedBySettings;

            mProvider = null;
            //...省略
        }

        // called from any thread
        @Override
        public void onReportLocation(Location location) {
            // no security check necessary because this is coming from an internal-only interface
            // move calls coming from below LMS onto a different thread to avoid deadlock
            mHandler.post(() -> {
                synchronized (mLock) {
                    //此处调用了
                    handleLocationChangedLocked(location, this);
                }
            });
        }

      //...省略
    }


```

由以上 LocationProvider 的构造函数可以分析出系统初始化的时候会构造一个 GPS_PROVIDER 的位置提供者用来提供 GPS 定位服务。通过在 LocationManagerService 初始化的分析找到了位置提供者初始化的逻辑。相关核心代码如下:

```
//LocationManagerService.java文件中

//此处调用了初始化
 public void systemRunning() {
        synchronized (mLock) {
            initializeLocked();
        }
    }

    @GuardedBy("mLock")
    private void initializeLocked() {
         //...省略
        // prepare providers
        //调用了初始化位置提供者函数
        initializeProvidersLocked();
  }
    private void initializeProvidersLocked() {
        //...省略
        if (GnssLocationProvider.isSupported()) {
            // Create a gps location provider
            //创建gps位置提供管理器
            LocationProvider gnssProviderManager = new LocationProvider(GPS_PROVIDER, true);
            mRealProviders.add(gnssProviderManager);
            addProviderLocked(gnssProviderManager);
            //创建Gnss位置提供者，实际的位置数据在这个provider中进行通知
            GnssLocationProvider gnssProvider = new GnssLocationProvider(mContext,
                    gnssProviderManager,
                    mHandler.getLooper());
            //gps位置提供管理器和gnss数据提供者关联上        
            gnssProviderManager.attachLocked(gnssProvider);

            mGnssSystemInfoProvider = gnssProvider.getGnssSystemInfoProvider();
            mGnssBatchingProvider = gnssProvider.getGnssBatchingProvider();
            mGnssMetricsProvider = gnssProvider.getGnssMetricsProvider();
            mGnssCapabilitiesProvider = gnssProvider.getGnssCapabilitiesProvider();
            mGnssStatusProvider = gnssProvider.getGnssStatusProvider();
            mNetInitiatedListener = gnssProvider.getNetInitiatedListener();
            mGnssMeasurementsProvider = gnssProvider.getGnssMeasurementsProvider();
            mGnssMeasurementCorrectionsProvider =
                    gnssProvider.getGnssMeasurementCorrectionsProvider();
            mGnssNavigationMessageProvider = gnssProvider.getGnssNavigationMessageProvider();
            mGpsGeofenceProxy = gnssProvider.getGpsGeofenceProxy();
        }
        //省略
    }


```

通过以上位置提供者初始化的逻辑可以分析到创建的 gps 位置管理器使用 GnssLocationProvider 来提供具体的位置更新服务。

### 2.3 GnssLocationProvider 中位置更新通知流程分析

GnssLocationProvider 源码文件路径位于:

> frameworks\base\services\core\java\com\android\server\location\GnssLocationProvider.java

以下是和位置改变更新通知的核心相关代码如下:

```
//GnssLocationProvider 继承AbstractLocationProvider 
public class GnssLocationProvider extends AbstractLocationProvider implements
        InjectNtpTimeCallback,
        GnssSatelliteBlacklistCallback {
        //省略
   //以下函数说明jni层去调用 reportLocation   
  @NativeEntryPoint
    private void reportLocation(boolean hasLatLong, Location location) {
        sendMessage(REPORT_LOCATION, hasLatLong ? 1 : 0, location);
    }
    //sendMessage
     private void sendMessage(int message, int arg, Object obj) {
        //...省略
        mHandler.obtainMessage(message, arg, 1, obj).sendToTarget();
    }
    //mHandler  初始化
     mHandler = new ProviderHandler(looper);

     //ProviderHandler为Handler类型，通过他发送消息通知位置改变
     private final class ProviderHandler extends Handler {
        public ProviderHandler(Looper looper) {
            super(looper, null, true /*async*/);
        }

        @Override
        public void handleMessage(Message msg) {
            int message = msg.what;
            switch (message) {
               //省略
                case REPORT_LOCATION:
                   //此处调用位置报告函数
                    handleReportLocation(msg.arg1 == 1, (Location) msg.obj);
                    break;
              //省略
            }
           
        }
    //    位置报告函数实现
    private void handleReportLocation(boolean hasLatLong, Location location) {
        //...省略
        reportLocation(location);
        //...省略
    }
}


```

以上分析可知 handleReportLocation 最终调用 reportLocation 通知位置信息变化。reportLocation 方法在父类 AbstractLocationProvider 中实现。如下是 AbstractLocationProvider 类中 reportLocation 核心实现代码:

```
public abstract class AbstractLocationProvider {

    //...省略
    //LocationManagerService初始化的时候传入赋值为gps 位置提供管理器
    private final LocationProviderManager mLocationProviderManager;

    protected AbstractLocationProvider(
            Context context, LocationProviderManager locationProviderManager) {
        mContext = context;
        mLocationProviderManager = locationProviderManager;
    }

    //reportLocation最终使用了mLocationProviderManager.onReportLocation方法
    protected void reportLocation(Location location) {
        mLocationProviderManager.onReportLocation(location);
    }
    //...省略
}


```

由于 mLocationProviderManager 在初始化的时候赋值为 GPS_PROVIDER 的位置提供管理者，所以 mLocationProviderManager.onReportLocation 调用就变成了 LocationProvider.onReportLocation。

三、GnssLocationProvider 中 jni 处理位置改变通知分析
---------------------------------------

在上面分析中, GnssLocationProvider 提供了一个通知位置改变的 java 函数 reportLocation(boolean hasLatLong, Location location)，该函数在 jni 中被调用。在 GnssLocationProvider 中提供了一系列 native 方法，比如如下:

```
  private static native void class_init_native();

    private static native boolean native_is_supported();

    private static native boolean native_is_gnss_visibility_control_supported();

    private static native void native_init_once(boolean reinitializeGnssServiceHandle);

    private native boolean native_init();


```

所以源码系统中存在 GnssLocationProvider 的 jni 实现文件。在源码中找到了对应的实现文件路径如下:

> frameworks\base\services\core\jni\com_android_server_location_GnssLocationProvider.cpp

以下是抽取 JNI 位置更新的相关核心代码部分:

```
1. 缓存java层中GnssLocationProvider类的reportLocation 方法ID
static jmethodID method_reportLocation;

2.定义位置改变回调接口
struct GnssCallback : public IGnssCallback {
    Return<void> gnssLocationCb(const GnssLocation_V1_0& location) override;
   
    Return<void> gnssLocationCb_2_0(const GnssLocation_V2_0& location) override;
   
private:
   
    Return<void> gnssLocationCbImpl(const T& location);

};

3. 获取reportLocation方法ID,方便后面反射调用
 method_reportLocation = env->GetMethodID(clazz, "reportLocation",
            "(ZLandroid/location/Location;)V");

4.位置改变回调方法具体实现
template<class T> Return<void> GnssCallback::gnssLocationCbImpl(const T& location) {
    JNIEnv* env = getJniEnv();

    jobject jLocation = translateGnssLocation(env, location);

    env->CallVoidMethod(mCallbacksObj,
                        method_reportLocation,
                        boolToJbool(hasLatLong(location)),
                        jLocation);
    checkAndClearExceptionFromCallback(env, __FUNCTION__);
    env->DeleteLocalRef(jLocation);
    return Void();
}

//1.0版本位置改变回调方法，最终通过反射调用通知java层位置改变
Return<void> GnssCallback::gnssLocationCb(const GnssLocation_V1_0& location) {
    return gnssLocationCbImpl<GnssLocation_V1_0>(location);
}

//2.0版本位置改变回调方法，最终通过反射调用通知java层位置改变
Return<void>
GnssCallback::gnssLocationCb_2_0(const GnssLocation_V2_0& location) {
    return gnssLocationCbImpl<GnssLocation_V2_0>(location);
}

5.初始化函数中通过gps硬件抽象层设置监听回调

static jboolean android_location_GnssLocationProvider_init(JNIEnv* env, jobject obj) {
   
    // Set top level IGnss.hal callback.
    sp<IGnssCallback> gnssCbIface = new GnssCallback();
    if (gnssHal_V2_0 != nullptr) {
        result = gnssHal_V2_0->setCallback_2_0(gnssCbIface);
    } else if (gnssHal_V1_1 != nullptr) {
        result = gnssHal_V1_1->setCallback_1_1(gnssCbIface);
    } else {
        result = gnssHal->setCallback(gnssCbIface);
    }
}



```

四、总结
----

gps 位置改变监听总的流程大概如下图所示:  
![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew54306CO1vMIy3euSvDCdtqQOXTrmHVVkC96oc2cgUWuNGGUB9PXmRC6Xu5E3tkaibFcoYEQVYFRk8BlQ/640?wx_fmt=png)

安卓系统、安卓 ndk 开发、安卓应用安全和逆向分析相关等 IT 知识分享，系统定制、frida、xposed(sandhook、edxposed) 系统集成、加固、脱壳、刷机交流等等。微信搜索公众号 "QDOIRD88888" 或者扫描以下二维码关注公众号。第一时间接收文章更新。

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew54306CO1vMIy3euSvDCdtqQOXoXz92OGlG2iabKI0vibriaHGKAQf9M7KhibgrBKhuVKe94icDnwcTPrWiaDA/640?wx_fmt=png)
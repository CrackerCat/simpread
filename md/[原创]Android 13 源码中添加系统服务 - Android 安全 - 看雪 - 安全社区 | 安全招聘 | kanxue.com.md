> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-276405.htm)

> [原创]Android 13 源码中添加系统服务

[原创]Android 13 源码中添加系统服务

2023-3-9 10:12 5229

### [原创]Android 13 源码中添加系统服务

 [![](http://passport.kanxue.com/upload/avatar/652/841652.png?1637199833)](user-home-841652.htm) [爱玩逆向](user-home-841652.htm) ![](https://bbs.kanxue.com/view/img/rank/0.png)  ![](http://passport.kanxue.com/pc/view/img/star.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 2023-3-9 10:12  5229

本文基于 Android 13 的 framework 层添加系统接口，为应用层提供读写函数、以及执行命令!

### 添加接口

frameworks/base/core/java/android/app/IDevices.aidl

```
package android.app;
interface IDevices
{
    //读取文件
    String readFile(String path);
    //写入文件
    void writeFile(String path,String data);
    //执行shell命令
    String shellExec(String cmd);
}

```

### 添加系统服务的 Manager

frameworks/base/core/java/android/app/DevicesManager.java

```
package android.app;
import android.annotation.SystemService;
import android.content.Context;
import android.os.RemoteException;
import android.util.Slog;
@SystemService(Context.DEVICES_SERVICE)
    public class DevicesManager {
        Context mContext;
        IDevices mService;
 
        public DevicesManager(Context context,IDevices service){
            if(service==null){
                Slog.e("DevicesManager","Construct service is null");
            }
            mContext = context;
            mService = service;
        }
 
        public String shellExec(String cmd){
            if(mService != null){
                try{
                    Slog.e("DevicesManager","shellExec");
                    return mService.shellExec(cmd);
                }catch(RemoteException e){
                    Slog.e("DevicesManager","RemoteException "+e);
                }
            }else{
                Slog.e("DevicesManager","mService is null");
            }
            return "";
        }
 
        public String readFile(String path){
            if(mService != null){
                try{
                    Slog.e("DevicesManager","readFile");
                    return mService.readFile(path);
                }catch(RemoteException e){
                    Slog.e("DevicesManager","RemoteException "+e);
                }
            }else{
                Slog.e("DevicesManager","mService is null");
            }
            return "";
        }
 
        public void writeFile(String path,String data){
            if(mService != null){
                try{
                    Slog.e("DevicesManager","writeFile");
                    mService.writeFile(path,data);
                }catch(RemoteException e){
                    Slog.e("DevicesManager","RemoteException "+e);
                }
            }else{
                Slog.e("DevicesManager","mService is null");
            }
        }
 
    }

```

### [](#添加系统服务，实现aidl文件的接口)添加系统服务，实现 aidl 文件的接口

frameworks/base/services/core/java/com/android/server/DevicesService.java

```
package com.android.server; 
import android.app.IDevices;
import android.content.Context;
import android.os.Build;
import android.util.Log;
import android.util.Slog;
 
import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
 
import libcore.io.IoUtils;
 
import java.io.BufferedReader;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.io.OutputStream;
import java.io.RandomAccessFile;
 
import android.util.Base64;
 
 
public class DevicesService extends IDevices.Stub {
    private Context mContext;
    private String TAG="DevicesService";
    public DevicesService(Context context){
        super();
        mContext = context;
        Slog.d(TAG,"Construct");
    }
 
    @Override
    public String shellExec(String cmd){
        Runtime mRuntime = Runtime.getRuntime();
        try {
            //Process中封装了返回的结果和执行错误的结果
            Slog.d(TAG,"shellExec data:"+cmd);
            Process mProcess = mRuntime.exec(cmd);
            BufferedReader mReader = new BufferedReader(new InputStreamReader(mProcess.getInputStream()));
            StringBuffer mRespBuff = new StringBuffer();
            char[] buff = new char[1024];
            int ch = 0;
            while ((ch = mReader.read(buff)) != -1) {
                mRespBuff.append(buff, 0, ch);
            }
            mReader.close();
            return mRespBuff.toString();
        } catch (IOException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
            Slog.d(TAG,"shellExec err:"+e.getMessage());
        }
        return "";
    }
 
 
    public static void writeTxtToFile(String strcontent, String filePath) {
        String strFilePath = filePath;
        String strContent = strcontent + "\n";  // \r\n 结尾会变成 ^M
        try {
            File file = new File(strFilePath);
            makeFilePath(file.getParent(),file.getName());
            if (!file.exists()) {
                file.getParentFile().mkdirs();
                file.createNewFile();
            }
            RandomAccessFile raf = new RandomAccessFile(file, "rwd");
            raf.setLength(0);
 
            // 写文件的位置标记,从文件开头开始,后续读取文件内容从该标记开始
            long writePosition = raf.getFilePointer();
            raf.seek(writePosition);
            raf.write(strContent.getBytes());
            raf.close();
            //
        } catch (Exception e) {
            Log.d("DevicesService","Error on write File:" + e);
        }
    }
 
    // 生成文件
    public static File makeFilePath(String filePath, String fileName) {
        File file = null;
        makeRootDirectory(filePath);
        try {
            file = new File(filePath +"/"+ fileName);
            if (!file.exists()) {
                file.createNewFile();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return file;
    }
 
    // 生成文件夹
    public static void makeRootDirectory(String filePath) {
        File file = null;
        try {
            Log.d("FileHelper", "makeRootDirectory "+filePath);
            file = new File(filePath);
            if (!file.exists()) {
                boolean isok= file.mkdir();
                Log.d("FileHelper", "makeRootDirectory "+filePath+" "+isok);
            }
        } catch (Exception e) {
            Log.d("DevicesService", e+"");
        }
    }
 
    public static String readFileAll(String path) {
        File file = new File(path);
        StringBuilder sb=new StringBuilder();
        if (file != null && file.exists()) {
            InputStream inputStream = null;
            BufferedReader bufferedReader = null;
            try {
                inputStream = new FileInputStream(file);
                bufferedReader = new BufferedReader(new InputStreamReader(
                        inputStream));
                String outData;
                while((outData=bufferedReader.readLine())!=null){
                    sb.append(outData+"\n");
                }
            } catch (Throwable t) {
            } finally {
                try {
                    if (bufferedReader != null) {
                        bufferedReader.close();
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
                try {
                    if (inputStream != null) {
                        inputStream.close();
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
        return sb.toString();
    }
 
 
    @Override
    public String readFile(String path){
        return readFileAll(path);
    }
    @Override
    public void writeFile(String path,String data){
        writeTxtToFile(data,path);
    }
 
 
}

```

### 添加服务名称

frameworks/base/core/java/android/content/Context.java

```
        @StringDef(suffix = { "_SERVICE" }, value = {
                POWER_SERVICE,
/*add_cxf*/++    DEVICES_SERVICE,
                //@hide: POWER_STATS_SERVICE,
                WINDOW_SERVICE,
                LAYOUT_INFLATER_SERVICE,
                ACCOUNT_SERVICE,
                ACTIVITY_SERVICE,
                ALARM_SERVICE,
 
 
 
  public static final String POWER_SERVICE = "power";
++public static final String DEVICES_SERVICE = "devices";

```

### 将实现的服务注册到系统中去

frameworks/base/core/java/android/app/SystemServiceRegistry.java

```
registerService(Context.SEARCH_SERVICE, SearchManager.class,
        new CachedServiceFetcher() {
    @Override
    public SearchManager createService(ContextImpl ctx) throws ServiceNotFoundException {
        return new SearchManager(ctx.getOuterContext(),
                ctx.mMainThread.getHandler());
    }});
//add cxf
registerService(Context.DEVICES_SERVICE,DevicesManager.class,
        new CachedServiceFetcher(){
    @Override
    public DevicesManager createService(ContextImpl ctx){
        IBinder b = ServiceManager.getService(Context.DEVICES_SERVICE);
        return new DevicesManager(ctx,IDevices.Stub.asInterface(b));
                } });   
//add end       
registerService(Context.SENSOR_SERVICE, SensorManager.class,
        new CachedServiceFetcher() {
    @Override
    public SensorManager createService(ContextImpl ctx) {
        return new SystemSensorManager(ctx.getOuterContext(),
          ctx.mMainThread.getHandler().getLooper());
    }}); 
```

### 将注册的服务设置成开机启动服务

frameworks/base/services/java/com/android/server/SystemServer.java

```
t.traceBegin("StartTelephonyRegistry");
telephonyRegistry = new TelephonyRegistry(
        context, new TelephonyRegistry.ConfigurationProvider());
ServiceManager.addService("telephony.registry", telephonyRegistry);
t.traceEnd();
//add cxf
t.traceBegin("StartDevicesService");
try{
    ServiceManager.addService(Context.DEVICES_SERVICE,new DevicesService(context));
} catch(Throwable e){
    Slog.e("DevicesService","Failed to start DevicesService Service "+e);
}
t.traceEnd();
//add end

```

### 让 lint 检查忽略掉自己的模块

注 (Android 11 以后谷歌强制开启 [lint](https://so.csdn.net/so/search?q=lint&spm=1001.2101.3001.7020) 检查，lint 检查不过编译会报错)  
frameworks/base/Android.bp

```
// TODO(b/145644363): move this to under StubLibraries.bp or ApiDocs.bp
metalava_framework_docs_args = "" +
    "--api-lint-ignore-prefix android.app. " +     //add_cxf
    "--api-lint-ignore-prefix android.icu. " +
    "--api-lint-ignore-prefix java. " +
    "--api-lint-ignore-prefix junit. " +
    "--api-lint-ignore-prefix org. " +
    "--error NoSettingsProvider " +
    "--error UnhiddenSystemApi " +
    "--force-convert-to-warning-nullability-annotations +*:-android.*:+android.icu.*:-dalvik.* " +
    "--hide BroadcastBehavior " +
    "--hide CallbackInterface " +
    "--hide DeprecationMismatch " +
    "--hide HiddenSuperclass " +
    "--hide HiddenTypeParameter " +
    "--hide MissingPermission " +
    "--hide-package android.audio.policy.configuration.V7_0 " +
    "--hide-package com.android.server " +
    "--hide RequiresPermission " +
    "--hide SdkConstant " +
    "--hide Todo " +
    "--hide Typo " +
    "--hide UnavailableSymbol " +
    "--manifest $(location core/res/AndroidManifest.xml) "

```

### 编译源码 更新 api 接口

```
make update-api

```

### 检查服务是否开启

将编译好的 rom 刷入手机, 查看 sevice list  
这里可以看到我们的服务 已经启动了：  
服务名称：79 devices: [android.app.IDevices]

```
cxf@cxf-System:~$ adb shell
coral:/ # service list |grep Device
60    companiondevice: [android.companion.ICompanionDeviceManager]
75    device_identifiers: [android.os.IDeviceIdentifiersPolicyService]
76    device_policy: [android.app.admin.IDevicePolicyManager]
77    device_state: [android.hardware.devicestate.IDeviceStateManager]
78    deviceidle: [android.os.IDeviceIdleController]
79    devices: [android.app.IDevices]
251    virtualdevice: [android.companion.virtual.IVirtualDeviceManager]
coral:/ #

```

### 调用测试

应用层项目中引入 aidl

```
// IDevices.aidl
package android.app;
 
// Declare any non-default types here with import statements
 
interface IDevices {
        String shellExec(String cmd);
        String readFile(String path);
        void writeFile(String path,String data);
}

```

![](https://bbs.kanxue.com/upload/attach/202303/841652_7FPNHPCXXBW7XVB.png)  
调用测试：  
![](https://bbs.kanxue.com/upload/attach/202303/841652_Q2ASP89WNQERWZJ.png)  
![](https://bbs.kanxue.com/upload/attach/202303/841652_H48TQ5D3CAF5ZMK.png)

  

[[培训]《安卓高级研修班 (网课)》月薪二万计划](https://www.kanxue.com/book-section_list-83.htm)

[#源码框架](forum-161-1-127.htm)
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/C0Vz_CW4fbG0j3AVe-GR4g)

**前言**

  

    本文将实现读取 "**/sys/class/net/wlan0/address**" 文件的方式获取 wifi mac 为例实现在安卓 10 源码中添加 framework 层自定义系统服务。

    安卓 10 中由于普通 App 直读取取 "**/sys/class/net/wlan0/address**"

文件获取 **wifi mac** 失败。然而添加系统服务读取可以读取成功，主要是由于安卓 10 的安全机制中限制了普通 **app** 不能读取 "**/sys/class/net/wlan0/address**" 文件获取 **wifi mac**。

    本文将参考 LocationManagerService 的实现方式，仿照写一个 GetWifiMacService 来实现获取 wifi mac。

   后续将以三篇文章完整实战操作讲述 framework 层添加自定义系统服务的过程。  

**Android Studio 开发自定义系统服务**

 本文使用的 Android Studio 版本 4.1.1。

**1. 创建 GetWifiMacService 工程**

 **图 1 选择创建工程:**

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432k0AkqrG9E5icShT0ANM4qKfkZ1ibMicGgibqtxSpZWTnyY5YnM3pzoib9xiaoNxnqiaUTPw4HIvUVSYuVg/640?wx_fmt=png)

 **图 2 选择工程模板：**

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432k0AkqrG9E5icShT0ANM4qKNoCBo6UgakXQIcqNI7GHdQfKXIu5ichCficzdNT19bVEvK0rrZBxybRA/640?wx_fmt=png)

**图 3 配置工程信息：**  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432k0AkqrG9E5icShT0ANM4qK67c1Qep79PjcTGczEZ50NhFtQUlC0ot3mv2uCcvI9rEhzuYlYBkAEA/640?wx_fmt=png)

**图 4 创建工程成功:**  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432k0AkqrG9E5icShT0ANM4qKZIn5KeKKyibwBnzufutFtb0QonC6VweMTbPP3o6sqd0mpSBIPLt6XXg/640?wx_fmt=png)

**2. 创建 IGetWifiMacInterface.aidl 文件**

 依次按照如下操作:

**图 1：**  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432k0AkqrG9E5icShT0ANM4qKyiaTg9lpZWZnCUsUbGwFQE8pyrX4pN8PfAWian5DuoOVru0dnenIao2g/640?wx_fmt=png)

**图 2：**

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432k0AkqrG9E5icShT0ANM4qKc6gTS6byEvBNZ3aKSKwB2yyvZwUmSnAsxYkEGPVia6LLJjK1xen9HcQ/640?wx_fmt=png)

**图 3:**  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432k0AkqrG9E5icShT0ANM4qK2xkmoy0MxGLHEyd2Y3omRem9pxQ3ekoj9V7YDMQZxO4LYY6qFqrjvQ/640?wx_fmt=png)

**图 3:**  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432k0AkqrG9E5icShT0ANM4qKJm7XkllPIVtiaTPHVe2uPjGHUeRtTzvNG8OPjJib28L0ia9SSq5kQtYXQ/640?wx_fmt=png)

**2. 创建 GetWifiMacServiceManager**

按照如下图所示的地方创建对应的 GetWifiMacServiceManager 文件:  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432k0AkqrG9E5icShT0ANM4qKSAdzNbaj8pT1ibibEViaMuDhl4H0I0UV7AH9znM4UAoN4eu2aEh11duRg/640?wx_fmt=png)

GetWifiMacServiceManager 完整实现代码如下:

```
package android.wifiex;
import android.content.Context;
import android.os.RemoteException;
public class GetWifiMacServiceManager {
    private final IGetWifiMacInterface mService;
    private Context mContext;
    GetWifiMacServiceManager(Context context,IGetWifiMacInterface service){
        mContext = context;
        mService = service;
    }
    public  String  getWifiMac(){
        try{
            return mService.getWifiMac();
        }catch (RemoteException ex){
        }
        return null;
    }
}

```

**3. 创建 GetWifiMacService**  

 在安卓系统中各种 java 层系统服务文件主要放在源码路径 "**/****frameworks****/****base****/****services****/****core****/****java****/****com****/****android****/****server****/**" 下面，为了后面直接将创建的服务拷贝过去，在 Android Studio 创建的 GetWifiMacService 服务的源文件包名以 "com.android.server" 为前缀。按照以下依次创建 GetWifiMacService 服务的源代码文件。

 **图 1:**

  最好提前 rebuild 一下工程，使 aidl 文件生成对应的 java 文件。

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5432k0AkqrG9E5icShT0ANM4qKDSuskfVtuvXk3QGJwGCvFFH1FQiaFUK4FM3M5dnC4ibr8XuNUD3MHuVA/640?wx_fmt=png)

GetMacService 完整实现代码如下:

```
package com.android.server.wifiex;
import android.os.Binder;
import android.os.RemoteException;
import android.util.Log;
import android.wifiex.IGetWifiMacInterface;
import java.io.BufferedReader;
import java.io.File;
import java.io.FileInputStream;
import java.io.InputStreamReader;
public class GetWifiMacService extends IGetWifiMacInterface.Stub{
    private static  final String TAG=GetWifiMacService.class.getSimpleName();
    @Override
    public String getWifiMac() throws RemoteException {
        Log.d(TAG,"getWifiMac call for pid:"+ Binder.getCallingPid()+" uid:"+Binder.getCallingUid());
        return getWifiMacFromFile();
    }
   //获取wifi mac
    private String  getWifiMacFromFile()
{
        String mypath="/sys/class/net/wlan0/address";
        String line = "";
        try {
            BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(new FileInputStream(new File(mypath))));
            line = bufferedReader.readLine();
            if(line==null)
            {
                line="";
            }
            bufferedReader.close();
        }catch (Exception eeee)
        {
        }
        return line;
    }
}

```

**总结**

*     创建 aidl 文件的时候先创建好包名，然后再指定包名下面右键菜单去新建 aidl 文件。Android Studio 会自动生成 aidl 目录以及对应包名的 aidl 文件。
    
*   创建系统服务源文件之前最好 **rebuild** 一下工程，使 **Android Studio** 自动生成 **aidl** 文件对应的 **java** 文件。不然创建的系统服务类继承的时候找不到如 **IGetWifMacInterface.Stub** 的类。
    

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjguCKrYZQfRXxK6hibNjOh10JibAdHj553dxk3PmoyUibjDCGcNdq3IQBKA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgibOWZXyrOLic5KPJ2y9A1gznt4xUa1H7MEhlgmcQgnE3IJvphZfOezfA/640?wx_fmt=png)  

![](https://mmbiz.qpic.cn/mmbiz_png/9vkUcew5433sxUAcMOjHULbEaeEkfGjgxGibv8NMwbmJuQo55Ry33RkQj6WTGwwyXgrcduXPL3xnUWeLUa3cDvA/640?wx_fmt=png)
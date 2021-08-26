> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-269094.htm)

> [原创] 安卓实现内存漫游 & Xposed 逆向必备~

前言：
---

一个安卓原生方法实现内存漫游的功能对象。

之前在摸索了很久，后来在 heap.cc 里面根据方法名字发现的。具体实现如下。

9.0 以下挺大的实现难度挺大的，需要通过 Debug 调试模式去发送对应的调试指令，才可以实现。

但是 9.0 原生的可以直接通过原生 API 实现。具体使用参考下方。感觉不错的还希望留个言支持一下。

```
/**
 * @author Zhenxi on 2021/5/5
 *
 * 源码参考:
 * http://androidxref.com/9.0.0_r3/s?refs=ClassD&project=art
 */
public class ChooseUtils {
 
    private static final Method startMethodTracingMethod;
    private static final Method stopMethodTracingMethod;
    private static final Method getMethodTracingModeMethod;
    private static final Method getRuntimeStatMethod;
    private static final Method getRuntimeStatsMethod;
    private static final Method countInstancesOfClassMethod;
    private static final Method countInstancesOfClassesMethod;
 
    private static  Method getInstancesOfClassesMethod ;
 
    static {
        try {
            Class c = Class.forName("dalvik.system.VMDebug");
            startMethodTracingMethod = c.getDeclaredMethod("startMethodTracing", String.class,
                    Integer.TYPE, Integer.TYPE, Boolean.TYPE, Integer.TYPE);
            stopMethodTracingMethod = c.getDeclaredMethod("stopMethodTracing");
            getMethodTracingModeMethod = c.getDeclaredMethod("getMethodTracingMode");
            getRuntimeStatMethod = c.getDeclaredMethod("getRuntimeStat", String.class);
            getRuntimeStatsMethod = c.getDeclaredMethod("getRuntimeStats");
 
            countInstancesOfClassMethod = c.getDeclaredMethod("countInstancesOfClass",
                    Class.class, Boolean.TYPE);
 
 
            countInstancesOfClassesMethod = c.getDeclaredMethod("countInstancesOfClasses",
                    Class[].class, Boolean.TYPE);
 
            //android 9.0以上才有这个方法
            if(android.os.Build.VERSION.SDK_INT>=28) {
                getInstancesOfClassesMethod = c.getDeclaredMethod("getInstancesOfClasses",
                        Class[].class, Boolean.TYPE);
            }
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
 
    /**
     * 根据Class获取当前进程全部的实例
     *
     * @param clazz 需要查找的Class
     * @return 当前进程的全部实例。
     */
    @TargetApi(28)
    public static ArrayList choose(Class clazz) {
        return choose(clazz, false);
    }
 
    /**
     * 根据Class获取当前进程全部的实例
     *
     * @param clazz      需要查找的Class
     * @param assignable 是否包含子类的实例
     * @return 当前进程的全部实例。
     */
    @TargetApi(28)
    public static synchronized ArrayList choose(Class clazz, boolean assignable) {
        ArrayList resut = null;
        try {
            Object[][] instancesOfClasses = getInstancesOfClasses(new Class[]{clazz}, assignable);
            if (instancesOfClasses != null) {
                resut = new ArrayList<>();
                for (Object[] instancesOfClass : instancesOfClasses) {
                    resut.addAll(Arrays.asList(instancesOfClass));
                }
            }
        } catch (Throwable e) {
            Log.e(Constants.TAG,"ChooseUtils choose error ", e);
            e.printStackTrace();
        }
        return resut;
    }
 
    @TargetApi(28)
    private static Object[][] getInstancesOfClasses(Class[] classes,
                                                    boolean assignable)
            throws Exception {
        return (Object[][]) getInstancesOfClassesMethod.invoke(
                null, new Object[]{classes, assignable});
    }
 
    public static void startMethodTracing(String filename, int bufferSize, int flags,
                                          boolean samplingEnabled, int intervalUs) throws Exception {
        startMethodTracingMethod.invoke(null, filename, bufferSize, flags, samplingEnabled,
                intervalUs);
    }
 
    public static void stopMethodTracing() throws Exception {
        stopMethodTracingMethod.invoke(null);
    }
 
    public static int getMethodTracingMode() throws Exception {
        return (int) getMethodTracingModeMethod.invoke(null);
    }
 
    /**
     *  String gc_count = VMDebug.getRuntimeStat("art.gc.gc-count");
     *  String gc_time = VMDebug.getRuntimeStat("art.gc.gc-time");
     *  String bytes_allocated = VMDebug.getRuntimeStat("art.gc.bytes-allocated");
     *  String bytes_freed = VMDebug.getRuntimeStat("art.gc.bytes-freed");
     *  String blocking_gc_count = VMDebug.getRuntimeStat("art.gc.blocking-gc-count");
     *  String blocking_gc_time = VMDebug.getRuntimeStat("art.gc.blocking-gc-time");
     *  String gc_count_rate_histogram = VMDebug.getRuntimeStat("art.gc.gc-count-rate-histogram");
     *  String blocking_gc_count_rate_histogram =VMDebug.getRuntimeStat("art.gc.gc-count-rate-histogram");
     */
    public static String getRuntimeStat(String statName) throws Exception {
        return (String) getRuntimeStatMethod.invoke(null, statName);
    }
 
    /**
     * 获取当前进程的状态信息
     */
    public static Map getRuntimeStats() throws Exception {
        return (Map) getRuntimeStatsMethod.invoke(null);
    }
 
    public static long countInstancesofClass(Class c, boolean assignable) throws Exception {
        return (long) countInstancesOfClassMethod.invoke(null, new Object[]{c, assignable});
    }
 
    public static long[] countInstancesofClasses(Class[] classes, boolean assignable)
            throws Exception {
        return (long[]) countInstancesOfClassesMethod.invoke(
                null, new Object[]{classes, assignable});
    }
 
 
} 
```

![](https://bbs.pediy.com/upload/attach/202108/819934_SFN9KEJAZ3HAPZY.jpg)  

![](https://bbs.pediy.com/upload/attach/202108/819934_U2UZC2S2RY5KTNA.jpg)

[[培训] 优秀毕业生寄语：恭喜 id 咸鱼炒白菜拿到远超 3W 月薪的 offer，《安卓高级研修班》火热招生！！！](https://zhuanlan.kanxue.com/article-16096.htm)

[#基础理论](forum-161-1-117.htm) [#逆向分析](forum-161-1-118.htm) [#NDK 分析](forum-161-1-119.htm) [#协议分析](forum-161-1-120.htm) [#混淆加固](forum-161-1-121.htm) [#脱壳反混淆](forum-161-1-122.htm) [#漏洞相关](forum-161-1-123.htm) [#程序开发](forum-161-1-124.htm) [#HOOK 注入](forum-161-1-125.htm) [#系统相关](forum-161-1-126.htm) [#源码分析](forum-161-1-127.htm) [#工具脚本](forum-161-1-128.htm) [#其他](forum-161-1-129.htm)
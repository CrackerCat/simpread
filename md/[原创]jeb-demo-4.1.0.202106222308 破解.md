> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268316.htm)

> [原创]jeb-demo-4.1.0.202106222308 破解

破解
--

### jeb-pro-3.19.1.202005071620

目标：破解`jeb-pro-3.19.1.202005071620`版本的时间限制，绕过自身完整性校验。

 

代码如下：

```
import javassist.*;
 
import java.io.IOException;
 
public class Main {
    public static final int FLAG_DEBUG = 1;
    public static final int FLAG_FULL = 2;
    public static final int FLAG_FLOATING = 4;
    public static final int FLAG_AIRGAP = 8;
    public static final int FLAG_ANYCLIENT = 16;
    public static final int FLAG_COREAPI = 32;
    public static final int FLAG_JEB2 = 128;
 
    public static void main(String[] args) throws NotFoundException, CannotCompileException, IOException {
        ClassPool pool = ClassPool.getDefault();
        String classPatch = "/Users/*/IdeaProjects/crack_jeb_pro/lib/jeb-3.19.jar";
        pool.insertClassPath(classPatch);
        // 修改许可证时间
        CtClass ctClass = pool.get("com.pnfsoftware.jeb.client.Licensing");
        CtMethod ctMethod = ctClass.getDeclaredMethod("getExpirationTimestamp");
        ctMethod.setBody("return 2000000000;");
        ctClass.writeFile("/Users/*/Public/");
 
        //修改 prepareCheckLicenseKey 函数
        ctClass = pool.get("com.pnfsoftware.jeb.client.AbstractClientContext");
        ctMethod = ctClass.getDeclaredMethod("prepareCheckLicenseKey");
        ctMethod.setBody("return ;");
        ctClass.writeFile("/Users/*/IdeaProjects/crack_jeb_pro/out/");
 
        // 修改 com.pnfsoftware.jebglobal.r.RF 函数 总共有四处
        ctClass = pool.get("com.pnfsoftware.jebglobal.r");
        ctMethod = ctClass.getDeclaredMethod
                (
                        "RF",
                        new CtClass[]
                                {
                                        pool.get("java.lang.Class")
                                }
                );
        ctMethod.setBody("return true;");
        ctMethod = ctClass.getDeclaredMethod
                (
                        "RF",
                        new CtClass[]
                                {
                                        pool.get("java.lang.Object"),
                                        pool.get("java.lang.Class")
                                }
                );
        ctMethod.setBody("return true;");
        ctMethod = ctClass.getDeclaredMethod
                (
                        "RF",
                        new CtClass[]
                                {
                                        pool.get("java.lang.Class[]")
                                }
                );
        ctMethod.setBody("return true;");
        ctMethod = ctClass.getDeclaredMethod
                (
                        "RF",
                        new CtClass[]
                                {
                                        pool.get("java.lang.Object"),
                                        pool.get("java.lang.Class[]")
                                }
                );
        ctMethod.setBody("return true;");
        ctClass.writeFile("/Users/*/IdeaProjects/crack_jeb_pro/out/");
 
 
    }
 
}

```

将生成的 class 文件替换原来`jeb.jar`中的文件即可，并且删除`MATA-INF`目录下的`JEBKEY.RSA`和`JEBKEY.SF`，并修改了`MANIFEST.MF`文件，至此我才终于开始步入正轨

 

`MANIFEST.MF`文件内容如下：

```
Manifest-Version: 1.0
Ant-Version: Apache Ant 1.9.10
Core-Version: 3.19.1.202005071620
Class-Path: lz4-java-1.4.1.jar xpp3-1.1.4c.jar commons-configuration2-
 2.2.jar jython-standalone-2.7.2.jar byte-buddy-1.10.8.jar netty-all-4
 .1.34.Final.jar commons-logging-1.2.jar commons-codec-1.11.jar sqlite
 -jdbc-3.20.0.jar z3-4.5.0.jar jsr305-3.0.2.jar commons-compress-1.16.
 1.jar okio-1.13.0.jar d8.jar okhttp-3.9.0.jar jsoup-1.10.3.jar common
 s-io-2.6.jar guava-25.0-jre.jar xz-1.6.jar antlr4-runtime-4.7.1.jar s
 nakeyaml-1.17.jar commons-beanutils-1.9.3.jar dx.jar commons-text-1.3
 .jar commons-collections4-4.1.jar objenesis-2.6.jar json-simple-1.1.1
 .jar commons-lang3-3.7.jar
Created-By: 1.8.0_242-b08 (Oracle Corporation)
Main-Class: com.pnfsoftware.jeb.Launcher

```

![](https://bbs.pediy.com/upload/attach/202107/821515_QP3VHWCHVSCCWXK.png)

### jeb-demo-4.1.0.202106222308-JEBDecompiler-121820464987384330

关于`jeb-demo-4.1.0`版本的破解，最主要的参照就是`JEB 3.24 Anti-BLM Edition by DimitarSerg`版本。

 

经过一定的测试，可以直接使用 IDEA 来编写 java 文件  
测试了几款反编译工具（JD-GUI、IDEA）效果都不太理想，所以可能还是得手撸代码了。  
[参考文章](https://www.jianshu.com/p/6b57c3131c1f)

```
java -cp  "/Applications/IntelliJ IDEA.app/Contents/plugins/java-decompiler/lib/java-decompiler.jar" org.jetbrains.java.decompiler.main.decompiler.ConsoleDecompiler -dgs=true jeb.jar src

```

这里列举一下简要步骤，感兴趣的师傅可以自行尝试。  
1. 文件比对  
2. 重写功能函数  
3. 测试

```
+ Fix Decompilation.
+ Fix Android: Generic string decryption via emulation.
+ Fix Saving or loading projects.
+ Fix Usage of the clipboard.
+ Fix Running time of a session.
+ Fixed all integrity checks/timebombs.
+ Restored display of variable values when hovering over them during debugging (Android).

```

下载地址如下：

```
链接: https://pan.baidu.com/s/14zuLVI5xekb-wKSru65_ig  密码: qg8w

```

最近 4.2.0 版本又更新了，后续在进行更新。

 

最后放张效果图。

 

![](https://bbs.pediy.com/upload/attach/202107/821515_GZKJMJT285U3SBP.png)

 

![](https://bbs.pediy.com/upload/attach/202107/821515_ZCT9TWAD7GF8VXT.png)

[第五届安全开发者峰会（SDC 2021）议题征集正式开启！](https://bbs.pediy.com/thread-266645.htm)

最后于 1 小时前 被 BadRer 编辑 ，原因：

[#调试逆向](forum-4-1-1.htm)
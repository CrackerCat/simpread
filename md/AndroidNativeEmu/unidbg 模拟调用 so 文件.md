> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/Hhql-5KxuAI_TWD-oL5dFw)

先自己动手整个 demo  

1. 打开 Android studio 创建 c++ 项目

 Android studio 版本：3.5.2

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr15fTict6sqaJxicjOVzUnBvlHa0jl3fJbjPsmDGtRpRBgNjVjPianicwFRA/640?wx_fmt=png)

2. 名字随意

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr1ibfn5QX5dZwOYSgpSFsAHRx5Wb1x9DzO4Iq0V8bpiaxeoruQIRXrlTfQ/640?wx_fmt=png)

3.File - Settings 下查看 NDK 安装配置情况，如果没有下载配置 NDK ，以及相关的包，对应下载相关的安装包，具体如下图

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr139g4ickMGBibT7fy5ewkicHoeuEuMYQ45ZjIFw3wJ7gkq4wsVndjqLoEA/640?wx_fmt=png)

4. 下载 ndk 后后，查看安装配置路径情况

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr1FJsT7uVfsAy4jw3JL8SmLibYMQ3djDgvTgqU3Im0ibaKcv72gDr2hEkA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr1R8SibmgDQo1dOmicyulEwaaTQR9kWWbqwRqJNknmnMZn1BHibBazEMATg/640?wx_fmt=png)

5. 在 src/main/cpp/native-lib.cpp 中实现定义的 JNI 接口，已经自动生成了一个静态注册的 demo，我们再整个动态注册  

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr1fIqic08vDksmDKf4qjUnsc4Q8IHHEoYfrD08VGbibWV51pe8IibqCOCdg/640?wx_fmt=png)

动态注册代码

```
#include <jni.h>
#include <string>
extern "C" JNIEXPORT jstring
JNICALL
Java_com_qxp_ndkdemo_MainActivity_stringFromJNI(
        JNIEnv *env,
        jobject /* this */) {
    std::string hello = "Hello AndroidNativeEmu";
    return env->NewStringUTF(hello.c_str());
}
//实现在MainActivity类的方法 stringFromJNI()
static jstring stringFromJNI(JNIEnv *env, jobject obj) {
    std::string hello2 ="this is registerNatives_stringFromJNI!";
    return env->NewStringUTF(hello2.c_str());
}
//参数映射表
//这是在MainActivity中的native方法
static JNINativeMethod gMethods_MainActivity[] = {
        {"registerNatives_stringFromJNI", "()Ljava/lang/String;",(void *) stringFromJNI},
};
//找到MainActivity.java类
static int registerNatives(JNIEnv *engv) {
    jclass clazz;
    clazz = engv->FindClass("com/qxp/ndkdemo/MainActivity");   //找到类
    if (clazz == NULL) {
        return JNI_FALSE;
    }
    if (engv->RegisterNatives(clazz, gMethods_MainActivity,sizeof(gMethods_MainActivity) / sizeof(gMethods_MainActivity[0])) <
        0) {
        return JNI_FALSE;
    }
    return JNI_TRUE;
}
JNIEXPORT jint JNI_OnLoad(JavaVM *vm, void *reserved) {
    JNIEnv *env = NULL;
    jint result = -1;
    if (vm->GetEnv((void **) &env, JNI_VERSION_1_4) != JNI_OK) {
        return -1;
    }
    assert(env != NULL);
    //为了方便管理我们将不同java类中的native方法分别注册
    registerNatives(env); //注册MainActivity类的native方法
    return JNI_VERSION_1_4;
}

```

6. 在 src/main/java/com/qxp/ndkdemo/MainActivity.java 下加载 so 库和定义 JNI 接口

```
package com.qxp.ndkdemo;
import androidx.appcompat.app.AppCompatActivity;
import android.os.Bundle;
import android.widget.TextView;
public class MainActivity extends AppCompatActivity {
    // Used to load the 'native-lib' library on application startup.
    static {
        System.loadLibrary("native-lib");
    }
    public native String stringFromJNI();
    public native String registerNatives_stringFromJNI();
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        TextView tv = findViewById(R.id.sample_text);
        tv.setText("静态注册函数调用结果："+stringFromJNI()+"\n"+"动态注册函数调用结果："+registerNatives_stringFromJNI());
    }
}

```

7. 在 CMakeLists.txt  添加一行代码：  

```
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/../jniLibs/${ANDROID_ABI})

```

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr1TAoBUib2ANpm1IcoJrRIOBeqpGmdqiagdoM6laofKNbBXo8bJcdtZiaUw/640?wx_fmt=png)

8. 修改 build.gradle，修改位置如下

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr12ibibl1paWXJVNLS5WnG2wsichwEUiaicicCpY3Lf4Nuicicx0KicJdHTe1MIlw/640?wx_fmt=png)

```
cmake {
      cppFlags "-std=c++11 -frtti -fexceptions"
      abiFilters 'armeabi-v7a','arm64-v8a'
            }

```

```
sourceSets {
        main{
            jniLibs.srcDirs = ['src/main/jniLibs/libs']
        }
    }

```

9. 编译 so  

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr152KE5FG4iaYyrFibTUicHWWmKLoTo95qgKbibSiaxEeXNCANbicO6dU7DGiag/640?wx_fmt=png)

10. 成功后 so 文件会放在 jniLibs 目录下

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr1YdA05E4j85ZP8gb0LdvxXrtbzic4vdtHDxuJic1cNNPZX0ibibezmBFT1A/640?wx_fmt=png)

11. 编译成 apk 安装到手机

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr1SZEWQTaAJibrxgnRrfZMSqRncBqFA0PCMlzOxQrYEpWvT1bYljbnSBQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr1K3PkicgGBW1bKSLThtDOsh9uSyZfGAIicM5sdhndCwiarJPOpRQ0zib0iaw/640?wx_fmt=png)

调用成功，nice！  

**AndroidNativeEmu**

AndroidNativeEmu 是一个 python 项目，是基于 Unicron 实现的一个指令解析器, 让您能够跨平台模拟 Android Native 库函数，例如 JNI_OnLoad，Java_XXX_XX 等函数

对于爬虫，简单来说就是可以摆脱手机来模拟调用 so 文件

```
AndroidNativeEmu github 地址
https://github.com/AeonLucid/AndroidNativeEmu
Clone the repository
//最好用python3.7
Run pip install -r requirements.txt
Run python example.py

```

运行测试案例，

```
AndroidNativeEmu-master\AndroidNativeEmu-master\samples\example.py

```

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr1sPsPrPa8xJSsk3ficItRibNxOphmoMfgbKkzEo11M4znyEO8oN1DhBiag/640?wx_fmt=png)  

能运行说明没毛病

试试我们自己写得 demo

1. 新建 ndktest.py

2. 加载 so 文件

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr1hcc3slJq6hxRqgHqRlKI7xs6m0MEfWnB4YdYLEwAXRtFbTUcC4v0Dg/640?wx_fmt=png)

3.

3. 创建一个类  

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr1H7XicNeXDibVOngic7fibNprAjHlwz8kycyDhaib1oRHutKt4KOZMmEQPlw/640?wx_fmt=png)

4. 模拟调用 native 函数

```
实例化 MainActivity() 通过类来调用

```

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr1jibteHNNxJvzE5susFetMVDh9IA8G0O4HXSTqfp7avicqJoGxMLqBZuw/640?wx_fmt=png)

静态方法直接调用：

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr17ibqojwxBDwJEiboiaPH5Djn9nsSP6dN63wGvTZYNRYkbb4gurCPTX1LA/640?wx_fmt=png)

结果  

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr1y77V6ZhWM9MUNfzSnXDRXib5bkGvYeKmbHUojGTNDg4yia7rdohnALGw/640?wx_fmt=png)

ndkdemo.py 代码：  

```
import logging
import posixpath
import sys
from unicorn import UcError, UC_HOOK_MEM_UNMAPPED
from unicorn.arm_const import *
from androidemu.emulator import Emulator
from androidemu.java.java_class_def import JavaClassDef
from androidemu.java.java_method_def import java_method_def
from samples import debug_utils
# Create java class.
class MainActivity(metaclass=JavaClassDef, jvm_name='com/qxp/ndkdemo/MainActivity'):
    def __init__(self):
        pass
    @java_method_def(name='registerNatives_stringFromJNI', signature='()Ljava/lang/String;', native=True)
    def registerNatives_stringFromJNI(self, mu):
        pass
    def test(self):
        pass
# Configure logging
logging.basicConfig(
    stream=sys.stdout,
    level=logging.DEBUG,
    format="%(asctime)s %(levelname)7s %(name)34s | %(message)s"
)
logger = logging.getLogger(__name__)
# Initialize emulator
emulator = Emulator(
    vfp_inst_set=True,
    vfs_root=posixpath.join(posixpath.dirname(__file__), "vfs")
)
# emulator.mu.hook_add(UC_HOOK_CODE, debug_utils.hook_code)
emulator.mu.hook_add(UC_HOOK_MEM_UNMAPPED, debug_utils.hook_unmapped)
# Register Java class.
emulator.java_classloader.add_class(MainActivity)
# Load all libraries.
emulator.load_library("example_binaries/libdl.so")
emulator.load_library("example_binaries/libc.so")
emulator.load_library("example_binaries/libstdc++.so")
emulator.load_library("example_binaries/libm.so")
lib_module = emulator.load_library(r"D:\Program_Files\android_NDK_demo\app\src\main\jniLibs\armeabi-v7a\libnative-lib.so")
# Show loaded modules.
logger.info("Loaded modules:")
for module in emulator.modules:
    logger.info("=> 0x%08x - %s" % (module.base, module.filename))
# Debug
# emulator.mu.hook_add(UC_HOOK_CODE, debug_utils.hook_code)
# emulator.mu.hook_add(UC_HOOK_MEM_UNMAPPED, debug_utils.hook_unmapped)
# emulator.mu.hook_add(UC_HOOK_MEM_WRITE, debug_utils.hook_mem_write)
# emulator.mu.hook_add(UC_HOOK_MEM_READ, debug_utils.hook_mem_read)
try:
    #直接调用
    result = emulator.call_symbol(lib_module, 'Java_com_qxp_ndkdemo_MainActivity_stringFromJNI',
                                  emulator.java_vm.jni_env.address_ptr, 0x00)
    logger.info("直接调用静态注册函数 stringFromJNI: %s" % result)
    # Run JNI_OnLoad.
    #   JNI_OnLoad will call 'RegisterNatives'.
    emulator.call_symbol(lib_module, 'JNI_OnLoad', emulator.java_vm.address_ptr, 0x00)
    # Do native stuff.
    main_activity = MainActivity()
    logger.info("调用动态注册函数 registerNatives_stringFromJNI: %s" % main_activity.registerNatives_stringFromJNI(emulator))
    # Dump natives found.
    logger.info("Exited EMU.")
    logger.info("Native methods registered to MainActivity:")
    for method in MainActivity.jvm_methods.values():
        if method.native:
            logger.info("- [0x%08x] %s - %s" % (method.native_addr, method.name, method.signature))
except UcError as e:
    print("Exit at %x" % emulator.mu.reg_read(UC_ARM_REG_PC))
    raise

```

**unidbg**  

unidbg 是一个基于 unicorn 的逆向工具，可以黑盒调用安卓和 iOS 中的 so 文件。unidbg 是一个标准的 java 项目。

```
unidbg github 地址
https://github.com/zhkl0228/unidbg

```

用 idea 打开此项目，需要准备好 java 环境 (jdk、maven)

运行测试 demo 在

```
unidbg-android\src\test\java\com\bytedance\frameworks\core\encrypt\TTEncrypt.java
unidbg-android\src\test\java\com\sun\jna\JniDispatch32.java

```

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr1f1rSic7eVQm9hJCWiawAJqu5jSQcOgfmnb6b4C2WgXJt4JgJI9ePibSYw/640?wx_fmt=png)

能运行说明没问题；  

来试试我们自己写得 demo

1. 新建一个 java 类 ndkdemo.java

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr1AtqvvdxLdUaYOlc6m0UiaT1PMl2rHlphNw3UGwcibK1qGwX8zb9G2FEg/640?wx_fmt=png)

2. 加载 so 文件，有两种方式加载 so 文件

a. 第一种：填写 apk 的路径，先加载整个 apk，再通过 so 库名来加载

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr11mxSJBJb2EwiccCkjRn1ibiaHbo8T47WsO0WPkWYvQicuMgm9HvEqflhwg/640?wx_fmt=png)

b. 第二种: 填写 so 文件的路径，直接加载

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr1zBxPOQmnRWFHVZhGeiccWNSkOLE1oePTQaLTaNscEQTWCMug8u6AdpA/640?wx_fmt=png)

3. 填写要调用函数所在类的类名  

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr1eyic6wN853vebkucDGniajObUm22BlGdsyuRB4ictXXnlc80sLqlNJnsA/640?wx_fmt=png)

4. 模拟调用  

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr1YofFbyicwyM4Va1zvQQw3XeIHzE4c1C7rjXcA4QI6ljpibPOPRJs0Oag/640?wx_fmt=png)

5. 结果：

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr1fmQ8OzfR7uibbH8xIHibATBZYOUlJ9aqBXOVq9lID8L4icPAysZP9KXXg/640?wx_fmt=png)

ndkdemo.java 代码：

```
package com.sun.jna;
import com.github.unidbg.*;
import com.github.unidbg.linux.android.AndroidARMEmulator;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.memory.Memory;
import java.io.File;
import java.io.IOException;
public class ndkdemo extends AbstractJni {
    private static LibraryResolver createLibraryResolver() {
        return new AndroidResolver(23);
    }
    private static AndroidEmulator createARMEmulator() {
        return new AndroidARMEmulator("com.qxp.ndkdemo");
    }
    private static final String APP_PACKAGE_NAME = "com.qxp.ndkdemo";
    private final AndroidEmulator emulator;
    private final Module module;
    private final VM vm;
    private final DvmClass Native;
    private ndkdemo() {
        emulator = new AndroidARMEmulator(APP_PACKAGE_NAME);
        System.out.println("== init ===");
        final Memory memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23));
       // vm = emulator.createDalvikVM(new File("D:\\Program_Files\\android_NDK_demo\\app\\build\\outputs\\apk\\debug\\app-debug.apk"));
       //DalvikModule dm = vm.loadLibrary("native-lib", false);
        vm = emulator.createDalvikVM(null);
        DalvikModule dm = vm.loadLibrary(new File("D:\\Program_Files\\android_NDK_demo\\app\\src\\main\\jniLibs\\armeabi-v7a\\libnative-lib.so"), false);
        vm.setJni(this);
        vm.setVerbose(true);// 设置是否打印Jni调用细节
        dm.callJNI_OnLoad(emulator);
        module = dm.getModule();
        Native = vm.resolveClass("com/qxp/ndkdemo/MainActivity");
    }
    private void destroy() throws IOException {
        emulator.close();
        System.out.println("destroy");
    }
    public static void main(String[] args) throws Exception {
        ndkdemo test = new ndkdemo();
        test.test();
        test.destroy();
    }
    private void test() {
        //静态注册函数 stringFromJNI
        String stringFromJNI = "stringFromJNI()Ljava/lang/String;";
        Object ret1 = Native.callStaticJniMethodObject(emulator, stringFromJNI);
        System.out.println("静态注册函数 stringFromJNI 执行结果:"+((DvmObject) ret1).getValue());
        //动态注册函数 registerNatives_stringFromJNI()
        String registerNatives_stringFromJNI = "registerNatives_stringFromJNI()Ljava/lang/String;";
        Object ret2 = Native.callStaticJniMethodObject(emulator, registerNatives_stringFromJNI);
        System.out.println("动态注册函数 registerNatives_stringFromJNI 执行结果:"+((DvmObject) ret2).getValue());
    }
}

```

两个框架都非常的牛逼，可以做的事情非常非常多，赶紧学起来![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH6FsSkd4NV1graxSznExTr1t84rypibfhCLKAZt7TicC4PvkMeXgvl11ExJETO7VgVib0ianupv4JXbBg/640?wx_fmt=png)
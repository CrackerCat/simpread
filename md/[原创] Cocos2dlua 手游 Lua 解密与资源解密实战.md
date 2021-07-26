> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268574.htm)

> [原创] Cocos2dlua 手游 Lua 解密与资源解密实战

前言
==

发过理论篇[浅谈 Cocos2d-x 下 Lua 文件的保护方式 – 翻车鱼 (blog.shi1011.cn)](https://blog.shi1011.cn/re/android/1216)，是时候来实战一番了。

 

**起因**

 

一位不知是港澳台哪地的师傅，加了我的 QQ

 

![](https://bbs.pediy.com/upload/attach/202107/883754_Q3T24BRNHEQ3TAH.png)

 

于是我和他进行了一场漫长的**探讨**

 

尽管轮子在手，但是依旧无法反编译出`.lua`（因为加了一层自定义加密）

 

结果就是：喜提样本一份

样本树
===

大致结构如下

```
.
├── assets
│   ├── res
│   │   ├── ani
│   │   │   └── logo
│   │   │       └── logo.png
│   ├── src
│   │   ├── main.lua
│   │   └── main.lua64
├── lib
│   ├── arm64-v8a
│   │   ├── libBugly.so
│   │   └── libcocos2dlua.so
│   └── armeabi-v7a
│       ├── libBugly.so
│       └── libcocos2dlua.so
└── stamp-cert-sha256

```

其中`.lua64`为标准 LuaJit 文件

 

而`.lua`的文件头有点奇怪

 

![](https://bbs.pediy.com/upload/attach/202107/883754_MD94JBWU7F6Z3RQ.png)

 

.lua 文件头：`abcd`

 

再看资源文件`.png`，发现也是加密的

 

![](https://bbs.pediy.com/upload/attach/202107/883754_RJKNB45XSC69AGT.png)

 

综上，我们需要实现的目标

2.  解密`.lua`文件
3.  解密`.png`文件

LuaJit 反汇编
==========

这个不多介绍了窝，在理论篇中已经提到了。对于此样本的 LuaJit 的版本是`2.1.0-Beta2`，并且没有魔改 Opcode

Lua 文件加载流程
==========

想要解密`.lua`文件，了解 coco2d-x 加载 Lua 的流程必不可少

Cocos2d-x 环境搭建
--------------

参考[官方 Docs](https://docs.cocos.com/cocos2d-x/manual/zh/)，搭建所需环境

 

![](https://bbs.pediy.com/upload/attach/202107/883754_88D665ZY37ZJK7H.png)

从 Android Activity 揭秘 cocos2d-x 的世界
-----------------------------------

由于 Android 的应用层是从 Activity 开始的，也就是创建完一个 Cocos2dx 后 src 文件夹下的 Java 文件，其中主要看 Activity 创建时的操作

```
package org.cocos2dx.lua;
 
import android.os.Bundle;
import org.cocos2dx.lib.Cocos2dxActivity;
 
public class AppActivity extends Cocos2dxActivity{
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.setEnableVirtualButton(false);
        super.onCreate(savedInstanceState);
        // Workaround in https://stackoverflow.com/questions/16283079/re-launch-of-activity-on-home-button-but-only-the-first-time/16447508
        if (!isTaskRoot()) {
            // Android launched another instance of the root activity into an existing task
            //  so just quietly finish and go away, dropping the user back into the activity
            //  at the top of the stack (ie: the last state of this task)
            // Don't need to finish it again since it's finished in super.onCreate .
            return;
        }
 
        // DO OTHER INITIALIZATION BELOW
 
    }
}

```

这个方法很简单，就只调用了父类`Cocos2dxActivity`的`onCreate`方法

```
@Override
    protected void onCreate(final Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
 
        // Workaround in https://stackoverflow.com/questions/16283079/re-launch-of-activity-on-home-button-but-only-the-first-time/16447508
        if (!isTaskRoot()) {
            // Android launched another instance of the root activity into an existing task
            //  so just quietly finish and go away, dropping the user back into the activity
            //  at the top of the stack (ie: the last state of this task)
            finish();
            Log.w(TAG, "[Workaround] Ignore the activity started from icon!");
            return;
        }
 
        this.hideVirtualButton();
 
        onLoadNativeLibraries();
 
        sContext = this;
        this.mHandler = new Cocos2dxHandler(this);
 
        Cocos2dxHelper.init(this);
 
        this.mGLContextAttrs = getGLContextAttrs();
        this.init();
 
        if (mVideoHelper == null) {
            mVideoHelper = new Cocos2dxVideoHelper(this, mFrameLayout);
        }
 
        if(mWebViewHelper == null){
            mWebViewHelper = new Cocos2dxWebViewHelper(mFrameLayout);
        }
 
        if(mEditBoxHelper == null){
            mEditBoxHelper = new Cocos2dxEditBoxHelper(mFrameLayout);
        }
 
        Window window = this.getWindow();
        window.setSoftInputMode(WindowManager.LayoutParams.SOFT_INPUT_ADJUST_PAN);
 
        // Audio configuration
        this.setVolumeControlStream(AudioManager.STREAM_MUSIC);
    }

```

在这个方法中主要看 Cocos2dxActivity 的初始化方法 init，Cocos2dxHandler 是工具辅助类，不是重点。

```
public Cocos2dxGLSurfaceView onCreateView() {
        Cocos2dxGLSurfaceView glSurfaceView = new Cocos2dxGLSurfaceView(this);
        //this line is need on some device if we specify an alpha bits
        // FIXME: is it needed? And it will cause afterimage.
        // if(this.mGLContextAttrs[3] > 0) glSurfaceView.getHolder().setFormat(PixelFormat.TRANSLUCENT);
 
        // use custom EGLConfigureChooser
        Cocos2dxEGLConfigChooser chooser = new Cocos2dxEGLConfigChooser(this.mGLContextAttrs);
        glSurfaceView.setEGLConfigChooser(chooser);
 
        return glSurfaceView;
    }
// ......
public void init() {
 
        // FrameLayout 初始化窗口布局
        ViewGroup.LayoutParams framelayout_params =
            new ViewGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT,
                                       ViewGroup.LayoutParams.MATCH_PARENT);
 
        mFrameLayout = new ResizeLayout(this);
 
        mFrameLayout.setLayoutParams(framelayout_params);
 
        // Cocos2dxEditText layout 初始化Cocos2dx的文本编辑布局
        ViewGroup.LayoutParams edittext_layout_params =
            new ViewGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT,
                                       ViewGroup.LayoutParams.WRAP_CONTENT);
        Cocos2dxEditBox edittext = new Cocos2dxEditBox(this);
        edittext.setLayoutParams(edittext_layout_params);
 
 
        mFrameLayout.addView(edittext);
 
        // Cocos2dxGLSurfaceView 初始化Cocos2dx视图
        this.mGLSurfaceView = this.onCreateView();
 
        // ...add to FrameLayout 将Cocos2dxGLSurfaceView加入到当前的窗口布局中
        mFrameLayout.addView(this.mGLSurfaceView);
 
        // Switch to supported OpenGL (ARGB888) mode on emulator
        // this line dows not needed on new emulators and also it breaks stencil buffer
        //if (isAndroidEmulator()) // 在模拟器中切换支持OpenGL模式的渲染(ARGB888)
        //   this.mGLSurfaceView.setEGLConfigChooser(8, 8, 8, 8, 16, 0);
 
        // 设置Cocos2dx的渲染器
        this.mGLSurfaceView.setCocos2dxRenderer(new Cocos2dxRenderer());
        //设置Cocos2dx的文本编辑
        this.mGLSurfaceView.setCocos2dxEditText(edittext);
 
        // Set framelayout as the content view
        // 把显示布局（即Cocos2dx的视图）绑定到Activity上，建立显示窗口
        setContentView(mFrameLayout);
    }

```

首先在 init 中先看`this.mGLSurfaceView = this.onCreateView();`this.mGLSurfaceView 是一个`Cocos2dxGLSurfaceView`类

 

进入到`Cocos2dxGLSurfaceView`这个类，可以看到时继承于`GLSurfaceView`（可以把`GLSurfaceView`看成一个视图，里面有个方法设置了这个视图的渲染器，然后通过这个渲染器来进行画面的渲染）  
在 Android 中，`GLSurfaceView`是一个支持 OpenGL 的渲染视图，通过继承`SurfaceView`中的 surface 来渲染 OpenGL  
并提供了以下特性（来源于网上）

1.  管理一个 surface，这个 surface 就是一块特殊的内存，能直接排版到 android 的视图 view 上
2.  管理一个 EGL display，它能让 opengl 把内容渲染到上述的 surface 上
3.  用户自定义渲染器 (render)
4.  让渲染器在独立的线程里运作，和 UI 线程分离
5.  支持按需渲染 (on-demand) 和连续渲染(continuous)
6.  一些可选工具，如调试

重点是自定义的渲染器，也就是 Cocos2dx 引擎封装的渲染器

 

进入`Cocos2dxRenderer`类，看到他是继承`GLSurfaceView.Renderer`接口，这个接口定义了三个方法：

*   `onSurfaceCreated`: 创建 GLSurfaceView 时被调用，只调用一次，做初始化工作
*   `onSurfaceChanged`: 当 GLSurfaceView 的几何体被改变时被调用
*   `onDrawFrame`: 绘制渲染 GLSurfaceView

**onSurfaceCreated**

```
@Override
    public void onSurfaceCreated(final GL10 GL10, final EGLConfig EGLConfig) {
        Cocos2dxRenderer.nativeInit(this.mScreenWidth, this.mScreenHeight);
        this.mLastTickInNanoSeconds = System.nanoTime();
        mNativeInitCompleted = true;
    }

```

`nativeInit`是一个 Native 函数

```
private static native void nativeInit(final int width, final int height);

```

具体实现在`frameworks\cocos2d-x\cocos\platform\android\javaactivity-android.cpp`中

```
JNIEXPORT void Java_org_cocos2dx_lib_Cocos2dxRenderer_nativeInit(JNIEnv*  env, jobject thiz, jint w, jint h)
{
    auto director = cocos2d::Director::getInstance();
    auto glview = director->getOpenGLView();
    if (!glview)
    {
        glview = cocos2d::GLViewImpl::create("Android app");
        glview->setFrameSize(w, h);
        director->setOpenGLView(glview);
 
        cocos2d::Application::getInstance()->run();
    }
    else
    {
        cocos2d::Director::getInstance()->resetMatrixStack();
        cocos2d::EventCustom recreatedEvent(EVENT_RENDERER_RECREATED);
        director->getEventDispatcher()->dispatchEvent(&recreatedEvent);
        director->setGLDefaultValues();
        cocos2d::VolatileTextureMgr::reloadAllTextures();
    }
    cocos2d::network::_preloadJavaDownloaderClass();
}

```

重点是`cocos2d::Application::getInstance()->run()`

```
int Application::run()
{
    // Initialize instance and cocos2d.
    if (! applicationDidFinishLaunching())
    {
        return 0;
    }
 
    return -1;
}

```

applicationDidFinishLaunching，没错这就是游戏逻辑的入口了

```
bool AppDelegate::applicationDidFinishLaunching()
{
    // set default FPS
    Director::getInstance()->setAnimationInterval(1.0 / 60.0f);
 
    // register lua module
 
      // 初始化 LuaEngine, 在 getInstance 中会初始化 LuaStack, LuaStack 初始化 Lua 环境相关
    auto engine = LuaEngine::getInstance();
    // 将 LuaEngine 添加到脚本引擎管理器 ScriptEngineManager 中
    ScriptEngineManager::getInstance()->setScriptEngine(engine);
    // 获取 Lua 环境
    lua_State* L = engine->getLuaStack()->getLuaState();
    // 注册额外的 C++ API 相关，比如 cocosstudio, spine, audio 相关
    lua_module_register(L);
 
    // 设置 cocos 自带的加密相关
    register_all_packages();
 
    // 在 LuaStack::executeScriptFile 执行脚本文件时，会通过 LuaStack::luaLoadBuffer 对文件进行解密
    LuaStack* stack = engine->getLuaStack();
    stack->setXXTEAKeyAndSign("2dxLua", strlen("2dxLua"), "XXTEA", strlen("XXTEA"));
 
    //register custom function
    //LuaStack* stack = engine->getLuaStack();
    //register_custom_function(stack->getLuaState());
 
#if CC_64BITS
    FileUtils::getInstance()->addSearchPath("src/64bit");
#endif
    FileUtils::getInstance()->addSearchPath("src");
    FileUtils::getInstance()->addSearchPath("res");
    if (engine->executeScriptFile("main.lua"))
    {
        return false;
    }
 
    return true;
}

```

`LuaEngine::getInstance();`这段代码实例化了 LuaEngine

 

下面分析 LuaEngine 初始化过程

```
LuaEngine* LuaEngine::getInstance(void)
{
    if (!_defaultEngine)
    {
        _defaultEngine = new (std::nothrow) LuaEngine();
        _defaultEngine->init();
    }
    return _defaultEngine;
}

```

接下来是`_defaultEngine->init();`

```
bool LuaEngine::init(void)
{
    _stack = LuaStack::create();
    _stack->retain();
    return true;
}

```

继续进入

```
LuaStack *LuaStack::create()
{
    LuaStack *stack = new (std::nothrow) LuaStack();
    stack->init();
    stack->autorelease();
    return stack;
}

```

下面是`stack->init()`

```
bool LuaStack::init()
{
    // 初始化Lua环境并打开标准库
    _state = lua_open();
    luaL_openlibs(_state);
    toluafix_open(_state);
 
    // Register our version of the global "print" function
    // 注册全局函数print到lua中，它会覆盖lua库中的print方法
    const luaL_Reg global_functions [] = {
        {"print", lua_print},
        {"release_print",lua_release_print},
        {nullptr, nullptr}
    };
    // 注册全局变量
    luaL_register(_state, "_G", global_functions);
 
    // 注册cocos2d-x引擎的API到lua环境中
    g_luaType.clear();
    register_all_cocos2dx(_state);
    register_all_cocos2dx_backend(_state);
    register_all_cocos2dx_manual(_state);
    register_all_cocos2dx_module_manual(_state);
    register_all_cocos2dx_math_manual(_state);
    register_all_cocos2dx_shaders_manual(_state);
    register_all_cocos2dx_bytearray_manual(_state);
 
    tolua_luanode_open(_state);
    register_luanode_manual(_state);
#if CC_USE_PHYSICS
    // 导入使用的physics相关API
    register_all_cocos2dx_physics(_state);
    register_all_cocos2dx_physics_manual(_state);
#endif
 
#if (CC_TARGET_PLATFORM == CC_PLATFORM_IOS || CC_TARGET_PLATFORM == CC_PLATFORM_MAC)
    // 导入ios下调用object-c相关API
    LuaObjcBridge::luaopen_luaoc(_state);
#endif
 
#if (CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID)
    // 导入android下调用java相关API
    LuaJavaBridge::luaopen_luaj(_state);
#endif
    register_all_cocos2dx_deprecated(_state);
    register_all_cocos2dx_manual_deprecated(_state);
 
    tolua_script_handler_mgr_open(_state);
 
    // add cocos2dx loader
    // 添加Lua的加载器,该方法将cocos2dx_lua_loader方法添加到Lua全局变量package下的loaders成员中
　　 // 当requires加载脚本时，Lua会使用package下的loaders中的加载器，即cocos2dx_lua_loader来加载
    // 设定cocos2dx_lua_loader，可以使得我们自定义设置搜索路径相关，且拓展实现对脚本的加密解密相关
    addLuaLoader(cocos2dx_lua_loader);
 
    return true;
}

```

重点在`cocos2dx_lua_loader`方法 (这个方法在`frameworks\cocos2d-x\cocos\scripting\lua-bindings\manual\Cocos2dxLuaLoader.cpp`)

```
int cocos2dx_lua_loader(lua_State *L)
    {
        // 后缀为luac和lua
        static const std::string BYTECODE_FILE_EXT    = ".luac";
        static const std::string NOT_BYTECODE_FILE_EXT = ".lua";
 
        // require传入的要加载的文件名，例如：require("a.b") 查找文件为：a/b.lua
        std::string filename(luaL_checkstring(L, 1));
        size_t pos = filename.rfind(BYTECODE_FILE_EXT);
         // 去掉后缀名".luac"或“.lua”
        if (pos != std::string::npos && pos == filename.length() - BYTECODE_FILE_EXT.length())
            filename = filename.substr(0, pos);
        else
        {
            pos = filename.rfind(NOT_BYTECODE_FILE_EXT);
            if (pos != std::string::npos && pos == filename.length() - NOT_BYTECODE_FILE_EXT.length())
                filename = filename.substr(0, pos);
        }
 
        // 将 "." 替换为 "/"
        pos = filename.find_first_of('.');
        while (pos != std::string::npos)
        {
            filename.replace(pos, 1, "/");
            pos = filename.find_first_of('.');
        }
 
        // search file in package.path
        Data chunk;
        std::string chunkName;
        FileUtils* utils = FileUtils::getInstance();
 
        // 获取 package.path 的变量
        lua_getglobal(L, "package");
        lua_getfield(L, -1, "path");
        // 通过 package.path 获取搜索路径相关，该路径为模版路径，格式类似于：
        // ?; ?.lua; c:\Users\?;  /usr/local/lua/lua/?/?.lua 以“;”作为分割符
        std::string searchpath(lua_tostring(L, -1));
        lua_pop(L, 1);
        size_t begin = 0;
        size_t next = searchpath.find_first_of(';', 0);
 
        // 遍历 package.path 中的所有路径，查找文件是否存在，若文件存在则通过 getDataFromFile 读取文件数据
        do
        {
            if (next == std::string::npos)
                next = searchpath.length();
            std::string prefix = searchpath.substr(begin, next-begin);
            if (prefix[0] == '.' && prefix[1] == '/')
                prefix = prefix.substr(2);
 
            pos = prefix.rfind(BYTECODE_FILE_EXT);
            if (pos != std::string::npos && pos == prefix.length() - BYTECODE_FILE_EXT.length())
            {
                prefix = prefix.substr(0, pos);
            }
            else
            {
                pos = prefix.rfind(NOT_BYTECODE_FILE_EXT);
                if (pos != std::string::npos && pos == prefix.length() - NOT_BYTECODE_FILE_EXT.length())
                    prefix = prefix.substr(0, pos);
            }
            pos = prefix.find_first_of('?', 0);
            while (pos != std::string::npos)
            {
                prefix.replace(pos, 1, filename);
                pos = prefix.find_first_of('?', pos + filename.length() + 1);
            }
 
            chunkName = prefix + BYTECODE_FILE_EXT;
            if (utils->isFileExist(chunkName)) // && !utils->isDirectoryExist(chunkName))
            {
                chunk = utils->getDataFromFile(chunkName);
                break;
            }
            else
            {
                chunkName = prefix + NOT_BYTECODE_FILE_EXT;
                if (utils->isFileExist(chunkName) ) //&& !utils->isDirectoryExist(chunkName))
                {
                    chunk = utils->getDataFromFile(chunkName);
                    break;
                }
                else
                {
                    chunkName = prefix;
                    if (utils->isFileExist(chunkName)) // && !utils->isDirectoryExist(chunkName))
                    {
                        chunk = utils->getDataFromFile(chunkName);
                        break;
                    }
                }
            }
 
            // 指定搜素路径下不存在该文件, 下一个
            begin = next + 1;
            next = searchpath.find_first_of(';', begin);
        } while (begin < searchpath.length());
        // 判断文件内容是否获取成功
        if (chunk.getSize() > 0)
        {
            // 加载文件
            LuaStack* stack = LuaEngine::getInstance()->getLuaStack();
            stack->luaLoadBuffer(L, reinterpret_cast(chunk.getBytes()),
                                 static_cast(chunk.getSize()), chunkName.c_str());
        }
        else
        {
            CCLOG("can not get file data of %s", chunkName.c_str());
            return 0;
        }
 
        return 1;
    } 
```

通过此处的代码，可以了解到 cocos2d-x 是如何搜索指定的 lua 文件

 

同时也会明白 require 为何可以使用 "." 来设定文件路径了，比如：

```
require "cocos.cocos2d.Cocos2d"
require "cocos.cocos2d.Cocos2dConstants"
require "cocos.cocos2d.functions"

```

再来看下`stack->luaLoadBuffer`的实现：

```
int LuaStack::luaLoadBuffer(lua_State *L, const char *chunk, int chunkSize, const char *chunkName)
{
    int r = 0;
 
    // 判断是否加密，若lua脚本加密，则解密后在加载脚本文件
    // luaL_loadbuffer 用于加载并编译Lua代码，并将其压入栈中
    if (_xxteaEnabled && strncmp(chunk, _xxteaSign, _xxteaSignLen) == 0)
    {
        // decrypt XXTEA
        xxtea_long len = 0;
        unsigned char* result = xxtea_decrypt((unsigned char*)chunk + _xxteaSignLen,
                                              (xxtea_long)chunkSize - _xxteaSignLen,
                                              (unsigned char*)_xxteaKey,
                                              (xxtea_long)_xxteaKeyLen,
                                              &len);
        unsigned char* content = result;
        xxtea_long contentSize = len;
        skipBOM((const char*&)content, (int&)contentSize);
        r = luaL_loadbuffer(L, (char*)content, contentSize, chunkName);
        free(result);
    }
    else
    {
        skipBOM(chunk, chunkSize);
        r = luaL_loadbuffer(L, chunk, chunkSize, chunkName);
    }
 
// 判定内容是否存在错误
#if defined(COCOS2D_DEBUG) && COCOS2D_DEBUG > 0
    if (r)
    {
        switch (r)
        {
            case LUA_ERRSYNTAX:
                // 语法错误
                CCLOG("[LUA ERROR] load \"%s\", error: syntax error during pre-compilation.", chunkName);
                break;
 
            case LUA_ERRMEM:
                // 内存分配错误
                CCLOG("[LUA ERROR] load \"%s\", error: memory allocation error.", chunkName);
                break;
 
            case LUA_ERRFILE:
                 // 文件错误
                CCLOG("[LUA ERROR] load \"%s\", error: cannot open/read file.", chunkName);
                break;
 
            default:
                // 未知错误
                CCLOG("[LUA ERROR] load \"%s\", error: unknown.", chunkName);
        }
    }
#endif
    return r;
}

```

至此，Lua 文件的加载流程结束

 

下面是 Lua 文件主要加载流程图

 

![](https://bbs.pediy.com/upload/attach/202107/883754_YWCWE24SJXZCA7Q.png)

Lua 解密之自定义文件头
=============

了解了 Coco2d-x Lua 文件基本的加载流程，可以帮助我们很快的定位关键方法

### 快速定位

由于所有的 Lua 文件加载必定经过`cocos2d::LuaStack::luaLoadBuffer`，所以可以直接定位，进行回溯

 

定位至`cocos2dx_lua_loader`

 

![](https://bbs.pediy.com/upload/attach/202107/883754_YDNVDXVYFXEEWDJ.png)

 

在`luaLoadBuffer`前调用了`decodeLuaData`，怀疑这就是解密 Lua 的关键方法

 

下面是伪代码

```
_BYTE *__fastcall decodeLuaData(cocos2d::Data *a1, int *size)
{
  _BYTE *v3; // r0
  _BYTE *v4; // r5
  int v6; // r0
  int v7; // r2
  int v8; // r3
  bool v9; // cc
  unsigned int v10; // r1
  _BYTE *v11; // r12
  _BYTE *v12; // r0
  char v13; // t1
  _BYTE *v14; // r0
  char v15[4]; // [sp+0h] [bp-18h]
 
  v3 = (_BYTE *)cocos2d::Data::getBytes(a1);
  v4 = v3;
  if ( *size > 8 && *v3 == 'a' && v3[1] == 'b' && v3[2] == 'c' && v3[3] == 'd' )
  {
    v6 = _hexToDecimal(v3 + 4, 4);
    v7 = *size;
    v8 = 0;
    v9 = *size <= 8;
    v15[3] = v6;
    v15[2] = (unsigned __int16)(v6 - 2048) >> 8;
    v10 = (unsigned int)(v6 - 2048) >> 24;
    v15[1] = (unsigned int)(v6 - 2048) >> 16;
    v15[0] = (unsigned int)(v6 - 2048) >> 24;
    if ( !v9 )
    {
      v11 = v4 - 1;
      v12 = v4 + 7;
      while ( 1 )
      {
        v13 = *++v12;
        ++v8;
        *++v11 = v13 ^ v10;
        v7 = *size;
        if ( *size <= v8 + 8 )
          break;
        LOBYTE(v10) = v15[v8 & 3];
      }
    }
    v14 = &v4[v7 - 8];
    *v14 = 0;
    v14[1] = 0;
    v14[2] = 0;
    v14[3] = 0;
    v14[4] = 0;
    v14[5] = 0;
    v14[6] = 0;
    v14[7] = 0;
    *size -= 8;
  }
  return v4;
}

```

此方法首先获取文件数据，而后判断文件头，文件头正是`abcd`

 

可以肯定这就是解密`.lua`的算法

 

如果不确定，我们可以使用 Frida 进行 Hook 验证猜想

 

我这里就不 Hook 了

算法还原
----

要实现算法的还原，我们就要依据伪代码或是汇编代码翻译成可用代码，可以发现 IDA 所反汇编的伪代码中包含了一些特定的宏，例如：`LOBYTE`

 

要了解这些宏的用途，我们可以参考`ida_root/plugins/def.h`

 

我这里使用了 Python

```
import struct
from loguru import logger
 
 
def _hexToDecimal(data, offset):
    if offset <= 0:
        return 0
    return struct.unpack(">I", data[:offset])[0]
 
 
def LOBYTE(d):
    return d & 0xff
 
 
def decodeLuaData(data, size):
    v4 = bytearray(data)
    v15 = [0] * 4
    if size > 8 and data[0] == ord('a') and data[1] == ord('b') and data[2] == ord('c') and data[3] == ord('d'):
        logger.info("find special exts")
        v6 = _hexToDecimal(data[4:], 4)
 
        v15[3] = v6
        v15[2] = (v6 - 2048) >> 8
        v10 = (v6 - 2048) >> 24
        v15[1] = (v6 - 2048) >> 16
        v15[0] = (v6 - 2048) >> 24
 
        if size > 8:
            i = 0
            while 1:
                v13 = v4[8 + i]
                v4[i] = v13 ^ v10
                i += 1
                if size <= i + 8:
                    break
                v10 = LOBYTE(v15[i & 3])
        size -= 8
 
        return v4[:-8]

```

解密后，发现还是 LuaJit，那么直接类同`.lua64`反汇编即可

 

![](https://bbs.pediy.com/upload/attach/202107/883754_8CJJ7QQK6KP9TCF.png)

Lua 资源加载流程
==========

回到 Lua 文件加载流程分析时提到的`LuaStack::init()`方法中，调用了`register_all_cocos2dx`

```
TOLUA_API int register_all_cocos2dx(lua_State* tolua_S)
{
    tolua_open(tolua_S);
 
    tolua_module(tolua_S,"cc",0);
    tolua_beginmodule(tolua_S,"cc");
 
    lua_register_cocos2dx_Ref(tolua_S);
    lua_register_cocos2dx_Material(tolua_S);
    lua_register_cocos2dx_Console(tolua_S);
    lua_register_cocos2dx_Node(tolua_S);
    lua_register_cocos2dx_Scene(tolua_S);
    lua_register_cocos2dx_TransitionScene(tolua_S);
    lua_register_cocos2dx_TransitionEaseScene(tolua_S);
    lua_register_cocos2dx_TransitionMoveInL(tolua_S);
    lua_register_cocos2dx_TransitionMoveInB(tolua_S);
    lua_register_cocos2dx_AtlasNode(tolua_S);
    lua_register_cocos2dx_TileMapAtlas(tolua_S);
    lua_register_cocos2dx_TransitionMoveInT(tolua_S);
    lua_register_cocos2dx_TMXTilesetInfo(tolua_S);
    lua_register_cocos2dx_TransitionMoveInR(tolua_S);
    lua_register_cocos2dx_Action(tolua_S);
    lua_register_cocos2dx_FiniteTimeAction(tolua_S);
    lua_register_cocos2dx_ActionInstant(tolua_S);
    lua_register_cocos2dx_Hide(tolua_S);
    lua_register_cocos2dx_ParticleSystem(tolua_S);
    lua_register_cocos2dx_ParticleSystemQuad(tolua_S);
    lua_register_cocos2dx_ParticleSpiral(tolua_S);
    lua_register_cocos2dx_GridBase(tolua_S);
    lua_register_cocos2dx_AnimationCache(tolua_S);
    lua_register_cocos2dx_ActionInterval(tolua_S);
    lua_register_cocos2dx_ActionCamera(tolua_S);
    lua_register_cocos2dx_ProgressFromTo(tolua_S);
    lua_register_cocos2dx_MoveBy(tolua_S);
    lua_register_cocos2dx_MoveTo(tolua_S);
    lua_register_cocos2dx_JumpBy(tolua_S);
    lua_register_cocos2dx_EventListener(tolua_S);
    lua_register_cocos2dx_EventListenerKeyboard(tolua_S);
    lua_register_cocos2dx_EventListenerMouse(tolua_S);
    lua_register_cocos2dx_TransitionRotoZoom(tolua_S);
    lua_register_cocos2dx_Event(tolua_S);
    lua_register_cocos2dx_EventController(tolua_S);
    lua_register_cocos2dx_Director(tolua_S);
    lua_register_cocos2dx_Scheduler(tolua_S);
    lua_register_cocos2dx_ActionEase(tolua_S);
    lua_register_cocos2dx_EaseElastic(tolua_S);
    lua_register_cocos2dx_EaseElasticOut(tolua_S);
    lua_register_cocos2dx_EaseQuadraticActionInOut(tolua_S);
    lua_register_cocos2dx_EaseBackOut(tolua_S);
    lua_register_cocos2dx_Texture2D(tolua_S);
    lua_register_cocos2dx_TransitionSceneOriented(tolua_S);
    lua_register_cocos2dx_TransitionFlipX(tolua_S);
    lua_register_cocos2dx_CameraBackgroundBrush(tolua_S);
    lua_register_cocos2dx_CameraBackgroundDepthBrush(tolua_S);
    lua_register_cocos2dx_CameraBackgroundColorBrush(tolua_S);
    lua_register_cocos2dx_GridAction(tolua_S);
    lua_register_cocos2dx_TiledGrid3DAction(tolua_S);
    lua_register_cocos2dx_FadeOutTRTiles(tolua_S);
    lua_register_cocos2dx_FadeOutUpTiles(tolua_S);
    lua_register_cocos2dx_FadeOutDownTiles(tolua_S);
    lua_register_cocos2dx_StopGrid(tolua_S);
    lua_register_cocos2dx_Technique(tolua_S);
    lua_register_cocos2dx_SkewTo(tolua_S);
    lua_register_cocos2dx_SkewBy(tolua_S);
    lua_register_cocos2dx_EaseQuadraticActionOut(tolua_S);
    lua_register_cocos2dx_TransitionProgress(tolua_S);
    lua_register_cocos2dx_TransitionProgressVertical(tolua_S);
    lua_register_cocos2dx_Layer(tolua_S);
    lua_register_cocos2dx_TMXTiledMap(tolua_S);
    lua_register_cocos2dx_Grid3DAction(tolua_S);
    lua_register_cocos2dx_BaseLight(tolua_S);
    lua_register_cocos2dx_SpotLight(tolua_S);
    lua_register_cocos2dx_FadeTo(tolua_S);
    lua_register_cocos2dx_FadeIn(tolua_S);
    lua_register_cocos2dx_DirectionLight(tolua_S);
    lua_register_cocos2dx_EventListenerCustom(tolua_S);
    lua_register_cocos2dx_FlipX3D(tolua_S);
    lua_register_cocos2dx_FlipY3D(tolua_S);
    lua_register_cocos2dx_EaseSineInOut(tolua_S);
    lua_register_cocos2dx_TransitionFlipAngular(tolua_S);
    lua_register_cocos2dx_EaseElasticInOut(tolua_S);
    lua_register_cocos2dx_EaseBounce(tolua_S);
    lua_register_cocos2dx_Show(tolua_S);
    lua_register_cocos2dx_FadeOut(tolua_S);
    lua_register_cocos2dx_CallFunc(tolua_S);
    lua_register_cocos2dx_EventMouse(tolua_S);
    lua_register_cocos2dx_GLView(tolua_S);
    lua_register_cocos2dx_EaseBezierAction(tolua_S);
    lua_register_cocos2dx_ParticleFireworks(tolua_S);
    lua_register_cocos2dx_MenuItem(tolua_S);
    lua_register_cocos2dx_MenuItemSprite(tolua_S);
    lua_register_cocos2dx_MenuItemImage(tolua_S);
    lua_register_cocos2dx_AutoPolygon(tolua_S);
    lua_register_cocos2dx_ParticleSmoke(tolua_S);
    lua_register_cocos2dx_TransitionZoomFlipAngular(tolua_S);
    lua_register_cocos2dx_EaseRateAction(tolua_S);
    lua_register_cocos2dx_EaseIn(tolua_S);
    lua_register_cocos2dx_EaseExponentialInOut(tolua_S);
    lua_register_cocos2dx_CardinalSplineTo(tolua_S);
    lua_register_cocos2dx_CatmullRomTo(tolua_S);
    lua_register_cocos2dx_Waves3D(tolua_S);
    lua_register_cocos2dx_EaseExponentialOut(tolua_S);
    lua_register_cocos2dx_Label(tolua_S);
    lua_register_cocos2dx_Application(tolua_S);
    lua_register_cocos2dx_DelayTime(tolua_S);
    lua_register_cocos2dx_LabelAtlas(tolua_S);
    lua_register_cocos2dx_EaseCircleActionOut(tolua_S);
    lua_register_cocos2dx_SpriteBatchNode(tolua_S);
    lua_register_cocos2dx_TMXLayer(tolua_S);
    lua_register_cocos2dx_AsyncTaskPool(tolua_S);
    lua_register_cocos2dx_ParticleSnow(tolua_S);
    lua_register_cocos2dx_EaseElasticIn(tolua_S);
    lua_register_cocos2dx_EaseCircleActionInOut(tolua_S);
    lua_register_cocos2dx_TransitionFadeTR(tolua_S);
    lua_register_cocos2dx_EaseQuarticActionOut(tolua_S);
    lua_register_cocos2dx_EventAcceleration(tolua_S);
    lua_register_cocos2dx_EaseCubicActionIn(tolua_S);
    lua_register_cocos2dx_TextureCache(tolua_S);
    lua_register_cocos2dx_ActionTween(tolua_S);
    lua_register_cocos2dx_TransitionFadeDown(tolua_S);
    lua_register_cocos2dx_ParticleSun(tolua_S);
    lua_register_cocos2dx_TransitionProgressHorizontal(tolua_S);
    lua_register_cocos2dx_ParticleFire(tolua_S);
    lua_register_cocos2dx_FlipX(tolua_S);
    lua_register_cocos2dx_FlipY(tolua_S);
    lua_register_cocos2dx_EventKeyboard(tolua_S);
    lua_register_cocos2dx_TransitionSplitCols(tolua_S);
    lua_register_cocos2dx_Timer(tolua_S);
    lua_register_cocos2dx_RepeatForever(tolua_S);
    lua_register_cocos2dx_Place(tolua_S);
    lua_register_cocos2dx_EventListenerAcceleration(tolua_S);
    lua_register_cocos2dx_TiledGrid3D(tolua_S);
    lua_register_cocos2dx_EaseBounceOut(tolua_S);
    lua_register_cocos2dx_RenderTexture(tolua_S);
    lua_register_cocos2dx_TintBy(tolua_S);
    lua_register_cocos2dx_TransitionShrinkGrow(tolua_S);
    lua_register_cocos2dx_ClippingNode(tolua_S);
    lua_register_cocos2dx_ActionFloat(tolua_S);
    lua_register_cocos2dx_ParticleFlower(tolua_S);
    lua_register_cocos2dx_EaseCircleActionIn(tolua_S);
    lua_register_cocos2dx_Image(tolua_S); // 加载图片资源
    lua_register_cocos2dx_LayerMultiplex(tolua_S);
    lua_register_cocos2dx_Blink(tolua_S);
    lua_register_cocos2dx_ShaderCache(tolua_S);
    lua_register_cocos2dx_JumpTo(tolua_S);
    lua_register_cocos2dx_ParticleExplosion(tolua_S);
    lua_register_cocos2dx_TransitionJumpZoom(tolua_S);
    lua_register_cocos2dx_Pass(tolua_S);
    lua_register_cocos2dx_Touch(tolua_S);
    lua_register_cocos2dx_CardinalSplineBy(tolua_S);
    lua_register_cocos2dx_CatmullRomBy(tolua_S);
    lua_register_cocos2dx_NodeGrid(tolua_S);
    lua_register_cocos2dx_TMXLayerInfo(tolua_S);
    lua_register_cocos2dx_EaseSineIn(tolua_S);
    lua_register_cocos2dx_EaseBounceIn(tolua_S);
    lua_register_cocos2dx_Camera(tolua_S);
    lua_register_cocos2dx_TMXObjectGroup(tolua_S);
    lua_register_cocos2dx_FastTMXTiledMap(tolua_S);
    lua_register_cocos2dx_ParticleGalaxy(tolua_S);
    lua_register_cocos2dx_Twirl(tolua_S);
    lua_register_cocos2dx_MenuItemLabel(tolua_S);
    lua_register_cocos2dx_EaseQuinticActionIn(tolua_S);
    lua_register_cocos2dx_LayerColor(tolua_S);
    lua_register_cocos2dx_FadeOutBLTiles(tolua_S);
    lua_register_cocos2dx_LayerGradient(tolua_S);
    lua_register_cocos2dx_EventListenerTouchAllAtOnce(tolua_S);
    lua_register_cocos2dx_GLViewImpl(tolua_S);
    lua_register_cocos2dx_ToggleVisibility(tolua_S);
    lua_register_cocos2dx_Repeat(tolua_S);
    lua_register_cocos2dx_TransitionFlipY(tolua_S);
    lua_register_cocos2dx_TurnOffTiles(tolua_S);
    lua_register_cocos2dx_TintTo(tolua_S);
    lua_register_cocos2dx_EaseBackInOut(tolua_S);
    lua_register_cocos2dx_TransitionFadeBL(tolua_S);
    lua_register_cocos2dx_TargetedAction(tolua_S);
    lua_register_cocos2dx_DrawNode(tolua_S);
    lua_register_cocos2dx_TransitionTurnOffTiles(tolua_S);
    lua_register_cocos2dx_RotateTo(tolua_S);
    lua_register_cocos2dx_TransitionSplitRows(tolua_S);
    lua_register_cocos2dx_Device(tolua_S);
    lua_register_cocos2dx_TransitionProgressRadialCCW(tolua_S);
    lua_register_cocos2dx_ScaleTo(tolua_S);
    lua_register_cocos2dx_TransitionPageTurn(tolua_S);
    lua_register_cocos2dx_RenderState(tolua_S);
    lua_register_cocos2dx_Properties(tolua_S);
    lua_register_cocos2dx_BezierBy(tolua_S);
    lua_register_cocos2dx_BezierTo(tolua_S);
    lua_register_cocos2dx_ParticleMeteor(tolua_S);
    lua_register_cocos2dx_SpriteFrame(tolua_S);
    lua_register_cocos2dx_Liquid(tolua_S);
    lua_register_cocos2dx_UserDefault(tolua_S);
    lua_register_cocos2dx_FastTMXLayer(tolua_S);
    lua_register_cocos2dx_TransitionZoomFlipX(tolua_S);
    lua_register_cocos2dx_EventFocus(tolua_S);
    lua_register_cocos2dx_TransitionFade(tolua_S);
    lua_register_cocos2dx_EaseQuinticActionInOut(tolua_S);
    lua_register_cocos2dx_SpriteFrameCache(tolua_S);
    lua_register_cocos2dx_PointLight(tolua_S);
    lua_register_cocos2dx_TransitionCrossFade(tolua_S);
    lua_register_cocos2dx_Ripple3D(tolua_S);
    lua_register_cocos2dx_Lens3D(tolua_S);
    lua_register_cocos2dx_EventListenerFocus(tolua_S);
    lua_register_cocos2dx_Spawn(tolua_S);
    lua_register_cocos2dx_EaseQuarticActionInOut(tolua_S);
    lua_register_cocos2dx_ShakyTiles3D(tolua_S);
    lua_register_cocos2dx_PageTurn3D(tolua_S);
    lua_register_cocos2dx_PolygonInfo(tolua_S);
    lua_register_cocos2dx_TransitionSlideInL(tolua_S);
    lua_register_cocos2dx_TransitionSlideInT(tolua_S);
    lua_register_cocos2dx_Grid3D(tolua_S);
    lua_register_cocos2dx_EventListenerController(tolua_S);
    lua_register_cocos2dx_TransitionProgressInOut(tolua_S);
    lua_register_cocos2dx_EaseCubicActionInOut(tolua_S);
    lua_register_cocos2dx_ParticleData(tolua_S);
    lua_register_cocos2dx_EaseBackIn(tolua_S);
    lua_register_cocos2dx_SplitRows(tolua_S);
    lua_register_cocos2dx_Follow(tolua_S);
    lua_register_cocos2dx_Animate(tolua_S);
    lua_register_cocos2dx_ShuffleTiles(tolua_S);
    lua_register_cocos2dx_CameraBackgroundSkyBoxBrush(tolua_S);
    lua_register_cocos2dx_ProgressTimer(tolua_S);
    lua_register_cocos2dx_EaseQuarticActionIn(tolua_S);
    lua_register_cocos2dx_Menu(tolua_S);
    lua_register_cocos2dx_EaseInOut(tolua_S);
    lua_register_cocos2dx_TransitionZoomFlipY(tolua_S);
    lua_register_cocos2dx_ScaleBy(tolua_S);
    lua_register_cocos2dx_EventTouch(tolua_S);
    lua_register_cocos2dx_Animation(tolua_S);
    lua_register_cocos2dx_TMXMapInfo(tolua_S);
    lua_register_cocos2dx_EaseExponentialIn(tolua_S);
    lua_register_cocos2dx_ReuseGrid(tolua_S);
    lua_register_cocos2dx_EaseQuinticActionOut(tolua_S);
    lua_register_cocos2dx_EventDispatcher(tolua_S);
    lua_register_cocos2dx_MenuItemAtlasFont(tolua_S);
    lua_register_cocos2dx_ActionManager(tolua_S);
    lua_register_cocos2dx_OrbitCamera(tolua_S);
    lua_register_cocos2dx_ClippingRectangleNode(tolua_S);
    lua_register_cocos2dx_EventCustom(tolua_S);
    lua_register_cocos2dx_ParticleBatchNode(tolua_S);
    lua_register_cocos2dx_Component(tolua_S);
    lua_register_cocos2dx_EaseCubicActionOut(tolua_S);
    lua_register_cocos2dx_EventListenerTouchOneByOne(tolua_S);
    lua_register_cocos2dx_Renderer(tolua_S);
    lua_register_cocos2dx_ParticleRain(tolua_S);
    lua_register_cocos2dx_Waves(tolua_S);
    lua_register_cocos2dx_ComponentLua(tolua_S);
    lua_register_cocos2dx_MotionStreak3D(tolua_S);
    lua_register_cocos2dx_EaseOut(tolua_S);
    lua_register_cocos2dx_MenuItemFont(tolua_S);
    lua_register_cocos2dx_TransitionFadeUp(tolua_S);
    lua_register_cocos2dx_LayerRadialGradient(tolua_S);
    lua_register_cocos2dx_EaseSineOut(tolua_S);
    lua_register_cocos2dx_JumpTiles3D(tolua_S);
    lua_register_cocos2dx_MenuItemToggle(tolua_S);
    lua_register_cocos2dx_RemoveSelf(tolua_S);
    lua_register_cocos2dx_SplitCols(tolua_S);
    lua_register_cocos2dx_ProtectedNode(tolua_S);
    lua_register_cocos2dx_MotionStreak(tolua_S);
    lua_register_cocos2dx_RotateBy(tolua_S);
    lua_register_cocos2dx_FileUtils(tolua_S);
    lua_register_cocos2dx_Sprite(tolua_S);
    lua_register_cocos2dx_ProgressTo(tolua_S);
    lua_register_cocos2dx_TransitionProgressOutIn(tolua_S);
    lua_register_cocos2dx_AnimationFrame(tolua_S);
    lua_register_cocos2dx_Sequence(tolua_S);
    lua_register_cocos2dx_Shaky3D(tolua_S);
    lua_register_cocos2dx_TransitionProgressRadialCW(tolua_S);
    lua_register_cocos2dx_EaseBounceInOut(tolua_S);
    lua_register_cocos2dx_TransitionSlideInR(tolua_S);
    lua_register_cocos2dx_AmbientLight(tolua_S);
    lua_register_cocos2dx_ParallaxNode(tolua_S);
    lua_register_cocos2dx_EaseQuadraticActionIn(tolua_S);
    lua_register_cocos2dx_WavesTiles3D(tolua_S);
    lua_register_cocos2dx_TransitionSlideInB(tolua_S);
    lua_register_cocos2dx_Speed(tolua_S);
    lua_register_cocos2dx_ShatteredTiles3D(tolua_S);
 
    tolua_endmodule(tolua_S);
    return 1;
}

```

在大量注册函数中寻找到关键方法`lua_register_cocos2dx_Image`

```
int lua_register_cocos2dx_Image(lua_State* tolua_S)
{
    tolua_usertype(tolua_S,"cc.Image");
    tolua_cclass(tolua_S,"Image","cc.Image","cc.Ref",nullptr);
 
    tolua_beginmodule(tolua_S,"Image");
        tolua_function(tolua_S,"new",lua_cocos2dx_Image_constructor);
        tolua_function(tolua_S,"hasPremultipliedAlpha",lua_cocos2dx_Image_hasPremultipliedAlpha);
        tolua_function(tolua_S,"reversePremultipliedAlpha",lua_cocos2dx_Image_reversePremultipliedAlpha);
        tolua_function(tolua_S,"isCompressed",lua_cocos2dx_Image_isCompressed);
        tolua_function(tolua_S,"hasAlpha",lua_cocos2dx_Image_hasAlpha);
        tolua_function(tolua_S,"getPixelFormat",lua_cocos2dx_Image_getPixelFormat);
        tolua_function(tolua_S,"getHeight",lua_cocos2dx_Image_getHeight);
        tolua_function(tolua_S,"premultiplyAlpha",lua_cocos2dx_Image_premultiplyAlpha);
        tolua_function(tolua_S,"initWithImageFile",lua_cocos2dx_Image_initWithImageFile); // 初始化图片文件
        tolua_function(tolua_S,"getWidth",lua_cocos2dx_Image_getWidth);
        tolua_function(tolua_S,"getBitPerPixel",lua_cocos2dx_Image_getBitPerPixel);
        tolua_function(tolua_S,"getFileType",lua_cocos2dx_Image_getFileType);
        tolua_function(tolua_S,"getFilePath",lua_cocos2dx_Image_getFilePath);
        tolua_function(tolua_S,"getNumberOfMipmaps",lua_cocos2dx_Image_getNumberOfMipmaps);
        tolua_function(tolua_S,"saveToFile",lua_cocos2dx_Image_saveToFile);
        tolua_function(tolua_S,"setPVRImagesHavePremultipliedAlpha", lua_cocos2dx_Image_setPVRImagesHavePremultipliedAlpha);
        tolua_function(tolua_S,"setPNGPremultipliedAlphaEnabled", lua_cocos2dx_Image_setPNGPremultipliedAlphaEnabled);
    tolua_endmodule(tolua_S);
    std::string typeName = typeid(cocos2d::Image).name();
    g_luaType[typeName] = "cc.Image";
    g_typeCast["Image"] = "cc.Image";
    return 1;
}

```

`lua_cocos2dx_Image_initWithImageFile`加载了我们的图片文件

```
int lua_cocos2dx_Image_initWithImageFile(lua_State* tolua_S)
{
    int argc = 0;
    cocos2d::Image* cobj = nullptr;
    bool ok  = true;
 
#if COCOS2D_DEBUG >= 1
    tolua_Error tolua_err;
#endif
 
 
#if COCOS2D_DEBUG >= 1
    if (!tolua_isusertype(tolua_S,1,"cc.Image",0,&tolua_err)) goto tolua_lerror;
#endif
 
    cobj = (cocos2d::Image*)tolua_tousertype(tolua_S,1,0);
 
#if COCOS2D_DEBUG >= 1
    if (!cobj)
    {
        tolua_error(tolua_S,"invalid 'cobj' in function 'lua_cocos2dx_Image_initWithImageFile'", nullptr);
        return 0;
    }
#endif
 
    argc = lua_gettop(tolua_S)-1;
    if (argc == 1)
    {
        std::string arg0;
 
        ok &= luaval_to_std_string(tolua_S, 2,&arg0, "cc.Image:initWithImageFile");
        if(!ok)
        {
            tolua_error(tolua_S,"invalid arguments in function 'lua_cocos2dx_Image_initWithImageFile'", nullptr);
            return 0;
        }
        bool ret = cobj->initWithImageFile(arg0);
        tolua_pushboolean(tolua_S,(bool)ret);
        return 1;
    }
    luaL_error(tolua_S, "%s has wrong number of arguments: %d, was expecting %d \n", "cc.Image:initWithImageFile",argc, 1);
    return 0;
 
#if COCOS2D_DEBUG >= 1
    tolua_lerror:
    tolua_error(tolua_S,"#ferror in function 'lua_cocos2dx_Image_initWithImageFile'.",&tolua_err);
#endif
 
    return 0;
}

```

下面看`cobj->initWithImageFile`

```
bool Image::initWithImageFile(const std::string& path)
{
    bool ret = false;
    _filePath = FileUtils::getInstance()->fullPathForFilename(path);
 
    Data data = FileUtils::getInstance()->getDataFromFile(_filePath); // 获取文件数据
 
    if (!data.isNull())
    {
        ret = initWithImageData(data.getBytes(), data.getSize()); // 解码/解密 处理文件数据
    }
 
    return ret;
}

```

下面是图片数据格式解码方法`initWithImageData`

```
bool Image::initWithImageData(const unsigned char * data, ssize_t dataLen)
{
    bool ret = false;
 
    do
    {
        CC_BREAK_IF(! data || dataLen <= 0);
 
        unsigned char* unpackedData = nullptr;
        ssize_t unpackedLen = 0;
 
        //detect and unzip the compress file
        if (ZipUtils::isCCZBuffer(data, dataLen))
        {
            unpackedLen = ZipUtils::inflateCCZBuffer(data, dataLen, &unpackedData);
        }
        else if (ZipUtils::isGZipBuffer(data, dataLen))
        {
            unpackedLen = ZipUtils::inflateMemory(const_cast(data), dataLen, &unpackedData);
        }
        else
        {
            unpackedData = const_cast(data);
            unpackedLen = dataLen;
        }
 
        _fileType = detectFormat(unpackedData, unpackedLen);
 
        switch (_fileType)
        {
        case Format::PNG:
            ret = initWithPngData(unpackedData, unpackedLen);
            break;
        case Format::JPG:
            ret = initWithJpgData(unpackedData, unpackedLen);
            break;
        case Format::WEBP:
            ret = initWithWebpData(unpackedData, unpackedLen);
            break;
        case Format::PVR:
            ret = initWithPVRData(unpackedData, unpackedLen);
            break;
        case Format::ETC:
            ret = initWithETCData(unpackedData, unpackedLen);
            break;
        case Format::S3TC:
            ret = initWithS3TCData(unpackedData, unpackedLen);
            break;
        case Format::ATITC:
            ret = initWithATITCData(unpackedData, unpackedLen);
            break;
        default:
            {
                // load and detect image format
                tImageTGA* tgaData = tgaLoadBuffer(unpackedData, unpackedLen);
 
                if (tgaData != nullptr && tgaData->status == TGA_OK)
                {
                    ret = initWithTGAData(tgaData);
                }
                else
                {
                    CCLOG("cocos2d: unsupported image format!");
                }
 
                free(tgaData);
                break;
            }
        }
 
        if(unpackedData != data)
        {
            free(unpackedData);
        }
    } while (0);
 
    return ret;
} 
```

图片资源的加载到这里就结束了

 

![](https://bbs.pediy.com/upload/attach/202107/883754_UJQZR6BB3JX2NTV.png)

图片资源解密
======

快速定位
----

根据调用流程，我们可以直接定位方法`cocos2d::Image::initWithImageData`

 

向上可以追溯到`cocos2d::Image::initWithImageFile`

 

![](https://bbs.pediy.com/upload/attach/202107/883754_6FNYXPWUBV3CWMD.png)

 

看到了可疑函数`cocos2d::Image::decodePngData`

```
cocos2d *__fastcall cocos2d::Image::decodePngData(cocos2d::Image *this, cocos2d::Data *a2)
{
  _BYTE *v4; // r7
  int v5; // r5
  signed int v6; // r0
  bool v7; // cc
  int v8; // r4
  void *v9; // r0
  int v10; // r2
  int v12; // r6
  int v13; // r4
  int v14; // r9
  int v15; // r2
  int v16; // r0
  int v17; // r3
  _BYTE *v18; // r5
  unsigned int *v19; // r2
  int v20; // r3
  int v21; // [sp+8h] [bp-28h] BYREF
 
  v4 = (_BYTE *)cocos2d::Data::getBytes(a2);
  v5 = cocos2d::Data::getSize(a2);
  sub_FBDCA8(&v21, (int *)this + 47);
  v6 = sub_FBC854((int)&v21, ".png", 0, 4u);
  v7 = v6 <= -1;
  if ( v6 != -1 )
    v7 = v5 <= 24;
  v8 = !v7;
  v9 = (void *)(v21 - 12);
  if ( (int *)(v21 - 12) != &dword_12F6978 )
  {
    if ( &pthread_create )
    {
      v19 = (unsigned int *)(v21 - 4);
      __dmb(0xFu);
      do
        v20 = __ldrex(v19);
      while ( __strex(v20 - 1, v19) );
      __dmb(0xFu);
    }
    else
    {
      v20 = *(_DWORD *)(v21 - 4);
      *(_DWORD *)(v21 - 4) = v20 - 1;
    }
    if ( v20 <= 0 )
      operator delete(v9);
  }
  if ( v8 && memcmp(&unk_11CBE0C, v4, 8u) )     // PNG Header
  {
    v12 = v5 - 4;
    v13 = v5 - 8;
    v14 = cocos2d::hexToDecimal((cocos2d *)&v4[v5 - 4], (unsigned __int8 *)&byte_4, v10);
    v16 = cocos2d::hexToDecimal((cocos2d *)&v4[v5 - 8], (unsigned __int8 *)&byte_4, v15);
    if ( v5 >= v16 )
      cocos2d::decodePng(v4, (unsigned __int8 *)(v14 - 2048), v16, v17);
    v18 = &v4[v5];
    *(v18 - 1) = 130;
    *(v18 - 2) = 96;
    *(v18 - 3) = 66;
    v4[v12] = 174;
    *(v18 - 5) = 68;
    *(v18 - 6) = 78;
    *(v18 - 7) = 69;
    v4[v13] = 73;
  }
  return (cocos2d *)v4;
}

```

明显是一个算法函数，看方法`cocos2d::decodePng`

```
_BYTE *__fastcall cocos2d::decodePng(_BYTE *this, unsigned __int8 *a2, int a3, int a4)
{
  char v4; // r12
  unsigned int v5; // r1
  int v6; // r12
  char v7[4]; // [sp+0h] [bp-10h]
 
  v7[1] = BYTE2(a2);
  v4 = BYTE1(a2);
  v7[3] = (char)a2;
  v5 = (unsigned int)a2 >> 24;
  v7[2] = v4;
  v6 = 0;
  v7[0] = v5;
  if ( a3 > 0 )
  {
    while ( 1 )
    {
      ++v6;
      *this++ ^= v5;
      if ( v6 == a3 )
        break;
      LOBYTE(v5) = v7[v6 % 4];
    }
  }
  return this;
}

```

这里也可以尝试 Frida Hook 验证猜想

 

同样的，我就不 Hook 了

算法还原
----

```
def _hexToDecimal(data, offset):
    if offset <= 0:
        return 0
    return struct.unpack(">I", data[:offset])[0]
 
 
def LOBYTE(d):
    return d & 0xff
 
 
def _decodePng(data, magic, rounds):
    dthis = bytearray(data)
 
    v7 = [0] * 4
    a2 = struct.pack("> 24
    v7[2] = a2[1]
 
    i = 0
    if rounds > 0:
        while 1:
            i += 1
            dthis[i-1] ^= v5
            if i == rounds:
                break
            v5 = LOBYTE(v7[i % 4])
    return dthis
 
 
def decodePng(data, size):
    v14 = _hexToDecimal(data[-4:], 4)
    v16 = _hexToDecimal(data[-8:], 4)
    if size >= v16:
        ret = _decodePng(data, v14 - 2048, v16)
        ret[-1] = 130
        ret[-2] = 96
        ret[-3] = 66
        ret[-4] = 174
        ret[-5] = 68
        ret[-6] = 78
        ret[-7] = 69
        ret[-8] = 73
    return ret 
```

解密结果

 

![](https://bbs.pediy.com/upload/attach/202107/883754_34NV5A8HC6F3TE5.png)

总结
==

写不出总结啦，大概最大的感想就是：对照源码逆向好爽！！！

[[注意] 招人！base 上海，课程运营、市场多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)

最后于 1 天前 被 sunfishi 编辑 ，原因：

[#逆向分析](forum-161-1-118.htm) [#NDK 分析](forum-161-1-119.htm) [#源码分析](forum-161-1-127.htm)
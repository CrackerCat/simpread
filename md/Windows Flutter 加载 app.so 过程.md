> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-2121108-1-1.html)

> [md]Windows Flutter 是怎么加载 app.so 的？从 wWinMain 一路跟到 Dart_LoadELF：为什么 Windows 上会出现一个「不像 Windows」的 .so 遇到......

![](https://avatar.52pojie.cn/data/avatar/002/09/05/26_avatar_middle.jpg)wolfSpicy

Windows Flutter 是怎么加载 app.so 的？

从 wWinMain 一路跟到 Dart_LoadELF：为什么 Windows 上会出现一个「不像 Windows」的 .so

遇到 Windows 版 Flutter 程序时，我一直有个疑问：

data\app.so 明明不是 PE，Windows 也不能 LoadLibrary 这种东西，它到底是怎么被加载进来的？

结论先说：

*   app.so 本质是 ELF 壳子包着的 Dart AOT 快照
*   Windows 不会拿系统加载器去加载它
*   Flutter / Dart 自己实现了一套 ELF Loader，用 CreateFileMapping / MapViewOfFile 一类接口把文件映射进进程
*   映射完以后，再从里面抽出 VM / Isolate 的 snapshot 数据，交给 Dart VM 跑

下面按源码调用链，从 exe 入口一路跟到真正打开 app.so 的地方。

示例程序以常见 Flutter Windows runner 为准（文中用 Reqable.exe 作示意），源码主要来自 Flutter / Engine / Dart SDK 公开仓库。

* * *

0.  总体流程先看一眼

先把整条链路摊开，后面每一节都在填这个骨架：

```
wWinMain                          [YourApp.exe]
  DartProject(L"data")
  FlutterWindow::Create / OnCreate
       │
       ▼
  FlutterViewController(w, h, project)     [client_wrapper，链进 exe]
       │  new FlutterEngine(project)
       │  FlutterDesktopEngineCreate(&props)   ← IAT → flutter_windows.dll
       │  FlutterDesktopViewControllerCreate
       ▼
flutter_windows.dll
  FlutterProjectBundle(props)              ← 相对路径 → 绝对路径
  FlutterWindowsEngine::Run
       │  LoadAotData()
       │  FlutterEngineCreateAOTData(ElfPath = ...\data\app.so)
       ▼
  CreateFileMapping / MapViewOfFile        ← 按 ELF 映射，不是 LoadLibrary
       │
       ▼
  进程中出现 app.so 映射区，Dart AOT 开始跑

```

读这条链路时，抓住两个分界就够了：

1.  exe → dll：FlutterDesktopEngineCreate 开始，真正进 flutter_windows.dll
2.  路径 → 内存：FlutterEngineCreateAOTData → Dart_LoadELF，这里才真正「打开」app.so

* * *

1.  入口：wWinMain

模板源码：

[https://github.com/flutter/flutter/blob/master/packages/flutter_tools/templates/app/windows.tmpl/runner/main.cpp.tmpl](https://github.com/flutter/flutter/blob/master/packages/flutter_tools/templates/app/windows.tmpl/runner/main.cpp.tmpl)

 ![](https://attach.52pojie.cn/forum/202608/03/211753fwwoyunz6yow6jzn.png)   
图 1　main.cpp：红线标出的三行，才是后续整条链路的起点

核心代码：

```
int APIENTRY wWinMain(_In_ HINSTANCE instance, _In_opt_ HINSTANCE prev,
                      _In_ wchar_t *command_line, _In_ int show_command) {
  if (!::AttachConsole(ATTACH_PARENT_PROCESS) && ::IsDebuggerPresent()) {
    CreateAndAttachConsole();
  }

  ::CoInitializeEx(nullptr, COINIT_APARTMENTTHREADED);

  flutter::DartProject project(L"data");

  std::vector<std::string> command_line_arguments =
      GetCommandLineArguments();

  project.set_dart_entrypoint_arguments(std::move(command_line_arguments));

  FlutterWindow window(project);
  Win32Window::Point origin(10, 10);
  Win32Window::Size size(1280, 720);
  if (!window.Create(L"{{projectName}}", origin, size)) {
    return EXIT_FAILURE;
  }
  window.SetQuitOnClose(true);

  ::MSG msg;
  while (::GetMessage(&msg, nullptr, 0, 0)) {
    ::TranslateMessage(&msg);
    ::DispatchMessage(&msg);
  }

  ::CoUninitialize();
  return EXIT_SUCCESS;
}

```

真正要盯的是这三行：

```
flutter::DartProject project(L"data");
FlutterWindow window(project);
if (!window.Create(L"{{projectName}}", origin, size)) {

```

大白话：

1.  告诉 Flutter：「资源目录就在 exe 旁边的 data」
2.  把这个 project 塞进窗口对象
3.  Create 时才会真正拉起引擎、建视图、后面才会碰到 app.so

前面那些 AttachConsole / CoInitializeEx / 消息循环，都是普通 Win32 壳子，跟 app.so 加载关系不大。

* * *

2.  DartProject("data")：把三个关键文件路径拼出来

头文件：

[https://github.com/flutter-team-archive/engine/blob/main/shell/platform/windows/client_wrapper/include/flutter/dart_project.h](https://github.com/flutter-team-archive/engine/blob/main/shell/platform/windows/client_wrapper/include/flutter/dart_project.h)

 ![](https://attach.52pojie.cn/forum/202608/03/211755c27hr87787cwmwxe.png)   
图 2　传入 "data" 后，直接拼出 flutter_assets / icudtl.dat / app.so

```
explicit DartProject(const std::wstring& path) {
  assets_path_ = path + L"\\flutter_assets";
  icu_data_path_ = path + L"\\icudtl.dat";
  aot_library_path_ = path + L"\\app.so";
}

```

也就是说，传进去的 "data" 会立刻变成：

成员                相对路径                用途  
assets_path_      data\flutter_assets 资源、字体、AssetManifest 等  
icu_data_path_    data\icudtl.dat     ICU 数据，国际化 / 文本相关  
aot_library_path_ data\app.so         Dart AOT 快照本体        

注意：这里还只是「相对路径字符串」，还没映射文件，更没有进 Dart VM。  

后面 FlutterProjectBundle 才会拿 exe 所在目录，把它们补成绝对路径。

* * *

3.  window.Create → FlutterWindow::OnCreate

模板源码：

[https://github.com/flutter/flutter/blob/master/packages/flutter_tools/templates/app/windows.tmpl/runner/flutter_window.cpp](https://github.com/flutter/flutter/blob/master/packages/flutter_tools/templates/app/windows.tmpl/runner/flutter_window.cpp)

 ![](https://attach.52pojie.cn/forum/202608/03/211757w56r99w529pe9sy2.png)   
图 3　窗口创建成功后，马上 new 一个 FlutterViewController

```
bool FlutterWindow::OnCreate() {
  if (!Win32Window::OnCreate()) {
    return false;
  }

  RECT frame = GetClientArea();

  flutter_controller_ = std::make_unique<flutter::FlutterViewController>(
      frame.right - frame.left, frame.bottom - frame.top, project_);

  if (!flutter_controller_->engine() || !flutter_controller_->view()) {
    return false;
  }
  RegisterPlugins(flutter_controller_->engine());
  SetChildContent(flutter_controller_->view()->GetNativeWindow());

  flutter_controller_->engine()->SetNextFrameCallback([&]() {
    this->Show();
  });

  flutter_controller_->ForceRedraw();
  return true;
}

```

这里的关键动作只有一句：

```
flutter_controller_ = std::make_unique<flutter::FlutterViewController>(...);

```

窗口本身不负责加载 app.so。  

它只是在 OnCreate 时把 project_ 交给 FlutterViewController，后面的引擎初始化、Run、加载 AOT，都从这里开始往下掉。

* * *

4.  FlutterViewController：创建 Engine，再创建 View

源码：

[https://github.com/flutter-team-archive/engine/blob/main/shell/platform/windows/client_wrapper/flutter_view_controller.cc](https://github.com/flutter-team-archive/engine/blob/main/shell/platform/windows/client_wrapper/flutter_view_controller.cc)

 ![](https://attach.52pojie.cn/forum/202608/03/211759c550r7shh0450i3k.png) 

图 4　先 FlutterEngine(project)，再 FlutterDesktopViewControllerCreate

```
FlutterViewController::FlutterViewController(int width,
                                             int height,
                                             const DartProject& project) {
  engine_ = std::make_shared<FlutterEngine>(project);
  controller_ = FlutterDesktopViewControllerCreate(width, height,
                                                   engine_->RelinquishEngine());
  if (!controller_) {
    std::cerr << "Failed to create view controller." << std::endl;
    return;
  }
  view_ = std::make_unique<FlutterView>(
      FlutterDesktopViewControllerGetView(controller_));
}

```

顺序很重要：

1.  先造 Engine：把 assets / icu / app.so 路径塞进引擎属性
2.  再造 ViewController：把引擎和窗口视图绑在一起，并在合适时机 Run()

可以把它理解成：

*   Engine = 脑子（Dart VM + AOT）
*   View = 脸（HWND / 渲染表面）
*   ViewController = 把脑子和脸接起来的中间层

* * *

5.  FlutterEngine：组装属性，准备跨进 dll

源码：

[https://github.com/flutter/engine/blob/main/shell/platform/windows/client_wrapper/flutter_engine.cc](https://github.com/flutter/engine/blob/main/shell/platform/windows/client_wrapper/flutter_engine.cc)

 ![](https://attach.52pojie.cn/forum/202608/03/211801hg1ctutphhojg6o1.png) 

图 5　把 project 里的路径填进 C 结构体，然后调用 FlutterDesktopEngineCreate

```
FlutterEngine::FlutterEngine(const DartProject& project) {
  FlutterDesktopEngineProperties c_engine_properties = {};
  c_engine_properties.assets_path = project.assets_path().c_str();
  c_engine_properties.icu_data_path = project.icu_data_path().c_str();
  c_engine_properties.aot_library_path = project.aot_library_path().c_str();
  c_engine_properties.dart_entrypoint = project.dart_entrypoint().c_str();

  // ... 处理 dart_entrypoint_argc / argv ...

  engine_ = FlutterDesktopEngineCreate(&c_engine_properties);

  auto core_messenger = FlutterDesktopEngineGetMessenger(engine_);
  messenger_ = std::make_unique<BinaryMessengerImpl>(core_messenger);
}

```

到这里为止，代码大多还在 exe 侧的 client_wrapper。

真正的分水岭是：

```
engine_ = FlutterDesktopEngineCreate(&c_engine_properties);

```

这个符号来自 flutter_windows.dll。  

对 PE 来说，就是 IAT 里的一次跨模块调用：exe 把路径信息递过去，dll 接手后面的事。

* * *

6.  进入 flutter_windows.dll：FlutterDesktopEngineCreate

源码：

[https://github.com/flutter/engine/blob/main/shell/platform/windows/flutter_windows.cc](https://github.com/flutter/engine/blob/main/shell/platform/windows/flutter_windows.cc)

 ![](https://attach.52pojie.cn/forum/202608/03/211804xhekl4kmfkyfyhff.png)   
图 6　exe → dll 的分界点

```
FlutterDesktopEngineRef FlutterDesktopEngineCreate(
    const FlutterDesktopEngineProperties* engine_properties) {
  flutter::FlutterProjectBundle project(*engine_properties);
  auto engine = std::make_unique<flutter::FlutterWindowsEngine>(project);
  return HandleForEngine(engine.release());
}

```

两件事：

1.  FlutterProjectBundle(props)：整理 / 解析路径
2.  new FlutterWindowsEngine(project)：创建真正的 Windows 嵌入层引擎对象

此时 app.so 还没加载，只是「引擎对象已经在 dll 里活下来了」。

* * *

7.  FlutterProjectBundle：相对路径变绝对路径

源码：

[https://github.com/flutter/engine/blob/main/shell/platform/windows/flutter_project_bundle.cc](https://github.com/flutter/engine/blob/main/shell/platform/windows/flutter_project_bundle.cc)

 ![](https://attach.52pojie.cn/forum/202608/03/211740le9ui75igu4el4dl.png)   
图 7　data\app.so 在这里变成 C:...\YourApp\data\app.so

```
FlutterProjectBundle::FlutterProjectBundle(
    const FlutterDesktopEngineProperties& properties)
    : assets_path_(properties.assets_path),
      icu_path_(properties.icu_data_path) {
  if (properties.aot_library_path != nullptr) {
    aot_library_path_ = std::filesystem::path(properties.aot_library_path);
  }

  // ... entrypoint 参数处理 ...

  // Resolve any relative paths.
  if (assets_path_.is_relative() || icu_path_.is_relative() ||
      (!aot_library_path_.empty() && aot_library_path_.is_relative())) {
    std::filesystem::path executable_location = GetExecutableDirectory();
    if (executable_location.empty()) {
      FML_LOG(ERROR)
          << "Unable to find executable location to resolve resource paths.";
      assets_path_ = std::filesystem::path();
      icu_path_ = std::filesystem::path();
    } else {
      assets_path_ = std::filesystem::path(executable_location) / assets_path_;
      icu_path_ = std::filesystem::path(executable_location) / icu_path_;
      if (!aot_library_path_.empty()) {
        aot_library_path_ =
            std::filesystem::path(executable_location) / aot_library_path_;
      }
    }
  }
}

```

properties.aot_library_path 就是外面一路传下来的 data\app.so。

这里做的事情非常朴素：

```
exe 目录 + data\app.so  →  绝对路径

```

后面所有「打开文件 / 映射文件」都以这个绝对路径为准。  

所以你在 x64dbg / Process Hacker 里看模块 / 映射区时，看到的也通常是带完整路径的那个 app.so。

* * *

8.  FlutterDesktopViewControllerCreate：准备 Run

源码同在 flutter_windows.cc：

 ![](https://attach.52pojie.cn/forum/202608/03/211742ib6rkcrkfz1wqr7f.png)   
图 8　对外 API 很薄，真正逻辑在 CreateViewController

```
FlutterDesktopViewControllerRef FlutterDesktopViewControllerCreate(
    int width,
    int height,
    FlutterDesktopEngineRef engine) {
  return CreateViewController(engine, width, height, /*owns_engine=*/true);
}

```

继续看同文件里的 CreateViewController：

 ![](https://attach.52pojie.cn/forum/202608/03/211744r44445f3wf4cz5q6.png)   
图 9　红圈处：controller->engine()->Run()，引擎真正启动

```
static FlutterDesktopViewControllerRef CreateViewController(
    FlutterDesktopEngineRef engine_ref,
    int width,
    int height,
    bool owns_engine) {
  flutter::FlutterWindowsEngine* engine_ptr = EngineFromHandle(engine_ref);

  std::unique_ptr<flutter::WindowBindingHandler> window_wrapper =
      std::make_unique<flutter::FlutterWindow>(
          width, height, engine_ptr->windows_proc_table());

  std::unique_ptr<flutter::FlutterWindowsEngine> engine;
  if (owns_engine) {
    engine = std::unique_ptr<flutter::FlutterWindowsEngine>(engine_ptr);
  }

  std::unique_ptr<flutter::FlutterWindowsView> view =
      engine_ptr->CreateView(std::move(window_wrapper));
  if (!view) {
    return nullptr;
  }

  auto controller = std::make_unique<flutter::FlutterWindowsViewController>(
      std::move(engine), std::move(view));

  // Launch the engine if it is not running already.
  if (!controller->engine()->running()) {
    if (!controller->engine()->Run()) {
      return nullptr;
    }
  }

  controller->view()->SendInitialBounds();
  controller->engine()->UpdateAccessibilityFeatures();

  return HandleForViewController(controller.release());
}

```

重点就这一句：

```
controller->engine()->Run();

```

前面那么长的链路，本质都是在给 Run() 准备参数和对象。  

真正开始「加载 AOT、拉起 Dart」是从这里起步的。

* * *

9.  FlutterWindowsEngine::Run → LoadAotData

源码：

[https://github.com/flutter/engine/blob/main/shell/platform/windows/flutter_windows_engine.cc](https://github.com/flutter/engine/blob/main/shell/platform/windows/flutter_windows_engine.cc)

Run() 本身比较长，这里不整段粘贴。对加载 app.so 来说，只要盯住这一句：

```
aot_data_ = project_->LoadAotData(embedder_api_);

```

 ![](https://attach.52pojie.cn/forum/202608/03/211746ng6udz5bmqx9vqb4.png)   
图 10　引擎 Run 时，会让 project 去加载 AOT 数据

含义很直白：

*   project_ 知道 aot_library_path_（也就是绝对路径版 app.so）
*   embedder_api_ 提供 CreateAOTData 这类函数指针
*   LoadAotData 负责把「路径」变成「可交给 Dart VM 的 AOT 数据结构」

* * *

10.  LoadAotData：声明「我要加载的是 ELF 路径」

源码仍在：

[https://github.com/flutter/engine/blob/main/shell/platform/windows/flutter_project_bundle.cc](https://github.com/flutter/engine/blob/main/shell/platform/windows/flutter_project_bundle.cc)

 ![](https://attach.52pojie.cn/forum/202608/03/211748zf1r4f8mwzakk118.png) 

图 11　红圈：明确告诉引擎，AOT 源是 ELF 文件路径

```
UniqueAotDataPtr FlutterProjectBundle::LoadAotData(
    const FlutterEngineProcTable& engine_procs) {
  if (aot_library_path_.empty()) {
    FML_LOG(ERROR)
        << "Attempted to load AOT data, but no aot_library_path was provided.";
    return UniqueAotDataPtr(nullptr, nullptr);
  }
  if (!std::filesystem::exists(aot_library_path_)) {
    FML_LOG(ERROR) << "Can't load AOT data from "
                   << aot_library_path_.u8string() << "; no such file.";
    return UniqueAotDataPtr(nullptr, nullptr);
  }

  std::string path_string = aot_library_path_.u8string();
  FlutterEngineAOTDataSource source = {};
  source.type = kFlutterEngineAOTDataSourceTypeElfPath;
  source.elf_path = path_string.c_str();

  FlutterEngineAOTData data = nullptr;
  auto result = engine_procs.CreateAOTData(&source, &data);
  if (result != kSuccess) {
    FML_LOG(ERROR) << "Failed to load AOT data from: " << path_string;
    return UniqueAotDataPtr(nullptr, nullptr);
  }

  return UniqueAotDataPtr(data, engine_procs.CollectAOTData);
}

```

这里有两个很值的点：

1.  先检查文件在不在  
    所以你把 data\app.so 删了 / 改名了，程序往往直接起不来，日志也会很直白。
2.  source.type 写死是 ElfPath  
    说明 Flutter 从一开始就没打算把它当 Windows DLL 处理，而是明确按 ELF 文件 去读。

接下来就进入 embedder 层的 FlutterEngineCreateAOTData。

* * *

11.  FlutterEngineCreateAOTData → Dart_LoadELF

源码：

[https://github.com/flutter/engine/blob/main/shell/platform/embedder/embedder.cc](https://github.com/flutter/engine/blob/main/shell/platform/embedder/embedder.cc)

 ![](https://attach.52pojie.cn/forum/202608/03/211750wf9hmletz471h9kb.png)   
图 12　红圈：Windows 上真正打开 app.so 的关键调用

```
FlutterEngineResult FlutterEngineCreateAOTData(
    const FlutterEngineAOTDataSource* source,
    FlutterEngineAOTData* data_out) {
  if (!flutter::DartVM::IsRunningPrecompiledCode()) {
    return LOG_EMBEDDER_ERROR(kInvalidArguments,
                              "AOT data can only be created in AOT mode.");
  } else if (!source) {
    return LOG_EMBEDDER_ERROR(kInvalidArguments, "Null source specified.");
  } else if (!data_out) {
    return LOG_EMBEDDER_ERROR(kInvalidArguments, "Null data_out specified.");
  }

  switch (source->type) {
    case kFlutterEngineAOTDataSourceTypeElfPath: {
      if (!source->elf_path || !fml::IsFile(source->elf_path)) {
        return LOG_EMBEDDER_ERROR(kInvalidArguments,
                                  "Invalid ELF path specified.");
      }

      auto aot_data = std::make_unique<_FlutterEngineAOTData>();
      const char* error = nullptr;

#if OS_FUCHSIA
      Dart_LoadedElf* loaded_elf = nullptr;
#else
      Dart_LoadedElf* loaded_elf = Dart_LoadELF(
          source->elf_path,               // file path
          0,                              // file offset
          &error,                         // error (out)
          &aot_data->vm_snapshot_data,    // vm snapshot data (out)
          &aot_data->vm_snapshot_instrs,  // vm snapshot instr (out)
          &aot_data->vm_isolate_data,     // vm isolate data (out)
          &aot_data->vm_isolate_instrs    // vm isolate instr (out)
      );
#endif

      if (loaded_elf == nullptr) {
        return LOG_EMBEDDER_ERROR(kInvalidArguments, error);
      }

      aot_data->loaded_elf.reset(loaded_elf);
      *data_out = aot_data.release();
      return kSuccess;
    }
  }

  return LOG_EMBEDDER_ERROR(
      kInvalidArguments,
      "Invalid FlutterEngineAOTDataSourceType type specified.");
}

```

到这里，谜底基本揭开了：

11.1 为什么 Windows 上也叫 app.so？

因为 Flutter / Dart 的 AOT 产物，跨平台统一用了 ELF 容器 来打包 snapshot。  

名字带 .so，不代表它要走 Linux 动态链接那一套，更不代表 Windows 会当 DLL 加载。

11.2 为什么不是 LoadLibrary？

因为 LoadLibrary 只认 PE。  

app.so 是 ELF，系统加载器直接拒绝。Flutter 的选择是：

自己解析 ELF，自己映射段，自己把 snapshot 指针交回给 Dart VM。

实现入口就在 Dart SDK：

[https://github.com/dart-lang/sdk/blob/main/runtime/bin/elf_loader.cc](https://github.com/dart-lang/sdk/blob/main/runtime/bin/elf_loader.cc)

在 Windows 上，这套自研 loader 最终会落到文件映射一类 API（你在概览里写的 CreateFileMapping / MapViewOfFile），而不是 LoadLibrary。

11.3 Dart_LoadELF 吐出来的四块是什么？

输出                    大致含义  
vm_snapshot_data      VM 快照数据（只读数据为主）  
vm_snapshot_instrs    VM 快照指令  
vm_isolate_data       Isolate 快照数据（应用侧对象 / 元数据更多在这边）  
vm_isolate_instrs     Isolate 快照指令（应用 AOT 代码主要在这边）

可以粗暴理解成：

*   前两块更偏 Dart VM 自己
*   后两块更偏 你的 App 逻辑

所以后面做符号恢复、池分析、看伪代码，很多时候真正关心的就是从这份 AOT 快照里挖出来的东西。

* * *

12.  串起来：一句话复盘

把整条链压缩成人话：

1.  wWinMain 里用 DartProject("data") 声明资源目录  
2.  DartProject 直接拼出 data\app.so  
3.  FlutterWindow::OnCreate 创建 FlutterViewController  
4.  FlutterEngine 组装属性，经 IAT 跨进 flutter_windows.dll  
5.  FlutterProjectBundle 把相对路径补成绝对路径  
6.  ViewController 创建时调用 engine->Run()  
7.  Run → LoadAotData → FlutterEngineCreateAOTData  
8.  最终 Dart_LoadELF(app.so)：按 ELF 映射文件，抽出 4 块 snapshot  
9.  Dart VM 拿到预编译快照，AOT 代码开始跑

所以回到最初那个问题：

Windows Flutter 为什么能加载一个不符合 Windows 格式的 app.so？

因为它根本没让 Windows 去「加载模块」，而是让 Dart 自己把这个 ELF 快照文件 映射进进程、解析出 snapshot、喂给 VM。

* * *

13.  对逆向有什么直接帮助

如果你后面要跟 Windows Flutter：

1.  先确认目录特征  
    data\app.so + flutter_windows.dll + flutter_assets，基本就能定性。
2.  别指望 IDA 直接当普通 DLL 看懂 app.so  
    它不是常规导入导出那套语义，核心在 Dart AOT。
3.  动态看加载点  
    关注 FlutterEngineCreateAOTData / Dart_LoadELF 前后，进程里会出现 app.so 的映射区。
4.  静态跟链路时记住分界  
    exe 侧 client_wrapper 只是壳；真正加载发生在 flutter_windows.dll + Dart ELF loader。
5.  符号问题另说  
    映射进来不等于可读。要把函数名 / 对象池弄出来，还得靠 Blutter 一类基于同版本 Dart VM 的导出方案（Windows 适配是另一条线）。

* * *

参考链接

*   runner 入口模板：  
    main.cpp.tmpl
*   DartProject：  
    dart_project.h
*   FlutterWindow::OnCreate：  
    flutter_window.cpp
*   FlutterViewController：  
    flutter_view_controller.cc
*   FlutterEngine：  
    flutter_engine.cc
*   Windows embedder API：  
    flutter_windows.cc
*   FlutterProjectBundle / LoadAotData：  
    flutter_project_bundle.cc
*   FlutterWindowsEngine::Run：  
    flutter_windows_engine.cc
*   FlutterEngineCreateAOTData：  
    embedder.cc
*   Dart_LoadELF：  
    elf_loader.cc

* * *

— 完 —

学习研究用途。文中示例程序名仅作示意，请支持正版软件。

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)dhhddhf Flutter 写的软件优点是跨平台，缺点就是体积大，写过个安卓软件用 Flutter 写生成的 apk 比 java 或 kotlin 写的大接近十倍，问过 ai 说体积大是 Flutter 软件都内置了个渲染引擎和虚拟机 ![](https://avatar.52pojie.cn/data/avatar/002/09/05/26_avatar_middle.jpg) wolfSpicy

> [dhhddhf 发表于 2026-8-4 00:00](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=55669688&ptid=2121108)  
> Flutter 写的软件优点是跨平台，缺点就是体积大，写过个安卓软件用 Flutter 写生成的 apk 比 java 或 kotlin 写的大 ...

是的 并且内置的虚拟机还只是塞进了 AOT 虚拟机还不包括 JIT  这类塞入虚拟机的好处就像涛之雨大佬说的 可以任意跨平台环境下使用![](https://avatar.52pojie.cn/images/noavatar_middle.gif)祈无路过人间 学习一下！![](https://avatar.52pojie.cn/images/noavatar_middle.gif)wangleizlj 学习一下![](https://avatar.52pojie.cn/data/avatar/000/87/90/80_avatar_middle.jpg)涛之雨

> [dhhddhf 发表于 2026-8-4 00:00](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=55669688&ptid=2121108)  
> Flutter 写的软件优点是跨平台，缺点就是体积大，写过个安卓软件用 Flutter 写生成的 apk 比 java 或 kotlin 写的大 ...

所以他跨平台啊他自己实现了一个虚拟机，这样在任意网页（有预处理），ios，Android，Windows 系统 / 环境都可以使用 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) fzlte0 分析深入浅出，学习了。![](https://avatar.52pojie.cn/data/avatar/000/06/83/64_avatar_middle.jpg)bioman 好高深莫测，看明白了也不会动手 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) ylgh678 有点难研究研究 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) 75233a20260721 感谢分享
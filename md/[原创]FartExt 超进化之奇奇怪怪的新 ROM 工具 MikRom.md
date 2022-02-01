> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-271358.htm)

> [原创]FartExt 超进化之奇奇怪怪的新 ROM 工具 MikRom

### 前言

​ 一眨眼就到春节了，好久没有写文章了，趁着新年空闲，赶紧把自己折腾了一段时间的东西整理了一下。在折腾的过程中，碰到了不少问题，感谢大佬们的帮助。目前这个工具不算是怎么完善吧，但是感觉能凑合使用了，剩下的部分在使用中再慢慢完善吧，其中部分代码我会开源，其实感觉实现的核心并不怎么复杂。算是一个萌新学习定制 ROM 过程的一个作品把。而且还有个调试超级慢的 BUG，如果有大佬知道是啥原因，还请指导一下。万分感谢。

### 感谢

> **看雪高研班课程**
> 
> **FART 脱壳王课程**
> 
> **珍惜大佬的 android 进阶课程**

### FartExt

​ 详细的介绍请看：[FartExt 之优化更深主动调用的 FART10](https://bbs.pediy.com/thread-268760.htm)

 

​ FartExt 是我之前学习脱壳实践时做的一个自动脱壳机，是基于 FART 的主动调用思想实现对特定的抽取壳进行优化处理的工具。由于原本的 FART 没有配置相关的，所以我增加了配置对指定 app 脱壳。大致就是对 FART 的简单优化。由于感觉当时做的功能并不怎么完善，所以只是短暂的放了下载地址，就删掉了。不知道有没有人实际使用。使用的效果到底咋样。现在放出开源代码。

 

github：[FartExt](https://github.com/dqzg12300/FartExt)

### MikRom

​ 首先回顾一下当时 FartExt 文章最后的思考部分

> 整个流程梳理完成后，我们可以由此来借鉴来思考延伸一下。
> 
> 比如，包装一些属于自己的系统层 api 调用。便于我们使用 xposed 或者是 frida 来调用一些功能。
> 
> 再比如，加载应用时，读取配置文件作为开关，我们来对网络流量进行拦截写入保存，或者对所有的 jni 函数调用，或者是 java 函数调用进行 trace。这种就属于是 rom 级别的打桩。
> 
> 再比如，可以做一个应用来读写作为开关的配置文件，而 rom 读取配置文件后，对一些流程进行调整。例如控制 FART 是否使用更深调用。控制是否开启 rom 级别的打桩。
> 
> 以上纯属个人瞎想。刚刚入门，想的有点多，以后了解更深了，我再看看如何定制一个专属的 rom 逆向集合

 

​ MikRom 就是当时个人瞎想的成果，做一个 Rom 层面的逆向工具，为我们提供比较常用的插桩、注入、脱壳等一系列功能。下面列上目前 MikRom 中包含的功能

> *   内核修改过反调试
> *   开启硬件断点
> *   USB 调试默认连接
> *   脱壳（黑名单、白名单过滤、更深的主动调用链）
> *   ROM 打桩（ArtMethod 调用、RegisterNative 调用、JNI 函数调用）
> *   frida 持久化（支持 listen,wait,script 三种模式）
> *   支持自行切换 frida-gadget 版本
> *   反调试（通过 sleep 目标函数，再附加进程来过掉起始的反调试）
> *   trace java 函数（smali 指令的 trace）
> *   内置 dobby 注入
> *   注入 so
> *   注入 dex（实现对应的接口触发调用。目前还未测试）

 

github：[MikRom](https://github.com/dqzg12300/MikRom)

### MikManager

​ 由于功能做的更加复杂了，我们自行编辑配置文件不再那么方便了，所以我特地做了一个界面化的工具来操作。最后将我们的设置保存到文件，然后 MikRom 会在打开 app 时读取文件，解析后做对应的操作。MikManager 就是用来管理这个配置的。界面较为简陋，如果对 MikRom 感兴趣的，但是感觉我的界面太丑，也可以自己编写一个界面管理工具。我正向开发比较渣，所以代码较为粗糙。不过目前使用没啥问题。

 

github：[MikManager](https://github.com/dqzg12300/MikManager)

### 开发的始末

​ 虽然功能不是很多，但是做这个工具却折腾了我很长的时间，这里我将记录下这样一个简陋的 Rom 工具开发的历程，由于并不是很清楚其他大佬是如何处理的，所以有些功能都是我不断的试错中找到的方案。在试错的过程中，走了不少弯路，希望能帮到一些小伙伴。如果我采用的方案犯了什么比较低级的错误，还请大佬能指点一下。另外有大佬说，做越简单化的工具，越危险。如果有什么和谐的风险或者是法律的问题，还请联系我进行修改。

### 编译版本

​ 早先 FartExt 我采用的是 aosp10r2 的源码进行修改编译。后来有同学觉得界面太简陋了，看起来就像山寨系统。确实如此，然后我就参考了 hanbingle 老师在 Fart 脱壳王使用的 rom，选择了 PixelExperience 来进行修改。编译的 marlin 版本，测试手机是 pixel XL。

> 参考文章：https://blog.csdn.net/weixin_42443075/article/details/118084535

 

​ 上面是我当时编译参考的文章。另外在编译 PixelExperience 时编译碰到错误，需要修改`build/blueprint/Blueprints`

```
bootstrap_go_package {
    name: "blueprint-pathtools",
    pkgPath: "github.com/google/blueprint/pathtools",
    deps: [
        "blueprint-deptools",
    ],
    srcs: [
        "pathtools/lists.go",
        "pathtools/fs.go",
        "pathtools/glob.go",
    ],
    testSrcs: [
        //修改处，这里的内容删掉
    ],
}

```

​ 另外一个问题就是这个 rom 居然没法在 mac 进行编译，最后没办法只能用 ubuntu 来开发了。

### 配置管理优化

​ 最早先在 FartExt 中的配置是我们自己在 / data/local/tmp / 中写入一个 fext.config 的文件，然后在应用启动过程中的 handleBindApplication 调用时，解析这个配置文件，来决定是否要脱壳。而现在配置更加复杂了，我们需要使用一个 app 来对配置进行管理生成，而 app 没有对 / data/local/tmp 写入文件的权限。所以我们这里就有了一个简单的需求，要将配置文件落地到一个所有应用都有权限访问的地方。

 

​ 有人就要说了，我们可以落地到 sdcard，是的，早期我就是这么干的。配置文件落地到 sdcard 后，所有的 app 要使用功能，就必须先打开 sdcard，并且，由于我使用的 rom 版本是安卓 10 的，而安卓 10 中，你想要直接访问 sdcard 的任意目录，是需要设置`requestLegacyExternalStorage="true"`。所以这就导致了，即使我们不嫌麻烦，每个想处理的 app 都打开 sdcard，也会出现有些 app 无法访问到配置文件。

 

​ 在这里我使用的方案是，创建一个系统服务，这个系统服务提供一个读和写的函数，然后通过调用系统服务将配置文件落地到`/data/system/`目录中，然后每个应用打开时再通过这个系统服务来读取配置。

### 添加系统服务

​ 这个系统服务目前主要就是为配置管理提供读写权限。可能这样干会有些漏洞的问题，不过我这个 rom 本身就是逆向使用的工具，而面向正常用户，所以就暂时不考虑漏洞问题了。

> 参考文章：[Android AOSP 添加系统服务【aidl 接口服务】Java 层](https://bbs.pediy.com/thread-260472.htm)
> 
> 参考文章：[android 10 添加系统服务步骤](https://blog.csdn.net/a546036242/article/details/118221349)

 

​ 下面贴上我定义的系统服务

```
interface IMikRom
{
    //读取文件
    String readFile(String path);
    //写入文件
    void writeFile(String path,String data);
    //执行shell命令
    String shellExec(String cmd);
}

```

​ 至于详细的读写方法我就不贴了，就是和 android 正常的读写文件处理一样。我就说一下在这里我碰到的坑，我按照参考文章一样的方式做完之后，发现无法找到服务，但是`service list|grep mikrom`是可以匹配到我自己定义的服务的。最后经过排查日志发现 selinux 提示缺少了 find 权限，于是我修改文件`system/sepolicy/public/untrusted_app.te`，内容如下

```
type untrusted_app, domain;
type untrusted_app_27, domain;
type untrusted_app_25, domain;
allow untrusted_app mikrom_service:service_manager find;
allow untrusted_app_27 mikrom_service:service_manager find;
allow untrusted_app_25 mikrom_service:service_manager find;

```

最后成功找到服务，将配置文件写入到了`/data/system/`目录中。

 

这里我只简单的贴一下相关

 

MikManager 的写入的相关代码

```
//获取服务
public class ServiceUtils {
    private static IMikRom iMikRom = null;
    public static IMikRom getiMikRom() {
        if (iMikRom == null) {
            try {
                Class localClass = Class.forName("android.os.ServiceManager");
                Method getService = localClass.getMethod("getService", new Class[] {String.class});
                if(getService != null) {
                    Object objResult = getService.invoke(localClass, new Object[]{"mikrom"});
                    if (objResult != null) {
                        IBinder binder = (IBinder) objResult;
                        iMikRom = IMikRom.Stub.asInterface(binder);
                    }
                }
            } catch (Exception e) {
                Log.d("MikManager",e.getMessage());
                e.printStackTrace();
            }
        }
        return iMikRom;
    }
}
//将json数据保存到指定路径
public static void SaveMikromConfig(List packageList){
    Log.e(ConfigUtil.TAG,"SaveMikromConfig");
    Gson gson = new Gson();
    String savejson=gson.toJson(packageList);
    try {
        ServiceUtils.getiMikRom().writeFile(ConfigUtil.configPath,savejson);
    } catch (RemoteException e) {
        Log.e(ConfigUtil.TAG,"writeConfig err:"+e.getMessage());
    }
} 
```

MikRom 中读取数据

```
private static IMikRom iMikRom=null;
public static IMikRom getiMikRom() {
    if (iMikRom == null) {
        try {
            Class localClass = Class.forName("android.os.ServiceManager");
            Method getService = localClass.getMethod("getService", new Class[] {String.class});
            if(getService != null) {
                Object objResult = getService.invoke(localClass, new Object[]{"mikrom"});
                if (objResult != null) {
                    IBinder binder = (IBinder) objResult;
                    iMikRom = IMikRom.Stub.asInterface(binder);
                }
            }
        } catch (Exception e) {
            Log.d("MikManager",e.getMessage());
            e.printStackTrace();
        }
    }
    return iMikRom;
}
 
public static String getMikConfig(){
    try {
        IMikRom mikrom=getiMikRom();
        if(mikrom==null){
            return "";
        }
        return mikrom.readFile("/data/system/mik.conf");
    } catch (RemoteException e) {
        e.printStackTrace();
    }
    return "";
}

```

测试的配置文件内容如下

```
[{"appName":"crackme","breakClass":"","dexClassName":"","dexPath":"","enabled":true,"fridaJsPath":"","gadgetArm64Path":"","gadgetPath":"","isDeep":false,"isDobby":false,"isInvokePrint":false,"isJNIMethodPrint":true,"isRegisterNativePrint":false,"isTuoke":false,"packageName":"com.kanxue.crackme","port":0,"sleepNativeMethod":"","soPath":"","traceMethod":"","whiteClass":"","whitePath":""}]

```

到这里第一步的优化就完成了，读写配置成功脱离了对 sdcard 的权限依赖，以及能全局应用都可以访问到。具体的详细相关代码可参考我放出的部分源码。

### 脱壳相关优化

​ 由于我走向了另外一条偏向更易于使用的优化方向，所以关于脱壳的关键部分我并没有做什么优化，所以如果脱壳需求较大的朋友可以考虑看看 hanbingle 大佬的脱壳王是否能满足需求。

 

​ 我对脱壳的优化主要分为两点，由于 FartExt 当时的脱壳结果保存是在 sdcard 中，所以对于权限有一定依赖，所以我优化掉了这块的依赖，让我们不用再手动开启 sdcard 权限，也能保存下脱壳结果。另外一点就是当时如果是抽取壳的情况，我们需要拿出两个文件来手动修复一下。我这里也优化了一下，会直接将函数执行完的 dex 重新再保存一份，这样就无需我们再手动修复了。当然同时也保留了原本的做法，仍然保存大佬说的几个重要元素。

#### [](#优化1方案：解决sdcard权限问题)优化 1 方案：解决 sdcard 权限问题

​ 我尝试了很多种办法来避免我们手动开启 sdcard 权限的情况下保存脱壳结果文件。最后测试成功了两种办法。这两种办法我都简单说下。

 

​ 第一种办法，我们直接将数据可以写入应用本身的内部空间中。也就是`/data/data/<PackageName>`中。但是这样有权限写入，但是没有 root 的话就无法访问到保存的结果了，当然没有 root 的情况也是能访问到应用内部数据的。我们可以通过命令`run-as <PackageName>`来直接进入应用数据内部。但是也有人会问，如果对方应用没有开启 debuggable，不就没办法使用 run-as 了吗？这就是改 Rom 的优势所在了。我们可以直接修改 PackageParser 中的函数 parseBaseApplication。在里面为我们默认打开 debuggable 即可。这样即使对方没有设置，在加载 xml 的时候，也会打开这个功能。

 

​ 第二种办法，既然第一个办法最后可以直接改 xml 为我们默认打开 debuggable，那么我们解决 sdcard 权限，也可以修改 PackageParser 中的函数 parseBaseApplication 来直接打开 sdcard 权限。不过由于安卓 10 的特殊性，即使打开了 sdcard 权限，也只能在 sdcard 中自己的目录写入数据，所以使用这个办法时，脱壳结果应保存在`/sdcard/Android/data/<PackageName>/`目录中。

 

​ 下面贴上相关的代码，其中包含了两种解决方案

```
//这里我判断如果非系统应用才增加这些权限
if((ai.flags&ApplicationInfo.FLAG_SYSTEM)!=1){
        //试错中发现，开启这些权限的时候，有个google和message相关的进程会崩溃，所以过滤一下
        if(!ai.packageName.contains("google") && !ai.packageName.contains("message")){
            //下面是sdcard权限开启，以及debuggable权限开启。如果担心检测，可以关掉debuggable的默认开启
            ai.privateFlags |= ApplicationInfo.PRIVATE_FLAG_REQUEST_LEGACY_EXTERNAL_STORAGE;
 
            ai.flags |= ApplicationInfo.FLAG_EXTERNAL_STORAGE;
 
            ai.flags |= ApplicationInfo.FLAG_DEBUGGABLE;
            // Debuggable implies profileable
            ai.privateFlags |= ApplicationInfo.PRIVATE_FLAG_PROFILEABLE_BY_SHELL;
        }
    }

```

#### [](#优化2方案：解决抽取壳在脱壳完成后还需要手动修复)优化 2 方案：解决抽取壳在脱壳完成后还需要手动修复

​ 在优化这个问题前，首先要意识到为什么会需要手动修复，当我们理解了大佬的处理之后，就发现 hanbingle 大佬为了避免每个函数主动调用都将 dex 给保存，所以只有文件不存在的时候才保存。也就意味着我们保存的 dex 是第一个主动调用执行时的 dex。如果这个抽取壳是必须函数执行后才会恢复的，那么后面的函数在这个保存 dex 中都依然是被抽取的。FART 的做法是将 codeitem 保存出来后，然后再修复。所以我将这里优化了一下。

 

​ 知道问题所在后，优化的思路就清晰了，我采用了比较简单的一种优化方式，就是每个 dex 文件保存时，将这个 dex 的地址以及长度给保存下来。最后在所有主动调用完成时，重新将所有 dex 文件再保存一次。下面看看优化后的相关代码

```
//存放dex的指针和长度
static std::map dex_map;
 
//主动调用函数的dump处理
extern "C" void dumpArtMethod(ArtMethod* artmethod)  REQUIRES_SHARED(Locks::mutator_lock_) {
                ...
                int dexfilefp=open(dexfilepath,O_RDONLY,0666);
                ///dex文件存在则不处理,避免主动调用每次都要重新保存dex
                if(dexfilefp>0){
                    close(dexfilefp);
                    dexfilefp=0;
                }else{
                    LOG(ERROR) << "mikrom ArtMethod::dumpdexfilebyArtMethod save dex_map";
                    //将这个地址给保存下来
                    dex_map.insert(std::pair((void*)begin_,size_));
                    int fp=open(dexfilepath,O_CREAT|O_APPEND|O_RDWR,0666);
                    ...
                }
}
//主动调用完成时,重新保存到文件名_dexfile_repair.dex中
extern "C" void dumpDexOver()  REQUIRES_SHARED(Locks::mutator_lock_) {
    if(dex_map.size()<=0){
            LOG(ERROR) << "mikrom dumpDexOver dex_map.size()<=0";
        return;
    }
    char *dexfilepath=(char*)malloc(sizeof(char)*1000);
    LOG(ERROR) << "mikrom ArtMethod::dumpDexOver";
    int result=0;
    char* packageName=ArtMethod::GetPackageName();
    std::map::iterator iter;
    for(iter = dex_map.begin(); iter != dex_map.end(); iter++) {
        void* begin_=iter->first;
        size_t size_=iter->second;
        int size_int_=(int)size_;
        memset(dexfilepath,0,1000);
        sprintf(dexfilepath,"/sdcard/Android/data/%s/files/dump",packageName);
        mkdir(dexfilepath,0777);
        memset(dexfilepath,0,1000);
        sprintf(dexfilepath,"/sdcard/Android/data/%s/files/dump/%d_dexfile_repair.dex",packageName,size_int_);
        int dexfilefp=open(dexfilepath,O_RDONLY,0666);
        if(dexfilefp>0){
          close(dexfilefp);
          dexfilefp=0;
 
        }else{
          int fp=open(dexfilepath,O_CREAT|O_APPEND|O_RDWR,0666);
          if(fp>0)
          {
              result=write(fp,(void*)begin_,size_);
              if(result<0)
              {
                  LOG(ERROR) << "mikrom ArtMethod::dumpDexOver,open dexfilepath error";
              }
              fsync(fp);
              close(fp);
              memset(dexfilepath,0,1000);
          }
        }
    }
    if(dexfilepath!=nullptr)
    {
        free(dexfilepath);
        dexfilepath=nullptr;
    }
} 
```

最后生成的_repair.dex 文件就是无需再修复的了，如果想要手动修复按照原来的方式也可以。当然，如果主动调用未能成功跑完，这个 repair 文件也是不会生成的。

### 新增功能

#### [](#1、修改内核反调试)1、修改内核反调试

​ 关于修改内核反调试已经有相当多的文章讲解了，我这里也是和他们用的同一个方案，基本都是根据正向检测调试的方式，来进行反调试，当然如果有人用其他方式来判断是否被调试，这个反调试就无效了。详细的过反调试原理可以看参考文章。

> 参考文章：[修改内核源码绕过反调试检测（Android10）](https://blog.csdn.net/u011426115/article/details/113144592)

 

​ 虽然我改了。但是没测试效果咋样。修改的代码如下，`kernel/google/marlin/fs/proc/array.c`

```
static const char * const task_state_array[] = {
    "R (running)",        /*   0 */
    "S (sleeping)",        /*   1 */
    "D (disk sleep)",    /*   2 */
    "S (sleeping)",        /*   4 */         //这里之前是 "T (stopped)",
    "S (sleeping)",        /*   8 */         //这里之前是 "t (tracing stop)"
    "X (dead)",        /*  16 */
    "Z (zombie)",        /*  32 */
};
 
static inline void task_state(struct seq_file *m, struct pid_namespace *ns,
                struct pid *pid, struct task_struct *p)
{
    struct user_namespace *user_ns = seq_user_ns(m);
    struct group_info *group_info;
    int g;
    struct fdtable *fdt = NULL;
    const struct cred *cred;
    pid_t ppid = 0, tpid = 0;
    struct task_struct *leader = NULL;
 
    rcu_read_lock();
    if (pid_alive(p)) {
        struct task_struct *tracer = ptrace_parent(p);
        if (tracer)
            tpid = task_pid_nr_ns(tracer, ns);
        ppid = task_tgid_nr_ns(rcu_dereference(p->real_parent), ns);
        leader = p->group_leader;
    }
    //这个直接强制改tpid为0。
    tpid=0;
    cred = get_task_cred(p);
    ...
}

```

当对方检测`/proc/<pid>/status`文件时，获取到的 TracerPid 就会一直是 0，并且应用的状态不会出现`stopped`和`tracing stop`。下面是 status 的部分内容

```
Name:    .kanxue.crackme
State:    S (sleeping)
Tgid:    12142
Pid:    12142
PPid:    545
TracerPid:    0
Uid:    10027    10027    10027    10027
Gid:    10027    10027    10027    10027
Ngid:    0
FDSize:    128
Groups:    9997 20027 50027
VmPeak:     5657524 kB
VmSize:     5210040 kB

```

还有一处修改文件是`kernel/google/marlin/fs/proc/base.c`

```
static int proc_pid_wchan(struct seq_file *m, struct pid_namespace *ns,
              struct pid *pid, struct task_struct *task)
{
    unsigned long wchan;
    char symname[KSYM_NAME_LEN];
 
    wchan = get_wchan(task);
 
    if (lookup_symbol_name(wchan, symname) < 0)
        if (!ptrace_may_access(task, PTRACE_MODE_READ_FSCREDS))
            return 0;
        else
            return seq_printf(m, "%lu", wchan);
    else
        if(strstr(symname,"trace")){        //这里是新增的，如果符号名称包含trace，就固定改成sys_epoll_wait
            return seq_printf(m,"%s","SyS_epoll_wait");
        }
        return seq_printf(m, "%s", symname);
}

```

#### [](#2、sleep反调试)2、sleep 反调试

​ 通过 sleep 的反调试方式是在看雪高研班中学习到的，据说这个办法可以过掉一些在前期检测的 native 的反调试。思路就是需要调试的 JNI 函数执行前会调用 JniMethodStart，那么我们只要在这里进行判断，如果这个函数是目标函数，就 sleep 睡眠若干秒。在睡眠期间，我们再调试附加上即可。相关代码如下

```
extern uint32_t JniMethodStart(Thread* self) {
  JNIEnvExt* env = self->GetJniEnv();
  DCHECK(env != nullptr);
  uint32_t saved_local_ref_cookie = bit_cast(env->GetLocalRefCookie());
  env->SetLocalRefCookie(env->GetLocalsSegmentState());
  ArtMethod* native_method = *self->GetManagedStack()->GetTopQuickFrame();
  // TODO: Introduce special entrypoint for synchronized @FastNative methods?
  //       Or ban synchronized @FastNative outright to avoid the extra check here?
  DCHECK(!native_method->IsFastNative() || native_method->IsSynchronized());
  const char* methodname=native_method->PrettyMethod().c_str();
  if(ArtMethod::GetDebugMethod()!=nullptr && strlen(ArtMethod::GetDebugMethod())>0){
    if(strstr(methodname,ArtMethod::GetDebugMethod())!=nullptr){
                    std::ostringstream oss1;
            oss1<< "fartext JniMethodStart methodname:"<IsFastNative()) {
    // When not fast JNI we transition out of runnable.
    self->TransitionFromRunnableToSuspended(kNative);
  }
  return saved_local_ref_cookie;
} 
```

#### [](#3、rom级打桩)3、ROM 级打桩

​ 这个其实没什么难的，在关键的函数位置直接打印即可。我只讲一下 jni 的打桩部分，我查资料的时候看到了两种做法，第一种是打印 jni 部分是在 jni 函数中找比较通用的调用函数，例如`InvokeVirtualOrInterfaceWithJValues、InvokeWithVarArgs、InvokeWithJValues`，然后在里面加上打印。第二种就是像 jnitracer 一样，通过偏移，找到所有的函数指针，然后再包装处理，比如加一个外层函数调用。由于我 c++ 非常烂，所以我采用最笨的方法，我写了几个较为通用的打印函数，jni_internal.cc 中的所有我关心的 jni 函数，就调用各自对应的打印函数。下面贴上我的例子。

```
void ShowVarArgs(const ScopedObjectAccessAlreadyRunnable& soa,
                                           const char* jniMethod,
                                           jmethodID mid,
                                           va_list args){
     if(ArtMethod::IsJNIMethodPrint()){
                 ArtMethod* method = jni::DecodeArtMethod(mid);
               uint32_t shorty_len = 0;
         const char* shorty =
               method->GetInterfaceMethodIfProxy(kRuntimePointerSize)->GetShorty(&shorty_len);
         JValue result;
         ArgArray arg_array(shorty, shorty_len);
         arg_array.VarArgsShowArg(soa, args,jniMethod,method->PrettyMethod().c_str());
     }
}
void ShowJValue(const ScopedObjectAccessAlreadyRunnable& soa,
                                           const char* jniMethod,
                                           jmethodID mid,
                                           const jvalue* args){
    if(ArtMethod::IsJNIMethodPrint()){
                ArtMethod* method = jni::DecodeArtMethod(mid);
                uint32_t shorty_len = 0;
                const char* shorty =
                        method->GetInterfaceMethodIfProxy(kRuntimePointerSize)->GetShorty(&shorty_len);
                JValue result;
                ArgArray arg_array(shorty, shorty_len);
                arg_array.JValuesShowArg(soa, args,jniMethod,method->PrettyMethod().c_str());
    }
}
 
void ShowJniField(const char* jniMethodName,jfieldID field) {
        if(ArtMethod::IsJNIMethodPrint()){
            ArtField* f = jni::DecodeArtField(field);
            std::ostringstream oss;
            oss << "mikrom jni "<
```

处理完判断部分，接下来就是对于参数的打印部分了，这里我只处理了下将字符串参数打印出来，其他特殊的参数我就没有处理了。

```
void VarArgsShowArg(const ScopedObjectAccessAlreadyRunnable& soa,
                                va_list ap,const char* jniMethodName,const char* methodName)
      REQUIRES_SHARED(Locks::mutator_lock_) {
    std::stringstream ss;
        ss << "mikrom jni "< receiver =soa.Decode(obj);
                      if(receiver==nullptr){
                          LOG(ERROR)<<"mikrom  "< cls=receiver->GetClass();
                      if (cls->DescriptorEquals("Ljava/lang/String;")){
                              ObjPtr argStr =soa.Decode(obj);
                              ss<<"arg"<GetValue();
                      }else{
                              ss<<"arg"< receiver =soa.Decode(obj);
                      if(receiver==nullptr){
                          LOG(ERROR)<<"mikrom  "< cls=receiver->GetClass();
                      if (cls->DescriptorEquals("Ljava/lang/String;")){
                              ObjPtr argStr =soa.Decode(obj);
                              ss<<"arg"<GetValue();
                      }else{
                              ss<<"arg"<
```

搞定了封装部分，就看看调用部分是怎么使用的

```
static jdouble CallDoubleMethodA(JNIEnv* env, jobject obj, jmethodID mid, const jvalue* args) {
  CHECK_NON_NULL_ARGUMENT_RETURN_ZERO(obj);
  CHECK_NON_NULL_ARGUMENT_RETURN_ZERO(mid);
  ScopedObjectAccess soa(env);
  //add
  ShowJValue(soa,__FUNCTION__,mid,args);
  //add end
  return InvokeVirtualOrInterfaceWithJValues(soa, obj, mid, args).GetD();
}
 
static jfloat CallFloatMethod(JNIEnv* env, jobject obj, jmethodID mid, ...) {
  va_list ap;
  va_start(ap, mid);
  ScopedVAArgs free_args_later(&ap);
  CHECK_NON_NULL_ARGUMENT_RETURN_ZERO(obj);
  CHECK_NON_NULL_ARGUMENT_RETURN_ZERO(mid);
  ScopedObjectAccess soa(env);
  //add
  ShowVarArgs(soa,__FUNCTION__,mid,ap);
  //add end
  JValue result(InvokeVirtualOrInterfaceWithVarArgs(soa, obj, mid, ap));
  return result.GetF();
}

```

到这里就搞定了 jni 的打印以及参数的打印，大多数调用我都打印了，由于我实战的少，所以可能会漏掉一些常用的打印，如果有发现什么比较有用的打印，也可以告诉我优化下。

#### [](#4、frida-gadget持久化)4、frida-gadget 持久化

​ 这个 frida 持久化的问题，论坛里面也发过一遍又一遍了。我也是参考他们的文章，然后测试优化。先贴上参考文章。

> 参考文章：[ubuntu 20.04 系统 AOSP(Android 11) 集成 Frida](https://www.mobibrw.com/2021/28588)
> 
> 参考文章：[从 0 开始实现一个简易的主动调用框架](https://bbs.pediy.com/thread-270028.htm)

 

​ 不过我的加载时机和他的不太一样，并且我扩展了可自行切换 frida-gadget 的版本，默认集成到系统中的版本是 15.1，如果你喜欢用 14.2 的版本，可以自己上传到手机进行切换，如果你想不 root 的情况下，也可以通过 MikManager 设置切换的版本。下面贴上加载的部分代码。

```
public static void loadGadget(){
        String processName = ActivityThread.currentProcessName();
        for(PackageItem item : mikConfigs){
            if(item.packageName.equals(processName)){
                try{
                    boolean flag=true;
                    if(item.fridaJsPath.length()<=0){
                        continue;
                    }
                    String configPath="/data/data/"+processName+"/libfdgg.config.so";
                    String configPath2="/data/data/"+processName+"/libfdgg32.config.so";
                    if(item.fridaJsPath.equals("listen")||item.fridaJsPath.equals("listen_wait")){
                        WriteConfig(configPath,item.fridaJsPath,item.port);
                        WriteConfig(configPath2,item.fridaJsPath,item.port);
                    }else{
                        File file = new File(item.fridaJsPath);
                        if(!file.exists()){
                            file = new File( "/data/data/" + processName +"/"+file.getName());
                        }
                        if(!file.exists()){
                            Log.e("mikrom", "initConfig package:" + processName+" frida js path:"+item.fridaJsPath+" not found");
                            continue;
                        }
                        WriteConfig(configPath,item.fridaJsPath,item.port);
                        WriteConfig(configPath2,item.fridaJsPath,item.port);
                    }
                    String tagPath = "/data/data/" + processName + "/libfdgg.so";//64位so的目录
                    String tagPath2 = "/data/data/" + processName + "/libfdgg32.so";//32位的so目录
                    //使用用户自己设置的gadget路径
                    if(item.gadgetPath!=null&&item.gadgetPath.length()>0){
                        mycopy(item.gadgetArm64Path, tagPath);//复制so到私有目录
                        mycopy(item.gadgetPath, tagPath2);
                    }else{
                        mycopy("/system/lib64/libfdgg.so", tagPath);//复制so到私有目录
                        mycopy("/system/lib/libfdgg.so", tagPath2);
                    }
                    int perm = FileUtils.S_IRWXU | FileUtils.S_IRWXG | FileUtils.S_IRWXO;
                    FileUtils.setPermissions(tagPath, perm, -1, -1);//将权限改为777
                    FileUtils.setPermissions(tagPath2, perm, -1, -1);
                    FileUtils.setPermissions(configPath, perm, -1, -1);
                    FileUtils.setPermissions(configPath2, perm, -1, -1);
                    File file1 = new File(tagPath);
                    File file2 = new File(tagPath2);
                    if (file1.exists()) {
                        Log.e("mikrom", "app: " +System.getProperty("os.arch"));//判断是64位还是32位
                        if (System.getProperty("os.arch").indexOf("64") >= 0) {
                            Log.e("mikrom", "initConfig package:" + processName+" frida js path:"+item.fridaJsPath+" load arch64");
                            System.load(tagPath);
                            file1.delete();//用完就删否则不会更新
                        } else {
                            Log.e("mikrom", "initConfig package:" + processName+" frida js path:"+item.fridaJsPath+" load 32");
                            System.load(tagPath2);
                            file2.delete();
                        }
                    }
                    Log.e("mikrom", "initConfig package:" + processName+" initConfig over");
                }catch(Exception ex){
                    Log.e("mikrom", "initConfig package:" + processName+" frida js path:"+item.fridaJsPath+" load err:"+ex.getMessage());
                }
                break;
            }
        }
    }

```

同时也是支持以三种模式加载 gadget 的，比如默认加载脚本，或者监听模式，也就是可以 frida 连接上去，或者是监听并阻塞等待。不过我留意到，有些自己写的简单 app 似乎使用 gadget 注入似乎并不成功。好像需要 app 对 native 有什么实现才行。

#### [](#5、so注入以及dobby的内置)5、so 注入以及 dobby 的内置

​ 前面 gadget 的部分实际就是注入 so 了，但是前面的部分主要是针对 gadget 的注入相关处理的。我又特地单独处理了一下对 so 的注入，如果我们自己写了一个 so 想要单独注入进去也是可以的。为了方便演示，我特地将 dobby 内置进去。由于 dobby 是一个 hook 框架，并且没有注入功能，往往都是需要通过其他手段注入到应用中。所以我直接内置进来了。下面贴上相关代码

```
public static void loadSo(String path){
    String processName = ActivityThread.currentProcessName();
    String fName = path.trim();
    String fileName = fName.substring(fName.lastIndexOf("/")+1);
    String tagPath = "/data/data/" + processName + "/"+fileName;//64位so的目录
    mycopy(path, tagPath);
    int perm = FileUtils.S_IRWXU | FileUtils.S_IRWXG | FileUtils.S_IRWXO;
    FileUtils.setPermissions(tagPath, perm, -1, -1);//将权限改为777
    File file = new File(tagPath);
    if (file.exists()){
        Log.e("mikrom", "load so:"+tagPath);
        System.load(tagPath);
        file.delete();//用完就删否则不会更新
    }
}
 
//注入so
public static void loadConfigSo(){
    String processName = ActivityThread.currentProcessName();
    for(PackageItem item : mikConfigs){
        if(!item.packageName.equals(processName))
            continue;
        if(item.soPath.length()<=0)
            continue;
 
        if(item.isDobby){
            if(System.getProperty("os.arch").indexOf("64") >= 0) {
                loadSo("/system/lib64/libdby_64.so");
            }else{
                loadSo("/system/lib/libdby.so");
            }
        }
        String[] soList=item.soPath.split("\n");
        for(String sopath :soList){
            loadSo(sopath);
        }
    }
}

```

#### 6、smali trace

​ 实际上这个功能在 ROM 中是有的，就是 TraceExecution 函数，只是需要控制打开就行了，但是必须走 switch 解释器执行才能执行到便于我们修改的地方。所以这个就有两个地方需要修改，第一个是判断打印的部分，第二个是强制以解释器执行的部分。下面看看我的修改部分

```
static inline void TraceExecution(const ShadowFrame& shadow_frame, const Instruction* inst,
                                  const uint32_t dex_pc)
    REQUIRES_SHARED(Locks::mutator_lock_) {
 
    if(shadow_frame.GetMethod()!=nullptr && ArtMethod::GetTraceMethod()!=nullptr&&strlen(ArtMethod::GetTraceMethod())>0){
        const char* methodName=shadow_frame.GetMethod()->PrettyMethod().c_str();
        if(strstr(methodName,ArtMethod::GetTraceMethod())) {
            std::ostringstream oss;
                    oss << android::base::StringPrintf("mikrom smaliTrace 0x%x: ", dex_pc)
                            << inst->DumpString(shadow_frame.GetMethod()->GetDexFile()) << "\t//";
                    for (uint32_t i = 0; i < shadow_frame.NumberOfVRegs(); ++i) {
                        uint32_t raw_value = shadow_frame.GetVReg(i);
                        ObjPtr ref_value = shadow_frame.GetVRegReference(i);
                        oss << android::base::StringPrintf(" vreg%u=0x%08X", i, raw_value);
                        if (ref_value != nullptr) {
                            if (ref_value->GetClass()->IsStringClass() &&
                                    !ref_value->AsString()->IsValueNull()) {
                                oss << "/java.lang.String \"" << ref_value->AsString()->ToModifiedUtf8() << "\"";
                            } else {
                                oss << "/" << ref_value->PrettyTypeOf();
                            }
                        }
                    }
                    LOG(ERROR)<
```

另外就是强制以解释器执行的部分

```
static inline JValue Execute(
    Thread* self,
    const CodeItemDataAccessor& accessor,
    ShadowFrame& shadow_frame,
    JValue result_register,
    bool stay_in_interpreter = false,
    bool from_deoptimize = false) REQUIRES_SHARED(Locks::mutator_lock_) {
  DCHECK(!shadow_frame.GetMethod()->IsAbstract());
  DCHECK(!shadow_frame.GetMethod()->IsNative());
  ...
  if(ArtMethod::GetTraceMethod()!=nullptr && strlen(ArtMethod::GetTraceMethod())>0){
 
            if(strstr(method->PrettyMethod().c_str(),ArtMethod::GetTraceMethod())){
                std::ostringstream oss;
                oss<< "mikrom Execute strstr:"<PrettyMethod().c_str()<<"\t"<(self, accessor, shadow_frame, result_register,false);
            }
  }
 
  //add end
  ...
} 
```

除了这里，在 LinkerCode 的地方也需要修改

```
static void LinkCode(ClassLinker* class_linker,
                     ArtMethod* method,
                     const OatFile::OatClass* oat_class,
                     uint32_t class_def_method_index) REQUIRES_SHARED(Locks::mutator_lock_) {
  ...
  const void* quick_code = method->GetEntryPointFromQuickCompiledCode();
  bool enter_interpreter = class_linker->ShouldUseInterpreterEntrypoint(method, quick_code);
  if(strlen(ArtMethod::GetTraceMethod())>0){
          std::ostringstream oss;
        oss<<"mikrom LinkCode method:"<PrettyMethod().c_str();
        LOG(ERROR)<
```

变量 enter_interpreter 决定了是否使用解释器执行，如果是 quick 快速模式执行，将无法执行到我们修改的函数。最后测试发现，函数第一次调用的时候，是走的解释执行，成功打印出 trace 记录，第二次调用时，又被优化到快速模式执行了。不过我们已经拿到想要的结果了，我就不继续优化了。关于 LinkCode 这里的修改如果不理解的，可以看看下面的参考文章，里面有详细的流程图了。我就不重复了。

> 参考文章：[将 FART 和 Youpk 结合来做一次针对函数抽取壳的全面提升](https://bbs.pediy.com/thread-260052.htm)

#### [](#7、dex注入（未完成）)7、dex 注入（未完成）

​ 这个功能我原本是想内置一个 IO 重定向，或者是内置一个 Sandhook，然后直接通过配置勾选就能自动注入使用了。然后我们修改的 dex 直接注入进去即可使用。不过目前还没想好怎么设计里面的一些交互部分。所以只是简单的做了下注入的步骤。后续也不一定有时间继续实现，先暂时搁置把。

### MikManager 展示

![](https://bbs.pediy.com/upload/attach/202201/659397_2YQMA39YAFF66RH.gif)

### 使用说明

​ 操作起来非常简单，选择想要处理的应用，然后选择对应功能即可。记得勾选完后，回到应用栏目点保存，如果是要使用 frida 脚本，或者是要选择 so，需要把文件放在 sdcard 中的对应应用的目录中。如果 sdcard 中没有对应目录，可以打开一下应用会自动生成的。

 

​ 脱壳功能的保存结果在`/sdcard/Android/data/<PackageName>/dump`中查看

 

​ 查看 logcat 输出日志统一搜索 mikrom，jni 调用的输出的格式如下

```
2022-01-30 19:43:30.198 4105-4105/com.kanxue.crackme E/.kanxue.crackm: mikrom jni CallNonvirtualVoidMethodV    void java.lang.ClassNotFoundException.(java.lang.String, java.lang.Throwable)    arg1:android.widget.ViewStubarg2:0x6fcc6f50   
2022-01-30 19:43:30.198 4105-4105/com.kanxue.crackme E/.kanxue.crackm: mikrom jni GetArrayLength
2022-01-30 19:43:30.198 4105-4105/com.kanxue.crackme E/.kanxue.crackm: mikrom jni GetStringUTFChars    android.widget.ViewStub
2022-01-30 19:43:30.199 4105-4105/com.kanxue.crackme E/.kanxue.crackm: mikrom jni CallNonvirtualVoidMethodV    void java.lang.ClassNotFoundException.(java.lang.String, java.lang.Throwable)    arg1:android.widget.ViewStubarg2:0x12c2a5e8   
2022-01-30 19:43:30.199 4105-4105/com.kanxue.crackme E/.kanxue.crackm: mikrom jni GetStringUTFChars    android.webkit.ViewStub
2022-01-30 19:43:30.199 4105-4105/com.kanxue.crackme E/.kanxue.crackm: mikrom jni NewStringUTF    android.webkit.ViewStub
2022-01-30 19:43:30.199 4105-4105/com.kanxue.crackme E/.kanxue.crackm: mikrom jni CallObjectMethodV    java.lang.Class java.lang.ClassLoader.loadClass(java.lang.String)    arg1:android.webkit.ViewStub
2022-01-30 19:43:30.199 4105-4105/com.kanxue.crackme E/.kanxue.crackm: mikrom jni GetStringUTFChars    android.webkit.ViewStub
2022-01-30 19:43:30.199 4105-4105/com.kanxue.crackme E/.kanxue.crackm: mikrom jni GetStringUTFChars    android.webkit.ViewStub
2022-01-30 19:43:30.199 4105-4105/com.kanxue.crackme E/.kanxue.crackm: mikrom jni CallNonvirtualVoidMethodV    void java.lang.ClassNotFoundException.(java.lang.String, java.lang.Throwable)    arg1:android.webkit.ViewStubarg2:0x6fcc6f50   
2022-01-30 19:43:30.199 4105-4105/com.kanxue.crackme E/.kanxue.crackm: mikrom jni GetArrayLength 
```

​ RegisterNative 的输出格式如下

```
2022-01-30 19:48:43.219 4576-4576/com.kanxue.crackme E/.kanxue.crackm: mikrom RomPrint RegisterNative name:boolean com.kanxue.crackme.MainActivity.jnicheck(java.lang.String) native_ptr:0x75a5735904 method_idx:568
2022-01-30 19:48:43.219 4576-4576/com.kanxue.crackme E/.kanxue.crackm: mikrom RomPrint RegisterNative name:boolean com.kanxue.crackme.MainActivity.crypt2(java.lang.String) native_ptr:0x75a5739750 method_idx:2

```

​ ArtInvoke 的输出格式如下

```
2022-01-30 19:50:16.247 4686-4686/com.kanxue.crackme E/.kanxue.crackm: mikrom [PerformCall] caller:boolean sun.misc.Unsafe.compareAndSwapInt(java.lang.Object, long, int, int)---called:boolean sun.misc.Unsafe.compareAndSwapInt(java.lang.Object, long, int, int)
2022-01-30 19:50:16.247 4686-4686/com.kanxue.crackme E/.kanxue.crackm: mikrom [PerformCall] caller:void android.graphics.Paint.nSetFlags(long, int)---called:void android.graphics.Paint.nSetFlags(long, int)
2022-01-30 19:50:16.247 4686-4686/com.kanxue.crackme E/.kanxue.crackm: mikrom [PerformCall] caller:int java.lang.String.hashCode()---called:int java.lang.String.hashCode()
2022-01-30 19:50:16.247 4686-4686/com.kanxue.crackme E/.kanxue.crackm: mikrom [PerformCall] caller:void android.graphics.Paint.nSetTextLocalesByMinikinLocaleListId(long, int)---called:void android.graphics.Paint.nSetTextLocalesByMinikinLocaleListId(long, int)
2022-01-30 19:50:16.247 4686-4686/com.kanxue.crackme E/.kanxue.crackm: mikrom [PerformCall] caller:void android.graphics.Paint.nSetDither(long, boolean)---called:void android.graphics.Paint.nSetDither(long, boolean)
2022-01-30 19:50:16.247 4686-4686/com.kanxue.crackme E/.kanxue.crackm: mikrom [PerformCall] caller:void android.graphics.Paint.nSetDither(long, boolean)---called:void android.graphics.Paint.nSetDither(long, boolean)

```

​ smali trace 日志的输出结果如下

```
2022-01-30 20:06:11.003 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0x0: invoke-static {}, void com.mik.miktest.CommonTools.Test() // method@7    // vreg0=0x00000000 vreg1=0x00000000 vreg2=0x00000000 vreg3=0x00000000 vreg4=0x00000000 vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.003 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0x3: const-string v0, "MainActivity" // string@30    // vreg0=0x00000000 vreg1=0x00000000 vreg2=0x00000000 vreg3=0x00000000 vreg4=0x00000000 vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.008 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0x5: const-string v1, "TraceTest" // string@33    // vreg0=0x12C89670/java.lang.String "MainActivity" vreg1=0x00000000 vreg2=0x00000000 vreg3=0x00000000 vreg4=0x00000000 vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.008 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0x7: invoke-static {v0, v1}, int android.util.Log.i(java.lang.String, java.lang.String) // method@0    // vreg0=0x12C89670/java.lang.String "MainActivity" vreg1=0x12C896A8/java.lang.String "TraceTest" vreg2=0x00000000 vreg3=0x00000000 vreg4=0x00000000 vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.008 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0xa: add-int v1, v5, v6    // vreg0=0x12C89670/java.lang.String "MainActivity" vreg1=0x12C896A8/java.lang.String "TraceTest" vreg2=0x00000000 vreg3=0x00000000 vreg4=0x00000000 vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.008 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0xc: new-instance v2, java.lang.StringBuilder // type@TypeIndex[16]    // vreg0=0x12C89670/java.lang.String "MainActivity" vreg1=0x00000010 vreg2=0x00000000 vreg3=0x00000000 vreg4=0x00000000 vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.010 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0xe: invoke-direct {v2}, void java.lang.StringBuilder.() // method@17    // vreg0=0x12C89670/java.lang.String "MainActivity" vreg1=0x00000010 vreg2=0x12C896C8/java.lang.StringBuilder vreg3=0x00000000 vreg4=0x00000000 vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.010 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0x11: const-string v3, "TraceTest," // string@34    // vreg0=0x12C89670/java.lang.String "MainActivity" vreg1=0x00000010 vreg2=0x12C896C8/java.lang.StringBuilder vreg3=0x00000000 vreg4=0x00000000 vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.010 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0x13: invoke-virtual {v2, v3}, java.lang.StringBuilder java.lang.StringBuilder.append(java.lang.String) // method@19    // vreg0=0x12C89670/java.lang.String "MainActivity" vreg1=0x00000010 vreg2=0x12C896C8/java.lang.StringBuilder vreg3=0x12C89708/java.lang.String "TraceTest," vreg4=0x00000000 vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.010 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0x16: move-result-object v2    // vreg0=0x12C89670/java.lang.String "MainActivity" vreg1=0x00000010 vreg2=0x12C896C8/java.lang.StringBuilder vreg3=0x12C89708/java.lang.String "TraceTest," vreg4=0x00000000 vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.010 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0x17: invoke-virtual {v2, v1}, java.lang.StringBuilder java.lang.StringBuilder.append(int) // method@18    // vreg0=0x12C89670/java.lang.String "MainActivity" vreg1=0x00000010 vreg2=0x12C896C8/java.lang.StringBuilder vreg3=0x12C89708/java.lang.String "TraceTest," vreg4=0x00000000 vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.010 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0x1a: move-result-object v2    // vreg0=0x12C89670/java.lang.String "MainActivity" vreg1=0x00000010 vreg2=0x12C896C8/java.lang.StringBuilder vreg3=0x12C89708/java.lang.String "TraceTest," vreg4=0x00000000 vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.010 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0x1b: invoke-virtual {v2}, java.lang.String java.lang.StringBuilder.toString() // method@20    // vreg0=0x12C89670/java.lang.String "MainActivity" vreg1=0x00000010 vreg2=0x12C896C8/java.lang.StringBuilder vreg3=0x12C89708/java.lang.String "TraceTest," vreg4=0x00000000 vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.010 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0x1e: move-result-object v2    // vreg0=0x12C89670/java.lang.String "MainActivity" vreg1=0x00000010 vreg2=0x12C896C8/java.lang.StringBuilder vreg3=0x12C89708/java.lang.String "TraceTest," vreg4=0x00000000 vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.010 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0x1f: invoke-static {v0, v2}, int android.util.Log.i(java.lang.String, java.lang.String) // method@0    // vreg0=0x12C89670/java.lang.String "MainActivity" vreg1=0x00000010 vreg2=0x12C89738/java.lang.String "TraceTest,16" vreg3=0x12C89708/java.lang.String "TraceTest," vreg4=0x00000000 vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.010 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0x22: mul-int v2, v5, v6    // vreg0=0x12C89670/java.lang.String "MainActivity" vreg1=0x00000010 vreg2=0x12C89738/java.lang.String "TraceTest,16" vreg3=0x12C89708/java.lang.String "TraceTest," vreg4=0x00000000 vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.010 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0x24: new-instance v4, java.lang.StringBuilder // type@TypeIndex[16]    // vreg0=0x12C89670/java.lang.String "MainActivity" vreg1=0x00000010 vreg2=0x0000000F vreg3=0x12C89708/java.lang.String "TraceTest," vreg4=0x00000000 vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.010 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0x26: invoke-direct {v4}, void java.lang.StringBuilder.() // method@17    // vreg0=0x12C89670/java.lang.String "MainActivity" vreg1=0x00000010 vreg2=0x0000000F vreg3=0x12C89708/java.lang.String "TraceTest," vreg4=0x12C89758/java.lang.StringBuilder vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.010 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0x29: invoke-virtual {v4, v3}, java.lang.StringBuilder java.lang.StringBuilder.append(java.lang.String) // method@19    // vreg0=0x12C89670/java.lang.String "MainActivity" vreg1=0x00000010 vreg2=0x0000000F vreg3=0x12C89708/java.lang.String "TraceTest," vreg4=0x12C89758/java.lang.StringBuilder vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.010 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0x2c: move-result-object v3    // vreg0=0x12C89670/java.lang.String "MainActivity" vreg1=0x00000010 vreg2=0x0000000F vreg3=0x12C89708/java.lang.String "TraceTest," vreg4=0x12C89758/java.lang.StringBuilder vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.010 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0x2d: invoke-virtual {v3, v2}, java.lang.StringBuilder java.lang.StringBuilder.append(int) // method@18    // vreg0=0x12C89670/java.lang.String "MainActivity" vreg1=0x00000010 vreg2=0x0000000F vreg3=0x12C89758/java.lang.StringBuilder vreg4=0x12C89758/java.lang.StringBuilder vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.010 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0x30: move-result-object v3    // vreg0=0x12C89670/java.lang.String "MainActivity" vreg1=0x00000010 vreg2=0x0000000F vreg3=0x12C89758/java.lang.StringBuilder vreg4=0x12C89758/java.lang.StringBuilder vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.011 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0x31: invoke-virtual {v3}, java.lang.String java.lang.StringBuilder.toString() // method@20    // vreg0=0x12C89670/java.lang.String "MainActivity" vreg1=0x00000010 vreg2=0x0000000F vreg3=0x12C89758/java.lang.StringBuilder vreg4=0x12C89758/java.lang.StringBuilder vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.011 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0x34: move-result-object v3    // vreg0=0x12C89670/java.lang.String "MainActivity" vreg1=0x00000010 vreg2=0x0000000F vreg3=0x12C89758/java.lang.StringBuilder vreg4=0x12C89758/java.lang.StringBuilder vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.011 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0x35: invoke-static {v0, v3}, int android.util.Log.i(java.lang.String, java.lang.String) // method@0    // vreg0=0x12C89670/java.lang.String "MainActivity" vreg1=0x00000010 vreg2=0x0000000F vreg3=0x12C897A8/java.lang.String "TraceTest,15" vreg4=0x12C89758/java.lang.StringBuilder vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.011 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0x38: add-int v0, v1, v2    // vreg0=0x12C89670/java.lang.String "MainActivity" vreg1=0x00000010 vreg2=0x0000000F vreg3=0x12C897A8/java.lang.String "TraceTest,15" vreg4=0x12C89758/java.lang.StringBuilder vreg5=0x00000001 vreg6=0x0000000F
2022-01-30 20:06:11.011 5157-5157/com.mik.miktest E/com.mik.miktes: mikrom smaliTrace 0x3a: return v0    // vreg0=0x0000001F vreg1=0x00000010 vreg2=0x0000000F vreg3=0x12C897A8/java.lang.String "TraceTest,15" vreg4=0x12C89758/java.lang.StringBuilder vreg5=0x00000001 vreg6=0x0000000F 
```

​ so 注入如果勾选自动注入 dobby 后，就会注入系统中自带的，如果你想注入自己编译的 dobby，也可以在 so 注入里面导入 dobby，注入顺序会默认按照导入顺序。

### 完结撒花

​ 整理一遍之后，感觉好像做了很多，又感觉好像也没搞点啥。也碰到很多问题让我一遍又一遍的编译测试。不过总算大致实现了当时的想法。后续应该不会有啥升级了。调转方向研究点其他东西了。如果以后内核功底深一些了，可能会回头扩展把。重要的事情再提一次，不知道修改到了 art 部分的哪个点，导致了调试超级慢，如果有大佬知道啥原因的，麻烦指导一下。

[【公告】欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)

[#源码分析](forum-161-1-127.htm)
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.freebuf.com](https://www.freebuf.com/articles/mobile/425782.html)

> 本文简述了移动端 GPU 安全研究方向、GPU 的攻击面梳理、漏洞分析等内容。

背景
--

GPU 是终端设备负责图形化渲染的硬件，和 CPU 相比，它能更高效地并行计算。传统的 PC 端 GPU 可以通过 PCIe 接口在主板上热插拔，而在移动端，GPU 往往和 CPU 集成在一块芯片上，再搭配负责网络通信的基带等配件，统一称为 SoC。目前移动端市场占有率比较高的 SoC 有高通的骁龙系列处理器、联发科天玑系列处理器、华为海思处理器等。这些 SoC 的 CPU 部分都有统一规范的 ARM 指令集，这保证了一套程序只需编译一次，在不同厂商的 CPU 上都能正确运行。但是这些厂商的 GPU 指令集却非常封闭，甚至有些没有公开的文档，每家厂商在自己的标准上发展了自己的生态，试图构建商业护城河。作为开发者如果需要根据每个硬件厂商定制不同的 GPU 操作逻辑，想必是一件非常复杂的事情，而事实上安卓开发者大多数情况下并不需要直接和 GPU 进行交互，我们在绘制一个窗口、展示一张图片时，是通过调用安卓系统封装的统一接口实现。那安卓又是如何保证这么多硬件兼容性的呢？

上面的兼容性问题其实不止出现在移动端，在 PC 端也普遍存在。为了保证各种 GPU 硬件的兼容性，业界制定了统一的 OpenGL、Vulkan、DirectX 等标准，这些标准约定了名称规范统一的 API 调用约定。各家 SoC 厂商如果想推广自己的硬件，就必须自己负责开发基于这些标准实现的系统驱动、软件链接库。以 OpenGL 标准为例，如果要绘制一个窗口，开发者需要编写下面这样的代码。至于 glfwInit、glfwCreateWindow、glfwMakeContextCurrent、glewInit 这些函数，是由硬件厂商负责编写成链接库，在系统里供我们动态链接。这些链接库函数会和 GPU 内核驱动交互，而 GPU 驱动控制硬件，处理用户的逻辑。

```
int main() {
    // 初始化 GLFW
    if (!glfwInit()) {
        std::cerr << "Failed to initialize GLFW" << std::endl;
        return -1;
    }

    GLFWwindow* window = glfwCreateWindow(800, 600, "OpenGL Rectangle", nullptr, nullptr);
    if (!window) {
        std::cerr << "Failed to create GLFW window" << std::endl;
        glfwTerminate();
        return -1;
    }
    glfwMakeContextCurrent(window);

    // 初始化 GLEW
    if (glewInit() != GLEW_OK) {
        std::cerr << "Failed to initialize GLEW" << std::endl;
        return -1;
    }
    ...
}
```

安卓操作系统约定，硬件厂商的 SoC 在出厂时同时需要提供软件支持，必须满足至少 OpenGL 以及 Vulkan（安卓 7 以后支持）两种调用约定。厂商负责编写的代码有两个部分，内核层的 GPU 驱动以及用户态的 GPU 链接库。这种代码模块分离的思想在其他硬件如音频驱动、摄像头上也有类似的使用，被安卓统一定义为硬件抽象层（Hardware Abstraction Layer），又称 HAL。

![](https://image.3001.net/images/20250325/1742895616_67e27a00c41e85c2442c0.png!small?1742895618045)

安卓只需要在 HAL 层声明需要哪些接口，明确定义好动态链接库导出的函数名称，传参方式，具体实现都交给了厂商。由于硬件厂商众多，他们编写的代码安全性必定不能和主线代码相比。与之矛盾的是，为了和硬件交互，厂商编写的代码往往都直接运行在内核态，一旦驱动代码出现安全问题，整个系统的安全建设将付之一炬，可谓安卓安全生态中最短的一块木板。

当然安卓研发人员也意识到这种问题的严重性，使用 SELinux 约束了这些硬件接口的调用权限。比如普通 app 没有权限直接使用摄像头、麦克风等硬件，必须通过系统的 service 进行数据中转，极大减少了驱动暴露出的攻击面。在 service 中转前，又使用 AOSP 的权限管控约束了 APP 的行为。

![](https://image.3001.net/images/20250325/1742895623_67e27a07652d47a303784.png!small?1742895623874)

然而，GPU 硬件出于性能的考虑，没办法使用 service 进行中转，任意一个 APP 默认就有权限和 GPU 驱动进行交互，在 SELinux 上也没有相应的进行权限管控。这也导致了 GPU 成为安卓安全生态中最为脆弱的一环，在过去一两年的安卓在野漏洞利用中，攻击者无一例外地瞄准了 GPU 驱动，借助 GPU 的驱动漏洞实现从普通 APP 到 root 的权限提升。

![](https://image.3001.net/images/20250325/1742895711_67e27a5f57b8360a1bcd7.png!small?1742895711936)

认识 GPU
------

在利用 GPU 漏洞进行系统提权之前，我们有必要理清楚 GPU 的正常交互逻辑，开发者是如何操作 GPU 进行图形绘制、并行计算呢？

### Shader 编程

在 CPU 计算领域，我们通常使用一些高级编程语言进行程序编写，交给编译器将我们的程序编译成 CPU 能理解的机器码运行。和 CPU 流程类似，GPU 领域也有专用的编程语言，称为 shader，它是控制图形硬件进行图像渲染或单元计算的程序。如下所示，是一个简单的并行计算数组绝对值的 shader 代码。

```
#version 430

layout(std430, binding = 0) buffer InputBuffer {
    float inputData[];
};

layout(std430, binding = 1) buffer OutputBuffer {
    float outputData[];
};

void main() {
    uint index = gl_GlobalInvocationID.x;
    outputData[index] = abs(inputData[index]);
}
```

当然 GPU 硬件肯定没办法理解上述语言是什么意义，从上面的语言到 GPU 硬件指令还需要额外的中间语言编译、硬件语言翻译两个阶段。

我们使用 [glslang](https://github.com/KhronosGroup/glslang) 编译工具链将 shader 代码编译为 SPIR-V 格式的字节码。

```
glslangValidator -V shader_abs.comp -o shader_abs.spv
```

SPIR-V 字节码是由业界提出的一种计算机图形学统一的中间语言。OpenGL、VulKan 等标准提供了接口来加载并运行 SPIR-V 字节码，在运行时，OpenGL 这些框架会动态地将字节码翻译为 GPU 能直接理解的指令码进行执行。

至此我们简单理解了 GPU 的交互过程，这对 GPU 驱动漏洞挖掘还不够，我们需要从更底层的视角去发现问题。像上面的 shader 代码，程序的输入输出是 float 数组，但对于硬件来说这些都是内存里的比特，程序的运行必定涉及到 GPU、CPU 的内存数据交换、共享，这些又是怎么处理的呢？

### GPU 内存模型

传统的 CPU 在使用内存时，提出了虚拟内存的思想。基于硬件控制，不同的进程内存空间相互独立，称为虚拟内存，每个进程的虚拟内存和实际的物理内存之间的映射关系由页表保存。GPU 在内存管理方面和 CPU 有着非常相似的地方。每个 GPU 程序上下文运行在相互独立的虚拟内存空间，GPU 内核驱动负责维护每个 GPU 上下文的页表，管理程序内存申请、释放、和 CPU 的共享内存逻辑。

以高通 GPU 驱动为例，用户态的程序可以通过 ioctl 和 GPU 驱动交互，和 GPU 共享内存。

```
// 打开GPU驱动，创建一个gpu程序上下文
int gslfd = open("/dev/kgsl-3d0",0);
// 申请一块内存
void *buffer = (void *)mmap((void *)0x40000000L, 0x1000L, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANONYMOUS|MAP_FIXED, -1, 0);
if (buffer == MAP_FAILED)
{
    printf("mmap error:%d\n",errno);
    return 0;
}
printf("mmap buffer return %p\n", buffer);
// 配置GPU共享内存参数
struct kgsl_map_user_mem args = {
    .flags = KGSL_MEMFLAGS_USE_CPU_MAP | KGSL_MEMFLAGS_USERMEM_ADDR | (KGSL_CACHEMODE_UNCACHED << KGSL_CACHEMODE_SHIFT) ,
    .memtype = KGSL_USER_MEM_TYPE_ADDR,
    .hostptr = (uint64_t)buffer,
    .offset = 0,
    .len = 0x1000L,
};
ret = ioctl(gslfd, IOCTL_KGSL_MAP_USER_MEM, &args);
if (ret)
{
    printf("ioctl IOCTL_KGSL_MAP_USER_MEM cmd_data error: %d %s\n",errno,strerror(errno));
    return 0;
}
```

继续查看高通 GPU 内核驱动的代码，可以看到驱动里大量复杂的 ioctl 交互逻辑，处理用户态的请求。

![](https://image.3001.net/images/20250325/1742895738_67e27a7a36a4c6fed6795.png!small?1742895738763)

正常的 GPU 交互逻辑，是 app 通过加载 OpenGL 的链接库，在链接库里构造好参数，来和驱动进行 ioctl 交互。但是一个恶意的攻击者可以直接不加载 OpenGL 库，直接调用内核态的驱动代码。如果驱动代码中未正确处理用户请求，就有可能导致 UAF、溢出等内存漏洞。

GPU 漏洞回顾
--------

2024 年 8 月，Google 的 Android Red Team 团队披露了一个高通 GPU 驱动的 UAF 漏洞 CVE-2024-23380，借助这个漏洞，攻击者可以从普通 APP 的权限提升到系统 root。接下来本文对该漏洞的成因、利用过程进行详细分析。

高通 GPU 驱动提供了一系列接口用于操作 GPU 的虚拟内存。如 IOCTL_KGSL_GPUOBJ_ALLOC 用于申请 GPU 对象（内存），API 返回一个 id 标志，用于区分不同的对象。可以通过 IOCTL_KGSL_GPUOBJ_INFO 查询到 id 对应的 GPU 对象所占居的虚拟内存地址、内存大小等信息。

```
struct kgsl_gpuobj_alloc param = {
    .size = 0x1000,
    .flags = KGSL_MEMFLAGS_FORCE_32BIT,
};
ret = ioctl(gslfd, IOCTL_KGSL_GPUOBJ_ALLOC, ¶m);
if(ret < 0) {
    printf("alloc failed %d %s\n",errno,strerror(errno));
    return -1;
}
printf("vbo obj size: 0x%llx, flags: 0x%llx mmapsize: 0x%llx id 0x%x\n", param.size, param.flags, param.mmapsize, param.id);
struct kgsl_gpuobj_info info = {
    .id = param.id,
};
ret = ioctl(gslfd, IOCTL_KGSL_GPUOBJ_INFO, &info);
if(ret < 0) {
    printf("get info failed %d %s\n",errno,strerror(errno));
    return -1;
}
uint64_t obj_addr = info.gpuaddr;
printf("obj_addr = 0x%lx\n",obj_addr);
```

类似地，IOCTL_KGSL_GPUOBJ_FREE 用于释放，IOCTL_KGSL_MAP_USER_MEM 用于将 CPU 的虚拟内存映射到 GPU 虚拟地址，实现内存共享。上述这些内存申请释放操作，实际上驱动的处理逻辑都是在读写 GPU 的页表，建立或释放从 GPU 到物理内存页的映射。

除了正常的 GPU 内存申请操作，IOCTL_KGSL_GPUOBJ_ALLOC 还支持一个特殊的 flag KGSL_MEMFLAGS_VBO。通过查阅驱动代码发现，带有这个特殊 flag 的 GPU 对象在申请时并没有申请对应的物理内存，而是以 zero page 进行占位填充。

![](https://image.3001.net/images/20250325/1742895746_67e27a827a34096f633e5.png!small?1742895747185)

而后又可以通过 IOCTL_KGSL_GPUMEM_BIND_RANGES 操作将其他 GPU 对象的内存页映射到自己对应的虚拟地址空间。相反，也有与之对应的 KGSL_GPUMEM_RANGE_OP_UNBIND 取消映射操作。而这也是本次漏洞的关键所在。

```
struct kgsl_gpumem_bind_range ranges = {
    .child_offset = 0,
    .target_offset = 0,
    .length = 0x1000,
    .child_id = victim_param.id,
    .op = KGSL_GPUMEM_RANGE_OP_BIND,
};
struct kgsl_gpumem_bind_ranges ranges_args = {
    .ranges = (uint64_t)&ranges,
    .ranges_nents = 1,
    .ranges_size = sizeof(struct kgsl_gpumem_bind_range),
    .id = vbo_param.id,
    .flags = 0,
    .fence_id = 0,
};
ret = ioctl(gslfd, IOCTL_KGSL_GPUMEM_BIND_RANGES, &ranges_args);
if(ret < 0) {
    printf("IOCTL_KGSL_GPUMEM_BIND_RANGES failed %d %s\n",errno,strerror(errno));
    return -1;
}
```

![](https://image.3001.net/images/20250325/1742895753_67e27a890a0f864aa7b6f.png!small?1742895753744)

### CVE-2024-23380 漏洞分析

内核驱动开发中，一个需要特别注意的点就是并发控制，当读写一些全局变量时，需要对加锁、释放锁的时机格外注意。如下时 GPU VBO 在进行 BIND_RANGES 时的处理逻辑，用于将 GPU 的虚拟内存页 VA1 映射到另外一块虚拟内存页 VA2。操作完成后，VA1 和 VA2 将指向同一块物理内存。

![](https://image.3001.net/images/20250325/1742895766_67e27a96361d7b9e89aab.png!small?1742895766832)

不难看出，上面的操作中 3 和 4 的顺序被搞反了，正确逻辑应该是先建立好映射后，再释放锁。

BIND_UNRANGES 与之相反，加锁、解除 VA2 和物理页的映射、添加 VA2 和 zero page 的映射、释放锁。

![](https://image.3001.net/images/20250325/1742895782_67e27aa618b2d046f5f66.png!small?1742895782905)

### 漏洞利用

上述的漏洞是由于锁释放的实际不对导致的 race，那触发漏洞必定通过多线程并发实现，那竞争成功后又能达到什么样的效果呢？

在 GPU 中申请分别一个正常的对象、一个带有 VBO 标记的对象，内存页映射关系如下所示。

![](https://image.3001.net/images/20250325/1742895787_67e27aabc9f9ccbef9942.png!small?1742895788437)

当正常 VBO 对象正常执行 BIND_RANGE 后，GPU 的 VA1 和 VA2 会同时指向一块物理内存。如果此时执行 UNBIND_RANGE 操作, 它们又会回到上面的初始状态。

![](https://image.3001.net/images/20250325/1742895793_67e27ab117ac71ef9ec66.png!small?1742895793487)

但当 race 的过程时，如果出现以下的执行过程

![](https://image.3001.net/images/20250325/1742895798_67e27ab63c6fac115cafd.png!small?1742895798978)

在第三步释放 VA1 后，GPU 对应的物理页会被还给系统的内存分配器，然而此时 VA2 仍保留着到物理内存的映射。导致出现 PAGE UAF，我们可以通过控制 GPU 指令，来实现对该物理内存块的读写。

在实际的代码利用过程中，我们可以频繁触发 PAGE UAF 漏洞，使 GPU 保留更多的物理地址页映射，方便后续的占位操作。此时 GPU 映射的物理页虽然处于空闲状态，但是仍保留在 kgsl_page_pool 这个页管理器中，并没有完全被系统回收，需要 Linux 内核触发 memory shrink 后，才会回调_kgsl_pool_shrink，进而将物理页完全交给系统。通过查阅文档得知，正常情况下高权限用户可以通过命令 echo 3> /proc/sys/vm/drop_caches 来触发内存碎片回收，对于一个普通 app 来说，没有权限修改 procfs，可以通过频繁申请内存来触发。

![](https://image.3001.net/images/20250325/1742895802_67e27abae8de6a8582da1.png!small?1742895803760)

在系统完全回收这些内存后，我们可以再次调用 mmap 申请内存，这一步是为了让 CPU 的 PTE table 占据这些空闲页。接下来，我们通过编写 shader 程序，去读写这些 PTE table 对应的 GPU VA，进而间接通过改变 CPU 的虚拟内存映射的物理地址，实现全局物理内存读写。在实验过程中发现，高通处理器系列的 kernel 虽然开启了 KASLR，内核虚拟地址是完全随机化的，但是物理地址却是固定值 0xa8000000L。从这个位置开始，我们就可以读写整个内核空间的代码、数据段，进而实现代码提权。

最后放出一个视频演示

[newdemo.mp4](https://apijoyspace.jd.com/v2/pages/7yQnWPjN3Nk4RAoMZOZt/files/JWF0D90S3dY77yMEpY6v/link)

总结
--

本文简述了移动端 GPU 安全研究方向、GPU 的攻击面梳理、漏洞分析等内容。从攻击者角度来看，纵观过去几年漏洞安全研究历史，传统的内存破坏、整数溢出等漏洞开始淡出历史舞台，这些都可以通过编译期检查、各种 sanitizer 机制加以缓解，但像 mmu misconfiguration 等逻辑型漏洞很难通过 fuzzer 来触达，驱动在运行时更需要硬件支持，这也为自动化漏洞挖掘带来了新的挑战。从防御者角度来看，内核驱动代码的漏洞不可避免，这也不失为当时安卓顶层安全设计的一个失误，既然无法收敛权限，那也许将驱动程序从内核中剥离出来是一种更安全的解决方案，但这更需要 SoC 厂商更多的技术支持，其中的技术路线选型、架构设计和原始的驱动代码开发不尽相同，涉及到的软件稳定性、兼容性、性能验证等工作量又是一个未知数，仅从安全收益的角度来说似乎很难说服 SoC 厂商做出这样的改变。如果安卓官方重新定义一种更为安全的 HAL 层的接口，强制约束厂商的接入方式，理论上是一种更合理的动机，不过这都需要 Android 官方和 SoC 厂商迈出艰难的第一步，究竟 GPU 安全未来的发展如何，我们拭目以待。

参考资料
----

【1】[https://i.blackhat.com/BH-US-24/Presentations/REVISED02-US24-Gong-The-Way-to-Android-Root-Wednesday.pdf](https://i.blackhat.com/BH-US-24/Presentations/REVISED02-US24-Gong-The-Way-to-Android-Root-Wednesday.pdf)

【2】[https://yanglingxi1993.github.io/dirty_pagetable/dirty_pagetable.html](https://i.blackhat.com/BH-US-24/Presentations/REVISED02-US24-Gong-The-Way-to-Android-Root-Wednesday.pdf)
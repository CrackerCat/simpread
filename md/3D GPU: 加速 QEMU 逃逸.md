> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [vul.360.net](https://vul.360.net/archives/368)

> 2020 年冬天，我心血来潮，突然想研究下 QEMU 逃逸。

概述

2020 年冬天，我心血来潮，突然想研究下 QEMU 逃逸。考虑到之前花了很多时间研究安卓手机 GPU，所以想先看看 QEMU 的 GPU。我发现 QEMU 使用了 virglrenderer 库来实现 3D 加速。关于这个库，过去有研究人员做过[**相关研究**](https://gsec.hitb.org/materials/sg2017/D2%20-%20Qiang%20Li%20and%20ZhiBin%20Hu%20-%20QEMU%20Attack%20Surface%20and%20Security%20Internals.pdf)。作为新人，我觉得这是一个很好的研究对象。经过一番审计，我发现 Ubuntu 使用的 virglrenderer 库存在 4 个漏洞，我使用其中的两个漏洞完成了 QEMU 的逃逸。

这篇博客主要介绍：

1.  分析漏洞根因；
2.  如何实现 QEMU 逃逸。在此之前，有[**相关议题**](https://www.blackhat.com/asia-20/briefings/schedule/index.html#d-red-pill--a-guest-to-host-escape-on-qemukvm-virtio-device-18583)介绍了如何利用 virglrenderer 的漏洞实现逃逸。幸运的是，我的方法和它有些不同: P
3.  在研究的过程中，我发现了一些非技术问题。所谓 “千里之堤，溃于蚁穴”，这些问题的存在使得漏洞难以得到及时修复。

攻击面分析

3D GPU（virtio-gpu）的架构如图所示：

![](https://vul.360.net/wp-content/uploads/2022/03/architecture.png)

架构的最上层是 Guest 系统里的应用程序，它通过设备节点（例如：/dev/dri/renderDxxx）与内核驱动通信。DRM_VIRTIO_GPU 驱动支持多种命令，下面列举了最常用到的几种请求命令：

<table><thead><tr><th data-align="center">命令</th><th data-align="center">说明</th></tr></thead><tbody><tr><td data-align="center">VIRTGPU_GET_CAPS</td><td data-align="center">获取 GPU 的能力信息</td></tr><tr><td data-align="center">VIRTGPU_GETPARAM</td><td data-align="center">获取 GPU 的特性信息</td></tr><tr><td data-align="center">VIRTGPU_RESOURCE_CREATE</td><td data-align="center">创建 resource</td></tr><tr><td data-align="center">VIRTGPU_RESOURCE_INFO</td><td data-align="center">获取 resource 信息</td></tr><tr><td data-align="center">VIRTGPU_MAP</td><td data-align="center">映射资源到用户空间（零拷贝）</td></tr><tr><td data-align="center">VIRTGPU_EXECBUFFER</td><td data-align="center">向 GPU 发送命令</td></tr><tr><td data-align="center">VIRTGPU_TRANSFER_FROM_HOST</td><td data-align="center">从 GPU 中读取数据</td></tr><tr><td data-align="center">VIRTGPU_TRANSFER_TO_HOST</td><td data-align="center">向 GPU 写入数据</td></tr></tbody></table>

如果需要的话，驱动会将请求转发给 GPU。具体来说，驱动通常会将 VIRTGPU_EXECBUFFER 命令进行转发，这就使得 Host 向 Guest 暴露了攻击面。当然，这是从 Guest 的应用层的角度来看，如果攻击者自己实现驱动，攻击面会进一步扩大。

vring 作为驱动和硬件通信的桥梁。驱动使用 virtio_gpu_queue_ctrl_buffer() 和 virtio_gpu_queue_cursor() 通过 vring 与 GPU 进行通信。这两个函数分别向 GPU 发送控制请求和光标请求。对应地，virtioio-gpu 中 virtio_gpu_ctrl_bh() 和 virtio_gpu_cursor_bh() 分别处理这两个请求。因为控制请求更复杂，出现问题的可能性更大，所以我主要关注控制请求的处理。 它的处理过程如下：

```
virtio_gpu_ctrl_bh()
|
|-> virtio_gpu_handle_ctrl()
    |
    |-> virtio_gpu_process_cmdq()
        |
        | /* 2D */
        |-> virtio_gpu_simple_process_cmd()
        |
        | /* 3D */
        |-> virtio_gpu_virgl_process_cmd()

```

作为消费者，virtio_gpu_process_cmdq() 从 vring 队列中读取请求数据。如果支持 3D 特性，它会调用 virtio_gpu_virgl_process_cmd() 来处理请求。否则，它会调用 virtio_gpu_simple_process_cmd()。当 3D 特性开启时，硬件支持更多的命令：

<table><thead><tr><th data-align="center">命令</th><th data-align="center">处理函数</th></tr></thead><tbody><tr><td data-align="center">VIRTIO_GPU_CMD_CTX_CREATE</td><td data-align="center">virgl_cmd_context_create()</td></tr><tr><td data-align="center">VIRTIO_GPU_CMD_CTX_DESTROY</td><td data-align="center">virgl_cmd_context_destroy()</td></tr><tr><td data-align="center">VIRTIO_GPU_CMD_RESOURCE_CREATE_2D</td><td data-align="center">virgl_cmd_create_resource_2d()</td></tr><tr><td data-align="center">VIRTIO_GPU_CMD_RESOURCE_CREATE_3D</td><td data-align="center">virgl_cmd_create_resource_3d()</td></tr><tr><td data-align="center">VIRTIO_GPU_CMD_SUBMIT_3D</td><td data-align="center">virgl_cmd_submit_3d()</td></tr><tr><td data-align="center">VIRTIO_GPU_CMD_TRANSFER_TO_HOST_2D</td><td data-align="center">virgl_cmd_transfer_to_host_2d()</td></tr><tr><td data-align="center">VIRTIO_GPU_CMD_TRANSFER_TO_HOST_3D</td><td data-align="center">virgl_cmd_transfer_to_host_3d()</td></tr><tr><td data-align="center">VIRTIO_GPU_CMD_TRANSFER_FROM_HOST_3D</td><td data-align="center">virgl_cmd_transfer_from_host_3d()</td></tr><tr><td data-align="center">VIRTIO_GPU_CMD_RESOURCE_ATTACH_BACKING</td><td data-align="center">virgl_resource_attach_backing()</td></tr><tr><td data-align="center">VIRTIO_GPU_CMD_RESOURCE_DETACH_BACKING</td><td data-align="center">virgl_resource_detach_backing()</td></tr><tr><td data-align="center">VIRTIO_GPU_CMD_SET_SCANOUT</td><td data-align="center">virgl_cmd_set_scanout()</td></tr><tr><td data-align="center">VIRTIO_GPU_CMD_RESOURCE_FLUSH</td><td data-align="center">virgl_cmd_resource_flush()</td></tr><tr><td data-align="center">VIRTIO_GPU_CMD_RESOURCE_UNREF</td><td data-align="center">virgl_cmd_resource_unref()</td></tr><tr><td data-align="center">VIRTIO_GPU_CMD_CTX_ATTACH_RESOURCE</td><td data-align="center">virgl_cmd_ctx_attach_resource()</td></tr><tr><td data-align="center">VIRTIO_GPU_CMD_CTX_DETACH_RESOURCE</td><td data-align="center">virgl_cmd_ctx_detach_resource()</td></tr><tr><td data-align="center">VIRTIO_GPU_CMD_GET_CAPSET_INFO</td><td data-align="center">virgl_cmd_get_capset_info()</td></tr><tr><td data-align="center">VIRTIO_GPU_CMD_GET_CAPSET</td><td data-align="center">virgl_cmd_get_capset()</td></tr><tr><td data-align="center">VIRTIO_GPU_CMD_GET_DISPLAY_INFO</td><td data-align="center">virtio_gpu_get_display_info()</td></tr><tr><td data-align="center">VIRTIO_GPU_CMD_GET_EDID</td><td data-align="center">virtio_gpu_get_edid()</td></tr></tbody></table>

上述大部分的命令会被转发到 virglrenderer，以 VIRTIO_GPU_CMD_SUBMIT_3D 命令为例，它的处理过程如下：

![](https://vul.360.net/wp-content/uploads/2022/03/3d-handle.png)

综上所述，Guest 应用可以通过发送 VIRTGPU_EXECBUFFER 命令直接请求 virtio-gpu，virtio-gpu 会将部分命令转发给 virglrenderer，这使得 virglrenderer 暴露了攻击面。

漏洞

经过一番审计，我发现了四个漏洞，包括：越界写（CVE-2022-0135）、信息泄露（CVE-2022-0175）以及整型下溢。目前这些漏洞已经得到正确修复，披露时间线如下：

```
2022年2月1日    Ubuntu公开细节
2021年12月29日    向redhat披露
2021年12月14日    主线完成漏洞修复
2021年11月27日    向virglrenderer主线披露
2021年11月12日    向Ubuntu官方披露 

```

CVE-2022-0135 (越界写)

越界写漏洞与 VIRTIO_GPU_CMD_SUBMIT_3D 命令处理相关。这个命令包含许多子命令，其中之一是 VIRGL_CCMD_RESOURCE_INLINE_WRITE。它的处理过程如下：

```
virgl_renderer_submit_cmd()
|
|-> vrend_decode_block()
    |
    | 
    |-> vrend_decode_resource_inline_write()
        |
        |-> vrend_transfer_inline_write()
            |
            |-> check_transfer_bounds()
            |
            |-> vrend_renderer_transfer_write_iov()

```

vrend_transfer_inline_write() 只是个包装函数，它调用 vrend_renderer_transfer_write_iov() 来完成真正的写入操作。如果 resource 的存储类型不是 VREND_STORAGE_GL_BUFFER，它会分配一块堆内存，堆的大小计算如下：

```
src/vrend_renderer.c

7019 vrend_renderer_transfer_write_iov(ctx, res, iov, num_iovs, info)
7023 {
7096     send_size = util_format_get_nblocks(res->base.format, info->box->width,
7097                                        info->box->height) * elsize;
7098     if (res->target == GL_TEXTURE_3D ||
7099        res->target == GL_TEXTURE_2D_ARRAY ||
7100        res->target == GL_TEXTURE_CUBE_MAP_ARRAY)
7101        send_size *= info->box->depth;
7103     if (need_temp) {
7104         data = malloc(send_size);
7107     read_transfer_data(iov, num_iovs, data, res->base.format, info->offset,
7108                    stride, layer_stride, info->box, invert);
7280 }

```

首先，util_format_get_nblocks() 根据 resource 的类型、长度和宽度计算出所需空间大小（行 7096）。如果类型是 GL_TEXTURE_3D、GL_TEXTURE_2D_ARRAY 或者 GL_TEXTURE_CUBE_MAP_ARRAY，它还会考虑深度（行 7101）。从这里可以看出，除了上述三种类型，其他类型的深度为 1。之后，read_transfer_data() 将数据拷贝到堆上：

```
src/vrend_renderer.c

6796 read_transfer_data(iov, num_iovs, data, format, offset, box, invert)
6805 {
6808     send_size = util_format_get_nblocks(format, box->width,
6809                                        box->height) * blsize * box->depth;
6814     if ((send_size == size || bh == 1) && !invert && box->depth == 1)
6815        vrend_read_from_iovec(iov, num_iovs, offset, data, send_size);
6816     else {
6817        for (d = 0; d < box->depth; d++) {
6818            myoffset = offset + d * src_layer_stride;
6819            for (h = bh - 1; h >= 0; h--) {
6820                void *ptr = data + (h * bwx) + d * (bh * bwx);
6821                vrend_read_from_iovec(iov, num_iovs, myoffset, ptr, bwx);
6822                myoffset += src_stride;
6823            }
6824         }
6825     }
6837 }

```

奇怪的是：read_transfer_data() 在计算 send_size 时没有考虑类型，直接将深度参与计算（行 6809）。

如果存在一种类型，它不属于上述三种类型，但是它的深度可以大于 1，那么 vrend_renderer_transfer_write_iov() 和 read_transfer_data() 计算出的 send_size 会不一致，最终导致越界写（行 6821）。要验证是否存在这种类型，我们需要知道都有哪些类型。实际上，resource 的类型是在它创建时由 tgsitargettogltarget() 确定的：

```
vrend_renderer_resource_create(args)
|
|-> vrend_renderer_resource_copy_args(args, gr)
|
|-> vrend_renderer_resource_allocate_texture(gr)
    |
    |-> gr->target = tgsitargettogltarget()

```

这个函数有两个参数：target 和 nr_samples，这两个参数决定了 resource 的类型，它们的关系如下：

<table><thead><tr><th data-align="center">Target</th><th data-align="center">Samples</th><th data-align="center">Type</th></tr></thead><tbody><tr><td data-align="center">PIPE_TEXTURE_1D</td><td data-align="center">–</td><td data-align="center">GL_TEXTURE_1D</td></tr><tr><td data-align="center">PIPE_TEXTURE_2D</td><td data-align="center">0</td><td data-align="center">GL_TEXTURE_2D</td></tr><tr><td data-align="center">PIPE_TEXTURE_2D</td><td data-align="center">&gt; 0</td><td data-align="center">GL_TEXTURE_2D_MULTISAMPLE</td></tr><tr><td data-align="center">PIPE_TEXTURE_3D</td><td data-align="center">–</td><td data-align="center">GL_TEXTURE_3D</td></tr><tr><td data-align="center">PIPE_TEXTURE_RECT</td><td data-align="center">–</td><td data-align="center">GL_TEXTURE_RECTANGLE_NV</td></tr><tr><td data-align="center">PIPE_TEXTURE_CUBE</td><td data-align="center">–</td><td data-align="center">GL_TEXTURE_CUBE_MAP</td></tr><tr><td data-align="center">PIPE_TEXTURE_1D_ARRAY</td><td data-align="center">–</td><td data-align="center">GL_TEXTURE_1D_ARRAY</td></tr><tr><td data-align="center">PIPE_TEXTURE_2D_ARRAY</td><td data-align="center">0</td><td data-align="center">GL_TEXTURE_2D_ARRAY</td></tr><tr><td data-align="center">PIPE_TEXTURE_2D_ARRAY</td><td data-align="center">&gt; 0</td><td data-align="center">GL_TEXTURE_2D_MULTISAMPLE_ARRAY</td></tr><tr><td data-align="center">PIPE_TEXTURE_CUBE_ARRAY</td><td data-align="center">–</td><td data-align="center">GL_TEXTURE_CUBE_MAP_ARRAY</td></tr></tbody></table>

有趣的是：resource 的类型有时由 nr_samples 决定。例如，如果 target 是 PIPE_TEXTURE_2D_ARRAY，nr_samples 值是 1 时，resource 的类型是 GL_TEXTURE_2D，而 nr_samples 的值大于 1 时，resource 的类型是 GL_TEXTURE_2D_MULTISAMPLE_ARRAY。

在执行写入操作之前，vrend_transfer_inline_write() 调用 check_transfer_bounds() 来检查应用传入的数据是否合法。这个函数的代码片段如下：

```
src/vrend_renderer.c

6881 check_transfer_bounds(res, info)
6883 {
6908    if (res->base.target == PIPE_TEXTURE_3D) {
6909        int ldepth = u_minify(res->base.depth0, info->level);
6910        if (info->box->depth > ldepth || info->box->depth < 0)
6911            return false;
6916    } else {
6917        if (info->box->depth > (int)res->base.array_size)
6918            return false;
6923    }
6925    return true;
6926 }

```

从中可以看到，当 res->base.target 不是 PIPE_TEXTURE_3D 时，resource 的深度受 res->base.array_size 限制（行 6917）。这两个变量是在 vrend_renderer_resource_copy_args() 中赋值的：

```
src/vrend_renderer.c

6347 vrend_renderer_resource_copy_args(args, gr)
6349 {
6361     gr->base.nr_samples = args->nr_samples;
6362     gr->base.array_size = args->array_size;
6363 }

```

args 参数由应用控制。如果 array_size 的值为 3，那么类型为 GL_TEXTURE_2D_MULTISAMPLE_ARRAY 的 resource 的深度可以大于 1！我们可以通过这种类型触发越界写。

在我看来，程序应该尽可能避免假设。这些假设可能随着时间的推移、代码的演进被遗忘，最终被打破，从而导致漏洞。

CVE-2022-0175(信息泄露)

在创建 resource 时，驱动会请求 GPU 同步 resource 的数据到驱动缓冲区：

```
virtio_gpu_object_create()
|
|-> virtio_gpu_object_attach()
    |
    |-> virtio_gpu_cmd_resource_attach_backing()


virtio_gpu_virgl_process_cmd
|
| VIRTIO_GPU_CMD_RESOURCE_ATTACH_BACKING
|
|-> virgl_cmd_ctx_attach_resource()
    |
    |-> virgl_renderer_resource_attach_iov()


virgl_renderer_resource_attach_iov()
|
|-> vrend_renderer_resource_attach_iov()

```

最终 vrend_renderer_resource_attach_iov() 将 resource 数据拷贝到驱动中。它的代码片段如下：

```
src/vrend_renderer.c

6057 int vrend_renderer_resource_attach_iov(iov, num_iovs)
6059 {
6073    if (has_bit(res->storage_bits, VREND_STORAGE_HOST_SYSTEM_MEMORY)) {
6074        vrend_write_to_iovec(res->iov, res->num_iovs, 0,
6075                            res->ptr, res->base.width0);
6076    }
6078    return 0;
6079 }

```

其中 res->ptr 指向 resource 自己的缓冲区（行 6074），这段缓冲区在 vrend_renderer_resource_create() 分配：

```
src/vrend_renderer.c

6618 int vrend_renderer_resource_create(args, iov, num_iovs, image_oes)
6620 {
6645    if (args->target == PIPE_BUFFER) {
6646        if (args->bind == VIRGL_BIND_CUSTOM) {
6648            gr->storage_bits |= VREND_STORAGE_HOST_SYSTEM_MEMORY;
6649            gr->ptr = malloc(args->width);
6654        }
6704    }
6706    ret = vrend_resource_insert(gr, args->handle);
6711    return 0;
6712 }

```

如果 resource 的 target 是 PIPE_BUFFER，且 bind 字段是 VIRGL_BIND_CUSTOM，那么 resource 的缓冲区由 malloc() 分配（行 6649）。这种情况下，堆上遗留的旧数据并未清空。这些旧数据最终被拷贝到驱动内存中。Guest 应用通过 mmap 系统调用，可以读取这些数据，从而导致信息泄露。由于堆的大小由 Guest 应用控制，因此，我们可以泄露多种大小堆的内容。

整型下溢

这里有两个整型下溢漏洞。第一个位于 vrend_decode_set_shader_images()：

```
src/vrend_decode.c
1210 int vrend_decode_set_shader_images(ctx, length)
1211 {
1213     uint32_t shader_type, start_slot;
1226     if (start_slot > PIPE_MAX_SHADER_IMAGES ||
1227        start_slot > PIPE_MAX_SHADER_IMAGES - num_images)
1228        return EINVAL;
1230     for (uint32_t i = 0; i < num_images; i++) {
1236        vrend_set_single_image_view(ctx->grctx, shader_type, start_slot + i, format, access,
1237                                    layer_offset, level_size, handle);
1238     }
1239     return 0;
1240 }

```

如果 num_images 大于 PIPE_MAX_SHADER_IMAGES，那么将发生整型下溢。假设 PIPE_MAX_SHADER_IMAGES 的值是 32，num_images 的值是 33，减法的结果是 - 1，由于 start_slot 的类型是 uint32_t，-1 被解释成 0xFFFFFFFF，它大于 start_slot 的值。之后，循环变量 i 的值会超过 PIPE_MAX_SHADER_IMAGES，导致 vrend_set_single_image_view() 发生越界写：

```
src/vrend_renderer.c

2941 void vrend_set_single_image_view(ctx, shader_type, index, ..., handle)
2947 {
2948     struct vrend_image_view *iview = &ctx->sub->image_views[shader_type][index];
2951     if (handle) {
2955        res = vrend_renderer_ctx_res_lookup(ctx, handle);
2960         iview->texture = res;
2961         iview->format = tex_conv_table[format].internalformat;
2962         iview->access = access;
2963         iview->u.buf.offset = layer_offset;
2964         iview->u.buf.size = level_size;
2971     }

```

image_views 是一个二维数组，它有 PIPE_SHADER_TYPES * PIPE_MAX_SHADER_IMAGES 个元素。由于 index 大于 PIPE_MAX_SHADER_IMAGES，所以 iview 指针指向的位置越界（行 2948），之后，发生越界写入问题（行 2960-2964）。类似的问题发生在 vrend_decode_set_shader_buffers() 中，它的代码片段如下：

```
src/vrend_decode.c
1151 int vrend_decode_set_shader_buffers(ctx, length)
1152 {
1168    if (start_slot > PIPE_MAX_SHADER_BUFFERS ||
1169        start_slot > PIPE_MAX_SHADER_BUFFERS - num_ssbo)
1170        return EINVAL;
1172    for (uint32_t i = 0; i < num_ssbo; i++) {
1176        vrend_set_single_ssbo(ctx->grctx, shader_type, start_slot + i, offset, buf_len,
1177                            handle);
1178    }
1179    return 0;
1180 }

```

最终导致 vrend_set_single_ssbo() 发生越界写。

QEMU 逃逸

整个逃逸过程分为三步：

1.  泄露 QEMU 某个符号的地址，借此绕过 ASLR；
2.  构造读写原语；
3.  劫持 QEMU 控制流，实现任意命令执行。

第二步包含以下细节：

1.  利用堆特性来猜测 resource 的地址；
2.  如何构建读原语，由于内存模型的限制，需要一些技巧来构造该原语。如果我没理解错的话，这篇 [**BlackHat 议题**](https://www.blackhat.com/asia-20/briefings/schedule/index.html#d-red-pill--a-guest-to-host-escape-on-qemukvm-virtio-device-18583)没有实现该原语；

绕过 ASLR

由于堆的大小我们可以控制，因此，目标对象的选择会灵活些。我希望能够泄露 QEMU 的基地址（而不是某个 so 的地址），它有以下好处：

1.  它不依赖于其他库；
2.  不需要泄露 libc 的基地址，我们可以通过 plt 表实现执行任意命令；

我选取的对象是 BlkAioEmAIOCB，它的字段如下：

```
block/block-backend.c

1325 typedef struct BlkAioEmAIOCB {
1326    BlockAIOCB common;
1327    BlkRwCo rwco;
1328    int bytes;
1329    bool has_returned;
1330 } BlkAioEmAIOCB;

include/block/aio.h

31 struct BlockAIOCB {
32        const AIOCBInfo *aiocb_info;
33        BlockDriverState *bs;
34        BlockCompletionFunc *cb;
35        void *opaque;
36        int refcnt;
37 };

```

其中 common 字段保存了回调函数的地址（行 34、1326），它指向 dma_blk_cb()。我们可以通过打开目录来迫使 QEMU 分配该对象：

```
unsigned long leak_address(void)
{
    unsigned long dma_off;
    struct BlkAioEmAIOCB *aio;
    while (1) {
        dir = opendir(DIR_PATH);
        closedir(dir);
        res = create_resource();
        ptr = map_resource(res);
        aio = (struct BlkAioEmAIOCB*)ptr;
        dma_off = (unsigned long)aio->common.cb - dma_blk_cb;
        if (dma_off % 0x1000 == 0)
            return (unsigned long)aio->common.cb;
    }
    return 0;
}

```

构造读写原语

我使用 vrend_resource 对象来构造原语，整个过程如下：

![](https://vul.360.net/wp-content/uploads/2022/03/rw.png)

通过堆风水，受害者对象紧邻越界写对象。我们借助漏洞篡改受害者对象的 ptr 指针，使其指向 power 对象。正常情况下，ptr 指向数据缓冲区，应用可以写入任意内容。现在我们将它指向 power 对象，那么，我们可以控制 power 对象所有的字段。这里之所以没有直接将 ptr 指向目标地址，是因为我们无法构造出读原语。为了克服这个问题，我们需要通过 power 对象来进一步转化。具体的方式是使 res->iov 指向 res->mipmap_offsets[0]。逻辑上 mipmap_offsets[0] 表示 iov 的起始地址，而 mipmap_offsets[1] 表示 iov 的长度。此时，我们可以通过 VIRGL_CCMD_COPY_TRANSFER3D 命令实现读写原语。要完成上述方法，我们需要解决以下问题：

1.  使受害者对象紧邻越界写对象；
2.  需要知道 power 对象的地址；

正常情况下，两个 resource 之间会插入其他对象。为了使 resource 对象相邻，我们可以先分配一些使用相同 chunk 的对象，然后将这些对象释放，从而迫使 libc 保留一些未使用的 chunk。在两个 resource 之间分配的对象可以使用这些 chunk，从而确保 resource 相邻。我们选取的对象是 pipe_depth_stencil_alpha_state，可以通过 VIRGL_CCMD_CREATE_OBJECT 和 VIRGL_CCMD_DESTROY_OBJECT 命令来分配和释放它：

```
src/vrend_decode.c

vrend_decode_block()
|
| 
|-> vrend_decode_create_object()
|    |
|    |
|    |-> vrend_decode_create_dsa()
|
| 
|-> vrend_decode_destroy_object()

```

为了构造原语，我们需要知道 power 对象的地址。实际上，我们可以基于堆的特性来计算它的地址：

1.  top chunk 新分配的 chunk 地址是连续的；
2.  fastbin 使用单链表来维护 chunk，我们可以借助漏洞来泄露 chunk 的地址；

命令执行

拥有读写原语后，有多种方法实现命令执行。我使用的是 QEMUTimer，它的调用路径如下：

```
til/qemu-timer.c

qemu_clock_run_all_timers()
|
|-> qemu_clock_run_timers()
    |
    |-> timerlist_run_timers()
    |
    | cb = ts->cb; 
    | opaque = ts->opaque;
    | cb(opaque);  

```

非技术性问题

当我在分析、处理这些漏洞的过程中，我发现了一些非技术性问题。这些问题的存在，导致漏洞难以及时修复。不单单是下游产品未能及时修复漏洞，甚至上游也是如此。我个人觉得未修复的 Nday 漏洞危害要比 0day 更大：

1. 漏洞细节已经公开，甚至利用代码都可以下载；

2. 随着时间的推移，这些漏洞会被遗忘，成为 “幽灵”；

问题 1：两年前公开的漏洞至今仍未修复

在我完成逃逸后的某天，我突然发现两年前的 [BlackHat 议题](https://www.blackhat.com/asia-20/briefings/schedule/index.html#d-red-pill--a-guest-to-host-escape-on-qemukvm-virtio-device-18583)已经公开信息泄露漏洞。我觉得造成这种情况的原因可能有：

1.  议题作者没有向主线披露信息泄露漏洞，只披露了其他漏洞；
2.  开发人员很少关注漏洞信息；

问题 2：Ubuntu 未能及时修复已公开的漏洞

在我向 Ubuntu 披露整型下溢漏洞前 5 个月，主线已经修复了 vrend_decode_set_shader_buffers() 中的[问题](https://gitlab.freedesktop.org/virgl/virglrenderer/-/commit/673f4d0c1dfe78f66e8d7f036c619065800022de)。我觉得出现这种情况的原因可能有：

1.  上游开发者在修复问题时，未对漏洞进行标识。这会给下游维护者带来困难：他们要在诸多补丁中准确识别漏洞。当代码差异到一定程度时，确认难度可想而知；
2.  上游开发者不会主动申请 CVE，这会导致漏洞难以得到有效追踪；

在我看来，上游开发者应该主动为漏洞申请 CVE（比如通过 RedHat），并定期在相关页面进行公告，方便下游开发者及时了解漏洞信息。

总结

本文首先分析了作者在 virglrenderer 中发现的四个漏洞，然后介绍如何实现 QEMU 逃逸，最后指出作者发现的一些非技术问题，这些问题的存在导致漏洞难以及时被修复。作者希望上游开发者能够为漏洞申请 CVE，并及时公告漏洞信息，以方便下游及时修复漏洞。
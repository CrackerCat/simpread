> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-222656.htm)

4. Hyper-V 虚拟机与宿主机的数据传输

         Hyper-V 虚拟机和宿主机之间的数据传输是 Hyper-V 安全研究中一个最重要的部分，弄清楚数据传输的方法，流程，才能清楚地知道 Hyper-V 的攻击面，并且能根据数据的流向和处理分析出函数的功能和某些重要结构体的成员分布，为逆向工作减轻了工作量。同时，也为复现漏洞的过程提供了帮助。

         下面，我们从虚拟机到宿主机的顺序，依次介绍数据的流向。这些顺序分别是：虚拟机内核层，Hypervisor 层，Windows 内核层，Windows 应用层。其中，Hypervisor 层我们不做过多的介绍，只介绍这层大致的功能。原因是 Hypervisor 层的代码多是有关 VT 指令的处理，要完全弄清楚内部的实现，需要大量时间做逆向工作，而且攻击面较窄，不容易出现问题，Hypervisor 层代码也很少有解析从虚拟机传来数据的过程。就 Windows 内核层和 Windows 应用层的数据传输来说，他们有很多解析虚拟机传来数据的操作，拥有广泛的攻击面。所以，我们着重介绍其他 3 个方面的数据传输。不过，有兴趣的读者也可以自行逆向、调试 Hypervisor 层代码，一方面能增加对 VT 技术的理解，另一方面能了解虚拟化原理相关的知识，但是在本书中就不多赘述。

         在介绍数据传输的过程中，希望读者能根据书中介绍自行调试一遍，以加深对数据传输流程的理解。

4.1. 数据传输流程概览

         在介绍具体的数据传输流程之前，我们先对它进行一个大体的认识，这样在之后的分层介绍时有一个整体的概览，到时候回看本小节的内容可以帮助您理清思路。

         如图 1-31，描述了顺序为虚拟机到宿主机的数据传输过程，下面几个小节也是根据这个框架分层展开介绍的。

![](https://bbs.pediy.com/upload/attach/201711/624619_kegwixk7ztu1ht2.png)

图 1-31

         图 1-31 描述了整个数据传输的大致流程，通过这幅图我们能了解 Hyper-V 的整体数据传输的原理。

        如果有数据要从虚拟机发送，那么会将数据填入环形内存 (buffer_ring)，然后经由 VMBus 调用 Hypercall 通知 Hypervisor 层有数据到来；如果宿主机发来数据，则通过虚拟机中断，调用中断处理程序中对应的回调函数(callback)，回调函数会从收到的环形内存(buffer_ring) 中读取数据，并加以解析。

        Hypervisor 层用于处理 VM-Exit 事件，触发宿主机和虚拟机中断来通知宿主机和虚拟机有数据到来。

        Windows 内核层同样使用中断方法获取到 Hypervisor 层发来的数据，在通过 Windows nt 模块分发到 vmbusr 模块中，然后根据数据的类型分发到 vmbkmclr 模块中，vmbkmclr 再分发到相应虚拟设备驱动中，或者通知用户态程序有数据来临；如果有数据从 VMBus 发到虚拟机的话，则会调用 Hypercall 产生一个 VM-Exit 事件，陷入 Hypervisor 层处理。

        Windows 用户态的某些 Dll 用于模拟显示，鼠标键盘，动态内存，集成服务等设备。用户态和内核的通信主要通过调用 Writefile/Readfile 读写不同设备对应的命名管道，如果用户态要发送数据到内核，调用 Writefile 直接将数据写入命名管道；如有数据从内核传到用户态，内核会通知用户态程序需要接收数据，用户态程序调用 Readfile 函数，从命名管道中读取数据。

        在下面的介绍中，我们会详细介绍图中的内容，并且会着重介绍 Windows 内核部分的数据传输。

4.2. 虚拟机操作系统层

         我们先从虚拟机内核部分说起，这里还是用 Linux 内核代码作为介绍。笔者使用的 Linux 内核版本是 4.7.2，可能不同版本的 Linux 内核代码会稍有不同。

         Hyper-V 设备在 Linux 内核中的位置如表 1-6。

![](https://bbs.pediy.com/upload/attach/201711/624619_jnnettrkl9olbkt.png)

         在介绍虚拟机内核部分的数据传输时，我们以 hyperv_fb 设备为例，分别介绍数据传出和传入虚拟机的流程。

         这个 hyperv_fb 设备是虚拟显示设备，可以理解为它虚拟出来一个 vga 显示器，用于传输和控制虚拟机显示的画面。这个模块对应的 Linux 内核模块文件为./linux-4.7.2/drivers/video/fbdev/hyperv_fb.c，下面我们从这个文件开始，解读虚拟机内核部分的数据传输。

从宿主机接收数据

         在 Linux 虚拟机中，每当宿主机发来数据都会以中断的方法通知虚拟机进行处理。图 1-32 中，显示了 Linux 虚拟机所有注册的中断。图中的倒数第五行就是 Hyper-V 在 Linux 内核中注册的中断，每当宿主机有数据发过来时，Hypervisor 层会触发这个中断来通知内核需要接收数据，并且对数据进行处理。

         下面我们以 hyperv_fb 设备进行整个数据接收过程的分析。

![](https://bbs.pediy.com/upload/attach/201711/624619_svjsq87xulkduml.png)

图 1-32

         先从 hyperv_fb 驱动中出发，当有数据传入虚拟机时，会调用回调函数 synthvid_receive，那么我们看一下这个回调函数是如何知道数据到达了呢。在驱动初始化过程中，synthvid_connect_vsp 函数调用 vmbus_open 函数来注册 hyperv_fb 驱动收到数据时的回调函数，如下代码所示。

```
static int synthvid_connect_vsp(struct hv_device *hdev)
{
struct fb_info *info = hv_get_drvdata(hdev);
struct hvfb_par *par = info->par;
int ret;
ret = vmbus_open(hdev->channel, RING_BUFSIZE, RING_BUFSIZE,
NULL, 0, synthvid_receive, hdev);
if (ret) {
pr_err("Unable to open vmbus channel\n");
return ret;
}
.....
return 0;
 
error:
vmbus_close(hdev->channel);
return ret;
}

```

         下面我们来看看 vmbus_open 函数中到底做了什么操作，代码如下。

```
int vmbus_open(struct vmbus_channel *newchannel, 
u32 send_ringbuffer_size,
             u32 recv_ringbuffer_size, void *userdata, u32 userdatalen,
             void (*onchannelcallback)(void *context), void *context)
{
struct vmbus_channel_open_channel *open_msg;
struct vmbus_channel_msginfo *open_info = NULL;
void *in, *out;
unsigned long flags;
int ret, err = 0;
unsigned long t;
struct page *page;
 
......
 
newchannel->onchannel_callback = onchannelcallback;
newchannel->channel_callback_context = context;
......
}

```

         vmbus_open 函数中，主要操作是新建了一个 channel，这个操作在驱动初始化的时候完成，所以这个 channel 也就是对应该设备建立的。在上面的代码中，函数将回调函数 synthvid_receive 的地址放入 newchannel 的 onchannel_callback 字段，这里初始化当数据传来时调用的回调函数。下面，我们继续研究是谁调用了 vmbus_channel 结构中 onchannel_callback 字段中存储的回调函数。

         在此之前，我们先了解下中断处理函数是如何注册以及调用的。文件./linux-4.7.2/drivers/hv/vmbus_drv.c 是 hv_vmbus 模块的源文件之一，其中的 vmbus_bus_init 函数是用来初始化 VMBus 驱动，并且注册 Hyper-V 在 Linux 虚拟机之中的驱动，如下面简化过的代码所示。

```
static int vmbus_bus_init(void)
{
......
 
hv_setup_vmbus_irq(vmbus_isr);
 
......
}

```

         vmbus_bus_init 函数通过调用 hv_setup_vmbus_irq 函数来注册 Hyper-V 在 Linux 虚拟机中的中断处理程序。函数 hv_setup_vmbus_irq 的原型如下面的代码所示，其源文件的位置为./linux-4.7.2/arch/x86/kernel/cpu/mshyperv.c。

```
void hv_setup_vmbus_irq(void (*handler)(void))
{
vmbus_handler = handler;
/*
 * Setup the IDT for hypervisor callback. Prevent reallocation
 * at module reload.
 */
if (!test_bit(HYPERVISOR_CALLBACK_VECTOR, used_vectors))
alloc_intr_gate(HYPERVISOR_CALLBACK_VECTOR,
hyperv_callback_vector);
}

```

         通过 hv_setup_vmbus_irq 函数，注册了中断处理函数 hyperv_callback_vector，并且将 handler 赋给全局变量 vmbus_handler，这里的 vmbus_handler 中指向的就是上文中的 vmbus_isr 函数。下面我们继续查看 hyperv_callback_vector 的定义，在./linux-4.7.2/arch/x86/entry/entry_64.S 文件中我们发现这么一段定义，如下面的代码所示。

```
apicinterrupt3 HYPERVISOR_CALLBACK_VECTOR \
                        hyperv_callback_vector hyperv_vector_handler

```

         这句语句说明 hyperv_callback_vector 中断处理函数实际上是 hyperv_vector_handler 函数的别名，我们再来查看 hyperv_vector_handler 函数的定义，如下面代码所示，定义这个函数的文件位置为./linux-4.7.2/arch/x86/kernel/cpu/mshyperv.c。

```
void hyperv_vector_handler(struct pt_regs *regs)
{
struct pt_regs *old_regs = set_irq_regs(regs);
 
entering_irq();
inc_irq_stat(irq_hv_callback_count);
if (vmbus_handler)
vmbus_handler();
 
exiting_irq();
set_irq_regs(old_regs);
}

```

         从上面的代码中可以看出，当中断来临时，函数 hyperv_vector_handler 会调用 vmbus_handler 来进行处理，从上文可知，vmbus_handler 指向的就是函数 vmbus_isr。那么也就是说，当 Hypervisor 层发来数据时，便会调用 vmbus_isr 函数用于对通知的处理。

         vmbus_isr 函数简化的原型如下面代码所示，定义函数所在文件位置为./linux-4.7.2/drivers/hv/vmbus_drv.c。

```
static void vmbus_isr(void)
{
......
if (handled)
tasklet_schedule(hv_context.event_dpc[cpu]);
 
page_addr = hv_context.synic_message_page[cpu];
msg = (struct hv_message *)page_addr + VMBUS_MESSAGE_SINT;
 
/* Check if there are actual msgs to be processed */
if (msg->header.message_type != HVMSG_NONE) {
if (msg->header.message_type == HVMSG_TIMER_EXPIRED)
hv_process_timer_expiration(msg, cpu);
else
tasklet_schedule(hv_context.msg_dpc[cpu]);
    }
 
add_interrupt_randomness(HYPERVISOR_CALLBACK_VECTOR, 0);
}

```

         上面的代码中，使用了 tasklet_schedule 函数来调用 hv_context.event_dpc[cpu] 注册的延迟函数。下面我们看看 hv_context.event_dpc[cpu] 描述符中注册了什么函数，代码所示，所在文件为./linux-4.7.2/drivers/hv/hv.c。

```
int hv_synic_alloc(void)
{
......
for_each_online_cpu(cpu) {
hv_context.event_dpc[cpu] = kmalloc(size, GFP_ATOMIC);
if (hv_context.event_dpc[cpu] == NULL) {
pr_err("Unable to allocate event dpc\n");
goto err;
}
tasklet_init(hv_context.event_dpc[cpu], vmbus_on_event, cpu);
 
hv_context.msg_dpc[cpu] = kmalloc(size, GFP_ATOMIC);
if (hv_context.msg_dpc[cpu] == NULL) {
pr_err("Unable to allocate event dpc\n");
goto err;
}
tasklet_init(hv_context.msg_dpc[cpu], vmbus_on_msg_dpc, cpu);
 
hv_context.clk_evt[cpu] = kzalloc(ced_size, GFP_ATOMIC);
if (hv_context.clk_evt[cpu] == NULL) {
pr_err("Unable to allocate clock event device\n");
goto err;
}
 
hv_init_clockevent_device(hv_context.clk_evt[cpu], cpu);
 
hv_context.synic_message_page[cpu] =
(void *)get_zeroed_page(GFP_ATOMIC);
 
if (hv_context.synic_message_page[cpu] == NULL) {
pr_err("Unable to allocate SYNIC message page\n");
goto err;
}
 
hv_context.synic_event_page[cpu] =
(void *)get_zeroed_page(GFP_ATOMIC);
 
if (hv_context.synic_event_page[cpu] == NULL) {
pr_err("Unable to allocate SYNIC event page\n");
goto err;
}
 
hv_context.post_msg_page[cpu] =
(void *)get_zeroed_page(GFP_ATOMIC);
 
if (hv_context.post_msg_page[cpu] == NULL) {
pr_err("Unable to allocate post msg page\n");
goto err;
}
}
return 0;
err:
return -ENOMEM;
}

```

         上述代码中，hv_synic_alloc 函数是用作初始化 tasklet 的函数，其中初始化了 hv_context.event_dpc[cpu] 描述符，并注册软中断函数为 vmbus_on_event。vmbus_on_event 函数的简化原型如下所示，文件为./linux-4.7.2/drivers/hv/connection.c。

```
void vmbus_on_event(unsigned long data)
{
......
if (!recv_int_page)
return;
for (dword = 0; dword < maxdword; dword++) {
if (!recv_int_page[dword])
continue;
for (bit = 0; bit < 32; bit++) {
if (sync_test_and_clear_bit(bit,
(unsigned long *)&recv_int_page[dword])) {
relid = (dword << 5) + bit;
 
if (relid == 0)
/*
* Special case - vmbus
* channel protocol msg
*/
continue;
process_chn_event(relid);
}
}
}
}

```

         函数 vmbus_on_event 调用 process_chn_event 函数，process_chn_event 函数如下所示，文件位置为./linux-4.7.2/drivers/hv/connection.c。

```
static void process_chn_event(u32 relid)
{
struct vmbus_channel *channel;
void *arg;
bool read_state;
u32 bytes_to_read;
channel = pcpu_relid2channel(relid);
 
if (!channel)
return;
 
if (channel->onchannel_callback != NULL) {
arg = channel->channel_callback_context;
read_state = channel->batched_reading;
 
do {
if (read_state)
hv_begin_read(&channel->inbound);
channel->onchannel_callback(arg);//调用特定channel的回调函数
if (read_state)
bytes_to_read = hv_end_read(&channel->inbound);
else
bytes_to_read = 0;
} while (read_state && (bytes_to_read != 0));
}
}

```

         上述代码中，注释部分调用了 vmbus_channel 结构中 onchannel_callback 字段的回调函数。这里的 onchannel_callback 的值就是上文中的 onchannelcallback 变量的值，即回调函数 synthvid_receive 的地址。也就是说，每当有数据到达虚拟机，便会经过这么多层函数的传递，然后调用 hyperv_fb 驱动特定 channel 的回调函数，执行 hyperv_fb 驱动接收数据的函数 synthvid_receive。

         以 hyperv_fb 驱动为例，当宿主机传来数据时，经过中断处理程序的分发，调用特定 channel 对应的回调函数处理发来的数据，并且不同的设备使用的回调函数不同，比如 hyperv_fb 驱动使用的是 synthvid_receive 函数。那么下面我们来看看 synthvid_receive 函数中是如何将数据从宿主机中读取出来的，如下面代码所示。

```
static void synthvid_receive(void *ctx)
{
  struct hv_device *hdev = ctx;
  struct fb_info *info = hv_get_drvdata(hdev);
  struct hvfb_par *par;
  struct synthvid_msg *recv_buf;
  u32 bytes_recvd;
  u64 req_id;
  int ret;
 
  if (!info)
    return;
 
  par = info->par;
  recv_buf = (struct synthvid_msg *)par->recv_buf;
  do {
    ret = vmbus_recvpacket(hdev->channel, recv_buf,
                           MAX_VMBUS_PKT_SIZE,
                           &bytes_recvd, &req_id);
    if (bytes_recvd > 0 &&
        recv_buf->pipe_hdr.type == PIPE_MSG_DATA)
        synthvid_recv_sub(hdev);
  } while (bytes_recvd > 0 && ret == 0);
}

```

         从上面的代码可以看出，synthvid_receive 函数调用了 vmbus_recvpacket 函数来获取从宿主机发来的数据。vmbus_recvpacket 函数如下，文件位置为./linux-4.7.2/drivers/hv/channel.c。

```
static inline int
__vmbus_recvpacket(struct vmbus_channel *channel, void *buffer,
                   u32 bufferlen, u32 *buffer_actual_len, u64 *requestid,
                   bool raw)
{
  int ret;
  bool signal = false;
 
  ret = hv_ringbuffer_read(&channel->inbound, buffer, bufferlen,
                           buffer_actual_len, requestid, &signal, raw);
 
  if (signal)
    vmbus_setevent(channel);
 
  return ret;
}

```

         在 vmbus_recvpacket 函数中，先调用了 hv_ringbuffer_read 函数读出环形内存中的数据，如果读满一个环形了，那就调用 vmbus_setevent 函数通知宿主机读取数据完成。若在这之后还有数据，则要通知虚拟机再去读取数据，再次执行从中断处理程序开始的流程；若没有数据则不通知虚拟机读数据。如果没有读完一整个环形内存，那么函数便直接返回。hv_ringbuffer_read 函数主要是读取环形内存的操作，读者可以自行去查看这部分代码，这里就不再赘述，vmbus_setevent 函数我们会在下文着重介绍。然后，虚拟机读取完宿主机传来的数据，解析，然后继续传递给内核其他函数加以处理。到这，虚拟机就算完成了数据的接收。

         到此为止，我们便介绍完虚拟机是如何从宿主机接收数据的流程。

发送数据至宿主机

         我们还是以 hyperv_fb 驱动为例子，探索虚拟机是如何将数据发送到宿主机的。在 hyperv_fb 驱动中，发送数据到宿主机的函数为 synthvid_send。下面是 synthvid_send 函数的原型，源文件位置为./linux-4.7.2/drivers/video/fbdev/hyperv_fb.c。

```
static inline int synthvid_send(struct hv_device *hdev,
                                struct synthvid_msg *msg)
{
  static atomic64_t request_id = ATOMIC64_INIT(0);
  int ret;
 
  msg->pipe_hdr.type = PIPE_MSG_DATA;
  msg->pipe_hdr.size = msg->vid_hdr.size;
 
  ret = vmbus_sendpacket(hdev->channel, msg,
                         msg->vid_hdr.size + sizeof(struct pipe_msg_hdr),
                         atomic64_inc_return(&request_id),
                         VM_PKT_DATA_INBAND, 0);
 
  if (ret)
    pr_err("Unable to send packet via vmbus\n");
 
  return ret;
}

```

         上面的代码调用了函数 vmbus_sendpacket 函数将 msg 指向的数据发送至宿主机，vmbus_sendpacket 紧接着调用了 vmbus_sendpacket_ctl 函数，下面是 vmbus_sendpacket_ctl 函数的原型，文件位置为./linux-4.7.2/drivers/hv/channel.c。

```
int vmbus_sendpacket_ctl(struct vmbus_channel *channel, void *buffer,
                         u32 bufferlen, u64 requestid,
                         enum vmbus_packet_type type, u32 flags, bool kick_q)
{
  struct vmpacket_descriptor desc;
  u32 packetlen = sizeof(struct vmpacket_descriptor) + bufferlen;
  u32 packetlen_aligned = ALIGN(packetlen, sizeof(u64));
  struct kvec bufferlist[3];
  u64 aligned_data = 0;
  int ret;
  bool signal = false;
  bool lock = channel->acquire_ring_lock;
  int num_vecs = ((bufferlen != 0) ? 3 : 1);
 
  /* Setup the descriptor */
  desc.type = type; /* VmbusPacketTypeDataInBand; */
  desc.flags = flags; /* VMBUS_DATA_PACKET_FLAG_COMPLETION_REQUESTED; */
  /* in 8-bytes granularity */
  desc.offset8 = sizeof(struct vmpacket_descriptor) >> 3;
  desc.len8 = (u16)(packetlen_aligned >> 3);
  desc.trans_id = requestid;
 
  bufferlist[0].iov_base = &desc;
  bufferlist[0].iov_len = sizeof(struct vmpacket_descriptor);
  bufferlist[1].iov_base = buffer;
  bufferlist[1].iov_len = bufferlen;
  bufferlist[2].iov_base = &aligned_data;
  bufferlist[2].iov_len = (packetlen_aligned - packetlen);
 
  ret = hv_ringbuffer_write(&channel->outbound, bufferlist, num_vecs,
                            &signal, lock);
 
  if (channel->signal_policy)
    signal = true;
  else
    kick_q = true;
 
  if (((ret == 0) && kick_q && signal) ||
       (ret && !is_hvsock_channel(channel)))
    vmbus_setevent(channel);
 
  return ret;
}

```

         上面的代码是将要发送的数据，即 buffer 指向的内存填入环形内存中，然后通过函数 vmbus_setevent 通知宿主机去读取环形内存中的内容。函数 hv_ringbuffer_write 便是将数据填入环形内存的函数，读者可以自行去查阅这部分代码，这里不再赘述。如果环形内存写满，则会调用 vmbus_setevent 函数来通知宿主机。由于虚拟机向宿主机传递数据是通过通知宿主机读取环形内存的方式，所以我们主要探究虚拟机是如何通知宿主机。函数 vmbus_setevent 原型如下，文件位置为./linux-4.7.2/drivers/hv/channel.c。

```
static void vmbus_setevent(struct vmbus_channel *channel)
{
  struct hv_monitor_page *monitorpage;
 
  if (channel->offermsg.monitor_allocated) {
    /* Each u32 represents 32 channels */
    sync_set_bit(channel->offermsg.child_relid & 31,
      (unsigned long *) vmbus_connection.send_int_page +
      (channel->offermsg.child_relid >> 5));
 
    /* Get the child to parent monitor page */
    monitorpage = vmbus_connection.monitor_pages[1];
 
    sync_set_bit(channel->monitor_bit,
      (unsigned long *)&monitorpage->trigger_group
            [channel->monitor_grp].pending);
 
  } else {
    vmbus_set_event(channel);
  }
}

```

         从上面的代码中可以看出，vmbus_setevent 函数通知宿主机的方法有两种。一种是利用 monitorpage，在它的 trigger_group 数组成员中的 pending 成员置位来实现通知宿主机；另一种是继续调用 vmbus_set_event 函数，vmbus_set_event 函数使用 hypercall 通知宿主机。从实际使用中，一般来说网卡 (hv_netvsc)，硬盘(hv_storvsc) 设备会使用 monitorpage 的方法通知宿主机，而集成服务 (hv_utils)，键盘(hyperv_keyboard)，鼠标(hid_hyperv)，动态内存(hv_balloon)，视频(hyperv_fb) 设备会使用 hypercall 方式通知宿主机。下面我们分别介绍这两种通知宿主机的方式。

1.monitorpage 方式

         如上面的代码所示，monitorpage 通知宿主机的方法十分直接，直接在特定内存位置置位即可。下面我们来看一下 monitorpage 的初始化过程，初始化操作在 vmbus_connect 函数中，文件位置为./linux-4.7.2/drivers/hv/connection.c。

```
int vmbus_connect(void)
{
  ......
  /*
   * Setup the vmbus event connection for channel interrupt
   * abstraction stuff
   */
  vmbus_connection.int_page =
  (void *)__get_free_pages(GFP_KERNEL|__GFP_ZERO, 0);
  if (vmbus_connection.int_page == NULL) {
  ret = -ENOMEM;
  goto cleanup;
  }
  vmbus_connection.recv_int_page = vmbus_connection.int_page;
  vmbus_connection.send_int_page =
    (void *)((unsigned long)vmbus_connection.int_page +
      (PAGE_SIZE >> 1));
  /*
   * Setup the monitor notification facility. The 1st page for
   * parent->child and the 2nd page for child->parent
   */
  vmbus_connection.monitor_pages[0] =
         (void *)__get_free_pages((GFP_KERNEL|__GFP_ZERO), 0);
  vmbus_connection.monitor_pages[1] =
 (void *)__get_free_pages((GFP_KERNEL|__GFP_ZERO), 0);
  if ((vmbus_connection.monitor_pages[0] == NULL) ||
       (vmbus_connection.monitor_pages[1] == NULL)) {
    ret = -ENOMEM;
    goto cleanup;
  }
  msginfo = kzalloc(sizeof(*msginfo) +
        sizeof(struct vmbus_channel_initiate_contact),
        GFP_KERNEL);
  if (msginfo == NULL) {
    ret = -ENOMEM;
    goto cleanup;
  }
  version = VERSION_CURRENT;
 
  do {
    ret = vmbus_negotiate_version(msginfo, version);//send to host machine
    if (ret == -ETIMEDOUT)
      goto cleanup;
 
    if (vmbus_connection.conn_state == CONNECTED)
      break;
 
    version = vmbus_get_next_version(version);
  } while (version != VERSION_INVAL);
 
  if (version == VERSION_INVAL)
    goto cleanup;
  vmbus_proto_version = version;
  kfree(msginfo);
  return 0;
 
cleanup:
  pr_err("Unable to connect to host\n");
  vmbus_connection.conn_state = DISCONNECTED;
  vmbus_disconnect();
  kfree(msginfo);
  return ret;
}

```

         然后，通过调用 vmbus_negotiate_version 函数，将刚刚分配的 monitor_pages 地址发送到宿主机中，完成了 monitorpage 的注册过程。之后，如果需要通知宿主机，便可直接在 monitorpage 特定内存位置置位即可

2.hypercall 方式

         Hypercall 方式通知虚拟机通过 vmbus_setevent 函数调用 vmbus_set_event 函数实现，下面我们来看看 vmbus_set_event 函数中的内容，代码如下。文件位置为./linux-4.7.2/drivers/hv/connection.c。

```
void vmbus_set_event(struct vmbus_channel *channel)
{
u32 child_relid = channel->offermsg.child_relid;
if (!channel->is_dedicated_interrupt) {
/* Each u32 represents 32 channels */
sync_set_bit(child_relid & 31,
(unsigned long *)vmbus_connection.send_int_page +
(child_relid >> 5));
}
hv_do_hypercall(HVCALL_SIGNAL_EVENT, channel->sig_event, NULL);
}

```

         从上面的代码可以看出，vmbus_set_event 函数继续调用 hv_do_hypercall 函数来通知宿主机。hv_do_hypercall 函数代码如下，文件位置为./linux-4.7.2/drivers/hv/hv.c。

```
u64 hv_do_hypercall(u64 control, void *input, void *output)
{
u64 input_address = (input) ? virt_to_phys(input) : 0;
u64 output_address = (output) ? virt_to_phys(output) : 0;
void *hypercall_page = hv_context.hypercall_page;
#ifdef CONFIG_X86_64
u64 hv_status = 0;
 
if (!hypercall_page)
return (u64)ULLONG_MAX;
 
__asm__ __volatile__("mov %0, %%r8" : : "r" (output_address) : "r8");
__asm__ __volatile__("call *%3" : "=a" (hv_status) :
"c" (control), "d" (input_address),
"m" (hypercall_page));
 
return hv_status;
 
#else
 
u32 control_hi = control >> 32;
u32 control_lo = control & 0xFFFFFFFF;
u32 hv_status_hi = 1;
u32 hv_status_lo = 1;
u32 input_address_hi = input_address >> 32;
u32 input_address_lo = input_address & 0xFFFFFFFF;
u32 output_address_hi = output_address >> 32;
u32 output_address_lo = output_address & 0xFFFFFFFF;
 
if (!hypercall_page)
return (u64)ULLONG_MAX;
 
__asm__ __volatile__ ("call *%8" : "=d"(hv_status_hi),
"=a"(hv_status_lo) : "d" (control_hi),
"a" (control_lo), "b" (input_address_hi),
"c" (input_address_lo), "D"(output_address_hi),
"S"(output_address_lo), "m" (hypercall_page));
 
return hv_status_lo | ((u64)hv_status_hi << 32);
#endif /* !x86_64 */
}

```

         上面的代码中，有一句嵌入汇编代码，直接运行了汇编语句 call hypercall_page，hypercall_page 是由 hv_context.hypercall_page 赋值而来，下面我们来探寻 hypercall_page 的初始化过程和其中的内容。在 hv_init 函数中，完成了对 hypercall_page 的初始化，代码片段如下，文件位置为./linux-4.7.2/drivers/hv/hv.c。

```
int hv_init(void)
{
  ......
  /* See if the hypercall page is already set */
  rdmsrl(HV_X64_MSR_HYPERCALL, hypercall_msr.as_uint64);
 
  virtaddr = __vmalloc(PAGE_SIZE, GFP_KERNEL, PAGE_KERNEL_EXEC);
 
  if (!virtaddr)
    goto cleanup;
 
  hypercall_msr.enable = 1;
  hypercall_msr.guest_physical_address = vmalloc_to_pfn(virtaddr);
  wrmsrl(HV_X64_MSR_HYPERCALL, hypercall_msr.as_uint64);
 
  /* Confirm that hypercall page did get setup. */
  hypercall_msr.as_uint64 = 0;
  rdmsrl(HV_X64_MSR_HYPERCALL, hypercall_msr.as_uint64);
 
  if (!hypercall_msr.enable)
    goto cleanup;
 
  hv_context.hypercall_page = virtaddr;
 
  ......
 
  return 0;
 
cleanup:
  if (virtaddr) {
    if (hypercall_msr.enable) {
      hypercall_msr.as_uint64 = 0;
      wrmsrl(HV_X64_MSR_HYPERCALL, hypercall_msr.as_uint64);
    }
    vfree(virtaddr);
  }
  return -ENOTSUPP;
}

```

       

        其中，hypercall_msr 结构如下。

```
union hv_x64_msr_hypercall_contents {
u64 as_uint64;
struct {
u64 enable:1;
u64 reserved:11;
u64 guest_physical_address:52;
};
};

```

         由上面代码可知，hypercall_page 是由读写 msr 寄存器来告知宿主机虚拟机中的 hypercall_page 的位置。虚拟机内核在读写 msr 寄存器时，实际上每次读写都被 hypervisor 层的代码截获，然后通过读写的内容进行不同的操作，这里便是将 hypercall_page 指向的内存写入数据，写入的数据如下。

```
0x0    0f01c1          vmcall
0x3    c3              ret
0x4    8bc8            mov     ecx,eax
0x6    b811000000      mov     eax,11h
0xb    0f01c1          vmcall
0xe    c3              ret
......

```

         从 hypercall_page 中过的数据可知，hv_do_hypercall 函数实际上是调用了 vmcall 汇编指令，产生 VM-Exit 事件，陷入 hypervisor 层进行处理，hypervisor 层处理完成后再通知宿主机数据到来，宿主机驱动再对虚拟机数据进行处理。也就是说，每次调用 vmbus_set_event 函数最终都会运行 vmcall 汇编指令陷入 hypervisor 层进行处理。

         综上，虚拟机收发数据都是通过 VMBus 进行传输的，并且需要 hypervisor 层处理和转发。以上便是虚拟机内核层的数据收发过程。

______________________________________________________________________________________________

本文如需引用转载请联系本文作者，看雪 ID：ifyou

[安卓应用层抓包通杀脚本发布！《高研班》2021 年 3 月班开始招生！](https://bbs.pediy.com/thread-264283.htm)

上传的附件：

*   [4.1.pdf](javascript:void(0)) （1.03MB，92 次下载）
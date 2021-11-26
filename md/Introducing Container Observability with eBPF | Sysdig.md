> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [sysdig.com](https://sysdig.com/blog/introducing-container-observability-with-ebpf-and-sysdig/)

> Sysdig extends container visibility and security with eBPF. This helps more enterprises successfully ......

Today [we’ve announced](https://sysdig.com/press-releases/sysdig-introduces-ebpf-instrumentation-to-extend-cloud-native-visibility-and-security-to-container-optimized-linux-platforms/) that we’ve officially added eBPF instrumentation to extend container observability with Sysdig monitoring, security and forensics solutions. eBPF – extended Berkeley Packet Filter – is a Linux-native in-kernel virtual machine that enables secure, low-overhead tracing for application performance and event observability and analysis. Don’t let the name fool you – eBPF delivers a lot more than network packet information (more on that below). Sysdig now taps into eBPF to offer the deep visibility for cloud-native and container environments – from host and network data to container processes, resource utilization, and more. This is something we are already well-known for. Now we are taking advantage of eBPF to expand how and where we provide container observability.

Why eBPF observability?
-----------------------

As our own Gianluca Borello explains in part one of his [eBPF blog series](https://sysdig.com/blog/sysdig-and-falco-now-powered-by-ebpf) released today, there are several motivations for working with eBPF as an alternative instrumentation model for container observability. One big one for us is the advent of container-optimized operating systems. These solutions like [Container-Optimized OS (COS)](https://cloud.google.com/container-optimized-os/) from Google Cloud Platform and [Project Atomic Host](https://www.projectatomic.io/) led by Red Hat, which pre-install container runtimes and Kubernetes components, feature an immutable infrastructure approach designed with a minimal footprint to enhance operational security and scale. In addition, these solutions disallow kernel modules altogether. How then do we perform The Sysdig magic of deriving deep data for monitoring and securing these platforms?  
The answer: eBPF.

Because eBPF is simply a part of Linux, it gives us a native entry point for extracting metrics and data from system calls (what we’ve always done), without the need for a kernel module. With eBPF, we simply run small programs quickly and safely inside the kernel. Data is collected in user space, aggregated, and passed to the backend to be stored and made available to users.

A little eBPF history
---------------------

eBPF is the “next-generation” of the Berkeley Packet Filter (BPF). BPF dates back to 1992 with [work done by Steve McCanne and Van Jacobson](http://www.tcpdump.org/papers/bpf-usenix93.pdf) at Lawrence Berkeley Laboratory. BPF was essentially designed for network packet filtering with Unix operating systems. eBPF originated with Linux in 2014 – first included with kernel version 3.18. eBPF took BPF’s concepts to new levels, adapting the approach to deliver more efficiency and also taking advantage of newer generations of hardware. Most importantly, with hooks all over the kernel, eBPF enables use cases well beyond packet filtering. This includes tracing, performance analysis, debugging, security, and more.

Today, there’s a lot of excitement about eBPF in the Linux and cloud community. Proponents in the industry are saying some great things about its capabilities. Here’s a sampling:

*   Brendan Gregg, Senior Performance Architect at NetFlix, describes eBPF as having “[superpowers](http://www.brendangregg.com/blog/2016-03-05/linux-bpf-superpowers.html).”
*   In late December, The New Stack published an article on eBPF titled: [Linux Technology for the New Year: eBPF](https://thenewstack.io/linux-technology-for-the-new-year-ebpf/)
*   Active eBPF contributor, David S. Miller, tweeted his New Year greeting, “[World domination for eBPF in 2019](https://twitter.com/davem_dokebi/status/1079998472235827200).”
*   And my favorite, a great eBPF talk by Jeff Dileo and Andy Olson, security consultants at NCC Group, carried the title on [Kernel Tracing with eBPF – Unlocking God Mode on Linux.”](https://media.ccc.de/v/35c3-9532-kernel_tracing_with_ebpf)

Taking advantage of eBPF with Sysdig
------------------------------------

The Linux world loves eBPF – and Sysdig too! Through our work with eBPF, we’ve been able to [contribute](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?qt=author&q=gianluca+borello) to the project upstream. The culmination of our work with eBPF is a set of Sysdig-engineered eBPF programs featured in our open source [sysdig](https://github.com/draios/sysdig) and [Falco](https://falco.org/) solutions, as well as our unified agent that serves [Sysdig Monitor](https://sysdig.com/products/monitor/) and [Sysdig Secure](https://sysdig.com/products/secure/). These eBPF programs are now a part of [ContainerVision](https://sysdig.com/platform/#howitworks), our instrumentation approach that leverages kernel system calls to deliver container insights. System calls are a data source as rich as logs – but in real-time. For a new generation of cloud-native platforms, eBPF is now the tap into this deep data source.

![](https://sysdig.com/wp-content/uploads/2019/02/ebpf-containervision.png)

It’s important to mention that our “classic” kernel module approach is alive and well across our open source and commercial offerings. It continues to be our go-to approach for instrumentation. You can think of eBPF as a step into a new option that expands the universe of coverage for our solutions – no cloud left behind!

What can you expect? If you’re working with a container-based environments, regardless of the instrumentation model, with Sysdig you’ll gain deep visibility into your container and Kubernetes infrastructure. We know containers and orchestration can get complex, fast. We help take the sting out of performance monitoring, security, vulnerability management, troubleshooting, and forensics for modern environments. As I like to say, we can help you see more, solve faster, and save money (and perhaps sleep more soundly!).

Conclusion
----------

We’re looking forward to advancing along with the eBPF capabilities in every Linux release. By providing the link to eBPF for observability, we’re able to help more enterprises successfully build and run applications on containers – and to respond quickly to any issues that pop up for fast resolution. For a deep dive into the fascinating technology of eBPF and our work with it – click over to Gianluca’s blog – [Sysdig and Falco now powered by eBPF](https://sysdig.com/blog/sysdig-and-falco-now-powered-by-ebpf) – and have a read.

**Want to hear more?** Join me and Sysdig product manager, Narayan Iyengar, next week for our live webinar: [Using eBPF for Container Monitoring, Security, and Forensics](https://bit.ly/2IFbDMi).

Post navigation
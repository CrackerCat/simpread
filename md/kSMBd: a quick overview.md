> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.thalium.re](https://blog.thalium.re/posts/ksmbd-trailer/)

> In this blogpost, we introduce the analysis of one SMB implementation: kSMBd. It will be followed up ......

### Table of Contents

*   [Introduction](#introduction)
*   [kSMBd](#ksmbd)
    *   [Presentation](#presentation)
    *   [Architecture overview](#architecture-overview)
    *   [Main attack surfaces](#attack-surfaces)
        *   [SMB core (kernel): compound request](#smb-core-kernel--compound-request)
        *   [SMB feature (kernel): VFS](#smb-feature-kernel--vfs)
        *   [SMB feature (user): MSRPC](#smb-feature-user--dcerpc)
*   [Generic SMB tooling](#generic-smb-tooling)
*   [Bugs identified](#bugs-identified)
*   [Disclosure timeline](#disclosure-timeline)
*   [Conclusion](#conclusion)

Nowadays, it is usual for a company to expose a file sharing service. Whether exposed on the Internet or on a LAN, these services are among the most commonly deployed.

Given the criticality of such services and the attack surface they expose, the security researchers’ interest to break through it is quite important. For instance, the Pwn2Own competition offers a good number of NAS system entries each year.

In order to serve files, several protocols have been elaborated:

*   NFS (Network File System)
*   SMB (Server Message Block)
*   AFP (Apple File Protocol)
*   9P (Plan 9 Filesystem Protocol)
*   etc.

Their usage may vary depending on the ecosystem around the file sharing service and/or its exposition.

For each of these protocols, multiple implementations exist:

<table><thead><tr><th>Protocol</th><th>Implementations</th></tr></thead><tbody><tr><td>NFS</td><td>Windows, Linux</td></tr><tr><td>SMB</td><td>Windows, samba, Linux (wink wink)</td></tr><tr><td>AFP</td><td>Netatalk</td></tr><tr><td>9P</td><td>Qemu, WSL</td></tr></tbody></table>

In this blogpost, we introduce the analysis of one SMB implementation: kSMBd. It will be followed up by a talk @ OffensiveCon 2023 named [Abusing Linux in-kernel SMB server to gain kernel remote code execution](https://www.offensivecon.org/speakers/2023/guillaume-teissier-and-quentin-minster.html).

At the end of 2021, an [LWN article](https://lwn.net/Articles/871866/) caught our attention. A new SMB server implementation was being actively developed and on top of that, it was an in-kernel Linux implementation. Thus, our spring break was booked.

Presentation
------------

[kSMBd](https://github.com/namjaejeon/ksmbd) is a Linux kernel module that implements the SMB/CIFS protocol. It is developed by Namjae Jeon and is open source. This module has eventually been pulled upstream in the Linux kernel. As a consequence, it is shipped by distributions, such as [Ubuntu](https://launchpad.net/ubuntu/jammy/+package/ksmbd-tools) or [OpenWrt](https://openwrt.org/packages/pkgdata/ksmbd-server). The main objective behind this new implementation and the reason why it resides in the kernel appears to be _performance_, by keeping most of the treatments inside the kernel, saving your precious CPU cycles.

Most of the publications related to kSMBd can be found on the [SambaXP conference website](https://sambaxp.org/). These talks were good starting points to understand where the project was going and what features we could expect.

kSMBd supports a [lot of features](https://winprotocoldoc.blob.core.windows.net/productionwindowsarchives/MS-SMB/%5BMS-SMB%5D.pdf) that make it a very attractive project, such as:

*   multiple dialects: SMB2.1 / SMB3.0 / SMB3.1.1
*   oplock cache mechanism
*   compound request
*   SMB Direct (RDMA)
*   ACL
*   DCE/RPC

This kernel module supports Linux kernel 5.4 or later, and was incorporated into the mainline kernel with the 5.15 release.

Architecture overview
---------------------

From a high-level perspective, the kSMBd architecture is split between user-land and kernel-land. Indeed, even if the aim of kSMBd is to increase performance by handling most of the file sharing service in the kernel, some components still reside in user-land.

The user-land component (ksmbd-tools) implements DCE/RPC endpoints such as:

*   `rpc_lsarpc`: Local Security Authority (Domain Policy) Remote Protocol
*   `rpc_samr`: Security Account Manager (SAM) Remote Protocol
*   `rpc_srvsvc`: Server Service Remote Protocol
*   etc.

On the other hand, the kernel-land module is focused on the protocol handling and the VFS compatibility.

Between user-land and kernel-land components, netlink is used as IPC and their interaction can be shown as below:

[![](https://blog.thalium.re/posts/img/trailer-archi.png)](https://blog.thalium.re/posts/img/trailer-archi.png)kSMBd high level architecture

Attack surfaces
---------------

From our perspective, we identified three major attack surfaces in the kSMBd project to look at. This is clearly not an exhaustive list.

The first and most straightforward attack surface is the SMB protocol handler. This portion of code handles incoming SMB messages from the network and will maintain the induced state machine. This is the core of the server and regardless of the attack surface that is addressed, it is important to understand roughly how it works.

Another attack surface that is readily apparent is the virtual file system (VFS). The VFS implements the interaction with the host file system and might expose features such as ACL, streams and so on. In many cases, it provides a compatibility layer for the remote client.

Last but not least, the SMB protocol is a complex beast which not only allows remote users to tinker with files and folders but also to call Remote Procedure Call routines. Those RPC interfaces allow clients to request information about the remote system, such as listing the users or shares present on the SMB server. This is also a compatibility feature coming from the Windows world. From the three nominated attack surfaces, this is the only one implemented in user-space.

These three attack surfaces have been addressed during the analysis.

### SMB core (kernel): compound request

From the first attack surface, we decided to analyze the SMB compound request handling.

Compound requests in the SMB protocol are a way for a client to group multiple SMB commands into a single SMB request. This can improve performance by reducing the number of network round-trips needed to complete a series of operations.

Each compound request contains a series of commands, which are processed sequentially by the server. The responses for each command are then combined into a single response message, which is sent back to the client.

Compound requests are used extensively in SMB, particularly for file and print operations, where multiple commands are often required to complete a given operation. By grouping these commands together, SMB is able to provide improved performance and reduced network overhead.

### SMB feature (kernel): VFS

A VFS is an abstraction layer that sits between the file system and the client application, providing a unified view of the file system regardless of the underlying storage device.

In the context of SMB, a VFS is implemented on the server side and allows the server to present a consistent view of the file system to SMB clients, regardless of the underlying file system or storage technology being used. This is particularly useful in heterogeneous environments where different clients may be accessing the same file system using different protocols or operating systems.

The SMB VFS provides a number of features and benefits, including:

1.  Cross-platform support: The VFS allows SMB servers to provide a consistent view of the file system to clients running on different operating systems.
2.  Security: The VFS can enforce access control policies and other security measures to ensure that only authorized users have access to files and directories.
3.  Performance optimization: The VFS can perform caching and other optimizations to improve the performance of file system operations.
4.  Extensibility: The VFS can be extended to support additional file system features and functionality, such as encryption, compression, or versioning.

Overall, the SMB VFS is a key component of the protocol that enables servers to provide a flexible and robust file sharing solution to clients across a wide range of environments and use cases.

### SMB feature (user): DCE/RPC

DCE/RPC stands for Distributed Computing Environment/Remote Procedure Call. In the SMB protocol, DCE/RPC is used as a means of communication between the client and server for performing various network operations.

In the context of SMB, DCE/RPC is used to enable a wide range of network operations, including file and printer sharing, authentication, and other administrative tasks. For example, when a client requests to access a file on a remote SMB server, the client sends an RPC request to the server using the DCE/RPC protocol. The server then processes the request and sends a response back to the client.

DCE/RPC is a critical component of the SMB protocol, as it enables efficient and secure communication between the client and server.

To avoid reinventing the wheel, we identified several tools and libraries that helped us during the analysis:

*   [smbtorture](https://wiki.samba.org/index.php/Writing_Torture_Tests): Samba’s SMB test framework
*   [smbclient](https://www.samba.org/samba/docs/current/man-html/smbclient.1.html): Samba’s SMB/CIFS client
*   [impacket](https://github.com/fortra/impacket/blob/master/impacket/smb.py): a very good Python library to play with network protocols
*   [aioSMB](https://github.com/skelsec/aiosmb): a pure-Python, fully asynchronous SMB library, inspired by impacket
*   [SMB_Fuzzer](https://github.com/mellowCS/SMB_Fuzzer): an SMB fuzzer

Our work session ended up with the discovery of 5 vulnerabilities affecting the kernel module and 5 others affecting the user-land daemon. All the vulnerabilities where reported through the [Zero Day Initiave](https://www.zerodayinitiative.com/advisories/published/) program.

<table><thead><tr><th>ZDI</th><th>CVE</th><th>Location</th><th>CVSS</th><th>Description</th></tr></thead><tbody><tr><td>ZDI-22-1687/ZDI-CAN-17815</td><td>CVE-2022-47941</td><td>Kernel</td><td>5.3</td><td>Linux Kernel ksmbd Memory Exhaustion Denial-of-Service Vulnerability</td></tr><tr><td>ZDI-22-1688/ZDI-CAN-17771</td><td>CVE-2022-47942</td><td>Kernel</td><td>8.5</td><td>Linux Kernel ksmbd Heap-based Buffer Overflow Remote Code Execution Vulnerability</td></tr><tr><td>ZDI-22-1689/ZDI-CAN-17818</td><td>CVE-2022-47938</td><td>Kernel</td><td>6.5</td><td>Linux Kernel ksmbd Out-Of-Bounds Read Denial-of-Service Vulnerability</td></tr><tr><td>ZDI-22-1690/ZDI-CAN-17816</td><td>CVE-2022-47939</td><td>Kernel</td><td>10.0</td><td>Linux Kernel ksmbd Use-After-Free Remote Code Execution Vulnerability</td></tr><tr><td>ZDI-22-1691/ZDI-CAN-17817</td><td>CVE-2022-47943</td><td>Kernel</td><td>9.6</td><td>Linux Kernel ksmbd Out-Of-Bounds Read Information Disclosure Vulnerability</td></tr><tr><td>ZDI-CAN-17770</td><td>not yet defined</td><td>User</td><td>not yet defined</td><td>not yet defined</td></tr><tr><td>ZDI-CAN-17820</td><td>not yet defined</td><td>User</td><td>not yet defined</td><td>not yet defined</td></tr><tr><td>ZDI-CAN-17821</td><td>not yet defined</td><td>User</td><td>not yet defined</td><td>not yet defined</td></tr><tr><td>ZDI-CAN-17823</td><td>not yet defined</td><td>User</td><td>not yet defined</td><td>not yet defined</td></tr><tr><td>ZDI-CAN-17822</td><td>not yet defined</td><td>User</td><td>not yet defined</td><td>not yet defined</td></tr></tbody></table>

Please note that the details regarding the user-land daemon vulnerabilities have not been published yet.

*   **2022-07-26**: Vulnerabilities reported to ZDI
*   **2022-07-29**: [CVE-2022-47941](https://github.com/namjaejeon/ksmbd/commit/0289a91c4d9371e9d04d5631fd2640f3c6019ae7), [CVE-2022-47938](https://github.com/namjaejeon/ksmbd/commit/64ccf4b8eb3804f6cb0fbcf14eaf2859abda62a2), [CVE-2022-47939](https://github.com/namjaejeon/ksmbd/commit/c88d9195ac11b947a9f5c4347f545f352472de0a), [CVE-2022-47943](https://github.com/namjaejeon/ksmbd/commit/bb29d2b46452206fa33f8b0d8845e7e7ba4a6bd9) are fixed in the kSMBd project
*   **2022-08-01**: [CVE-2022-47941](https://github.com/torvalds/linux/commit/aa7253c2393f6dcd6a1468b0792f6da76edad917), [CVE-2022-47938](https://github.com/torvalds/linux/commit/824d4f64c20093275f72fc8101394d75ff6a249e), [CVE-2022-47939](https://github.com/torvalds/linux/commit/cf6531d98190fa2cf92a6d8bbc8af0a4740a223c), [CVE-2022-47943](https://github.com/torvalds/linux/commit/ac60778b87e45576d7bfdbd6f53df902654e6f09) are fixed in the kernel
*   **2022-08-02**: [CVE-2022-47942](https://github.com/namjaejeon/ksmbd/commit/ac478bb574592c326b8c784c2aec9f170c53f095) is fixed in the kSMBd project
*   **2022-08-04**: [CVE-2022-47942](https://github.com/torvalds/linux/commit/8f0541186e9ad1b62accc9519cc2b7a7240272a7) is fixed in the kernel
*   **2022-08-17**: [Release](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tag/?h=v5.15.61) of the Linux kernel version [5.15.61](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/1990162) (CVE-2022-47941 & CVE-2022-47938 & CVE-2022-47939)
*   **2022-08-21**: [Release](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/tag/?h=v5.15.62) of the Linux kernel version [5.15.62](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/1990554) (CVE-2022-47943 & CVE-2022-47942)
*   **2022-10-17**: Fix in the kernel shipped by [Ubuntu 22.04](https://launchpad.net/ubuntu/+source/linux/5.15.0-53.59)
*   **2022-12-22/23**: Coordinated public release of advisory (~Merry Christmas 🎅~)
*   **2023-01-23**: Advisory updated
*   **2023-05-19**: Talk @ OffensiveCon

It goes without saying that the date chosen for the public release was out of our range of actions.

This article briefly presented the results of our analysis of kSMBd. The exploitation of some of the kernel vulnerabilities, resulting in a **kernel RCE**, will be presented at OffensiveCon 2023.
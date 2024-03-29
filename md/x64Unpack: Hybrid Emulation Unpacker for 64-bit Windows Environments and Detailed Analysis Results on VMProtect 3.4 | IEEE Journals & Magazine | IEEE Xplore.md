> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [ieeexplore.ieee.org](https://ieeexplore.ieee.org/document/9139515?denied=)

> In spite of recent remarkable advances in binary code analysis, malware developers are still using co......

![](https://ieeexplore.ieee.org/ielx7/6287639/8948470/9139515/graphical_abstract/access-gagraphic-3008900.jpg)

Memory layout of x64Unpack.

**Abstract:**In spite of recent remarkable advances in binary code analysis, malware developers are still using complex anti-reversing techniques to make analysis difficult. To protec...

View more

**Metadata**

**Abstract:**

In spite of recent remarkable advances in binary code analysis, malware developers are still using complex anti-reversing techniques to make analysis difficult. To protect malware, they use packers, which are (commercial) tools that contain various anti-reverse engineering techniques such as code encryption, anti-debugging, and code virtualization. In this paper, we present x64Unpack: a hybrid emulation scheme that makes it easier to analyze packed executable files and automatically unpacks them in 64-bit Windows environments. The most distinguishable feature of x64Unpack compared to other dynamic analysis tools is that x64Unpack and the target program share virtual memory to support both instruction emulation and direct execution. Emulation runs slow but provides detailed information, whereas direct execution of the code chunk runs very fast and can handle complex cases regarding to operating systems or hardware devices. With x64Unpack, we can monitor major API (Application Programming Interface) function calls or conduct fine-grained analysis at the instruction-level. Furthermore, x64Unpack can detect anti-debugging code chunks, dump memory, and unpack the packed files. To verify the effectiveness of x64Unpack, experiments were conducted on the obfuscation tools: UPX 3.95, MPRESS 2.19, Themida 2.4.6, and VMProtect 3.4. Especially, VMProtect and Themida are considered as some of the most complex commercial packers in 64-bit Windows environments. Experimental results show that x64Unpack correctly emulates the packed executable files and successfully produces the unpacked version. Based on this, we provide the detailed analysis results on the obfuscated executable file that was generated by VMProtect 3.4.

**Date of Publication:** 13 July 2020[](http://ieeexplore.ieee.org/Xplorehelp/Help_Pubdates.html "Get help with using Publication Dates")

**Electronic ISSN:** 2169-3536

**INSPEC Accession Number:** 19800573

Publisher: IEEE

![](https://ieeexplore.ieee.org/ielx7/6287639/8948470/9139515/graphical_abstract/access-gagraphic-3008900.jpg)

Memory layout of x64Unpack.

* * *
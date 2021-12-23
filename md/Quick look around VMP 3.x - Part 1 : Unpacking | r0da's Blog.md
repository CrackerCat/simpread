> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [whereisr0da.github.io](https://whereisr0da.github.io/blog/posts/2021-01-05-vmp-1/)

> Hi This is my exploration around VMProtect security. VMP is a well known protection with a lot of fea......

Hi

This is my exploration around VMProtect security. VMP is a well known protection with a lot of features, main ones are Code Mutation and Virtualization, and compared to them, this part is the simplest regarding VMP. I will talk about all of those in future posts, but now I will focuse myself on the Packing and the Import Obfuscation.

![](https://whereisr0da.github.io/blog/post_images/vmp_2/Screenshot_677.png)

Unpacking
---------

Packing is about compressing / ciphering executable’s sections to prevent static analysis. It’s not very effective considering that the code and the sections should be deciphered at some point during the execution.

![](https://whereisr0da.github.io/blog/post_images/vmp_2/Screenshot_675.png)

In this case, VMProtect doesn’t store the real “RawFile” sections informations in the PE header, which is a good thing. But.. the virtual addresses and size should be stored in the header so the kernel could alloc the correct size for the executable, so we can still notice sections sizes and possible addresses (without considering ASLR).

Regarding the packing in raw file, everything is melt in one of the VMP section `.vmp1` (ciphered sections, unpacking routine, section informations, ..)

The only section that is not “protected” is `.rsrc` because Windows need to read it to extract icons and other informations to display on properties menu. VMProtect has an option to protect resources regarding this, so it could split resources in two parts, one readable for Windows. And one containing only “program’s data”, ciphered, and deciphered during the execution of the program.

As we can see on the screenshot, unless `.data` section, all sections are not writable. So VMP will have a to call `VirtualProtect` (or something similar) to change the section’s attributes in order to change them. In fact, VMP used `VirtualProtect` in the past, but since 3.x, it use an higher “undocumented” kernel API called `ZwProtectVirtualMemory` which has the same purpose.

So now we can start dynamic unpacking ! Considering that `ZwProtectVirtualMemory` is called a lot of times by Windows internals, we will put a breakpoint on it only when the the entrypoint hit.

![](https://whereisr0da.github.io/blog/post_images/vmp_2/Screenshot_189.png)

Note that the entrypoint looks like a VMProtect virtualized routine, and in fact, it is ! In the past (version 2.x), VMP unpacking routine were not virtualized ([like in this video](https://www.youtube.com/watch?v=aoa89Khfgr0)), so it was easy to rush and breakpoint the jump to OEP (`push` and `ret`).

![](https://whereisr0da.github.io/blog/post_images/vmp_2/Screenshot_190.png)

Ok, now we know that VMP should modify section’s attributs one time to write, and one time to restore section’s attributs. After a little bit of hit, here is the number of `ZwProtectVirtualMemory` calls we should get.

```
n = count of section that are not writable
n + 1 (.vmp0) : change protect to writable flag
1             : remove COPY flag
n + 1 (.vmp0) : restore original flags


```

So `((n + 1) * 2) + 1` to get the PE unpacked, here is a GIF of this in action (see the flags on the right) :

![](https://whereisr0da.github.io/blog/post_images/vmp_2/44.gif)

And from here, we can just dump the executable with any tool to get a clear dump (without import fixed) of the PE.

From that (and the part 2 of my research around VMP), I will assume that `.vmp0` section contain virtualized / mutated programer’s code (and IAT related code), and `.vmp1` contain all things linked to the unpacking process (ciphered sections, virtualized unpacking routine, section informations, ..)

Finding the OEP
---------------

As the unpacking routine is virtualized, it’s pretty hard to find it from VMP code. So we need to do it with some tricks. In this case I though it could work if I monitor the EIP and store it when it switch from `.vmp1` to another section which is not `.vmp0`, and it worked. I tried a Qiling script for this, but as Qiling doesn’t implements `ZwProtectVirtualMemory` I can’t do much. So I did an unicorn python script to accomplish this.

We can find the same result by putting an hardware memory breakpoint about `execution` on the `.text` section. In my case, the OEP was the first function on top of `.text`, but it’s a rare case, so you will probably never meat this one.

You can spot the OEP by looking at the first function stack cookie ([like in this very good article](https://baboonrce.github.io/category/unpacking.html)) which as the value `0x2B992DDFA232` when the executable is compiled using VC++.

You can still find the OEP by hand, by trying to go on the bottom of the stack to spot the possible first return address.

IAT Obfusaction
---------------

VMP’s IAT obfuscation is in option, and not set by default. So IAT reconstruction is not necessary when the dev don’t know how to use VMP (and trust me, it happens sometimes). Of course, I turned this ON for my post :)

First thing to see is that the original IAT is still saved, but not used.

![](https://whereisr0da.github.io/blog/post_images/vmp_2/Screenshot_676.png)

So you can see what API the program is using, but you can’t link them to the code (cross references) because import addresses are “calculated” runtime. In my point of view is that they should import one random function of the DLL if they want to keep the IAT clear like that, not just leave everything in place. Or just completly remove the IAT content and load each DLL with LoadLibrary and resolve imports later.

So if we take a look at a call to an API in the executable, we can notice that luckily, the API call’s are only mutated, not virtualized !

Each call to IAT in the `.text`, exemple : `call dword ptr ds:[<&CreateProcessW>]`, is 6 bytes long.

VMP changed each API call to this kind of call :

```
push random_register
call mutated_api_resolver 


```

Thoses two instructions are also about 6 bytes in order to keep the code “alignment”. There is one `mutated_api_resolver` function for each API call, it could explain why VMP makes such big outputs (also because of virtualization). So VMP use a register `random_register` to pass something to the api resolver.

Here is a mutated version (edi is random_register) :

NOTE : this code is, like I said, in the `.vmp0` section

```
nop
not di
bswap di
jmp ...
pop edi
jmp ...
xchg dword ptr ss:[esp], edi 
push edi
not edi 
xchg di, di 
jmp ...
mov edi, 0x401113
mov edi, dword ptr ds:[edi + 0x2B21E] 
jmp ...
lea edi, dword ptr ds:[edi + 0x724F2141] 
jmp ...
xchg dword ptr ss:[esp], edi
jmp ...  
ret 


```

And here is a cleaned version (reg is random_register) :

```
# reg is the pushed register
# grab the return address of the call
pop reg
# swap the pushed "reg" and the return address of the call
# so the return of the API call will return on the API caller
xchg dword ptr ss:[esp], reg 
# setup future return address to jump to the API function
push reg 
# calc the function address
# offset to CreateProcessW in kernel32.dll in VMP calculation
mov reg, 0x401113 
# calc the function address from offset
# those values are static IAT and API offsets
# get the VMP kernel32.dll IAT offset
mov reg, dword ptr ds:[reg + 0x2B21E]
# get the VMP CreateProcessW address from kernel32.dll IAT offset
lea reg, dword ptr ds:[reg + 0x724F2141]
# put the API function address on stack top, using the "push reg" above
xchg dword ptr ss:[esp], reg  
# jump to the API function
ret 


```

So to summerise, VMP setup a variable on the stack to jump to the next API address by doing a `push` and `ret`.

This address is calculated by :

```
reg = 0x401113        : offset of CreateProcessW + kernel32.dll
ds:[reg + 0x2B21E]    : address of kernel32.dll IAT in VMP
ds:[reg + 0x724F2141] : address of CreateProcessW


```

I don’t have enough time to code an import fixer, but it’s doable runtime using emulation :) (unicorn, ..).

Some tools were made by 0xnobody, can1357 and mrxodia to unpack and fix imports in x64 ([here](https://github.com/0xnobody/vmpdump)).
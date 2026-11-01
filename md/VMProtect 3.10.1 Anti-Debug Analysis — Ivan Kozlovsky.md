> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [ivankozlovsky.com](https://ivankozlovsky.com/blog/vmprotect-3101-anti-debug-analysis)

> Comparison with 3.9.4 across 27 API cases and 13 saved-context cases: matching tested predicates desp......

same small x64 test program, this time protected with 3.10.1  
user-mode + kernel-mode selected, the other protection options off  
x64dbg for the native execution, Binary Ninja for the code and Unicorn for controlled inputs

the tested startup checks match 3.9.4  
the TitanHide action, debug object counters and thread-hide query are still present  
the exception filter still selects breakpoint addresses through DR7

earlier results were changed in the running process to reach the later checks  
the screenshots below are real native observations, while the condition tables describe emulated inputs  
addresses in the text are RVAs

the PEB byte
------------

the reader at `710E1A` loads through `[r10+rdx-28AF61DFh]`  
RDX is PEB+2 and R10 is 28AF61DFh at this point, so the effective address is `BeingDebugged`  
under x64dbg the consumed value is 1

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/01-01_peb.png)

the same instruction in Binary Ninja

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/02-02_bn_peb.png)

for the continuation i cleared the consumed CX value and kept the upper RCX bits  
the actual PEB byte stayed unchanged

the instrumentation callback
----------------------------

`NtSetInformationProcess` receives class `28h`, ProcessInstrumentationCallback  
the input is 16 zero bytes and the native call succeeds  
this clears the callback, it is an observed startup action rather than a separate rejection condition established by these tests

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/03-03_callback.png)

the TitanHide device
--------------------

this follows the same path as 3.9.4  
`NtOpenFile` tries the exact device path `\??\TitanHide` with desired access `C0100080h`

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/04-04_titan.png)

there was no TitanHide device in the native run  
the real return is `C0000034`, STATUS_OBJECT_NAME_NOT_FOUND

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/05-05_titan_status.png)

to follow the other branch i supplied a successful open in the emulator  
the protected code then builds a 12-byte NtWriteFile input

```
02 00 00 00  00 00 00 00  00 00 00 00
Command = 2
Type    = 0
Pid     = 0
```

in [TitanHide's HIDE_INFO layout](https://github.com/mrexodia/TitanHide/blob/master/TitanHide/TitanHide.h) command 2 is UnhideAll  
the [handler](https://github.com/mrexodia/TitanHide/blob/master/TitanHide/hider.cpp) clears the hiding entries  
so this is an attempt to undo hiding, not just a name comparison followed by a warning

modeled open statuses 0 and 1 take the write path  
modeled access denied skips it  
both a successful write and a failed write with C0000008 continue in the tested cases  
the positive-open branch was emulated, no write to a real TitanHide driver was performed

the first debug port
--------------------

`ProcessDebugPort` still uses class 7, the current process and an 8-byte output  
the native call succeeds and writes `FFFFFFFFFFFFFFFF`  
zero was supplied for the controlled continuation

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/06-06_port.png)

the overlapping process query
-----------------------------

`ProcessDebugObjectHandle` uses class `1Eh`  
the output buffer and ReturnLength point to the same qword

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/07-07_overlap.png)

the native call succeeds and the final overlapped qword is 8  
continuation uses `C0000353`, STATUS_PORT_NOT_SET, with that qword retained

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/08-08_overlap_result.png)

the condition tests match 3.9.4 here  
a successful result or a zero qword reaches the warning

the misaligned output
---------------------

3.9.4 already used this alignment probe  
the handle here is -1, the output is unaligned, size is 8 and ReturnLength is NULL  
the class is still `ProcessDebugObjectHandle`, not ProcessBasicInformation

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/09-09_misaligned.png)

the real kernel returns `80000002`, STATUS_DATATYPE_MISALIGNMENT

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/10-10_misaligned_status.png)

supplying 0, C0000005 or C0000001 instead reaches the warning in the emulator  
the expected 80000002 continues

the two other malformed calls are still present  
ReturnLength is 1, once with length 8 and once with length 0

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/11-11_bad_length.png)

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/12-12_zero_length.png)

both native calls return C0000005 and leave the seed unchanged  
changing those two statuses to success in the model still continues  
these tests do not establish a separate rejection predicate for either call

creating a debug object
-----------------------

3.10.1 creates its own object with `NtCreateDebugObject`  
requested access is 8 and flags are 0

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/13-13_create.png)

this is a real created handle, A4h in the screenshot run

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/14-14_create_result.png)

next comes `NtQueryObject` with class `2`, ObjectTypeInformation  
this is a query for the created object's type, not ObjectTypesInformation over every type

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/15-15_object_size.png)

the first query has a NULL output and size 0  
the native result is C0000004 and the required size is 80h

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/16-16_size_status.png)

the code allocates that size and queries the type information  
the dump contains the DebugObject name and the two counters at offsets 10h and 14h  
this capture has A4h objects and 3 handles, these are live values rather than constants

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/17-17_type.png)

there is another query with ReturnLength at `output+8`  
that overlaps the low DWORD of UNICODE_STRING.Buffer  
after the native call it contains 80h

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/18-18_type_overlap.png)

then the created object is closed  
the controlled branch tests gave these results

<table><thead><tr><th>modeled input</th><th>result</th></tr></thead><tbody><tr><td>TotalNumberOfObjects = 0</td><td>warning</td></tr><tr><td>TotalNumberOfHandles = 0</td><td>warning</td></tr><tr><td>counts 1 / 1, or object count 2</td><td>continues</td></tr><tr><td>create fails</td><td>skips the object-query group</td></tr><tr><td>size query returns success or C0000005</td><td>skips the dependent data queries</td></tr><tr><td>first data query fails</td><td>skips the second data query</td></tr><tr><td>overlapping ReturnLength becomes zero</td><td>continues in the tested case</td></tr></tbody></table>

so a successful zero-length query is not automatically a detection in this sample  
the branch around the data queries matters too

checking that thread hiding worked
----------------------------------

`NtSetInformationThread` still uses class 11h, current thread -2, NULL input and size 0  
for the native trace the real call was given an invalid handle, then the original handle and a successful return were supplied  
this kept debugger events visible

as in 3.9.4, `NtQueryInformationThread` follows with the same class and a one-byte output

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/19-19_query_hide.png)

the native query returns success and byte 0 because the earlier hide was deliberately suppressed  
the byte was changed to 1 for continuation

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/20-20_hidden_result.png)

<table><thead><tr><th>modeled query result</th><th>result</th></tr></thead><tbody><tr><td>success, hidden byte 0</td><td>warning</td></tr><tr><td>success, hidden byte 1 or 2</td><td>continues</td></tr><tr><td>query fails</td><td>continues in the tested case</td></tr><tr><td>set fails, later query reports hidden = 1</td><td>continues in the tested case</td></tr></tbody></table>

kernel debugger and the driver list
-----------------------------------

`SystemKernelDebuggerInformation` is class 23h  
this sample uses a direct syscall here, so R10 contains the first argument at the syscall instruction

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/21-21_kernel_args.png)

the real two-byte result is `00 01`

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/22-22_kernel.png)

the tested predicate is still success and KernelDebuggerEnabled and not KernelDebuggerNotPresent  
success with 01 00 warns, while 00 00, 00 01 and 01 01 continue  
a failed query with 01 00 also continues in the model

3.8 queried SystemModuleInformation, class 0Bh, and compared sice.sys, siwvid.sys, ntice.sys, iceext.sys and syser.sys  
that query and five-name loop do not occur on the complete modeled 3.10.1 startup path traced here  
the pre-exception system query is class 23h, the later runtime query is class 0  
native execution also reached the later port check and original entry  
the same absence was recorded for 3.9.4  
this describes the supplied samples' traced paths, not every possible branch

the second port check
---------------------

a later ProcessDebugPort query uses a separate buffer  
the reader at `84982B` consumes the native -1 result in RDI  
the hardware access stop is immediately after it at 849830

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/23-23_second_port.png)

both the consumed value and its buffer were cleared for continuation  
another thread-hide call follows  
the two NtProtectVirtualMemory calls around this area change a region of the main image and restore its protection, they are not a patch of NtClose

invalid handle and the single-step exception
--------------------------------------------

the invalid-handle probe still calls `ntdll!NtClose` directly with DEADC0DE  
this matches 3.9.4; the older 3.8 path used kernel32!CloseHandle

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/24-24_close.png)

after swallowing the native C0000008 exception the next stop is a real single-step exception, 80000004  
POPFQ is at 89F70A, RDTSC at 89F70B and the stop is at the following NOP

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/25-25_tf.png)

the same sequence in Binary Ninja

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/26-26_bn_tf.png)

delivering the exception enters the application's filter at `89F800`

DR7 selects which addresses count
---------------------------------

the filter reads the saved CONTEXT, including ContextFlags at +30h  
DR0 starts at +48h, followed by DR1, DR2, DR3, DR6 and DR7

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/27-27_context.png)

3.8 combined the four saved breakpoint addresses directly in the tested filter  
3.9.4 and 3.10.1 both read the low byte of DR7 at +70h before selecting each address

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/28-28_bn_dr7.png)

the same byte read in the decompiled view  
the first expression loads a byte from arg2 + arg7 and widens it before storing it  
the runtime trace ties that effective address to saved Dr7  
this is a local reader fragment, not a decompilation of the complete filter

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/29-35_bn_dr7_il.png)

the selected Dr0 address is read at 876B9D  
the corresponding Dr1, Dr2 and Dr3 reads are at 875DB5, 74ECB3 and 82235C

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/30-29_bn_dr0.png)

the effective addresses above were resolved from the execution state, not inferred from the obfuscated register names alone  
using saved addresses 1, 2, 4 and 8 in the emulator makes the selection clear

<table><thead><tr><th>DR7</th><th>aggregate</th></tr></thead><tbody><tr><td>00</td><td>0</td></tr><tr><td>01 or 02</td><td>1 — Dr0</td></tr><tr><td>04 or 08</td><td>2 — Dr1</td></tr><tr><td>10 or 20</td><td>4 — Dr2</td></tr><tr><td>40 or 80</td><td>8 — Dr3</td></tr><tr><td>FF</td><td>F — all four</td></tr><tr><td>100 or 400</td><td>0</td></tr></tbody></table>

either local or global enable bit makes that slot count  
nonzero saved addresses with DR7 = 0 are ignored in these tests  
the aggregate is stored at frame+480h by the block at 78D743

for the native continuation the four saved addresses were zero only while the filter ran, then restored before unwinding  
the filter returned 1 and the separate aggregate was zero

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/31-30_filter.png)

modeled continuations with aggregate 1 or F reach NtRaiseHardError with 50000018  
aggregate zero reaches the later runtime query and then original entry

the late attach hook
--------------------

both 3.9.4 and 3.10.1 install a hook at DbgUiRemoteBreakin  
the native entry redirects through a bridge to this image

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/32-31_hook.png)

the 3.10.1 destination is RVA A020  
it sets EDX to DEADC0DE and RCX to pseudo handle -1, then jumps through an import slot  
the live slot resolves to TerminateProcess

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/33-32_hook_target.png)

the same stub bytes in Binary Ninja  
the static slot is not yet the resolved native import, so the debugger supplies the API identification

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/34-33_bn_target.png)

3.9.4 already prepared -1 directly in its stub at BBB0  
3.10.1 keeps that behavior, with the stub now at A020  
installation and destination were checked, a later attach attempt was not used to terminate the test process

what changed from 3.9.4
-----------------------

all 27 API-condition cases produced the same outcomes and the same last API stages  
the 13 saved-context cases also match the DR7 selection rule  
no new rejection condition was found in the tested paths

<table><thead><tr><th>part</th><th>3.9.4 and 3.10.1</th></tr></thead><tbody><tr><td>PEB and callback</td><td>same signal and callback-clearing action</td></tr><tr><td>TitanHide</td><td>same device path and modeled 12-byte UnhideAll command</td></tr><tr><td>process queries</td><td>same two ports, overlapping output and alignment condition</td></tr><tr><td>created debug object</td><td>same type queries and nonzero counter conditions</td></tr><tr><td>thread hiding</td><td>same set followed by a one-byte hidden-state query</td></tr><tr><td>kernel debugger</td><td>same tested two-byte predicate</td></tr><tr><td>five-name driver scan</td><td>absent from both complete modeled startup paths traced here</td></tr><tr><td>invalid handle and TF</td><td>NtClose and application single-step filter in both</td></tr><tr><td>hardware breakpoint addresses</td><td>same DR7 local/global enable-bit selection</td></tr><tr><td>attach hook</td><td>same TerminateProcess(-1, DEADC0DE) destination behavior</td></tr></tbody></table>

the code layout is different even though these tested conditions match

<table><thead><tr><th>location</th><th>3.9.4 RVA</th><th>3.10.1 RVA</th></tr></thead><tbody><tr><td>initial PEB reader</td><td>9C5944</td><td>710E1A</td></tr><tr><td>second port reader</td><td>87DFB4</td><td>84982B</td></tr><tr><td>single-step filter</td><td>6EF1E0</td><td>89F800</td></tr><tr><td>aggregate store</td><td>82537B, frame+440h</td><td>78D743, frame+480h</td></tr><tr><td>attach hook target</td><td>BBB0</td><td>A020</td></tr></tbody></table>

register choices and syscall sites also differ  
this comparison covers the supplied samples and the tested branches, not every build in either version family

back to the program
-------------------

the controlled native run reached original entry RVA 1000  
RIP was not assigned to that address, execution reached the hardware breakpoint normally

![](https://ivankozlovsky.com/uploads/images/vmprotect-3101-anti-debug-analysis/35-34_oep.png)

the separate modeled tail also reached original entry  
27 API-condition cases completed, with 13 saved-context cases tested separately  
this covers the observed startup and tested variations, not every Windows-version fallback or allocation-failure path

the decision
------------

condensed pseudocode for the checks established above  
this is a reconstruction from the native trace and condition tests, not automatic decompiler output  
the callback action, malformed probes and fallback branches are described in their sections

```
if PEB.BeingDebugged:
    reject

clear instrumentation callback

if opening TitanHide succeeds:
    write HIDE_INFO { UnhideAll, 0, 0 }

status, port = query ProcessDebugPort
if NT_SUCCESS(status) and port != 0:
    reject

status, overlapped = query ProcessDebugObjectHandle
if NT_SUCCESS(status) or overlapped == 0:
    reject

status = query ProcessDebugObjectHandle with unaligned output
if status != STATUS_DATATYPE_MISALIGNMENT:
    reject

if creating a debug object succeeds:
    status, size = query ObjectTypeInformation with NULL output
    if status == STATUS_INFO_LENGTH_MISMATCH:
        allocate and query the type information
        if data query succeeds:
            require nonzero object and handle counters
            perform the overlapping ReturnLength query
    close the created object

set ThreadHideFromDebugger
status, hidden = query ThreadHideFromDebugger
if NT_SUCCESS(status) and hidden == 0:
    reject

status, kernel = query SystemKernelDebuggerInformation
if NT_SUCCESS(status) and kernel.Enabled and not kernel.NotPresent:
    reject

status, port = query ProcessDebugPort again
if NT_SUCCESS(status) and port != 0:
    reject
hide the current thread again

NtClose(0xDEADC0DE)
set TF and execute RDTSC
deliver the single-step exception to the application filter

for each saved debug-register slot i:
    if the context includes the required debug-register state:
        if (Context.Dr7 >> (2 * i)) & 3:
            aggregate |= Context.Dr[i]

if aggregate != 0:
    reject

continue startup and install DbgUiRemoteBreakin hook
hook destination: TerminateProcess(-1, 0xDEADC0DE)
continue to original entry
```

references
----------

[Microsoft — NtQueryInformationProcess](https://learn.microsoft.com/en-us/windows/win32/api/winternl/nf-winternl-ntqueryinformationprocess)  
[Microsoft — x64 CONTEXT and debug registers](https://learn.microsoft.com/en-us/windows/win32/api/winnt/ns-winnt-context)  
[Microsoft — NTSTATUS values](https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/using-ntstatus-values)  
[Microsoft — x64 exception handling](https://learn.microsoft.com/en-us/cpp/build/exception-handling-x64?view=msvc-170)  
[System Informer PHNT — process and thread information](https://github.com/winsiderss/systeminformer/blob/master/phnt/include/ntpsapi.h)  
[System Informer PHNT — object information](https://github.com/winsiderss/systeminformer/blob/master/phnt/include/ntobapi.h)  
[TitanHide — command structure](https://github.com/mrexodia/TitanHide/blob/master/TitanHide/TitanHide.h)  
[TitanHide — UnhideAll implementation](https://github.com/mrexodia/TitanHide/blob/master/TitanHide/hider.cpp)

If this content was useful you can **donate**. [Donate →](https://ivankozlovsky.com/donate)
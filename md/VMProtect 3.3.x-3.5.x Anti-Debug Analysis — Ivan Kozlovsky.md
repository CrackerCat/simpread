> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [ivankozlovsky.com](https://ivankozlovsky.com/blog/vmprotect-33x-35x-anti-debug-analysis)

> Tracing the early anti-debug checks: API return values, exact driver-name matching, and opposite hand......

same small x64 test program, protected separately with 3.3.1, 3.4.0 and 3.5.0  
user-mode + kernel-mode selected, the other protection options off  
x64dbg for the native execution and controlled inputs, IDA for the code and Unicorn for the driver-name read traces

the observed checks and tested rejection conditions match across these three samples  
the screenshots and RVAs below come from the 3.3.1 sample, the code layout and addresses change in the later builds

the first checks are ordinary Windows APIs  
after those come the kernel debugger query, a driver list scan, an invalid handle and a single-step exception check

to reach each later check I changed the earlier results in the running process  
the screenshots distinguish native outputs from controlled inputs  
addresses in the text are RVAs, x64dbg keeps the live addresses and IDA uses image base 140000000

IsDebuggerPresent
-----------------

the first call is at `40FED7`  
the Windows implementation reads the PEB through `GS:[60h]`, then reads `BeingDebugged` at offset 2  
the byte is 1 and the API returns 1

![](https://ivankozlovsky.com/uploads/images/vmprotect-33x-35x-anti-debug-analysis/01-01_isdebugger.png)

that is enough to reach the debugger warning  
changing only the return value to zero exposes the next check

CheckRemoteDebuggerPresent
--------------------------

the protected return address is `3CEDA2`  
the process handle is `-1`, so it is checking its own process

inside the Windows wrapper there is an `NtQueryInformationProcess` call with class `7`, `ProcessDebugPort`  
the native query returns a debug port value of `FFFFFFFFFFFFFFFF`  
the wrapper turns that into an output BOOL of 1 and returns TRUE

![](https://ivankozlovsky.com/uploads/images/vmprotect-33x-35x-anti-debug-analysis/02-02_remote.png)

both values matter here  
leaving the output BOOL at 1 but forcing the API return to FALSE continues to the next query  
with a successful API return and BOOL 1 it rejects

the debug object query
----------------------

next is a direct call to `NtQueryInformationProcess`  
class `1Eh`, `ProcessDebugObjectHandle`  
current process, an 8-byte output, return address `372903`

![](https://ivankozlovsky.com/uploads/images/vmprotect-33x-35x-anti-debug-analysis/03-03_debug_object.png)

the real call returns status zero and a nonzero handle  
zeroing only the handle does not help, it still rejects  
the important part is the signed NTSTATUS result

<table><thead><tr><th>status supplied to the consumer</th><th>result</th></tr></thead><tbody><tr><td>00000000, native handle</td><td>debugger warning</td></tr><tr><td>00000000, handle changed to zero</td><td>debugger warning</td></tr><tr><td>00000001 / 7FFFFFFF</td><td>debugger warning</td></tr><tr><td>80000000 / C0000001 / C0000353</td><td>continues</td></tr></tbody></table>

these tests match `NT_SUCCESS(status)`  
it is not just checking for status zero or one particular failure code  
Microsoft describes the same signed-success range in its [NTSTATUS reference](https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/using-ntstatus-values)

ThreadHideFromDebugger
----------------------

the next call is `NtSetInformationThread`  
thread handle `-2`, class `11h`, null buffer and length zero  
the protected return is `434FE1`

![](https://ivankozlovsky.com/uploads/images/vmprotect-33x-35x-anti-debug-analysis/04-04_hide_thread.png)

this is the thread-hiding action  
I stopped it before the syscall and supplied a return value so the thread would remain visible to x64dbg  
both success and `C0000001` continued to the kernel debugger query in these tests

kernel debugger information
---------------------------

`NtQuerySystemInformation` is called with class `23h`  
the output is two bytes, `KernelDebuggerEnabled` and `KernelDebuggerNotPresent`  
the protected return is `346D1F`

![](https://ivankozlovsky.com/uploads/images/vmprotect-33x-35x-anti-debug-analysis/05-05_kernel.png)

the native result on this machine is `00 01`  
I changed the two bytes after the real call to check all four combinations

<table><thead><tr><th>Enabled</th><th>NotPresent</th><th>result with successful query</th></tr></thead><tbody><tr><td>0</td><td>0</td><td>continues</td></tr><tr><td>0</td><td>1</td><td>continues</td></tr><tr><td>1</td><td>0</td><td>debugger warning</td></tr><tr><td>1</td><td>1</td><td>continues</td></tr></tbody></table>

so the condition is enabled AND present  
with `01 00`, a nonnegative status still rejects  
a failed query with `C0000001` skips this rejection and continues

```
if (NT_SUCCESS(status) &&
    info.KernelDebuggerEnabled &&
    !info.KernelDebuggerNotPresent)
    reject();
```

the driver names
----------------

another `NtQuerySystemInformation` call follows  
this time class `0Bh`, `SystemModuleInformation`

the first call asks for the required length  
it returns `C0000004` and `FE68h` bytes  
the loader allocates twice that size, `1FCD0h`, then calls again  
the two protected return addresses are `3DA9C1` and `3FC36A`

![](https://ivankozlovsky.com/uploads/images/vmprotect-33x-35x-anti-debug-analysis/06-06_module_size.png)

the successful native query returned 220 modules  
the buffer starts with the count, followed by the module records

![](https://ivankozlovsky.com/uploads/images/vmprotect-33x-35x-anti-debug-analysis/07-07_module_native.png)

```
buffer + 00h       module count
buffer + 08h       first record
record size        128h
record + 26h       OffsetToFileName
record + 28h       FullPathName[256]
```

the loader uses `OffsetToFileName` and compares the basename byte by byte  
the byte read is at `4387D8`  
below I replaced the returned module list with a single `ntice.sys` record  
this is controlled input, no driver was installed

![](https://ivankozlovsky.com/uploads/images/vmprotect-33x-35x-anti-debug-analysis/08-08_driver_read.png)

the same byte-reading instruction in IDA

![](https://ivankozlovsky.com/uploads/images/vmprotect-33x-35x-anti-debug-analysis/09-09_ida_driver.png)

the comparisons are inside the protected control flow  
the screenshot shows the actual read, while replaying the consumer gives the comparison order  
each of these names was also tested natively and reached the debugger warning

```
sice.sys
siwvid.sys
ntice.sys
iceext.sys
syser.sys
```

for `ntice.sys`, the consumer tries the first byte three times before reading the rest of the name and its terminating zero  
for `test.sys`, it tries the first byte five times and finishes without a match

<table><thead><tr><th>additional name</th><th>result</th></tr></thead><tbody><tr><td>NTICE.SYS</td><td>matches</td></tr><tr><td>xntice.sys</td><td>no match</td></tr><tr><td>ntice.sysx</td><td>no match</td></tr><tr><td>syserboot.sys / syserdbg.sys</td><td>no match</td></tr></tbody></table>

the NTICE cases show case-insensitive matching with full-name boundaries  
forcing the second module query to fail skips the scan, even when its buffer contains `ntice.sys`  
changing only the first query's status to a failure while keeping its positive required length still leads to the second query

the invalid handle
------------------

after the driver scan, there is a `CloseHandle(0xDEADC0DE)`  
the protected return is `3B1E61`  
inside the Windows wrapper this reaches `NtClose` with the same invalid handle

![](https://ivankozlovsky.com/uploads/images/vmprotect-33x-35x-anti-debug-analysis/10-10_invalid_handle.png)

the real call raises `STATUS_INVALID_HANDLE`, `C0000008`  
Microsoft documents this debugger-specific behavior for [CloseHandle](https://learn.microsoft.com/en-us/windows/win32/api/handleapi/nf-handleapi-closehandle)

passing that exception to the application led to an access violation in the protected exception machinery  
swallowing it in x64dbg allowed the next check to run  
this failure path did not produce the ordinary debugger-warning dialog

the single-step exception
-------------------------

at `339E0C` there is a `popfq`  
the supplied flags were `346h`, which includes TF  
then `rdtsc` at `339E0D` and a NOP at `339E0F`

![](https://ivankozlovsky.com/uploads/images/vmprotect-33x-35x-anti-debug-analysis/11-11_single_step.png)

the exception is `80000004` and its address is the NOP  
the same instruction sequence in IDA

![](https://ivankozlovsky.com/uploads/images/vmprotect-33x-35x-anti-debug-analysis/12-12_ida_tf.png)

if x64dbg swallows the single-step exception, the program shows the debugger warning  
if it delivers the exception, the application filter runs

the exception directory associates this region with the language handler at `22320`  
the scope is `339E0C..339E1A`, its filter is `339ED8`  
the native filter receives the exception record below and returns 1

![](https://ivankozlovsky.com/uploads/images/vmprotect-33x-35x-anti-debug-analysis/13-13_exception_record.png)

this is the relevant part of `__C_specific_handler` in IDA  
it calls the scope filter and tests the signed return value  
this is the SEH dispatcher, not a decompilation of the whole protected check

![](https://ivankozlovsky.com/uploads/images/vmprotect-33x-35x-anti-debug-analysis/14-14_ida_dispatch.png)

the observed positive filter result selects the handler continuation at `339E1A`  
there are other instruction islands nearby, including another RDTSC and a CPUID  
they were not executed on this successful tested path, so they are not counted as additional checks here

the warning
-----------

the API, kernel and matching-driver rejection paths reach the same message  
the protected return from `MessageBoxW` is `47A0E9`

![](https://ivankozlovsky.com/uploads/images/vmprotect-33x-35x-anti-debug-analysis/15-15_warning.png)

after the checks
----------------

the controlled run reached the original entry at `1000` with the real kernel module list  
the earlier user-mode results were changed, thread hiding was intercepted, the invalid-handle exception was swallowed and the single-step exception was delivered  
the unchanged executable also completed normally without a debugger

the decision
------------

simplified reconstruction of the observed checks and actions

```
if IsDebuggerPresent():
    reject

present = false
if CheckRemoteDebuggerPresent(current_process, &present) and present:
    reject

status = query ProcessDebugObjectHandle
if NT_SUCCESS(status):
    reject

NtSetInformationThread(current_thread, ThreadHideFromDebugger, null, 0)

status, info = query SystemKernelDebuggerInformation
if NT_SUCCESS(status) and info.Enabled and not info.NotPresent:
    reject

required_size = query SystemModuleInformation size
allocate the module buffer
status, modules = query SystemModuleInformation
if NT_SUCCESS(status):
    for each module basename:
        reject if it matches one of:
            sice.sys
            siwvid.sys
            ntice.sys
            iceext.sys
            syser.sys

CloseHandle(0xDEADC0DE)

set TF and execute RDTSC
require the single-step exception to reach the application handler

continue startup
```

references
----------

[Microsoft — IsDebuggerPresent](https://learn.microsoft.com/en-us/windows/win32/api/debugapi/nf-debugapi-isdebuggerpresent)  
[Microsoft — CheckRemoteDebuggerPresent](https://learn.microsoft.com/en-us/windows/win32/api/debugapi/nf-debugapi-checkremotedebuggerpresent)  
[Microsoft — CloseHandle and invalid handles under a debugger](https://learn.microsoft.com/en-us/windows/win32/api/handleapi/nf-handleapi-closehandle)  
[Microsoft — NTSTATUS values](https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/using-ntstatus-values)  
[Microsoft — x64 exception handling](https://learn.microsoft.com/en-us/cpp/build/exception-handling-x64?view=msvc-170)  
[System Informer PHNT — process and thread information classes](https://github.com/winsiderss/systeminformer/blob/master/phnt/include/ntpsapi.h)  
[System Informer PHNT — system information classes and structures](https://github.com/winsiderss/systeminformer/blob/master/phnt/include/ntexapi.h)

If this content was useful you can **donate**. [Donate →](https://ivankozlovsky.com/donate)
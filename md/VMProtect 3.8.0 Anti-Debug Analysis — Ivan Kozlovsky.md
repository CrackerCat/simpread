> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [ivankozlovsky.com](https://ivankozlovsky.com/blog/vmprotect-380-anti-debug-analysis)

> Three malformed object queries, controlled exception delivery, and native verification of the late-at......

same small x64 test program, this time protected with 3.8.0  
user-mode + kernel-mode selected, the other protection options off  
x64dbg for the native execution, IDA for the code and Unicorn for controlled inputs

the main checks still look like 3.6  
the object-query sequence has two extra calls now  
the late attach hook is present too, but native verification shows that 3.6 installs it as well

earlier results were changed in the running process to reach the later checks  
native outputs and emulated condition tests are identified below  
addresses in the text are RVAs, while the screenshots show live addresses

the PEB byte
------------

the initial reader at `98A14E` loads a byte through `[rbp+rcx-25h]`  
at this stop RBP points to `PEB+2` and RCX is `25h`  
so this is the direct `BeingDebugged` read, with a native value of 1 under x64dbg

![](https://ivankozlovsky.com/uploads/images/vmprotect-380-anti-debug-analysis/01-01_peb.png)

the same instruction in IDA

![](https://ivankozlovsky.com/uploads/images/vmprotect-380-anti-debug-analysis/02-02_ida_peb.png)

the initial debugger state was cleared for the controlled continuation, then restored before the later exception probes  
leaving the PEB byte changed through those probes alters exception delivery, so that is not a new 3.8 detection result

clearing the instrumentation callback
-------------------------------------

the startup path also calls `NtSetInformationProcess` with class `28h`, `ProcessInstrumentationCallback`  
the 16-byte structure is zero, including the callback pointer  
the native call succeeds  
this is an observed startup action, not a separately established rejection condition

the first debug port
--------------------

`ProcessDebugPort` uses class `7`, current process and an 8-byte output  
the native call succeeds and writes `FFFFFFFFFFFFFFFF`  
at `8D0914` that qword is loaded into RDX

![](https://ivankozlovsky.com/uploads/images/vmprotect-380-anti-debug-analysis/03-03_port.png)

clearing the buffer and the consumed value allows the next stage to run  
there are many syscall sites inside the protected image, with timestamp-dependent dispatch  
the call arguments identify the query more reliably than one captured syscall address  
the service numbers seen here belong to this Windows build

the overlapping object query
----------------------------

next is class `1Eh`, `ProcessDebugObjectHandle`  
R8 points to the output and the fifth argument points to exactly the same address  
R9 is 8 and R10 is the current-process pseudo handle

![](https://ivankozlovsky.com/uploads/images/vmprotect-380-anti-debug-analysis/04-04_overlap.png)

for the direct syscall, the first argument is in R10  
the ordinary fastcall panel's RCX row is not the process handle at this point

the native debugged call returns success and the overlapped qword ends as 8

![](https://ivankozlovsky.com/uploads/images/vmprotect-380-anti-debug-analysis/05-05_overlap_result.png)

an independent undebugged Windows call returned `C0000353` with the same final qword 8  
that 8 cannot be read as an ordinary standalone debug-object handle because ReturnLength shares the storage

<table><thead><tr><th>controlled replay result</th><th>observed path</th></tr></thead><tbody><tr><td>status 0 or 1, nonzero qword</td><td>warning</td></tr><tr><td>C0000001 or C0000353, nonzero qword</td><td>next checks</td></tr><tr><td>negative status, zero qword</td><td>warning</td></tr></tbody></table>

the tested condition fits `NT_SUCCESS(status) || slot == 0`  
this matches the tested 3.6 behavior

three malformed object queries
------------------------------

3.6 had one object query with `ReturnLength=1`  
3.8 executes three

first, current process and length 8, with the fifth argument set to 1

![](https://ivankozlovsky.com/uploads/images/vmprotect-380-anti-debug-analysis/06-06_bad_length.png)

the native syscall returns `C0000005` and leaves the seeded qword unchanged  
this is a returned NTSTATUS, not a process access-violation stop

![](https://ivankozlovsky.com/uploads/images/vmprotect-380-anti-debug-analysis/07-07_bad_result.png)

then the process handle becomes zero, while length stays 8

![](https://ivankozlovsky.com/uploads/images/vmprotect-380-anti-debug-analysis/08-08_null_handle.png)

then the handle becomes -1 again, but length becomes zero

![](https://ivankozlovsky.com/uploads/images/vmprotect-380-anti-debug-analysis/09-09_zero_length.png)

both extra native calls also return `C0000005`  
the controlled object matrix varied the statuses and output values across 24 cases  
the malformed-query variations still reached the kernel query  
these are two extra probes in the observed sequence, not two demonstrated extra rejection conditions

thread hiding
-------------

the next syscall is `NtSetInformationThread`  
class `11h`, current thread `-2`, null buffer and zero length

![](https://ivankozlovsky.com/uploads/images/vmprotect-380-anti-debug-analysis/10-10_hide.png)

for the controlled run I substituted an invalid thread handle during the call  
the real kernel returned `C0000008`, then I restored the original input and supplied success  
this kept the thread visible to x64dbg while the later checks ran  
it was not a native successful thread-hide call

the kernel debugger
-------------------

`NtQuerySystemInformation`, class `23h`, reads a two-byte `SystemKernelDebuggerInformation` result  
the real output here is `00 01`

![](https://ivankozlovsky.com/uploads/images/vmprotect-380-anti-debug-analysis/11-11_kernel.png)

the replay matrix matches 3.6  
success with Enabled=1 and NotPresent=0 reaches the warning  
00 00, 00 01 and 01 01 continue  
a failed query with 01 00 also continues

the loaded drivers
------------------

class `0Bh` obtains the module-list size, then the list itself  
the native data call has capacity `1FCD0h`  
the dump below is the actual returned module list

![](https://ivankozlovsky.com/uploads/images/vmprotect-380-anti-debug-analysis/12-12_modules.png)

the parser uses count at +0, first record at +8 and a `128h` record stride  
inside each record, `OffsetToFileName` is at +26h and the path starts at +28h  
the same five case-insensitive exact basenames are checked

```
sice.sys
siwvid.sys
ntice.sys
iceext.sys
syser.sys
```

each name independently reached the warning in the 3.8 replay tests  
`NTICE.SYS` matched too  
`test.sys`, `xntice.sys` and `ntice.sysx` continued  
a failed data query skipped an otherwise matching record  
these individual 3.8 driver cases are emulated tests, separate from the native list capture above

the second port query
---------------------

a later group repeats the debug-port query and thread hiding  
the second port query uses a different buffer  
at `90DB02` its result is added into RBX

![](https://ivankozlovsky.com/uploads/images/vmprotect-380-anti-debug-analysis/13-13_repeat.png)

the native output is -1 again  
clearing the earlier buffer alone would not cover this read  
the second consumed value was cleared separately, and thread hiding was controlled in the same way as the first call

invalid handle, trap flag and saved registers
---------------------------------------------

`CloseHandle` receives `DEADC0DE`

![](https://ivankozlovsky.com/uploads/images/vmprotect-380-anti-debug-analysis/14-14_close.png)

this raises the native invalid-handle exception `C0000008`  
after swallowing that exception, execution reaches the trap-flag sequence

![](https://ivankozlovsky.com/uploads/images/vmprotect-380-anti-debug-analysis/15-15_tf.png)

IDA shows the short sequence at `794DE4`  
`popfq`, `rdtsc`, then the NOP at `794DE7`

![](https://ivankozlovsky.com/uploads/images/vmprotect-380-anti-debug-analysis/16-16_ida_tf.png)

the single-step exception is `80000004`  
delivering it to the program reaches the real filter at `794F2C`

![](https://ivankozlovsky.com/uploads/images/vmprotect-380-anti-debug-analysis/17-17_exception.png)

the filter follows EXCEPTION_POINTERS to the saved CONTEXT  
Dr0 starts at +48h, Dr1 at +50h, Dr2 at +58h and Dr3 at +60h  
these are the hardware breakpoint addresses saved in the exception context

![](https://ivankozlovsky.com/uploads/images/vmprotect-380-anti-debug-analysis/18-18_context.png)

one protected reader is at `8AEF2C`  
the address arithmetic resolves to the saved Dr0 field in the traced execution

![](https://ivankozlovsky.com/uploads/images/vmprotect-380-anti-debug-analysis/19-19_ida_dr.png)

the controlled register matrix produces the bitwise OR of Dr0 through Dr3  
one nonzero register set to `12345678` produces that same value  
1, 1, 2, 4 produces 7  
all zero produces zero  
the aggregate is stored at the captured frame +400h

the filter returns 1 in all those cases  
that return is an exception-filter decision, not an anti-debug pass result  
the later continuation rejects the nonzero aggregate

![](https://ivankozlovsky.com/uploads/images/vmprotect-380-anti-debug-analysis/20-20_filter.png)

for the native continuation, the saved DR fields were zero only while the filter ran  
the original saved values were restored before unwinding so the original-entry breakpoint survived  
the single-register and mixed-register matrix remains emulated evidence

the late attach hook
--------------------

later startup enumerates threads and installs a hook on `ntdll!DbgUiRemoteBreakin`  
the native entry changes from its original instructions to a relative jump

![](https://ivankozlovsky.com/uploads/images/vmprotect-380-anti-debug-analysis/21-21_hook.png)

a nearby bridge uses `FF 25 00 00 00 00` and an absolute target pointer  
the target in this sample is `BA70`  
its native imports resolve to `GetCurrentProcess` and `TerminateProcess`

![](https://ivankozlovsky.com/uploads/images/vmprotect-380-anti-debug-analysis/22-22_target.png)

the destination in IDA

![](https://ivankozlovsky.com/uploads/images/vmprotect-380-anti-debug-analysis/23-23_ida_target.png)

the exit code is `DEADC0DE`  
3.6 also installs this hook, with its target at `B870`  
that was confirmed by actual WriteProcessMemory calls and before/after native bytes  
so this is not a new 3.8-only check  
installation and destination were verified, without using a later attach attempt to terminate the native process

what changed from 3.6
---------------------

<table><thead><tr><th>part</th><th>3.6</th><th>3.8.0</th></tr></thead><tbody><tr><td>PEB byte and callback clearing</td><td>present</td><td>same observed actions</td></tr><tr><td>port and overlapping object conditions</td><td>tested</td><td>same tested predicates</td></tr><tr><td>malformed object queries</td><td>one, ReturnLength=1</td><td>three, adds null handle and zero length</td></tr><tr><td>kernel debugger and driver list</td><td>two-byte query, five names</td><td>same tested conditions and names</td></tr><tr><td>later port and hide group</td><td>present</td><td>present, separate port buffer</td></tr><tr><td>TF and saved Dr0–Dr3</td><td>OR aggregate</td><td>same tested behavior, different addresses</td></tr><tr><td>DbgUiRemoteBreakin hook</td><td>native installation confirmed, target B870</td><td>native installation confirmed, target BA70</td></tr></tbody></table>

the main tested rejection conditions match  
the binaries and call sequences are not identical  
this comparison covers the supplied samples, not every build in either version family

after the checks
----------------

the controlled native run reached original entry `1000`  
RIP was not assigned to that address

![](https://ivankozlovsky.com/uploads/images/vmprotect-380-anti-debug-analysis/24-24_oep.png)

the real kernel-debugger output and module list were used  
earlier debugger results were changed, thread hiding was kept inactive, the invalid-handle exception was swallowed and the single-step was delivered  
the executable on disk was not patched  
the covered startup path and condition matrices do not establish every Windows-version fallback or allocation-failure path

the decision
------------

simplified reconstruction from the observed execution and controlled tests  
this is explanatory pseudocode, not pasted decompiler output

```
if PEB.BeingDebugged != 0:
    reject

clear the process instrumentation callback

status, port = query ProcessDebugPort
if NT_SUCCESS(status) and port != 0:
    reject

slot = 0
status = query ProcessDebugObjectHandle(
    handle = -1, output = &slot, length = 8,
    ReturnLength = (ULONG *)&slot
)
if NT_SUCCESS(status) or slot == 0:
    reject

for handle, length in [(-1, 8), (0, 8), (-1, 0)]:
    query ProcessDebugObjectHandle(
        handle, &seed, length, ReturnLength = (ULONG *)1
    )
    # no independent rejection established for these malformed calls

hide the current thread

status, info = query SystemKernelDebuggerInformation
if NT_SUCCESS(status) and info.Enabled and not info.NotPresent:
    reject

query the module-list size
allocate and query SystemModuleInformation
if NT_SUCCESS(status):
    for each module basename:
        reject on an exact case-insensitive match with:
            sice.sys, siwvid.sys, ntice.sys, iceext.sys, syser.sys

status, port = query ProcessDebugPort again
if NT_SUCCESS(status) and port != 0:
    reject
hide the current thread again

CloseHandle(0xDEADC0DE)
set TF and execute RDTSC
require single-step delivery to the application filter

saved_breakpoints = Context.Dr0 | Context.Dr1 |
                    Context.Dr2 | Context.Dr3
if saved_breakpoints != 0:
    reject

continue startup and install DbgUiRemoteBreakin hook
hook destination: TerminateProcess(GetCurrentProcess(), 0xDEADC0DE)
continue to original entry
```

references
----------

[Microsoft — x64 CONTEXT and debug registers](https://learn.microsoft.com/en-us/windows/win32/api/winnt/ns-winnt-context)  
[Microsoft — NTSTATUS values](https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/using-ntstatus-values)  
[Microsoft — CloseHandle](https://learn.microsoft.com/en-us/windows/win32/api/handleapi/nf-handleapi-closehandle)  
[Microsoft — x64 exception handling](https://learn.microsoft.com/en-us/cpp/build/exception-handling-x64?view=msvc-170)  
[System Informer PHNT — process and thread information classes](https://github.com/winsiderss/systeminformer/blob/master/phnt/include/ntpsapi.h)  
[System Informer PHNT — system information classes and structures](https://github.com/winsiderss/systeminformer/blob/master/phnt/include/ntexapi.h)

If this content was useful you can **donate**. [Donate →](https://ivankozlovsky.com/donate)
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [ivankozlovsky.com](https://ivankozlovsky.com/blog/vmprotect-36-anti-debug-analysis)

> Overlapping debug-object outputs, repeated debug-port queries, and the saved DR0–DR3 state inside the......

same small x64 test program, this time protected with 3.6  
user-mode + kernel-mode selected, the other protection options off  
x64dbg for the native execution, IDA for the code and Unicorn for controlled read traces

the kernel debugger query and the five driver names are still familiar  
the user-mode path is different now, and delivering the single-step exception is no longer enough when the saved context contains hardware breakpoint addresses

to reach the later checks I changed earlier results in the running process  
native outputs and controlled inputs are identified below  
addresses in the text are RVAs, the debugger shows live addresses

the PEB byte
------------

the first observed debugger-state check reads `PEB.BeingDebugged` directly  
at `3A2A40`, `movzx r10w, byte ptr [rdx]` reads from `PEB + 2`  
the native byte is 1

![](https://ivankozlovsky.com/uploads/images/vmprotect-36-anti-debug-analysis/01-01_peb.png)

the same reader in IDA

![](https://ivankozlovsky.com/uploads/images/vmprotect-36-anti-debug-analysis/02-02_ida_peb.png)

leaving the consumed value at 1 reaches the warning, and 2 does too  
changing that consumed value to zero exposes the next call  
this is the same debugger-state signal as before, reached through protected code rather than the IsDebuggerPresent export on this path

clearing the instrumentation callback
-------------------------------------

next is `NtSetInformationProcess` with class `28h`, `ProcessInstrumentationCallback`  
current process, 16-byte structure, protected return `3D04A5`

![](https://ivankozlovsky.com/uploads/images/vmprotect-36-anti-debug-analysis/03-03_callback.png)

the whole structure is zero, including the callback pointer  
the real call succeeds  
replacing its status with `C0000001` still reaches the next query, so this is an observed action rather than a separately demonstrated rejection condition

the first debug-port query
--------------------------

the next query uses a syscall inside the protected image at `4C5A12`  
class `7`, `ProcessDebugPort`, current process and an 8-byte output  
on this Windows build its service number is `19h`

![](https://ivankozlovsky.com/uploads/images/vmprotect-36-anti-debug-analysis/04-04_port.png)

the native result is status zero with output `FFFFFFFFFFFFFFFF`  
that rejects, and success with output 1 rejects too  
zeroing the output continues

the status matters as well  
`C0000001` or `80000000` with a nonzero output continued in the tests  
the tested behavior fits a successful query followed by a nonzero-port check

the overlapping debug-object output
-----------------------------------

the next syscall is at `4C596C`, class `1Eh`, `ProcessDebugObjectHandle`  
look at the third and fifth arguments  
`ProcessInformation` and `ReturnLength` point to the same 8-byte location

![](https://ivankozlovsky.com/uploads/images/vmprotect-36-anti-debug-analysis/05-05_object_overlap.png)

the native debugged call returns status zero and the final qword is 8  
that 8 is not a standalone debug-object handle  
the return-length write overlaps the handle output

I also called the API from a separate process without a debugger  
with the same overlap it returned `C0000353`, and the final qword was still 8

<table><thead><tr><th>status</th><th>final qword</th><th>result</th></tr></thead><tbody><tr><td>0 or 1</td><td>8</td><td>warning</td></tr><tr><td>C0000001 or C0000353</td><td>8</td><td>next query</td></tr><tr><td>C0000353</td><td>0</td><td>warning</td></tr><tr><td>C0000353</td><td>1, 9 or 100000000h</td><td>next query</td></tr></tbody></table>

so this is not an exact length-equals-8 comparison  
the tested condition fits `NT_SUCCESS(status) || output_qword == 0`  
the negative status by itself is not enough if the overlapping buffer was also zeroed

ReturnLength = 1
----------------

another class `1Eh` query follows at `4C59C4`  
this time the output contains a seed and the fifth argument is the invalid pointer `1`

![](https://ivankozlovsky.com/uploads/images/vmprotect-36-anti-debug-analysis/06-06_invalid_length.png)

the real kernel returns `C0000005` and leaves the seed unchanged  
this is an NTSTATUS returned by the call, not an access-violation exception delivered to the application  
the separate process without a debugger returned the same status and preserved its seed too

changing the status to success, 1 or `C0000353` still reached the next stage  
changing the output to zero also continued  
success plus a zero output was tested through original entry

the call is there, but those tests did not establish an independent rejection condition for it

ThreadHideFromDebugger
----------------------

the protected code uses `4C5A12` again, now with a different service number  
`NtSetInformationThread`, current thread `-2`, class `11h`, null buffer and zero length

![](https://ivankozlovsky.com/uploads/images/vmprotect-36-anti-debug-analysis/07-07_hide.png)

I changed the thread handle before executing this syscall so it would not hide the debugged thread  
the kernel returned `C0000008` for that invalid handle, then I supplied the status used by the test  
both success and `C0000001` continued

the same syscall instruction can serve different APIs  
the service number and arguments identify the operation, not the address alone

kernel debugger information
---------------------------

`NtQuerySystemInformation`, class `23h`  
two output bytes and protected return `42CA68`  
the native result is `00 01`

![](https://ivankozlovsky.com/uploads/images/vmprotect-36-anti-debug-analysis/08-08_kernel.png)

<table><thead><tr><th>Enabled</th><th>NotPresent</th><th>successful query</th></tr></thead><tbody><tr><td>0</td><td>0</td><td>continues</td></tr><tr><td>0</td><td>1</td><td>continues</td></tr><tr><td>1</td><td>0</td><td>warning</td></tr><tr><td>1</td><td>1</td><td>continues</td></tr></tbody></table>

this part matches the earlier samples  
successful query AND enabled AND present  
with `01 00`, changing the status to `C0000001` skips the rejection

the driver list
---------------

the two `SystemModuleInformation` queries use class `0Bh`  
the size query returns to `39CFC9`, the data query to `4A40CD`  
the native required size was `FE68`, and the data buffer capacity was `1FCD0`

![](https://ivankozlovsky.com/uploads/images/vmprotect-36-anti-debug-analysis/09-09_modules.png)

the first DWORD is the module count, 220 on this run  
the first record starts at offset 8, each record is `128h` bytes  
the filename offset is at record + `26h`, and the path starts at record + `28h`

the same five basenames are checked

```
sice.sys
siwvid.sys
ntice.sys
iceext.sys
syser.sys
```

here I supplied a controlled `ntice.sys` record and stopped at the byte reader, `3BF756`

![](https://ivankozlovsky.com/uploads/images/vmprotect-36-anti-debug-analysis/10-10_driver.png)

![](https://ivankozlovsky.com/uploads/images/vmprotect-36-anti-debug-analysis/11-11_ida_driver.png)

each of the five names was independently tested natively and reached the warning  
`NTICE.SYS` also matched  
`xntice.sys` and `ntice.sysx` did not  
failing the data query with `C0000001` skipped a matching `ntice.sys` record

a second debug-port query
-------------------------

the driver scan is not the end of the user-mode checks  
after an intervening protected loop with CRC-style table updates, there is another debug-port query

![](https://ivankozlovsky.com/uploads/images/vmprotect-36-anti-debug-analysis/12-12_port_repeat.png)

this one uses syscall address `4C596C`, which handled the overlapping debug-object query earlier  
now the information class is `7`  
the real kernel again returns success and `FFFFFFFFFFFFFFFF`

changing only the first port result therefore does not get past this later check  
zeroing this second output continues  
a failing `C0000001` status with the native nonzero output also continues

a second `ThreadHideFromDebugger` call follows at `4C5A12`  
the same controlled success and failure statuses both reached the invalid-handle probe

the invalid handle
------------------

`CloseHandle(DEADC0DE)`, protected return `3A891E`

![](https://ivankozlovsky.com/uploads/images/vmprotect-36-anti-debug-analysis/13-13_close.png)

under x64dbg the real call raises `C0000008`  
swallowing it allows the next test to run  
delivering it in the recorded run caused an access violation at `4C5851`, rather than the ordinary debugger warning

the invalid-handle exception behavior under a debugger is also described in Microsoft's [CloseHandle documentation](https://learn.microsoft.com/en-us/windows/win32/api/handleapi/nf-handleapi-closehandle)

the single-step exception
-------------------------

`popfq` at `3802E7`, `rdtsc` at `3802E8`, then the NOP at `3802EA`  
the flags enable TF and the exception code is `80000004`

![](https://ivankozlovsky.com/uploads/images/vmprotect-36-anti-debug-analysis/14-14_single_step.png)

the same sequence in IDA

![](https://ivankozlovsky.com/uploads/images/vmprotect-36-anti-debug-analysis/15-15_ida_tf.png)

swallowing the single-step exception reaches the warning  
delivering it runs the application's filter at `380437`  
but this time delivering the exception alone is not enough

the saved hardware breakpoint addresses
---------------------------------------

the filter receives the exception pointers and reads the saved `CONTEXT`  
in this run DR0 and DR1 contain our hardware breakpoint addresses

![](https://ivankozlovsky.com/uploads/images/vmprotect-36-anti-debug-analysis/16-16_saved_dr.png)

the actual load used for the DR0 read in IDA  
the native RCX value ties this generic load to `CONTEXT + 48h`

![](https://ivankozlovsky.com/uploads/images/vmprotect-36-anti-debug-analysis/17-17_ida_dr.png)

<table><thead><tr><th>saved field</th><th>CONTEXT offset</th><th>reader RVA</th></tr></thead><tbody><tr><td>ContextFlags</td><td>30</td><td>46C11F</td></tr><tr><td>Dr2</td><td>58</td><td>48E69E</td></tr><tr><td>Dr3</td><td>60</td><td>4B4136</td></tr><tr><td>Dr1</td><td>50</td><td>3B92D8</td></tr><tr><td>Dr0</td><td>48</td><td>3ADC7D</td></tr></tbody></table>

the four addresses are combined  
individual nonzero DR0, DR1, DR2 and DR3 values each affect the stored result  
in replay, values `1, 1, 2, 4` produced `7`, matching their bitwise OR  
the aggregate store is at `41F9FA`

the native run warns with the saved breakpoint addresses present  
temporarily zeroing those four saved addresses while the filter executes permits the later continuation  
I restored them after the filter returned, so the original-entry hardware breakpoint could still stop the process after unwinding

the filter returned 1 in both cases  
that return is an SEH decision, not the anti-debug pass/fail result

![](https://ivankozlovsky.com/uploads/images/vmprotect-36-anti-debug-analysis/18-18_ida_dispatcher.png)

this is the actual IDA F5 output of the SEH dispatcher  
it calls the scope filter and tests its signed return  
it is not a decompilation of the whole protected loader

the filter returns to `22313` and the scope continuation is `380360`  
nearby RDTSC and CPUID islands were not executed on the successful observed path, so they are not counted as extra checks here

the warning
-----------

the observed rejection path uses `NtRaiseHardError`  
argument `50000018`, protected return `37FEE8`

![](https://ivankozlovsky.com/uploads/images/vmprotect-36-anti-debug-analysis/19-19_warning.png)

what changed from 3.3.1–3.5.0
-----------------------------

<table><thead><tr><th>part</th><th>3.6 compared with the earlier samples</th></tr></thead><tbody><tr><td>debugger-state byte</td><td>same signal, direct protected PEB read on the initial path</td></tr><tr><td>process/thread queries</td><td>direct syscalls in the protected image</td></tr><tr><td>instrumentation callback</td><td>clearing call observed; no separate rejection demonstrated</td></tr><tr><td>debug object</td><td>overlapping ReturnLength and output, plus a nonzero-buffer condition</td></tr><tr><td>invalid ReturnLength</td><td>additional probe observed; no independent rejection established</td></tr><tr><td>kernel debugger</td><td>same tested truth table</td></tr><tr><td>driver names</td><td>same five case-insensitive exact basenames</td></tr><tr><td>later queries</td><td>second debug-port and thread-hiding group</td></tr><tr><td>exception filter</td><td>saved DR0–DR3 aggregation demonstrated</td></tr><tr><td>warning</td><td>NtRaiseHardError on the observed path</td></tr></tbody></table>

these comparisons describe the supplied test binaries  
they do not establish that every build in each version family has identical code

after the checks
----------------

the controlled run reached original entry `1000` with the real kernel-debugger result and module list  
the earlier user-mode results were changed, thread hiding was kept inactive, the invalid-handle exception was swallowed and the single-step was delivered  
the saved hardware breakpoint addresses were cleared only during the filter test, then restored

![](https://ivankozlovsky.com/uploads/images/vmprotect-36-anti-debug-analysis/20-20_original_entry.png)

the unchanged executable also completed normally without a debugger  
the tested startup and controlled cases are covered here, not every Windows-version fallback or allocation-failure branch

the decision
------------

simplified reconstruction from the observed path and tested conditions  
this is an explanation of the behavior, not pasted decompiler output

```
if PEB.BeingDebugged != 0:
    reject

clear the process instrumentation callback

status, port = query ProcessDebugPort
if NT_SUCCESS(status) and port != 0:
    reject

slot = 0
status = query ProcessDebugObjectHandle(
    output = &slot,
    length = 8,
    ReturnLength = (ULONG *)&slot
)
if NT_SUCCESS(status) or slot == 0:
    reject

seed = a seeded qword
query ProcessDebugObjectHandle(
    output = &seed,
    length = 8,
    ReturnLength = (ULONG *)1
)
# no independent rejection was established for this call

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

continue through the intervening protected code

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

continue startup
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
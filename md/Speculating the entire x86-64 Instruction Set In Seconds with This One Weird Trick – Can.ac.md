> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.can.ac](https://blog.can.ac/2021/03/22/speculating-x86-64-isa-with-one-weird-trick/)

> As cheesy as the title sounds, I promise it cannot beat the cheesiness of the technique I’ll be te......

[x86 Internals](https://blog.can.ac/category/x86/)

Mar 22, 2021

As cheesy as the title sounds, I promise it cannot beat the cheesiness of the technique I’ll be telling you about in this post. The morning I saw [Mark Ermolov’s tweet](https://twitter.com/_markel___/status/1373059797155778562) about the undocumented instruction reading from/writing to the CRBUS, I had a bit of free time in my hands and I knew I had to find out the opcode so I started theory-crafting right away. After a few hours of staring at numbers, I ended up coming up with a method of discovering practically every instruction in the processor using a side(?)-channel. It’s an interesting method involving even more interesting components of the processor so I figured I might as well write about it, so here it goes.

_You can find the full data-set and the implementation source at [haruspex.can.ac](https://haruspex.can.ac/browse/) / [Github](https://github.com/can1357/haruspex/)._

Preface 1/2: If storks are busy delivering babies, where do micro-instructions come from?
-----------------------------------------------------------------------------------------

Modern processors are built with a crazy amount of microarchitectural complexity these days and as one would expect the good old instruction decoders no longer decode for the execution unit directly. They end up decoding them into micro-instructions according to the micro-code of the processor to dispatch to the execution ports of the processor. There are two units that perform this translation in a modern Intel processor:

1.  **Micro-instruction Translation Engine (MITE)**. The unit responsible for the translation of simple legacy instructions that translate to four or less micro-instructions.
2.  **Microcode Sequencer (MS)**. The unit responsible for the translation of much more complex instructions that drive the CISC craziness of the Intel architecture we all hold dearly.

Another unit that dispatches these micro-instructions is the **Decoded Stream Buffer (DSB)** more frequently known as the **iCache** but it is not really relevant to the experiment we’re going to do. Now, why am I telling you all of this? Mainly because we can profile these extremely low-level units thanks to the performance counters Intel gracefully provides us; mainly these two bad boys:

![](https://i.can.ac/Qm944.png) ![](https://i.can.ac/dSppw.png)

One of the advantages of utilizing these events compared to a Sandsifter-like approach is that even if the microcode of the instruction throws a #UD (say if the password does not match or if the conditions are not met) it cannot fool us as it’d still have to be decoded.

Preface 2/2: Executing in a parallel universe
---------------------------------------------

Now the problem is getting those instructions there. There are a million of ways you can shoot yourself in the foot by executing random instructions. What happens if we hit an instruction that changes our interruptibility state, causes a system reset, messes with the internal caches or the very performance counters we’re using, the stack we’re executing on, the code of the profiler, the code segment we’re executing in…

The easy solution is to simply, not execute, at least in our visible universe, which brings us to our second cool microarchitectural detail: **speculative and out-of-order execution**. The processor evaluating the branch condition at the branch has apparently grown out of fashion so what really happens when you do a branch is that, first, the branch prediction engine attempts to guess the branch you are going to take to reduce the cost of the reversal of the instruction pipelines, given an equal possibility both branches execute at the same time where possible and no speculation fences are present and then one of them gets their net change reverted… which is pretty much what we wanted isn’t it? Now although probing the branch to reset the branch prediction state is possible and I’ve experimented with it before, it’s a bit bothersome so an easier way is to simply do a CALL. Consider this snippet:

```
call x
	<speculated code>
x:
	lea        rax,     [rip+z]
	xchg       [rsp],   rax
	ret
z:
```

This will cause the execution of the speculated code and the invoked subroutine out-of-order if possible. The `XCHG` may seem a bit overkill compared to a simpler solution popping the stack but as far as my experiments went, the processor is too smart to split the execution if the routine is non-returning so we need to feed the branch target buffer what it wants. I’ve used `XCHG` here instead of a `MOV` since it implies `LOCK` which will cause the processor to end up in a memory stall given that it has to end up visible given the atomicity –or at least so I theorize.

We will also need to trash the **Decoded Stream Buffer** I’ve mentioned previously so that we do not get any micro-instructions fed from the cache but there’s a very simple solution for that. The instruction cache also has to handle self-rewriting code so execute memory is extremely sensitive to any memory writes, given this information adding the simple snippet below before every measurement takes care of this issue.

```
auto ip = ia32::get_ip() & ~63ull;
std::copy_n( ( volatile uint8_t* ) ip, 512, ( volatile uint8_t* ) ip );
ia32::sfence();
for ( size_t n = 0; n != 512; n += 64 )
	ia32::clflush( ip + n );
```

0x0: Precise data collection
----------------------------

Finally, we’ve hit the exciting part of the experiment, experimenting. We want precise results which implies non-interruptibility. You might be tricked into thinking being in kernel-mode and doing a `CLI` solves this problem but this does not really work that way in reality. The first thing that worries me is an `#SMI` being delivered, although I keep hearing it freezes PMCs, as far as I’ve experimented it really doesn’t do a great job at that, but even if it did it’s still an impurity we have to eliminate so I’ll repeat the experiment until the mighty `IA32_MSR_SMI_COUNT` stays constant during the execution. `#NMI` is the other bugger, so setting the interrupt handler so that it signals a retry solves this problem (since I’m too lazy to unwind). `#MC` would also be considered in this category but at that point might as well let it all burn.

Repeating the experiment multiple times and picking the `Mod(x)`, and writing the code in a tight manner eliminates pretty much every other problem we’ve left. The next step is simply writing the code and actually collecting the data. The speculated code will be 15 copies of `NOP` and a `0xCE` at the end to cause an actual `#UD` and halt the speculative execution. We’ll be trying pretty much every opcode in the range of `0x00-0xFF` and they will optionally take a prefix of `0x0F` and optionally another prefix in the set `{ 0x66, 0xF2, 0xF3 }` since Intel likes to use them to create new opcodes out of thin-air (e.g. the recent FRED instruction `ERETS` with its `F2 0F 01 CA` encoding). We also need to add a suffix for discovering ModR/M variants. This process completes in mere seconds, which gave the title to this post.

0x1: Reducing the results
-------------------------

First of all, we’ll get two baseline measurements, the `NOP`s left as is, and the `0xCE` as the first opcode which will reveal the counter values for a complete execution and the `for-real-#UD` case (_I’ve tested other opcodes, 0xCE really isn’t an NSA backdoor, it’s the real deal as far as #UD’s go._).

Simply removing all measurements matching the `for-real-#UD` case measurement gets rid of most of the garbage, now we need to get rid of the redundant prefixes, taking a look at the data below you can see a pattern emerge:

```
┌─────────┬──────────────────────────────────┬──────┬────┐
│ (index) │             decoding             │ mits │ ms │
├─────────┼──────────────────────────────────┼──────┼────┤
│   90    │              'nop'               │  54  │ 80 │ /*Baseline*/
│  6690   │           'data16 nop'           │  53  │ 67 │
│  f290   │              'nop'               │  53  │ 80 │
│ f20f90  │ 'seto byte ptr [rax-0x6f6f6f70]' │  48  │ 80 │
└─────────┴──────────────────────────────────┴──────┴────┘
```

Every nop translates to a single micro-instruction that will be handled by the MITE, which means if the prefix is redundant MITS should be always off by just one and MS should stay the same, additionally, we can filter out some redundant checks by declaring `0x0F` never redundant. Combining both we get rid of most of the redundancies in a simple fashion and even might be able to calculate the instruction length, neat! The code below gets rid of 54954 entries.

```
const propertyMatch = (i1, i2) => {
	return i2.ms == i1.ms && i2.outOfOrder == i1.outOfOrder && i2.iclass == i1.iclass;
};

// Purge redundant prefixes.
//
for (const k1 of Object.keys(instructions)) {
	// Skip if already deleted.
	//
	const i1 = instructions[k1];
	if (!i1) {
		continue;
	}

	// Iterate each prefix (apart from 0f):
	//
	for (const pfx of prefixList) {
		// If the instruction exists:
		//
		const k2 = pfx + k1;
		if (k2 in instructions) {
			// If the instruction has matching properties as the derived from parent, delete the entry.
			//
			const i2 = instructions[k2];
			if (propertyMatch(i1, i2)) {
				// MITS#1 == MITS#2 can indicate same instruction if instruction halts.
				// Otherwise MITS#1 has to be one more than MITS#2 since it should execute one more NOP.
				//
				if (i1.mits != i2.mits) {
					if (i1.mits != i2.mits + 1) {
						continue;
					}
				} else if (i1.mits > faultBaseline.mits) {
					continue;
				}
				delete instructions[k2];
			}
		}
	}
}
```

Suffix-based purging is also more or less the same logic which gets rid of 72869 instructions, leaving us with 1699 entries, which is good enough to start the analysis!

```
// Purge redundant suffixes.
//
for (const k1 of Object.keys(instructions)) {
	// Skip if already deleted or not relevant.
	//
	const i1 = instructions[k1];
	if (!i1 || k1.length <= 2) {
		continue;
	}

	// Find maching entries:
	//
	for (const k2 of Object.keys(instructions)) {
		// If it is matching except the last byte:
		//
		if (k2.startsWith(k1.substr(0, k1.length - 2)) && k2 != k1) {
			// If it has matching properties ignoring the length, erase it
			//
			const i2 = instructions[k2];
			if (propertyMatch(i1, i2)) {
				delete instructions[k2];
			}
		}
	}
}
```

0x2: Deduction of behavior
--------------------------

Let’s demonstrate the amazing amount of information we’ve gathered just from these two counters and some bytes existing at some fixed place without even being executed. If MS is below the nop baseline, this would indicate that the control flow is interrupted meaning that it must be a branch or an exception and if the MITS is the same as the fault-baseline, this likely indicates a serializing instruction which dispatched a number of micro-instructions (given that it passed our initial filter of MS or MITS not remaining same) but then halted the speculative flow (since none of the NOP opcodes were decoded).

```
┌──────────┬────────────────────────────────────────────────────────────┬──────┬─────┬─────────────┬──────────────────┐
│ (index)  │                          decoding                          │ mits │ ms  │ serializing │ speculationFence │
├──────────┼────────────────────────────────────────────────────────────┼──────┼─────┼─────────────┼──────────────────┤
│  668690  │            'xchg byte ptr [rax-0x6f6f6f70], dl'            │  47  │ 88  │    true     │      false       │
│    6c    │                          'insb '                           │  39  │ 112 │    true     │       true       │
│    6d    │                          'insd '                           │  39  │ 99  │    true     │       true       │
│    6e    │                          'outsb '                          │  39  │ 98  │    true     │       true       │
│    6f    │                          'outsd '                          │  39  │ 98  │    true     │       true       │
│   8e90   │            'mov ss, word ptr [rax-0x6f6f6f70]'             │  42  │ 86  │    true     │       true       │
│   c290   │                        'ret 0x9090'                        │  43  │ 107 │    true     │       true       │ // <---  Likely errors since
│    c3    │                           'ret '                           │  41  │ 106 │    true     │       true       │ // <-/   CF will be interrupted
│   ca90   │                      'ret far 0x9090'                      │  39  │ 145 │    true     │       true       │ //       but will continue from a valid IP.
│    cb    │                         'ret far '                         │  39  │ 145 │    true     │       true       │
│    cc    │                          'int3 '                           │  39  │ 94  │    true     │       true       │
│   cd90   │                         'int 0x90'                         │  39  │ 91  │    true     │       true       │
│    cf    │                          'iretd '                          │  39  │ 136 │    true     │       true       │
│   e490   │                       'in al, 0x90'                        │  39  │ 110 │    true     │       true       │
│   e590   │                       'in eax, 0x90'                       │  39  │ 110 │    true     │       true       │
│   e690   │                       'out 0x90, al'                       │  39  │ 110 │    true     │       true       │
│   e790   │                      'out 0x90, eax'                       │  39  │ 110 │    true     │       true       │
│    ec    │                        'in al, dx'                         │  39  │ 109 │    true     │       true       │
│    ed    │                        'in eax, dx'                        │  39  │ 109 │    true     │       true       │
│    ee    │                        'out dx, al'                        │  39  │ 109 │    true     │       true       │
│    ef    │                       'out dx, eax'                        │  39  │ 109 │    true     │       true       │
│    f1    │                          'int1 '                           │  39  │ 112 │    true     │       true       │
│    f4    │                           'hlt'                            │  39  │ 124 │    true     │       true       │
│  0f0090  │              'lldt word ptr [rax-0x6f6f6f70]'              │  47  │ 93  │    true     │       true       │
│  0f0098  │              'ltr word ptr [rax-0x6f6f6f70]'               │  39  │ 110 │    true     │       true       │
│  0f0080  │              'sldt word ptr [rax-0x6f6f6f70]'              │  47  │ 87  │    true     │      false       │
│  0f0081  │              'sldt word ptr [rcx-0x6f6f6f70]'              │  47  │ 87  │    true     │      false       │
│  0f0088  │              'str word ptr [rax-0x6f6f6f70]'               │  47  │ 87  │    true     │      false       │
│  0f00a0  │              'verr word ptr [rax-0x6f6f6f70]'              │  47  │ 91  │    true     │       true       │
│  0f00a8  │              'verw word ptr [rax-0x6f6f6f70]'              │  47  │ 91  │    true     │       true       │
│  0f00d8  │                          'ltr ax'                          │  39  │ 108 │    true     │       true       │
│  0f0190  │                'lgdt ptr [rax-0x6f6f6f70]'                 │  47  │ 94  │    true     │       true       │
│  0f0198  │                'lidt ptr [rax-0x6f6f6f70]'                 │  47  │ 94  │    true     │       true       │
│  0f0180  │                'sgdt ptr [rax-0x6f6f6f70]'                 │  47  │ 89  │    true     │      false       │
│  0f0188  │                'sidt ptr [rax-0x6f6f6f70]'                 │  47  │ 88  │    true     │      false       │
│  0f01b0  │              'lmsw word ptr [rax-0x6f6f6f70]'              │  39  │ 103 │    true     │       true       │
│  0f01b8  │             'invlpg byte ptr [rax-0x6f6f6f70]'             │  39  │ 114 │    true     │       true       │
│  0f01a0  │              'smsw word ptr [rax-0x6f6f6f70]'              │  47  │ 85  │    true     │      false       │
│ f20f22a4 │                       'mov cr4, rsp'                       │  39  │ 103 │    true     │       true       │
│ f20f2396 │                       'mov dr2, rsi'                       │  39  │ 110 │    true     │       true       │
│ f20f2380 │                       'mov dr0, rax'                       │  39  │ 109 │    true     │       true       │
│ f20fc788 │           'cmpxchg8b qword ptr [rax-0x6f6f6f70]'           │  46  │ 95  │    true     │      false       │
│ f20fc78a │           'cmpxchg8b qword ptr [rdx-0x6f6f6f70]'           │  46  │ 95  │    true     │      false       │
│  f38690  │       'xrelease xchg byte ptr [rax-0x6f6f6f70], dl'        │  47  │ 88  │    true     │      false       │
│  f38790  │      'xrelease xchg dword ptr [rax-0x6f6f6f70], edx'       │  47  │ 88  │    true     │      false       │
│  f38890  │        'xrelease mov byte ptr [rax-0x6f6f6f70], dl'        │  47  │ 84  │    true     │      false       │
│  f38990  │       'xrelease mov dword ptr [rax-0x6f6f6f70], edx'       │  47  │ 84  │    true     │      false       │
│   f36c   │                        'rep insb '                         │  39  │ 112 │    true     │       true       │
│   f36d   │                        'rep insd '                         │  39  │ 112 │    true     │       true       │
│   f36e   │                        'rep outsb '                        │  39  │ 111 │    true     │       true       │
│   f36f   │                        'rep outsd '                        │  39  │ 111 │    true     │       true       │
│   f3a4   │         'rep movsb byte ptr [rdi], byte ptr [rsi]'         │  43  │ 118 │    true     │       true       │ //
│   f3a6   │         'rep cmpsb byte ptr [rsi], byte ptr [rdi]'         │  43  │ 123 │    true     │       true       │ //
│   f3a7   │        'rep cmpsd dword ptr [rsi], dword ptr [rdi]'        │  43  │ 123 │    true     │       true       │ //
│   f3aa   │                 'rep stosb byte ptr [rdi]'                 │  43  │ 125 │    true     │       true       │ // Likely errors since
│   f3ac   │                 'rep lodsb byte ptr [rsi]'                 │  43  │ 106 │    true     │       true       │ // rcx is undefined.
│   f3ad   │                'rep lodsd dword ptr [rsi]'                 │  43  │ 106 │    true     │       true       │ //
│   f3ae   │                 'rep scasb byte ptr [rdi]'                 │  43  │ 123 │    true     │       true       │ //
│   f3af   │                'rep scasd dword ptr [rdi]'                 │  43  │ 123 │    true     │       true       │ //
│ f30f0082 │              'sldt word ptr [rdx-0x6f6f6f70]'              │  46  │ 87  │    true     │      false       │
│ f30f0088 │              'str word ptr [rax-0x6f6f6f70]'               │  46  │ 87  │    true     │      false       │
│ f30f0180 │                'sgdt ptr [rax-0x6f6f6f70]'                 │  46  │ 89  │    true     │      false       │
│ f30f018a │                'sidt ptr [rdx-0x6f6f6f70]'                 │  46  │ 88  │    true     │      false       │
│ f30f01a1 │              'smsw word ptr [rcx-0x6f6f6f70]'              │  46  │ 85  │    true     │      false       │
│ f30f2190 │                       'mov rax, dr2'                       │  39  │ 107 │    true     │       true       │
│ f30f22a4 │                       'mov cr4, rsp'                       │  39  │ 103 │    true     │       true       │
│ f30f2380 │                       'mov dr0, rax'                       │  39  │ 109 │    true     │       true       │
│ f30f238e │                       'mov dr1, rsi'                       │  39  │ 110 │    true     │       true       │
│ f30f7890 │                             ''                             │  39  │ 87  │    true     │       true       │
│ f30f7990 │                             ''                             │  39  │ 87  │    true     │       true       │
│ f30fc789 │           'cmpxchg8b qword ptr [rcx-0x6f6f6f70]'           │  46  │ 95  │    true     │      false       │
│ f30fc78f │           'cmpxchg8b qword ptr [rdi-0x6f6f6f70]'           │  46  │ 95  │    true     │      false       │
│ f30fc7b0 │             'vmxon qword ptr [rax-0x6f6f6f70]'             │  39  │ 116 │    true     │       true       │
│ f30fc733 │                  'vmxon qword ptr [rbx]'                   │  39  │ 119 │    true     │       true       │
│ f30fc776 │                'vmxon qword ptr [rsi-0x70]'                │  39  │ 120 │    true     │       true       │
└──────────┴────────────────────────────────────────────────────────────┴──────┴─────┴─────────────┴──────────────────┘
```

Considering how simplistic the conditions are, not bad right?

0x3: Deduction of speculative behavior
--------------------------------------

You might have noticed the “Speculation fence” indication in the previous data dump. I’ve gotten a bit greedy and wanted to also know if the instructions speculatively execute or if they halt the queue for the sake of side-channeling the results so I went ahead and collected another piece of information, which comes from a rather unexpected performance counter:

![](https://i.can.ac/ToW0H.png)

You might be wondering what this has to do with anything. This particular counter is very useful given the fact that we don’t do any divisions in our previous code. Why? Because this means adding a division right after the instruction and seeing if the cycles is non-zero will let us know if the speculative execution was halted or not. We achieve this by first squeezing a `divps xmm4, xmm5` there right before the `#UD`‘ing `0xCE` and finally we waste some cycles at the non-speculative counterpart to cause a stall giving the speculative code more execution time from the execution port. Changing the previous routine code to the following pretty much gives us the perfect setup:

```
vmovups    ymm0,    [temp]
vmovups    ymm1,    [temp]
vmovups    ymm2,    [temp]
vmovups    ymm3,    [temp]
vzeroupper
addps      xmm0,    xmm1
vaddps     ymm2,    ymm0,    ymm3
vaddps     ymm1,    ymm0,    ymm2
vaddps     ymm3,    ymm0,    ymm1
vaddps     ymm0,    ymm0,    [temp]
vaddps     ymm1,    ymm0,    [temp]
vaddps     ymm2,    ymm0,    [temp]
vaddps     ymm3,    ymm0,    [temp]
vaddps     ymm0,    ymm0,    ymm1
vaddps     ymm2,    ymm0,    ymm3
vaddps     ymm1,    ymm0,    ymm2
vaddps     ymm3,    ymm0,    ymm1
vaddps     ymm0,    ymm0,    [temp]
vaddps     ymm1,    ymm0,    [temp]
vaddps     ymm2,    ymm0,    [temp]
vaddps     ymm3,    ymm0,    [temp]
vaddps     ymm0,    ymm0,    ymm1
vaddps     ymm2,    ymm0,    ymm3
vaddps     ymm1,    ymm0,    ymm2
vaddps     ymm3,    ymm0,    ymm1
vaddps     ymm0,    ymm0,    [temp]
vaddps     ymm1,    ymm0,    [temp]
vaddps     ymm2,    ymm0,    [temp]
vaddps     ymm3,    ymm0,    [temp]
vaddps     ymm0,    ymm0,    ymm1
vaddps     ymm2,    ymm0,    ymm3
vaddps     ymm1,    ymm0,    ymm2
vaddps     ymm3,    ymm0,    ymm1
lea        rax,     [rip+z]
xchg       [rsp],   rax
ret
```

AVX to SSE switch, memory stall, register dependencies, atomicity, this one has it all! Marking the entries that have non-zero counters as “speculative friendly” essentially lets us know what instructions we can speculatively execute and leak information from, and as you can see in the next example it seems to work pretty nicely.

```
-- These indeed leak data under speculative execution:
┌──────────┬────────────────────────────────────────────────────────────┬──────┬─────┬─────────────┬──────────────────┐
│ (index)  │                          decoding                          │ mits │ ms  │ serializing │ speculationFence │
├──────────┼────────────────────────────────────────────────────────────┼──────┼─────┼─────────────┼──────────────────┤
│   8690   │            'xchg byte ptr [rax-0x6f6f6f70], dl'            │  48  │ 88  │    false    │      false       │
│   e090   │                'loopne 0xffffffffffffff92'                 │  56  │ 87  │    false    │      false       │
│    fb    │                           'sti '                           │  56  │ 83  │    false    │      false       │
│    fc    │                           'cld '                           │  53  │ 83  │    false    │      false       │
│  0f0080  │              'sldt word ptr [rax-0x6f6f6f70]'              │  47  │ 87  │    true     │      false       │
│  0f0081  │              'sldt word ptr [rcx-0x6f6f6f70]'              │  47  │ 87  │    true     │      false       │
│  0f0088  │              'str word ptr [rax-0x6f6f6f70]'               │  47  │ 87  │    true     │      false       │
│  0f00c0  │                         'sldt eax'                         │  51  │ 85  │    false    │      false       │
│  0f00c8  │                         'str eax'                          │  51  │ 85  │    false    │      false       │
│  0f0009  │                    'str word ptr [rcx]'                    │  51  │ 87  │    false    │      false       │
│  0f0180  │                'sgdt ptr [rax-0x6f6f6f70]'                 │  47  │ 89  │    true     │      false       │
│  0f0188  │                'sidt ptr [rax-0x6f6f6f70]'                 │  47  │ 88  │    true     │      false       │
│  0f01a0  │              'smsw word ptr [rax-0x6f6f6f70]'              │  47  │ 85  │    true     │      false       │
│  0f01a1  │              'smsw word ptr [rcx-0x6f6f6f70]'              │  47  │ 85  │    true     │      false       │
│  0f01d0  │                         'xgetbv '                          │  51  │ 88  │    false    │      false       │
│  0f01d5  │                           'xend'                           │  51  │ 84  │    false    │      false       │
│  0f01e0  │                         'smsw eax'                         │  51  │ 84  │    false    │      false       │
│  0f010f  │                      'sidt ptr [rdi]'                      │  51  │ 88  │    false    │      false       │
│  0f0140  │                   'sgdt ptr [rax-0x70]'                    │  50  │ 89  │    false    │      false       │
│  0f2098  │                       'mov rax, cr3'                       │  51  │ 88  │    false    │      false       │
│  0f2080  │                       'mov rax, cr0'                       │  51  │ 85  │    false    │      false       │
│   0f31   │                          'rdtsc '                          │  52  │ 93  │    false    │      false       │
│   0f77   │                           'emms'                           │  52  │ 111 │    false    │      false       │
│   0fa1   │                          'pop fs'                          │  52  │ 87  │    false    │      false       │
│  0fa390  │            'bt dword ptr [rax-0x6f6f6f70], edx'            │  51  │ 86  │    false    │      false       │
													   ...
│ f30f0082 │              'sldt word ptr [rdx-0x6f6f6f70]'              │  46  │ 87  │    true     │      false       │
│ f30f0088 │              'str word ptr [rax-0x6f6f6f70]'               │  46  │ 87  │    true     │      false       │
│ f30f0006 │                   'sldt word ptr [rsi]'                    │  50  │ 87  │    false    │      false       │
│ f30f000a │                    'str word ptr [rdx]'                    │  50  │ 87  │    false    │      false       │
│ f30f0180 │                'sgdt ptr [rax-0x6f6f6f70]'                 │  46  │ 89  │    true     │      false       │
│ f30f018a │                'sidt ptr [rdx-0x6f6f6f70]'                 │  46  │ 88  │    true     │      false       │
│ f30f01a1 │              'smsw word ptr [rcx-0x6f6f6f70]'              │  46  │ 85  │    true     │      false       │
│ f30f010f │                      'sidt ptr [rdi]'                      │  50  │ 88  │    false    │      false       │
│ f30f0126 │                   'smsw word ptr [rsi]'                    │  50  │ 85  │    false    │      false       │
│ f30f0140 │                   'sgdt ptr [rax-0x70]'                    │  49  │ 89  │    false    │      false       │
│ f30faed0 │                       'wrfsbase eax'                       │  50  │ 87  │    false    │      false       │
│ f30faed8 │                       'wrgsbase eax'                       │  50  │ 87  │    false    │      false       │
│ f30faec0 │                       'rdfsbase eax'                       │  50  │ 86  │    false    │      false       │
│ f30faec8 │                       'rdgsbase eax'                       │  50  │ 86  │    false    │      false       │
│ f30fb391 │           'btr dword ptr [rcx-0x6f6f6f70], edx'            │  50  │ 86  │    false    │      false       │
│ f30fb39f │           'btr dword ptr [rdi-0x6f6f6f70], ebx'            │  50  │ 86  │    false    │      false       │
│ f30fbb92 │           'btc dword ptr [rdx-0x6f6f6f70], edx'            │  50  │ 86  │    false    │      false       │
│ f30fbb94 │        'btc dword ptr [rax+rdx*4-0x6f6f6f70], edx'         │  49  │ 86  │    false    │      false       │
│ f30fc789 │           'cmpxchg8b qword ptr [rcx-0x6f6f6f70]'           │  46  │ 95  │    true     │      false       │
│ f30fc78f │           'cmpxchg8b qword ptr [rdi-0x6f6f6f70]'           │  46  │ 95  │    true     │      false       │
└──────────┴────────────────────────────────────────────────────────────┴──────┴─────┴─────────────┴──────────────────┘

-- Yet these do not:
┌──────────┬─────────────────────────────────────────────────┬──────┬─────┬─────────────┬──────────────────┐
│ (index)  │                    decoding                     │ mits │ ms  │ serializing │ speculationFence │
├──────────┼─────────────────────────────────────────────────┼──────┼─────┼─────────────┼──────────────────┤
│    6c    │                     'insb '                     │  39  │ 112 │    true     │       true       │
│    6d    │                     'insd '                     │  39  │ 99  │    true     │       true       │
│    6e    │                    'outsb '                     │  39  │ 98  │    true     │       true       │
│    6f    │                    'outsd '                     │  39  │ 98  │    true     │       true       │
│   8e90   │       'mov ss, word ptr [rax-0x6f6f6f70]'       │  42  │ 86  │    true     │       true       │
│    9d    │                    'popfq '                     │  55  │ 87  │    false    │       true       │
│   c290   │                  'ret 0x9090'                   │  43  │ 107 │    true     │       true       │
│    c3    │                     'ret '                      │  41  │ 106 │    true     │       true       │
│   c890   │              'enter 0x9090, 0x90'               │  50  │ 93  │    false    │       true       │
│   ca90   │                'ret far 0x9090'                 │  39  │ 145 │    true     │       true       │
│    cb    │                   'ret far '                    │  39  │ 145 │    true     │       true       │
│    cc    │                     'int3 '                     │  39  │ 94  │    true     │       true       │
│   cd90   │                   'int 0x90'                    │  39  │ 91  │    true     │       true       │
│    cf    │                    'iretd '                     │  39  │ 136 │    true     │       true       │
│   e190   │           'loope 0xffffffffffffff92'            │  64  │ 83  │    false    │       true       │
│   e490   │                  'in al, 0x90'                  │  39  │ 110 │    true     │       true       │
│   e590   │                 'in eax, 0x90'                  │  39  │ 110 │    true     │       true       │
│   e690   │                 'out 0x90, al'                  │  39  │ 110 │    true     │       true       │
│   e790   │                 'out 0x90, eax'                 │  39  │ 110 │    true     │       true       │
│    ec    │                   'in al, dx'                   │  39  │ 109 │    true     │       true       │
│    ed    │                  'in eax, dx'                   │  39  │ 109 │    true     │       true       │
│    ee    │                  'out dx, al'                   │  39  │ 109 │    true     │       true       │
│    ef    │                  'out dx, eax'                  │  39  │ 109 │    true     │       true       │
│    f1    │                     'int1 '                     │  39  │ 112 │    true     │       true       │
│   f390   │                     'pause'                     │  52  │ 86  │    false    │       true       │
│    f4    │                      'hlt'                      │  39  │ 124 │    true     │       true       │
│    fd    │                     'std '                      │  53  │ 83  │    false    │       true       │
│  0f0090  │        'lldt word ptr [rax-0x6f6f6f70]'         │  47  │ 93  │    true     │       true       │
│  0f0098  │         'ltr word ptr [rax-0x6f6f6f70]'         │  39  │ 110 │    true     │       true       │
│  0f00a0  │        'verr word ptr [rax-0x6f6f6f70]'         │  47  │ 91  │    true     │       true       │
│  0f00a8  │        'verw word ptr [rax-0x6f6f6f70]'         │  47  │ 91  │    true     │       true       │
│  0f00d0  │                    'lldt ax'                    │  51  │ 91  │    false    │       true       │
│  0f00d8  │                    'ltr ax'                     │  39  │ 108 │    true     │       true       │
                                                     ...
│  0f00e0  │                    'verr ax'                    │  51  │ 90  │    false    │       true       │
│  0f00e8  │                    'verw ax'                    │  51  │ 90  │    false    │       true       │
│  0f0190  │           'lgdt ptr [rax-0x6f6f6f70]'           │  47  │ 94  │    true     │       true       │
│  0f0198  │           'lidt ptr [rax-0x6f6f6f70]'           │  47  │ 94  │    true     │       true       │
│  0f01b0  │        'lmsw word ptr [rax-0x6f6f6f70]'         │  39  │ 103 │    true     │       true       │
│  0f01b8  │       'invlpg byte ptr [rax-0x6f6f6f70]'        │  39  │ 114 │    true     │       true       │
│  0f01d1  │                    'xsetbv '                    │  51  │ 117 │    false    │       true       │
│  0f01d2  │                       ''                        │  39  │ 87  │    true     │       true       │
│  0f01d4  │                    'vmfunc '                    │  39  │ 83  │    true     │       true       │
│ 0f2193   │                 'mov rbx, dr2'                  │  39  │ 107 │    true     │       true       │
└──────────┴─────────────────────────────────────────────────┴──────┴─────┴─────────────┴──────────────────┘
```

0x![](https://blog.can.ac/wp-content/uploads/2021/03/AngryBitterAustralianshelduck-max-1mb.gif): Some of the interesting results
--------------------------------------------------------------------------------------------------------------------------------

Here is the full list of ~undocumented~ easter-egg instructions my i7 6850k comes with:

```
┌──────────┬──────────┬──────┬─────┬─────────────┬──────────────────┐
│ (index)  │ decoding │ mits │ ms  │ serializing │ speculationFence │
├──────────┼──────────┼──────┼─────┼─────────────┼──────────────────┤
│  0f01d2  │    ''    │  39  │ 87  │    true     │       true       │
│  0f01c6  │    ''    │  39  │ 83  │    true     │       true       │
│  0f01cc  │    ''    │  39  │ 104 │    true     │       true       │
│  0f0c90  │    ''    │  39  │ 138 │    true     │       true       │ /* Recent CRBUS leaking instruction, 90 is the next NOP. */
│   0f0e   │ 'femms'  │  52  │ 101 │    false    │       true       │ /* Recent CRBUS leaking instruction */
│  0faed0  │    ''    │  39  │ 87  │    true     │       true       │
│  0fc790  │    ''    │  39  │ 87  │    true     │       true       │
│ 660f3883 │    ''    │  39  │ 81  │    true     │       true       │
│ 660f3860 │    ''    │  39  │ 87  │    true     │       true       │
│ 660f3a80 │    ''    │  39  │ 87  │    true     │       true       │
│ f30f7890 │    ''    │  39  │ 87  │    true     │       true       │
│ f30f7990 │    ''    │  39  │ 87  │    true     │       true       │
│ f30fe7fc │    ''    │  73  │ 80  │    false    │       true       │
└──────────┴──────────┴──────┴─────┴─────────────┴──────────────────┘
```

Contrary to popular belief `mov cr2, reg` is not serializing.

```
┌──────────┬────────────────┬──────┬─────┬─────────────┬──────────────────┐
│ (index)  │    decoding    │ mits │ ms  │ serializing │ speculationFence │
├──────────┼────────────────┼──────┼─────┼─────────────┼──────────────────┤
│  0f2090  │ 'mov rax, cr2' │  51  │ 83  │    false    │      false       │
│  0f2098  │ 'mov rax, cr3' │  51  │ 88  │    false    │      false       │
│  0f2080  │ 'mov rax, cr0' │  51  │ 85  │    false    │      false       │
│  0f2290  │ 'mov cr2, rax' │  51  │ 87  │    false    │       true       │ /* ! */
│  0f2298  │ 'mov cr3, rax' │  39  │ 161 │    true     │       true       │
│  0f2299  │ 'mov cr3, rcx' │  39  │ 151 │    true     │       true       │
│  0f229b  │ 'mov cr3, rbx' │  39  │ 155 │    true     │       true       │
│  0f2280  │ 'mov cr0, rax' │  39  │ 110 │    true     │       true       │
│  0f2281  │ 'mov cr0, rcx' │  39  │ 153 │    true     │       true       │
│  0f22a0  │ 'mov cr4, rax' │  39  │ 103 │    true     │       true       │
│  0f22a1  │ 'mov cr4, rcx' │  39  │ 120 │    true     │       true       │
│  0f22a4  │ 'mov cr4, rsp' │  39  │ 104 │    true     │       true       │
│ 660f22a4 │ 'mov cr4, rsp' │  39  │ 103 │    true     │       true       │
│ f20f22a4 │ 'mov cr4, rsp' │  39  │ 103 │    true     │       true       │
│ f30f22a4 │ 'mov cr4, rsp' │  39  │ 103 │    true     │       true       │
└──────────┴────────────────┴──────┴─────┴─────────────┴──────────────────┘
```

Despite lacking the CPL check of `int imm8`, `int1` has more logic in the microcode.

```
┌─────────┬────────────┬──────┬─────┬─────────────┬──────────────────┐
│ (index) │  decoding  │ mits │ ms  │ serializing │ speculationFence │
├─────────┼────────────┼──────┼─────┼─────────────┼──────────────────┤
│   cc    │  'int3 '   │  39  │ 94  │    true     │       true       │
│  cd90   │ 'int 0x90' │  39  │ 91  │    true     │       true       │
│   f1    │  'int1 '   │  39  │ 112 │    true     │       true       │
└─────────┴────────────┴──────┴─────┴─────────────┴──────────────────┘
```

`mov ss` is a speculation fence whereas `cli` isn’t. `lss` is a speculation fence whereas `lgs` isn’t.

```
┌─────────┬────────────────────────────────────────────────┬──────┬────┬─────────────┬──────────────────┐
│ (index) │                    decoding                    │ mits │ ms │ serializing │ speculationFence │
├─────────┼────────────────────────────────────────────────┼──────┼────┼─────────────┼──────────────────┤
│  8890   │      'mov byte ptr [rax-0x6f6f6f70], dl'       │  49  │ 80 │    false    │      false       │
│  8990   │     'mov dword ptr [rax-0x6f6f6f70], edx'      │  49  │ 80 │    false    │      false       │
│ 668890  │      'mov byte ptr [rax-0x6f6f6f70], dl'       │  48  │ 80 │    false    │      false       │
│  8a90   │      'mov dl, byte ptr [rax-0x6f6f6f70]'       │  49  │ 80 │    false    │      false       │
│  8b90   │     'mov edx, dword ptr [rax-0x6f6f6f70]'      │  49  │ 80 │    false    │      false       │
│  8c90   │      'mov word ptr [rax-0x6f6f6f70], ss'       │  50  │ 80 │    false    │      false       │
│  8e90   │      'mov ss, word ptr [rax-0x6f6f6f70]'       │  42  │ 86 │    true     │       true       │ /* ! */
│   fa    │                     'cli '                     │  56  │ 80 │    false    │      false       │
│ 0fb290  │        'lss edx, ptr [rax-0x6f6f6f70]'         │  39  │ 89 │    true     │       true       │ /* ! */
│ 0fb590  │        'lgs edx, ptr [rax-0x6f6f6f70]'         │  47  │ 89 │    true     │      false       │
└─────────┴────────────────────────────────────────────────┴──────┴────┴─────────────┴──────────────────┘
```

* * *

![](https://blog.can.ac/wp-content/uploads/2020/04/EF49184E-56D7-4165-B4F3-A6F4E3798B7E-e1587490487648.jpg)

##### [Can Bölük](https://blog.can.ac/author/can1357/)

Security researcher and reverse engineer; mostly interested in Windows kernel development and low-level programming.  
Founder of [Verilave Inc.](https://verilave.com/)
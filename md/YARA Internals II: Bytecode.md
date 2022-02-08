> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bnbdr.github.io](https://bnbdr.github.io/posts/extracheese/)

> and how it can still be used to run arbitrary code

YARA Internals II: Bytecode

and how it can still be used to run arbitrary code

Unlike my [fastidious explanation](https://bnbdr.github.io/posts/swisscheese/) of YARA’s binary rule format, I’ll try to keep this post short by focusing on YARA’s VM architecture and how a compiled rule can **still** be used to run arbitrary code, despite the mitigations added to the latest release of YARA.

The issues I’d discovered were assigned [CVE-2018-19975](https://cve.mitre.org/cgi-bin/cvename.cgi?>CVE-2018-19974</a>, <a href=) and [here](https://cve.mitre.org/cgi-bin/cvename.cgi?>CVE-2018-19976</a>.</p><p>To skip straight to the exploitation part jump <a href=), head over to the [repo](https://github.com/bnbdr/swisscheese) or the [GitHub issue](https://github.com/VirusTotal/yara/issues/999).

YARA’s Virtual Machine
----------------------

YARA’s virtual machine (henceforth referenced as `yvm` for brevity) uses a stack (`vstack`) and has a small scratch memory (`vmem`). All operations of the bytecode use `QWORD` values (`yval`). The virutal machine also supports branching, arithmic operations, etc (implemented in [exec.c](https://github.com/VirusTotal/yara/blob/v3.8.1/libyara/exec.c#L190)).

`yvm` allocates a pretty big virtual stack by default (0x4000 bytes). This size is configurable by passing `stack-size` as a command-line argument.

Most opcodes either explicitly or implicitly push or pop off the `vstack`, changing the stack-pointer (`sp`) accordingly. When the opcode `OP_HALT` is reached `yvm` asserts the `vstack` is empty (`sp == 0`).

Contrary to the virtual stack, `vmem` is placed on the **real stack** and is quite small - holding only 16 `yval`s. Accessing this memory region is possible using `OP_PUSH_M` and `OP_POP_M`(amongst others). Both of which use the `vstack` as target or source and have one operand that specifies the index in `vmem` to access.

`yvm` has a `None` equivalent known as `UNDEFINED` (which equals `0xFFFABADAFABADAFF`) to differentiate between “Falsy” and `UNDEFINED` cases. This is used for “fail” cases like converting strings to numbers, dividing by zero, etc.

### types

To fully support all of YARA’s features, the bytecode has several different core types of `yval`s:

`int64` `double` `pointer` `object` `match-string` `string` `regex`

Out of the above, most noteworthy is the `object` type which corresponds to the `YR_OBJECT` struct [in YARA’s implementation](https://github.com/VirusTotal/yara/blob/v3.8.1/libyara/include/yara/types.h#L681). This is used to support “named” objects (`yobj`) that are exposed by `yvm` and its modules, and fall between the following “class” types:

<table><thead><tr><th><strong>class</strong></th><th><strong>exposed functionality</strong></th><th><strong>opcode</strong></th><th><strong>yara rule example</strong></th></tr></thead><tbody><tr><td><strong>structure</strong></td><td>fetch member <code>yobj</code> by name</td><td><code>OP_OBJ_FIELD</code></td><td><code>pe.number_of_sections</code></td></tr><tr><td><strong>dictionary</strong></td><td>fetch <code>yobj</code> by name</td><td><code>OP_LOOKUP_DICT</code></td><td><code>pe.version_info["CompanyName"]</code></td></tr><tr><td><strong>array</strong></td><td>fetch <code>yobj</code> at index</td><td><code>OP_INDEX_ARRAY</code></td><td><code>dotnet.streams[i]</code></td></tr><tr><td><strong><code>yval</code> wrapper</strong></td><td>fetch <code>yobj</code>’s <code>yval</code></td><td><code>OP_OBJ_VALUE</code></td><td><code>pe.number_of_sections == 1</code></td></tr><tr><td><strong>function</strong></td><td>peform action with <code>yval</code> list</td><td><code>OP_CALL</code></td><td><code>pe.imports("LoadLibrary")</code></td></tr></tbody></table>

Before performing any of the above the bytecode must load the `yobj` to the stack using `OP_OBJ_LOAD`. To check `pe.number_of_sections` the bytecode would have to do something like this:

```
OP_OBJ_LOAD     ascii   "pe"                    
# stack: [pe<yobj>]
OP_OBJ_FIELD    ascii   "number_of_sections"    
# stack: [number_of_sections<yobj>]
OP_OBJ_VALUE                                    
# stack: [1<yval>]
...


```

### modules

If you go ahead and disassemble the following rule you’ll notice a few extra opcodes were generated compared to the above `yarasm`:

```
import "pe"

rule single_section
{
    condition:
        pe.number_of_sections == 1
}


```

One of them is `OP_IMPORT`. It tells `yvm` to load the `"pe"` module- otherwise `OP_OBJ_LOAD` would fail to locate a `yobj` with that identifier. At this point `yvm` hands over the heavylifting to `yr_modules_load` where the loading logic is performed in two steps:

1.  **declarations**: creating a tree of “named” objects (`yobj`s) for the `yvm` runtime.
2.  **parsing**: iterating over the input to be scanned and initializing the relavant `yobj`s that were decalred in the previous step.

Basically, all the `yobs` declared in the **declarations** step can now be “found” by `OP_OBJ_LOAD`.

### functions

`yvm` allows its modules to “export” functions for the bytecode to call. Just like any other `yobj`, function-class `yobj`s are found in the same manner. The only difference as far as the bytecode goes is using them.

As noted by the [docs](https://yara.readthedocs.io/en/v3.8.1/writingmodules.html), YARA allows function overloading:

```
begin_declarations;
/*                                        code
                                ret-val    |
                        arg-list  |        |
                   name     |     |        | 
                     V      V     |        V                     */
   declare_function("md5", "ii", "s", data_md5);
   declare_function("md5", "s", "s", string_md5);

end_declarations;


```

Therefore, whenever `OP_CALL` is encountered `yvm` will check the length of the arg-list format (the operand) and pop `yval`s from the `vstack` to an `args` array (limited to 128 `yval`s). `yvm` will then use the next `yval` off the `vstack` as a function-`yobj` and search it for the matching prototype.

* * *

“PARANOID_EXEC”
---------------

YARA version 3.8.1 introduced `PARANOID_EXEC` to mitigate maliciously compiled bytecode (with added checks on the rule file itself too). Most importantly it added:

*   boundry checks on all opcodes that access `vmem`
*   boundry checks before writing to `args` array (which is on the **real stack**)
*   extra checks on `vstack` boundries
*   a canary in every `yobj` created by `yvm`, randomized when YARA is initialized.

The paranoid is never entirely mistaken
---------------------------------------

This too started with pure intentions. I wish to contribute a feature to YARA; one I couldn’t implement without looking carefully at YARA’s modules with regards to their life-cycle and their interaction with the bytecode.

#### [literally](https://cve.mitre.org/cgi-bin/cvename.cgi?>CVE-2018-19976</a></h4><p>Only after some time did it hit me - the entire concept of loading a <code>yobj</code> to the stack is itself an info-leak, <strong>by design</strong>. And due to the architecture of <code>yvm</code> I could change that pointer however I liked using arithmic opcodes to point someplace else, hopefully user-controlled.</p><p>Even if I leave it to <del>chance</del> ASLR and hope for the best, almost all the promising opcodes check the canary and the class type of the <code>yobj</code> before touching it, making it <em><a href=) impossible.

#### [](https://cve.mitre.org/cgi-bin/cvename.cgi?>CVE-2018-19975</a></h4><p>Cue in <code>OP_COUNT</code>:</p><pre>    // no checks here
    r1.i = r1.s->matches[tidx].count;

</pre><p>Which to those less familiar with YARA code, loosly translates to:</p><pre>    *TOS <-- *(UINT_PTR)(*TOS+0x38)

</pre><h4 id=)[It hurt itself in its confusion](https://cve.mitre.org/cgi-bin/cvename.cgi?>CVE-2018-19974</a></h4><p>To top it all of, I realized that <code>vmem</code> was uninitalized, leaking some more addresses (but not necessary for my PoC).</p><h2 id=)

[I’d set onwards to write a PoC exploit. The master plan was building a fake **function** `yobj` on the `vstack`. To make sure it works I had to set the overload prototype for the function, point the code to a gadget, populate the leaked canary, and then make YARA use that fake `yobj` to my advantage by executing `OP_CALL`.](https://cve.mitre.org/cgi-bin/cvename.cgi?>CVE-2018-19974</a></h4><p>To top it all of, I realized that <code>vmem</code> was uninitalized, leaking some more addresses (but not necessary for my PoC).</p><h2 id=)

[There was one lingering issue - where exactly is the `vstack`?](https://cve.mitre.org/cgi-bin/cvename.cgi?>CVE-2018-19974</a></h4><p>To top it all of, I realized that <code>vmem</code> was uninitalized, leaking some more addresses (but not necessary for my PoC).</p><h2 id=)

[Before I started skimming through the available leaked values in the uninitalized `vmem`, I remembered `OP_IMPORT` causes a lot of allocations and I can control when. Long story short, the `vstack` is reliably positioned `0x20450`(or `0x20490` if debugging in VS) behind the `"pe"` module.](https://cve.mitre.org/cgi-bin/cvename.cgi?>CVE-2018-19974</a></h4><p>To top it all of, I realized that <code>vmem</code> was uninitalized, leaking some more addresses (but not necessary for my PoC).</p><h2 id=)

[So now I got a lovely fake `yobj` whose “function” address will be called using the following prototype:](https://cve.mitre.org/cgi-bin/cvename.cgi?>CVE-2018-19974</a></h4><p>To top it all of, I realized that <code>vmem</code> was uninitalized, leaking some more addresses (but not necessary for my PoC).</p><h2 id=)

```
[typedef int (*OP_CALL_TARGET)(void*, void*, void*);](https://cve.mitre.org/cgi-bin/cvename.cgi?>CVE-2018-19974</a></h4><p>To top it all of, I realized that <code>vmem</code> was uninitalized, leaking some more addresses (but not necessary for my PoC).</p><h2 id=) 
```

[There was still the matter of building and placing a ROP chain. As for building- getting to a **real** function `yobj` is really easy with `OP_OBJ_LOAD`/`OB_OBJ_FIELD`. And since all of YARA’s modules are statically compiled I can easily infer YARA’s base address by “loading” an existing function `yobj`.](https://cve.mitre.org/cgi-bin/cvename.cgi?>CVE-2018-19974</a></h4><p>To top it all of, I realized that <code>vmem</code> was uninitalized, leaking some more addresses (but not necessary for my PoC).</p><h2 id=)

[Ironically, the largest buffer under my control that’s also placed on the real stack is the `args` array. By setting a large enough arg-list format for my fake function `yobj` I can populate it with up to 128 `yvals` - 256 gadgets! The only thing missing is that first gadget to start it all- to return right into my `args` array. Luckily finding a rogue `add esp, 0XXh; ret` was easy enough.](https://cve.mitre.org/cgi-bin/cvename.cgi?>CVE-2018-19974</a></h4><p>To top it all of, I realized that <code>vmem</code> was uninitalized, leaking some more addresses (but not necessary for my PoC).</p><h2 id=)

[At this point I felt pleased with my PoC and settled on locating `WinExec` by calculating its offset from `GetProcAddress`, which is imported by YARA.](https://cve.mitre.org/cgi-bin/cvename.cgi?>CVE-2018-19974</a></h4><p>To top it all of, I realized that <code>vmem</code> was uninitalized, leaking some more addresses (but not necessary for my PoC).</p><h2 id=)

[![](https://bnbdr.github.io/posts/extracheese/calc.gif)](https://cve.mitre.org/cgi-bin/cvename.cgi?>CVE-2018-19974</a></h4><p>To top it all of, I realized that <code>vmem</code> was uninitalized, leaking some more addresses (but not necessary for my PoC).</p><h2 id=)

* * *

### [Epilogue](https://cve.mitre.org/cgi-bin/cvename.cgi?>CVE-2018-19974</a></h4><p>To top it all of, I realized that <code>vmem</code> was uninitalized, leaking some more addresses (but not necessary for my PoC).</p><h2 id=)

[I tried making the](https://cve.mitre.org/cgi-bin/cvename.cgi?>CVE-2018-19974</a></h4><p>To top it all of, I realized that <code>vmem</code> was uninitalized, leaking some more addresses (but not necessary for my PoC).</p><h2 id=) [yara assembly code](https://github.com/bnbdr/swisscheese/tree/d789b187de43241f9c408b39436bafbf2c7d07f8/extracheese.yarasm) for this exploit as readable as possible and wrote a [syntax highlighting extension](https://github.com/bnbdr/swisscheese/tree/d789b187de43241f9c408b39436bafbf2c7d07f8/yarasm-syntax/) for VSCode. I encourage those curious to take a look.
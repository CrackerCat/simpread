> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.ashw.io](https://www.ashw.io/blog-sysreg-lib)

> Reveals a header-only C library for reading/writing 64-bit Arm system registers - simply include the ......

Writeable system registers define a convenience macro of the form safe_write_<reg>( … ), and system registers that are both readable and writeable define a convenience macro of the form read_modify_write_<reg>( … ). These convenience macros are one of the main highlights of the library and allow for powerful manipulation of 64-bit system registers while still optimising down to only a handful of A64 assembly instructions with no branches and no static storage.

As mentioned earlier, the unsafe_write_<reg>() function does not account for currently or previously RES1 bits, which may lead to portability issues if the programmer does not correctly set these bits based on their desired behaviour. The safe_write_<reg>() convenience macro solves this by allowing you to set a variadic list of fields, with any unspecified fields defaulting to their current/previous RES value i.e. all currently or previously RES1 fields not specified in the variadic list will be set to 1.

```
/* C code */
#include "sysreg/sctlr_el1.h"
void foo( void )
{
    safe_write_sctlr_el1( .m=1, .c=1, .i=1 );
}

/* compiler output */
mov     w8, #0x1985
movk    w8, #0x30d0, lsl #16
msr     sctlr_el1, x8
ret

```

Note how while we only specified .m=1, .c=1, and .i=1, the value written to SCTLR_EL1 also has all currently or previously RES1 fields set to 1, such as .itd=1, .sed=1, .eos=1, etc.

Repeating the same, but this time explicitly clearing one of those currently or previously RES1 fields to 0 in the variadic list:

```
/* in C */
#include "sysreg/sctlr_el1.h"
void foo( void )
{
    safe_write_sctlr_el1( .m=1, .c=1, .i=1, .itd=0 );
}

/* compiler output */
mov     w8, #0x1905
movk    w8, #0x30d0, lsl #16
msr     sctlr_el1, x8
ret

```

Here we can see bit [7] corresponding to .itd in the first MOV has been cleared; the value moved into w8 is now 0x1905 vs 0x1985 in the earlier example.

The read_modify_write_<reg>() convenience macro works in a similar way, but instead reads the current value of a system register, overwrites a variadic list of fields, then writes the result back to the system register, the key thing here being that any fields not specified in the variadic list are untouched.

```
/* in C */
void foo( void )
{
    read_modify_write_sctlr_el1( .m=1, .c=1, .i=1 );
}

/* compiler output */
mrs     x8, sctlr_el1
mov     w9, #0x1005
orr     x8, x8, x9
msr     sctlr_el1, x8
ret

```

Repeating this but clearing a previously RES1 field such as .itd=0:

```
/* in C */
void foo( void )
{
    read_modify_write_sctlr_el1( .m=1, .c=1, .i=1, .itd=0 );
}

/* compiler output */
mrs     x8, sctlr_el1
and     x8, x8, #0xffffffffffffff7f
mov     w9, #0x1005
orr     x8, x8, x9
msr     sctlr_el1, x8
ret

```

Note how the .itd field is cleared using an AND instruction before the other specified fields are ORR’d in. I’ll dive into the technical details of how the variadic macro is able to do that in a future blog post.
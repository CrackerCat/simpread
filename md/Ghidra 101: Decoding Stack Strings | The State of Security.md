> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.tripwire.com](https://www.tripwire.com/state-of-security/security-data-protection/ghidra-101-decoding-stack-strings/)

In this blog series, I will be putting the spotlight on some useful [Ghidra](https://www.tripwire.com/state-of-security/vert/tripwirebookclub-ghidra-book/) features you might have missed. Each post will look at a different feature and show how it helps you save time and be more effective in your reverse engineering workflows. Ghidra is an incredibly powerful tool, but much of this power comes from knowing how to use it effectively.

In this post, I’ll be discussing how to use the Ghidra Decompiler to identify strings being constructed on the stack at runtime. This stack string technique is an easy way for a programmer to obscure string data within a program by blending it as opaque operand instructions. A typical pattern for constructing a stack string would be a series of MOV instructions transferring constant values into adjacent locations on the stack as shown in these basic examples:

![](https://3b6xlt3iddqmuq5vy2w0s5d3-wpengine.netdna-ssl.com/state-of-security/wp-content/uploads/sites/3/single-byte-copies.png)Single Byte Copies ![](https://3b6xlt3iddqmuq5vy2w0s5d3-wpengine.netdna-ssl.com/state-of-security/wp-content/uploads/sites/3/DWORD-copies.png)DWORD Copies

The above examples as seen in the Decompiler:

![](https://3b6xlt3iddqmuq5vy2w0s5d3-wpengine.netdna-ssl.com/state-of-security/wp-content/uploads/sites/3/decompiler-single-byte-copies.png)Single Byte Copies ![](https://3b6xlt3iddqmuq5vy2w0s5d3-wpengine.netdna-ssl.com/state-of-security/wp-content/uploads/sites/3/decompiler-DWORD-copies.png)DWORD Copies

Decoding Stack Strings
----------------------

Some stack strings can be identified or deciphered with simple operations in the Decompiler view. In the case of the single byte copies, it may be possible to read out stack string values by updating Ghidra’s stack frame description so that the individual opaque bytes are interpreted as elements in a character array. The easiest way to do this is to retype the earliest element from _undefined_ to _char[n]_ where _n_ is the length of the reconstructed string.

![](https://3b6xlt3iddqmuq5vy2w0s5d3-wpengine.netdna-ssl.com/state-of-security/wp-content/uploads/sites/3/retype-variable.png) ![](https://3b6xlt3iddqmuq5vy2w0s5d3-wpengine.netdna-ssl.com/state-of-security/wp-content/uploads/sites/3/data-type-chooser-dialog.png)

The resulting changes in the Decompiler reveal the string clearly:

![](https://3b6xlt3iddqmuq5vy2w0s5d3-wpengine.netdna-ssl.com/state-of-security/wp-content/uploads/sites/3/string-in-decompiler.png)

### DWORD Value Previews

This is an easy way to evaluate that simple example, but it is not so trivial to convert multibyte stack string construction. In this case, it is possible to glimpse the data through the data type preview which appears when mousing over the constants.

![](https://3b6xlt3iddqmuq5vy2w0s5d3-wpengine.netdna-ssl.com/state-of-security/wp-content/uploads/sites/3/data-type-preview-constants-1.png)Preview of 0x72636553 ![](https://3b6xlt3iddqmuq5vy2w0s5d3-wpengine.netdna-ssl.com/state-of-security/wp-content/uploads/sites/3/data-type-preview-constants-2.png)Preview of 0x217465

###  Scripted Analysis

While these techniques can work, they are lacking in robustness and may lead to confusion related to endianness or complicated strings. Fortunately, Ghidra includes a script, SimpleStackString.py, for dealing with these examples.

To use the script:

1.  Set the cursor to the first line of the stack string construction.
2.  Open the Script Manager window and filter for ‘stack.’
3.  Select SimpleStackString.py and click the ‘Run Script’ button.
4.  Output is logged to the Console and added in Comments.

Example output from using the SimpleStackString.py script on a MalwareTech Blog challenge is shown below:

![](https://3b6xlt3iddqmuq5vy2w0s5d3-wpengine.netdna-ssl.com/state-of-security/wp-content/uploads/sites/3/simplestackstring-py.png)

Keep in mind, however, that this script is rather limited and will fail in some scenarios as shown here when parsing the DWORD copy example above:

![](https://3b6xlt3iddqmuq5vy2w0s5d3-wpengine.netdna-ssl.com/state-of-security/wp-content/uploads/sites/3/parsed-DWORD-copy.png)

The analyzed program would actually read out ‘Secret!\0,’ but the script apparently didn’t correctly infer the byte order which jumbled the string. The BBBB above was actually adjacent data in the program that was misinterpreted because the null-termination was missed or otherwise ignored.

### Further Challenges

An evasive developer can do a lot to hinder our ability to recognize and decode stack strings via static analysis. The writes which construct the stack string may be executed in any order, interleaved with functional code or further obfuscated with decoy writes or encodings. In cases where static recovery of stack strings is highly tedious, it may be more appropriate to consider dynamic analysis techniques such as running the code or sections of the code within a debugger or emulator. This is not currently supported in Ghidra, but it has been one of the most hotly anticipated upcoming features teased by the NSA, and a [recently pushed ‘debugger’ branch](https://github.com/NationalSecurityAgency/ghidra/tree/debugger) on GitHub finally makes this feature available for early testing. Within that branch, there is also a note in the [developer’s guide](https://github.com/NationalSecurityAgency/ghidra/blob/debugger/DebuggerDevGuide.md) that emulation will become an integral feature of the UI, as well. In time, these features could certainly simplify the process of recovering obfuscated functionality from compiled code. 

#### Read More about Ghidra

[Ghidra 101: Cursor Text Highlighting](https://www.tripwire.com/state-of-security/security-data-protection/cyber-security/ghidra-101-cursor-text-highlighting/)

[Ghidra 101: Slice Highlighting](https://www.tripwire.com/state-of-security/security-data-protection/ghidra-101-slice-highlighting/)
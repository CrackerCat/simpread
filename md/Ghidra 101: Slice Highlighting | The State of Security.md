> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.tripwire.com](https://www.tripwire.com/state-of-security/security-data-protection/ghidra-101-slice-highlighting/)

In this blog series, I will be putting the spotlight on useful [Ghidra](https://ghidra-sre.org/) features you may have missed. Each post will look at a different feature and show how it helps you save time and be more effective in your reverse engineering workflows. Ghidra is an incredibly powerful tool, but much of this power comes from [knowing how to use it](https://www.tripwire.com/state-of-security/vert/tripwirebookclub-ghidra-book/) effectively.

In this post, I will be discussing what slice highlighting is and how it helps us visualize relationships between variables to better understand a program. Before we can discuss how to use slice highlighting in Ghidra, I’d like to take a moment to introduce the concept of program slicing. In terms of software reverse engineering, program slicing is a way of abstracting code into smaller groups of statements known as slices. Slices are formed by following how a particular variable’s value affects or is affected by the values of other variables. The Ghidra Decompiler exposes functionality to quickly apply highlighting to visualize these program slices.

The easiest way to access slice highlighting within Ghidra is to right-click on a variable in the Decompiler:

![](https://3b6xlt3iddqmuq5vy2w0s5d3-wpengine.netdna-ssl.com/state-of-security/wp-content/uploads/sites/3/Right-click-Ghidra.png)

It is important to note that the slice highlighting options are only available when the cursor is at a variable whose value is being used or set. Slice highlighting is not available in the context menu if the cursor is at a variable declaration or a program point where the _address of_ operator (&) is being used. The Highlight Forward Slice action will highlight variables whose values are affected by the value of the variable under the cursor.

![](https://3b6xlt3iddqmuq5vy2w0s5d3-wpengine.netdna-ssl.com/state-of-security/wp-content/uploads/sites/3/forward-slice.png)

In the above example, a forward slice was requested for _index_ on line 31. Note that on line 31, the usage of _surface­_area_ would not be affected by _index_ until after the loop restarts.

Highlight Backward Slice will highlight variables whose values contributed to the value of the variable under the cursor.

![](https://3b6xlt3iddqmuq5vy2w0s5d3-wpengine.netdna-ssl.com/state-of-security/wp-content/uploads/sites/3/backward-slice.png)

In the above example, a backward slice was requested for _index_ on line 31. The only highlighted variable in this case is _index_, and it includes the initialization of _index_ prior to entering the loop.

In addition to these options, Ghidra also has forward and backward ‘Inst Slice’ options. This is short for instruction slice and will include all instructions that affect or are affected by the value of the variable under the cursor.

In the context of vulnerability research, slice highlighting provides a way to quickly identify the scope of control an attacker has when they control a particular variable or alternatively which variables they would need to control to influence the value of a variable.

#### Read More about Ghidra

[Ghidra 101: Cursor Text Highlighting](https://www.tripwire.com/state-of-security/security-data-protection/cyber-security/ghidra-101-cursor-text-highlighting/)
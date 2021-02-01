> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.tripwire.com](https://www.tripwire.com/state-of-security/security-data-protection/cyber-security/ghidra-101-cursor-text-highlighting/)

In this blog series, I will be putting the spotlight on useful Ghidra features that you may have missed. Each post will look at a different feature and show how it helps you save time and be more effective while reverse engineering. Ghidra is an incredibly powerful tool, but much of this power comes from knowing how to use it effectively.

What is Cursor Text Highlighting?
---------------------------------

In this post, I will be discussing the Cursor Text Highlighting feature. By default, this feature is activated by clicking on text in the Listing or Decompiler view with the middle mouse button (scroll wheel for most of us). As you might have guessed from the name, this feature can highlight occurrences of a string within the Listing or Decompiler view. This is a rather standard feature we’ve grown to expect in any basic graphical IDE, but use Ghidra’s Cursor Text Highlighting on a register in the Listing view and you might notice it doing something more. When using this feature to follow register usage, Ghidra will color the register according to whether it is being read from or being written to.

Text highlighting can draw your attention to how an address or label is being used, especially in the early stages of a reversing workflow where many variables may share similar names. As noted in the [Ghidra issue tracker](https://github.com/NationalSecurityAgency/ghidra/issues/25), this is a useful feature, but it is not particularly well advertised, and having it tied to the middle-click by default means that many users never notice it on their own.

It is important to understand, however, that there are two distinct matching behaviors:

1.  Clicking on a register highlights use of that register with the scoped read/write colors configured in the Tool Options.
2.  Clicking on any other text will highlight other occurrences of this string in the highlight color specified under the Tool Options. This direct string search will highlight text throughout the view including in comments.

Highlights are made within the active view as shown here:

![](https://3b6xlt3iddqmuq5vy2w0s5d3-wpengine.netdna-ssl.com/state-of-security/wp-content/uploads/sites/3/Ghidra-101-active-view-800x332.png)

For systems without a middle-mouse button, the feature can be reassigned via the Tool Options to use the left or right mouse button instead of the middle.

That’s everything you need to know to start utilizing cursor text and scope highlighting in Ghidra. Keep an eye on the [State of Security Blog](https://info.tripwire.com/state-of-security-subscription-center) for my next Ghidra 101 post in which I’ll be going over how to highlight program slices to reveal relationships between variable values.

Later on in this series, I’ll be looking at automatically creating data structures, recovering stack strings, applying multi-level table sorts, and more.
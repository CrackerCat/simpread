> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.ret2.io](https://blog.ret2.io/2020/09/11/lucid-hexrays-microcode-explorer/)

Recently, we blogged about the [Hex-Rays microcode](https://ret2.io/2020/07/22/ida-pro-avx-decompiler/#the-hex-rays-microcode) that powers the [IDA Pro](https://www.hex-rays.com/products/ida/) decompiler. We showed how a few days spent hacking on the microcode API could dramatically reduce the cost of certain reverse engineering tasks. But developing for the microcode API can be challenging due to the limited examples to crib from, and the general complexity of working with decompiler internals.

Today, we are publishing a developer-oriented plugin for IDA Pro called [Lucid](https://github.com/gaasedelen/lucid). Lucid is an interactive Hex-Rays microcode explorer that makes it effortless to study the optimizations of the decompilation pipeline. We use it as an aid for developing and debugging microcode-based augmentations.

[![](https://blog.ret2.io/assets/img/lucid_demo.gif)](https://blog.ret2.io/assets/img/lucid_demo.gif)

Lucid is a Hex-Rays microcode explorer for microcode plugin developers

Usage[](#usage)
---------------

Lucid requires IDA 7.5. It will automatically load for any architecture with a Hex-Rays decompiler present. Simply right click anywhere in a Pseudocode window and select `View microcode` to open the Lucid Microcode Explorer.

[![](https://blog.ret2.io/assets/img/lucid_view_microcode.gif)](https://blog.ret2.io/assets/img/lucid_view_microcode.gif)

Lucid adds a right click 'View microcode' context menu entry to Hex-Rays Pseudocode windows

By default, the Microcode Explorer will synchronize with the last active Hex-Rays Pseudocode window. This can be toggled on / off in the ‘Settings’ groupbox of the explorer window.

‘Basically Magic’[](#basically-magic)
-------------------------------------

Lucid’s advantage over the existing microcode explorers is that it is backed by a complete text-to-microcode structure mapping. This means that a cursor position in the rendered microcode text can be used to retrieve the underlying microinstruction, sub-instruction, or sub-operand under the user cursor (and vice versa).

As the chief example, we use these mappings to help the explorer ‘project’ the user cursor through the layers, maintaining focus on the selected instruction or operand as it flows through the decompilation pipeline:

[![](https://blog.ret2.io/assets/img/lucid_blog_demo.gif)](https://blog.ret2.io/assets/img/lucid_blog_demo.gif)

Scrolling through the microcode maturity layers while tracking a specific operand

At the end of the day, the intention is to provide more natural experiences for studying the microcode and developing related tooling. This is just one example of how we might _‘get there.’_

Sub-instruction Graphs[](#sub-instruction-graphs)
-------------------------------------------------

Lucid was originally created to serve as an interactive platform for lifting late-stage microcode expressions and generalizing them into ‘optimization’ patterns for the decompiler (think inline call detection/rewriting).

While I am not sure I’ll find the time/motivation to revisit this area of research, Lucid does include a rudimentary feature (inspired by [genmc](https://github.com/patois/genmc)) to view the sub-instruction tree of a given microinstruction which is still useful by itself.

[![](https://blog.ret2.io/assets/img/lucid_subtree.gif)](https://blog.ret2.io/assets/img/lucid_subtree.gif)

Viewing the sub-instruction tree of a given microinstruction

You can view these individual trees by right clicking an instruction and selecting `View subtree`.

Bits and Bobs[](#bits-and-bobs)
-------------------------------

The code for Lucid is available on [github](https://github.com/gaasedelen/lucid) where it has been licensed permissively under the MIT license. As the initial release, the codebase is a bit messy and the `README` contains a few known issues/bugs at the time of publication.

Finally, there is no regular development scheduled for this plugin (outside of maintenance) but I always welcome external contributions, issues, and feature requests.

Conclusion[](#conclusion)
-------------------------

In this post, we presented a new IDA Pro plugin called [Lucid](https://github.com/gaasedelen/lucid). It is a developer-oriented plugin designed to aid in the research and development of microcode-based plugins/extensions for the Hex-Rays decompiler.

Our experience developing for these technologies is second to none. [RET2](https://ret2.io/) is happy to consult in these spaces, providing plugin development services, the addition of custom features to existing works, or other unique opportunities with regard to security tooling. If your organization has a need for this expertise, please feel free to [reach out](https://ret2.io/contact).
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.ret2.io](https://blog.ret2.io/2021/04/20/tenet-trace-explorer/)

Debugging is traditionally a tedious, monotonous endeavor. While some people love the archaeological process of using a debugger to uncover software defects or perform tasks in software reverse engineering, virtually everyone agrees that the tools have started to show their age against modern software.

Our methods of runtime introspection are still hyper-focused on individual states of execution. This is a problem because software has grown dramatically more complex, requiring more context and ‘situational awareness’ to properly interpret. The emergence of [timeless debugging](https://blog.ret2.io/2018/06/19/pwn2own-2018-root-cause-analysis/) has alleviated some of these growing pains, but these solutions are still built around conventional methods of inspecting individual states rather than how they relate to one another.

I am open sourcing [Tenet](https://github.com/gaasedelen/tenet), an [IDA Pro](https://www.hex-rays.com/products/ida/) plugin for exploring execution traces. The goal of this plugin is to provide more natural, human controls for navigating execution traces against a given binary. The basis of this work stems from the desire to research new or innovative methods to examine and distill complex execution patterns in software.

[![](https://blog.ret2.io/assets/img/tenet_overview.gif)](https://blog.ret2.io/assets/img/tenet_overview.gif)

Tenet is an experimental plugin for exploring software execution traces in IDA Pro

Background[](#background)
-------------------------

Tenet is directly inspired by [QIRA](http://qira.me/). The earliest prototypes I wrote for this date back to 2015 when I was working as a software security engineer at Microsoft. These prototypes were built on top of the private ‘trace reader’ APIs for [TTD](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/time-travel-debugging-overview), previously known as TTT, or iDNA.

I abandoned that work because I wasn’t convinced an assembly level trace explorer would make sense at the one company in the world where Windows is called _‘open source software’_.

I’m revisiting the idea of a trace explorer because it’s 2021 and the space is still **wildly** undeveloped. Among other ideas, I firmly believe in the notion that there is a geographical landscape to program execution, and that the principles of traditional cartography can be applied to the summarization, illustration, and navigation of these landscapes.

As cool as I hope you find the plugin, Tenet is only a stepping stone to further research these ideas.

Usage[](#usage)
---------------

Tenet can be downloaded or cloned from [GitHub](https://github.com/gaasedelen/tenet). It requires IDA 7.5 and Python 3.

Once properly [installed](https://github.com/gaasedelen/tenet#installation), there will be a new menu entry available in the disassembler. This can be used to load externally-collected execution traces into Tenet.

[![](https://blog.ret2.io/assets/img/tenet_load_trace.gif)](https://blog.ret2.io/assets/img/tenet_load_trace.gif)

Tenet adds a menu entry to the IDA --> Load file submenu to load execution traces

As this is the initial release, Tenet only accepts simple human-readable text traces. Please refer to the [tracing readme](https://github.com/gaasedelen/tenet/tree/master/tracers) for additional information on the trace format, limitations, and reference tracers.

Bidirectional Exploration[](#bidirectional-exploration)
-------------------------------------------------------

While using Tenet, the plugin will ‘paint’ trails to indicate the flow of execution forwards (blue) and backwards (red) from your present position in the active execution trace.

[![](https://blog.ret2.io/assets/img/tenet_trails.gif)](https://blog.ret2.io/assets/img/tenet_trails.gif)

Tenet provides locality to past and present execution flow while navigating the trace

To `step` forwards or backwards through time, you simply _scroll while hovering over the timeline_ on the right side of the disassembler. To `step over` function calls, hold `SHIFT` while scrolling.

Trace Timeline[](#trace-timeline)
---------------------------------

The trace timeline will be docked on the right side of the disassembler. This widget is used to visualize different types of events along the trace timeline and perform basic navigation as described above.

[![](https://blog.ret2.io/assets/img/tenet_timeline.gif)](https://blog.ret2.io/assets/img/tenet_timeline.gif)

The trace timeline is a key component to navigating Tenet traces and focusing on areas of interest

By _clicking and dragging across the timeline_, it is possible to zoom in on a specific section of the execution trace. This action can be repeated any number of times to reach the desired granularity.

Execution Breakpoints[](#execution-breakpoints)
-----------------------------------------------

Clicking the instruction pointer in the registers window will highlight it in red, revealing all the locations the instruction was executed across the trace timeline.

[![](https://blog.ret2.io/assets/img/tenet_breakpoints.gif)](https://blog.ret2.io/assets/img/tenet_breakpoints.gif)

Placing a breakpoint on the current instruction and navigating between its executions by scrolling

To jump between executions, _scroll up or down while hovering the highlighted instruction pointer_.

Additionally, you can _right click in the disassembly listing_ and select one of the navigation-based menu entries to quickly seek to the execution of an instruction of interest.

[![](https://blog.ret2.io/assets/img/tenet_seek_to_first.gif)](https://blog.ret2.io/assets/img/tenet_seek_to_first.gif)

Using a Tenet navigation shortcut to seek the trace to the first execution of the given instruction

IDA’s native `F2` hotkey can also be used to set breakpoints on arbitrary instructions.

Memory Breakpoints[](#memory-breakpoints)
-----------------------------------------

By clicking a byte in either the stack or memory views, you will instantly see all reads/writes to that address visualized across the trace timeline. Yellow indicates a memory _read_, blue indicates a memory _write_.

[![](https://blog.ret2.io/assets/img/tenet_memory_breakpoint.gif)](https://blog.ret2.io/assets/img/tenet_memory_breakpoint.gif)

Seeking the trace across states of execution, based on accesses to a selected byte of memory

Memory breakpoints can be navigated using the same technique described for execution breakpoints. Click a byte, and _scroll while hovering the selected **byte**_ to seek the trace to each of its accesses.

_Right clicking a byte_ of interest will give you options to seek between memory read / write / access if there is a specific navigation action that you have in mind.

[![](https://blog.ret2.io/assets/img/tenet_memory_seek.png)](https://blog.ret2.io/assets/img/tenet_memory_seek.png)

Tenet provides a number of navigation shortcuts for memory accesses

To navigate the memory view to an arbitrary address, click onto the memory view and hit `G` to enter either an address or database symbol to seek the view to.

Region Breakpoints[](#region-breakpoints)
-----------------------------------------

A rather experimental feature is setting access breakpoints for a region of memory. This is possible by highlighting a block of memory, and selecting the _Find accesses_ action from the right click menu.

[![](https://blog.ret2.io/assets/img/tenet_region_breakpoint.gif)](https://blog.ret2.io/assets/img/tenet_region_breakpoint.gif)

Tenet allows you to set a memory access breakpoint over a region of memory, and navigate between its accesses

As with normal memory breakpoints, hovering the region and _scrolling_ can used to traverse between the accesses made to the selected region of memory.

Register Seeking[](#register-seeking)
-------------------------------------

In reverse engineering, it’s pretty common to encounter situations where you ask yourself _“Which instruction set this register to its current value?”_

Using Tenet, you can seek backwards to that instruction in a single click.

[![](https://blog.ret2.io/assets/img/tenet_seek_to_register.gif)](https://blog.ret2.io/assets/img/tenet_seek_to_register.gif)

Seeking to the timestamp responsible for setting a register of interest

Seeking backwards is by far the most common direction to navigate across register changes… but for dexterity you can also seek forward to the next register assignment using the blue arrow on the right of the register.

Timestamp Shell[](#timestamp-shell)
-----------------------------------

A simple ‘shell’ is provided to navigate to specific timestamps in the trace. Pasting (or typing…) a timestamp into the shell with or without commas will suffice.

[![](https://blog.ret2.io/assets/img/tenet_idx_shell.gif)](https://blog.ret2.io/assets/img/tenet_idx_shell.gif)

The 'timestamp shell' can be used to navigate the trace reader to a specific timestamp

Using an exclamation point, you can also seek a specified ‘percentage’ into the trace. Entering `!100` will seek to the final instruction in the trace, where `!50` will seek approximately 50% of the way through the trace.

Themes[](#themes)
-----------------

Tenet ships with two default themes – a ‘light’ theme, and a ‘dark’ one. Depending on the colors currently used by your disassembler, Tenet will attempt to select the theme that seems most appropriate.

[![](https://blog.ret2.io/assets/img/tenet_themes.png)](https://blog.ret2.io/assets/img/tenet_themes.png)

Tenet has both light, and dark themes that will be selected based on the user's disassembly theme

The theme files are stored as simple JSON on disk and are highly configurable. If you are not happy with the default themes or colors, you can create your own themes and simply drop them in the user theme directory.

Tenet will remember your theme preference for future loads and uses.

Open Source[](#open-source)
---------------------------

Tenet is currently an unfunded research project. It is available for free on [GitHub](https://github.com/gaasedelen/tenet), published under the MIT license. In the public README, there is additional information on how to get started, future ideas, and even an FAQ that have not been covered in this post.

Without funding, the time I can devote to this project is limited. If your organization is excited by the ideas put forth here and capable of providing capital to sponsor dedicated development time towards Tenet, please [contact us](https://ret2.io/contact).

There aren’t many robust projects (or products) in this space, and it’s important to raise awareness for the few that exist. If you found this blogpost interesting, please consider exploring and supporting more of these technologies:

*   [QIRA](https://github.com/geohot/qira) – QEMU Interactive Runtime Analyser
*   [Microsoft TTD](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/time-travel-debugging-overview) – Time Travel Debugging for Windows
*   [RR](https://github.com/rr-debugger/rr) // [Pernosco](https://www.pernos.co/) – Timeless debugging for Linux
*   [PANDA](https://github.com/panda-re/panda) – Full system tracing, built on QEMU
*   [REVEN](https://www.tetrane.com/) – Full system tracing, built on PANDA / VirtualBox
*   [UndoDB](https://undo.io/solutions/products/udb/) – Reversible debugging for Linux and Android
*   [DejaVM](https://www.raytheonintelligenceandspace.com/capabilities/products/dejavm) – :^)

Conclusion[](#conclusion)
-------------------------

In this post, we presented a new IDA Pro plugin called [Tenet](https://github.com/gaasedelen/tenet). It is an experimental plugin designed to explore software execution traces. It can be used both as an aid in the reverse engineering process, and a research technology to explore new ideas in program analysis, visualization, and the efficacy of next generation debugging experiences.

Our experience developing for these technologies is second to none. [RET2](https://ret2.io/) is happy to consult in these spaces, providing plugin development services, the addition of custom features to existing works, or other unique opportunities with regard to security tooling. If your organization has a need for this expertise, please feel free to [reach out](https://ret2.io/contact).
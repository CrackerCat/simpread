> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [secrary.com](https://secrary.com/Random/InstrumentationCallback/)

> https://secrary.com - Does it matter?

I want to write about very interesting epilogue hooking method presented by `Alex Ionescu` at [`REcon 2015`](https://www.youtube.com/watch?v=bqU0y4FzvT0).

> An epilogue detour allows for post-processing. They are useful for filtering output parameters once an original routine has performed its duties.

I know it’s not a new technique, but I found this very interesting, also all POC codes, which I found crashes.

I tried to create POC which does not crashes (at least for me) and ends normally (by the way, it’s EXE, not DLL)

[KPROCESS](https://channel9.msdn.com/Shows/Going+Deep/Arun-Kishan-Process-Management-in-Windows-Vista) structure contains field called `InstrumentationCallback` at `0x2c8`:

`Windows Vista` and later you can specify callback address using `InstrumentationCallback` field and the callback will be called after each time any function returns from kernel to user mode.

One way to specify callback address is by using a driver, but as turns out there is a much easier way via `NtSetInformationProcess` API from user mode without any special privileges, we just need to specify correct structures and that’s all.

There are pitfalls, we have to use assembly inside our code for the callback function, due to we need to deal with registers directly.

That’s all :) Works on `Windows 10 v1709 x64`

You can get POC code from [GitHub](https://github.com/secrary/Hooking-via-InstrumentationCallback)

![](https://user-images.githubusercontent.com/16405698/33432212-8dd030da-d5f0-11e7-9524-aa23246aeebe.gif)

My twitter: [@_qaz_qaz](https://twitter.com/_qaz_qaz)

Resourses:

*   [Hooking Nirvana](https://www.youtube.com/watch?v=bqU0y4FzvT0)
*   [Windows x64 system service hooks and advanced debugging](https://www.codeproject.com/Articles/543542/Windows-x-system-service-hooks-and-advanced-debu)
*   [Anti-Dbg trick](https://pastebin.com/9TqRGsM5)
*   [Windows 10 Hooking Nirvana explained](https://sww-it.ru/2016-04-11/1332)
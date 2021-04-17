> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [jumpnowtek.com](https://jumpnowtek.com/shellcode/linux-arm-shellcode-part0.html)

These are some notes I was gathering for a talk on Linux ARM [shellcode](https://en.wikipedia.org/wiki/Shellcode) to present at a local [OWASP](https://www.meetup.com/OWASP-Maine/) meetup.

Some familiarity with Linux systems programing is assumed, but not necessarily any ARM assembly knowledge.

Below were my primary references putting together these posts

*   Computerphile [Buffer Overflow Attack](https://www.youtube.com/watch?v=1S0aBV-Waeo) video - General overview and a good start
*   [Azeria Labs](https://azeria-labs.com/) - Probably the best reference on ARM specific shellcoding
*   [ARM Systems Developers Guide](https://www.elsevier.com/books/arm-system-developers-guide/sloss/978-1-55860-874-0) - Assembly reference book
*   [The Shellcoder’s Handbook](https://www.wiley.com/en-us/The+Shellcoder%27s+Handbook%3A+Discovering+and+Exploiting+Security+Holes%2C+2nd+Edition-p-9780470080238) - Discovering and Exploiting Security Holes, 2nd Edition
*   Shellstorm [Shellcode on ARM](http://shell-storm.org/blog/Shellcode-On-ARM-Architecture/) - Tutorial with link to a database of examples
*   [Shellcoding for Linux and Windows Tutorial](http://www.vividmachines.com/shellcode/shellcode.html) - Focus on x86 but good explanations
*   [Billy Ellis](https://billy-ellis.github.io/) - Focus on iOS targets
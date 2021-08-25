> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.orange.tw](https://blog.orange.tw/2021/08/proxyshell-a-new-attack-surface-on-ms-exchange-part-3.html)

> This is 🍊 speaking

_Author: Orange Tsai([@orange_8361](https://twitter.com/orange_8361)) from DEVCORE  
P.S. This is a cross-post blog from [Zero Day Initiative (ZDI)](https://www.zerodayinitiative.com/blog/2021/8/17/from-pwn2own-2021-a-new-attack-surface-on-microsoft-exchange-proxyshell)_

This is a guest post DEVCORE collaborated with Zero Day Initiative (ZDI) and published at their blog, which describes the exploit chain we demonstrated at Pwn2Own 2021!  Please visit the following link to read that :)

*   [FROM PWN2OWN 2021: A NEW ATTACK SURFACE ON MICROSOFT EXCHANGE - PROXYSHELL!](https://www.zerodayinitiative.com/blog/2021/8/17/from-pwn2own-2021-a-new-attack-surface-on-microsoft-exchange-proxyshell)

If you are interesting in more Exchange Server attacks, please check the following articles:

*   [A New Attack Surface on MS Exchange Part 1 - ProxyLogon!](https://blog.orange.tw/2021/08/proxylogon-a-new-attack-surface-on-ms-exchange-part-1.html)
*   [A New Attack Surface on MS Exchange Part 2 - ProxyOracle!](https://blog.orange.tw/2021/08/proxyoracle-a-new-attack-surface-on-ms-exchange-part-2.html)
*   [A New Attack Surface on MS Exchange Part 3 - ProxyShell!](https://blog.orange.tw/2021/08/proxyshell-a-new-attack-surface-on-ms-exchange-part-3.html)
*   A New Attack Surface on MS Exchange Part 4 (coming soon…)

With ProxyShell, an unauthenticated attacker can execute arbitrary commands on Microsoft Exchange Server through an exposed 443 port! Here is the [demonstration video](https://youtu.be/FC6iHw258RI):
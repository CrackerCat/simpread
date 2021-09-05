> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [devco.re](https://devco.re/blog/2019/06/21/operation-crack-hacking-IDA-Pro-installer-PRNG-from-an-unusual-way-en/)

> Is it possible to install IDA Pro without kowning installation password? Linux or MacOS version can f......

[English Version](https://devco.re/blog/2019/06/21/operation-crack-hacking-IDA-Pro-installer-PRNG-from-an-unusual-way-en/)  
[中文版本](https://devco.re/blog/2019/06/21/operation-crack-hacking-IDA-Pro-installer-PRNG-from-an-unusual-way/)

Introduction
------------

Today, we are going to talk about the installation password of Hex-Rays IDA Pro, which is the most famous decompiler. What is installation password? Generally, customers receive a custom installer and installation password after they purchase IDA Pro. The installation password is required during installation process. However, if someday we find a leaked IDA Pro installer, is it still possible to install without an installation password? This is an interesting topic.

After brainstorming with our team members, we verified the answer: Yes! With a Linux or MacOS version installer, we can easily find the password directly. With a Windows version installer, we only need 10 minutes to calculate the password. The following is the detailed process:

### * Linux and MacOS version

The first challenge is Linux and MacOS version. The installer is built with an installer creation tool called InstallBuilder. We found the plaintext installation password directly in the program memory of the running IDA Pro installer. Mission complete!

![](https://devco.re/assets/img/blog/20190621/1.png)

This problem is fixed after we reported through Hex-Rays. [BitRock](https://blog.bitrock.com/2019/02/installbuilder-1920-released.html) released InstallBuilder 19.2.0 with the protection of installation password on 2019/02/11.

### * Windows version

It gets harder on Windows version because the installer is built with [Inno Setup](http://www.jrsoftware.org/isinfo.php), which store its password with [160-bit SHA-1 hash](http://www.jrsoftware.org/ishelp/index.php?topic=setup_password). Therefore, we cannot get the password simply with static or dynamic analyzing the installer, and brute force is apparently not an effective way. But the situation is different if we can grasp the methodology of password generation, which lets us enumerate the password more effectively!

Although we have realized we need to find how Hex-Rays generate password, it is still really difficult, as we do not know what language the random number generator is implemented with. There are at least [88 random number generators](https://rosettacode.org/wiki/Random_number_generator_(included)) known. It is such a great variation.

We first tried to find the charset used by random number generator. We collected all leaked installation passwords, such as hacking team’s password, which is leaked by WikiLeaks.

*   FgVQyXZY2XFk ([link](https://wikileaks.org/hackingteam/emails/emailid/62729))
*   7ChFzSbF4aik ([link](https://wikileaks.org/hackingteam/emails/emailid/311769))
*   ZFdLqEM2QMVe ([link](https://wikileaks.org/hackingteam/emails/emailid/62956))
*   6VYGSyLguBfi ([link](https://wikileaks.org/hackingteam/emails/emailid/70250))

From the collected passwords we can summarize the charset:  
`23456789ABCDEFGHJKLMPQRSTUVWXYZabcdefghijkmpqrstuvwxyz`

The missing of `1`, `I`, `l`, `0`, `O`, `o`, `N`, `n` seems to make sense because they are confusing characters.  
Next, we guess the possible charset ordering like these:

```
23456789ABCDEFGHJKLMPQRSTUVWXYZabcdefghijkmpqrstuvwxyz
ABCDEFGHJKLMPQRSTUVWXYZ23456789abcdefghijkmpqrstuvwxyz
23456789abcdefghijkmpqrstuvwxyzABCDEFGHJKLMPQRSTUVWXYZ
abcdefghijkmpqrstuvwxyz23456789ABCDEFGHJKLMPQRSTUVWXYZ
abcdefghijkmpqrstuvwxyzABCDEFGHJKLMPQRSTUVWXYZ23456789
ABCDEFGHJKLMPQRSTUVWXYZabcdefghijkmpqrstuvwxyz23456789


```

Lastly, we picked some common languages（c/php/python/perl）to implement a random number generator and enumerate all the combinations. Then we examined whether the collected passwords appears in the combinations. For example, here is a generator written in C language:

```
#include<stdio.h>
#include<stdlib.h>

char _a[] = "23456789ABCDEFGHJKLMPQRSTUVWXYZabcdefghijkmpqrstuvwxyz";
char _b[] = "ABCDEFGHJKLMPQRSTUVWXYZ23456789abcdefghijkmpqrstuvwxyz";
char _c[] = "23456789abcdefghijkmpqrstuvwxyzABCDEFGHJKLMPQRSTUVWXYZ";
char _d[] = "abcdefghijkmpqrstuvwxyz23456789ABCDEFGHJKLMPQRSTUVWXYZ";
char _e[] = "abcdefghijkmpqrstuvwxyzABCDEFGHJKLMPQRSTUVWXYZ23456789";
char _f[] = "ABCDEFGHJKLMPQRSTUVWXYZabcdefghijkmpqrstuvwxyz23456789";

int main()
{
        char bufa[21]={0};
        char bufb[21]={0};
        char bufc[21]={0};
        char bufd[21]={0};
        char bufe[21]={0};
        char buff[21]={0};

        unsigned int i=0;
        while(i<0x100000000)
        {
                srand(i);

                for(size_t n=0;n<20;n++)
                {
                        int key= rand() % 54;
                        bufa[n]=_a[key];
                        bufb[n]=_b[key];
                        bufc[n]=_c[key];
                        bufd[n]=_d[key];
                        bufe[n]=_e[key];
                        buff[n]=_f[key];

                }
                printf("%s\n",bufa);
                printf("%s\n",bufb);
                printf("%s\n",bufc);
                printf("%s\n",bufd);
                printf("%s\n",bufe);
                printf("%s\n",buff);
                i=i+1;
        }
}


```

After a month, we finally generated the IDA Pro installation passwords successfully with Perl, and the correct charset ordering is `abcdefghijkmpqrstuvwxyzABCDEFGHJKLMPQRSTUVWXYZ23456789`. For example, we can generate the hacking team’s leaked password `FgVQyXZY2XFk` with the following script:

```
#!/usr/bin/env perl
#
@_e = split //,"abcdefghijkmpqrstuvwxyzABCDEFGHJKLMPQRSTUVWXYZ23456789";

$i=3326487116;
srand($i);
$pw="";

for($i=0;$i<12;++$i)
{
        $key = rand 54;
        $pw = $pw . $_e[$key];
}
print "$i $pw\n";


```

With this, we can build a dictionary of installation password, which effectively increase the efficiency of brute force attack. Generally, we can compute the password of one installer in 10 minutes.

We have reported this issue to Hex-Rays, and they promised to harden the installation password immediately.

Summary
-------

In this article, we discussed the possibility of installing IDA Pro without owning installation password. In the end, we found plaintext password in the program memory of Linux and MacOS version. On the other hand, we determined the password generation methodology of Windows version. Therefore, we can build a dictionary to accelerate brute force attack. Finally, we can get one password at a reasonable time.

We really enjoy this process: surmise wisely and prove it with our best. It can broaden our experience no matter the result is correct or not. This is why we took a whole month to verify such a difficult surmise. We also take this attitude in our [Red Team Assessment](https://devco.re/en/services/red-team). You would love to give it a try!

Lastly, we would like to thank for the friendly and rapid response from Hex-Rays. Although this issue is not included in [Security Bug Bounty Program](https://www.hex-rays.com/bugbounty.shtml), they still generously awarded us IDA Pro Linux and MAC version, and upgraded the Windows version for us. We really appreciate it.

Timeline
--------

*   Jan 31, 2019 - Report to Hex-Rays
*   Feb 01, 2019 - Hex-Rays promised to harden the installation password and reported to BitRock
*   Feb 11, 2019 - BitRock released InstallBuilder 19.2.0
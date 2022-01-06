> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.zecops.com](https://blog.zecops.com/research/persistence-without-persistence-meet-the-ultimate-persistence-bug-noreboot/)

> Mobile Attacker’s Mindset Series – Part II Evaluating how attackers operate when there are no rules l......

Mobile Attacker’s Mindset Series – Part II
------------------------------------------

Evaluating how attackers operate when there are no rules leads to discoveries of advanced detection and response mechanisms. ZecOps is proudly researching scenarios of attacks and sharing the information publicly for the benefit of all the mobile defenders out there.

iOs persistence is presumed to be the hardest bug to find. The attack surface is somewhat limited and constantly analyzed by Apple’s security teams.

Creativity is a key element of the hacker’s mindset. Persistence can be hard if the attackers play by the rules. As you may have guessed it already – attackers are not playing by the rules and everything is possible.

In part II of the Attacker’s Mindset blog we’ll go over the ultimate persistence bug: a bug that cannot be patched because it’s not exploiting any persistence bugs at all – only playing tricks with the human mind.

Meet “NoReboot”: The Ultimate Persistence Bug
---------------------------------------------

We’ll dissect the iOS system and show how it’s possible to alter a shutdown event, tricking a user that got infected into thinking that the phone has been powered off, but in fact, it’s still running. The “NoReboot” approach simulates a real shutdown. The user cannot feel a difference between a real shutdown and a “fake shutdown”. There is no user-interface or any button feedback until the user turns the phone back “on”.

To demonstrate this technique, we’ll show a remote microphone & camera accessed after “turning off” the phone, and “persisting” when the phone will get back to a “powered on” state.

This blog can also be an excellent tutorial for anyone who may be interested in learning how to reverse engineer iOS.

Nowadays, many of us have tons of applications installed on our phones, and it is difficult to determine which among them is abusing our data and privacy. Constantly, our information is being collected, uploaded.

![](https://blog.zecops.com/wp-content/uploads/2021/12/persistence-noreboot-1.png)

This [story](https://arstechnica.com/information-technology/2019/08/armed-with-ios-0days-hackers-indiscriminately-infected-iphones-for-two-years/) by Dan Goodin, speaks about an iOS malware discovered in-the-wild. One of the sentences in the article says: “The installed malware…can’t persist after a device reboot, … phones are disinfected as soon as they’re restarted.”.

The reality is actually a bit more complicated than that. As we will be able to demonstrate in this blog, we cannot, and should not, trust a “normal reboot”.

### How Are We Supposed to Reboot iPhones?

According to [Apple](https://support.apple.com/en-us/HT201559), a phone is rebooted by clicking on the Volume Down + Power button and dragging the slider.

![](https://blog.zecops.com/wp-content/uploads/2021/12/persistence-noreboot-2.png)

Given that the iPhone has no internal fan and oftentimes it keeps its temperature cool, it’s not trivial to tell if our phones are running or not. For end-users, the most intuitive indicator that the phone is the feedback from the screen. We tap on the screen or click on the side button to wake up the screen.

Here is a list of physical feedback that constantly reminds us that the phone is powered on:

*   Ring/Sound from incoming calls and notifications
*   Touch feedback (3D touch)
*   Vibration (silent mode switch triggers a burst of vibration)
*   Screen
*   Camera indicator

“NoReboot”: Hijacking the Shutdown Event
----------------------------------------

Let’s see if we can disable all of the indicators above while keeping the phone with the trojan still running. Let’s start by hijacking the shutdown event, which involves injecting code into three daemons.

![](https://blog.zecops.com/wp-content/uploads/2021/12/persistence-noreboot-3.png)

When you slide to power off, it is actually a system application **_/Applications/InCallService.app_** sending a shutdown signal to SpringBoard, which is a daemon that is responsible for the majority of the UI interaction.

We managed to hijack the signal by hooking the Objective-C method **_-[FBSSystemService shutdownWithOptions:]_**. Now instead of sending a shutdown signal to SpringBoard, it will notify both _SpringBoard_ and _backboardd_ to trigger the code we injected into them.

![](https://blog.zecops.com/wp-content/uploads/2021/12/persistence-noreboot-4.png)

In backboardd, we will hide the spinning wheel animation, which automatically appears when SpringBoard stops running, the magic spell which does that is **_[[BKSDefaults localDefaults]setHideAppleLogoOnLaunch:1]_**. Then we make SpringBoard exit and block it from launching again. Because SpringBoard is responsible for responding to user behavior and interaction, without it, the device looks and feels as if it is not powered on. which is the perfect disguise for the purpose of mimicking a fake poweroff.

![](https://blog.zecops.com/wp-content/uploads/2021/12/persistence-noreboot-5.png)_Example of SpringBoard respond to user’s interaction: Detects the long press action and evokes Siri_

Despite that we disabled all physical feedback, the phone still remains fully functional and is capable of maintaining an active internet connection. The malicious actor could remotely manipulate the phone in a blatant way without worrying about being caught because the user is tricked into thinking that the phone is off, either being turned off by the victim or by malicious actors using “low battery” as an excuse. 

Later we will demonstrate eavesdropping through cam & mic while the phone is “off”. In reality, malicious actors can do anything the end-user can do and more. 

System Boot In Disguise
-----------------------

Now the user wants to turn the phone back on. The system boot animation with Apple’s logo can convince the end-user to believe that the phone has been turned off. 

When SpringBoard is not on duty, backboardd is in charge of the screen. According to the description we found on theiphonewiki regarding backboardd.

![](https://blog.zecops.com/wp-content/uploads/2021/12/persistence-noreboot-6.png)_Ref:_ [_https://www.theiphonewiki.com/wiki/Backboardd_](https://www.theiphonewiki.com/wiki/Backboardd)

_“All touch events are first processed by this daemon, then translated and relayed to the iOS application in the foreground”._ We found this statement to be accurate. Moreover, backboardd not only relay touch events, also physical button click events. 

![](https://blog.zecops.com/wp-content/uploads/2021/12/persistence-noreboot-7.png)

backboardd logs the exact time when a button is pressed down, and when it’s been released. 

![](https://blog.zecops.com/wp-content/uploads/2021/12/persistence-noreboot-8.png)

With the help from cycript, We noticed a way that allows us to intercept that event with Objective-C Method Hooking. 

A **__BKButtonEventRecord_** instance will be created and inserted into a global dictionary object **_BKEventSenderUsagePairDictionary_**.  We hook the insertion method when the user attempts to “turn on” the phone.

![](https://blog.zecops.com/wp-content/uploads/2021/12/persistence-noreboot-9.png)

The file will unleash the SpringBoard and trigger a special code block in our injected dylib. What it does is to leverage local SSH access to gain root privilege, then we execute **_/bin/launchctl reboot userspace._** This will exit all processes and restart the system without touching the kernel. The kernel remains patched. Hence malicious code won’t have any problem continuing to run after this kind of reboot.

![](https://blog.zecops.com/wp-content/uploads/2021/12/persistence-noreboot-10.png)

The user will see the Apple Logo effect upon restarting. This is handled by backboardd as well. Upon launching the SpringBoard, the backboardd lets SpringBoard take over the screen.

![](https://blog.zecops.com/wp-content/uploads/2021/12/persistence-noreboot-11.png)

From that point, the interactive UI will be presented to the user. Everything feels right as all processes have indeed been restarted. Non-persistent threats achieved “persistency” without persistence exploits.

### **Hijacking the Force Restart Event?**

A user can perform a “force restart” by clicking rapidly on “Volume Up”, then “Volume Down”, then long press on the power button until the Apple logo appears.

We have not found an easy way to hijack the force restart event. This event is implemented at a much lower level. According to the [post below](https://apple.stackexchange.com/questions/216402/does-a-force-restart-in-ios-do-anything-different-from-a-normal-restart), it is done at a hardware level. Following a brief search in the iOS kernel, we can confirm that we didn’t see what triggers the force-restart event. The good news is that it’s harder for malicious actors to disable force restart events, but at the same time end-users face a risk of data loss as the system does not have enough time to securely write data to disk in case of force-restart events.

![](https://blog.zecops.com/wp-content/uploads/2021/12/persistence-noreboot-12.png)

Misleading Force Restart
------------------------

Nevertheless, It is entirely possible for malicious actors to observe the user’s attempt to perform a  force-restart (via backboardd) and deliberately make the Apple logo appear a few seconds earlier, deceiving the user into releasing the button earlier than they were supposed to. Meaning that in this case, the end-user did not successfully trigger a force-restart.  We will leave this as an exercise for the reader.

![](https://blog.zecops.com/wp-content/uploads/2021/12/persistence-noreboot-13.png)_Ref:_ [_https://support.apple.com/guide/iphone/force-restart-iphone-iph8903c3ee6/ios_](https://support.apple.com/guide/iphone/force-restart-iphone-iph8903c3ee6/ios)

NoReboot Proof of Concept
-------------------------

You can find the source code of NoReboot POC [**here**](https://github.com/ZecOps/public/tree/master/fake_shutdown_POC).

Never trust a device to be off
------------------------------

Since iOS 15, Apple introduced a new feature allowing users to track their phone even when it’s been turned off. Malware researcher @naehrdine [wrote](https://naehrdine.blogspot.com/2021/09/always-on-processor-magic-how-find-my.html) a technical analysis on this feature and shared her opinion on “Security and privacy impact”. We agree with her on “Never trust a device to be off, until you removed its battery or even better put it into a Blender.”

![](https://blog.zecops.com/wp-content/uploads/2021/12/persistence-noreboot-14.png)

Checking if your phone is compromised
-------------------------------------

ZecOps for Mobile leverages extended data collection and enables responding to security events. If you’d like to inspect your phone – please feel free to request a free trial [here](https://www.zecops.com/contact/free-trial?utm_medium=referral&tum_source=blog&utm_campaign=mobile-attacker-mindset-2).
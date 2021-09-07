> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.takemyhand.xyz](http://blog.takemyhand.xyz/2021/02/android-task-hijacking-with.html)

> What is task hijacking in Android? Task hijacking and it's impact in Android was first presented in 2......

What is task hijacking in Android?
----------------------------------

Task hijacking and it's impact in Android was first presented in 2015 at [USENIX](https://www.usenix.org/sites/default/files/conference/protected-files/sec15_slides_ren.pdf). It refers to an attack wherein a malicious app takes over the "back stack" of the vulnerable app, and thereafter whenever the user tries to open the vulnerable app, he will instead by greeted by the activity of the malicious app.

What are tasks and back stacks?
-------------------------------

Android developer's documentation states -"A **task** is a collection of activities that users interact with when performing a certain job. The activities are arranged in a stack—called the **back stack**—in the order in which each activity is opened. So when a user runs an application, and goes from activity 1 to activity 2, and finally to activity 3 - when the user presses the Back button, the current activity is popped from the top of the stack (the activity 3 is destroyed) and the previous activity (activity 2) resumes (the previous state of its UI is restored). Activities in the stack are never rearranged, only pushed and popped from the stack—pushed onto the stack when started by the current activity and popped off when the user leaves it using the Back button. As such, the back stack operates as a "last in, first out" object structure."

<table><tbody><tr><td><img class="" src="https://developer.android.com/images/fundamentals/diagram_backstack.png"></td></tr><tr><td>From Android developer documentation</td></tr></tbody></table>

Android launch modes
--------------------

Launch modes are activity attributes that are specified in the AndroidManifest.xml (or mentioned as flags in the calling intent). They provide the underlying OS an instruction on how the activity should be launched. There are four modes that work in conjunction with activity flags (FLAG_ACTIVITY_* constants) in Intent objects to determine what should happen when the activity is called upon to handle an intent. They are:

1.  standard
2.  singleTop
3.  singleTask
4.  singleInstance

For the attack described here, we are mostly concerned with the "**singleTask**" mode.

A "**singleTask**"activity allows other activities to be part of its task. It's always at the root of its task, but other activities (necessarily "standard" and "singleTop" activities) can be launched into that task.

Task hijacking with moveTaskToBack() function and excludeFromRecents attribute
------------------------------------------------------------------------------

Activities in android can call the moveTaskToBack() function, which will move the task containing this activity to the **back of the activity stack** (basically minimise the activity)

What does the **excludeFromRecents** attribute do? This attribute will simply hide an application from popping up in the overview screen.

Now let's look at how this can be used to create an exploit. An attacker can make an app (**com.tmh.attacker**) with the following features:

As you can see, the app has defined a **taskAffinity** for the entire application - this means that unless otherwise specified, all activities of this application will associate themselves with the task of **com.tmh.victim**

Let us look at the attack in action now, after installing the victim and attacker apps.

1.  When the user opens the attacker's app, it immediately minimises the task.
2.  As you can see in the GIF, this does not show up at all in the overview screen.
3.  After that, when the user opens the victim app, and presses the back button, instead of being taken to the home screen, he is taken to the attacker's application - thanks to the **taskAffinity** mentioned by the attacker's app which is set to the victim's app.

For demonstration purposes, I have explicitly named the app as "Attacker", but a real life exploitation can involve a very stealthy and convincing UI spoofing.

Preventing task hijacking
-------------------------

Setting taskAffinity="" can be a quick fix for this issue. The launch mode can also be set to **singleInstance** if the app does not want other activities to join tasks belonging to it. A custom **onBackPressed()** function can also be added, to override the default behaviour.
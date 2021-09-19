> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [nibarius.github.io](https://nibarius.github.io/learning-frida/2021/08/29/hpandro-hidden-levels)

> In the two previous blog posts (part1, part 2) we’ve tackled a bunch of the hpAndro Vulnerable Applic......

In the two previous blog posts ([part1](https://nibarius.github.io/learning-frida/2021/08/26/hpandro-part1), [part 2](https://nibarius.github.io/learning-frida/2021/08/28/hpandro-part2)) we’ve tackled a bunch of the [hpAndro Vulnerable Application](http://ctf.hpandro.raviramesh.info/) challenges, but there’s still a bit remaining.

After working with a couple challenges you’ll start to understand how the app works. Challenges that are available have a pale green background in the menu while challenges that are planned but have not yet been created have a white background. If you compare the list of challenges on the flag submission page with the levels available in the app you’ll spot something odd.

There are two levels that you can submit flags for even though they look unavailable in the app (in versions 1.1.12 - 1.1.15 at least). This is what we are going to be focusing on today.

![](https://nibarius.github.io/learning-frida/assets/hpandro/hidden_levels.png) This doesn't add up

Looking closer
--------------

If you open up the “QEMU” or the “Check Package Name” challenges they behave like many of the other incomplete challenges. There is a description there, but the task button does nothing when clicked. If we analyze the apk we can see that there are classes and an implementation available for both these challenges.

![](https://nibarius.github.io/learning-frida/assets/hpandro/qemu_code.png) There is some kind of implementation available for the QEMU challenge

It seems like the challenge is available, it’s just that the task button is not yet hooked up to the actual task activity. So to be able to solve these challenges we have to first implement the missing implementation and connect the task button with the appropriate activities.

Figuring out what to do
-----------------------

If you analyze the code and track down where the task button is handled you will eventually find your way to `HomeWebViewFragment.redirectToTask()`. Even though it’s a huge method with if statements for all challenges, it’s fairly simple. It basically just calls `startActivity` with an intent pointing to the correct activity.

![](https://nibarius.github.io/learning-frida/assets/hpandro/redirect_to_task.png) The start of the redirectToTask method

What we want to do is to replace the `redirectToTask()` method with one that can open the two hidden challenges. In Java it would look something like this:

```
    private final void redirectToTask() {
        String menuName = this.mMenuModel.getMenuName();
        if (Intrinsics.areEqual(menuName, "Check Package Name")) {
            startActivity(new Intent(getActivity(), PackageNamesActivity.class));
        } else if (Intrinsics.areEqual(menuName, "QEMU")) {
            startActivity(new Intent(getActivity(), QEMUDetectionActivity.class));
        } else {
            // the original implementation
        }
    }

```

Implementing the missing functionality
--------------------------------------

### Detecting which challenge is open

The first thing we need here is the menu name to know which task we are on. To get this we can start by calling the original `redirectToTask()` method and hook the `getMenuName()` method to pick up the return value from there.

```
  var lastMenuName = "";
  var act = Java.use('com.hpandro.androidsecurity.utils.fragment.HomeWebViewFragment');
  act.redirectToTask.implementation = function() {
    this.redirectToTask();
    ...
    var targetClass = null;
    if (lastMenuName == "Check Package Name") {
      targetClass = Java.use('com.hpandro.androidsecurity.ui.activity.task.emulatorDetection.PackageNamesActivity');
    }
    else if (lastMenuName == "QEMU") {
      targetClass = Java.use('com.hpandro.androidsecurity.ui.activity.task.emulatorDetection.QEMUDetectionActivity');
    }
    ...
  }
  
  Java.use('com.hpandro.androidsecurity.ui.menu.MenuModel').getMenuName.implementation = function() {
    var ret = this.getMenuName();  
    lastMenuName = ret;
    return ret;
  }

```

The nice thing with doing this is that since we call the original implementation all other challenges will be handled normally and we won’t break any other functionality. We only add new functionality for the two challenges that weren’t implemented.

### Connecting the Task button with the correct activity

With this in place we now need to call the `getActivity()` method to get the activity, create an Intent and then call `startActivity()` on it. To be able to call class methods a live instance of the `HomeWebViewFragment` class is needed which can be obtained by using the Frida function `Java.choose()`. With access to a live instance of the class it’s pretty straightforward to create the intent and start the activity.

```
    Java.choose('com.hpandro.androidsecurity.utils.fragment.HomeWebViewFragment', {
      onMatch: function(instance) {
        var activity = instance.getActivity();
        
        ... // Select targetClass as described above
        
        if (activity != null && targetClass != null) {
          var intent = Java.use('android.content.Intent').$new(activity, targetClass.class);
          instance.startActivity(intent);
        }
      },
      onComplete: function() {}
    });

```

With all of this in place it’s now possible to run the script with Frida and open the hidden challenges.

Solving the actual challenges
-----------------------------

First up is the QEMU challenge. Let’s click the “Detect emulator” button and see what happens.

![](https://nibarius.github.io/learning-frida/assets/hpandro/qemu_flag.png) We got the flag!?

Okay, we were given the flag right away without having to do any emulator check bypass. Let’s try with the package name check challenge… oh, same there as well. Maybe there’s a reason why these challenges are not generally available yet?

But hey, a flag is a flag and finding and solving these two challenges was probably my favorite part of the whole hpAndro CTF. I felt a great sense of accomplishment when I managed to solve these and being the first person who found these flags definitely added to the feeling.

![](https://nibarius.github.io/learning-frida/assets/hpandro/meme.png) This was my [feeling](https://twitter.com/NiklasBarsk/status/1415755131887497217) when the flags were accepted.

[The full code for these levels are available on GitHub](https://github.com/nibarius/learning-frida/blob/master/src/hpandro/emulator.js)
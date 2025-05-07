> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [rambo.codes](https://rambo.codes/posts/2025-04-24-how-a-single-line-of-code-could-brick-your-iphone)

> Gui Rambo writes about his coding and reverse engineering adventures.

This is the story of how I found one of my favorite iOS vulnerabilities so far. It’s one of my favorites because of how simple it was to implement an exploit for it. There’s also the fact that it uses a legacy public API that’s still relied upon by many components of Apple’s operating systems, and that many developers have never heard of.

Darwin Notifications
--------------------

Most iOS developers are likely used to [NSNotificationCenter](https://developer.apple.com/documentation/foundation/notificationcenter?language=objc), and most Mac developers are also likely used to [NSDistributedNotificationCenter](https://developer.apple.com/documentation/foundation/distributednotificationcenter?language=objc). The former only works within a single process, the latter allows simple notifications to be exchanged between processes, with the option to include a string with additional data to be transmitted alongside the notification.

[Darwin notifications](https://developer.apple.com/documentation/darwinnotify/darwin-notification-api) are even simpler, as they’re a part of the CoreOS layer. They provide a low-level mechanism for simple message exchange between processes on Apple’s operating systems. Instead of objects or strings, each notification may have a `state` associated with it, which is a `UInt64`, and typically is only used to indicate a boolean `true` or `false` by specifying `0` or `1`.

A simple use case for the API would be for a process that just wants to notify other processes about a given event, in which case it can call the `notify_post` function, which takes a string that’s usually a reverse DNS value like `com.apple.springboard.toggleLockScreen`.

Processes interested in receiving such a notification can register by using the `notify_register_dispatch` function, which will invoke a block on a given queue any time another process posts the notification with the specified name.

A process that’s interested in posting a Darwin notification with a state has to first register a handle for it, which can be done by calling the `notify_register_check` function, which takes the name of the notification and the pointer to an `Int32`, which is where the function returns a token that can be used to call `notify_set_state`, which also takes a `UInt64` value for the state.

Via the same `notify_register_check` mechanism, a process that wants to get the state of a notification can call `notify_get_state` to get its current state. This allows Darwin notifications to be used for certain types of events, but also hold some state that any process on the system can query at any given time.

The Vulnerability
-----------------

Any process on Apple’s operating systems — including iOS — can register to be notified about any Darwin notification, from within its sandbox, without the need for special entitlements. This makes sense given that some system frameworks used by third-party apps rely on Darwin notifications for important functionality.

Given that the amount of data transferred through them is very limited, Darwin notifications are not a significant risk for sensitive data leaks, even though the API is public, and sandboxed apps can register for notifications.

However, just as any process on the system can register to **receive** Darwin notifications, the same is true for **sending** them.

To summarize, Darwin notifications:

*   Require no special privileges for receiving
*   Require no special privileges for **sending**
*   Are available as public API
*   Have no mechanism for **verifying the sender**

Considering these properties, I began to wonder if there were places on iOS using Darwin notifications for powerful operations that could potentially be exploited as a denial-of-service attack from within a sandboxed app.

You’re reading this blog post, so I’ve already spoiled it: the answer was “yes”.

Proof of Concept: EvilNotify
----------------------------

With that question in mind, I grabbed a fresh copy of the iOS root filesystem — one of the early iOS 18 betas at the time, I think — and began looking for processes that used `notify_register_dispatch` and `notify_check`.

I quickly found a bunch of them, and made a test app called “EvilNotify” that I could use for testing.

Unfortunately, I no longer have a vulnerable device I could use to record a proper on-device video, but the iOS Simulator demo above shows most of what the proof of concept was able to do. Some of them don’t work in the Simulator, so I couldn’t demo them in the video.

You can see a hint at the end of the video of what the ultimate denial of service was, but let me mention all the other things it was capable of doing. Keep in mind, all of them would affect the entire system, even if the user force-quit the app.

*   Cause the “liquid detection” icon to show up in the status bar
*   Trigger the Display Port connection status to show up in the Dynamic Island
*   Block system-wide gestures for pulling down Control Center, Notification Center, and Lock Screen
*   Force the system to disregard Wi-Fi and use the cellular connection instead
*   Lock the screen
*   Trigger a “data transfer in progress” UI that prevented the device from being used until the user cancelled it
*   Simulate the device entering and leaving Find My’s “Lost Mode”, triggering an Apple ID password dialog prompt to re-enable Apple Pay
*   Trigger device entering a “restore in progress” mode

“Restore in Progress”
---------------------

Since I was looking for a denial-of-service attack, this last one seemed to be the most promising, as there was no way out of it other than by tapping the “Restart” button, which would always cause the device to reboot.

It was also quite neat, since it consisted of a single line of code:

```
notify_post("com.apple.MobileSync.BackupAgent.RestoreStarted")
```

That’s it! That single line of code was enough to make the device enter “Restore in Progress”. The operation would inevitably fail after a timeout since the device was not actually being restored, for which the only remedy was tapping the “Restart” button, which would then reboot the device.

Looking into the binaries, SpringBoard was observing that notification to trigger the UI. The notification is triggered when the device is being restored from a local backup via a connected computer, but as established before, any process could send the notification and trick the system into entering that mode.

Denial of Service: VeryEvilNotify
---------------------------------

Now that I had a Darwin notification with the potential of becoming a denial of service, I just had to figure out a way to trigger it repeatedly across device reboots.

At first, this sounded quite tricky, since apps on iOS have very limited opportunities for background processing, and quite a few APIs with side effects are prevented from working when an app is not in the foreground. The latter I found out would not be a problem, as I could verify that `notify_post` worked even when the app was not in the foreground.

As for being able to post the notification again and again as the device rebooted multiple times, I wasn’t so sure, but I had a hunch that an app extension would be the most likely to succeed.

Some types of third-party app extensions may run before first unlock on iOS devices, so I decided to try a type of app extension I’m quite familiar with, and created a widget extension, in a new app that I called “VeryEvilNotify”.

Widget extensions are periodically woken up in the background by iOS. They have a limited amount of time for generating snapshots and timelines, which the system then displays in various places, including the Lock Screen, Home Screen, Notification Center, and Control Center.

Because of how widespread the use of widgets is on the system, when a new app that includes a widget extension is installed and launched, the system is very eager to execute its widget extension. That gets an app’s widgets ready for the user to pick and add to the various supported placements.

A widget extension is ultimately just a process that can run code, so I added the aforementioned line of code to my widget extension. I had configured the extension to include every possible type of widget, just to make it as likely as possible that iOS would execute it as quickly as possible.

There’s a problem though: widget extensions produce placeholders, snapshots, and timelines, which are then cached by the system in order to preserve resources. These extensions are not running in the background all the time, and even if the extension requests very frequent updates, the system will enforce a time budget and delay updates if the extension attempts to request them too frequently.

To circumvent that, I decided to try making my widget extension always crash shortly after running the `notify_post` function, which I did by calling Swift’s `fatalError()` function in every extension point method of its `TimelineProvider`.

The call to `notify_post` was made as part of the entry point of the extension, before handing off execution to the extension runtime:

```
import WidgetKit
import SwiftUI
import notify

struct VeryEvilWidgetBundle: WidgetBundle {
    var body: some Widget {
        VeryEvilWidget()
        if #available(iOS 18, *) {
            VeryEvilWidgetControl()
        }
    }
}

/// Override extension entry point to ensure the exploit code is always run whenever
/// our extension gets woken up by the system.
@main
struct VeryEvilWidgetEntryPoint {
    static func main() {
        notify_post("com.apple.MobileSync.BackupAgent.RestoreStarted")

        VeryEvilWidgetBundle.main()
    }
}
```

With that widget extension in place, as soon as I installed the VeryEvilNotify app on my security research device, the “Restore in Progress” UI was shown, then failed with a prompt to restart the system.

After restarting, as soon as SpringBoard was initialized, the extension would be woken up by the system, since it had failed to produce any widget entries before, which would then start the process all over again.

The result is a device that’s soft-bricked, requiring a device erase and restore from backup. I suspect that if the app ended up in the backup and the device was restored from it, the bug would eventually be triggered again, making it even more effective as a denial of service.

My theory was that iOS would have some sort of retry mechanism when a widget extension crashes, which would obviously have some sort of throttling mechanism. I still think that’s true, but something about the timing of the extension crashing and the restore starting then failing probably prevented such a mechanism from working.

Satisfied with my proof of concept, I reported the issue to Apple.

Timeline
--------

Below is a summarized timeline of events for this vulnerability report. There were additional status updates via automated messages from Apple’s security reports system that I have not included for brevity.

*   June 26, 2024: initial report sent to Apple
*   September 27, 2024: got a message from Apple informing me that mitigation was in progress
*   January 28, 2025: issue flagged as resolved and bounty eligibility confirmed
*   March 11, 2025: bug assigned CVE-2025-24091, [addressed in iOS/iPadOS 18.3](https://support.apple.com/122066)
*   Bug bounty amount: US$17,500

Even though the CVE has already been assigned and Apple has provided a link where the advisory and credit are supposed to be published, that hasn’t happened yet. I’ve been informed that it will be published soon, but you can read the advisory below in case it hasn’t been published yet by the time this post goes out.

 ![](https://rambo.codes/assets/img/CVE-2025-24091/CVE-2025-24091-box.png) 

Notice how the advisory mentions that “sensitive notifications now require restricted entitlements”, hinting at what the mitigation was. You can read more about that in the following section.

Mitigation
----------

As mentioned by Apple in the advisory, sending sensitive Darwin notifications now requires the sending process to possess restricted entitlements. It’s not a single entitlement that just allows posting any sensitive notification, but a prefix entitlement in the form of `com.apple.private.darwin-notification.restrict-post.<notification>`.

From what I could gather from a brief look into the disassembly, what causes a notification to be “restricted” is the prefix `com.apple.private.restrict-post.` in the name of the notification.

For example, the `com.apple.MobileBackup.BackupAgent.RestoreStarted` notification is now posted as `com.apple.private.restrict-post.MobileBackup.BackupAgent.RestoreStarted`, which causes `notifyd` to verify that the posting process has the `com.apple.private.darwin-notification.restrict-post.MobileBackup.BackupAgent.RestoreStarted` entitlement before it allows the notification to be posted.

Processes observing the notification will also be using its new name with the `com.apple.private.restrict-post` prefix, thus preventing any random unentitled app or process from posting a notification that can have serious side effects on the system.

I didn’t have the opportunity to bisect numerous older iOS releases to find the exact version where this mechanism was introduced, but thanks to [ipsw-diffs](https://github.com/blacktop/ipsw-diffs), it appears that the entitlement first showed up in [iOS 18.2 build 22C5125e](https://github.com/blacktop/ipsw-diffs/blob/fee5b3c8c18e4639e74677dd3cc1fa80203e64f6/18_2_22C5109p__vs_18_2_22C5125e/Entitlements.md?plain=1#L2336), AKA iOS 18.2 beta 2.

The first adopters were `backupd`, `BackupAgent2`, and `UserEventAgent`, all gaining entitlements related to notifying the system about device restores, mitigating the most egregious exploit presented in my proof of concept.

Throughout the various iOS 18 betas and releases, more and more processes began adopting the new entitlement for restricted notifications, and with the release of iOS 18.3, all issues demonstrated in my PoC were addressed.
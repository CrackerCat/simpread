> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [konata.github.io](https://konata.github.io/posts/android-screen-cast-issues/)

> Background

Background[](#background)
-------------------------

Backing to 2015, Google introduced the [MediaProjectionManager](https://developer.android.com/reference/android/media/projection/MediaProjectionManager) API in Android Lollipop, gave applications the ability to record the device’s screen. While this feature is incredibly powerful, it also raises some privacy and security concerns. Therefore, Android handle screen cast with caution, for example

*   SystemUI displays a confirm dialog to warn user while initiating the cast recording session,

[![](https://konata.github.io/assets/images/confirm-dialog.jpg)](https://konata.github.io/assets/images/confirm-dialog.jpg)

*   a casting indicator is always on the StatusBar to notify user that they are been watched,

[![](https://konata.github.io/assets/images/indicator.jpg)](https://konata.github.io/assets/images/indicator.jpg)

*   a togglable Screen Cast Tile would show up in the Quick Settings panel, along with its current status and the name of the casting application, clicking the tile leads SystemUI to immediately terminate the casting sessions

[![](https://konata.github.io/assets/images/tile.jpg)](https://konata.github.io/assets/images/tile.jpg)

Assuming that an application called `CaptureCat` is going to implement screen casting functionality, the following is an overview of the process.

Start casting[](#start-casting)
-------------------------------

Spare the hassle process of user grants, permission verification and condition validation, start casting mainly involves three procedures

1.  Request `MediaProjection` token
    
    Once the user agreed the confirm dialog, the SystemUI will request a new `MediaProjection` from `MediaProjectionManagerService` for `CaptureCat` and redirect the `MediaProjection` token back to `CaptureCat`
    
2.  Set current casting session
    
    `CaptureCat` calls `IMediaProjection.start` to notify `MediaProjectionManagerService` that it will start casting. Upon receiving the call, `MediaProjectionManagerService` will
    
    1.  terminate current ongoing cast session (if there is one)
    2.  notify the SystemUI to display the cast indicator, set cast tile state to “on”, and display `CaptureCat`’s name on the tile,
    3.  set the `MediaProjectionManagerService.mProjectionGrant` field to the `MediaProjection` provided by `CaptureCat`, which acknowledge that `CaptureCat` is the app who’s carrying out the screen casting
3.  Create a privileged `VirtualDisplayDevice` which features `VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR` flag
    
    Although `CaptureCat` is unanimous the casting app, it has not received any screen frames so far. To receive these frames, `CaptureCat` must create a privileged `VirtualDisplayDevice` and associate it with the surface on which `CaptureCat` wants to render the frames. To achieve this, `CaptureCat` call `IDisplayManager.createVirtualDevice` with the following parameters
    
    *   `surface`, the surface to which CaptureCat wants to render frame contents. This surface can be obtained using ImageReader, MediaRecorder, or SurfaceView.
    *   `callback`, an instance of `VirtualDisplayCallback`, which is a binder that will be called back whenever there is a state change in the created display.
    *   `flag`, i.e `VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR`, `DisplayManagerService` posed some restriction on calling app if you provide this flag, one of which is that the calling app must be the ongoing screen casting app. Fortunately, `CaptureCat` happens to satisfy
    
    Once the call reaches the `DisplayManagerService`, it will delegate the `VirtualDisplayAdapter` to create the privileged `VirtualDisplayDevice` for the `CaptureCat`, Internally, the `VirtualDisplayAdapter` stores the created `VirtualDisplayDevice` in its `mVirtualDisplayDevices` field (an ArrayMap), using the `callback` parameter provided by `CaptureCat` as the key. Screen frames are then delivered continuously to the surface.
    

Terminate casting[](#terminate-casting)
---------------------------------------

What happens when a user decides to terminate the current screen casting by toggling the casting tile?

*   The `SystemUI` calls `MediaProjectionManager.stopActiveProjection`.
*   The `MediaProjectionManagerService` receives the IPC call and looks for the current ongoing casting session in its `mProjectionGrant` field.
*   The `mProjectionGrant` field triggers the `IMediaProjectionCallback` callback that `DisplayManagerService` installed in it.
*   The `DisplayManagerService` retrieves the `appToken` inside that `callback`, looks for the corresponding `VirtualDisplayDevice` by searching the `mVirtualDisplayDevices` ArrayMap with the `appToken` as the key.
*   Once the `VirtualDisplayDevice` is found, `DisplayManagerService` clears the `surface` field of that `VirtualDisplayDevice` and sets its `mStopped` field to true, thus terminating the screen casting.

Exploit[](#exploit)
-------------------

As described above, `mVirtualDisplayDevices` is an `ArrayMap`, and the key is provided by the `CaptureCat` app (the callback parameter). If the `CaptureCat` app inserts another non-privileged `VirtualDisplayDevice` with the same key into the ArrayMap during casting, the value will be overridden. This can be achieved by simply call `IDisplayManager.createVirtualDevice` without set the `VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR` flag but with the same `callback` as the first call.

Then when user decide to terminate ongoing casting, the non-privileged `VirtualDisplayDevice` will be found, clear its `surface` and set its `mStopped` states. However, this will not terminate the privileged `VirtualDisplayDevice`, then the SystemUI will dismiss the casting indicator, set the casting tile state to “off,” and remove the current casting application name from the tile. This could mislead the user into believe that no casting is currently taking place, causing them to input sensitive information (such as a banking password) without realizing that their screen is still being silently recorded by `CaptureCat`

I came across this vulnerability and reported it to Google on June 28. However, after approximately a month (July 29), they responded by rejecting the report and categorizing it as “Won’t fix.” Their reasoning was that “the user initially granted access to the malicious app to record the screen”. Given this outcome, I have decided to disclose the vulnerability here, with the hope that it will raise awareness among Android users. It is essential to understand that even if you explicitly close the screen casting, there is no casting indicator, and the “Screen casting” tile suggests no ongoing recording, you may still be under surveillance. Therefore, it is crucial to exercise caution in all your activities.

Windfall[](#windfall)
---------------------

After discussing this vulnerability with [Mishaal Rahman](https://twitter.com/MishaalRahman) (BTW his twitter is a goldmine for Android enthusiasts), he pointed towards a relevant compatibility change in the Android 14 beta 4, After exploring it briefly, I found some intriguing aspects, yet it falls short in mitigating the aforementioned vulnerabilities.

Though not explicitly documented, it is possible to reuse the `MediaProjection` token returned by the SystemUI. For instance, when a user explicitly terminates an ongoing screen cast session, the app can invoke `IMediaProjection.start` and `IDisplayManager.createVirtualDisplay(mediaProjection, {flag = VIRTUAL_DISPLAY_FLAG_AUTO_MIRROR, surface }, callback)` again to initiate screen casting. This action does not trigger a confirmation dialog (since you already have the `MediaProjection`), but it will display the casting indicator/QuickSetting tiles with the state “on” and the name of the casting application, ensuring that the user is not misled.

In Android 14, a new AIDL call, `IMediaProjection.isValid()` , is introduced to verify the legitimacy of the `MediaProjection` provided by the app. The criteria for the check are as follows:

*   If the `MediaProjection` was created within the last 5 minutes and has not invoked `IMediaProjection.start` or created a privileged VirtualDisplayDevice, it is considered legal.
*   Otherwise, the judgement is made based on the compat-change settings of the device.

```
<compat-change description="Determines how to respond to an app re-using a consent token; either failing or allowing the user to re-grant consent. <p>Enabled after version 33 (Android T), so applies to target SDK of 34+ (Android U+)." enableSinceTargetSdk="10000"  />
```

Based on my observation, the implementation of this compatibility change is not yet complete (as there is no caller of `isValid` in beta 4). Once it is fully implemented, it will address the issue of `MediaProjection` reuse to some extent, but it will not impact the vulnerability discussed in the “Exploit” section.

This post is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) by the author.
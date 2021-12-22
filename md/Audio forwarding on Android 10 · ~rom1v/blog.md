> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.rom1v.com](https://blog.rom1v.com/2020/06/audio-forwarding-on-android-10/)

> 09 Jun 2020

09 Jun 2020

Audio forwarding is one of the most requested features in [scrcpy](https://blog.rom1v.com/2018/03/introducing-scrcpy/) (see [issue #14](https://github.com/Genymobile/scrcpy/issues/14)).

Last year, I published a small experiment ([USBaudio](https://blog.rom1v.com/2019/06/introducing-usbaudio/)) to forward audio over USB, using the [AOA2](https://source.android.com/devices/accessories/aoa2) protocol. Unfortunately, this Android feature was unreliable, and has been deprecated in Android 8.

Here is a new tool I developed to play the device audio output on the computer, using the [Playback Capture API](https://developer.android.com/guide/topics/media/playback-capture) introduced in Android 10: [`sndcpy`](https://github.com/rom1v/sndcpy).

_The name follows the same logic:_

*   _`strcpy`: **str**ing **c**o**py**_
*   _`scrcpy`: **scr**een **c**o**py**_
*   _`sndcpy`: **s**ou**nd** **c**o**py**_

This is a quick proof-of-concept, composed of:

*   an Android application, which captures and streams the device audio over a socket:
*   a shell script, which starts the app and runs VLC to play the audio stream.

The long-term goal is to implement this feature properly in `scrcpy`.

How to use sndcpy
-----------------

You could either [download the release](https://github.com/rom1v/sndcpy/blob/master/README.md#get-the-app) or [build the app](https://github.com/rom1v/sndcpy/blob/master/BUILD.md).

[VLC](https://www.videolan.org/) must be installed on the computer.

Plug an Android 10 device with USB debugging enabled, and execute:

```
./sndcpy

```

If several devices are connected (listed by `adb devices`):

```
./sndcpy <serial>  # replace <serial> by the device serial

```

_(omit `./` on Windows)_

It will install the app on the device, and request permission to start audio capture:

![](https://blog.rom1v.com/assets/sndcpy/request.png)

Once you clicked on _START NOW_, press _Enter_ in the console to start playing on the computer. Press `Ctrl`+`c` in the terminal to stop (except on Windows, just disconnect the device or stop capture from the device notifications).

The sound continues to be played on the device. The volume can be adjusted independently on the device and on the computer.

Apps restrictions
-----------------

`sndcpy` may only forward audio from apps which do not [prevent audio capture](https://developer.android.com/guide/topics/media/playback-capture#allowing_playback_capture). The rules are detailed in [§capture policy](https://developer.android.com/guide/topics/media/playback-capture#capture_policy):

> *   By default, apps that target versions up to and including to Android 9.0 do not permit playback capture. To enable it, include `android:allowAudioPlaybackCapture="true"` in the app’s `manifest.xml` file.
> *   By default, apps that target Android 10 (API level 29) or higher allow their audio to be captured. To disable playback capture, include `android:allowAudioPlaybackCapture="false"` in the app’s `manifest.xml` file.

So some apps might need to be updated to support audio capture.

Integration in scrcpy
---------------------

Ideally, I would like `scrcpy` to support audio forwarding directly. However, this will require quite a lot of work.

In particular, _scrcpy_ does not use an Android app ([required](https://github.com/Genymobile/scrcpy/issues/14#issuecomment-575920604) for capturing audio), it currently only runs a [Java main](https://blog.rom1v.com/2018/03/introducing-scrcpy/#run-a-java-main-on-android) as _shell_ (required to inject events and capture the screen without asking).

And it will require to implement audio playback (done by VLC in this PoC), but also audio recording (for `scrcpy --record file.mkv`), encoding and decoding to transmit a compressed stream, handle audio-video synchronization…

Since I develop _scrcpy_ on my free time, this feature will probably not be integrated very soon. Therefore, I prefer to release a working proof-of-concept which does the job.
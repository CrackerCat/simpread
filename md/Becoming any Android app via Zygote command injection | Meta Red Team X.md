> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [rtx.meta.security](https://rtx.meta.security/exploitation/2024/06/03/Android-Zygote-injection.html)

> We have discovered a vulnerability in Android that allows an attacker with the WRITE_SECURE_SETTINGS ......

We have discovered a vulnerability in Android that allows an attacker with the WRITE_SECURE_SETTINGS permission, which is held by the ADB shell and certain privileged apps, to execute arbitrary code as any app on a device. By doing so, they can read and write any app’s data, make use of per-app secrets and login tokens, change most system configuration, unenroll or bypass Mobile Device Management, and more. Our exploit involves no memory corruption, meaning it works unmodified on virtually any device running Android 9 or later, and persists across reboots.

[A patch](https://android.googlesource.com/platform/frameworks/base/+/e25a0e394bbfd6143a557e1019bb7ad992d11985) for the issue, tracked as [CVE-2024-31317](https://www.cve.org/CVERecord?id=CVE-2024-31317), is included in [today’s Android Security Bulletin](https://source.android.com/docs/security/bulletin/2024-06-01). As is Google’s practice, device vendors were sent the bulletin a month ago, so updates for supported devices should be forthcoming or already available. Android builds with a June 2024 or later patch level are no longer vulnerable.

Background: Android app isolation
---------------------------------

Despite its Linux kernel, Android’s security model differs fundamentally from that of desktop Linux. Linux is often called a multi-user operating system, but Android might be more appropriately called a multi-app operating system. On Android, what a process can do is determined not by which user started it but by which app it belongs to, and the OS [guarantees](https://source.android.com/docs/security/app-sandbox) that one app cannot impersonate another.

That concept of app identity—which Android implements by giving each app its own Linux UID—underpins most Android security policy. Per-app [permissions](https://developer.android.com/guide/topics/permissions/overview) gate sensitive API calls, [cryptographic keys](https://developer.android.com/privacy-and-security/keystore) and [account credentials](https://developer.android.com/reference/android/accounts/AccountManager) are visible only to the apps that created them, and [device management actions](https://source.android.com/docs/devices/admin) are exclusive to a designated “device owner” app. If an attacker finds a way to impersonate a highly-privileged app, that’s often all they need to achieve their objective.

In [my last post](https://rtx.meta.security/exploitation/2024/03/04/Android-run-as-forgery.html), we impersonated apps by exploiting an injection vulnerability in a file used by run-as, a tool designed to debug apps during development. run-as was an attractive target because it’s one of the few processes on Android that’s [allowed to change its UID](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:system/sepolicy/public/runas.te;l=25-26)[1](#fn:cap-setuid). However, run-as can only be invoked from the ADB shell, quite a high bar for an attacker. In this post, we’ll lower that bar by instead exploiting **Zygote**, one of the few other processes that [can change its UID](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:system/sepolicy/private/zygote.te;l=10-11).

Background: Zygote
------------------

When an app starts, Zygote is what creates its main process and sets that process’s identity. Although only System Server[2](#fn:system-server) can send commands to Zygote, it does so in response to requests (e.g. Activity launches) made by ordinary apps. When System Server receives a request for an app that’s not running, it starts that app by telling Zygote to spawn a process with the appropriate package name, data directory, UID, SELinux domain, and so forth.

Notably, System Server controls security-critical parameters like the new app’s UID. Zygote, perhaps because of its early position in the boot sequence, doesn’t query those parameters from the Android package database itself. That means we can impersonate arbitrary apps if we can control the commands System Server sends—no Zygote exploit needed!

Zygote runs as a daemon and [accepts commands](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/core/java/com/android/internal/os/ZygoteServer.java;l=508-579) on a UNIX stream socket at `/dev/socket/zygote`. Stream sockets aren’t message-oriented, so Zygote’s wire protocol must define where one command ends and the next begins. It does so [very simply](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/core/java/android/os/ZygoteProcess.java;l=430-440): each command is UTF-8 text and consists of a decimal number followed by that many [arguments](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/core/java/com/android/internal/os/ZygoteArguments.java;l=266-540), each on its own line. The line after the final argument begins the next command.

A command consists only of a sequence of arguments. Unlike most command protocols, Zygote’s has no concept of a “command type”. Every command by default spawns a new process, and the arguments specify the details of that process. Certain [special arguments](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/core/java/com/android/internal/os/ZygoteConnection.java;l=137-187) override that default, causing Zygote to instead perform some other action.

Here’s an example of a typical process spawn command (with many arguments elided for brevity), followed by a special “set API denylist exemptions” command, which will prove relevant very soon. The text in brackets is explanatory and not part of the protocol:

```
8                              [command #1 arg count]
--runtime-args                 [arg #1: vestigial, needed for process spawn]
--setuid=10266                 [arg #2: process UID]
--setgid=10266                 [arg #3: process GID]
--target-sdk-version=31        [args #4-#7: misc app parameters]
--nice-name=com.facebook.orca
--app-data-dir=/data/user/0/com.facebook.orca
--package-name=com.facebook.orca
android.app.ActivityThread     [arg #8: Java entry point]
3                              [command #2 arg count]
--set-api-denylist-exemptions  [arg #1: special argument, don't spawn process]
LClass1;->method1(             [args #2, #3: denylist entries]
LClass1;->field1:


```

Vulnerability details
---------------------

We have found a [global setting](https://developer.android.com/reference/android/provider/Settings.Global) in Android, “hidden_api_blacklist_exemptions”, whose value gets included directly in a Zygote command. System Server doesn’t expect the setting to contain newlines, so it neither escapes them nor denotes them in the command’s argument count. By writing a malicious value to that setting, an attacker can place lines of their choosing after the last declared argument. When Zygote sees those lines, it believes them to be a separate command, which it executes!

The vulnerable code path begins at [a ContentObserver callback](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=2329-2350) in System Server, which fires when hidden_api_blacklist_exemptions is changed for any reason:

```
private void update() {
    String exemptions = Settings.Global.getString(mContext.getContentResolver(),
            Settings.Global.HIDDEN_API_BLACKLIST_EXEMPTIONS);
    if (!TextUtils.equals(exemptions, mExemptionsStr)) {
        mExemptionsStr = exemptions;
        if ("*".equals(exemptions)) {
            mBlacklistDisabled = true;
            mExemptions = Collections.emptyList();
        } else {
            mBlacklistDisabled = false;
            mExemptions = TextUtils.isEmpty(exemptions)
                    ? Collections.emptyList()
                    : Arrays.asList(exemptions.split(","));
        }
        if (!ZYGOTE_PROCESS.setApiDenylistExemptions(mExemptions)) {
          Slog.e(TAG, "Failed to set API blacklist exemptions!");
          
          mExemptions = Collections.emptyList();
        }
    }
    mPolicy = getValidEnforcementPolicy(Settings.Global.HIDDEN_API_POLICY);
}


```

From this code, we see that the setting contains a comma-separated list of strings which gets split into an array and passed down to [`ZYGOTE_PROCESS.setApiDenylistExemptions()`](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/core/java/android/os/ZygoteProcess.java;l=903-921). The code incidentally prevents the attacker from using commas, but it does nothing to newlines.

`ZYGOTE_PROCESS` is a singleton instance of [ZygoteProcess](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/core/java/android/os/ZygoteProcess.java), a client for Zygote’s wire protocol. `setApiDenylistExemptions()` just calls another method, [`maybeSetApiDenylistExemptions()`](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/core/java/android/os/ZygoteProcess.java;l=953-982), twice: once for the primary (64-bit) Zygote, and once for the secondary (32-bit) one:

```
@GuardedBy("mLock")
private boolean maybeSetApiDenylistExemptions(ZygoteState state, boolean sendIfEmpty) {
    if (state == null || state.isClosed()) {
        Slog.e(LOG_TAG, "Can't set API denylist exemptions: no zygote connection");
        return false;
    } else if (!sendIfEmpty && mApiDenylistExemptions.isEmpty()) {
        return true;
    }

    try {
        state.mZygoteOutputWriter.write(Integer.toString(mApiDenylistExemptions.size() + 1));
        state.mZygoteOutputWriter.newLine();
        state.mZygoteOutputWriter.write("--set-api-denylist-exemptions");
        state.mZygoteOutputWriter.newLine();
        for (int i = 0; i < mApiDenylistExemptions.size(); ++i) {
            state.mZygoteOutputWriter.write(mApiDenylistExemptions.get(i));
            state.mZygoteOutputWriter.newLine();
        }
        state.mZygoteOutputWriter.flush();
        int status = state.mZygoteInputStream.readInt();
        if (status != 0) {
            Slog.e(LOG_TAG, "Failed to set API denylist exemptions; status " + status);
        }
        return true;
    } catch (IOException ioe) {
        Slog.e(LOG_TAG, "Failed to set API denylist exemptions", ioe);
        mApiDenylistExemptions = Collections.emptyList();
        return false;
    }
}


```

And just like that, the command goes out on the wire. None of these three methods reject or escape newlines.

Interestingly, ZygoteProcess has [a method](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/core/java/android/os/ZygoteProcess.java;l=407-454) that issues an arbitrary command and sanitizes newlines, but it’s hardcoded to expect a “spawn process” response, making it unfit for use here. Since not all Zygote commands spawn processes, the inclusion of that assumption in what would otherwise be a generic helper function likely led directly to this bug.

Exploitation
------------

### Challenge #1: NativeCommandBuffer

On Android 11 and below, exploitation is as simple as described above. In Android 12, however, Google augmented Zygote’s [Java command parser](https://cs.android.com/android/platform/superproject/+/android-11.0.0_r48:frameworks/base/core/java/com/android/internal/os/ZygoteConnection.java;l=113-286) with a [fast-path C++ one](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/core/jni/com_android_internal_os_ZygoteCommandBuffer.cpp;l=367-506) and made both parsers read from the socket via a new class, [NativeCommandBuffer](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/core/jni/com_android_internal_os_ZygoteCommandBuffer.cpp;l=52-288).

NativeCommandBuffer makes this vulnerability harder to exploit, not because of its architecture but because of a bug. [`readLine()`](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/core/jni/com_android_internal_os_ZygoteCommandBuffer.cpp;l=59-97), which reads bytes from the wire, fills a local buffer with bytes from the socket and then extracts lines from that buffer, refilling it as necessary. But after parsing all of a command’s lines, Zygote [discards](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/core/jni/com_android_internal_os_ZygoteCommandBuffer.cpp;l=454-455)[3](#fn:java-parser) any remaining bytes in the buffer and reads the next command from the socket. This behavior causes three problems:

1.  If a client writes two commands in a row before Zygote gets around to reading, Zygote will ignore the second one.
2.  If a client writes a command and a half (e.g. because the second command takes multiple `write()` calls) before Zygote reads, Zygote will ignore the start of the second command as above. Furthermore, it will parse the end of the second command as if it were the beginning, which is itself a security flaw. Note, however, that System Server (Zygote’s only client) never writes multiple commands at a time, so this scenario (and the previous one) does not happen in practice.
3.  If we as attackers use the exact exploit described above, we’ll hit case #1 and our injected lines won’t do anything.

Despite this roadblock, we can still exploit the bug on Android 12+! All we need is a way to keep our malicious command out of Zygote’s first `read()` call. We initially tried lengthening our exploit to exceed the buffer length Zygote passes to `read()`, but unfortunately Zygote [aborts](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/core/jni/com_android_internal_os_ZygoteCommandBuffer.cpp;l=79-82) if a read ever fills its ([12200-byte](https://cs.android.com/android/platform/superproject/+/android-12.0.0_r34:frameworks/base/core/jni/com_android_internal_os_ZygoteCommandBuffer.cpp;l=48), expanded to [32768-byte](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/core/jni/com_android_internal_os_ZygoteCommandBuffer.cpp;l=49) in Android 13) buffer completely. So instead we turned to timing: we can assume that Zygote spends most of its time blocked in `read()`, which means any write we make is likely to trigger an immediate short read, even if we make another write shortly after.

As we saw, `maybeSetApiDenylistExemptions()` makes multiple calls to `state.mZygoteOutputWriter.write()`. But do those calls map directly to socket writes? It turns out they don’t, as `mZygoteOutputWriter` is a [BufferedWriter](https://developer.android.com/reference/java/io/BufferedWriter), which [aggregates](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:libcore/ojluni/src/main/java/java/io/BufferedWriter.java;l=125-137) writes in an internal buffer before writing to the underlying transport.

This is a stroke of luck, as it gives us a ready-made way to issue two socket writes with a decent delay between them. BufferedWriter has a [buffer size of 8192](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:libcore/ojluni/src/main/java/java/io/BufferedWriter.java;l=73), smaller than Zygote’s buffer. By padding System Server’s command to exactly 8192 bytes before inserting our malicious command, we force BufferedWriter to write those 8192 bytes first. Zygote will ignore the padding, but it won’t ignore the remainder of our exploit, since—Linux scheduler willing—that will arrive in a separate `read()` call.

To make this outcome more likely, we can insert a large number of commas at the end of our setting value, causing `maybeSetApiDenylistExemptions()` to spend time looping after the first write but before the second. Those commas also increase the legitimate command’s argument count, but that’s not a problem as long as we ensure the first 8192 bytes contain at least that many newlines. We just need to stay within two limits:

1.  We shouldn’t write more total bytes than Zygote’s command buffer can hold. If we do, we risk crashing Zygote if it happens to read them all at once.
2.  The first command’s argument count shouldn’t exceed Zygote’s limit, which it sets to [half its buffer size](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/core/jni/com_android_internal_os_ZygoteCommandBuffer.cpp;l=139-141), because that will also cause a crash.

We wrote a script to generate an proof-of-concept that combines these techniques, respecting all relevant constraints. See the Appendix for detailed discussion of a sample output. In testing across multiple devices, our PoC reliably executes on the first attempt.

### Challenge #2: return value confusion

A successful exploit degrades or prevents subsequent process launches until a reboot. That’s because the injected Zygote command outputs extra result bytes that System Server doesn’t consume. System Server uses a single connection to Zygote for all non-[USAP](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/core/jni/com_android_internal_os_Zygote.cpp;l=2109-2117) commands, so those bytes stick around until it tries to spawn another process, at which point it reads them instead of that process’s PID. System Server [won’t bind a process](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java;l=4496-4498) without record of its PID, and processes that fail to bind get killed.

We avoided this issue on Android 12+ by slightly modifying our exploit: we declared an argument count for our injected command that exceeded the number of newlines in our final socket write, which forced Zygote to perform an additional socket read while parsing it. That read ate whatever command happened to follow ours (overwhelmingly likely to be a process spawn) and prevented Zygote from executing it. Our malicious command in effect replaced that legitimate command, and System Server consumed its PID (actually our PID) as normal, allowing subsequent PIDs to remain in sync.

This modification also made persistence feasible, as the setting can retain its malicious value across reboots without disrupting the boot process.

Attack scenarios
----------------

### Scenario #1: privilege escalation

Any app [with android.permission.WRITE_SECURE_SETTINGS](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/SettingsProvider.java;l=1471-1472) can write to hidden_api_blacklist_exemptions and trigger the exploit. Android [declares](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/core/res/AndroidManifest.xml;l=4502-4505) that permission’s protection level as `signature|privileged|development|role|installer`, which means unprivileged apps can’t request it[4](#fn:grantability). Various preinstalled apps hold it, though, and an attacker who compromises any of those can use this bug to further escalate privilege.

### Scenario #2: ADB shell

The ADB shell can also read and write settings; it even has a `settings` command to make doing so easy. An attacker with physical access to an unlocked device—or a user who wants to bypass system policy (e.g. MDM restrictions) on a device in their possession—can trigger the exploit that way.

### Scenario #3: [Signed Config](https://source.android.com/docs/core/runtime/signed-config)

There’s one other way to set hidden_api_blacklist_exemptions, which is why it exists to begin with: any app (even an [Instant App](https://developer.android.com/topic/google-play-instant)!) may contain a [special pair of <meta-data> tags](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:cts/hostsidetests/signedconfig/app/version1_AndroidManifest.xml;l=22-25) in its manifest, containing

1.  a Base64-encoded value to store in hidden_api_blacklist_exemptions; and
2.  an ECDSA signature of that value by a [hardcoded, Google-controlled key](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/services/core/java/com/android/server/signedconfig/SignatureVerifier.java;l=47-49).

If such an app is installed and the signature is valid, Android will immediately [apply the setting value](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/services/core/java/com/android/server/signedconfig/GlobalSettingsConfigApplicator.java;l=98-101), potentially triggering the exploit.

We believe the signature verification and surrounding logic to be correctly implemented, so it’s likely that the only actor who can exploit devices this way is Google themselves. Nevertheless, most Android devices are not 1st-party Google devices, and this bug could give Google much greater access to those devices than OEMs and users expect. Notably, CTS [requires](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:cts/hostsidetests/signedconfig/hostside/src/com/android/cts/signedconfig/SignedConfigHostTest.java;l=146-152) that Google-signed metadata be accepted, meaning most OEMs couldn’t remove this exploitation path even if they tried.

The intended purpose of this functionality is benign: hidden_api_blacklist_exemptions was [created](https://android-review.googlesource.com/c/platform/frameworks/base/+/647221) to be nothing more than an escape hatch to the [undocumented API restrictions](https://developer.android.com/guide/app-compatibility/restrictions-non-sdk-interfaces) that Android 9 introduced[5](#fn:signedconfig-motivation). Were it not for the vulnerability we’ve detailed, malicious values would pose no great threat.

Response
--------

We reported our findings privately to Google on December 12th, 2023. On December 20th, the Android Security Team rated it High severity. Google shared a patch for the immediate issue with us on March 26th, 2024, which we reviewed and verified prevents all known exploitation paths. That is [the patch Google released today](https://android.googlesource.com/platform/frameworks/base/+/e25a0e394bbfd6143a557e1019bb7ad992d11985).

Today’s patch does not address the architectural weaknesses we identified, like Zygote’s use of a hand-rolled stream protocol or ZygoteProcess’s lack of a reusable function to safely serialize commands, as those entail bigger changes and are not directly exploitable. Google has has communicated that they’re considering such changes going forwards, though.

Issue list
----------

For ease of reference, here’s a numbered list of the technical flaws we identified in this report:

1.  [Bug] Newlines contained in hidden_api_blacklist_exemptions are not sanitized before inclusion in Zygote’s newline-delimited wire protocol, allowing command injection.
2.  [Weakness] As of Android 12, Zygote will only process one command per `read()` call, dropping any extra bytes. It’s never permissible to condition behavior on the `read()` boundaries of a stream, as the kernel can batch or split writes arbitrarily. (Our original report to Google identified this as an exploitable bug, but Google correctly pointed out that all existing Zygote clients are fully synchronous, meaning at most one command will be buffered in practice.)
3.  [Weakness] [ZygoteProcess](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/core/java/android/os/ZygoteProcess.java) has no single abstraction to serialize an array of arguments into a wire-format command, which means each newly-implemented Zygote command presents a fresh opportunity for an injection bug.
4.  [Weakness] Zygote uses UNIX stream sockets, which require a custom message framing protocol, instead of UNIX datagram sockets, which provide built-in framing.
5.  [Weakness] Zygote uses a rudimentary, hand-rolled command protocol instead of a mature RPC protocol like Binder.

Appendix: proof-of-concept
--------------------------

For illustrative purposes, let’s imagine that BufferedWriter buffers only 64 bytes and that Zygote limits commands to 100 bytes (meaning it will abort if a single read ever returns 100 bytes or more). Plugging those parameters into our script, along with a 3-argument injected command—`["--some", "--malicious", "command"]`, results in the following value for hidden_api_blacklist_exemptions:

```

AAAAAAAAAAAAAAAAAAAAAAAAAAA3
--some
--malicious
command
,,,,X


```

System Server sees this as a comma-separated list with 5 **entries**. Note that we distinguish “entries” from **arguments**: the former are the comma-separated list items provided to System Server via hidden_api_blacklist_exemptions, while the latter are the Zygote command arguments that go out on the wire. In this example, the 5 entries are as follows…

```
[
  "\n\n\n\n\nAAAAAAAAAAAAAAAAAAAAAAAAAAA3\n--some\n--malicious\ncommand\n",
  "",
  "",
  "",
  "X",
]


```

…but because we’ve injected newlines, those entries don’t correspond directly to arguments. Instead, the first entry spans 5 arguments and then continues on to start a second 64-byte block with our malicious command! Here’s what `maybeSetApiDenylistExemptions()` ends up writing to Zygote’s socket (brackets for annotation, as before):

```
6                              [arg count: special arg + 5 entries]
--set-api-denylist-exemptions  [uncontrolled arg #1: action to take]
                               [beginning of entry #1: empty args #2-#6]




AAAAAAAAAAAAAAAAAAAAAAAAAAA3   [pad to exactly 64 bytes, then arg count]
--some                         [args #1-#3: malicious command]
--malicious
command

                               [entries #2-#5, emitted in loop, each
                                lengthening the delay between writes]

X


```

There are just enough `A` characters to make `3`, the beginning of our malicious command, occur at offset 64. And there are just enough empty “delay entries” to bring the total size to 99, as high as it can go without exceeding Zygote’s length limit. That gives us the best timing we can get while still keeping failures silent.

Note that the last delay entry isn’t empty like the rest. That’s to work around the fact that Java’s `String.split()` function, used by System Server to parse the setting value, [discards trailing empty strings](https://developer.android.com/reference/java/lang/String#split(java.lang.String)).

Appendix: disclosure timeline
-----------------------------

*   June–November, 2023: We find and document the bug after noticing weaknesses in Zygote’s wire protocol.
*   December 12th, 2023: We report our findings to Google, who passes them to the Android Security Team.
*   December 20th, 2023: Google notifies us that they’ve rated the issue High Severity.
*   February 6th, 2024: We ask Google for a progress update. They respond on February 20th that they’re developing a fix but have no ETA.
*   February 15th, 2024: We extend our tentative disclosure date from March 12th (90 days after disclosure) to April 4th to accomodate planned time off within RTX.
*   March 12th, 2024: Google proposes a call to discuss their fix, which we schedule for March 26th.
*   March 26th, 2024: On the call, Google shares a proposed patch with us and we agree on a coordinated disclosure date of June 3rd, 2024. Google also disputes our assertion that Zygote’s `read()` semantics pose a security threat in practice, which we accept after further investigation. Google requests a draft of this post to help with their messaging.
*   April 11th, 2024: Google offers us a $7,000 bounty for our report, which we request be donated to charity. (Google, like Meta, doubles bounties paid to charity.)
*   May 6th, 2024: Meta is sent the June 2024 Android Security Bulletin preview, and RTX confirms the patch we saw is present and learns the CVE ID, CVE-2024-31317.
*   May 21st, 2024: Google shares the CVE ID with us via our bug report.
*   June 3rd, 2024: We share a draft of this post with Google (and apologize for not sharing one earlier). Later in the day, it, [our disclosure](https://github.com/metaredteam/external-disclosures/security/advisories/GHSA-x9q9-2r8c-hg2p), the [April ASB](https://source.android.com/docs/security/bulletin/2024-06-01), and [CVE-2024-31317](https://www.cve.org/CVERecord?id=CVE-2024-31317) all go live.
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [github.com](https://github.com/metaredteam/external-disclosures/security/advisories/GHSA-x9q9-2r8c-hg2p?continueFlag=20d18b66519cac8e0a6b030754b4b5df)

> **For more detailed analysis of this issue, see [Red Team X's blog post about it](https://rtx.meta.se......

**For more detailed analysis of this issue, see [Red Team X's blog post about it](https://rtx.meta.security/exploitation/2024/06/03/Android-Zygote-injection.html).**

We have discovered a vulnerability in Android that allows an attacker with the WRITE_SECURE_SETTINGS permission, which is held by the ADB shell and certain privileged apps, to execute arbitrary code as any app on a device. By doing so, they can read and write any app's data, make use of per-app secrets and login tokens, change most system configuration, unenroll or bypass Mobile Device Management, and more. Our exploit involves no memory corruption, meaning it works unmodified on virtually any device running Android 9 or later, and persists across reboots.

We have found a [global setting](https://developer.android.com/reference/android/provider/Settings.Global) in Android, "hidden_api_blacklist_exemptions", whose value gets included directly in a Zygote command. System Server doesn't expect the setting to contain newlines, so it neither escapes them nor denotes them in the command's argument count. By writing a malicious value to that setting, an attacker can place lines of their choosing after the last declared argument. When Zygote sees those lines, it believes them to be a separate command, which it executes!

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
          // leave mExemptionsStr as is, so we don't try to send the same list again.
          mExemptions = Collections.emptyList();
        }
    }
    mPolicy = getValidEnforcementPolicy(Settings.Global.HIDDEN_API_POLICY);
}

```

From this code, we see that the setting contains a comma-separated list of strings which gets split into an array and passed down to [`ZYGOTE_PROCESS.setApiDenylistExemptions()`](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/core/java/android/os/ZygoteProcess.java;l=903-921). The code incidentally prevents the attacker from using commas, but it does nothing to newlines.

`ZYGOTE_PROCESS` is a singleton instance of [ZygoteProcess](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/core/java/android/os/ZygoteProcess.java), a client for Zygote's wire protocol. `setApiDenylistExemptions()` just calls another method, [`maybeSetApiDenylistExemptions()`](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/core/java/android/os/ZygoteProcess.java;l=953-982), twice: once for the primary (64-bit) Zygote, and once for the secondary (32-bit) one:

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

Interestingly, ZygoteProcess has [a method](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/core/java/android/os/ZygoteProcess.java;l=407-454) that issues an arbitrary command and sanitizes newlines, but it's hardcoded to expect a"spawn process" response, making it unfit for use here. Since not all Zygote commands spawn processes, the inclusion of that assumption in what would otherwise be a generic helper function likely led directly to this bug.

Scenario #1: privilege escalation
---------------------------------

Any app [with android.permission.WRITE_SECURE_SETTINGS](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/SettingsProvider.java;l=1471-1472) can write to hidden_api_blacklist_exemptions and trigger the exploit. Android [declares](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/core/res/AndroidManifest.xml;l=4502-4505) that permission's protection level as `signature|privileged|development|role|installer`, which means unprivileged apps can't request it[1](#user-content-fn-grantability-02ef6255aff654c14c1bd3512711e362). Various preinstalled apps hold it, though, and an attacker who compromises any of those can use this bug to further escalate privilege.

Scenario #2: ADB shell
----------------------

The ADB shell can also read and write settings; it even has a `settings` command to make doing so easy. An attacker with physical access to an unlocked device—or a user who wants to bypass system policy (e.g. MDM restrictions) on a device in their possession—can trigger the exploit that way.

Scenario #3: [Signed Config](https://source.android.com/docs/core/runtime/signed-config)
----------------------------------------------------------------------------------------

There's one other way to set hidden_api_blacklist_exemptions, which is why it exists to begin with: any app (even an [Instant App](https://developer.android.com/topic/google-play-instant)!) may contain a [special pair of <meta-data> tags](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:cts/hostsidetests/signedconfig/app/version1_AndroidManifest.xml;l=22-25) in its manifest, containing

1.  a Base64-encoded value to store in hidden_api_blacklist_exemptions; and
2.  an ECDSA signature of that value by a [hardcoded, Google-controlled key](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/services/core/java/com/android/server/signedconfig/SignatureVerifier.java;l=47-49).

If such an app is installed and the signature is valid, Android will immediately [apply the setting value](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/services/core/java/com/android/server/signedconfig/GlobalSettingsConfigApplicator.java;l=98-101), potentially triggering the exploit.

We believe the signature verification and surrounding logic to be correctly implemented, so it's likely that the only actor who can exploit devices this way is Google themselves. Nevertheless, most Android devices are not 1st-party Google devices, and this bug could give Google much greater access to those devices than OEMs and users expect. Notably, CTS [requires](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:cts/hostsidetests/signedconfig/hostside/src/com/android/cts/signedconfig/SignedConfigHostTest.java;l=146-152) that Google-signed metadata be accepted, meaning most OEMs couldn't remove this exploitation path even if they tried.

The intended purpose of this functionality is benign: hidden_api_blacklist_exemptions was [created](https://android-review.googlesource.com/c/platform/frameworks/base/+/647221) to be nothing more than an escape hatch to the [undocumented API restrictions](https://developer.android.com/guide/app-compatibility/restrictions-non-sdk-interfaces) that Android 9 introduced[2](#user-content-fn-signedconfig-motivation-02ef6255aff654c14c1bd3512711e362). Were it not for the vulnerability we've detailed, malicious values would pose no great threat.

For ease of reference, here's a numbered list of the technical flaws this advisory encompasses:

1.  [Bug] Newlines contained in hidden_api_blacklist_exemptions are not sanitized before inclusion in Zygote's newline-delimited wire protocol, allowing command injection.
2.  [Weakness] As of Android 12, Zygote will only process one command per `read()` call, dropping any extra bytes. It's never permissible to condition behavior on the `read()` boundaries of a stream, as the kernel can batch or split writes arbitrarily. (Our original report to Google identified this as an exploitable bug, but Google correctly pointed out that all existing Zygote clients are fully synchronous, meaning at most one command will be buffered in practice.)
3.  [Weakness] [ZygoteProcess](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r11:frameworks/base/core/java/android/os/ZygoteProcess.java) has no single abstraction to serialize an array of arguments into a wire-format command, which means each newly-implemented Zygote command presents a fresh opportunity for an injection bug.
4.  [Weakness] Zygote uses UNIX stream sockets, which require a custom message framing protocol, instead of UNIX datagram sockets, which provide built-in framing.
5.  [Weakness] Zygote uses a rudimentary, hand-rolled command protocol instead of a mature RPC protocol like Binder.
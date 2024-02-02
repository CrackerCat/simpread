> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [rtx.meta.security](https://rtx.meta.security/exploitation/2024/01/30/Android-vendors-APEX-test-keys.html)

> We recently discovered that Android devices from multiple major brands sign APEX modules—updatable un......

We recently discovered that Android devices from multiple major brands sign APEX modules—updatable units of highly-privileged OS code—using private keys from Android’s public source repository. Anyone can forge an APEX update for such a device to gain near-total control over it. Rather than negligence by any particular manufacturer (OEM), we believe that unsafe defaults, poor documentation, and incomplete CTS coverage in the Android Open Source Project (AOSP) were the main causes of this issue.

Google assigned the issue [CVE-2023-45779](https://www.cve.org/CVERecord?id=CVE-2023-45779), and most affected OEMs have now fixed it. Any device that comes with the Play Store is no longer vulnerable if it advertises at least a 2023-12-05 Security Patch Level (SPL).

Background
----------

[APEX modules](https://source.android.com/docs/core/ota/apex) allow OEMs to update certain files in an OS image without issuing a full OTA. To do so, they locate each updatable unit (e.g. Bionic or ART) in its own ext4 filesystem image inside an `.apex` ZIP file, which gets mounted under `/apex`. An initial version of each APEX is preinstalled in `/system/apex` (or `/vendor/apex`, etc) during the OS build, but those versions can be superseded by updates installed later in `/data/apex`.

To ensure APEX updates are trustworthy, Android checks that each one is signed with the same keys as the preinstalled version of that APEX. APEXes carry both a standard APK signature and an [AVB](https://android.googlesource.com/platform/external/avb/+/main/README.md) signature on their interior filesystem, and both are checked in this way. So to create a valid APEX update, one must possess both the APK and AVB private keys that were used to sign that APEX when the OS was built.

When it comes to signatures, though, an “OS build” isn’t just one step. Android’s core build system signs every APEX, APK, and OTA image it produces with a fixed set of “test keys”, and there’s no way to change that. Test keys are public in AOSP’s source tree: for example, [this](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r18:art/build/apex/com.android.art.pem) is the test key that signs the filesystem of the `com.android.art` APEX.

As described [in AOSP’s documentation](https://source.android.com/docs/core/ota/sign_builds), the job of re-signing a build with OEM-held “release keys” falls to a Python script called [`sign_target_files_apks`](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r18:build/make/tools/releasetools/sign_target_files_apks.py), which unpacks a built image, replaces all the signatures, and repacks it. Re-signing as a separate step has several benefits, but it also introduces the risk that not every test signature will get replaced. And that’s exactly what seems to have happened for several OEMs.

Vulnerability details
---------------------

We analyzed OS images of recent Android devices from 14 reputable brands (listed below) and found that seven of those devices contained at least one preinstalled APEX signed only with AOSP test keys, for which anyone can produce an update.

Every vulnerable device we found had one highly-privileged vulnerable APEX in common—`com.android.vndk`. This APEX holds shared libraries that [HALs](https://source.android.com/docs/core/architecture/hal) in the `/vendor` partition link against. Thanks to the existence of Same-Process HALs, those libraries get transitively loaded into most processes on an Android system, including

*   `zygote64`, from which System Server and every app process is forked;
*   `surfaceflinger`, through which all screen contents pass;
*   and of course all the HALs.

We demonstrated code execution in all of the above via a malicious `com.android.vndk.v31` APEX update for a vulnerable device, a Lenovo Tab M10 Plus (Gen 3, Wi-Fi) running Android 13. You can find our proof-of-concept, as well as a script to check for vulnerable APEXes, [here](https://github.com/metaredteam/rtx-cve-2023-45779).

As an aside, `com.android.vndk` is somewhat odd because there are multiple copies of it—one for each Android API level `/vendor` can target, of which there are several thanks to [Project Treble](https://android-developers.googleblog.com/2017/05/here-comes-treble-modular-base-for.html). Each copy has a different APEX name (e.g. `com.android.vndk.v33` for Android 13), and it’s only useful to exploit the one `/vendor` actually uses. On all the devices we tested, every copy was equally vulnerable.

Attack scenarios
----------------

Fortunately, Android tightly controls who can install APEX updates. Although APEXes look like APKs and are installed via PackageManager, the `INSTALL_APEX` flag is [restricted](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r18:frameworks/base/services/core/java/com/android/server/pm/PackageInstallerService.java;l=745-753) to packages that hold `android.permission.INSTALL_PACKAGES` or `android.permission.INSTALL_PACKAGE_UPDATES`, both of which are `signature|privileged` permissions and cannot be obtained by third-party apps. And even packages with that permission [must additionally](https://cs.android.com/android/_/android/platform/frameworks/base/+/e501fd8379b1849cd3ae4f7a0485c7890aa87179) either

*   run as the `system` user
*   run as the `shell` user
*   be designated as “module installer” in `/{system,vendor,product,odm}/etc/sysconfig/`

In practice, that limits the exploitability to four attack scenarios:

1.  A user uses `adb shell` to exploit their own device, gaining access to nearly everything a typical “root” gets them. Since access is distributed across many SELinux contexts, root-aware tools and apps won’t work unmodified. On the other hand, root detections won’t trip: to them, the malicious APEX will be indistinguishable from legitimate OS code.
2.  A malicious actor with physical access to an unlocked device uses `adb shell` to install persistent malware without the user’s knowledge and gains long-term access to all data and activity on the device. The malware will likely go undetected by on-device scanners for the same reason a root will.
3.  A malicious actor chains this exploit to one that gets them code execution in `com.android.vending` (the “modules installer” on all [Project Mainline](https://android-developers.googleblog.com/2019/05/fresher-os-with-projects-treble-and-mainline.html) devices) or a `system` UID app to escalate their privileges and gain persistence.
4.  A malicious actor who gains access to whatever Google Play backend serves APEX updates remotely exploits devices en masse. We do not know the details of Project Mainline’s infrastructure so cannot assess how feasible this is. For example, it becomes far more plausible if OEMs are allowed to upload APEX updates to Google Play than if only Google is, as there are more credentials for an attacker to compromise. In that case, depending on the specifics of update targeting, a malicious OEM could potentially even exploit other OEMs’ devices.

Root cause
----------

Why did so many OEMs make the exact same mistake? Recall that AOSP comes with [instructions](https://source.android.com/docs/core/ota/sign_builds) to re-sign a build, [a script](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r18:build/make/tools/releasetools/sign_target_files_apks.py) to perform that re-signing, and—as a last line of defense—its extensive [Compatibility Test Suite](https://source.android.com/docs/compatibility/cts), which enforces various compatibility and security guarantees. Shouldn’t one of those have warned OEMs of vulnerable APEXes?

In answering that question, we uncovered a number of deficiencies in AOSP that, together, make it far easier to create a vulnerable build than a secure one.

### Incomplete CTS coverage

Most critically, we found that several APEXes are not checked by CTS at all. Two CTS tests look for test keys: [PackageSignatureTest](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r18:cts/tests/tests/security/src/android/security/cts/PackageSignatureTest.java;l=81-109) checks both APKs and APEXes for insecure APK signatures, while [ApexSignatureVerificationTest](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r18:cts/hostsidetests/appsecurity/src/android/appsecurity/cts/ApexSignatureVerificationTest.java) checks APEXes for insecure AVB signatures. But both tests hardcode lists of test keys, which over time have diverged from those actually in use. As a result, several vulnerable APEXes are not caught by either test.

The nature of APEXes makes such divergence inevitable: unlike APKs, which share [just a few test keys](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r18:build/make/target/product/security/), APEXes all have different test keys, meaning each new APEX is a new opportunity for divergence. In our view, the only fix is for CTS to check signatures against the source of truth—the build system. In fact, the build system already records which test keys it uses to a file called `apexkeys.txt`, which plays a part in…

### Unsafe defaults

[`sign_target_files_apks`](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r18:build/make/tools/releasetools/sign_target_files_apks.py), which re-signs a build, doesn’t guarantee replacement of every test signature. On the contrary, it doesn’t replace any test signatures by default! Run without arguments, it signs each APEX and APK using the very same test keys the build system did, which it finds by parsing `apexkeys.txt` and `apkcerts.txt` respectively. We assume this default was intended as a starting state for arguments like `--key_mapping`, but we’re unsure why the absence of such arguments doesn’t result in at least a warning.

Here too, a risk that was low for APKs—forgetting to specify a test key’s replacement—became a near certainty once APEXes appeared. Because each APEX has its own test keys, each must be mapped to a release key individually. And although enumerating every APEX in Android is clearly error-prone—especially as new Android versions regularly add and remove APEXes—that’s exactly what AOSP’s documentation [instructs](https://source.android.com/docs/core/ota/sign_builds#apex-signing-key-replacement) OEMs to do. And speaking of that documentation…

### Poor documentation

AOSP’s “[Sign builds for release](https://source.android.com/docs/core/ota/sign_builds)” article begins by declaring that Android uses signatures in “two places”, APKs and OTA updates. There is no mention of APEX signatures in the introduction, nor anywhere prior to a section titled “Advanced signing options”, which gives the guidance above.

Furthermore, neither the names nor the locations of APEX test keys themselves (e.g. [this one](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r18:art/build/apex/com.android.art.pem)) make it clear that they’re test keys. In contrast, APK test keys all live in [a single directory](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r18:build/make/target/product/security/) alongside a notice that they should “NEVER be used to sign packages in publicly released images”.

Without documentation to the contrary, an OEM might believe that APEX re-signing is optional or unimportant until CTS failures arise. They then might address those failures and think nothing more of the matter, confident that CTS and `sign_target_files_apks` know what needs signing. And who could fault them for that?

### Other factors

Although far less important than the three issues above, these two details may have also misled OEMs:

1.  In `Android.bp` files, some APEXes, including `com.android.vndk`, are marked [`updatable: false`](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r18:packages/modules/vndk/apex/Android.bp;l=25). That might lead OEMs to believe that such APEXes cannot be updated and so don’t need secure keys, but in fact all it means is that the build system will not “enforce additional rules for making sure that the APEX is truly updatable”.
2.  `com.android.vndk` is not in Google’s [documented list of APEXes](https://source.android.com/docs/core/ota/modular-system). Additionally, the [AVB test key files](https://cs.android.com/android/platform/superproject/+/android-14.0.0_r18:packages/modules/vndk/apex/com.android.vndk.pubkey) for `com.android.vndk` end in `.pubkey` rather than the more common `.avbpubkey`, which could cause naive enumeration strategies to miss them.

Response
--------

We reported our findings privately to Google on September 19th, 2023. Google acknowledged our report immediately, and the Android Security Team confirmed it as valid within a week. On September 25th, Google issued Partner Security Advisory 2023-11, advising its OEM partners of the issue and how to fix it. Around the same time, Google individually contacted each affected OEM we identified.

To ensure that OEMs re-signed vulnerable APEXes, Google added a test to their proprietary Build Test Suite (BTS), through which all updates to [Play Protect certified](https://www.android.com/certified/partners/) devices pass, that warned of vulnerable APEXes starting November 1st and rejected them starting December 4th for updates claiming a [December 2023 patch level](https://source.android.com/docs/security/bulletin/2023-12-01) or higher.

Google has also fixed the deficiencies we identified in CTS. We have not seen those fixes, which won’t be made public until the release of Android V. Nonetheless, we believe the vast majority of real-world risk is now gone: OEMs have had ample time to patch their devices since Google’s initial advisory, and a spot check we performed on January 25th revealed that most have done so.

[CVE-2023-45779](https://www.cve.org/CVERecord?id=CVE-2023-45779) was made public by Google on December 4th but contained no details. This post and [our accompanying disclosure](https://github.com/metaredteam/external-disclosures/security/advisories/GHSA-wmcc-g67r-9962) are the first public descriptions of the issue. Google plans to update the CVE with more detail after we publish this post.

Tested devices
--------------

### Entry format

*   **Device name**  
    `ro.build.fingerprint`  
    Where we obtained the OS image  
    Vulnerable APEXes, if any

### Vulnerable

_NOTE: On every device we checked, either all versions of the VNDK APEX were vulnerable or none of them were. For brevity, we’ve only listed the VNDK version that’s actually used by each vulnerable device._

*   **Asus Zenfone 9**  
    `asus/WW_AI2202/ASUS_AI2202:13/TKQ1.220807.001/33.0804.2060.142:user/release-keys`  
    [dumps.tadiphone.dev, vivo/v2219](https://www.asus.com/mobile-handhelds/phones/zenfone/zenfone-9/helpdesk_bios/?model2>Official ASUS OTA ZIP</a><br>
    com.android.vndk.v32, com.android.uwb, com.android.wifi</li>
      <li><strong>vivo X90 Pro</strong><br>
    <code>vivo/V2219/V2219:14/UP1A.230620.001/compiler07281738:user/release-keys</code><br>
    <a href=)  
    com.android.vndk.v33, com.android.rkpd, com.android.uwb, com.android.virt, com.android.wifi
*   **Nokia G50**  
    `Nokia/Punisher_00WW/PHR_sprout:13/TKQ1.220807.001/00WW_3_320:user/release-keys`  
    [dumps.tadiphone.dev, nokia/phr_sprout](https://dumps.tadiphone.dev/dumps/nokia/phr_sprout)  
    com.android.vndk.v30
*   **Microsoft Surface Duo 2**  
    `surface/duo2/duo2:12/2023.429.67/202304290067:user/release-keys`  
    [Official Microsoft OTA ZIP](https://support.microsoft.com/en-us/surface-recovery-image)  
    com.android.vndk.v30, com.android.appsearch, com.android.wifi
*   **Lenovo Tab M10 Plus (Gen 3, Wi-Fi)**  
    `Lenovo/TB125FU/TB125FU:13/TP1A.220624.014/S100078_230713_ROW:user/release-keys`  
    Physical device  
    com.android.vndk.v31, com.android.uwb, com.android.wifi
*   **Nothing Phone 2**  
    `Nothing/Pong/Pong:13/TKQ1.221220.001/2308181943:user/release-keys`  
    [dumps.tadiphone.dev, Nothing/Pong](https://dumps.tadiphone.dev/dumps/nothing/pong)  
    com.android.vndk.v32, com.android.uwb, com.android.wifi
*   **Fairphone 5**  
    _NOTE: Fairphone did their own investigation in response to our report and discovered that Fairphone 3, 3+, and 4 were also vulnerable. You can read their statement [here](https://www.fairphone.com/en/2023/12/22/security-update-apex-modules-vulnerability-fixed)._  
    `Fairphone/FP5/FP5:13/TKQ1.230127.002/TT3G:user/release-keys`  
    [dumps.tadiphone.dev, fairphone/fp5](https://dumps.tadiphone.dev/dumps/fairphone/fp5)  
    com.android.vndk.v30, com.android.uwb, com.android.wifi

### Not vulnerable

*   **Google Pixel 5**  
    `google/redfin/redfin:13/TQ3A.230805.001.A2/10385117:user/release-keys`  
    Physical device
*   **Samsung Galaxy S23**  
    `samsung/dm1quew/dm1q:13/TP1A.220624.014/S911U1UEU1AWGH:user/release-keys`  
    Official Samsung OTA ZIP, fetched with [samloader](https://github.com/martinetd/samloader)
*   **Xiaomi Redmi Note 12 4G**  
    `Redmi/tapas_global/tapas:13/TKQ1.221114.001/V14.0.12.0.TMTMIXM:user/release-keys`  
    [Official Xiaomi OTA ZIP](https://bigota.d.miui.com/V14.0.12.0.TMTMIXM/miui_TAPASGlobal_V14.0.12.0.TMTMIXM_7e3f673289_13.0.zip)
*   **OPPO Find X6 Pro**  
    `OPPO/PGEM10/OP528BL1:13/TP1A.220905.001/T.10b3891-27825-556fa:user/release-keys`  
    [dumps.tadiphone.dev, oppo/op528bl1](https://dumps.tadiphone.dev/dumps/oppo/op528bl1)
*   **Sony Xperia 1 V**  
    `Sony/pdx234/pdx234:13/TKQ1.221114.001/1:user/release-keys`  
    [dumps.tadiphone.dev, sony/pdx234](https://dumps.tadiphone.dev/dumps/sony/pdx234)
*   **moto razr 40 Ultra**  
    `motorola/zeekr_cn/msi:13/T2TZ33M.18-35-5/dcdd5:user/release-keys`  
    [dumps.tadiphone.dev, motorola/zeekr](https://dumps.tadiphone.dev/dumps/motorola/zeekr)
*   **OnePlus 10T**  
    `OnePlus/CPH2413/OP5552L1:13/SKQ1.221119.001/S.123ec2a_6b801_6ff30:user/release-keys`  
    [dumps.tadiphone.dev, oneplus/op5552l1](https://dumps.tadiphone.dev/dumps/oneplus/op5552l1)

Appendix: disclosure timeline
-----------------------------

*   September 6th, 2023: We notice that an Android device we use for testing has APEXes signed with test keys, which prompts us to check other devices.
*   September 13th, 2023: We complete our survey and conclude that the issue is widespread enough that Google should coordinate the response.
*   September 19th, 2023: We report our findings to Google, who passes them to the Android Security Team.
*   September 25th, 2023: Google releases an Android Partner Security Advisory to their OEM partners detailing the issue. The next day, they respond to us, rating the issue High Severity.
*   September 28th, 2023: Google informs us of the Partner Advisory and indicates they’ve contacted affected OEMs directly.
*   October 18th, 2023: Google updates the Partner Security Advisory with details on remediation, stating that the issue will be part of the December 4th Android Security Bulletin and detailing the BTS enforcement schedule.
*   October 26th, 2023: We ask Google if the December ASB will contain enough details to warrant simultaneous release of this post, even if most OEMs haven’t released a fix. Google replies that the bulletin text won’t contain “specific technical information” but that they do “consider [the issue] publicly disclosed” at that point.
*   November 1st: BTS purportedly begins warning OEMs when a build has vulnerable APEXes.
*   November 6th, 2023: We notify affected OEMs that we’ll name them in this post on December 4th, as we believe the ASB will include CTS patches indicating APEXes have been signed with test keys. We offer to publish statements from them. We receive an automated acknowledgement from Nothing, and Nokia Corporation tells us they’ve passed the email to HMD Global, who makes Nokia-branded phones but “don’t have [a] similar responsible disclosure program”.
*   November 7th, 2023: Google updates the Partner Security Advisory to add the CVE number and a note that only “builds … claiming the 2023-12-05 SPL or higher” will be subject to BTS enforcement on December 4th.
*   November 13th, 2023: Lenovo confirms receipt of our email and says they don’t yet have a statement.
*   Week of November 13th: Multiple OEMs create keys to re-sign their vulnerable APEXes, as evidenced by metadata in the updates they subsequently released.
*   November 15th, 2023: Google asks permission to share our detection tooling with OEMs who want an offline way to find vulnerable APEXes. We grant it.
*   November 15th, 2023: We ask Google explicitly if the December ASB will include CTS patches, as our plan to disclose on December 4th relies on that assumption.
*   November 21st, 2023: Google sends us a generic update which formally shares the CVE ID and states in part that they “will be releasing a patch for this issue in an upcoming bulletin”.
*   November 27th, 2023: We notice that the December ASB partner preview, which Meta has access to but most researchers don’t, contains no APEX-related CTS patches. We ask Google to confirm that the ASB will expose details of the issue via a patch. Google replies that CTS patches actually won’t become public until Android V and that they support us giving OEMs more time.
*   November 28th, 2023: Lenovo asks us if we plan to disclose on December 4th, like our notice claimed, or on December 18th (90 days from our initial report), like Google’s Partner Advisory claimed. We reply that it’ll be 4th but we’re considering postponement given the new information from Google.
*   December 1st, 2023: Lenovo follows-up on their question. We opt to officially postpone disclosure until January 30th, 2024 and notify all OEMs of the change. At this point, no OEMs had yet released fixes to our knowledge.
*   December 4th, 2023: The [December ASB](https://source.android.com/docs/security/bulletin/2023-12-01) comes out. As promised, it contains no details an attacker could use to discern the issue.
*   December 7th, 2023: Fairphone thanks us for the postponement, says they intend to provide a statement, and asks permission to credit us in their own disclosure. We accept, and a couple weeks later we exchange statements and links where our respective disclosures will appear.
*   January 16th, 2024: Google offers us a $7,000 bounty for our report, which we ask them on January 25th to donate to charity. (Google, like Meta, doubles bounties paid to charity.)
*   January 25th, 2024: Lenovo asks for confirmation that January 30th is still our planned disclosure date, which we give.
*   January 30th, 2024: This post, [our disclosure](https://github.com/metaredteam/external-disclosures/security/advisories/GHSA-wmcc-g67r-9962), [our PoC code](https://github.com/metaredteam/rtx-cve-2023-45779), and [Fairphone’s post](https://www.fairphone.com/en/2023/12/22/security-update-apex-modules-vulnerability-fixed) all go live.
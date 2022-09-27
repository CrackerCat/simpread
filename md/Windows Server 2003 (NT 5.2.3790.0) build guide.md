> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [rentry.co](https://rentry.co/build-win2k3)

> Version 10b, last updated 2021/10/21 Instructions tested under XP SP3 x86, Win7 SP1 x86/x64 & Win10 x......

Windows Server 2003 (NT 5.2.3790.0) build guide[](#windows-server-2003-nt-5237900-build-guide "Permanent link")
---------------------------------------------------------------------------------------------------------------

_Version 10b, last updated 2021/10/21_

![](https://i.imgur.com/emAYGzM.png)

Instructions tested under XP SP3 x86, Win7 SP1 x86/x64 & Win10 x64, results may vary under other operating systems.

Guide maintained by a nameless anon with no affiliation with any group working on the code. **Do not trust anyone claiming authorship!**

* * *

Contents[](#contents "Permanent link")
--------------------------------------

1.  [A small request](#a-small-request)
2.  [FAQ](#faq)
3.  [Build Preparations](#build-preparations)
4.  [Building](#building)  
    a. [Troubleshooting](#troubleshooting)
5.  [Debugging](#debugging)
6.  [Additional Info](#additional-info)
7.  [Code Additions](#code-additions)
8.  [Changelog](#changelog)
9.  [File Hashes](#file-hashes)

A small request[](#a-small-request "Permanent link")
----------------------------------------------------

Hey all, been a while, thought I'd lost the password for editing this page but actually did have it tucked away somewhere, phew!

Unfortunately it seems over the past year almost everything linked here has died - whoever came up with the "everything on the internet is forever" meme was a real idiot.

Luckily I still have most of the things mentioned on this page, so I've reupped them to (hopefully) better file hosts, and might try getting a torrent working soon.

There's just a few things that I'm still missing however:

*   ~`win2003_x86-missing-binaries_v2.7z`~ (thanks to an anon for reupping this! link updated below)
*   ~the source-port from WXP for `processr.sys, amdk6.sys, amdk7.sys, p3.sys & crusoe.sys`~ (thanks to the same anon for reupping it, have added link below)
*   the source-port for `shell\osshell\accessib\magnify` folder
*   the source-port for `base\fs\utils\dfrg` folder
*   ~the `20201202__KERNEL32.DLL.v6.zip` package linked by [https://rentry.co/kernel32-extras](https://rentry.co/kernel32-extras)~ (found! many thanks to the anon that reupped it)
*   ~the `20201202_downlevel.zip` package linked by [https://rentry.co/extras-downlevel](https://rentry.co/extras-downlevel)~ (found! thanks to the anon that reupped it)
*   vaguely recall that a smart RE-anon made a better pidgen than pidgenXP_cracked, unfortunately can't remember much about it now though (or it might have been license-related things that were reversed besides pidgen, stuff related to server licensing maybe?)

Would really like to get hold of those so I can make sure the page is fully updated, if you happen to still have any of these files from the old /wxp/ threads please help us by reuploading them! (you can post them in the /t/ [source-code thread](https://boards.4chan.org/t/thread/1070062), I usually lurk there so should see it hopefully, feel free to leak any other sources you have there too of course!)

FAQ[](#faq "Permanent link")
----------------------------

###### What source code leaked?[](#what-source-code-leaked "Permanent link")

On 23rd September 2020 a ~2.9GB `nt5src.7z` file was posted to 4chan's /g/ board, containing leaked partial (~70% complete) source code for Windows XP SP1 & Server 2003.

This code has apparently been going around in private circles for several years, but was mostly unknown to the wider internet until this recent leak.

The archive contains a majority of the code needed for a full Windows XP/Server2003 install, minus any activation/cryptographic or third-party code.

<table><tbody><tr><td></td><td><pre>Filename: nt5src.7z
Size: 3,149,677,191 bytes

MD5: 94DEA413D439DDA8ABCAC83CFE799FC7
SHA-1: 350B2617D3095517A8D1981062C9D88A48B5D1A2
SHA-256: 2BB3609FA4C2B2641F43AEF751A84DB5820B64748B7D2D0891D1CB1E55268CE9

</pre></td></tr></tbody></table>

If you want to find a clean, original copy of the leak, just search for "nt5src.7z notrepacked" on your favorite search engine, the legit torrent magnet starts with `magnet:?xt=urn:btih:1a4e5...`

###### Can we build a working OS from the code?[](#can-we-build-a-working-os-from-the-code "Permanent link")

Yes! Many anons have built their own versions of Server2003, and XP-on-2003-kernel. Users on /g/ were actually the first to publicly show a working build made from the code (installing & booting successfully) [less than a week after the leak showed up!](https://desuarchive.org/g/thread/77963845/#77965472)

Unfortunately the process requires some files that don't have source included though, but these are all linked in the build guide below.

It's also worth pointing out that while the leak contained two different source trees (XPSP1 build 2600.1106 + Server2003 build 3790), we've only managed to get a Server2003 build completely working so far, which is why this guide focuses more on Server2003 than XP.

This is probably a good thing anyway as the Server2003 code is almost a year newer than XPSP1 (though it'd still be nice if XPSP1 could be made to work eventually...)

###### What about 64-bit? Can I build x64 editions?[](#what-about-64-bit-can-i-build-x64-editions "Permanent link")

64-bit support is kinda here, the leak has code for both IA64 & AMD64 processors, but while XP/Server2003 both released with IA64 versions from day one, AMD64 support was only added many years later with XP x64 & Server2003 SP1 (in other words, many years after this source code...)

IA64 support will probably work fine since the XP/Server2003 from the time of this source code had support for it (though afaik it hasn't actually been tried yet), but seems AMD64 was still in development at the time.

The v10 release of this guide improves support for making AMD64 builds, and some anons have even some success getting AMD64 builds to start booting, but unfortunately problems with Wow64 & possibly other things are preventing it from starting properly atm. (see [amd64 build support](#amd64-build-support) section for more info)

###### What's needed to build? Do I need Visual Studio?[](#whats-needed-to-build-do-i-need-visual-studio "Permanent link")

Fortunately the complete build toolchain was included in the leak, all that's required is a Windows build machine running XP or later.

Visual Studio isn't needed, and currently there aren't any VS solution files/projects for interacting with the code, would be really nice to have them though.

###### Will this help get XP tools/kernel running on *nix?[](#will-this-help-get-xp-toolskernel-running-on-42nix "Permanent link")

Unlikely since the tools are coded to work in a completely different environment, maybe if someone wanted to put in the months of work to port it over, but it's far from being code you can just copy-paste into *nix and then build.

###### Is this code going to help ReactOS/WINE in any way?[](#is-this-code-going-to-help-reactoswine-in-any-way "Permanent link")

No, those projects would never want to make use of such stolen/illegal code (ReactOS even performed their own year-long code audit when it was claimed that they used an older leaked codebase!).

Even hinting that you have knowledge of Windows internals learned through code like this is enough to disqualify you from contributing to them (on that note, **if you ever want to work on ReactOS/WINE in the future you should probably quit looking into this source code while you still can!**)

###### Did this leak come from the passworded RAR I heard about?[](#did-this-leak-come-from-the-passworded-rar-i-heard-about "Permanent link")

No, the passworded RAR was a completely different thing which turned out to be fake.

Before the real leak came about some anons on /g/ were trying to organise a source code collection for various OS's, with that collection also containing a passworded `windows_xp_source.rar` file that was dug up after years of being unseeded (posted in ~2007), which some anons spent several weeks trying to crack.

Sometime after the nt5src file leaked the password for that RAR was eventually found, being `internaldev`, which revealed that the RAR was nothing but a fake archive that made use of similar directory structures to the older NT4 leak.

Unfortunately the weeks of posts about the lost-then-found RAR file being followed by an actual source code leak led many anons to confuse the two things as being the same, so just to emphasise: **the passworded-rar has nothing to do with the recent leak!**.

###### What's the deal with this NOTREPACKED/repackfag guy?[](#whats-the-deal-with-this-notrepackedrepackfag-guy "Permanent link")

The confusion around the leaks was made even worse thanks to some idiot almost immediately deciding to repack the original nt5src.7z leak with different compression - using the exact same filename for his repacked version - and then making the first torrent of the leak contain that repack instead of the original, pretty much fragmenting the leak & attempting to erase the history behind it, all just to save a few hundred MB.  
Luckily it didn't take long for NOTREPACKED-chad to come through with a proper torrent of the non-repacked files that were originally gifted to us by leaker-anon, which has been linked in the threads ever since.

The repackfag will usually come along to the thread every now and then to stir things up & spam schizo-tier narratives, likely in some attempt at convincing any tourists that might be visiting (which is pretty sad when 90% of the time the thread has the exact same people inside it who have long tired of his act). It's safe to say this guy has no interest in actually contributing and just wants to bring down the threads (odd considering this guy spent weeks making begging threads for the XP code...)

Anyway if you need to check what version of the leak you have, refer to the hashes of the original nt5src.7z above.

Build Preparations[](#build-preparations "Permanent link")
----------------------------------------------------------

* * *

*   It's recommended to disable any AV before extracting/building, as both of those actions create a lot of new files (your AV will likely try scanning every one, slowing down the extraction/build by quite a bit) - this also counts for any other tools that monitor files such as voidtools' Everything.
*   **Ensure build machine date is current** - there's no longer any need to set date to 2003 or anything.
*   Extract source tree to a folder named `srv03rtm` on the root of a drive (**important, as pre-built DirectUI files will only link properly under this path**), any drive letter (**besides C:**) should be fine, use `D:\srv03rtm\` as the path to match RTM binaries.
*   Unset Read-only on extracted directory, including subfolders and files (note that after unsetting this and closing/reopening the folder properties again you might see that read-only has been set again, this is fine, as long as you unset it once it should let the build work without issue)
*   Extract the [win2003_prepatched_v10a.zip](https://mirrorace.org/m/3Lxas) into your source tree, overwriting existing files as necessary
*   **If using a 64-bit host OS to build with:** copy the contents of the ZIPs `_x64` folder into the source tree, overwriting if asked.
*   **If using this guide after Oct 2021:** you'll need to renew the test certificates used during the build first - a helpful anon has made a guide for doing this here: [https://rentry.co/win2k3-certutil](https://rentry.co/win2k3-certutil)

**If your OS doesn't use UAC (XP/2003)**:

*   Create desktop shortcut for `%windir%\system32\cmd.exe /k D:\srv03rtm\tools\razzle.cmd free offline` (see below for explanation) and change `Start in` to `D:\srv03rtm`
*   **If using 64-bit host OS** use `razzle64.cmd` in the shortcut instead of `razzle.cmd`
*   Open razzle window using shortcut you created.

**If your OS uses UAC (Vista+)**:

*   Run command-prompt as administrator (can usually be done by typing cmd into start menu, right click `Command Prompt` -> `Run as administrator`)
*   In the command-prompt, change to the drive you extracted source-code to by typing in the drive letter , eg. `E:`
*   Change to the source folder: `cd srv03rtm`
*   Now start razzle: `tools\razzle.cmd free offline` (**If using 64-bit host OS** use `tools\razzle64.cmd free offline` instead)

The first time you run razzle inside that copy of the source code it'll need to initialise a few things, give it a few minutes, after a while a Notepad window will appear - make sure to close this for the initialisation to continue.

**Important:** Once razzle has initialised run `tools\prebuild.cmd` to finish preparing the build environment (only need to run once after initing razzle for first time in this tree)

Building[](#building "Permanent link")
--------------------------------------

* * *

**Important:** Currently the build doesn't seem to play well when building with more than 4 threads. If your build machine has more than that it's recommended to cap it to 4 threads maximum via the `-M 4` switch, added to the build command (eg. `build /cZP -M 4`, or `bcz -M 4`)

### Clean build[](#clean-build "Permanent link")

Performs clean rebuild of all components (**recommended for first build!**):

*   `build /cZP` (`bcz` is also aliased to this)

### "Dirty" build[](#dirty-build "Permanent link")

Builds only components that have changed since last clean build:

*   `build /ZP` (`bz` is also aliased to this)

### Post-build[](#post-build "Permanent link")

*   Download the [win2003_x86-missing-binaries_v2.7z](https://mirrorace.org/m/4rv80) pack, which contains missing binaries for both x86fre & x86chk builds.
*   (unfortunately this is quite a big pack, and it's likely the link will inevitably go down some day, however this pack isn't actually required - instead you can make use of `missing.cmd` with each of the win2k3 ISOs listed below, which should be pretty simple to track down)
*   From that 7z, extract the contents of the binaries folder for the build type you're building into your build trees binaries folder (eg. `D:\binaries.x86fre`, should have been created during the build), the 7z should contain files for all SKUs (uses pidgen.dll from Win2003 Enterprise, so your builds should accept Enterprise product keys)
*   **When asked during extraction to overwrite folders select `Yes`, but when asked to overwrite files like DUser.pdb/dll make sure to select `No`!**
*   Once missing files have been added, you should have files such as `binaries.x86{fre/chk}\_pop3_00.htm`, `binaries.x86{fre/chk}\ql10wnt.sys`, etc.
*   Inside the razzle window run `tools\postbuild.cmd` (use `-sku:{sku}` if you want to process only specific one (no brackets!), expect `filechk` errors if you ignore this and didn't use missing.7z / missing.cmd with every sku)

Once postbuild has finished, assuming you used the `win2003_x86-missing-binaries.7z` file above and followed the guide properly, it should have hopefully succeeded without errors, and there shouldn't be any `binaries.x86fre\build_logs\postbuild.err` file!

Otherwise take a look inside the `postbuild.err` - most messages in here are negligible, but if you see `filechk` errors associated with the edition you want to use, you may need to re-run `missing.cmd`, or extract `2k3-missing.7z` again.

If `postbuild.err` contains messages like `(crypto.cmd) ERROR` or `(ntsign.cmd) ERROR` try re-importing the `tools\driver.pfx` key-file (double-click it, press Next through the prompts, password is empty), and make sure your system date is set to the current date (updated test certs are only valid from October 2020 to October 2021)

### Creating bootable ISO files[](#creating-bootable-iso-files "Permanent link")

*   Execute `tools\oscdimg.cmd {sku} [destination-file (optional)]` where `{sku}` is one of:
    *   `srv` - Windows Server 2003 Standard Edition
    *   `sbs` - Windows Server 2003 Small Business Edition
    *   `ads` - Windows Server 2003 Enterprise Edition
    *   `dtc` - Windows Server 2003 Datacenter Edition
    *   `bla` - Windows Server 2003 Web Edition
    *   `per` - Windows XP Home Edition
    *   `pro` - Windows XP Professional
*   ISO will be saved to `{build-drive}\{build-tag}_{sku}.iso`, unless `[destination-file]` is provided as a parameter.

### Troubleshooting[](#troubleshooting "Permanent link")

If you get any problems during the build hopefully your problem might be answered here, if it's not feel free to post in the thread.

###### After running razzle it shows a bunch of errors, "tfindcer not recognized" etc[](#after-running-razzle-it-shows-a-bunch-of-errors-tfindcer-not-recognized-etc "Permanent link")

You're likely on a 64-bit system but running `razzle.cmd` directly, you should use `razzle64.cmd` instead, this'll set things up so that razzle will use the correct tools for you. Hopefully with that the `tfindcer not recognized` & other errors should go away.

###### During build I get a `directui.lib(parse.obj) LNK2011` error, mentioning `precompiled object not linked in`[](#during-build-i-get-a-directuilibparseobj-lnk2011-error-mentioning-precompiled-object-not-linked-in "Permanent link")

This is caused by the pre-built parse.obj file we currently have to use in order to build a working directui.lib, currently this file requires your source tree to be inside a `srv03rtm` folder on the root of a drive, using any other folder will cause the error to happen. Sadly attempts to fix this such as editing the parse.obj or updating the code to build parse.cpp instead haven't been successful yet.

###### Any idea how to fix `CK1011: type information corrupt, recompile module map_kv.cxx` error?[](#any-idea-how-to-fix-ck1011-type-information-corrupt-recompile-module-map_kvcxx-error "Permanent link")

Seems to be caused by switching between fre/chk, clean-build fails to delete some remnant from whatever you built last which breaks the build.  
Manually deleting the `\com\ole32\olethunk\ole16\compobj\obj\` folder & `\com\ole32\olethunk\ole16\compobj\msvc.pdb` file should allow clean-build to work again, after that just `bcz` inside the `\com\ole32\olethunk\ole16\` dir.

###### After a full build I get 1000+ errors, build.err has `has bad storage class` errors inside[](#after-a-full-build-i-get-1000-errors-builderr-has-has-bad-storage-class-errors-inside "Permanent link")

Seems to be caused by your locale, this is reported to happen with Simplified Chinese language, but can probably happen with other languages too.  
Besides changing your locale/language settings, you could also try editing the source files to remove the characters that cause problems, an anon posted a list of the files that need changes [here](https://archive.rebeccablacktech.com/g/thread/78406946/#78416976)

###### On Windows 7 I get NTVDM crashes while building[](#on-windows-7-i-get-ntvdm-crashes-while-building "Permanent link")

All NTVDM errors should have hopefully been solved in the v8 release of this guide/ZIP, but if you still run into any NTVDM error a copy of your build.log file would help a lot to figure out what caused it, if you ZIP that up and post in the thread hopefully we can figure something out.

Debugging[](#debugging "Permanent link")
----------------------------------------

* * *

Kernel debugging can be enabled by editing the `C:\boot.ini` file in your installed build, and adding `/debug /debugport=COM1 /baudrate=115200` to the end of a config line.

For example here's a boot.ini with two choices, one with debugging & one without, with this NTLDR will show an OS boot choice menu at startup:

> [boot loader]  
> timeout=30  
> default=multi(0)disk(0)rdisk(0)partition(1)\WINDOWS  
> [operating systems]  
> multi(0)disk(0)rdisk(0)partition(1)\WINDOWS="Windows Server 2003, Enterprise" /fastdetect  
> multi(0)disk(0)rdisk(0)partition(1)\WINDOWS="Windows Server 2003, Enterprise" /fastdetect /debug /debugport=COM1 /baudrate=115200

(even though they're named the same `Windows Server 2003, Enterprise`, NTLDR will add a `[debugger enabled]` tag to it on the OS selection menu.)

`chk (checked)` builds are preferred for debugging as they enable asserts & debug messages, though `chk` will run a lot slower than `fre` builds. `fre` builds can also be debugged fine, but usually with a lot less debug output.

Ideally you should also configure WinDbg to use the `binaries.{buildtype}\symbols.pri\retail` & `binaries.{buildtype}\symbols\retail` symbols folders too, if WinDbg is running on the same machine that compiled the build it should then also be able to do source-level debugging against the actual source files.

Note that if you're using a VM to host your build on you'll have to pass-through the emulated COM port over to a pipe somehow, for WinDbg/KD/IDA/etc to make use of.

There is apparently a way to get KDNET working on 2003 too (see [https://github.com/MovAX0xDEAD/KDNET](https://github.com/MovAX0xDEAD/KDNET)), but this is as-of-yet untested with our builds, seems like it should [have support](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/supported-ethernet-nics-for-network-kernel-debugging-in-windows-8-1) for VMware's emulated network hardware at least.

(TODO: add more VM guides here... if anyone wants to post one in the thread I'm happy to add it here)

### VMware setup[](#vmware-setup "Permanent link")

For a VMware VM, just open the VM hardware options, **remove the printer hardware**, then click `Add...` and choose `Serial Port`.

In the serial-port config panel, set it to `Use named pipe`, and enter a pipe name into the textbox, eg. `\\.\pipe\vm`. The pipe should be setup as `This end is the server` & `The other end is a virtual machine`.

It's apparently recommended to use the `Yield CPU on poll` option, but I've had success without it, it's up to you.

With that setup now just open WinDbg, choose `Attach to kernel`, open the `COM` tab and enter the baud-rate and pipe name you selected for your VM, then hit OK.

Now WinDbg should start with `Waiting for reconnect` text showing, and hopefully when you next boot up your VM it'll connect up to WinDbg by itself.

### Debugging Setup/Installation[](#debugging-setupinstallation "Permanent link")

A debugger can be attached to setup by pressing F8 at the right time - for text-mode setup the right time is when asked `Press F6 if you need to install a third party SCSI or RAID driver`, spamming it while that message is showing should make it connect to COM1 with 19200 baud-rate (debug options can be changed inside `txtsetup.sif`, the `SetupDebugOptions` line specifies options to apply when F8 is pressed). After debug connection is made it might take a couple extra minutes for it to pass the `Setup is starting Windows` stage.

Similarly, GUI setup can be debugged by pressing F8 before the Windows bootloader starts (progress bar etc), this should bring up a menu with options for `Safe Mode`, `Debugging Mode`, etc. Choosing `Debugging Mode` will make it attach to COM1 before starting setup, again at 19200 baudrate.

### Changing default baudrate[](#changing-default-baudrate "Permanent link")

The default 19200 baudrate can be changed by editing the `base\boot\kdcom\xxkdsup.c` file, search for `BD_19200` inside there and change it to e.g. `BD_115200`, now it should use that by default without needing any `/baudrate` parameter, or eg. when using `Debugging Mode` from the boot options menu.

### Un-filtering kernel messages[](#un-filtering-kernel-messages "Permanent link")

Even with a chk build you may notice that the kernel doesn't seem to output much over KD, this seems to be because most of the kernel components use `KdPrintEx`, which allows filtering KD messages to certain components (although MS's docs seem to suggest KdPrintEx filtering was only used in Vista, it does seem that 2003 makes use of it too)

To enable components you'll need to open the registry of your installed build to the following key (or create it if it doesn't exist)

> HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\Debug Print Filter\

Then create a DWORD value named the component you want to enable (eg. `LDR`), and set the value to hexadecimal `FFFFFFFF`.

A list of the component registry names can be found inside `base\published\obj\i386\dpfiltercm.c` after a build. This also contains the symbol names for the component (eg. `Kd_LDR_Mask`), which can be set directly through WinDbg/KD via the symbol name. (eg. `ed nt!Kd_LDR_Mask 0xFFFFFFFF`, with symbols loaded this should set the LDR components filter value in real-time)

The `WIN2000` components value (default `1`) is added to every components value, essentially making `WIN2000` the default value for all components - so setting this to `FFFFFFFF` should make every component print to KD. You'll get a ton more debug output from this, but all the KD prints will likely slow down the system a lot (unfortunately it doesn't seem possible to disable a component when using WIN2000 to enable them all)

If you need to enable components before your build is installed (eg. kernel is having problems before setup is completed), you'll need to edit the `mergedcomponents\setupinfs\hivesys.inx` file, anon [made a post about that here](https://archive.rebeccablacktech.com/g/thread/78588821/#78626788), though it hasn't been tried yet afaik.

Additional Info[](#additional-info "Permanent link")
----------------------------------------------------

* * *

### prepatched.zip additions list[](#prepatched46zip-additions-list "Permanent link")

*   New unexpired test-signing certificates (valid to October 2021 - tools\openssl.txt describes how to generate them)
*   Updated `midl.exe`/`midlc.exe` from Win2003 SP1 DDK, fixes olepro32.dll errors
*   Reordered `dirs` file to ensure that `conlibk.lib` is built before it gets used
*   parse.cpp/parse.hpp files required to build DirectUI.lib, and the bison.exe/Bison.skl files used to generate them.
*   Pre-compiled parse.obj file, as the parse.cpp/parse.hpp mentioned above has some issues parsing things.. (parse.obj taken from win2003 directuid.lib - causes LNK4206 warnings when using it though, suppressed via changes to `sources` files)
*   Pre-compiled GdiPlus v1.0.100.0 from RTM ISO, to be included into `asms01.cab` (x86 only), as the GdiPlus code sadly can't be built.
*   Updated DUser build scripts, to make sure it gets placed in the proper location & gets built with the right optimization flags.
*   Updated `windows\advcore\dirs` file to allow DUser/DirectUI to get built during main build
*   Updated 16-bit build tools inside com\ole32\olethunk\ole16\tools\, and updated olethunk code so it'll build fine with them. (updates are only used if OS requires them to build, XP/2003 should be able to build them fine without changes)
*   Disabled fixprn.pl, ntbackuponpersonal.cmd, gpmc.cmd, msi.cmd & incbbt.cmd calls from pbuild.dat, as we're missing required build-files for those (you can grab the results of those scripts from RTM ISO, or from the `2k3-missing-x86fre.7z` pack)
*   Updated `setupw95.cmd` & `drivercab.cmd` to add a delay between async calls, fixes an issue with filename-collisions. (update only needed under newer OS's, older OS's like XP don't seem to have this issue)
*   Updated `razzle.cmd` to always set `__BUILDMACHINE__` variable, allows build tag built into kernel to match with BuildName.txt (and won't leak your build OS username into the build-tag any more)
*   Updated `razzle.cmd` to always set `NO_PDB_PATHS=1`, the build already uses FixPdbPaths.exe to remove file-paths from PDB, but that method leaves null-characters in place which aren't in the retail files, may as well enable this since there's no reason not to.
*   Updated `ixsso` makefile, to prevent MIDL2346 warning from breaking the build (warning seems locale related, for some reason `BUILD_ALLOW_MIDL_WARNINGS` only worked when defined inside makefile, pretty strange)
*   Reordered `windows\appcompat\dirs` file to help with single-core builds (from idkwhy's OpenXP git repo)
*   Updated `msitoddf.cpp`, now closes the MSI package handle when finished (prevents stalling postbuild on Win10), and added fix for 64-bit build OS (redirects SysWOW64 to System32)
*   Updated `mshtml.ref` reference file - used to compare mshtml output against known-good one? last updated in 1999(!), without updating it seems to randomly cause build error in certain conditions (I'm not sure why we never had problems with this until we tried 64-bit stuff though...)
*   [x64] Added 32-bit mapsym.exe & rc.bat MSDOS-Player wrapper to `printscan\faxsrv\print\faxprint\faxdrv\win9x\sdk\binw16` dir, since some anons seemed to get errors from this folder.
*   [x64] Replaced `masm.exe/mkpublic.exe` with 32-bit versions (taken from Sizzle), as some anons had issues with MS-DOS Player reporting a bad DOS version, breaking those two tools as they require DOS 2.0+
*   [x64] Replaced MS-DOS Player for some 16-bit tools with recompiled amd64 versions, provided by an anon in the threads (many thanks!), source available inside `_x64\tools\tools16\16_bit_build_tools_v2.zip`
*   Added missing `public\internal\windows\lib\amd64\usp10p.lib` library needed for amd64 build, taken from XPSP1 tree.
*   Added parse.obj to `windows\advcore\duser\directui\engine\parser\obj\amd64` & `objd` folder, extracted from amd64 `directui.lib` file.
*   Added `msgina_sp1.def` & `userenv_sp1.def` from the winlogon200X pack, these will make amd64 builds of msgina/userenv use the SP1 export ordinal numbers, improving compatibility with SP1 winlogon & possibly other SP1 files.
*   Added `exinit.c` & `systime.c` from anon's decompilations, along with original exp.h (to overwrite older modified exp.h from earlier prepatched zip)
*   Added pre-generated `inetsrv\iis\svcs\cmp\webdav\davcprox\fhcache_p.c` file, includes defs for both x86/x64, should help with issues switching between x86/x64 builds.
*   Added `link.bat`/`link16.bat` wrappers for `link.exe`/`link16.exe`, which randomize TEMP env var and give it 5 retries before failing.
*   Updated rc16 call inside `net\tapi\thunk\makefile.inc`, changed `WINNT=1` define to `WINNT` to fix an error under some NTVDM versions.
*   `razzle64.cmd` which can take care of converting 16-bit tools to 32-bit (via `MS-DOS Player`), and setting required environment variables before launching razzle.
*   `prebuild.cmd` that can handle installing driver.pfx keys, fixing file attributes, removing updated files if OS doesn't require them, and copying GdiPlus SxS policies.
*   `missing.cmd` that can copy files we don't have source for from a mounted ISO.
*   `oscdimg.cmd` to generate an ISO image from a finished post-build.

### x64 build OS support[](#x64-build-os-support "Permanent link")

prepatched.zip v9 adds support for using x64 build OS's such as Win10 x64, this is done by wrapping certain 16-bit tools using `MS-DOS Player`, using .bat files to redirect calls to use the player, changing some makefiles to use 32-bit equivalents, etc.

Unfortunately some of the wrapped 16-bit tools can still randomly error without rhyme or reason for it, as a workaround the .bat files of the worst offenders will give it 5 attempts before failing, hopefully this should be enough to allow builds to complete fine, but there's still a chance one of the other 16-bit tools could error too... maybe in future I'll apply this 5-attempts bandaid over all the 16-bit tools.

Huge thanks to the anon who initially worked on fixing the 16-bit tools over at [https://rentry.co/16bit-msbuild](https://rentry.co/16bit-msbuild)!

### amd64 build support[](#amd64-build-support "Permanent link")

As of update v10 an amd64 build can be created by initialising razzle with the `win64 amd64` options, the build should mostly complete without errors, but note that postbuild+ hasn't been updated at all to work properly with amd64 yet.

Unfortunately as the leak didn't come with exinit.obj/systime.obj for amd64 these need to be built from anon's decompilations instead. v10a adds newer exinit.c/systime.c decompilations that are apparently an exact match to the x86 .obj files included in the leak, hopefully these should work well with other architectures too.

Some anons have been slowly working on amd64, being able to get past text-mode setup and start booting GUI-mode setup, though sadly as of this release nobody has been able to actually get GUI mode to fully start up.

Note that this source code is from around ~2 years before amd64 was officially released by MS as Server2003 SP1, so there's likely a lot missing. (WRK may have some newer kernel-mode parts, being both based on SP1 and including support for amd64, though note that WRK also has many things removed too...)

However there's also many indications in the leak that MS did have amd64 running at this point, so it should eventually be possible for us to get it working too.

### Timebomb[](#timebomb "Permanent link")

*   Time can be adjusted by editing `DAYS` variable inside `\tools\postbuildscripts\timebomb.cmd` (line 44)
*   Setting `DAYS` to `0` will disable the timebomb.
*   Only certain `DAYS` parameters are valid (0, 5, 15, 30, 60, 90, 120, 150, 180, 240, 360, 444)

### Different build options[](#different-build-options "Permanent link")

You can modify your razzle shortcut (or execute it manually inside your source folder) to include (or remove) additional argument(s):

*   `free` - build 'free' bits (production, omitting it will generated checked bits)
*   `chkkernel` - build 'checked' (testing) kernel/hal/ntdll when building 'free' bits
*   `no_opts` - disable binary optimization (useful for debugging, but will most likely fail a full build, some code can't be built without optimization)
*   `verbose` - enable verbose output of the build process
*   `binaries_dir <basepath>` - specifies custom output directory (default is `binaries`, the suffix added after `.` is non-customizable)
*   `officialbuild` - sets razzle to build this as an "official" build, requires updating BuildMachines.txt, see the section below
*   `win64 amd64` - builds for amd64 instead of x86, see `amd64 build support` section above.

Other options are not described here, see `razzle.cmd /?` for details.

### 'OfficialBuild' parameter / BuildMachines.txt[](#officialbuild-parameter-buildmachinestxt "Permanent link")

The `OfficialBuild` razzle parameter changes a few things in the build, which will make it match up closer to the retail builds, should be useful if you need to compare against retail for any reason.

For a list of things affected by the OfficialBuild parameter see [https://pastebin.com/VgVph3Xv](https://pastebin.com/VgVph3Xv) & [https://pastebin.com/gYzWGLM5](https://pastebin.com/gYzWGLM5), thanks to the anon that compiled them! (note that these aren't complete lists, and not all things mentioned here are guaranteed to take effect).

**However, using this parameter requires a file to be updated with info about your build machine first!**

An easy way to update the file required is to run the following command inside a razzle window, at the root of the source tree:

`echo %COMPUTERNAME%,primary,%_BuildBranch%,%_BuildArch%,%_BuildType%,ntblus >> tools\BuildMachines.txt`

After that you can run `tools\verifybuildmachine.cmd` to make sure it was setup correctly, if there's any problem an error message will show, otherwise the command will return without any message.

With that in place you should now be able to use the OfficialBuild parameter next time you init razzle, eg. `tools\razzle.cmd free offline officialbuild`

Some small notes to be aware of:

*   if you change build arch or build type (eg. to amd64, or to a checked build) you'll need to run the echo command again to add your machine for that build arch/type combination
*   if you see `Clearing OFFICIAL_BUILD_MACHINE variable` after initing razzle, rerun the echo command and then close down/reinit razzle again, else the build won't properly count itself as official.

### Pseudo-localization builds[](#pseudo-localization-builds "Permanent link")

An anon has made some progress with localization, allowing "Pseudo-Localization" (PLOC) builds to be created in 3 different configurations, via certain razzle options & postbuild script changes, these builds should come in useful for people looking into creating non-English builds.

The three configs available are PSU (Pseudo), FE (Far East) and MIR (Mirrored), representing some of the main changes that localization might require (such as right-to-left text, etc)

Their instructions for creating these builds have been archived here: [http://archive.rebeccablacktech.com/g/post/78862319/](http://archive.rebeccablacktech.com/g/post/78862319/) & [http://archive.rebeccablacktech.com/g/post/78862415/](http://archive.rebeccablacktech.com/g/post/78862415/)

### Creating fresh postbuild[](#creating-fresh-postbuild "Permanent link")

*   `tools\postbuild.cmd -full`
*   `tools\missing.cmd`
*   `tools\postbuild.cmd`

Use `-sku:{sku}` if you want to process only specific one (no brackets!)

### Building specific components[](#building-specific-components "Permanent link")

Most components can be built seperately. For example, if you wish to rebuild `ntos` component, perform these steps:

*   `cd base\ntos` (you can also use `ntos` alias that razzle has set up for you)
*   `bcz` (alias for `build /cPZ`)

Generally `postbuild.cmd` is clever enough to include your changes properly without needing fresh build as it uses `bindiff` to find differences.

### Generating new build number/name[](#generating-new-build-numbername "Permanent link")

Version information is stored in `\public\sdk\inc\ntverp.h`

You can also use `m0 set_builddate set_buildnum set_buildname` to generate new build name quickly.

### Original CD filenames[](#original-cd-filenames "Permanent link")

*   `5.2.3790.0.srv03_rtm.030324-2048_x86fre_server-standard_retail_en-us-NRMSFPP_EN.iso` (SHA1: A600409482A5678EF6AF2B26D3576D6D9894178D)
*   `5.2.3790.0.srv03_rtm.030324-2048_x86fre_server-datacenter_retail_en-us-NRMDOEM_EN.iso` (SHA1: E2B47A7CE45C6C6305594CEE4C1B64894805AAF4)
*   `5.2.3790.0.srv03_rtm.030324-2048_x86fre_server-enterpriseserver_retail_en-us-NRMEFPP_EN.iso` (SHA1: 0309FFB4181BA5122C692A6E1079E9FC1D53FCE4)
*   `5.2.3790.0.srv03_rtm.030324-2048_x86fre_server-webserver_retail_en-us-NRMWFPP_EN.iso` (SHA1: 46C1CCB2CFC96803E304A35BEF50CD71B2C1DE38)
*   `sbs.iso` (converted from mdf; SHA1: CDB30C80FDE314C16CA11F5CD31650ECBEC7A214)
*   `5.2.3790.0.srv03_rtm.030324-2048_x86chk_server-enterpriseserver_retail_en-us-NRMECHK_EN.iso` (SHA1: EEF5F921CC8FC20FB29A862E1E132359E0D151BB)
*   `5.2.3790.1830.srv03_sp1_rtm.050324-1447_amd64fre_server-enterprise_retail_en-us-ARMEXFPP_EN.iso` (SHA1: 076EDCF017EDE0B2D0D8067FA52CF3D44EEEF79A)
*   `5.2.3790.1830.srv03_sp1_rtm.050324-1447_amd64chk_server-enterprise_retail_en-us-AX2EXCFPP_EN.iso` (SHA1: 8916DFBB1D93A9CECB1FE8600BE2E2C752E85E7F)
*   `5.1.2600.0.xpclient.010817-1148_x86fre_client-home_retail_en-us-WXHFPP_EN.iso` (SHA1: B273C8D41E3844E3E46722F52F5A4CF9F206C8D0)
*   `5.1.2600.0.xpclient.010817-1148_x86fre_client-professional_retail_en-us-WXPFPP_EN.iso` (SHA1: 1400DED4402D50F3864ED3D8DCF5CC52BA79A04A)
*   `5.1.2600.0.xpclient.010817-1148_x86chk_client-professional_retail_en-us-WXPFPP_EN.iso` (SHA1: 017F10E4555D1A9280874B9B0243442F045F1B2D)

### Product keys[](#product-keys "Permanent link")

*   Standard Edition: M6RJ9-TBJH3-9DDXM-4VX9Q-K8M8M
*   Enterprise Edition: QW32K-48T2T-3D2PJ-DXBWY-C6WRJ
*   Enterprise x64: KK2WD-BFYJ6-77X87-8TRBF-9B343

Code Additions[](#code-additions "Permanent link")
--------------------------------------------------

* * *

Some /g/ users have also added missing components & other fixes to the source, where available links have been provided here (though due to the nature of the internet, it's very likely these links will eventually go down... if anyone ever makes a torrent for these files please let me know, I'd be happy to help seed it!)

A set of patches created by a helpful anon, recommend checking their guide out here, contains some very useful info: [https://rentry.co/win2k3-extra-patches](https://rentry.co/win2k3-extra-patches)

###### XP source-ports[](#xp-source-ports "Permanent link")

Some components missing from the Win2003 source can be found in the XPSP1 tree, although usually older than the component binaries shipped with Win2003 RTM (missing certain bugfixes etc). Some anons have had success porting them over to the Win2003 tree, & even updating them based on the Win2003 binaries.

*   `base\hals\processor\`: responsible for `processr.sys, amdk6.sys, amdk7.sys, p3.sys & crusoe.sys`

Wouldn't build in the Win2k3 tree at first but an anon found what edits were needed to build, another anon later added updates to the code based on the Win2003 driver binaries. Has mostly been updated to Win2003 SP0 level, except for the `GetAcpiTable` function.

**Latest version as of last guide update:** [processorXP_win2003_update.7z](https://mirrorace.org/m/57szw)

*   `base\ntsetup\pidgen\`: responsible for `pidgen.dll`

The Win2003 RTM binaries expose some new exports missing from the XP version, required for Win2003 setup and other things to use it.  
These were added by an anon, but it was later found that Win2003 also makes use of a newer crypto library that allows for larger public keys to be used, which sadly isn't available anywhere in the source packages.

This makes updating the XP code to match Win2003's binaries a lot more difficult, but the updated code that uses newer exports can still come in useful for making a DLL that win2003's setup can work with - eg. an anon has used this code to make a pidgen that accepts any key, seems to work great as long as winlogon doesn't have WPA crap active.

**Latest version as of last guide update:** [win2003_pidgenXP_cracked.zip](https://mirrorace.org/m/3Lxaw) (requires winlogon200X or patched winlogon 2003)

*   `shell\osshell\accessib\magnify`: responsible for `magnify.exe & mag_hook.dll`

Builds fine under Win2003, one anon figured how to update magnify.exe code to match the Win2003 binary (as narrator.exe code had the same changes made, and luckily we have the Win2003 version of that, the changes could easily be copied over)

*   `base\fs\utils\dfrg`: responsible for `defrag.exe, dfrgntfs.exe, dfrgfat.exe, dfrgres.dll, dfrgsnap.dll & dfrgui.dll`

Builds fine under Win2003, code is older than the actual Win2003 binaries though and missing some small updates/bugfixes.  
An anon later provided patched code that brings it closer to the Win2003 versions, although:

> I couldn't get GetExtentListManuallyFat & ScanNtfs matching, decided to leave those as they were in XPSP1 as I don't trust my filesystem-code-writing skills that much.

Maybe some filesystem-wizard could get it updated fully, besides that there were also troubles with PDB paths, these were found to be mostly caused by razzle.cmd which was then fixed in a newer update of this guides accompanying ZIP file.

###### winlogon.exe[](#winlogonexe "Permanent link")

Unfortunately code for winlogon wasn't included in this source kit, likely due to winlogon being one of the main files used in WPA. This prevents our builds from being able to boot up properly unless winlogon.exe is taken from a retail release (which is tampered with WPA activation & code obfuscation and who knows what else...)

To try getting around this an anon posted a version of winlogon's code from win2000 ported over to server2003, and later another anon then fixed this port to re-enable the InitShutdown service (which was removed during the port due to missing files), and then found a fix to allow this built winlogon to get to the logon screen without restarting, while this was a good start at making the login prompt actually show without crashing unfortunately trying to logon would only greet you with an error message.

A few weeks went by with little progress, until a day when some anons started talking about getting the amd64 build booting, one of the main problems identified was winlogon - the only available amd64 binaries being from SP1+, which didn't seem to work properly on our SP0 amd64 builds. This talk about winlogon eventually spurred an anon to start working on updating the win2000 code to match closer with the win2003 binary (a list of some of the changes made are available [here](https://archive.rebeccablacktech.com/g/thread/78500072/#78504881))

Not long later a fixed version was produced that finally allowed logging into the system successfully, albeit with some issues such as fast-user switching not being supported, and profiles not being setup correctly, besides these small issues it overall seemed to work pretty well.

Sadly by the time anons had winlogon working & were starting to play with AMD64, the /wxp/ threads were also starting to die out, with idiots invading the thread to cause stupid arguments over trivial shit, making more & more people leave the threads for good...

However, one anon did actually stick around & managed to reverse XP's winlogon properly, even releasing their decompilation that worked pretty much perfectly, sadly most anons never saw this as it was during the dying days of /wxp/, so many still think we're stuck with the hacked up win2000 build - but now you know that isn't the case!

*   **Latest version as of last guide update:** [ds.zip_decompiled_XPSP1_winlogon.zip](https://mirrorace.org/m/3Lxat) - decompiled XPSP1 winlogon (not the frankenstein win2000 version!) - alternatively, the franken-win2000 version: [Winlogon200X_v3c.zip](https://mirrorace.org/m/5Nq31)

(note that winlogon200X requires a small change for postbuild to succeed with this winlogon: edit `winlogon\sources.inc`, find the `DELAYLOAD` line, and change it to `DELAYLOAD=winmm.dll;netapi32.dll;ole32.dll;msgina.dll` - I'm not sure whether ds.zip also requires this or not)

###### systime.c/exinit.c[](#systimecexinitc "Permanent link")

These two source files are missing from the base\ntos\ex folder, instead replaced by pre-compiled .obj files with the same name. Some investigating into these obj files shows a lot of WPA code being present, likely why code wasn't provided for them.

An anon posted re-created .c files based on WRK's objs which were made a while ago, these seemed to work fine on server2003, but were later found to cause BSODs. That anon later posted a fixed version which apparently solved the BSODs, but later anons also found this fixed version still had some issues of it's own (using spinlocks while the .obj never did, missing functions that win2003 had but WRK didn't), and another anon also found that the system-clock seemed to move a lot slower when using these re-creations.

Over time some small improvements were made to these files (with fixes like removing spinlocks), and an anon did some work getting systime.c to match with the .obj, mostly with success besides some minor parts.

After a few weeks an anon started posting exinit.c files which were "100% matching" for anons to test, once other anons tried them out and a few minor bugs were found & fixed that anon then posted a pack that also contained a systime.c recreation they were working on, that was also "100% matching". Since then it's had a few small updates to improve checked builds & fix 64-bit building, but generally appears to work fine.

So now after only a couple months since the leak dropped we finally have source code for exinit & systime that match up with the pre-compiled objs included in the leak, letting us make any changes to it that we like (such as removing expiration/timebomb code), as well as compile well-working versions of exinit/systime for AMD64 (since obj files for that platform weren't included). Many thanks to all the anons that worked on it up to this point!

*   **Latest version as of last guide update:** win2003_missing-ex-files_v2.zip (included in prepatched v10a)

###### EncodePointer/DecodePointer kernel32 functions[](#encodepointerdecodepointer-kernel32-functions "Permanent link")

These were added in XP SP2+ as a method for masking pointer addresses (as a security measure?), some apps that "require SP2" or "require Vista" actually only require these two functions. A /g/ user posted changes to allow stub functions to be exported, and a later user updated that so the functionality of the functions is also restored.

*   **Latest version as of last guide update:** [encdecptr.7z](https://mirrorace.org/m/3Lxax)

###### Sizzle dev-environment[](#sizzle-dev-environment "Permanent link")

A /g/ user ported over the OpenNT NTOSBE build-environment to the Win2003 source, as a replacement for the razzle build-environment that was included with the source.

Currently it can mostly build the sources fine, with a few issues noted by users (which are still waiting for a patch?). Post-build currently isn't implemented with this though, but maybe using razzle afterwards to postbuild would allow for setup files to be generated?

Apparently can also handle building under AMD64 build OS's too, along with Win10 support, and can mostly build all the 16-bit objects fine without any NTVDM issues.

*   **Latest version as of last guide update:** [Sizzle-devtest.7z](https://mirrorace.org/m/3Lxc6)

###### DirectUI.lib[](#directuilib "Permanent link")

An anon found a period-appropriate version of Bison.exe & managed to fix up the Bison.skl file to allow building DirectUI.lib, but after some testing it was found that this version sadly doesn't work properly compared to the pre-built XP version, producing a parse error when the code is ran.

The files needed for DirectUI are still included in the prepatched.zip for anyone that wants to try tackling this, it's likely MS had customized the Bison.skl quite a bit, hopefully someone can figure out what exactly was changed...

For now we found that using pre-compiled parse.obj from the directuid.lib included with the source seems to work fine, of course using anything pre-compiled isn't ideal, but at least this way we can still build the rest of DirectUI.lib from the source, rather than needing to rely on an entirely pre-compiled (and version-mismatching) directui.lib from XP.

###### Themes[](#themes "Permanent link")

An anon posted disassembled versions of all the officially released MS themes (Royale/Zune/Embedded etc), these disassembled versions just slot in to the existing source code, and will then be built as part of the normal build, with some small modifications to layout.inx they can also be included into our builds automatically.

*   **Latest version as of last guide update:** [win2003_themeadds_v1.zip](https://mirrorace.org/m/3Lxav) (thanks to anon for reupping it)

A fellow dev has worked on adding some extra functions to kernel32.dll, required by some newer apps, you can find more info about that here: [https://rentry.co/kernel32-extras](https://rentry.co/kernel32-extras)

(sadly the linked `20201202__KERNEL32.DLL.v6.zip` file has since been deleted, but thanks to an anon has been reupped here: [https://mirrorace.org/m/4rIp2](https://mirrorace.org/m/4rIp2))

###### Downlevel DLLs[](#downlevel-dlls "Permanent link")

Like with kernel32.dll, it seems there's"downlevel" DLLs that some apps require to work, a user has made available versions of these for XP here: [https://rentry.co/extras-downlevel](https://rentry.co/extras-downlevel)

(and just like kernel32.dll the file has been deleted since, but thankfully has been reuploaded to [https://mirrorace.org/m/5NDjE](https://mirrorace.org/m/5NDjE)

Changelog[](#changelog "Permanent link")
----------------------------------------

* * *

Each major release will likely break dirty builds, requiring a fresh clean-build, it's recommended to apply new releases onto newly extracted source code if possible.

###### v10[](#v10 "Permanent link")

*   (v10b) Replaced dead links with new mirrors, hopefully these will last longer than the older links... maybe worth making a torrent to keep these alive too, otherwise only option for people is shady chinese download sites...
*   (v10b) Updated winlogon section to mention the ds.zip that was posted late in /wxp/'s lifetime, haven't tested it out myself yet though
*   (v10a) Added `exinit.c` & `systime.c` from anons decompilations, along with original exp.h (to overwrite older modified exp.h from earlier prepatched zip)
*   (v10a) Updated prebuild.cmd to remove the fre `exinit.obj`/`systime.obj` that was included with the leak, so our recreated versions will be used instead (allows building a closer-to-original chk kernel, instead of chk kernel containing fre exinit/systime bits) **Recommended to run prebuild.cmd again if you updated from an earlier prepatched ZIP!**
*   (v10a) Updated missing.cmd to include some chk-only files, and srv_info.chm
*   (v10a) Added pre-generated `inetsrv\iis\svcs\cmp\webdav\davcprox\fhcache_p.c` file, includes defs for both x86/x64, should help with issues switching between x86/x64 builds.
*   (v10a) Added `link.bat`/`link16.bat` wrappers for `link.exe`/`link16.exe`, which randomize TEMP env var and give it 5 retries before failing.
*   (v10a) Updated other .bat wrapper files to use 5 retries instead of 3.
*   (v10a) Updated rc16 call inside `net\tapi\thunk\makefile.inc`, changed `WINNT=1` define to `WINNT` to fix an error under some NTVDM versions.
*   [x64] Replaced MS-DOS Player for some 16-bit tools with recompiled amd64 versions, provided by an anon in the threads (many thanks!), source available inside `_x64\tools\tools16\16_bit_build_tools_v2.zip`
*   Added missing `public\internal\windows\lib\amd64\usp10p.lib` library needed for amd64 build, taken from XPSP1 tree.
*   Added parse.obj to `windows\advcore\duser\directui\engine\parser\obj\amd64` & `objd` folder, extracted from amd64 `directui.lib` file.
*   Added updated `base\ntos\ex\exp.h` & amd64 exinit.obj/systime.obj files based on anon decompilations.
*   Added `msgina_sp1.def` & `userenv_sp1.def` from the winlogon200X pack, these will make amd64 builds of msgina/userenv use the SP1 export ordinal numbers, improving compatibility with SP1 winlogon & possibly other SP1 files.

###### v9[](#v9 "Permanent link")

*   (v9c) [x64] Added 32-bit mapsym.exe & rc.bat MSDOS-Player wrapper to `printscan\faxsrv\print\faxprint\faxdrv\win9x\sdk\binw16` dir, since some anons seemed to get errors from this folder.
*   (v9b) [x64] Replaced `masm.exe/mkpublic.exe` with 32-bit versions (taken from Sizzle), as some anons had issues with MS-DOS Player reporting a bad DOS version, breaking those two tools as they require DOS 2.0+ - thanks to the anon from the thread who helped test!
*   64-bit builder support: added `_x64` subdirectory containing fixed files, & `razzle64.cmd` for building under 64-bit hosts (tested under Win7SP1 x64 & Win10 x64)
*   Updated razzle.cmd to set `NO_PDB_PATHS=1` without needing officialbuild parameter (reasoning in the additions list above)
*   Fix `ixsso` build error by adding `BUILD_ALLOW_MIDL_WARNINGS=1` to makefile, stops MIDL2346 warning from breaking the build (warning seems to be caused by using a non-US locale? can this be fixed via parameters, or a locale-emulator?)
*   Reorder `windows\appcompat\dirs` file to help with single-core builds (from idkwhy's OpenXP git repo)
*   Updated `msitoddf.cpp`, now closes the MSI package handle when finished (prevents stalling postbuild on Win10), and added fix for 64-bit build OS (redirects SysWOW64 to System32)
*   Updated `mshtml.ref` reference file - used to compare mshtml output against known-good one? last updated in 1999(!), without updating it seems to randomly cause build error in certain conditions (only started appearing on win10 though, never had them on win7/XP...)

###### v8[](#v8 "Permanent link")

*   (v8d) Revert `srv_info.chm` changes, as an anon managed to find that inside an XP x64 ISO image, now included inside `2k3-missing-x86fre-v7.7z`
*   (v8d) Updated `cddirs.lst` to allow `valueadd\3rdparty` folder to be included into the CD image.
*   (v8c) Updated razzle.cmd to always set `__BUILDMACHINE__` variable regardless of "offline" parameter, allows build tag built into kernel to match BuildName.txt
*   **Note: the razzle.cmd change above will require clean-building afterwards! If you want to continue 'dirty'-building with your current build make sure you don't extract the updated razzle.cmd from this pack**
*   (v8c) Made oscdimg.cmd use build tag from BuildName.txt as output filename if destination path isn't provided
*   (v8c) Removed directui_XP.lib file, as our built directui.lib seems to work fine
*   (v8b) Added pre-built parse.obj files for DirectUI, taken from win2003 directuid.lib - allows us to build a working version of DirectUI fine!
*   (v8b) Updated `shell\cpls\appwzdui\winnt\sources`, `shell\ext\logondui\sources` & `shell\shell32\winnt\sources` files to suppress LNK4206 warning generated by pre-built parse.obj.
*   (v8b) Deleted all references to `srv_info.chm` (which can't be located anywhere, no RTM ISOs seem to have it), may as well remove it so it can't mess up any more builds.
*   (v8b) Updated `setupw95.cmd` & `drivercab.cmd` to add a delay between async calls, fixes an issue with filename-collisions. (update only needed under newer OS's, older OS's like XP don't seem to have this issue)
*   (v8b) Updated `windows\advcore\dirs` file to allow DUser/DirectUI to get built during main build
*   (v8a) Disabled incbbt.cmd from pbuild.dat, as the incbbt.cmd file is missing
*   Updated 16-bit build tools inside com\ole32\olethunk\ole16\tools\, and updated olethunk code so it'll build fine with them.
*   Removed pre-built 16-bit code
*   Updated prebuild.cmd to handle the updated 16-bit tools instead of using pre-built stuff (will only use updated files if OS requires them)
*   Disabled fixprn.pl, ntbackuponpersonal.cmd, gpmc.cmd and msi.cmd calls from pbuild.dat, as we're missing required build-files for those (you can grab the results of those scripts from RTM ISO, or from the `2k3-missing-x86fre.7z` pack)
*   Thanks to anon who made `2k3-missing-x86fre`, we should now be able to build without any postbuild errors!

###### v7[](#v7 "Permanent link")

*   (7e) removed DUser/DirectUI building instructions, readded DUser.dll to missing.cmd - sadly our built one doesn't work properly and breaks `pro` sku's login screen. Likely because of Bison.skl being customized by MS, which was missing from the source code.
*   (7d) removed DUser.dll from missing.cmd
*   (7c) changed DUser placefil.txt so DUser.dll gets placed in binaries.x86fre root
*   (7c) updated DUser bldenv.cmd so that 'bldenv release' will also reset optimization flags, now release builds will get proper optimization
*   includes Win7 fixes from guideanon's v4 toolkit, hopefully reduces NTVDM errors for people
*   adds files & instructions for building DUser.dll/DirectUI.lib, can now build a proper server2003 version of it instead of needing the WinXP one
*   adds pre-built GdiPlus 1.0.100.0, since we can't build GdiPlus code atm (should allow building a working `asms01.cab`/`hivesxs.inf`)
*   removed `asms01.cab`/`hivesxs.inf` from missing.cmd
*   moved prebuild to `tools\prebuild.cmd`, no longer deletes itself afterwards in case it needs to be ran again
*   re-ordered some of the build preparation steps, added details about building `DirectUI.lib` before the main build

File Hashes[](#file-hashes "Permanent link")
--------------------------------------------

SHA-256 hashes of the files used by this guide:

*   **nt5src.7z** (search for "nt5src notrepacked" on your favorite search engine): 2BB3609FA4C2B2641F43AEF751A84DB5820B64748B7D2D0891D1CB1E55268CE9
*   [win2003_prepatched_v10a.zip](https://mirrorace.org/m/3Lxas): CF2079B49A9119D308D61F46DF21EC064BFF700F4518EC35DE02974E5C7784DA
*   [win2003_x86-missing-binaries_v2.7z](https://mirrorace.org/m/4rv80): D141ECE0EB7710B655903EE301DEC197B1A114618A567100677F0C4D93998C81
*   [processorXP_win2003_update.7z](https://mirrorace.org/m/57szw): 21EE0C471A4CE603DC36ED90C47FC207CA25539460DE212D72E8304E27EBB5A3
*   [win2003_pidgenXP_cracked.zip](https://mirrorace.org/m/3Lxaw): E0D7A6C2B2A17034589849AB6C64D45F6AA4B7873874A894434F3F00F528CE57
*   [ds.zip_decompiled_XPSP1_winlogon.zip](https://mirrorace.org/m/3Lxat): 01D291FCDD6F9D8AFA52872B08398796C0C6C988743CF4447A557A52600BA5B4
*   [Winlogon200X_v3c.zip](https://mirrorace.org/m/5Nq31): 1D9EC8E35665FAB5747CFB0DF14D1ABF4E79080FA2A18C5A6DD7BF8018FC4231
*   [encdecptr.7z](https://mirrorace.org/m/3Lxax): 2A9DFB9896C3B936DFF7E07F9D5C274187D01920B2245E326DE4B96EC188FA1F
*   [Sizzle-devtest.7z](https://mirrorace.org/m/3Lxc6): B165A1CDD1AC6C4C4C46A991387902C7547E1FAEA81E82E5C9901E6C7831D90E
*   [win2003_themeadds_v1.zip](https://mirrorace.org/m/3Lxav): 94AC075197790A5E2B28A51CF7DAD5B89864746AC14408EC4F14B50D23901EF3
*   [20201202__KERNEL32.DLL.v6.zip](https://mirrorace.org/m/4rIp2): C03F13DB8E814B673AA6AE3F7AE2D18B794CD5F33BE912D4BBC6B58F4EE9637D
*   [20201202_downlevel.zip](https://mirrorace.org/m/5NDjE): B7CB9F5023D1AF39B7698D826D050917D4EA566C0EB02159EC3A78AED9A43587
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [github.com](https://github.com/nathanchance/android-kernel-clang)

> Information on compiling Android kernels with Clang - nathanchance/android-kernel-clang: Information ......

[](#compiling-an-android-kernel-with-clang)Compiling an Android kernel with Clang
=================================================================================

[](#background)Background
-------------------------

Google compiles the Pixel 2 kernel with Clang. They shipped the device on Android 8.0 with a kernel compiled with Clang 4.0 ([build.config commit](https://android.googlesource.com/kernel/msm/+/1282b122796d12f42e650216b40172eae4dc4162) and [prebuilt kernel commit](https://android.googlesource.com/device/google/wahoo-kernel/+/8c65a7e83f8bc602a05f077d221d4648db189ef8)) and upgraded to Android 8.1 with a kernel compiled with Clang 5.0 ([build.config commit](https://android.googlesource.com/kernel/msm/+/1eaefe4575b5c39dacb724344d427e34d12c15df) and [prebuilt kernel commit](https://android.googlesource.com/device/google/wahoo-kernel/+/e03cfae0fa716983ae7af64bf8f1c50003637ffb)).

Google uses Clang's link-time optimization and control flow integrity in the Pixel 3 kernel, hardening it against return oriented programming attacks ([LTO commit](https://android.googlesource.com/kernel/msm/+/f641ef709bd4894d9143c9d47af2dc46d3e5ecf4), [CFI commit](https://android.googlesource.com/kernel/msm/+/4ca69fba291799969e4330178379e2ce97ba84dc)).

Google started compiling all Chromebook 4.4 kernels with Clang in R67 ([commit](https://chromium-review.googlesource.com/809774), [LKML](https://lore.kernel.org/lkml/20180403180658.GE87376@google.com/)) and going forward, Clang is the default compiler for all future versions of the kernel ([commit](https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+/9ded75331ed0b7a6f00006d4ffd96ac5210d0976)).

Further information including videos of talks on the motive behind compiling with Clang can be found in [the ClangBuiltLinux wiki](https://github.com/ClangBuiltLinux/linux/wiki/Talks,-Presentations,-and-Communications).

TL;DR: Helps find bugs, easier for Google since all of AOSP is compiled with Clang, compiler specific features such as link-time optimization, and better static analysis for higher code quality.

[](#requirements)Requirements
-----------------------------

*   A compatible kernel (4.4 or 4.9 LTS work best)
*   arm64 or x86_64
*   Patience

msm-4.14 and newer uses clang by default so there is no need for a patch stack.

[](#how-to-compile-the-kernel-with-clang-standalone)How to compile the kernel with Clang (standalone)
-----------------------------------------------------------------------------------------------------

NOTE: I am not going to write this for beginnings. I assume if you are smart enough to pick some commits, you are smart enough to know how to run `git clone` and know the paths of your system.

1.  [Add the Clang commits to your kernel source](#getting-the-clang-patchset)
2.  [Download/build a compatible Clang toolchain](#how-to-get-a-clang-toolchain)
3.  [Download/build a copy of binutils](#how-to-get-binutils)
4.  Compile the kernel (for arm64, x86_64 is similar - example using AOSP's toolchains):

```
make O=out ARCH=arm64 <defconfig>

PATH="<path to clang folder>/bin:<path to 64-bit gcc folder>/bin:<path to 32-bit gcc folder>/bin:${PATH}" \
make -j$(nproc --all) O=out \
                      ARCH=arm64 \
                      CC=clang \
                      CLANG_TRIPLE=aarch64-linux-gnu- \
                      CROSS_COMPILE=aarch64-linux-android- \
                      CROSS_COMPILE_ARM32=arm-linux-androideabi-
```

After compiling, you can verify the toolchain used by opening `out/include/generated/compile.h` and looking at the `LINUX_COMPILER` option.

A couple of notes:

1.  `CLANG_TRIPLE` is only needed when using AOSP's version of Clang.
2.  `export CC=clang` does not work. You need to pass `CC=clang` to `make` like above.
3.  `CROSS_COMPILE_ARM32` is needed in recent kernels to support the new compat vDSO.

[](#how-to-compile-the-kernel-with-clang-inline-with-a-custom-rom)How to compile the kernel with Clang (inline with a custom ROM)
---------------------------------------------------------------------------------------------------------------------------------

1.  [Add the Clang commits to your kernel source](#getting-the-clang-patchset)
2.  Make sure your ROM has [this commit](https://github.com/LineageOS/android_vendor_lineage/commit/da32895b61ef2b3e8899f011110f8eab11da5470)
3.  Add the following to your `BoardConfig.mk` file in your device tree: `TARGET_KERNEL_CLANG_COMPILE := true`

To test and verify everything is working:

1.  Build a kernel image: `m kernel` or `m bootimage`
2.  Open the `out/target/product/*/obj/KERNEL_OBJ/include/generated/compile.h` file and look at the `LINUX_COMPILER` option.

[](#getting-the-clang-patchset)Getting the Clang patchset
---------------------------------------------------------

The core Clang patchset comes from mainline. It has been backported to three places:

*   Linux [`4.4.165`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/log/?h=v4.4.165) and [`4.9.139`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/log/?h=v4.9.139)
*   kernel/common: [4.4](https://android.googlesource.com/kernel/common/+log/f0907aa15ed9f9c7541bb244ed3f52c376ced19c) | [4.9](https://android.googlesource.com/kernel/common/+log/5d15d2e00da4bcb0bcc5e6d27dc18fe1646214f1)
*   Chromium: [4.4](https://chromium.googlesource.com/chromiumos/third_party/kernel/+log/sandbox/mka/llvm/v4.4) | [4.9](https://chromium.googlesource.com/chromiumos/third_party/kernel/+log/sandbox/mka/llvm/v4.9)

If your kernel has 4.4.165 or 4.9.139 and newer, you automatically have the patchset and can start building with Clang!

[](#branch-information)Branch information
-----------------------------------------

The branches in this repository will be dedicated to adding this patchset when it does not exist and enhancing it by fixing all of the warnings from Clang (from mainline, the Pixel 2 and 3, msm-4.9/msm-4.14, and my own knowledge).

*   [msm-3.18-oreo](https://github.com/nathanchance/android-kernel-clang/tree/msm-3.18-oreo) - based on [kernel.lnx.3.18.r33-rel](https://source.codeaurora.org/quic/la/kernel/msm-3.18/log?h=kernel.lnx.3.18.r33-rel). Uses `msm-perf_defconfig`.
    
*   [msm-3.18-pie](https://github.com/nathanchance/android-kernel-clang/tree/msm-3.18-pie) - based on [kernel.lnx.3.18.r34-rel](https://source.codeaurora.org/quic/la/kernel/msm-3.18/log?h=kernel.lnx.3.18.r34-rel). Uses `msm-perf_defconfig`.
    
*   [msm-3.18-android-10](https://github.com/nathanchance/android-kernel-clang/tree/msm-3.18-android-10) - based on [kernel.lnx.3.18.r41-rel](https://source.codeaurora.org/quic/la/kernel/msm-3.18/log/?h=kernel.lnx.3.18.r41-rel). All relevant configs should build with `-Werror`.
    
*   [msm-4.4-oreo](https://github.com/nathanchance/android-kernel-clang/tree/msm-4.4-oreo) - based on [kernel.lnx.4.4.r27-rel](https://source.codeaurora.org/quic/la/kernel/msm-4.4/log?h=kernel.lnx.4.4.r27-rel). Uses `msmcortex-perf_defconfig`.
    
*   [msm-4.4-pie](https://github.com/nathanchance/android-kernel-clang/tree/msm-4.4-pie) - based on [kernel.lnx.4.4.r35-rel](https://source.codeaurora.org/quic/la/kernel/msm-4.4/log?h=kernel.lnx.4.4.r35-rel). Uses `msmcortex-perf_defconfig`.
    
*   [msm-4.4-android-10](https://github.com/nathanchance/android-kernel-clang/tree/msm-4.4-android-10) - based on [kernel.lnx.4.4.r38-rel](https://source.codeaurora.org/quic/la/kernel/msm-4.4/log/?h=kernel.lnx.4.4.r38-rel). All relevant configs should build with `-Werror`.
    
*   [msm-4.9-oreo](https://github.com/nathanchance/android-kernel-clang/tree/msm-4.9-oreo) - based on the latest Oreo branch for the Snapdragon 845 [kernel.lnx.4.9.r7-rel](https://source.codeaurora.org/quic/la/kernel/msm-4.9/log?h=kernel.lnx.4.9.r7-rel). Uses `sdm845-perf_defconfig`.
    
*   [msm-4.9-pie](https://github.com/nathanchance/android-kernel-clang/tree/msm-4.9-pie) - based on the latest Pie branch for the Snapdragon 845 [kernel.lnx.4.9.r11-rel](https://source.codeaurora.org/quic/la/kernel/msm-4.9/log?h=kernel.lnx.4.9.r11-rel). Uses `sdm845-perf_defconfig`.
    
*   [msm-4.9-android-10](https://github.com/nathanchance/android-kernel-clang/tree/msm-4.9-android-10) - based on [kernel.lnx.4.9.r25-rel](https://source.codeaurora.org/quic/la/kernel/msm-4.9/log/?h=kernel.lnx.4.9.r25-rel). All relevant configs should build with `-Werror`.
    

The general structure of these commits is as follows:

1.  The core compilation support (if needed)
2.  Fixing Qualcomm specific drivers to compile with Clang
3.  Fixing warnings that come from code in mainline
4.  Fixing warnings that come from code outside of mainline

You should pick the commits that I have committed (nathanchance).

Additionally, there are fixes for:

*   qcacld-2.0 available in [my Pixel XL kernel](https://github.com/nathanchance/marlin/commits/oreo-m4/drivers/staging/qcacld-2.0).
*   qcacld-3.0 available in [my OnePlus 5 kernel](https://github.com/nathanchance/op5/commits/8.1.0-unified/drivers/staging/qcacld-3.0).

Every time there is a branch update upstream, the branch will be rebased, there is no stable history here! Ideally, I will not need to add any commits but I will do my best to keep everything in the same order.

**NOTE:** 3.18 Clang is not supported officially by an OEM. I've merely added it here as I decided to support it with [my Pixel XL kernel](https://github.com/nathanchance/marlin).

[](#how-to-get-a-clang-toolchain)How to get a Clang toolchain
-------------------------------------------------------------

You can either use a prebuilt version of Clang (like from AOSP) or build a version yourself from the upstream sources.

For prebuilts, I recommend [AOSP's Clang](https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/), which is used for both the kernel and the platform so it is highly tested. You can `git clone` that repo to always have access to the latest version available, which is what I recommend. That is currently [Clang 9.0.8](https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+/ee5ad7f5229892ff06b476e5b5a11ca1f39bf3a9/clang-r365631c/). If you would like to just download the minimum version of Clang supported for the branches in this repo, direct tarball links are provided below:

*   [clang-4053586](https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+archive/android-9.0.0_r1/clang-4053586.tar.gz) for Oreo branches
*   [clang-4691093](https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+archive/android-9.0.0_r1/clang-4691093.tar.gz) for Pie branches
*   [clang-r353983c](https://android.googlesource.com/platform/prebuilts/clang/host/linux-x86/+archive/android-10.0.0_r3/clang-r353983c.tar.gz) for Android 10 branches

If you would like to build the latest version of Clang, you can do so with [`tc-build`](https://github.com/ClangBuiltLinux/tc-build). There are a lot of fixes for the Linux kernel that happen in LLVM/Clang so staying up to date is critical. If you experience any issues with Clang and an Android kernel, please report them to this repo.

[](#how-to-get-binutils)How to get binutils
-------------------------------------------

binutils are used for assembling (and usually linking) the Linux kernel right now. When using AOSP Clang, you should use [AOSP's GCC](https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/) to avoid weird incompatibility issues. If you want source-built binutils, there is a `build-binutils.py` script available in [`tc-build`](https://github.com/ClangBuiltLinux/tc-build).

[](#getting-help)Getting help
-----------------------------

The preferred method for getting help is either opening an issue on this repository or joining the [Linux kernel newbies chat on Telegram](https://t.me/LinuxKernelNewbies).
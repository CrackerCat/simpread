> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.trendmicro.com](https://www.trendmicro.com/en_us/research/17/a/practical-android-debugging-via-kgdb.html)

Kernel debugging gives security researchers a tool to monitor and control a device under analysis. On desktop platforms such as Windows, macOS, and Linux, this is easy to perform. However, it is more difficult to do kernel debugging on Android devices such as the Google Nexus 6P . In this post, I describe a method to perform kernel debugging on the Nexus 6P and the Google Pixel, without the need for any specialized hardware.

Joshua J. Drake and Ryan Smith built a [UART debug cable](https://www.optiv.com/blog/building-a-nexus-4-uart-debug-cable) for this purpose, which works well. However, for some people not as skilled in hardware building (such as software engineers like myself), this can be difficult. Alternately, [kernel debugging via the serial-over-usb channel](http://bootloader.wikidot.com/android:kgdb) is also possible.

The method proposed for this dates back to 2010, which means that some parts of the instructions are now outdated. The method I've found still uses this as the key point, but can be used on modern Android devices. Researchers can use debugging to determine the current state of CPU execution, making analysis go more quickly. So how does this process work?

Android is built on the Linux kernel, which includes a built-in kernel debugger, [KGDB](https://kgdb.wiki.kernel.org/index.php/Main_Page). KGDB relies on a serial port to connect the debugging device and the target device. A typical scenario is shown below:

![](https://www.trendmicro.com/content/dam/trendmicro/global/en/migrated/security-intelligence-migration-spreadsheet/trendlabs-security-intelligence/2017/01/android-debug-1.jpg)

_Figure. 1 KGDB working model_

The target and debugging devices are connected via a serial cable. The user on the debugging machine uses GDB to attach the serial device file (for example, _/dev/ttyS1_) using the command _target remote /dev/ttyS1_. After that, GDB can communicate with KGDB in the target device via the serial cable.

The KGDB core component handles the actual debugging tasks such as setting break points and fetching the data in memory. The KGDB I/O component is a glue that connects the KGDB core component with low-level serial drivers to take care of the transmission of debug information.

However, Android devices generally don’t have hardware serial ports. The first challenge becomes how to find a channel so that KGDB can send the debugging information to an outside device with GDB. Various channels have been used, but the most practical solution is a USB cable.

The Linux kernel's USB driver supports [USB ACM class devices,](https://cscott.net/usb_dev/data/devclass/usbcdc11.pdf) which can emulate a serial device. In short, an Android device can connect to a serial device by USB cable. This is all in code that is already part of the Linux kernel, so we do not need to add any specific lines of code.  Here are the steps to activate this debugging feature:

1.  Build a version of the AOSP (Android Open Source Project) with_aosp_angler-eng_ and the corresponding linux kernel. Please refer [here](https://source.android.com/source/building.html).
2.  Connect the target device with the debugging machine by USB cable.
3.  Use the _flashboot_ command to write the image to the target device with the command _fastboot flashall –w_.
4.  Run the _adb_ command to enable the adb network service: _adb tcpip 6666_
5.  On the debugging machine, run _adb_ to connect the device: _adb connect <device ‘s IP>:6666_
6.  Run the _adb_ shell: _adb shell_
7.  In the _adb_ shell, go to the USB gadget driver control folder _/sys/class/android_usb/android0/_.
8.  In the _adb_ shell, using following commands to enable the USB ACM function: _echo 0 > enable       //close USB connection echo tty > f_acm/acm_transports    //specific transport type echo acm > functions           //enable ACM function on USB gadget driver echo 1 > enable       //enable USB connection_

At this point, the USB ACM function should be enabled. Two checks should be done to verify if it is the case: first, in the _adb_ shell, use the command _ls /dev/ttyGS*._ A device file should exist. Secondly, on the debugging machine, use the command _ls /dev/ttyACM*._ A device file should be here as well. The debugging machine can communicate with the target device using these two device files.

The second challenge is KGDB needs a lower-level communication driver (either the serial driver or USB gadget driver) to provide a polling interface to both get and write a character. Why? Because KGDB's communication channel needs to work in the KGDB's kernel exception handler. In that context, the interrupt is disabled and only one CPU works to run this code. The lower-level driver doesn't depend on interrupts and needs to actively poll for changed registers or memory I/O space. Do _not_ use _sleep_ or _spinlock_ in this context.

The Nexus 6P use a DWC3 controller for its USB connection. This USB driver doesn't provide a polling function directly, so [I added this feature](https://github.com/jacktang310/KernelDebugOnNexus6P) into the DWC3 device driver. The concept behind this code is simple: I have removed the dependencies on interrupts. Instead, I use loops to query the corresponding changed in registers or memory space. To keep things simple, I did the following:

*   I hardcoded the ACM device file name as _/dev/ttyGS0_ in the _kgdboc_init_jack_ function in kgdboc.c.
*   I changed the handle function of _f_acm/acm_transports_ to enable KGDB. This allows KGDB to be turned on with the simple command _echo kgdb > f_acm/acm_transports_

This turns the KGDB working model into the following:

![](https://www.trendmicro.com/content/dam/trendmicro/global/en/migrated/security-intelligence-migration-spreadsheet/trendlabs-security-intelligence/2017/01/android-debug-2.jpg)

_Figure 2. New KGDB working model_

Here are the steps to merge the code above to kernel code. While you can use any kernel code, I have taken mine from [Google's own Git repository](https://android.googlesource.com/kernel/msm.git), specifically the branch _origin/android-msm-angler-3.10-nougat-hwbinder._

1.   Compile the kernel code with following configuration changing on the original configure setting:
    1.  CONFIG_KGDB=y CONFIG_KGDB_SERIAL_CONSOLE=y
    2.  CONFIG_MSM_WATCHDOG_V2 = n If this is enabled, the kernel will take the polling loop as a dead loop and restart the device. This must be disabled.
    3.  CONFIG_FORCE_PAGES=y If this is not enabled, the soft breakpoint isn't set.
2.  Use the _fastboot_ command to flash the kernel and Android image onto the target device.
3.  Start the Android system on the debugging machine. Run the following command to enable the _adb_ network service: _adb tcpip 6666_
4.  On the debugging machine, run the following commands: _adb connect <device’s ip address>:6666 adb shell_
5.  Inside the _adb_ shell , run the following commands: echo 0 > enable echo tty > f_acm/acm_transports echo acm > functions echo 1 > enable echo kgdb > f_acm/acm_transports
6.  On the debugging machine, run _gdb_. The Nexus 6P has an aarch64 kernel, so you need [gdb for aarch64](https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.8): _sudo <path>/aarch64-linux-android-gdb <path>/vmlinux target remote /dev/ttyACM0_
7.  Inside the _adb_ shell, input following command: _echo g > /proc/sysrq-trigger_ Note: The time interval between step 6's last command and step 7 should not be too long. The shorter, the better.
8.  If successful, GDB will output the following:

![](https://www.trendmicro.com/content/dam/trendmicro/global/en/migrated/security-intelligence-migration-spreadsheet/trendlabs-security-intelligence/2017/01/android-debug-3.jpg)

_Figure 3. GDB output_

I have used this method to perform kernel debugging on a Nexus 6P. This method should work for Google Pixel, so long as they also use the DWC3 USB controller.

We hope that sharing this technique may be useful to other Android researchers by giving the research community a new method to better understand the behavior of mobile malware. Debugging gives researchers better reverse engineering capabilities, giving them a clearer understanding of how malware behaves within an Android device.
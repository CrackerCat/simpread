> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.pucarasec.com](https://blog.pucarasec.com/2020/06/23/debugging-the-samsung-android-kernel-part-3/)

![](https://pucarasec.files.wordpress.com/2020/06/dbb9d-1gigzuzrcfbh8hh92eiavng.png)

### How to enable Serial-over-USB debugging for the Samsung Kernel.

In this tutorial, I will be covering how to modify the Samsung DWC3 USB drivers in order to enable polling support, so as to be able to use the ttyGS0 interface on the device and the ttyACM0 interface on the debugging host to finally **debug** the Android Kernel with **KGDB**. It is also necessary to modify and recompile KGDB to work around a bug, which prevents the aarch64 architecture from being debugged.

This whole tutorial would have been much harder without the following previous work:

*   [Enabling KGDB for Android by dankex](http://bootloader.wikidot.com/android:kgdb)
*   [Practical Android Debugging Via KGDB by Jack Tang](https://blog.trendmicro.com/trendlabs-security-intelligence/practical-android-debugging-via-kgdb/)

### Preparing the environment

Since we are going to compile and flash a custom kernel for this, I highly recommend you [go through part 1 & 2 of this series](https://blog.pucarasec.com/2020/06/23/debugging-the-samsung-android-kernel-part-2/), as they explain this process throughly.

Additionally, we’ll need the to run the following commands in order to install the dependencies necessary to compile gdb with aarch64 and python support. Run the following as root:

```
apt-get -y update
apt-get -y upgrade
apt-get -y dist-upgrade
apt-get install python2.7-dev libexpat1 libexpat1-dev

```

### A little background on what we’re dealing with

Anyone who has debugged a Linux Kernel and has dealt with KGDB, knows that unless we have a hypervisor that allows us to debug the Kernel without additional setup, it can get complicated.

However, in the case of an Android device, it can get even more complicated: There are several ways in which KGDB is able to connect to the debugger apart from the hypervisor that was mentioned before, KGDB works usually over serial (KGDBOC), on the same console (KDB) and even over ethernet (KGDBOE) if the hardware supports it.

In the case of a Samsung device, there are serial devices present. However, these are not easily accessible and probably mean that you need to open the device and connect via JTAG or directly on the debugging port of the microprocessor. This is, however, very much out of my league.

But, there is another way: Samsung’s Kernel (among others) implements Serial over USB. Basically whenever one connects a USB cable to the phone, a device is created on the host called “ttyACM0” and another one called “ttyGS0” (The “G” standing for gadget, which is the driver that takes care of this) which is there by default.

This creates a bi-directional channel between the phone and the host computer, which can be used to communicate. So, my first thought was, okay this sounds easy enough. We just point KGDBOC to ttyGS0 and that should be it, right? Well, no. Sadly, the Samsung kernel does not implement “polling mode” which is an operation mode in which interrupts can’t be used, and neither can spinlocks or operations which sleep.

Additionally, even if we can implement polling mode in the USB driver, we need to disable a few features from Samsung:

*   Samsung Real-Time Kernel Protection (RKP): This feature, among other things, protects parts of memory so that they can’t be overwritten, even by the Kernel itself. Unless we disable this, we won’t be able to use software breakpoints, and as far as I’ve tried, there are no hardware breakpoints available for this.
*   Watchdog: The watchdog checks if the Kernel gets stuck somewhere for too long, and reboots the phone. Unless we disable this, we won’t be able to debug for long.

This can all by achieved by using the configuration menu.

### Patching and compiling our custom Android Samsung Kernel to enable KGDB over serial.

I based myself on the work done by Jack Tang on for the Nexus 6P, however, the DWC USB drivers that are used on the Nexus 6P are quite different from the one used on the S7, and had to be modified. This took a while and a lot of compile-flash-reboot-debug cycles, which can take a long time. But it was worth it in the end.

Now, as with the “Compiling a custom Samsung Android Kernel, and living to tell the tale.” guide, go to the toolchain directory and run the following commands:

```
export CROSS_COMPILE=$(pwd)/bin/aarch64-linux-android-
export ARCH=arm64 
export SUBARCH=arm64
export ANDROID_MAJOR_VERSION=o

```

And then, on the samsung kernel folder, run the following commands. Make sure that the configuration is the correct one for your device. In my case, it’s the S7:

```
make clean
make mrproper
make exynos8890-herolte_defconfig

```

Then, we need to modify a few Kernel configurations, in order to enable KGDBOC and bypass a few annoying features. For this, run “make menuconfig” and follow this instructions:

**Enable KGDBOC:** Kernel Hacking → KGDB: Kernel debugger

![](https://pucarasec.files.wordpress.com/2020/06/f36b6-1o6ehqc8qavx08euogo5n6g.png)Enable KGDBOC

**Disable the Watchdog:** Device Drivers → Watchdog Timer Support

![](https://pucarasec.files.wordpress.com/2020/06/81a59-1v7yts_qm-cnxdmy3z3omlg.png)Disable the Watchdog

**Disable TIMA RKP:** Kernel Features → ARM errata workarounds via the alternatives framework → Enable TIMA(Trustzone based Integrity Measurement Archtecture)

![](https://pucarasec.files.wordpress.com/2020/06/61a56-1yfykiigmu3ufylz-7a4zpw.png)Disable TIMA RKP

After this, apply [this](https://github.com/alex91ar/samsung-debug/blob/master/patch.diff) patch (on the samsung kernel folder):

```
git apply patch.diff

```

Once everything’s ready, run:

```
make -j(Number of cores)

```

And then flash it as shown on the Part 2 of these tutorials.

I’ve forked a copy from the gdb-multiarch android toolchain, and patched it to resolve a compiling issue. Whenever you compile it, you’d get the following error:

```
amd64-linux-nat.c:248:1: error: conflicting types for ‘ps_get_thread_area’
 ps_get_thread_area (const struct ps_prochandle *ph,
 ^~~~~~~~~~~~~~~~~~
In file included from gdb_proc_service.h:30:0,
                 from amd64-linux-nat.c:30:
/usr/include/proc_service.h:72:17: note: previous declaration of ‘ps_get_thread_area’ was here
 extern ps_err_e ps_get_thread_area (struct ps_prochandle *,
                 ^~~~~~~~~~~~~~~~~~
Makefile:1141: recipe for target 'amd64-linux-nat.o' failed
make[2]: *** [amd64-linux-nat.o] Error 1
make[2]: Leaving directory '/home/apok/gdb/gdb-7.11/gdb'
Makefile:9156: recipe for target 'all-gdb' failed
make[1]: *** [all-gdb] Error 2
make[1]: Leaving directory '/home/apok/gdb/gdb-7.11'
Makefile:846: recipe for target 'all' failed
make: *** [all] Error 2

```

![](https://pucarasec.files.wordpress.com/2020/06/9027a-1rodiramvgitxrjh7epu_jw.png)GDB Compilation error.

Also, there’s a bug in GDB which prevents us from debugging the aarch64 architecture. Some packets are marked as “too big”:

```
Sending packet: $g#67...Ack
Packet received: 30b40a02c0ffffff...[Snipped]
Traceback (most recent call last):
  File "<string>", line 20, in <module>
gdb.error: Remote 'g' packet reply is too long: 30b40a02c0ffffff...[Snipped]

```

![](https://pucarasec.files.wordpress.com/2020/06/52dff-16w3ei9qwmklvenr3lphiqa.png)Runtime error on GDB.

My forked version corrects those errors, and allows us to compile it and use it without problems. To acquire it and install it:

```
git clone https://github.com/alex91ar/gdb-multiarch.git

```

Once you’ve installed it, go to the gdb-7.11 folder in the folder of the patched GDB, and run the following:

```
./configure --enable-targets=all --with-expat --with-python
make
sudo make install
sudo ln -s /usr/bin/gdb $(which gdb)

```

This will create a GDB with python enabled, and compatible with any architecture. It will also create a symlink, which for some reason is not created properly.

![](https://pucarasec.files.wordpress.com/2020/06/7e0b7-1xhfwfnmafocmdljptvwpqq.png)Compiled multiarch version of GDB

### Debug the Samsung Android Kernel

And finally, we’re here. We have a Kernel which is ready to be debugged. Every annoying feature is disabled and KGDBOC is happy with its new polling mode. Now, all we have to do is use our debugger and enable KGDBOC.

For this, I created a script which can be loaded by GDB, which will automatically disable the protections on kallsyms, acquire the offset for the Kernel image (to bypass KASLR), set every parameter to work with a serial connection, and perform the connection by listening on ttyACM0 and sending “g” to sysrq-trigger on the phone, to trigger a software breakpoint on the Kernel.

```
python
import subprocess
import time
subprocess.check_output(['adb', 'shell', 'su -c "echo 0 > /proc/sys/kernel/kptr_restrict"'])
subprocess.check_output(['adb', 'shell', 'su -c "echo ttyGS0 > /sys/module/kgdboc/parameters/kgdboc"'])
output = subprocess.check_output(['adb', 'shell', 'cat /proc/kallsyms | grep _stext -m 1 | cut -d " " -f1'])
gdb.execute("set serial baud 115200")
gdb.execute("set architecture aarch64")
gdb.execute("add-symbol-file vmlinux 0x" + output)
while True:
 try:
  gdb.write("Trying to connect...\n")
  output = subprocess.check_output(['timeout', '1', 'cat','/dev/ttyACM0'])
  break
 except subprocess.CalledProcessError as err:
  if err.returncode == 124:
   break
  time.sleep(1)
subprocess.Popen(['adb', 'shell', 'su -c "sleep 1 && echo g > /proc/sysrq-trigger"'])
gdb.execute("target remote /dev/ttyACM0")
end

```

Save this file on the samsung kernel folder named “samdeb” and once your phone is ready and connected, you may start your debugging session by running:

```
gdb -x samdeb

```

If everything went correctly, you should see the following:

![](https://pucarasec.files.wordpress.com/2020/06/f05fc-17txxsvumq7tater0k8o3qq.png)Hooray!

You may also set **breakpoints**:

![](https://pucarasec.files.wordpress.com/2020/06/99b23-1rwudemb6qbccnafwv7-jsg.png)Breaking on sys_read()

And **read memory**:

![](https://pucarasec.files.wordpress.com/2020/06/3815f-16v1uxghxo6cjdtnofmyhww.png)Memory contents

### Known issues

*   It is fairly slow, nothing that can be done about this.
*   If you run “detach” the phone will hang. Instead, use quit and kill the debugging session.
*   After a successful detaching, you may not be able to use the usb interface until you reset the phone.
*   Single stepping does not work (Working on this).

### Last thoughts

This was a very fun project which later on helped me in exploit development, however there is still a lot of room for improvement. If you have any suggestions, please send me an **issue or a pull request** via my [GitHub](https://github.com/alex91ar) and I promise to make my best to take care of it.
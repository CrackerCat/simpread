> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bootloader.wikidot.com](http://bootloader.wikidot.com/android:kgdb)

KGDB may be one of the best tools for kernel debugging, besides the famous printk. It allows a developer to connect to the target device from GDB on a host PC. Basic information about KGDB can be found here: [http://kgdb.linsyssoft.com](http://kgdb.linsyssoft.com/). KGDB has two parts: the kgdb core (kernel/kgdb.c) and a few connection interfaces — currently supporting a serial tty device (specified by kgdboc=<serial tty device>) and the ethernet connection (kgdboe=<ip:port>). It supports many architectures like x86 and ARM, but there are still some challenges to use it effectively on Android.

The main challenge is that Android phones usually don't have an exposed serial port or an ethernet port, which KGDB usually requires. To use KGDB on a device without a serial or an ethernet port, one may need to get the USB driver to work with KGDB, or to get KGDB to work on an emulator (like Android goldfish). So far, it doesn't seem these options are working yet.

So I will first try to enable KGDB over the USB connection available on the Android phone. To use KGDB over a tty device, the tty device would have to explicitly support character I/O under an atomic context. For example, the serial port driver would need to be able to send/receive characters by polling the I/O ports (or memory-mapped addresses), in addition to its standard equivalent operations which may use interrupts and may contain complicated operations on upper tty layers. For KGDB, all it can do is quickly get a character over the serial line, restore all hardware states, and return the character to KGDB (the same for sending a character). This can be a bit more challenging for the USB device, since a USB device may have multiple interfaces and endpoints with data going on. When KGDB stops the kernel, and the KGDB driver is doing I/O with one specific interface (two endpoints), some hardware interrupts for other interfaces may get missed by their normal handlers. Fortunately, it turned out that this sometimes is not too big a problem if the interrupts are level-triggered, and can stay there for a while while we debug with KGDB.

Here, I will first quickly explain the theory of operation for the KGDB USB driver. To use KGDB with "kgdboc=<tty-device>", the tty driver needs to support three ops: poll_init, poll_get_char, and poll_put_char to communicate with a remote GDB client. They are special versions of character I/O in addition to what every tty driver supports, such as put_char, and some char push mechanism based on interrupts.

```
static const struct tty_operations tty_ops = {
    ... normal ops like open/close/put_char ...
    .poll_init = gs_poll_init, 
    .poll_get_char = gs_poll_get_char,
    .poll_put_char = gs_poll_put_char,
}

```

It's not too hard to implement these functions for the USB controller on an MSM chip. They can be based on the USB drivers in Android's MSM kernel. I will make the code available after some more testing.

After implementing the I/O functions for KGDB, there are still a couple of things to do. One of them is that KGDB is initialized very early during boot-up, and it might not be able to access the USB driver when it is first invoked by the kernel. I added a small delay in it to retry the USB driver after a few seconds. This turns out to be working quite well. Of course, another option is to initialize the USB driver earlier if its dependencies can be resolved too.

On the host side, one need to fire up a USB serial driver in order to see the serial device over the USB. This can be a command like "modprobe usbserial vendor=<vendor id> product=<product id>". The good thing is that one probably won't need to worry about the baud rate at all for USB serial drivers — they are either not fully supported by the drivers, or no driver really cares about them. The USB hardware can work well without knowing what baud rate has been specified on either end of the channel.

One remaining issue might be that the usbserial driver may conflict with the ADB driver, which makes it hard to use ADB and KGDB at the same time. The problem is after the "modprobe usbserial…" command, all USB devices that look like a serial port are opened, which include the one for KGDB and the one normally for ADB, so when ADB is run, it gets access denied. Of course, to fix this, one can modify the usbserial driver to avoid the ADB interface. Hopefully all these will work out. I will update this post and post code when things work a bit beyond the proof-of-concept stage.

KGDB for Android download
-------------------------

The source is available at [http://github.com/dankex/kgdb-android](http://github.com/dankex/kgdb-android). This is the first version. Please post comments on github.

Compile KGDB for Android
------------------------

After merging the KGDB USB code into the Android Kernel, you would need to turn on the followings flags:

### Configure the kernel

CONFIG_KGDB (for KGDB)  
CONFIG_HAVE_ARCH_KGDB (for KGDB)  
CONFIG_CONSOLE_POLL (for Android USB support)  
CONFIG_MAGIC_SYSRQ (use sysrq to invoke KGDB)

### Configure Android USB composite driver to include "acm"

Enable CONFIG_USB_ANDROID_ACM  
Modify the board file (e.g., arch/arm/mach-msm/board-mahimahi.c) to include the "acm" usb function. Thanks to Mike for the suggestion. To use the f_acm function, one may create a small acm+adb composite for kgdb debugging.

```
#ifdef CONFIG_USB_ANDROID_ACM
static char *usb_functions_adb_acm[] = {
    "adb",
    "acm",
};
#endif

```

Use these functions in the usb composite:

```
static struct android_usb_platform_data android_usb_pdata = {
    .vendor_id    = 0x18d1,
    .product_id    = 0x4e11,
    .version    = 0x0100,
    .product_name        = "Nexus One",
    .manufacturer_name    = "Google, Inc.",
    .num_products = ARRAY_SIZE(usb_products),
    .products = usb_products,
    .num_functions = ARRAY_SIZE(usb_functions_adb_acm),  /* adb + acm */
    .functions = usb_functions_adb_acm,
};

```

This composite will include adb and a USB serial port ttyGS0 (through f_acm.c), which we can use for the kgdb-host connection.

### Configure kernel command line

Specify ttyGS0 as the kgdboc device. Add the following into the kernel command line (possibly in BoardConfig.mk)

```
kgdboc=ttyGS0 kgdbretry=4

```

The second option "kgdbretry=4" is a new parameter added to kgdboc.c. It means that if kgdb cannot find the device "ttyGS0" in early boot, it will retry once after the specified number of seconds. This is a work-around if the USB device is not immediately initialized during system boot.

### Connect to kgdb from a host PC

When the device is in the KGDB mode, you can use gdb to connect to it. The device enters the KGDB mode in one of the following conditions:  
1) A break point has been hit  
2) Sysrq-g has been triggered  
3) A system exception is caught  
4) The option "kgdbwait" halts the kernel during boot-up (not supported yet)

Before a break point is set, to invoke KGDB for the first time, what we can do is through ADB:

```
$ adb shell sh -c "echo -n g>/proc/sysrq-trigger"

```

Then, we can connect to the phone with gdb for ARM:

```
$ arm-eabi-gdb ./vmlinux
(gdb) target remote /dev/ttyACM0        // this is the cdc-acm serial port on host PC

```

The tool "arm-eabi-gdb" can be found in the "prebuilt" directory in an Android build. "vmlinux" is the Android kernel containing symbols. You can also use "set architecture" to choose a specific ARM architecture before connecting to the device.

If everything turns out to be working right, you would see the following prompt in GDB:

```
GNU gdb 6.6
Copyright (C) 2006 Free Software Foundation, Inc.
...
This GDB was configured as "--host=i686-pc-linux-gnu --target=arm-elf-linux"...
(gdb) target remote /dev/ttyACM0
Remote debugging using /dev/ttyACM0
warning: shared library handler failed to enable breakpoint
0x800a1380 in kgdb_breakpoint () at /.../kernel/arch/arm/include/asm/atomic.h:37
37        __asm__ __volatile__("@ atomic_set\n"
(gdb)

```

Troubleshooting
---------------

After loading the kernel including the "acm" device, one may want to check if the "cdc-acm" gadget device is recognized on the host. This can be done with "lsusb" and "cat /proc/bus/usb/devices". The command "lsusb" lists all usb devices. It should include the usb composite with specific vendor/product ids specified in the board file above. For example,

```
$ lsusb
Bus 001 Device 006: ID 18d1:4e99 // The Android USB composite device
Bus 001 Device 004: ID 04f2:b008 Chicony Electronics Co., Ltd 
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
...

```

The usb device file can show the details in the composite device, for example:

```
$ cat /proc/bus/usb/devices
T:  Bus=01 Lev=01 Prnt=01 Port=01 Cnt=01 Dev#=  6 Spd=480 MxCh= 0
D:  Ver= 2.00 Cls=00(>ifc ) Sub=00 Prot=00 MxPS=64 #Cfgs=  1
P:  Vendor=18d1 ProdID=4e99 Rev= 2.26
S:  Manufacturer=Google, Inc.
S:  Product=Nexus One
S:  SerialNumber=0123456789ABCDEF
C:* #Ifs= 3 Cfg#= 1 Atr=e0 MxPwr=500mA
I:* If#= 0 Alt= 0 #EPs= 2 Cls=ff(vend.) Sub=42 Prot=01 Driver=usbfs
E:  Ad=81(I) Atr=02(Bulk) MxPS= 512 Ivl=0ms
E:  Ad=01(O) Atr=02(Bulk) MxPS= 512 Ivl=0ms
I:* If#= 1 Alt= 0 #EPs= 1 Cls=02(comm.) Sub=02 Prot=01 Driver=cdc_acm        // acm is supported
E:  Ad=83(I) Atr=03(Int.) MxPS=  10 Ivl=32ms
I:* If#= 2 Alt= 0 #EPs= 2 Cls=0a(data ) Sub=00 Prot=00 Driver=cdc_acm
E:  Ad=82(I) Atr=02(Bulk) MxPS= 512 Ivl=0ms
E:  Ad=02(O) Atr=02(Bulk) MxPS= 512 Ivl=0ms

```

If the above steps show that the composite driver supports the cdc-acm serial port, then you can probably find the port as /dev/ttyACM0 or ttyACM1, which is ready to use for GDB.

Some examples of using KGDB
---------------------------

Here are a couple of examples of using KGDB to do something in kernel debugging.

*   [The mkdir syscall](http://bootloader.wikidot.com/android:kgdb:mkdir)

Newer articles about Android hacking with KGDB
----------------------------------------------

*   [KGDB on Android: Debugging the kernel like a boss (Hacking Nexus 6 with KGDB!)](http://www.contextis.com/resources/blog/kgdb-android-debugging-kernel-boss/)
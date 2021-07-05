> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-268345.htm#msg_header_h2_7)

> [原创]android 11 手动 root 笔记

android 11 手动 root 笔记
=====================

0. 前言
-----

目标是在已解锁的 android11 手机上，通过修改原有系统的方式，以尽量少的痕迹获取 (adb)root 权限并全局装入 frida-gadget。  
关于为什么不用 magisk 或其他框架，因为感觉 magisk 的解决方案对我来说功能过于冗余，并且重视安全的应用都会试图通过各种姿势检测 magisk 或其他框架以及 root _(在某些群天天看到群友问某某应用检测 root 怎么办)_ 一想到有可能会遇到有关 magisk 的兼容性问题，还要去定位解决，就头大 _(也是还没用过 magisk)_

1. root 思路
----------

之前了解过 adb root 的原理：adbd 服务最初以 root 用户的权限运行，读取判断一些系统配置后会自行降权，切换至 shell 用户。直接修改系统配置的值虽然操作不难，但是可以被检测到，在某些系统上还会导致 log 增多影响性能，所以想法是 patch 系统上的 adbd 文件使其不会执行降权。

2. 获取原系统镜像文件
------------

手机的系统发布商向公众提供了 OTA 包下载途径，特点是 zip 文件内含 payload.bin。  
可以使用 https://github.com/vm03/payload_dumper 解包其中的镜像文件。  
使用前需要运行如下命令以安装其依赖：  
`pip install protobuf bsdiff4`  
在`git clone https://github.com/vm03/payload_dumper`之后解压 OTA 包将 payload.bin 移至 payload_dumper 文件夹下，运行`python payload_dumper.py payload.bin`，输出在 output 文件夹里。

3. 获取 recovery 模式下的 root 权限
---------------------------

recovery 模式的系统文件在 boot.img 中，可以使用 https://github.com/cfig/Android_boot_image_editor 解包。  
使用前需要安装较高版本的 java，我安装的是 jdk16，安装后需要手动将 jdk 中的 bin 路径添加到系统变量中。  
此工具使用 gradle 进行项目部署及运行，需要联网从 maven 仓库下载一些包，如感觉速度慢则需要高速国际网络或者换源。这里讲一下换源的方法：在`git clone https://github.com/cfig/Android_boot_image_editor`之后编辑其中的 build.gradle.kts 找到

```
buildscript {
    repositories {
        mavenCentral()
    }
    ...
}

```

在 repositories 中添加三行

```
buildscript {
    repositories {
        maven {
          setUrl("http://maven.aliyun.com/nexus/content/groups/public/")
        }
        mavenCentral()
    }
    ...
}

```

将解包出的 boot.img 与 vbmeta.img 复制进 Android_boot_image_editor 文件夹里，运行`gradlew unpack`输出的文件系统在`build/unzip_boot/root`下。  
编辑`build/unzip_boot/vbmeta.avb.json`将 header.flags 改为 2 来禁用全局 verity，否则手机不会接受我们修改后的 boot.img。  
编辑`build/unzip_boot/root/prop.default`，将 ro.secure 改为 0，ro.adb.secure 改为 0，ro.debuggable 改为 1，该修改仅影响 recovery 模式下的系统属性。

 

运行`gradlew pack`，usb 连接手机依次运行

```
adb reboot bootloader
fastboot flash --disable-verity --disable-verification vbmeta vbmeta.img.signed
fastboot flash boot boot.img.signed
fastboot reboot recovery

```

之后在 recovery 模式下运行`adb shell`，提示符已经变为 #表示为 root 权限。

4. 在 recovery 模式下修改 system 文件的方式
--------------------------------

该系统使用虚拟 A/B 的方式升级，system 文件系统在`/dev/block/byname/super`中偏移为 1024*1024 的位置处。为了有效率地在 windows 下直接编辑 android 系统中未挂载的 ext4 文件系统而不用重新刷写，找到了 https://github.com/cubinator/ext4 并添加了覆写文件的方法，fork 并修改后的项目见 https://github.com/tacesrever/ext4 ，并编写了一个简陋的 socket 程序用于转发文件读写操作。  
服务器端：

```
#include #include #include #include #include #include #include #include #define BLOCK_SIZE 4096
#define LISTEN_PORT 4096
 
typedef enum {
    READ,
    SEEK,
    WRITE,
    TELL,
    OPEN,
    CLOSE
} foperate;
 
typedef struct {
    foperate op;
    u_int64_t arg;
} fcommand;
 
int recv_all(int sock_fd, u_char* buffer, uint64_t size) {
    uint64_t recved_size = 0, recv_size;
    while (recved_size < size) {
        recv_size = recv(sock_fd, buffer + recved_size, size, MSG_WAITALL);
        if(recv_size <= 0) {
            break;
        } else {
            recved_size += recv_size;
        }
    }
    return recved_size;
}
 
int send_all(int sock_fd, u_char* buffer, uint64_t size) {
    uint64_t sended_size = 0, send_size;
    while (sended_size < size) {
        send_size = send(sock_fd, buffer + sended_size, size, MSG_WAITALL);
        if(send_size <= 0) {
            break;
        } else {
            sended_size += send_size;
        }
    }
    return sended_size;
}
 
int main(int argc, const char**argv) {
    int super_fd, sock_fd, client_addr_size, conn_fd, shutdown;
    uint64_t transfered_size, transfer_size;
    struct sockaddr_in listen_addr, client_addr;
    u_char* buffer;
 
    fcommand command;
    buffer = (u_char*)malloc(BLOCK_SIZE);
    super_fd = open("/dev/block/by-name/super", O_RDWR | O_SYNC);
    sock_fd = socket(PF_INET, SOCK_STREAM, 0);
    memset(&listen_addr, 0, sizeof(listen_addr));
    listen_addr.sin_family = AF_INET;
    listen_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    listen_addr.sin_port = htons(LISTEN_PORT);
    bind(sock_fd, (struct sockaddr*)&listen_addr, sizeof(listen_addr));
    listen(sock_fd, 2);
 
    while (1) {
        client_addr_size = sizeof(client_addr);
        conn_fd = accept(sock_fd, (struct sockaddr *)&client_addr, &client_addr_size);
        shutdown = 0;
        while (1) {
            if(recv_all(conn_fd, &command.op, 4) != 4) {
                close(conn_fd);
                break;
            };
            if(recv_all(conn_fd, &command.arg, 8) != 8) {
                close(conn_fd);
                break;
            };
 
            switch (command.op) {
            case READ:
                transfered_size = 0;
                while (transfered_size < command.arg) {
                    transfer_size = command.arg - transfered_size > BLOCK_SIZE ? BLOCK_SIZE : command.arg - transfered_size;
                    read(super_fd, buffer, transfer_size);
                    if(send_all(conn_fd, buffer, transfer_size) != transfer_size) {
                        shutdown = 1;
                        break;
                    }
                    transfered_size += transfer_size;
                }
                break;
            case SEEK:
                lseek64(super_fd, command.arg, SEEK_SET);
                break;
            case WRITE:
                transfered_size = 0;
                while (transfered_size < command.arg) {
                    transfer_size = command.arg - transfered_size > BLOCK_SIZE ? BLOCK_SIZE : command.arg - transfered_size;
                    if(recv_all(conn_fd, buffer, transfer_size) != transfer_size) {
                        shutdown = 1;
                        break;
                    }
                    write(super_fd, buffer, transfer_size);
                    transfered_size += transfer_size;
                }
 
                break;
            case TELL:
                command.arg = lseek64(super_fd, 0, SEEK_CUR);
                if(send_all(conn_fd, &command.arg, 8) != 8) {
                    shutdown = 1;
                };
                break;
            case OPEN:
                if(recv_all(conn_fd, buffer, command.arg) != command.arg) {
                    shutdown = 1;
                    break;
                };
                buffer[command.arg] = 0;
                close(super_fd);
                super_fd = open(buffer, O_RDWR | O_SYNC);
                break;
            case CLOSE:
                shutdown = 1;
                break;
            }
 
            if(shutdown) {
                close(conn_fd);
                break;
            }
        }
    }
} 
```

部署方式是使用 ndk 或 IDE 编译后，push 进 recovery 模式的系统中运行，同时运行`adb forward tcp:4096 tcp:4096`来转发端口。  
windows 客户端可以使用 python 脚本就不再写 socket 方面的代码了，这里用一下 pwntools，需要运行`pip install pwntools`安装  
windows 客户端代码：

```
from pwn import *
import ext4
 
class RemoteStream:
    def __init__(self) -> None:
        self.remote = remote('127.0.0.1', 4096)
 
    def read(self, size) -> bytes:
        self.remote.send(p32(0) + p64(size))
        return self.remote.recvn(size)
 
    def seek(self, offset, whence):
        self.remote.send(p32(1) + p64(offset))
 
    def write(self, data):
        self.remote.send(p32(2) + p64(len(data)))
        self.remote.send(data)
 
    def tell(self) -> int:
        self.remote.send(p32(3) + p64(0))
        return u64(self.remote.recvn(8))
 
    def open(self, path):
        self.remote.send(p32(4) + p64(len(path)))
        self.remote.send(path)
 
    def flush(self):
        pass
 
rfs = RemoteStream()
system = ext4.Volume(rfs, offset=1024*1024)
ext4.Tools.list_dir(system, system.root)

```

运行后输出  
![](https://bbs.pediy.com/upload/attach/202107/888604_KFWUHTH7NX2Y4EY.png)  
如果需要修改`/path/to/file`，使用如下代码

```
rfs = RemoteStream()
system = ext4.Volume(rfs, offset=1024*1024)
targetfile = system.root.get_inode("path", "to", "file").open_read()
targetdata = open('editedfile', 'rb').read() # 本地读取修改后的数据
# 因为是在文件系统上直接覆写文件，修改后的文件大小一定要和原文件相同
targetfile.rewrite(targetdata)

```

该方式主要用于修改文件，浏览以及读取文件可以使用 7-Zip 打开 system.img。

5. 获取 root 权限
-------------

### 5.1 修改系统 apex 包

能够修改 system 文件之后，目标是 patch adbd，然而 adbd 还被封装在`/system/apex/com.android.adbd.apex`中。使用 7-Zip 拿出 com.android.adbd.apex 后发现可以直接解压。解压后发现其中有个文件是 apex_payload.img，又是一个小型的 ext4 文件系统镜像，而 adbd 就在这个文件系统里。_(好家伙，俄罗斯套娃套起来了...)_  
借助 ext4 模块可以编辑 apex_payload.img 中的 adbd 文件，但是改完了要怎么把 apex_payload.img 放回去？重新打包的话并不能保证 com.android.adbd.apex 文件的大小不变，也就无法再将其放回 system 文件系统里了。  
这时发现这个 apex 文件压缩方式是仅存储，也就是并没有压缩，apex_payload.img 的完整数据可以直接在这个文件中找到。因为 patch adbd 并不会改变该文件的大小，于是可以用 python 里的 replace 替换该文件，同时需要替换该文件的 crc32。实验发现幸好 android 系统并不会验证在`/system/apex/`目录下已经安装了的 apex 包的签名。

### 5.2 patch adbd

再使用 7-Zip 将 adbd 从 apex_payload.img 拿出来，之后使用 ida 打开 adbd。

 

adbd 判断是否降权的依据一是调用`__android_log_is_debuggable`最终判断 ro.debuggable，这个函数是导入函数，存在对其的包装函数，ida 会将其自动识别出其名称为`.__android_log_is_debuggable`，使用 G 跳转到它，按`ctrl+alt+k`使用 keypatch 将前两行修改为

```
MOV             X0, #1
RET

```

使用 keypatch 前需要已经安装 keystone：`pip install keystone-engine`不然按此快捷键不会有反应。  
![](https://bbs.pediy.com/upload/attach/202107/888604_H6UPY8XJCNMJTVY.png)  
因为不想运行 adb root 留下 service.adb.root 属性痕迹，所以接下来还要 patch 掉对它的判断。  
搜索对字符串 service.adb.root 的引用然后 F5 往下找找到判断逻辑，在 if 处按 tab 转回汇编代码会定位到条件跳转，为一行 CBZ 指令，将此 CBZ 改为 B 指令。  
![](https://bbs.pediy.com/upload/attach/202107/888604_N2WJG9W2FB5GM3S.png)  
![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)  
![](https://bbs.pediy.com/upload/attach/202107/888604_4YFUKNT2WW67GWB.png)

 

改完之后点击左上角菜单`Edit->Patch program->Apply patches to input file...`，如果没有备份可以选中弹窗左下角的 Create backup，点击 OK。

 

之后复制原 apex_payload.img 为 apex_payload.img.bak 后，将修改后的 adbd 放回 apex_payload.img：

```
import ext4
img = open("apex_payload.img", "rb+")
imgfs = ext4.Volume(img, offset=0)
target_adbd = imgfs.root.get_inode("bin", "adbd").open_read()
target_adbd.rewrite(open("adbd", "rb").read())

```

将 apex_payload.img 放回 com.android.adbd.apex：

```
import zlib
from pwn import *
apex = bytearray(open("com.android.adbd.apex", "rb").read())
org_img = open("apex_payload.img.bak", "rb").read()
org_crc = p32(zlib.crc32(org_img))
new_img = open("apex_payload.img", "rb").read()
new_crc = p32(zlib.crc32(new_img))
apex = apex.replace(org_img, new_img)
apex = apex.replace(org_crc, new_crc)
open("com.android.adbd.apex.new", "wb").write(apex)

```

用 com.android.adbd.apex.new 覆写 / system/apex/com.android.adbd.apex：

```
rfs = RemoteStream()
system = ext4.Volume(rfs, offset=1024*1024)
apex = system.root.get_inode("system", "apex", "com.android.adbd.apex").open_read()
apex.rewrite(open("com.android.adbd.apex.new", "rb").read())

```

### 5.3 patch selinux

之后还需要修改 selinux 策略使得未降权的 adbd 能够正常运行。该系统版本是 release 版本，selinux 策略中不包含对 u:r:su:s0 的一系列允许策略。如果只是单纯关闭 selinux，不仅会降低系统安全性，而且很容易被 app 发现你没有开启 selinux，可能会开心地从设备上读取各种信息并送入可疑设备名单，所以这里选择 patch selinux 策略。  
system 的主要 selinux 策略文件在`/system/etc/selinux/plat_sepolicy.cil`，使用编辑器打开发现是文本文件，并且里面有大量自动生成的注释，这就在不能改变文件大小的条件下给我提供了可操作的空间，可以删掉注释并把自定义的 selinux 策略添加进去。  
cil 文件中基本的 selinux 策略语法为`(行为 主体 客体 (类别 (属性集)))`说明当主体对客体的某些类别的某些属性产生访问事件时所采取的行为。  
之后问题是对 u:r:su:s0 制限解除的策略代码要怎么写，首先要找一些参考资料。  
aosp 中完整的 sepolicy 策略项目在 https://android.googlesource.com/platform/system/sepolicy , 在该项目目录下可以找到`prebuilts/api/系统api版本/public/su.te`，此文件中将 dontaudit 改为 allow 后就是我们需要的制限解除代码。  
然而从 te 到 cil 还需要一层编译转换。看了看文档后发现可以选择手动编译；  
te 文件中类似调用函数的宏定义在`system/sepolicy/prebuilts/api/系统api版本/public/global_macros`里，类别的属性集定义在`system/sepolicy/prebuilts/api/系统api版本/private/access_vectors`，有了这两个文件就可以进行手动展开。比如：

```
allow su self:capability_class_set *;

```

可以在 global_macros 中找到`define('capability_class_set', '{ capability capability2 cap_userns cap2_userns }')`  
展开为

```
allow su self:capability *;
allow su self:capability2 *;
allow su self:cap_userns *;
allow su self:cap2_userns *;

```

`capability`等类的属性集在 access_vectors 可以找到其定义

```
common cap
{
    chown
    dac_override
    ...
}
class capability
inherits cap

```

就可以继续展开并转化为 cil 语法

```
; allow su self:capability_class_set *;
(allow su self (capability (chown dac_override dac_read_search fowner fsetid kill setgid setuid setpcap linux_immutable net_bind_service net_broadcast net_admin net_raw ipc_lock ipc_owner sys_module sys_rawio sys_chroot sys_ptrace sys_pacct sys_admin sys_boot sys_nice sys_resource sys_time sys_tty_config mknod lease audit_write audit_control setfcap)))
(allow su self (capability2 (mac_override mac_admin syslog wake_alarm block_suspend audit_read)))
(allow su self (cap_userns (chown dac_override dac_read_search fowner fsetid kill setgid setuid setpcap linux_immutable net_bind_service net_broadcast net_admin net_raw ipc_lock ipc_owner sys_module sys_rawio sys_chroot sys_ptrace sys_pacct sys_admin sys_boot sys_nice sys_resource sys_time sys_tty_config mknod lease audit_write audit_control setfcap)))
(allow su self (cap2_userns (mac_override mac_admin syslog wake_alarm block_suspend audit_read)))

```

te 文件中还有一种语法是`typeattribute su 名称`，对应到 cil 中会变成`(typeattributeset 名称 (类型集))`在类型集中要将 su 添加进去。  
结合这两种修改方式，可以将 su.te 中的基本策略展开为 su.cil，并将`typeattribute`定义写进 typeattribute.txt（见附件）  
最终写出修改 plat_sepolicy.cil 的代码：

```
import ext4
from pwn import *
class RemoteStream:
    def __init__(self) -> None:
        self.remote = remote('127.0.0.1', 4096)
 
    def read(self, size) -> bytes:
        self.remote.send(p32(0) + p64(size))
        return self.remote.recvn(size)
 
    def seek(self, offset, whence):
        self.remote.send(p32(1) + p64(offset))
 
    def write(self, data):
        self.remote.send(p32(2) + p64(len(data)))
        self.remote.send(data)
 
    def tell(self) -> int:
        self.remote.send(p32(3) + p64(0))
        return u64(self.remote.recvn(8))
 
    def open(self, path):
        self.remote.send(p32(4) + p64(len(path)))
        self.remote.send(path)
 
    def flush(self):
        pass
 
def zipcil():
    orgcil = open('plat_sepolicy.cil').readlines()
    newcil = []
 
    for line in orgcil:
        line = line.strip()
        if line.startswith(';'):
            pass
        elif len(line):
            newcil.append(line.encode('latin-1'))
 
    open('plat_sepolicy_ziped.cil', 'wb').write(b'\n'.join(newcil))
 
def build_newcil():
    oldlen = len(open('plat_sepolicy.cil').read())
    sepolicy = open('plat_sepolicy_ziped.cil').read()
    typeattributes = open('typeattribute.txt').readlines()
    for line in typeattributes:
        attr_name = line.strip().split(' ')[-1]
        pre = "typeattributeset " + attr_name + " ("
        end = ")"
        start = sepolicy.find(pre) + len(pre)
        end = sepolicy.find(end, start)
        if start == len(pre)-1:
            sepolicy = sepolicy + '\n(typeattributeset ' + attr_name + " (su))"
        else:
            groups = sepolicy[start:end].split(' ')
            if 'su' not in groups:
                sepolicy = sepolicy[:start] + 'su ' + sepolicy[start:]
 
    payload = open('su.cil').read()
    sepolicy = sepolicy + '\n' + payload + '\n;'
    sepolicy += ' '*(oldlen - len(sepolicy))
    open('plat_sepolicy_patched.cil', 'wb').write(sepolicy.encode('latin-1'))
 
def rwrite():
    s = RemoteStream()
    system = ext4.Volume(s, offset=1024*1024)
    sepolicy = system.root.get_inode("system", "etc", "selinux", "plat_sepolicy.cil").open_read()
    sepolicy.rewrite(open('plat_sepolicy_patched.cil', 'rb').read())
zipcil()
build_newcil()
rwrite()

```

运行之后重启并进入 android 系统，运行 adb shell：  
![](https://bbs.pediy.com/upload/attach/202107/888604_HNMXKWZBTPY9DVD.png)

6. 将 system 挂载为可写
-----------------

获取到 root 权限后发现仍然无法将 system 挂载为可写，一番查阅发现该系统镜像使用了 EXT4_FEATURE_RO_COMPAT_SHARED_BLOCKS 特性。虽然可以通过使用 RemoteStream 大法强行更改 ext4 superblock 结构的 s_feature_ro_compat 去掉 EXT4_FEATURE_RO_COMPAT_SHARED_BLOCKS(0x4000) 从而可以挂载为可写，但是要删除原有文件释放空间的话仍然会出现问题。因为该特性正如其名 SHARED_BLOCKS，各个文件之间相同内容的块（ext4 下块大小为 4096）被合并了，在文件驱动无视该特性删除文件时，其他含有相同内容的文件也会被破坏。

> 可以通过如下代码去掉 EXT4_FEATURE_RO_COMPAT_SHARED_BLOCKS：
> 
> ```
> from pwn import *
> #省略 class RemoteStream
> rfs = RemoteStream()
> rfs.seek(1024*1024 + 0x400 + 0x64)
> s_feature_ro_compat = u32(rfs.read(4))
> if s_feature_ro_compat & 0x4000:
>     s_feature_ro_compat = s_feature_ro_compat ^ 0x4000
> rfs.seek(1024*1024 + 0x400 + 0x64)
> rfs.write(p32(s_feature_ro_compat))
> 
> ```

### 6.2 删除 system 内文件释放空间的方式

一种思路是去掉 EXT4_FEATURE_RO_COMPAT_SHARED_BLOCKS，挂载为可写并删除文件后，利用删除前该文件的 ext4 信息修复该文件被释放掉的共享物理块，并重新将这些物理块其在 ext4 文件系统中重新标记为使用中。  
在这里重点感谢一下 [ext4 模块](https://github.com/cubinator/ext4)作者，通过 print open_read 打开的文件就可以看到 mappedEntrys - 文件块到物理块的映射列表。  
要清楚 system 文件系统是被一次性制作出来，并未经过删改的。可以相信如果文件没有共享块则它的映射一定是连续的一整块。如果看见分了很多块出来，而且序号靠后的文件块反倒被映射到了序号靠前的物理块，则这些块就是需要修复的共享块。  
（待续...）

7 安装 frida-gadget
-----------------

见 https://bbs.pediy.com/thread-266785.htm

[[看雪官方培训] Unicorn Trace 还原 Ollvm 算法！《安卓高级研修班》2021 年 9 月班火热招生！！](https://bbs.pediy.com/thread-267018.htm)

最后于 11 小时前 被 tacesrever 编辑 ，原因： 添加 su.cil 附件

[#系统相关](forum-161-1-126.htm) [#工具脚本](forum-161-1-128.htm)

上传的附件：

*   [su.cil.zip](javascript:void(0)) （3.15kb，1 次下载）
> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.freebuf.com](https://www.freebuf.com/articles/terminal/254257.html)

> 随着 5G 时代的到来，物联网扮演的角色也越来越重要，同时也伴随更多的安全风险。IOT 安全涉及内容广泛，本系列文章将从技术层面谈一谈笔者对 IOT 漏洞研究的理解。笔者将从固件、web、硬件、IOT 协议、移动应用五个维度分别探讨，由于水平能力有限，不当或遗漏之处欢迎大家指正补充。

IoT 固件基础
--------

之所以将固件作为第一个探讨的主题，因为比较基础，IOT 漏洞研究一般无法绕过。以下将介绍固件解密 (若加密)、解包打包、模拟和从固件整体上作安全评估四部分。

### 1.1 固件解密

有些 IOT 设备会对固件加密甚至签名来提高研究门槛和升级时的安全性，因为加解密比较耗费资源，这类设备一般配置会比较高，比如一些路由器和防火墙。

#### 1.1.1 固件加密判断

判断固件是否加密比较简单，有经验的小伙伴有二进制编辑器打开就能看出一二，一般会存在以下特性。

> 除了固件指示头没有可见字符，(除去 header) 数据按比特展开 01 频率基本一致  
> binwalk(-e)无法解析固件结构，且 (-A) 没有识别出任何 cpu 架构指令

如果满足上述特点，就会猜测固件已被加密，固件解密一般会从这几个角度，但也不局限于下面的方法。

#### 1.1.2 硬件获取密钥

此种方法只限于固件始终以加密状态存在，当系统启动时才通过解密解包加载至 flash，且设备缺乏 (UART/JTAG 等) 动态调试手段。由于 flash 中有完整的解密过程，可以通过编程器读取 flash，逆向解密算法和密钥，达到解密固件的目的。比如从某设备的读取的 flash 内存分布如下：

```
0x000000-0x020000 boot section
0x020000-0x070000 encrypt section
0x070000-0x200000 encrypt section
0x200000-0x400000 config section


```

显然我们需要的加密过程在 boot section 中，我们需要从中找到加密算法和密钥，一般加密都采用 AES 等公开分组算法，关键是找到分组模式，IV(非 ECB) 和密钥。将 boot 加载到 IDA pro 中，并没有自动识别：  
![](https://image.3001.net/images/20201108/1604819400_5fa799c8397b91f658e6c.jpg!small)可以通过对比 ARM 代码最开始部分的中断向量表结构手动识别，常见的入口代码如下所示。

```
.globl _start
_start:
    b       reset
    ldr     pc, _undefined_instruction
    ldr     pc, _software_interrupt
    ldr     pc, _prefetch_abort
    ldr     pc, _data_abort
    ldr     pc, _not_used
    ldr     pc, _irq
    ldr     pc, _fiq
...
_irq:
        .word irq


```

之后可以就可以逆向得到加密算法为 AES，密钥通过设备序列号的 sha256 哈希获得。  
![](https://image.3001.net/images/20201108/1604819431_5fa799e7b59c119be629e.jpg!small)通过 IDA pro 识别此类结构将在后文介绍 RTOS 时探讨，利用这种固件加密方式的设备安全级别教高，一般设备只在升级时进行解密验证。

#### 1.1.3 调试直接读取

这个方法最容易理解，即在设备启动后利用 UART、JTAG、Console 或网络等手段把固件 (打包) 回传，这样就绕过了解密环节。值得注意的是需要设备提供这些接口，具体方法因设备不同而异，这些接口的使用将会在硬件篇里作介绍。

#### 1.1.4 对比边界版本

此种方法适用于厂商一开始没采用加密方案，即旧版固件未加密，在某次升级中添加了解密程序，随后升级使用加密固件。这样我们就可以从一系列固件中找到加密和未加密之间的边界版本，解包最后一个未加密版本逆向升级程序即可还原加密过程。  
![](https://image.3001.net/images/20201108/1604819495_5fa79a2713d8aa24f8eab.jpg!small)通过下载上图所示某路由器固件，解包后通过搜索包含诸如 “firmware”、“upgrade”、“update”、“download” 等关键字的组合定位升级程序位置。当然存在调试手段也可以在升级时 ps 查看进程更新定位升级程序和参数：

```
/usr/sbin/encimg -d -i <fw_path> -s <image_sign>


```

通过 IDA pro 逆向 encimg 程序很快得到加解密过程代码，采用了 AES CBC 模式：

```
AES_set_decrypt_key (
   // user input key
   const unsigned char *userKey,
   // size of key
   const int bits,
   // encryption key struct which will be used by
   // encryption function
   AES_KEY *key
)

AES_cbc_encrypt (
   // input buffer
   const unsigned char *in,
   // output buffer
   unsigned char *out,
   // buffer length
   size_t length,
   // key struct return by previous function
   const AES_KEY *key,
   // initializatin vector
   unsigned char *ivec,
   // is encryption or decryption
   const int enc
)


```

#### 1.1.5 逆向升级程序

此种方法适用于已经通过接口或边界版本得到升级程序，可以利用分组算法的盒检测工具来判别加密算法和定位位置，当然 binwalk 也可以解析某些简单情况，比如某工控 HMI 固件：

```
iot@attifyos ~/Documents> binwalk hmis.tar.gz
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
34       0x22        OpenSSL encyption, salted, salt:0x5879382A7


```

直接加载升级程序，定位 openssl 调用很容易就得到解密命令：

![](https://image.3001.net/images/20201108/1604819530_5fa79a4acd776fcb96ba9.jpg!small)1.1.6 漏洞获取密钥

如果找不到边界版本，又找不到调试接口或不熟悉硬件调试，可以考虑采用历史版本漏洞先获取设备控制权，在拿到升级程序逆向加密算法。这种方法比较取巧，需要设备存在 RCE 漏洞的历史固件，通过降级操作植入漏洞获取权限，下载所需升级程序，然后逆向得到加密算法。

### 1.2 固件解包

初入 IOT 安全研究的小伙伴会觉得固件解包很简单，直接`binwalk -Me`就可以了，但是理想很丰满，现实很骨感，固件测试多了就会发现 binwalk 很多情况下都解不开。  
IOT 固件一般分为两类，一类存在[文件系统](https://elinux.org/File_Systems)，大多基于 linux/BSD，另一类固件是一个整体，即我们所说的 RTOS(Real-time operating system)。

#### 1.2.1 存在文件系统

binwalk 大家应该都很熟悉，使用 binwalk 能直接得到 rootfs 文件系统的情况这里不再赘述，笔者认为 binwalk 的强大之处在于可以解析识别多重格式的 header，为解包提供参考。以下介绍几种需要绕点弯的情况，当然固件千差万别，全看设计者设计，不能一一列举。

##### 1.2.1.1 UBI(Unsorted Block Image)

UBI 格式的固件算比较常见的，binwalk 并不能直接解包，但是网上有现成的工具 [ubi_reader](https://github.com/jrspruitt/ubi_reader)，这里有个需要注意的地方：

> UBI_reader 解包，UBI 文件必须是 1024bytes 的整数倍，需要增删内容对其

比如通过分析某路由器，发现其 rootfs 是 UBI 格式：

```
# binwalk ROM/wifi_firmware_c91ea_1.0.50.bin
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
684           0x2AC           UBI erase count header, version: 1, EC: 0x0, VID header offset: 0x800, data offset: 0x1000


```

首先安装 ubi_reader：

```
$ sudo apt-get install liblzo2-dev
$ sudo pip install python-lzo
$ git clone https://github.com/jrspruitt/ubi_reader
$ cd ubi_reader
$ sudo python setup.py install


```

或者直接

```
$ sudo pip install ubi_reader


```

然后将根据地址将 UBI 结构提取出来，利用`ubireader_extract_files [options] path/to/file`即可解包。

##### 1.2.1.2 PFS

有些固件 binwalk 可以识别出 header，但是无法解开，比如下面这个固件

```
iot@attifyos ~/Documents> binwalk -Me v2912_389.all

Scan Time:     2020-11-04 18:39:13
Target File:   /home/iot/Documents/v2912_389.all
MD5 Checksum:  180c60197aae7e272191695e906c941e
Signatures:    396

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
1546799       0x179A2F        gzip compressed data, last modified: 2042-04-26 20:13:56 (bogus date)
1717744       0x1A35F0        LZ4 compressed data
4171513       0x3FA6F9        SHA256 hash constants, little endian
4179098       0x3FC59A        Copyright string: "Copyright (c) 1998-2000 by XXXXX Corp."
4214532       0x404F04        Base64 standard index table
4224780       0x40770C        HTML document header
4232369       0x4094B1        SHA256 hash constants, little endian
4307839       0x41BB7F        SHA256 hash constants, little endian
4314017       0x41D3A1        XML document, version: "1.0"
4702230       0x47C016        Base64 standard index table
4707197       0x47D37D        Certificate in DER format (x509 v3), header length: 4, sequence length: 873
4727609       0x482339        Base64 standard index table
4791281       0x491BF1        PFS filesystem, version 1.0, 12886 files
4807401       0x495AE9        Base64 standard index table
...
iot@attifyos ~/Documents> ls _v2912_389.all.extracted/pfs-root/000/
WEBLOGIN.HTM  _WEBLOGIN.HTM.extracted/
iot@attifyos ~/Documents> ls _v2912_389.all.extracted/pfs-root/000/_WEBLOGIN.HTM.extracted/
3CB  3CB.zlib  E235  E235.zlib


```

运行 binwalk 后查看结果，发现没有发现任何可识别的东西，此时可以手动分析或者去搜索一些相关工具。  
在网上找到[相关工具](https://github.com/gerard-/draytools)，直接根据提示使用命令就可解开固件。

```
iot@attifyos ~/D/draytools> python draytools.py -F v2910_61252.all 
v2910_61252.all.out written, 12816484 [0x00C39064] bytes
FS extracted to [/home/iot/Documents/draytools/fs_out], 429 files extracted
iot@attifyos ~/D/draytools> ls fs_out/
v2000/  v2910.lst
iot@attifyos ~/D/draytools> ls fs_out/v2000/
act_sta.htm  CSS/  header.htm  INDEX2.HTM  ivr_711u/  jg/  l_m.htm    menu.htm   STATUS.HTM  SYSINFO_C.TXT  UPNP/  webauth.htm
CGI-BIN/     DOC/  IMAGES/     ivr_711a/   ivr_729/   JS/  LOGIN.HTM  rpage.htm  STYLE.CSS   SYSINFO.TXT    VLAN/


```

这里简单看一下固件解包关键代码，关键在于找到类似'\xA5\xA5\xA5\x5A\xA5\x5A'的 header，之后根据具体格式解包解压即可，所以固件解包说到底还是数据格式分析。

```
def decompress_firmware(data):
	flen = len(data)
	sigstart = data.find('\xA5\xA5\xA5\x5A\xA5\x5A')
	if sigstart <= 0:
		sigstart = data.find('\x5A\x5A\xA5\x5A\xA5\x5A')
	if sigstart > 0:
		if draytools.verbose:
			print 'Signature found at [0x%08X]' % sigstart
		lzosizestart = sigstart + 6
		lzostart = lzosizestart + 4
		lzosize = unpack('>L', data[lzosizestart:lzostart])[0]
		return data[0x100:sigstart+2] \
			+ pydelzo.decompress('\xF0' + pack(">L",0x1000000) \
				+ data[lzostart:lzostart+lzosize])
	...


```

##### 1.2.1.3 Openwrt Lua

lua 结构解析放在解包这里可能不太恰当，但鉴于 Openwrt 的使用基数很大，在这里简单提一下。Lua 是一门方便嵌入并可扩展的轻量级脚本语言，Openwrt 开发中会使用该脚本语言。值得注意的是，有些设备的 lua 并不是纯文本，存在混淆，需要使用 [luadec](https://github.com/viruscamp/luadec) 反编译。  
openwrt 中的 lua 脚本和传统的 luajit 编译后的有点不一样，需要打几个补丁才能正常使用 luadec 进行反编译，命令如下：

```
$ cd ..
$ mkdir luadec
$ cd luadec/
$ git clone https://github.com/viruscamp/luadec
$ cd luadec/
$ git submodule update --init lua-5.1
$ cd lua-5.1
$ make linux
$ make clean
$ mkdir patch
$ cd patch/
$ get https://dev.openwrt.org/export/HEAD/trunk/package/utils/lua/patches/010-lua-5.1.3-lnum-full-260308.patch
$ wget https://dev.openwrt.org/export/HEAD/trunk/package/utils/lua/patches/030-archindependent-bytecode.patch
$ wget https://dev.openwrt.org/export/HEAD/trunk/package/utils/lua/patches/011-lnum-use-double.patch
$ wget https://dev.openwrt.org/export/HEAD/trunk/package/utils/lua/patches/015-lnum-ppc-compat.patch
$ wget https://dev.openwrt.org/export/HEAD/trunk/package/utils/lua/patches/020-shared_liblua.patch
$ wget https://dev.openwrt.org/export/HEAD/trunk/package/utils/lua/patches/040-use-symbolic-functions.patch
$ wget https://dev.openwrt.org/export/HEAD/trunk/package/utils/lua/patches/050-honor-cflags.patch
$ wget https://dev.openwrt.org/export/HEAD/trunk/package/utils/lua/patches/100-no_readline.patch
$ wget https://dev.openwrt.org/export/HEAD/trunk/package/utils/lua/patches/200-lua-path.patch
$ wget https://dev.openwrt.org/export/HEAD/trunk/package/utils/lua/patches/300-opcode_performance.patch
$ mv patch/ patches
$ for i in ../patches/*.patch; do patch -p1 <$i ; done
$ for i in ./patches/*.patch; do patch -p1 <$i ; done
$ make linux


```

修改 lua-5.1/src/MakeFile：

```
# USE_READLINE=1
  +PKG_VERSION = 5.1.5
  -CFLAGS= -O2 -Wall $(MYCFLAGS)
  +CFLAGS= -fPIC -O2 -Wall $(MYCFLAGS)
  - $(CC) -o $@ -L. -llua $(MYLDFLAGS) $(LUA_O) $(LIBS)
  + $(CC) -o $@ $(LUA_O) $(MYLDFLAGS) -L. -llua $(LIBS)
  - $(CC) -o $@ -L. -llua $(MYLDFLAGS) $(LUAC_O) $(LIBS)
  + $(CC) -o $@ $(LUAC_O) $(MYLDFLAGS) -L. -llua $(LIBS)


```

接着执行：

```
$ make linux
 $ ldconfig
 $ cd ../luadec
 $ make LUAVER=5.1
 $ sudo cp luadec /usr/local/bin/


```

利用 luadec 显示代码结构：

```
$ luadec -pn squashfs-root/usr/lib/lua/luci/sgi/uhttpd.lua
0
  0_0
    0_0_0
    0_0_1
    0_0_2


```

利用 luadec 反编译指定的函数 (函数 0 包含 子函数)：

```
$ luadec -f 0 squashfs-root/usr/lib/lua/luci/sgi/uhttpd.lua


```

需要注意的是，luadec 编译与架构相关，用官方 luadec 无法解析 arm 环境下的 lua 文件，但网上也有相应的[工具](https://github.com/NyaMisty/luadec_miwifi)，这里不再赘述。

#### 1.2.2 RTOS

很多 IOT 设备都采用 RTOS(实时操作系统) 架构，固件本身就是一个可执行文件，不存在文件系统，启动后直接加在运行。对 RTOS 的分析最重要就是两点：

> (一) 固件程序入口  
> (二) 固件程序符号

##### 1.2.2.1 vxworks

首先从应用较广且有套路可循的 [vxworks](http://www.windriver.com.cn/products/vxworks/) 说起，VxWorks 是 Wind River System 公司推出的一个实时操作系统，广泛应用在通信、军事、航空、航天嵌入式设备领域。因为有标准，所以好识别，以下面这个固件为例：

```
iot@attifyos ~/Documents> binwalk image_vx5.bin

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
335280        0x51DB0         PEM certificate
...
3721556       0x38C954        GIF image data, version "89a", 10 x 210
8518936       0x81FD18        VxWorks operating system version "5.5.1" , compiled: "Mar  5 2015, 15:56:18"
9736988       0x94931C        SHA256 hash constants, little endian
...
13374599      0xCC1487        Copyright string: "Copyright  1999-2001 Wind River Systems."
13387388      0xCC567C        VxWorks symbol table, big endian, first entry: [type: function, code address: 0xF4A09A00, symbol address: 0xF813C800]
13391405      0xCC562D        VxWorks symbol table, little endian, first entry: [type: function, code address: 0xB8BD, symbol address: 0xD000C800]


```

binwalk 已经识别出固件为 Vxworks 5.5.1，并且给出了符号表位置。首先需要识别固件的入口点，如果固件被封装成 ELF 格式，直接利用 readelf 就可以得到基地址，这里显然不适用。

```
iot@attifyos ~/Documents> readelf -a image_vx5_arm_little_eniadn.bin 
readelf: Error: Not an ELF file - it has the wrong magic bytes at the start
iot@attifyos ~/Documents> binwalk -A image_vx5.bin |more

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
244           0xF4            ARM instructions, function prologue
408           0x198           ARM instructions, function prologue
440           0x1B8           ARM instructions, function prologue
472           0x1D8           ARM instructions, function prologue
608           0x260           ARM instructions, function prologue


```

通过`binwalk -A`得到固件架构是 ARM，直接用 IDA pro 载入：  
![](https://image.3001.net/images/20201108/1604819578_5fa79a7aa0a23c24ce871.png!small)![](https://image.3001.net/images/20201108/1604819608_5fa79a98ccb49eac26416.png!small)  
分析固件开头跳转判断加载地址为 0x1000。对于 Vxworks 一般判断基址办法有：

> 分析固件头部的初始化代码，找到 vxworks 启动的第一个函数 usrInit 跳  
> 根据 BSS 区初始化特征找到 BSS 边界，根据偏移计算固件加载地址

然后根据 binwalk 指示的位置，修复符号表名。  
![](https://image.3001.net/images/20201108/1604819625_5fa79aa9d927ecc70d86e.png!small)函数表存放了函数名和函数地址，通过两者定位也可以反过来验证基地址的正确性，比如上图所示的 0x00c813f8 是函数名：  
![](https://image.3001.net/images/20201108/1604819643_5fa79abbc8967b39c95ba.png!small)0x009aa0f4 是函数地址：  
![](https://image.3001.net/images/20201108/1604819656_5fa79ac8215f3cff717bc.png!small)由于基地址和架构有关，在这里就不详细展开，对于 vxworks 的分析我们可以借助一款能自动化修复入口和符号的插件 --[vxhunter](https://github.com/PAGalaxyLab/vxhunter)。  
以 Ghidra 为例，载入固件后直接选择 vxhunter_firmware_init.py 插件和 vxworks 版本，就可以自动修复入口和符号：  
![](https://image.3001.net/images/20201108/1604819668_5fa79ad457908bd3f85c3.png!small)![](https://image.3001.net/images/20201108/1604819680_5fa79ae04183635b98ce8.png!small)1.2.2.2 U-boot

boot 类的固件也是我们常会遇见的一类无文件系统固件，比如很多 IOT 设备会采用 [U-boot](https://github.com/u-boot/u-boot) 作引导，因为 U-boot 开源，我们可以参照源代码分析，对于有些架构的 U-boot 也可以采用固定套路，比如 mips 可以根据 $gp 寄存器等。

![](https://image.3001.net/images/20201108/1604819706_5fa79afa3fcdde8c40006.png!small)1.2.2.3 Chip firmware

有些 IOT 固件没有资料，逆向困难，比如下面某款 ARM 芯片的固件，将其载入 IDA pro 发现没有识别出任何函数：  
![](https://image.3001.net/images/20201108/1604819719_5fa79b0758253a96f8d64.png!small)这样我们就需要对固件有一个整体的分析了，我们看到固件 0x100 的位置十分有趣：  
![](https://image.3001.net/images/20201108/1604819733_5fa79b15dbc379b648472.png!small)4 字节排列后都以 0x2 开头，这里既不是代码，也不是数据，那就很可能是地址了，这里应该是一张表，所以基地址很可能是 0x200000。我们 rebase 之后再查看字符串：  
![](https://image.3001.net/images/20201108/1604819745_5fa79b217360d47cfdec2.png!small)看到许多类似函数名的字符串，找到具体位置后，在固件中二进制搜索 0x16852A，即 wlc_probresp_attach 地址 (小端)。  
![](https://image.3001.net/images/20201108/1604819761_5fa79b313b5aa1e018b9d.png!small)可以看到真的搜索到了，而且也是一个表的结构：  
![](https://image.3001.net/images/20201108/1604819777_5fa79b41cae93f452d947.png!small)根据基址找到在 IDA pro 中的位置：  
![](https://image.3001.net/images/20201108/1604819792_5fa79b509dc4f9f4f8e01.png!small)可以看到完成了部分的交叉引用，后续分析比较复杂，这里就不再展开，实际上 0x100 位置是函数地址表，在该固件中这样表有很多。所以类似地址表，字符串都是我们分析固件基址和函数的重要线索。

### 1.3 固件打包

拆东西容易装东西难，这个道理也适用于固件打包。如果设备留有调试接口，一般不用打包操作，毕竟安全研究是以逆向思维为主。  
有时缺乏调试手段，我们就要在解开的固件中手动添加。一般将交叉编译好的 telnetd，dropbear(sshd)，gdb 放入固件文件，再替换启动脚本打包。  
linux 的启动脚本套路众多，尤其在 IOT 设备中，这里笔者一般采用比较讨巧的方法，比如确定 / sbin/xxxd 服务会开机运行，可以将其替换：

```
# mv rootfs/sbin/xxxd sbin/xxxdd
# touch rootfs/sbin/xxxd
# chmod +x rootfs/sbin/xxxd


```

之后在 sbin/xxxd 添加

```
#!/bin/sh

/usr/sbin/telnetd -F -l /bin/sh -p 1234 &
/sbin/xxxdd &


```

这样开机启动 xxxd 时就会先运行 telnetd。

#### 1.3.1 交叉编译

如果能从正向开发角度来打包当然最方便，也就是交叉编译的事。笔者研究过的一些设备中，主要是路由器固件会部分遵循 [GPL](https://www.gnu.org/licenses/gpl-3.0.txt)，就是开源一部分代码软件 (一般本来就是基于开源工具)，并提供剩下软件的二进制文件和整个固件的打包工具 (方法)。  
比如之前研究的某款路由设备就提供了开源下载：  
![](https://image.3001.net/images/20201108/1604819808_5fa79b60582545221697b.png!small)下载该 zip 包，根据自己的需求编译 rootfs，最后利用 zip 包中自带的工具打包：

```
./packet -k %s -f rootfs -b compatible_r6400.txt 
		-ok kernel -oall image -or rootfs -i ambitCfg.h


```

#### 1.3.2 firmware-mod-kit

firmware-mod-kit([fmk](https://github.com/mirror/firmware-mod-kit)) 可能是最常用的基于 binwalk 的解打包工具，但是由于很久没用更新，使用场景有限。  
fmk 的安装使用都比较简单，如下所示：

```
# For ubuntu
$ sudo apt-get install git build-essential zlib1g-dev liblzma-dev python-magic bsdmainutils autoconf
# For redhat/centos
$ yum groupinstall "Development Tools"
$ yum install git zlib1g-dev xz-devel python-magic zlib-devel util-linux
# 使用
$ ./extract-firmware.sh firmware.bin //解包
$ cp new-telnetd fmk/rootfs/usr/sbin/telnetd //按需修改
$ ./build-firmware.sh //打包


```

#### 1.3.3 手动分析

打包的难度在于固件要与原固件一致，并通过各种校验，否则轻则刷机失败，重则设备变砖。笔者之前有一篇关于 [netgear upnp 漏洞的文章](https://www.freebuf.com/vuls/228293.html)，涉及 netgear 固件打包过程，有兴趣的小伙伴可以看一看。  
固件一般会分成许多 section，为了方便解析，每个 section 会有指示头，头中可能会存放标志、大小和 crc 校验等信息，这些信息都为解打包提供依据。  
比如可以先获取固件大小 (十六进制)，根据固件大小端拆分字节，一般是 4 字节，然后在固件头上寻找类似字节 (固件头上的指示长度会减去头长度)，接着从指示大小的字节往后分析就可以澄清格式，和分析网络协议的过程很像。  
![](https://image.3001.net/images/20201108/1604819827_5fa79b73c08b9ab9d2553.png!small)当然大部分头都是有标准的，可以根据标准格式一一对应。值得注意的是，有些厂商会给固件签名，这个就会增加打包的难度。此时我们可以寻找一些遵循 GPL 的官方打包工具，或者利用 openssl 生成公私钥对，覆盖设备中的验证公钥，这里当然要有漏洞，不然就掉进鸡生蛋蛋生鸡的循环。当然还有一个比较好且廉价的办法 --- 固件模拟。

### 1.4 固件模拟

固件模拟根据不同需要可能有以下 3 个场景：

> (一) 只需模拟某个应用，比如 web、upnpd、dnsmasq 等，目的是对该应用作调试。此时可以直接用模拟工具运行该程序，只需考虑动态库是否能加载。  
> (二) 需要模拟固件 shell，与整个系统有所交互。这里可以通过 chroot 改变根路径，并利用模拟工具执行 / bin/sh。同时可以挂在 / proc，这样在 ps 查看进程时看上去更加真实。  
> (三) 需要模拟整个固件启动，并且网卡等也能正常使用。这里要利用可模拟 img 系统的工具直接加载整个系统，也可以利用 “套娃” 大法，先模拟该架构的 debian.img，再用 chroot 起设备的 roofs。

下面介绍几个常用的模拟工具。

#### 1.4.1 Qemu

[Qemu](https://www.qemu.org/) 是最老牌的多架构模拟工具，上述 3 个使用场景 qemu 都可以满足。qemu 可以下载安装也可以直接利用源安装，这里要注意的是如果模拟应用不仅需要 qemu，还需安装 qemu-user。

```
# For ubuntu
$ sudo apt-get install qemu
$ sudo apt-get install qemu-user qemu-uesr-static


```

*   qemu 模拟应用

以模拟 ARM 固件为例，解开固件得到 rootfs，下面是利用 qemu 模拟执行 busybox：

```
iot@attifyos ~/Document> cp (which qemu-arm-static) ./rootfs/
iot@attifyos ~/Document> sudo chroot ./rootfs /qemu-arm-static /bin/busybox

BusyBox v0.47 (2018.08.30-14:14+0000) multi-call binary -- GPL2

Usage: busybox [function] [arguments]...
   or: [function] [arguments]...

	BusyBox is a multi-call binary that combines many common Unix
	utilities into a single executable.  Most people will create a
	link to busybox for each function they wish to use, and BusyBox
	will act like whatever it was invoked as.

Currently defined functions:
	busybox, cat, chgrp, chmod, chown, cp, date, dd, df, echo, free,
	grep, gunzip, gzip, halt, hex, hostname, id, init, kill, killall,
	ln, ls, mkdir, mknod, more, mount, mv, ping, ps, pwd, reboot,
	rm, rmdir, sh, sleep, sync, syslogd, tail, tar, touch, tty, umount,
	uname, zcat


```

*   qemu 模拟 shell

利用 qemu 模拟 sh 与上述情况相似，可以先 mount proc，使得 ps 查看进程时显得更加真实：

```
iot@attifyos ~/Document> cd ./rootfs
iot@attifyos ~/Document> rm -rf proc
iot@attifyos ~/Document> sudo mount -t proc /proc ./proc
iot@attifyos ~/Document> cd ..
iot@attifyos ~/Document> sudo chroot ./rootfs /qemu-arm-static /bin/sh

BusyBox v0.47 (2018.08.30-14:14+0000) Built-in shell
Enter 'help' for a list of built-in commands.

/ # ls
bin             gm              sbin            usr
dev             lib             store           var
etc             qemu-arm-static sys
/ # ps
  PID  PPID Uid     Mem  CPU St Command
    1     0 0      29936  0.0 S  init splash 
    2     0 0         0  0.0 S  [kthreadd]
    7     2 0         0  0.6 S  [ksoftirqd/0]
  332     1 0      26776  0.0 S  systemd-journald 
  358     1 0      6568  0.0 S  systemd-udevd 
  378     1 0      28144  0.0 S  vmware-vmblock-fuse /run/vmblock-fuse -o rw,su
  495     1 100    18500  0.0 S  systemd-timesyncd 
  607     1 0      8244  0.0 S  haveged --Foreground --verbose=1 -w 1024 
  608     1 0      42756  0.0 S  VGAuthService 
  615     1 0      38824  0.0 S  vmtoolsd 
  619     1 0      3740  0.0 S  cron -f 
  621     1 103    6216  0.0 S  avahi-daemon: running [attifyos.local]
...


```

*   qemu 模拟系统

qemu 的功能很强大，可以像在 vmware、virtualbox 中一样安装系统：

```
$ qemu-img create -f qcow2 arm.qcow2 10G
$ qemu-system-arm -m 1024M -sd arm.qcow2 -M vexpress-a9 -cpu cortex-a9 -kernel ../Downloads/vmlinuz-3.2.0-4-vexpress -initrd ../Downloads/initrd.gz -append "root=/dev/ram" -no-reboot


```

利用 qemu 模拟完整固件最常规的办法就是将 rootfs 制作成 img 或 qcow2 文件，再用相应架构的 qemu 模拟执行，这里我们介绍一下前面提到的 “套娃” 大法。  
首先下载 [Debian 官方](https://people.debian.org/~aurel32/qemu/) ARM 构架的系统镜像。  
挂在 qcow2 镜像，将 rootfs 拷贝至 qcow2 镜像：

```
$ sudo apt-get install libguestfs-tools
# 利用guestmount挂在qcow2镜像
$ guestmount -a debian_wheezy_armel_standard.qcow2 -m /dev/sda1 /mnt
# 拷贝rootfs
$ cp -rf ./rootfs /mnt/root/rootfs
$ guestunmount /mnt


```

然后启动 qcow2 文件：

```
qemu-system-arm -M versatilepb -kernel vmlinuz-3.2.0-4-versatile -initrd initrd.img-3.2.0-4-versatile -hda debian_wheezy_armel_standard.qcow2 -append "root=/dev/sda1"


```

利用 chroot“套娃” 系统：

```
$ chroot -u ./rootfs /bin/sh

BusyBox v0.47 (2018.08.30-14:14+0000) Built-in shell
Enter 'help' for a list of built-in commands.

/ #


```

使用 debian 现成的 qcow2 镜像可以省去网卡和其他一些配置过程，提高模拟效率。  
值得注意的是，qemu 在对程序黑盒测试时也经常使用，比如 AFL 的 qemu_mode，在后面的篇章里我们会探讨，当然 AFL 同样可以使用 unicorn_mode，使用的就是下面介绍的模拟器 unicorn。

#### 1.4.2 Unicorn

[Unicorn](https://www.unicorn-engine.org/) 一个基于 Qemu 的 cpu 指令模拟框架，也就是可以模拟任何的 cpu 指令。我们一般可以利用 Unicorn 指令级模拟特性：

> 对 (IOT) 程序作模糊测试  
> 用于 gdb 插件，或代码模拟执行的插桩，修改代码逻辑  
> 模拟执行一些复杂混淆代码，提高人工逆向效率

关于 Unicorn 模拟执行修改代码逻辑的教程比较多，这里不再赘述，下面介绍一款基于 Unicorn 的 [IDA pro 插件](https://github.com/H4lo/IDA_MIPS_EMU)，该插件可以在 IDA pro 中模拟执行，并给出执行结果，UnicornFuzz 相关内容会在后面的篇章中介绍。

```
#include <stdlib.h>

int calc(int a,int b){
        int sum;
        sum = a+b;
        return sum;

}

int main(){
        calc(2,3);
}


```

上面非常简单的计算器 C 代码是该插件的官方示例，其 mipsel IDA pro 反汇编代码如下：

```
.text:00400640                 .globl calc
.text:00400640 calc:                                    # CODE XREF: main+18↓p
.text:00400640
.text:00400640 var_10          = -0x10
.text:00400640 var_4           = -4
.text:00400640 arg_0           =  0
.text:00400640 arg_4           =  4
.text:00400640
.text:00400640                 addiu   $sp, -0x18
.text:00400644                 sw      $fp, 0x18+var_4($sp)
.text:00400648                 move    $fp, $sp
.text:0040064C                 sw      $a0, 0x18+arg_0($fp)
.text:00400650                 sw      $a1, 0x18+arg_4($fp)
.text:00400654                 lw      $v1, 0x18+arg_0($fp)
.text:00400658                 lw      $v0, 0x18+arg_4($fp)
.text:0040065C                 addu    $v0, $v1, $v0
.text:00400660                 sw      $v0, 0x18+var_10($fp)
.text:00400664                 lw      $v0, 0x18+var_10($fp)
.text:00400668                 move    $sp, $fp
.text:0040066C                 lw      $fp, 0x18+var_4($sp)
.text:00400670                 addiu   $sp, 0x18
.text:00400674                 jr      $ra
.text:00400678                 nop
.text:00400678  # End of function calc


```

创建一个 emu object

```
Python>a = EmuMips()


```

配置模拟地址和参数

```
Python>a.configEmu(0x00400640,0x00400678,[2,3])
[*] Init registers success...
[*] Init code and data segment success! 
[*] Init Stack success...
[*] set args...


```

开始指令模拟

```
Python>a.beginEmu()
[*] emulating...
[*] Done! Emulate result return: 0x5


```

通过 Unicorn 模拟执行得到 sum(2+3) 的结果为 0x5。

#### 1.4.3 Qiling

[Qiling](https://www.qiling.io/) 是一款很年轻的基于 Unicorn 的模拟器，专为 IOT 研究而生。Qiling 可以作为 IDA pro 插件，也可以利用 Qiling Unicornalf 进行 fuzz，还可以模拟 IOT 设备固件。Qiling 用 python3 开发，可以直接用`pip3 install qiling`安装，以下是官方模拟 ARMj 架构路由器固件的部分代码：

```
import os, socket, sys, threading
sys.path.append("..")
from qiling import *

def patcher(ql):
    ...

def nvram_listener():
    ...

def my_sandbox(path, rootfs):
    ql = Qiling(path, rootfs, output = "debug")
    ql.add_fs_mapper("/dev/urandom","/dev/urandom")
    ql.hook_address(patcher ,ql.loader.elf_entry)
    ql.run()

if __name__ == "__main__":
    nvram_listener_therad =  threading.Thread(target=nvram_listener, daemon=True)
    nvram_listener_therad.start()
    my_sandbox(["rootfs/bin/httpd"], "rootfs")


```

可以看到 Qiling 框架进一步简化了模拟所需代码，同时提供了指令级插桩功能，还是很强大的。

#### 1.4.4 Firmadyne

[Firmadyne](https://github.com/firmadyne/firmadyne) 是一个自动化和可扩展的系统，用于对基于 Linux 的嵌入式固件执行仿真和动态分析。Firmadyne 也是是基于 Qemu，使用起来大同小异，网上教程也很多，这里主要介绍一款使用 Firmadyne 的固件灰盒 fuzz 工具的 --[Firm-AFL](https://github.com/zyw-200/FirmAFL)。  
![](https://image.3001.net/images/20201108/1604819857_5fa79b91ac2672ac9a7c4.png!small)Firm-AFL 的安装分为用户模式和系统模式，  
用户模式：

```
$ cd user_mode/
$ ./configure --target-list=mipsel-linux-user,mips-linux-user,arm-linux-user --static --disable-werror
$ make


```

系统模式：

```
$ cd qemu_mode/DECAF_qemu_2.10/
$ ./configure --target-list=mipsel-softmmu,mips-softmmu,arm-softmmu --disable-werror
$ make


```

以 Dlink DIR-815 固件为例，首先安装好 Firmadyne，做一些初始化工作：

```
$ cd firmadyne
$ ./sources/extractor/extractor.py -b dlink -sql 127.0.0.1 -np -nk "../firmware/DIR-815_FIRMWARE_1.01.ZIP" images
$ ./scripts/getArch.sh ./images/9050.tar.gz
$ ./scripts/makeImage.sh 9050
$ ./scripts/inferNetwork.sh 9050
$ cd ..
$ python FirmAFL_setup.py 9050 mipsel


```

根据路由器架构修改 image_9050 目录中的 run.sh：

```
ARCH=mipsel
QEMU="./qemu-system-${ARCH}"
KERNEL="./vmlinux.${ARCH}_3.2.1" 
IMAGE="./image.raw"
MEM_FILE="./mem_file"
${QEMU} -m 256 -mem-prealloc -mem-path ${MEM_FILE} -M ${QEMU_MACHINE} -kernel ${KERNEL} \ 


```

然后就可以启动 fuzz 脚本开始测试：

```
$ cd image_9050
$ sudo python start.py 9050


```

### 1.5 总结

固件研究是 IOT 漏洞研究的基础，本篇尽可能全的介绍了固件研究方面的内容，一方面笔者经验有限，还有些点未能涉及，另一方面篇幅有限，许多内容没能进一步展开，以后有机会再和大家一起探讨。下一篇将介绍 IOT web 漏洞研究的相关内容。

参考资料
----

> https://cloud.tencent.com/developer/article/1005700  
> https://5alt.me/2017/08 / 某智能设备固件解密 /  
> http://blog.nsfocus.net/hmi-firmware-decryption-0522/  
> https://www.ershicimi.com/p/8e818120ac6352368837ef614dd496e4  
> http://blog.nsfocus.net/hmi-firmware-decryption-0522/  
> https://gorgias.me/2019/12/27 / 固件提取系列 - UBI 文件系统提取以及重打包 /  
> https://paper.seebug.org/771/  
> https://paper.seebug.org/1090/  
> https://blog.csdn.net/yalecaltech/article/details/104113779  
> https://dassecurity-labs.github.io/HatLab_IOT_Wiki/firmware_security/firmware_security_tools/IDA_unicorn_base_tool/
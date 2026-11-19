> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.kanxue.com](https://bbs.kanxue.com/thread-293601.htm)

> 看雪安全社区是一个非营利性质的技术交流平台，致力于汇聚全球的安全研究者和开发者，专注于软件与系统安全、逆向工程、漏洞研究等领域的深度技术讨论与合作。

> 基于 **HarmonyOS 7.0.0 (API 20) + macOS Apple Silicon + DevEco Studio** 实测成功

*   `hdc shell id` → **uid=0(root) gid=0(root)**，groups 含 `user_data_rw`
*   SELinux → **Permissive**
*   `/data/app` 可访问，可提取已安装应用的 HAP
*   无 "非法来源镜像" 红字警告

<table><thead><tr><th>项</th><th>值</th></tr></thead><tbody><tr><td>system.img 路径</td><td><code>~/Library/Huawei/Sdk/system-image/HarmonyOS-7.0.0/phone_all_arm/system.img</code></td></tr><tr><td>实例目录</td><td><code>~/.Huawei/Emulator/deployed/Mate 90 Pro/</code></td></tr><tr><td>ohos.para 块号</td><td>398894</td></tr><tr><td>hdc.para 块号</td><td>398853</td></tr><tr><td>hdcd.cfg 块号 / 大小</td><td>396814-396815 / 5961B → 6307B</td></tr><tr><td>selinux config 块号</td><td>399489</td></tr><tr><td>hdc 路径</td><td><code>~/Library/OpenHarmony/Sdk/20/toolchains/hdc</code></td></tr></tbody></table>

> ⚠️ 若 DevEco 版本不同，镜像内容可能变化，动手前仍建议用第 3 步命令重新核对块号。

```
mkdir -p ~/emulator_root_backup
cp /Applications/DevEco-Studio.app/Contents/tools/emulator/Emulator ~/emulator_root_backup/Emulator.bak
cp ~/Library/Huawei/Sdk/system-image/HarmonyOS-7.0.0/phone_all_arm/system.img ~/emulator_root_backup/Mate90Pro.system.img.bak
cp ~/.Huawei/Emulator/deployed/Mate\ 90\ Pro/ram.bin ~/emulator_root_backup/Mate90Pro.ram.bin.bak 2>/dev/null


```

```
brew install qemu


```

```
cd ~/emulator_root_work   
curl -LO https://mirrors.edge.kernel.org/pub/linux/kernel/people/tytso/e2fsprogs/v1.47.4/e2fsprogs-1.47.4.tar.xz
tar xf e2fsprogs-1.47.4.tar.xz && cd e2fsprogs-1.47.4
./configure --disable-nls


```

**编译坑 1**：新版 Apple `ar` 对 64 位目标文件非 8 字节对齐，链接报错。写 ar 包装脚本：

```
cat > /tmp/myar << 'EOF'

if [ "$1" = "rc" ]; then
  shift
  exec /usr/bin/libtool -static -o "$@"
elif [ "$1" = "r" ]; then
  shift
  [ -f "$1" ] && exec /usr/bin/ranlib "$1"
  exit 0
else
  exec /usr/bin/ar "$@"
fi
EOF
chmod +x /tmp/myar


```

**编译坑 2**：PATH 里若装了 homebrew binutils，其 GNU ranlib 会损坏 Apple 格式归档，必须显式指定：

```
find lib -name "*.a" -delete
make AR=/tmp/myar RANLIB=/usr/bin/ranlib -j8


```

产物：`debugfs/debugfs` 和 `e2fsck/e2fsck`。

Emulator 二进制符号未剥离，直接查：

```
EMU=/Applications/DevEco-Studio.app/Contents/tools/emulator/Emulator
nm -C "$EMU" | grep -iE "VerifyReleaseCms|IsIllegalRoot"


```

实测地址（vmaddr）：

*   `CheckSign::VerifyReleaseCmsFile` → `0x100d9e14c`
*   `EmulatorWindowEvent::IsIllegalRoot()` → `0x100c57110`

**文件偏移 = vmaddr − 0x100000000**（`__TEXT` 段 vmaddr 基址，fileoff 为 0）：

*   VerifyReleaseCmsFile → fileoff `0xd9e14c`
*   IsIllegalRoot → fileoff `0xc57110`

```
objdump --start-address=0x100d9e14c --stop-address=0x100d9e164 -d "$EMU"


objdump --start-address=0x100c57110 --stop-address=0x100c57118 -d "$EMU"



```

```
import struct
EMU = '/Applications/DevEco-Studio.app/Contents/tools/emulator/Emulator'


off1 = 0xd9e14c
with open(EMU, 'r+b') as f:
    f.seek(off1)
    assert f.read(8) == bytes.fromhex('ff4303d1f6570aa9'), 'VerifyReleaseCmsFile 字节不匹配'
    f.seek(off1)
    f.write(struct.pack('<II', 0x52800000, 0xD65F03C0))


off2 = 0xc57110
with open(EMU, 'r+b') as f:
    f.seek(off2)
    assert f.read(4) == bytes.fromhex('ff0302d1'), 'IsIllegalRoot 字节不匹配'
    f.seek(off2)
    f.write(struct.pack('<I', 0xD65F03C0))
print('patched OK')


```

```
python3 patch_emulator.py


```

```
EMU=/Applications/DevEco-Studio.app/Contents/tools/emulator/Emulator
codesign -d --entitlements :- "$EMU" > /tmp/emu_ent.plist
xattr -c "$EMU"
codesign --force --sign - --entitlements /tmp/emu_ent.plist "$EMU"
codesign -v "$EMU"   


```

```
cd ~/emulator_root_work/e2fsprogs-1.47.4   
DEBUGFS=./debugfs/debugfs
SYSIMG=~/Library/Huawei/Sdk/system-image/HarmonyOS-7.0.0/phone_all_arm/system.img

$DEBUGFS -R "blocks /system/etc/param/ohos.para" "$SYSIMG"      
$DEBUGFS -R "blocks /system/etc/param/hdc.para" "$SYSIMG"       
$DEBUGFS -R "blocks /system/etc/init/hdcd.cfg" "$SYSIMG"        
$DEBUGFS -R "blocks /system/etc/selinux/config" "$SYSIMG"       


```

先看一眼原内容，确认格式：

```
$DEBUGFS -R "cat /system/etc/param/ohos.para" "$SYSIMG" | grep -E "const.secure|debuggable"
$DEBUGFS -R "cat /system/etc/param/hdc.para" "$SYSIMG" | grep hdc.mode
$DEBUGFS -R "dump /system/etc/init/hdcd.cfg /tmp/hdcd.image.cfg" "$SYSIMG"
$DEBUGFS -R "cat /system/etc/selinux/config" "$SYSIMG"


```

> 技术要点：系统文件带 SELinux xattr，debugfs 重写文件会丢上下文导致开不了机，  
> 所以全部走 **按块号 × 4096 定位 + 等长字节替换**（变长文件用块尾 padding + 扩 inode）。

```
BS = 4096
SYSIMG = '/Users/deadbeef/Library/Huawei/Sdk/system-image/HarmonyOS-7.0.0/phone_all_arm/system.img'

with open(SYSIMG, 'r+b') as f:
    
    f.seek(398894 * BS)
    data = f.read(BS)
    for old, new in [(b'const.secure=1', b'const.secure=0'),
                     (b'const.debuggable=0', b'const.debuggable=1')]:
        i = data.find(old)
        assert i >= 0, f'{old} not found'
        data = data[:i] + new + data[i+len(old):]
    assert len(data) == BS
    f.seek(398894 * BS)
    f.write(data)

    
    f.seek(398853 * BS)
    data = f.read(BS)
    old = b'persist.hdc.mode.usb = "disable"'
    new = b'persist.hdc.mode.usb  = "enable"'   
    assert len(old) == len(new)
    i = data.find(old)
    assert i >= 0, 'hdc.mode not found'
    data = data[:i] + new + data[i+len(old):]
    assert len(data) == BS
    f.seek(398853 * BS)
    f.write(data)

print('para patched OK')


```

```
python3 patch_para.py


```

```
import json
BS = 4096
SYSIMG = '/Users/deadbeef/Library/Huawei/Sdk/system-image/HarmonyOS-7.0.0/phone_all_arm/system.img'
BLOCKS = [396814, 396815]   


import subprocess
raw = subprocess.check_output(
    ['./debugfs/debugfs', '-R', 'cat /system/etc/init/hdcd.cfg', SYSIMG])
cfg = json.loads(raw)


for job in cfg.get('jobs', []):
    for i, c in enumerate(job.get('cmds', [])):
        if c == 'setparam const.hdc.secure 1':
            job['cmds'][i] = 'setparam const.hdc.secure 0'
    if 'const.secure=1' in job.get('name', ''):
        job['name'] = job['name'].replace('const.secure=1', 'const.secure=0')
    if 'const.secure=1' in job.get('condition', ''):
        job['condition'] = job['condition'].replace('const.secure=1', 'const.secure=0')


svc = [s for s in cfg['services'] if s.get('name') == 'hdcd'][0]
svc['uid'] = 'root'
svc['gid'] = ['root', 'shell', 'log', 'readproc', 'file_manager', 'user_data_rw']
svc['setuid'] = True



new = json.dumps(cfg, indent=4, ensure_ascii=False).encode()
print('new size:', len(new))       
assert len(new) <= BS * len(BLOCKS)
padded = new + b'\x00' * (BS * len(BLOCKS) - len(new))
with open(SYSIMG, 'r+b') as f:
    for bi, b in enumerate(BLOCKS):
        f.seek(b * BS)
        f.write(padded[bi*BS:(bi+1)*BS])
print('hdcd.cfg patched OK')


```

```
python3 patch_hdcd.py

$DEBUGFS -w -R "sif /system/etc/init/hdcd.cfg size 6307" "$SYSIMG"

$DEBUGFS -R "cat /system/etc/init/hdcd.cfg" "$SYSIMG" | python3 -c "import json,sys; c=json.load(sys.stdin); print([s['uid'] for s in c['services'] if s['name']=='hdcd'])"


```

```
BS = 4096
SYSIMG = '/Users/deadbeef/Library/Huawei/Sdk/system-image/HarmonyOS-7.0.0/phone_all_arm/system.img'
BLOCK = 399489

with open(SYSIMG, 'r+b') as f:
    f.seek(BLOCK * BS)
    data = f.read(BS)
    old_seg = b'#\n\nSELINUX=enforcing\n'
    new_seg = b'#\nSELINUX=permissive\n'   
    assert len(old_seg) == len(new_seg)
    i = data.find(old_seg)
    assert i >= 0, 'SELINUX=enforcing not found'
    data = data[:i] + new_seg + data[i+len(old_seg):]
    assert len(data) == BS
    f.seek(BLOCK * BS)
    f.write(data)
print('selinux patched OK')


```

```
python3 patch_selinux.py


```

```
./e2fsck/e2fsck -fn "$SYSIMG"



```

> ⚠️ **ram.bin 陷阱**：实例目录里的 `ram.bin` 是内存快照，模拟器会**从快照恢复**  
> 而不是冷启动——镜像改对了但运行系统仍是旧状态（Enforcing、hdcd shell 用户），  
> 极有迷惑性。patch 后首次启动**必须删除 ram.bin** 强制冷启动。

```
rm -f ~/.Huawei/Emulator/deployed/Mate\ 90\ Pro/ram.bin \
      ~/.Huawei/Emulator/deployed/Mate\ 90\ Pro/system.img.qcow2


/Applications/DevEco-Studio.app/Contents/tools/emulator/Emulator \
    -start "Mate 90 Pro" -instancePath ~/.Huawei/Emulator/deployed


```

也可以直接在 DevEco Studio 图形界面里启动，效果相同。

```
HDC=~/Library/OpenHarmony/Sdk/20/toolchains/hdc
T=127.0.0.1:5555

$HDC -t $T shell id        
$HDC -t $T shell getenforce   
$HDC -t $T shell "ls /data/app/el1"   
$HDC -t $T shell "ps -ef | grep hdcd | grep -v grep"   


```

```
$HDC -t $T shell "find /data/app/el1 -name '*.hap' | head -3"
$HDC -t $T file recv /data/app/el1/.../entry.hap ./extracted.hap




```

<table><thead><tr><th>目标</th><th>修改内容</th><th>目的</th></tr></thead><tbody><tr><td>Emulator 二进制 @0xd9e14c</td><td><code>VerifyReleaseCmsFile</code> → <code>mov w0,#0; ret</code></td><td>跳过镜像完整性校验</td></tr><tr><td>Emulator 二进制 @0xc57110</td><td><code>IsIllegalRoot()</code> → <code>ret</code></td><td>去掉 "非法来源镜像" 红字（宿主侧覆盖层）</td></tr><tr><td>ohos.para（块 398894）</td><td><code>const.secure=0</code>、<code>const.debuggable=1</code></td><td>解锁调试态</td></tr><tr><td>hdc.para（块 398853）</td><td><code>persist.hdc.mode.usb = "enable"</code></td><td>hdc USB 模式</td></tr><tr><td>hdcd.cfg（块 396814-15）</td><td>secure job 置 0；service <code>uid: root</code>、<code>setuid: true</code>、gid 加 root/user_data_rw；secon 保持 <code>u:r:hdcd:s0</code></td><td><strong>hdcd 以 root 运行（核心）</strong></td></tr><tr><td>selinux config（块 399489）</td><td><code>SELINUX=enforcing</code> → <code>permissive</code></td><td>放宽 SELinux</td></tr></tbody></table>

1.  **ram.bin 快照陷阱（7.0.0 实测最重要）**：镜像内容验证正确但系统行为是旧的 → 十有八九是快照恢复，删 `ram.bin` + `system.img.qcow2` 冷启动
2.  **每次重启前删除 `system.img.qcow2`**，让模拟器从补丁后的 system.img 重建 overlay
3.  **secon 不能照搬 `u:r:su:s0`**：模拟器 sepolicy 无 su 域，hdcd 会起不来
4.  **`setenforce` 被挡**：sh 域策略限制，只能改 selinux `config` 文件；往 cil 追加 `(typepermissive sh)` 实测无效
5.  **`param get` 报错 ≠ 参数不存在**：可能是 SELinux 拒读的假阴性，以 `hdc shell id` 为准
6.  **必须先核对原始字节再 patch**：DevEco 版本不同，函数地址 / 字节可能不同，`nm` + `objdump` 重新定位
7.  **自定义 qcow2 会被模拟器重建**：不要绕道改 overlay，必须 patch 二进制 + 直改 system.img

```
cp ~/emulator_root_backup/Emulator.bak /Applications/DevEco-Studio.app/Contents/tools/emulator/Emulator
cp ~/emulator_root_backup/Mate90Pro.system.img.bak ~/Library/Huawei/Sdk/system-image/HarmonyOS-7.0.0/phone_all_arm/system.img
rm -f ~/.Huawei/Emulator/deployed/Mate\ 90\ Pro/system.img.qcow2 \
      ~/.Huawei/Emulator/deployed/Mate\ 90\ Pro/ram.bin



```

[冰与火的战歌：Windows 内核攻防实战高级班！从零到实战，融合 AI 与 Windows 内核攻防全技术栈，打造具备自动化能力的内核开发高手。](https://www.kanxue.com/book-section_list-227.htm)
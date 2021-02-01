> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-265792.htm)

最近刷了很多机型的 Magisk，经历了 N 次变砖和修复，也有了一些经验，Magisk 提供了三种方式刷入 Magisk，分别是：

1.  具有 root 权限情况下直接 manager 中直接安装
2.  在能获取到 boot.img 的情况下，patch boot.img 然后 fastboot 刷入
3.  在三方 recovery 中直接刷入 magisk.zip

这三种安装方式教程很多，主要介绍一下直接安装或者 patch boot.img 刷完 magisk 变砖之后遇到的分区验证导致变砖问题以及如何解决。

 

从刷入 Magisk 的方式都需要操作 boot.img 也可以看出来，magisk 的核心原理就是修改和替换了负责安卓系统启动的 boot.img 中的 Ramdisk 部分，该映像包括了一系列初始化 init 进程和启动时使用的 rc 文件。

 

如果直接能拿到 Root 权限，可以考虑直接安装或者选择并修补 boot.img 的方式刷入，两种方式的本质相同，均是获取到 boot.img 后进行 patch，只不过实现方式不同，一种依赖 Fastboot 进行刷入，一种则是 Magisk Manager 中进行刷入。

 

获取 boot.img 的方式有两种：

1.  从官方的 ROM 中提取，不同 ROM 的打包方式不同，例如一加是 payload.bin，有专门的脚本提取，而 google 原生就是一个 zip 包解压即可，再例如三星可能是 AP 开头的 tar 文件。
2.  如果有 root 权限，直接从设备中提取，/dev/block/by-name 下根据软连接找到 boot.img，然后使用 dd 命令将 img 写出。如果是 A/B 分区，一般采用 boot_a.img, 但为了防止变砖后无法还原，建议用这种方式将 recovery.img boot.img vbmeta.img 都备份一份。
    
    ```
    ls -l /dev/block/by-name/ | grep boot
    dd if=/dev/block/sdc6 of=/sdcard/boot.img
    
    ```
    

如果在刷入 Magisk 时变砖无法开机或着无限重启，通常只需在 fastboot 中将正常的 boot.img 重新刷回即可，也是为什么要提前备份 boot.img 等的原因。

avb2.0 验证导致手机无法启动
=================

在部分机型中，可能由于 vbmeta.img 的验证导致设备无法启动，验证启动（Verified Boot）是 Android 一个重要的安全功能，主要是为了访问启动镜像被篡改，提高系统的抗攻击能力，简单描述做法就是在启动过程中增加一条校验链，即 ROM code 校验 BootLoader，确保 BootLoader 的合法性和完整性，BootLoader 则需要校验 boot image，确保 Kernel 启动所需 image 的合法性和完整性，而 Kernel 则负责校验 System 分区和 vendor 分区。

 

由于 ROM code 和 BootLoader 通常都是由设备厂商 OEM 提供，而各家实际做法和研发能力不尽相同，为了让设备厂商更方便的引入 Verified boot 功能，Google 在 Android O 上推出了一个统一的验证启动框架 Android verified boot 2.0，好处是既保证了基于该框架开发的 verified boot 功能能够满足 CDD 要求，也保留了各家 OEM 定制启动校验流程的弹性。

Fastboot 解决方案
-------------

简单的说，就是部分厂商机型可能由于 avb2.0 验证 boot.img 是否被修改，导致刷入 magisk 或者三方 Recovery 后陷入假变砖无限重启的情况，此时将备份的 vbmeta.img 重新刷入并关闭验证即可：

 

fastboot --disable-verity --disable-verification flash vbmeta vbmeta.img

 

这种方式简单易用，适合个人玩家，但如果是用在生产中，成千台设备需要批量刷入 magisk 时，这种方式太过人工，因此还可以通过定制 Magisk 代码的方式来一键刷入 Magisk。

Magisk 源代码定制解决方案
----------------

来看一下 Fastboot 的原理，查阅[上一步中 FastBoot 的源码](https://android.googlesource.com/platform/system/core/+/master/fastboot/fastboot.cpp)可以看到：

```
// There's a 32-bit big endian |flags| field at offset 120 where
    // bit 0 corresponds to disable-verity and bit 1 corresponds to
    // disable-verification.
    //
    // See external/avb/libavb/avb_vbmeta_image.h for the layout of
    // the VBMeta struct.
    uint64_t flags_offset = 123 + vbmeta_offset;
    if (g_disable_verity) {
        data[flags_offset] |= 0x01;
    }
    if (g_disable_verification) {
        data[flags_offset] |= 0x02;
    }

```

实际上只是对 vbmeta.img 中 120 偏移处进行了两个 bit 位的处理，即 vbmeta.img 中是否关闭验证有两个 flag 可以控制，我们只需依葫芦画瓢:

1.  从手机提取原始 vbmeta.img（参考 Magisk 直接安装时寻找 boot.img 的操作）
2.  修改 120 偏移处的 flag (参考 Magisk 对三星 AP 文件中的 vbmeta.img 修复操作)
3.  重新刷回系统中

### Magisk 如何在非 Fastboot 下刷入 boot.img

这三步中比较困难的就是第三步，将 patch 完的 vbmeta.img 重新刷回系统中，因此这里参考了 Magisk 在有 root 权限时 patch 完 boot.img 之后直接刷入 new_boot.img 的操作。

```
// MagiskInstaller.kt 直接安装的入口
protected fun direct() = findImage() && extractZip() && patchBoot() && flashBoot()
 
private fun flashBoot(): Boolean {
        if (!"direct_install $installDir $srcBoot".sh().isSuccess)
            return false
        "run_migrations".sh()
        return true
    }
 
// shell脚本
direct_install() {
  rm -rf $MAGISKBIN/* 2>/dev/null
  mkdir -p $MAGISKBIN 2>/dev/null
  chmod 700 $NVBASE
  cp -af $1/. $MAGISKBIN
  rm -f $MAGISKBIN/new-boot.img
  echo "- Flashing new boot image"
  flash_image $1/new-boot.img $2
  if [ $? -ne 0 ]; then
    echo "! Insufficient partition size"
    return 1
  fi
  rm -rf $1
  return 0
}
 
# $1参数是patch后的new-boot.img路径，$2参数是设备系统中boot.img路径
flash_image() {
  # Make sure all blocks are writable
  $MAGISKBIN/magisk --unlock-blocks 2>/dev/null
  case "$1" in
    *.gz) CMD1="$MAGISKBIN/magiskboot decompress '$1' - 2>/dev/null";;
    *)    CMD1="cat '$1'";;
  esac
  if $BOOTSIGNED; then
    CMD2="$BOOTSIGNER -sign"
    ui_print "- Sign image with verity keys"
  else
    CMD2="cat -"
  fi
  if [ -b "$2" ]; then
    local img_sz=`stat -c '%s' "$1"`
    local blk_sz=`blockdev --getsize64 "$2"`
    [ $img_sz -gt $blk_sz ] && return 1
    eval $CMD1 | eval $CMD2 | cat - /dev/zero > "$2" 2>/dev/null
  elif [ -c "$2" ]; then
    flash_eraseall "$2" >&2
    eval $CMD1 | eval $CMD2 | nandwrite -p "$2" - >&2
  else
    ui_print "- Not block or char device, storing image"
    eval $CMD1 | eval $CMD2 > "$2" 2>/dev/null
  fi
  return 0
}

```

看起来很复杂，核心写入的步骤其实就是 eval $CMD1 | eval $CMD2 | cat - /dev/zero > "$2" 2>/dev/null，通过这行命令就可以实现不在 Fastboot 中通过 shell 命令完成 boot.img 和 vbmeta.img 的写入（也许也可以类推到其他 img 映像分区）

### 实现 vbmeta.img 的重写

而且 vbmeta.img 相比于 boot.img 比较简单，不需要签名、验证等步骤，因此可以简单提炼为:

```
# 先找到vbmeta.img
vbmeta_block_name=`find /dev/block \( -type b -o -type c -o -type l \) -iname vbmeta | head -n 1`
vbmeta_block=`readlink -f $vbmeta_block_name`
# 将patch好的vbmeta.img写入
eval 'cat '/sdcard/patched_vbmeta.img'' | eval 'cat -' | cat - /dev/zero > $vbmeta_block 2>/dev/null

```

再加上 patch 操作，完整代码为：

```
fun disableVbMetaImg() {
        try {
            // find vbmeta.img block
            val vbmetaPath = ShellUtils.fastCmd("readlink -f `find /dev/block \\( -type b -o -type c -o -type l \\) -iname vbmeta | head -n 1`")
            if (TextUtils.isEmpty(vbmetaPath)) {
                Log.w(Const.TAG, "could not find vbmeta.img!")
                return
            }
            val patchedVbmetaPath = "/sdcard/patched_vbmeta.img"
            if (!Shell.sh("dd if=$vbmetaPath of=$patchedVbmetaPath").exec().isSuccess) {
                Log.w(Const.TAG, "extract vbmeta.img failed!")
                return
            }
            val patchedVbMeta = SuFile.open(patchedVbmetaPath).readBytes()
            // There's a 32-bit big endian |flags| field at offset 120 where
            // bit 0 corresponds to disable-verity and bit 1 corresponds to
            // disable-verification.
            ByteBuffer.wrap(patchedVbMeta).putInt(120, 2)
            File(patchedVbmetaPath).writeBytes(patchedVbMeta)
            // rewrite vbmeta.img
            Log.d(Const.TAG, "rewrite vbmeta.img finished: $patchedVbmetaPath to $vbmetaPath")
            Shell.sh("eval 'cat '$patchedVbmetaPath'' | eval 'cat -' | cat - /dev/zero > $vbmetaPath 2>/dev/null").exec()
        } catch (e: Throwable) {
            Log.i(Const.TAG, "disable vb_meta.img avb2.0 error:", e)
        }
    }

```

[看雪侠者千人榜，看看你上榜了吗？](https://www.kanxue.com/rank-2.htm)

最后于 16 分钟前 被 alienhe 编辑 ，原因：
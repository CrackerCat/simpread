> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_38384263/article/details/130984430?spm=1001.2014.3001.5502)

#### 使用 [LLVM](https://so.csdn.net/so/search?q=LLVM&spm=1001.2101.3001.7020) 编译 ARM64 的 LINUX 内核和可执行程序

*   [使用 LLVM 编译 ARM64 的优点](#LLVMARM64_2)
*   [需要准备的工具](#_6)
*   *   [下载源码](#_8)
    *   [编译 binutils](#binutils_10)
    *   [编译带 lto 功能的 llvm](#ltollvm_26)
*   [使用 LLVM 编译 ARM64 内核](#LLVMARM64_41)
*   *   [内核相关修改 patch](#patch_43)
    *   [编译内核](#_327)
*   [使用 LLVM 编译 ARM64 可执行程序](#LLVMARM64_337)

使用 LLVM 编译 [ARM64](https://so.csdn.net/so/search?q=ARM64&spm=1001.2101.3001.7020) 的优点
-------------------------------------------------------------------------------------

最近研究了下怎么使用 LLVM 编译 ARM64 的 [linux 内核](https://so.csdn.net/so/search?q=linux%E5%86%85%E6%A0%B8&spm=1001.2101.3001.7020)，发现编译速度快了 62%，本来 2 分多编译完的内核，使用 LLVM 编译后，只要 1 分多点。而且 gcc 编译出的内核大小是 14.3MB，LLVM 编出的内核是 12.9MB，大小也小了点。在内核这里可以看出，编译速度快，文件大小也会更小点。

需要准备的工具
-------

使用 LLVM 编译的话，要准备的当然是 llvm 了。llvm 是支持 lto 功能的，这里我选择编译带 lto 功能的 llvm。

### 下载源码

首先需要下载 llvm 的源码 [llvm](https://github.com/llvm/llvm-project/releases/)。llvm 里 lto 功能是需要 llvm-gold 插件，而这个插件的编译是需要 binutils 模块，所有还需要下载 [binutils 源码](http://mirrors.ustc.edu.cn/gnu/binutils/)。

### 编译 binutils

```
#解压
tar -xvf binutils-2.40.tar.xz  
cd binutils-2.40
mkdir build
cd build
#这里$(BINUTILS_INSTALL_DIR)改为自己想安装的路径，CC和CXX指定为自己的GCC和G++路径即可
../configure --enable-gold --enable-plugins --disable-werror --prefix=$(BINUTILS_INSTALL_DIR) CC=gcc CXX=g++
#编译
make all-gold -j32
make -j32
#安装到$(BINUTILS_INSTALL_DIR)目录
make install

```

### 编译带 lto 功能的 llvm

```
#解压
tar -xvf llvm-project-16.0.4.src.tar.xz
cd llvm-project-16.0.4.src
mkdir build
cd build
#$(LLVM_INSTALL_DIR)改为自己想安装的路径，CC和CXX指定为自己的GCC和G++路径即可。
cmake -G "Unix Makefiles" -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra;lld" -DCMAKE_INSTALL_PREFIX=$(LLVM_INSTALL_DIR) -DCMAKE_BUILD_TYPE=Release ../llvm -DCMAKE_CXX_COMPILER=g++ -DCMAKE_C_COMPILER=gcc -DLLVM_BINUTILS_INCDIR=$(BINUTILS_INSTALL_DIR)/include
#编译
make -j32
#安装到$(LLVM_INSTALL_DIR)目录
make install

```

使用 LLVM 编译 ARM64 内核
-------------------

其实 linux 官方在 linux5.10 以后的内核原生就支持了 LLVM 编译了，但目前我在用的内核是 4.19.90 的内核，想支持 LLVM 编译的话需要响应的修改。这里提供下 4.19.90 使用 LLVM 编译 ARM64 的修改 PATCH。如果是要编译 5.10 以前的内核，可以参考修改下。

### 内核相关修改 patch

```
From fd96b1fd201f73ed65b01017b775c470b5c4d0df Mon Sep 17 00:00:00 2001
From: =?UTF-8?q?=E9=83=91=E6=96=87=E5=B3=B0?=
 <“zheng_wenfeng@dahuatech.com”>
Date: Wed, 24 May 2023 16:24:39 +0800
Subject: [PATCH] =?UTF-8?q?LLVM=E7=BC=96=E8=AF=91=E9=80=9A=E8=BF=87?=
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit

---
 Trunk/linux-4.19.90/Makefile                  |   42 +-
 .../arch/arm64/include/asm/alternative.h      |    2 +-
 .../arch/arm64/kernel/module.lds              |    2 +-
 .../arch/arm64/kernel/vdso/gettimeofday.S     |    8 +-
 .../arch/arm64/kernel/vdso/note.S             |    4 +-
 Trunk/linux-4.19.90/arch/arm64/lib/memcpy.S   |    3 +-
 Trunk/linux-4.19.90/arch/arm64/lib/memmove.S  |    3 +-
 Trunk/linux-4.19.90/arch/arm64/lib/memset.S   |    3 +-
 Trunk/linux-4.19.90/drivers/Makefile          |    2 +-
 Trunk/linux-4.19.90/include/linux/elfnote.h   |    8 +
 Trunk/linux-4.19.90/include/linux/poll.h      |    4 +
 Trunk/linux-4.19.90/lib/string.c              |   24 +
 Trunk/linux-4.19.90/make.sh                   |   20 +
 Trunk/linux-4.19.90/scripts/dtc/dtc-lexer.l   |    1 -
 16 files changed, 4802 insertions(+), 28 deletions(-)
 create mode 100755 Trunk/linux-4.19.90/make.sh

diff --git a/Trunk/linux-4.19.90/Makefile b/Trunk/linux-4.19.90/Makefile
index 7a1e7a89..b37068cd 100644
--- a/Trunk/linux-4.19.90/Makefile
+++ b/Trunk/linux-4.19.90/Makefile
@@ -359,8 +359,13 @@ HOST_LFS_CFLAGS := $(shell getconf LFS_CFLAGS 2>/dev/null)
 HOST_LFS_LDFLAGS := $(shell getconf LFS_LDFLAGS 2>/dev/null)
 HOST_LFS_LIBS := $(shell getconf LFS_LIBS 2>/dev/null)
 
+ifneq ($(LLVM),)
+HOSTCC       = clang
+HOSTCXX      = clang++
+else
 HOSTCC       = gcc
 HOSTCXX      = g++
+endif
 KBUILD_HOSTCFLAGS   := -Wall -Wmissing-prototypes -Wstrict-prototypes -O2 \
 		-fomit-frame-pointer -std=gnu89 $(HOST_LFS_CFLAGS) \
 		$(HOSTCFLAGS)
@@ -369,15 +374,26 @@ KBUILD_HOSTLDFLAGS  := $(HOST_LFS_LDFLAGS) $(HOSTLDFLAGS)
 KBUILD_HOSTLDLIBS   := $(HOST_LFS_LIBS) $(HOSTLDLIBS)
 
 # Make variables (CC, etc...)
+CPP		= $(CC) -E
+ifneq ($(LLVM),)
+CC		= clang
+LD		= ld.lld
+AR		= llvm-ar
+NM		= llvm-nm
+OBJCOPY		= llvm-objcopy
+OBJDUMP		= llvm-objdump
+READELF		= llvm-readelf
+STRIP		= llvm-strip
+else
 AS		= $(CROSS_COMPILE)as
 LD		= $(CROSS_COMPILE)ld
 CC		= $(CROSS_COMPILE)gcc
-CPP		= $(CC) -E
 AR		= $(CROSS_COMPILE)ar
 NM		= $(CROSS_COMPILE)nm
 STRIP		= $(CROSS_COMPILE)strip
 OBJCOPY		= $(CROSS_COMPILE)objcopy
 OBJDUMP		= $(CROSS_COMPILE)objdump
+endif
 LEX		= flex
 YACC		= bison
 AWK		= awk
@@ -485,20 +501,25 @@ ifneq ($(KBUILD_SRC),)
 	    $(srctree) $(objtree) $(VERSION) $(PATCHLEVEL)
 endif
 
-ifeq ($(cc-name),clang)
+ifneq ($(shell $(CC) --version 2>&1 | head -n 1 | grep clang),)
 ifneq ($(CROSS_COMPILE),)
 CLANG_FLAGS	+= --target=$(notdir $(CROSS_COMPILE:%-=%))
 GCC_TOOLCHAIN_DIR := $(dir $(shell which $(CROSS_COMPILE)elfedit))
-CLANG_FLAGS	+= --prefix=$(GCC_TOOLCHAIN_DIR)
+CLANG_FLAGS	+= --prefix=$(GCC_TOOLCHAIN_DIR)$(notdir $(CROSS_COMPILE))
 GCC_TOOLCHAIN	:= $(realpath $(GCC_TOOLCHAIN_DIR)/..)
 endif
 ifneq ($(GCC_TOOLCHAIN),)
 CLANG_FLAGS	+= --gcc-toolchain=$(GCC_TOOLCHAIN)
 endif
+ifneq ($(LLVM_IAS),1)
 CLANG_FLAGS	+= -no-integrated-as
+endif
+$(info CLANG_FLAGS=$(CLANG_FLAGS))
 CLANG_FLAGS	+= -Werror=unknown-warning-option
 KBUILD_CFLAGS	+= $(CLANG_FLAGS)
 KBUILD_AFLAGS	+= $(CLANG_FLAGS)
+$(info KBUILD_CFLAGS=$(KBUILD_CFLAGS))
+$(info KBUILD_AFLAGS=$(KBUILD_AFLAGS))
 export CLANG_FLAGS
 endif
 
@@ -701,16 +722,13 @@ stackp-flags-$(CONFIG_STACKPROTECTOR_STRONG)      := -fstack-protector-strong
 KBUILD_CFLAGS += $(stackp-flags-y)
 
 ifeq ($(cc-name),clang)
-KBUILD_CPPFLAGS += $(call cc-option,-Qunused-arguments,)
-KBUILD_CFLAGS += $(call cc-disable-warning, format-invalid-specifier)
-KBUILD_CFLAGS += $(call cc-disable-warning, gnu)
-# Quiet clang warning: comparison of unsigned expression < 0 is always false
-KBUILD_CFLAGS += $(call cc-disable-warning, tautological-compare)
+KBUILD_CPPFLAGS += -Qunused-arguments
+KBUILD_CFLAGS += -Wno-format-invalid-specifier
+KBUILD_CFLAGS += -Wno-gnu
 # CLANG uses a _MergedGlobals as optimization, but this breaks modpost, as the
 # source of a reference will be _MergedGlobals and not on of the whitelisted names.
 # See modpost pattern 2
-KBUILD_CFLAGS += $(call cc-option, -mno-global-merge,)
-KBUILD_CFLAGS += $(call cc-option, -fcatch-undefined-behavior)
+KBUILD_CFLAGS += -mno-global-merge
 else
 
 # These warnings generated too much noise in a regular build.
@@ -740,8 +758,10 @@ KBUILD_CFLAGS   += $(call cc-option, -gsplit-dwarf, -g)
 else
 KBUILD_CFLAGS	+= -g
 endif
+ifneq ($(LLVM_IAS),1)
 KBUILD_AFLAGS	+= -Wa,-gdwarf-2
 endif
+endif
 ifdef CONFIG_DEBUG_INFO_DWARF4
 KBUILD_CFLAGS	+= $(call cc-option, -gdwarf-4,)
 endif
@@ -858,7 +878,7 @@ include scripts/Makefile.ubsan
 KBUILD_CPPFLAGS += $(ARCH_CPPFLAGS) $(KCPPFLAGS)
 KBUILD_AFLAGS   += $(ARCH_AFLAGS)   $(KAFLAGS)
 KBUILD_CFLAGS   += $(ARCH_CFLAGS)   $(KCFLAGS)
-
+$(info KBUILD_CFLAGS=$(KBUILD_CFLAGS))
 # Use --build-id when available.
 LDFLAGS_BUILD_ID := $(call ld-option, --build-id)
 KBUILD_LDFLAGS_MODULE += $(LDFLAGS_BUILD_ID)
diff --git a/Trunk/linux-4.19.90/arch/arm64/include/asm/alternative.h b/Trunk/linux-4.19.90/arch/arm64/include/asm/alternative.h
index 4b650ec1..64c4425a 100644
--- a/Trunk/linux-4.19.90/arch/arm64/include/asm/alternative.h
+++ b/Trunk/linux-4.19.90/arch/arm64/include/asm/alternative.h
@@ -211,7 +211,7 @@ alternative_endif
 
 .macro user_alt, label, oldinstr, newinstr, cond
 9999:	alternative_insn "\oldinstr", "\newinstr", \cond
-	_ASM_EXTABLE 9999b, \label
+	_asm_extable 9999b, \label
 .endm
 
 /*
diff --git a/Trunk/linux-4.19.90/arch/arm64/kernel/module.lds b/Trunk/linux-4.19.90/arch/arm64/kernel/module.lds
index 22e36a21..23451385 100644
--- a/Trunk/linux-4.19.90/arch/arm64/kernel/module.lds
+++ b/Trunk/linux-4.19.90/arch/arm64/kernel/module.lds
@@ -1,5 +1,5 @@
 SECTIONS {
-	.plt (NOLOAD) : { BYTE(0) }
+	.plt : { BYTE(0) }
 	.init.plt (NOLOAD) : { BYTE(0) }
 	.text.ftrace_trampoline (NOLOAD) : { BYTE(0) }
 }
diff --git a/Trunk/linux-4.19.90/arch/arm64/kernel/vdso/gettimeofday.S b/Trunk/linux-4.19.90/arch/arm64/kernel/vdso/gettimeofday.S
index 856fee6d..7ee685d9 100644
--- a/Trunk/linux-4.19.90/arch/arm64/kernel/vdso/gettimeofday.S
+++ b/Trunk/linux-4.19.90/arch/arm64/kernel/vdso/gettimeofday.S
@@ -122,7 +122,7 @@ x_tmp		.req	x8
 9998:
 	.endm
 
-	.macro clock_gettime_return, shift=0
+	.macro clock_gettime_return shift=0
 	.if \shift == 1
 	lsr	x11, x11, x12
 	.endif
@@ -227,7 +227,7 @@ realtime:
 	seqcnt_check fail=realtime
 	get_ts_realtime res_sec=x10, res_nsec=x11, \
 		clock_nsec=x15, xtime_sec=x13, xtime_nsec=x14, nsec_to_sec=x9
-	clock_gettime_return, shift=1
+	clock_gettime_return shift=1
 
 	ALIGN
 monotonic:
@@ -250,7 +250,7 @@ monotonic:
 		clock_nsec=x15, xtime_sec=x13, xtime_nsec=x14, nsec_to_sec=x9
 
 	add_ts sec=x10, nsec=x11, ts_sec=x3, ts_nsec=x4, nsec_to_sec=x9
-	clock_gettime_return, shift=1
+	clock_gettime_return shift=1
 
 	ALIGN
 monotonic_raw:
@@ -271,7 +271,7 @@ monotonic_raw:
 		clock_nsec=x15, nsec_to_sec=x9
 
 	add_ts sec=x10, nsec=x11, ts_sec=x13, ts_nsec=x14, nsec_to_sec=x9
-	clock_gettime_return, shift=1
+	clock_gettime_return shift=1
 
 	ALIGN
 realtime_coarse:
diff --git a/Trunk/linux-4.19.90/arch/arm64/kernel/vdso/note.S b/Trunk/linux-4.19.90/arch/arm64/kernel/vdso/note.S
index e20483b1..2931eda3 100644
--- a/Trunk/linux-4.19.90/arch/arm64/kernel/vdso/note.S
+++ b/Trunk/linux-4.19.90/arch/arm64/kernel/vdso/note.S
@@ -24,8 +24,6 @@
 #include <linux/elfnote.h>
 #include <linux/build-salt.h>
 
-ELFNOTE_START(Linux, 0, "a")
-	.long LINUX_VERSION_CODE
-ELFNOTE_END
+ELFNOTE_LINUX(.long LINUX_VERSION_CODE)
 
 BUILD_SALT
diff --git a/Trunk/linux-4.19.90/arch/arm64/lib/memcpy.S b/Trunk/linux-4.19.90/arch/arm64/lib/memcpy.S
index 67613937..dfedd4ab 100644
--- a/Trunk/linux-4.19.90/arch/arm64/lib/memcpy.S
+++ b/Trunk/linux-4.19.90/arch/arm64/lib/memcpy.S
@@ -68,9 +68,8 @@
 	stp \ptr, \regB, [\regC], \val
 	.endm
 
-	.weak memcpy
 ENTRY(__memcpy)
-ENTRY(memcpy)
+WEAK(memcpy)
 #include "copy_template.S"
 	ret
 ENDPIPROC(memcpy)
diff --git a/Trunk/linux-4.19.90/arch/arm64/lib/memmove.S b/Trunk/linux-4.19.90/arch/arm64/lib/memmove.S
index a5a44590..e3de8f05 100644
--- a/Trunk/linux-4.19.90/arch/arm64/lib/memmove.S
+++ b/Trunk/linux-4.19.90/arch/arm64/lib/memmove.S
@@ -57,9 +57,8 @@ C_h	.req	x12
 D_l	.req	x13
 D_h	.req	x14
 
-	.weak memmove
 ENTRY(__memmove)
-ENTRY(memmove)
+WEAK(memmove)
 	cmp	dstin, src
 	b.lo	__memcpy
 	add	tmp1, src, count
diff --git a/Trunk/linux-4.19.90/arch/arm64/lib/memset.S b/Trunk/linux-4.19.90/arch/arm64/lib/memset.S
index f2670a9f..316263c4 100644
--- a/Trunk/linux-4.19.90/arch/arm64/lib/memset.S
+++ b/Trunk/linux-4.19.90/arch/arm64/lib/memset.S
@@ -54,9 +54,8 @@ dst		.req	x8
 tmp3w		.req	w9
 tmp3		.req	x9
 
-	.weak memset
 ENTRY(__memset)
-ENTRY(memset)
+WEAK(memset)
 	mov	dst, dstin	/* Preserve return value.  */
 	and	A_lw, val, #255
 	orr	A_lw, A_lw, A_lw, lsl #8
 

diff --git a/Trunk/linux-4.19.90/scripts/dtc/dtc-lexer.l b/Trunk/linux-4.19.90/scripts/dtc/dtc-lexer.l
index 615b7ec6..d3694d6c 100644
--- a/Trunk/linux-4.19.90/scripts/dtc/dtc-lexer.l
+++ b/Trunk/linux-4.19.90/scripts/dtc/dtc-lexer.l
@@ -38,7 +38,6 @@ LINECOMMENT	"//".*\n
 #include "srcpos.h"
 #include "dtc-parser.tab.h"
 
-YYLTYPE yylloc;
 extern bool treesource_error;
 
 /* CAUTION: this will stop working if we ever use yyless() or yyunput() */
-- 
2.29.2

```

### 编译内核

```
#使用正常的交叉编译链编译linux内核，为了对比编译时间这里用了time
make CROSS_COMPILE=$(YOUR_CROSS_COMPILE) ARCH=arm64 -j32 $(YOUR_LINUX_DEFCONFIG)
time make CROSS_COMPILE=$(YOUR_CROSS_COMPILE) ARCH=arm64 -j32
#使用LLVM编译内核
make LLVM=1 LLVM_IAS=1 CROSS_COMPILE=$(YOUR_CROSS_COMPILE) ARCH=arm64 $(YOUR_LINUX_DEFCONFIG)
time make LLVM=1 LLVM_IAS=1 CROSS_COMPILE=$(YOUR_CROSS_COMPILE) ARCH=arm64 -j32

```

使用 LLVM 编译 ARM64 可执行程序
----------------------

使用 LLVM 编译可执行程序需要给 clang 指定如下参数：

```
#指定要编译的目标平台
--target=
#指定编译链根，可以从对应的gcc中获取
--sysroot=
#指定交叉编译链根路径
--gcc-toolchain=

```

写个最简单的 helloworld，main.c：

```
#include "stdio.h"

int main(void)
{
    printf("helloworld\n");
    return 0;
}


```

用 LLVM 编译的 makefile 如下：

```
#这里填自己的交叉编译链
CROSS_COMPILE:=aarch64-linux-gnu

CROSS_COMPILE_SYSROOT = \
  $(shell $(CROSS_COMPILE)-gcc -print-sysroot 2>&1)

CROSS_COMPILE_PATH = \
  $(strip $(patsubst %$(CROSS_COMPILE)-gcc, %, $(shell which $(CROSS_COMPILE)-gcc 2>&1)))../

COMPILER_SPECIFIC_CFLAGS += \
  --target=$(CROSS_COMPILE) \
  --sysroot=$(CROSS_COMPILE_SYSROOT) \
  --gcc-toolchain=$(CROSS_COMPILE_PATH)


CFLAGS += $(COMPILER_SPECIFIC_CFLAGS)

all:
#这些是测试
	clang $(CFLAGS) -S main.c -o - | llvm-mca --mtriple=aarch64 -mcpu=cortex-a55
	clang $(CFLAGS) -S -emit-llvm main.c
	clang $(CFLAGS) -S main.c -o main_clang.S
	clang $(CFLAGS) -S -Oz main.c -o main_clang_Oz.S
	clang $(CFLAGS) -S -Oz main.c -o main_clang_Oz.S
#这些是实际编译可执行程序
	clang $(CFLAGS) main.c -o main_clang

clean:
	rm -rf main_* main main.bc main.ll main.opt.yaml

```

直接 make 即可生成 llvm 编译的可执行程序 main_clang
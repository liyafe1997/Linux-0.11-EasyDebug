Linux-0.11-EasyDebug
==========

The old Linux kernel source ver 0.11 which has been tested under modern Linux,  Mac OSX and Windows.

Forked from https://github.com/yuan-xy/Linux-0.11

## What did I do in this repo (Compare to the original [Linux-0.11](https://github.com/yuan-xy/Linux-0.11) )
1. Improve and fix bugs in Makefile.
2. Add tasks.json and launch.json for VSCode for easier debugging. Now you can just open this repository in VSCode and click Debug. (Of cause you have to make sure the toolchain is installed)
3. Fix `bochsrc-hd.bxrc` to make it work on newer version of bochs.

Linux (Windows with WSLg):
![VSCodeScreenshotImg](https://github.com/liyafe1997/Linux-0.11-EasyDebug/assets/18359157/7c0cf92d-4650-4fdb-88fb-e1443fe012a0)

macOS:
![macOS_VSCode_debug](https://github.com/liyafe1997/Linux-0.11-EasyDebug/assets/18359157/2ec5caa6-b329-478a-8b2e-950a6e8b68d4)

## Build bochs and configurate it for debugging
gdb & VSCode is work not very well with bochs, so I recommend you to use QEMU for debugging with gdb and VSCode,

If you need some magic debug functions that only bochs has, you can use bochs built-in debugger. (See below down)

Because the pre-built bochs in major Linux distribution is not enabled with gdb support, you have to build it yourself.
Here is the steps to build bochs with gdb support.

1. Build bochs with gdb support
```
tar zxvf bochs-2.7.tar.gz
cd bochs-2.7
mkdir bin
./configure --enable-debugger --enable-plugins --enable-disasm --enable-gdb-stub  --prefix=$(realpath ./bin)
make -j8
make install
```
2. Put the bochs binary into the `Makefile`'s `BOCHS` variable.
```
# indicate the path of the bochs
#BOCHS=$(shell find tools/ -name "bochs" -perm 755 -type f)
BOCHS=~/bochs-2.7/bin/bin/bochs
```

3. Try to start with bochs
```
make bochs-start
```
If you want to debug with bochs, run
```
make bochs-debug
```
You can just run "GDB with bochs" in VSCode to start debugging.

## Debug with bochs built-in debugger
### Build bochs with built-in debugger
Bochs has a built-in debugger, which supports some magic features that gdb and QEMU doesn't support.

(E.g. `info gdt` can directly show the items of GDT, while gdb and QEMU can't do this)

Since the built-in debugger is conflict with gdb, you have to disable gdb support when building bochs(remove `--enable-gdb-stub`), and add `--enable-debugger` to enable the built-in debugger.
```
./configure --enable-debugger --enable-plugins --enable-disasm --prefix=$(realpath ./bin)
```
Then `make` and `make install` as usual.

### Set "magic" breakpoint
Bochs can be breaked by `xchgw %bx, %bx` instruction. And in `bochsrc-hd.bxrc`, I have set `magic_break: enabled=1`.
Just in the place you want to break, insert `xchgw %bx, %bx` instruction.

For example, I want to see the differece before and after `lgdt` in `setup.s`, I can put this "magic breakpoint instruction `xchgw %bx, %bx`" in `setup.s`:

```
end_move:
	xchgw %bx, %bx
	mov	$SETUPSEG, %ax	# right, forgot this at first. didn't work :-)
	mov	%ax, %ds
	
	lidt	idt_48		# load idt with 0,0
	lgdt	gdt_48		# load gdt with whatever appropriate
	xchgw %bx, %bx
```

Then in bochs, you can use `info gdt` to see the GDT before and after `lgdt`:

```
<bochs:1> c
(0) Magic breakpoint
Next at t=34963705
(0) [0x000000090351] 9020:0151 (unk. ctxt): mov ax, 0x9020            ; b82090
<bochs:2> info gdt
Global Descriptor Table (base=0x000f9ad7, limit=48):
GDT[0x0000]=??? descriptor hi=0x00000000, lo=0x00000000
GDT[0x0008]=??? descriptor hi=0x00000000, lo=0x00000000
GDT[0x0010]=Code segment, base=0x00000000, limit=0xffffffff, Execute/Read, Non-Conforming, Accessed, 32-bit
GDT[0x0018]=Data segment, base=0x00000000, limit=0xffffffff, Read/Write, Accessed
GDT[0x0020]=Code segment, base=0x000f0000, limit=0x0000ffff, Execute/Read, Non-Conforming, Accessed, 16-bit
GDT[0x0028]=Data segment, base=0x00000000, limit=0x0000ffff, Read/Write, Accessed
You can list individual entries with 'info gdt [NUM]' or groups with 'info gdt [NUM] [NUM]'
<bochs:3> c
(0) Magic breakpoint
Next at t=34963710
(0) [0x000000090362] 9020:0162 (unk. ctxt): in al, 0x92               ; e492
<bochs:4> info gdt
Global Descriptor Table (base=0x000903c9, limit=2048):
GDT[0x0000]=??? descriptor hi=0x00000000, lo=0x00000000
GDT[0x0008]=Code segment, base=0x00000000, limit=0x007fffff, Execute/Read, Non-Conforming, 32-bit
GDT[0x0010]=Data segment, base=0x00000000, limit=0x007fffff, Read/Write
GDT[0x0018]=??? descriptor hi=0x08000000, lo=0x00000000
GDT[0x0020]=16-Bit Call Gate target=0x0009:0x890003c9, DPL=0
GDT[0x0028]=32-Bit Trap Gate target=0x04c2:0x200ec1c2, DPL=0
GDT[0x0030]=Code segment, base=0x043a3c30, limit=0x000204d0, Execute-Only, Conforming, 16-bit
GDT[0x0038]=16-Bit TSS (Busy) at 0x0dece210, length 0x8cd07
GDT[0x0040]=??? descriptor hi=0xc310cd0a, lo=0xb010cd0e
GDT[0x0048]=??? descriptor hi=0x65772077, lo=0x6f4e0a0d
GDT[0x0050]=32-Bit TSS (Available) at 0x20206572, length 0xe6120
GDT[0x0058]=??? descriptor hi=0x2e2e2070, lo=0x75746573
GDT[0x0060]=16-Bit TSS (Busy) at 0x720a0d0a, length 0x50d2e
GDT[0x0068]=32-Bit Trap Gate target=0x2072:0x3a536f73, DPL=2
GDT[0x0070]=Code segment, base=0x53726f6d, limit=0x0000654d, Execute-Only, Non-Conforming, Accessed, 16-bit
GDT[0x0078]=LDT
GDT[0x0080]=32-Bit TSS (Available) at 0x66204448, length 0xe0a0d
GDT[0x0088]=32-Bit Call Gate target=0x430a:0x6e690d6f, DPL=3
GDT[0x0090]=??? descriptor hi=0x6165483a, lo=0x73726564
GDT[0x0098]=Data segment, base=0x633a7372, limit=0x00056564, Read/Write, Accessed
GDT[0x00a0]=??? descriptor hi=0x0000003a, lo=0x7372746f
GDT[0x00a8]=??? descriptor hi=0x00000000, lo=0x00000000
GDT[0x00b0]=??? descriptor hi=0x00000000, lo=0x00000000
GDT[0x00b8]=??? descriptor hi=0x00000000, lo=0x00000000
GDT[0x00c0]=??? descriptor hi=0x00000000, lo=0x00000000
GDT[0x00c8]=??? descriptor hi=0x00000000, lo=0x00000000
GDT[0x00d0]=??? descriptor hi=0x00000000, lo=0x00000000
......
```

## 1. Build on Linux

### 1.1. Linux Setup

* a linux distribution: debian , ubuntu and mint are recommended
* some tools: gcc gdb qemu
* a linux-0.11 hardware image file: hdc-0.11.img, please download it from http://www.oldlinux.org, or http://mirror.lzu.edu.cn/os/oldlinux.org/, ant put it in the root directory.
* Now, This version already support the Ubuntu 16.04, enjoy it.

### 1.2. hack linux-0.11
```bash
$ make help		// get help
$ make  		// compile
$ make start		// boot it on qemu
$ make debug		// debug it via qemu & gdb, you'd start gdb to connect it.
```
```gdb
$ gdb tools/system
(gdb) target remote :1234
(gdb) b main
(gdb) c
```

## 2. Build on Mac OS X

### 2.1. Mac OS X Setup

* install cross compiler gcc and binutils
* install qemu
* install gdb.

```
brew tap nativeos/i386-elf-toolchain
brew install nativeos/i386-elf-toolchain/i386-elf-binutils
brew install nativeos/i386-elf-toolchain/i386-elf-gcc
brew install qemu
brew install gdb
```
Add the following environment variable to your `~/.bash_profile` or `~/.zshrc`

```
export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES
```

### 2.2. hack linux-0.11
same as section 1.2


## 3. Build on Windows
todo...

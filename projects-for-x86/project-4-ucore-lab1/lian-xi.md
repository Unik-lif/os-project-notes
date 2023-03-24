---
description: 简单的任务练习
---

# 练习

### 练习1：

1. 操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)
2. 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？

利用`make V=`可以得到最终的编译顺序，可以看到：

```sh
1 + cc kern/init/init.c
2   gcc -Ikern/init/ -march=i686 -fno-builtin -fno-PIC -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -
    Ikern/trap/ -Ikern/mm/ -c kern/init/init.c -o obj/kern/init/init.o

// -I: for directories.
// -fno-builtin: allow same name function declaration
// -march=i686: i686 is a subtype of i386 machine.
// -fno-PIC: possition independent code.
// -Wall: warning for all.
// -ggdb: allow gdb message.
// -m32: i386 is a 32-bit machine.
// -gstabs: more gdb message.
// -nostdinc: Don't search standard position, search directories with "I" marks.
// -fno-stack-protector: Don't provide stack protection.
```

通过RTFM阅读相关的标志，我们在`code block`底部简要解析如下，可以看到对于`kern`文件夹和`libs`文件夹下对源文件进行了向目标文件的转换。

之后，利用`ld`链接器将上述源文件链接在一起：链接脚本为`kernel.ld`.

```sh
  1 + ld bin/kernel
34  ld -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel obj/kern/init/init.o obj/kern/libs/readline.o obj/kern/libs/stdio.o obj/kern/debug/kd
    ebug.o obj/kern/debug/kmonitor.o obj/kern/debug/panic.o obj/kern/driver/clock.o obj/kern/driver/console.o obj/kern/driver/intr.o obj/kern/driver/p
    icirq.o obj/kern/trap/trap.o obj/kern/trap/trapentry.o obj/kern/trap/vectors.o obj/kern/mm/pmm.o obj/libs/printfmt.o obj/libs/string.o

// -T: read the files in the command line.
// -m: emulation linker.
// elf_i386: 
// -nostdlib: don't use standard libc.    
    
```

第三步，编译内核启动源码：方法类似，同时编译了工具`sign.c`.

第四步，将内核启动源码链接到`0x7c00`这个位置上，并对这个东西签名。

```sh
  1 + ld bin/bootblock
43  ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o

// -N: set the data/code segment to be readable and writable.
// -e: set the beginning of our code as the 'start' entry.
// -Ttext: the start of text segment will be 0x7c00. Which means the code will be placed here.

// Makefile scripts:
  5 $(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
  4         @echo + ld $@
  3         $(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
  2         @$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
  1         @$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
166         @$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
```

在完成上面的操作后，利用`sign`对生成的`bootblock`文件添上签名再写回。

```c
  1     buf[510] = 0x55;
32      buf[511] = 0xAA;
```

这个数据很奇怪，不过我们还是找到了相关的资料，是主引导记录扇区的有效位。主引导扇区，即MBR，是计算机访问硬盘后读入的第一个扇区。

<figure><img src="../../.gitbook/assets/Screenshot 2023-03-24 165508.png" alt=""><figcaption><p>MBR结构</p></figcaption></figure>

最后，采用下述操作：将生成的文件塞到`ucore.img`这个镜像中。

```sh
  1 dd if=/dev/zero of=bin/ucore.img count=10000
  2 dd if=bin/bootblock of=bin/ucore.img conv=notrunc
  3 dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
  
  // dd: convert and copy a file.
  // for /dev/zero, Read operations from /dev/zero return as many null characters (0x00) as requested in the read operation.
  // if: file which we are read from.
  // of: file which we will write in.
  // count: copy only N input blocks.
  // conv: convert the file as per the comma separated symbol list.
  // seek: skip a block. Here will simply jump 512 bytes. For default block size is 512 bytes.
```

至于硬盘的主引导扇区的特征，我们已经在表格中提到了，即末尾的特殊字节。

### 练习2：

1. 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。
2. 在初始化位置0x7c00设置实地址断点,测试断点正常。
3. 从0x7c00开始跟踪代码运行，将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。
4. 自己找一个bootloader或内核中的代码位置，设置断点并进行测试。

根据做`JOS`的经验以及`RTFM`的体会，加电后第一个位置是`0xffff0`是一件很了然的事情，然而这里在调试时是有点`bug`，首先不知道为什么这边的`gdb`非常操蛋的没有出现相关的指令，需要手动在里头编写一个函数来帮忙：

```sh
define nI # for next Instruction.
x/i $eip # print the next instruction.
end
```

其次，似乎在这种模式下没有`ip`寄存器，其对应的其实是`eip`这个东东。虽然e是extension的意思，但其实架构上是有点小冲突（这应该是386的机器，但是应该能让我去直接找`ip`寄存器才对）。箭头指向的是`ip`的位置，打印的也是那部分的地址，但按照常理这显然是出了问题，如果要看下一条指令到底是什么，我们应该把它做一下修正。这下终于可以打印出正确的指令了。

```sh
(gdb) nI
=> 0xfff0:	add    %al,(%eax) # 0xfff0 -> eip.

# 探索了一下下
(gdb) x/i $cs << 4 + $pc
Argument to arithmetic operation not a number or boolean.
(gdb) x/i $($cs << 4 + $pc)
The history is empty.
(gdb) x/i ($cs << 4 + $pc)
Argument to arithmetic operation not a number or boolean.
(gdb) x/i (($cs << 4) + $pc)
   0xfe05b:	cmpw   $0xffc8,%cs:(%esi)
   
# gdbinit function
define nI # for next Instruction.
x/i (($cs << 4) + $eip) # print the next instruction.
end
```

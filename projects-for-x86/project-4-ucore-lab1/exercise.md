---
description: 简单的任务练习......喂，气死了，简单个毛！
---

# Exercise

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

利用`si`和`ni`相关指令稍微跟一下BIOS加电后的指令即可，它们并没有什么太大的意思，而且就人力来说应该是跟不完的，在`JOS Lab1`中我也尝试跟了一小部分，可以参考，这一部分其实是很`dirty`的代码，真正开始起作用的其实是`0x7c00`之后的东东，它是我们之前装载进去的`bootblock`内容。

### 练习3：

* 为何开启A20，以及如何开启A20
* 如何初始化GDT表
* 如何使能和进入保护模式

而且，在`Makefile`中装载是有顺序的，具体来说先装的是`bootasm.S`，然后再是`bootmain.c`这一部分。我们尝试对它们进行解读。

首先，我们来看一下A20的开启原因和方法：A20开启是让实模式可以访问高端地址，即高于自身16位的地址空间。如果不开启A20的话，访问高地址会自动回转回低地址，这类似于一个取`mod`操作。

方法的话参考下面这个网址所提的内容：写的很好

{% embed url="http://hengch.blog.163.com/blog/static/107800672009013104623747/" %}
A20 relative info
{% endembed %}

```nasm
 39 # start address should be 0:7c00, in real mode, the beginning address of the running bootloader
 38 .globl start
 37 start:
 36 .code16                                             # Assemble for 16-bit mode
 35     cli                                             # Disable interrupts
 34     cld                                             # String operations increment
 33
 32     # Set up the important data segment registers (DS, ES, SS).
 31     xorw %ax, %ax                                   # Segment number zero
 30     movw %ax, %ds                                   # -> Data Segment
 29     movw %ax, %es                                   # -> Extra Segment
 28     movw %ax, %ss                                   # -> Stack Segment
 27
 26     # Enable A20:
 25     #  For backwards compatibility with the earliest PCs, physical
 24     #  address line 20 is tied low, so that addresses higher than
 23     #  1MB wrap around to zero by default. This code undoes this.
 22 seta20.1:
 21     inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
 20     testb $0x2, %al                                 # bit 1: input buffer empty or not.
 19     jnz seta20.1
 18
 17     movb $0xd1, %al                                 # 0xd1 -> port 0x64
 16     outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port
 15
 14 seta20.2:
 13     inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
 12     testb $0x2, %al
 11     jnz seta20.2
 10
  9     movb $0xdf, %al                                 # 0xdf -> port 0x60
  8     outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
  
  # Process:
  # 1. start is the entry name, in Makefile process we inform ld to use start as the begin entry.
  # 2. cli, cld: please refer to Intel Manual. It disable the interrupt flag.
  # 3. clear segment register.
  
  # A20 port should be opened here.
  # How to open this port? -> 8042 port. 并且是output port上的某一位来控制
  # 标准的写output的方法：
  # 向64h发送0d1h命令，然后向60h写入Output Port的数据
  # 因此方案就很清楚了：
  # 1. 等待input buffer清空
  # 2. 向64h发送0d1h表示要写入的命令
  # 3. 再次等待input buffer清空
  # 4. 写入数据0xdfh，使得倒数第二位为1即可，上面的0xdf只是一个选择
```

我们根据80386手册对于初始化的描述章节可以得知，在从实模式转换为保护模式的时候，需要把全局描述符表也做好初始化。这意味着`GDTR`需要指向一个合法的`GDT`位置。

```nasm
  4     # Switch from real to protected mode, using a bootstrap GDT
  3     # and segment translation that makes virtual addresses
  2     # identical to physical addresses, so that the
  1     # effective memory map does not change during the switch.
49      lgdt gdtdesc # define gdt before protected mode. the address is in gdtdesc.


## skip some code, simply check the gdt here. ##
  1 gdt:
80      SEG_NULLASM                                     # null seg
  1     SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)           # code seg for bootloader and kernel
  2     SEG_ASM(STA_W, 0x0, 0xffffffff)                 # data seg for bootloader and kernel
  3
  4 gdtdesc:
  5     .word 0x17                                      # sizeof(gdt) - 1
  6     .long gdt                                       # address gdt
  
  
```

为了理解这个，我们先`RTFM`一下，在李忠老师的书《x86汇编语言：从实模式到保护模式》中也可以看到类似的表述：

<figure><img src="../../.gitbook/assets/Screenshot 2023-03-25 191825.png" alt=""><figcaption><p>GDT Info</p></figcaption></figure>

也就是说，`GDTR`寄存器首先要规定好`limit`值，然后再把gdt的地址放在`base`处，至于`limit`的值，它应该被设置成`8N-1`这个值。我们看到此处的`gdt`共有三个项，所以设置`limit`为`0x17`是很合理的选择。

此外，和在前置知识中我们提到的那样，第一项是作为`NULL`特殊处理的，这边也就得到了一个不错的解释。

如要了解剩下的两个项干了什么，请参考这个资料。

{% embed url="https://wiki.osdev.org/Global_Descriptor_Table" %}
GDT项资料
{% endembed %}

```c
 11 #ifndef __BOOT_ASM_H__
 10 #define __BOOT_ASM_H__
  9
  8 /* Assembler macros to create x86 segments */
  7
  6 /* Normal segment */
  5 #define SEG_NULLASM                                             \
  4     .word 0, 0;                                                 \
  3     .byte 0, 0, 0, 0
  2
  1 #define SEG_ASM(type,base,lim)                                  \
12      .word (((lim) >> 12) & 0xffff), ((base) & 0xffff);          \
  1     .byte (((base) >> 16) & 0xff), (0x90 | (type)),             \
  2         (0xC0 | (((lim) >> 28) & 0xf)), (((base) >> 24) & 0xff)
  3
  4
  5 /* Application segment type bits */
  6 #define STA_X       0x8     // Executable segment
  7 #define STA_E       0x4     // Expand down (non-executable segments)
  8 #define STA_C       0x4     // Conforming code segment (executable only)
  9 #define STA_W       0x2     // Writeable (non-executable segments)
 10 #define STA_R       0x2     // Readable (executable segments)
 11 #define STA_A       0x1     // Accessed
 12
 13 #endif /* !__BOOT_ASM_H__ */
```

我们尝试对`SEG_ASM`逐项分析：

首先，前16位是`Limit`的低16位，而输入的`lim`值是`0xffffffff`，严格意义上的`lim`是20 bit，所以需要作12位的右移工作，把多余的12位删掉。base则是同理，但它自然便是32 bit，所以右移便是不需要的了。

之后，第32-39位正好是一个字节，它反映的是Base的第16-23位。下一个字节则是Access Byte，其中按照下面的方式分成六个信息组件，我们已经在前置知识中提过了。这边仅给出表格结构。

```
Access Byte
7	6	5	4	3	2	1	0
P	DPL	        S	E	DC	RW	A
```

程序中让`type`与作`0x90`或操作，这很显然是把`Present`位和`S`位设置成1了。Present位自不必说，S位设置为1表示我们要做的是代码和数据段。

有了这个知识后，底下的几个Define基本是把功能写在了脸上。于是我们可以确定，GDT表中其中允许读和执行的描述符是代码段，而仅允许修改的是数据段。

对于第48-51位，这是`Limit`的高四位所在地，让它右移一下再与`0xf`与一下就行了，而52-55位，则是flag，这边设置成了`1100`这个样子

```
Flags
3	2	1	0
G	DB	L	Reserved

G: granularity flag, 0->Limit value One Byte. 1->Limit value 4 KiB.
DB: size flag. 1->32-bit protected mode segment. 0 for 16-bit.
L: for 64-bit. skip here.
```

于是我们清楚了Limit的粒度是`4` KiB，这是一个很重要的信息。

对于最高的那个比特，太简单了，不提。

在完成了GDT表的装载之后，下一步是通过使能进入保护模式。在前置知识我们提到，CR0寄存器中有一位用于控制保护模式与否。

哦，对了，这个东西和刚刚的GDT写在一起：我们刚刚建立了GDT表，所以对于代码段和数据段的地址现在也非常清楚了。CR0的控制寄存器只要修改最后一位，就能通知硬件时刻准备好进入保护模式。

之后，只要轻轻一跃，我们就进入了保护模式。

```nasm
  2 .set PROT_MODE_CSEG,        0x8                     # kernel code segment selector
  1 .set PROT_MODE_DSEG,        0x10                    # kernel data segment selector
10  .set CR0_PE_ON,             0x1                     # protected mode enable flag

 15     movl %cr0, %eax
 16     orl $CR0_PE_ON, %eax
 17     movl %eax, %cr0
 18
 19     # Jump to next instruction, but in 32-bit code segment.
 20     # Switches processor into 32-bit mode.
 21     ljmp $PROT_MODE_CSEG, $protcseg
```

---
description: bootstrap of JOS lab1.
---

# Project 3: JOS lab1

#### 声明：努力不再重复与uCore相同的知识栈部分。

### **Exercise1: Assembly familiarness.**

The difference of AT\&T and Intel is quite big. 至少有一个东西我经常搞混球:

```
AT&T:  movl %eax, %ebx ;; load ebx with the value of eax. ->
Intel: mov ebx, eax ;; <-
```

此外，还有一个比较值得学习的东西。即Inline Assembly。这玩意我是真的不怎么会，用到了再查询吧。

简单看一下资料**GCC-Inline-Assembly-HOWTO**

```nasm
// 学习基本流程，查阅Intel手册去寻找指令的意思。
 asm ("cld\n\t" // cld指令，清空EFLAGS中DF寄存器，DF为0时，字符串操作会增加ESI或EDI的值
             "rep\n\t"
             "stosl"
             : /* no output registers */
             : "c" (count), "a" (fill_value), "D" (dest)
             : "%ecx", "%edi" 
             );
```

RTFM一下rep的功能，可以注意到下面的细节：

<figure><img src="../.gitbook/assets/Screenshot 2023-03-11 192842.png" alt=""><figcaption><p>REP STOS的固定用法</p></figcaption></figure>

上面的汇编用了这边的固定用法，因此上面的代码解读为在`dest`位置处用`fill_value`写入一共`count`次。并且告知GCC你的`ecx`和`edi`寄存器已经过修改，不再是`valid`的值。即通知GCC不要再用这一部分寄存器存放其他的值。

这说明RTFM是一个合理的方法。

```nasm
// 下面这个例子在HOW-TO中有很不错的解读，我们这边就用注释稍微说说吧。
 int a=10, b;
 asm ("movl %1, %%eax; // 1表示第二个出现的东东，即存放a值的寄存器
       movl %%eax, %0;" // 0表示第一个出现的东东，即存放b值的寄存器
      :"=r"(b)        /* output =r 表示output写值，r表示可以使用任意寄存器*/
      :"r"(a)         /* input */
      :"%eax"         /* clobbered register 通知GCC eax寄存器被修改过了，别塞其他的东西给他*/
      );    
```

Other cases:

```nasm
// case for input & output for the same register.
asm ("leal (%0,%0,4), %0"
   : "=r" (five_times_x) // %0
   : "0" (x) // set the input register to be 0, that's also the output register.
   );
   
// for clobber list: 一般情况明确写出来应该不需要添加在Clobber List中，GCC往往知道，主要
// 小心的是隐性地被修改的寄存器东东。
// 隐性修改影响到内存时还要加入volatile关键字，如果在asm的input和output中没有他们的话。
// 下面的这个case中假设_foo函数会用eax和ecx 
asm ("movl %0,%%eax;
     movl %1,%%ecx;
     call _foo"
    : /* no outputs */
    : "g" (from), "g" (to)
    : "eax", "ecx"
    );
// 有时候的volatile关键字是防止被优化的，需要被小心谨慎地执行。
asm volatile (:::);

// 关键词设置: commonly used constraints.
// 1. register operand constraints (r)
+---+--------------------+
| r |    Register(s)     |
+---+--------------------+
| a |   %eax, %ax, %al   |
| b |   %ebx, %bx, %bl   |
| c |   %ecx, %cx, %cl   |
| d |   %edx, %dx, %dl   |
| S |   %esi, %si        |
| D |   %edi, %di        |
+---+--------------------+
// 2. Memory operand constraint (m)
// when the operands are in the memory, use m.
asm("sidt %0\n" : :"m"(loc)); // store idtable on 'loc' position in the memory.
// 3. digit constraints.
asm ("incl %0" :"=a"(var):"0"(var)); // 0 means the first register, here is eax.

```

我们再看之后的一些有趣的例子：

```nasm
// Some code
__asm__ __volatile__(   "btsl %1,%0"
                      : "=m" (ADDR)
                      : "Ir" (pos)
                      : "cc"
                      );
```

通过RTFM我们可以得知，由于6.828基于32位架构，这里的btsl其实表示对于long words的操作，即4 Bytes大小单位的操作。具体来说，对于Input，设置了pos，`I`表示其范围在0\~31之间，ADDR则是存在内存中的一个数。

因此具体来说，ADDR内存对应的数的第pos位被提取出来赋值给CF，与此同时原位置被修改为1。在清楚btsl指令的情况下，gcc编译器知道内存被修改了，但可能不知道状态寄存器改变了，所以需要添上cc。

还有两个例子，尝试解读之（不一定要自己写成这样，那样感觉还是有点难度的）

```nasm
// 比较有技巧性
static inline char * strcpy(char * dest,const char *src)
{
int d0, d1, d2;
__asm__ __volatile__(  "1:\tlodsb\n\t" // load byte from 'src', into al.
                       "stosb\n\t" // store al at 'dest'.
                       "testb %%al,%%al\n\t" // test '\0' or not.
                       "jne 1b" // not 0, then jump back to 1.
                     : "=&S" (d0), "=&D" (d1), "=&a" (d2) // changed before finishing these lines.
                     : "0" (src),"1" (dest) // d0 就是第一个出现的寄存器
                     : "memory");
return dest;
}
```



<figure><img src="../.gitbook/assets/Screenshot 2023-03-12 212420.png" alt=""><figcaption><p>LODSB</p></figcaption></figure>

<figure><img src="../.gitbook/assets/Screenshot 2023-03-12 213003.png" alt=""><figcaption><p>STOSB</p></figcaption></figure>

就这么对照吧，当然为了实现同一个字符串copy的操作，可以用上面诸如`cld`等等指令的配合。

最后还有一个例子，那么大致上inline汇编就不再是个问题了。

```nasm
// syscall typical usage.
#define _syscall3(type,name,type1,arg1,type2,arg2,type3,arg3) \
type name(type1 arg1,type2 arg2,type3 arg3) \
{ \
long __res; \
__asm__ volatile (  "int $0x80" \
                  : "=a" (__res) \
                  : "0" (__NR_##name),"b" ((long)(arg1)),"c" ((long)(arg2)), \
                    "d" ((long)(arg3))); \
__syscall_return(type,__res); \
}N
```

Whenever a system call with three arguments is made, the macro shown above is used to make the call. The syscall number is placed in eax, then each parameters in ebx, ecx, edx. And finally "int 0x80" is the instruction which makes the system call work. The return value can be collected from eax.

仿照以前写cs61c的经验，准备好参数然后`int $0x80`中断一下就了事。



**PC physical address:**

```

+------------------+  <- 0xFFFFFFFF (4GB)
|      32-bit      |
|  memory mapped   |
|     devices      |
|                  |
/\/\/\/\/\/\/\/\/\/\

/\/\/\/\/\/\/\/\/\/\
|                  |
|      Unused      |
|                  |
+------------------+  <- depends on amount of RAM
|                  |
|                  |
| Extended Memory  |
|                  |
|                  |
+------------------+  <- 0x00100000 (1MB)    -------
|     BIOS ROM     |                            ^
+------------------+  <- 0x000F0000 (960KB)     |
|  16-bit devices, |                            |
|  expansion ROMs  |                            |
+------------------+  <- 0x000C0000 (768KB) legacy 8086
|   VGA Display    |                            |
+------------------+  <- 0x000A0000 (640KB)     |
|                  |                            |
|    Low Memory    |                            |
|                  |                            |
+------------------+  <- 0x00000000          -------
```

JOS启动为了兼容性保留了8086机器的一部分。材料中提到，以前的BIOS是真实存在ROM之中，而现在往往放在闪存里头。

start point:

```nasm

The target architecture is set to "i8086".
[f000:fff0]    0xffff0: ljmp   $0xf000,$0xe05b # strat from 0xffff0 as the cold reset vector.
0x0000fff0 in ?? () # 0xfe05b: the actual start of the POST code in the BIOS ROM.
```

上述即为一开始启动的代码，可以看到启动位置为0xffff0，在上述框图的`BIOS ROM`的位置附近。

### **Exercise 2. Use gdb** **to Trace Instructions.**

我们使用`si`进行追踪。

```sh
[f000:e05b]    0xfe05b: cmpl   $0x0,%cs:0x6ac8
0x0000e05b in ?? () # 看一下地址0xf6ac8, 但并没有研究出来这个东西到底是哪个dirty的小饼干
(gdb) x/w *0xf6ac8 # test for ROM memory.
0x0:    0x00000000 

[f000:e062]    0xfe062: jne    0xfd2e1 # won't jump for zero.
0x0000e062 in ?? ()
[f000:e066]    0xfe066: xor    %dx,%dx # 置零
0x0000e066 in ?? ()
[f000:e068]    0xfe068: mov    %dx,%ss # ss will be set to zero.
0x0000e068 in ?? ()
[f000:e06a]    0xfe06a: mov    $0x7000,%esp # esp set to 0x7000
0x0000e06a in ?? ()
[f000:e070]    0xfe070: mov    $0xf34c2,%edx # 为什么要赋值？
0x0000e070 in ?? ()
[f000:e076]    0xfe076: jmp    0xfd15c # 跳走了！
0x0000e076 in ?? ()
[f000:d15c]    0xfd15c: mov    %eax,%ecx # %ecx被赋值.
0x0000d15c in ?? ()
(gdb) i r eax
eax            0x0                 0
[f000:d15f]    0xfd15f: cli # 清除中断位，IF, 我们可以看一下ELFLAG
0x0000d15f in ?? ()
(gdb) i r eflags
eflags         0x46                [ PF ZF ]
[f000:d160]    0xfd160: cld # clear DF, direction flag 0, when using string operation, ESI/EDI will increase.
0x0000d160 in ?? ()
(gdb) i r eflags
eflags         0x46                [ PF ZF ]
[f000:d161]    0xfd161: mov    $0x8f,%eax # eax = 8f. 
0x0000d161 in ?? () 
(gdb) i r eflags
eflags         0x46                [ PF ZF ]
[f000:d167]    0xfd167: out    %al,$0x70 # output %al to 0x70 IO Port.
0x0000d167 in ?? ()
[f000:d169]    0xfd169: in     $0x71,%al # input from port 71 to al
0x0000d169 in ?? () 
# in fact, in & out functions are for CMOS
# The CMOS memory exists outside of the normal address space and cannot
# contain directly executable code. It is reachable through IN and OUT
# commands at port number 70h (112d) and 71h (113d). To read a CMOS byte,
# an OUT to port 70h is executed with the address of the byte to be read and
# an IN from port 71h will then retrieve the requested information. The
# following BASIC fragment will read 128 CMOS bytes and print them to the
# screen in 8 rows of 16 values.
#
[f000:d16b]    0xfd16b: in     $0x92,%al # input from port 92 to al
0x0000d16b in ?? ()
(gdb) i r al
al             0x0                 0
[f000:d16d]    0xfd16d: or     $0x2,%al # or.
0x0000d16d in ?? ()
[f000:d16f]    0xfd16f: out    %al,$0x92 # output to 0x92 port.
0x0000d16f in ?? ()
[f000:d171]    0xfd171: lidtw  %cs:0x6ab8 # They are commonly executed in real-address mode to allow processor initialization prior to switching to protected mode. load the number in.
0x0000d171 in ?? ()
[f000:d177]    0xfd177: lgdtw  %cs:0x6a74 # Load Global/Interrupt Descriptor Table 
0x0000d177 in ?? ()
[f000:d17d]    0xfd17d: mov    %cr0,%eax # It seems that I can't get cr0 info using gdb.
0x0000d17d in ?? ()
[f000:d180]    0xfd180: or     $0x1,%eax #
0x0000d180 in ?? ()
(gdb) i r eax
eax            0x60000010          1610612752
(gdb) si
[f000:d184]    0xfd184: mov    %eax,%cr0
0x0000d184 in ?? ()
(gdb) i r eax
eax            0x60000011          1610612753
[f000:d187]    0xfd187: ljmpl  $0x8,$0xfd18f // jump
0x0000d187 in ?? ()
(gdb) si
The target architecture is set to "i386". // world change!! Before this we are in real mode, but now we are in protected mode.
=> 0xfd18f:     mov    eax,0x10
0x000fd18f in ?? ()
```

阅读资料：

{% embed url="http://web.archive.org/web/20040619131941/http://members.iweb.net.au/~pstorr/pcbook/book1/post.htm" %}
Phil Storr's book for booting legacy 8086 machine.
{% endembed %}

{% embed url="https://bochs.sourceforge.io/techspec/CMOS-reference.txt" %}
CMOS-Memory usage
{% endembed %}

```
Background

The CMOS (complementary metal oxide semiconductor) memory is actually
a 64 or 128 byte battery-backed RAM memory module that is a part of the
system clock chip. Some IBM PS/2 models have the capability for a
2k (2048 byte) CMOS ROM Extension.

First used with clock-calender cards for the IBM PC-XT, when the PC/AT
(Advanced Technology) was introduced in 1985, the Motorola MC146818
became a part of the motherboard. Since the clock only uses fourteen of
the RAM bytes, the rest are available for storing system configuration data.

Interestingly, the original IBM-PC/AT (Advanced Technology) standard for
the region 10h-3Fh is nearly universal with one notable exception: The
IBM PS/2 systems deviate considerably (Note: AMSTRAD 8086 machines were
among the first to actively use the CMOS memory available and since they
*predate* the AT, do not follow the AT standard).
```

可以看出对于CMOS的使用是8086的一个convention，也算是8086启动的时候的一个dirty knowledge. 我们简单总结一下在实模式下系统启动做了什么：检查内存和寄存器能否使用，打开CMOS内存接口，设置中断表a

进入了386模式，即保护模式之后，我们来检查一下干了什么。现在可能希望做一次`console`上的输出，所以先把`0x10`这个值分布到`eax`上和其余段寄存器上。

```nasm
=> 0xfd18f:     mov    $0x10,%eax
0x000fd18f in ?? ()
(gdb) si
=> 0xfd194:     mov    %eax,%ds
0x000fd194 in ?? ()
(gdb) si
=> 0xfd196:     mov    %eax,%es
0x000fd196 in ?? ()
(gdb)
=> 0xfd198:     mov    %eax,%ss // stack segment. 当然其他这里的寄存器都是data segment.
0x000fd198 in ?? ()
(gdb)
=> 0xfd19a:     mov    %eax,%fs
0x000fd19a in ?? ()
(gdb)
=> 0xfd19c:     mov    %eax,%gs
0x000fd19c in ?? ()
(gdb)
=> 0xfd19e:     mov    %ecx,%eax
0x000fd19e in ?? ()
=> 0xfd1a0:     jmp    *%edx // 再次跳转
0x000fd1a0 in ?? ()
(gdb) i r edx
edx            0xf34c2             996546
// 之后就不再用gdb si命令逐步调试了，似乎再在汇编上逐步玩没有太大意思。建议一键按C去往下继续跟踪。
```

并没在后续中找到我想要的\`int\`指令以开启VGA输入功能，令本鼠鼠感叹。不得不说底层还是有很多dirty的知识，没有太大必要一定要掌握。直接继续看材料吧。

**Part 2. The Boot Loader**

在初始化了PCI总线和其余BIOS所知的主要设备后，为了启动，机器将会开始去寻找可以用于开机启动，即含有`boot-loader`的设备。如果找到，BIOS将会读取`boot loader`并对其展开控制。

PC扇区大小为512字节，是磁盘传输的最小粒度。第一个扇区被称为启动扇区，内部有`boot loader`组件。第一个扇区会被装载到`0x7c00~0x7dff`这个范围。

与现在利用CD-ROM来启动系统不同，6.828中依然使用较早的方式来启动，维持512 bytes启动扇区这一规则。

**各模式简介，**参考[https://pdos.csail.mit.edu/6.828/2018/readings/pcasm-book.pdf](https://pdos.csail.mit.edu/6.828/2018/readings/pcasm-book.pdf)

**实模式：16-bit -> 20-bit memory address for physical.**

**16-bit 保护模式：16-bit -> 20-bit memory address for virtual. (too small)**

**32-bit 保护模式：添加了更大的地址空间，满足4GB，同时添加了分页的机制。**

在源代码中已经有了很多的解读，做好理解感觉就足够了。

#### JOS启动具体的流程：

**boot.S**

**实模式：关闭中断flag ->** 段寄存器清空 ->  **远古设备兼容I/O设置** **->** 中断表读入 -> 修改CR0寄存器 -> 跳转进入**保护模式**！

**保护模式：设置好数据段寄存器 -> 进入main函数 (由ASM->C)**

特别的：**CR0**寄存器的最后一位是保护位，如果为1，则会准备进入保护模式。

```nasm
[   0:7c2d] => 0x7c2d:  jmp    0x8:0x7c32                                │
0x00007c2d in ?? ()
```

**main.c**

```c
// Some code
// Read 'count' bytes at 'offset' from kernel into physical address 'pa'
// Might copy more than asked, because sectors mechanism. -> copy should be a multiples
// of sector size.
void readseg(uint32_t pa, uint32_t count, uint32_t offset) {
    // .... skip some codes here ....
    // ~(SECTSIZE - 1) => 0x200 => starts address from the multiples of 0x200.
    pa &= ~(SECTSIZE - 1);
    // .... skip some codes here ....
}

void readsect(void *dst, uint32_t offset) {
    waitdisk(); // wait for disk ready or not.
    // .... skip some codes here ....
    outb(0x1f7, 0x20); // cmd 0x20 - read sectors.
    waitdist();
    // read a sector.
    insl(0x1f0, dst, SECTSIZE/4);
}

void bootmain(void) {
    readseg((uint32_t) ELFHDR, SECTSIZE*8, 0);
}
```

看不懂具体的端口号对我这种强迫症来说确实挺难受的，chatgpt似乎明白我的苦恼。

在6.828实验1中，您需要了解的一些I/O ports包括：

1. 串口端口(COM1): 用于与计算机串口通信。在x86架构中，COM1的端口号是0x3f8，COM2的端口号是0x2f8，COM3的端口号是0x3e8，COM4的端口号是0x2e8。
2. 键盘端口(PS/2键盘): 用于与计算机键盘通信。在x86架构中，键盘控制器的端口号是0x60。
3. 显示器端口(EGA/VGA显示器): 用于控制计算机显示器。在x86架构中，VGA显示器的控制器端口号为0x3d4和0x3d5，EGA显示器的控制器端口号为0x3c0和0x3c1。

对于以上提到的端口号和功能，您可以参考一些资料来了解更详细的信息。以下是一些可能有用的资源：

1. Bochs模拟器手册（英文版）：Bochs是一个开源的x86模拟器，它的手册中包含了一些有关x86体系结构和I/O端口的信息。
2. 6.828课程的实验指导（英文版）：这是6.828课程的官方实验指导，其中包含了一些有关I/O端口的信息和代码示例。

### Exercise 3: trace

I've already traced once.

After the loading the kernel, we have these codes below: this tell us where we'll jump to.

```c
 18     // call the entry point from the ELF header
 17     // note: does not return!
 16     ((void (*)(void)) (ELFHDR->e_entry))();
```

* At what point does the processor start executing 32-bit code? What exactly causes the switch from 16-bit to 32-bit mode?
  * `ljmp $PROT_MODE_CSEG, $protcseg`
* What is the _last_ instruction of the boot loader executed, and what is the _first_ instruction of the kernel it just loaded?
  * `movw $0x1234,0x472`
* _Where_ is the first instruction of the kernel?
  * `movw $0x1234,0x472`
* How does the boot loader decide how many sectors it must read in order to fetch the entire kernel from disk? Where does it find this information?
  * from ELF. We have these codes below:

```c
  2     // load each program segment (ignores ph flags)
  1     ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
52      eph = ph + ELFHDR->e_phnum;
  1     for (; ph < eph; ph++)
  2         // p_pa is the load address of this segment (as well
  3         // as the physical address)
  4         readseg(ph->p_pa, ph->p_memsz, ph->p_offset);
```

To understand this, read the following link: **ESPECIALLY FOR LOADING PART.**

{% embed url="https://wiki.osdev.org/ELF" %}
ELF Format & Usage.
{% endembed %}

### Exercise 4: C pointer.

Skip this part.

### Exercise 5: Change link address to a wrong one.

```sh
[ 0:7c2d] => 0x7c2d: ljmp $0xb866,$0x88c32 // change text entry to 0x8c00.
[ 0:7c2d] => 0x7c2d: ljmp $0xb866,$0x87c3 // text entry 0x7c00.
```

**A very strange phenonmenon:** even when we change the entry of the `.text` to `0x7c00`, the initialization seems to act rightly.

`BIOS`会把指令装载到`0x7c00`这个位置，这其实是一个历史包袱，所以本质上并不会影响前述指令的执行，但链接器异常的影响依然存在，所以我们看到后续的指令执行出现了错误。

### Exercise 6: Examine the memory address 0x00100000

首先，我们利用下面的命令来查看利用`bootloader`装载的内核信息。

```sh
linkvm@link:/mnt/d/jos$ objdump -f obj/kern/kernel

obj/kern/kernel:     file format elf32-i386
architecture: i386, flags 0x00000112:
EXEC_P, HAS_SYMS, D_PAGED
start address 0x0010000c
```

可以看到ELF的起始地址是`0x001000c`，尝试用`GDB`跟踪一下。

首先，我们需要在`0x001000c`这个位置打一个断点，然后才能保证kernel确实已经加载完成了。

```sh
(gdb) x/10x 0x00100000
0x100000:       0x1badb002      0x00000000      0xe4524ffe      0x7205c766
(gdb) x/10x 0x0010000c
0x10000c:       0x7205c766      0x34000004      0x1000b812      0x220f0011
```

说实话，并没有太清楚这样做的意义何在。但0xc这样的偏移的存在似乎能够给人一个提示，即，即便利用`linker`设置了源码的进入入口，也不完全就会把这些代码设置在这个位置。

此外，在`kernel.asm`中我们可以看到，内核实际上被放在了一个比较高的位置上，在正式编译成内核时，似乎会添加`3`个字长的奇怪的代码信息，它们也不会被拿去执行（因为直接通过`entry`跳到了`0x10000c`处了，猜测是`x86`的一些历史遗留问题）：

```nasm
   3 .globl entry                                                          
   4 entry:                                                                
   5     movw    $0x1234,0x472           # warm boot                       
   6 f0100000:   02 b0 ad 1b 00 00       add    0x1bad(%eax),%dh           
   7 f0100006:   00 00                   add    %al,(%eax)                 
   8 f0100008:   fe 4f 52                decb   0x52(%edi)                 
   9 f010000b:   e4                      .byte 0xe4
```

### Exercise 7: use QEMU and GDB to trace into the JOS kernel

操作系统似乎很喜欢被链接到很高的位置上运行，这样的行为出于很多原因，其一便是给用户低地址空间自由使用的权利。注意，当我们说链接时，往往指的是虚拟内存。如果是装载，这个名词显然是指在物理内存中进行的。因此，在`JOS`的模型中，系统被装载在从`1MB`开始的空间中，但通过虚拟内存我们提供了一个假象，即`Linker`是把这个东东装载到虚拟内存的`0xf010000`处再运行内核的。

根据JOS文档，干这件事的是`kern/entrypgdir.c`，不过幸运的是现在我们并不需要对此程序做完全的解读，只需知道在我们开启`CR0`最后一位，并且进入保护模式之时，这个转换就已经发生了。

跟踪一下发现，确实如此，调整CR0后，于此同时这个东东就开始装载虚拟内存了。

```sh
=> 0x100025:    mov    %eax,%cr0

Breakpoint 1, 0x00100025 in ?? ()
(gdb) x/10x 0x00100000
0x100000:       0x1badb002      0x00000000      0xe4524ffe      0x7205c766
0x100010:       0x34000004      0x1000b812      0x220f0011      0xc0200fd8
0x100020:       0x0100010d      0xc0220f80
(gdb) x/10x 0xf0100000
0xf0100000 <_start-268435468>:  Cannot access memory at address 0xf0100000
(gdb) si
=> 0x100028:    mov    $0xf010002f,%eax
0x00100028 in ?? ()
(gdb) x/10x 0xf0100000
0xf0100000 <_start-268435468>:  0x1badb002      0x00000000      0xe4524ffe      0x7205c766
0xf0100010 <entry+4>:   0x34000004      0x1000b812      0x220f0011      0xc0200fd8
0xf0100020 <entry+20>:  0x0100010d      0xc0220f80
```

把这一行，即`mov %eax, %cr0`注释掉，重新跟踪一遍，发现下面的结果：

```sh
=> 0x10002a:    jmp    *%eax
0x0010002a in ?? ()
(gdb) x/10x 0xf010002c
0xf010002c <relocated>: Cannot access memory at address 0xf010002c
(gdb) si
=> 0xf010002c <relocated>:      Error while running hook_stop:
Cannot access memory at address 0xf010002c
relocated () at kern/entry.S:74
74              movl    $0x0,%ebp                       # nuke frame pointer
(gdb)
```

在CR0没有开启保护模式下出现了这样的结局，因为并没有把原本位于物理地址`0x00100000`附近的指令给装载过来，那如果想要跳转过来运行的话，不出问题才怪呢。

### Exercise 8: Format Printing.

We have omitted a small fragment of code - the code necessary to print octal numbers using patterns of the form "%o". Find and fill in this code fragment.

这个练习甚至不用读代码，抄一下其它case就完事了，如果要加颜色看一下ANSI就好了，总之很简单。

* Explain the interface between `printf.c` and `console.c`. Specifically, what function does `console.c` export? How is this function used by `printf.c`?

`console.c`文件为`printf.c`文件提供了`cputchar`, `vprintfmt`这样的函数。

根据源码分析，在完善终端输出字符串这一过程中，需要完成序列化初始化，这一部分我在代码里做了简要的注释，然而参考资料过少确实也让解读起来摸不着头脑，为什么要按照这样的顺序做更是完全不知道，不过老旧的设备也有令人觉得有趣的地方，比如读写共享寄存器作为`buffer`之类的玩意儿。

`Parallel port`部分的链接全都挂了，不过在OSDEV上能够找到简要的介绍，其流程与console.c中所示的很接近。

{% embed url="https://wiki.osdev.org/Parallel_port#Centronics_Handshaking" %}
Parallel port part
{% endembed %}

对于`CGA port`部分的解读请参考这一部分资料，大致可以猜猜看他在干什么：

{% embed url="https://pdos.csail.mit.edu/6.828/2018/readings/hardware/vgadoc/CGA.TXT" %}
CGA text
{% endembed %}

对于`cga_init`初始化函数，首先似乎有一层判断，判断是否是常规的`CGA`模式还是`MONO`模式，之后再对照手册校准`cursor`位置，不过只能说大意如此，全都是坑。

在console.c中我们可以注意到`cga_putc`与`cons_putc`的交叉递归，只需要考虑几个特殊`case`就能想清楚这个递归的调用顺序了。这里涉及部分与换行相关的特殊字符的处理，值得学习。

* Explain the following from `console.c`:

```c
1      if (crt_pos >= CRT_SIZE) {
2              int i;
3              memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t));
4              for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
5                      crt_buf[i] = 0x0700 | ' ';
6              crt_pos -= CRT_COLS;
7      }
```

解读到这里答案便是呼之欲出了，如果`crt_pos`大于等于页面大小`CRT_SIZE`，则需要准备往下继续添加了，同时把最上面一行去掉，因为这个页面已经装不下了。我们查看`memmove`函数，发现恰如其分地完成了这个需求：

```c
  8 void *
  7 memmove(void *dst, const void *src, size_t n)
  6 {
  5     const char *s;
  4     char *d;
  3
  2     s = src;
  1     d = dst;
195     if (s < d && s + n > d) {
  1         s += n;
  2         d += n;
  3         while (n-- > 0)
  4             *--d = *--s;
  5     } else // our case.
  6         while (n-- > 0)
  7             *d++ = *s++; // copy from the first pixel. remove the first line of pixels of last plane.
  8
  9     return dst;
 10 }
```

最后一行这时候应该显示为空，所以我们用了一个`for`循环进行赋值。有一说一这也过于简陋了。`crt_pos`这时候也该后退一下。第二题结束。

我们到这里也确实完成`kern/printf.c`中的`putch`函数的口胡解读。

下面尝试解读函数`vcprintf`：其中调用了函数`vprintfmt`，我们追溯这个函数。

```c
 5 int
  4 vcprintf(const char *fmt, va_list ap)
  3 {
  2     int cnt = 0;
  1
21      vprintfmt((void*)putch, &cnt, fmt, ap);
  1     return cnt;
  2
```

函数太长，本人的注释写在愿文件中，建议看一下源文件。特别的，va\_list相关的操作可参考下面的资料：

{% embed url="https://learn.microsoft.com/en-us/cpp/c-runtime-library/reference/va-arg-va-copy-va-end-va-start?view=msvc-170" %}
va\_list info
{% endembed %}

{% embed url="https://en.cppreference.com/w/c/variadic/va_arg" %}
usage
{% endembed %}

具体来说，`va_list`相关的宏被广泛使用在非定长的函数中。在实现`vprintfmt`时，请参考原本`printf`所示的资料。令人感叹，`printf`居然还有这么多功能。

{% embed url="https://www.runoob.com/cprogramming/c-function-printf.html" %}
printf function
{% endembed %}

有上面的基础后，我们看一下习题：

* Trace the execution of the following code step-by-step:

```c
int x = 1, y = 3, z = 4;
cprintf("x %d, y %x, z %d\n", x, y, z);
```

* In the call to `cprintf()`, to what does `fmt` point? To what does `ap` point?

`fmt` points to `"x %d, y %x, z %d\n"`, `ap` points to the list of `[x, y, z]`.

* List (in order of execution) each call to `cons_putc`, `va_arg`, and `vcprintf`. For `cons_putc`, list its argument as well. For `va_arg`, list what `ap` points to before and after the call. For `vcprintf` list the values of its two arguments.

我们追踪启动时的代码，把上面的代码放在`i386_init`函数内部，即可利用`gdb`调用查看。调用结果如下展示：`cputchar`确实会一个一个地把字符输出并且拼接。把这个东东跟完似乎没有太大意义，我们就跟一个吧。

```sh
=> 0xf0100a51 <cprintf+6>:      lea    0xc(%ebp),%eax
cprintf (fmt=0xf0101a77 "x %d, y %x, z %d\n") at kern/printf.c:
31     va_start(ap, fmt);
gdb) p fmt
$1 = 0xf0101a77 "x %d, y %x, z %d\n"

vcprintf (fmt=0xf0101a77 "x %d, y %x, z %d\n", ap=0xf010efd4 "\001")
// let's check ap!
(gdb) x/8x 0xf010efd4
 0xf010efd4:     0x00000001 (x)     0x00000003 (y)     0x00000004 (z)     0xf0112060
 0xf010efe4:     0x00000000      0x00000660      0x00000000      0x00000000 

vprintfmt (putch=0xf01009f2 <putch>, putdat=0xf010ef9c, fmt=0xf0101a77 "x %d, y %x, z %d\n", ap=0xf010efd4 "\001")

va_arg(*ap, int); // *ap = 1, which is x.

printnum (putch=0xf01009f2 <putch>, putdat=0xf010ef9c, num=1, base=10, width=-1, padc=32)
// successfully print 1 at end!!
// SKIP!!
```

在跟踪这个过程中感觉这个代码写得真漂亮啊，可惜我写不来。

第三题结束。

* 第四题修改一下跑一下即可，

```c
    unsigned int i = 0x00646c72;
    cprintf("H%x Wo%s", 57616, &i); // 57616 -> e110. 
    // i: little end. treated as string: 72 6c 64 00 -> rld\0
```

倒是挺有黑客的意思。

*   In the following code, what is going to be printed after `'y='`? (note: the answer is not a specific value.) Why does this happen?

    ```
        cprintf("x=%d y=%d", 3);
    ```

```sh
vcprintf (fmt=0xf0101a77 "x=%d y=%d", ap=0xf010efe4 "\003") at kern/printf.│
c:19                                                                       │
19              int cnt = 0;                                               │
(gdb) p ap                                                                 │
$2 = (va_list) 0xf010efe4 "\003"                                           │
(gdb) p *ap                                                                │
$3 = 3 '\003'                                                              │
(gdb) p *(ap + 1)                                                          │
$4 = 0 '\000'                                                              │
(gdb) x/2x ap                                                              │
0xf010efe4:     0x00000003      0x00000660

(gdb) p num                                                                │
$6 = 1632                                                                  │
(gdb) n                                                                    │
=> 0xf0101263 <vprintfmt+1044>: add    $0x20,%esp                          │
234                             break;                                     │
(gdb)                                 
```

通过`gdb`观察，y的值完全取决于`ap list`后一项到底是什么东东。这玩意由于不受控制，完全是个随机的东东。`1632`其实就是`0x660`。

* Let's say that GCC changed its calling convention so that it pushed arguments on the stack in declaration order, so that the last argument is pushed last. How would you have to change `cprintf` or its interface so that it would still be possible to pass it a variable number of arguments?

这个问题其实是个不那么`trivial`的问题，首先，为什么`GCC`调用规范要要求从最后一个参数开始往栈内压数据呢？有很多网上的答案说因为变长参数函数不知道到底有多少参数，所以要用这个方式来做。但实际上这句话只说对了一半。在我们大部分情况下使用`printf`这样的函数时，编译器完全可以很轻松地把参数个数搞到。

真正的应用场景其实是在我们调用动态库利用里面的函数的时候，在加载它们进入内存并与我们的文件建立链接、重定位并且运行之前，没有人知道这个函数长什么样，有几个参数。

那么，应该怎么做才能打破这个调用规范呢？我想到了一个很笨笨的方法，手动拆分掉我们的`va_list`，转而调用很多很多个一次只能解析一个参数的函数。也就是说，原本我们是对一个函数分配不确定的栈，现在我们用很多函数分配确定的栈空间，至于函数有多少个，我们用一个`loop`就好了捏。

### Exercise 9: About Stack.

Determine where the kernel initializes its stack, and exactly where in memory its stack is located. How does the kernel reserve space for its stack? And at which "end" of this reserved area is the stack pointer initialized to point to?

最早内核是在文件`kern/entry.S`中实现内核栈的建立。具体代码如下：

```nasm
  8 relocated:
  7
  6     # Clear the frame pointer register (EBP)
  5     # so that once we get into debugging C code,
  4     # stack backtraces will be terminated properly.
  3     movl    $0x0,%ebp           # nuke frame pointer
  2
  1     # Set the stack pointer
77      movl    $(bootstacktop),%esp
  1
  2     # now to C code
  3     call    i386_init
  4
  5     # Should never get here, but in case we do, just spin.
  6 spin:   jmp spin
  7
  8
  9 .data
 10 ###################################################################
 11 # boot stack
 12 ###################################################################
 13     .p2align    PGSHIFT     # force page alignment
 14     .globl      bootstack
 15 bootstack:
 16     .space      KSTKSIZE    # Kernel Stack Size ensurance.
 17     .globl      bootstacktop
 18 bootstacktop:N
```

其中`KSTKSIZE`是内核栈的大小，我们可以看到这个栈的建立与存在，其栈底为0，栈顶为`bootstacktop`所指向的地址。

### Exercise 10: backtrace stack.

To become familiar with the C calling conventions on the x86, find the address of the `test_backtrace` function in obj/kern/kernel.asm, set a breakpoint there, and examine what happens each time it gets called after the kernel starts. How many 32-bit words does each recursive nesting level of `test_backtrace` push on the stack, and what are those words?

这个问题需要调试来得知，不过感觉效果确实不错。

我们利用断点打在函数`test_backtrace`之上，可以看到return address应该是`0xf01000f4`。与此同时，来看一下此时的函数`ebp`与`esp`信息和栈之间存储的信息。

```nasm
   3     test_backtrace(5);
   2 f01000e8:   c7 04 24 05 00 00 00    movl   $0x5,(%esp)
   1 f01000ef:   e8 4c ff ff ff          call   f0100040 <test_backtrace>
169  f01000f4:   83 c4 10                add    $0x10,%esp



   4     # now to C code
   3     call    i386_init
   2 f0100039:   e8 68 00 00 00          call   f01000a6 <i386_init>
   1
64   f010003e <spin>:
   1
   2     # Should never get here, but in case we do, just spin.
   3 spin:   jmp spin
   4 f010003e:   eb fe                   jmp    f010003e <spin>
```

```sh
=> 0xf0100040 <test_backtrace>: push   %ebp
Breakpoint 2, test_backtrace (x=5) at kern/init.c:13
13      {
(gdb) i r ebp
ebp            0xf010eff8          0xf010eff8
(gdb) i r esp                                                                                                                                       
esp            0xf010efdc          0xf010efdc                                                                                                                                                                   
(gdb) i r eip
eip            0xf0100040          0xf0100040 <test_backtrace>
(gdb) x/8x 0xf010efdc                                                                                   
0xf010efdc:     0xf01000f4      0x00000005      0x00001aac      0x00000660                              
0xf010efec:     0x00000000      0x00000000      0x00010094      0x00000000
(gdb) x *0xf010effc                                                                                    
0xf010003e:     0x8955feeb

```

在运行`test_backtrace`函数之前，栈信息尚为`i386_init`的函数栈信息，我们可以看到`5`被压入了函数栈中，但这并不重要。注意到`ebp + 4`其实是返回地址，打印以后发现确实如此。

进入函数后，继续查看：我们做好记号，运用之妙令人感叹。

```sh
# test_backtrace(5).
(gdb) i r ebp                                                                                           │
ebp            0xf010efd8          0xf010efd8                                                           │
(gdb) i r esp                                                                                           │
esp            0xf010efd0          0xf010efd0
(gdb) x/4x 0xf010efd0                                                                                   │
0xf010efd0:     0xf0110308      0x00010094      0xf010eff8      0xf01000f4
# | 0xf01000f4 | <- return address. -> last function stack frame.
# | - - - - - -|
# | 0xf010eff8 | <- ebp  - ---> store the last ebp here.
# | 0x00010094 |           \
# | 0xf0110308 | <- esp  - - -> this function stack frame.

   4     // Test the stack backtrace function (lab 1 only)
   3     test_backtrace(5);
   2 f01000e8:   c7 04 24 05 00 00 00    movl   $0x5,(%esp)
   1 f01000ef:   e8 4c ff ff ff          call   f0100040 <test_backtrace>
169  f01000f4:   83 c4 10                add    $0x10,%esp # return address.
# test_backtrace(4).
# 刚进入时
(gdb) i r ebp                                                                                           │
ebp            0xf010efb8          0xf010efb8                                                           │
(gdb) i r esp                                                                                           │
esp            0xf010efb0          0xf010efb0                                                           │
(gdb) x/4x 0xf010efb0                                                                                   │
0xf010efb0:     0xf0110308      0x00000005      0xf010efd8      0xf0100076
# very similar to test_backtrace(5)

# Why not consistent? -> 'cprintf' is also included, the 'parameter' , i.e. 4 and 5, can be found in between
# on the function stack frame.
(gdb) x/16x 0xf010efb0                                                                                  │
0xf010efb0:     0xf0110308      0x00000005      0xf010efd8      
                                                4_ebp.
                                                                0xf0100076                              │
0xf010efc0:     0x00000004      0x00000005      0x00000000      0xf010004a                              │
0xf010efd0:     0xf0110308      0x00010094      0xf010eff8      
                                                5_ebp.
                                                                0xf01000f4                              │
0xf010efe0:     0x00000005      0x00001aac      0x00000660      0x00000000
```

所以可以看出来其实会有`8`个`32-bit`长度的字被存放进去了。从`asm`代码中可以看到此处是离开递归的入口。

```nasm
99   f0100076:   83 c4 10                add    $0x10,%esp
   1     else
   2         mon_backtrace(0, 0, 0);
   3     cprintf("leaving test_backtrace %d\n", x);
   4 f0100079:   83 ec 08                sub    $0x8,%esp
```

虽然跟踪很好玩，但也很累人，不是特别清楚一些函数参数如何压栈，因此仍需要直接了当的方式来处理。

### Exercise 11: back trace stack frame

Implement the backtrace function as specified above. Use the same format as in the example, since otherwise the grading script will be confused. When you think you have it working right, run make grade to see if its output conforms to what our grading script expects, and fix it if it doesn't. _After_ you have handed in your Lab 1 code, you are welcome to change the output format of the backtrace function any way you like.

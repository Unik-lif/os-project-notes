---
description: bootstrap of xv6 lab1.
---

# Project 3: JOS lab1

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

* At what point does the processor start executing 32-bit code? What exactly causes the switch from 16-bit to 32-bit mode?
  * `ljmp $PROT_MODE_CSEG, $protcseg`
* What is the _last_ instruction of the boot loader executed, and what is the _first_ instruction of the kernel it just loaded?
  * `movw $0x1234,0x472`
* _Where_ is the first instruction of the kernel?
  * `movw $0x1234,0x472`
* How does the boot loader decide how many sectors it must read in order to fetch the entire kernel from disk? Where does it find this information?
  * I guess these messages are gained from ELF.

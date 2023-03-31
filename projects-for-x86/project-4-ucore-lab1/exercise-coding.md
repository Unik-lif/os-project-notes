---
description: 剩下的一些coding练习
---

# Exercise-Coding

### 练习五：

**实现函数调用堆栈跟踪函数**

这个任务原则上并不太需要阅读很多代码，但出于学习还是尝试读完了，感觉收获还是蛮大的。

首先，代码定义了名为`stabs`的数据结构，它以数组的方式进行存储，该数据结构在`GDB`之中得到了广泛的应用，可以用于读取符号表内的行、函数、文件信息，是一个很强大的跟踪用数据结构。该数据结构如下方代码块所示，其中`n_strx`是存放名字信息很重要的组件，而`n_type`定义了符号的类型，主流类型包括源代码类型、函数类型、行类型等等，具体可以查看源码文件进行学习。

此外，为了更好完成信息的存储和传递，该文件内设计了`eipdebuginfo`函数，其中会存放某地址`addr`或者`ip`对应的文件信息、行信息、函数信息等等，他们在之后会发挥很好的功效。

```c
  7 #define STACKFRAME_DEPTH 20
  8
  9 extern const struct stab __STAB_BEGIN__[];  // beginning of stabs table
 10 extern const struct stab __STAB_END__[];    // end of stabs table
 11 extern const char __STABSTR_BEGIN__[];      // beginning of string table
 12 extern const char __STABSTR_END__[];        // end of string table
 13
 14 /* debug information about a particular instruction pointer */
 15 struct eipdebuginfo {
 16     const char *eip_file;                   // source code filename for eip
 17     int eip_line;                           // source code line number for eip
 18     const char *eip_fn_name;                // name of function containing eip
 19     int eip_fn_namelen;                     // length of function's name
 20     uintptr_t eip_fn_addr;                  // start address of function
 21     int eip_fn_narg;                        // number of function arguments
 22 };
 
   9 /* Entries in the STABS table are formatted as follows. */
  8 struct stab {
  7     uint32_t n_strx;        // index into string table of name
  6     uint8_t n_type;         // type of symbol
  5     uint8_t n_other;        // misc info (usually empty)
  4     uint16_t n_desc;        // description field
  3     uintptr_t n_value;      // value of symbol
  2 };
```

下面是本文件的核心函数，我在原本代码的基础上添加了更多的注释以方便读者理解。简言之，`stab_binsearch`是一个根据`addr`信息在`stabs`数组中全部块内，根据`type`类型，利用二分查找策略寻找能够包住addr的左`stab`项值和右`stab`项值。这边的工作逻辑是，首先找到确实能够包住`type`类型`addr`块的左边界和右边界，找到边界之后再尽可能缩小这个区间。因为二分查找确实会略过一些数据，所以有必要在最后进行一次缩小区间的小型遍历，以锁定最小范围。

```c
66  static void
  1 stab_binsearch(const struct stab *stabs, int *region_left, int *region_right,
  2            int type, uintptr_t addr) {
  3     int l = *region_left, r = *region_right, any_matches = 0;
  4
  5     while (l <= r) {
  6         int true_m = (l + r) / 2, m = true_m;
  7
  8         // search for earliest stab with right type
  9         // when fail at the end, l will become bigger than r.
 10         // The logic here is simply to find the first m that suitable for our type.
 11         while (m >= l && stabs[m].n_type != type) {
 12             m --;
 13         }
 14         if (m < l) {    // no match in [l, m]
 15             l = true_m + 1;
 16             continue;
 17         }
 18
 19         // actual binary search
 20         // For this while loop:
 21         // 1. We assure the m is of right type. -> that will be computed for region_left or region_right.
 22         // 2. We use binsearch to accelerate the speed to find m, we ensure at least one is smaller than addr,
 23         // one is bigger than addr, but not ensure this m is the closest to the addr. When we are using binsearch,
 24         // we implicitly omit some l and some r values. What we do here is simply found the right&left bounds.
 25         any_matches = 1;
 26         if (stabs[m].n_value < addr) {
 27             *region_left = m;
 28             l = true_m + 1;
 29         } else if (stabs[m].n_value > addr) {
 30             *region_right = m - 1;
 31             r = m - 1;
 32         } else {
 33             // exact match for 'addr', but continue loop to find
 34             // *region_right
 35             *region_left = m;
 36             l = m;
 37             addr ++;
 38         }
 39     }
 107    if (!any_matches) {
  1         *region_right = *region_left - 1;
  2     }
  3     else {
  4         // find rightmost region containing 'addr'
  5         // Now we want to have a closer result.
  6         l = *region_right;
  7         for (; l > *region_left && stabs[l].n_type != type; l --)
  8             /* do nothing */;
  9         *region_left = l;
 10     }
 11 }
```

在这里我对`stabs`机制也有一些疑惑，其实只要简单利用`objdump -G`来看一看。对照这个表格，函数`debuginfo_eip`就差不多能够明白了。简单来说，这个函数实现了文件、函数、代码行的信息追踪和解析。

```c
bin/kernel:     file format elf32-i386

Contents of .stab section:

Symnum n_type n_othr n_desc n_value  n_strx String

31     FUN    0      0      00100093 236    grade_backtrace1:F(0,4)
32     PSYM   0      0      00000008 188    arg0:p(0,1)
33     PSYM   0      0      0000000c 200    arg1:p(0,1)
34     SLINE  0      52     00000000 0
35     SLINE  0      53     00000009 0
36     SLINE  0      54     00000029 0
37     FUN    0      0      001000c4 260    grade_backtrace0:F(0,4)
38     PSYM   0      0      00000008 188    arg0:p(0,1)
39     PSYM   0      0      0000000c 200    arg1:p(0,1)
40     PSYM   0      0      00000010 212    arg2:p(0,1)
41     SLINE  0      57     00000000 0
42     SLINE  0      58     00000006 0
43     SLINE  0      59     00000018 0
44     FUN    0      0      001000e1 284    grade_backtrace:F(0,4)
45     SLINE  0      62     00000000 0
46     SLINE  0      63     00000006 0
47     SLINE  0      64     00000023 0
48     FUN    0      0      00100109 307    lab1_print_cur_status:f(0,4)
```

`n_value`说白了就是地址偏移量，且`stabs`是一个地址增长的排列，这就给二分查找带来了可能。特别的，如果`n_type`是`SLINE`，其`n_value`会减去其所对应的`Function`偏移量。如果`n_type`是`FUN`，`String`中会利用`:`来帮助分割其函数名。利用这些信息，可以较为清楚地解读`debuginfo_eip`函数。由于冗长且细节繁多，笔者直接做的代码间注释，其余部分姑且不表。

`print_debuginfo`函数则是一个换皮打印函数，能够打印存在`info`内的基本信息。下面看到函数read\_eip，这个函数特地设置成了`__noinline`，这是个很关键的细节，我们暂时不谈。

```c
  4 print_debuginfo(uintptr_t eip) {
  5     struct eipdebuginfo info;
  6     if (debuginfo_eip(eip, &info) != 0) {
  7         cprintf("    <unknow>: -- 0x%08x --\n", eip);
  8     }
  9     else {
 10         char fnname[256];
 11         int j;
 12         for (j = 0; j < info.eip_fn_namelen; j ++) {
 13             fnname[j] = info.eip_fn_name[j];
 14         }
 15         fnname[j] = '\0';
 16         cprintf("    %s:%d: %s+%d\n", info.eip_file, info.eip_line,
 17                 fnname, eip - info.eip_fn_addr);
 18     }
 19 }
 20
 21 static __noinline uint32_t
 22 read_eip(void) {
 23     uint32_t eip;
 24     asm volatile("movl %%ebp, %0" : "=r" (eip));
 25     return eip;
 26 }
```

下面是本人对`print_stackframe`函数的一个简单实现：这个原理其实很简单，通过函数栈帧不断向更高的地址内迁移，可以查看`Ucore`官方文档进行学习。

为什么一开始对于`eip`一定要调用函数`read_eip`来呢？注意到函数`read_eip`是`non_inline`的，这说明一定会通过间接跳转进入该函数，而此时再去看`return address`，就能恰好找到`print_stackframe`函数当前的`eip`值。

之后就正常流程跑跑就好了，非常轻松。

```c
 31 void
 30 print_stackframe(void) {
 29      /* LAB1 YOUR CODE : STEP 1 */
 28      /* (1) call read_ebp() to get the value of ebp. the type is (uint32_t);
 27       * (2) call read_eip() to get the value of eip. the type is (uint32_t);
 26       * (3) from 0 .. STACKFRAME_DEPTH
 25       *    (3.1) printf value of ebp, eip
 24       *    (3.2) (uint32_t)calling arguments [0..4] = the contents in address (uint32_t)ebp +2 [0..4]
 23       *    (3.3) cprintf("\n");
 22       *    (3.4) call print_debuginfo(eip-1) to print the C calling function name and line number, etc.
 21       *    (3.5) popup a calling stackframe
 20       *           NOTICE: the calling funciton's return addr eip  = ss:[ebp+4]
 19       *                   the calling funciton's ebp = ss:[ebp]
 18       */
 17     int time = 0;
 16     uint32_t ebp = read_ebp(); // | return address  | <-- caller's eip. read_ebp will simply return ebp value.
 15                                // - - - - - - - - - -
 14     uint32_t eip = read_eip(); // |   last  ebp     | <-- ebp point to this position. read_eip will load the memory 4(%%ebp).
 13     // BTW: the read_eip function here is very tricky!! Understand this thing is of great fun!
 12
 11     // End point: when ebp is equal to 0, then our stackframe chain ends.
 10     while (ebp != 0 && time < STACKFRAME_DEPTH) {
  9         cprintf("ebp:%08x eip:%08x ", ebp, eip);
  8         uint32_t* args = (uint32_t*) ebp + 2;
  7         cprintf("args:%08x %08x %08x %08x\n", args[0], args[1], args[2], args[3]);
  6         // eip - 1 is sufficient for us to use binsearch to locate the line.
  5         print_debuginfo(eip - 1);
  4         eip = *((uint32_t*) ebp + 1);
  3         ebp = *((uint32_t*) ebp);
  2         time++;
  1     }
331 }
```

### 练习六：

1. 中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？
2. 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt\_init。在idt\_init函数中，依次对所有中断入口进行初始化。使用mmu.h中的SETGATE宏，填充idt数组内容。每个中断的入口由tools/vectors.c生成，使用trap.c中声明的vectors数组即可。
3. 请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print\_ticks子程序，向屏幕上打印一行文字”100 ticks”。

中断描述符表的一个表项占据8个字节，如下所示：

<figure><img src="../../.gitbook/assets/Screenshot 2023-03-28 103843.png" alt=""><figcaption><p>中断描述符表</p></figcaption></figure>

其中`Selector`可以帮助在`LDT`或者`GDT`表中找到对应的基地址，而`Offset`可以作为偏移量。二者进行相加后，就能得到最终的中断处理代码的入口。

后面的两个练习，第一个要比第二个麻烦的多。第二个我开摆了，就按照他说的这样做吧，暂时不细究细节了。

首先，我们会发现`idt_init`这个函数会在`init.c`函数中出现，`init.c`则是在链接中第一个装上的目标文件，这和一开始利用`linker.ld`链接的顺序是一致的。

```c
 15 int
 16 kern_init(void) {
 17     extern char edata[], end[];
 18     memset(edata, 0, end - edata);
 19
 20     cons_init();                // init the console
 21
 22     const char *message = "(THU.CST) os is loading ...";
 23     cprintf("%s\n\n", message);
 24
 25     print_kerninfo();
 26
 27     grade_backtrace();
 28
 29     pmm_init();                 // init physical memory management
 30
 31     pic_init();                 // init interrupt controller
 32     idt_init();                 // init interrupt descriptor table
 33
 34     clock_init();               // init clock interrupt
 35     intr_enable();              // enable irq interrupt
 36
 37     //LAB1: CAHLLENGE 1 If you try to do it, uncomment lab1_switch_test()
 38     // user/kernel mode switch test
 39     //lab1_switch_test();
 40
 41     /* do nothing */
 42     while (1);
 43 }
```

于是`idt_init`函数的目的就昭然若揭：需要利用`vectors`对中断表做一个简单的初始化，我们看向这个`vectors`的结构，在`vectors.S`之中。

```nasm
   2 # vector table
   1 .data
1280 .globl __vectors
   1 __vectors:
   2   .long vector0
   3   .long vector1
   4   .long vector2
   5   .long vector3
```

可以看到所有中断向量存放在`__vectors`这一数组之中，他们存放的位置是在数据段上，并且可以简单地通过4字节偏移地址去访问这些向量。那么，这些向量到底存放了什么东西？查看该汇编文件的代码段信息，我们不难发现这样一个事实：

```nasm
   1 # handler
2    .text
   1 .globl __alltraps
   2 .globl vector0
   3 vector0:
   4   pushl $0
   5   pushl $0
   6   jmp __alltraps
   7 .globl vector1
   8 vector1:
   9   pushl $0
  10   pushl $1
  11   jmp __alltraps
  12 .globl vector2
  13 vector2:
  14   pushl $0
  15   pushl $2
  16   jmp __alltraps
```

在简单地进行压栈等特质化操作后，控制流将会跳转到`__alltraps`处。`__alltraps`最早被定义在文件`kern/trap/trapentry.S`之中，以此开始针对各自情况进行异常处理，你会看到压栈和`CALL`的操作。

不过这里并不是我们的重点，暂时忽略。

为了实现初始化，题目建议采用宏的方式进行逐项处理（具体请看原题的注释）：

```nasm
  12 /* *
 13  * Set up a normal interrupt/trap gate descriptor
 14  *   - istrap: 1 for a trap (= exception) gate, 0 for an interrupt gate
 15  *   - sel: Code segment selector for interrupt/trap handler
 16  *   - off: Offset in code segment for interrupt/trap handler
 17  *   - dpl: Descriptor Privilege Level - the privilege level required
 18  *          for software to invoke this interrupt/trap gate explicitly
 19  *          using an int instruction.
 20  * */
 
 27 #define SETGATE(gate, istrap, sel, off, dpl) {            \
 26     (gate).gd_off_15_0 = (uint32_t)(off) & 0xffff;        \
 25     (gate).gd_ss = (sel);                                \
 24     (gate).gd_args = 0;                                    \
 23     (gate).gd_rsv1 = 0;                                    \
 22     (gate).gd_type = (istrap) ? STS_TG32 : STS_IG32;    \
 21     (gate).gd_s = 0;                                    \
 20     (gate).gd_dpl = (dpl);                                \
 19     (gate).gd_p = 1;                                    \
 18     (gate).gd_off_31_16 = (uint32_t)(off) >> 16;        \
 17 }
```

本质上这是对`IDT`表格中的项进行处理的一个流程，根据我们在前置资料中的笔记，理解他们是不难的。特别的，`istrap`在我们题目的情况下是可以置零，不过还有一个问题，我们需要找到选择子以及偏移量，但现在我们拥有的条件只有`vector`本身的地址信息，这是不足够的，因为它其实只不过就是个偏移量罢了。

于是我们需要寻找任何和选择子有关的东东，找了一下发现这玩意在`memlayout.h`里头。我们看到下面这些规定：

```nasm
  9 #ifndef __KERN_MM_MEMLAYOUT_H__
  8 #define __KERN_MM_MEMLAYOUT_H__
  7
  6 /* This file contains the definitions for memory management in our OS. */
  5
  4 /* global segment number */
  3 #define SEG_KTEXT    1
  2 #define SEG_KDATA    2
  1 #define SEG_UTEXT    3
10  #define SEG_UDATA    4
  1 #define SEG_TSS        5
  2
  3 /* global descriptor numbers */
  4 #define GD_KTEXT    ((SEG_KTEXT) << 3)        // kernel text
  5 #define GD_KDATA    ((SEG_KDATA) << 3)        // kernel data
  6 #define GD_UTEXT    ((SEG_UTEXT) << 3)        // user text
  7 #define GD_UDATA    ((SEG_UDATA) << 3)        // user data
  8 #define GD_TSS        ((SEG_TSS) << 3)        // task segment selector
```

考虑到现在电脑才刚启动，分权机制也没有添加，所以答案就很简单了。我们的选择子应该是`SEG_KTEXT`，用以跳转到`vector`所指向的中断处理代码。

解答：

```c
 /* idt_init - initialize IDT to each of the entry points in kern/trap/vectors.S */
  4 void
  5 idt_init(void) {
  6      /* LAB1 YOUR CODE : STEP 2 */
  7      /* (1) Where are the entry addrs of each Interrupt Service Routine (ISR)?
  8       *     All ISR's entry addrs are stored in __vectors. where is uintptr_t __vectors[] ?
  9       *     __vectors[] is in kern/trap/vector.S which is produced by tools/vector.c
 10       *     (try "make" command in lab1, then you will find vector.S in kern/trap DIR)
 11       *     You can use  "extern uintptr_t __vectors[];" to define this extern variable which will be used later.
 12       * (2) Now you should setup the entries of ISR in Interrupt Description Table (IDT).
 13       *     Can you see idt[256] in this file? Yes, it's IDT! you can use SETGATE macro to setup each item of IDT
 14       * (3) After setup the contents of IDT, you will let CPU know where is the IDT by using 'lidt' instruction.
 15       *     You don't know the meaning of this instruction? just google it! and check the libs/x86.h to know more.
 16       *     Notice: the argument of lidt is idt_pd. try to find it!
 17       */
 18
 19     // call extern global variabels __vectors.
 20     extern uintptr_t __vectors[];
 21
 22     // use __vectors to initialize the idt entry.
 23     for (int i = 0; i < 256; i++) {
 24         SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], 0);
 25     }
 26
 27     // lidt instruction to set the IDT with IDTR.
 28     lidt(&idt_pd);
 29 }
```

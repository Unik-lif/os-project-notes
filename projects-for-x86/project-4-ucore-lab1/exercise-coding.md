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


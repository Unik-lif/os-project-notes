---
description: boot xv6
---

# Project 4: xv6-boot

这玩意要比`JOS`简单多了，代码量要小好多捏。

然而上次更新已经是两个月前的事情了，相较于之前，我已经把rCore卓完一遍了，但是x86本身还是有很多非常麻烦的地方，我们还是把节奏放慢，好好看一下这个项目吧。

拿到一个项目，首先应该看`Makefile`文件：

在其中的TOOLPREFIX和QEMU参数的环节上设置了一些相关的参数judge，具体可以问chatgpt其中一些参数的设置妙用，比如2>&1这个将标准错误同样作为标准输出来打印的方式。

```makefile
xv6memfs.img: bootblock kernelmemfs
	dd if=/dev/zero of=xv6memfs.img count=10000
	dd if=bootblock of=xv6memfs.img conv=notrunc
	dd if=kernelmemfs of=xv6memfs.img seek=1 conv=notrunc
```

从这边镜像的映射方式，我们可以得知最低的一部分地址空间将会被设置为0，之后紧跟着的是bootblock和kernelmemfs相关的信息，他们会作为镜像的一部分载入到镜像的这个位置上。

进来之后，这个流程与JOS只能说完全一致，我们之后会进入bootmain函数之中，这边就是bootloader了，通过ELF的装载和检验之后，将会运行elf->entry位置的程序，这对应的部分其实就是内核部分了。

我们跟踪，会发现进入内核后起始位置是在entry.S处的\_start，这一点从读取Makefile信息也可以得出，至于为什么是从start段开始，这是ELF的一个convention.

```nasm
# By convention, the _start symbol specifies the ELF entry point.
# Since we haven't set up virtual memory yet, our entry point is
# the physical address of 'entry'.
.globl _start
_start = V2P_WO(entry)

# Entering xv6 on boot processor, with paging off.
.globl entry
entry:
  # Turn on page size extension for 4Mbyte pages
  movl    %cr4, %eax
  orl     $(CR4_PSE), %eax
  movl    %eax, %cr4
  # Set page directory
  movl    $(V2P_WO(entrypgdir)), %eax
  movl    %eax, %cr3
  # Turn on paging.
  movl    %cr0, %eax
  orl     $(CR0_PG|CR0_WP), %eax
  movl    %eax, %cr0

  # Set up the stack pointer.
  movl $(stack + KSTACKSIZE), %esp

  # Jump to main(), and switch to executing at
  # high addresses. The indirect call is needed because
  # the assembler produces a PC-relative instruction
  # for a direct jump.
  mov $main, %eax
  jmp *%eax

.comm stack, KSTACKSIZE

```

在这边我们姑且不再看究竟干了什么，源码的阅读我希望放在后面一些些，首先解决这边的作业题目：

* Begin by restarting qemu and gdb, and set a break-point at 0x7c00, the start of the boot block (bootasm.S). Single step through the instructions (type si at the gdb prompt). Where in bootasm.S is the stack pointer initialized? (Single step until you see an instruction that moves a value into %esp, the register for the stack pointer.)\
  最早是在这个地方进行初始化的，设置esp为0x7c00。

```
movl    $start, %esp
//----------------------------------------
(gdb) i r ebp
ebp            0x0                 0x0
(gdb) i r esp
esp            0x7bfc              0x7bfc
```

在检查.asm文件的时候，Makefile文件中也提供了一个小技巧，利用objdump -S来获取一个比较漂亮的反汇编结果，这个返回结果的可读性非常好。

之后，进入bootmain函数之中，在这边额外调用了一些push和pop的命令，以实现函数的调用，比如readseg函数的调用，需要事先将0,4096，elf起始地址0x10000塞进去。

```
=> 0x7d4d:      push   $0x10000 # esp 本身当前指向的位置是有意义的
0x00007d4d in ?? ()
(gdb) x/24x $esp
0x7bd4: 0x00001000      0x00000000      0x00000000      0x00000000
0x7be4: 0x00000000      0x00000000      0x00000000      0x00000000
0x7bf4: 0x00000000      0x00000000      0x00007c4d      0x8ec031fa
0x7c04: 0x8ec08ed8      0xa864e4d0      0xb0fa7502      0xe464e6d1
0x7c14: 0x7502a864      0xe6dfb0fa      0x16010f60      0x200f7c78
0x7c24: 0xc88366c0      0xc0220f01      0x087c31ea      0x10b86600
(gdb) i r esp
esp            0x7bd4              0x7bd4
```



* Single step through the call to bootmain; what is on the stack now?
* What do the first assembly instructions of bootmain do to the stack? Look for bootmain in bootblock.asm.

这也没啥好找的，本质上跟一下就好了

* Continue tracing via gdb (using breakpoints if necessary -- see hint below) and look for the call that changes eip to 0x10000c. What does that call do to the stack? (Hint: Think about what this call is trying to accomplish in the boot sequence and try to identify this point in bootmain.c, and the corresponding instruction in the bootmain code in bootblock.asm. This might help you set suitable breakpoints to speed things up.)

说白了就是那个entry位置罢了，没什么好评价的。

现在我们来解答这个问题：在bootmain进入的时间点，栈上存了什么信息。

很简单，存放了当时的esp栈信息，以方便返回。7c4d是最早进入bootmain时存放的esp栈信息。

```
(gdb) x/24x $esp
0x7bdc: 0x00007d87      0x00000000      0x00000000      0x00000000
0x7bec: 0x00000000      0x00000000      0x00000000      0x00000000
0x7bfc: 0x00007c4d      0x8ec031fa      0x8ec08ed8      0xa864e4d0
0x7c0c: 0xb0fa7502      0xe464e6d1      0x7502a864      0xe6dfb0fa
0x7c1c: 0x16010f60      0x200f7c78      0xc88366c0      0xc0220f01
0x7c2c: 0x087c31ea      0x10b86600      0x8ed88e00      0x66d08ec0
```

那么，这个实验到这边是结束了，不过我们最好还是需要跟一遍全部的代码，之后会单开一个章节放在github上，就当做业余爱好了。

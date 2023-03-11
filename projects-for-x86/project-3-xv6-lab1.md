---
description: bootstrap of xv6 lab1.
---

# Project 3: xv6 lab1

**Exercise1: Assembly familiarness.**

The difference of AT\&T and Intel is quite big. 至少有一个东西我经常搞混球:

```
AT&T:  movl %eax, %ebx ;; load ebx with the value of eax. ->
Intel: mov ebx, eax ;; <-
```

此外，还有一个比较值得学习的东西。即Inline Assembly。这玩意我是真的不怎么会，用到了再查询吧。

简单看一下资料**GCC-Inline-Assembly-HOWTO**

```
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

```
// 下面这个例子在HOW-TO中有很不错的解读，我们这边就用注释稍微说说吧。
 int a=10, b;
 asm ("movl %1, %%eax; // 1表示第二个出现的东东，即存放a值的寄存器
       movl %%eax, %0;" // 0表示第一个出现的东东，即存放b值的寄存器
      :"=r"(b)        /* output =r 表示output写值，r表示可以使用任意寄存器*/
      :"r"(a)         /* input */
      :"%eax"         /* clobbered register 通知GCC eax寄存器被修改过了，别塞其他的东西给他*/
      );    
```


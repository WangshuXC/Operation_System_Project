<h1><center>lab0.5实验报告</center></h1>

一、实验过程：

通过终端进入`riscv64-ucore-labcodes/lab0`中，进而进行后续操作，在`make debug`情况下进行`make gdb`操作。

**首先**，输入指令`x/10i $pc`，查看即将执行的10条汇编指令代码。

```HTML
(gdb) x/10i $pc
=&gt; 0x1000:	auipc	t0,0x0
    0x1004:	addi	a1,t0,32
    0x1008:	csrr	a0,mhartid
    0x100c:	ld	    t0,24(t0)
    0x1010:	jr	    t0
    0x1014:	unimp
    0x1016:	unimp
    0x1018:	unimp
    0x101a:	0x8000
    0x101c:	unimp
```

借此可看出，`t0=pc+0<<12=0x1000` ;`a1=t0+32=0x1020`;`a0=mhartid=0`;

`t0= [t0+24] =0x8000000`最后该程序跳转到`t0`地址`0x80000000`

**之后**，输入`si`单步执行且使用`info r t0`的指令查看相关寄存器结果。

```HTML
(gdb) si
0x0000000000001004 in ?? ()
(gdb) info r t0
t0             0x0000000000001000	4096
(gdb) si
0x0000000000001008 in ?? ()
(gdb) info r t0
t0             0x0000000000001000	4096
(gdb) si
0x000000000000100c in ?? ()
(gdb) info r t0
t0             0x0000000000001000	4096
(gdb) si
0x0000000000001010 in ?? ()
(gdb) info r t0
t0             0x0000000080000000	2147483648
(gdb) si
0x0000000080000000 in ?? ()
```

由于`t0`的值发生了变化，故`t0`寄存器的值也存在变化，且最后跳转地址为`0x80000000`，可通过该地址继续判断操作系统运作情况。

```HTML
(gdb) x/10i 0x80000000
=&gt;0x80000000:	csrr	a6,mhartid
   0x80000004:	bgtz	a6,0x80000108
   0x80000008:	auipc	t0,0x0
   0x8000000c:	addi	t0,t0,1032
   0x80000010:	auipc	t1,0x0
   0x80000014:	addi	t1,t1,-16
   0x80000018:	sd	    t1,0(t0)
   0x8000001c:	auipc	t0,0x0
   0x80000020:	addi	t0,t0,1020
   0x80000024:	ld	    t0,0(t0)
```

通过分析汇编指令，可得出`a6=mhartid`;若`a6`>0,则跳转到`0x80000108`;

`t0=pc+(0x0<<12)=0x80000008`;`t0 = t0 + 1032 = 0x80000408`;

`t1 = pc + (0x0 << 12) = 0x80000010`;`t1 = t1 - 16 = 0x80000000`

`t1（0x80000000）存储在地址0x80000408处`;

`t0 = pc + (0x0 << 12) = 0x8000001c`;`t0 = t0 + 1020 = 0x80000400`;

`t0 = [t0 + 0] = [0x80000400] (从地址0x80000400加载一个双字到t0)`

由此可得出，该地址处加载的是作为bootloader的`OpenSBI.bin`，该处的作用为加载操作系统内核并启动操作系统的执行。

**接着**，设置断点讨论情况，输入指令`break kern_entry`，在目标函数kern_entry的第一条指令处设置断点。

```HTML
(gdb) break kern_entry
Breakpoint 1 at 0x80200000: file kern/init/entry.S, line 7.
```

地址`0x80200000`由`kernel.ld`中定义的`BASE_ADDRESS`（加载地址）所决定，标签`kern_entry`是在`kernel.ld`中定义的`ENTRY`（入口点）

`kernel_entry`标志的汇编代码及解释如下：

+  `la sp, bootstacktop`：将`bootstacktop`的地址赋给`sp`，作为栈
+ `tail kern_init`：尾调用，调用函数`kern_init`

再输入指令`x/5i 0x80200000`，查看相关汇编代码，判断函数调用情况。

```HTML
(gdb) x/5i 0x80200000
   0x80200000 &lt;kern_entry&gt;:	auipc	sp,0x3
   0x80200004 &lt;kern_entry+4&gt;:	mv	sp,sp
   0x80200008 &lt;kern_entry+8&gt;:	j	0x8020000c &lt;kern_init&gt;
   0x8020000c &lt;kern_init&gt;:	auipc	a0,0x3
   0x80200010 &lt;kern_init+4&gt;:	addi	a0,a0,-4
```

可以发现在`kern_entry`之后，紧接着就是`kern_init`，再输入`continue`执行直到断点，查看debug输出情况。

```HTML
<p>OpenSBI v0.4 (Jul  2 2019 11:53:53)</p>
<hr />
<p>  / <em>_ \                  / <strong></strong>|  _ _   _|
 | |  | |</em> <em>_   <strong>_ _ </strong> | (___ | |</em>) || |
 | |  | | &#39;_ \ / _ \ &#39;_ \ _<em>_ |  _ &lt; | |
 | |__| | |</em>) |  <strong>/ | | |_</strong><em>) | |</em>) || |_
  _<strong>_/| .</strong>/ _<strong>|<em>| |</em>|_</strong><strong>/|_</strong><em>/<strong>_</strong>|
        | |
        |</em>|</p>
<p>Platform Name          : QEMU Virt Machine
Platform HART Features : RV64ACDFIMSU
Platform Max HARTs     : 8
Current Hart           : 0
Firmware Base          : 0x80000000
Firmware Size          : 112 KB
Runtime SBI Version    : 0.1</p>
<p>PMP0: 0x0000000080000000-0x000000008001ffff (A)
PMP1: 0x0000000000000000-0xffffffffffffffff (A,R,W,X)</p>

```

此时OpenSBI已经启动成功，之后再对kern_init进行断点判断情况。

```HTML
(gdb) break kern_init
Breakpoint 2 at 0x8020000c: file kern/init/init.c, line 8.
```

这里很清晰地指向了之前显示为`<kern_init>`的地址`0x8020000c`。

输入`continue`运行系统后，再输入`disassemble kern_init`查看相关反汇编代码。

```HTML
(gdb) disassemble kern_init
Dump of assembler code for function kern_init:
=&gt; 0x000000008020000c &lt;+0&gt;:	auipc	a0,0x3
   0x0000000080200010 &lt;+4&gt;:	addi	a0,a0,-4 # 0x80203008
   0x0000000080200014 &lt;+8&gt;:	auipc	a2,0x3
   0x0000000080200018 &lt;+12&gt;:	addi	a2,a2,-12 # 0x80203008
   0x000000008020001c &lt;+16&gt;:	addi	sp,sp,-16
   0x000000008020001e &lt;+18&gt;:	li	a1,0
   0x0000000080200020 &lt;+20&gt;:	sub	a2,a2,a0
   0x0000000080200022 &lt;+22&gt;:	sd	ra,8(sp)
   0x0000000080200024 &lt;+24&gt;:	jal	ra,0x802004ce <memset>
   0x0000000080200028 &lt;+28&gt;:	auipc	a1,0x0
   0x000000008020002c &lt;+32&gt;:	addi	a1,a1,1208 # 0x802004e0
   0x0000000080200030 &lt;+36&gt;:	auipc	a0,0x0
   0x0000000080200034 &lt;+40&gt;:	addi	a0,a0,1232 # 0x80200500
   0x0000000080200038 &lt;+44&gt;:	jal	ra,0x80200058 <cprintf>
   0x000000008020003c &lt;+48&gt;:	j	0x8020003c &lt;kern_init+48&gt;
End of assembler dump.
```

可以发现这个函数最后一个指令是`j 0x8020003c <kern_init+48>`，也就是跳转到自己，所以代码将一直循环下去。

**最后**，输入`continue`，debug窗口将出现

```HTML
(THU.CST) os is loading ...
```

二.问题回答

1. 通过前几步汇编语句的反馈，可得出RISCV加电后的指令主要在地址`0x1000`到地址`0x1010`。
2. 该指令完成的功能如下：
   + `auipc t0,0x0`：用于加载一个20bit的立即数，`t0`中保存的数据是`(pc)+(0<<12)`。用于PC相对寻址。

   + `addi a1,t0,32`：将`t0`加上`32`，赋值给`a1`。

   + `csrr a0,mhartid`：读取状态寄存器`mhartid`，将信息存入`a0`中。`mhartid`为正在运行代码的硬件线程的整数ID。

   + `ld t0,24(t0)`：双字，加载从`t0+24`地址处读取8个字节，存入`t0`。

   + `jr t0`：寄存器跳转，跳转到寄存器指向的地址处（此处为`0x80000000`）。


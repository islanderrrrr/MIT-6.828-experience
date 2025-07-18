# 前言
这是一个操作系统创立最初始的地方，我将从这个MIT-6.828的实验中，逐步了解一个操作系统创立的始末，其用意在于深刻理解，并且手操性的学习操作系统  

# 简介
第一个练习的目的是向你介绍 x86 汇编语言和 PC 引导过程，并让你开始用 QEMU 和 QEMU/GDB 进行调试。你不需要为实验室的这一部分编写任何代码，但你应该在自己的理解范围内仔细阅读它，并准备好回答下面提出的问题。

**以下均是本人拙见，以及部分原文摘抄**

---

## 从 x86 汇编开始

我们都知道，一个操作系统，是有amd64以及i386两种版本的，例如：i386是由32位架构需求编写而成的，而amd64则是64位架构需求  

其中x86包含的也就是64位以及32位

两者在部分方面有所不同
- 寄存器
- 内存

抛开那些，x86的汇编要如何熟悉呢？

对于汇编语言，建议可以从一些基础的汇编语言入手，[汇编基础语言学习](https://pwn.college/computing-101/)   
你也可以从一些视频中进行学习   

## 模拟x86  
我们不是在真实的、实际的个人计算机(PC)上开发操作系统，而是使用一个程序来忠实地模拟一个完整的 PC：你为模拟器编写的代码也将在真实的 PC 上启动。使用模拟器简化了调试；例如，你可以在模拟的 x86 中设置断点，这在 x86 的硅版中很难做到。

在 6.828 中，我们将使用QEMU 模拟器，这是一个现代且相对快速的模拟器。虽然 QEMU 的内置监视器只提供有限的调试支持，但 QEMU 可以充当GNU 调试器(GDB)的远程调试目标，我们将在本实验中使用它来逐步完成早期引导过程。

首先将lab1的文件解压到一个自己定的一个地方，进入后输入  
```
cd lab

make
```

此刻应该出现如下回应  

```
+ as kern/entry.S
+ cc kern/entrypgdir.c
+ cc kern/init.c
+ cc kern/console.c
+ cc kern/monitor.c
+ cc kern/printf.c
+ cc kern/kdebug.c
+ cc lib/printfmt.c
+ cc lib/readline.c
+ cc lib/string.c
+ ld obj/kern/kernel
+ as boot/boot.S
+ cc -Os boot/main.c
+ ld boot/boot
boot block is 380 bytes (max 510)
+ mk obj/kern/kernel.img
```

(如果你得到像“undefined reference to '__udivdi3'”这样的错误，你可能没有 32 位的 gcc multilib。如果你正在 Debian 或 Ubuntu 上运行，尝试安装 gcc-multilib 包。)

现在你已经准备好运行 QEMU，并提供文件 obj/kern/kernel.img，创建在上面，作为模拟 PC 的“虚拟硬盘”的内容。这个硬盘映像包含我们的引导加载程序(obj/boot/boot)和我们的内核(obj/kernel)。  

**接下来创建虚拟窗口**  
```
$ make qemu
```
或  
```
$ make qemu-nox  
```
将执行 QEMU，其中包含设置硬盘和直接串口输出到终端所需的选项。执行后，这些文本应该出现在 QEMU 窗口里：  
```
Booting from Hard Disk...
6828 decimal is XXX octal!
entering test_backtrace 5
entering test_backtrace 4
entering test_backtrace 3
entering test_backtrace 2
entering test_backtrace 1
entering test_backtrace 0
leaving test_backtrace 0
leaving test_backtrace 1
leaving test_backtrace 2
leaving test_backtrace 3
leaving test_backtrace 4
leaving test_backtrace 5
Welcome to the JOS kernel monitor!
Type 'help' for a list of commands.
K>
```
‘Booting from Hard Disk...’之后的一切都是我们的 JOS 内核打印的；K> 是由我们在内核中包含的小监视器或交互式控制程序打印的提示符。如果执行 make qemu，内核打印的这些行将出现在运行 qemu 的常规 shell 窗口和 qemu 显示窗口中。这是因为测试和打分的目的，我们设立了 JOS 内核写它的控制台输出不仅在虚拟 VGA 显示(见 QEMU 窗口)，而且模拟电脑的虚拟串口，QEMU 反过来输出自己的标准输出。类似地，JOS 内核将从键盘和串口接受输入，因此你可以在 VGA 显示窗口或运行 QEMU 的终端中向它提供命令。或者，你可以通过运行 make qemu-nox 在没有虚拟 VGA 的情况下使用串行控制台。这可能是方便的，如果你是 SSH 进入一个 Athena 拨号。要退出 qemu，输入 Ctrl+a x。  

内核监视器只有两个命令可以用，help 和 kerninfo。  
```
Welcome to the JOS kernel monitor!
Type 'help' for a list of commands.
K> help
help - Display this list of commands
kerninfo - Display information about the kernel
K> kerninfo
Special kernel symbols:
  _start                  0010000c (phys)
  entry  f010000c (virt)  0010000c (phys)
  etext  f01019e1 (virt)  001019e1 (phys)
  edata  f0112060 (virt)  00112060 (phys)
  end    f01126c0 (virt)  001126c0 (phys)
Kernel executable memory footprint: 74KB
K> 
```

help 命令很明显，我们将很快讨论 kerninfo 命令输出的含义。虽然很简单，但需要注意的是，这个内核监控器“直接”运行在模拟 PC 的“原始(虚拟)硬件”上。这意味着你应该能够复制 obj/kern/kernel 的内容。将硬盘插入真正的 PC，打开它，并在 PC 的真实屏幕上看到与你在上面的 QEMU 窗口中所做的完全相同的事情。(但是，我们不建议你在硬盘上有有用信息的真正机器上这样做，因为复制 kernel.img 添加到其硬盘的开始部分将回收主引导记录和第一个分区的开始部分，从而有效地导致硬盘上之前的所有内容丢失！)  

## The ROM BIOS
在本部分中，你将使用 QEMU 的调试工具来研究 IA-32 兼容的计算机是如何引导的。

打开两个终端窗口并将**两个 shell cd 到你的实验室目录中**。**其中，输入 make qemu-gdb(或 make qemu-nox-gdb)**。这将启动 QEMU，但是 QEMU 在处理器执行第一个指令之前停止，并等待来自 GDB 的调试连接。**在第二个终端中，在运行 make 的同一个目录运行 make gdb**。你应该看到这样的东西，  

可以看到如下(gdb版本不同可能有些内容不同，但是主要内容不会变)
```
$ make gdb
gdb -n -x .gdbinit
GNU gdb (Ubuntu 12.1-0ubuntu1~22.04.2) 12.1
Copyright (C) 2022 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
+ target remote localhost:25000
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
The target architecture is set to "i8086".
[f000:fff0]    0xffff0: ljmp   $0x3630,$0xf000e05b
0x0000fff0 in ?? ()
+ symbol-file obj/kern/kernel
warning: A handler for the OS ABI "GNU/Linux" is not built into this configuration
of GDB.  Attempting to continue with the default i8086 settings.

(gdb) 

```

注意以下行  
```
[f000:fff0]    0xffff0: ljmp   $0x3630,$0xf000e05b
```
这是 GDB 反汇编第一行执行的指令。从输出端你可以得到一些结论：(与官方所言略有出入)

IBM PC 在物理地址 0x000ffff0 处开始执行，也就是位 BIOS ROM 预留的 64KB 的最顶端。
PC 开始执行时 CS = 0xf000(应该是，但是我这里为0x3630)，IP = 0xfff0。
第一个执行的指令是 jmp 指令，它会跳转到 CS = 0xf000 和 IP = 0xe05b 处。

为什么和官方有所差异呢，让我们进一步实验  
```
(gdb) x/5i 0xffff0
   0xffff0:     ljmp   $0x3630,$0xf000e05b
   0xffff7:     das    
   0xffff8:     xor    (%ebx),%dh
   0xffffa:     das    
   0xffffb:     cmp    %edi,(%ecx)
(gdb) x/10bx 0xffff0
0xffff0:        0xea    0x5b    0xe0    0x00    0xf0    0x30    0x36    0x2f
0xffff8:        0x32    0x33
```
似乎只有跳转地址有所不同，不会有所影响，Ai解释如下  
- PC 仍然从 0xffff0 开始执行
- 这仍然是 BIOS ROM 区域的最后16字节
- 第一条指令仍然是跳转指令
- 跳转的目的是去到 BIOS 中更早的位置执行初始化代码

因此无需在意  

---

**对于练习二：**

追踪得到以下内容  
```
[f000:fff0]    0xffff0: ljmp   $0x3630,$0xf000e05b
0x0000fff0 in ?? ()
+ symbol-file obj/kern/kernel
warning: A handler for the OS ABI "GNU/Linux" is not built into this configuration
of GDB.  Attempting to continue with the default i8086 settings.

(gdb) x/5i 0xffff0
   0xffff0:     ljmp   $0x3630,$0xf000e05b
   0xffff7:     das    
   0xffff8:     xor    (%ebx),%dh
   0xffffa:     das    
   0xffffb:     cmp    %edi,(%ecx)
(gdb) x/10bx 0xffff0
0xffff0:        0xea    0x5b    0xe0    0x00    0xf0    0x30    0x36    0x2f
0xffff8:        0x32    0x33
(gdb) si
[f000:e05b]    0xfe05b: cmpw   $0xffc8,%cs:(%esi)
0x0000e05b in ?? ()
(gdb) si
[f000:e062]    0xfe062: jne    0xd241d0b0
0x0000e062 in ?? ()
(gdb) si
[f000:e066]    0xfe066: xor    %edx,%edx
0x0000e066 in ?? ()
(gdb) si
[f000:e068]    0xfe068: mov    %edx,%ss
0x0000e068 in ?? ()
(gdb) si
[f000:e06a]    0xfe06a: mov    $0x7000,%sp
0x0000e06a in ?? ()
(gdb) si
[f000:e070]    0xfe070: mov    $0xfc1c,%dx
0x0000e070 in ?? ()
(gdb) si
[f000:e076]    0xfe076: jmp    0x5576cf2d
0x0000e076 in ?? ()
(gdb) si
[f000:cf2b]    0xfcf2b: cli    
0x0000cf2b in ?? ()
(gdb) si
[f000:cf2c]    0xfcf2c: cld    
0x0000cf2c in ?? ()
(gdb) si
[f000:cf2d]    0xfcf2d: mov    %ax,%cx
0x0000cf2d in ?? ()
(gdb) si
[f000:cf30]    0xfcf30: mov    $0x8f,%ax
0x0000cf30 in ?? ()
(gdb) si
[f000:cf36]    0xfcf36: out    %al,$0x70
^[[A0x0000cf36 in ?? ()
(gdb) si
[f000:cf38]    0xfcf38: in     $0x71,%al
0x0000cf38 in ?? ()
(gdb) si
[f000:cf3a]    0xfcf3a: in     $0x92,%al
0x0000cf3a in ?? ()
(gdb) si
[f000:cf3c]    0xfcf3c: or     $0x2,%al
0x0000cf3c in ?? ()
(gdb) si
[f000:cf3e]    0xfcf3e: out    %al,$0x92
0x0000cf3e in ?? ()
(gdb) si
[f000:cf40]    0xfcf40: mov    %cx,%ax
0x0000cf40 in ?? ()
(gdb) si
[f000:cf43]    0xfcf43: lidtl  %cs:(%esi)
0x0000cf43 in ?? ()
(gdb) si
[f000:cf49]    0xfcf49: lgdtl  %cs:(%esi)
0x0000cf49 in ?? ()
(gdb) si
[f000:cf4f]    0xfcf4f: mov    %cr0,%ecx
0x0000cf4f in ?? ()
(gdb) si
[f000:cf52]    0xfcf52: and    $0xffff,%cx
0x0000cf52 in ?? ()
(gdb) si
[f000:cf59]    0xfcf59: or     $0x1,%cx
0x0000cf59 in ?? ()
(gdb) si
[f000:cf5d]    0xfcf5d: mov    %ecx,%cr0
0x0000cf5d in ?? ()
(gdb) si
[f000:cf60]    0xfcf60: ljmpw  $0xf,$0xcf68
0x0000cf60 in ?? ()
(gdb) si
The target architecture is set to "i386".
=> 0xfcf68:     mov    $0x10,%ecx
0x000fcf68 in ?? ()
(gdb) si
=> 0xfcf6d:     mov    %ecx,%ds
0x000fcf6d in ?? ()
(gdb) 

```

让我们逐个分析  

1. 初始跳转和内存检查 (0xffff0 - 0xfe066)
```
0xffff0: ljmp   $0x3630,$0xf000e05b    # 跳转到BIOS主程序
0xfe05b: cmpw   $0xffc8,%cs:(%esi)     # 内存/配置检查
0xfe062: jne    0xd241d0b0              # 条件跳转
0xfe066: xor    %edx,%edx               # 清零EDX寄存器
```

2. 栈初始化 (0xfe068 - 0xfe06a)
```
0xfe068: mov    %edx,%ss               # 设置栈段寄存器SS = 0
0xfe06a: mov    $0x7000,%sp            # 设置栈指针SP = 0x7000
```
**作用：** 建立栈空间，栈位于物理地址 0x7000  

3.  硬件初始化准备 (0xfe070 - 0xfcf2c)
```
0xfe070: mov    $0xfc1c,%dx            # 设置DX寄存器
0xfe076: jmp    0x5576cf2d             # 跳转到硬件初始化代码
0xfcf2b: cli                           # 关闭中断
0xfcf2c: cld                           # 清除方向标志
```

4. 硬件端口操作 (0xfcf36 - 0xfcf3e)
```
0xfcf36: out    %al,$0x70              # 写CMOS/RTC端口
0xfcf38: in     $0x71,%al              # 读CMOS数据
0xfcf3a: in     $0x92,%al              # 读系统控制端口A
0xfcf3c: or     $0x2,%al               # 设置A20门使能位
0xfcf3e: out    %al,$0x92              # 写回端口92，启用A20线
```

5. 保护模式准备 (0xfcf43 - 0xfcf5d)
```
0xfcf43: lidtl  %cs:(%esi)             # 加载中断描述符表
0xfcf49: lgdtl  %cs:(%esi)             # 加载全局描述符表
0xfcf4f: mov    %cr0,%ecx              # 读取CR0控制寄存器
0xfcf52: and    $0xffff,%cx            # 清除高16位
0xfcf59: or     $0x1,%cx               # 设置PE位(保护模式使能)
0xfcf5d: mov    %ecx,%cr0              # 写回CR0，切换到保护模式
```

6. 进入保护模式 (0xfcf60 - 0xfcf6d)
```
0xfcf60: ljmpw  $0xf,$0xcf68           # 长跳转，加载新的代码段
=> 0xfcf68: mov    $0x10,%ecx          # 设置数据段选择子
=> 0xfcf6d: mov    %ecx,%ds            # 加载数据段寄存器
```
暂时只分析了这些  

**BIOS 主要功能总结**
BIOS 在做什么：
  1. 硬件初始化
     - 设置栈空间
     - 初始化CPU状态
     - 配置系统硬件
     
  2. 内存管理准备
     - 启用A20地址线（允许访问1MB以上内存）
     - 设置内存映射
    
  3. 保护模式转换
     - 设置GDT（全局描述符表）
     - 设置IDT（中断描述符表）
     - 从16位实模式切换到32位保护模式
    
  4. 系统环境建立
     - 关闭中断
     - 设置段寄存器
     - 为后续操作系统加载做准备

这就是典型的PC BIOS启动序列：硬件检测 → 内存初始化 → 保护模式切换 → 准备加载引导程序。  

当 BIOS 运行时，它设置一个中断描述符表并初始化各种设备，例如 VGA 显示器。这就是你在 QEMU 窗口看到的 "Starting SeaBIOS" 消息来自的地方。

在 BIOS 初始化它所知道的 PCI bus 和所有重要的设备之后，它搜索一个可启动设备，比如软盘、硬盘或者 CD-ROM。最终，当它找到一个可启动的磁盘时，BIOS 读取磁盘里的 boot loader，然后把控制权转交给它。

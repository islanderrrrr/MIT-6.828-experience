# 简介
此篇主要讲解**引导加载程序**  

Boot Loader（引导加载程序）是计算机启动过程中的关键组件，位于 BIOS 和操作系统内核之间。它是存储在磁盘第一个扇区（512字节）中的小程序。

Boot Loader 的主要任务
1. 模式切换
  - 从16位实模式切换到32位保护模式
  - 实模式只能访问1MB内存，保护模式可以访问4GB内存
  - 这是访问现代计算机全部内存的必要步骤
2. 加载内核
  - 从磁盘读取操作系统内核到内存
  - 使用x86的I/O指令直接操作IDE磁盘控制器
  - 解析ELF格式的内核文件
将内核各个段加载到正确的内存位置

# 正文

软盘和硬盘被分成很多 512 比特的区域，这些区域被叫做扇区。一个扇区是磁盘最小的传输粒度：每次读写操作必须是一个或多个扇区的大小，并在扇区边界对齐。

如果磁盘是可启动的，第一个扇区被叫做启动扇区，因为这是 boot loader 代码所处的地方。

当 BIOS 找到一个可启动的软盘或硬盘时，它加载 512 比特的 boot 扇区到物理地址 0x7c00-0x7dff 处，然后使用一个 jmp 指令设置 CS:IP 为 0000:7c00，传递控制权给 boot loader。就像 BIOS 加载地址一样，这些地址是相当随意的，但对于 PC 来说是固定和标准化的。  
(译者注：因为历史原因，BIOS 的加载地址被约定为 0x7c00，不用想为什么是这个地址，没有什么直接的原因。)  

从 CD-ROM 启动的能力是在 PC 进化过程中很久以后才出现的，因此，PC 架构师利用这个机会稍微重新思考了一下引导过程。因此，现代 BIOS 从 CD-ROM 启动的方式有点复杂(也更强大)。CD-ROM 使用一个 2048 比特的扇区大小而不是 512 比特，BIOS 可以加载一个更大的来自磁盘的 boot 镜像进入内存（不仅只是一个扇区），然后在转交控制权。

然而，对于 6.828 来说，我们会使用传统的硬盘 boot 机制，也就意味着我们的 boot loader 必须填入一个 512 比特的磁盘区域。这个 boot loader 包含了一个汇编语言的源文件，boot/boot.S 和一个 C 源文件，boot/main.c，细心地把这些源文件通读一遍，确保你理解了运作原理。boot loader 必须执行两个主要的功能：
  1. 首先，这个 boot loader 把处理器从实模式切换到 32 位保护模式，因为在保护模式下，软件才可以访问处理器物理地址空间的所有 1MB 以上的内存。PC Assembly Language的 1.2.7 和 1.2.8 小结对保护模式作了简单的描述，详细的细节可以参考 Intel 架构手册。此刻你只需要理解分段地址在保护模式中转化为物理地址的方式不一样，转换的 offset 是 32 位而不是 16 位的。
  2. 第二，这个 boot loader 从磁盘上读取内核，通过使用 x86 特殊的 I/O 指令来直接访问 IDE 磁盘寄存器来做到这一点。如果你想要更好地理解特别的 I/O 指令的含义，查看the 6.828 reference page的"IDE hard drive controller"小结。你在这门课程里不必学习太多特殊设备的编程：写设备驱动程序在操作系统开发里是非常重要的部分，但是从概念和架构的角度来看，这也是最无聊的部分。

## boot.s

**文件头部定义**
```
.set PROT_MODE_CSEG, 0x8         # 内核代码段选择子
.set PROT_MODE_DSEG, 0x10        # 内核数据段选择子  
.set CR0_PE_ON,      0x1         # 保护模式使能标志
```
这些常量定义了保护模式下的段选择子和控制寄存器标志。

**程序入口点（16位实模式）**
```
.globl start
start:
  .code16                     # 汇编为16位模式代码
  cli                         # 关闭中断
  cld                         # 清除方向标志（字符串操作递增）
```

**初始化段寄存器**
```
  xorw    %ax,%ax             # AX = 0
  movw    %ax,%ds             # 数据段 = 0
  movw    %ax,%es             # 扩展段 = 0  
  movw    %ax,%ss             # 栈段 = 0
```
将所有段寄存器设置为0，建立平坦的内存模型。

**A20 地址线使能**
```
seta20.1:
  inb     $0x64,%al               # 等待键盘控制器不忙
  testb   $0x2,%al                # 测试状态位
  jnz     seta20.1                # 如果忙则继续等待

  movb    $0xd1,%al               # 发送命令：写输出端口
  outb    %al,$0x64

seta20.2:
  inb     $0x64,%al               # 再次等待不忙
  testb   $0x2,%al
  jnz     seta20.2

  movb    $0xdf,%al               # 0xdf = 11011111b (A20使能)
  outb    %al,$0x60               # 写入键盘数据端口
```
目的：启用 A20 地址线，允许访问超过 1MB 的内存。在早期 PC 中，为了向后兼容，A20 线被禁用。

**切换到保护模式**
```
  lgdt    gdtdesc                 # 加载全局描述符表
  movl    %cr0, %eax              # 读取CR0控制寄存器
  orl     $CR0_PE_ON, %eax        # 设置PE位（保护模式使能）
  movl    %eax, %cr0              # 写回CR0
```

**关键步骤：**
  1. 加载 GDT：lgdt gdtdesc 告诉 CPU 全局描述符表的位置
  2. 设置 PE 位：CR0 寄存器的第 0 位控制是否启用保护模式

**进入 32 位模式**
```
  ljmp    $PROT_MODE_CSEG, $protcseg
```
长跳转：这条指令同时完成两个任务：
  - 跳转到 protcseg 标签
  - 加载新的代码段选择子 0x8，正式进入保护模式

**32 位保护模式初始化**
```
  .code32                     # 从这里开始汇编32位代码
protcseg:
  # 设置保护模式数据段寄存器
  movw    $PROT_MODE_DSEG, %ax    # 数据段选择子 = 0x10
  movw    %ax, %ds                # 设置所有段寄存器
  movw    %ax, %es
  movw    %ax, %fs  
  movw    %ax, %gs
  movw    %ax, %ss
```

将所有段寄存器指向数据段描述符。

**设置栈并调用 C 代码**
```
  movl    $start, %esp            # 设置栈指针
  call bootmain                   # 调用 C 函数 bootmain
```

注意：栈指针指向 start（0x7c00），栈向下增长。

**全局描述符表（GDT）**
```
gdt:
  SEG_NULL                        # 空描述符（必需）
  SEG(STA_X|STA_R, 0x0, 0xffffffff)  # 代码段：可执行+可读，0-4GB
  SEG(STA_W, 0x0, 0xffffffff)         # 数据段：可写，0-4GB

gdtdesc:
  .word   0x17                    # GDT大小-1 (3个描述符×8字节-1=23=0x17)
  .long   gdt                     # GDT地址
```

**GDT 结构：**
  - 索引 0：空描述符（CPU 要求）
  - 索引 1：代码段（选择子 0x8）
  - 索引 2：数据段（选择子 0x10）

## 总结

这个引导程序完成了从 BIOS 到内核的关键过渡：

  1. 实模式初始化：设置段寄存器，关闭中断
  2. 硬件准备：启用 A20 线
  3. 保护模式切换：加载 GDT，设置 CR0
  4. 32位环境建立：设置段寄存器，建立栈
  5. 调用 C 代码：跳转到 bootmain 函数

执行完这些步骤后，CPU 就运行在 32 位保护模式下，可以访问全部 4GB 地址空间，为加载操作系统内核做好了准备。

## main.c

### 文件概述

**磁盘布局**
```
+----------------+
| Boot Loader    |  <- 第1个扇区 (512字节)
| (boot.S+main.c)|
+----------------+
| Kernel Image   |  <- 第2个扇区开始
| (ELF format)   |
+----------------+
```

**启动流程**
  1. BIOS 加载第一个扇区到 0x7c00 并执行
  2. boot.S 设置保护模式，建立栈，调用 bootmain()
  3. bootmain() 读取内核，解析 ELF，跳转到内核入口

### 常量定义

```
#define SECTSIZE	512
#define ELFHDR		((struct Elf *) 0x10000) // scratch space
```
  - ``SECTSIZE``: 磁盘扇区大小 512 字节
  - ``ELFHDR``: ELF 头的加载地址 0x10000（64KB），作为临时空间

### bootmain() 函数详解

**1. 读取 ELF 头**
```
// read 1st page off disk
readseg((uint32_t) ELFHDR, SECTSIZE*8, 0);
```
  - 从磁盘偏移 0 开始读取 8 个扇区（4KB）
  - 加载到物理地址 0x10000
  - 这足够包含完整的 ELF 头和程序头表

**2. 验证 ELF 魔数**
```
// is this a valid ELF?
if (ELFHDR->e_magic != ELF_MAGIC)
    goto bad;
```
检查文件是否为有效的 ELF 格式（魔数：0x7F454C46 = "\x7FELF"）  

**3. 解析并加载程序段**
```
// load each program segment (ignores ph flags)
ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
eph = ph + ELFHDR->e_phnum;
for (; ph < eph; ph++)
    readseg(ph->p_pa, ph->p_memsz, ph->p_offset);
```
  - ph: 指向程序头表的开始
  - eph: 程序头表的结束
  - ELFHDR->e_phoff: 程序头表在文件中的偏移
  - ELFHDR->e_phnum: 程序头的数量
  - ph->p_pa: 段的物理加载地址
  - ph->p_memsz: 段在内存中的大小
  - ph->p_offset: 段在文件中的偏移

**4. 跳转到内核**
```
// call the entry point from the ELF header
// note: does not return!
((void (*)(void)) (ELFHDR->e_entry))();
```
  - ELFHDR->e_entry: 内核的入口地址
  - 强制类型转换为函数指针并调用
  - 注意：这个调用不会返回！

### readseg() 函数详解
```
void readseg(uint32_t pa, uint32_t count, uint32_t offset)
```
参数：

  - pa: 目标物理地址
  - count: 要读取的字节数
  - offset: 文件中的字节偏移

**实现细节**
```
end_pa = pa + count;

// round down to sector boundary
pa &= ~(SECTSIZE - 1);
```
将地址向下对齐到扇区边界（512字节对齐）

```
// translate from bytes to sectors, and kernel starts at sector 1
offset = (offset / SECTSIZE) + 1;
```
  - 将字节偏移转换为扇区号
  - ``+1`` 因为内核从第2个扇区开始（第1个扇区是boot loader）

```
while (pa < end_pa) {
    readsect((uint8_t*) pa, offset);
    pa += SECTSIZE;
    offset++;
}
```
循环读取扇区直到完成所需的数据传输

### 磁盘 I/O 函数

**waitdisk() - 等待磁盘就绪**
```
void waitdisk(void)
{
    // wait for disk ready
    while ((inb(0x1F7) & 0xC0) != 0x40)
        /* do nothing */;
}
```
  - 端口 0x1F7: IDE 主控制器状态寄存器
  - 等待状态为 0x40 (READY, 不BUSY)

**readsect() - 读取单个扇区**
```
void readsect(void *dst, uint32_t offset)
{
    waitdisk();
    
    outb(0x1F2, 1);		            // count = 1
    outb(0x1F3, offset);            // LBA[7:0]
    outb(0x1F4, offset >> 8);       // LBA[15:8]
    outb(0x1F5, offset >> 16);      // LBA[23:16]
    outb(0x1F6, (offset >> 24) | 0xE0); // LBA[27:24] + LBA模式
    outb(0x1F7, 0x20);	            // cmd 0x20 - read sectors
    
    waitdisk();
    
    // read a sector
    insl(0x1F0, dst, SECTSIZE/4);
}
```
IDE 端口映射：

  - 0x1F2: 扇区计数
  - 0x1F3-0x1F6: LBA 地址（28位）
  - 0x1F7: 命令寄存器
  - 0x1F0: 数据端口

LBA 模式： 0xE0 = 11100000b

  - 位7: 固定为1
  - 位6: LBA模式使能
  - 位5: 固定为1
  - 位4: 驱动器选择（0=主驱动器）

### 错误处理
```
bad:
    outw(0x8A00, 0x8A00);
    outw(0x8A00, 0x8E00);
    while (1)
        /* do nothing */;
```
如果 ELF 头无效，向特定端口输出错误代码并进入无限循环。

## 总结

这个 boot loader 实现了一个简单但完整的 ELF 加载器：

  - **读取 ELF 头**：验证文件格式
  - **解析程序头**：找到所有需要加载的段
  - **加载段到内存**：按照 ELF 指定的地址加载
  - **跳转执行**：将控制权交给内核


## 练习3

在你理解了 boot loader 的源代码之后，查看 obj/boot/boot.asm 文件。这个文件是我们的 GUNMakefile 创建 boot loader 之后对其进行反汇编。这个反汇编文件可以很容易地看到这个 boot loader 的代码会驻留在物理内存的哪些确切位置，然后就很容易在 GDB 里追踪 boot loader 发生了什么事情。同样的，obj/kern/kernel.asm 包含了一个对 JOS 内核的反汇编，对于 debug 来说很有用。

你可以在 GDB 里用 b 命令来设置地址断点。例如，b *0x7c00 在地址 0x7c00 处设置了一个断点。一旦到某个断点，你可以进用 c 和 si 命令继续执行：c 让 QEMU 继续执行直到下一个断点（或者直到你按下 Ctrl-C），si N 可以一次性步进 N 条指令。

为了查看内存里的指令（除了即将执行的下一个，GDB 会自动打印），你可以使用 x/i 指令。这个命令的语法是 x/Ni ADDR，N 是连续反汇编指令的数目，ADDR 是开始反汇编的内存地址。

- 在什么时候处理器开始执行 32 位代码？是什么导致了 16 位到 32 位到模式的转变？

处理器在执行``.ljmp $PROT_MODE_CSEG, $protcseg`` 指令时开始执行 32 位代码。这条指令会将处理器从 16 位实模式切换到 32 位保护模式。  
具体流程是：首先通过设置 CR0 的 PE 位（``movl %cr0, %eax; orl $CR0_PE_ON, %eax; movl %eax, %cr0``）打开保护模式，然后通过远跳转（``ljmp``）进入 32 位代码段，从而切换到 32 位模式。

- boot loader 最后执行的指令是什么，kernel 刚加载时执行的第一个指令是什么？

boot loader最后执行的指令是``((void (*)(void)) (ELFHDR->e_entry))()``，是对 ELF 格式的内核主入口点进行跳转，也就是kernel内核，kernel刚加载时，第一个指令是``movw	$0x1234,0x472			# warm boot``  

- kernel 的第一个指令在哪儿？

0x0010000c的位置，实验全程结果为 
```
Breakpoint 2, 0x00007d71 in ?? ()
(gdb) x/i
   0x7d77:      mov    $0x8a00,%edx
(gdb) ni
=> 0x10000c:    movw   $0x1234,0x472
```

- boot loader 如何决定要读取多少扇区才能加载整个内核？它是在哪里找到这个信息的？

boot loader 首先读取 ELF 文件头（``readseg((uint32_t)ELFHDR, SECTSIZE*8, 0);``），然后通过解析 ELF 文件头中的 program header（``e_phoff``, ``e_phnum``），获取每个段的物理地址、内存大小和文件偏移（``p_pa``, ``p_memsz``, ``p_offset``），再根据这些信息循环调用 ``readseg`` 读取各个段的数据。每个段需要读取多少字节由 ``p_memsz`` 决定，分多次读取。所有这些信息都来自 ELF 文件头和 program header。

---

# Loading the Kernel（加载内核）

我们现在将进一步详细研究 boot loader 的 C 语言部分，boot/main.c。在开始之前，最好可以先好好回顾一下 C 编程的基础。  

## 练习4

阅读 C 语言里的指针部分。  

- **关于pointers.c的解析**

源码
```
#include <stdio.h>
#include <stdlib.h>

void
f(void)
{
    int a[4];
    int *b = malloc(16);
    int *c;
    int i;

    printf("1: a = %p, b = %p, c = %p\n", a, b, c);

    c = a;
    for (i = 0; i < 4; i++)
	a[i] = 100 + i;
    c[0] = 200;
    printf("2: a[0] = %d, a[1] = %d, a[2] = %d, a[3] = %d\n",
	   a[0], a[1], a[2], a[3]);

    c[1] = 300;
    *(c + 2) = 301;
    3[c] = 302;
    printf("3: a[0] = %d, a[1] = %d, a[2] = %d, a[3] = %d\n",
	   a[0], a[1], a[2], a[3]);

    c = c + 1;
    *c = 400;
    printf("4: a[0] = %d, a[1] = %d, a[2] = %d, a[3] = %d\n",
	   a[0], a[1], a[2], a[3]);

    c = (int *) ((char *) c + 1);
    *c = 500;
    printf("5: a[0] = %d, a[1] = %d, a[2] = %d, a[3] = %d\n",
	   a[0], a[1], a[2], a[3]);

    b = (int *) a + 1;
    c = (int *) ((char *) a + 1);
    printf("6: a = %p, b = %p, c = %p\n", a, b, c);
}

int
main(int ac, char **av)
{
    f();
    return 0;
}
```

执行结果
```
1: a = 000000000061FDC0, b = 00000000007126F0, c = 0000000000000010
2: a[0] = 200, a[1] = 101, a[2] = 102, a[3] = 103
3: a[0] = 200, a[1] = 300, a[2] = 301, a[3] = 302
4: a[0] = 200, a[1] = 400, a[2] = 301, a[3] = 302
5: a[0] = 200, a[1] = 128144, a[2] = 256, a[3] = 302
6: a = 000000000061FDC0, b = 000000000061FDC4, c = 000000000061FDC1
```

这是一个很好的指针学习资料，接下来逐步解析  

**输出1**: ``1: a = 000000000061FDC0, b = 00000000007126F0, c = 0000000000000010``  
- a 是栈上的数组，地址为 0x61FDC0
- b 是通过 malloc(16) 分配的堆内存，地址为 0x7126F0
- c 是未初始化的指针，包含垃圾值 0x10

**输出2**:``2: a[0] = 200, a[1] = 101, a[2] = 102, a[3] = 103``  
```
c = a;                    // c 现在指向数组 a
for (i = 0; i < 4; i++)
    a[i] = 100 + i;       // a[] = {100, 101, 102, 103}
c[0] = 200;               // 通过 c 修改 a[0] = 200
```

**输出3**: ``3: a[0] = 200, a[1] = 300, a[2] = 301, a[3] = 302``  
```
c[1] = 300;       // a[1] = 300
*(c + 2) = 301;   // a[2] = 301 (指针算术)
3[c] = 302;       // a[3] = 302 (数组下标的交换律：3[c] 等同于 c[3])
```

**输出4**: ``4: a[0] = 200, a[1] = 400, a[2] = 301, a[3] = 302``  
```
c = c + 1;        // c 现在指向 a[1]
*c = 400;         // 修改 a[1] = 400
```

**输出5**: ``5: a[0] = 200, a[1] = 128144, a[2] = 256, a[3] = 302``  
```
c = (int *) ((char *) c + 1);
*c = 500;
```

1. c 当前指向 a[1] (地址 0x61FDC4)
2. (char *) c + 1 将指针按字节移动1位，到 0x61FDC5
3. (int *) ((char *) c + 1) 将其转换回 int*
4. *c = 500 在未对齐的地址写入4字节的整数500

由于地址未对齐（不是4的倍数），写入的数据会跨越两个整数边界，导致：
- a[1] 的高3字节被覆盖
- a[2] 的低1字节被覆盖

**输出6**: ``6: a = 000000000061FDC0, b = 000000000061FDC4, c = 000000000061FDC1``  
```
b = (int *) a + 1;              // b 指向 a[1]，地址为 a + 4 字节
c = (int *) ((char *) a + 1);   // c 指向 a 的地址 + 1 字节
```

- a = 0x61FDC0（数组起始地址）
- b = 0x61FDC4（a + 4，指向 a[1]）
- c = 0x61FDC1（a + 1，未对齐的地址）

---

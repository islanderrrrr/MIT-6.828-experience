# 介绍

在本实验中，你将为你的操作系统编写内存管理代码。内存管理有两个组件。

第一个组件是内核的物理内存分配器，这样内核可以分配内存，然后释放它。你的分配器将以 4096 字节(称为页)为单位进行操作。你的任务将是维护数据结构，这些数据结构记录了哪些物理页面是空闲的，哪些被分配了，以及有多少进程共享每个分配的页面。你还将编写用于分配和释放内存页的例程。

内存管理的第二个组件是虚拟内存，它将内核和用户软件使用的虚拟地址映射到物理内存中的地址。当指令使用内存时，x86 硬件的内存管理单元(MMU)会参考一组页表来执行映射。你将根据我们提供的规范修改 JOS 以设置 MMU 的页表。

# 开始
在这个以及未来的实验中，你会逐渐地构建你的内核。我们会给你提供一些额外的源代码。

Lab 2 包括了如下新的源代码文件，你应该浏览一遍：

- inc/memlayout.h
- kern/pmap.c
- kern/pmap.h
- kern/kclock.h
- kern/kclock.c

memlayout.h 描述了虚拟地址空间的布局，你要通过修改 pmap.c 来实现的这个布局。memlayout.h 和 pmap.h 定义了 PageInfo 结构体，你将使用它来跟踪哪些物理内存页面是空闲的。
kclock.c 和 kclock.h 操作 PC 的时钟（里面有电池）和 CMOS RAM 硬件，在这些硬件中，BIOS 记录了 PC 所包含的物理内存的数量以及其他内容。pmap.c 中的代码需要读取这个设备硬件，以便弄清楚有多少物理内存，
但是这部分代码已经为你完成了：你不需要知道 CMOS 硬件如何工作的细节。

要特别注意 memlayout.h 和 mmap.h，因为本实验要求你使用和理解它们所包含的许多定义。你可能还想回顾一下 inc/mmu.h，因为它还包含了一些对本实验室有用的定义。

---

既然如此，我们可以对此些文件进行一些刨析

# memlayout.h

```
#define GD_KT     0x08     // kernel text
#define GD_KD     0x10     // kernel data
#define GD_UT     0x18     // user text
#define GD_UD     0x20     // user data
#define GD_TSS0   0x28     // Task segment selector for CPU 0
```
- 0x08 = 1000b: 索引1，TI=0（GDT），RPL=00（ring 0）
- 0x10 = 10000b: 索引2，TI=0（GDT），RPL=00（ring 0）
- 0x18 = 11000b: 索引3，TI=0（GDT），RPL=00（ring 0）
- 0x20 = 100000b: 索引4，TI=0（GDT），RPL=00（ring 0）

```
#define KERNBASE    0xF0000000  // 240MB边界，刚好是3/4的4GB空间
#define ULIM        (MMIOBASE)  // 用户空间上限
```
**为什么选择0xF0000000作为KERNBASE？**
1. 简化地址转换: 物理地址 + 0xF0000000 = 虚拟地址
2. 预留足够的用户空间: 3.75GB给用户，256MB给内核
3. 页表管理便利: 正好对应页目录的最后几个条目


## 内核栈设计的巧思
```
#define KSTACKTOP   KERNBASE
#define KSTKSIZE    (8*PGSIZE)    // 32KB栈空间
#define KSTKGAP     (8*PGSIZE)    // 32KB保护间隙
```
这种设计防止了栈溢出导致的内存损坏。

**MMIO区域的布局**
```
#define MMIOLIM     (KSTACKTOP - PTSIZE)  // 0xefc00000
#define MMIOBASE    (MMIOLIM - PTSIZE)    // 0xef800000
```
- 大小正好是PTSIZE（4MB），对应一个页目录项
- 位置在内核栈下方，便于管理

## 用户空间的精心设计
```
#define UVPT        (ULIM - PTSIZE)       // 用户虚拟页表
#define UPAGES      (UVPT - PTSIZE)       // 页面结构数组
#define UENVS       (UPAGES - PTSIZE)     // 环境结构数组
```
UVPT的巧妙之处:

- 用户进程可以读取自己的页表
- 实现了用户态的页表遍历
- 支持用户态的内存管理操作（如fork）

**栈布局的安全考虑**
```
#define UXSTACKTOP  UTOP              // 异常栈顶
#define USTACKTOP   (UTOP - 2*PGSIZE) // 普通栈顶，中间隔了一个保护页
```
两个栈之间的保护页防止异常处理时的栈溢出影响正常栈。

## 临时映射区域的用途

**UTEMP和PFTEMP的区别**
```
#define UTEMP       ((void*) PTSIZE)     // 0x400000
#define PFTEMP      (UTEMP + PTSIZE - PGSIZE) // 0x7ff000
```
- UTEMP: 内核为用户进程进行临时映射（如页面复制）
- PFTEMP: 用户态页面错误处理程序的临时映射
- 两者不冲突，PFTEMP在UTEMP区域的最后一页

## 类型定义的考虑
```
#ifndef __ASSEMBLER__
typedef uint32_t pte_t;  // 页表项
typedef uint32_t pde_t;  // 页目录项
#endif
```
使用条件编译是因为汇编代码不理解typedef，但需要使用这个头文件中的宏定义。



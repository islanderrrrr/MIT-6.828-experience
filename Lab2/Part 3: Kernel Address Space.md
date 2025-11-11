# 前言
JOS 会将处理器的 32 位线性地址空间分为两部分。我们将在 lab 3 中开始加载和运行用户环境(进程)，它将控制下面部分的布局和内容，而内核始终保持对上面部分的完全控制。在 inc/memlayout.h 中，分隔线由符号 ULIM 任意定义，为内核保留了大约 256MB 的虚拟地址空间。这解释了为什么我们需要在 lab 1 中给内核一个如此高的链接地址：否则内核的虚拟地址空间将没有足够的空间来同时映射到下面的用户环境中。

# 权限和故障隔离
由于内核和用户内存都存在于每个环境的地址空间中，所以我们必须在 x86 页表中使用权限位来允许用户代码只访问地址空间的用户部分。否则，用户代码中的 bug 可能会覆盖内核数据，导致崩溃或更微妙的故障；用户代码也可以窃取其他环境的私有数据。请注意，可写权限位(PTE_W)同时影响用户和内核代码！

[ULIM, ): 只有内核能够访问

[UTOP,ULIM): 都能够读，但是不能够write。实际上是内核向用户态主动暴露一些系统的信息，方便用户态进行读取和调用。

[0, UTOP): 用户态可以自由的进行权限的控制。

# 初始化内核地址空间

现在你可以设置 UTOP 之上的地址空间了：地址空间的内核部分。inc/memlayout.h 展示你应该使用的内存布局。

**练习5**：完成mem_init() 里面函数功能，使得能够通过 check_kern_pgdir() 和check_page_installed_pgdir() 。

## 映射 pages 数组到 UPAGES
```
boot_map_region(kern_pgdir, 
                UPAGES, 
                ROUNDUP(npages * sizeof(struct PageInfo), PGSIZE),
                PADDR(pages), 
                PTE_U | PTE_P);
```
**参数说明：**  
- kern_pgdir：内核页目录，要修改的页目录
- UPAGES：0xef000000，虚拟地址起始位置
- size：ROUNDUP(npages * sizeof(struct PageInfo), PGSIZE)，向上取整到页大小
- PADDR(pages)：pages 数组的物理地址，源物理地址
- PTE_U：PTE_P，用户可读，内核可读写`
**内存布局：**  
```
虚拟地址                      物理地址
┌─────────────────┐
│ UPAGES          │ ────────→ │ pages 数组      │
│ 0xef000000      │           │ (PageInfo[])    │
│                 │           │                 │
│ 用户只读(R-)    │           │ 物理内存中的    │
│ 内核可读写(RW)  │           │ 某个位置        │
└─────────────────┘           └─────────────────┘
     512 KB                        512 KB
```

## 映射内核栈
```
boot_map_region(kern_pgdir,
                KSTACKTOP - KSTKSIZE,
                KSTKSIZE,
                PADDR(bootstack),
                PTE_W | PTE_P);
```

**参数说明：**  

- kern_pgdir：内核页目录，要修改的页目录
- KSTACKTOP - KSTKSIZE： 0xf0000000 - 0x8000 = 0xefff8000，栈底虚拟地址
- KSTKSIZE：0x8000 = 32 KB，栈大小
- PADDR(bootstack)：bootstack 的物理地址，在 entry.S 中定义
- `PTE_W：PTE_P`，内核可读写

内核栈的特殊设计：

```
虚拟地址布局：

KSTACKTOP (0xf0000000)
         ↓
    ┌─────────────────┐  ← 栈顶（向下增长）
    │                 │
    │  Kernel Stack   │  32 KB (KSTKSIZE)
    │  已映射物理内存  │  权限: PTE_W | PTE_P
    │                 │  (内核可读写，用户不可访问)
    └─────────────────┘  ← KSTACKTOP - KSTKSIZE
    │                 │
    │  Guard Page     │  4MB - 32KB (未映射！)
    │  故意不映射      │  如果栈溢出访问这里 → Page Fault
    │                 │  防止栈溢出破坏其他内存
    └─────────────────┘  ← KSTACKTOP - PTSIZE
```
保护页（Guard Page）机制：  

```
// PTE_W | PTE_P 但没有 PTE_U
// 意味着：
// - 内核（CPL=0）：可读可写 ✓
// - 用户（CPL=3）：完全无法访问 ✗（Page Fault）
```

## 映射所有物理内存到 KERNBASE

```
boot_map_region(kern_pgdir,
                KERNBASE,
                0xFFFFFFFF - KERNBASE + 1,
                0,
                PTE_W | PTE_P);
```

**参数说明：**

- kern_pgdir： 内核页目录， 要修改的页目录
- KERNBASE：0xF0000000，虚拟地址起始
- 0xFFFFFFFF - KERNBASE + 1：256 MB，映射大小
- 0：物理地址 0，从物理地址 0 开始
- `PTE_W：PTE_P`， 内核可读写

**大小计算：**

```
// 虚拟地址空间只有 32 位 = 4GB
// KERNBASE 以上的空间：
size = 0xFFFFFFFF - 0xF0000000 + 1
     = 0x10000000
     = 268435456 bytes
     = 256 MB

// 注意：实际物理内存可能少于 256MB
// 但我们仍然设置这个映射
// 如果访问不存在的物理内存，由硬件处理
```

**映射示意图：**

```
虚拟地址                    物理地址
┌───────────────┐           ┌───────────────┐
│ 0xFFFFFFFF    │ ────────→ │ 256 MB        │
│               │           │ (如果存在)    │
│               │           │               │
│ ...           │ ────────→ │ ...           │
│               │           │               │
│ 0xF0400000    │ ────────→ │ 4 MB          │
│ 0xF0000000    │ ────────→ │ 0 (物理地址0) │
│ (KERNBASE)    │           │               │
└───────────────┘           └───────────────┘
```

**为什么需要这个映射？**

```
// 1. 内核需要访问物理内存
//    但 x86 分页模式下，CPU 只能使用虚拟地址
//    通过这个映射，内核可以：
physaddr_t pa = 0x00100000;  // 某个物理地址
char *va = (char *)KADDR(pa);  // 转换为虚拟地址 0xF0100000
*va = 'A';  // 现在可以访问了！

// 2. KADDR 和 PADDR 宏的实现
#define KADDR(pa) ((void *)((pa) + KERNBASE))
#define PADDR(va) ((physaddr_t)((va) - KERNBASE))

// 例子：
// 物理地址 0x00001000 → 虚拟地址 0xF0001000
// 虚拟地址 0xF0001000 → 物理地址 0x00001000
```

## 三个映射的对比

| 映射        | 虚拟地址               | 物理地址     | 大小       | 权限               | 用途         |
|-------------|------------------------|--------------|------------|--------------------|--------------|
| UPAGES      | 0xef000000             | pages 数组   | ~512 KB    | `PTE_U`    |        `PTE_P`       |
| Kernel Stack| KSTACKTOP-KSTKSIZE     | bootstack    | 32 KB      | `PTE_W`     |       `PTE_P`       |
| KERNBASE    | 0xf0000000             | 0            | 256 MB     | `PTE_W`     |       `PTE_P`       |


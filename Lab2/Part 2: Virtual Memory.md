# 虚拟地址、线性地址和物理地址
在 x86 术语中，虚拟地址由段选择器和段内的偏移量组成。线性地址是在段转换之后，页面转换之前得到的地址。一个物理地址是你在段和页转换之后最终得到的，它最终会通过硬件总线发送到你的 RAM。
```

           Selector  +--------------+         +-----------+
          ---------->|              |         |           |
                     | Segmentation |         |  Paging   |
Software             |              |-------->|           |---------->  RAM
            Offset   |  Mechanism   |         | Mechanism |
          ---------->|              |         |           |
                     +--------------+         +-----------+
            Virtual                   Linear                Physical

```

C 指针是虚地址的“偏移量”组件。在 boot/boot.S，我们建立了一个全局描述表(Global Descriptor Table， GDT)，通过将所有段的基址设置为 0 并限制至 0xffffffff，有效地禁用了段转换。因此，“选择子”没有作用，线性地址总是等于虚拟地址的偏移量。在实验 3 中，我们将不得不与分段机制进行更多的交互，以设置特权级别，但是对于内存转换，我们可以忽略整个 JOS 实验室中的分段机制，只关注分页机制。

回想一下，在实验 1 的第 3 部分，我们创建了一个简单的页表，以便内核可以在它的链接地址 0xf0100000 处运行，尽管它实际上是在物理内存中加载的，就在 ROM BIOS 上面的 0x00100000 处。这个页表只映射了 4MB 的内存。在这个实验室中，你将为 JOS 设置虚拟地址空间布局，我们将对其进行扩展，以映射 0xf0000000 开始的虚拟地址到物理内存的前 256MB，并映射虚拟地址空间的许多其他区域。

# 引用计数
在之后的实验室中，你通常会将相同的物理页面同时映射到多个虚拟地址(或多个环境的地址空间)。你将在与物理页对应的结构 PageInfo 的 pp_ref 字段中保持对每个物理页的引用数量的计数。当物理页的这个计数为 0 时，该页可以被释放，因为它不再被使用。通常，这个计数应该等于物理页面在所有页表中出现在 UTOP 下面的次数(UTOP 上面的映射大部分是在内核启动时设置的，不应该被释放，所以不需要引用计数)。我们还将使用它来跟踪指向页目录页的指针的数量，以及页目录对页表页的引用数量。

使用 page_alloc 时要小心。它返回的页面的引用计数总是 0，所以 pp_ref 应该在你对返回的页面做了一些操作(比如将其插入到页表中)之后递增。有时这是由其他函数(例如，page_insert)处理的，有时调用 page_alloc 的函数必须直接执行。

# 页表管理

## pgdir_walk() - 页表遍历函数

**函数作用**  
给定一个虚拟地址，找到它对应的页表项（PTE）的指针。这是页表操作的核心函数。

**背景知识**  
```
虚拟地址 → 页目录(Page Directory) → 页表(Page Table) → 物理页
```

```
虚拟地址结构（32位）：
┌──────────┬──────────┬────────────┐
│ 10 bits  │ 10 bits  │  12 bits   │
│   PDX    │   PTX    │   Offset   │
└──────────┴──────────┴────────────┘
 页目录索引  页表索引    页内偏移
```

**实现思路**  
```
pte_t *
pgdir_walk(pde_t *pgdir, const void *va, int create)
{
    // 步骤1: 从虚拟地址中提取页目录索引
    uint32_t pdx_index = PDX(va);  // 高10位
    
    // 步骤2: 从虚拟地址中提取页表索引
    uint32_t ptx_index = PTX(va);  // 中间10位
    
    // 步骤3: 获取页目录项的指针
    pde_t *pde = &pgdir[pdx_index];
    
    // 步骤4: 检查页表是否存在（PTE_P 是 Present 位）
    if (!(*pde & PTE_P)) {
        // 页表不存在
        if (!create) {
            return NULL;  // 不创建，返回 NULL
        }
        
        // 步骤5: 分配新的页表页
        struct PageInfo *new_page = page_alloc(ALLOC_ZERO);
        if (!new_page) {
            return NULL;  // 分配失败
        }
        
        // 步骤6: 增加引用计数（重要！）
        new_page->pp_ref++;
        
        // 步骤7: 设置页目录项
        // page2pa(): PageInfo* → 物理地址
        // PTE_P: Present（存在）
        // PTE_W: Writable（可写）
        // PTE_U: User accessible（用户可访问）
        *pde = page2pa(new_page) | PTE_P | PTE_W | PTE_U;
    }
    
    // 步骤8: 获取页表的虚拟地址
    // PTE_ADDR(*pde): 提取物理地址（清除标志位）
    // KADDR(): 物理地址 → 虚拟地址
    pte_t *page_table = (pte_t *)KADDR(PTE_ADDR(*pde));
    
    // 步骤9: 返回对应的页表项指针
    return &page_table[ptx_index];
}
```

## boot_map_region() - 静态映射函数
**函数作用**  

在页表中建立静态映射，将一段连续的虚拟地址映射到连续的物理地址。主要用于内核启动时的映射（如映射内核代码、设备内存等）。  

**应用场景**

```
示例：映射物理内存 [0, 256MB) 到虚拟地址 [0xf0000000, 0xf0000000+256MB)
boot_map_region(kern_pgdir, 0xf0000000, 256*1024*1024, 0, PTE_W);
```

**实现思路**
```
static void
boot_map_region(pde_t *pgdir, uintptr_t va, size_t size, physaddr_t pa, int perm)
{
    // 步骤1: 遍历需要映射的每一页
    // size 是总字节数，PGSIZE 是页大小（4KB = 4096字节）
    for (size_t i = 0; i < size; i += PGSIZE) {
        
        // 步骤2: 获取对应虚拟地址的页表项
        // create=1 表示如果页表不存在就创建
        pte_t *pte = pgdir_walk(pgdir, (void *)(va + i), 1);
        
        if (!pte) {
            panic("boot_map_region: page table allocation failed");
        }
        
        // 步骤3: 设置页表项
        // pa + i: 对应的物理地址
        // perm: 权限位（如 PTE_W）
        // PTE_P: Present 位必须设置
        *pte = (pa + i) | perm | PTE_P;
    }
    
    // 注意：这个函数不修改 pp_ref！
    // 因为是静态映射，这些页不会被释放
}
```

**示意图**  

```
虚拟地址空间                    物理地址空间
┌─────────────┐
│ 0xf0000000  │──────────────→ │ 0x00000000  │
│ 0xf0001000  │──────────────→ │ 0x00001000  │
│ 0xf0002000  │──────────────→ │ 0x00002000  │
│     ...     │      ...        │     ...     │
└─────────────┘                 └─────────────┘
```

## page_lookup() - 页查找函数
**函数作用**  

根据虚拟地址查找对应的物理页（PageInfo 结构），可选返回页表项指针。  

**使用场景**

- 检查某个虚拟地址是否已映射
- 获取映射的物理页信息
- 为 page_remove() 等函数提供支持

**实现思路**

```
struct PageInfo *
page_lookup(pde_t *pgdir, void *va, pte_t **pte_store)
{
    // 步骤1: 获取页表项
    // create=0 表示不创建新页表
    pte_t *pte = pgdir_walk(pgdir, va, 0);
    
    // 步骤2: 检查页表项是否存在且有效
    if (!pte || !(*pte & PTE_P)) {
        return NULL;  // 未映射
    }
    
    // 步骤3: 如果调用者需要 PTE 指针，保存它
    if (pte_store) {
        *pte_store = pte;
    }
    
    // 步骤4: 返回对应的 PageInfo 结构
    // PTE_ADDR(*pte): 提取物理地址
    // pa2page(): 物理地址 → PageInfo*
    return pa2page(PTE_ADDR(*pte));
}
```

## page_remove() - 页移除函数

**函数作用**  

取消虚拟地址的映射，释放引用的物理页（如果引用计数降为0）。  

**实现思路**  

```
void
page_remove(pde_t *pgdir, void *va)
{
    pte_t *pte;
    
    // 步骤1: 查找虚拟地址对应的物理页和 PTE
    struct PageInfo *page = page_lookup(pgdir, va, &pte);
    
    // 步骤2: 如果没有映射，直接返回
    if (!page) {
        return;
    }
    
    // 步骤3: 减少引用计数
    // page_decref() 会在计数为0时自动释放页
    page_decref(page);
    
    // 步骤4: 清除页表项
    *pte = 0;
    
    // 步骤5: 刷新 TLB（重要！）
    // CPU 会缓存虚拟地址到物理地址的映射
    // 必须告诉 CPU 这个映射已经失效
    tlb_invalidate(pgdir, va);
}
```

## page_insert() - 页插入函数

**函数作用**  

将物理页映射到指定的虚拟地址，处理所有复杂情况（如重新映射）。  

**实现思路**

```
int
page_insert(pde_t *pgdir, struct PageInfo *pp, void *va, int perm)
{
    // 步骤1: 获取或创建页表项
    // create=1 表示如果页表不存在就创建
    pte_t *pte = pgdir_walk(pgdir, va, 1);
    if (!pte) {
        return -E_NO_MEM;  // 页表分配失败
    }
    
    // 步骤2: 先增加引用计数（关键！）
    // 这样做可以优雅地处理同一物理页重新映射的情况
    pp->pp_ref++;
    
    // 步骤3: 如果虚拟地址已经有映射，移除旧映射
    if (*pte & PTE_P) {
        page_remove(pgdir, va);
        // page_remove 会减少引用计数
    }
    
    // 步骤4: 设置新的页表项
    *pte = page2pa(pp) | perm | PTE_P;
    
    return 0;
}
```

## 完整流程图
```
page_insert(pgdir, pp, va, perm)
         |
         v
┌────────────────────┐
│ 获取/创建页表项 PTE │
└────────┬───────────┘
         |
         v
┌────────────────────┐
│  pp->pp_ref++      │  先增加引用（重要！）
└────────┬───────────┘
         |
         v
    ┌────────┐
    │ PTE 已 │ 是
    │ 映射？ ├───→ page_remove(pgdir, va)
    └───┬────┘           ↓
        |否           减少旧页引用
        |               ↓
        └──────→ ┌──────────────┐
                 │ 设置新 PTE    │
                 └──────────────┘
```

# 总结：五个函数的关系
```
              pgdir_walk()
                  ↑
                  | 被所有函数使用
    ┌─────────────┼─────────────┐
    |             |              |
    v             v              v
boot_map_region  page_lookup  page_insert
    |             ↑              ↑
    |             |              |
    |             └──page_remove─┘
    |                    ↑
    └────────────────────┘
         用于内核启动
```

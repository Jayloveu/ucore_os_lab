### OS Lab 2 Report
#### 计55  陈齐斌  2015011403

### 练习 1：实现 first-fit 连续物理内存分配算法
#### 1.1 alloc_pages
- 给定的分配算法已经完成大半。需要解决的问题是，块插入free_list操作严格按地址顺序。解决此问题即可。

```c
if (page != NULL) {
    if (page->property > n) {
        struct Page *p = page + n;
        p->property = page->property - n;
        SetPageProperty(p);
        list_add_after(&(page->page_link), &(p->page_link));
    }
    list_del(&(page->page_link));
    nr_free -= n;
    ClearPageProperty(page);
}
```

#### 1.2 default_free_pages

- 给定的释放算法已经完成大半。需要解决的问题与 alloc_pages 类似。
- 由于之前把合并后的整块 free 的空间都从 `free_list` 里中删除，之后遍历一下，找到它应该在的位置即可。

```c
le = list_next(&free_list);
while (le != &free_list) {
    p = le2page(le, page_link);
    le = list_next(le);
    if (base + base->property < p) {
        break;
    }
}
list_add_before(le, &(base->page_link));
```

#### 1.3 first-fit 进一步的改进空间？

- 改进：
    - 释放空间在合并时，无需从头开始遍历，只需检查空闲块前后是否可以合并即可，但是限于框架，无法方便的实现。

### 练习 2：实现寻找虚拟地址对应的页表项（需要编程）
#### 2.1 代码实现
- 虚拟地址对应的页表项，如果不存在且需要创建则新建页表项，需要处理创建过程中失败的情况。
- 具体实现思路见下段注释。

```c
pte_t *
get_pte(pde_t *pgdir, uintptr_t la, bool create) {
    // 页目录项 = 页目录首地址[线性地址]
    pde_t *pdep = &pgdir[PDX(la)];
    if (!(*pdep & PTE_P)) {
        // if 不创建 return 空
        // if 新建一个页表为空(内存分配失败) return 空
        if (!create) return NULL;
        struct Page *page;
        page = alloc_page();
        if (page == NULL) return NULL;
        // 设置页表引用
        set_page_ref(page, 1);
        // 得到页表首地址
        uintptr_t pa = page2pa(page);
        // 将页表内容清空
        memset(KADDR(pa), 0, PGSIZE);
        // 页目录项内容 = 页表首地址 + 所有标志位全开
        *pdep = pa | PTE_U | PTE_W | PTE_P;
    }
    return &((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)];
}
```

#### 2.2 请描述页目录项(Page Directory Entry)和页表(Page Table Entry)中每个组合部分的含义和以及对ucore而言的潜在用处。

- 查看 `kern/mm/mmu.h` 中关于 PDE 和 PTE 的宏定义。

```c
/* page table/directory entry flags */
#define PTE_P           0x001                   // Present
#define PTE_W           0x002                   // Writeable
#define PTE_U           0x004                   // User
#define PTE_PWT         0x008                   // Write-Through
#define PTE_PCD         0x010                   // Cache-Disable
#define PTE_A           0x020                   // Accessed
#define PTE_D           0x040                   // Dirty
#define PTE_PS          0x080                   // Page Size
#define PTE_MBZ         0x180                   // Bits must be zero
#define PTE_AVAIL       0xE00                   // Available for software use
```

- 对ucore而言，有如下作用：
    - 维护二级页表，进行虚地址与实地址的转换。
    - 设置各种功能，通过修改标记位实现，例如记录状态，设置访问权限等。

#### 2.3 如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？
- 硬件将进行以下操作：
    - 把产生异常的线性地址存储在CR2寄存器中；
    - 保存中断现场；

### 练习 3：释放某虚地址所在的页并取消对应二级页表项的映射（需要编程）

#### 3.1 代码实现如下，过程见注释。
```c
static inline void
page_remove_pte(pde_t *pgdir, uintptr_t la, pte_t *ptep) {
    if (!(*ptep & PTE_P)) return;
    struct Page *page = pte2page(*ptep);
    // 如果页引用为 0，释放该页
    if (page_ref_dec(page) == 0) {
        free_page(page);
    }
    // 清除页表项
    *ptep = 0;
    // 更新TLB
    tlb_invalidate(pgdir, la);
}
```

#### 3.2 数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？
- 有，由 `kern/mm/pmm.h` 的 `pa2page` 函数和`kern/mm/mmu.h`的`PPN`宏可知。
- 在从页表中的页目录项和页表项得到页的物理地址后，再根据 $physical address >> 12$ 得到对应的页pages[index]。

#### 3.3 如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？鼓励通过编程来具体完成这个问题。
- 当前的 Lab2 建立的映射关系为：`virtual addr = linear addr = physical addr + 0xC0000000`。
- `kern/mm/memlayout.h`中`#define KERNBASE 0xC0000000`改为`#define KERNBASE 0x00000000`
- `tools/kernel.ld`中`. = 0xC0100000`改为`. = 0x100000`
- 修改后能正常运行。

### 4 总结
#### 4.1 与参考答案区别：
- 练习 1
  1. `default_init_memmap`函数结尾，答案为`list_add_before`，事实上跟`list_add`在这里没有不同。
  2. `default_alloc_pages`函数中，答案调用了`SetPageProperty(p)`，由于一开始没写也没有问题，因此不太明白这句话的作用。

- 练习 2
    - 基本一致，有一部分参考了答案。

- 练习3
    - 基本一致。

#### 4.2 重要知识点：
- 物理内存分配算法（first-fit内存管理算法）
- 分页机制，二级页表，虚实地址转换

#### 4.3 缺失知识点：
- 其余的内存（如best-fit）分配算法

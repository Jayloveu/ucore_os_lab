### OS Lab 3 Report

#### 计55  陈齐斌  2015011403

### 练习 1：给未被映射的地址映射上物理页（需要编程）

#### 1.1 设计实现过程

- 首先使用 `get_pte` 得到 page fault 信息中虚拟地址所对应的 page table entry (若不存在则创建之)，若创建失败则无法进行后续处理。
- 在得到页表项后，如果该页表项为空，则说明目标虚拟地址指向的空间还未分配，则通过申请物理页面并添加映射来解决此 page fault。
- 这样就完成了练习 1。事实上，`pgdir_alloc_page` 中有可能发生页换出，虽然 `fifo` 还未实现，但不影响第一部分的完成。

```c
if ((ptep = get_pte(mm->pgdir, addr, 1)) == NULL) {
    goto failed;
}
if (*ptep == 0)
{
    if (pgdir_alloc_page(mm->pgdir, addr, perm) == NULL) {
        goto failed;
    }
}
```

#### 1.2 请描述页目录项（Pag Director Entry）和页表项（Page Table Entry）中组成部分对ucore实现页替换算法的潜在用处。

- `PDE_P` 确定了二级页表是否存在，如果不存在，则需要创建一个；`PTE_P` 同理。
- 进行 swap out 时，需要确保这些页在内存中是存在的，可以用 assert 这些位帮助验证。
- 另外，控制读写权限权限的几位如 `PTE_W` 等，在 pgfault 需要新建物理页时需要设定这些位以控制权限。

#### 1.3 如果ucore的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

- 与一般的页访问异常一样，把产生异常的地址存储在 CR2 中，并且把页访问异常错误码 errorCode 保存在中断栈中。

### 练习 2：补充完成基于FIFO的页面替换算法（需要编程）

#### 2.1 设计实现过程

- 首先在 `vmm.c` 中，紧接练习 1，处理该页面已有映射的情况，即对应的页面在外存中。
- 因此将页面从硬盘中读出放到内存，注意 `swap_in` 中需要新建内存空间，以存放外存中读入的数据，这个过程中可能会发生页换出。
- 读入数据后，重新建立映射关系，并通过 `swap_map_swappable` 把它加入页面替换算法中。

```c
if (swap_init_ok) {
    struct Page *page;
    swap_in(mm, addr, &page);
    page_insert(mm->pgdir, page, addr, perm);
    swap_map_swappable(mm, addr, page, 1);
    page->pra_vaddr = addr;
}
else {
    cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);
    goto failed;
}
```

- 此时在 `do_pgfault` 中外部调用的接口已经完成，还剩 `fifo` 的具体实现。
- 在 `swap_fifo.c` 中，由于新加入页的内容已经在 `swap_in` 中处理完毕，映射也做好了，只需加到队列中，用 `list_add_before(head, entry);` 即可完成
- 调出页面时，从队列头部取出并从列表中删除，然后将该页面的地址通过 `ptr_page` 传递出去，给 `swap_out` 的后续剩余操作，如写到外存中使用。

```c
static int
_fifo_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in)
{
    list_entry_t *head=(list_entry_t*) mm->sm_priv;
    list_entry_t *entry=&(page->pra_page_link);
    assert(entry != NULL && head != NULL);
    //record the page access situlation
    //(1)link the most recent arrival page at the back of the pra_list_head qeueue.
    list_add_before(head, entry);
    return 0;
}
```

```c
static int
_fifo_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick)
{
    list_entry_t *head=(list_entry_t*) mm->sm_priv;
    assert(head != NULL);
    assert(in_tick==0);
    /* Select the victim */
    /*LAB3 EXERCISE 2: YOUR CODE*/ 
    //(1)  unlink the  earliest arrival page in front of pra_list_head qeueue
    //(2)  assign the value of *ptr_page to the addr of this page
    assert(head->next != head);
    list_entry_t *victim_le = head->next;
    list_del(victim_le);
    *ptr_page = le2page(victim_le, pra_page_link);
    return 0;
}
```

#### 2.2 如果要在ucore上实现"extended clock页替换算法"请给你的设计方案，现有的swap_manager框架是否足以支持在ucore中实现此算法？如果是，请给你的设计方案。如果不是，请给出你的新的扩展和基此扩展的设计方案。并需要回答如下问题

可以支持。首先考虑 clock 替换算法，在访问 page 的时候，由于 `Page` 类中可以找到用于替换算法的 clock 链表中的结点，因此可以设置访问位等，此外再用一个指针表示时针，记录下次开始按 clock 找的位置，即可实现；对于 extended clock，由于 pte 中含有 dirty bit，因此可以使用这个信息来作为修改位的判断，这一位的数据应由硬件写入。

- 需要被换出的页的特征是什么？
  - 特征由所使用的页替换算法决定，如在 `FIFO` 中，特征是最早被调入的页面；在时钟置换算法中，特征是上一轮中没有被访问且当前被轮到的页面；在改进的时钟算法中，还需要是上一轮没有被访问到的页面。
- 在ucore中如何判断具有这样特征的页？
  - 在 `mm_struct` 维护的页的列表中，可以方便的找到最早被掉入的页面，即队列头；在时钟算法中，可以添加一个附加的指针来模拟时针的位置，并根据上的访问位、修改问等判断是否具有这个特征。
- 何时进行换入和换出操作？
  - 当因访问的页不在内存中而产生 pgfault 时，do_pgfault 会进行换入操作。
  - 在尝试新建内存中的物理页，即所有使用 `alloc_page` 的地方，都有可能进行换出操作。

### 4 总结

#### 4.1 与参考答案区别：

- 练习 1
  1. 参考答案输出了失败时的调试信息，如，`cprintf("pgdir_alloc_page in do_pgfault failed\n");`，自己第一次做的时候没有写到。

- 练习 2
  - 参考答案将 `head->prev` 作为队列开头，`head->next` 作为队列尾，与我的实现正好相反。问题不大。
  - 参考答案在 `_fifo_swap_out_victim` 函数中的 `le2page` 后增加了一句，`assert` 得到的 page 地址非 NULL，更有助于错误的发现。
  - `vmm.c` 中有部分参考了答案完成。

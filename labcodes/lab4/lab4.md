### OS Lab 4 Report

#### 计55  陈齐斌  2015011403

### 练习 1：分配并初始化一个进程控制块（需要编码）

#### 1.1 设计实现过程

- 根据提示，在 `alloc_proc` 函数的实现中，需要初始化的proc_struct结构中的成员变量至少包括：state/pid/runs/kstack/need_resched/parent/mm/context/tf/cr3/flags/name
- 以下为各属性的对应的初始化过程。需要注意的是 `proc->context` 为 struct, `proc->name` 为 char[]，使用了 `memset` 函数进行初始化。

```c
proc->state = PROC_UNINIT;
proc->pid = -1;
proc->runs = 0;
proc->kstack = 0;
proc->need_resched = 0;
proc->parent = NULL;
proc->mm = NULL;
memset(&(proc->context), 0, sizeof(struct context));
proc->tf = NULL;
proc->cr3 = boot_cr3;
proc->flags = 0;
memset(proc->name, 0, PROC_NAME_LEN + 1);
```

#### 1.2 请说明proc_struct中struct context context和struct trapframe *tf成员变量含义和在本实验中的作用是啥？（提示通过看代码和编程调试可以判断出来）

1. 根据 `struct context` 的如下定义可知，此为控制流切换时的上下文，即 cpu 的各需要保存的寄存器。eax 作为返回值存放寄存器不包含在内。在此实验中的作用相当于这几个寄存器的 placeholder，在 `switch_to` 函数进行上下文切换时用到。

```c
struct context {
    uint32_t eip;
    uint32_t esp;
    uint32_t ebx;
    uint32_t ecx;
    uint32_t edx;
    uint32_t esi;
    uint32_t edi;
    uint32_t ebp;
};
```

2. 根据 trap.h 中对 `struct trapframe` 的定义，可知此为中断帧。在此实验中内核线程切换时用来表示中断帧的空间。

### 练习 2：为新创建的内核线程分配资源（需要编码）

#### 2.1 设计实现过程

- 根据提示，需要处理各个步骤中不成功的情况。实验中依次将这些情况 goto 到各个清理现场的入口。
- 前面 1-4 步较简单，需要注意的是后面 `get_pid`, `hash_proc`, `list_add`, `nr_process++` 等操作需要禁用中断。这是因为链表操作不是线程安全的，需要加锁，或变为原子操作。且 `get_pid` 中涉及到链表的访问，`nr_process++` 涉及总进程数不超过 `MAX_PROCESS`。此处实现参考了标答

```c
if ((proc = alloc_proc()) == NULL) {
    goto fork_out;
}
proc->parent = current;
if (setup_kstack(proc) == -E_NO_MEM) {
    goto bad_fork_cleanup_proc;
}
if (copy_mm(clone_flags, proc) != 0) {
    goto bad_fork_cleanup_kstack;
}

copy_thread(proc, stack, tf);

bool intr_flag;
local_intr_save(intr_flag);
{
    proc->pid = get_pid();
    hash_proc(proc);
    list_add(&proc_list, &(proc->list_link));
    nr_process++;
}
local_intr_restore(intr_flag);

wakeup_proc(proc);
ret = proc->pid;
```

#### 2.2 请说明ucore是否做到给每个新fork的线程一个唯一的id？请说明你的分析和理由。

- 是，首先 `get_pid` 不允许中断。
- 并且 `get_pid` 的分配 pid 的策略是在 [0, MAX_PID) 间找一个 proc_list 中没有的 pid，用来返回，因此每个进程都唯一的 pid。

### 练习 3 阅读代码，理解 proc_run 函数和它调用的函数如何完成进程切换的。（无编码工作）

- 请在实验报告中简要说明你对proc_run函数的分析。并回答如下问题：
  - 在本实验的执行过程中，创建且运行了几个内核线程？
  - 语句local_intr_save(intr_flag);....local_intr_restore(intr_flag);在这里有何作用?请说明理由

- `proc_run` 分析：
  - 首先需要保证执行下面的过程不能被打断，因此如果中断开启，则关闭之。
  - `current = proc` 即改变 current 指向的 process
  - 设置 esp0 新线程的栈顶 next->kstack + KSTACKSIZE
  - `lcr3(next->cr3)` 完成了进程间的页表切换，此实验中由于都是内核线程，实际上都指向 bootcr3
  - `switch_to(&(prev->context), &(next->context));` 完成执行上下文的切换。

- 创建并运行了两个，分别为 `idleproc` 和 `initproc`
- local_intr_save(intr_flag);....local_intr_restore(intr_flag); 为关闭中断（如果使能），使能中断（如果关闭）。这里可以使得两句话中间的语句不被打断，可以保证进程切换的正确性。

### 4 总结

#### 4.1 与参考答案区别：

- 练习 2
  - 忘记设置 `proc->parent = proc`，比较后补上。
  - 关闭并使能中断的部分参考了标准实现。

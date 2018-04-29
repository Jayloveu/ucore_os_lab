### OS Lab 5 Report

#### 计55  陈齐斌  2015011403

### 练习 1：加载应用程序并执行（需要编码）

#### 1.1 设计实现过程

- `load_icode` 函数中，程序先加载并解析一个处于内存中的ELF执行文件格式的应用程序
- 然后建立相应的用户内存空间，并拷贝了用程序的代码段、数据段等
- 需要编程实现的是新 trapframe 的设置
- 根据提示可知，需要设置的变量有 `tf_cs, tf_ds, tf_es, tf_ss, tf_esp, tf_eip, tf_eflags`
- 根据宏定义 `USER_CS`, `USER_DS`, `USTACKTOP` 可设置各寄存器的值，并且由 elf->e_entry 的值可以知道编译时确定的程序的入口地址
- 需要能处理中断，因此还需设 eflags 为 FL_IF

```c
tf->tf_cs = USER_CS;
tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
tf->tf_esp = USTACKTOP;
tf->tf_eip = elf->e_entry;
tf->tf_eflags = FL_IF;
```

#### 1.2 请在实验报告中描述当创建一个用户态进程并加载了应用程序后，CPU是如何让这个应用程序最终在用户态执行起来的

- 在 `load_icode` 函数中，current 进程将用户程序加载到了当前进程中，之后设置进程名后 `do_execve` 结束
- 返回到 `sys_exec` -> `syscall` -> `trap_dispatch` -> `trap` 函数，在结束中断（系统调用）处理后，执行 iret 在此出恢复中断帧，在返回跳转时即到了用户程序的入口地址。

### 练习 2：父进程复制自己的内存空间给子进程（需要编码）

#### 2.1 设计实现过程

- 根据提示，可以使用 `memcpy` 直接对源和目的虚拟内存空间进行拷贝，每次新建并拷贝一个 page
- 使用 page2kva(page) 得到对应页管理的虚拟地址
- 最后建立为新 page 建立映射 page_insert(to, npage, start, perm);

```c
void * kva_src = page2kva(page);
void * kva_dst = page2kva(npage);

memcpy(kva_dst, kva_src, PGSIZE);

ret = page_insert(to, npage, start, perm);
```

#### 2.2 请在实验报告中简要说明如何设计实现"Copy on Write 机制"，给出概要设计，鼓励给出详细设计。

- Copy on Write 机制
- 当一个用户父进程创建自己的子进程时，父进程把其申请的用户空间权限设置为只读，子进程共享父进程占用的用户内存空间中的页面，避免了不必要的拷贝。
- 其中一个进程修改某共享页面时，ucore 会通过 `do_pgfault` 中异常获知该操作，并可以完成拷贝内存页面，而这个进程可以正常继续运行，另一个进程也不会感知到变化。
- 即在 `copy_range` 函数并不进行深拷贝，而是直接设 to 同 from。并且更改权限为只读，并且缺页处理中加上相应的处理代码。

### 练习 3：阅读分析源代码，理解进程执行 fork/exec/wait/exit 的实现，以及系统调用的实现（不需要编码）

- 请在实验报告中简要说明你对 fork/exec/wait/exit函数的分析。并回答如下问题：
  - 请分析fork/exec/wait/exit在实现中是如何影响进程的执行状态的？
  - 请给出ucore中一个用户态进程的执行状态生命周期图（包执行状态，执行状态之间的变换关系，以及产生变换的事件或函数调用）。（字符方式画即可）”

- exec 加载了用户程序的内容并更新了中断帧，使得返回时可以改变执行地址，将自己继续执行。
- exit 则先释放了当前进程能释放的自己的空间，改变状态则是把当前进程变为僵尸态，并调用 reschedule
- fork 则比较复杂，调用了 wake_up
- wait 将当前进程设为 sleeping 并 reschedule
- ![执行状态周期图]()

### 4 总结

- 除了上述内容，`trap.c`, `proc.` 还有几处更新了之前几个 lab 的处理过程。

#### 4.1 与参考答案区别：

- `trap.c` 中一开始没有注释掉 print_ticks 调用，由于使得输出不一致导致 make grade 判错，与参考答案对比后发现。

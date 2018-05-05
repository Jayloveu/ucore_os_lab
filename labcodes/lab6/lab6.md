### OS Lab 6 Report

#### 计55  陈齐斌  2015011403

### 练习 1：使用 Round Robin 调度算法（不需要编码）

#### 1.1 设计实现过程

- 首先更新进程块的初始化，设置 `rq, run_link, time_slice` 等，顺便把 stride scheduling 算法需要的变量也初始化。

```c
proc->rq = NULL;
list_init(&(proc->run_link));
proc->time_slice = 0;
proc->lab6_run_pool.left = proc->lab6_run_pool.right = proc->lab6_run_pool.parent = NULL;
proc->lab6_stride = 0;
proc->lab6_priority = 1;
```

- 在 `trap.c` 中，用 `sched_class_proc_tick(current);` 接口代替先前的 `current->need_resched = 1;` 以支持各种调度算法。
- 运行 make grade 直到 priority 之前都成功。

#### 1.2 请理解并分析sched_calss中各个函数指针的用法，并接合Round Robin 调度算法描ucore的调度执行过程

```c
struct sched_class default_sched_class = {
    .name = "stride_scheduler",
    .init = stride_init,
    .enqueue = stride_enqueue,
    .dequeue = stride_dequeue,
    .pick_next = stride_pick_next,
    .proc_tick = stride_proc_tick,
};
```

- `init` 为初始化运行队列，在调度器初始化时调用
- `enqueue` 将进程 p 插入队列 rq，即有新进程可调度（就绪），或者当前进程需要被重新调度时，将调用 enqueue。
- `dequeue` 将进程 p 从队列 rq 中删除，在被选中执行之后，讲该即将运行的进程从就绪队列中删除。
- `pick_next` 则根据相应的调度算法，返回运行队列中应该被执行的进程
- `proc_tick` 在时钟中断到来时调用，可以改变队列中与调度有关的属性，比如当前进程的时间片等。

#### 1.3 请在实验报告中简要说明如何设计实现”多级反馈队列调度算法“，给出概要设计，鼓励给出详细设计

- 在优先级高的队列中的进程总是先被调度。时间片随着优先级的降低而越来越短。
- 对于同一个队列中的进程，按照时间片轮转法调度。
- 但如果一个队列中某个进程在 N 个时间片内还没有完成，则会进入下一个优先级队列。

### 练习 2：实现 Stride Scheduling 调度算法（需要编码）

#### 2.1 设计实现过程

- 首先确定 BIG_STRIDE 的大小。由于需要在 32 位 int 内表示 stride 之差，而 stride 之差的绝对值 <= BIG_STRIDE。要使得差不上溢或下溢，就设 BIG_STRIDE 为 0x7FFFFFFF。
- `init` 中，主要是初始化 rq 队列，由于实现用了 skew heap 而不是 list，对 run_list 的初始化其实没有必要

```c
list_init(&(rq->run_list));
rq->lab6_run_pool = NULL;
rq->proc_num = 0;
```

- `enqueue` 中，调用 skew heap insert 接口把该进程的 skew heap entry 加入其中，并让 rq 记录的进程数目 +1，设置时间片。
- `dequeue` 的实现类似

```c
rq->lab6_run_pool = skew_heap_insert(rq->lab6_run_pool, &(proc->lab6_run_pool), proc_stride_comp_f);
proc->rq = rq;
rq->proc_num ++;
if (proc->time_slice == 0 || proc->time_slice > rq->max_time_slice) {
    proc->time_slice = rq->max_time_slice;
}
```

- `pick_next` 中则取出在 rq->lab6_run_pool 中记录的堆顶并由此找到 proc。这一步不需要将其从堆中删除
- 需要注意这里要判断 rq 不为空。

```c
if (rq->lab6_run_pool == NULL) {
    return NULL;
}
struct proc_struct *p = le2proc(rq->lab6_run_pool, lab6_run_pool);
p->lab6_stride += BIG_STRIDE / p->lab6_priority;
return p;
```

- `proc_tick` 的实现比较简单。

### 3 总结

#### 3.1 与参考答案区别：

- `trap.c` 开始 sched_class_proc_tick(current); 直接替掉了 need_resched = `，即放在了 ticks % TICK_NUM == 0 判断内。使得时钟中断不能每次都起到作用，导致了 run-priority 的错误，对比参考答案后发现
- `proc.c` 中参考答案将优先级初始化为了 0，但因为算法要求优先级 >=1，不妨直接初始化为 1.
- stride 的实现部分参考了参考答案，尤其是异常情况的判断。

### OS Lab 7 Report

#### 计55  陈齐斌  2015011403

### 练习 0：填写已有实验

- 在 `trap.c` 中需要修改 `trap_dispatch` 函数，使用 `run_timer_list` 代替之前 lab 的代码。

### 练习 1：理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题

#### 1.1 请在实验报告中给出内核级信号量的设计描述，并说其大致执行流流程。

- 定义如下，包括其值以及睡在之上的进程的等待队列。

```c
typedef struct {
    int value;
    wait_queue_t wait_queue;
} semaphore_t;
```

- semaphore 最重要的操作为 P 操作函数 `down` 以及 V 函数 `up`。
- 下面说明这两个操作的大致执行流程。
- 执行 P 操作时，首先关闭中断，然后检查当前信号量的值，即 value 是否大于 0，若为真，则表明信号量可以被获得而无需等待，因此可以让 value--，并且打开中断直接返回该进程；反之，则需要将当前进程变为等待并加入到等待队列中，然后打开中断让调度器选择其他进程执行。在这种情况下，P 操作还有剩下的一半部分，在下次被 V 操作唤醒的时候执行，即在等待队列中删除该进程，并且可以往下执行。
- 执行 V 操作时，首先同样关中断，如果等待队列中没有进程在等待，则直接将信号量 +1，如果有，则唤醒等待队列中的进程（将其设置为就绪态），开中断后返回。

#### 1.2 请在实验报告中给出给用户态进程/线程提供信号量机制的设计方案，并比较说明给内核级提供信号量机制的异同。

- 用户态的进程/线程的信号量的设计应跟内核态类似，不同的是由于 P，V 操作涉及到开关中断以及改变进程状态、调度进程，因此需要由内核提供该服务，可以增加专门的系统调用如下。
  - sema_init：向内核申请一个信号量，并且由操作系统管理。
  - sema_down：P 操作。调用效果上由于其可能导致进程进入等待状态，可以与 sleep 调用进行对比。但其需要由操作系统保证原子性。下同。
  - sema_up：V 操作。

- 用户态与内核信号量的异同如下：
  - 同：表示的数据结构相同。
  - 异：用户态进程/线程信号量通过系统调用实现，内核信号量通过内核函数实现。

### 练习 2：完成内核级条件变量和基于内核级条件变量的哲学家就餐问题（需要编码）

#### 2.1 请在实验报告中给出内核级条件变量的设计描述，并说其大致执行流流程。

- 管程的定义如下
  - mutex 信号量，保证了管程的互斥访问
  - next 信号量是由于某进程发出 signal 唤醒另一个进程时，自己将转为睡眠状态，直到后者离开管程它才能继续执行，因此 next 量用作这里的同步。next_count 表示由于此原因睡眠的进程个数。
  - cv 即管程的条件变量。

```c
typedef struct monitor{
    semaphore_t mutex;      // the mutex lock for going into the routines in monitor, should be initialized to 1
    semaphore_t next;       // the next semaphore is used to down the signaling proc itself, and the other OR wakeuped waiting proc should wake up the sleeped signaling proc.
    int next_count;         // the number of of sleeped signaling proc
    condvar_t *cv;          // the condvars in monitor
} monitor_t;
```

- 其中 cond variable 的定义如下
- 因为 ucore 管程基于信号量实现，因此 sem 用作调用信号量功能的借口；owner 指向该条件变量所属的管程；count 记录等待在该 cv 上的进程数目。

```c
typedef struct condvar{
    semaphore_t sem;        // the sem semaphore  is used to down the waiting proc, and the signaling proc should up the waiting proc
    int count;              // the number of waiters on condvar
    monitor_t * owner;      // the owner(monitor) of this condvar
} condvar_t;
```

- 其大致执行流程如下。
  - 执行 cond_wait，使得等待该条件的进程能够离开管程并睡眠，且让其他进程进入管程继续执行。但需要判断是否有由于执行 signal 而睡眠的优先进程，如果有，则使 next up，此时由于没有释放 mutex，相当于把管程进入的互斥锁直接转移给了睡在 next 上的某个将要被唤醒的进程。
  - 而某进程执行 cond_signal 时，可以让等在该条件上的进程被唤醒，并进入管程执行。
- 具体实现代码如下：

```c
void cond_signal (condvar_t *cvp) {
    if (cvp->count > 0) {
        cvp->owner->next_count++;
        up(&(cvp->sem));
        down(&(cvp->owner->next));
        cvp->owner->next_count--;
    }
}

void cond_wait (condvar_t *cvp) {
    cvp->count++;
    if (cvp->owner->next_count > 0) {
        up(&(cvp->owner->next));
    } else {
        up(&(cvp->owner->mutex));
    }
    down(&(cvp->sem));
    cvp->count--;
}
```

- 利用实现的内核级 monitor 可以防止哲学家就餐的死锁问题
  - 如果哲学家无法取得叉子就餐，则调用 cond_wait 进入阻塞状态。
  - 当某哲学家进餐完毕之后，会相邻哲学家是否能否进餐，如果可以进餐，则解除对方的 wait

```c
void phi_take_forks_condvar(int i) {
...
//--------into routine in monitor--------------
    // LAB7 EXERCISE1: 2015011403
    // I am hungry
    // try to get fork
    state_condvar[i] = HUNGRY;
    phi_test_condvar(i);
    if (state_condvar[i] != EATING) {
        cprintf("phi_take_forks_condvar: %d didn't get fork and will wait\n",i);
        cond_wait(&mtp->cv[i]);
    }
//--------leave routine in monitor--------------
...
}

void phi_put_forks_condvar(int i) {
...
//--------into routine in monitor--------------
    // LAB7 EXERCISE1: 2015011403
    // I ate over
    // test left and right neighbors
    state_condvar[i] = THINKING;
    phi_test_condvar(LEFT);
    phi_test_condvar(RIGHT);
//--------leave routine in monitor--------------
...
}
```

#### 2.2 请在实验报告中给出给用户态进程/线程提供条件变量机制的设计方案，并比较说明给内核级提供条件变量机制的异同。

- 由于 ucore 里的 monitor 基于 semaphore 实现，因此在增加用户态的 semaphore 支持后（增加对应的系统调用），可以在用户层面实现用户态的条件变量，而由于都用到信号量接口而不直接设计底层代码，实现方法可以参考内核级的 monitor 的实现。

- 异：前者使用用户态信号量实现，后者使用内核级信号量实现。
- 同：表示的数据结构和接口相同。

### 3 总结

#### 3.1 与参考答案区别：

- `trap.c` 中一开始没有删除掉 lab6 的代码，后发现 run_timer_list 中已经包含 sched_class_proc_tick 的处理。
- `check_sync.c` 的实现中由于已经给了提示的伪代码，很大程度上辅助了设计的思考和实现。

#### 3.2 重要与缺失知识点

- 重要
  - 同步互斥
  - semaphore, monitor
  - 哲学家就餐问题
- 缺失
  - 利用原子指令实现同步互斥

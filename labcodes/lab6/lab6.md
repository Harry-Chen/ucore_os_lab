# uCore Lab6 实验报告

## 练习0：填写已有实验

需要进行的修改为：

* 在时钟中断到来时，调用 `sched_class_proc_tick` 函数而不设置 `need_resched` 属性（每次中断都需要调用，而不是每个 `tick`）
* 在 `alloc_proc` 函数中，使用 `list_init` 初始化进程控制块的 `run_link` 属性（作为链表节点）

## 练习1: 使用 Round Robin 调度算法

### 调度器实现中每个函数指针的用途

* `init`：用于初始化调度器内部的存储结构（如链表），并置为初始状态
* `enqueue`：将一个进程放入可调度序列中，并增加进程个数
* `dequeue`：将一个进程从可调度序列中移除，并减少当前进程个数
* `pick_next`：从可调度序列中寻找下一个被调度的进程并返回
* `proc_tick`：告知调度器当前进程已经运行了一个时间片，以做出相应处理

### Round Robin 算法的调度过程

每个进程在刚被放入可调度序列时（`enqueue`），剩余时间片都被设置成一个固定值，并将会被放在待调度队列的尾部，而 `pick_next` 总会返回队列的队首。每次 `proc_tick`，系统都会将当前进程的时间片减一，如果为0，则标记 `need_resched`，调度器被调用：首先将当前进程 `enqueue`，而后 `pick_next` 选出下一个可调度进程，将其 `dequeue`（如果没有，则选择 `idle` 进程），并标记为当前进程。最后，调用 `proc_run` 继续当前进程的工作。

### 多级反馈队列调度算法的设计

对于每一个可能的优先级，调度器都需要维护一个队列，并在每个进程控制块中增加一个属性记录优先级。队列的优先级越高，则其中进程对应的每次执行时间片越少。进程的初始优先级总是最高的。

* `init`：初始化各个队列
* `enqueue`：如果进程用完了时间片，则将进程优先级降低（如果可能）并重置时间片为对应值，而后总是插入进程到其对应优先级的队尾
* `dequeue`：从进程对应的优先级队列中移除该进程
* `pick_next`：返回当前有任务的最高优先级队列的首元素
* `proc_tick`：将当前进程时间片减一，如果归零，则设置 `need_resched` 通知调度器工作

## 练习2: 实现 Stride Scheduling 调度算法

### 代码实现

stride 调度本身的实验并不复杂，可以基于一个斜堆（skew heap），具体各个调度器函数的实现都详细地写在注释中，大致如下:

* `init`：初始化 `run_list` 和 `lab6_run_pool`
* `enqueue`：将进程插入堆中，重置进程的时间片（在已经用完的情况下），更新进程到堆的指针。注意此时要判断优先级是否为0，否则可能导致算法出错
* `dequeue`：从堆中移除该进程
* `pick_next`：将堆顶元素的 stride 加上 `BIG_STRIDE / PRIORITY`，并返回
* `proc_tick`：将当前进程时间片减一，如果归零，则设置 `need_resched` 通知调度器工作（同上面各个算法）

注意我没有直接用 stride 算法的实现覆盖 `default_sched.c`，而是将原本 RR 算法定义的 `default_sched_class` 设置为弱符号，这样，新的算法实现就能在链接时覆盖默认实现，而同时还能保留原有 RR 算法的代码供参考。

### `BIG_STRIDE` 的选择问题

首先有性质：$\text{STRIDE\_MAX} – \text{STRIDE\_MIN} \leq \text{PASS\_MAX}$

这个性质的证明是简单的，很容易知道初始状态总是满足这个性质，只需要保证每一次 `pick_next` 时对选中进程的 `stride` 增加不破坏这个性质即可。而每次挑选的进程恰好是 `stride` 最小的，设它的值增加了 $p \leq \text{PASS\_MAX}$，分两种情况：

* 如果 $\text{STRIDE\_MIN} + p \leq \text{STRIDE\_MAX}$，那么 `STRIDE_MIN` 增大，`STRIDE_MAX` 不变，它们的差不会增大
* 如果 $\text{STRIDE\_MIN} + p > \text{STRIDE\_MAX}$，那么 `STRIDE_MAX` 变为 $\text{STRIDE\_MIN} + p$，`STRIDE_MIN` 增大，但是它们的差依旧不会超过 `PASS_MAX`，否则将导致 $p > \text{PASS\_MAX}$ 的矛盾

这样就证明了给定的结论，那么自然有 $\text{STRIDE\_MAX} – \text{STRIDE\_MIN} \leq \text{BIG\_STRIDE}$。因此只需要让 `BIG_STRIDE` 能够被存储使用的 32 位有符号整数表示，则任意的 `stride` 之差都能表示它们真实的大小关系。由上可得 `BIG_STRIDE` 最大值是 `0x7fffffff`。

## 参考答案对比

参考答案直接用 `stride` 调度算法覆盖了原有算法，而我保留了 RR 算法。并且参考答案中根据一个 `USE_SKEW_HEAP` 的宏决定使用斜堆还是链表的实现，很明显链表的时间复杂度更高，因此我并没有考虑。其余的实现基本一致。

在向 Lab7 填写代码时，还修复了 `stride` 调度算法实现的一个 bug，即在没有等待进程时，`pick_next` 需要返回 `NULL`，否则将引发未定义行为导致内核崩溃。

## 总结

本实验中涉及到 OS 课程比较重要的知识点有：

* uCore 的调度算法框架和总体调度过程
* Round Robin 与 Stride 调度算法

还有没有涉及到的知识点有：

* 实时调度、多处理器调度
* 一些其他的调度算法（先到先得、公平共享等）

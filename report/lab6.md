# UCORE OS LAB6 调度器

计54 马子轩

## 实验目的

理解操作系统的调度管理机制

熟悉 ucore 的系统调度器框架，以及缺省的Round-Robin 调度算法

基于调度器框架实现一个(Stride Scheduling)调度算法来替换缺省的调度算法

## 实验内容

### 填写已有实验

使用meld工具合并项目，其中部分项目需要根据lab6进行相应修改。

kern/trap/trap.c

此部分是根据时钟中断执行调度，此时我才发现框架本身提供了ticks这个全局变量用来记录，就去掉了自己的count，此处实验框架出了问题sched_class_proc_tick()函数不能够直接进行调用。所以我针对这个问题对框架进行了相应的修改。并提交了[pull request](https://github.com/chyyuu/ucore_os_lab/pull/35)，之后将就这个pull request进行的修改和不修改的后果进行简单讲解。

```c
case IRQ_OFFSET + IRQ_TIMER:
	ticks++;
	assert(current != 0);
	sched_class_proc_tick(current);
	break;
```

kern/process/proc.c

其中添加了对调度的初始化，包括链表和优先级，已运行的时间片个数等。

```c
static struct proc_struct *
alloc_proc(void) {
	struct proc_struct *proc = kmalloc(sizeof(struct proc_struct));
	if (proc != NULL) {
		proc->state = PROC_UNINIT;
		proc->pid = -1;
		proc->runs = 0;
		proc->kstack = 0;
		proc->need_resched = 0;
		proc->parent = 0;
		proc->mm = 0;
		memset(&proc->context, 0, sizeof(struct context));
		proc->tf = 0;
		proc->cr3 = boot_cr3;
		proc->flags = 0;
		memset(proc->name, 0, PROC_NAME_LEN);
		proc->wait_state = 0;
		proc->cptr = proc->yptr = proc->optr = 0;
		proc->rq = 0;
		list_init(&proc->run_link);
		proc->time_slice = 0;
		proc->lab6_run_pool.left = 0;
		proc->lab6_run_pool.right = 0;
		proc->lab6_run_pool.parent = 0;
		proc->lab6_stride = 0;
		proc->lab6_priority = 0;
	}
	return proc;
}
```

### 使用 Round Robin 调度算法

1、请理解并分析sched_calss中各个函数指针的用法，并接合Round Robin 调度算法描ucore的调度执行过程

init初始化调度器，清空链表

```c
static void
RR_init(struct run_queue *rq) {
	list_init(&(rq->run_list));
	rq->proc_num = 0;
}
```

enqueue添加进程到链表末尾，初始化时间片长度。

```c
static void
RR_enqueue(struct run_queue *rq, struct proc_struct *proc) {
	assert(list_empty(&(proc->run_link)));
	list_add_before(&(rq->run_list), &(proc->run_link));
	if (proc->time_slice == 0 || proc->time_slice > rq->max_time_slice) {
		proc->time_slice = rq->max_time_slice;
	}
	proc->rq = rq;
	rq->proc_num ++;
}
```

dequeue将进程删除

```c
static void
RR_dequeue(struct run_queue *rq, struct proc_struct *proc) {
	assert(!list_empty(&(proc->run_link)) && proc->rq == rq);
	list_del_init(&(proc->run_link));
	rq->proc_num --;
}
```

pick_next返回最优先的进程

```c
static struct proc_struct *
RR_pick_next(struct run_queue *rq) {
	list_entry_t *le = list_next(&(rq->run_list));
	if (le != &(rq->run_list)) {
		return le2proc(le, run_link);
	}
	return NULL;
}
```

proc_tick根据时钟中断，调整剩余时间片，并控制进程调度状态。

```c
static void
RR_proc_tick(struct run_queue *rq, struct proc_struct *proc) {
	if (proc->time_slice > 0) {
		proc->time_slice --;
	}
	if (proc->time_slice == 0) {
		proc->need_resched = 1;
	}
}
```

实际使用中，每次proc_tick，init函数判断是否需要调度，schedule函数进行调用，函数将可以运行的加入队列，同时选取并执行。

2、请在实验报告中简要说明如何设计实现”多级反馈队列调度算法“，给出概要设计，鼓励给出详细设计

设定多个链表，分别用来处理不同优先级。

不同优先级设定不同时间片，优先级越高时间片越小。

同级队列中时间片轮转。

选择下一个进程从高优先级依次选择。

### 实现 Stride Scheduling 调度算法

1、可以参考上述RR算法进行对比。

proc_stride_comp_f是优先队列中用来进行比较的函数，就是根据优先度排序

init是初始化，和原来的一致。

enqueue是插入，插入优先队列。

dequeue是删除，同时从优先队列删除。

pick_next选取下一个进程，同时根据优先级填写时间片，这部分通过用一个大常数除以优先度。此处大常数我本来设置为一个阶乘，后来发现没有意义就改成了0x7fffffff。

proc_tick和之前的一致。

简单来说，Stride Scheduling就是使用了优先队列来代替链表，根据有限度选择调度进程，从而实现了进程调度。

```c
static int
proc_stride_comp_f(void *a, void *b)
{
	struct proc_struct *p = le2proc(a, lab6_run_pool);
	struct proc_struct *q = le2proc(b, lab6_run_pool);
	int32_t c = p->lab6_stride - q->lab6_stride;
	if (c > 0) return 1;
	else if (c == 0) return 0;
	else return -1;
}

static void
stride_init(struct run_queue *rq) {
	list_init(&rq->run_list);
	rq->lab6_run_pool = 0;
	rq->proc_num = 0;
}

static void
stride_enqueue(struct run_queue *rq, struct proc_struct *proc) {
	rq->lab6_run_pool = skew_heap_insert(rq->lab6_run_pool, &proc->lab6_run_pool, proc_stride_comp_f);
	if (proc->time_slice == 0 || proc->time_slice > rq->max_time_slice) {
		proc->time_slice = rq->max_time_slice;
	}
	proc->rq = rq;
	rq->proc_num ++;
}

static void
stride_dequeue(struct run_queue *rq, struct proc_struct *proc) {
	rq->lab6_run_pool = skew_heap_remove(rq->lab6_run_pool, &proc->lab6_run_pool, proc_stride_comp_f);
	rq->proc_num--;
}

static struct proc_struct *
stride_pick_next(struct run_queue *rq) {
	if (!rq->lab6_run_pool) {
		return 0;
	}
	struct proc_struct *proc = le2proc(rq->lab6_run_pool, lab6_run_pool);

	if (proc->lab6_priority == 0) {
		proc->lab6_stride += BIG_STRIDE;
	} else {
		proc->lab6_stride += BIG_STRIDE / proc->lab6_priority;
	}
	return proc;
}

static void
stride_proc_tick(struct run_queue *rq, struct proc_struct *proc) {
	if (proc->time_slice > 0) {
		proc->time_slice --;
	}
	if (proc->time_slice == 0) {
		proc->need_resched = 1;
	}
}
```

### 实验框架更改

kern/schedule/sched.c

去掉static使该方法可以应用于其他文件

```c
void
sched_class_proc_tick(struct proc_struct *proc) {
	if (proc != idleproc) {
		sched_class->proc_tick(rq, proc);
```
kern/schedule/sched.h

添加函数头

```c
void sched_class_proc_tick(struct proc_struct *proc);
```

在以上修改的同时，修改了答案
kern/trap/trap.c

此修改如果不进行，时钟中断时候不会进行调度。退化成了简单的轮转。
具体讨论见[pull request](https://github.com/chyyuu/ucore_os_lab/pull/35/)

```c
ticks ++;
assert(current != NULL);
sched_class_proc_tick(current);
break;
```

## 实验结果

由于我的实验环境问题，badsegment没有qemu输出，应该换正常的实验环境即可。

```shell
john:lab6/ (master) $ make grade                                     [15:49:56]
badsegment:              (4.4s)
  -check result:                             no $qemu_out
divzero:                 (1.6s)
  -check result:                             OK
  -check output:                             OK
softint:                 (1.7s)
  -check result:                             OK
  -check output:                             OK
faultread:               (1.8s)
  -check result:                             OK
  -check output:                             OK
faultreadkernel:         (2.4s)
  -check result:                             OK
  -check output:                             OK
hello:                   (2.2s)
  -check result:                             OK
  -check output:                             OK
testbss:                 (1.8s)
  -check result:                             OK
  -check output:                             OK
pgdir:                   (1.7s)
  -check result:                             OK
  -check output:                             OK
yield:                   (1.7s)
  -check result:                             OK
  -check output:                             OK
badarg:                  (1.7s)
  -check result:                             OK
  -check output:                             OK
exit:                    (1.7s)
  -check result:                             OK
  -check output:                             OK
spin:                    (2.0s)
  -check result:                             OK
  -check output:                             OK
waitkill:                (2.3s)
  -check result:                             OK
  -check output:                             OK
forktest:                (1.7s)
  -check result:                             OK
  -check output:                             OK
forktree:                (1.7s)
  -check result:                             OK
  -check output:                             OK
matrix:                  (16.7s)
  -check result:                             OK
  -check output:                             OK
priority:                (11.7s)
  -check result:                             OK
  -check output:                             OK
Total Score: 160/165
Makefile:314: recipe for target 'grade' failed
make: *** [grade] Error 1
```

## 实验总结

经过lab4 lab5 lab6，完成了基本的线程管理，和程序运行调度。内核的基本功能完成。这一部分还需要之前的内存管理作为基础，才能够顺利进行。同时，也是后续两个实验的基础。
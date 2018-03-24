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

1、



## 实验结果

## 实验总结

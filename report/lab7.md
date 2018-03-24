# UCORE LAB7 同步互斥

计54 马子轩

## 实验目的

理解操作系统的同步互斥的设计实现；

理解底层支撑技术：禁用中断、定时器、等待队列；

在ucore中理解信号量（semaphore）机制的具体实现；

理解管程机制，在ucore内核中增加基于管程（monitor）的条件变量（condition variable）的支持；

了解经典进程同步问题，并能使用同步机制解决进程同步问题。

## 实验内容

### 填写已有实验

使用meld工具直接合并即可

### 理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题

1、请在实验报告中给出内核级信号量的设计描述，并说其大致执行流流程。

信号量的定义，包含value和wait_queue

```c
typedef struct {
	int value;
	wait_queue_t wait_queue;
} semaphore_t;
```

up操作，先关中断，判断wait_queue是否有进程等待，如果没有那么value++，如果有就唤醒第一个进程，并在队列中删除。再开中断。

```c
static __noinline void __up(semaphore_t *sem, uint32_t wait_state) {
	bool intr_flag;
	local_intr_save(intr_flag);
	{
		wait_t *wait;
		if ((wait = wait_queue_first(&(sem->wait_queue))) == NULL) {
			sem->value ++;
		}
		else {
			assert(wait->proc->wait_state == wait_state);
			wakeup_wait(&(sem->wait_queue), wait, wait_state, 1);
		}
	}
	local_intr_restore(intr_flag);
}
```

down，先关中断，判断value大小，如果>0那么开中断，-1返回即可。如果不>0则需要加入进程队列，再打开中断，调度其他进程。

```c
static __noinline uint32_t __down(semaphore_t *sem, uint32_t wait_state) {
	bool intr_flag;
	local_intr_save(intr_flag);
	if (sem->value > 0) {
		sem->value --;
		local_intr_restore(intr_flag);
		return 0;
	}
	wait_t __wait, *wait = &__wait;
	wait_current_set(&(sem->wait_queue), wait, wait_state);
	local_intr_restore(intr_flag);

	schedule();

	local_intr_save(intr_flag);
	wait_current_del(&(sem->wait_queue), wait);
	local_intr_restore(intr_flag);

	if (wait->wakeup_flags != wait_state) {
		return wait->wakeup_flags;
	}
	return 0;
}
```

总的来说是通过开关中断来使共享资源同步互斥。

2、请在实验报告中给出给用户态进程/线程提供信号量机制的设计方案，并比较说明给内核级提供信号量机制的异同。

我认为用户态信号量机制算法应该和内核态使一致的。而在用户态调用信号量时需要进行系统调用进入内核态。

### 完成内核级条件变量和基于内核级条件变量的哲学家就餐问题

1、判断是否有睡眠进程，有的话唤醒，把自己睡了。

```c
void 
cond_signal (condvar_t *cvp) {
	cprintf("cond_signal begin: cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);  
	if (cvp->count > 0) {
		cvp->owner->next_count++;
		up(&(cvp->sem));
		down(&(cvp->owner->next));
		cvp->owner->next_count--;
	}
	cprintf("cond_signal end: cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
}
```

2、唤醒下一个进程，或是唤醒正在阻塞的进程，把自己睡了。

```c
void
cond_wait (condvar_t *cvp) {
	cprintf("cond_wait begin:  cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
	cvp->count++;
	if (cvp->owner->next_count > 0) {
		up(&(cvp->owner->next));
	} else {
		up(&(cvp->owner->mutex));
	}
	down(&(cvp->sem));
	cvp->count--;
	cprintf("cond_wait end:  cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
}
```

3、哲学家就餐问题按照定义填写即可。

```c
void phi_take_forks_condvar(int i) {
	down(&(mtp->mutex));
	state_condvar[i] = HUNGRY;
	phi_test_condvar(i);
	while (state_condvar[i] != EATING) {
		cond_wait(&(mtp->cv[i]));
	}
	if(mtp->next_count>0)
		up(&(mtp->next));
	else
		up(&(mtp->mutex));
}
```

```c
void phi_put_forks_condvar(int i) {
	down(&(mtp->mutex));
	state_condvar[i] = THINKING;
	phi_test_condvar(LEFT);
	phi_test_condvar(RIGHT);
	if(mtp->next_count>0)
		up(&(mtp->next));
	else
		up(&(mtp->mutex));
}
```

4、请在实验报告中给出内核级条件变量的设计描述，并说其大致执行流流程。

条件变量定义如下，调用方法有两个，再上述报告中有所描述。其中signal方法把自己加入队列，而wait先唤醒signal再减少信号量。

```c
typedef struct condvar{
	semaphore_t sem;
	int count;
	monitor_t * owner;
} condvar_t;
```

5、请在实验报告中给出给用户态进程/线程提供条件变量机制的设计方案，并比较说明给内核级提供条件变量机制的异同。

条件变量机制我认为用户态和内核态一致，也应当通过系统调用实现。

6、请在实验报告中回答：能否不用基于信号量机制来完成条件变量？如果不能，请给出理由，如果能，请给出设计说明和具体实现。

可以，我认为可以使用基于原子操作的方式来完成条件变量。

## 实验结果

make grade通过

make qemu-nox结果正常

由于长度过长，不在报告中展示

## 实验总结

这个实验解决了我一直以来对同步互斥问题的疑问，而基于信号量的设计也令我眼前一亮。此部分为止，进程管理基本已经完成了。ucore中的核心部分也基本完成了。


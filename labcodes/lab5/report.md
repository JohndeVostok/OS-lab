# UCORE LAB5 用户进程管理

计54 马子轩

## 实验目的

了解第一个用户进程创建过程

了解系统调用框架的实现机制

了解ucore如何实现系统调用sys_fork/sys_exec/sys_exit/sys_wait来进行进程管理

## 实验内容

### 填写已有实验

使用meld工具进行合并，其中部分代码需要根据lab5需求进行修改

kern/trap/trap.c

trap_dispatch()函数中添加需要调度的符号

```c
case IRQ_OFFSET + IRQ_TIMER:
	count++;
	if (count == TICK_NUM) {
		count = 0;
		assert(current != 0);
		current->need_resched = 1;
	}
	break;
```

kern/process/proc.c

alloc_proc()函数中添加等待状态和进程指针的初始化

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
	}
	return proc;
}
```

do_fork()函数中，添加对等待状态的特判，将进程技术改为set_links()函数，方便进程调度。

```c
int
do_fork(uint32_t clone_flags, uintptr_t stack, struct trapframe *tf) {
	int ret = -E_NO_FREE_PROC;
	struct proc_struct *proc;
	if (nr_process >= MAX_PROCESS) {
		goto fork_out;
	}
	ret = -E_NO_MEM;

	proc = alloc_proc();

	if (!proc) {
		goto fork_out;
	}
	proc->parent = current;

	assert(!current->wait_state);

	if (setup_kstack(proc)) {
		goto bad_fork_cleanup_proc;
	}

	if (copy_mm(clone_flags, proc)) {
		goto bad_fork_cleanup_kstack;
	}

	copy_thread(proc, stack, tf);

	bool flag;
	local_intr_save(flag);
	proc->pid = get_pid();
	hash_proc(proc);
	set_links(proc);
	local_intr_restore(flag);

	wakeup_proc(proc);
	ret = proc->pid;


fork_out:
	return ret;

bad_fork_cleanup_kstack:
	put_kstack(proc);
bad_fork_cleanup_proc:
	kfree(proc);
	goto fork_out;
}
```

### 加载应用程序并执行

1、在proc.c种的load_icode()函数中添加trapframe中的段选择子CS，DS，ES。程序入口，eflags，使其支持中断。

```c
tf->tf_cs = USER_CS;
tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
tf->tf_esp = USTACKTOP;
tf->tf_eip = elf->e_entry;
tf->tf_eflags = FL_IF;
```

2、请在实验报告中描述当创建一个用户态进程并加载了应用程序后，CPU是如何让这个应用程序最终在用户态执行起来的。即这个用户态进程被ucore选择占用CPU执行（RUNNING态）到具体执行应用程序第一条指令的整个经过。

mm_create()，创建内存空间，初始化

setup_pgdir()，创建页目录所需空间，拷贝内核空间页表。并指定pgdir

mm_map()，建立vma，插入到mm中

根据各个段大小分配物理空间，确定虚拟地址，建立映射关系，拷贝内容。

设置用户栈，调用mm_mmap函数建立vma

赋值mm->pgdir到cr3寄存器

清空进程中断帧，重新设置，使iret后能正常返回

### 父进程复制自己的内存空间给子进程

1、将src进程的内容直接拷贝到dst进程。并插入页面

```c
void *src_kvaddr = page2kva(page);
void *dst_kvaddr = page2kva(npage);

memcpy(dst_kvaddr, src_kvaddr, PGSIZE);
ret = page_insert(to, npage, start, perm);
```

2、请在实验报告中简要说明如何设计实现”Copy on Write 机制“，给出概要设计，鼓励给出详细设计。

考虑copy on write中需要共享某些页面，那么在pmm.c中copy_range()函数中应当禁止写入并插入页表。此时，vma中的写权限才是真正的写权限，而尝试写会触发pg_fault。copy在do_pgfault()完成，判断页面是否是未被拷贝的页面，如果是，申请一个，拷贝原来的，插入。

### 阅读分析源代码，理解进程执行 fork/exec/wait/exit 的实现，以及系统调用的实现

1、请分析fork/exec/wait/exit在实现中是如何影响进程的执行状态的？

fork调用do_fork()完成进程创建和资源分配

exec调用do_execve()，切换到内核态，加载新程序，分配资源切换进程

wait调用do_wait()，遍历进程，找到PROC_ZOMBIE，清理资源

exit调用do_exit(),切换到内核态，设置状态，通过父进程来清理资源

2、请给出ucore中一个用户态进程的执行状态生命周期图（包执行状态，执行状态之间的变换关系，以及产生变换的事件或函数调用）。

alloc_proc() --> proc_init()/wakeup_proc() --> proc_run() --> do_wait()/do_sleep() --> do_exit()

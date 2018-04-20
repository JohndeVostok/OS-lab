# UCORE LAB8 文件系统

计54 马子轩

## 实验目的

了解基本的文件系统系统调用的实现方法；
了解一个基于索引节点组织方式的Simple FS文件系统的设计与实现；
了解文件系统抽象层-VFS的设计与实现；

## 实验过程

### 填写已有实验

使用meld工具合并项目

其中kern/process/proc.c进行修改

添加filesp的初始化

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
		proc->filesp = 0;
	}
	return proc;
}
```

### 完成读文件操作的实现

1、调用sfs_bmap_load_nolock()函数，得到硬盘上的块号，使用函数指针sfs_buf_op()，用sfs_wbuf写，用sfs_rbuf 读。

```c
static int
sfs_io_nolock(struct sfs_fs *sfs, struct sfs_inode *sin, void *buf, off_t offset, size_t *alenp, bool write) {
	struct sfs_disk_inode *din = sin->din;
	assert(din->type != SFS_TYPE_DIR);
	off_t endpos = offset + *alenp, blkoff;
	*alenp = 0;
	if (offset < 0 || offset >= SFS_MAX_FILE_SIZE || offset > endpos) {
		return -E_INVAL;
	}
	if (offset == endpos) {
		return 0;
	}
	if (endpos > SFS_MAX_FILE_SIZE) {
		endpos = SFS_MAX_FILE_SIZE;
	}
	if (!write) {
		if (offset >= din->size) {
			return 0;
		}
		if (endpos > din->size) {
			endpos = din->size;
		}
	}

	int (*sfs_buf_op)(struct sfs_fs *sfs, void *buf, size_t len, uint32_t blkno, off_t offset);
	int (*sfs_block_op)(struct sfs_fs *sfs, void *buf, uint32_t blkno, uint32_t nblks);
	if (write) {
		sfs_buf_op = sfs_wbuf, sfs_block_op = sfs_wblock;
	}
	else {
		sfs_buf_op = sfs_rbuf, sfs_block_op = sfs_rblock;
	}

	int ret = 0;
	size_t size, alen = 0;
	uint32_t ino;
	uint32_t blkno = offset / SFS_BLKSIZE;		  // The NO. of Rd/Wr begin block
	uint32_t nblks = endpos / SFS_BLKSIZE - blkno;  // The size of Rd/Wr blocks

	blkoff = offset % SFS_BLKSIZE;
	if (blkoff) {
		size = (nblks != 0) ? (SFS_BLKSIZE - blkoff) : (endpos - offset);
		ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino);
		if (ret) {
			goto out;
		}

		ret = sfs_buf_op(sfs, buf, size, ino, blkoff);
		if (ret) {
			goto out;
		}

		alen += size;
		if (!nblks) {
			goto out;
		}
		buf += size;
		blkno++;
		nblks--;
	}
	
	size = SFS_BLKSIZE;
	while (nblks) {
		ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino);
		if (ret) {
			goto out;
		}

		ret = sfs_block_op(sfs, buf, ino, 1);
		if (ret) {
			goto out;
		}

		alen += size;
		blkno++;
		nblks--;
	}

	size = endpos % SFS_BLKSIZE;
	if (size) {
		ret = sfs_bmap_load_nolock(sfs, sin, blkno, &ino);
		if (ret) {
			goto out;
		}

		ret = sfs_buf_op(sfs, buf, size, ino, 0);
		if (ret) {
			goto out;
		}
		alen += size;
	}
out:
	*alenp = alen;
	if (offset + alen > sin->din->size) {
		sin->din->size = offset + alen;
		sin->dirty = 1;
	}
	return ret;
}
```

2、请在实验报告中给出设计实现”UNIX的PIPE机制“的概要设方案，鼓励给出详细设计方案

PIPE是相关进程间交流的全双工通信机制，实现上就是个buffer，一个进程读，一个进程写即可。只需要对buffer实现对应函数功能即可。

### 完成基于文件系统的执行程序机制的实现

1、对load_icode的修改主要是elf读入改为从文件读入，建立用户堆栈，处理参数，设置中断帧。

```c
static int
load_icode(int fd, int argc, char **kargv) {
	assert(argc >= 0 && argc <= EXEC_MAX_ARG_NUM);
	if (current->mm != NULL) {
		panic("load_icode: current->mm must be empty.\n");
	}

	int ret;
	struct mm_struct *mm;
	struct Page *page;
	struct elfhdr __elf, *elf = &__elf;
	struct proghdr __ph, *ph = &__ph;

	ret = -E_NO_MEM;
	mm = mm_create();
	if (!mm) {
		goto bad_mm;
	}
	if (setup_pgdir(mm)) {
		goto bad_pgdir_cleanup_mm;
	}

	ret = load_icode_read(fd, elf, sizeof(struct elfhdr), 0);
	if (ret) {
		goto bad_elf_cleanup_pgdir;
	}

	if (elf->e_magic != ELF_MAGIC) {
		ret = -E_INVAL_ELF;
		goto bad_elf_cleanup_pgdir;
	}

	uint32_t vm_flags, perm;
	int i;
	off_t phoff, offset;
	size_t off, size;
	for (i = 0; i < elf->e_phnum; i++) {
		phoff = elf->e_phoff + i * (sizeof(struct proghdr));
		ret = load_icode_read(fd, ph, sizeof(struct proghdr), phoff);
		if (ret) {
			goto bad_cleanup_mmap;
		}
		if (ph->p_type != ELF_PT_LOAD) {
			continue;
		}
		if (ph->p_filesz > ph->p_memsz) {
			ret = -E_INVAL_ELF;
			goto bad_cleanup_mmap;
		}
		if (!ph->p_filesz) {
			continue;
		}

		vm_flags = 0;
		perm = PTE_U;
		if (ph->p_flags & ELF_PF_X) {
			vm_flags |= VM_EXEC;
		}
		if (ph->p_flags & ELF_PF_W) {
			vm_flags |= VM_WRITE;
		}
		if (ph->p_flags & ELF_PF_R) {
			vm_flags |= VM_READ;
		}
		if (vm_flags & VM_WRITE) {
			perm |= PTE_W;
		}
		ret = mm_map(mm, ph->p_va, ph->p_memsz, vm_flags, 0);
		if (ret) {
			goto bad_cleanup_mmap;
		}
		offset = ph->p_offset;
		uintptr_t start, end, la;
		start = ph->p_va;
		end = ph->p_va + ph->p_filesz;
		la = ROUNDDOWN(start, PGSIZE);
		while (start < end) {
			page = pgdir_alloc_page(mm->pgdir, la, perm);
			if (!page) {
				ret = -E_NO_MEM;
				goto bad_cleanup_mmap;
			}
			off = start - la;
			size = PGSIZE - off;
			la += PGSIZE;
			if (end < la) {
				size -= la - end;
			}
			ret = load_icode_read(fd, page2kva(page) + off, size, offset);
			if (ret) {
				goto bad_cleanup_mmap;
			}
			start += size;
			offset += size;
		}
		end = ph->p_va + ph->p_memsz;
		if (start < la) {
			if (start == end) {
				continue;
			}
			off = start + PGSIZE - la;
			size = PGSIZE - off;
			if (end < la) {
				size -= la - end;
			}
			memset(page2kva(page) + off, 0, size);
			start += size;
			assert((end < la && start == end) || (end >= la && start == la));
		}
		while (start < end) {
			page = pgdir_alloc_page(mm->pgdir, la, perm);
			if (!page) {
				ret = -E_NO_MEM;
				goto bad_cleanup_mmap;
			}
			off = start - la;
			size = PGSIZE - off;
			la += PGSIZE;
			if (end < la) {
				size -= la - end;
			}
			memset(page2kva(page) + off, 0, size);
			start += size;
		}
	}
	sysfile_close(fd);
	
	vm_flags = VM_READ | VM_WRITE | VM_STACK;
	ret = mm_map(mm, USTACKTOP - USTACKSIZE, USTACKSIZE, vm_flags, 0);
	if (ret) {
		goto bad_cleanup_mmap;
	}
	assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-PGSIZE , PTE_USER) != NULL);
	assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-2*PGSIZE , PTE_USER) != NULL);
	assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-3*PGSIZE , PTE_USER) != NULL);
	assert(pgdir_alloc_page(mm->pgdir, USTACKTOP-4*PGSIZE , PTE_USER) != NULL);
	mm_count_inc(mm);
	current->mm = mm;
	current->cr3 = PADDR(mm->pgdir);
	lcr3(PADDR(mm->pgdir));

	uint32_t argv_size;
	argv_size = 0;
	for (i = 0; i < argc; i++) {
		argv_size += strnlen(kargv[i], EXEC_MAX_ARG_LEN + 1) + 1;
	}

	uintptr_t stacktop = USTACKTOP - (argv_size / sizeof(long) + 1) * sizeof(long);
	char **uargv = (char**)(stacktop - argc * sizeof(char *));
	argv_size = 0;
	for (i = 0; i < argc; i++) {
		uargv[i] = strcpy((char*)(stacktop + argv_size), kargv[i]);
		argv_size += strnlen(kargv[i], EXEC_MAX_ARG_LEN + 1) + 1;
	}

	stacktop = (uintptr_t) uargv - sizeof(int);
	*(int *) stacktop = argc;
	

	struct trapframe *tf = current->tf;
	memset(tf, 0, sizeof(struct trapframe));
	tf->tf_cs = USER_CS;
	tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
	tf->tf_esp = stacktop;
	tf->tf_eip = elf->e_entry;
	tf->tf_eflags = FL_IF;

	ret = 0;
out:
	return ret;
bad_cleanup_mmap:
	exit_mmap(mm);
bad_elf_cleanup_pgdir:
	put_pgdir(mm);
bad_pgdir_cleanup_mm:
	mm_destroy(mm);
bad_mm:
	goto out;
}
```

2、请在实验报告中给出设计实现基于”UNIX的硬链接和软链接机制“的概要设方案，鼓励给出详细设计方案

硬链接两个文件仅文件名不同，本质是相同的文件，创建inode指向相同data block即可。

软链接有自己的inode号和data block，所以创建时候指向绝对路径即可。

## 实验结果

执行hello

```c
Hello world!!.
I am process 14.
hello pass.
```

执行ls

```shell
 @ is  [directory] 2(hlinks) 23(blocks) 5888(bytes) : @'.'
   [d]   2(h)       23(b)     5888(s)   .
   [d]   2(h)       23(b)     5888(s)   ..
   [-]   1(h)       10(b)    40200(s)   softint
   [-]   1(h)       10(b)    40192(s)   pgdir
   [-]   1(h)       10(b)    40208(s)   faultreadkernel
   [-]   1(h)       10(b)    40204(s)   faultread
   [-]   1(h)       10(b)    40292(s)   priority
   [-]   1(h)       10(b)    40200(s)   badarg
   [-]   1(h)       10(b)    40200(s)   yield
   [-]   1(h)       10(b)    40224(s)   testbss
   [-]   1(h)       10(b)    40360(s)   ls
   [-]   1(h)       10(b)    40204(s)   sleepkill
   [-]   1(h)       11(b)    44508(s)   sh
   [-]   1(h)       10(b)    40200(s)   hello
   [-]   1(h)       10(b)    40228(s)   forktest
   [-]   1(h)       10(b)    40332(s)   waitkill
   [-]   1(h)       10(b)    40220(s)   divzero
   [-]   1(h)       10(b)    40196(s)   spin
   [-]   1(h)       10(b)    40204(s)   badsegment
   [-]   1(h)       10(b)    40224(s)   exit
   [-]   1(h)       10(b)    40304(s)   matrix
   [-]   1(h)       10(b)    40252(s)   forktree
   [-]   1(h)       10(b)    40220(s)   sleep
lsdir: step 4
```

实验与预期结果一致

## 实验总结

经过紧张的一个月学习，终于完成了完整的ucore lab，其中有很多机遇与挑战，在解决问题的过程中通过和同学的交流，也得到了很大的收获。整体看下来，我和参考答案的主要区别还是在各种边界条件的特判上，自己的考虑总不够周全。但是整个实验做下来，自己还是很有收获感的。希望后续课程设计能够继续保持这种好奇心。
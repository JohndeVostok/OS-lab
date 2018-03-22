# UCORE LAB3 虚拟内存管理

计54 马子轩

## 实验目的

了解虚拟内存的Page Fault异常处理实现

了解页替换算法在操作系统中的实现

## 实验内容

### 填写已有实验

使用meld工具，修改文件

kern/trap/trap.c

kern/debug/kdebug.c

kern/mm/default_pmm.c

kern/mm/pmm.c

make 编译通过

### 给未映射的地址映射上物理地址页

1、修改vmm.c中的do_pgfault()函数，先判断PTE是否存在，不存在的话申请。如果页面已经被替换出去了，切回来并设置flags。

```c
ptep = get_pte(mm->pgdir, addr, 1);
struct Page *page;

if (!ptep) {
	cprintf("GET PTE FAILED.\n");
	return ret;
}
if (!*ptep) {
	page = pgdir_alloc_page(mm->pgdir, addr, perm);
	if (!page) {
		cprintf("ALLOC PAGE FAILED.\n");
		return ret;
	}
} else {
	if (swap_init_ok) {
		ret = swap_in(mm, addr, &page);
		if (ret) {
			cprintf("SWAP IN FAILED.\n");
			return ret;
		}
		page_insert(mm->pgdir, page, addr, perm);
		swap_map_swappable(mm, addr, page, 1);
		page->pra_vaddr = addr;
	} else {
		cprintf("SWAP INIT FAILED. PTEP: %x\n", *ptep);
		return ret;
	}
}
```

2、请描述页目录项（Pag Director Entry）和页表（Page Table Entry）中组成部分对ucore实现页替换算法的潜在用处。

PTE/PDE的高20位为下一级页表的基址，低12位为标志位。定义如下

```c
/* page table/directory entry flags */
#define PTE_P           0x001                   // Present
#define PTE_W           0x002                   // Writeable
#define PTE_U           0x004                   // User
#define PTE_PWT         0x008                   // Write-Through
#define PTE_PCD         0x010                   // Cache-Disable
#define PTE_A           0x020                   // Accessed
#define PTE_D           0x040                   // Dirty
#define PTE_PS          0x080                   // Page Size
#define PTE_MBZ         0x180                   // Bits must be zero
#define PTE_AVAIL       0xE00                   // Available for software use
                                                // The PTE_AVAIL bits aren't used by the kernel or interpreted by the
                                                // hardware, so user processes are allowed to set them arbitrarily.

#define PTE_USER        (PTE_U | PTE_W | PTE_P)
```

3、如果ucore的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

按照中断处理流程处理中断，执行中断处理例程，此时

### 补充完成基于FIFO的页面替换算法

## 实验总结


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

按照中断处理流程处理中断，执行中断处理例程，此时会触发新的缺页异常，由于优先级原因，该中断无法处理。在qemu中表现为qemu崩溃。

### 补充完成基于FIFO的页面替换算法

1、页换入，将entry加入链表末尾即可。

```c
/*
 * (3)_fifo_map_swappable: According FIFO PRA, we should link the most recent arrival page at the back of pra_list_head qeueue
 */
static int
_fifo_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in)
{
    list_entry_t *head=(list_entry_t*) mm->sm_priv;
    list_entry_t *entry=&(page->pra_page_link);
 
    assert(entry != NULL && head != NULL);
    //record the page access situlation
    /*LAB3 EXERCISE 2: YOUR CODE*/ 
    //(1)link the most recent arrival page at the back of the pra_list_head qeueue.
    list_add(head, entry);
	return 0;
}
```

2、页换出，找到链表的头，删除链表中该项，并指定为换出。

```c
/*
 *  (4)_fifo_swap_out_victim: According FIFO PRA, we should unlink the  earliest arrival page in front of pra_list_head qeueue,
 *                            then set the addr of addr of this page to ptr_page.
 */
static int
_fifo_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick)
{
     list_entry_t *head=(list_entry_t*) mm->sm_priv;
         assert(head != NULL);
     assert(in_tick==0);
     /* Select the victim */
     /*LAB3 EXERCISE 2: YOUR CODE*/ 
     //(1)  unlink the  earliest arrival page in front of pra_list_head qeueue
     //(2)  set the addr of addr of this page to ptr_page
	list_entry_t *le;
	le = head->prev;
	struct Page *page = le2page(le, pra_page_link);
	list_del(le);
	*ptr_page = page;
	return 0;
}
```

3、如果要在ucore上实现"extended clock页替换算法"请给你的设计方案，现有的swap_manager框架是否足以支持在ucore中实现此算法？如果是，请给你的设计方案。如果不是，请给出你的新的扩展和基此扩展的设计方案。并需要回答如下问题：1、需要被换出的页的特征是什么？2、在ucore中如何判断具有这样特征的页？3、何时进行换入和换出操作？

可以支持，此时需要修改_fifo_map_swappable(), _fifo_swap_out_victim()两个函数。需要被换出的页面特征为(PTE_A|PTE_D) == 0，在ucore中判断!(&(PTE_A|PTE_D))即可。换入换出操作发生于，缺页异常和换入时物理页面已满。

## 实验总结

lab2和lab3两个实验基本实现了完整的虚拟地址物理地址映射过程，为后续进程管理打下基础。
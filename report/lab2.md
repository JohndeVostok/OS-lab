# UCORE LAB2 物理内存管理

计54 马子轩

## 实验目的

理解基于段页式内存地址的转换机制

理解页表的建立和使用方法

理解物理内存的管理方法

## 实验内容

### 填写已有实验

使用meld工具，修改文件

kern/trap/trap.c

kern/debug/kdebug.c

make 编译通过

### 实现first-fit连续物理内存分配算法

1、全局初始化

```c
static void
default_init(void) {
	list_init(&free_list);
	nr_free = 0;
}
```

2、初始化memmap

从base开始，初始化n个page。设置每个页面的可用性，长度，引用，并添加进链表。

```c
static void
default_init_memmap(struct Page *base, size_t n) {
	assert(n > 0);
	struct Page *page;
	for (page = base; page != base + n; page++)
	{
		assert(PageReserved(page));
		page->flags = page->property = 0;
		ClearPageReserved(page);
		SetPageProperty(page);
		set_page_ref(page, 0);
		list_add_before(&free_list, &(page->page_link));
	}
	base->property = n;
	nr_free += n;
}
```

3、分配空间

先从全局的剩余页面数判断能不能分配。

如果可以，遍历整个链表，如果发现连续长度可以分配出n的块，对这段设置长度和可用性，如果有剩余，重置长度，重新添加进链表。最终修改全局剩余页面。

```c
static struct Page *
default_alloc_pages(size_t n) {
	assert(n > 0);
	if (n > nr_free) {
		return NULL;
	}
	list_entry_t *le, *le_next;

	for (le = list_next(&free_list); le != &free_list; le = list_next(le)) {
		struct Page *page_base = le2page(le, page_link);
		if (page_base->property >= n) {
			int i;
			for (i = 0; i < n; i++) {
				le_next = list_next(le);
				struct Page *page = le2page(le, page_link);
				ClearPageProperty(page);
				SetPageReserved(page);
				list_del(le);
				le = le_next;
			}
			if (page_base->property > n) {
				le2page(le, page_link)->property = page_base->property - n;
			}
			nr_free -= n;
			return page_base;
		}
	}
	return NULL;
}
```

4、释放空间

先找到释放地址在链表中的位置，并确认前驱和后继。将释放段加入链表，包括设置长度和可用性。之后考虑和前驱后继合并，如果块是连续的，就合并。

```c
static void
default_free_pages(struct Page *base, size_t n) {
	assert(n > 0);
	assert(PageReserved(base));

	list_entry_t *le;
	struct Page *page_next;

	for (le = list_next(&free_list); le != &free_list; le = list_next(le)) {
		page_next = le2page(le, page_link);
		if (page_next > base) break;
	}


	struct Page *page;
	for (page = base; page < base + n; page++) {
		page->flags = page->property = 0;
		ClearPageReserved(page);
		SetPageProperty(page);
		set_page_ref(page, 0);
		list_add_before(le, &(page->page_link));
	}
	base->property = n;
	nr_free += n;

	if (base + n == page_next) {
		base->property += page_next->property;
		page_next->property = 0;
	}

	struct Page *page_prev;
	le = list_prev(&(base->page_link));
	page_prev = le2page(le, page_link);
	if (le != &free_list && page_prev + 1 == base) {
		while (!page_prev->property)
		{
			le = list_prev(le);
			page_prev = le2page(le, page_link);
		}
		page_prev->property += base->property;
		base->property = 0;
	}
}
```

5、first-fit算法的改进空间

使用链表来存储，会有大量遍历操作，在坏情况会退化使得整体复杂度过大，可以使用平衡树，来实现插入，删除，合并等操作。

### 实现寻找虚拟地址对应的页表项

1、先找到一级页表地址，如果存在flag为0，重新分配物理页。申请空间并设置引用和flags。找到相应的页表项，并返回相应的物理地址。

```c
pte_t *
get_pte(pde_t *pgdir, uintptr_t la, bool create) {
	pde_t *pdep = pgdir + PDX(la);
	if (!(*pdep & PTE_P)) {
		struct Page *page = alloc_page();
		if (!create || !page) return 0;
		set_page_ref(page, 1);
		uintptr_t pa = page2pa(page);
		memset(KADDR(pa), 0, PGSIZE);
		*pdep = pa | PTE_U | PTE_W | PTE_P;
	}
	pte_t *ptep = KADDR(PDE_ADDR(*pdep));
	return ptep + PTX(la);
}
```

2、请描述页目录项pde和页表pte中每个组成部分的含义和以及对ucore而言的潜在用处。

PDE与PTE和各自对应的table构成了ucore的二级页表。

两者结构都是20位的基址和12位的标志位。基址表示下一级结构对应的物理地址基地址，配合虚拟地址的偏移值，能够找到相应的物理地址。而标志位中PTE_P、PTE_W、PTE_U三项分别是页表是否存在，是否可写，是否能被用户态访问。而PTE中没有PTE_U项。

ucore利用这个二级页表结构。

3、如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

ucore在发生页访问异常的时候，首先会按照正常的中断处理流程，访问中断处理向量，到IDT中找到对应的中断描述符，并访问到中断服务例程的段选择子，利用段选择子在GDT中取得段描述符，得到中断服务例程的起始地址。保存参数，执行中断服务例程，恢复程序继续执行。

### 释放某虚地址所在的页并取消对应二级页表项的映射

1、如果该页表项可用，对引用计数减一，如果不再有人引用的话释放，同时在相应的页目录删除该页表项，同时更新tlb。

```c
static inline void
page_remove_pte(pde_t *pgdir, uintptr_t la, pte_t *ptep) {
	if (*ptep & PTE_P) {
		struct Page *page = pte2page(*ptep);
		if (!page_ref_dec(page)) free_page(page);
		*ptep = 0;
		tlb_invalidate(pgdir, la);
	}
}
```

2、数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？

Page的全局变量实际就是总的物理地址，也就是物理页的指针集合。因此，当一个页表项或页目录项可用，他一定对应物理地址。

3、如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？ 

我认为如果先要虚拟地址与物理地址相等，首先要不启动页机制，同时设置ucore启动位置为0。这样虚拟地址就是线性地址，也就等于物理地址了。


# UCORE LAB1 系统软件启动过程

计54 马子轩 2015012283

## 实验目的

通过分析bootloader和实现lab内容，了解计算机机制的基本原理，bootloader软件的使用过程，以及ucore的基本框架。具体包括

​	计算机原理：

​	1、CPU编址寻址

​	2、CPU中断机制

​	3、外设使用

​	Bootloader:

​	1、编译运行过程

​	2、调试方法

​	3、启动过程

​	4、ELF执行与加载

​	5、外设访问

​	ucore:

​	1、编译运行ucore过程

​	2、ucore启动过程

​	3、调试ucore方法

​	4、函数调用关系

​	5、中断管理

​	6、外设管理

## 实验内容

### 理解通过make生成执行文件的过程

1、操作系统镜像文件是如何一步一步生成的?

```makefile
HOSTCC		:= gcc
HOSTCFLAGS	:= -g -Wall -O2
CC		:= $(GCCPREFIX)gcc
CFLAGS	:= -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc $(DEFS)
CFLAGS	+= $(shell $(CC) -fno-stack-protector -E -x c /dev/null >/dev/null 2>&1 && echo -fno-stack-protector)
```

这一部分是gcc编译参数。

-fno-builtin: 只将带_\_builtin\_的前缀进行优化

-ggdb: 生成gdb调试信息

-m32: 生成32位代码

-ggstabs: 生成stab调试信息

-nostdinc: 不调用标准库

-fno-stack-protector: 不使用编译器保护堆栈

```makefile
LIBDIR	+= libs
$(call add_files_cc,$(call listf_cc,$(LIBDIR)),libs,)
```

```makefile
KSRCDIR		+= kern/init \
			   kern/libs \
			   kern/debug \
			   kern/driver \
			   kern/trap \
			   kern/mm
KCFLAGS		+= $(addprefix -I,$(KINCLUDE))
$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))
```

```makefile
$(kernel): tools/kernel.ld
$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
$(call create_target,kernel)
```

使用脚本kernal.ld链接

```makefile
# create bootblock
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
```

编译bootasm.o和bootmain.o

```makefile
bootblock = $(call totarget,bootblock)

$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
$(call create_target,bootblock)
```

链接.o生成bootblock.o

```makefile
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)
```

添加sign

```makefile
UCOREIMG	:= $(call totarget,ucore.img)

$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
$(call create_target,ucore.img)
```

创建镜像



2、一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？

长512字节，以0x55AA结尾



###使用qemu执行并调试lab1中的软件

由于我的linux环境是远端的ubuntu，没有图形界面，通过ssh访问，make debug不能够直接使用，这部分实验我是在其他同学的ubuntu上面做的，并没有直接使用我自己的代码运行，但是由于此处不需要任何自己写的代码支持，所以应该也没什么问题。实际我在后续实验的调试过程中，也没用到make debug这个东西，我更多的依靠读代码和打印中间变量来调试。

修改gdbinit

```gdb
define hook-stop
x/i $pc
end
target remote localhost:1234
```

1、从CPU加电第一条指令开始，单步跟踪BIOS执行

make debug

si

即可开始单步执行

2、在初始化位置0x7c00设置实地址断点,测试断点正常。

make debug

b *0x7c00

c

si

```gdb
=> 0x7c01:      cld
0x00007c01 in ?? ()
=> 0x7c02:      xor    %eax,%eax
0x00007c02 in ?? ()
=> 0x7c04:      mov    %eax,%ds
0x00007c04 in ?? ()
=> 0x7c06:      mov    %eax,%es
0x00007c06 in ?? ()
=> 0x7c08:      mov    %eax,%ss
```

3、从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。

c

si

```gdb
(gdb) c
Continuing.
=> 0x7c00:	cli    
Breakpoint 1, 0x00007c00 in ?? ()
(gdb) si
=> 0x7c01:	cld    
0x00007c01 in ?? ()
(gdb)
=> 0x7c02:	xor    %eax,%eax
0x00007c02 in ?? ()
(gdb)
=> 0x7c04:	mov    %eax,%ds
0x00007c04 in ?? ()
(gdb)
=> 0x7c06:	mov    %eax,%es
0x00007c06 in ?? ()
(gdb)
=> 0x7c08:	mov    %eax,%ss
0x00007c08 in ?? ()
(gdb)
...
...
```

与bootblock.asm一致

4、自己找一个bootloader或内核中的代码位置，设置断点并进行测试。

设置断点0x7d0d

```gdb
(gdb) c
Continuing.
=> 0x7d0d:	push   %ebp
Breakpoint 1, 0x00007d0d in ?? ()
(gdb) si
=> 0x7d0e:	xor    %ecx,%ecx
0x00007d0e in ?? ()
(gdb)
=> 0x7d10:	mov    $0x1000,%edx
0x00007d10 in ?? ()
(gdb)
=> 0x7d15:	mov    $0x10000,%eax
0x00007d15 in ?? ()
...
...
```


```c
 /* bootmain - the entry of bootloader */
void
bootmain(void) {
    7d0d:   55                      push   %ebp
    // read the 1st page off disk
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
    7d0e:   31 c9                   xor    %ecx,%ecx
    7d10:   ba 00 10 00 00          mov    $0x1000,%edx
    7d15:   b8 00 00 01 00          mov    $0x10000,%eax
    }
}
```

两者一致

### 分析bootloader进入保护模式的过程

1、初始化，置零

```assembly
# start address should be 0:7c00, in real mode, the beginning address of the running bootloader
.globl start
start:
.code16                                             # Assemble for 16-bit mode
    cli                                             # Disable interrupts
    cld                                             # String operations increment

    # Set up the important data segment registers (DS, ES, SS).
    xorw %ax, %ax                                   # Segment number zero
    movw %ax, %ds                                   # -> Data Segment
    movw %ax, %es                                   # -> Extra Segment
    movw %ax, %ss                                   # -> Stack Segment
```

2、开启A20地址栈

```assembly
    # Enable A20:
    #  For backwards compatibility with the earliest PCs, physical
    #  address line 20 is tied low, so that addresses higher than
    #  1MB wrap around to zero by default. This code undoes this.
seta20.1:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port

seta20.2:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                                 # 0xdf -> port 0x60
    outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
```

3、初始化GDT，开启保护模式

```assembly
    # Switch from real to protected mode, using a bootstrap GDT
    # and segment translation that makes virtual addresses
    # identical to physical addresses, so that the
    # effective memory map does not change during the switch.
    lgdt gdtdesc
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0

    # Jump to next instruction, but in 32-bit code segment.
    # Switches processor into 32-bit mode.
    ljmp $PROT_MODE_CSEG, $protcseg
```

4、跳转基地址

```assembly
    # Jump to next instruction, but in 32-bit code segment.
    # Switches processor into 32-bit mode.
    ljmp $PROT_MODE_CSEG, $protcseg
```

5、设置段寄存器，并开启bootmain

```assembly
.code32                                             # Assemble for 32-bit mode
protcseg:
    # Set up the protected-mode data segment registers
    movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
    movw %ax, %ds                                   # -> DS: Data Segment
    movw %ax, %es                                   # -> ES: Extra Segment
    movw %ax, %fs                                   # -> FS
    movw %ax, %gs                                   # -> GS
    movw %ax, %ss                                   # -> SS: Stack Segment

    # Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
    movl $0x0, %ebp
    movl $start, %esp
    call bootmain
```



### 分析bootloader加载ELF格式的OS的过程

1、readsect:从secno读取数据到dst

```c
/* readsect - read a single sector at @secno into @dst */
static void
readsect(void *dst, uint32_t secno) {
    // wait for disk to be ready
    waitdisk();

    outb(0x1F2, 1);                         // count = 1
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors

    // wait for disk to be ready
    waitdisk();

    // read a sector
    insl(0x1F0, dst, SECTSIZE / 4);
}
```

2、readseg: 读取一定长度的内容

```c
/* *
 * readseg - read @count bytes at @offset from kernel into virtual address @va,
 * might copy more than asked.
 * */
static void
readseg(uintptr_t va, uint32_t count, uint32_t offset) {
    uintptr_t end_va = va + count;

    // round down to sector boundary
    va -= offset % SECTSIZE;

    // translate from bytes to sectors; kernel starts at sector 1
    uint32_t secno = (offset / SECTSIZE) + 1;

    // If this is too slow, we could read lots of sectors at a time.
    // We'd write more to memory than asked, but it doesn't matter --
    // we load in increasing order.
    for (; va < end_va; va += SECTSIZE, secno ++) {
        readsect((void *)va, secno);
    }
}
```

3、bootmain: 加载ELF

```c
/* bootmain - the entry of bootloader */
void
bootmain(void) {
    // read the 1st page off disk
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    // is this a valid ELF?
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }

    struct proghdr *ph, *eph;

    // load each program segment (ignores ph flags)
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

    // call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
```



### 实现函数调用堆栈跟踪函数

按照注释填写代码内容，打印堆栈状态

```c
void
print_stackframe(void) {
     /* LAB1 YOUR CODE : STEP 1 */
     /* (1) call read_ebp() to get the value of ebp. the type is (uint32_t);
      * (2) call read_eip() to get the value of eip. the type is (uint32_t);
      * (3) from 0 .. STACKFRAME_DEPTH
      *    (3.1) printf value of ebp, eip
      *    (3.2) (uint32_t)calling arguments [0..4] = the contents in address (unit32_t)ebp +2 [0..4]
      *    (3.3) cprintf("\n");
      *    (3.4) call print_debuginfo(eip-1) to print the C calling function name and line number, etc.
      *    (3.5) popup a calling stackframe
      *           NOTICE: the calling funciton's return addr eip  = ss:[ebp+4]
      *                   the calling funciton's ebp = ss:[ebp]
      */
	uint32_t ebp = read_ebp();
	uint32_t eip = read_eip();
	int i;
	uint32_t *p;
	for (i = 0; i < STACKFRAME_DEPTH && ebp; i++)
	{
		cprintf("ebp:%08x eip:%08x ", ebp, eip);
		p = (uint32_t*)ebp;
		cprintf("args:%08x %08x %08x %08x ", p[2], p[3], p[4], p[5]);
		cprintf("\n");
		print_debuginfo(eip-1);
		eip = p[1];
		ebp = p[0];
	}
}
```

### 完善中断初始化和处理

初始化中断向量表，特判0x80

```c
/* idt_init - initialize IDT to each of the entry points in kern/trap/vectors.S */
void
idt_init(void) {
     /* LAB1 YOUR CODE : STEP 2 */
     /* (1) Where are the entry addrs of each Interrupt Service Routine (ISR)?
      *     All ISR's entry addrs are stored in __vectors. where is uintptr_t __vectors[] ?
      *     __vectors[] is in kern/trap/vector.S which is produced by tools/vector.c
      *     (try "make" command in lab1, then you will find vector.S in kern/trap DIR)
      *     You can use  "extern uintptr_t __vectors[];" to define this extern variable which will be used later.
      * (2) Now you should setup the entries of ISR in Interrupt Description Table (IDT).
      *     Can you see idt[256] in this file? Yes, it's IDT! you can use SETGATE macro to setup each item of IDT
      * (3) After setup the contents of IDT, you will let CPU know where is the IDT by using 'lidt' instruction.
      *     You don't know the meaning of this instruction? just google it! and check the libs/x86.h to know more.
      *     Notice: the argument of lidt is idt_pd. try to find it!
      */
	extern uintptr_t __vectors[];
	int i;
	for (i = 0; i < 256; i++)
	{
		if (i == 0x80)
		{
			SETGATE(idt[i], 1, GD_KTEXT, __vectors[i], 3)
		}
		else
		{
			SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], 0)
		}
	}
	lidt(&idt_pd);
}
```

在时钟中断增加计时器，并递增。

在后续实验中我发现有已经提供的ticks变量可以直接使用因此直接调用ticks++即可。

```c
static int count = 0;

case IRQ_OFFSET + IRQ_TIMER:
    /* LAB1 YOUR CODE : STEP 3 */
    /* handle the timer interrupt */
    /* (1) After a timer interrupt, you should record this event using a global variable (increase it), such as ticks in kern/driver/clock.c
     * (2) Every TICK_NUM cycle, you can print some info using a funciton, such as print_ticks().
     * (3) Too Simple? Yes, I think so!
     */
	count++;
	if (count == TICK_NUM)
	{
		print_ticks();
		count = 0;
	}
    break;
```

## 实验结果

最终运行make qemu-nox如下

```
WARNING: Image format was not specified for 'bin/ucore.img' and probing guessed raw.
         Automatically detecting the format is dangerous for raw images, write operation
s on block 0 will be restricted.
         Specify the 'raw' format explicitly to remove the restrictions.
(THU.CST) os is loading ...

Special kernel symbols:
  entry  0x00100000 (phys)
  etext  0x00103470 (phys)
  edata  0x0010ea16 (phys)
  end    0x0010fd40 (phys)
Kernel executable memory footprint: 64KB
ebp:00007b38 eip:00100a28 args:00010094 00010094 00007b68 0010007f
    kern/debug/kdebug.c:306: print_stackframe+22
ebp:00007b48 eip:00100d25 args:00000000 00000000 00000000 00007bb8
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp:00007b68 eip:0010007f args:00000000 00007b90 ffff0000 00007b94
    kern/init/init.c:48: grade_backtrace2+19
ebp:00007b88 eip:001000a1 args:00000000 ffff0000 00007bb4 00000029
    kern/init/init.c:53: grade_backtrace1+27
ebp:00007ba8 eip:001000be args:00000000 00100000 ffff0000 00100043
    kern/init/init.c:58: grade_backtrace0+19
ebp:00007bc8 eip:001000df args:00000000 00000000 00000000 00103480
    kern/init/init.c:63: grade_backtrace+26
ebp:00007be8 eip:00100050 args:00000000 00000000 00000000 00007c4f
    kern/init/init.c:28: kern_init+79
ebp:00007bf8 eip:00007d6e args:c031fcfa c08ed88e 64e4d08e fa7502a8
    <unknow>: -- 0x00007d6d --
++ setup timer interrupts
100 ticks
End of Test.
kernel panic at kern/trap/trap.c:18:
    EOT: kernel seems ok.
Welcome to the kernel debug monitor!!
Type 'help' for a list of commands.
```



## 实验总结

经过lab1的学习，我对ucore的工作原理与系统框架有了初步了解。并熟悉了编译与调试方法。为后续7个lab打下了好基础。

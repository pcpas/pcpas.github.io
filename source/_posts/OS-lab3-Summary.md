---
title: OS_lab3_进程与异常
date: 2023-05-01 13:03:39
excerpt: BUAA_OS_Lab3总结
tags: [操作系统,内存管理]
categories: [技术,操作系统]
---

# 实验目的

1. 创建进程并运行
2. 实现时钟中断
3. 实现进程调度
# 进程
## PCB进程控制块
> PCB 是系统感知进程存在的唯一标志。进程与 PCB 是一一对应的.

在MOS中PCB使用一个`env`结构体实现, 主要包含以下信息:
```c
struct Env {
	struct Trapframe env_tf;  // Saved registers
	LIST_ENTRY(Env) env_link; // Free list
	u_int env_id;		  // Unique environment identifier
	u_int env_asid;		  // ASID
	u_int env_parent_id;	  // env_id of this env's parent
	u_int env_status;	  // 只能有三种取值: ENV_FREE/ENV_NOT_RUNNABLE/ENV_RUNNABLE
	Pde *env_pgdir;		  // Kernel virtual address of page dir
	TAILQ_ENTRY(Env) env_sched_link;
	u_int env_pri;
	// Lab 4 IPC
	u_int env_ipc_value;   // data value sent to us
	u_int env_ipc_from;    // envid of the sender
	u_int env_ipc_recving; // env is blocked receiving
	u_int env_ipc_dstva;   // va at which to map received page
	u_int env_ipc_perm;    // perm of page mapping received

	// Lab 4 fault handling
	u_int env_user_tlb_mod_entry; // user tlb mod handler

	// Lab 6 scheduler counts
	u_int env_runs; // number of times been env_run'ed
};
```
这里的`TrapFrame`结构体信息如下:
```c
struct Trapframe {
	/* Saved main processor registers. */
	unsigned long regs[32];

	/* Saved special registers. */
	unsigned long cp0_status;
	unsigned long hi;
	unsigned long lo;
	unsigned long cp0_badvaddr;
	unsigned long cp0_cause;
	unsigned long cp0_epc;
};
```

`struct Env` 中的链表项共涉及调度队列 `env_sched_list` (LIST结构)和空闲队列 `env_free_list` (TAILQ结构)两个队列, TAILQ的好处是可以在头部和尾部插入取出.

## 实现进程控制的流程
![Drawing 2023-04-16 16.19.53.excalidraw](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/Drawing%202023-04-16%2016.19.53.excalidraw.svg)
## 初始化
### env_init
`env_init`实现了Env 控制块的空闲队列和调度队列的初始化功能, 同时还实现了一个模板页目录的映射.
映射部分如下: 
```c
//申请一个物理页面
struct Page *p;
panic_on(page_alloc(&p));
p->pp_ref++;

base_pgdir = (Pde *)page2kva(p);
map_segment(base_pgdir, 0, PADDR(pages), UPAGES, ROUND(npage * sizeof(struct Page), BY2PG),
		PTE_G);
map_segment(base_pgdir, 0, PADDR(envs), UENVS, ROUND(NENV * sizeof(struct Env), BY2PG),
		    PTE_G);
```
### map_segment
`void map_segment(Pde *pgdir, u_long pa, u_long va, u_long size, u_int perm)`
功能是在一级页表基地址 `pgdir` 对应的两级页表结构中做段地址映射，将虚拟地址段 `[va,va+size) `映射到物理地址段`[pa,pa+size)`，因为是按页映射，要求 size 必须是页 面大小的整数倍。同时为相关页表项的权限为设置为 perm。它在这里的作用是**将内核中的 `Page` 和` Env `数据结构映射到用户地址**，以供用户程序读取。
> 注意: 虽然每个进程都具有自己独立的虚拟地址空间,但是这些虚拟地址空间中的内核空间,其实关联的都是同一块物理内存, 因此在初始化进程的时候, 应该将这个映射完成, 以便于用户程序读取内核中的 `Page` 和` Env `数据.

要搞懂这个函数, 首先要理解什么叫做 `Map [va, va+size) of virtual address space to physical [pa, pa+size) in the 'pgdir'.`
即理解此处MAP的含义: 使用**页表**为虚拟地址和物理地址建立映射关系, 使得可以通过页表找到虚拟地址对应的物理地址. 
那么, 我们使用什么函数来映射物理地址和虚拟地址呢? 
>` int page_insert(Pde *pgdir, u_int asid, struct Page *pp, u_long va, u_int perm)`
>- Map the physical page 'pp' at virtual address 'va'. The permission (the low 12 bits) of the page table entry should be set to 'perm|PTE_V'.

## 申请新进程
### env_alloc
两个进程管理链表初始化结束之后, 就可以开始创建进程了
1. 申请空闲的PCB(从 env_free_list里, 因为进程的最大数量是有限制的, 如果没有空闲的PCB块的话, 说明不能再申请新的进程)
2. 手动初始化进程
3. 为进程初始化页目录(每个进程有独立的地址空间, 体现为有独立的页目录\*)(`env_setup_vm`函数来做这件事)
4. 将PCB块从空闲链表里面取出
>此处比较迷惑的一点是独立的地址空间为什么只需要独立的页目录就能实现:
>因为地址空间只是虚拟的, 实际体现出来的就是va到pa的转换, 而页目录和页表实际上就记录了这个映射关系, 所以并不需要一个数据结构之类的专门存储一个"地址空间".

其中这段代码的含义要懂:
```c
/* Step 4: Initialize the sp and 'cp0_status' in 'e->env_tf'. */
	// Timer interrupt (STATUS_IM4) will be enabled.
	e->env_tf.cp0_status = STATUS_IM4 | STATUS_KUp | STATUS_IEp;
	// Keep space for 'argc' and 'argv'.
	e->env_tf.regs[29] = USTACKTOP - sizeof(int) - sizeof(char **);
```

### env_setup_vm
```c
/* Overview:
 *   Initialize the user address space for 'e'.
 */
static int env_setup_vm(struct Env *e) {
	/* Step 1:
	 *   Allocate a page for the page directory with 'page_alloc'.
	 *   Increase its 'pp_ref' and assign its kernel address to 'e->env_pgdir'.
	 */
	struct Page *p;
	try(page_alloc(&p));
	/* Exercise 3.3: Your code here. */
	p->pp_ref++;
	e->env_pgdir = (Pde *)page2kva(p);
	/* Step 2: Copy the template page directory 'base_pgdir' to 'e->env_pgdir'. */
	memcpy(e->env_pgdir, base_pgdir, BY2PG);
	/*memcpy(e->env_pgdir + PDX(UTOP), base_pgdir + PDX(UTOP),
	       sizeof(Pde) * (PDX(UVPT) - PDX(UTOP)));
	只复制赋值了的一段也是ok的, 但是没必要*/

	/* Step 3: Map its own page table at 'UVPT' with readonly permission.
	 * As a result, user programs can read its page table through 'UVPT' */
	e->env_pgdir[PDX(UVPT)] = PADDR(e->env_pgdir) | PTE_V;
	return 0;
}
```
这里说三点:
1. 怎么申请一个新的pgdir: 其实就是申请一个新的页面, 因为一个页目录其实就存放在一个物理页里面.
2. 使用`void *_memcpy_(void *destin, void *source, unsigned n)；`来进行内存的拷贝
	- 所有系统函数使用的都是虚拟地址
	- 如果只复制其中一段的话要注意地址是怎么算的
3. 关于最后一行代码的理解
>`e->env_pgdir`本身是页目录的指针, 因为MOS 中，将页表和页目录映射到了用户空间中的 0x7fc00000-0x80000000区域(即UVPT所指的4M空间), 因而, 根据自映射的原理, 页目录中的第PDX(UVPT)项就应该映射的是页目录本身. 因此, `e->env_pgdir[PDX(UVPT)]`这一个PDE的值就应该是`PADDR(e->env_pgdir)+perm`. 

## 将程序加载进进程
前面我们完成了一个进程的申请和初始化, 那么问题来了, 这进程是用来干嘛的?
每个进程一开始都是一样的, 只有加载了不同的程序代码段后, 才会发挥不一样的功能.
即我们需要将程序加载到新进程的地址空间中(程序是指elf文件中的的可执行文件).
要想正确加载一个 ELF 文件到内存，只需将 ELF 文件中所有需要加载的程序段（program segment）加载到对应的虚拟地址上即可。
> 我们已经写好了用于解析 ELF 文件的代码中的大部 分内容，你可以直接调用相应函数获取 ELF 文件的各项信息，并完成加载过程。
> ```c
> // lib/elfloader.c
> const Elf32_Ehdr *elf_from(const void *binary, size_t size);
> int elf_load_seg(Elf32_Phdr *ph, const void *bin, elf_mapper_t map_page, void *data);
> 
> // kern/env.c
> static void load_icode(struct Env *e, const void *binary, size_t size);
> static int load_icode_mapper(void *data, u_long va, size_t offset, u_int perm,
> 	const void *src, size_t len);
> ```

这部分的问题其实不是很大

## 创建进程
创建进程包括了申请新进程和加载程序, 且是指在操作系统内核初始化时直接创建进程.
使用的是`struct Env *env_create(const void *binary, size_t size, int priority)`
```c
struct Env *env_create(const void *binary, size_t size, int priority) {
	struct Env *e;
	/* Step 1: Use 'env_alloc' to alloc a new env. */
	env_alloc(&e, 0);
	/* Step 2: Assign the 'priority' to 'e' and mark its 'env_status' as runnable. */
	e->env_pri = priority;
	e->env_status = ENV_RUNNABLE;
	/* Step 3: Use 'load_icode' to load the image from 'binary', and insert 'e' into
	 * 'env_sched_list' using 'TAILQ_INSERT_HEAD'. */
	load_icode(e, binary, size);
	TAILQ_INSERT_HEAD(&env_sched_list, e, env_sched_link);
	return e;
}
```

## 进程运行和切换
`env_run` 是进程运行使用的基本函数，它包括两部分： 
- 保存当前进程上下文 (如果当前没有运行的进程就跳过这一步) 
- 恢复要启动的进程的**上下文**(其实就是Trapframe, 即进程执行时所有寄存器的状态)，然后运行该进程。
> **Overview:** 
> 	Switch CPU context to the specified env 'e'
> **Post-Condition:** 
> 	Set 'e' as the current running env 'curenv'.

> 运行一个新进程往往意味着是进程切换，而不是单纯的进程运行.
> 要理解进程切换，我们 就要知道进程切换时系统需要做些什么。实际上进程切换的时候，为了保证下一次进入这个进程 的时候我们不会再“从头来过”，而是有记忆地从离开的地方继续往后走，我们要保存一些信息， 那么，需要保存什么信息呢？
> 事实上，我们只需要保存进程的上下文信息，包括通用寄存器、HI、 LO 和 CP0 中的 SR，EPC，Cause 和 BadVAddr 寄存器。进程控制块除了 env_tf 其他的字段在 进程切换后还保留在原本的进程控制块中，并不会改变，因此不需要保存。
> 在 Lab3 中，我们在本实验里的寄存器状态保存的地方是 `KSTACKTOP` 以下的一个 `sizeof(TrapFrame)` 大小的区域中。(`curenv->env_tf = *((struct Trapframe *)KSTACKTOP - 1)`)

总结以上说明，我们不难看出 env_run 的执行流程：
1. 保存当前进程的上下文信息。 
2. 切换 curenv 为即将运行的进程。 
3. 设置全局变量 cur_pgdir 为当前进程页目录地址，在 TLB 重填时将用到该全局变量。 
4. 调用 env_pop_tf 函数，恢复现场、异常返回。

# 中断与异常
>我们实验里认为中断是异常的一种，并且是仅有的一种异步异常。


CPU 不仅仅有我们常见的 32 个通用寄存器，还有 功能广泛的协处理器，而中断/异常部分就用到了其中的协处理器 CP0.
下面的表格介绍了编号 为 12，13，14 的三个 CP0 寄存器的具体功能。
![image.png](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/20230416210751.png)
![image.png](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/20230416210853.png)
![image.png](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/20230416210904.png)
> - **SR 寄存器**：图3.1(在设置进程控制块部分给出) 是 MIPS R3000 中 Status Register 寄存器， 15-8 位为中断屏蔽位，每一位代表一个不同的中断活动，其中 15-10 位使能外部中断源，9-8 位 是 Cause 寄存器软件可写的中断位。 
> - **Cause 寄存器**：图3.3是 MIPS R3000 中 Cause 寄存器。其中保存着 CPU 中哪一些中断或 者异常已经发生。15-8 位保存着哪一些中断发生了，其中 15-10 位来自硬件，9-8 位可以由软件 写入，当 SR 寄存器中相同位允许中断（为 1）时，Cause 寄存器这一位活动就会导致中断。6-2 位（ExcCode），记录发生了什么异常。

MIPS CPU 处理一个异常时大致要完成四项工作： 
1. 设置 EPC 指向从异常返回的地址。 
2. 设置 SR 位，强制 CPU 进入内核态（行使更高级的特权）并禁止中断。 
3. 设置 Cause 寄存器，用于记录异常发生的原因。 
4. CPU 开始从异常入口位置取指，此后**一切交给软件处理**。
而这句“一切交给软件处理”，就是我们当前任务的开始。
![image.png](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/20230416211423.png)

## 异常的分发
当发生异常时，处理器会进入一个用于**分发异常**的程序，这个程序的作用就是检测发生了哪 种异常，并调用相应的异常处理程序。
```armasm
.section .text.exc_gen_entry
exc_gen_entry:
	SAVE_ALL
	mfc0 t0, CP0_CAUSE #将 Cause 寄存器的内容拷贝到 t0 寄存器中
	andi t0, 0x7c #取得 Cause 寄存器中的 2~6 位，也就是对应的异常码，这是区别不同异常的重要标志
	lw t0, exception_handlers(t0) #以得到的异常码作为索引在 exception_handlers 数组中找到对应的中断处理函数
	jr t0 #跳转到对应的异常处理函数中，从而响应了异常
```

在我们的系统中，CPU 发生异常（除了用户态地址的 TLB Miss 异常）后，就会自动跳转到地址 0x80000080 处；发生用户态地址的 TLB Miss 异常时，会自动跳转到地址 0x80000000 处。
因此, 在`kernel.lds`中添加代码, 以保证异常被响应:
```
. = 0x80000000;
	.tlb_miss_entry : {
		*(.text.tlb_miss_entry)
	}
	. = 0x80000080;
	.exc_gen_entry : {
		*(.text.exc_gen_entry)
	}
```

## 中断的处理
接上文, 跳转到对应的异常处理函数中, 那么异常处理函数有哪些呢?
```c
void (*exception_handlers[32])(void) = {
    [0 ... 31] = handle_reserved,
    [0] = handle_int,
    [2 ... 3] = handle_tlb,
    [1] = handle_mod,
    [8] = handle_sys,
};
```
>- **0 号异常** 的处理函数为 handle_int，表示**中断**，由时钟中断、控制台中断等中断造成 
>- **1 号异常** 的处理函数为 handle_mod，表示存储异常，进行存储操作时该页被标记为只读 
>- **2 号异常** 的处理函数为 handle_tlb，表示 TLB load 异常 
>- **3 号异常** 的处理函数为 handle_tlb，表示 TLB store 异常 
>- **8 号异常** 的处理函数为 handle_sys，表示系统调用，用户进程通过执行 syscall 指令陷 入内核

这里先谈**中断**.
Cause 寄存器中有 8 个独立的中断位。
其中 6 位是硬件中断，另外 2 位是软件中断，且不同中断处理起来也会有差异。
我们首先来介绍一下中断处理的流程:
1. 通过异常分发，判断出当前异常为**中断异常**，随后进入相应的中断处理程序。在 MOS 中即对应 handle_int 函数。 
2. 在中断处理程序中进一步判断 Cause 寄存器中是由**几号中断**位引发的中断，然后进入不同中断对应的中断服务函数。 
3. 中断处理完成，通过 ret_from_exception 函数恢复现场，继续执行。

我们只需要完成1,2步, 并且只需要完成时钟中断(**4 号中断**)的部分就可以了.
>时钟中断和操作系统的时间片轮转算法是紧密相关的。
>时间片轮转调度是一种进程调度算法，每个进程被分配一个时间段，称作它的时间片，即该进程允许运行的时间。如果在时间片结束时进程还在运行，则该进程将挂起，切换到另一个进程运行。
>那么 CPU 是如何知晓一个进程的时间片结束的呢？
>通过定时器产生的时钟中断。当时钟中断产生时，当前运行的进程被挂起，我们需要在调度队列中选取一个合适的进程运行。
>如何“选取”，就要涉及到**进程的调度**了。

![Drawing 2023-04-16 21.47.50.excalidraw](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/Drawing%202023-04-16%2021.47.50.excalidraw.svg)
调度函数的实现比较简单, 就不细讲了.

## 进程调度
### 轮询调度
今天课上提到晚上要考进程调度, 这样的话还是看看吧!
先来看看最初始版本的进程调度是怎么实现的:
```c
/* Overview:
 *   Implement a round-robin scheduling(轮询调度算法) to select a runnable env and schedule it using 'env_run'.
*/*
void schedule(int yield) {
	static int count = 0; // remaining time slices of current env
	struct Env *e = curenv;
	//有两种情况: 需要切换进程和不需要切换进程
	if(yield!=0 || count<=0 || e==NULL || e->env_status != ENV_RUNNABLE)
	{
		if(e!=NULL && e->env_status == ENV_RUNNABLE)
		{
			TAILQ_REMOVE(&env_sched_list, e, env_sched_link);
			TAILQ_INSERT_TAIL(&env_sched_list, e, env_sched_link);
		}
		panic_on(TAILQ_EMPTY(&env_sched_list)!=0);
		e = TAILQ_FIRST(&env_sched_list);
		count = e->env_pri;	
	}
	count--;
	env_run(e);
}
```
这个调度算法是最简单的**轮询调度**, 并使用时间片判断, 逻辑如下:
- 此时有两种情况: 需要切换进程和不需要切换进程
- 如果当前没有进程/进程时间片用光/进程不再可运行/yield指定切换, 那么**切换进程**
	- (如果当前进程不是空进程且仍然处在`ENV_RUNNABLE`状态的话) 将当前进程移动到调度队列尾部
	- **从进程调度队列`env_sched_list`取出一个新进程(不要把他从调度队列里删掉)(没有可用时panic)**
	- 设定count为进程优先级`e->env_pri`
	- count--
	- 运行该进程
- 否则**继续运行进程**
	- count--
	- 运行该进程

其中调度算法即实现中间这一步: 
>**从进程调度队列`env_sched_list`取出一个新进程(没有可用时panic)**

在课下实验我们并没有按照优先级来实现算法, 而是简单的插入尾部和取头部.
### 其他可能的调度方法
（待补充）

# 总结 
至此，我们的 Lab3 就算是圆满完成了。
中断部分我们只学习了时钟中断, 其实TLB_miss的处理也是一个值得学习的地方.
指导书中简单介绍了系统如何响应TLB_miss.
![image.png](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/20230416215810.png)

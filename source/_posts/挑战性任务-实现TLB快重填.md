---
title: OS_挑战性任务-实现TLB快重填
date: 2023-06-22 13:03:39
excerpt: BUAA_OS_挑战性任务总结
tags: [操作系统,内存管理]
categories: [技术,操作系统]
topic: os
---

# 题目

实现TLB快重填

![image-20240407200928561](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/Image/image-20240407200928561.png)

![image-20240407200943261](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/Image/image-20240407200943261.png)

# 实现思路

总的来说，lab2挑战性任务的实现过程主要分为四个部分，分别是：

1. 将页表映射到 kseg2 区域。
2. 初始化Context寄存器。
3. 实现 TLB 快速重填程序。
4. 实现一个测试程序体现重填加速效果。
   接下来，我将一步一步地对我做的工作进行介绍。

## 将页表映射到 kseg2 区域

首先我们要明确页表映射的位置。
kesg2空间较大（1G），且似乎未被使用过，故可以随意选择页表的映射位置。我这里选择将页表映射到kseg的起始位置，即`KSEG2`。
我们知道每个进程初始化其虚拟内存是在`\kern\env.c`文件中的`env_setup_vm`函数内完成的, 函数将进程的页表映射至`UVPT`。为了在在不影响 MOS 已有的页面管理机制的情况下，将 MOS 目前的页表映射到kseg2，我没有破坏原本在`UVPT`的映射, 而是选择加入新的到`KSEG2`的映射。

```c
static int env_setup_vm(struct Env *e) {
	struct Page *p;
	try(page_alloc(&p));
	p->pp_ref++;
	e->env_pgdir = (Pde *)page2kva(p);
	memcpy(e->env_pgdir, base_pgdir, BY2PG);
	e->env_pgdir[PDX(UVPT)] = PADDR(e->env_pgdir) | PTE_V;
	p->pp_ref++;
	e->env_pgdir[PDX(KSEG2)] = PADDR(e->env_pgdir) | PTE_V;
	return 0;
}
```

## 初始化Context寄存器

首先要了解Contest寄存器的工作原理：
![image.png](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/20230616204601.png)
`PTEBase`：kseg2 页表基地址的高11位。
`Bad VPN`：出现 TLB miss 异常的地址的虚页号，与 BadVaddr 的高位相同。

这里kseg2页表基址是11位，也就是说context寄存器要求我们的页表大小是$2^{21}$位，也即是2MB对齐的，这正好对应了用户空间的大小。
这里的基址需要我们自己去设置，使用`mtc0`指令即可，设置一次，后面都无需设置。

我是在`page_init`分页管理初始化函数中对CONTEXT寄存器进行设置的，调用了我在`tlb_asm.S`中定义的函数`tlb_init` 。

```
LEAF(tlb_init)
    li  t0, 0xC0000000
    mtc0    t0, CP0_CONTEXT
    jr ra
END(tlb_init)
```

##  实现 TLB 快速重填程序

这是整个任务中最困难和最复杂的部分。
先看代码，然后我根据代码添加注释梳理实现的思路：

```
# 声明三个变量，分别用于暂存ra，ENTRYHI和BADVADDR的值
.data
temp_EPC:
.word 0
temp_ENTRYHI:
.word 0
temp_BADVADDR:
.word 0
.text
# 快速重填函数的主体部分
fast_tlb_refill:
.set noreorder
.set noat
	# 在这个函数中最好只使用k0,k1寄存器以避免未知错误
	# 从对应寄存器中取出我们想要暂存的值，包括CP0_EPC,CP0_ENTRYHI,CP0_BADVADDR
	mfc0 k0, CP0_EPC
	la k1, temp_EPC
	sw k0, 0(k1)
	mfc0 k0, CP0_ENTRYHI
	la k1, temp_ENTRYHI
	sw k0, 0(k1)
	mfc0 k0, CP0_BADVADDR
	la k1, temp_BADVADDR
	sw k0, 0(k1)

	# 取得CP0_CONTEXT的值
	mfc0 k1, CP0_CONTEXT
	# 可能会发生异常重入，重入后进入通用处理函数
	lw k1, 0(k1) 
	nop

	# 将我们获取和保存的CP0_ENTRYLO0和CP0_ENTRYHI值填入寄存器,
	la k0, temp_ENTRYHI
	lw k0, 0(k0)
	mtc0 k1, CP0_ENTRYLO0
	mtc0 k0, CP0_ENTRYHI

	# 检查PTE是否有效
	andi k1, k1, PTE_V
	# 若PTE无效，跳转至default_tlb_refill
	beqz k1, default_tlb_refill 
	nop

	# 恢复CP0_EPC的值
	la k1, temp_EPC
	lw k1, 0(k1)
	nop
	# TLB重填
	tlbwr
	nop
	jr k1
	rfe
	
default_tlb_refill:
	# 因为可能在之前发生了异常重入，导致BADVADDR和EPC的值发生变化，因此必须在这里手动填充
	la k0, temp_BADVADDR
	la k1, temp_EPC
	lw k0, 0(k0)
	lw k1, 0(k1)
	mtc0 k0, CP0_BADVADDR #恢复寄存器的值，下同
	mtc0 k1, CP0_EPC
	# 填好寄存器的值后跳转到通用处理
	j exc_gen_entry
	nop
.set at
.set reorder
```

这里需要注意的就是两次异常处理：
首先，访问Kseg2区域会经过TLB，因此可能会发生异常重入现象：
我们访问页面的逻辑是这样的：
欲访问一个页面->查找TLB->TLB Miss进入快充填程序->查找位于KSEG2的页表->查找TLB
如果在最后一个TLB处发生了缺页，那么也就是在异常处理的过程中再次发生异常，也就是这里所说的异常重入现象。
MIPS R3000的机制是在此处（KESG2）发生TLB MISS是不会进入我们的快重填程序的，而是直接进入通用异常处理程序。我们必须要对这个机制有所了解，**因为二次异常的发生会破坏第一次异常寄存器所保存的现场**。

第二就是如果发生PTE无效的情况，我们需要有所处理。
这里由于笔者时间有限等综合因素，其实没有什么很好的处理方法，故选择直接跳转至原本的通用异常处理程序进行处理。
由于局部性原理，这种情况发生的概率总体来看较小，故这是可以接受的。

第三我们要记得在进程销毁时无效化旧的TLB表项。

我们将页表项映射至KSEG2后，还需要在**合适的时间对其无效化**，如果不进行这个工作的话，**可能在多进程的情况下出错。**

<img src="https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/Image/image-20240407201337033.png" alt="image-20240407201337033" style="zoom:50%;" />

## 小结

到这里TLB快重填的实现部分就结束了，从代码量来看较为简单，但是考虑到资料较少，要写汇编代码，要对MOS和MIPS R3000的各种机制有深刻了解，debug困难等因素，其实难度并不低。
还剩一个部分，即编写一个测试程序进行测试，这部分我留在下文的测试程序部分进行说明。

# 测试程序

要求为实现一个测试程序体现重填加速效果，测试程序本身及运行测试程序得到的运行结果应具有足够的可读性。
其实想要测试快重填的加速效果非常简单，因为用户进程运行的过程实质上不断地在进行TLB重填，故对任何用户程序都有一定的加速效果。
但是为了体现出直观的差异，我们需要简单地对测试程序进行设计。
我们知道一个物理页框的大小是4KB，如果用于存放int数组可以容纳的长度为1024，而TLB可以容纳的表项数为32个表项，所以我们可以创建一个长度为64个页框大小的数组并循环依次访问每一个页框。
这样可以最大化TLB MISS的次数，使得优化过后的程序体现出速度上的明显优势。

程序：

```c
#include <lib.h>

int temp[65536];

int main() {
	u_int s1, us1;
	s1 = get_time(&us1);
	debugf("测试程序开始运行...\n");
	debugf("请耐心等待\n");
	//循环访问500000次
	for(int i=0; i<500000 ; i++){
		//依次访问每一个页框
		for(int j=0; j < 65536; j+=1024){
			temp[j] = j;
		}
	}

	u_int s2, us2;
	s2 = get_time(&us2);
	debugf("测试程序运行结束! :)\n");
	debugf("总耗时约 %d 秒\n", s2-s1); 
	return 0;
}
```

注：这里因为时间的数量级为十秒，我忽略了毫秒带来的小于1秒的差距。

使用快重填的实验结果：
![image.png](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/20230616225058.png)

不使用快重填的实验结果：
![image.png](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/20230616225303.png)

对比发现，在测试程序上，快重填的速度大约为原方法的4.5倍，取得了较好的效果😊

# 实现过程中遇到的问题及相应的解决方案

我认为本实验任务和其他任务的一个明显特点就是其他实验都是写C语言代码，但是这个实验需要写汇编代码。
LAB2的挑战性任务从代码量来看较为简单，但是考虑到资料较少，要写汇编代码，要对MOS和MIPS R3000的各种机制有深刻了解，debug困难等因素，其实难度并不低。
下面我罗列一下我遇到的主要问题和解决方案。

## 怎么将页表映射到KSEG2上

这里其实非常考验对于页表映射的理解，需要对我们是怎样建立页表从物理空间到内核空间，再到用户空间的映射有一个全面的把握，过程非常复杂。
但是其实我们也可以化繁为简，抓住主要因素。
我们要重点理解`env_setup_vm`函数中的这行代码：

```c
e->env_pgdir[PDX(UVPT)] = PADDR(e->env_pgdir) | PTE_V;
```

其实当时的思考题也让我们去重点理解了，但是现在看来我对它的理解还是略显不足，
通过去梳理和理解这个过程，就可以知道其实映射很简单，就是再添加下列代码即可。

```c
	p->pp_ref++;
	e->env_pgdir[PDX(KSEG2)] = PADDR(e->env_pgdir) | PTE_V;
```


## 怎么理解TLB Miss的处理过程

TLB miss 异常与其他异常不同，在访问用户空间 `kuseg` 地址，触发 TLB miss 异常时，CPU 的 PC 不会跳转到通用的异常处理程序入口，而是跳转到专用的处理程序入口。在我们的 MOS 中，通用异常入口位于 `0x80000080`，TLB miss 异常专用入口位于 `0x80000000`。目前的 MOS 没有实现 TLB miss 的专用快速重填程序，而是在此处直接跳转至通用异常入口进行处理。

所以实现TLB的快重填实际上是要求我们完成`0x80000000`处的这个处理函数。

（这里也要注意，访问 `kseg2`地址时不会进入`0x80000000` ，而是直接进入`0x80000080`， 这一点也卡了我很久。）

如果我们不知道原本的TLB Miss是如何处理的，那么就很难去理解实验到底要求我们去做什么。所以在此之前我们先来学习一下通用的异常处理流程。

### 通用异常处理流程

首先，在`kernel.lds`中增加如下代码, 为操作系统增加异常分发功能:

```c
. = 0x80000000;
.tlb_miss_entry : {
	*(.text.tlb_miss_entry)
}
. = 0x80000080;
.exc_gen_entry : {
	*(.text.exc_gen_entry)
}
```

这里不需要进行任何改动.

经过这里的声明, 系统在发生异常的时候就知道该去访问哪里来解决异常, 接下来我们在`entry.S`文件中完成我们两个异常处理函数

```c
.section .text.tlb_miss_entry
tlb_miss_entry:
	j       exc_gen_entry

.section .text.exc_gen_entry
exc_gen_entry:
	SAVE_ALL
	mfc0 t0, CP0_CAUSE
	andi t0, 0x7c
	lw t0, exception_handlers(t0)
	jr t0
```

可以看到, 在原本的MOS中, TLB Miss后虽然会交给`tlb_miss_entry`处理, 但是实际上`tlb_miss_entry`又会紧接着转交给`exc_gen_entry`这一通用处理程序.

1. 首先，指令SAVE_ALL被执行。这是一个宏指令，它将所有通用寄存器保存在栈上，以便在处理异常时能够恢复它们的值。
2. 接下来，mfc0指令用于将CP0_CAUSE寄存器的内容加载到t0寄存器中。CP0_CAUSE寄存器包含了异常的原因码以及其他相关的标志位。
3. 最后, 通过异常码跳转到对应的处理程序中, 开始异常处理, 这里借用了`exception_handlers`函数, 该函数在`traps.c`文件中声明:

```c
void (*exception_handlers[32])(void) = {
    [0 ... 31] = handle_reserved,
    [0] = handle_int,
    [2 ... 3] = handle_tlb,
};
```


这里我们学习一下通用寄存器的TLB Miss的异常处理方法`handle_int`, 来启发我们如何编写我们自己的快速重填函数.
值得注意的是, `CP0_CAUSE`又将TLB Miss分为了2, 3两个异常码(load和store), 但是MOS中用了一个统一的处理函数进行处理.


`handle_int`是在`genex.S`文件中进行定义的:

```
NESTED(handle_tlb, TF_SIZE + 8, zero)
	move    a0, sp
	addiu   sp, sp, -8
	jal     do_tlb_refill
	addiu   sp, sp, 8
	j       ret_from_exception
END(handle_tlb)
```

1. 首先, `move`指令将当前栈指针（sp）的值复制到寄存器a0中, 以便作为参数传递给`do_tlb_refill`.
2. 然后，`addiu`指令将栈指针向下调整8个字节，为`do_tlb_refill`函数的局部变量或其他需要的数据腾出空间。
3. 接着，`jal`指令调用`do_tlb_refill`函数。`jal`指令会将当前指令的地址（返回地址）保存到链接寄存器（$ra）中，并跳转到目标函数的地址。
4. 在`do_tlb_refill`函数执行完毕后，`addiu`指令将栈指针恢复回原来的位置，以释放先前分配的局部变量的空间。
5. 最后，`j`指令跳转到`ret_from_exception`，这是一个通用的异常返回例程，用于从异常处理中返回到正常的执行流程中。

这里涉及到`do_tlb_refill`和`ret_from_exception`两个值得研究的函数, 我们先看看`ret_from_exception`有何作用:

```
FEXPORT(ret_from_exception)
	RESTORE_SOME
	lw      k0, TF_EPC(sp)
	lw      sp, TF_REG29(sp) /* Deallocate stack */
.set noreorder
	jr      k0
	rfe
.set reorder
```

1. 首先函数执行了`RESTORE_SOME`指令, 这和之前进入异常通用处理程序执行的`SAVE ALL`一样是宏指令, 在`stackframe.h`函数中被提前定义。这两个宏函数是用于在异常处理过程中保存和恢复寄存器状态的工具。这两个宏函数一起工作，用于在异常处理过程中保存寄存器状态，并在异常处理结束后恢复寄存器状态。这样可以确保异常处理完成后，程序能够正确地恢复到异常发生之前的状态，并继续执行。
2. 然后，lw指令从栈中加载`TF_EPC(sp)`的值到k0寄存器中。`TF_EPC`是异常发生时保存的EPC（Exception Program Counter）寄存器的偏移量，用于存储引发异常的指令地址。
3. 紧接着，lw指令从栈中加载`TF_REG29(sp)`的值到sp寄存器中。`TF_REG29`是异常发生时保存的栈指针的偏移量，通过加载它的值到sp寄存器，可以恢复之前的栈指针状态。
4. 然后，`jr`指令将跳转到k0寄存器中存储的地址，也就是之前保存的EPC值。这样，程序将继续执行引发异常的指令之后的代码。
5. 最后，`rfe`指令用于从异常处理中返回，并恢复处理器的状态。

可以看到, 其实`ret_from_exception`做的是一个异常处理后恢复现场的工作, 而主要的异常处理的过程是在`do_tlb_mod`中完成的


现在我们来看看`do_tlb_refill`的内容:

```
.data
tlb_refill_ra:
.word 0
.text
NESTED(do_tlb_refill, 0, zero)
	mfc0    a0, CP0_BADVADDR
	mfc0    a1, CP0_ENTRYHI
	srl     a1, a1, 6
	andi    a1, a1, 0b111111
	sw      ra, tlb_refill_ra
	jal     _do_tlb_refill
	lw      ra, tlb_refill_ra
	mtc0    v0, CP0_ENTRYLO0
	// See <IDT R30xx Family Software Reference Manual> Chapter 6-8
	nop
	/* Hint: use 'tlbwr' to write CP0.EntryHi/Lo into a random tlb entry. */
	/* Exercise 2.10: Your code here. */
	tlbwr
	jr      ra
END(do_tlb_refill)
```

`do_tlb_refill`函数的主要作用是传参并调用了`_do_tlb_refill`函数，真正的处理在`_do_tlb_refill`函数中进行。

```c
/* Overview:
 *  Refill TLB.
 */
Pte _do_tlb_refill(u_long va, u_int asid) {
	Pte *pte;
	while(page_lookup(cur_pgdir, va, &pte)==NULL)
	{
		passive_alloc(va, cur_pgdir, asid);
	}
	return *pte;
}
```

这是真正处理缺页的函数, 可以看到逻辑很简单:

1. 使用`page_lookup(cur_pgdir, va, &pte)`函数去查找当前进程的页表,
   1. 如果存在, 说明已经为这个虚拟页分配了物理页框,进入下一步
   2. 如果不存在, 那么使用`passive_alloc(va, cur_pgdir, asid)`为这页分配一个页框, 并更改页表
2. 返回对应的页表项

### 在此基础上建立快重填

如果理解了通用处理和快重填的关系，就知道其实我们不是要去重写通用处理，而是相当于去利用CONTEXT寄存器快速完成`page_lookup`的过程，同时也不需要那么保存那么多寄存器，省去了很多时间和工作。


## 汇编程序编写的细节问题

这部分真是让人非常头疼，我下面列举一些问题，这些问题都可以通过查阅资料，反复尝试和手动调试解决，但是问题多，细节多，而且发生错误后很难定位问题，故让人非常困扰，当然也有笔者是软件学院学生，对汇编理解不够深刻的原因。

- nop问题：汇编程序编写过程中很多时候需要手动添加nop。什么时候添加？
- 寄存器问题：每个寄存器应该在何时被保存和恢复，又该如何保存和恢复哪些寄存器？
- 汇编函数调用问题：汇编函数之间的调用涉及到栈和sp的问题
- 值的保存和传递问题：有些值需要被保存和传递，我该怎样实现？
- 指令条数问题：笔者在实现快重填时发生了因为`tlb_miss_entry`太长和`exc_gen_entry`发生重叠的问题，怎么解决？

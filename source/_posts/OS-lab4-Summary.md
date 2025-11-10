---
title: OS_lab4_系统调用和fork
date: 2023-05-08 13:03:39
excerpt: BUAA_OS_Lab4总结
tags: [操作系统,内存管理]
categories: [技术,操作系统]
topic: os
---

本次实验实现了用户态的系统调用, 进程间通信和fork.
这是在Lab3创建了一个用户进程之后, 对用户态功能的进一步完善.
# 系统调用(syscall)
在用户态下，用户进程不能访问系统的内核空间，也就是说它一般不能存取内核使用的内 存数据，也不能调用内核函数，这一点是由体系结构保证的。然而，用户进程在特定的场景下往 往需要执行一些只能由内核完成的操作，如操作硬件、动态分配内存，以及与其他进程进行通信 等。允许在内核态执行用户程序提供的代码显然是不安全的，因此操作系统设计了一系列内核空 间中的函数，当用户进程需要进行这些操作时，会引发特定的异常以陷入内核态，由内核调用对 应的函数，从而安全地为用户进程提供受限的系统级操作，我们把这种机制称为**系统调用**。
因此, 为了深入的理解系统调用的作用, 我们必须首先理解用户态和内核态的含义和区别.

# 用户态和内核态
## 页与页表的问题
首先我们从一个简单的问题讲起:
```
下列六项分别存放在内存的哪里?
1. p
2. page2pa(p)
3. page2kva(p)
4. 一级页表（页目录）的存放位置
5. 二级页表的存放位置
6. 页的存放位置
```
回答此问题应该从虚拟地址和物理地址两方面考虑。实际上，访问这些对象需要分内核和用户两个角色，因为内核态访问某个地址一般是通过将物理地址高位置为1转化为kseg0虚拟地址来访问的，而用户态可访问的虚拟地址只有kuseg，需要经过二级页表转换来访问物理内存。

以下是内核态视图下，这些数据的存放位置：

1.  `p`: 位于 `pages` 数组。`pages` 数组是在 `mips_vm_init` 中通过 `alloc` 函数动态分配的，所以其位置位于kseg0中紧邻 `KSTACKTOP` 的上方区域。
2.  `page2pa(p)`: 表示 `Page *` 指针 `p` 代表的**物理地址**，可以表示物理地址的任何区域，内核态访问时一般会将其转化为kseg0虚拟地址来访问。
3.  `page2kva(p)`: 表示 `Page *` 指针 `p` 代表的物理地址转化成的kseg0虚拟地址，可以位于kseg0内存区域的任何位置。
4.  一级页表（页目录）的存放位置：一级页表是在 `pmap.c` 中通过 `page_alloc` 分配的页，其位于物理内存的**之前的某个空闲位置**，对应的kseg0虚拟地址位于 `KSTACKTOP`  的上方。
5.  二级页表的存放位置：与一级页表相同，都是通过 `page_alloc` 分配的页，对应的kseg0虚拟地址都位于 `KSTACKTOP` 的上方。
6.  页的存放位置：一般的页都是通过 `page_alloc` 分配的，对应的kseg0虚拟地址都位于 `KSTACKTOP` 的上方。

以下是用户态视图下，这些数据的存放位置：

1.  `p`：在 `env.c` 中，`pages` 数组被映射到kuseg虚拟地址的 `UPAGES` 区域。
2.  `page2pa(p)`：用户态无法随意访问物理地址。
3.  `page2kva(p)`: 用户态无法访问kseg0虚拟地址。
4.  一级页表（页目录）的存放位置：页表被映射到了kuseg的 `UVPT` 区域。根据页目录的自映射原理，结合 `env_setup_vm(env.c)` 中的最后一句，可以得出页目录的位置在 `UVPT + (UVPT >> 10)`。
5.  二级页表的存放位置：位于kuseg的 `UVPT` 区域，大小为4M。
6.  页的存放位置：对于某个进程的kuseg虚拟内存，如果存在一个虚拟地址 `va` 能够映射到该页，那么该进程就可以通过 `va` 来访问这个物理页。否则不能访问。

用户态有自己独立的地址空间(但是每个地址空间都共享一部分内核空间暴露出来的部分), 但是无法直接访问和操作内核态的大部分内容, 如果想要做这件事情, 就必须通过系统调用.

# Fork
Fork整体的实现比较自然, 但是其中有一个地方比较难以理解, 下面仔细讲讲:

## 页写入异常
我们使用了写时复制（COW）特性，这种特性也是依赖于异常处理的。
当用户程序写入一个在 TLB 中被标记为不可写入（无 PTE_D）的页面时，MIPS 会陷入页写入异常（TLB Mod）， 我们在异常向量组中为其注册了一个处理函数 `handle_mod`，这一函数会跳转到 `kern/tlbex.c`中的`do_tlb_mod` 函数中，这个函数正是处理页写入异常的内核函数。对于需要写时复制（COW） 的页面，我们只需取消其 PTE_D 标记，即可在它们被写入时触发 `do_tlb_mod` 中的处理逻辑。

我们来分析一下`do_tlb_mod`的具体实现:

```c
void do_tlb_mod(struct Trapframe *tf) {
	struct Trapframe tmp_tf = *tf;
	//如果此时在用户的栈空间之外, 那么将sp设置为用户异常栈栈顶
	if (tf->regs[29] < USTACKTOP || tf->regs[29] >= UXSTACKTOP) {
		tf->regs[29] = UXSTACKTOP;
	}
	//sp减去一个tf的大小,相当于取这个tf之下的一个tf
	tf->regs[29] -= sizeof(struct Trapframe);
	//将这个tf的值保存为我们传入的异常现场tf, 相当于把这个异常现场保存在了用户空间中
	*(struct Trapframe *)tf->regs[29] = tmp_tf;

	if (curenv->env_user_tlb_mod_entry) {
		//将保存的现场传入异常处理函数
		tf->regs[4] = tf->regs[29];
		//再取前一个tf
		tf->regs[29] -= sizeof(tf->regs[4]);
		// Hint: Set 'cp0_epc' in the context 'tf' to 'curenv->env_user_tlb_mod_entry'.
		/* Exercise 4.11: Your code here. */
		tf->cp0_epc = curenv->env_user_tlb_mod_entry;
	} else {
		panic("TLB Mod but no user handler registered");
	}
}
```

看mips的内容真的有点头疼, 因为基础不好, 对于mips的栈的内容理解并不是很好.

## 页写入异常全过程

![image.png](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/20230507195606.png)
### tlb_mod处理函数的注册
首先, 只有fork产生的函数(及父函数)需要用到COW机制, 所以我们在fork时对`env_user_tlb_mod_entry`进行注册.
```c
//fork.c
int fork(void) {
	u_int child;
	u_int i;
	extern volatile struct Env *env;
	//首先对父进程的env_user_tlb_mod_entry进行赋值
	if (env->env_user_tlb_mod_entry != (u_int)cow_entry) {
		try(syscall_set_tlb_mod_entry(0, cow_entry));
	}
	child = syscall_exofork();
	if (child == 0) {
		//debugf("\ntest_2\n");
		env = envs + ENVX(syscall_getenvid());
		//debugf("%d\n" ,env->env_id);
		return 0;
	}
	for(i=0; i<VPN(USTACKTOP); i++)
	 	if (*(vpd+ (i>>10)) & PTE_V && *(vpt + i) & PTE_V)
			duppage(child, i);
	//对子进程的env_user_tlb_mod_entry进行赋值
	try(syscall_set_tlb_mod_entry(child, cow_entry));
	try(syscall_set_env_status(child, ENV_RUNNABLE));
	return child;
}
```

### env_user_tlb_mod_entry在何时被使用
当查到某页的标志位COW为真而不可写的时候会触发tlb_mod异常.
1. 触发异常后我们将直接进入`exc_gen_entry`
2. `exc_gen_entry`将我们带去在`kern/genex.S`中注册的`handle_mod`中
3. 在`handle_mod`中, 我们申请TF_SIZE+8大小的栈帧,将此时的sp赋给$a0寄存器传入`do_tlb_mod` , 并将sp-8回到TF_SIZE的位置, 接着进入函数`do_tlb_mod`处理异常
```c
//handle_mod
NESTED(handle_\exception, TF_SIZE + 8, zero)
    move    a0, sp
    addiu   sp, sp, -8
    jal     \handler
    addiu   sp, sp, 8
    j       ret_from_exception
END(handle_\exception)
.endm
```
4. 让我们仔细看看`do_tlb_mod`函数:
```c
void do_tlb_mod(struct Trapframe *tf) {
	struct Trapframe tmp_tf = *tf;
	//此时我们在内核态, 按理来说是不会在用户态的栈空间里的
	//如果我们在的话, 说明发生了嵌套异常, 那么不用覆盖之前的异常栈
	//如果此时在用户的栈空间之外(内核态), 那么将sp设置为用户异常栈栈顶
	//这样的话, 我们可以将tf保存在用户的栈空间
	if (tf->regs[29] < USTACKTOP || tf->regs[29] >= UXSTACKTOP) {
		tf->regs[29] = UXSTACKTOP;
	}
	//sp减去一个tf的大小,相当于新申请一个tf
	tf->regs[29] -= sizeof(struct Trapframe);
	//将这个tf的值保存为我们传入的异常现场tf, 相当于把这个异常现场保存在了用户空间中
	*(struct Trapframe *)tf->regs[29] = tmp_tf;

	if (curenv->env_user_tlb_mod_entry) {
		//将保存的现场作为参数传入异常处理函数
		tf->regs[4] = tf->regs[29];
		//减去tf->regs[4]的大小, 腾出一个新的tf的空间
		tf->regs[29] -= sizeof(tf->regs[4]);
		//进入函数的env_user_tlb_mod_entry函数, 即cow_entry
		tf->cp0_epc = curenv->env_user_tlb_mod_entry;
	} else {
		panic("TLB Mod but no user handler registered");
	}
}

```

### 研究cow_entry
进入了cow_entry后又发生了什么呢?
这里比较简单, 就是复制出了新页赋给子进程, 并取消了标记.
最后保存了现场, 返回.
```c
static void __attribute__((noreturn)) cow_entry(struct Trapframe *tf) {
	u_int va = tf->cp0_badvaddr;
	u_int perm;
	/* Step 1: Find the 'perm' in which the faulting address 'va' is mapped. */
	perm = *(vpt + VPN(va)) & 0xfff;
	if(!(perm&PTE_COW))
		user_panic("Perm doesn't have PTE_COW");
	/* Step 2: Remove 'PTE_COW' from the 'perm', and add 'PTE_D' to it. */
	perm |= PTE_D;
	perm &= (~PTE_COW);
	/* Step 3: Allocate a new page at 'UCOW'. */
	syscall_mem_alloc(0, UCOW, perm);
	/* Step 4: Copy the content of the faulting page at 'va' to 'UCOW'. */
	memcpy(UCOW, PTE_ADDR(va), BY2PG);
	// Step 5: Map the page at 'UCOW' to 'va' with the new 'perm'.
	syscall_mem_map(0, UCOW, 0, (va&(~0xfff)), perm);
	// Step 6: Unmap the page at 'UCOW'.
	syscall_mem_unmap(0, UCOW);
	// Step 7: Return to the faulting routine.
	//注意这里, 我们把tf传入了sys_set_trapframe, 如果tf不在用户空间就会报错
	int r = syscall_set_trapframe(0, tf);
	user_panic("syscall_set_trapframe returned %d", r);
}
```


# 补充
## 如何注册一个系统调用
我们首先跟踪一个系统调用函数运行的路径:
1. 用户进程的某个函数调用了用户空间的`syscall_*`函数
2. `syscall_*`函数调用了msyscall函数, 系统陷入内核态
3. 内核态中将异常分发到 `handle_sys` 函数，将系统调用所需要的信息传递入内核
4. 内核取得信息，执行对应的内核空间的系统调用函数`sys_*`
5. 系统调用完成, 返回用户态, 同时将返回值传递回用户态
6. 系统调用完成

所以为了注册一个系统调用, 我们要修改以下文件:
1. 在`user/include/lib.h`中添加用户态的系统调用定义`syscall_*`
2. 在`user/lib/syscall_lib.c`中完成`syscall_*`函数体
3. 在`include/syscall.h`中注册系统调用
4. 在`kern/syscall_all.c`中的`syscall_table[MAX_SYSNO]`数组中注册系统调用, 并完成系统调用函数 

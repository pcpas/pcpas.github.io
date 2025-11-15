---
title: OS_lab2_内存管理
date: 2023-04-04 16:16:20
excerpt: BUAA_OS_Lab2总结
categories: [技术, 操作系统]
tags: [内存管理]
topic: os
---

# 内存管理
## 映射
- kseg0 -> 最高位置0(内核代码与数据结构)
- kseg1 -> 最高三位置0(外设)
- kuseg -> TLB访存(用户空间)

## 建立过程
>现在还没有建立内存管理机制，那么是如何操作内存呢？ 
>**这是因为在 Lab2 中通过 kseg0 段地址直接操作物理内存。虽然实验一直在操作如 0x80xxxxxx 的虚拟地址，由于 kseg0 段的性质，操作系统可以通过这一段地址来直接操作物理内存，从而逐步建立内存管理机制.** 例如当写虚拟地址 0x80012340 时，由于 kseg0 段的性质，事实上在写物理地址 0x12340。 
>可以回顾前面的“虚拟地址映射到物理地址”小节，其中有提到，在 kuseg 段的虚拟地址才需要通过 MMU 的转换获得物理地址后访存。在 Lab2 中所做的内存管理工作，正是使将来在 kuseg 段的用户程序能够正常工作。

![image-20230404172009689](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/image-20230404172009689.png)


### 初始化页控制块的控制系统(借用alloc和pages实现)
![image-20230404172053973](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/image-20230404172053973.png)
 补充: alloc函数用于在操作系统尚未建立分页系统时进行物理空间的分配
![image-20230404172117022](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/image-20230404172117022.png)

### 在页控制块管理物理内存的基础上完善函数
 - `page_alloc(struct Page** pp)`: 获取一页物理内存
    ![image-20230404172156774](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/image-20230404172156774.png)
  - `page_decref(struct Page *pp)` : 令pp对应的页控制块引用次数减少1.
  - `page_free(struct Page *pp)`: 释放空闲(assert)页面
### 在页控制块管理物理内存的基础上建立两级页表
  - 两级页表解读:
      ![image-20230404172404825](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/image-20230404172404825.png)
   - 两级页表下的检索机制:
   - `int pgdir_walk(Pde *pgdir, u_long va, int create, Pte **ppte)` 
	   给定一个页目录基址`pgdir`, 一个虚拟地址 `va`, 返回该虚拟地址对应的页表项.
	   利用这个函数,我们可以获得任何虚拟地址对应的页表项,从而获得虚拟地址和物理地址的映射.
	![image-20230404172425991](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/image-20230404172425991.png)
   - `int page_insert(Pde *pgdir, u_int asid, struct Page *pp, u_long va, u_int perm)`
	  ![image-20230404172444819](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/image-20230404172444819.png)
  - `struct Page * page_lookup(Pde *pgdir, u_long va, Pte **ppte)`  ：查找
  - `void page_remove(Pde *pgdir, u_int asid, u_long va)`  ： 移除
自此，二级页表的内存管理机制建立。
### 两级页表例子
为了加深对于二级页表机制的理解, 附上两道例题:

>1. 如果当前进程的页目录物理基地址、页目录和相应页表内容如图下所示，请描述访问0x00803004时系统进行地址转换的过程，如可行给出最终访存获取到的数据。

![Pasted image 20230402181110](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/Pasted%20image%2020230402181110.png)
![Pasted image 20230402174714](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/Pasted%20image%2020230402174714.png)

答:
0x00803004转换为二进制为:0000000010 0000000011 000000000100
页目录号为2,查询得0x5001, 有效位为1, 页表物理地址为0x5000
二级页表号为3, 查询0x5000处页表的第三项, 得0x20001, 有效位为1, 页面物理地址为0x20000.
页内偏移为4,故访问到的数据为0x326001.

>2. 在现代的 64 位系统中，提供了 64 位的字长，但实际上不是 64 位页式存储系统。假设在 64 位系统中采用三级页表机制，页面大小 4KB。由于 64 位系统中字长为 8B，且页目录也占用一页，因此页目录中有 512 个页目录项，因此每级页表都需要 9 位。 因此在 64 位系统下，总共需要 3 × 9 + 12 = 39 位就可以实现三级页表机制，并不需要 64 位。 现考虑上述 39 位的三级页式存储系统，虚拟地址空间为 512 GB，若三级页表的基地址为 PTbase，请计算： 
>- 三级页表页目录的基地址。 
>- 映射到页目录自身的页目录项（自映射）。

(1) 答: 
三级页表的基地址为PTBase，因为128M个页表项和512GB空间是线性映射的，所以PTBase这一地址对应的应为第(PTBase>>12)个页表项(512个项x8B). 对应的是第一个二级页目录项. 因此,二级页目录基址为: PTBase + (PTBase>>12)\*8 =  PTBase + PTBase>>9 
一二级的映射可以参照二三级的映射用一样的方法计算.
(PTBase>>9)这一地址对应的应为第(PTBase>>9)>>12个页表项,对应的是第一个一级页目录项. 偏移为:((PTBase>>9)>>12)<<3. 因此, 一级页目录基址为: PTBase + (PTBase>>9) + (PTBase>>18)
(2)答:
那么, 自映射到自身的就是再重复一次, 为: PTBase + (PTBase>>9) + (PTBase>>18) + (PTBase>>27) 

## 虚拟地址, 物理地址和内核虚拟地址

程序运行时,在建立了虚拟地址空间后,并没有分配实际的物理内存,而是当进程需要实际访问内存资源的时候就会由内核的请求分页机制产生缺页中断,这时才会建立虚拟地址和物理地址的映射,调入物理内存页.通过这种方式,就能够保证我们的物理内存只有在实际使用时才进行分配,避免了内存浪费的问题.
虚拟地址空间的内部又被划分为**用户空间**与**内核空间**.
内核空间即进程陷入内核态后才能访问的空间.**虽然每个进程都具有自己独立的虚拟地址空间,但是这些虚拟地址空间中的内核空间,其实关联的都是同一块物理内存**.
![Pasted image 20230402170006](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/Pasted%20image%2020230402170006.png)
体现在MOS中为:
![Pasted image 20230319211651](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/Pasted%20image%2020230319211651.png)

- 内核空间
	- 若虚拟地址处于 0x80000000~0x9fffffff (kseg0)，则将虚拟地址的最高位置 0 得到物理地址，通过 cache 访存。这一部分用于存放内核代码与数据结构。
	- 若虚拟地址处于 0xa0000000~0xbfffffff (kseg1)，则将虚拟地址的最高 3 位置 0 得到物理地址，不通过 cache 访存。这一部分可以用于映射外设(使用 MMIO(Memory-Mapped I/O)技术)。 
- 用户空间	
	 - 若虚拟地址处于 0x00000000~0x7fffffff (kuseg)，则需要通过 TLB 来获取物理地址, 通过 cache 访存。
>关于kseg2: 0xC0000000-0xFFFFFFFF(1 GB)：这段地址只能在内核态下使用并且需要 MMU 中 TLB 将虚拟地址转换为物理地址。对这段地址的存取都会通过 cache。目前的实验中还没有涉及到这部分, 应该对应的是LINUX中的高端内存映射区, 暂时先不关注.

![Pasted image 20230402170735](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/Pasted%20image%2020230402170735.png)

在LAB2包括后续LAB3的实验过程中, 经常会困惑的一点是: 这个地址到底是虚拟地址, 物理地址还是内核虚拟地址? 这些地址之间有什么区别和联系? 能不能做一个归纳和分类? 没错, 现在就是在解答这个疑惑.

### 虚拟地址(VA)

>CPU 运行程序时通过访存指令发出访存请求，进行内存读写操作。在计算机组成原理等硬件实验中, CPU 通常直接发送物理地址，这是为了简化内存操作，让大家关注 CPU 内部的计算与控制逻辑。
>而在实际程序中，访存、跳转等指令以及用于取指的 PC 寄存器中的访存目标地址都是虚拟地址。我们编写的 C 程序中也经常通过对指针解引用来进行访存，其中指针的值也会被视为虚拟地址，经过编译后生成相应的访存指令。

也就是说, 不管是向MIPS传入的访存地址, 还是在C程序中使用的指针(只要是需要使用的指针, 比如说要取指针指向的值, 或者要传入memset函数等)都是虚拟地址.

### 内核虚拟地址(KVA)
内核虚拟地址也是虚拟地址的一部分, 但是由于其特殊性, 可以单独拎出来.
- 虽然每个进程都具有自己独立的虚拟地址空间,但是这些虚拟地址空间中的内核空间,其实关联的都是同一块物理内存
- MOS 中采用 PADDR 与 KADDR 这两个宏就可以对位于 kseg0 的虚拟地址和对应的物理地址进行转换.
	很多常用的宏都是在操作kva, 比如说`PADDR`, `page2kva`等, 其本质原因是因为这些控制块, 结构等都在内核态中操作.
- 值得注意的是, 页表都存在实际的物理内存中, 但是, 对页表进行操作时处于内核态，因此使用宏 KADDR 获得其位于 kseg0中的虚拟地址.


### 物理地址
MIPS R3000 发出的地址均为虚拟地址，因此如果程序想访问某个物理地址，需要通过映射到该物理地址的虚拟地址来访问。
通常来说, 在页表映射时会容易出现物理地址, 此时通常都要将物理地址转换为虚拟地址再进行操作.

## 常用的宏和函数

```c
#define PGSHIFT 12 //页面的大小: 4K
#define PADDR(kva) //返回虚拟地址kva(必须位于kseg0)所对应的物理地址
#define KADDR(pa)  //返回物理地址所对应的虚拟地址kva(必须位于kseg0)

//Page指针转为该Page对应的物理页面的虚拟地址
static inline u_long page2kva(struct Page *pp)
//Page指针获得该Page对应的物理页面的物理地址
static inline u_long page2pa(struct Page *pp)
//物理地址转为Page指针
static inline struct Page *pa2page(u_long pa)
//Page指针获得该Page是第几页
static inline u_long page2ppn(struct Page *pp)

//解析VA的宏
#define PDX(va) ((((u_long)(va)) >> 22) & 0x03FF) //获取页目录偏移
#define PTX(va) ((((u_long)(va)) >> 12) & 0x03FF) //获取二级页表偏移
#define PTE_ADDR(pte) ((u_long)(pte) & ~0xFFF)    //获取该pte对应的物理页面地址
```

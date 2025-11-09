---
title: OS_lab1_内核启动
date: 2023-03-20 16:16:24
excerpt: BUAA_OS_Lab1总结
tags: [操作系统,内存管理]
categories: [技术,操作系统]
---

# 多个C文件的连接和编译
## 基本流程
![](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/Pasted image 20230319200455.png)
 多个C语言文件编译成为一个可执行文件,首先通过`gcc -c <filename>`将.c文件编译为.o二进制文件, 再通过链接的方式将多个.o文件链接在一起(此时最后每条指令的地址也因此确定), 生产最后的可执行文件.

##  ELF文件 - 深入探究编译链接
链接器通过哪些信息来链接多个目标文件呢？---- 目标文件(比如说: elf文件)

在目标文件中，记录了代码各个段的具体信息。链接器通过这些信息来将目标文件链接到一起。而 ELF(Executable and Linkable Format) 正是 Unix 上常用的一种 目标文件格式。其实，不仅仅是目标文件，可执行文件也是使用 ELF 格式记录的。这一点通过 ELF 的全称也可以看出来。

**.o 文件就是 ELF 所包含的三种文件类型中的一种**，称为可重 定位 (relocatable) 文件，其它两种文件类型分别是可执行 (executable) 文件和共享对象 (shared object) 文件，这两种文件都需要链接器对可重定位文件进行处理才能生成。

ELF文件结构:
![](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/Pasted image 20230319201114.png)
段头表（或程序头表，program header table），主要包含程序中各个段（segment）的信息， **段的信息需要在运行时刻使用**。
节头表（section header table），主要包含程序中各个节（section）的信息，**节的信息需要 在程序编译和链接的时候使用**。(因此在这里我们主要关心节头表)
段头表和节头表指向了同样的地方，这意味着两者只是程序数据的两种视图:

1. 组成可重定位文件，参与可执行文件和可共享文件的链接。此时使用节头表。
2. 组成可执行文件或者可共享文件，在运行时为加载器提供信息。此时使用段头表。

现在来逐一分析ELF文件内容:
### ELF文件头
ELF 的文件头，就是一个存了关于 ELF 文件信息的结构体。 首先，结构体中存储了 ELF 的魔数，以验证这是一个有效的 ELF。当我们验证了这是个 ELF 文件之后，便可以通过 ELF 头中提供的信息，进一步地解析 ELF 文件了。
存放了`Elf32_Ehdr`, `Elf32_Shdr`和`Elf32_Phdr`三种数据结构.
Ehdr : elf文件结构 ; **Shdr: 节头表结构** ; Phdr: 段头表结构
```c
typedef struct {
    unsigned char e_ident[EI_NIDENT]; /* Magic number and other info */
    Elf32_Half e_type;        /* Object file type */
    Elf32_Half e_machine;         /* Architecture */
    Elf32_Word e_version;         /* Object file version */
    Elf32_Addr e_entry;       /* Entry point virtual address */
    Elf32_Off e_phoff;        /* Program header table file offset */
    Elf32_Off e_shoff;        /* Section header table file offset */
    Elf32_Word e_flags;       /* Processor-specific flags */
    Elf32_Half e_ehsize;          /* ELF header size in bytes */
    Elf32_Half e_phentsize;       /* Program header table entry size */
    Elf32_Half e_phnum;       /* Program header table entry count */
    Elf32_Half e_shentsize;       /* Section header table entry size */
    Elf32_Half e_shnum;       /* Section header table entry count */
    Elf32_Half e_shstrndx;        /* Section header string table index */
} Elf32_Ehdr;
```
可以看到, ehdr中包含了很多关键信息,如`e_phoff`, `e_shoff`. 
```c
Elf32_Ehdr *ehdr = (Elf32_Ehdr *)binary;
```
我们在读取elf文件一开始将ELF文件的入口作为一个`Elf32_Ehdr *`类型的指针, 这样做的好处是在地址空间自动的划出一个`sizeof(Elf32_Ehdr)`大小的区域, 这片区域自然就是ELF文件头.
### 读取ELF文件
```c
int readelf(const void *binary, size_t size) {
    Elf32_Ehdr *ehdr = (Elf32_Ehdr *)binary;
    // Check whether `binary` is a ELF file.
    if (!is_elf_format(binary, size)) {
        fputs("not an elf file\n", stderr);
        return -1;
    }
    // Get the address of the section table, the number of section headers and the size of a section header.
    const void *sh_table;
    Elf32_Half sh_entry_count;
    Elf32_Half sh_entry_size;
    /* Exercise 1.1: Your code here. (1/2) */
    //这里主要考察的就是有没有去读elf.h文件,如果读了就知道这些数据全部在文件头中给出
    sh_table = binary + ehdr->e_shoff;
    sh_entry_count = ehdr->e_shnum;
    sh_entry_size = ehdr->e_shentsize;
    // For each section header, output its index and the section address.
    //注意,Elf32_Shdr这个结构体，是一个节头部表的一个条目（即一个表项），而不是一个一整个段头部表, 起到存储对应的节的信息. 这里我们将对应的节的地址取出
    // The index should start from 0.
    for (int i = 0; i < sh_entry_count; i++) {
        const Elf32_Shdr *shdr;
        unsigned int addr;
        /* Exercise 1.1: Your code here. (2/2) */
        addr = ((Elf32_Shdr *)(sh_table + i*sh_entry_size))->sh_addr;
        printf("%d:0x%x\n", i, addr);
    }
    return 0;
}
```

## 关于ELF32和ELF64文件
我们编写的 readelf 程序是不能解析 readelf 文件本身的，而我们刚才介绍的系统工具 readelf 则可以解析，这是为什么呢？
![](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/Pasted%20image%2020230319210414.png)
可以发现, readelf文件是一个ELF64类型的文件, 而我们编写的`readelf.c`只能读取elf32类型的文件.
对比官方`elf.h`和MOS的`elf.h`

```c
//linux elf.h
typedef struct elf64_hdr {
  unsigned char	e_ident[EI_NIDENT];	 //魔数 7F+’E’+’L’+’F’
  Elf64_Half e_type;				 //文件类型 
  Elf64_Half e_machine;				 //机器类型
  Elf64_Word e_version;				 //文件版本
  Elf64_Addr e_entry;			     //进程开始的虚地址，即系统将控制转移的位置
  Elf64_Off e_phoff;				 //program header table的文件偏移
  Elf64_Off e_shoff;		         //section header表的文件偏移
  Elf64_Word e_flags;				 //处理器相关的标志
  Elf64_Half e_ehsize;				 //ELF文件头的长度
  Elf64_Half e_phentsize;			 //program header表中的入口的长度
  Elf64_Half e_phnum;				 //program header表中的入口数目
  Elf64_Half e_shentsize;			 //section header表中的入口的长度
  Elf64_Half e_shnum;				 //section header表中的入口数目
  Elf64_Half e_shstrndx;			 //section名表的位置，指出在section header表中的索引
} Elf64_Ehdr;

typedef struct elf64_phdr {
  Elf64_Word p_type;		//段的类型
  Elf64_Word p_flags;       //段标志
  Elf64_Off p_offset;		//段偏移
  Elf64_Addr p_vaddr;		//段虚地址
  Elf64_Addr p_paddr;		//段物理地址
  Elf64_Xword p_filesz;		//段大小
  Elf64_Xword p_memsz;		//段在内存的占用
  Elf64_Xword p_align;		//段对齐
} Elf64_Phdr;

typedef struct elf64_shdr {
  Elf64_Word sh_name;		//段名，值为符号表索引
  Elf64_Word sh_type;		//类别
  Elf64_Xword sh_flags;		//程执行时的特性
  Elf64_Addr sh_addr;		//进程的内存映像中出现，则给出开始的虚地址
  Elf64_Off sh_offset;		//偏移
  Elf64_Xword sh_size;		//大小
  Elf64_Word sh_link;		//相对其他段的索引
  Elf64_Word sh_info;		//信息
  Elf64_Xword sh_addralign;	//对齐
  Elf64_Xword sh_entsize;	//
} Elf64_Shdr;

typedef struct elf64_sym {
  Elf64_Word st_name;		//符号名
  unsigned char	st_info;	//符号类型
  unsigned char	st_other;	//其他
  Elf64_Half st_shndx;		//段索引
  Elf64_Addr st_value;		//符号地址值
  Elf64_Xword st_size;		//符号大小
} Elf64_Sym;

```
```c
//MOS elf.h
typedef struct {

    unsigned char e_ident[EI_NIDENT]; /* Magic number and other info */
    Elf32_Half e_type;        /* Object file type */
    Elf32_Half e_machine;         /* Architecture */
    Elf32_Word e_version;         /* Object file version */
    Elf32_Addr e_entry;       /* Entry point virtual address */
    Elf32_Off e_phoff;        /* Program header table file offset */
    Elf32_Off e_shoff;        /* Section header table file offset */
    Elf32_Word e_flags;       /* Processor-specific flags */
    Elf32_Half e_ehsize;          /* ELF header size in bytes */
    Elf32_Half e_phentsize;       /* Program header table entry size */
    Elf32_Half e_phnum;       /* Program header table entry count */
    Elf32_Half e_shentsize;       /* Section header table entry size */
    Elf32_Half e_shnum;       /* Section header table entry count */
    Elf32_Half e_shstrndx;        /* Section header string table index */
} Elf32_Ehdr;

//余下相似,不在复制
```
但是因为readelf.c源代码几乎全部使用宏, 感觉如果想写一个解析elf64的readelf.c几乎不需要什么改动, 很疑惑(感觉会在这里出题呀)?

## 可执行文件的运行
### 内核在哪里:内存布局
只要我们能够将内核加载到正确的位置上，我们的内核就应该 可以运行起来。
思考到这里，我们又发现了几个重要的问题。 
1. 什么叫做正确的位置？到底放在哪里才叫正确。 
2. 哪个段被加载到哪里是记录在编译器编译出来的 ELF 文件里的，我们怎么才能修改它呢？

关于内核应该被加载到哪里: kseg0
![](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/Pasted%20image%2020230319211651.png)

```
/*
 o     4G ----------->  +----------------------------+------------0x100000000
 o                      |       ...                  |  kseg2
 o      KSEG2    -----> +----------------------------+------------0xc000 0000
 o                      |          Devices           |  kseg1
 o      KSEG1    -----> +----------------------------+------------0xa000 0000
 o                      |      Invalid Memory        |   /|\
 o                      +----------------------------+----|-------Physical Memory Max
 o                      |       ...                  |  kseg0
 o      KSTACKTOP-----> +----------------------------+----|-------0x8040 0000-------end
 o                      |       Kernel Stack         |    | KSTKSIZE            /|\
 o                      +----------------------------+----|------                |
 o                      |       Kernel Text          |    |                    PDMAP
 o      KERNBASE -----> +----------------------------+----|-------0x8001 0000    |
 o                      |      Exception Entry       |   \|/                    \|/
 o      ULIM     -----> +----------------------------+------------0x8000 0000-------
 o                      |         User VPT           |     PDMAP                /|\
 o      UVPT     -----> +----------------------------+------------0x7fc0 0000    |
 o                      |           pages            |     PDMAP                 |
 o      UPAGES   -----> +----------------------------+------------0x7f80 0000    |
 o                      |           envs             |     PDMAP                 |
 o  UTOP,UENVS   -----> +----------------------------+------------0x7f40 0000    |
 o  UXSTACKTOP -/       |     user exception stack   |     BY2PG                 |
 o                      +----------------------------+------------0x7f3f f000    |
 o                      |                            |     BY2PG                 |
 o      USTACKTOP ----> +----------------------------+------------0x7f3f e000    |
 o                      |     normal user stack      |     BY2PG                 |
 o                      +----------------------------+------------0x7f3f d000    |
 a                      |                            |                           |
 a                      ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~                           |
 a                      .                            .                           |
 a                      .                            .                         kuseg
 a                      .                            .                           |
 a                      |~~~~~~~~~~~~~~~~~~~~~~~~~~~~|                           |
 a                      |                            |                           |
 o       UTEXT   -----> +----------------------------+------------0x0040 0000    |
 o                      |      reserved for COW      |     BY2PG                 |
 o       UCOW    -----> +----------------------------+------------0x003f f000    |
 o                      |   reversed for temporary   |     BY2PG                 |
 o       UTEMP   -----> +----------------------------+------------0x003f e000    |
 o                      |       invalid memory       |                          \|/
 a     0 ------------>  +----------------------------+ ----------------------------
 o
*/
```

在发现了内核的正确位置后，我们只需要想办法让内核被加载到那里就可以了。
编译器在生成 ELF 文件时就已经记录了各节所需要被加载到的位置。最终的可执行文件实际上是由链接器产生的（它将多个目标文件链接 产生最终可执行文件）。因此，我们所需要做的，就是控制链接器的链接过程。
### Linker Script
Linker Script 中记录了各个节应该如何映射到段，以及各个段应该被加载到的位置.
在链接过程中，目标文件被 看成节的集合，并使用节头表来描述各个节的组织。
换句话说，节记录了在链接过程中所需要的必要信息。
其中最为重要的三个节为.text、.data、.bss。
这三个节的意义是必须要掌握的：
.text 保存可执行文件的操作指令。 
.data 保存已初始化的全局变量和静态变量。
.bss 保存未初始化的全局变量和静态变量。

这里比较简单,略过~

自此, 内核就成功被装在内存中了.

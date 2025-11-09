---
title: OS_lab5_文件系统
date: 2023-05-22 13:03:39
excerpt: BUAA_OS_Lab5总结
tags: [操作系统,内存管理]
categories: [技术,操作系统]
---

# 文件系统概述
广义上，一切带标识的、在逻辑上有完整意义的字节序列都可以称为“文件”。文件系统将外部设备中的资源抽象为文件，从而可以统一管理外部设备，实现对数据的存储、组织、访问和修改等操作。在本实验中，我们拟实现一个精简的文件系统，其中需要对**三种设备**进行统一管理， 即文件设备（file，即狭义的“文件”）、控制台（console）和管道（pipe）。其中，后两者将在下一个实验“管道与 Shell”中进行使用。

下图为我们文件系统的总览：
![image.png](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/20230515141829.png)

还有一张图很好的展示了不同层级的抽象函数调用关系。

![image.png](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/20230522152337.png)

# 外部存储设备驱动
通常，我们需要按一定顺序读写设备寄存器，来实现对外部设备的操作。为了将这种操作转化为具有通用、明确语义的接口，必须实现相应的驱动程序。在本部分，我们将实现 IDE 磁盘的用户态驱动程序，该驱动程序将通过系统调用的方式陷入内核，对磁盘镜像进行读写操作。
MOS已经在`kern/console.c`中使用MMIO技术为我们实现了简单的console驱动程序, 我们的实验里需要我们实现IDE 磁盘驱动，也就是下图的部分：
![image.png](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/20230515141939.png)

相比于console驱动程序, IDE的驱动程序更加复杂, 并且本次要编写的驱动程序完全运行在用户空间。
## 内存映射 I/O (MMIO)
几乎每一种 外设都是通过读写设备上的寄存器来进行数据通信，外设寄存器也称为 I/O 端口，主要用来访问 I/O 设备。外设寄存器通常包括控制寄存器、状态寄存器和数据寄存器，这些寄存器被映射到**指定的物理地址空间**。

IDE 磁盘驱动程序位于用户空间，用户态进程若是直接 读写内核虚拟地址将会由处理器引发一个地址错误（ADEL/S）。所以对于设备的读写必须通过 系统调用来实现。
因此我们引入`sys_write_dev` 和 `sys_read_dev` 两个系统调用:
```c
//这里使用定义的宏也许会更好? 
//但是因为没有引用头文件并不能使用...
static inline int is_illegal_pa_range(u_int pa, u_int len) {
	if (len <= 0) 
		return 1;
	if(pa >= 0x10000000 && pa + len <= 0x10000000 + 0x20)
		return 0;
	if(pa >= 0x13000000 && pa + len <= 0x13000000 + 0x4200)
		return 0;
	if(pa >= 0x15000000 && pa + len <= 0x15000000 + 0x200)
		return 0;
	return 1;
}

int sys_write_dev(u_int va, u_int pa, u_int len) {
	/* Exercise 5.1: Your code here. (1/2) */
	if(is_illegal_va_range(va, len))
		return -E_INVAL;
	if(is_illegal_pa_range(pa, len))
		return -E_INVAL;
	//注意地址转换
	memcpy(pa+KSEG1, va, len);
	return 0;
}

int sys_read_dev(u_int va, u_int pa, u_int len) {
	/* Exercise 5.1: Your code here. (2/2) */
	if(is_illegal_va_range(va, len))
		return -E_INVAL;
	if(is_illegal_pa_range(pa, len))
		return -E_INVAL;
	//注意地址转换
	memcpy(va, pa+KSEG1, len);
	return 0;
}
```
通过这两个系统调用，我们就可以对IDE磁盘映射到内核空间的I/O寄存器进行读写了。
## IDE磁盘
>下面简单介绍与磁盘相关的基本知识，首先是几个基本概念： 
>1.  扇区 (sector): 磁盘盘片被划分成很多扇形的区域，这些区域叫做扇区。**扇区是磁盘执行读写操作的单位**，一般是 **512 字节**。扇区的大小是一个磁盘的硬件属性。 
>2.  磁道 (track): 盘片上以盘片中心为圆心，不同半径的同心圆。 
>3.  柱面 (cylinder): 硬盘中，不同盘片相同半径的磁道所组成的圆柱面。 
>4.  磁头 (head): 每个磁盘有两个面，每个面都有一个磁头。当对磁盘进行读写操作时，磁头在盘片上快速移动。

![image.png](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/20230515144052.png)
接下来我们看看IDE内核的I/O寄存器映射:
![image.png](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/20230515144340.png)
通过系统调用`sys_write_dev` 和 `sys_read_dev`,我们可以读写寄存器的值,从而实现对IDE磁盘的读写。


# 文件系统结构
## 磁盘空间的基本布局
![image.png](https://zzq-typora-picgo.oss-cn-beijing.aliyuncs.com/20230519145425.png)
不同于扇区，磁盘块是一个**虚拟概念**，是操作系统与磁盘交互的最小单位；操作系统将相邻的扇区组合在一起，形成磁盘块进行整体操作。
磁盘块的大小由操作系统决定，一般由 2 的 n 次方个扇区构成。

MOS 操作系统把磁盘最开始的一个磁盘块 (4096 字节) 当作**引导扇区和分区表**使用。接下来的一个磁盘块作为**超级块 (Super Block)**，用来描述文件系统的基本信息。
```c
struct Super 
{ 
	u_int s_magic; // Magic number: FS_MAGIC 
	u_int s_nblocks; // Total number of blocks on disk
	struct File s_root; // Root directory node 
};
```

## 磁盘资源管理
在文件系统中，我们使用**位图 (Bitmap) 法**来管理空闲的磁盘资源，用一个二进制位 bit 标识磁盘中的一个磁盘块的使用情况（1 表示空闲，0 表示占用）。
首先来梳理一下定义:
```c
//disk->以磁盘块为单位
struct Block {
    uint8_t data[BY2BLK];
    uint32_t type;
} disk[NBLOCK];
//需要多少block来装位图(天花板除)
nbitblock = (NBLOCK + BIT2BLK - 1) / BIT2BLK;
//初始化时,标记为空闲
memset(disk[2 + i].data, 0xff, BY2BLK);
//但是如果最后有一些多余的bit,全部标记为占用
memset(disk[2 + (nbitblock - 1)].data + diff, 0x00, BY2BLK - diff);
```

```c
//bitmap->将磁盘中的位图读入文件系统(通过read_bitmap)
//bitmap数组的一个成员可以存放32位
uint32_t *bitmap;
//Mark a block as free in the bitmap
bitmap[blockno / 32] |= (1 << (blockno % 32));
```

##  文件的描述和管理
操作系统要想管理一类资源，就得有相应的数据结构。对于描述和管理文件来说，一般使用 文件控制块（File 结构体）。
此处需要注意, 不论是f_direct数组的内容, 还是f_indirect, 都是uint32_t, 即这不是传统意义上的指针, 而是指disk数组使用的bno.
```c
//size为256B
struct File {
    char f_name[MAXNAMELEN]; // filename
    uint32_t f_size;     // file size in bytes
    uint32_t f_type;     // file type
    uint32_t f_direct[NDIRECT];
    uint32_t f_indirect;
    struct File *f_dir; // the pointer to the dir where this file is in, valid only in memory.
    char f_pad[BY2FILE - MAXNAMELEN - (3 + NDIRECT) * 4 - sizeof(void *)];
} __attribute__((aligned(4), packed));
```

## 块缓存
我理解的块缓存: 因为文件进程的需要, 我们希望能使用虚拟地址直接实现对磁盘的读写, 但是实际上我们只能调用磁盘驱动非常复杂地进行操作.
解决办法就是将磁盘直接全部映射到文件服务进程虚拟空间的某处, 如果读取就可以直接读取, 如果发生了修改就将缓存写回磁盘, 从而提高了效率.

# 梳理调用关系
## ide磁盘
ide磁盘中的寄存器会映射在内核空间中, 允许我们在此基础上建立驱动读写ide.
## 内核
内核通过接受来自驱动程序的`sys_read_dev`和`sys_write_dev`来读写寄存器
## 磁盘驱动--ide.c
在驱动层次, 我们封装了`ide_write`和`ide_read`来读写磁盘, 驱动属于文件进程的一部分, 为文件提供操作磁盘的方法.
## 文件系统底层部分--fs.c
此时, 大多数操作是对进程的块缓存进行操作, 最后再调用驱动对磁盘修改.
下面是fs.c中提供的对缓存区的一些操作函数, 借助这些函数, 我们可以比较轻松地操作缓存区了,此时的操作主要以block为单位:
```c
void *diskaddr(u_int blockno)//查块对应的虚拟地址

int va_is_mapped(void *va)
void *block_is_mapped(u_int blockno)//调用va_is_mapped

int va_is_dirty(void *va)
int block_is_dirty(u_int blockno)//调用va_is_mapped和va_is_dirty

//为va对应的物理页加上PTE_DIRTY权限
//diskaddr, va_is_mapped, va_is_dirty, syscall_mem_map
int dirty_block(u_int blockno)
//将缓存内容写入对应的磁盘块
//block_is_mapped, diskaddr, ide_write
void write_block(u_int blockno)

//将磁盘块写入缓存区,将缓存的va保存在blk中返回
int read_block(u_int blockno, void **blk, u_int *isnew)
//为block申请一个物理页
int map_block(u_int blockno)
//Unmap a disk block in cache(会将缓存写回)
void unmap_block(u_int blockno)
//查询bitmap中block是否空闲(不涉及缓存)
int block_is_free(u_int blockno)
//修改bitmap, 标记某个block为free(没有清空内容)
void free_block(u_int blockno)
//Search in the bitmap for a free block and allocate it.(同时会修改bitmap)
int alloc_block_num(void)
//Allocate a block -- first find a free block in the bitmap, then map it into memory.
//调用alloc_block_num,map_block,free_block
int alloc_block(void)

//  Initialize the file system.
void fs_init(void) {
    read_super();
    check_write_block();
    read_bitmap();
}
```
接下来是一些更上一级的函数,这些函数的特点是以file为单位:
```c
//找到某文件的第filebno个块在整个磁盘的bno的指针*ppdiskbno, 如果alloc=1, 那么会在没有indirect块的时候帮忙建立
int file_block_walk(struct File *f, u_int filebno, uint32_t **ppdiskbno, u_int alloc)
//将diskbno的值设置为某文件的第filebno个块在整个磁盘的bno
int file_map_block(struct File *f, u_int filebno, u_int *diskbno, u_int alloc)
//删掉文件的第filebno个块
int file_clear_block(struct File *f, u_int filebno)
//得到文件的第filebno个块的va
int file_get_block(struct File *f, u_int filebno, void **blk)
//Mark the offset/BY2BLK'th block dirty in file f.
int file_dirty(struct File *f, u_int offset)
//Find a file named 'name' in the directory 'dir'. If found, set *file to it.
int dir_lookup(struct File *dir, char *name, struct File **file)
//在dir下申请一个新的File,并借助file返回
int dir_alloc_file(struct File *dir, struct File **file)
//用path找到对应的File结构体,用pfile返回
int walk_path(char *path, struct File **pdir, struct File **pfile, char *lastelem)
```

在此基础上, 还有几个最能代表这一层次的函数:
```c
//用path找到对应的File结构体,用file返回
int file_open(char *path, struct File **file)
//使用path创建文件
int file_create(char *path, struct File **file)
//文件大小发生变化后, 释放不需要的磁盘块(如果更大则不做处理)
void file_truncate(struct File *f, u_int newsize)
//将file的缓存写入磁盘
void file_flush(struct File *f)
//设置文件的大小
int file_set_size(struct File *f, u_int newsize)
//将所有的缓存写入磁盘
void fs_sync(void)
//关闭文件: 将相关缓存写入
void file_close(struct File *f)
//删除文件:名字清空/磁盘块全部释放
int file_remove(char *path)
```

## 文件系统服务层--serv.c
这一层次的函数主要是对来自其他进程的ipc请求进行处理.
本层次有一个结构体, 用来记录一个打开的文件的基本信息.
```c
struct Open {
    struct File *o_file; // mapped descriptor for open file
    u_int o_fileid;      // file id
    int o_mode;      // open mode
    struct Filefd *o_ff; // va of filefd page
};
// Max number of open files in the file system at once
#define MAXOPEN 1024
#define FILEVA 0x60000000

// initialize to force into data section
struct Open opentab[MAXOPEN] = {{0, 0, 1}};

// Virtual address at which to receive page mappings containing client requests.
#define REQVA 0x0ffff000
```

接下来是一些函数:
```c
//Initialize file system server process.
//值得注意的是每个Open结构体是在初始化的时候确定序号和FileFd的地址
void serve_init(void)
//Allocate an open file.
//o传递Open结构体,返回对应的o_fileid
int open_alloc(struct Open **o)
//Look up an open file for envid.
int open_lookup(u_int envid, u_int fileid, struct Open **po)
```
实际提供服务的函数, 使用ipc_recv接受请求, 其中具体的请求处理都用ipc_send来进行回复.
```c
void serve(void) {
	u_int req, whom, perm;

	for (;;) {
		perm = 0;

		req = ipc_recv(&whom, (void *)REQVA, &perm);

		// All requests must contain an argument page
		if (!(perm & PTE_V)) {
			debugf("Invalid request from %08x: no argument page\n", whom);
			continue; // just leave it hanging, waiting for the next request.
		}

		switch (req) {
		case FSREQ_OPEN:
			serve_open(whom, (struct Fsreq_open *)REQVA);
			break;

		case FSREQ_MAP:
			serve_map(whom, (struct Fsreq_map *)REQVA);
			break;

		case FSREQ_SET_SIZE:
			serve_set_size(whom, (struct Fsreq_set_size *)REQVA);
			break;

		case FSREQ_CLOSE:
			serve_close(whom, (struct Fsreq_close *)REQVA);
			break;

		case FSREQ_DIRTY:
			serve_dirty(whom, (struct Fsreq_dirty *)REQVA);
			break;

		case FSREQ_REMOVE:
			serve_remove(whom, (struct Fsreq_remove *)REQVA);
			break;

		case FSREQ_SYNC:
			serve_sync(whom);
			break;

		default:
			debugf("Invalid request code %d from %08x\n", whom, req);
			break;
		}

		syscall_mem_unmap(0, (void *)REQVA);
	}
}
```

定义的请求结构体
```c
//保存在fsreq.h文件中
#define FSREQ_OPEN 1
#define FSREQ_MAP 2
#define FSREQ_SET_SIZE 3
#define FSREQ_CLOSE 4
#define FSREQ_DIRTY 5
#define FSREQ_REMOVE 6
#define FSREQ_SYNC 7

struct Fsreq_open {
	char req_path[MAXPATHLEN];
	u_int req_omode;
};

struct Fsreq_map {
	int req_fileid;
	u_int req_offset;
};

struct Fsreq_set_size {
	int req_fileid;
	u_int req_size;
};

struct Fsreq_close {
	int req_fileid;
};

struct Fsreq_dirty {
	int req_fileid;
	u_int req_offset;
};

struct Fsreq_remove {
	char req_path[MAXPATHLEN];
};
```

至此, 文件系统的部分就基本结束了, 文件系统使用`ipc_recv`来接受其他进程的请求.

## 用户服务的提供者--fd.c
在用户的视角看, 对文件的一切操作其实都是通过文件描述符fd实现的.
因此我们从高到低, 从fd.c来分析MOS是怎么为用户进程提供服务的:
首先是一些值得注意的定义:
```c
/*
内存空间中有专门的地址存放文件描述符，也就是FDTABLE，这里可以看成一个数组，每个Fd占一个Page大小，可以直接用fdnum作为下标获得；此外，文件的具体数据存在FILEBASE部分的内存空间，每个文件的数据最多占PDMAP大小，也可以实现fdnum到文件数据的一一映射（FILEBASE + PDMAP * fdnum）
*/
#define MAXFD 32
#define FILEBASE 0x60000000
#define FDTABLE (FILEBASE - PDMAP)

//MOS中有三种dev, file,console和pipe.
//不同的dev需要绑定有自己的5个基本方法, 以便进行操作
struct Dev {
	int dev_id;
	char *dev_name;
	int (*dev_read)(struct Fd *, void *, u_int, u_int);
	int (*dev_write)(struct Fd *, const void *, u_int, u_int);
	int (*dev_close)(struct Fd *);
	int (*dev_stat)(struct Fd *, struct Stat *);
	int (*dev_seek)(struct Fd *, u_int);
};

// file descriptor
struct Fd {
	u_int fd_dev_id;
	u_int fd_offset;
	u_int fd_omode;
};

// State不同的dev的相同部分,抽象出来称为Stat
struct Stat {
	char st_name[MAXNAMELEN];
	u_int st_size;
	u_int st_isdir;
	struct Dev *st_dev;
};


// file descriptor + file
struct Filefd {
	struct Fd f_fd;
	u_int f_fileid;
	struct File f_file;
};

//使用fd的id来找到对应的fd和数据
#define INDEX2FD(i) (FDTABLE + (i)*BY2PG)
#define INDEX2DATA(i) (FILEBASE + (i)*PDMAP)
```
下面是一些方法:
```c
//Find the smallest i from 0 to MAXFD-1 that doesn't have its fd page mapped
//申请fd,通过fd得到返回的fd
int fd_alloc(struct Fd **fd)
//使用syscall_mem_unmap释放fd占用的页
void fd_close(struct Fd *fd)
//Find the 'Fd' page for the given fd number.
int fd_lookup(int fdnum, struct Fd **fd)

//一些id,data和结构体指针的转换
void *fd2data(struct Fd *fd) {
	return (void *)INDEX2DATA(fd2num(fd));
}
int fd2num(struct Fd *fd) {
	return ((u_int)fd - FDTABLE) / BY2PG;
}
int num2fd(int fd) {
	return fd * BY2PG + FDTABLE;
}

//fd主要的操作都是通过fd的序号来做的,这也和实际一致.

//关闭fd
int close(int fdnum)
void close_all(void)

//应该是复制fd吧
int dup(int oldfdnum, int newfdnum)

//Read at most 'n' bytes from 'fd' at the current seek position into 'buf'.
//Return the number of bytes read successfully.
int read(int fdnum, void *buf, u_int n)
//不是很懂
int readn(int fdnum, void *buf, u_int n)
//写
int write(int fdnum, const void *buf, u_int n)
//单纯偏移offset
int seek(int fdnum, u_int offset)
//应该是初始化相关的, 目前不太清楚
int fstat(int fdnum, struct Stat *stat)
int stat(const char *path, struct Stat *stat)
```

## 用户空间的二级抽象--file.c
以上的fd.c我们都是对dev进行的操作, 但是具体到我们目前的文件系统, 我们实际上需要指定我们的dev是file
```c
//声明了devfile这个dev
struct Dev devfile = {
    .dev_id = 'f',
    .dev_name = "file",
    .dev_read = file_read,
    .dev_write = file_write,
    .dev_close = file_close,
    .dev_stat = file_stat,
};
```

下面是file的方法:
```c
// Open a file (or directory).
int open(const char *path, int mode)
//  Close a file descriptor
int file_close(struct Fd *fd)
//  Read 'n' bytes from 'fd' at the current seek position into 'buf'.
static int file_read(struct Fd *fd, void *buf, u_int n, u_int offset)
//  Find the virtual address of the page that maps the file block starting at 'offset'.
int read_map(int fdnum, u_int offset, void **blk)
//  Write 'n' bytes from 'buf' to 'fd' at the current seek position.
static int file_write(struct Fd *fd, const void *buf, u_int n, u_int offset)
//将dev初始化为file
static int file_stat(struct Fd *fd, struct Stat *st)
//  Truncate or extend an open file to 'size' bytes
int ftruncate(int fdnum, u_int size)
//  Delete a file or directory.
int remove(const char *path)
//  Synchronize disk with buffer cache
int sync(void)
```

## 用户进程与文件管理进程的通讯--fsipc.c
```c
//首先是定义的缓存区
u_char fsipcbuf[BY2PG] __attribute__((aligned(BY2PG)));

//Send an IPC request to the file server, and wait for a reply.
//这里的type实际上是上面文件系统需要接受的类型宏
static int fsipc(u_int type, void *fsreq, void *dstva, u_int *perm)
static int fsipc(u_int type, void *fsreq, void *dstva, u_int *perm)
int fsipc_open(const char *path, u_int omode, struct Fd *fd)
the server sends
// send us back a mapping(也就是va) for a page containing that block.
int fsipc_map(u_int fileid, u_int offset, void *dstva)
int fsipc_set_size(u_int fileid, u_int size)
int fsipc_close(u_int fileid)
int fsipc_dirty(u_int fileid, u_int offset)
int fsipc_remove(const char *path)
int fsipc_sync(void)
```

至此, 这个关系算是捋清楚了!
# 速记
## 设备映射的物理地址
```c
//console
#define DEV_CONS_ADDRESS 0x10000000
#define DEV_CONS_LENGTH 0x00000020
#define DEV_CONS_PUTGETCHAR 0x0000
#define DEV_CONS_HALT 0x0010

//disk
#define DEV_DISK_ADDRESS 0x13000000
#define DEV_DISK_OFFSET 0x0000
#define DEV_DISK_OFFSET_HIGH32 0x0008
#define DEV_DISK_ID 0x0010
#define DEV_DISK_START_OPERATION 0x0020
#define DEV_DISK_STATUS 0x0030

#define DEV_DISK_BUFFER 0x4000
#define DEV_DISK_BUFFER_LEN 0x200
/*  Operations:  */
#define DEV_DISK_OPERATION_READ 0
#define DEV_DISK_OPERATION_WRITE 1

//rtc
#define DEV_RTC_ADDRESS 0x15000000
#define DEV_RTC_LENGTH 0x00000200

#define DEV_RTC_TRIGGER_READ 0x0000
#define DEV_RTC_SEC 0x0010
#define DEV_RTC_USEC 0x0020

#define DEV_RTC_HZ 0x0100
#define DEV_RTC_INTERRUPT_ACK 0x0110
```
## 几个结构体
```c
struct Block {
    uint8_t data[BY2BLK];
    uint32_t type;
} disk[NBLOCK];
```

```c
struct File {
    char f_name[MAXNAMELEN]; // filename
    uint32_t f_size;     // file size in bytes
    uint32_t f_type;     // file type
    uint32_t f_direct[NDIRECT];
    uint32_t f_indirect;
    struct File *f_dir; // the pointer to the dir where this file is in, valid only in memory.
    char f_pad[BY2FILE - MAXNAMELEN - (3 + NDIRECT) * 4 - sizeof(void *)];
} __attribute__((aligned(4), packed));
```

# Debug
这次debug的时间比较短, 但是错的地方都挺隐蔽+脑残的...
总结几点:
## for循环
首先要对自己写的代码有一个基础的理解: 为什么要用for循环
接着, for循环里面的变量是什么----老是忘记在for循环里面使用变量导致错误!

## 根据测试寻找bug
1. 总体理解测试的过程, 了解有哪些测试点
2. 卡在哪个点, 就一层层梳理函数的调用关系, 看看能不能直接找到错误函数
3. 如果找不到或者不知道错在哪, 通过改变测试函数(特别是结合测试点)来尝试理解错误的原因

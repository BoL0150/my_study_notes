

# CSAPP实验记录（四）：Malloc lab

此实验要求实现自己的存储分配器，特别是malloc、free 和 realloc 函数的实现。

我们将修改和提交的唯一文件是mm.c。

mdriver.c程序是一个驱动程序，它可以评估我们的实现方案的性能。使用命令make编译生成驱动程序并使用该命令运行它：`./mdriver -V`（-V标志显示有用的摘要信息。）

我们的动态内存分配器将包含四个函数，声明在mm.h中，是现在mm.c中

```c
int mm_init(void);
void *mm_malloc(size_t size);
void mm_free(void *ptr);
void *mm_realloc(void *ptr, size_t size);
```

我们的mm.c文件已经实现了最简单但在功能上仍然正确的malloc，使用此作为起点，修改这些函数（并可能定义其他私有静态函数）：

- `mm_init`： 在调用`mm_malloc` `mm_realloc`或`mm_free`之前，应用程序（即将用于评估实现的trace-driven driver程序）调用mm_init来执行初始化，例如分配初始的堆区域。如果在执行初始化时出现问题，返回值为-1，否则为0。
- `mm_malloc`：`mm_malloc`程序返回一个指向**有效载荷**至少为size字节的分配块的指针。不能超出堆的范围，也不能覆盖其他已分配的区域。我们的实现将与标准C库(libc)中提供的malloc版本进行比较。**分配的块应该8字节对齐**。
- `mm_free`：释放 `ptr` 指针指向的区域（这个区域必须是由之前的mm_malloc或mm_realloc分配的，而且还没有被释放的），什么都不返回。
- `mm_realloc`：返回一个指向**有效载荷**至少为size字节的分配块的指针，限制条件：
  - 如果指针为空，则等同于`mm_malloc(size)`，返回分配块的指针
  - `size` 为 0 时，等同于 `mm_free(ptr)`，返回null
  - 如果ptr不为null，该指针必须是之前对mm_malloc或mm_realloc的调用返回的，且没有被释放。调用`mm_realloc`会将ptr（旧块）指向的内存块的大小更改为size字节，并返回新块的地址。新块的内容与旧块相同，比如：
    - 如果旧块的大小为8字节，新块的大小为10字节，则新块的前8块与旧块的内容相同，其余的块未初始化。
    - 如果旧块的大小为8字节，新块的大小为4字节，新块的内容与旧块的前4个字节相同。

以上的函数的语义与libc中的malloc、realloc和free的语义相同。



###  **Heap Consistency Checker**

一致性检查器应该检查的内容：

- Is every block in the free list marked as free?
- Are there any contiguous free blocks that somehow escaped coalescing?
- Is every free block actually in the free list?
- Do the pointers in the free list point to valid free blocks?
- Do any allocated blocks overlap?
- Do the pointers in a heap block point to valid heap addresses?

内存检查器由mm.c中的 `int mm_check(void)`组成，如果我们的堆没有问题，此函数返回非0值。

### **Support Routines**

memlib.c包为我们的动态内存分配器模拟内存系统，我们可以调用其中的这些函数：

- `void *mem_sbrk(int incr)`：将堆扩展incr个字节，返回一个指向新分配的堆区的首字节的泛型指针。与Unix的sbrk函数语义相同，除了此函数**只接收正整数**，即只能扩大堆，不能缩小。
- `void *mem_heap_lo(void)`: 返回一个指向堆的第一个字节的泛型指针
- `void *mem_heap_hi(void)`: 返回一个指向堆的最后一个字节的泛型指针
- `size_t mem_heapsize(void)`: 返回堆当前的大小（字节）
- `size_t mem_pagesize(void)`: 返回系统的页大小（在Linux中是4K）

### **Programming Rules**

- 不能改动`mm.c`中的任何接口
- `mm.c` 中不能定义任何全局或静态**复合**数据结构，比如struct，array, tree 或 list。但是可以定义全局标量变量诸如 integer, float 和 pointer 等
- 我们的代码应该被分解为函数，并尽可能地少用全局变量
- 为了和libc中的malloc保持一致，返回的指针必须是 8-byte 对齐的
- 编译代码不能有警告

可以通过 `mdriver.c` 来进行测试，每个测试会跑 12 次，一次检测正确性，一次检测空间使用，十次测试性能，下面是具体的参数：

- `-p`：完整测试

- `-t <tracedir>`：在自定义的`tracedir`文件夹中搜索测试文件，而不是定义在`config.h`中的默认的文件

- `-f <tracefile>`：使用`tracefile`文件进行一个特定的测试。

  提供了两个简单的验证文件`short1-bal.rep`和`short2-bal.rep`来供我们初期开发使用。我们可以调用`./mdriver -f short1-bal.rep -V`来查看单个文件的测试结果

- `-c <tracefile>`：执行特定的测试 1 次，用来检测正确性很方便

- `-h`：输出命令行参数

- `-l`：用真实的 `malloc` 函数来测试，可以比较自己写的代码和系统代码的差距

- `-V`：输出各种信息。在一个紧凑的表格中打印每个测试文件的性能明细表

- `-v <verbose level>`：设置需要输出的日志等级

- `-d <i>`： 有 0,1,2 三个层级，检查的标准越来越严格

- `-D`：等于 `-d2`

- `-s <s>`：超过 s 秒则认为是超时，默认是永远不会超时的

## 遇到的问题及解决方法

首次make时报错

```text
/usr/include/linux/errno.h:1:10: fatal error: asm/errno.h: 没有那个文件或目录
```

出现此错误是因为在64位Linux操作系统中打算编译出32位的程序，

- 解决方案一：

   `sudo apt-get install build-essential libc6-dev libc6-dev-i386`

- 解决方案二：使用如下命令安装32位的库

  `apt-get install gcc-multilib`

- 解决方案三：[如何在64位Linux系统上编译32位程序](https://blog.csdn.net/qq_36287943/article/details/103601514)

`./mdriver -V`时报错

```text
Testing mm malloc
Reading tracefile: amptjp-bal.rep
Could not open ... /amptjp-bal.rep in read_trace: No such file or directory
```

在config.h中设置TRACEDIR为自己traces文件夹的路径重新make

```text
#define TRACEDIR "/home/bolee/csapp/malloclab-handout/traces/"
```

调试方式：

要debug首先需要更改Makefile中的编译选项，在CFLAGS后加上-g，去掉-O2。

进入`gdb`首先在mm_init、mm_malloc和mm_free函数上打上断点，再run。

## 具体实现

![image-20211002223851515](https://raw.githubusercontent.com/BoL0150/image2/master/image-20211002223851515.png)

### 隐式空闲链表

```c
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>
#include <unistd.h>
#include <string.h>

#include "mm.h"
#include "memlib.h"

/*********************************************************
 * NOTE TO STUDENTS: Before you do anything else, please
 * provide your team information in the following struct.
 ********************************************************/
team_t team =
{
    /* Team name */
    "boleeteam",
    /* First member's full name */
	"bolee",
    /* First member's email address */
	"hhh@gm.com",
    /* Second member's full name (leave blank if none) */
    "",
    /* Second member's email address (leave blank if none) */
    ""
};

/* single word (4) or double word (8) alignment */
#define ALIGNMENT 8

// 向上舍入到8字节对齐
// 对~0x7求与后将低三位变成了0，也就是与8字节对齐
// 加上7保证了向上舍入
#define ALIGN(size) (((size) + (ALIGNMENT-1)) & ~0x7)

// 最小块的大小为16字节，8字节用于额外开销（头部和脚部），8字节用于对齐（不分配有效载荷为0的块）
#define MIN_BLOCK_SIZE 16

#define SIZE_T_SIZE (ALIGN(sizeof(size_t)))

#define WSIZE 4 // 字的大小
#define DSIZE 8 // 双字的大小
#define CHUNKSIZE (1<<12) // 扩展堆时的默认大小，4KB

#define MAX(x,y) ((x) > (y)? (x) : (y))
// 将一个块的大小（size）和分配标志位（allocated bit）打包成一个字
// 分配标志位占据低三位
#define PACK(size, alloc) ((size) | (alloc))

// 在地址p处，读取和写入一个字
// 参数p典型地是一个（void *）指针，不可以直接进行解引用，
// 要读取一个字,需要先将泛型指针转化为32位类型的指针（比如unsigned int），
// 再进行解引用
#define GET(p) (*(unsigned int *)(p))
#define PUT(p, val) (*(unsigned int *)(p) = (val)) 

// 从地址p的头部或脚部分别返回大小和已分配标志位
#define GET_SIZE(p) (GET(p) & ~0x7)
#define GET_ALLOC(p) (GET(p) & 0x1)
// 给定一个块指针bp，分别返回指向这个块的头部和脚部的指针
#define HDRP(bp) ((char *)(bp) - WSIZE)
#define FTRP(bp) ((char *)(bp) + GET_SIZE(HDRP(bp)) - DSIZE)
// 给定一个块指针bp，分别返回指向后面块和前面的块的块指针
#define NEXT_BLKP(bp) ((char *)(bp) + GET_SIZE(HDRP(bp)))
#define PREV_BLKP(bp) ((char *)(bp) - GET_SIZE((char *)(bp) - DSIZE))

static char * heap_listp;
// 上一次在堆中匹配的位置，用于next_fit，从此处开始匹配
static char * pre_listp;
static void * extend_heap(size_t words);
static void * coalesce(void *bp);
static void * first_fit(size_t asize);
static void * best_fit(size_t asize);
static void * next_fit(size_t asize);
static void place(void *bp,size_t asize);
int mm_init(void);
void *mm_malloc(size_t size);
void mm_free(void *bp);
void *mm_realloc(void *ptr,size_t size);

// 在调用mm_malloc或者mm_free之前，应用必须先调用mm_init函数来初始化堆
int mm_init(void)
{
    // 从内存系统得到四个字
    // 从void*隐式转化为char* 在C中可以，在C++中不行，需要使用static_cast进行强制类型转换
    if((heap_listp = (char*)mem_sbrk(4*WSIZE)) == (void *)-1)
        return -1;
    // 初始化堆
    // 初始化填充块
    PUT(heap_listp, 0);
    // 初始化序言块。
    // 序言块和结尾块总是标记为已分配，用来消除合并时的边界条件
    // 否则我们在释放块时，需要先判断bp前后是否有块，在来判断是否空闲。
    PUT(heap_listp + (1*WSIZE), PACK(DSIZE,1));
    PUT(heap_listp + (2*WSIZE), PACK(DSIZE,1));
    // 初始化结尾块
    PUT(heap_listp + (3*WSIZE), PACK(0,1));

    heap_listp += (2*WSIZE);
    pre_listp = heap_listp;
    // 将这个堆扩展 CHUNKSIZE个字节，并创建初始的空闲块
    if(extend_heap(CHUNKSIZE/WSIZE) == NULL){
        return -1;
    }
    return 0;
}
static void *extend_heap(size_t words){
    char *bp;
    size_t size;
    // 向上舍入，分配偶数个字来保持双字对齐
    size = (words % 2) ? (words+1) * WSIZE : words*WSIZE;
    if((long)(bp=(char*)mem_sbrk(size))==-1){
        return NULL;
    }
    // mem_sbrk给堆新分配的内存片的地址在之前的结尾块的头部后面
    // 这个头部就变成了新的空闲块的头部，并且这个片的最后一个字变成了
    // 新的结尾块的头部,倒数第二个字变成了新的空闲块的尾部
    PUT(HDRP(bp),PACK(size,0));
    PUT(FTRP(bp),PACK(size,0));
    // 结尾块的头部
    PUT(HDRP(NEXT_BLKP(bp)),PACK(0,1));

    // 如果前一个块是空闲块，就合并，并返回指向合并后的块的块指针
    return coalesce(bp);
}
// 合并空闲块，使用场景：一、释放块时；二、内存系统给堆分配新的块时
static void * coalesce(void *bp){
    // 获取前后两个相邻块的分配标志位
    size_t prev_alloc = GET_ALLOC(FTRP(PREV_BLKP(bp)));
    size_t next_alloc = GET_ALLOC(HDRP(NEXT_BLKP(bp)));
    size_t size = GET_SIZE(HDRP(bp));

    // 如果前后的两个块已分配
    if(prev_alloc && next_alloc){
        pre_listp = (char*)bp;
        return bp;
    }
    else if (prev_alloc && !next_alloc){ // 如果前面的块已分配，后面的块未分配
        size += GET_SIZE(HDRP(NEXT_BLKP(bp)));
        PUT(HDRP(bp), PACK(size,0));
        PUT(FTRP(bp), PACK(size,0));
    }
    else if (!prev_alloc && next_alloc){
        size += GET_SIZE(HDRP(PREV_BLKP(bp)));
        PUT(FTRP(bp), PACK(size, 0));
        PUT(HDRP(PREV_BLKP(bp)), PACK(size, 0));
        bp = PREV_BLKP(bp);
    }
    else{
        size += GET_SIZE(HDRP(PREV_BLKP(bp))) +
            GET_SIZE(HDRP(NEXT_BLKP(bp)));
        PUT(HDRP(PREV_BLKP(bp)), PACK(size, 0));
        PUT(FTRP(NEXT_BLKP(bp)), PACK(size, 0));
        bp = PREV_BLKP(bp);
    }
    pre_listp = (char*)bp;
    return bp;
}
/* 
 * mm_malloc - Allocate a block by incrementing the brk pointer.
 *     Always allocate a block whose size is a multiple of the alignment.
 *     size的单位是字节
 */
void *mm_malloc(size_t size)
{
    // asize和 extendsize的单位都是字节
    // 分配的块的大小
    size_t asize;
    // 如果没有匹配，堆应该扩展的大小
    size_t extendsize;
    char *bp;

    if(size == 0)
        return NULL;
    
    // 8字节用来放头部和脚部，而又不分配有效载荷为0的块，所以对齐后最小块的大小为16字节
    if(size <= DSIZE){
        asize = 2*DSIZE;
    }else{
        // 块的大小为：应用请求分配的有效载荷 加上 开销字节（头部和脚部）然后再向上舍入到最接近的8字节的整数倍
        asize = DSIZE * ((size + (DSIZE) + (DSIZE-1))/DSIZE);
    }

    // 分配器确定的了请求的块的大小后，搜索空闲链表来寻找一个合适的空闲块
    if((bp = (char *)next_fit(asize)) != NULL){
        // 如果有合适的，那么就在bp指向的位置放置asize大小的请求块，并可选地分割出多余的部分
        place(bp,asize);
        // 然后返回新分配块的地址
        return bp;
    }
    // 如果没有合适的空闲块，就向系统请求更多的空间，用一个新的空闲块来扩展堆，
    // 再把请求块放置在这个新的空闲块中，然后返回一个指针，指向这个新分配的块
    extendsize = MAX(asize,CHUNKSIZE);
    if((bp = (char *)extend_heap(extendsize/WSIZE)) == NULL)
        return NULL;
    place(bp, asize);
    return bp;
}
// 使用first fit策略来寻找空闲块
static void *first_fit(size_t asize){
    // 每一次都从堆的头部开始搜索，一直搜索到结尾块（SIZE等于0的块）
    for(char* bp = heap_listp; GET_SIZE(HDRP(bp)) > 0;bp = NEXT_BLKP(bp)){
        if(!GET_ALLOC(HDRP(bp)) && GET_SIZE(HDRP(bp)) >= asize){
            return bp;
        }
    }
    return NULL;
}
static void * best_fit(size_t asize){
    return NULL;
}
static void * next_fit(size_t asize){
    // 从上一次匹配的位置开始搜索，直到结尾块
    for(char* bp = pre_listp;GET_SIZE(HDRP(bp)) > 0;bp = NEXT_BLKP(bp)){
        if(!GET_ALLOC(HDRP(bp)) && GET_SIZE(HDRP(bp)) >= asize){
            pre_listp = bp;
            return bp;
        }
    }
    // 如果从上一次的位置开始没有匹配的，则需要从头开始搜索
    for(char* bp = heap_listp;bp != pre_listp;bp = NEXT_BLKP(bp)){
        if(!GET_ALLOC(HDRP(bp)) && GET_SIZE(HDRP(bp)) >= asize){
            pre_listp = bp;
            return bp;
        }
    }
    return NULL;
}
// 在bp指向的空闲块处放置请求块，当剩余的部分大于等于最小块时，才将此空闲块分割
static void place(void* bp,size_t asize){
    size_t size = GET_SIZE(HDRP(bp));
    size_t remain_size = size - asize;
    if(remain_size >= MIN_BLOCK_SIZE){
        // bp块的头部中的块大小变成了asize
        PUT(HDRP(bp),PACK(asize,1));
        PUT(FTRP(bp),PACK(asize,1));
        bp = NEXT_BLKP(bp);
        PUT(HDRP(bp),PACK(remain_size,0));
        PUT(FTRP(bp),PACK(remain_size,0));
    }else{
        // 将空闲块的分配标志位改为1即可
        PUT(HDRP(bp),PACK(size,1));
        PUT(FTRP(bp),PACK(size,1));
    }
    pre_listp = (char*)bp;
}
/*
 * mm_free - Freeing a block does nothing.
 */
void mm_free(void *ptr)
{
    // 将要释放的块的头部和脚部的分配标志位改为0即可
    size_t size = GET_SIZE(HDRP(ptr));

    PUT(HDRP(ptr), PACK(size, 0));
    PUT(FTRP(ptr), PACK(size, 0));
    // 释放时需要与相邻的空闲块合并
    coalesce(ptr);
}

/*
 * mm_realloc - Implemented simply in terms of mm_malloc and mm_free
 */
void *mm_realloc(void *ptr, size_t size)
{
    if(ptr == NULL)
        return mm_malloc(size);
    if(size == 0){
        mm_free(ptr);
        return NULL;
    }
    void *oldptr = ptr;
    void *newptr;
    size_t copySize;

    newptr = mm_malloc(size);
    if (newptr == NULL)
      return NULL;
    size = GET_SIZE(HDRP(oldptr));
    copySize = GET_SIZE(HDRP(newptr));
    if (size < copySize)
      copySize = size;
    memcpy(newptr, oldptr, copySize-WSIZE);
    mm_free(oldptr);
    return newptr;
}
```

**隐式空闲链表 + first fit：**

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211002155104707.png" alt="image-20211002155104707" style="zoom:67%;" />

**隐式空闲链表 + next fit：**

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211002154825861.png" alt="image-20211002154825861" style="zoom:67%;" />

### 显式空闲链表

空闲块的header后的两个字用来存储两个指针，一个指向前驱的空闲块，一个指向后继的空闲块

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916170112887.png" alt="image-20210916170112887" style="zoom:50%;" />

```c
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>
#include <unistd.h>
#include <string.h>

#include "mm.h"
#include "memlib.h"

/*********************************************************
 * NOTE TO STUDENTS: Before you do anything else, please
 * provide your team information in the following struct.
 ********************************************************/
team_t team =
{
    /* Team name */
    "boleeteam",
    /* First member's full name */
	"bolee",
    /* First member's email address */
	"hhh@gm.com",
    /* Second member's full name (leave blank if none) */
    "",
    /* Second member's email address (leave blank if none) */
    ""
};

/* single word (4) or double word (8) alignment */
#define ALIGNMENT 8

// 向上舍入到8字节对齐
// 对~0x7求与后将低三位变成了0，也就是与8字节对齐
// 加上7保证了向上舍入
#define ALIGN(size) (((size) + (ALIGNMENT-1)) & ~0x7)

// 最小块的大小为16字节，8字节用于额外开销（头部和脚部），8字节用于对齐（不分配有效载荷为0的块）
#define MIN_BLOCK_SIZE 16

#define SIZE_T_SIZE (ALIGN(sizeof(size_t)))

#define WSIZE 4 // 字的大小
#define DSIZE 8 // 双字的大小
#define CHUNKSIZE (1<<12) // 扩展堆时的默认大小，4KB

#define MAX(x,y) ((x) > (y)? (x) : (y))
// 将一个块的大小（size）和分配标志位（allocated bit）打包成一个字
// 分配标志位占据低三位
#define PACK(size, alloc) ((size) | (alloc))

// 在地址p处，读取和写入一个字
// 参数p典型地是一个（void *）指针，不可以直接进行解引用，
// 要读取一个字,需要先将泛型指针转化为32位类型的指针（比如unsigned int），
// 再进行解引用
#define GET(p) (*(unsigned int *)(p))
#define PUT(p, val) (*(unsigned int *)(p) = (val)) 

// 从地址p的头部或脚部分别返回大小和已分配标志位
#define GET_SIZE(p) (GET(p) & ~0x7)
#define GET_ALLOC(p) (GET(p) & 0x1)
// 给定一个块指针bp，分别返回指向这个块的头部和脚部的指针
#define HDRP(bp) ((char *)(bp) - WSIZE)
#define FTRP(bp) ((char *)(bp) + GET_SIZE(HDRP(bp)) - DSIZE)
// 给定一个块指针bp，分别返回指向后面块和前面的块的块指针
#define NEXT_BLKP(bp) ((char *)(bp) + GET_SIZE(HDRP(bp)))
#define PREV_BLKP(bp) ((char *)(bp) - GET_SIZE((char *)(bp) - DSIZE))
// 给定一个空闲块指针p，获取前驱或后继空闲块的块指针
// 这两个指针存储在header后的两个字中
#define PRED_FBLKP(p) (*(unsigned int *)(p))
#define SUCC_FBLKP(p) (*((unsigned int *)(p)+1))
// 设置p的前驱或后继指针指向哪个空闲块
#define SET_PRED(p,pred) (*(unsigned int *)(p) = (pred))
#define SET_SUCC(p,succ) (*((unsigned int *)(p)+1) = (succ))
// 指向堆的头部
static char * heap_listp;
// 指向显式空闲链表的根节点
static char * free_list_root;
// 指向显式空闲链表的头部
static char * free_listp;
// 上一次在堆中匹配的位置，用于next_fit，从此处开始匹配
static char * pre_listp;
static void * extend_heap(size_t words);
static void * coalesce(void *bp);
static void * first_fit(size_t asize);
static void * best_fit(size_t asize);
static void * next_fit(size_t asize);
static void remove_from_free_list(void *bp);
static void insert_to_free_list(void *bp);
static void place(void *bp,size_t asize);
int mm_init(void);
void *mm_malloc(size_t size);
void mm_free(void *bp);
void *mm_realloc(void *ptr,size_t size);

// 在调用mm_malloc或者mm_free之前，应用必须先调用mm_init函数来初始化堆
int mm_init(void)
{
    // 从内存系统得到四个字
    // 从void*隐式转化为char* 在C中可以，在C++中不行，需要使用static_cast进行强制类型转换
    if((heap_listp = (char*)mem_sbrk(4*WSIZE)) == (void *)-1)
        return -1;
    // 初始化堆
    // 初始化填充块
    PUT(heap_listp, 0);
    // 初始化序言块。
    // 序言块和结尾块总是标记为已分配，用来消除合并时的边界条件
    // 否则我们在释放块时，需要先判断bp前后是否有块，在来判断是否空闲。
    PUT(heap_listp + (1*WSIZE), PACK(DSIZE,1));
    PUT(heap_listp + (2*WSIZE), PACK(DSIZE,1));
    // 初始化结尾块
    PUT(heap_listp + (3*WSIZE), PACK(0,1));

    heap_listp += (2*WSIZE);
    // 显式空闲链表不需要next_fit
    // pre_listp = heap_listp;
    // 显式空闲链表的根节点为序言块
    free_list_root = heap_listp;
    // 显式空闲链表的头部被初始化为空
    free_listp = NULL;
    // 将这个堆扩展 CHUNKSIZE个字节，并创建初始的空闲块
    if(extend_heap(CHUNKSIZE/WSIZE) == NULL){
        return -1;
    }
    return 0;
}
static void *extend_heap(size_t words){
    char *bp;
    size_t size;
    // 向上舍入，分配偶数个字来保持双字对齐
    size = (words % 2) ? (words+1) * WSIZE : words*WSIZE;
    if((long)(bp=(char*)mem_sbrk(size))==-1){
        return NULL;
    }
    // mem_sbrk给堆新分配的内存片的地址在之前的结尾块的头部后面
    // 这个头部就变成了新的空闲块的头部，并且这个片的最后一个字变成了
    // 新的结尾块的头部,倒数第二个字变成了新的空闲块的尾部
    PUT(HDRP(bp),PACK(size,0));
    PUT(FTRP(bp),PACK(size,0));
    
    // 由于该空闲块还没插入空闲链表中，所以先设置该空闲块的前驱和后继指针为NULL，coalesce负责将空闲块加入空闲链表
    SET_PRED(bp,0);
    SET_SUCC(bp,0);
    // 结尾块的头部
    PUT(HDRP(NEXT_BLKP(bp)),PACK(0,1));

    // 如果前一个块是空闲块，就合并，并返回指向合并后的块的块指针
    return coalesce(bp);
}
// 合并空闲块，使用场景：一、释放块时；二、内存系统给堆分配新的块时
static void * coalesce(void *bp){
    // 获取前后两个相邻块的块指针和分配标志位
    void *prev_bp = PREV_BLKP(bp);
    void *next_bp = NEXT_BLKP(bp);
    size_t prev_alloc = GET_ALLOC(FTRP(PREV_BLKP(bp)));
    size_t next_alloc = GET_ALLOC(HDRP(NEXT_BLKP(bp)));

    size_t size = GET_SIZE(HDRP(bp));
    // 判断相邻的块是否为空闲块，如果是空闲块，就要将它们从空闲链表中移除
    // 然后与bp指向的块合并，再将合并后的空闲块加入显式空闲链表的头结点
    
    // 如果前后的两个块已分配
    if(prev_alloc && next_alloc){
        // pre_listp = (char*)bp;
    }
    else if (prev_alloc && !next_alloc){ // 如果前面的块已分配，后面的块未分配
        // 将相邻的空闲块从空闲链表中移除
        remove_from_free_list(next_bp);
        // 与bp指向的块合并
        size += GET_SIZE(HDRP(NEXT_BLKP(bp)));
        PUT(HDRP(bp), PACK(size,0));
        PUT(FTRP(bp), PACK(size,0));
    }
    else if (!prev_alloc && next_alloc){
        remove_from_free_list(prev_bp);
        size += GET_SIZE(HDRP(PREV_BLKP(bp)));
        PUT(FTRP(bp), PACK(size, 0));
        PUT(HDRP(PREV_BLKP(bp)), PACK(size, 0));
        bp = PREV_BLKP(bp);
    }
    else{
        remove_from_free_list(prev_bp);
        remove_from_free_list(next_bp);
        size += GET_SIZE(HDRP(PREV_BLKP(bp))) +
            GET_SIZE(HDRP(NEXT_BLKP(bp)));
        PUT(HDRP(PREV_BLKP(bp)), PACK(size, 0));
        PUT(FTRP(NEXT_BLKP(bp)), PACK(size, 0));
        bp = PREV_BLKP(bp);
    }
    // pre_listp = (char*)bp;
    // 将合并后的空闲块插入显式空闲链表的头结点
    insert_to_free_list(bp);
    return bp;
}
// 将指定的空闲块从空闲链表中移除
static void remove_from_free_list(void *bp){
    void *pred = PRED_FBLKP(bp);
    void *succ = SUCC_FBLKP(bp);
    SET_PRED(bp,0);
    SET_SUCC(bp,0);

    if(pred == NULL && succ ==NULL){
        free_listp = NULL;
    }else if(pred == NULL){
        SET_PRED(succ,0);
        free_listp = succ;
    }else if(succ == NULL){
        SET_SUCC(pred,0);
    }else{
        SET_SUCC(pred,succ);
        SET_PRED(succ,pred);
    }
}
static void insert_to_free_list(void *bp){
    if(free_listp == NULL){
        free_listp = bp;
        return;
    }
    SET_SUCC(bp,free_listp);
    SET_PRED(free_listp,bp);
    free_listp = bp;
}
/* 
 * mm_malloc - Allocate a block by incrementing the brk pointer.
 *     Always allocate a block whose size is a multiple of the alignment.
 *     size的单位是字节
 */
void *mm_malloc(size_t size)
{
    // asize和 extendsize的单位都是字节
    // 分配的块的大小
    size_t asize;
    // 如果没有匹配，堆应该扩展的大小
    size_t extendsize;
    char *bp;

    if(size == 0)
        return NULL;
    
    // 8字节用来放头部和脚部，而又不分配有效载荷为0的块，所以对齐后最小块的大小为16字节
    if(size <= DSIZE){
        asize = 2*DSIZE;
    }else{
        // 块的大小为：应用请求分配的有效载荷 加上 开销字节（头部和脚部）然后再向上舍入到最接近的8字节的整数倍
        asize = DSIZE * ((size + (DSIZE) + (DSIZE-1))/DSIZE);
    }

    // 分配器确定的了请求的块的大小后，搜索空闲链表来寻找一个合适的空闲块
    if((bp = (char *)first_fit(asize)) != NULL){
        // 如果有合适的，那么就在bp指向的位置放置asize大小的请求块，并可选地分割出多余的部分
        place(bp,asize);
        // 然后返回新分配块的地址
        return bp;
    }
    // 如果没有合适的空闲块，就向系统请求更多的空间，用一个新的空闲块来扩展堆，
    // 再把请求块放置在这个新的空闲块中，然后返回一个指针，指向这个新分配的块
    extendsize = MAX(asize,CHUNKSIZE);
    if((bp = (char *)extend_heap(extendsize/WSIZE)) == NULL)
        return NULL;
    place(bp, asize);
    return bp;
}
// 使用first fit策略来寻找空闲块
static void *first_fit(size_t asize){
    // 每一次都从显式空闲链表的头部开始搜索，一直搜索到NULL
    for(char* bp = free_listp; bp != 0;bp = SUCC_FBLKP(bp)){
        if(GET_SIZE(HDRP(bp)) >= asize){
            return bp;
        }
    }
    return NULL;
}
static void * best_fit(size_t asize){
    return NULL;
}
static void * next_fit(size_t asize){
    // 从上一次匹配的位置开始搜索，直到结尾块
    for(char* bp = pre_listp;GET_SIZE(HDRP(bp)) > 0;bp = NEXT_BLKP(bp)){
        if(!GET_ALLOC(HDRP(bp)) && GET_SIZE(HDRP(bp)) >= asize){
            pre_listp = bp;
            return bp;
        }
    }
    // 如果从上一次的位置开始没有匹配的，则需要从头开始搜索
    for(char* bp = heap_listp;bp != pre_listp;bp = NEXT_BLKP(bp)){
        if(!GET_ALLOC(HDRP(bp)) && GET_SIZE(HDRP(bp)) >= asize){
            pre_listp = bp;
            return bp;
        }
    }
    return NULL;
}
// 在bp指向的空闲块处放置请求块，当剩余的部分大于等于最小块时，才将此空闲块分割
// 将bp指向的块从空闲链表中移除，将分割出来的空闲块与相邻的空闲块合并，合并完后插入空闲链表的头部
static void place(void* bp,size_t asize){
    remove_from_free_list(bp);
    size_t size = GET_SIZE(HDRP(bp));
    size_t remain_size = size - asize;
    if(remain_size >= MIN_BLOCK_SIZE){
        // bp块的头部中的块大小变成了asize
        PUT(HDRP(bp),PACK(asize,1));
        PUT(FTRP(bp),PACK(asize,1));
        bp = NEXT_BLKP(bp);
        PUT(HDRP(bp),PACK(remain_size,0));
        PUT(FTRP(bp),PACK(remain_size,0));
        // 设置bp的前驱和后继块为空
        SET_SUCC(bp,0);
        SET_PRED(bp,0);
        coalesce(bp);
    }else{
        // 将空闲块的分配标志位改为1即可
        PUT(HDRP(bp),PACK(size,1));
        PUT(FTRP(bp),PACK(size,1));
    }
    // pre_listp = (char*)bp;
}
/*
 * mm_free - Freeing a block does nothing.
 */
void mm_free(void *ptr)
{
    // 将要释放的块的头部和脚部的分配标志位改为0即可
    size_t size = GET_SIZE(HDRP(ptr));

    PUT(HDRP(ptr), PACK(size, 0));
    PUT(FTRP(ptr), PACK(size, 0));
    SET_PRED(ptr,0);
    SET_SUCC(ptr,0);
    // 释放时需要与相邻的空闲块合并
    coalesce(ptr);
}

/*
 * mm_realloc - Implemented simply in terms of mm_malloc and mm_free
 */
void *mm_realloc(void *ptr, size_t size)
{
    if(ptr == NULL)
        return mm_malloc(size);
    if(size == 0){
        mm_free(ptr);
        return NULL;
    }
    void *oldptr = ptr;
    void *newptr;
    size_t copySize;

    newptr = mm_malloc(size);
    if (newptr == NULL)
      return NULL;
    size = GET_SIZE(HDRP(oldptr));
    copySize = GET_SIZE(HDRP(newptr));
    if (size < copySize)
      copySize = size;
    memcpy(newptr, oldptr, copySize-WSIZE);
    mm_free(oldptr);
    return newptr;
}

```

**显式空闲链表 + first fit：**

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211002234510035.png" alt="image-20211002234510035" style="zoom:67%;" />

堆中的空闲块的组织方式有三种：

- 隐式空闲链表
  - 给定一个块的指针，为了方便找到这个块的前一个块和后一个块，**堆中每一个块除了有效载荷之外，在开头和结尾还各有一个字的空间**，用来记录这个块的大小（有效载荷加上头部和脚部），由于一个块的大小要求双字对齐，也就是八字节的倍数，所以头部和脚部的低三位总是0，我们使用这低三位记录该块是否是空闲。
  
    隐式空闲链表就是这样将堆中的所有块用双向链表的方式连接起来，通过一个块可以迅速找到相邻的两个块
  
    调用malloc之前先调用sbrk初始化堆，分配四个字的空间，第一个字作为填充块，第二三个字作为序言块的头部和脚部，第四个字作为结尾块，结尾块size为0，以减少合并时的边界条件，再调用sbrk给堆分配空间，目前是堆中的唯一一个空闲块
  
  - malloc的实现：malloc实际分配的块大小为：应用请求分配的有效载荷 加上 头部和脚部 然后再向上舍入与8字节对齐
  
    再使用first fit或next fit策略在空闲链表中找到大小匹配的块
  
    - 如果空闲链表中有合适的块，就该块中放置请求的块，并分割出多余的部分作为一个块插入链表中
    - 如果没有合适的块，就调用sbrk向操作系统请求更多的空间，用一个新的空闲块来扩展堆，再把请求块放在这个新的空闲块中
  
    返回指向分配的请求块的指针
  
  - free的实现：将要释放的块的头部和脚部的分配标志位改为0即可，如果有相邻的空闲块，需要与相邻的空闲块合并，将他们先从链表中移除，合并之后再加入链表
  
- 显式空闲链表
  - 在隐式空闲链表的基础上，在所有空闲块中加入一个指向前一个空闲块的指针，和执行下一个空闲块的指针，将隐式空闲链表中的空闲块组织成双向链表的形式，使首次适配的分配时间从块总数的线性时间减少到了**空闲块数量的线性时间**。显式链表的缺点是空闲块必须足够大，提高了内部碎片的程度
  
- 分离空闲链表
  - 将所有空闲块根据块大小分成不同类别，称为**大小类（Size Class）**，比如可以根据2幂次分成![img](https://raw.githubusercontent.com/BoL0150/image2/master/v2-b13008ebc3b408acc716fd392139c6a1_1440w.png)，每个类别一条链表，按照类别将空闲链表放入数组中，类似HashMap的实现方式，

在调用malloc、realloc、free之前，应用程序需要先调用init来初始化堆，初始化序言块，结尾块和填充块，再调用sbrk函数向内存系统申请分配初始的堆区域。将初始的堆区域加入空闲链表。

应用程序调用malloc函数在堆中分配空间。先用应用请求的有效载荷加上额外开销（比如块的头部和脚部），然后再向上舍入，与双字对齐，计算出实际分配的块的大小。然后在空闲链表中搜索，根据不同的放置策略来确定使用的空闲块：

- 首次适配
- 下一次适配
- 最佳适配

如果链表中有合适的空闲块，就将块地址返回。（如果是显示空闲链表，需要将此块从空闲链表中移除）。如果没有合适的空闲块，就向系统请求更多的空间，用一个新的空闲块来扩展堆，将这个新分配的块地址返回。

应用程序可以调用free函数来释放之前由malloc或realloc分配的块，只需要将指针所指向位置（header）的已分配标志位变为0即可。再与相邻的空闲块合并

- 如果是显式空闲链表，**需要将相邻的空闲块从空闲链表中移除，与要释放的块合并，再插入空闲链表的头部**。
- 如果是隐式空闲链表，直接改变相邻空闲块的头部和脚部的size即可。

### Segregated Free Lists

空闲块的结构与显式空闲链表的结构相同

我们需要确定分离的空闲链表的大小类，由于一个空闲块包含头部、后继、前驱和脚部，至少需要16字节，所以空闲块的最小块为16字节，小于16字节就无法记录完整的空闲块内容，所以大小类设置为

```text
{16-31},{32-63},{64-127},{128-255},{256-511},{512-1023},{1024-2047},{2048-4095},{4096-inf}
```

我们需要在堆的开始包含这些大小类的root节点，指向各自对应的空闲链表，则root需要一个字的空间用来保存地址。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211004211550667.png" alt="image-20211004211550667" style="zoom:67%;" />

```c
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>
#include <unistd.h>
#include <string.h>

#include "mm.h"
#include "memlib.h"

/*********************************************************
 * NOTE TO STUDENTS: Before you do anything else, please
 * provide your team information in the following struct.
 ********************************************************/
team_t team =
{
    /* Team name */
    "boleeteam",
    /* First member's full name */
	"bolee",
    /* First member's email address */
	"hhh@gm.com",
    /* Second member's full name (leave blank if none) */
    "",
    /* Second member's email address (leave blank if none) */
    ""
};

/* single word (4) or double word (8) alignment */
#define ALIGNMENT 8

// 向上舍入到8字节对齐
// 对~0x7求与后将低三位变成了0，也就是与8字节对齐
// 加上7保证了向上舍入
#define ALIGN(size) (((size) + (ALIGNMENT-1)) & ~0x7)

// 最小块的大小为16字节，16字节用于额外开销（头部、脚部以及指向前驱和后继空闲块的指针）
#define MIN_BLOCK_SIZE 16

#define SIZE_T_SIZE (ALIGN(sizeof(size_t)))

#define WSIZE 4 // 字的大小
#define DSIZE 8 // 双字的大小
#define CHUNKSIZE (1<<12) // 扩展堆时的默认大小，4KB

#define MAX(x,y) ((x) > (y)? (x) : (y))
// 将一个块的大小（size）和分配标志位（allocated bit）打包成一个字
// 分配标志位占据低三位
#define PACK(size, alloc) ((size) | (alloc))

// 在地址p处，读取和写入一个字
// 参数p典型地是一个（void *）指针，不可以直接进行解引用，
// 要读取一个字,需要先将泛型指针转化为32位类型的指针（比如unsigned int），
// 再进行解引用
#define GET(p) (*(unsigned int *)(p))
#define PUT(p, val) (*(unsigned int *)(p) = (val)) 

// 从地址p的头部或脚部分别返回大小和已分配标志位
#define GET_SIZE(p) (GET(p) & ~0x7)
#define GET_ALLOC(p) (GET(p) & 0x1)
// 给定一个块指针bp，分别返回指向这个块的头部和脚部的指针
#define HDRP(bp) ((char *)(bp) - WSIZE)
#define FTRP(bp) ((char *)(bp) + GET_SIZE(HDRP(bp)) - DSIZE)
// 给定一个块指针bp，分别返回指向后面块和前面的块的块指针
#define NEXT_BLKP(bp) ((char *)(bp) + GET_SIZE(HDRP(bp)))
#define PREV_BLKP(bp) ((char *)(bp) - GET_SIZE((char *)(bp) - DSIZE))
// 给定一个空闲块指针p，获取前驱或后继空闲块的块指针
// 这两个指针存储在header后的两个字中
#define PRED_FBLKP(p) (*(unsigned int *)(p))
#define SUCC_FBLKP(p) (*((unsigned int *)(p)+1))
// 设置p的前驱或后继指针指向哪个空闲块
#define SET_PRED(p,pred) (*(unsigned int *)(p) = (pred))
#define SET_SUCC(p,succ) (*((unsigned int *)(p)+1) = (succ))
// 指向堆的头部
static char * heap_listp;
static char * block_list_start = NULL;
// 指向大小类数组的起始位置
static char * listp;
// 给定一个大小，获取指向对应大小类的空闲链表的头结点的指针
static void * get_free_list_head(size_t size);
static void * extend_heap(size_t words);
static void * coalesce(void *bp);
static int get_index(size_t size);
static void LIFO(void *bp,void *root);
static void * first_fit(size_t asize);
static void * best_fit(size_t asize);
static void * next_fit(size_t asize);
static void remove_from_free_list(void *bp);
static void insert_to_free_list(void *bp);
static void place(void *bp,size_t asize);
int mm_init(void);
void *mm_malloc(size_t size);
void mm_free(void *bp);
void *mm_realloc(void *ptr,size_t size);
```

init首先申请了12个字的空间，在堆空间的开头分配九个字作为存放分离空闲链表首节点地址的数组，初始为NULL。然后后面3个字用来保存序言块和结尾块，让`listp`指向大小类数组的起始位置，让`heap_listp`指向序言块的有效载荷，然后调用`expend_heap`来申请堆空间。

当分配器需要一个大小为 n 的块时，它就搜索相应的空闲链表。如果找到了一个，那么就分割它，并将剩余的部分插入到适当的空闲链表中。如果不能找到合适的块与之匹配，它就搜索下一个链表，以此类推。

如果空闲链表中没有合适的块，那么就向操作系统请求额外的堆内存，从这个新的堆内存中分配出一个块，将剩余部分放置在适当的大小类中。要释放一个块，我们执行合并，并将结果放置到相应的空闲链表中。

```c
// 在调用mm_malloc或者mm_free之前，应用必须先调用mm_init函数来初始化堆
int mm_init(void)
{
    // 从内存系统得到十二个字,九个字作为存放分离空闲链表首节点地址的数组，
    // 最后三个字作为填充块和序言块将数组与堆中实际的块分隔开
    
    if((heap_listp = (char*)mem_sbrk(12*WSIZE)) == (void *)-1)
        return -1;
    // 初始化大小类数组
    PUT(heap_listp, 0);// 16~31
    PUT(heap_listp + (1*WSIZE), 0);// 32~63
    PUT(heap_listp + (2*WSIZE), 0);// 64~127
    PUT(heap_listp + (3*WSIZE), 0);// 128~255
    PUT(heap_listp + (4*WSIZE), 0);// 256~511
    PUT(heap_listp + (5*WSIZE), 0);// 512~1023
    PUT(heap_listp + (6*WSIZE), 0);// 1023~2047
    PUT(heap_listp + (7*WSIZE), 0);// 2048~4095
    PUT(heap_listp + (8*WSIZE), 0);// 4096~inf

    // 序言块和结尾块
    PUT(heap_listp + (9*WSIZE), PACK(DSIZE,1));
    PUT(heap_listp + (10*WSIZE), PACK(DSIZE,1));
    PUT(heap_listp + (11*WSIZE), PACK(0,1));

    // listp指向大小类数组的起始位置， heap_listp指向序言块的有效载荷
    listp = heap_listp;
    heap_listp += (10*WSIZE);
    
    // 将这个堆扩展 CHUNKSIZE个字节，并创建初始的空闲块
    if(extend_heap(CHUNKSIZE/WSIZE) == NULL){
        return -1;
    }
    return 0;
}

static void *extend_heap(size_t words){
    char *bp;
    size_t size;
    // 向上舍入，分配偶数个字来保持双字对齐
    size = (words % 2) ? (words+1) * WSIZE : words*WSIZE;
    if((long)(bp=(char*)mem_sbrk(size))==-1){
        return NULL;
    }
    // mem_sbrk给堆新分配的内存片的地址在之前的结尾块的头部后面
    // 这个头部就变成了新的空闲块的头部，并且这个片的最后一个字变成了
    // 新的结尾块的头部,倒数第二个字变成了新的空闲块的尾部
    PUT(HDRP(bp),PACK(size,0));
    PUT(FTRP(bp),PACK(size,0));
    
    // 由于该空闲块还没有插入空闲链表中，设置bp的前驱和后继指针都为空，coalesce负责将空闲块加入空闲链表
    SET_PRED(bp,0);
    SET_SUCC(bp,0);
    // 结尾块的头部
    PUT(HDRP(NEXT_BLKP(bp)),PACK(0,1));

    // 如果前一个块是空闲块，就合并，并返回指向合并后的块的块指针
    return coalesce(bp);
}
// 合并空闲块，使用场景：一、释放块时；二、内存系统给堆分配新的块时
static void * coalesce(void *bp){
    // 获取前后两个相邻块的块指针和分配标志位
    void *prev_bp = PREV_BLKP(bp);
    void *next_bp = NEXT_BLKP(bp);
    size_t prev_alloc = GET_ALLOC(FTRP(PREV_BLKP(bp)));
    size_t next_alloc = GET_ALLOC(HDRP(NEXT_BLKP(bp)));

    size_t size = GET_SIZE(HDRP(bp));
    // 判断相邻的块是否为空闲块，如果是空闲块，就要将它们从空闲链表中移除
    // 然后与bp指向的块合并，再将合并后的空闲块加入显式空闲链表的头结点
    
    // 如果前后的两个块已分配
    if(prev_alloc && next_alloc){
        // pre_listp = (char*)bp;
    }
    else if (prev_alloc && !next_alloc){ // 如果前面的块已分配，后面的块未分配
        // 将相邻的空闲块从空闲链表中移除
        remove_from_free_list(next_bp);
        // 与bp指向的块合并
        size += GET_SIZE(HDRP(NEXT_BLKP(bp)));
        PUT(HDRP(bp), PACK(size,0));
        PUT(FTRP(bp), PACK(size,0));
    }
    else if (!prev_alloc && next_alloc){
        remove_from_free_list(prev_bp);
        size += GET_SIZE(HDRP(PREV_BLKP(bp)));
        PUT(FTRP(bp), PACK(size, 0));
        PUT(HDRP(PREV_BLKP(bp)), PACK(size, 0));
        bp = PREV_BLKP(bp);
    }
    else{
        remove_from_free_list(prev_bp);
        remove_from_free_list(next_bp);
        size += GET_SIZE(HDRP(PREV_BLKP(bp))) +
            GET_SIZE(HDRP(NEXT_BLKP(bp)));
        PUT(HDRP(PREV_BLKP(bp)), PACK(size, 0));
        PUT(FTRP(NEXT_BLKP(bp)), PACK(size, 0));
        bp = PREV_BLKP(bp);
    }
    // pre_listp = (char*)bp;
    // 将合并后的空闲块插入显式空闲链表的头结点
    insert_to_free_list(bp);
    return bp;
}
// 将指定的空闲块从空闲链表中移除
static void remove_from_free_list(void *bp){
    void *pred = (void *)PRED_FBLKP(bp);
    void *succ = (void *)SUCC_FBLKP(bp);

    size_t size = GET_SIZE(HDRP(bp));
    int index = get_index(size);
    void *root = listp + index*WSIZE;
    if(pred == root){
        PUT(pred,succ);
    }else{
        SET_SUCC(pred,succ);
    }

    if(succ != NULL){
        SET_PRED(succ,pred);
    }
    SET_PRED(bp,0);
    SET_SUCC(bp,0);

}
static void insert_to_free_list(void *bp){
    size_t size = GET_SIZE(HDRP(bp));
    int index = get_index(size);
    void *root = listp + index*WSIZE;
    LIFO(bp,root);
}
// 根据空闲块的大小，选择对应的大小类
static int get_index(size_t size){
    int index = 0;
    if(size >= 4096){
        return 8;
    }
    size = size >> 5;
    while(size > 0){
        size = size >> 1;
        index++;
    }
    return index;
}
// 将空闲块插入头结点
static void LIFO(void *bp,void *root){
    if(GET(root)!=0){
        // 将root后继的前驱变为bp，把bp的后继变为root的后继
        SET_PRED(GET(root),bp);
        SET_SUCC(bp,GET(root));
    }else{
        // 将bp的后继变为空
        SET_SUCC(bp,0);
    }
    PUT(root,bp);
    SET_PRED(bp,root);
}
/* 
 * mm_malloc - Allocate a block by incrementing the brk pointer.
 *     Always allocate a block whose size is a multiple of the alignment.
 *     size的单位是字节
 */
void *mm_malloc(size_t size)
{
    // asize和 extendsize的单位都是字节
    // 分配的块的大小
    size_t asize;
    // 如果没有匹配，堆应该扩展的大小
    size_t extendsize;
    char *bp;

    if(size == 0)
        return NULL;
    
    // 8字节用来放头部和脚部，而又不分配有效载荷为0的块，所以对齐后最小块的大小为16字节
    if(size <= DSIZE){
        asize = 2*DSIZE;
    }else{
        // 块的大小为：应用请求分配的有效载荷 加上 开销字节（头部和脚部）然后再向上舍入到最接近的8字节的整数倍
        asize = DSIZE * ((size + (DSIZE) + (DSIZE-1))/DSIZE);
    }

    // 分配器确定的了请求的块的大小后，搜索空闲链表来寻找一个合适的空闲块
    if((bp = (char *)first_fit(asize)) != NULL){
        // 如果有合适的，那么就在bp指向的位置放置asize大小的请求块，并可选地分割出多余的部分
        place(bp,asize);
        // 然后返回新分配块的地址
        return bp;
    }
    // 如果没有合适的空闲块，就向系统请求更多的空间，用一个新的空闲块来扩展堆，
    // 再把请求块放置在这个新的空闲块中，然后返回一个指针，指向这个新分配的块
    extendsize = MAX(asize,CHUNKSIZE);
    if((bp = (char *)extend_heap(extendsize/WSIZE)) == NULL)
        return NULL;
    place(bp, asize);
    return bp;
}
// 使用first fit策略来寻找空闲块
static void *first_fit(size_t asize){
    // 首先要确定asize大小的空闲块处于哪个大小类，然后搜索该大小类对应的显示空闲链表，
    // 如果找到合适的空闲块，则直接返回，如果没有找到，则遍历下一个大小类的显示空闲链表
    for(int index = get_index(asize);index <= 8;index++){
        void *root = listp + index*WSIZE;
        for(char* bp = GET(root); bp != 0;bp = SUCC_FBLKP(bp)){
            if(GET_SIZE(HDRP(bp)) >= asize){
                return bp;
            }
        }
    }
    return NULL;
}
static void * best_fit(size_t asize){
    return NULL;
}
// 在bp指向的空闲块处放置请求块，当剩余的部分大于等于最小块时，才将此空闲块分割
// 将bp指向的块从空闲链表中移除，将分割出来的空闲块与相邻的空闲块合并，合并完后插入空闲链表的头部
static void place(void* bp,size_t asize){
    remove_from_free_list(bp);
    size_t size = GET_SIZE(HDRP(bp));
    size_t remain_size = size - asize;
    if(remain_size >= MIN_BLOCK_SIZE){
        // bp块的头部中的块大小变成了asize
        PUT(HDRP(bp),PACK(asize,1));
        PUT(FTRP(bp),PACK(asize,1));
        bp = NEXT_BLKP(bp);
        PUT(HDRP(bp),PACK(remain_size,0));
        PUT(FTRP(bp),PACK(remain_size,0));
        // 设置bp的前驱和后继块为空
        SET_SUCC(bp,0);
        SET_PRED(bp,0);
        coalesce(bp);
    }else{
        // 将空闲块的分配标志位改为1即可
        PUT(HDRP(bp),PACK(size,1));
        PUT(FTRP(bp),PACK(size,1));
    }
    // pre_listp = (char*)bp;
}
/*
 * mm_free - Freeing a block does nothing.
 */
void mm_free(void *ptr)
{
    // 将要释放的块的头部和脚部的分配标志位改为0即可
    size_t size = GET_SIZE(HDRP(ptr));

    PUT(HDRP(ptr), PACK(size, 0));
    PUT(FTRP(ptr), PACK(size, 0));
    SET_PRED(ptr,0);
    SET_SUCC(ptr,0);
    // 释放时需要与相邻的空闲块合并
    coalesce(ptr);
}

/*
 * mm_realloc - Implemented simply in terms of mm_malloc and mm_free
 */
void *mm_realloc(void *ptr, size_t size)
{
    if(ptr == NULL)
        return mm_malloc(size);
    if(size == 0){
        mm_free(ptr);
        return NULL;
    }
    void *oldptr = ptr;
    void *newptr;
    size_t copySize;

    newptr = mm_malloc(size);
    if (newptr == NULL)
      return NULL;
    size = GET_SIZE(HDRP(oldptr));
    copySize = GET_SIZE(HDRP(newptr));
    if (size < copySize)
      copySize = size;
    memcpy(newptr, oldptr, copySize-WSIZE);
    mm_free(oldptr);
    return newptr;
}
```

**分离空闲链表+first fit：**

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211005154258260.png" alt="image-20211005154258260" style="zoom: 67%;" />


# MIT6.s081实验记录（三）：lab3

## Print a page table

打印页表内容

参考freewalk函数：

![image-20220110120411649](https://raw.githubusercontent.com/BoL0150/image2/master/image-20220110120411649.png)

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220110120457708.png" alt="image-20220110120457708" style="zoom:67%;" />

## A kernel page table per process

**修改内核让每一个用户进程在内核执行时使用它自己的内核页表的副本**，而不是使用一个全局页表。修改proc结构体来为每一个进程维护一个内核页表，修改调度程序使得切换内核线程时也切换内核页表。每个内核页表都与全局页表映射到相同的物理地址。

## Simplify `copyin`/`copyinstr`

内核的`copyin`函数将用户指针指向的内容读取到内核空间的指定地址。它通过软件模拟的方式，先调用`walkaddr`查找用户页表找到用户指针对应的物理地址，再读取物理地址中的内容来实现。

在本实验中，我们需要简化`copyin`函数，**将每个进程的用户空间的映射添加到每个进程的内核页表**。**即不仅要求用户空间的物理地址在进程的内核页表和用户页表都建立映射，并且要映射到页表中的同一个虚拟地址**。以允许copyin不需要通过软件模拟翻译，而是直接通过硬件解引用用户页表的虚拟地址。

此方案要求用户的虚拟地址范围不与内核使用的虚拟地址的范围重叠，**所以用户进程的虚拟地址最大大小限制为小于内核的最低虚拟地址**。内核启动后，在XV6中该地址是`0xC000000`，即PLIC寄存器的地址；我们需要防止用户进程增长到超过PLIC的地址。

> 在 `memlayout.h` 可看到CLINT的值为0x2000000L，这个值比0xC000000小，0xC000000是PLIC（Platform-Level Interrput Controller，中断控制器）的地址，实际上，CLINT（Core Local Interruptor）定义了本地中断控制器的地址，我们只会在内核初始化的时候用到这段地址。所以，为用户进程生成内核页表的时候，可以不必映射这段地址。用户页表是从虚拟地址0开始，用多少就建多少，但最高地址不能超过内核的起始地址，这样用户的可用虚拟地址空间就为0x0 - 0xC000000。

- 每次内核修改用户页表时，都要将修改复制到进程的内核页表。包括`fork()`, `exec()`, 和`sbrk()`.

  - 先实现一个`u2kvmcopy`来将指定虚拟地址范围的user page table复制到process kernel page table，注意在复制的过程中需要清除原先PTE中的PTE_U标志位，否则kernel无法访问

    ```c
    void
    u2kvmcopy(pagetable_t pagetable, pagetable_t kpagetable, uint64 lowwer, uint64 upper){
      pte_t *pte_from, *pte_to;
      if(upper < lowwer){
        return;
      }
      lowwer = PGROUNDUP(lowwer);
      for(uint64 a = lowwer; a < upper; a += PGSIZE){
        if((pte_from = walk(pagetable, a, 0)) == 0)
          panic("pte should exist");
        if((pte_to = walk(kpagetable, a, 1)) == 0)
          panic("walk fails");
        uint64 pa = PTE2PA(*pte_from);
        uint flags = (PTE_FLAGS(*pte_from) & ~PTE_U);
        *pte_to = PA2PTE(pa) | flags;
      }
    }
    ```

  - 在`fork()`, `exec()`, `sbrk()`以及`userinit`中调用`u2kvmcopy`（第一个进程由userinit创建，其他的用户进程由fork和exec创建）

  

# lab4

## Backtrace

当发生错误时，backtrace打印当前函数的函数调用关系，即递归地打印这个函数的父函数的返回地址。

```
backtrace:
0x0000000080002cda
0x0000000080002bb6
0x0000000080002898
// 实际是以下地址
kernel/sysproc.c:74
kernel/syscall.c:224
kernel/trap.c:85
```

 **编译器在每一个栈帧中放置一个帧指针（frame pointer）保存调用者帧的地址**，也就是指向调用者帧的顶部。GCC编译器将**当前**正在执行的函数的帧指针（即帧的地址）保存在`s0`寄存器。返回地址位于栈帧帧指针的固定偏移(-8)位置，并且保存的帧指针位于帧指针的固定偏移(-16)位置。

你的`backtrace`应当使用这些帧指针来遍历栈，并在每个栈帧中打印保存的返回地址。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/p2.png" alt="img" style="zoom: 50%;" />

XV6在内核中以页面对齐的地址为每个栈分配一个页面。你可以通过`PGROUNDDOWN(fp)`和`PGROUNDUP(fp)`来计算栈页面的顶部和底部地址。这些数字对于`backtrace`终止循环是有帮助的。

```c
void
backtrace(void){
  printf("backtrace:");
  uint64 fp = r_fp();
  uint64 top = PGROUNDUP(fp);
  uint64 bottom = PGROUNDDOWN(fp);
  while(fp < top && fp > bottom){
    // fp - 8是储存ra的内存地址，要对它进行解引用才能得到ra的值
    // 所以我们要先将地址转换为uint64的指针类型才能解引用
    uint64 ra = *(uint64 *)(fp - 8);
    fp = *(uint64 *)(fp - 16);
    printf("%p\n",ra);
  }
}
```

## Alarm

添加一个新的`sigalarm(interval, handler)`系统调用，如果一个程序调用了`sigalarm(n, fn)`，那么每当程序消耗了CPU时间达到n个“滴答”，内核应当使应用程序函数`fn`被调用。当`fn`返回时，应用应当在它离开的地方恢复执行。

首先修改Makefile使得alarmtest.c能够被编译，在user/user.h中添加函数声明

```c
int sigalarm(int ticks, void (*handler)()); 
int sigreturn(void); 
```

在user/user.pl、kernel/syscall.h、kernel/syscall.c等添加`sys_sigalarm`和`sys_sigreturn`这两个syscall的注册

在kernel/proc.h的proc结构体中添加几个成员。

- `int interval`是保存的定时器触发的周期，
- `void (*handler)()`是指向handler函数的指针

这两个成员变量在`sys_sigalarm`中被保存（分别是系统调用的两个参数，从a0和a1寄存器获取）。

还需要在proc结构体中添加`ticks`成员，用来记录上一次调用sigalarm后到现在的ticks；`in_handler`用来记录当前是否在handler函数中，用来防止在handler函数的过程中定时被触发再次进入handler函数

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220125213545719.png" alt="image-20220125213545719" style="zoom:50%;" />

用户调用sigalarm，先进入到内核的`sys_sigalarm`中，需要将从userfunction传入的两个参数分别保存到`p`结构体相应的成员变量中

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220125214513313.png" alt="image-20220125214513313" style="zoom:50%;" />

再回到用户空间中继续执行该进程。执行了一段时间后，进程所在的CPU核发生定时器中断，进入内核，在usertrap中处理定时器的中断。

该进程所在的CPU核每发生一次定时器中断，就将ticks加1 。直到ticks等于指定的interval，将ticks清空，trapframe中的epc置为handler的地址。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220125215212973.png" alt="image-20220125215212973" style="zoom: 50%;" />

然后通过usertrapret返回到用户空间中epc的位置，也就是handler的地址。handler函数的最后，再调用sigreturn系统调用，再次进入内核中的sys_sigreturn函数。

此时我们想要返回到用户程序最后一次被计时器中断的指令处执行，然而我们已经丢失了这个指令的上下文。所以在处理定时器中断时，我们在把handler的地址放入trapframe的epc中之前，**需要将原来的上下文保存在某一个位置，这样以后调用sigreturn时才能返回**。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220125221037398.png" alt="image-20220125221037398" style="zoom:50%;" />

```c
 syscall();
  } else if((which_dev = devintr()) != 0){
    // ok
    if (which_dev == 2 && p->in_handler == 0) {
      p->ticks += 1;
      if ((p->ticks == p->interval) && (p->interval != 0)) {
        p->in_handler = 1;
        p->ticks = 0;
        p->saved_epc = p->trapframe->epc;
        p->saved_ra = p->trapframe->ra;
        p->saved_sp = p->trapframe->sp;
        p->saved_gp = p->trapframe->gp;
        p->saved_tp = p->trapframe->tp;
        p->saved_t0 = p->trapframe->t0;
        p->saved_t1 = p->trapframe->t1;
        p->saved_t2 = p->trapframe->t2;
        p->saved_t3 = p->trapframe->t3;
        p->saved_t4 = p->trapframe->t4;
        p->saved_t5 = p->trapframe->t5;
        p->saved_t6 = p->trapframe->t6;
        p->saved_s0 = p->trapframe->s0;
        p->saved_s1 = p->trapframe->s1;
        p->saved_s2 = p->trapframe->s2;
        p->saved_s3 = p->trapframe->s3;
        p->saved_s4 = p->trapframe->s4;
        p->saved_s5 = p->trapframe->s5;
        p->saved_s6 = p->trapframe->s6;
        p->saved_s7 = p->trapframe->s7;
        p->saved_s8 = p->trapframe->s8;
        p->saved_s9 = p->trapframe->s9;
        p->saved_s10 = p->trapframe->s10;
        p->saved_s11 = p->trapframe->s11;
        p->saved_a0 = p->trapframe->a0;
        p->saved_a1 = p->trapframe->a1;
        p->saved_a2 = p->trapframe->a2;
        p->saved_a3 = p->trapframe->a3;
        p->saved_a4 = p->trapframe->a4;
        p->saved_a5 = p->trapframe->a5;
        p->saved_a6 = p->trapframe->a6;
        p->saved_a7 = p->trapframe->a7;
        p->trapframe->epc = (uint64)p->handler;
      }
    }
  } else {
```

```c
uint64
sys_sigreturn(void)
{
  struct proc *p = myproc();
  p->trapframe->epc = p->saved_epc;
  p->trapframe->ra = p->saved_ra;
  p->trapframe->sp = p->saved_sp;
  p->trapframe->gp = p->saved_gp;
  p->trapframe->tp = p->saved_tp;
  p->trapframe->t0 = p->saved_t0;
  p->trapframe->t1 = p->saved_t1;
  p->trapframe->t2 = p->saved_t2;
  p->trapframe->t3 = p->saved_t3;
  p->trapframe->t4 = p->saved_t4;
  p->trapframe->t5 = p->saved_t5;
  p->trapframe->t6 = p->saved_t6;
  p->trapframe->s0 = p->saved_s0;
  p->trapframe->s1 = p->saved_s1;
  p->trapframe->s2 = p->saved_s2;
  p->trapframe->s3 = p->saved_s3;
  p->trapframe->s4 = p->saved_s4;
  p->trapframe->s5 = p->saved_s5;
  p->trapframe->s6 = p->saved_s6;
  p->trapframe->s7 = p->saved_s7;
  p->trapframe->s8 = p->saved_s8;
  p->trapframe->s9 = p->saved_s9;
  p->trapframe->s10 = p->saved_s10;
  p->trapframe->s11 = p->saved_s11;
  p->trapframe->a0 = p->saved_a0;
  p->trapframe->a1 = p->saved_a1;
  p->trapframe->a2 = p->saved_a2;
  p->trapframe->a3 = p->saved_a3;
  p->trapframe->a4 = p->saved_a4;
  p->trapframe->a5 = p->saved_a5;
  p->trapframe->a6 = p->saved_a6;
  p->trapframe->a7 = p->saved_a7;
  p->in_handler = 0;
  return 0;
}
```

用户程序调用sigalarm进入内核，告知内核需要跟踪多少次tick，以及handler函数地址，然后返回用户空间。在内核的该进程的结构体中维护一个字段，记录该进程的中断次数。该进程以后每一次遇到定时器中断到内核中，都会增加这个字段，直到满足interval。然后就保存用户进程的上下文，进入到另一个用户程序handler。handler的最后会调用sigreturn，再次进入内核。在sigreturn中，将之前保存的用户进程上下文恢复到trapframe中，回到用户进程继续执行。

这实际上类似于用户级中断处理程序，

# Lab5: xv6 lazy page allocation

此lab实现lazy allocation。

- 核心思想非常简单，sbrk系统调基本上不做任何事情，唯一需要做的事情就是将*p->sz*增加n，其中n是需要新分配的内存page数量。内核此时并不会分配任何物理内存。之后如果应用程序使用到了新申请的**超过当前page的**那部分内存，这时因为我们还没有将新的内存映射到page table，会触发page fault。进入trap处理程序后通过`kalloc`再分配一页物理page，将这个page的内容初始化为0，将它映射到页表中被访问到的那部分内存。随后再重新执行指令

## Lazy allocation 

在`sbrk()`时只增长进程的`myproc()->sz`而不实际分配内存，再修改usertrap中的代码以响应来自用户空间的page fault，新分配一个物理页面并映射到发生错误的虚拟地址，然后返回到用户空间，让进程继续执行。

相应的虚拟地址可以通过`r_stval()`获得。`r_scause()`为13或15表明trap的原因是page fault

具体实现：

修改sys_sbrk系统调用，让他只对p->sz加n，并不执行增加内存的操作。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220204172423883.png" alt="image-20220204172423883" style="zoom:50%;" />

再修改usertrap中处理page fault的代码：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220204173126161.png" alt="image-20220204173126161" style="zoom:50%;" />

如果scause为15，表示因为写操作发生了page fault，此时需要进行lazy allocation（因为新分配的空间**只会先写入内容**，再进行别的操作）。首先从stval寄存器中获取出现page fault的虚拟地址，再调用kalloc分配一个物理内存page

- 如果ka等于0，表明没有物理内存我们现在OOM（Out Of Memory）了，我们会杀掉进程。
- 如果分配了物理内存，首先会将内存内容设置为0。之后将物理page映射到用户地址空间中合适的虚拟内存地址。
  - 具体地，我们先将虚拟地址向下取整，这里引起page fault的虚拟地址是0x4008，向下取整之后是0x4000。
  - 之后我们将新分配的物理内存地址跟取整之后的虚拟内存地址的关系映射到page table中。对应的PTE需要设置常用的权限标志位，在这里是u，w，r bit位。

除此之外，在进程结束后释放页表和页表映射的物理内存时，还会出现问题。

```c
void
proc_freepagetable(pagetable_t pagetable, uint64 sz)
{
  uvmunmap(pagetable, TRAMPOLINE, 1, 0);
  uvmunmap(pagetable, TRAPFRAME, 1, 0);
  uvmfree(pagetable, sz);
}
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0)
      panic("uvmunmap: not mapped");
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}
```

这段代码释放了用户页表中从0到p->sz的所有页，

- 对lazy allocation，sbrk除了增加p->sz什么也没做，所以从旧的p->sz到新的p->sz的虚拟页并没有建立映射，所以返回的pte的valid为0，会出现panic。
- 对于之前的eager allocation则永远不会出现p->sz以下的用户内存没有映射的情况

所以对于lazy allocation，此处没有映射到物理内存，我们不需要对这个page做任何事情，可以直接continue跳到下一个page

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220205014744680.png" alt="image-20220205014744680" style="zoom:50%;" />

## Lazytests and Usertests

上面的代码并不完善，我们还需要做：

- 处理`sbrk()`参数为负的情况。

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220204225208655.png" alt="image-20220204225208655" style="zoom:50%;" />

- 按照上面的做法，所有的page fault都被我们当成lazy allocation处理了。但实际上，有一些page fault应该直接杀死进程。

  - 判断出现page fault的地址是否是新申请的虚拟地址，如果某个进程在高于`sbrk()`分配的虚拟内存地址（`p->sz`）以上，或者在进程的用户栈的guard  page中（guard page是用户不可访问的）出现page fault时，则终止该进程。

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220205163255599.png" alt="image-20220205163255599" style="zoom:50%;" />

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220205163332019.png" alt="image-20220205163332019" style="zoom:50%;" />

- `fork()`中将父进程的页表和物理内存复制给子进程的过程中用到了`uvmcopy`，`uvmcopy`原本在调用walk函数查找虚拟地址对应的PTE的时候

  - 发现缺失相应的PTE的二级页表或三级页表的页表页，导致无法查找到对应的PTE，walk返回0
  - 查找到了PTE，但是PTE为invalid

  （本质上都是这个虚拟地址没有被映射）会panic，这里也要`continue`。

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220204231625117.png" alt="image-20220204231625117" style="zoom: 67%;" />

- **上面修改了usertrap，是在用户空间中发生page fault时的处理流程**，但是如果用户向系统调用（如`write`或`read`）传递sbrk增长的、但是还没有分配物理页的地址，就会在**内核**中发生page fault。我们需要处理这种情况：在内核解析某个用户空间的虚拟地址的时候，如果这个虚拟地址在合法的范围内，就触发一次分配内存。

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220205163715605.png" alt="image-20220205163715605" style="zoom:50%;" />

### 用户栈

用户栈的guard page不是invalid的，否则调用uvmcopy复制到guard page时也会panic，而是分配了PTE，但是被设置为用户无法访问。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220205114925340.png" alt="image-20220205114925340" style="zoom:50%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220205115033536.png" alt="image-20220205115033536" style="zoom:50%;" />

# Lab6: Copy-on-Write Fork for xv6

xv6中的`fork()`系统调用将父进程的所有用户空间内存复制到子进程中。如果父进程较大，则复制可能需要很长时间。更糟糕的是，这项工作经常造成大量浪费；例如，子进程中的`fork()`后跟`exec()`将导致子进程丢弃复制的内存

COW fork()只为子进程创建一个页表，用户内存的PTE指向父进程的物理页。COW fork()将父进程和子进程中的所有用户PTE标记为不可写。当任一进程试图写入其中一个COW页时，CPU将强制产生页面错误。内核页面错误处理程序检测到这种情况将为出错进程分配一页物理内存，将原始页复制到新页中，并修改出错进程中的相关PTE指向新的页面，将PTE标记为可写。当页面错误处理程序返回时，用户进程将能够写入其页面副本。

COW fork()将使得释放用户内存的物理页面变得更加棘手。给定的物理页可能会被多个进程的页表引用，并且只有在最后一个引用消失时才应该被释放。

- 使用PTE中的RSW（reserved for software，即为软件保留的）位来记录每个PTE是否是COW映射。

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220205210305800.png" alt="image-20220205210305800" style="zoom:50%;" />

- fork调用uvmcopy复制父进程的内存时不立刻复制物理内存，而是建立指向原物理页的映射，并将父子两端的页表项都设置为不可写和COW

  ```c
  int
  uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
  {
    pte_t *pte;
    uint64 pa, i;
    uint flags;
    // char *mem;
  
    for(i = 0; i < sz; i += PGSIZE){
      if((pte = walk(old, i, 0)) == 0)
        panic("uvmcopy: pte should exist");
      if((*pte & PTE_V) == 0)
        panic("uvmcopy: page not present");
      pa = PTE2PA(*pte);
      // 清除父进程的所有pte的写标志位，加上COW标志位
      *pte = (*pte & ~PTE_W) | PTE_COW;
      flags = PTE_FLAGS(*pte);
      
      // 不需要为子进程分配物理内存
      // if((mem = kalloc()) == 0)
      //   goto err;
      // memmove(mem, (char*)pa, PGSIZE);
  
      // 将父进程的物理地址直接映射到子进程，权限设置为和父进程一致，父子进程都是不可写，cow
      if(mappages(new, i, PGSIZE, (uint64)pa, flags) != 0){
        // kfree(mem);
        goto err;
      }
      // 将物理页的引用计数加一
      krefpage((void*)pa);
    }
    return 0;
  
   err:
    uvmunmap(new, 0, i / PGSIZE, 1);
    return -1;
  }
  ```

- 这样，fork 时就不会立刻复制内存，只会创建一个映射了。这时候如果尝试修改懒复制的页，会出现 page fault 被 usertrap() 捕获。接下来需要在 usertrap() 中处理这个 page fault。

  在 usertrap() 中添加对 page fault 的检测，并在当前访问的地址符合COW条件时，对懒复制页进行实复制操作

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220205212843891.png" alt="image-20220205212843891" style="zoom:50%;" />

- 在内核中的copyout() 由于是内核软件模拟访问页表，不会触发缺页异常，所以需要手动添加同样的监测代码（同 lab5），检测接收的页是否是一个懒复制页，若是，执行实复制操作：

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220205213748224.png" alt="image-20220205213748224" style="zoom:50%;" />

- 实现懒复制页的检测（`iscowpage()`）

  ```c
  // kernel/vm.c
  // 检查一个地址指向的页是否是懒复制页
  int iscowpage(uint64 va) {
    pte_t *pte;
    struct proc *p = myproc();
    
    return va < p->sz // 在进程内存范围内
      && ((pte = walk(p->pagetable, va, 0))!=0)
      && (*pte & PTE_V) // 页表项存在
      && (*pte & PTE_COW); // 页是一个懒复制页
  }
  ```

- 触发page fault后实际复制（`cowcopy()`）的操作：首先调用`kalloc.c` 中的 `kcopy_n_deref` 方法将va指向的物理页复制到一个新的物理页，将原来的物理页的引用计数减一，返回新的物理页的指针。再将va对应的pte映射到新的物理页，标志位改为可写，并去掉COW。

  ```c
  // 实复制一个懒复制页，并重新映射为可写
  int cowcopy(uint64 va) {
    pte_t *pte;
    struct proc *p = myproc();
  
    if((pte = walk(p->pagetable, va, 0)) == 0)
      panic("uvmcowcopy: walk");
    
    // 调用 kalloc.c 中的 kcopy_n_deref 方法，复制页
    // (如果懒复制页的引用已经为 1，，只需清除 PTE_COW 标记并标记 PTE_W 即可)
    uint64 pa = PTE2PA(*pte);
    uint64 new = (uint64)kcopy_n_deref((void*)pa); // 将一个懒复制的页引用变为一个实复制的页
    if(new == 0)
      return -1;
    
    // 重新映射为可写，并清除 PTE_COW 标记
    uint64 flags = (PTE_FLAGS(*pte) | PTE_W) & ~PTE_COW;
    uvmunmap(p->pagetable, PGROUNDDOWN(va), 1, 0);
    if(mappages(p->pagetable, va, 1, new, flags) == -1) {
      panic("uvmcowcopy: mappages");
    }
    return 0;
  }
  ```

  

### 物理页生命周期以及引用计数

在 kalloc.c 中，我们需要定义一系列的新函数，用于完成在支持懒复制的条件下的物理页生命周期管理。

在原本的 xv6 实现中，一个物理页的生命周期内，可以支持以下操作：

- kalloc(): 分配物理页
- kfree(): 释放回收物理页

而在支持了懒分配后，由于一个物理页可能被多个进程（多个虚拟地址）引用，并且必须在最后一个引用消失后才可以释放回收该物理页，所以一个物理页的生命周期内，现在需要支持以下操作：

- kalloc(): 分配物理页，将其引用计数置为 1
- krefpage(): 创建物理页的一个新引用，引用计数加 1
- kcopy_n_deref(): 将物理页的内容复制到一个新的物理页上（引用计数为 1），返回得到的副本页；并将本物理页的引用计数减 1。如果物理页的引用计数为1，就直接返回这个物理页。
- kfree(): 释放物理页的一个引用，引用计数减 1；如果计数变为 0，则释放回收物理页

一个物理页 p 首先会被父进程使用 kalloc() 创建，将引用计数置为1 。fork 的时候，新创建的子进程每将一个PTE映射到父进程的物理页的时候，都会使用 krefpage() 增加父进程物理页的引用计数。当尝试修改父进程或子进程中的页时，kcopy_n_deref() 负责将想要修改的物理页复制到独立的副本，并记录解除旧的物理页的引用（引用计数减 1）。最后 kfree() 保证只有在所有的引用者都释放该物理页的引用时，才释放回收该物理页。

```c
#define PA2PGREF_ID(p) (((p)-KERNBASE)/PGSIZE)
#define PGREF_MAX_ENTRIES PA2PGREF_ID(PHYSTOP)

struct spinlock pgreflock;  // 用于pageref数组的锁
int pageref[PGREF_MAX_ENTRIES]; // 从KERNBASE到PHYSTOP之间每个物理页的引用计数

// 通过物理地址获得引用计数
#define PA2PGREF(p) pageref[PA2PGREF_ID((uint64)(p))]

struct run {
  struct run *next;
};

struct {
  struct spinlock lock;
  struct run *freelist;
} kmem;

void
kinit()
{
  initlock(&kmem.lock, "kmem");
  initlock(&pgreflock,"pgref");
  freerange(end, (void*)PHYSTOP);
}

void
freerange(void *pa_start, void *pa_end)
{
  char *p;
  p = (char*)PGROUNDUP((uint64)pa_start);
  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE)
    kfree(p);
}

// Free the page of physical memory pointed at by v,
// which normally should have been returned by a
// call to kalloc().  (The exception is when
// initializing the allocator; see kinit above.)
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  acquire(&pgreflock);
  if(--PA2PGREF(pa) <= 0) {
    // 当页面的引用计数小于等于 0 的时候，释放页面

    // Fill with junk to catch dangling refs.
    // pa will be memset multiple times if race-condition occurred.
    memset(pa, 1, PGSIZE);

    r = (struct run*)pa;

    acquire(&kmem.lock);
    r->next = kmem.freelist;
    kmem.freelist = r;
    release(&kmem.lock);
  }
  release(&pgreflock);
}

// Allocate one 4096-byte page of physical memory.
// Returns a pointer that the kernel can use.
// Returns 0 if the memory cannot be allocated.
void *
kalloc(void)
{
  struct run *r;

  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r)
    kmem.freelist = r->next;
  release(&kmem.lock);

  if(r){
    memset((char*)r, 5, PGSIZE); // fill with junk
    PA2PGREF(r) = 1;
  }
  return (void*)r;
}
// 当引用已经小于等于 1 时，不创建和复制到新的物理页，而是直接返回该页本身
void *kcopy_n_deref(void *pa) {
  acquire(&pgreflock);

  if(PA2PGREF(pa) <= 1) { // 只有 1 个引用，无需复制
    release(&pgreflock);
    return pa;
  }

  // 分配新的内存页，并复制旧页中的数据到新页
  uint64 newpa = (uint64)kalloc();
  if(newpa == 0) {
    release(&pgreflock);
    return 0; // out of memory
  }
  memmove((void*)newpa, (void*)pa, PGSIZE);

  // 旧页的引用减 1
  PA2PGREF(pa)--;

  release(&pgreflock);
  return (void*)newpa;
}

// 为 pa 的引用计数增加 1
void krefpage(void *pa) {
  acquire(&pgreflock);
  PA2PGREF(pa)++;
  release(&pgreflock);
}
```

# Lab7: Multithreading

内核调度器无论是通过时钟中断进入（usertrap），还是线程自己主动放弃 CPU（sleep、exit），最终都会调用到 yield 进一步调用 swtch。 由于上下文切换永远都发生在函数调用的边界（swtch 调用的边界），恢复执行相当于是 swtch 的返回过程，会从堆栈中恢复 caller-saved 的寄存器， 所以用于保存上下文的 context 结构体只需保存 callee-saved 寄存器，以及 返回地址 ra、栈指针 sp 即可。恢复后执行到哪里是通过 ra 寄存器来决定的（swtch 末尾的 ret 转跳到 ra）

而 trapframe 则不同，一个中断可能在任何地方发生，不仅仅是函数调用边界，也有可能在函数执行中途，所以恢复的时候需要靠 pc 寄存器来定位。 并且由于切换位置不一定是函数调用边界，所以几乎所有的寄存器都要保存（无论 caller-saved 还是 callee-saved），才能保证正确的恢复执行。 这也是内核代码中 `struct trapframe` 中保存的寄存器比 `struct context` 多得多的原因。

# Lab8: locks

## Memory allocator

kalloc 原本的实现中，使用 freelist 链表，将空闲物理页**本身**直接用作链表项（这样可以不使用额外空间）连接成一个链表，在分配的时候，将物理页从链表中移除，回收时将物理页放回链表中。

在这里无论是分配物理页或释放物理页，都需要修改 freelist 链表。由于修改是多步操作，为了保持多线程一致性，必须加锁。但这样的设计也**使得多线程无法并发申请物理内存**，限制了并发效率。

我们重新设计内存分配器，以避免使用单个锁和列表，从而消除锁的争用。

这里解决性能热点的思路是「将共享资源变为不共享资源」。锁竞争优化一般有几个思路：

- 只在必须共享的时候共享（对应为将资源从 CPU 共享拆分为每个 CPU 独立）
- 必须共享时，尽量减少在关键区中停留的时间（对应“大锁化小锁”，降低锁的粒度）

基本思想是：

- **为每个 CPU 分配独立的 freelist**，这样多个 CPU 并发分配物理页就不再会互相排斥了，提高了并行性。
- 当一个CPU的空闲列表为空，而另一个CPU的列表有空闲内存的情况下，一个CPU必须“窃取”另一个CPU空闲列表的一部分。所以一个 CPU 的 freelist 并不是只会被其对应 CPU 访问，还可能在“偷”内存页的时候被其他 CPU 访问，故仍然需要使用单独的锁来保护每个 CPU 的 freelist

```c
// kernel/kalloc.c
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem[NCPU]; // 为每个 CPU 分配独立的 freelist，并用独立的锁保护它。

char *kmem_lock_names[] = {
  "kmem_cpu_0",
  "kmem_cpu_1",
  "kmem_cpu_2",
  "kmem_cpu_3",
  "kmem_cpu_4",
  "kmem_cpu_5",
  "kmem_cpu_6",
  "kmem_cpu_7",
};

void
kinit()
{
  for(int i=0;i<NCPU;i++) { // 初始化所有锁
    initlock(&kmem[i].lock, kmem_lock_names[i]);
  }
  freerange(end, (void*)PHYSTOP);
}
```

```c
// kernel/kalloc.c
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  push_off();

  int cpu = cpuid();

  acquire(&kmem[cpu].lock);
  r->next = kmem[cpu].freelist;
  kmem[cpu].freelist = r;
  release(&kmem[cpu].lock);

  pop_off();
}

void *
kalloc(void)
{
  struct run *r;

  push_off();

  int cpu = cpuid();

  acquire(&kmem[cpu].lock);

  if(!kmem[cpu].freelist) { // no page left for this cpu
    int steal_left = 64; // steal 64 pages from other cpu(s)
    for(int i=0;i<NCPU;i++) {
      if(i == cpu) continue; // no self-robbery
      acquire(&kmem[i].lock);
      struct run *rr = kmem[i].freelist;
      while(rr && steal_left) {
        kmem[i].freelist = rr->next;
        rr->next = kmem[cpu].freelist;
        kmem[cpu].freelist = rr;
        rr = kmem[i].freelist;
        steal_left--;
      }
      release(&kmem[i].lock);
      if(steal_left == 0) break; // done stealing
    }
  }

  r = kmem[cpu].freelist;
  if(r)
    kmem[cpu].freelist = r->next;
  release(&kmem[cpu].lock);

  pop_off();

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}

```

全局维护一个空闲链表数组，数组槽位数量为NCPU，每个槽位都维护一个freelist，对应一个CPU核。内核启动初始化物理内存（的freelist）时，直接调用freerange将每个物理页都插入到同一个freelist中。

释放物理页时，先获取当前CPU的id，在kmem数组中找到当前CPU的freelist，获取这个槽位的锁，将物理页插入到当前CPU的freelist中。

分配物理页时，先获取当前CPU的id

- 如果当前CPU的freelist为空，就遍历kmem数组，找到一个非空的freelist。这里选择在内存页不足的时候，从其他的 CPU “偷” 64 个页。从这个freelist中移除物理页，插入到当前CPU的freelist中。如果这个freelist也空了，则选择kmem数组的下一个freelist继续偷物理页，直到偷完64个页。

- 如果当前CPU的freelist不为空，就在当前CPU的freelist中分配物理页

## Buffer cache

锁竞争优化一般有几个思路：

- 只在必须共享的时候共享
  - 对kalloc来说，每次调用都是随机分配物理页，没有必要在CPU之间共享freelist，所以可以将freelist拆分为每个CPU独立
- 必须共享时，尽量减少在关键区中停留的时间（对应“大锁化小锁”，降低锁的粒度）
  - 对于bcache来说，一个buffer可能被多个进程访问，属于必须共享的情况，所以 kmem 中为每个 CPU 预先分割一部分专属的页的方法在这里是行不通的。所以我们需要降低锁的粒度，用更精细的锁来降低出现竞争的概率。

原版 xv6 的设计中，使用双向链表存储所有的区块缓存，每次尝试获取一个区块 blockno 的时候，会遍历链表，如果目标区块已经存在缓存中则直接返回，如果不存在则选取一个最近最久未使用的，且引用计数为 0 的 buf 块作为其区块缓存，并返回。

新的改进方案，可以**建立一个从 blockno 到 buf 的哈希表，并为每个桶单独加锁**。这样，仅有在两个进程同时访问的区块同时哈希到同一个桶的时候，才会发生锁竞争。当桶中的空闲 buf 不足的时候，从其他的桶中获取 buf。

- 对比bustub的BPM：BPM和LRUreplacer各用一个大锁，只要对BPM进行任何操作（比如fetchpage或者Unpinpage）就要进行加锁，结束操作后释放锁。对LRUReplacer进行任何操作也要进行加锁，结束操作后释放锁（理论上LRUReplacer在BPM的内部，既然对BPM加了锁就不需要对LRUReplacer再加锁，但是测试时会对LRUReplacer单独测试）

给定一个blockno，通过hash算法，找到所在的桶，对该桶上锁。在该桶的链表中查找指定blockno的buffer。如果没找到buffer，则保留key桶的锁，从该桶后面开始遍历hashtable的每个槽位。

- 对每个槽位先上锁，再遍历内部的buffer list，找到这个list中lru的buffer。

直到将hashmap中的所有桶遍历完，找到lru的buffer。将这个buffer从所在的list中移除，添加到key桶的list中。

这里的问题是：我们发现 blockno 不存在缓存中之后，需要在拿着 key 桶锁的同时，**遍历所有的桶并依次获取它们每个的锁**。可能会出现： CPU1 持有锁 2 的情况下去申请锁 5，而 CPU2 持有锁 5 的情况下申请锁 2，造成了**环路等待**。

死锁的四个条件：

1. 互斥（一个资源在任何时候只能属于一个线程）
2. 请求保持（线程在拿着一个锁的情况下，去申请另一个锁）
3. 不剥夺（外力不强制剥夺一个线程已经拥有的资源）
4. 环路等待（请求资源的顺序形成了一个环）

为了避免这样的死锁，我们添加 eviction_lock，将查找LRU块+驱逐的过程限制为单线程

注意此处应该先释放桶锁后，再获取 eviction_lock。写反会导致 eviction_lock 和桶锁发生死锁。

- 线程 1 拿着桶 A 锁查找LRU块，此时需要请求 eviction_lock； 线程 2 正在查找LRU块，此时拿着 eviction_lock ，遍历查找到桶A，需要请求桶 A 的锁

这种情况就会造成桶A的锁和eviction_lock之间的空窗期。

- 线程1释放了A的锁，再获取eviction_lock后，如果此时有线程2要获取和线程1相同的blockno，也会马上获取A的锁，随后会等待eviction_lock，准备查找LRU块。线程1找到LRU块后，释放eviction_lock，将LRU块插入桶A，标记为blockno；线程2随后获取eviction_lock，又会查找到一个LRU块，再次插入到桶A。这就**会导致桶A中有两块相同blockno的buffer**。

解决方案是：获取了eviction_lock之后，马上**再次判断 blockno 的缓存是否存在**，若是直接返回，若否继续执行

各个锁的作用：

- bufmap_locks 保护单个桶的链表结构，以及桶内所有节点的 refcnt
- eviction_lock 保护所有桶的链表结构，但是不保护任何 refcnt

驱逐过程中，首先需要拿 eviction_lock，使得可以遍历所有桶的链表结构。然后遍历链表结构寻找可驱逐块的时候，由于在某个桶i中判断是否有可驱逐块的过程需要读取 refcnt，所以需要再拿该桶的 bufmap_locks[i]。

# Lab9: file system

目前，xv6文件限制为268个块或`268*BSIZE`字节（在xv6中`BSIZE`为1024）。此限制来自以下事实：一个xv6 inode包含12个“直接”块号和一个“间接”块号，“一级间接”块指一个最多可容纳256个块号的块，总共12+256=268个块。

您将更改xv6文件系统代码，以支持每个inode中可包含256个一级间接块地址的“二级间接”块，每个一级间接块最多可以包含256个数据块地址。结果将是一个文件将能够包含多达65803个块，或256*256+256+11个块

主要修改在指定inode中将逻辑块号映射到物理块号的过程（bmap），以及修改itrunc释放文件的所有块，包括二级间接块

## Symbolic links

符号链接（或软链接）是指按路径名链接的文件；当一个符号链接打开时，内核跟随该链接指向引用的文件。符号链接类似于硬链接，但硬链接仅限于指向同一磁盘上的文件，而符号链接可以跨磁盘设备。

- 实现`symlink(target, path)`系统调用，以在`path`处创建一个新的指向`target`的符号链接。请注意，系统调用的成功不需要`target`已经存在。
  - 实际上就是在path处创建一个inode，类型为T_SYMLINK，再将target字符串写入这个inode的第一个数据块中。
- 修改`open`系统调用以处理路径指向符号链接的情况。如果文件不存在，则打开必须失败。当进程向`open`传递`O_NOFOLLOW`标志时，`open`应打开符号链接（而不是跟随符号链接）。
- 如果链接文件也是符号链接，则必须递归地跟随它，直到到达非链接文件为止。如果链接形成循环，则必须返回错误代码。

# Lab10: mmap

实现 *nix 系统调用 mmap 的简单版：支持**将文件映射到一片用户虚拟内存区域内**，并且支持将对其的修改写回磁盘。

mmap 指令除了可以用来将文件映射到内存上，还可以用来将创建的进程间共享内存映射到当前进程的地址空间内。本 lab 只需实现前一功能即可

```c
void *mmap(void *addr, size_t length, int prot, int flags,int fd, off_t offset);
```

- 可以假设`addr`始终为零，这意味着内核应该决定该文件映射到用户地址空间的虚拟地址。
- `length`是要映射的字节数；它可能与文件的长度不同。
- `prot`指示内存是否应映射为可读、可写，以及/或者可执行的；您可以认为`prot`是`PROT_READ`或`PROT_WRITE`或两者兼有。
- `flags`要么是`MAP_SHARED`（映射内存的修改应写回文件），要么是`MAP_PRIVATE`（映射内存的修改不应写回文件）。
- `fd`是要映射的文件的打开文件描述符。
- 可以假定`offset`为零（它是要映射的文件的起点）

`mmap`返回映射到用户页表的虚拟地址，如果失败则返回`0xffffffffffffffff`。

`munmap(addr, length)`应删除指定地址范围内的`mmap`映射。如果进程修改了内存并将其映射为`MAP_SHARED`，则应首先将修改写入文件。`munmap`调用可能只覆盖`mmap`区域的一部分，但它取消映射的位置要么在区域起始位置，要么在区域结束位置，要么就是整个区域(但不会在区域中间“打洞”)

read和write系统调用的过程：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220301204441347.png" alt="image-20220301204441347" style="zoom:50%;" />

mmap的过程：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220301204536075.png" alt="image-20220301204536075" style="zoom:50%;" />

为了尽量使得 map 的文件使用的地址空间不要和进程所使用的地址空间产生冲突，我们选择将 mmap 映射进来的文件 map 到尽可能高的位置，也就是刚好在 trapframe 下面。并且若有多个 mmap 的文件，则向下生长。

接下来定义 vma 结构体，其中包含了 mmap 映射的内存区域的各种必要信息，比如开始地址、大小、所映射文件、文件内偏移以及权限等。

并且在 proc 结构体末尾为每个进程加上 16 个 vma 空槽。

```c
// kernel/proc.h

struct vma {
  int valid;
  uint64 vastart;
  uint64 sz;
  struct file *f;
  int prot;
  int flags;
  uint64 offset;
};

#define NVMA 16

// Per-process state
struct proc {
  struct spinlock lock;

  // p->lock must be held when using these:
  enum procstate state;        // Process state
  struct proc *parent;         // Parent process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  int xstate;                  // Exit status to be returned to parent's wait
  int pid;                     // Process ID

  // these are private to the process, so p->lock need not be held.
  uint64 kstack;               // Virtual address of kernel stack
  uint64 sz;                   // Size of process memory (bytes)
  pagetable_t pagetable;       // User page table
  struct trapframe *trapframe; // data page for trampoline.S
  struct context context;      // swtch() here to run process
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
  struct vma vmas[NVMA];       // virtual memory areas
};
```

mmap函数的具体功能是：在进程的 16 个 vma 槽中，找到可用的空槽，并且顺便计算所有 vma 中使用到的最低的虚拟地址（作为新 vma 的结尾地址 vaend，开区间），然后将当前文件映射到该最低地址下面的位置（vastart = vaend - sz）（**实际上此时没有真正在用户页表中建立映射，而是将映射的虚拟地址范围，映射的打开文件结构体等填入到该槽中的vma结构体**）。

```c
  v->vastart = vaend - sz;
  v->sz = sz;
  v->prot = prot;
  v->flags = flags;
  v->f = f; // assume f->type == FD_INODE
  v->offset = offset;

  filedup(v->f);
```

最后记得使用 `filedup(v->f);`，将文件的引用计数增加一

调用mmap时没有直接将文件的内容从磁盘中复制到物理内存，**而是对映射的页实行懒加载**，仅仅在PCB的vma结构体中初始化了相关字段，在访问到这段虚拟地址的时候，通过page fault handler，先通过传入的地址找到对应的vma结构体，将vma结构体中指向的文件从磁盘中加载出来，再映射到相应的用户虚拟地址处。这里采用 Lazy Page Allocation 类似的方式实现。

```c
// kernel/trap.c
void
usertrap(void)
{
  int which_dev = 0;

  // ......

  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
    uint64 va = r_stval();
    if((r_scause() == 13 || r_scause() == 15)){ // vma lazy allocation
      if(!vmatrylazytouch(va)) {
        goto unexpected_scause;
      }
    } else {
      unexpected_scause:
      printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
      printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
      p->killed = 1;
    }
  }

  // ......

  usertrapret();
}
```

接下来实现 munmap 调用，将一个 vma 所分配的所有页释放，并在必要的情况下，将已经修改的页写回磁盘。

这里首先通过传入的地址找到对应的 vma 结构体（通过前面定义的 findvma 方法），然后检测了一下在 vma 区域中间“挖洞”释放的错误情况，计算出应该开始释放的内存地址以及应该释放的内存字节数量

计算出来释放内存页的开始地址以及释放的个数后，调用自定义的 vmaunmap 方法（vm.c）对物理内存页进行释放，并在需要的时候将数据写回磁盘。

- 查找范围内的每一个页，检测其 dirty bit (D) 是否被设置，如果被设置，则代表该页被修改过，需要将其写回磁盘。

在调用 vmaunmap 释放内存页之后，对 v->offset、v->vastart 以及 v->sz 作相应的修改，并在所有页释放完毕之后，关闭对文件的引用，并完全释放该 vma。

最后需要做的，是在 proc.c 中添加处理进程 vma 的各部分代码。

- 让 allocproc 初始化进程的时候，将 vma 槽都清空
- freeproc 释放进程时，调用 vmaunmap 将所有 vma 的内存都释放，并在需要的时候写回磁盘
- fork 时，拷贝父进程的所有 vma，但是不拷贝物理页

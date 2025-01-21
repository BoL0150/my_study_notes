# MIT6.s081实验记录（二）：lab2

## System call tracing

首先添加一个用户程序trace：在Makefile的UPROGS中添加$U/_trace

trace.c中使用了trace系统调用，而trace系统调用在用户空间的存根还不存在，所以此时运行make qemu，将无法编译trace.c。

所以我们需要在用户空间添加trace系统调用的存根：

- 将系统调用在用户空间的存根的函数原型声明在**user/user.h**中
- 再将存根添加到`user/usys.pl`，编译时Makefile会调用perl脚本user/usys.pl，它生成实际的系统调用存根user/usys.S
- 为这个系统调用选择一个编号，将系统调用编号添加到`kernel/syscall.h`

在kernel/sysproc.c中添加sys_trace函数，也就是真正的系统调用

在syscall.c中的函数指针数组中添加SYS_trace到sys_trace函数指针的映射关系

**在用户空间调用了trace的存根后，trace函数的第一个参数是要跟踪的系统调用，放在了a0寄存器；而trace本身的系统调用号SYS_trace在ecall之前被放在了a7寄存器**。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220107183940695.png" alt="image-20220107183940695" style="zoom: 80%;" />

![image-20220107184515111](https://raw.githubusercontent.com/BoL0150/image2/master/image-20220107184515111.png)

xv6提供了`argint(int n, int *ip)`函数来获取系统调用的参数，第一个参数n表示要获取第几个参数，第二个参数表示保存到的位置

```c
int
argint(int n, int *ip)
{
  *ip = argraw(n);
  return 0;
}
static uint64
argraw(int n)
{
  struct proc *p = myproc();
  switch (n) {
  case 0:
    return p->trapframe->a0;
  case 1:
    return p->trapframe->a1;
  case 2:
    return p->trapframe->a2;
  case 3:
    return p->trapframe->a3;
  case 4:
    return p->trapframe->a4;
  case 5:
    return p->trapframe->a5;
  }
  panic("argraw");
  return -1;
}
```

我的做法是：在trace系统调用中将要追踪的系统调用掩码记录在proc结构体中，再修改fork系统调用，将trace mask从父进程复制到子进程的proc。在syscall中每次系统调用结束后，都判断该系统调用的掩码是否符合当前进程中的trace mask，如果符合就打印语句。

```c
void
syscall(void)
{
  int num;
  struct proc *p = myproc();
  char* syscall_name[] = {"fork", "exit", "wait", "pipe", "read", "kill", "exec", "fstat", "chdir", "dup", "getpid", "sbrk", "sleep", "uptime", "open", "write", "mknod", "unlink", "link", "mkdir", "close", "trace"};
  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
    // 当前的系统调用掩码是否等于要追踪的系统调用掩码
    if((1 << num) & p->traced_syscall_mask){
      printf("%d: syscall %s -> %d\n",p->pid,syscall_name[num - 1],p->trapframe->a0);
    }
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}

uint64 
sys_trace(void){
  int traced_syscall_mask = 0;
  // 获取a0寄存器的值，也就是系统调用的第一个参数，也就是要求trace追踪的系统调用号的掩码
  if(argint(0,&traced_syscall_mask) < 0)
    return -1;
  myproc()->traced_syscall_mask = traced_syscall_mask;
  return 0;
}
```

## Sysinfo

> 添加一个系统调用`sysinfo`，它收集有关正在运行的系统的信息。系统调用采用一个参数：一个指向`struct sysinfo`的指针（参见**kernel/sysinfo.h**）。内核应该填写这个结构的字段：`freemem`字段应该设置为空闲内存的字节数，`nproc`字段应该设置为`state`字段不为`UNUSED`的进程数。

在sysinfo内部获取内核的freemem信息和nproc信息，赋值给一个sysinfo结构体，再调用`copyout()`将这个结构体从内核复制到指定的用户空间的虚拟地址（也就是用户调用sysinfo的参数）。

获取内核的freemem信息需要在`kernel/kalloc.c`中添加一个函数

```c
uint64 
freemem(){
  struct run *r = kmem.freelist;
  uint64 cnt = 0;
  while(r){
    cnt++;
    r = r->next;
  }
  uint64 size  = cnt * PGSIZE;
  return size;
}
```

获取内核的nproc信息需要在`kernel/proc.c`中添加一个函数

```c
// 获取进程表中状态不为unused的proc
uint64
not_unused_proc(){
  struct proc *p;
  uint64 cnt = 0;
  for(p = proc;p < &proc[NPROC];p++){
    if(p->state != UNUSED){
      cnt++;
    } 
  } 
  return cnt;
}
```

sysinfo系统调用：

```c
uint64 
sys_sysinfo(void){
  struct sysinfo sysinfo;
  sysinfo.freemem = freemem();
  sysinfo.nproc = not_unused_proc();
  struct proc *p = myproc();
  uint64 arg = p->trapframe->a0;
  // 将内核中的sysinfo地址中的值复制到用户空间中的arg地址
  if(copyout(p->pagetable, arg, (char *)&sysinfo, sizeof(sysinfo)) < 0)
    return -1;
  return 0;
}
```


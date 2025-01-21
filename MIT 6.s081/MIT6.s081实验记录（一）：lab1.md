# MIT6.s081实验记录（一）：gdb调试qemu方法 & lab1

## 准备工作

### xv6调试

1. 在xv6文件夹下`make qemu-gdb`启动qemu上的gdbserver。

2. xv6的ISA是riscv，所以我们需要使用riscv的调试器`riscv64-unknown-elf-gdb`来调试xv6。

   使用tmux分屏，在另一窗口中，在相同的xv6文件夹下`riscv64-unknown-elf-gdb` 再加上要调试的文件名（比如`kernel/kernel`或者`user/_sleep`），也可以进入gdb后再输入`file user/_[execname]`加载可执行文件。

3. 可以看到riscv的gdb跑起来了，输入：

   ```text
   (gdb) target remote localhost:26000
   ```

   与qemu中的gdbserver建立连接，不然会提醒你the program is not being run.

   现在可以看到以下输出

   ```text
   Remote debugging using localhost:26000
   0x0000000000001000 in ?? ()
   ```

   每次都要输入target remote localhost:26000很麻烦，根据提示

   <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210914144855599.png" alt="image-20210914144855599" style="zoom:50%;" />

   我们在**用户目录**下的.gdbinit文件中添加以下内容即可：

   ```
   add-auto-load-safe-path /home/bolee/s081/xv6-labs-2020/.gdbinit
   ```

4. 通过`b [linenum]`设置断点。gdb在user程序中打断点会出现Cannot access memory at address错误，需要自己在.gdbinit.tmpl-riscv里加一行set riscv use-compressed-breakpoints yes。

5. 设置了断点后再输入`c`继续，此时gdb窗口会卡住，等待qemu中的相关程序执行

   <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210914143134537.png" alt="image-20210914143134537" style="zoom:67%;" />

6. 在`make qemu-gdb`的终端中执行该可执行文件，此时才可以在gdb窗口中开始debug

   <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210914143301478.png" alt="image-20210914143301478" style="zoom: 50%;" />

   

### 编译和测试

在编写好程序之后，需要在makefile中把我们写好的sleep.c加进去：

```makefile
UPROGS=\
    $U/_cat\
    $U/_echo\
    $U/_forktest\
        ........
    $U/_kalloctest\
    $U/_bcachetest\
    $U/_alloctest\
    $U/_bigfile\
    $U/_sleep\
```

`make qemu`就会将我们加入的程序编译，sleep.c就会被编译成可执行文件_sleep，并保存在xv6的文件系统中，然后就可以在xv6 shell中运行我们写好的程序。



我们可以在user目录下看到我们编译生成的可执行文件，

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210914120101664.png" alt="image-20210914120101664" style="zoom: 50%;" />

**这些可执行文件的格式为riscv，无法在本地x86的机器上执行，只能在qemu中模拟的riscv环境上运行**

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210914120422533.png" alt="image-20210914120422533" style="zoom: 33%;" />

运行`make grade`可以测试我们的程序是否正确。`make grade`会运行所有的程序，如果只是想针对某一个assignment运行测试，运行

```bash
$ ./grade-lab-util sleep
```

或者

```bash
$ make GRADEFLAGS=sleep grade
```

以上会针对sleep程序进行测试。

## sleep 

Implement the UNIX program `sleep` for xv6; your `sleep` should pause for a user-specified number of ticks. A tick is a notion of time defined by the xv6 kernel, namely the time between two interrupts from the timer chip.

在`kernel/sysproc.c` 中查看System系统调用的实现，在`user/user.h`中查看可从用户程序调用的sleep的 C 定义，在`user/usys.s`中查看从用户代码跳转到内核以进行sleep的汇编代码。

可以使用atoi将传入的字符串转为integer，该函数返回转换后的长整数，**如果没有执行有效的转换，则返回零**。

```cpp
#include "../kernel/types.h"
#include "user.h"
int main(int argc, char *argv[])
{
    if(argc != 2){
        // 向标准错误中输出错误信息
        fprintf(2,"you must input sleep time");
        exit(1);
    }
    // atoi将字符串转为int，再进行系统调用sleep
    sleep(atoi(argv[1]));
    exit(0);
}
```

## pingpong 

编写一个程序，使用 UNIX 系统调用在两个进程之间通过一对管道“ping-pong”一个字节，每个管道一个。 父进程应该向子进程发送一个字节； 子进程应该打印“`<pid>: received ping`”，其中 `<pid>` 是它的进程 ID，将管道上的字节写入父进程，然后退出； 父进程应该从子进程那里读取字节，打印“`<pid>: received pong`”，然后退出。我们的解决方案应该在文件 `user/pingpong.c` 中。

```c
#include "../kernel/types.h"
#include "user.h"
#define READ_END 0
#define WRITE_END 1
int main(int argc,char *argv[])
{
    int p_child[2];
    int p_parent[2];
    char buf[1];
    pipe(p_child);
    pipe(p_parent);
    // 子进程
    if(fork() == 0){
        close(p_child[READ_END]);
        close(p_parent[WRITE_END]);
        read(p_parent[READ_END],buf,1);
        printf("%d: received ping\n", getpid());
        write(p_child[WRITE_END]," ",1);
        close(p_child[WRITE_END]);
        close(p_parent[READ_END]);
        exit(0);
    }else{
        // 关闭父进程接收管道的写入端描述符
        close(p_child[WRITE_END]);
        // 关闭父进程输出管道的接收端描述符
        close(p_parent[READ_END]);
        write(p_parent[WRITE_END]," ",1);
        read(p_child[READ_END],buf,1);
        printf("%d: received pong\n", getpid());
        close(p_child[READ_END]);
        close(p_parent[WRITE_END]);
        exit(0);
    }
}

```

## primes

使用管道编写并发版本的素数筛

Your goal is to use `pipe` and `fork` to set up the pipeline. The first process feeds the numbers 2 through 35 into the pipeline. For each prime number, you will arrange to create one process that reads from its left neighbor over a pipe and writes to its right neighbor over another pipe.

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210118093050894.png" alt="image-20210118093050894" style="zoom:67%;" />

```c
#include "../kernel/types.h"
#include "user.h"
#define MAX_NUM 35
#define READ_END 0
#define WRITE_END 1
void func(int);
int main()
{
    int p[2];
    pipe(p);
    if(fork() == 0){
        close(p[WRITE_END]);
        func(p[READ_END]);
    }else{
        // READ_END被传递给下一个进程，
        // 在本进程中不再使用，需要关闭
        close(p[READ_END]);
        for(int i=2;i <= 35;i++){
            write(p[WRITE_END],&i,sizeof(int));
        }
        // 向管道中的输入结束，WRITE_END在本进程中不再使用，也需要关闭
        close(p[WRITE_END]);
        // 等待子进程结束，父进程才能结束
        wait(0);
    } 
    exit(0);
}
void func(int read_end){
    
    int p[2];
    pipe(p);
    int base = 0;
    // 如果父进程只有一个数字，就没有数字可以
    // 传给子进程，子进程直接结束
    if( read(read_end,&base,sizeof(int))==0)
        exit(0);
    // 否则该数字就是筛选出的素数
    printf("prime %d\n",base);
    
    if(fork() == 0){
        close(p[WRITE_END]);
        func(p[READ_END]);
    }else{
        close(p[READ_END]);
        // 从父进程读取数字，清理后发送给子进程
        int temp =0; 
        while(read(read_end,&temp,sizeof(int))){
            if(temp % base!=0){
                write(p[WRITE_END],&temp,sizeof(int));
            }
        }
        close(read_end);
        close(p[WRITE_END]);
        wait(0);
        exit(0);
    }
}
```

## find 

编写一个简单版本的 UNIX 查找程序：查找指定文件夹下符合某个名字的所有文件。实际上使用了深度优先搜索，查找一棵树中的叶结点是否等于目标文件的名字。

对文件系统的更改在 qemu 的运行中持续存在；要获得一个干净的文件系统先运行 `make clean`清除之前编译的结果， 然后使用 `make qemu`重新编译

```c
#include "../kernel/types.h"
#include "../kernel/stat.h"
#include "user.h"
#include "../kernel/fs.h"

void find(char*path,char*target_file){
    char buf[512],*p;
    int fd;
    // 为了获取某文件夹的内容，所使用的结构体
    struct dirent de;
    // 获取某文件或文件夹的信息
    struct stat st;

    // 如果打不开指定的路径，输出错误信息
    if((fd = open(path,0)) < 0){
        fprintf(2, "find: cannot open %s\n",path);
        return;
    }
    // 从该目录所指向的inode检索信息
    if(fstat(fd, &st) < 0){
        fprintf(2, "find: cannot stat %s\n",path);
        close(fd);
        return;
    }
    // 遍历该路径下的所有文件或文件夹,判断名字是否与目标文件名字相同
    // de.name就是文件的名字
    while(read(fd,&de,sizeof(de)) == sizeof(de)){
        strcpy(buf,path);
        // 此时p指向路径字符串的最后一个字符的下一位置
        p = buf+strlen(buf);
        // 在路径的最后加上/,再将p向后移动一个位置
        *p++ = '/';

        if(de.inum == 0){
            continue;
        }
        // 跳过. 和 ..
        if(strcmp(de.name,".") == 0 || strcmp(de.name,"..") == 0){
            continue;
        }
        
        // 当前所遍历的文件/目录的完整路径
        memmove(p,de.name,DIRSIZ);
        p[DIRSIZ] = 0;
        if(stat(buf,&st) < 0){
            fprintf(2,"find: cannot stat %s\n",buf);
        }
        // 如果是文件，那么直接比较
        switch(st.type){
            case T_FILE:
                if(strcmp(target_file,de.name) == 0){
                    printf("%s\n",buf);
                }
                break;
            case T_DIR:
                find(buf,target_file); 
        }
    }
    close(fd);
    return;
}
int main(int argc,char*argv[])
{
    if(argc != 3){
        fprintf(2,"We need 3 para");
        exit(0);
    }
    find(argv[1],argv[2]);
    exit(0);
}
```

## xargs

编写一个简单版本的 UNIX xargs 程序：从标准输入中读取行并为每一行运行一个命令，将该行作为命令的参数提供。


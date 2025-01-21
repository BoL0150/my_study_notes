# CMU15-213学习笔记（八）System-Level I/O

比起其他的操作系统，Unix中的I/O更简单而且更一致，用文件来描述很多抽象的事物。与windows和早期的mac不同，Unix不区分文件类型，只是**把文件看作一个字节序列**，操作系统基本上不了解文件内部的详细结构。

**输入/输出（I/O）**是**在主存和外部设备（磁盘驱动器、终端和网络）之间复制数据的过程**。输入操作就是从I/O设备复制数据到主存，输出操作就是将主存数据复制到I/O设备。这里我们**将所有I/O设备都抽象为Linux文件，即m个字节的序列**，这样所有输入和输出操作都可以看成是对文件的读写，使得所有输入和输出都能以统一的方法来执行：

- 所有I/O设备都是用文件来表示的
  - `./dev/sda2` (`/usr` 磁盘分区)
  - `/dev/tty2` (终端)
- 网络连接中的套接字
- 甚至连内核也是用文件来表示的：
  - `/boot/vmlinuz-3.13.0-55-generic` (内核镜像)
  - `/proc` (内核数据结构)

通过将I/O设备抽象为文件的形式，使得对I/O设备的输入输出统一为对文件的读写操作

- **打开文件：**一个应用程序通过要求内核打开某个文件，来确定要访问的I/O设备。此时**内核会记录有关该文件的所有信息**，而应用程序会获得该文件的**描述符**，来标识该文件。

- **修改当前读写文件的位置：**对于每个打开的文件，内核会维护一个**文件位置k**来表示当前要读写文件的位置，是从文件头开始的字节偏移量，初始化为0。应用程序可以通过`seek`操作来修改读写文件的位置。

- - 但是无法修改基于终端的输入的文件位置，因为无法移动、备份和恢复先前已读入的数据，也无法提前接收还未键入的数据。比如套接字。

- **读写文件：**读文件就是在文件中从k开始复制n个字节到内存，并更新`k=k+n`，如果k超出了文件大小，则会给引用程序返回EOF。而写文件就是从内存 复制n个字节到文件位置k处，然后更新`k=k+n`。

- **关闭文件：**当应用程序通知内核关闭文件时，**内核会释放打开文件的数据结构**，并**将该描述符恢复到可用的描述符池中**。

应用程序可以通过**文件描述符**来对指定的I/O设备进行操作

在Linux中，**文件具有不同的类型：**

- **普通文件（Regular File）：**包含任意数据。通常分为：

- - **文本文件（Text File）：**只含有ASCII或Unicode字符的普通文件，是一系列文本行的序列，以`\n`符号间隔。
  - **二进制文件（Binary File）：**所有其他的文件。比如：目标文件，图像或音视频文件等。

- 应用程序会进行区分，而对于内核而言没有区别。

- **目录（Directory）：**包含一组通过文件名映射到文件的**链接（Link）**的文件。每个链接都是目录文件中的一个条目。

- **套接字（Socket）：**用来与另一个进程进行跨网络通信的文件。

### 目录

**目录包含一个链接(link)数组**，每个链接都是从文件名到文件的映射。

每个目录至少包含两条记录：

- `.`(dot) 引用当前目录的链接
- `..`(dot dot) 引用上一层目录的链接

用来操作目录的命令主要有 `mkdir`, `ls`, `rmdir`

Linux将所有文件组织成**目录层次结构（Directory Hierarchy）**，其中`/`表示**根目录**，其他所有文件都是根目录的直接或间接的后代。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210918165917984.png" alt="image-20210918165917984" style="zoom: 67%;" />

**内核会为每个进程保存当前工作目录(cwd, current working directory)**，可以用 `cd` 命令来进行更改。

我们通过路径名来确定文件的位置，一般分为绝对路径和相对路径。

**打开文件**:

打开一个文件通知内核你准备好访问该文件

```c
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
int open(char *filename, int flags, mode_t mode);
```

**关闭文件**：

```c
#include <unistd.h>
int close(int fd);
```

**读写文件**：

```c
#include <unistd.h>
ssize_t read(int fd, void *buf, size_t n);
ssize_t write(int fd, const void *buf, size_t n);
```

### RIO

RIO（Robust I/O）包对底层IO提供了封装，可以处理short count的问题（比如网络编程）。例如，如果调用RIO包中的readn函数，并指定字节数，则在读完该字节数之前函数不会返回。但是对于网络编程要小心使用，如果没有读到指定字节数，它将挂起等待。

它提供了两类不同的函数：

- **无缓冲的输入输出函数：**直接在内存和文件之间传送数据，没有应用级缓冲，主要用来克服不足值问题。
  - `rio_readn` 和`rio_writen`
- **带缓冲的输入输出函数：**允许你高效地从文件中读取文本行和二进制数据，这些文件的内容缓存在应用级缓冲区中，主要用来克服不足值问题，和反复调用系统函数的耗时。
  - `rio_readlineb`和`rio_readnb`

#### 无缓冲的输入输出函数

`rio_readn` ：只有当遇到EOF时才会返回short count

- 只有在你知道要读取多少字节时才使用

`rio_writen`：永远不返回short count

```c
#include "csapp.h"
ssize_t rio_readn(int fd, void *usrbuf, size_t n);
ssize_t rio_writen(int fd, void *usrbuf, size_t n);
```

实现：

```c
#include <csapp.h>
ssize_t rio_readn(int fd, void *usrbuf, size_t n) 
{
    size_t nleft = n;
    ssize_t nread;
    char *bufp = usrbuf;

    while (nleft > 0) {  //不断循环，直到读取到n个字节
	if ((nread = read(fd, bufp, nleft)) < 0) {
	    if (errno == EINTR) /* Interrupted by sig handler return */
		nread = 0;      /* and call read() again */
	    else
		return -1;      /* errno set by read() */ 
	} 
	else if (nread == 0)
	    break;              /* EOF */
	nleft -= nread;
	bufp += nread;
    }
    return (n - nleft);         /* Return >= 0 */
}

ssize_t rio_writen(int fd, void *usrbuf, size_t n) 
{
    size_t nleft = n;
    ssize_t nwritten;
    char *bufp = usrbuf;

    while (nleft > 0) {  //不断循环，直到写了n个字节
	if ((nwritten = write(fd, bufp, nleft)) <= 0) {
	    if (errno == EINTR)  /* Interrupted by sig handler return */
		nwritten = 0;    /* and call write() again */
	    else
		return -1;       /* errno set by write() */
	}
	nleft -= nwritten;
	bufp += nwritten;
    }
    return n;
}
```

#### 带缓冲的输入输出函数

从一个一部分缓存在内部内存缓冲区中的文件中高效读取文本行和二进制数据

```c
#include "csapp.h"
void rio_readinitb(rio_t *rp, int fd);
ssize_t rio_readlineb(rio_t *rp, void *usrbuf, size_t maxlen);
ssize_t rio_readnb(rio_t *rp, void *usrbuf, size_t n); 
```

`rio_readlineb`从fd引用的文件中读取最多maxlen个字节的文本行，并将它们存储在usrbuf中。停止读取的条件：

- 读取maxlen个字节
- 遇到EOF
- 遇到'\n'

`rio_readnb`从fd引用的文件中读取n个字节，停止读取的条件：

- 读取到maxlen个字节
- 遇到EOF

带缓冲的I/O会给文件分配关联的缓冲区，如果程序正在执行读操作，它将填满该缓冲区，当用户程序想要提取若干字节时，它会先看看缓冲区内是否有还未读取的字节，如果有就继续读取，如果没有就重新填满缓冲区。带缓冲区的好处在于，不用每次需要一个或少两字节就调用系统调用，而是**先调用一次系统调用，将若干字节缓存起来，后面需要时就直接从缓冲区读取就行了**。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210918203038722.png" alt="image-20210918203038722" style="zoom:67%;" />

绿色表示已被应用程序读取的字节，红色表示已经通过调用操作系统从文件中读取到缓冲区，但是还没有被应用程序读取的内容。

缓冲区的数据结构：

```c
#define RIO_BUFSIZE 8192
typedef struct{
  int rio_fd;  //该缓冲区的文件描述符
  int rio_cnt;  //缓冲区中未读的字节数
  char *rio_bufptr;  //缓冲区中未读的下一个字节
  char rio_buf[RIO_BUFSIZE];  //读缓冲区
} rio_t;
```

### File Metadata

**File元数据是用来描述文件中数据的数据**，由内核维护，可以通过 `stat` 和 `fstat` 函数来访问

```c
#include <unistd.h>
#include <sys/stat.h>
int stat(const *filename, struct stat *buf);
int fstat(int fd, struct stat *buf);
```

**该函数会将metadata以stat数据结构的形式填入参数buf指向的位置**（这些函数的一个典型特征就是它们含有一些预定义的struct，如果想获得信息，可以先分配一个struct，再传一个指针给函数，库函数就会将文件信息填入该struct）：

```c
// 被stat和fstat函数返回的metadata
struct stat
{
    dev_t           st_dev;     // Device
    ino_t           st_ino;     // inode
    mode_t          st_mode;    // Protection & file type
    nlink_t         st_nlink;   // Number of hard links
    uid_t           st_uid;     // User ID of owner
    gid_t           st_gid;     // Group ID of owner
    dev_t           st_rdev;    // Device type (if inode device)
    off_t           st_size;    // Total size, in bytes
    unsigned long   st_blksize; // Blocksize for filesystem I/O
    unsigned long   st_blocks;  // Number of blocks allocated
    time_t          st_atime;   // Time of last access
    time_t          st_mtime;   // Time of last modification
    time_t          st_ctime;   // Time of last change
}
```

其中`st_size`包含了文件的字节数大小，`st_mode`编码了文件访问许可位，我们可以通过`sys/stat.h`中定义的宏来确定该部分的信息：

- `S_ISREG(st_mode)`： 是否为普通文件
- `S_ISDIR(st_mode)`：是否为目录文件
- `S_ISSOCK(st_mode)`：是否为套接字

使用例子：

```c
int main (int argc, char **argv)
{
    struct stat stat;
    char *type, *readok;
    
    Stat(argv[1], &stat);
    if (S_ISREG(stat.st_mode)) // 确定文件类型
        type = "regular";
    else if (S_ISDIR(stat.st_mode))
        type = "directory";
    else
        type = "other";
    
    if ((stat.st_mode & S_IRUSR)) // 检查读权限
        readok = "yes";
    else
        readok = "no";
    
    printf("type: %s, read: %s\n", type, readok);
    exit(0);
}
```

对于目录文件的操作，我们将**目录文件中的项**定义为以下数据结构

```c
struct dirent{
  int_t d_ino;  //文件位置
  char d_name[256];  //文件名
}
```

我们可以传递一个路径名给以下函数来获得目录项的列表

```c
#include <sys/types.h>
#include <dirent.h>
DIR *opendir(const char *name);
```

然后可通过循环调用以下函数来不断获得列表中的下一个目录项，并**根据目录项的数据结构来获得目录的信息**

```c
#include <dirent.h>
struct dirent *readdir(DIR *dirp);
```

### 共享文件

文件系统的组成如下：

- **描述符表（Descriptor Table）：**每个进程有自己独立的描述符表，进程打开的所有文件描述符都包含在该表中，每个文件描述符指向文件表中的一个表项。
- **文件表（File Table）：**所有进程共享文件表，包含所有**打开文件**的**文件位置**和指向的v-node表项，由于可能在不同进程中共享同一个文件表表项，所以会有一个引用次数表示有多少个描述符指向当前文件表表项，只有当表项的引用次数为0才会删除该表项。**每次打开一个文件，就会在文件表中分配一个表项**。该文件表描述了指向对应文件表象的描述符的信息，有操作系统维护。
- **v-node表：**所有进程共享v-node表，每个表项包含了`stat`结构中的大多数信息，用来描述文件的信息。在系统中的每个文件无论是否打开，都在v-node表汇总有一个对应的表项。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210919113334746.png" alt="image-20210919113334746" style="zoom: 50%;" />

每个描述符都有自己的文件位置，表明了当前读取文件的位置。如果在同一个进程中使用open打开同一个文件，则会生成不同的文件表表项，分配一个不同的文件描述符指向该文件表表项。但是由于是同一个文件，会指向同一个v-node。这就使得这两个文件描述符可以对同一个文件的不同文件位置进行读写，将不同的描述符操作独立开来

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210919113900571.png" alt="image-20210919113900571" style="zoom: 50%;" />

在父进程使用`fork`函数创建一个子进程时，子进程会复制父进程的描述符表，由于描述符表中包含指向文件表中的指针，所以子进程中相同的描述符也指向了相同的文件表表项，所以父子进程对文件位置的修改是共享的

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210919114025382.png" alt="image-20210919114025382" style="zoom: 50%;" />

**只有fork和dup才能让不同的文件描述符指向同一个文件表表项**

**同一个文件表表项表示同一个文件的同一个位置**。

不同的文件描述符可能指向同一个文件，但是指向不同的文件表表项，也就是指向同一个文件的不同位置。

文件表表项只是说明一个文件在某一个进程中被打开了（open），如果对同一个文件open多次，就会有多个文件表表项指向同一个v-node。如果一个v-node没有文件表表项，只是说明该文件没有被打开，但是在文件系统中仍然有链接（文件名）指向底层的v-node。

以下是xv6系统调用，非linux标准系统调用。

- **link和unlink系统调用用来给v-node绑定链接，相当于给文件起别名。不生成文件表表项**。
- dup系统调用分配一个文件描述符，与指定fd指向同一个文件表表项
- open系统调用给指定文件名（link）生成文件表表项，再给该文件表表项分配文件描述符。
- close系统调用释放指定的文件描述符，对应的文件表表项引用计数减一 

```c
// 创建并打开一个文件名为a的文件
int fd0 = open("a", O_CREATE | O_WRONLY);// 创建一个v-node，还创建一个指向v-node的名为a的链接，创建一个文								  // 件表表项，指向v-node，对该文件表表项分配一个文件描述符
// 创建一个文件名为b，与文件a引用同一个inode
link("a", "b");
// 创建一个文件表表项，与a指向同一个v-node，对该文件表表项分配一个文件描述符
int fd = open("b", O_RDONLY);
// 分配一个文件描述符，与fd指向同一个文件表表项
int fd2 = dup(fd);
// 以上三个文件描述符最终指向同一个v-node
```

### I/O重定向

```c
#include <unistd.h>
int dup2(int oldfd, int newfd); 
```

如果`newfd`是已打开的文件，`dup2`会首先关闭`newfd`，然后将`oldfd`对应的描述符表项替换`newfd`描述符表项，使得将`newfd`重定向到`oldfd`，如果原始`newfd`对应的文件表表项的引用计数为0，则会删除对应的文件表表项和v-node表项。

![img](https://raw.githubusercontent.com/BoL0150/image2/master/v2-d2b02e1f6af81eee349bfb66fbb24ae5_1440w.jpg)

### 标准I/O

C语言基于Unix I/O提出了一组高级输入输出函数，称为标准I/O。**标准I/O将一个打开文件抽象为一个流，是对文件描述符和内存中的缓冲区的抽象，就是一个指向`FILE`类的指针**。程序开始时就会打开3个流，对应于标准输入、标准输出和标准错误：

```c
#include <stdio.h>
extern FILE *stdin;
extern FILE *stdout;
extern FILE *stderr;
```

程序经常会一次读入或者写入一个字符，比如 `getc`, `putc`, `ungetc`，同时也会一次读入或者写入一行，比如 `gets`, `fgets`。**如果用 Unix I/O 的方式来进行调用，是非常昂贵的**，比如说 `read` 和 `write` 因为需要内核调用，需要大于 10000 个时钟周期。

解决的办法就是利用 `read` 函数一次读取一块数据，然后再由高层的接口，一次从缓冲区读取一个字符（当缓冲区用完的时候需要重新填充）

比如：printf的输出先存放在内存中的缓冲区中，直到遇到`\n`或调用`fflush`函数时，才清空缓冲区，使用系统调用write，将缓冲区的内容一次性输出到标准输出。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210920120551797.png" alt="image-20210920120551797" style="zoom: 50%;" />

缓冲区刷新到输出描述符的条件：

- 遇到换行符`'\n'`
- 调用fflush
- 调用exit
- 从main函数返回

当遇到'\n'时会自动flush，所以此处不使用fflush效果也是一样的。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210920125507021.png" alt="image-20210920125507021" style="zoom:67%;" />

以下命令查看指定程序的系统调用，trace选项查看指定的系统调用。

```bash
$ strace -e trace=write ./xxx
```

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210920133108790.png" alt="image-20210920133108790" style="zoom:67%;" />

### summary

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210920134631582.png" alt="image-20210920134631582" style="zoom:67%;" />

**Unix I/O的特点：**

- **优点：**

- - Unix I/O是最通用，开销最低的I/O形式
  - 所有其他I/O软件包均使用Unix I/O函数实现
  - Unix I/O提供用于访问文件元数据的功能
  - Unix I/O函数是异步信号安全的，可以在信号处理程序中安全使用

- **缺点：**

- - 无法处理short count
  - 高效读取文本行需要某种形式的缓冲
  - 这两个问题均通过标准I/O和RIO包解决

**标准I/O的特点：**

- **优点：**

- - **通过缓冲减少读写系统调用的数量来提高效率**
  - 不足值自动处理

- **缺点：**

- - 不提供访问文件元数据的功能，只能使用Unix I/O中的`stat`系统调用。
  - 标准I/O功能不是异步信号安全的，不适用于信号处理程序
  - **标准I/O不适合网络套接字上的输入和输出**

**对于使用什么I/O，建议：**

- 只要有可能就使用标准I/O，当操作硬盘或中断时要使用标准I/O
- **对套接字的I/O使用RIO包**
- 当写信号处理程序时要使用Unix I/O，因为它是异步信号安全的

**注意：**处理文本文件的函数，比如`fgets`、`scanf`或`rio_readlineb`函数，是基于文本行的概念针对文本文件进行操作的，在处理0xa字符时会停止读入。而处理字符串的函数，比如`strlen`、`strcpy`或`strcat`函数，会将`0`视为字符串的结尾。这些对某些特殊字节进行特殊解释的函数，**不适合用来读写二进制文件**。

## cs162环境

启动docker：`docker-compose up -d`，然后再连上docker：`ssh workspace@127.0.0.1 -p 16222`，停止docker：`docker-compose down`

密码是workspace

## 环境配置

运行docker镜像并且将宿主机上的pintos挂载在容器中

```
docker run -it --rm --name pintos --mount type=bind,source=./pintos,target=/home/PKUOS/pintos pkuflyingpig/pintos bash
```

- **`--rm:`** Tell docker to delete the container after running

- **`--name pintos:`** Name the container as `pintos`, which will be helpful in the debugging part.

退出容器： **`Ctrl+d`** 

运行pintos

```
cd pintos/src/threads/
make
cd build
pintos --
```

在宿主机上运行容器，容器相当于又是一个Ubuntu虚拟机，这个虚拟机中的环境都配置好了。pintos的代码在宿主机上，但是由于宿主机上没有环境（比如qemu），所以无法运行pintos。所以我们要将pintos挂载到容器中，然后启动容器，此时在容器中就会发现pintos的代码。我们就可以在容器中运行qemu，模拟一个裸机，在qemu模拟出来的机器上运行pintos。**所有对pintos的调试、运行、测试都是在容器中进行的**，但是我们对pintos修改可以在宿主机上进行。



## 编译pintos

每个lab的目录下都有一个Makefile文件，在lab目录下用make命令，再进入build目录下

build目录下的kernel.o文件是内核的目标文件，包含调试信息

kernel.bin文件是内核的内存镜像，也就是被加载到内存中的内核的字节，这就是去除了调试信息的kernel.o文件，以节省空间

loader.bin是内核loader的内存镜像，是用汇编语言写的一小块代码，将内核从磁盘中读到内存中并将其启动，正好是512字节的长度，由PC的BIOS固定的长度

## Running Pintos

在build目录下我们可以使用一些命令行指令来运行和测试pintos

如：`pintos -- run alarm-multiple`

- run表示让内核运行一个测试，`alarm-multiple`表示测试的名字
- 该指令启动了qemu，然后pintos启动并运行了`alarm-multiple`这个测试，When it's done, you can close Qemu by `Ctrl+a+c` .
- we can log the output to a file by redirecting at the command line, e.g. `pintos -- run alarm-multiple > logfile`.

option在前（每个option前面要加上--），arg在后，二者之间用--分割，option表示配置pintos程序，arg表示真正希望pintos程序执行的动作。**You can invoke** **`pintos -h`** **to see a list of available options.** 

```
pintos --option1 --option2 ... -- arg1 arg2 ....
```

- options可以选择使用的模拟器，默认使用的是Qemu， `--bochs` 选择Bochs
- `--gdb`表示选择使用gdb

如下面这句，表示使用gdb调试，使用bochs模拟器运行，运行某个特定的测试，运行时间不超过60秒

```
pintos -v -k -T 60 --gdb --bochs  -- -q  -mlfqs run mlfqs-load-1
```

### GDB

You can run the simulator with a debugger. Just select `--gdb` option.如：`pintos --gdb -- run mytest`.然后pintos程序会阻塞，等待GDB来连接到这个gdb session

随后在同一个机器上再开一个终端，并且使用 `pintos-gdb` 来在kernel.o上调用GDB

- 由于我使用的是docker，那么需要一个方式来将pintos容器连接到**宿主机**上的另一个终端。在宿主机上打开一个新的终端，输入 `docker exec -it pintos bash`。（之前我们在启动pintos容器的时候，给它的名字是pintos，所以这里可以直接用pintos）

- 然后，在这个新的终端中，使用 `pintos-gdb` 在kernel.o上调用GDB

  ```
  cd pintos/src/threads/build
  pintos-gdb kernel.o
  ```

  然后输入 `debugpintos`。这是一个GDB macro，用来表示 `target remote localhost:1234`的简写。

- 此时GDB就通过本地的网络连接到了模拟器上的pintos

## Debug and Test

当我们调试多线程的程序时，我们需要能够多次运行一个程序并且该程序始终做完全相同的动作，这种特性叫做可复现性。pintos支持的模拟器之一Bochs，就可以支持复现性，通过`--bochs`option指令使用该模拟器。 具体的，如果我们 invoke `pintos` with the option `-j` seed,  Within a single seed value, execution will still be reproducible, but timer behavior will change as seed is varied.所以我们需要用不同的种子来测试代码

qemu模拟器比Bochs要快很多，但是它只支持实时模拟，而非可复现模式

### Test

我们应该采用测试驱动的方式写代码，而不是一次性写上千行再测试

#### 目录结构

在src/tests下，每个lab（除了lab 0）都有其对应的测试目录：

- "src/tests/threads"

- "src/tests/userprog"

- "src/tests/vm"

- "src/tests/filesys"

一些labs（如lab3：虚拟内存）可能包含其他labs的测试用例，您可以查看其相应Make.vars文件中定义的TEST_SUBDIRS变量以了解详细信息（例如对于lab3，该文件是src/vm/Make.vars） 。

在每个lab的测试目录下，文件可以分类如下：

-  “foo.c” foo 测试用例的 C 源代码 

- “foo.ck” 用来检查输出与参考输出的perl脚本，您可以在其中找到 foo 测试用例的期望输出。 

- “Rubric.bar",bar是测试集的名称，Rubric 文件定义该集包含哪些测试及其分配的分数。 例如，src/tests/threads/Rubric.alam定义了alarm测试集，其中包含“Functionality and robustness of alarm clock”的所有测试：

  ```
  Functionality and robustness of alarm clock:
  4	alarm-single
  4	alarm-multiple
  4	alarm-simultaneous
  4	alarm-priority
  
  1	alarm-zero
  1	alarm-negative
  ```

- "Grading"，定义每个测试集的测试点的比例。例如， src/tests/threads/Grading ：

  ```
  # Percentage of the testing point total designated for each set of
  # tests.
  
  20.0%	tests/threads/Rubric.alarm
  40.0%	tests/threads/Rubric.priority
  40.0%	tests/threads/Rubric.mlfqs
  ```

#### 运行测试

在project的目录下（如：`~/pintos/src/threads`目录下）使用make check运行该project的所有测试，

> For **project 1**, the tests will probably run faster in **Bochs**. 
>
> For **the rest of the projects**, they will run much faster in **QEMU**.
>
> - `make check` will select the faster simulator by default, but you can override its choice by specifying `SIMULATOR=--bochs` or `SIMULATOR=--qemu` on the `make check` command line.

运行一个测试会将该测试的输出写到 `~/pintos/src/threads/build/tests/threads`目录下，文件名为`foo.output`，然后perl脚本将输出评价为pass或fail，并将测试结果写到`foo.result`文件中

运行并测试一个单独的测试，我们可以在project目录下的bulid目录中make相应的`.result`目录

- Type `make tests/threads/alarm-multiple.result` in the build directory to test the `alarm-multiple` case. 
- 如果make说该测试结果已经是最新的了，我们可以手动删掉该`.result`文件，或者使用`make clean`

默认情况下，每个测试只有在运行结束时才会给出运行反馈，我们也可以在make命令中使用 `VERBOSE=1` ，从而可以观察每个测试运行的过程

我们也可以向pintos提供任意的选项，通过在make命令中使用`PINTOSOPTS='...'`

-  `make check PINTOSOPTS='-j 1'` to select a jitter value of 1

### debug

1. printf

2. assert：理想情况下，每个函数的一开始都应该有一些assert来检查参数的合法性。它们对检查循环不变量很有用。pintos提供了assert宏，ASSERT (expression)，如果expression的结果是false或0，则panic。

   - backtrace：当kernel panic的时候会打印backtrace。我们也可以调用`debug_backtrace()`，在代码中的任意一点打印backtrace。或者`debug_backtrace_all()`，打印所有线程的backtrace。（二者都在`<debug.h>`中）

   - backtrace的结果是一系列十六进制的地址，pintos提供了一个叫backtrace的工具来讲这些地址转换成函数名和对应的文件行数

     - Give it the name of your `kernel.o` as the first argument and the hexadecimal numbers composing the backtrace (including the 0x prefixes) as the remaining arguments.

     - It outputs the function name and source file line numbers that correspond to each address.

     具体用法：我们假设kernel.o在当前的目录下

     ```
     backtrace kernel.o 0xc0106eff 0xc01102fb 0xc010dc22 0xc010cf67 0xc0102319 0xc010325a 0x804812c 0x8048a96 0x8048ac8
     ```

     如果backtrace的结果很混乱说不通，就说明我们有可能破坏了内核线程的栈，因为backtrace就是从栈中抽取出来的；或者我们传递给backtrace的kennel.o文件和产生backtrace的kernel.o不是同一个。

     或者：

     - When a function has called another function as its final action (a *tail call*), the calling function may not appear in a backtrace at all. Similarly, when function A calls another function B that never returns, the compiler may optimize such that an unrelated function C appears in the backtrace instead of A. Function C is simply the function that happens to be in memory just after A. In the threads project, this is commonly seen in backtraces for test failures.

3. GDB

# 框架代码

80386处理器有四种运行模式：实模式、保护模式、SMM模式和虚拟8086模式

实模式：这是个人计算机早期的8086处理器采用的一种简单运行模式。80386加电启动后处于实模式运行状态，在这种状态下软件可访问的物理内存空间不能超过1MB，且无法发挥Intel 80386以上级别的32位CPU的4GB内存管理能力。实模式下，操作系统和用户程序并没有区别对待，而且每一个指针都是指向实际的物理地址。这样用户程序的一个指针如果指向了操作系统区域或其他用户程序区域，并修改了内容，那么其后果就很可能是灾难性的。

在实模式下，内存寻址方式和8086相同，由16位段寄存器的内容乘以16（10H）当做段基地址，加上16位偏移地址形成20位的物理地址，最大寻址空间1MB，最大分段64KB。

> 对于ucore其实没有必要涉及，这主要是Intel x86的向下兼容需求导致其一直存在。其他一些CPU，比如ARM、MIPS等就没有实模式，而是只有类似保护模式这样的CPU模式。

保护模式：

- 保护模式支持内存保护机制，这意味着一个程序不能越界访问其他程序的内存，从而提高了系统的稳定性和安全性。实际上，80386就是通过在实模式下初始化控制寄存器（如GDTR，LDTR，IDTR与TR等管理寄存器）以及页表，然后再通过设置CR0寄存器使其中的保护模式使能位置位，从而进入到80386的保护模式。
- 当80386工作在保护模式下的时候，其所有的32根地址线都可供寻址，物理寻址空间高达4GB。
- 在保护模式下，支持内存分页机制，提供了对虚拟内存的良好支持。
- 保护模式下80386支持多任务，还支持优先级机制，不同的程序可以运行在不同的特权级上。特权级一共分0～3四个级别，操作系统运行在最高的特权级0上，应用程序则运行在比较低的级别上；配合良好的检查机制后，既可以在任务间实现数据的安全共享也可以很好地隔离各个任务。



pintos物理地址空间布局：

```
+------------------+  <- 0xFFFFFFFF (4GB)
|      32-bit      |
|  memory mapped   |
|     devices      |
|                  |
/\/\/\/\/\/\/\/\/\/\
/\/\/\/\/\/\/\/\/\/\
|                  |
|      Unused      |
|                  |
+------------------+  <- depends on amount of RAM
|                  |
|                  |
| Extended Memory  |
|                  |
|                  |
+------------------+  <- 0x00100000 (1MB)
|     BIOS ROM     |
+------------------+  <- 0x000F0000 (960KB)
|  16-bit devices, |
|  expansion ROMs  |
+------------------+  <- 0x000C0000 (768KB)
|   VGA Display    |
+------------------+  <- 0x000A0000 (640KB)  从640KB以下属于low memory
|  pintos kernel   |  内核代码，数据
+------------------+  <- 0x00020000 (128KB)
|  page tables     |
|  for startup code|
+------------------+  <- 0x00010000 (64KB)
|  page directory  |
|  for startup code|
+------------------+  <- 0x0000f000 (60KB)
|  initial kernel  |
|   thread struct  |  初始线程的内核栈和thread结构体
+------------------+  <- 0x0000e000 (56KB)
|        /         |
+------------------+  <- 0x00007e00 (31.5KB)
|   pintos loader  |
+------------------+  <- 0x00007c00 (31KB)
|        /         |
+------------------+  <- 0x00000600 (1536B)
|     BIOS data    |
+------------------+  <- 0x00000400 (1024B)
|     CPU-owned    |
+------------------+  <- 0x00000000
```

最早的PC是基于16位的8088处理器，只能访问1MB的物理地址。**640KB的区域是早期的PC唯一可以使用的RAM区域**，The 384KB area from 0x000A0000 through 0x000FFFFF 被保留给硬件使用，如VGA和BIOS，早期的PC中BIOS是在真的只读的ROM中，但是现在的BIOS被存放在可以更新的闪存中

虽然现在处理器的内存变大了，但是PC的架构还是保留了原始的低1MB的布局，为了保证向下兼容。

因此现代的PC在物理地址的640KB到1MB之间会有一个洞，将RAM分为**"low" or "conventional memory" (the first 640KB)** and **"extended memory" (everything else)**. 

## 启动流程

PC启动之后由BIOS（在ROM中）获得机器的控制权，BIOS将bootloader（在第一块硬盘的第一个sector中，叫做MBR）加载到ram中，从0x7c00 through 0x7dff，然后使用 `jmp` instruction to set the CS:IP to `0000:7c00`，就把机器的控制权移交给bootloader。

- ​	由于**bootloader在实模式下运行**，所以使用的是16位的段寄存器， 使用此公式来计算地址**`address= 16 * segment + offset`**. 所以每一段的大小是64KB，所以bootloader的大小也不能超过64KB

由bootloader负责找到磁盘上的内核，由于现在还处于实模式，只能使用低1MB的物理内存，所以只能将内核代码加载到物理地址 `0x20000`处（通常都是将内核加载到很低的物理内存，除了x86的实模式的限制之外还因为物理内存不一定会很大），然后将内核的entry point从内核镜像（在ELF头文件中）中提取出来，并跳转到内核代码的entry point处，即 `start()` **in** `threads/start.S`

- 基本所有的OS都是这么干的，启动的时候先把内核加载到较低的物理内存中（因为物理内存不一定有3GB这么大），然后再把低位的物理内存一对一映射到虚拟内存，还要映射到虚拟内存的高位（低位留给用户空间），也就是两个虚拟内存映射到同一个物理内存。然后启用分页之后才能跳转到虚拟内存的高位，正式开始执行内核代码。还需要重新创建一个页表，此时只映射虚拟内存的高位，将此页表替换到CR3中

start代码**主要负责将CPU从16位的实模式转换成32位的保护模式**：

1. start代码首先获取机器的实际RAM的大小，BIOS只能检测到最多64MB的内存，所以这就是pintos实际能支持的最大的内存大小
   
   然后再启用A20 line，在没有启用之前，无法访问超过1MB的内存，所以需要启用它。
   
2. 创建基本的页表，并且启用保护模式和分页

   - 这个页表将虚拟地址的低64MB空间映射到相同的物理地址
   - 还将虚拟地址LOADER_PHYS_BASE（默认 `0xc0000000` (3 GB)）也映射到0到64MB物理地址处
   - 实际上我们只想要将3GB以上的位置映射，但是由于我们需要先启用分页才能跳转到3GB的位置，并且启用分页之后PC中的物理地址就变成虚拟地址了，所以如果不将现在所在的64MB以下的位置一一映射到相同的物理地址，那么启用分页之后就会造成混乱
   - 此时创建的是临时页表，pintos_init函数中会调用paging_init，该函数会创建一个新的页表，将3GB以上的虚拟地址一一映射到从0开始的物理地址处，然后使用这个新的页表

3. 页表初始化后，将状态寄存器变成保护模式，并且启用分页，然后初始化段寄存器。我们还没有准备好在保护模式下处理中断，因此我们还要将中断禁用

4. 调用pintos内核的C代码 `pintos_init()`

## 内核初始化（C代码）

`pintos_init()`主要是调用pintos中各个模块的初始化函数，通常命名为`module_init`（`module.c` is the module's source code, and `module.h` is the module's header.）

pintos_init的具体内容在loading中

### pintos库函数

在pintos中实现了一些库函数如malloc： `threads/malloc.h`，要引用这些函数我们需要include他们的头文件。

他们的头文件与C标准的头文件也有相同的名字，但是是完全不同的文件。比如如果我们只引用stdlib.h，试图调用malloc会报错，因为这在pintos中引用的是 `pintos/src/lib/stdlib.h`下的文件，而不是 `/usr/include/stdlib.h`，而malloc在pintos中定义在了 `threads/malloc.h`中。

- 并且pintos也不能引用C标准库的头文件，因为pintos和C标准库不在一个层面上执行

那么编译器是如何知道是`include<stdlib.h>`引用的是pintos代码空间中的stdlib而不是C语言标准库中的 `/usr/include/stdlib.h`？

- 在默认情况下include指令会默认搜索系统中的头文件而不是代码项目中的头文件，但是我们可以通过GCC的flags来实现这一点：
  1. 使用 `-nostdinc`来告诉GCC不要使用C标准库的头文件作为它的头文件搜索路径
  2. 使用`-I/path/to/my/headers`来告诉GCC 使用 `/path/to/my/headers` 作为它的头文件搜索路径

## Thread

`enum thread_status status`：

- `thread_current()` 返回当前正在运行的线程

- ready的线程被放在一个名为ready_list的双向链表中

- `thread_unblock()`可以将blocked状态的线程转换成ready状态，backtrace可以看到一个blocked的线程在等待什么东西

- dying的线程在调度器切换到下一个线程后会被销毁

`uint8_t *stack`：

- 每个线程有一个内核栈，当线程运行时，CPU中的栈指针寄存器会指向线程内核栈的栈顶，**当CPU切换到其他线程时，需要在线程的tcb中保存该线程的栈指针寄存器的值**。除此之外，不需要将该线程其他的寄存器保存在tcb中，它们会被保存在该线程的内核栈上

- 当中断发生时：

  > When an interrupt occurs, whether in the kernel or a user program, an struct intr_frame is pushed onto the stack. When the interrupt occurs in a user program, the struct intr_frame is always at the very top of the page. See section Interrupt Handling, for more information.

`int priority`：

- 线程优先级ranging from PRI_MIN (0) to PRI_MAX (63). 

`struct list_elem allelem`：

- 用来将线程连接到所有线程列表中，每个线程被创建时插入这个线程列表，exit时从线程列表中移除。
- 使用thread_foreach函数来遍历所有的线程，并且对每个线程作用一个函数。
- list_elem结构体内部只有指向前一个list_elem和后一个list_elem的指针，遍历所有的线程和进程列表的方式是：**遍历list_elem链表，通过其中的list_elem对象反向定位到它外层的thread结构体的地址**
- <img src="C:\Users\李博\Desktop\mynotes\OS\pintos学习笔记.assets\image-20230928210636655.png" alt="image-20230928210636655" style="zoom:67%;" />
- <img src="C:\Users\李博\Desktop\mynotes\OS\pintos学习笔记.assets\image-20230929150855472.png" alt="image-20230929150855472" style="zoom:67%;" />

`struct list_elem elem`：

- 用来将线程连接到ready_list的双向链表中

`unsigned magic`：

- 一个特殊的值，被设为THREAD_MAGIC，用来检测线程的内核栈的溢出，所以当我们往thread结构体中添加成员时，需要把magic放在最后

每个thread结构体都独自占据一个page，在page的起始地址的位置，剩下的位置用来存放thread的内核栈，从高地址向低地址增长。也就是说，thread结构体和它对应的栈是相向增长的

```
                  4 kB +---------------------------------+
                       |          kernel stack           |
                       |                |                |
                       |                |                |
                       |                V                |
                       |         grows downward          |
                       |                                 |
                       |                                 |
                       |                                 |
                       |                                 |
                       |                                 |
                       |                                 |
                       |                                 |
                       |                                 |
sizeof (struct thread) +---------------------------------+
                       |              magic              |
                       |                :                |
                       |                :                |
                       |              status             |
                       |               tid               |
                  0 kB +---------------------------------+
```

所以thread结构体不能太大了，内核栈也不能太大了，因此**内核函数不应该分配非静态的大的结构体或数组**，应该使用动态内存分配malloc或者`palloc_get_page()`函数



`void thread_init (void)`：

- 由pintos_inti()调用，主要目的是**创造pintos中的初始线程的thread结构体**。这个初始线程的名字叫main，tid是1

- 在内核的最初的汇编代码start中，为内核初始的C代码分配了内核栈，这样才能运行pintos_init函数。仅有内核栈仍然不够，任何代码都是属于某一个线程的，但是使用汇编代码创建thread结构体太麻烦了，所以我们只能先运行C代码，然后使用C代码为自己创建一个thread结构体。这与其他任何的线程都不同，其他线程是先创建thread结构体和内核栈，再运行代码；而初始线程是先运行代码，再为自己创建一个thread结构体。

- thread_init调用`running_thread`函数，

  - 该函数通过对当前的CPU中的esp向下取整得到当前线程的thread结构体的地址，所以切换了内核栈就相当于切换了thread结构体

  我们通常使用running_thread函数来获取当前运行的thread（如thread_current函数就调用了running_thread），这里实际上是特殊的用法，因为此时当前运行的线程还没有thread结构体，所以我们调用init_thread在该地址处创造一个thread结构体

  <img src="C:\Users\李博\Desktop\mynotes\OS\pintos学习笔记.assets\image-20230928220650255.png" alt="image-20230928220650255" style="zoom:50%;" />

  <img src="C:\Users\李博\Desktop\mynotes\OS\pintos学习笔记.assets\image-20230928220747410.png" alt="image-20230928220747410" style="zoom: 50%;" />

  这个过程还需要start.s的配合，将一整个page留给初始线程的内核栈和thread结构体，并且把内核栈的栈底放在页分界处

  <img src="C:\Users\李博\Desktop\mynotes\OS\pintos学习笔记.assets\image-20230928221110928.png" alt="image-20230928221110928" style="zoom: 67%;" />

`void thread_start (void)`：

- 开启中断并且创建idle线程（idle线程的tid是2），此线程会在没有其他线程ready的情况下被调度。
- 开启中断会导致开始进程调度，因为在中断处理程序返回之前会检查进入中断的次数，如果超过时间片会调用`intr_yield_on_return()`启用调度器。

`void thread_tick (void)`：

- 当每次定时器中断时被定时器中断处理程序调用，用来跟踪中断次数，如果当时间片超时时启用调度器

`void thread_block (void)`：

- 将thread从ready转换成blocked状态

`void thread_unblock (struct thread *thread)`：

- 将给定的thread从blocked转换成ready的状态

`void thread_exit (void) NO_RETURN`：

- 将当前的线程exit，从不返回

`void thread_yield (void)`：

- 出让CPU给调度器，选择一个新的线程运行，有可能还是当前线程

`schedule()`：

- <img src="C:\Users\李博\Desktop\mynotes\OS\pintos学习笔记.assets\image-20230929130432476.png" alt="image-20230929130432476" style="zoom:50%;" />

- 负责切换线程，只被thread_block()，thread_exit()，和thread_yield()三个函数调用。

- 在这些函数调用schedule()之前，他们都要禁用中断，因为要保证线程从running状态切换到其他状态是原子性的

- schedule调用switch_threads()来进行真正的线程切换，switch_thread主要做的事情是将旧线程的寄存器保存在栈上，**将栈顶指针保存在thread结构体中；再将新线程的栈顶指针从新线程的thread中恢复**，从新线程的栈中恢复新线程的寄存器。

  - 这个操作和函数调用有什么区别？为什么不能直接用函数调用？因为它比函数调用多了几个步骤，需要把栈顶指针保存在thread结构体中，还要把新线程的栈顶指针从thread结构体中恢复，这个过程是函数调用做不到的，所以要手动写汇编

  当新线程的栈顶指针恢复到CPU中后，**内核栈改变了，我们实际上就完成了线程的切换**，虽然现在rip没变，但是**由于栈改变了，并且每次函数调用返回地址rip都保存在栈中，所以从这一刻起，我们的代码执行轨迹也就改变了**。

  从宏观的代码执行轨迹上看，线程切换就像所有线程都汇聚到一个点，这个点是switch_threads函数，执行完这个函数后，代码执行轨迹就被内核栈控制着走向另一个轨道。

  所以**线程切换其实本质上是切换内核栈**

- `struct thread *switch_threads (struct thread *cur, struct thread *next)`：

  ```assembly
  .globl switch_threads
  .func switch_threads
  switch_threads:
  	# 将旧线程的寄存器保存在自己的内核栈上
  	pushl %ebx
  	pushl %ebp
  	pushl %esi
  	pushl %edi
  
  	# 获取thread结构体中stack成员的偏移量
  .globl thread_stack_ofs
  	mov thread_stack_ofs, %edx
  
  	# SWITCH_CUR是20，栈顶指针esp加上20，刚好可以跳过上面保存的四个寄存器和schedule调用switch_threads时保存在栈中的返回地址eip，指向保存在栈中的函数参数cur
  	# 所以这行代码相当于把旧线程的thread结构体的地址cur保存到了eax中
  	# 并且返回时，eax是函数的返回值，所以这行代码还实现了将前一个线程的thread结构体返回给prev变量
  	movl SWITCH_CUR(%esp), %eax
  	# eax是旧线程的thread结构体，edx是stack成员相对于thread的偏移量，所以这行代码是将旧线程的栈顶指针保存在thread结构体中
  	movl %esp, (%eax,%edx,1)
  
  	# 同样的，下面的两行代码是将新线程的栈顶指针从thread结构体中恢复到CPU的esp寄存器中
  	movl SWITCH_NEXT(%esp), %ecx
  	movl (%ecx,%edx,1), %esp
  
  	# Restore caller's register state.
  	popl %edi
  	popl %esi
  	popl %ebp
  	popl %ebx
  	# 返回值是eax，也就是旧线程的thread结构体的地址
          ret
  .endfunc
  ```

- 所以**线程切换要求新的线程必须也要在执行switch_thread代码**，即在switch_thread代码中挂起。**所谓线程在执行某一段代码是一个抽象的说法，本质上就是该线程的CPU中的rip指向某段代码，并且内核栈上的保存的现场与这段代码一致**。

  在这个例子中，就要求新线程的内核栈必须满足下面的要求，只有这样，在`movl (%ecx,%edx,1), %esp`后续的代码才能将新线程导向正确的函数调用轨迹

  ```
                         +---------------------------------+
                         |               next              | 
                         |               cur               | eax
  switch_threads_frame   | eip (指向schedule函数中对thread_scheudle_tail的调用)| 
                         |               ebx               | 
                         |               ebp               | 
                         |               esi               | 
                         |               edi               |                                                                        +---------------------------------+ esp
                         |          kernel stack           |
                         |                |                |
                         |                |                |
                         |                V                |
                         |         grows downward          | 
  sizeof (struct thread) +---------------------------------+
                         |              magic              |
                         |                :                |
                         |                :                |
                         |               name              |
                         |              status             |
                    0 kB +---------------------------------+
  ```

  

schedule最后调用`thread_schedule_tail()`函数，该函数：

- 将新的thread标记为running
- 如果旧的thread是dying状态，就释放该线程的tcb所在的page

### 线程和进程的创建流程

完整的进程线程创建流程见lab2

`tid_t thread_create (const char *name, int priority,thread_func *function, void *aux) `:创建一个新的线程，并且在线程中执行function函数，以aux为函数参数

- 分配一个page，将这个page初始化为PCB和该线程对应的内核栈，上面是内核栈，下面是PCB，二者相向增长

- 此时只是创建内核线程，不需要创建页表，所有内核线程暂时共享全局的内核页表（init_page_dir）（实际上此时线程pcb中的pagedir为null，在schedule切换线程时检测到为null会将cr3更新为init_page_dir）

- 如果创建的是进程，需要创建vma链表，child_list和thread_list，并把新创建的进程的状态记录到当前进程的child_list中；如果创建的是线程，那么直接从当前进程复制vma链表，child_list和thread_list即可，并且还需要和当前进程共享pagedir；对于二者都需要将新的线程的状态加入到thread_list中

- 因为我们的调度程序的代码执行流程要求：准备切换到的新线程需要在switch_thread中挂起，并且从switch_thread中退出后还需要执行thread_schedule_tail()进行线程切换的收尾工作，最后完成切换到新线程后还需要执行function函数。

- **所以我们除了为新线程创建thread结构体外，还需要创建上述的三个假的函数栈帧，从而引导新线程第一次被调度时程序控制流的正确性，执行我们期望的代码**。

  ```
                    4 kB +---------------------------------+
                         |               aux               |
  kernel_thread_frame    |            function             |
                         |            eip (NULL)           |                                                    +---------------------------------+
  switch_entry_frame     |      eip (to kernel_thread)     |                                                    +---------------------------------+
                         |               next              | 
                         |               cur               |
  switch_threads_frame   |      eip (to switch_entry)      | 
                         |               ebx               | 
                         |               ebp               | 
                         |               esi               | 
                         |               edi               |                                                    +---------------------------------+
                         |          kernel stack           |
                         |                |                |
                         |                |                |
                         |                V                |
                         |         grows downward          | 
  sizeof (struct thread) +---------------------------------+
                         |              magic              |
                         |                :                |
                         |                :                |
                         |               name              |
                         |              status             |
                    0 kB +---------------------------------+
  ```

- 按照上述的内核栈，新线程的执行流程如下：

  - 从switch_thread函数返回后会执行switch_entry函数，它负责调整栈顶指针越过next和cur，并且调用thread_schedule_tail()函数执行线程切换的收尾工作

    <img src="C:\Users\李博\Desktop\mynotes\OS\pintos学习笔记.assets\image-20230929172844329.png" alt="image-20230929172844329" style="zoom:50%;" />

  - 从switch_entry返回后执行kernel_thread函数，主要是调用function，执行完后调用exit终止线程

    <img src="C:\Users\李博\Desktop\mynotes\OS\pintos学习笔记.assets\image-20230929173219683.png" alt="image-20230929173219683" style="zoom:50%;" />

### 初始线程

所以pintos中，在创建其他线程之前的初始线程有两个，一个是执行内核初始化的pintos_init代码的main线程，tid是1；一个是空闲线程idle，tid是2；空闲线程不属于ready_list中，main线程属于ready_list中，只有当ready_list为空时才会调用idle线程。

## synchronization

最简单粗暴的同步方式是关中断，但是我们应该很少直接设置中断状态，大多数时候我们都是使用其他的同步primitive。

- 关中断的最主要的原因是同步内核线程和外部中断处理程序，**因为外部中断处理程序不能sleep，所以无法使用其他的同步形式**
- 即使关中断，一些外部中断也无法被推迟，如NMI（non-maskable interrupt）
- **关闭中断只能在单核CPU上实现同步**，在多核CPU上每个核都有自己的中断控制和状态，对一个核心关闭中断不会影响其他核心的线程并发访问变量

在pintos中信号量是用禁用中断和thread block和unblock实现的，并且每个信号量都维护了一个等待该信号量的线程list

> **Semaphores can also be initialized to values larger than 1.** These are rarely used.

和其他的同步primitive不同，sema_up()可以在外部中断处理程序中使用

lock（这里指的是睡眠锁）本质上就是初始值为1的信号量，所以在**pintos中lock是用信号量实现的**

- lock和信号量唯一的区别在于：锁只有它的owner才能将它释放，而信号量是任何线程都可以将它增加
- 睡眠锁中要关中断（或者拿自旋锁），在完成进程切换之后就会将中断打开

使用`lock_try_acquire` 可以实现自旋锁，该函数就是尝试获取锁，如果成功了就返回true，失败了不需要阻塞，直接返回false。

条件变量：在pintos的条件变量中，为每个线程维护一个信号量，线程的睡眠和唤醒都是通过信号量来进行的。而不是直接对线程调用block和unblock进行的

- ```c
  char buf[BUF_SIZE];     /* Buffer. */
  size_t n = 0;           /* 0 <= n <= BUF_SIZE: # of characters in buffer. */
  size_t head = 0;        /* buf index of next char to write (mod BUF_SIZE). */
  size_t tail = 0;        /* buf index of next char to read (mod BUF_SIZE). */
  struct lock lock;       /* Monitor lock. */
  struct condition not_empty; /* Signaled when the buffer is not empty. */
  struct condition not_full; /* Signaled when the buffer is not full. */
  
  ...initialize the locks and condition variables...
  
  void put (char ch) {
    lock_acquire (&lock);
    while (n == BUF_SIZE)            /* Can't add to buf as long as it's full. */
      cond_wait (&not_full, &lock);
    buf[head++ % BUF_SIZE] = ch;     /* Add ch to buf. */
    n++;
    cond_signal (&not_empty, &lock); /* buf can't be empty anymore. */
    lock_release (&lock);
  }
  
  char get (void) {
    char ch;
    lock_acquire (&lock);
    while (n == 0)                  /* Can't read buf as long as it's empty. */
      cond_wait (&not_empty, &lock);
    ch = buf[tail++ % BUF_SIZE];    /* Get ch from buf. */
    n--;
    cond_signal (&not_full, &lock); /* buf can't be full anymore. */
    lock_release (&lock);
  }
  ```

优化屏障（optimization barrier）：

- ```c
  #define barrier() asm volatile ("" : : : "memory")
  ```

- 优化屏障的作用是阻止编译器的代码优化和指令重排：

  - 指令asm告诉编译程序要插入汇编语言片段(这种情况下为空)。
  - `volatile`关键字禁止编译器把asm指令与程序中其他指令重新组合。
  - `memory`关键字强制编译器假定RAM中所有内存单元已经被汇编语言指令修改；

  因此，编译器保证编译程序时在优化屏障之前的指令不会在优化屏障之后执行，并且默认经过屏障后变量的值已经被修改了，所以屏障后变量的值必须要从内存读取，所以不能使用存放在CPU寄存器中的变量的值来优化asm指令前的代码。

如这段代码，如果不使用barrier，编译器会认为这段代码永远不会终止，因为start和ticks是相等的并且永远不会改变，因此会把它优化成无限循环。而使用了barrier后，默认ticks和start经过barrier后都被修改了，无法假设他们是否相等，所以编译器就无法优化这段代码；

```c
/* Wait for a timer tick. */
int64_t start = ticks;
while (ticks == start)
  barrier ();
```

如果没有barrier，编译器会把这段代码完全删掉，因为它没有产生任何输出和对系统产生任何作用。barrier强制编译器假装这段代码有重要的作用

```c
while (loops-- > 0)
  barrier ();
```

以及用来保证代码的顺序

```
timer_put_char = 'x';
barrier ();
timer_do_put = true;
```

## Interrupt Handling

中断上下文：处理器总处于以下状态中的一种：

1. 内核态，运行于进程上下文，内核代表进程运行于内核空间；
2. 内核态，运行于中断上下文，内核代表硬件运行于内核空间；
3. 用户态，运行于用户空间。

所谓“**中断上下文**”，就是硬件通过触发信号，导致内核调用中断处理程序，进入内核空间。这个过程中，硬件的一些变量和参数也要传递给内核，内核通过这些参数进行中断处理。中断上下文，其实也可以看作就是硬件传递过来的这些参数和内核需要保存的一些其他环境（**主要是当前被中断的进程环境**）。

中断分为内部中断和外部中断：

- 内部中断是由CPU指令引起的中断，比如系统调用，访问非法内存导致的page fault，除0等；因为它们是由CPU指令引起的 ，所以说他们是和CPU指令同步的，并且**关中断只会禁用外部中断，不会影响内部中断**
- 外部中断通常是由硬件引起的，如定时器，键盘，串口，硬盘等。外部中断和指令执行是不同步的，也就是异步，处理外部中断可以通过关中断推迟。**外部中断运行时是需要关闭中断的，所以外部中断从不嵌套，也不会被抢占CPU，外部中断的处理程序也不会sleep**。所以外部中断中只能考虑自旋锁。
  - 并且在中断上下文中不能访问用户空间虚拟内存。**因为中断上下文是和特定进程无关的，它是内核代表硬件运行在内核空间，所以在中断上下文无法访问用户空间的虚拟地址**。

默认情况下我们说的中断就是外部中断，异常是内部中断。甚至内部中断可以叫同步异常，外部中断可以叫异步异常。这些只是翻译的不同，实际上CSAPP所描述的**中断，陷阱，故障，终止**都是IDT（中断描述符表）中的表项，所以它们都属于中断



8086架构支持256种中断，每张中断各自有一个中断处理程序，这些中断处理程序维护在中断描述符表中（IDT，interrupt decriptor table）。

### 中断处理流程：

- os在启动时会调用`intr_init()`初始化中断描述符表。`intr_init()`中调用`make_gate`为每个表项创建一个64位的中断门（每个中断或异常都有一个特定的向量号，用来索引IDT中的中断门）。这个中断门中包含了中断服务例程的地址和其他信息（如中断门类型和中断的特权级别）。这个中断服务例程就是intr-stub.S中的一个独一无二的入口点，名为intrNN_stub()，中断从这里开始执行，NN是这个中断的数字。

- `intr_init()`只是初始化中断例程，也就是所有的中断此时都是执行同样的操作，还需要为中断注册相应的handler。调用`intr_register_ext` 注册外部中断，调用`intr_register_int`注册内部中断，将指定的handler放在全局的intr_handlers数组中的对应位置。

- 在8086架构中，**int指令是用户进行系统调用的最常用的方法，用户通过使用 `int $0x30`来进行系统调用**。在使用int之前，需要把系统调用号和任何多余的参数放在栈上。因此**在内核中，要获取系统调用的参数，可以通过intr_frame中的esp指针来获取，从esp指针往上，依次是系统调用号和参数0、1、2**。（注意，在系统调用stub中，从int $30回来后还需要把esp往回移动，清除栈中压入的参数）

  <img src="pintos学习笔记.assets/image-20231124153714572.png" alt="image-20231124153714572" style="zoom: 50%;" />

  **然后CPU使用TSS来切换到内核栈，再将一些寄存器保存在内核栈上，包括eip和esp。**

  **`0x30`是系统调用对应的中断向量号**

  <img src="pintos学习笔记.assets/image-20231125173852973.png" alt="image-20231125173852973" style="zoom:67%;" />

- 然后CPU根据中断向量号来索引IDT，找到相应的中断门，从中提取出中断服务例程（即intr_stub）的地址，还要提取出中断门的特权级别，确保有执行的权限。然后CPU跳转到intr_stub

- 因为CPU没有给我们其他方式来得知中断数字，所以`intr_stub`入口点将中断数字push到栈上，随后跳转到intr_entry()函数，**它将所有CPU没有保存的寄存器push到内核栈上**，然后调用intr_handler()，开始执行C代码。
- intr_handler()的主要工作根据中断号从intr_handlers数组中找到之前注册的handler，并调用该函数。它也会对外部中断做一些额外的操作
- intr_handler()返回后，intr-stubs.S中的汇编代码会将之前保存的寄存器恢复到CPU中，并且调用iret从中断中返回

内部中断处理：

- 在内部中断处理程序中，检查甚至修改传给中断handler的intr_frame结构体都是合理的，**对intr_frame的修改在中断返回后会导致对发生中断的线程或进程状态的修改**。比如如果系统调用有返回值，就需要修改intr_frame中的eax寄存器

- 内部中断处理程序和普通的内核代码一样，通常没有特殊的规定哪些事情不能做。内部中断处理程序的中断要打开，所以可以被其他线程抢占，也可以中断嵌套，比如系统调用就有可能导致page fault，出现外部中断。

- `void intr_register_int (uint8_t vec, int dpl, enum intr_level level, intr_handler_func *handler, const char *name)`：

  注册一个内部中断处理程序，中断号是vec，中断处理程序的地址是handler，处理该中断时的中断的开关由level决定；dpl表示中断描述符的特权级别，如果是0，则只有内核线程才能触发这个中断；如果是3，用户程序也能使用INT指令触发中断。dpl不影响用户程序间接触发中断的能力，比如不管dpl是多少，非法内存访问都可以导致page fault

外部中断处理：

- 我们说外部中断运行在一个“中断上下文”中

- **In an external interrupt, the** `**struct intr_frame**` **passed to the handler is not very meaningful.** It describes the state of the thread or process that was interrupted, but there is no way to predict which one that is. **It is possible, although rarely useful, to examine it, but modifying it is a recipe for disaster.**

- 外部中断会垄断机器，所以外部中断应该尽可能快地完成。**任何需要较长CPU时间的操作都应在内核线程中完成，这个内核线程可能是中断通过同步原语触发的**

- 外部中断被一些CPU之外的设备控制，叫做PIC（programmable interrupt controllers）

- `bool intr_context (void)`：如果当前运行在一个中断的上下文中就返回true。通常用在可能会睡眠的函数中

  ```
  ASSERT (!intr_context ());
  ```

- `void intr_yield_on_return (void)`：使用在计时器中断处理程序中，当一个线程的时间片过期时。会**导致在中断处理程序返回之前thread_yield()被调用（即出让CPU，调度另一个线程）**

修改intr_frame中的eax来改变返回值

外部中断和内部中断（即异常）在内核看来是没有区别的，都是通过同样的执行流处理，即先在IDT中查找，执行汇编代码保存现场之类的工作，然后跳转到intr_handler，找到对应的处理函数。唯一的区别就是在intr_handler中，如果发现是外部中断，那么需要将中断关闭，不允许睡眠，从处理函数返回后再开中断。

### intr_frame

intr_frame：int $30之后，CPU首先把用户栈指针esp和eip和标志位保存在内核栈上，然后查询IDT表找到中断向量号对应的中断例程所在的位置，开始执行intr_stub和intr_entry，主要也是将寄存器保存在内核栈上。最后**在调用intr_handler之前，将内核栈的指针esp作为参数传入intr_handler，也就是intr_frame**。**所以intr_frame中成员变量的排列顺序和寄存器保存的先后顺序是一样的**。

```c
/** Interrupt stack frame. */
struct intr_frame
  {
    /* Pushed by intr_entry in intr-stubs.S.
       These are the interrupted task's saved registers. */
    uint32_t edi;               /**< Saved EDI. */
    uint32_t esi;               /**< Saved ESI. */
    uint32_t ebp;               /**< Saved EBP. */
    uint32_t esp_dummy;         /**< Not used. */
    uint32_t ebx;               /**< Saved EBX. */
    uint32_t edx;               /**< Saved EDX. */
    uint32_t ecx;               /**< Saved ECX. */
    uint32_t eax;               /**< Saved EAX. */
    uint16_t gs, :16;           /**< Saved GS segment register. */
    uint16_t fs, :16;           /**< Saved FS segment register. */
    uint16_t es, :16;           /**< Saved ES segment register. */
    uint16_t ds, :16;           /**< Saved DS segment register. */

    /* Pushed by intrNN_stub in intr-stubs.S. */
    uint32_t vec_no;            /**< Interrupt vector number. */

    /* Sometimes pushed by the CPU,
       otherwise for consistency pushed as 0 by intrNN_stub.
       The CPU puts it just under `eip', but we move it here. */
    uint32_t error_code;        /**< Error code. */

    /* Pushed by intrNN_stub in intr-stubs.S.
       This frame pointer eases interpretation of backtraces. */
    void *frame_pointer;        /**< Saved EBP (frame pointer). */

    /* Pushed by the CPU.
       These are the interrupted task's saved registers. */
    void (*eip) (void);         /**< Next instruction to execute. */
    uint16_t cs, :16;           /**< Code segment for eip. */
    uint32_t eflags;            /**< Saved CPU flags. */
    void *esp;                  /**< Saved stack pointer. */
    uint16_t ss, :16;           /**< Data segment for esp. */
  };
```



## 软中断

- 处理器在进行中断处理时，处理器进行异常模式切换，此时会将中断进行关闭，处理完成后再将中断打开；


- 如果中断不分上下半部处理，那么意味着只有等上一个中断完成处理后才会打开中断，下一个中断才能得到响应。当某个中断处理处理时间较长时，很有可能就会造成其他中断丢失而无法响应，这个显然是难以接受的，比如典型的时钟中断，作为系统的脉搏，它的响应就需要得到保障；并且中断处理程序运行在中断上下文中，所以中断处理程序无法使用阻塞的函数，限制了它的作用范围


- 中断分成上下半部处理可以提高中断的响应能力，在上半部处理完成后便将中断打开（通常上半部处理越快越好），这样就可以响应其他中断了，等到中断退出的时候再进行下半部的处理；


##  为什么中断上下文不可以休眠

进程有进程描述符（PCB），可以把现场保存在PCB中；中断运行在中断上下文，没有一个所谓的中断描述符来描述它，它不是操作系统调度的单位，不属于任何进程（虽然中断程序在执行过程中要使用线程的内核栈，但是那只是借用，并不属于中断程序）。

一旦在中断上下文中睡眠，

- 首先无法切换上下文（因为没有中断描述符，当前上下文的状态得不到保存），
- 其次，没有人来唤醒它，因为它不是操作系统的调度单位。

并且中断（外部中断）和当前进程是异步的，与当前的进程没有任何关系；如果因为中断睡眠了导致进程睡眠了，那么对于进程是很不公平的。

并且如果中断可以睡眠，那么会高优先级的进程出现中断后反而会将CPU让给低优先级的进程，这样调度算法就失去了意义。

那么中断为什么没有中断描述符呢？

- 中断的产生是很频繁的（至少每毫秒（看配置，可能10毫秒或其他值）会产生一个时钟中断），并且中断处理过程会很快。如果为中断上下文维护一个对应的task_struct结构，那么这个结构频繁地分配、回收，并且影响调度器的管理，这样会对整个系统的吞吐量有所影响。

中断的时候进程的状态还是运行态，但是它在用户空间的执行暂停了，进到了内核态在它的内核栈上执行中断处理程序

## 虚拟内存的管理

用户进程的虚拟内存是每个进程独一无二的，内核的虚拟内存是所有内核线程全局共享的

OS启动时由bootloader负责找到磁盘上的内核，由于现在还处于实模式，只能使用低1MB的物理内存，所以只能将内核代码加载到物理地址 `0x20000`处，然后跳转到内核代码的entry point处开始执行，即 `start()` **in** `threads/start.S`

start汇编：

1. 创建基本的页表，并且启用保护模式和分页

   - 这个页表将虚拟地址的低64MB空间映射到相同的物理地址
   - 还将虚拟地址LOADER_PHYS_BASE（默认 `0xc0000000` (3 GB)）也映射到0到64MB物理地址处（这里是64MB是因为pintos的BIOS最多只能检测64MB的物理地址）
   - 实际上我们只想要将3GB以上的位置映射，但是由于我们需要先启用分页才能跳转到3GB的位置，并且启用分页之后PC中的物理地址就变成虚拟地址了，所以如果不将现在所在的64MB以下的位置一一映射到相同的物理地址，那么启用分页之后就会造成混乱
   - 此时创建的是临时页表，pintos_init函数中会调用paging_init，该函数会创建一个新的页表，将3GB以上的虚拟地址一一映射到从0开始的物理地址处，然后使用这个新的页表
2. 页表初始化后，将状态寄存器变成保护模式（从此可以访问32位的空间），并且启用分页，然后初始化段寄存器。我们还没有准备好在保护模式下处理中断，因此我们还要将中断禁用
3. 调用pintos内核的C代码 `pintos_init()`

操作系统中有很多先有鸡还是先有蛋的问题，比如在虚拟内存管理中，有几个要点：

1. 所有的操作系统都是将RAM划分为两部分，一部分用来存放内核的代码和 **静态数据**，剩下一部分由虚拟内存系统来动态分配给内核或者用户态
2. 必须要**将内核启用分页的代码所在的物理地址一对一映射到虚拟地址**，否则执行完启用分页的指令后会造成混乱
3. **内核的初始页表必须映射到所有的物理内存地址，因为只有这样内核才能分配物理内存**。所以**内核是通过虚拟地址来管理物理内存的分配的**。分配的方式是通过内核获取一个没有使用的frame，然后通过内核页表的映射方式（内核页表的映射方式往往非常简单，xv6采用的是一对一映射，pintos采用的是直接加上一个偏移量即可）找到frame对应的物理地址，然后映射到用户页表中。所以**用户页表中的所有物理frame都同时映射到两个页表**。包括xv6也是这样的，之前做xv6的时候没有注意到这一点，因为xv6的内核虚拟地址和物理地址是一一映射，所以从内核内存分配器拿到page后不需要将其转换为物理地址，直接映射到用户地址空间即可。

在pintos_init中，当内存管理（palloc_init）初始化结束后，内核重新创建一个页表，将3GB以上的虚拟地址一一映射到从0开始的物理地址处，映射完所有的物理内存。页目录（pd）3GB以下的空间是没有使用的。然后使用这个新页表

**随后，如果只创建内核线程，是不会创建页表的，所有的内核线程共享一张全局的内核页表**。

- 在thread_create和thread_init中都没有创建页表的操作，只会创建thread结构体和内核栈。
- 在schedule中，内核线程切换时，如果切换到的线程有页表，那么就加载新线程的页表；如果没有，就使用全局的内核页表

切换线程时只需要切换esp（即内核栈指针）即可，通过内核栈指针可以寻址到线程结构体。也不需要切换PC，因为通过用户栈可以控制程序流（在一个线程最开始创建的时候，在它的内核栈上创建假的栈帧，从而控制切换到它之后的程序控制流的走向）

<img src="pintos学习笔记.assets/image-20240111204948313.png" alt="image-20240111204948313" style="zoom:67%;" />

![IMG_6342](pintos学习笔记.assets/IMG_6342.JPG)

thread_create函数中不需要创建页表，这里仅仅是创建内核线程，所有刚创建出来的内核线程临时共享init_page_dir。如果需要加载用户程序（比如执行start_process），再调用pagedir_create将init_page_dir复制一份。这里**仅仅是分配一个物理页复制init_page_dir页目录所在的那一页**，所以所有页目录3GB以上的空间永远都是相同的，并且指向同样的页表，创建出来的新的页目录在3GB以下的空间是没有被使用的。

<img src="pintos学习笔记.assets/image-20240111204745052.png" alt="image-20240111204745052" style="zoom:67%;" />

然后把可执行文件加载到新的页目录中（如果使用按需分页，都不需要将用户程序映射到页表上，只需要分配vm_area_struct即可）

<img src="pintos学习笔记.assets/image-20231125163752275.png" alt="image-20231125163752275" style="zoom:67%;" />

在process_active中将新的页表激活，**然后将tss指向内核栈。只有这样，发生中断时，CPU才知道要切换回哪个内核栈**。所以如果CPU有多个核，那么就有多个TSS。

注意！**在页表销毁时，不是将整个页目录所有指向的空间全部释放，而是先释放页目录3GB以下空间所指向的页表和物理页，再释放整个页目录所在的物理页；页目录3GB以上空间所指向的页表和物理页不要释放！因为是和其他页目录共享的！**所以在上图中，是先释放1，再释放2

<img src="pintos学习笔记.assets/image-20240111205846674.png" alt="image-20240111205846674" style="zoom:67%;" />

注意！**对页表已经存在的PTE进行修改时一定要刷新TLB**！原因：

- 本质上是因为只有通过硬件进行地址转换的时候才会经过TLB，将PTE缓存在TLB中。而我们自己写的代码对页表进行的任何读写操作都是不经过TLB的，所以当页表被更新时，TLB并不知情，其中相应的条目不会被更新。

PHYS_BASE（3GB）划分了内核虚拟内存和用户虚拟内存，3GB以上的是内核虚拟内存，以下的是用户虚拟内存

pintos只有在刚开始启动时（实模式）可以直接访问物理地址，后来在init中启用分页后只能访问虚拟地址。所以创建全局的内核页表时，**pintos将内核虚拟内存（也就是PHYS_BASE以上的部分）一对一映射到物理内存中（从0开始）**，直到物理内存耗尽。

**此后分配内存分配的都是内核虚拟地址，包括将可执行程序加载到用户进程的地址空间中时**：首先向用户的pool中申请一个page，得到的是内核虚拟地址。然后将文件读取到这个内核虚拟地址（对应的物理页）中，再调用install_page将这个内核虚拟地址（对应的物理地址）映射到给定的用户虚拟地址上。这样才完成了向用户地址空间中写入数据。

- 实际上，**所有的操作系统一定都是通过虚拟内存来管理page的分配的**，之前做xv6的时候没有注意到这一点，因为xv6的内核虚拟地址和物理地址是一一映射，所以拿到page后不需要将其转换为物理地址，直接映射到用户地址空间即可。

  freememory是用户空间和内核共用的，由内核负责分配，当用户空间内存不够的时候就从freememory申请内存映射到页表，内核需要内存的时候也要从freememory申请内存；kalloc返回的是虚拟地址，但是也等同于物理地址，因为freememory是物理地址和虚拟地址直接映射的，所以物理地址和虚拟地址的数值是一样的

<img src="pintos学习笔记.assets/image-20231202214948336.png" alt="image-20231202214948336" style="zoom:67%;" />





用户进程空间布局：

```
   PHYS_BASE +----------------------------------+
             |            user stack            |
             |                 |                |
             |                 |                |
             |                 V                |
             |          grows downward          |
             |                                  |
             |                                  |
             |                                  |
             |                                  |
             |           grows upward           |
             |                 ^                |
             |                 |                |
             |                 |                |
             +----------------------------------+
             | uninitialized data segment (BSS) |
             +----------------------------------+
             |     initialized data segment     |
             +----------------------------------+
             |           code segment           |
  0x08048000 +----------------------------------+
             |                                  |
             |                                  |
             |                                  |
             |                                  |
             |                                  |
           0 +----------------------------------+
```

用户空间的布局由链接脚本决定，通过指定程序段的名字和地址。

calling convention：

f(1, 2, 3)函数最开始的内存布局：

```
                             +----------------+
                  0xbffffe7c |        3       |
                  0xbffffe78 |        2       |
                  0xbffffe74 |        1       |
stack pointer --> 0xbffffe70 | return address |
                             +----------------+
```

pintos的C库将用户程序的入口点指定为`_start()`函数，

```c
void
_start (int argc, char *argv[])
{
  exit (main (argc, argv));
}
```

所以执行 `/bin/ls -l foo bar` 命令的过程是：内核中execute函数以这个字符串为参数，先将它切成单词，然后**将这些单词放在用户栈的栈顶**，也就是从PYHS_BASE开始向下增长。再向栈中push刚才这些单词的地址，最后push argv和argc，然后push一个假的返回地址，因为main函数是用户进程中的第一个函数，不存在返回地址。

| Address    | Name           | Data       | Type          |
| ---------- | -------------- | ---------- | ------------- |
| 0xbffffffc | `argv[3][...]` | bar\0      | `char[4]`     |
| 0xbffffff8 | `argv[2][...]` | foo\0      | `char[4]`     |
| 0xbffffff5 | `argv[1][...]` | -l\0       | `char[3]`     |
| 0xbfffffed | `argv[0][...]` | /bin/ls\0  | `char[8]`     |
| 0xbfffffec | word-align     | 0          | `uint8_t`     |
| 0xbfffffe8 | `argv[4]`      | 0          | `char *`      |
| 0xbfffffe4 | `argv[3]`      | 0xbffffffc | `char *`      |
| 0xbfffffe0 | `argv[2]`      | 0xbffffff8 | `char *`      |
| 0xbfffffdc | `argv[1]`      | 0xbffffff5 | `char *`      |
| 0xbfffffd8 | `argv[0]`      | 0xbfffffed | `char *`      |
| 0xbfffffd4 | `argv`         | 0xbfffffd8 | `char **`     |
| 0xbfffffd0 | `argc`         | 4          | `int`         |
| 0xbfffffcc | return address | 0          | `void (*) ()` |

可以使用stdio.h中定义的hex_dump函数来debug传参代码

### 系统调用

你在user目录下添加一个sleep文件，这个文件会被编译成可执行文件，挂载到xv6上，然后你就可以在xv6的shell的命令行里面调用这个sleep程序，比如sleep 10，然后你的这个命令行字符串“sleep 10”会传入到shell程序中，shell程序会先把自己fork一份，由fork出的子进程exec sleep进程，exec的时候会把“sleep 10”这个字符串传递给sleep进程，作为sleep进程main函数的参数。你要做的是先从sleep程序的main函数的参数中提取出10这个数字，然后把10作为sleep系统调用的参数，调用sleep系统调用在用户态的接口（定义在user/user.h中，实现在user/usys.S中）就行了

![image-20240129164000055](pintos学习笔记.assets/image-20240129164000055.png)

## 内核内存分配

pintos有两个内存分配器，一个以page为单位分配内存，一个可以分配任意大小的内存。

**注意！pintos是通过虚拟内存来管理page的分配的**

**注意！pintos是通过虚拟内存来管理page的分配的**

实际上，所有的操作系统一定都是通过虚拟内存来管理page的分配的，之前做xv6的时候没有注意到这一点，因为xv6的内核虚拟地址和物理地址是一一映射，所以拿到page后不需要将其转换为物理地址，直接映射到用户地址空间即可。

page allocator将内存分为两个pool，内核和用户空间各一个，默认情况下二者各占ram的一半。初始化时从1MB到ram的结束部分的物理内存转换成虚拟内存后平分给二者（因为1MB以下是内核的代码和静态数据）。将内核和用户的pool隔离开是为了保证当用户疯狂swap page时内核的稳定运行。

![image-20231123180111109](pintos学习笔记.assets/image-20231123180111109.png)

在这两个pool的起始处都有一个bitmap，用来追踪pool中的page的使用情况。一个分配n个page的请求会遍历bitmap，找到连续的n个值为false的bit，然后将他们变成ture（也就是first fit分配策略）

请求分配内存时有三种flag：

**`PAL_ASSERT`**

- **If the pages cannot be allocated, panic the kernel.** 

**`PAL_ZERO`**

- **Zero all the bytes in the allocated pages before returning them.** 

**`PAL_USER`**

- **Obtain the pages from the user pool.** 如果没有设置，则默认从内核pool分配

当一个page被释放后，所有的byte都变成 `0Xcc`

block allocator基于page allocator，每次当一个block描述符中的block不足时都要从page allocator请求一个page再将其打碎成相应大小的block，添加到block 描述符的list中。

具体策略看代码

由于任何大于一个连续page的内存请求都有可能会失败（由于外部碎片），所以我们应该尽量减少大于4kb的内存请求



## 页表

pintos中使用`uint32_t*`来表示页表，因为每个PTE的大小刚好是32位，uint32_t*就表示一个指针，指向PTE的数组。pintos中使用的是二级页表，因为地址是32位，每个PTE或者PDE的大小是4字节，所以用10位来索引页目录，10位来索引页表；而xv6中使用的是三级页表，地址是39位，每个PTE或者PDE的大小是8字节，用9位来索引页目录和页表，一共使用27位索引，剩下12位是页内偏移

pintos中的页表结构如下：**PDE和PTE中的前20位存放的都是物理地址，是用来给MMU寻址的，MMU直接访问这个物理地址**，从而访问页表或者物理数据页。

**但是内核软件寻址的时候不能访问物理地址，所以需要将前20位转化成内核虚拟地址后再访问**。

```
 31                  22 21                  12 11                   0
+----------------------+----------------------+----------------------+
| Page Directory Index |   Page Table Index   |    Page Offset       |
+----------------------+----------------------+----------------------+
             |                    |                     |
     _______/             _______/                _____/
    /                    /                       /
   /    Page Directory  /      Page Table       /    Data Page
  /     .____________. /     .____________.    /   .____________.
  |1,023|____________| |1,023|____________|    |   |____________|
  |1,022|____________| |1,022|____________|    |   |____________|
  |1,021|____________| |1,021|____________|    \__\|____________|
  |1,020|____________| |1,020|____________|       /|____________|
  |     |            | |     |            |        |            |
  |     |            | \____\|            |_       |            |
  |     |      .     |      /|      .     | \      |      .     |
  \____\|      .     |_      |      .     |  |     |      .     |
       /|      .     | \     |      .     |  |     |      .     |
        |      .     |  |    |      .     |  |     |      .     |
        |            |  |    |            |  |     |            |
        |____________|  |    |____________|  |     |____________|
       4|____________|  |   4|____________|  |     |____________|
       3|____________|  |   3|____________|  |     |____________|
       2|____________|  |   2|____________|  |     |____________|
       1|____________|  |   1|____________|  |     |____________|
       0|____________|  \__\0|____________|  \____\|____________|
                           /                      /
```

实际PTE中的页内偏移量部分用来存储标志位，pintos的PTE中只有一个present位，用来表示该PTE对应的 数据在不在内存中，没有valid位。如果PTE对应的数据页调入内存了，则将该位设为true，调出去了则设为false

```
 31                                   12 11 9      6 5     2 1 0
+---------------------------------------+----+----+-+-+---+-+-+-+
|           Physical Address            | AVL|    |D|A|   |U|W|P|
+---------------------------------------+----+----+-+-+---+-+-+-+
```

## 虚拟内存的两种方式

传统方式是像pintos这样的，**内核线程和用户进程使用同一张页表，所有页表的内核部分映射到相同的PTE和物理地址，用户部分映射到各自的用户程序**。虽然可以将内核空间对应的PTE的权限设置为用户不能访问，但是用户还是可以通过某些手段攻击内核。

一种更加安全的方式是将内核线程和用户进程的页表分离（像XV6这样的），**所有的内核线程使用同一张内核页表，每个用户进程使用各自的用户页表**，这样无论如何用户都无法访问到内核空间了。

二者在实现上有些区别：

1. 前者进入内核时不需要切换页表，所以发生中断时，**直接跳转到内核中的中断例程，将现场保存在内核栈上即可**。

2. 后者进入内核时需要切换页表，但是不能直接切换，因为用户页表和内核页表的映射不一样，如果直接切换会导致程序崩溃（PC不变，但是物理地址变了）。所以需要在用户页表和内核页表中的相同的虚拟地址处分配一块空间，映射到一块相同的物理地址。这块空间叫做trampoline，用来存放中断例程（中断入口代码），负责切换页表和保存现场。所以还需要在用户页表中分配一块区域保存现场，叫做trapframe。

   所以这种方法还是得有一部分内核代码存放在用户页表中，执行完这段代码之后才能跳转到内核页表。

## 请求0的页

未初始化的全局变量和静态变量和初始化为0的全局变量静态变量放在bss段中，bss段在编译成可执行文件时基本不占空间，只有元数据；在load时会根据数据的大小为bss段分配虚拟内存为全0；访问bss段时发生pagefault，将一个物理页用0填充后映射到对应的虚拟地址中，不需要从磁盘读取数据；这种在虚拟内存空间中初始化全为0的页称为请求0的页（demand-zero page）。实际上mmap的匿名映射原理差不多，也是将虚拟地址空间中的某一段映射为全0（而不是像普通映射一样，将这一段映射到某个指定的文件），访问时发生pagefault，也是向内核请求全为0的物理页映射到这里。

- 请求0的页的实现机制：在vma结构体中有字段表示这个映射范围中，有多少字节需要read，有多少字节是0，对于前者需要按照偏移量去硬盘中读取，对于后者，则内核直接给它分配0即可

## 信号机制

信号只能由内核向某一个进程发出，或者由内核代替某一个进程向其他进程发出（包括自己）

| 编号 |   名称    |  默认动作   |                           对应事件                           |
| :--: | :-------: | :---------: | :----------------------------------------------------------: |
|  2   | `SIGINT`  |    终止     | 当用户输入`Ctrl+C`时，内核会向前台作业发送`SIGINT`信号，该信号默认终止该作业。 |
|  9   | `SIGKILL` |    终止     |     该信号的默认行为是用来终止进程的，无法被修改或忽略。     |
|  11  | `SIGSEGV` | 终止且 Dump | 段冲突 Segmentation violation，当你试图访问受保护的或非法的内存区域，就会出现段错误，内核会发送该信号给进程，默认终止该进程。 |
|  14  | `SIGALRM` |    终止     |          时间信号，设置定时器，超时后将信号发给自己          |
|  17  | `SIGCHLD` |    忽略     | 当子进程终止或停止时，内核会发送该信号给父进程，由此父进程可以对子进程进行回收。 |

信号的实现是在每个进程的PCB中维护了标志位，表明此时该进程收到了哪些信号，还有一些其他的标志位，表明要阻塞哪些信号

我们将发送了但是还没被接收（处理）的信号称为**待处理信号（Pending Signal）**。**同类型的信号至多只会有一个待处理信号(pending signal)**，比如说进程有一个 `SIGCHLD` 信号处于等待状态，那么之后进来的 `SIGCHLD` 信号都会被直接扔掉。

- **内核为每个进程在`pending`位向量中维护待处理信号的集合**，pending位向量中的每一位都对应着一个特定的信号。这就是为什么信号的接收不排队，因为每个信号在位向量中仅有一个位表示。当内核传递一个信号时，设置pending中的对应的位。当信号被接收时，pending中的对应的位被清除。

SIGSEGV信号：

- 当发生pagefault时，进程会立即从用户态进入内核态，调用内核中的pagefault handler，
  - 如果操作系统在处理pagefault时发现这个pagefault是不可恢复的，比如写入了一个只读页面，那么就会给当前进程的PCB中标志位进行标记；然后内核处理完pagefault handler之后，从内核中返回到用户态之前，会检查进程PCB中收到了哪些信号；此时就会发现收到了SIGSEGV信号，然后将进程终止
  - 如果OS发现pagefault时可以恢复的，比如按需分页，那么完成后就照常返回用户态

SIGKILL信号：`SIGKILL`信号的处理与大多数其他信号有所不同。当进程收到`SIGKILL`信号时，它几乎会立即被操作系统终止：不存在“等待进程收到中断或异常进入内核态，从内核态返回用户态之前检查到`SIGKILL`信号”的过程。`SIGKILL`的处理不需要进程参与，也不会等待任何条件，它直接由操作系统内核执行，操作系统会立即从进程表中移除进程，并回收它所使用的资源。

可以在函数中调用`kill`函数来对目的进程发送信号

每个信号类型都有一个预定义的**默认行为**，是下面中的一种：

- 进程终止。
- 进程终止并转储内存。
- 进程停止（挂起）直到被 SIGCONT 信号重启。
- 进程忽略该信号。

比如，收到 SIGKILL 的默认行为就是终止接收进程。另外，接收到 SIGCHLD 的默认行为就是忽略这个信号。我们这里可以通过`signal`函数来修改信号的默认行为，给某个信号绑定处理函数，但是无法修改`SIGSTOP`和`SIGKILL`信号的默认行为

- 从内核态回到进程A的用户态时，内核检测到进程A的标志位pending的未处理信号，转而执行对应的信号处理程序。执行结束后，先返回内核态，再返回进程A的用户态，继续执行进程A的程序。

```c++
#include <signal.h>
typedef void (*sighandler_t)(int); 
sighandler_t signal(int signum, sighandler_t handler);
```

**系统调用可以被中断。像 read、write 和 accept 这样的系统调用潜在地会阻塞进程一段较长的时间，称为慢速系统调用**。

当睡眠中的进程收到信号的时候，如果该信号没有被进程屏蔽，操作系统会将进程的状态从睡眠（或等待）状态改变为可运行状态，进程会被加入到调度器的就绪队列中，等待再次被调度执行。被调度了之后，进程仍然在从内核态回到用户态之前处理信号，调用信号处理函数或者直接返回用户态。

- 在某些较早版本的 Unix 系统中，被中断的进程在信号处理程序返回时不再继续，而是立即返回给用户一个错误条件，并将 errno 设置为 EINTR。**在这些系统上，程序员必须包括手动重启被中断的系统调用的代码**。
- 某些系统环境和编程语言运行时提供了自动重新启动被信号中断的系统调用的能力。这意味着，如果系统调用（如`sleep()`）因为信号到来而被中断，它会在信号处理完成后自动重新开始，而不是直接返回`EINTR`错误。

## futex

`futex`系统依赖于一个普通的整数变量来表示锁的状态，这个变量位于普通的进程地址空间中，而不是特定的共享内存区域。

1. **用户空间的锁操作**：线程首先在用户空间尝试获取锁，通过对共享变量进行原子操作（如`compare_and_swap`）。**如果获取锁成功，无需进行系统调用**；如果失败，则表示锁已被占用。
2. **等待锁**：如果线程无法立即获得锁，它将使用`FUTEX_WAIT`操作进入睡眠状态，直到锁变量改变。这一步涉及系统调用，因为需要内核的帮助来挂起线程并在条件满足时将其唤醒。
3. **唤醒操作**：当持有锁的线程释放锁（即修改了共享变量），它将使用`FUTEX_WAKE`操作唤醒一个或多个正在等待该锁的线程。这也涉及到系统调用，以便内核能够管理等待队列并唤醒睡眠中的线程。

`futex`设计的一个关键优势：如果`futex`（快速用户空间互斥）操作在尝试获取锁时成功了，那么它就不需要进入内核态。在锁没有竞争的情况下（即锁可以被立即获取），所有的操作都可以在用户空间完成，避免了进行系统调用和进入内核态的开销。

futex实际上类似于sleep和wakeup，主要依赖于两个系统调用：`FUTEX_WAIT`和`FUTEX_WAKE`。

- **`FUTEX_WAIT`**：这个调用使当前线程挂起，直到`futex`变量的值发生变化，或者发生超时。
- **`FUTEX_WAKE`**：这个调用唤醒一个或多个等待指定`futex`变量的线程。



## 库用法

### strtok用法

第一次调用strtok传入的字符串应该是原来的字符串，`strtok` 在内部使用一个静态指针来记住这个字符串的起始位置。

然后`strtok` 搜索第一个分隔符（由第二个参数指定的字符集中的任一字符）。找到后，`strtok` 会将该分隔符替换为 `\0`（空字符），返回这个token

然后更新内部的静态指针，指向刚刚替换的分隔符的下一个字符。

后面调用strtok传入的应该是NULL，告诉strtok从上次停止的地方（即静态指针指向的地方）继续操作；

当strtok在字符串的剩余部分找不到更多的分隔符是，返回NULL

**由于 `strtok` 使用的是静态指针来记住上次操作的位置，所以它不是线程安全的。**在一个线程中对 `strtok` 的调用可能会干扰另一个线程中的 `strtok` 调用。此外，因为**它会修改原始字符串**，所以使用时需要小心处理。

![image-20231124230047955](pintos学习笔记.assets/image-20231124230047955.png)

### const规则

**const修饰它紧邻的类型**，比如

- `const int *ptr;`，这里的const修饰int，表示ptr指向的int数据的值不能被修改
- `int *const ptr;`，这里的const修饰*，表示指针的值不能被修改

### strlcpy用法

**size的长度是字符串的长度加上终止符**

![image-20231124231545295](pintos学习笔记.assets/image-20231124231545295.png)

### memcpy用法

可以接受任意类型的字符串，因为void*是一种通用类型的指针，任何类型的指针都可以隐式转换成void *

![image-20231125101827653](pintos学习笔记.assets/image-20231125101827653.png)

### memset

**memset是把目标地址处的n个byte每一个都变成value，而不是把连续的一段内存变成一个value**！

如果想在指定的内存地址处放入数字，使用如下写法：

```c
  uint32_t sp = PYHS_BASE;
  *(uint32_t *)sp = sp + 4; // argv地址
```

### 函数指针

`uint64_t (*syscalls[])(void)`：先从`(*syscalls[])`开始读起，表示声明了一个变量叫syscalls，先与`[]`结合，表示这个变量是一个数组，再与*结合，表示数组中的每一个元素是一个指针。

- `[]`优先级高于*号

最后再与`uint64_t (void)`结合，表示指针指向一个函数，函数的参数是`void`，返回值是`uint64_t`。

# Lab 0

现在的pintos的pintos_init函数在把内核初始化完后需要处理输入的命令行中的命令，如`pintos -- run alarm-zero`，要处理的命令就是run alarm-zero。

<img src="C:\Users\李博\Desktop\mynotes\OS\pintos学习笔记.assets\image-20230927182025561.png" alt="image-20230927182025561" style="zoom:67%;" />

如果没有命令（比如`pintos --`）则直接退出内核

我们要做的是在没有命令的情况下弹出一个交互式的shell（这是一个内核级的shell，在后面我们会实现常规的用户级的shell）

> Enhance **threads/init.c** to implement a tiny kernel monitor in Pintos.
>
> Requirments:
>
> - It starts with a prompt `**PKUOS>**` and waits for user input.
> - **As the user types in a printable character, display the character.**
> - When a newline is entered, it parses the input and checks if it is `**whoami**`. If it is `whoami`, print your student id. Afterward, the monitor will print the command prompt `PKUOS>` again in the next line and repeat.
> - If the user input is `**exit**`, the monitor will quit to allow the kernel to finish. For the other input, print `invalid command`. Handling special input such as backspace is not required.
> - If you implement such an enhancement, mention this in your design document.

```c
  if (*argv != NULL) {
    /* Run actions specified on kernel command line. */
    run_actions (argv);
  } else {
    // TODO: no command line passed to kernel. Run interactively 
    while(1){
      puts("PKUOS>");
      bool is_exit = false;
      char *buffer = (char*)malloc(10 * sizeof(char));
      for(int i = 0;;i++){
        char c = input_getc();
        putchar(c);
        if(c != '\n' && c != '\r'){
          // 处理backspace
          if(c != 127)buffer[i] = c;
          else i--;
          continue;
        } 
        // 注意！一定要记得在字符数组结束后添加终止符0
        buffer[i] = 0;
        // strcmp相等返回0
        if(strcmp(buffer,"whoami") == 0){
          printf("fuckyou ,i dont have student id\n");
          break;
        }
        if(strcmp(buffer,"exit") == 0){
          is_exit = true;
          break;
        }
        printf("invalid commmand\n");
      }
      if(is_exit)break;
    }
  }
```

# Lab 1



## 实验记录

首先寻找更新了ticks变量的位置，使用grep命令在项目根目录下所有文件中递归查找变量名

```bash
grep -r -n --include='*.c' '\<ticks\>' .
```

- -r表示在指定目录下递归查询，
- -n表示查询结果标注行号，
- --include=用来指定要搜索的文件名或扩展名，使用`*.c`来匹配所有的c语言文件，
- `'\<ticks\>'`表示搜索的字符串的名字，并且使用`\<` 表示词的开头，`\>` 表示词的结尾，从而匹配完整的ticks，而不是作为某个单词的一部分；
- 最后使用.表示在当前目录下查询

还可以将grep的输出管道到wc，统计输出的行数

```bash
grep -r -n --include='*.c' '\<ticks\>' . | wc -l
```

还可以统计加不加`\<>\`的grep结果的差异：

```bash
grep -r --include='*.c' 'ticks' . > gp1.txt
grep -r --include='*.c' '\<ticks\>' . > gp2.txt
diff gp1.txt gp2.txt
```

最后发现，只有在timer.c文件的定时器中断handler中才更新了ticks变量



方便编译：

创造名为recmp.sh的shell文件：

```shell
cd ..
make
cd build
```

然后 `chmod +x ./recmp.sh`，再向.bashrc中添加 `alias recmp="~/recmp.sh"`，再重启shell `source ~/.bashrc`



### 初始线程

pintos中，在创建其他线程之前的初始线程有两个，一个是执行内核初始化的pintos_init代码的main线程，tid是1；一个是空闲线程idle，tid是2；空闲线程不属于ready_list中，main线程属于ready_list中，只有当ready_list为空时才会调用idle线程。

### list的使用

list中有两个哨兵节点head和tail

- begin返回的是head.next，即第一个元素；但是end返回的不是最后一个元素，而是tail；这样方便我们遍历list；
- rbegin返回的是最后一个元素，rend返回的是head，而不是第一个元素，这样方便我们反向遍历list

具体使用方式：

```c
      struct foo
        {
          struct list_elem elem;
          int bar;
          ...other members...
        };
      struct list foo_list;

      list_init (&foo_list);
       struct list_elem *e;
// 正向遍历
      for (e = list_begin (&foo_list); e != list_end (&foo_list);
           e = list_next (e))
        {
          struct foo *f = list_entry (e, struct foo, elem);
          ...do something with f...
        }
// 反向遍历
      for (e = list_rbegin (&foo_list); e != list_rend (&foo_list);
           e = list_prev (e))
        {
          struct foo *f = list_entry (e, struct foo, elem);
          ...do something with f...
        }
// remove操作
   for (e = list_begin (&list); e != list_end (&list); e = list_remove (e))
     {
       ...do something with e...
     }
```

### 傻逼错误

长时间不写C和C++就容易出现的傻逼错误：

- 对于结构体和对象，不要用值传递，用指针传递！

  ```c
      struct list * threads = &sema->semaphore.waiters;
      ASSERT (list_size (threads) == 1);
      // 上下的结果天差地别
      struct list threads = sema->semaphore.waiters;
      ASSERT (list_size (&threads) == 1);
  ```

### Task 1

重新实现timer_sleep，原来的版本是在while循环中反复调用thread_yield出让CPU，等当前线程下次被调度到了再判断时间是否满足要求，如果不满足就继续yield。

<img src="C:\Users\李博\Desktop\mynotes\OS\pintos学习笔记.assets\image-20231006211421154.png" alt="image-20231006211421154" style="zoom:67%;" />

修改后，在全局添加一个链表，和一个结构体记录睡眠的线程的信息（内部记录睡眠时间和thread结构体（从设计上来说为了睡眠这种边缘功能修改核心的thread结构体不太好，所以单独使用一个结构体，记录睡眠时间，并且指向thread结构体）），每当有线程调用timer_sleep，就在链表中添加一个结构体，然后将线程block。每次tick的时候，遍历整个链表，将所有睡眠线程的时间减一，将时间为0的线程unblock

### Task 2

> 实现优先级调度：
>
> - 当一个线程被加入ready_list时，如果该线程比当前线程的优先级高，则立马将CPU让给该线程
> - 唤醒在条件变量、信号量或者睡眠锁上阻塞的线程时，优先唤醒优先级最高的线程
> - 当降低线程的优先级的时候，如果该线程不再是优先级最高的线程了，那么要马上让出CPU
>
> 注意，在中断上下文中不允许出让CPU，只能调用intr_yield_on_return函数，在中断结束后让出CPU

实现方法就是遍历ready_list找到优先级最高的线程

> 优先级调度中还有一个问题是优先级反转：
>
> - 比如优先级分别为高中低的H,M,L三个线程，如果L拿着一个锁，而H等待L的锁，M正在CPU中运行，所以这就导致了H永远无法运行，反而低优先级的M可以运行
> - 解决方法是当L拿着锁时，H把优先级先捐赠给L，等L把锁释放后（因此H可以获取锁了）再把优先级拿回来

这里的优先级捐赠应该不是阻塞的线程优先级和拿锁线程的优先级直接相加，否则就打破了优先级的规则了，应该只是将阻塞线程中最大的优先级给拿锁线程重置

实现的方法就是在pcb里维护一个list，将当前线程所持有的所有的锁都记录下来，从而找到被当前线程阻塞的线程，再维护一个指针，指向阻塞当前线程的线程。每次有线程被阻塞，都可以沿着指针，用该线程的优先级更新前面线程的优先级。

### Task3

上面的优先级算法会导致低优先级的进程被饿死，于是使用多级反馈队列算法，它可以自动改变线程优先级。

优先级算法适用于实时系统，因为它可以让程序员控制哪些任务拥有多少CPU的时间。但是对于通用的操作系统，还是多级反馈队列更好。

#### 多级反馈队列

**recent_cpu用来衡量该线程最近使用了多少CPU时间，使用的CPU时间越多，优先级下降地就越快**（此方法避免了线程饿死，因为如果一个线程最近没有使用CPU，那么它的优先级会很快地升高）。CPU时间**以ticks为单位，每次tick给当前正在运行的线程增加一次计数**。所以准确的计算方法应该是维护一个长为n的数组，每个元素记录某一秒分配给该线程的ticks数。每一秒结束后将这个数组中的所有元素求和再平均。但是这种方法会占用较多空间，所以实际pintos中使用了指数加权移动平均（exponentially weighted moving average）来**估计**每个线程过去所使用的CPU时间。具体使用下面的公式：

![image-20231117172524097](pintos学习笔记.assets/image-20231117172524097.png)

使用这个公式估计的是线程**最近**的CPU使用时间，因为将它展开后，可以发现，距离当前越远的recent_cpu的权重就越低，距离当前越近的权重就越高。

具体的：

- recent_cpu的初始值为0，**每次tick**给当前正在运行的线程加一，**每秒**给所有线程（不管是running、ready还是blocked）按照上面的公式重新计算一次

load_avg表示最近正在竞争CPU的线程个数，它也是使用移动平均估计出来的数值，系统启动时初始值是0，然后**每秒**按照下面的公式更新一次，ready_threads是当前正在竞争CPU的线程个数，即running和ready线程个数之和，不包括idle线程。

![image-20231117173534578](pintos学习笔记.assets/image-20231117173534578.png)

所以**当前竞争CPU的线程越多，load_avg就越高，recent_cpu衰减地就越慢，priority就下降地越快**

![image-20231117173933462](pintos学习笔记.assets/image-20231117173933462.png)

除了这些之外，每个线程还有一个nice值，表示它对其他线程有多好（nice），从-20到20。nice值越高就对其他线程越好，也就是越谦让，优先级越低。init线程的nice值是0，其他线程的nice值从他们的父进程继承而来。

priority数值的范围是从0到63，每四次tick按照上面的公式重新计算一次。每次计算截断小数，向下取整

- 除了每4次tick计算一次priority之外，每次调用 ` void thread_set_nice (int new_nice)`设置nice值时也要重新计算一次priority。如果当前运行的线程不再是最高优先级了，那么就让出CPU

所以本算法的主要实现内容就是：

- **每次定时器中断更新mlfqs的数据，更新完后再更新优先队列**
  - **每次定时器中断就按照上面的算法更新一次mlfqs的数据**，如果某个线程的优先级被更新了，那么就将该线程移动到优先级数组指定位置的链表中。到时间片了就调用intr_yield_on_return，按照多级队列调度器调度新的线程即可。
- **调度时从优先队列数组选线程**
  - 在全局维护一个数组，数组下标为从0到63，每个位置是一个链表，再在pcb内部加一个链表节点，将所有的ready的线程串在数组对应位置的链表上。调度器每次从高到低遍历这个数组，再从最高优先级的链表中选择一个线程调度。使用mlfqs算法时同时维护ready_list和这个优先队列数组，调度器中不再是从ready_list中轮询，而是**从优先队列数组中选择**。

除此之外在内核中不能使用浮点数进行计算，而是使用整型来模拟定点数计算。

## 进程调度和锁总结

信号量的实现：信号量结构体中维护了一个等待线程的链表，由于是单核的操作系统，所以实现信号量直接是关中断，如果value是0则将线程加入链表中，然后状态变为阻塞；而lock则是使用1值信号量实现的

优先级调度：

- 当一个线程被unblock加入ready_list时，如果该线程比当前线程的优先级高，则立马将CPU让给该线程
- 唤醒在条件变量、信号量或者睡眠锁上阻塞的线程时，遍历信号量的等待链表，优先唤醒优先级最高的线程
- 每次改变线程的优先级的时候，都要在ready_list中遍历，如果该线程不再是优先级最高的线程了，那么要马上让出CPU
  - 在中断上下文中不允许出让CPU，只能调用intr_yield_on_return函数，在中断结束后让出CPU



> 优先级调度中还有一个问题是优先级反转：
>
> - 比如优先级分别为高中低的H,M,L三个线程，如果L拿着一个锁，而H等待L的锁，M正在CPU中运行，所以这就导致了H永远无法运行，反而低优先级的M可以运行
> - 解决方法是当L拿着锁时，H把优先级先捐赠给L，等L把锁释放后（因此H可以获取锁了）再把优先级拿回来

这里的优先级捐赠应该不是阻塞的线程优先级和拿锁线程的优先级直接相加，否则就打破了优先级的规则了，应该只是将阻塞线程中最大的优先级给拿锁线程重置

实现的方法就是在pcb里维护一个list，将当前线程所持有的所有的锁都记录下来，从而找到被当前线程阻塞的线程，再维护一个指针，指向阻塞当前线程的线程。**每次有线程被阻塞，都可以沿着指针，用该线程的优先级更新前面线程的优先级**。

mlfqs：

- **调度时从优先队列数组选线程**
  - 在全局维护一个数组，数组下标为从0到63，每个位置是一个链表，再在pcb内部加一个链表节点，将所有的ready的线程串在数组对应位置的链表上。调度器每次从高到低遍历这个数组，再从最高优先级的链表中选择一个线程调度。使用mlfqs算法时同时维护ready_list和这个优先队列数组，调度器中不再是从ready_list中轮询，而是**从优先队列数组中选择**。
- **每次定时器中断更新mlfqs的数据，更新完后再更新优先队列**
  - 每次定时器中断就按照上面的算法计算所有线程的mlfqs的数据，更新他们的优先级，如果某个线程的优先级被更新了，那么就将该线程移动到优先级数组指定位置的链表中。到时间片了就调用intr_yield_on_return，按照多级队列调度器调度新的线程即可。

# Lab 2

在执行文件系统的代码时需要使用同步，一次只能有一个进程执行文件系统的代码。

Lab1的测试代码是内核的一部分，所以我们测试的指令是：在--后面指定测试文件的名字，代表让pintos内核运行这个文件

```
pintos -v -k -T 60 --gdb --bochs  -- -q  -mlfqs run mlfqs-load-1
```

但是lab2是实现系统调用和用户程序，测试是在用户态的，所以测试是运行在pintos之上的，所以**我们在运行任何测试或者用户程序之前需要在qemu中创建一个带有文件系统分区的模拟磁盘，然后格式化文件系统，再把测试文件（或者用户程序）复制到文件系统中**，

具体的：

pintos-mkdisk程序提供了创建模拟磁盘的功能：在userprog/build目录下执行 `pintos-mkdisk filesys.dsk --filesys-size=2`，会创建一个名为filesys.dsk的模拟磁盘，包含2MB的pintos文件系统分区。

格式化文件系统分区：`pintos -- -f -q`，-f表示文件系统格式化，-q表示格式化结束后pintos立马exit

使用`pintos -p file -- -q`将文件复制到pintos文件系统中，这个过程由官方提供的pintos脚本实现，所以-p和file名字在--前面。将文件复制出文件系统使用-g替代-q

当然，**复制到文件系统（指想要执行的文件）的文件必须要是已经编译好的elf格式的文件**。

完整过程：

```
$ pintos-mkdisk filesys.dsk --filesys-size=2
$ pintos -- -f -q
$ pintos -p ../../examples/echo -a echo -- -q
$ pintos -- -q run 'echo PKUOS'
```

我们可以使用 `pintos -q rm file`从pintos文件系统中删除文件

在此lab中，用户空间中无法使用malloc和浮点数运算

## Exercise 1.1

用户进程的创建过程：

在内核中调用process_execute将指定可执行文件加载为用户进程。其中，先调用thread_create创建一个内核线程，

<img src="pintos学习笔记.assets/image-20231124220700304.png" alt="image-20231124220700304" style="zoom:67%;" />

**该内核线程负责将可执行文件加载到此线程的用户空间中，也就是将自己3GB以下的页表覆盖**。然后调用intr_exit，假装从内核中断返回到用户空间。

<img src="pintos学习笔记.assets/image-20231124220920905.png" alt="image-20231124220920905" style="zoom:67%;" />

thread_create的过程就相当于fork，不管表面如何变化，本质上来说，加载可执行程序就像一个夺舍的过程，先准备好肉身，也就是fork出来的新的进程/线程，然后才能让可执行程序借尸还魂，也就是将exe文件加载到用户页表中。

## Exercise 2.1

为了向新创建的用户进程传递命令行参数，我们需要在start_process中将可执行文件load进页表中后，向从PHYS_BASE开始的用户栈中保存命令行参数的字符串和指针。

```c
static uint32_t pass_argument(const char **argv) {
  uint32_t sp = PHYS_BASE;
  uint32_t u_arg_address[MAXARG];
  int argc;
  for (argc = 0; argv[argc]; argc++) {
    sp -= strlen(argv[argc]) + 1;
    sp -= sp % 4;
    strlcpy((char*)sp, argv[argc], strlen(argv[argc]) + 1); // strlcpy的size是字符串长度加终止符
    u_arg_address[argc] = sp;
  }
  u_arg_address[argc] = 0;
  sp -= (argc + 1) * sizeof(uint32_t);
  sp -= sp % 4;
  strlcpy((char*)sp, (char*)u_arg_address, (argc + 1) * sizeof(uint32_t));
  sp -= 4;
  *(uint32_t *)sp = sp + 4; // argv地址
  sp -= 4;
  *(uint32_t *)sp = argc;
  sp -= 4;
  *(uint32_t *)sp = 0; // fake return address
  return sp;
}
static char ** split_command_line(char *command_line) {
  // command_line的大小不会超过128字节，我使用给它分配的后半的page存指针
  char **argv = (char**)(command_line + PGSIZE / 2);
  char *save_ptr;
  int argc = 0;
  for (char *token = strtok_r(command_line, " ", &save_ptr); token != NULL; token = strtok_r(NULL, " ", &save_ptr)) {
      argv[argc++] = token; 
  }
  argv[argc] = 0;
  return argv;
}
```

实现完了发现测试还没跑操作系统就结束了，经排查发现是process_wait还没实现，应该让os等待task跑完再结束

<img src="pintos学习笔记.assets/image-20231125172231178.png" alt="image-20231125172231178" style="zoom:50%;" />

## Task 3: Accessing User Memory

验证系统调用时用户传入的参数中的地址是否合法，比如地址不能是未映射的区域，不能是内核区域。如果用户进程传入了不合法的地址，需要直接将进程终止并释放资源。

有两种实现方式：

1. 在系统调用handler中对传入的地址软件遍历页表判断是否合法，如果不合法直接终止进程即可
2. 在系统调用handler中只判断地址不大于PHYS_BASE即可，然后直接执行对应的系统调用程序，当程序中解引用该指针时会发生page fault，在page fault handler中再终止进程释放资源
   - 此方法的好处是解引用指针使用的是MMU，比软件遍历页表更快。但是由于中途发生了page fault后线程直接exit，打乱了程序控制流，导致内核线程之前分配的资源没有来得及释放，比如lock、malloc等。而方法1的进程终止时内核线程还没来得及执行，所以直接exit即可。

启示：写内核时控制流是不受控制的，很有可能出现一个exception导致线程终止，从而该内核线程之前在内核中申请的资源无法释放。内核中尽量避免使用malloc，真实的内核中可能会有GC的机制

我的实现：在系统调用handler中对传入的地址软件遍历页表，取出对应的PTE，查看是否valid即可

## Task 4: System Calls

exit时先调用process_exit，切换到全局内核页表，将之前使用的页表释放。但是**exit不会释放pcb等资源，因为当前进程还在运行，它不能自己把自己的资源释放了**。再调用thread_exit，将当前线程从进程列表中移除，将状态设为dying，再将CPU让出，进行进程调度。

不管什么系统，进程的PCB都是由另一个进程来释放的，而pintos中的exit与一般系统的区别在于：

- 正常系统的exit流程：**子进程exit后的资源由父进程释放**。如果父进程在子进程之前exit了，那么子进程就会变成**孤儿进程**，由init进程接管。

- pintos的exit流程：**僵尸进程的资源不是由父进程释放的，而是由调用exit后调度的下一个进程释放的**：在schedule中调用`thread_schedule_tail`，它将激活新进程的页表，然后如果旧进程是僵死状态，就释放旧进程的PCB以及内核栈。**并且孤儿进程不需要分配给新的进程**

<img src="pintos学习笔记.assets/image-20231130133741356.png" alt="image-20231130133741356" style="zoom: 50%;" />

<img src="pintos学习笔记.assets/image-20231130133816913.png" alt="image-20231130133816913" style="zoom:50%;" />

pintos要求**有且只有**父进程能够**随时用**wait查看子进程的退出状态，所以我在PCB中维护了一个链表，记录进程的所有子进程的退出信息；每次exec一个新的子进程，就在链表中记录一下进程的id。

- 当进程wait一个子进程时，在链表中查找这个子进程，如果不存在，则返回错误；如果存在，则在链表中对应的进程结构体中sleep，直到该子进程退出；

- 当线程exit时只销毁PCB，不销毁这个链表中的元素，并且通过当前进程的parent指针找到父进程，遍历父进程中的这个链表，找到exit进程的结构体，将exit_state记录在其中，并且wakeup上面阻塞的父进程（将信号量加一）。

这样即使子进程销毁了，父进程也能随时调用wait查看退出状态。销毁PCB时，将链表中的所有结构体也销毁，因为只有父进程能够查看退出状态，这些结构体已经没有存在的意义了。每次wait完之后也要把结构体删了，因为pintos规定只能在一个pid上wait一次

- 本质上，父进程要想wait子进程，不管是正常unix的wait（负责等待子进程exit并且回收子进程的资源）还是pintos中的wait（负责等待子进程exit并且查看子进程的退出状态），就**必须在PCB中记录有哪些子进程**。

join的实现：wait规定只能由父进程wait子进程，而join规定一个进程的线程之间可以相互join；实现wait需要在进程中维护链表记录所有的子进程的信息，实现join则需要在进程中再维护一个链表记录进程中的所有线程的信息。具体的实现逻辑与wait基本相同

### 进程和线程总结

所以**每个进程中维护两个链表，一个child_list链表记录所有子进程中主线程的状态，一个thread_list链表记录当前进程中的所有线程的状态**；

- 这两个链表分别用来实现进程的wait、exit和线程的join、pthread_exit，线程状态结构体中记录了线程的tid、是否是主线程、线程的退出状态码、和一个信号量。
  - 当父进程wait某个子进程、或者join同一个进程中的某个线程的时候，需要分别在这两个链表中查找到对应进程或线程的状态结构体，然后在结构体中的信号量上睡眠；
  - 当调用exit或者pthread_exit时，前者需要去父进程的child_list中查找自己的状态结构体，后者需要去当前进程的thread_list中查找自己的状态结构体，然后唤醒在上面睡眠的线程
- 由于实现了多线程之后有可能出现一个线程调用exit导致整个进程退出的情况，此时其他的线程可能还持有锁，所以需要在每个进程里给持有的锁和信号量维护两个链表，不管是用户态还是内核态，每次拿锁的时候都记录在链表中，然后进程exit的时候就unlock这个链表里的所有锁
- 当一个线程调用了exit（不是pthread_exit）的时候，此时需要exit整个进程，需要先在线程的thread_list中找到所属进程的主线程的tid，然后使用这个tid去父进程的child_list链表中找到当前进程的状态，然后更新它。



`tid_t thread_create (const char *name, int priority,thread_func *function, void *aux) `:创建一个新的线程，并且在线程中执行function函数，以aux为函数参数

- 分配一个page，将这个page初始化为PCB和该线程对应的内核栈，上面是内核栈，下面是PCB，二者相向增长

- 此时只是创建内核线程，并且在pintos中会马上load新进程的数据，所以这里甚至不需要像copy on write fork一样复制一份页表，而是根本就不需要创建页表，所有内核线程暂时共享全局的内核页表（init_page_dir）（实际上此时线程pcb中的pagedir为null，在schedule切换到此线程时检测到为null会将cr3更新为init_page_dir）

  - 如果是fork一个新的进程，那么这里就需要根据当前进程的页表创建新的页表了，当前进程3G以上的部分只复制页目录，3G以下的部分页表和页目录全部复制一遍，并且把3G以下的pte映射到的frame的引用计数全部加1，pte全部改成只读的，**并且复制一份当前进程的vma链表**；然后thread_create就可以结束了，function内什么都不需要执行，直接返回用户态即可。写用户态数据时会发生pagefault，进入内核后找到发生pagefault处的pte，将frame的数据复制一份到另一个frame，映射到当前进程的页表中，并且将两个pte都改成可读可写的

    当exec时，把vma链表释放，重新根据可执行文件创建一个vma链表，再把3G以下的pte全部设为invalid

    进程释放时，对于3G以下的空间，释放页表和页目录，释放页表时会将frame的引用计数减1，如果引用计数变为0了，那么就说明这个frame空闲了；对于3G以上的空间，只释放页目录，因为3G以上的页表是所有进程共享的

- 如果创建的是进程，需要**创建vma链表，child_list和thread_list**，并把新创建的进程的状态记录到当前进程的child_list中；如果创建的是线程，那么直接从当前进程复制vma链表，child_list和thread_list即可，并且还需要和当前进程共享pagedir；对于二者都需要将新的线程的状态加入到thread_list中

- 因为我们的调度程序的代码执行流程要求：准备切换到的新线程需要在switch_thread中挂起，并且从switch_thread中退出后还需要执行thread_schedule_tail()进行线程切换的收尾工作，最后完成切换到新线程后还需要执行function函数。

- **所以我们除了为新线程创建thread结构体外，还需要创建上述的三个假的函数栈帧，从而引导新线程第一次被调度时程序控制流的正确性，执行我们期望的代码**。

  ```
                    4 kB +---------------------------------+
                         |               aux               |
  kernel_thread_frame    |            function             |
                         |            eip (NULL)           |                                                    +---------------------------------+
  switch_entry_frame     |      eip (to kernel_thread)     |                                                    +---------------------------------+
                         |               next              | 
                         |               cur               |
  switch_threads_frame   |      eip (to switch_entry)      | 
                         |               ebx               | 
                         |               ebp               | 
                         |               esi               | 
                         |               edi               |                                                    +---------------------------------+
                         |          kernel stack           |
                         |                |                |
                         |                |                |
                         |                V                |
                         |         grows downward          | 
  sizeof (struct thread) +---------------------------------+
                         |              magic              |
                         |                :                |
                         |                :                |
                         |               name              |
                         |              status             |
                    0 kB +---------------------------------+
  ```

- 创建完内核线程之后，新的线程的状态被设为ready，等待CPU的调度；等到CPU调度到它时，按照上述的内核栈，新线程的执行流程如下：

  - 从switch_thread函数返回后会执行switch_entry函数，它负责调整栈顶指针越过next和cur，并且调用thread_schedule_tail()函数执行线程切换的收尾工作

    <img src="C:\Users\李博\Desktop\mynotes\OS\pintos学习笔记.assets\image-20230929172844329.png" alt="image-20230929172844329" style="zoom:50%;" />

    thread_schedule_tail函数中会将新的线程的页表激活，但是此时新的线程刚刚创建完，页表还是NULL，于是激活的是全局的内核页表

  - 从switch_entry返回后执行kernel_thread函数，主要是调用function，执行完后调用exit终止线程

    <img src="C:\Users\李博\Desktop\mynotes\OS\pintos学习笔记.assets\image-20230929173219683.png" alt="image-20230929173219683" style="zoom:50%;" />

如果function是start_process，那么主要工作就是load磁盘中的可执行文件到内存中：先调用pagedir_create将init_page_dir复制一份。这里**仅仅是分配一个物理页复制init_page_dir页目录所在的那一页**，所以所有页目录3GB以上的空间永远都是相同的，并且指向同样的页表，**创建出来的新的页目录在3GB以下的空间是没有被使用的**。然后完全不需要在页表中进行任何映射和分配任何空间，只需要创建vma结构体并添加到进程的vma链表中，记录load的可执行文件的每一个section的信息即可，比如文件描述符、section在文件中的范围、加载到内存后的虚拟地址的范围等等

如果function是start_pthread，那么主要工作是为pthread线程创建一个用户栈的vma，然后添加到进程的vma链表中，并且在用户栈中为新的线程传递参数

注意，线程和进程主要的区别就是

- 如果创建的是进程，需要**创建vma链表，child_list和thread_list**，并把新创建的进程的状态记录到当前进程的child_list中；如果创建的是线程，那么直接从当前进程复制vma链表，child_list和thread_list即可，并且还需要和当前进程共享pagedir，然后在vma链表中新建一个用户栈；对于二者都需要将新的线程的状态加入到thread_list中

#### 用户态同步的实现

在内核中维护一个lock的数组，当调用lock_init时，进入内核，在数组中的空闲位置创建一个lock，将该lock在数组中的下标返回给用户。用户态的锁本质上就是一个int类型，就用这个下标作为锁的值。

如果要对这个锁获取或者释放，都要经过系统调用进入内核，在内核的数组中找到这个锁，然后对它进行获取或释放，或者在这个锁上睡眠



open系统调用的实现：**pintos中的进程打开文件表与unix中不同，不能继承给子进程**

### bug1：所有内核线程共享内存

为新进程创建页表时，是直接复制一份全局的内核页目录，所以不同进程的内核页目录指向同一个页表，更指向同样的物理页。**所有内核线程共享内存**，所以即使是在不同的内核线程中，也不能重复释放同一个内核虚拟page

错误写法：父进程执行`process_execute`创建子进程，子进程执行`start_process`，两个线程共享内核虚拟地址fn_copy。当子进程执行结束时会释放fn_copy，但是当子进程load失败的时候，父进程也会释放fn_copy。所以出现错误

```c
tid_t
process_execute (const char *command_line) 
{
  char *fn_copy;
  fn_copy = palloc_get_page (0);
  strlcpy (fn_copy, command_line, PGSIZE);

  char **argv = split_command_line(fn_copy);
  bool load_success = true;
  struct semaphore sema;
  sema_init(&sema, 0);
  void* args[3] = {argv, &load_success, &sema};
  /* Create a new thread to execute command_line. */
  tid = thread_create (argv[0], PRI_DEFAULT, start_process, args);

  if (tid != TID_ERROR) {
    sema_down(&sema); // 如果成功创建了新线程，还要等新线程成功load可执行文件才能返回
    tid = load_success ? tid : TID_ERROR;
  }
  if (tid == TID_ERROR) {
    palloc_free_page (fn_copy);
  }
  return tid;
}

/** A thread function that loads a user process and starts it
   running. */
static void
start_process (void *args)
{
  ASSERT(args != NULL);
  void **args_array = (void**)args;
  const char **argv = (const char**)args_array[0];
  bool *load_success= (bool*)args_array[1];
  struct semaphore *sema = (struct semaphore*)args_array[2];

  const char *file_name = argv[0];
  struct intr_frame if_;
  bool success;

  success = load (file_name, &if_.eip, &if_.esp);
  *load_success = success;

  // 加载完成，唤醒父进程
  sema_up(sema); 

  /* If load failed, quit. */
  palloc_free_page ((void*)file_name);
  if (!success) {
    thread_exit ();
  }
```

正确的做法是：

![image-20231201131649458](pintos学习笔记.assets/image-20231201131649458.png)

### bug2：list_remove的正确用法

![image-20231201150442258](pintos学习笔记.assets/image-20231201150442258.png)

## Task 5: Denying Writes to Executables

阻止向正在运行的可执行文件写入。

方法是在open文件时，比较文件名和当前所有正在运行（所有状态）的线程的名字，如果有相同的，就调用`file_deny_write`将open的打开文件结构体置为不可写。close文件是要调用`file_allow_write`将打开文件结构体重新置为可写。

# Lab 3a

在VM目录下工作，此目录只包含makefile文件，其与userprog中的相比唯一的改变就是打开了 `-DVM`的设置

当读或写一个page时，CPU会将PTE的accessed位设为1；当写page时，CPU将脏位设为1 。CPU从来不会将这些位设为0，但是OS会。

swap slot是磁盘上的交换分区的一个连续的，page大小的区域。**交换分区用来保存被OS换出内存的、并且在内存中被修改过的page，没有被修改过的page被驱逐时不需要写回磁盘**。所以发生page fault时，如果该数据页从未被读入过内存，就从文件系统读；如果之前被读入后来被换出，则从交换分区读。

- 如果OS支持共享内存，那么发生page fault的数据还有可能已经在内存的frame中了，只不过没有映射到当前进程的页表中。在这种情况下OS也需要定位到这个frame
- 如果补充页表表示用户进程不应该访问这个地址处的任何数据，或者这个数据页在内核虚拟地址（只有发生page fault的地址在用户空间才能为其按需分配），或者尝试向只读的page写入，那么访问是非法的。任何非法访问应该终止进程，并且释放内核线程所有的资源。**补充页表的一个作用就是记录内核线程有哪些资源需要释放。**另一个作用是记录相应的数据页的位置

映射到用户地址空间的page（frame）必须是从user pool中获取的。如果内核中没有空闲的frame了，并且swap分区满了，那么就panic内核；在真实的OS通常会有一系列的策略来从这种情况恢复。

**user pool的page不会常驻内存，kernel pool的page必须常驻内存，不允许换出**；**所以frame实际上就是用户pool中的page**，因为：

- 在操作系统的设计中，一个基本的原则是：并非所有的物理页都可以交换出去的，只有映射到用户空间且被用户程序直接访问的页面才能被交换，而被内核直接使用的内核空间的页面不能被换出。这里面的原因是什么呢？操作系统是执行的关键代码，需要保证运行的高效性和实时性，如果在操作系统执行过程中，发生了缺页现象，则操作系统不得不等很长时间（硬盘的访问速度比内存的访问速度慢2~3个数量级），这将导致整个系统运行低效。而且，不难想象，处理缺页过程所用到的内核代码或者数据如果被换出，整个内核都面临崩溃的危险。
- 在内核中不允许发生page fault，所以如果内核访问用户空间的代码时会出现危险。我们应该在lab2系统调用的基础上，对访问用户空间的地址进一步做限制，**在进入系统调用时，将传入的用户虚拟地址对应的frame pin住，表示在内核中的时候，不要将这个frame替换出去**。同时应该还要限制访问用户地址空间的范围不能横跨多个page？否则有可能访问到下一个page的时候发生page fault？

驱逐frame的过程：

1. 根据frame对应的PTE上的acc位和脏位来运行置换算法，选择一个要驱逐的frame
2. 移除所有页表对该frame的引用（除了内核页表）
3. 如果有必要，将这个frame中的数据写回文件系统或者swap分区
4. 然后用该frame存储另一个page的数据

当两个虚拟页映射到同一个frame时，访问这个frame只会更新访问所通过的PTE，不会更新所有映射到同一个frame的PTE，这会导致不准确，**我们应该更新所有的PTE**。

- 在pintos中，所有的内核页和用户页都映射到同一个物理页，为了避免这个问题，在内核中应该只通过用户虚拟地址访问用户数据。（如果使用了用户进程之间共享内存，则另当别论）

交换表（swap table）：负责追踪使用和空闲的swap slot

- 当把物理页驱逐到交换分区时，使用它选择一个未使用的swap slot
- 当一个进程终结时，还要释放这个进程在交换分区中占据的swap slot

swap slot应该使用懒分配，也就是说只有在被驱逐数据需要的时候才分配；当swap slot中的内容被写回frame的时候，将其释放

使用`pintos-mkdisk swap.dsk --swap-size=n`命令，当我们在运行pintos的时候会自动挂载一个额外的硬盘作为交换分区；或者在run pintos时直接使用`--swap-size=n`命令，会使用一个临时的nMB交换磁盘

## 我的实现

按需分页本质上和mmap是一样的，按需分页本质上也是将可执行文件中的段映射到地址空间中，发生pagefault时将其导入即可。

在PCB中维护一个 mm_struct结构体，该结构体中维护一个vm_area_struct链表。该链表描述该进程地址空间中的段。

<img src="pintos学习笔记.assets/image-20231219225849179.png" alt="image-20231219225849179" style="zoom: 67%;" />

exec中load可执行文件时，只需要为每一个加载入内存的可执行文件的段创建一个vm_area结构体即可，不需要将数据实际加载入内存。

- 当发生pagefault时，根据fault_addr找到vm_area，再根据fault_addr在该段中的偏移量从vm_area记录的文件中的对应偏移量的位置加载数据。为该页数据分配page和frame结构体，再将该page install到页表中。

在内核中维护一个frame table，frame table是一个数组，元素是frame结构体，每个结构体代表一个frame。

- frame结构体和user pool的每个page之间一一映射，通过frame结构体在frame table中的偏移量可以找到对应的frame的地址
- 如果frame被使用，还会记录该frame对应的页表和虚拟地址是多少。

当调用palloc分配page失败时（即frame满了），使用页面替换算法遍历frame table，需要从中选出frame驱逐出去。使用的方式是使用clock算法，在frame_table中维护一个clock指针，遍历所有frame，通过frame中的页表和虚拟地址找到对应的PTE，访问PTE的acc位和脏位，记为（0,0）。CPU会将acc位和脏位置为1

1. 第一轮查找（0,0），也就是既没有被访问过也没有被修改过的
2. 如果一轮找完了，就把clock指针再找一圈，这一次找(0,1)，并且将所有扫描过的访问位变为0
3. 如果还没找到，就重复第一轮。最多四轮一定能找到可替换的page

找到替换的page后，如果是脏的，就写回交换分区，否则直接丢弃。维护一个swap_table来追踪交换分区的使用情况。将数据在交换分区中的位置维护在PTE的前20位中，并且在PTE中新增一个swap位。

当发生pagefault时，就根据PTE中有没有swap标志位，决定是去交换分区中获取数据还是从文件中获取数据。

![image-20231212195752134](pintos学习笔记.assets/image-20231212195752134.png)

内核和用户空间是同一张页表，访问用户空间数据可以通过内核的user pool分配的page，也可以通过用户页表，但是实际上只有可能通过用户页表，因为系统调用传入的地址就是用户虚拟地址，所以访问用户空间数据一定会更新相关PTE上的数据。

## bug1

像page_fault 懒加载这种东西是典型的对称式的代码，load的时候保存，page_fault的时候恢复，只需要在保存和恢复的时候都打印一下内容，判断是否对的上即可

## bug2

![image-20231212145300823](pintos学习笔记.assets/image-20231212145300823.png)

## bug3

运行page_linear测试时，调试方法是：将每一次驱除page对应的用户虚拟地址，frame_no和交换分区的idx，每次pagefault时读取的page对应的用户虚拟地址，frame no和交换分区的idx都打印出来，一一对比哪里有问题。找此bug唯一的困难就是耐心。

经排查发现，将脏页驱逐到交换分区，再从交换分区中read到内存中，install的时候没有将PTE的脏位保持为true，导致PTE的脏位是false，所以下一次该page被驱逐的时候内容被直接丢弃了，再fetch的时候丢失了前一次的修改。

## bug4：不允许在内核发生pagefault

不允许在内核中发生pagefault，否则会造成死锁，而内核发生pagefault唯一的可能性就是执行系统调用时访问传入的地址。所以需要在系统调用的入口处将传入的地址全部访问一遍，在这里触发pagefault。并且触发完后，在整个系统调用过程中，这些page不能被驱逐出去，否则又会发生pagefault了

![image-20231214220627413](pintos学习笔记.assets/image-20231214220627413.png)

等待系统调用执行完毕，再将这些page unpin

<img src="pintos学习笔记.assets/image-20231214220650391.png" alt="image-20231214220650391" style="zoom:67%;" />

但是实际上内核中还是无法完全避免pagefault： `tests/vm/pt-write-code2`

## bug5：frame和page的释放顺序

alloc的顺序是先获取page再标记frame（ref和pin），那么释放的时候就应该是先释放frame再释放page，否则会导致其他线程获取了page后发现frame正在被使用

![image-20231215112545213](pintos学习笔记.assets/image-20231215112545213.png)

## bug6 

发现两次free了同一个kpage，经排查发现是两个pte指向同一个kpage，那么显然是在驱逐其中一个PTE的数据时没有将PTE重置为0

![image-20231216123534163](pintos学习笔记.assets/image-20231216123534163.png)

## bug7 删代码的时候长点心

lab3测试通过之后我就注释掉了下面的这行printf，导致编译器报警说swap_slot_idx没用过，然后没有经过思考就把swap_slot_idx那一句删了，导致没有把数据从交换分区fetch到frame中

![image-20240113214102709](pintos学习笔记.assets/image-20240113214102709.png)

## bug8 并发控制的方法不能混着用

比如我指定用mm->lock保护mm->list，那么就不能用关中断简单了事，是不会起到任何保护作用的

## bug9

线程7想要释放frame 59，但是执行到一半的时候发生了中断；然后切换到了线程7

<img src="pintos学习笔记.assets/image-20240116153247327.png" alt="image-20240116153247327" style="zoom:67%;" />

线程7在pagefault中通过替换算法想要把frame 59驱逐到swap上，然后把frame 59换成了自己的内容；然后又切换到了线程6，线程6不知情，把frame 59的内容释放了。这就出现了问题，因为线程6想要释放的frame 59的内容已经被线程7驱逐了，此时线程6释放的是线程7的frame。

本质上来说，在并发的情况下，**判断是否要执行一个操作和执行这个操作的过程，和使得执行这个操作的条件不成立的代码应该是互斥的**。

在pintos中，一个并发的场景是pagefault和pagefault之间的竞争，或者是pagefault和pagedir_destory之间的竞争；这二者的竞争中，又具体分为页表的竞争和user pool的竞争

比如：

- 页表的竞争：线程A要exit，调用pagedir_destory，线程A先判断pte是存在的，可以释放；

  ![image-20240116164459472](pintos学习笔记.assets/image-20240116164459472.png)

  if执行完后切换到了线程B，B发生了pagefault，请求frame的时候使用替换算法恰好选到了线程A准备释放的frame，然后把PTE清空，把指向的frame驱逐出去。再切换到A，此时A再调用pte_get_page获取到的frame就是错误的frame

- user pool的竞争：A执行完if之后继续执行pte_get_page，获得pte指向的frame F；然后切换到了线程B，B使用替换算法把F的内容换成自己的内容，然后再切换到A，此时A再执行palloc_free_page，释放的就是B的frame

修改：

直接关中断是不行的，因为pintos中关中断无法完全实现原子性，如果获取睡眠锁，还是会进行线程切换；尤其是在palloc_free_page和palloc_get_page中，需要和外设进行IO，在write和read中必须要开中断才行。

对每个进程的页表设置一把锁是很复杂的，因为其他进程发生pagefault时有可能把另一个进程的页表修改了。于是我在全局设置了一把大锁pagedir_lock保护所有的进程的页表，pagefault和pagedir_destory中直接拿这把大锁，其他凡是要修改页表的操作也要拿这把锁

此外，加锁的顺序必须要是一致的，在pagefault中pagedir_lock在vm_area_list的锁和vma的锁之外，那么其他的情况下也要保持这个顺序，否则会死锁。

## 感悟

如果锁的范围太小了，需要在函数每一次提前返回时都释放锁，非常容易漏写；如果发现有内存泄漏或者重复拿锁，那么很有可能是你在某一个函数中提前返回时忘了释放资源

不能随便使用全局变量！看到全局变量要警惕！！！

不能随便使用全局变量！看到全局变量要警惕！！！

不能随便使用全局变量！看到全局变量要警惕！！！

**多写ASSERT！**

**多写ASSERT！**

**多写ASSERT！**

**多写ASSERT！**

**多写ASSERT！**

**多写ASSERT！**

每写一段代码，最需要做的事情就是绞尽脑汁地想ASSERT，越多越好！

### 并发的弦要时刻绷紧

访问全局变量要记得加锁；函数中的执行流被打断，比如提前return，要记得释放锁

# lab 3b

实现栈增长：load用户进程的时候，给栈也创建一个vm_area_struct，范围提前确定好，比如8MB。还是使用lazy allocation，当发生pagefualt时，fault_addr只有在栈区内，并且和esp的距离在一个page内才能增长新的栈。如果fault_addr没问题就是install一个新的page

实现MMAP：MMAP的实现和按需分页的实现一模一样，唯一添加的代码就是在驱逐脏页的时候要写回文件，而不是交换分区

## 按需分页和mmap总结

在每个PCB中维护一个vm_area_struct链表。每个vm_area_struct描述了该进程地址空间中的一个段的信息，包括这个section对应的文件描述符、在文件中的范围、大小、有多少内容需要从文件中读取，有多少内容需要由内核填0，以及加载到内存中的虚拟内存的范围

exec中load可执行文件时，只需要为每一个加载入内存的可执行文件的段创建一个vm_area结构体即可，不需要将数据实际加载入内存。

在内核启动时将空闲的物理内存分成两半，一半用来分配给内核，另一半用来分配给用户空间称为user pool，user pool中的物理页就是frame；然后在内核中还要维护一个数组，数组中的元素是frame结构体，这个数组中的每个frame结构体和user pool里真正的frame一一对应，通过frame结构体在数组中的偏移量可以找到对应的frame的位置；每个frame结构体中维护了该frame的引用计数，映射到的虚拟地址和页表

最后内核中还要维护一个bitmap swap table来追踪交换分区的使用情况，如果数据被驱逐到交换分区，就将它在交换分区中的位置记录在对应PTE的前20位中，并且在PTE中新增一个swap位，再将swap table中的对应bit翻转。

所以当发生pagefault时：

- 通过查看PTE的swap位，如果检测到PTE对应的数据在交换分区上，那么就从user pool申请一个frame，将数据从交换分区上读到内存中，映射到发生pagefault的位置
- 如果PTE不在交换分区上，那么根据pagefault的地址在vma链表中遍历找到vm_area，再根据fault_addr在该段中的偏移量从vm_area记录的文件中的对应偏移量的位置加载数据。

在pagefault的过程中，如果user pool空间不足，申请frame失败了，就使用页面替换算法遍历frame table，需要从中选出frame驱逐出去。使用的方式是使用clock算法，在frame_table中维护一个指针，表明上一次遍历到数组的哪个位置了；遍历所有frame，通过frame中的页表和虚拟地址找到对应的PTE，访问PTE的acc位和脏位，记为（0,0）。CPU会将acc位和脏位置为1

1. 第一轮查找（0,0），也就是既没有被访问过也没有被修改过的
2. 如果一轮找完了，就把clock指针再找一圈，这一次找(0,1)，并且将所有扫描过的访问位变为0
3. 如果还没找到，就重复第一轮。最多四轮一定能找到可替换的page

找到替换的frame后，如果是脏的，就写回交换分区，否则直接丢弃。如果数据被驱逐到交换分区，就将它在交换分区中的位置记录在对应PTE的前20位中

# Lab 4

在pintos中有多种块设备，常用的块设备如下所示。

<img src="pintos学习笔记.assets/image-20231219160133030.png" alt="image-20231219160133030" style="zoom:50%;" />

每个块设备在内核中有对应的结构体表示，其中包含了这个块的size（以sector为单位，一个sector是读写磁盘的基本单位，通常是512B）和名字等。其中还封装了可以对这个块设备进行读写操作的驱动函数的函数指针。

<img src="pintos学习笔记.assets/image-20231219160038615.png" alt="image-20231219160038615" style="zoom:50%;" />

inode就存放在文件系统块设备中。一个inode独占一个磁盘sector（并且还要求inode_disk结构体刚好是一个sector大小）。inode和数据块分离存储，在inode结构体中有一个start指针，指向数据块的sector号。但是数据块是连续存储的

- 这会导致容易出现外部碎片
- 并且只有在数据块后面紧跟着空闲空间时才能进行文件增长，但是索引inode可以实现无论何时都可以进行文件增长

内存中的inode中有磁盘inode结构体所在sector的位置，可以方便读取。

![image-20231219154738220](pintos学习笔记.assets/image-20231219154738220.png)

内核中在全局维护一个链表记录打开的inode

<img src="pintos学习笔记.assets/image-20231219155357768.png" alt="image-20231219155357768" style="zoom:50%;" />

在内核中维护一个名为free_map的bitmap，记录文件系统块设备（filesys block device）中的sector的使用情况。

内核初始化的时候，会按照文件系统块设备的大小初始化free_map（并且将free_map的前两位设为false，因为要留给bitmap的inode和根目录inode使用），然后对文件系统进行格式化：

- 在第一个sector创建一个inode作为free_map的inode，再在第二个块创建一个inode作为根目录的inode，

- 注意，这里有一个很奇特的做法，将bitmap也作为一个文件，在磁盘上为它创造inode和数据块，数据块用来存放bitmap的数据，我们就可以直接调用文件的接口对bitmap进行更新了

  <img src="pintos学习笔记.assets/image-20231219194235113.png" alt="image-20231219194235113" style="zoom: 50%;" />

  <img src="pintos学习笔记.assets/image-20231219195704376.png" alt="image-20231219195704376" style="zoom:67%;" />

  <img src="pintos学习笔记.assets/image-20231219195720043.png" alt="image-20231219195720043" style="zoom:67%;" />

- 使用inode_create创建inode：给定一个sector号和文件长度，此函数会在指定sector处创建一个inode，然后在free_map中找到空闲的连续slot作为inode的数据块，并数据块的块号作为inode的start

  - 所以create一个新文件时要先在free_map中为该文件的inode分配一个sector，然后再调用inode_create

    <img src="pintos学习笔记.assets/image-20231219230051401.png" alt="image-20231219230051401" style="zoom: 50%;" />

目录和文件本质上没有区别，唯一的区别就是目录的inode的数据块中存放的是文件的dir_entry，指明了文件的inode块号和名字

<img src="pintos学习笔记.assets/image-20231219230541021.png" alt="image-20231219230541021" style="zoom:67%;" />

pintos中目录层的实现就是将指向目录inode的指针包装了一层，本质上对目录中的查找（lookup）、添加文件（dir_add）、删除文件（dir_remove）都是通过对目录inode的数据块进行操作的

- 查找是遍历目录数据块中的dir_entry，对比目录项的name和查找的name

- 添加文件则直接在目录inode的数据块中添加一条dir_entry即可（不负责创建inode，inode需要提前创建）

- 删除文件则是根据name在目录inode的数据块中找到dir_entry，然后删除entry（将in_use置为false），并且删除entry指向的inode和inode数据块（只需要将free_map对应slot置为true即可，不需要对磁盘进行任何操作）

  - **注意！由于删除文件仅仅释放free_map的slot，所以创建文件时除了要分配free_map的slot，还要     把磁盘上的sector变成0，因为sector还保留着原来的数据**！

    <img src="pintos学习笔记.assets/image-20231224230834297.png" alt="image-20231224230834297" style="zoom: 67%;" />

- 注意，这些对dir_entry的修改只是在内存中进行的，修改完后还需要写回磁盘

  <img src="pintos学习笔记.assets/image-20231219232628872.png" alt="image-20231219232628872" style="zoom:67%;" />

**对inode读写本质上就是先根据inode的数据块号确定读写的位置在块设备上的哪一块，然后调用块设备的驱动读取数据入内存即可**

**所以从文件系统角度来看，磁盘很直观，就是读写sector**

**所以从文件系统角度来看，磁盘很直观，就是读写sector**

~~**但是，我们不能直接把内存中的数据块写到磁盘中的数据块上，需要先把磁盘上的数据块读到内存中，再把内存中的数据块的内容复制给内存中的数据块副本，再将数据块副本写回磁盘**。我们不能把一个块写到磁盘中的任意一个位置。~~

**unix文件系统中通常是：可以有多个file结构体指向同一个inode，也可以有多个文件描述符指向同一个file结构体**；pintos中也是如此，只不过没有全局的打开文件表，但是本质上没有区别，进程的打开文件表中维护的还是file结构体的指针。

inode_close和inode_remove的区别：

- inode_close是在关闭文件时减少对inode的引用计数，如果引用计数变为0了，那么需要释放内存中的inode

- inode_remove是删除磁盘上的inode和inode的数据块；实际inode_remove是将inode标记为删除，然后当inode的引用计数变为0的时候，再真正地删除磁盘上的inode数据

## 实现

首先实现buffer，还要实现pre-fetch和定时刷盘，pre-fetch只有在异步时才有意义，所以每次bread将一个sector读取到buf里面来的时候，都开一个后台线程把下一个sector也读取到buf中来。buf的具体实现和xv6一样，维护一个free_list实现LRU替换算法。

缓冲池的并发控制：整个缓冲池（buffer cache）有一个大锁（这里是自旋锁，或者单核OS直接关中断）来保护缓冲池的metadata和所有frame（即buf）的metadata，每个buf中有一个小锁（这里是sleeplock）来保护对buf数据的读写

bget从缓冲池中获取buf的时候要对buf的引用计数递增，此时buf不能被替换出内存，这个操作相当于pin；使用完buf之后调用brelease，将buf的引用计数递减，并且维护LRU替换算法，**brelease相当于unpin**。

- 所以在设计上，缓冲池通常没有一个显式的方法pin，而是由get或者fetch方法包含了pin的作用；有时会有方法unpin，也可以像xv6一样用brelease函数替代unpin。

缓冲池只负责保护共享的数据结构的并发访问，即buf（或者frame）的data部分。如果用户调用bread将buf的数据复制到另一个缓冲区，然后用户调用brealease释放锁，这个缓冲区的数据就不归缓冲池管了

最重要的一点是，**设计系统时，层次应该分明，不能越过某一层直接访问另一层**：既然有了缓冲池，那么所有对磁盘的操作都要经过缓冲池，不允许有人直接调用block_read或者block_write了，避免内存中出现同一个sector的多个副本；

- 在pintos中没有实现log，并且还要求延迟落盘，那么落盘的任务应该由缓冲池负责，将修改写入buf后就相当于落盘了，上层不应该调用block_write！如下图，将disk_inode写回磁盘应该改成先bread，再直接修改buf即可。

  <img src="pintos学习笔记.assets/image-20231227151327574.png" alt="image-20231227151327574" style="zoom:67%;" />

同理，有了inode cache，所有对inode的获取都要先经过inode cache，**如果cache中已经存在某一个inode了，则直接返回这个inode即可，不允许再拷贝一份该inode的副本**。

- 在pintos中inode cache就是open_inodes链表。我们需要对这个链表和其中的每个inode都添加并发保护，和bufCache的保护方式一样，inode cache使用自旋锁保护，inode使用睡眠锁保护

  <img src="pintos学习笔记.assets/image-20231227151808471.png" alt="image-20231227151808471" style="zoom:67%;" />

- inode的并发保护与buf有一点不一样：bget获取了buf之后对buf的引用计数加一，同时还拿了buf的锁；直到上层将buf使用结束后才将引用计数减一，并且释放锁；也就是说**引用buf和拿锁是同步的**，这种加锁方式对buf来说是可行的，因为没有人会长期拿着某个buf的引用，都是读写完buf的数据后就释放引用了；但是对于inode来说不能这么做，因为在内核中有些数据结构会长期持有inode的引用，但是不对其进行任何读写（比如file、dir）。如果引用inode的同时还要拿锁，那么系统的并发程度将会非常低。

- 所以inode的并发保护采用的方法是：调用inode_open获得inode的引用的时候，如果inode cache中存在这个inode，那么直接将引用计数增加，不拿inode的锁；如果inode cache中不存在，那么就在inode cache中分配一个inode结构体，将结构体的metadata初始化（如下图），但是不从磁盘中读取inode的数据，也不拿inode的锁，直接将inode的引用返回即可。

  <img src="pintos学习笔记.assets/image-20231227164159750.png" alt="image-20231227164159750" style="zoom:50%;" />

  只有当上层真正需要访问inode的数据的时候，才需要拿inode的锁，然后根据inode是否valid，再从磁盘中读取真正的信息到内存的inode中。访问结束后，就释放inode的锁，同样也不能减少inode的引用。

  这种方法**将inode的引用与拿锁分离，并且将inode的数据读取也推迟了，不在open的时候读取，而是在真正需要的时候读取，相当于lazy allocation**。

- 使用这种方式，也就意味着inode的锁只能用来保护inode的数据，即下图的部分；而inode结构体的其他的数据要靠inode cache的自旋锁来保护

  <img src="pintos学习笔记.assets/image-20231227164747082.png" alt="image-20231227164747082" style="zoom:50%;" />

缓存应该起到隔绝的作用，用户的任何操作都必须经过缓存，由缓存代理；如果有用户绕过缓存那么就会出现缓存一致性的问题。因此对缓存中的数据更新可以不用及时写回上一层缓存，因为所有的用户都要通过当前缓存获取数据，如果当前缓存中有数据，那么就不会去上一层缓存获取没更新的数据。但是，如果当前缓存中的数据被驱逐了，那么就必须要更新上一层的缓存的数据。

这个设计思路在bufCache和inode cache都是一样的：对inode cache中inode的更新不需要及时写回buf cache，只有当inode被驱逐出inode cache时才需要写回buf。

**reopen的作用**：

- pintos中file和dir都有reopen这个函数，本质上是因为不能直接给一个file或dir指针赋值，这会导致多个指针指向同一个对象，如果其中一个释放了对象，那么其他的就变成了野指针。pintos中有些地方长期维持着file指针和dir指针，比如pcb的打开文件表和cwd，给他们赋值的时候一定要使用reopen！
- 除此之外，在reopen dir和file时，对他们内部的inode也要进行reopen，这个reopen的作用是将inode计数增加，否则关闭一个dir或file时会导致其他的dir和file的inode变成野指针

然后实现文件扩展，就是在inode中加几个间接指针，然后修改给定文件offset计算数据块块号的函数实现；再修改write_inode_at函数，取消每次写入时inode_length的限制

![image-20240104143248370](pintos学习笔记.assets/image-20240104143248370.png)

当删除一个还有其他线程引用的文件或目录时，只是将该inode的nlink数变为0，但是由于inode的open_cnt不为0，所以并没有真正的删除；只有等所有线程把对该inode的引用都释放了，最后一个释放引用的线程才真正地删除此inode的所有数据（内存和磁盘上的）。在此期间，不允许有线程对此inode进行open或者写入，当检测到inode的nlink数为0时，直接返回open失败或者写入失败。

## 文件系统总结

为了简单起见，pintos中的一个inode独占一个磁盘sector，一个磁盘sector的大小是512B，剩下未使用的空间在inode结构体中用0填充。在inode结构体中维护了inode数据的长度和一个数组，里面记录了这个inode的数据块的块号

在内核中维护了一个bitmap，记录磁盘中sector的使用情况。内核初始化的时候，会按照文件系统块设备的大小初始化free_map（并且将free_map的前两位设为false，因为要留给bitmap的inode和根目录inode使用），然后对文件系统进行格式化：

- 在第一个sector创建一个inode作为free_map的inode，再在第二个块创建一个inode作为根目录的inode，

- 这里将bitmap也作为一个文件，在磁盘上为它创造inode和数据块，数据块用来存放bitmap的数据，我们就可以直接调用文件的接口对bitmap进行更新了


然后在内核中维护一个数组作为buffer cache，每个buffer中除了有sector的数据之外，还维护了这个buffer是否脏了，和引用计数。

创建inode的时候只在inode块中记录长度，并在指定的磁盘块处写入inode，不需要给inode的数据块分配磁盘空间，当读取时返回0，当写入的时候再分配数据块，类似于懒分配

在内核中还维护了一个inode链表作为inode的cache，当要open指定sector处的inode时，先查看inode cache中是否有这个块号的inode，如果有就直接使用cache中的inode，否则就malloc创建一个inode；这个时候并没有真正从磁盘中读取inode的数据，而是创建了一个空壳，因为在内核中有的数据结构会长期持有对inode的引用，但是却不访问它的数据，比如打开文件结构体，所以可以等到真正需要读取inode的数据的时候再从磁盘读入。

- inode cache的作用是避免同一个inode在内存中有多个副本，导致数据不一致

每次通过inode读取它的数据中的指定偏移量的数据时，先通过偏移量算出在inode的第几个数据块，然后在inode的索引数组中找到这个块的块号，然后通过buffer cache读取这个块的数据，如果数据不在buffer中，就从磁盘中读取。将buffer的引用计数加一，表示这个buffer的数据正在被使用，不能被驱逐出去。

通过inode写入数据的时候也是一样的，找到写入偏移量对应的块号之后就写入到对应的buffer中。没写入的地方可以不用分配空间。

每个进程维护一个cwd，表示当前的进程所在的目录路径，如果访问的文件是相对路径，就从cwd开始搜索



mkdir：根据路径一级一级地向下查找，遍历每一级目录inode的数据块中的目录项，对比目录项的name和查找的名字，找到了最后一级目录之后在目录的inode的数据块中添加一条dir_entry即可

**unix文件系统中通常是：可以有多个file结构体指向同一个inode，也可以有多个文件描述符指向同一个file结构体**；pintos中也是如此，只不过没有全局的打开文件表，但是本质上没有区别，进程的打开文件表中维护的还是file结构体的指针。

inode_close和inode_remove的区别：

- inode_close是在关闭文件时减少对inode的引用计数，如果引用计数变为0了，那么需要释放内存中的inode

- inode_remove是删除磁盘上的inode和inode的数据块；实际inode_remove是将inode标记为删除，然后当inode的引用计数变为0的时候，再真正地删除磁盘上的inode数据

## bug0

我使用的inode缓存，每次更新时没有写回buf，只有在inode被驱逐出inode缓存时才写回buf。但是后台的刷盘线程每次只从buf中刷脏，这就导致了inode的数据没有持久化

解决方法是：

- 要么每次修改inode之后都写回buf
- 要么在刷脏的时候遍历一次inodeCache，把所有inode都刷回buf

## bug1

在下面的代码中，1.首先创建了一个文件，然后open这个文件是正常的；但是执行到第三个代码块，重新open一次这个文件就无法找到这个文件；如果在中间的while中加上第二个代码块，每次循环都open一次，那么2和3代码块都能成功open。

所以，很明显，这个bug是因为在目录中创建了文件之后，目录的数据块被驱逐出内存，并且驱逐时没有写回导致的

<img src="pintos学习笔记.assets/image-20231228142202726.png" alt="image-20231228142202726" style="zoom:67%;" />

沿着这个思路，我们在驱逐（或者说复用）buf时，打印一下被复用的buf之前的sector号和脏位

经排查发现，是在brealease中释放buf时，用dirty覆盖了之前的dirty，导致驱逐时没有写回。

<img src="pintos学习笔记.assets/image-20231228154823267.png" alt="image-20231228154823267" style="zoom: 50%;" />

改为：

<img src="pintos学习笔记.assets/image-20231228155223692.png" alt="image-20231228155223692" style="zoom:67%;" />

## bug2

load文件时发现文件第一块的数据不对，直接在extract的时候输出第一块的内容，如下图：发现刚向文件系统中写完第一块的时候，红框的代码输出的数据是正确的；但是向文件系统中继续写后面的块后，第一块的数据就错了

<img src="pintos学习笔记.assets/image-20231228175931767.png" alt="image-20231228175931767" style="zoom:67%;" />

经排查发现是写入数据时根据pos得到sector_offset出错了，导致写入后面的块覆盖了前面的块

修改后：

![image-20231228180159523](pintos学习笔记.assets/image-20231228180159523.png)

## bug3

发生死锁，经排查发现是一个线程获取inode的lock时死锁的，并且还不是自己获取的；那么说明有另外一个线程先拿了inode的锁，再拿了某一个资源的锁后陷入了阻塞；然后当前线程先拿了该资源的锁，在拿inode的锁时陷入了阻塞。

解决死锁的办法是：在拿锁的时候打印一下哪个进程拿了哪个锁；在释放锁的时候打印一下是哪个进程释放了哪个锁。就能发现是在哪里发生了死锁

最后发现是拿了某个锁后没有释放

<img src="pintos学习笔记.assets/image-20231228231022931.png" alt="image-20231228231022931" style="zoom:67%;" />

通常这种情况是由于函数提前返回，锁忘了释放了

## bug4

疯狂创建新的目录，但是内存不够了；很显然是内存泄漏，inode和dir忘了释放了。解决方法就是在inode_open和inode_reopen中打印inode的地址和open_cnt，在inode_close也打印。一边单步调试一边查看日志，找出open了但是没有释放的地方

![image-20240104211254746](pintos学习笔记.assets/image-20240104211254746.png)

## bug5

测试执行到一半直接`exit(-1)`了，单步调试时也会莫名其妙跳转到中断，难以排查。**既然无法顺着执行流找到bug，那么就扼住必经之路，也就是在exit函数上打断点，然后执行到此处后再用backtrace查看调用栈**

最后发现是没有对inode初始化，inode_close对一个野指针进行操作

<img src="pintos学习笔记.assets/image-20240104211947637.png" alt="image-20240104211947637" style="zoom:67%;" />

四个lab所有测试全部通过：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20240105134858043.png" alt="image-20240105134858043" style="zoom:67%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20240105132601484.png" alt="image-20240105132601484" style="zoom:67%;" />

总代码量：

```
git diff --stat master..lab4
52 files changed, 2822 insertions(+), 407 deletions(-)
```

# multithreading

当线程A向线程B传递一个指向其局部变量的指针作为参数，然后线程A的函数结束，导致局部变量被销毁，这时线程B持有的指针将变成野指针，访问它将导致未定义行为。

这种情况不应该由我们（pthread的实现者）来处理，处理线程之间的参数传递和生命周期的管理是用户的事情，用户应该保证传入另一个线程的参数不能被销毁。

thread_create创建新线程时不需要初始化mm_struct，而是与父线程共用一个mm_struct

thread_create函数中不需要创建页表，这里仅仅是创建内核线程，所有刚创建出来的内核线程临时共享init_page_dir，如果需要加载用户程序（比如执行start_process），再将init_page_dir复制一份，创建一个新的页表，然后把可执行文件加载到新的页表中，把新的页表激活（如果使用按需分页，都不需要将用户程序映射到页表上，只需要分配vm_area_struct即可）

![image-20240110210847637](pintos学习笔记.assets/image-20240110210847637.png)

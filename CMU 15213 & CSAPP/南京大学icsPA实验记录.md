# 南京大学icsPA实验记录

![image-20210807174921510](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210807174921510.png)



PA的目的是要实现NEMU, 一款经过简化的全系统模拟器。

随着时代的发展, 你已经很难在市场上看到红白机的身影了. 当你正在为此感到苦恼的时候, **红白机模拟器可以模拟出红白机的所有功能**. 有了它, 你就好像有了一个真正的红白机,

事实上, **NEMU模拟了一个硬件的世界**, 你可以在这个硬件世界中执行程序. 换句话说, 你将要在PA中编写一个用来执行其它程序的程序! 

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210807181123507.png" alt="image-20210807181123507" style="zoom:67%;" />

图中(b)展示了"在GNU/Linux中通过红白机模拟器玩超级玛丽"的情况. 在GNU/Linux看来, 运行在其上的红白机模拟器NES Emulator（NEMU）和上面提到的Hello World程序一样, 都只不过是一个用户程序而已. 神奇的是, **红白机模拟器的功能是负责模拟出一套完整的红白机硬件, 让超级玛丽可以在其上运行**（同样，图中(c)的**NEMU的功能是负责模拟出一套计算机硬件, 让程序可以在其上运行**）. 事实上, 对于超级玛丽来说, 它并不能区分自己是运行在真实的红白机硬件之上, 还是运行在模拟出来的红白机硬件之上, 这正是"模拟"的障眼法.

**NEMU具备的是物理计算机系统的功能, 是用来执行程序的. 因此我们说, NEMU是一个用来执行其它程序的程序.**

## PA1

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210808122906742.png" alt="image-20210808122906742"  />

程序的入口main.c文件：

```c
void init_monitor(int, char *[]);
void engine_start();
int is_exit_status_bad();

int main(int argc, char *argv[]) {
  /* 初始化monitor. */
  init_monitor(argc, argv);

  /* Start engine. */
  engine_start();

  return is_exit_status_bad();
}
```

### monitor初始化

`NEMU`是一个用来执行客户程序的程序, 但客户程序一开始并不存在于客户计算机中. **我们需要将客户程序读入到客户计算机中, 这件事是monitor来负责的**. 于是`NEMU`在开始运行的时候, 首先会调用`init_monitor()`函数(在`nemu/src/monitor/monitor.c`中定义) 来进行一些和monitor相关的初始化工作.

```c
void init_monitor(int argc, char *argv[]) {
  /* Perform some global initialization. */

  /* Parse arguments. */
  parse_args(argc, argv);

  /* Open the log file. */
  init_log(log_file);

  /* Fill the memory with garbage content. */
  init_mem();

  /* Perform ISA dependent initialization. */
  init_isa();

  /* Load the image to memory. This will overwrite the built-in image. */
  long img_size = load_img();

  /* Compile the regular expressions. */
  init_regex();

  /* Initialize the watchpoint pool. */
  init_wp_pool();

  /* Initialize differential testing. */
  init_difftest(diff_so_file, img_size, difftest_port);

  /* Display welcome message. */
  welcome();
}
```

接下来在`init_monitor`中会调用`init_isa()`函数(在`nemu/src/isa/$ISA/init.c`中定义), 来进行一些ISA相关的初始化工作.

```c
void init_isa() {
  /* Load built-in image. */
  memcpy(guest_to_host(IMAGE_START), img, sizeof(img));

  /* Initialize this virtual computer system. */
  restart();
}
```

`init_isa()`的工作：

- 第一项工作就是将一个**内置的客户程序**读入到**内存**中

  - 程序是由指令构成的, 而不同ISA的指令也各不相同。因此, 我们把内置客户程序放在`nemu/src/isa/$ISA/init.c`中.
  - 可以把内存看作一段连续的存储空间, 而内存又是字节编址的(即一个内存位置存放一个字节的数据), 在C语言中我们就很自然地使用一个`uint8_t`类型的数组来对内存进行模拟. NEMU默认为客户计算机提供128MB的物理内存(见`nemu/src/memory/paddr.c`中定义的`pmem`),
  - 为了让客户计算机的CPU可以执行客户程序, 因此我们需要一种方式让客户计算机的CPU知道客户程序的位置. 我们让`monitor`直接把客户程序读入到一个固定的内存位置`IMAGE_START`(也就是`0x100000`).

- `init_isa()`的第二项任务是**初始化寄存器**. 这是通过`restart()`函数来实现的. 

  在C语言中我们就很自然地使用相应的**结构体**来描述CPU的寄存器结构. 不同ISA的寄存器结构也各不相同, 为此我们把寄存器结构体`CPU_state`的定义放在`nemu/include/isa/$ISA.h`中

  ```c
  typedef struct {
    struct {
      rtlreg_t _32;
    } gpr[32];
  
    vaddr_t pc;
  } riscv32_CPU_state;
  ```

   并在`nemu/src/moinitor/cpu-exec.c`中定义一个全局变量`cpu`

  ```c
  CPU_state cpu = {};
  ```

  **初始化寄存器的一个重要工作就是设置`cpu.pc`的初值**, 我们需要将它设置成刚才加载客户程序的内存位置, 这样就可以让CPU从我们约定的内存位置开始执行客户程序了.对于mips32和riscv32, 它们的0号寄存器总是存放`0`, 因此我们也需要对其进行初始化.

  ```c
  static void restart() {
    /* Set the initial program counter. */
    cpu.pc = PMEM_BASE + IMAGE_START;
  
    /* The zero register is always 0. */
    cpu.gpr[0]._32 = 0;
  }
  ```

`init_isa()`执行完后，NEMU返回到`init_monitor()`函数中, 继续调用`load_img()`函数。这个函数会将一个有意义的客户程序从[镜像文件](https://en.wikipedia.org/wiki/Disk_image)读入到内存, 覆盖刚才的内置客户程序. 这个镜像文件是运行NEMU的一个可选参数, 在运行NEMU的命令中指定. 如果运行NEMU的时候没有给出这个参数, NEMU将会运行内置客户程序.

最后`monitor`会调用`welcome()`函数输出欢迎信息. 现在可以在`nemu/`目录下编译并运行NEMU了:

```
make ISA=$ISA run
```

### start engine

Monitor的初始化工作结束后, `main()`函数会继续调用`engine_start()`函数 (在`nemu/src/engine/interpreter/init.c`中定义)

```c
void ui_mainloop();
void init_device();

void engine_start() {
  /* 对设备进行初始化. */
  init_device();

  /* Receive commands from user. */
  ui_mainloop();
}
```

代码还会对设备进行初始化(目前无需关心), 然后进入用户界面主循环`ui_mainloop()`(在`nemu/src/monitor/debug/ui.c`中定义),并输出NEMU的命令提示符

```
(nemu)
```

用户界面主循环是monitor的核心功能, 我们可以在命令提示符中输入命令, 对客户计算机的运行状态进行监控和调试.


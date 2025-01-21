# CSAPP实验记录（一）：环境配置&datalab

## 1、环境配置

1. [下载Ubuntu虚拟机](https://zhuanlan.zhihu.com/p/141033713)。我之前用的是Ubuntu18.04，非常坑，强烈建议换成Ubuntu20.04

2. [windows和Ubuntu共享文件](https://blog.csdn.net/qq_37040516/article/details/87688408)

3. 将[实验网站](http://csapp.cs.cmu.edu/3e/labs.html)上下载的实验Handout放入windows下的共享文件夹

4. 在Ubuntu中打开共享文件夹

   1. <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210708221138406.png" alt="image-20210708221138406" style="zoom: 50%;" />

   2. 继续打开/mnt/hgfs/<你的共享文件夹名字>.

      <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210708222953050.png" alt="image-20210708222953050" style="zoom: 50%;" />

   3. 右键选择该文件，复制到另一个自己创建的目录下

      <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210709121434862.png" alt="image-20210709121434862" style="zoom:50%;" />

   4. 进入该目录，打开终端，输入命令：

      ```bash
      tar xvf datalab-handout.tar
      ```

      这会把许多文件解压到目录中。你唯一需要修改和提交的文件是 bits.c。

   5. Ubuntu缺省情况下，并没有提供C/C++的编译环境，因此还需要手动安装。但是如果单独安装gcc以及g++比较麻烦，幸运的是，Ubuntu提供了一个build-essential软件包。直接安装这个包即可，build-essential软件包列表内包含libc6-dev、libc-dev、gcc、g++、make、dpkg等C/C++编译必须的工具。
   
      安装gcc：
   
      ```bash
      sudo apt-get install  build-essential
      ```
   
      完善编译环境：
   
      ```bash
      sudo apt-get install gcc-multilib
      ```

## 2、解题记录

1. 首先阅读bits.c的注释，然后在代码处修改它

2. 使用 dlc 编译器（**./dlc**）自动检查 **bits.c** 版本是否符合编码准则。使用 **-e** 参数运行 dlc：

   ```bash
   unix> ./dlc -e bits.c
   ```

   可让 dlc 打印每个函数使用的运算符数目。

3. 有了合法解答，就可以使用 **./btest** 程序测试它的正确性。

   `btest.c`是测试代码的工具，编译它（使用终端`make btest`）我们可以获得可执行文件，执行它会对你的代码进行测试，注意每次修改完代码之后都需要重新编译。

   `btest.c`使用`make`编译`btest`文件，要编译并运行 `btest` 程序，请键入：

   ```bash
   unix> make btest
   unix> ./btest [optional cmd line args]
   ```

   每次更改 `bits.c` 程序时，都需要重新编译 `btest`。

   测试所有功能的正确性并打印出错误消息：

   ```bash
   unix> ./btest
   ```

   以紧凑的形式测试所有函数，不含错误消息：

   ```bash
   unix> ./btest -g
   ```

   测试函数 foo 的正确性：

   ```bash
   unix> ./btest -f foo
   ```

   使用特定参数测试函数 foo 的正确性：

   ```bash
   unix> ./btest -f foo -1 27 -2 0xf
   ```

   btest 不会检查你的代码是否符合代码准则，需使用 dlc。

### 1、bitXor

用与门&和非门~实现异或操作。

```c
int bitXor(int x, int y) {
  return (~(~x&~y))&(~(x&y));
}
```

![image-20210711154703169](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210711154703169.png)

### 2、tmin

求32位补码的最小值

```c
int tmin(void) {
  return 1<<31;
}
```

![image-20210711155715054](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210711155715054.png)

### 3、isTmax

判断一个数是否是最大的补码，只能使用! ~ & ^ | +，且整数常量的范围在0到255之间

我们可以发现当x是最大的补码时，加一得到最小的补码，二者再相加得到`0XFFFFFFFF`，求反等于0。除此之外还有一个特殊情况，当这个数是`0XFFFFFFFF`时，加一得到0，相加也得到`0XFFFFFFFF`，所以需要排除这个情况。！的作用是将非0的数转化为1 。

```c
int isTmax(int x) {
  return (!~(x+x+1))&(!!~x);
}
```

![image-20210711213518473](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210711213518473.png)

### 4、allOddBits

如果一个数的奇数位全都为1则返回1，否则返回0。

由于规定只能使用0到255之间的整数，所以`0XAAAAAAAA`只能通过移位来产生。^表示异或，来判断两个数是否每一位都相同，相同则结果为0，不同则为1 。

```c
int allOddBits(int x) {
  int mask=(0xAA<<8)+0xAA;
  mask=(mask<<16)+mask;
  return !((x&mask)^mask);
}
```

![image-20210711221401112](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210711221401112.png)

### 5、negate

求相反数。

逐位取反再加1即可。

```c
int negate(int x) {
  return ~x+1;
}
```

### 6、isAsciiDigit

如果x在`0X30`到`0X39`之间则返回1，否则返回0 。

x在`0X30`到`0X39`之间的条件为`x-0X3A<0&&x-0X2F>0`，依据最高位来判断结果的正负，将结果右移31位即可。

```c
int isAsciiDigit(int x) {
  return !!((x+~0X3A+1)>>31)&!((x+~0X30+1)>>31);
}
```

![image-20210712121328114](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210712121328114.png)

### 7、conditional

和 `x?y:z`相同，x是true时返回y，false返回z 。

此题应该满足这样一个式子`(x1&y)|(x2&z)`，而x1和x2应该满足如下关系：

```
x			x1			x2
0	    0X00000000   0XFFFFFFFF
其他    0XFFFFFFFF   0X00000000
```

由`0X00000001`变成`0XFFFFFFFF`只需求反加1即可。

```c
int conditional(int x, int y, int z) {
  x=!!x;
  x=~x+1;
  return (x&y)|(~x&z);
}
```

### 8、isLessOrEqual

实现一个小于等于符号。

判断两个数的大小分两种情况

- 符号位相同，通过做差，然后在通过符号位判断。
- 符号位不相同，可以直接判断`x`的符号位。

```c
int isLessOrEqual(int x, int y) {
  int SignX=x>>31;
  int SignY=y>>31;
  int isEqual=SignX^SignY;
  return (isEqual&!!SignX)|(!isEqual&!((y+~x+1)>>31));
}
```

![image-20210712133727699](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210712133727699.png)

### 9、logicalNeg

实现逻辑非。

```c
int logicalNeg(int x) {
  return ((x|(~x+1))>>31)+1;
}
```

### 10、howManyBits

### 11、floatScale2

> Return bit-level equivalent of expression 2*f for floating point argument f. Both the argument and result are passed as unsigned int's, but they are to be interpreted as the bit-level representation of single-precision floating point values. When argument is NaN, return argument.

此题分为三种情况：

- 阶码全为1，可能是INF或者NaN。直接返回参数

- 阶码全为0，非规格化数。整体左移一位，再修改符号位（并不会越界，如下所示：

  ```
  0 0000 1111 ==> 非规格化数：0.1111*2^(1-127)
  		整体左移一位后，变成了规格化数
  0 0001 1110 ==> 规格化数：  1.1110*2^(1-127)  
  ```

  虽然从非规格化数变成了规格化数，但是值正好是之前的两倍。

- 阶码既非全0也非全1，规格化数。将阶码加一即可。

```c
unsigned floatScale2(unsigned uf) {
  int exp=(uf>>23)&0XFF;
  if(exp==0XFF)return uf;
  if(exp==0X00)return (uf<<1)|(uf&(1<<31));
    //阶码加一
  return uf+(1<<23);
}
```

![image-20210712231014954](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210712231014954.png)

### 12、floatFloat2Int

> Return bit-level equivalent of expression (int) f for floating point argument f. Argument is passed as unsigned int, but it is to be interpreted as the bit-level representation of a single-precision floating point value. Anything out of range (including NaN and infinity) should return `0x80000000u`.

将float转化为int。

此题分为：

- 指数大于31时，一定溢出，返回`0x80000000u`
- 指数等于31时，只有当尾数全为0，且符号位为负时，才不会溢出。其他的情况一定溢出，返回`0x80000000u`
- 指数小于0时，返回0 。
- 指数小于23时，尾数右移；指数大于等于23时，尾数左移。再加上符号位。

```c
int floatFloat2Int(unsigned uf) {
  int E=((uf>>23)&0XFF)-127;
    //获取尾数部分
  int frac=(uf&0X007FFFFF)|0X00800000;
  int Tmin=1<<31;
  int sign=uf&Tmin;
  int f;
  if(E>31)return 0x80000000u;
  if(E==31&&((frac^0X00800000)||(sign^Tmin)))return  0x80000000u;
  if(E<0)return 0;
  f=(E>23)?(frac<<(E-23)):(frac>>(23-E));
  return (uf&Tmin)?-f:f;
}
```

![image-20210713005421316](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210713005421316.png)

### 13、floatPower2

> Return bit-level equivalent of the expression 2.0^x (2.0 raised to the power x) for any 32-bit integer x.
>  The unsigned value that is returned should have the identical bit representation as the single-precision floating-point number 2.0^x. If the result is too small to be represented as a denorm, return 0 . If too large, return +INF.

将2的x次方用浮点数表示出来。

- 当x大于127时，超过了规格化数的最大值，返回INF
- 当x小于-149时，超过了非规格化数的最小值，无法表示，返回0。

- 当x小于等于127且大于等于-126时，用规格化数表示，除了阶码其他部分都为0
- 当X小于-126时，用非规格化数表示，此时阶码始终是-127，超过的部分对尾数除2.

```c
unsigned floatPower2(int x) {
    int E=x+127;
    if(x>127)return 0X7f800000;
    if(x<-149)return 0;
    if(x>=-126&&x<=127)return E<<23;
    return 1<<(x+149); 
}
```


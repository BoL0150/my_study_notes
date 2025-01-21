# CMU15-213学习笔记（七）Dynamic Memory Allocation

## 动态内存分配

程序员通过动态内存分配（例如 `malloc`）来让程序在运行时得到虚拟内存。动态内存分配器会管理一个虚拟内存区域，称为**堆(heap)**。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210915200550579.png" alt="image-20210915200550579" style="zoom:67%;" />

动态内存分配器将堆视为一组不同大小的块（block）的集合，每个块就是一个连续的虚拟内存片（chunk），每个块具有**两种状态：**

- **已分配：**已分配的块能为应用程序所用，且块会保持已分配状态直到被释放
- **空闲的：**空闲的块无法使用，直到它被分配

在最开始进行内存映射时，堆是与匿名文件关联起来的，所以堆是一个全0的段，即处于空闲状态，它紧跟在未初始的数据段后面，**向地址更大的方向延伸**，且**内核对每个进程都维护了`brk`变量来指向堆顶**。

动态内存分配器具有两种类型，**都要求由应用程序显示分配块**，但是由不同实体来负责释放已分配的块：

- 显示分配器（Explicit Allocator）：**要求应用程序显示释放已分配的块**。比如C中通过`malloc`来分配块，再通过`free`来显示释放已分配的块，C++中的`new`和`delete`相同。
- 隐式分配器（Implicit Allocator）：由分配器检测哪些块已不被应用程序使用，就**自动释放**这些块。这种隐式分配器称为垃圾收集器（Garbage Collector），而这种过程称为垃圾收集（Garbage Collection）。比如Java、ML和Lisp。

程序使用动态内存分配器来动态分配内存的意义在于：有些数据结构只有在程序运行时才知道大小。通过这种方式就无需通过硬编码方式来指定数组大小，而是根据需要动态分配内存

### `malloc`和`free`函数

C中提供了malloc显示分配器，程序可以通过`malloc`函数来**显式地从堆中分配块**

```c
#include <stdlib.h>
void *malloc(size_t size); // 返回一个泛型void指针，需要将它强转为int*，才能通过编译
```

该函数会返回一个指向大小至少为`size`字节的未初始化内存块的指针，且根据程序的编译时选择的字长，来确定内存地址对齐的位数，比如`-m32`表示32位模式，地址与8对齐，`-m64`表示64位模式，地址与16对齐。如果函数出现错误，则返回NULL，并设置`errno`。我们也可以使用`calloc`函数来将分配的内存块初始化为0，也可以使用`realloc`函数来改变已分配块的大小。

程序可以通过`free`函数来释放已分配的堆块

```c
#include <stdlib.h>
void free(void *ptr);
```

其中`ptr`参数要指向通过`malloc`、`calloc`或`realloc`函数获得的堆内存。

动态内存分配器可以使用`mmap`和`munmap`函数，也可以使用`sbrk`函数来向内核申请堆内存空间，**只有先申请获得堆内存空间后，才能尝试对块进行分配让应用程序使用**。

```c
#include <unistd.h>
void *sbrk(intptr_t incr); 
int brk(void *addr);
```

`brk`函数会将`brk`设置为`addr`指定的值。`sbrk`函数通过将内核的`brk`指针增加`incr`来扩展和收缩堆，**如果成功返回`brk`的旧值**，否则返回-1，并将errno设置为`ENOMEM`。

- 当`incr`小于0时，会减小`brk`来解除已分配的堆内存
- 当`incr`等于0时，会返回当前的`brk`值
- 当`incr`大于0时，会增加`brk`来分配更多的堆内存

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916111238305.png" alt="image-20210916111238305" style="zoom: 50%;" />

限制：

- 应用：
  - 程序可以用任意的顺序发送 `malloc` 和 `free` 请求
  - `free` 请求必须作用于已被分配的 block。

- 分配器
  - 不能控制已分配块的数量和大小
  - 必须立即响应 `malloc` 请求（不能缓存或者给请求重新排序）
  - 必须在未分配的内存中分配
  - 不同的块需要对齐（32 位中 8 byte，64 位中 16 byte）
  - **只能操作和修改未分配的内存**
  - **不能移动已分配的块**
    - 比如分配器不能将已分配的块放在一起以达到压缩块，创造更大的空闲块的目的

### 性能指标

**吞吐量**：

- 在单位时间内完成的请求数量。假设在 10 秒中之内进行了 5000 次 `malloc` 和 5000 次 `free` 调用，那么吞吐量是 1000 operations/second

峰值内存利用率（Peak Memory Utilization）：

- 一个系统中所有进程分配的虚拟内存的全部数量是受磁盘上的交换空间限制的，所以要尽可能最大化内存使用率。

  - 有效载荷（payload）：应用程序请求一个p字节的块，但是分配器为了对齐要求和块的格式，可能会申请比p更大的块，该已分配的块的有效载荷为p字节。
  - 聚合载荷（aggregate payload）：**当前已分配的所有有效载荷之和**。在完美的分配器中，聚合有效载荷等于所有已分配块的总大小，也就是每个块都是有效载荷。

  峰值利用率就是![[公式]](https://www.zhihu.com/equation?tex=U_k%3D\frac{max_{i\le+k}P_i}{H_k})，即~~当前的聚合载荷~~**在此之前最大**的聚合载荷除以**当前**的堆的大小。**假设我们的堆只扩张不缩小**，在理想状态下，每个块的内容都是有效载荷，所以利用率为1。

影响内存利用率的主要因素就是**内存碎片**，分为内部碎片和外部碎片两种。

- **内部碎片（Internal Fragmentation）**

  内部碎片指的是对于给定的块，因为对齐和维护堆所需的数据结构的缘故，**需要存储的数据(payload)小于分配的块的大小**，就会出现无法利用的空间

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916114140401.png" alt="image-20210916114140401" style="zoom:67%;" />

  因为内部碎片的数量取决于**之前**请求的模式，所以比较量化，可以通过已分配块的大小与其有效载荷的差来量化内部碎片。

- **外部碎片（External Fragmentation）：**当空闲的内存**合起来**够满足一个分配请求，但单独一个空闲内存不够时，就会产生外部碎片。外部碎片比较难进行量化，因为它主要取决于**未来**请求的模式，所以分配器通常试图维持**少量的大的空闲块**。

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916114643334.png" alt="image-20210916114643334" style="zoom: 50%;" />

由于地址对齐要求和分配器对块格式的选择，会对**最小块**的大小有限制，已分配的块不能比最小块要小，空闲块也不能比最小块还小，如果比最小块还小，就会变成外部碎片（因为无法分配给任何块）（所以最小块越大，内部碎片程度越高）。比如这里如果对齐要求是双字8字节的，则最小块大小为双字：第一个字用来保存头部，另一个字用来满足对齐要求。

### 实现问题

- 给定一个指针，我们如何知道需要释放多少内存（即指针所指的块的大小是多少）

  在每个块的开头使用一个字大小的区域记录这个块的大小，这个字通常称为header field或header。

  也就是说，**所有分配的块都需要一个额外的字**。

  ![image-20210916132014877](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916132014877.png)

  当用户想用malloc请求一个大小为4字的载荷，分配器需要找到大小为5的块。然后**返回指向有效载荷开头的指针p0**，而不是指向header的指针。

- 如何记录未分配的块？

  - 方法一：implicit free list

    **在堆中的每个块的前面放置一个头部，不管是被分配的还是空闲的**。我们可以使用header记录的大小来访问堆。

    我们称这种方法为implicit free list，因为没有真正的空闲块列表。通过遍历堆中的所有块，我们可以找到堆中的所有空闲块，只需要忽略已分配的块就可以了。

    <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916132730898.png" alt="image-20210916132730898" style="zoom:67%;" />

  - 方法二：explicit free list

    用块中的一个字指向下一个空闲块。此链表中只有空闲块，寻找空闲块只需要遍历所有的空闲块即可，不需要遍历已分配的块。

    <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916133258065.png" alt="image-20210916133258065" style="zoom:67%;" />

  - 方法三：Segregated free list

    有多个空闲链表，每个空闲链表包含特定大小或特定大小范围的块，对于不同大小的块有不同的链表。

  

### implicit free list

每个块都需要记录大小和分配状态

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916135209948.png" alt="image-20210916135209948" style="zoom:50%;" />

- **头部：**大小为一个字（一个字为4字节），可以用来保存块大小，如果我们添加一个双字的对齐要求，则**块大小就总是8字节的倍数**，则头部中表示块大小的低3位就总是0，我们可以拿这3位来表示该块是否被分配。
- **有效载荷：**应用通过`malloc`请求的有效载荷。**有效载荷的起始地址需要满足8字节的对齐要求**。
- **填充：**可选的，分配器可用来处理外部碎片，或满足对齐要求。

malloc返回的指针指向有效载荷的开始处，而不是指向头部。

我们通过这种数据结构来组织**堆内存**，通过块头部的块大小来将堆中的所有块链接起来。分配器可以遍历所有块，然后通过块头部的字段来判断该块是否空闲的，来**间接遍历整个空闲块集合**。我们可以通过一个大小为0的已分配块来作为**终止头部（Terminating Header）**，来表示结束块。

堆的头部也有一个未使用块，目的是让第一块的有效载荷地址为8字节，满足对齐要求。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916140819212.png" alt="image-20210916140819212" style="zoom: 50%;" />

**注意：**先将有效载荷加上块头部大小，然后再满足对齐要求，得出来的就是块的大小。

#### 找到空闲块

当应用请求一个k字节的空闲块时，分配器会搜索空闲链表，并根据不同的**放置策略（Placement Policy）**来确定使用的空闲块：

- **首次适配（First Fit）：**分配器从头开始搜索空闲链表，选择第一个块大小大于k的空闲块。

  - **优点：**趋向于将大的空闲块保留在空闲链表后面。
  - **缺点：**空闲链表开始部分会包含很多碎片

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916142412162.png" alt="image-20210916142412162" style="zoom:67%;" />

- **下一次适配（Next Fit）：**分配器从上一次查询结束的地方开始进行搜索，选择第一个块大小大于k的空闲块。

- - **优点：**运行比首次适配块一些，可以跳过开头的碎片
  - **缺点：**内存利用率比首次适配低很多

- **最佳适配（Best Fit）：**分配器会查找所有空闲块，**选择块大小大于k的最小空闲块**。

- - **优点：**内存利用率比前两者都高一些
  - **缺点：**需要遍历完整的空闲链表

#### 在空闲块中分配

- 如果空闲块与k大小相近，则可以直接使用这一整个空闲块
- 如果空闲块比k大很多，如果直接使用整个空闲块，则会造成很大的内部碎片，所以**会尝试对该空闲块进行分割**，一部分用来保存k字节数据，另一部分构成新的空闲块。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916144959364.png" alt="image-20210916144959364" style="zoom: 50%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916145127259.png" alt="image-20210916145127259" style="zoom: 67%;" />

如果分配器找不到满足要求的空闲块，则会首先尝试将物理上相邻的两个空闲块合并起来创建一个更大的空闲块，如果还是不满足要求，则分配器会调用`sbrk`函数来向内核申请额外的堆内存，然后将申请到的新空间当做是一个空闲块。

#### 释放块

只需要将指针所指向位置（header）的已分配标志位变为0即可。

```c
 void free_block(ptr p) { *p = *p & -2 } 
```

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916145729967.png" alt="image-20210916145729967" style="zoom:50%;" />

但是如果被释放的块与其他空闲块相邻，则会产生**假碎片（Fault Fragmentation）**现象，即许多可用的空闲块被分割为小的无法使用的空闲块。所以**当我们释放块时，还需要合并与之相邻的空闲块**。

**优秀的分配器不应该允许有连续的空闲块出现**。

释放块有以下的策略：

- **立即合并（Immediate Coalescing）：**当我们释放一个分配块时，就合并与其相邻的空闲块。

- - **优点：**可在常数时间内完成
  - **缺点：**可能一个空闲块会被来回分割和合并，产生抖动

- **推迟合并（Deferred Coalescing）：**当找不到合适的空闲块时，再扫描整个堆来合并所有空闲块。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916150505000.png" alt="image-20210916150505000" style="zoom:50%;" />

合并相邻的空闲块时，我们可以利用header中的块大小找到下一个块的位置，但是我们却没有高效的方法找到前一个块的位置（只能从链表起始处重新遍历一次）。

为了高效合并前一个空闲块，需要使用**边界标记（Boundary Tag）**技术，使得当前块能迅速判断前一个块是否为空闲的。

在块的数据结构中，会添加一个块头部的副本得到脚部。这样当前块从起始位置向前偏移一个字长度，就能得到前一个块的脚部，通过脚部就能判断前一个快是否为空闲的。这实际上就是双向链表，允许我们反向遍历free list

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916151020445.png" alt="image-20210916151020445" style="zoom:67%;" />

可以将所有情况分成以下几种：

![img](https://raw.githubusercontent.com/BoL0150/image2/master/v2-e6f2612ae7281df6d7155817294184b6_1440w.jpg)

由于引入了脚部，增加了额外开销（overhead），使得内部碎片变多了，并且最小块的大小变大导致外部碎片也变多了。

我们可以对其进行优化，有些情况是不需要边界标记的，只有在合并时才需要脚部，而我们只会在空闲块上进行合并，所以在已分配的块上可以不需要脚部，那空闲块如何判断前一个块是否为已分配的呢？可以在自己的头部的3个位中用一个位来标记前一个块是否为空闲的，如果前一个块为已分配的，则无需关心前一个块的大小，因为不会进行合并；如果前一个块为空闲的，则前一个块自己就有脚部，说明了前一个块的大小，则可以顺利进行合并操作。

### explicit free list

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916164405639.png" alt="image-20210916164405639" style="zoom:67%;" />

因为空闲块中除了头部和脚部以外都是没用的，我们可以在implicit free list的空闲块中加入一个指向前一个空闲块的`pred`指针，还有一个指向下一个空闲块的`succ`指针，**将implicit free list中的空闲块组织成双向链表形式**。这样implicit free list就变成了了显式空闲块列表（explicit free list）。 

但是这种方法需要更大的空闲最小块，否则不够存放两个指针，这就提高了外部碎片的程度。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916170112887.png" alt="image-20210916170112887" style="zoom:50%;" />

对于已分配块，可以通过头部和脚部来得到地址相邻两个块的信息，而对于空闲块，可以通过头部和脚部来得到地址相邻两个块，**也可以通过两个指针直接获得相邻的两个空闲块**。**注意：**逻辑上看这两个空闲块是相邻的，但物理地址上不一定是相邻的。

显式空闲块链表逻辑上是有序的，但是**实际上所连接的空闲块的地址可能是无序的**，空闲块可以以任意顺序连接。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916170835000.png" alt="image-20210916170835000" style="zoom:50%;" />

比如我们这里存在以下3个空闲块的双向链表，此时想要分配中间的空闲块，且对其进行分割

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916173546127.png" alt="image-20210916173546127" style="zoom:50%;" />

**因为已分配块可以根据指针来定位，所以不需要额外进行链接**。而空闲块会从中分割出合适的部分用于分配，其余部分作为新的空闲块，此时只要更新6个指针使其指向和的位置就行。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916173607972.png" alt="image-20210916173607972" style="zoom:50%;" />

而当我们想要释放已分配块时，它并不在空闲链表中，要将其放在空闲链表什么位置？

- **后进先出（LIFO）策略：**将释放的已分配块放到空闲链表开始的地方，则只需要常数时间就能释放一个块。**最后释放的块会最先被分配**（如果匹配），所以叫LIFO。如果使用后进先出和首次适配策略，则分配器会先检索最近使用过的块。但是碎片化会比地址顺序策略严重。
- **地址顺序策略：**释放一个块需要遍历空闲链表，**保证空闲链表的地址是有序的**。即每个空闲块的地址都大于它前驱的地址，小于它后继的地址。这种策略的首次适配会比后进先出的首次适配有更高的内存利用率。

#### LIFO

- 情况一：要释放的块前后都为已分配的块

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916192712966.png" alt="image-20210916192712966" style="zoom:67%;" />

  我们可以通过后面块的头部以及前面块的脚部来得知相邻两个块的已分配状况（这就是保留头部和脚部的意义）。由于相邻的都是已分配的块，所以不会进行空闲块合并，直接更新Root的`succ`指针使其指向要释放的块，让要释放的块的`pred`指向Root，`succ`指向原来第一个空闲块，然后更新原来的第一个空闲块的`pred`指针。

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916192943723.png" alt="image-20210916192943723" style="zoom:67%;" />

- 情况二：要释放的块前面为空闲块，后面为已分配的块

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916193217298.png" alt="image-20210916193217298" style="zoom:67%;" />

  要释放的块前面为空闲块，则需要将当前块和前一块进行合并。我们可以简单地修改头部和脚部直接将两个空闲块合并，将空闲块的前驱和后继节点指针指向改变，再将合并的新的空闲块插入root的后面。（这是LIFO的做法，**对于地址顺序策略，我们不需要将合并的空闲块移动，可以把它留在原地，不更新任何东西**）。

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916193303737.png" alt="image-20210916193303737" style="zoom:67%;" />

- 情况三：要释放的块后面为空闲块，前面为已分配的块

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210929114102941.png" alt="image-20210929114102941" style="zoom: 67%;" />

- 情况四：当前块的前后两个块都为空闲块

  对于前后两个空闲块，直接让其指针前后的两个空闲块修改指针跳过，然后修改头部和脚部进行合并。再插入root的下一个位置

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210916194717059.png" alt="image-20210916194717059" style="zoom: 50%;" />

#### summary

相比于implicit list，explicit list的分配操作与**空闲块的数量**成线性相关，而不是所有块的数量。因此，**对于已分配的块的数量多的情况，explicit list的速度要快得多**。

### Segregated free lists

为了减少分配时间，可以使用**分离存储（Segregrated Storage）**方法，首先将所有空闲块根据块大小分成不同类别，称为**大小类（Size Class）**，比如可以根据2幂次分成![img](https://raw.githubusercontent.com/BoL0150/image2/master/v2-b13008ebc3b408acc716fd392139c6a1_1440w.png)，每个类别一条链表，按照类别将空闲链表放入数组中，类似HashMap的实现方式，由此能极大加快分配速度。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/v2-757ca6274f2a20664cb61fba657cc8a4_1440w.jpg" alt="img" style="zoom:67%;" />

- 当我们想要分配一个大小为n的块时，会首先根据空闲链表数组确定对应的大小类，找到合适的空闲链表，
- 在这个空闲链表中搜索是否有合适的空闲块
  - 如果有，可以对其进行分割，则剩下的部分要放到合适的空闲链表中
  - 如果没有合适的空闲块，在数组中找下一条大小类更大的空闲链表，重复上述步骤。
- 如果遍历了所有大小类的空闲链表还是找不到合适的空闲块时，分配器就会向内核申请更大的堆内存空间，然后将作为一个空闲块放在最大的大小类的空闲链表中。

当我们想要释放一个块时，需要对其地址周围的空闲块进行合并，然后将其放在合适的大小类中。

分离的空闲链表是当前最好的分配器类型

- 对于吞吐量方面，由于将原来巨大的空闲链表根据大小类将其划分为很多小的空闲链表，使得在单一空闲链表中搜索速度快很多，
- 对于内存利用率方面，由于大小类的存在，使得你正在所的空闲链表是最适合你想要分配的大小，在这里**使用first fit就能得到接近在整个空闲链表中使用best fit的性能**。
- 最极端的情况是为每个size都设置一个大小类，这样就等于最佳适配策略的性能了。

对于显式的空闲链表（分离的和显式的）释放的时候都是先与地址周围的空闲块合并，将被合并的空闲块从空闲链表中移除，然后将新的空闲块插入到合适的位置：分离的是插入到合适的空闲链表，显式的是按照不同的算法（如LIFO）插入到不同的位置，

## Garbage Collection

```c
void foo() {
    int *p = malloc(128);
    return; /* p block is now garbage*/
}
```

在隐式分配器中，分配器会释放程序不再使用的已分配块，**自动对其调用`free`函数进行释放**。则应用程序只需要显示分配自己需要的块，而**回收过程由分配器自动完成**。

内存分配器如何知道什么时候一个内存区域可以被释放呢？

- 只要没有指针指向的内存区域就可以被释放（因为没有指针指向，所以永远不可能被程序使用）。
  - 所以我们可以扫描内存，识别内存中的所有指针，并查看指向哪些块。如果一个块没有被任何指针指向，那么它们就是垃圾。

但是这就要求：

- 内存管理器必须能够区分指针和非指针。
  - 当我们看到一个8字节的值时，我们无法区分这是一个long int还是一个指针。
- 所有的指针都必须指向一个块的开始。
  - 如果它指向一个块的内部，我们如何找到该块的开头？我们如何知道该块有多大？
- 指针不能被隐藏

本章主要介绍Mark&Sweep算法，它建立在malloc包的基础上，使得C和C++就有垃圾收集的能力。

垃圾收集器将内存视为一个**有向可达图**（Reachability Graph），其中：

- 具有两种节点：
  - **根节点（Root Node）：**对应于不在堆中但包含指向堆中的指针，可以是寄存器、栈中变量或全局变量等等。
  - **堆节点（Heap Node）：**对应于堆中的一个已分配的块。

- **每个指针都被视为图中的一条边**

如果一个节点从根节点出发不可达，那么该节点就是垃级。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210917171504789.png" alt="image-20210917171504789" style="zoom: 50%;" />

对于像ML和Java语言，其对指针创建和使用有严格的要求，由此来构建十分精确的可达图，所以能回收所有垃圾。而对于像C和C++这样的语言，垃圾收集器无法维护十分精确的可达图，只能正确地标记所有可达节点，而有一些不可达节点会被错误地标记为可达的，所以会遗留部分垃圾，这种垃圾收集器称为**保守的垃圾收集器（Conservative Garbage Collector）**。

在C中使用垃圾收集器可以有两种方法：

- **按需的：**将其集成到`malloc`函数中。当引用调用`malloc`函数来分配块时，如果无法找到合适的空闲块，就会调用垃圾收集器来识别出所有垃圾，并调用`free`函数来进行释放。
- **自动的：** 可以将垃圾收集器作为一个和应用并行的独立线程，不断更新可达图和回收垃圾。

### Mark&Sweep垃圾收集器

Mark&Sweep垃圾收集器由两个阶段组成：

- **标记（Mark）阶段：**从所有的根节点开始，遍历所有的节点，标记出根节点的所有可达的和已分配的子节点。为此，需要在块的头部和脚部的低3位中用一位来标记其是否可达的。
- **清除（Sweep）阶段：**从堆的最开始扫描整个堆，释放所有未标记的已分配块。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210917173316926.png" alt="image-20210917173316926" style="zoom:50%;" />

这两个阶段的伪代码如下所示：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210917175649043.png" alt="image-20210917175649043" style="zoom: 50%;" />

上面实现的困难是：

- 内存管理器如何区分指针和非指针？
  - 当我们看到一个8字节的值时，我们无法区分这是一个long int还是一个指针。比如`p`对应的是一个`double`类型数据，但是C误以为是指针，而将该数据作为指针又正好指向某个不可达的已分配块中，则分配器会误以为该分配块是可达的，造成无法对该垃圾进行回收。这也是C程序的Mark&Sweep垃圾收集器必须是保守的原因。
- 无法确保所有的指针都指向一个块的开始。
  - 如果它指向一个块的内部，我们如何找到该块的开头？我们如何知道该块有多大？

## Memory-Related Perils and Pitfalls

指针的转换：

- 从 `<type_a>*`转换到 `<type_b>*`（一个类型的指针转换为另一个类型的指针）

  - 值并不会改变
  - **改变的是解析引用的行为**，比方说原来是 int 指针，那么一次会读 4 个字节，现在转换成了 char 指针，一次就读 1 个字节

- 如果一个指针的类型是 `void *`，那么是不能对其进行算术操作的，因为我们没办法确定其大小。

  我们不能够对空指针进行解引用，来看一个例子：

  ```c
  int *ptr1 = malloc(sizeof(int)); // malloc返回的是void *，隐式转化为int *
  *ptr1 = 0xdeadbeef;
  int val1 = *ptr1;
  int val2 = (int) *((char *) ptr1);
  ```

  `val1` 的值比较简单，因为没有改动，所以就是 `0xdeadbeef`，不过 `val2` 的值就比较特别了，首先指针 `ptr1` 被转换成了 `char` 指针，按照小端规则，指针指向的值变成了 `0xef`，解引用之后在转换成 `int` 类型，前面会补上 `ffffff`，最后就是 `0xffffffef`
  
- 在C中，void\*可以隐式转换为char\*，**在C++的转换规则中，不能隐式地将void\*转换为char\*，因此，需要使用`reinterpret_cast`进行类型转换。** 

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210917181712484.png" alt="image-20210917181712484" style="zoom:67%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210917181938440.png" alt="image-20210917181938440" style="zoom: 50%;" />

- 主要掌握两点：
  1. ()和[]的优先级比*和&更高
  2. 从里（变量名）向外看，与优先级高的符号先结合

- Dereferencing Bad Pointers

  这是非常常见的例子，没有引用对应的地址，少了 `&`

  ```c
  int val;
  ...
  scanf("%d", val);
  ```

- Reading Uninitialized Memory

  不能假设堆中的数据会自动初始化为 0，**堆中的数据可以是任意值**。（C++中的new与malloc不同！）

  ```c
  /* return y = Ax */
  int *matvec(int **A, int *x) {
      int *y = malloc(N * sizeof(int));
      int i, j;
      
      for (i = 0; i < N; i++)
          for (j = 0; j < N; j++)
              y[i] += A[i][j] * x[j];
      return y;
  }
  ```

  

- Overwriting Memory

  第一种是分配了错误的大小，下面的例子中，一开始不能用 `sizeof(int)`，因为指针的长度不一定和 int 一样。

  ```c
  int **p;
  p = malloc(N * sizeof(int));
  
  for (i = 0; i < N; i++) 
      p[i] = malloc(M * sizeof(int));
  ```

- 第二个问题是超出了分配的空间，下面代码的 for 循环中，因为使用了 `<=`，会写入到其他位置

  ```c
  int **p;
  
  p = malloc(N * sizeof (int *));
  
  for (i = 0; i <= N; i++)
      p[i] = malloc(M * sizeof(int));
  ```

  第三种是因为没有检查字符串的长度，超出部分就写到其他地方去了（经典的缓冲区溢出攻击也是利用相同的机制）

  ```c
  char s[8];
  int i;
  
  gets(s); /* reads "123456789" from stdin */
  ```

- 没有正确理解指针的大小以及对应的操作。**如果增加一个指针，该指针会往后移动指针所指向对象的大小的字节**。比如：如果将int *指针加一，那么该指针会向后移动四个字节。

  ```c
  int *search(int *p, int val) {
      while (*p && *p != null)
          p += sizeof(int);// 这会往后移动四个int的大小，也就是16个字节，而不是4个字节
      
      return p;
  }
  ```

- 引用了指针，而不是其指向的对象，下面的例子中，`*size--` 一句因为 `--` 的优先级比较高，所以实际上是对指针进行了操作，正确的应该是 `(*size)--`

  ```c
  int *BinheapDelete(int **binheap, int *size) {
      int *packet;
      packet = binheap[0];
      binheap[0] = binheap[*size - 1];
      *size--;
      Heapify(binheap, *size, 0);
      return (packet);
  }
  ```

- 引用不存在的变量。注意**局部变量会在函数返回的时候失效（所以对应的指针也会无效）**，这是传引用和返回引用需要注意的，传值的话则不用担心

  ```c
  int *foo() {
      int val;
      return &val;
  }
  ```

- 多次释放同一个块

  ```c
  x = malloc(N * sizeof(int));
  //  <manipulate x>
  free(x);
  
  y = malloc(M * sizeof(int));
  //  <manipulate y>
  free(x);
  ```

- 引用已被释放的块

  ```c
  x = malloc(N * sizeof(int));
  //  <manipulate x>
  free(x);
  //  ....
  
  y = malloc(M * sizeof(int));
  for (i = 0; i < M; i++)
      y[i] = x[i]++;
  ```

- 没有释放分配的块

  ```c#
  foo() {
      int *x = malloc(N * sizeof(int));
      // ...
      return ;
  }
  ```

- 只释放了数据结构的一部分：

  只释放了链表的一部分

  ```c
  struct list {
      int val;
      struct list *next;
  };
  
  foo() {
      struct list *head = malloc(sizeof(struct list));
      head->val = 0;
      head->next = NULL;
      //...
      free(head);
      return;
  }
  ```

  

处理内存bug：

- 数据结构一致性检查器（consistency checker）

  我们写一个函数，确定数据结构应该始终保持结构的不变性，迭代数据结构，**检查所有不变量的结构是否为真**。检查器需要检查：

  - 在分配器中，永远不应该有两个连续的空闲块
  - 每个空闲块都应该出现在某个空闲链表中。一致性检查器将扫描堆中空闲块的数量，空闲链表中的块数应该与其数量相同。




# CMU15-213学习笔记（三） 内存与缓存

## Memory Hierarchy存储体系 

随机存取存储器(RAM, Random-Access Memory) 有两种类型：SRAM(Static RAM) 和 DRAM(Dynamic RAM)，SRAM 非常快，也不需要定期刷新，通常用在处理器做缓存，但是比较贵；DRAM 比较慢（大概是 SRAM 速度的十分之一），需要刷新，通常用作主存，相比来说很便宜（是 SRAM 价格的百分之一）。

RAM通常打包成芯片，基本存储单元是cell（每个cell一位），多个RAM芯片构成一个存储器。

![image-20210726160541481](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210726160541481.png)

（Transistor晶体管）

无论是 DRAM 还是 SRAM，一旦不通电，所有的信息都会丢失。这就是为什么，当我们的电脑断点后，我们会失去内存中的所有东西，再把电脑打开后，需要从硬盘中重新加载所有东西。如果想要让数据持久化，可以考虑 ROM, PROM, EPROM, EEPROM 等介质。

- 固件程序会存储在 ROM 中（比如 BIOS，磁盘控制器，网卡，图形加速器，安全子系统等等）。
- 另外一个趋势就是 SSD 固态硬盘，取代了旋转硬盘（rotating disk），取消了机械结构，更稳定速度更快更省电。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210726175901163.png" alt="image-20210726175901163" style="zoom:67%;" />

对内存的读写通常需要50或100纳秒，由于寄存器离ALU较近，寄存器之间的一些操作只需要不到1纳秒，它们之间大约差了两个数量级。

### Rotating Disk机械硬盘

虽然现在越来越多电脑已经改为使用固态硬盘，但是还是有必要了解一下硬盘的组成的。传统的机械硬盘有许多不同的部件：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210823121914698.png" alt="image-20210823121914698" style="zoom:50%;" />

机械硬盘有许多片磁盘(platter)组成，每一片磁盘有两面；每一面由一圈圈的磁道(track)组成，而每个磁道会被分隔成不同的扇区(sector)。这里概念层层递进，可以结合下图仔细辨析清楚。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210823122121452.png" alt="image-20210823122121452" style="zoom: 33%;" />

上图是一个磁盘的视图，多个磁盘组合起来是这样的：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210823122153124.png" alt="image-20210823122153124" style="zoom: 50%;" />

硬盘的容量指的是最大能存储的比特数，通常用 GB 来做单位。1 GB 相当于 10 的 9 次方个 Byte。与硬盘的结构分层类似，容量取决于下面三个方面：

- 记录密度(bits/in)：track 中 1 英寸能保存的字节数
- 磁道密度(tracks/in)：1 英寸直径能保存多少条 track
- Areal 密度(bits/in 的平方)：上面两个数值的乘积

现在硬盘会把相邻的若干个磁道切分成小块，每一块叫做记录区(recording zone)。记录区中的每条磁道都包含同样数量的扇区(sector)；但是每个记录区中包含的扇区和磁道的数目是不一样的，外层的更多，内层的更少；正因为如此，我们计算容量是，用的是平均的数值。

容量 Capacity = 每个扇区的字节数(bytes/sector) x 磁道上的平均扇区数(avg sectors/track) x 磁盘一面的磁道数(tracks/surface) x 磁盘的面数(surfaces/platter) x 硬盘包含的磁盘数(platters/disk)

举个例子，假如一个硬盘有：

- 512 bytes/sector
- 300 sectors/track (平均)
- 20000 tracks/surface
- 2 surfaces/platter
- 5 platters/disk

总的容量为 = 512 x 300 x 20000 x 2 x 5 = 30,720,000,000 = 30.72 GB

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210823122222850.png" alt="image-20210823122222850" style="zoom:50%;" />

假设我们现在已经从蓝色区域读取完了数据，接下来需要从红色区域读，首先需要寻址，把读取的指针放到红色区域所在的磁道，然后等待磁盘旋转，旋转到红色区域之后，才可以开始真正的数据传输过程。

总的访问时间 Taccess = 寻址时间 Tavg seek + 旋转时间 Tavg rotation + 传输时间 Tavg transfer

- 寻址时间 Tavg seek 因为物理规律的限制，一般是 3-9 ms
- 旋转延迟 Tavg rotation 取决于硬盘具体的转速，一般来说是 7200 RPM
- 传输时间 Tavg tranfer 就是需要读取的扇区数目

举个例子，假设转速是 7200 RPM，平均寻址时间 9ms，平均每个磁道的 sector 数目是 400，那么我们有：

- Tavg rotation = 1/2 x (60 secs / 7200 RPM) x 1000 ms/sec = 4 ms
- Tavg transfer = 60 / 7200 RPM x 1/400 secs/track x 1000 ms/sec = 0.02 ms
- Taccess = 9 ms + 4 ms + 0.02 ms

从这里可以看出，主要决定访问时间的是寻址时间和旋转延迟；读取一个扇区的第一个比特是非常耗时的，之后的都几乎可以忽略不计；硬盘比 SRAM 慢 40,000 倍，比 DRAM 慢 2500 倍。

最后需要知道的就是逻辑分区和实际的物理分区的区别，为了使用方便，会用连续的数字来标志所有可用的扇区，具体的映射工作由磁盘控制器完成。

DMA方式读取硬盘：

1. 假设 CPU 需要从硬盘中读取一些数据，会给定指令，逻辑块编号和目标地址，并发送给磁盘控制器。

   <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210726192954517.png" alt="image-20210726192954517" style="zoom: 50%;" />

2. 然后磁盘控制器会读取对应的数据，并通过 DMA(direct memory access)把数据传输到内存中；

   <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210726193203918.png" alt="image-20210726193203918" style="zoom:67%;" />

3. 传输完成后，磁盘控制器通过中断的方式通知 CPU，然后 CPU 完成之后的工作。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210726193430960.png" alt="image-20210726193430960" style="zoom:67%;" />

### Solid State Disks固态硬盘

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/14612490867850.jpg" alt="img" style="zoom:67%;" />

固态硬盘中分成很多的块(Block)，每个块又有很多页(Page)，大约 32-128 个，每个页可以存放一定数据（大概 4-512KB），页是进行数据读写的最小单位。但是有一点需要注意，对一个页进行写入操作的时候，需要先把整个块清空（设计限制），而一个块大概在 100,000 次写入之后就会报废。

与传统的机械硬盘相比，固态硬盘在读写速度上有很大的优势，如果需要写入 Page，那么需要移动其他 Page，擦除整个 Block，然后才能写入。因为设计本身的约束，连续访问会比随机访问快；读操作比写操作块。现在固态硬盘的读写速度差距已经没有以前那么大了，但是仍然有一些差距。

不过与机械硬盘相比，固态硬盘存在一个具体的寿命限制，价格也比较贵，但是因为速度上的优势，越来越多设备开始使用固态硬盘。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210726201315687.png" alt="image-20210726201315687" style="zoom: 50%;" />

我们可以发现CPU的速度变得越来越快，而存储设备的访问速度却几乎不变，而我们的程序都需要数据，数据存储在内存和磁盘中，所以我们计算机的实际性能不会增加，因为我们会受到访问数据所需时间的限制。

### Locality局部性原理 

弥补CPU和存储器之间速度差距的关键在于一个程序的基本属性：程序的局部性

- 时间局部性(Temporal Locality): 如果一个信息项正在被访问，那么在近期它很可能还会被再次访问。程序循环、堆栈等是产生时间局部性的原因。
- 空间局部性(Spatial Locality): 在最近的将来将用到的信息很可能与现在正在使用的信息在空间地址上是临近的

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210726202824719.png" alt="image-20210726202824719" style="zoom:67%;" />

数据引用和指令引用都分别有局部性的属性。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210726203154422.png" alt="image-20210726203154422" style="zoom: 67%;" />

作为一个程序员，应该能够在看到代码时获取对代码局部性的一些定性感觉。

### Memory Hierarchy存储体系 

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210726210252772.png" alt="image-20210726210252772" style="zoom: 50%;" />

缓存可以看作是把大且缓慢的设备中的数据的一部分拿出来存储到其中的更快的存储设备。在金字塔式存储体系中，每一层都可以看作是下一层的缓存。利用局部性原理，程序会更倾向于访问第 k 层的数据，而非第 k+1 层，这样就减少了访问时间。

**内存层次结构创建了一个大型存储池，其成本与底部附近的廉价存储一样，但它以顶部的快速存储的速度向程序提供数据**。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210726224701355.png" alt="image-20210726224701355" style="zoom:67%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210726224732938.png" alt="image-20210726224732938" style="zoom:67%;" />

缓存未命中有三种，这里进行简要介绍

- 强制性失效(Cold/compulsory Miss): CPU 第一次访问相应缓存块，缓存中肯定没有对应数据，这是不可避免的
- 冲突失效(Confilict Miss): 在直接相联或组相联的缓存中，不同的缓存块由于索引相同相互替换，引起的失效叫做冲突失效
  - 假设这里有 32KB 直接相联的缓存
  - 如果有两个 8KB 的数据需要来回访问，但是这两个数组都映射到相同的地址，缓存大小足够存储全部的数据，但是因为相同地址发生了冲突需要来回替换，发生的失效则全都是冲突失效（第一次访问失效依旧是强制性失效），这时缓存并没有存满
- 容量失效(Capacity Miss): 有限的缓存容量导致缓存放不下而被替换，被替换出去的缓存块再被访问，引起的失效叫做容量失效
  - 假设这里有 32KB 直接相联的缓存
  - 如果有一个 64KB 的数组需要重复访问，数组的大小远远大于缓存大小，没办法全部放入缓存。第一次访问数组发生的失效全都是强制性失效。之后再访问数组，再发生的失效则全都是容量失效，这时缓存已经存满，容量不足以存储全部数据

完整的缓存金字塔结构：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210726225006909.png" alt="image-20210726225006909" style="zoom:67%;" />

我们可以看到，**从最微观的寄存器，到浏览器缓存，web缓存，都属于缓存**。它们的原理都相同：存储着下一层内容的子集，向上一层提供服务。当上一层的请求没有命中时，向下一层请求该内容，存储在本层中，同时返回结果给上一层。

## Cache Memories

高速缓存存储器(Cache Memory)是 CPU 缓存系统甚至是金字塔式存储体系中最有代表性的缓存机制，前面我们了解了许多概念，这一节我们具体来看看高速缓存存储器是如何工作的。

首先要知道的是，高速缓存存储器是由硬件自动管理的 SRAM 内存，CPU 会首先从这里找数据，其所处的位置如下（蓝色部分）：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210823122335926.png" alt="image-20210823122335926" style="zoom: 50%;" />

然后我们需要关注高速缓冲存储器的三个关键组成部分（注意区分大小写）：

- S 表示集合(set)数量（**组**）
- E 表示数据行(line)（相当于缓存快）的数量（**路**）
- B 表示每个缓存块(block)保存的字节数目

在图上表示出来就是

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210823122416486.png" alt="image-20210823122416486" style="zoom:50%;" />

所以缓存中存放数据的空间大小为：C=S×E×B

### Cache Read

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210727105015712.png" alt="image-20210727105015712" style="zoom: 50%;" />

具体在从缓存中读取一个地址时，首先我们通过 set index 确定要在哪个 set （行）中寻找，确定后利用 tag 和同一个 set 中的每个 line （块）进行比对，找到 tag 相同的那个 line，最后再根据 block offset 确定要从 line 的哪个位置读起（这里的 line 和 block 是一个意思）。

- 当 E=1 时，也就是每个 set（组、行） 只有 1 个 line（路） 的时候，称之为**直接映射缓存(Direct Mapped Cache)**，如下图所示

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/14612642281687.jpg" alt="img" style="zoom:67%;" />

  具体的检索过程就是先通过 set index（组索引） 确定哪个 set（组），然后看是否 valid，然后比较那个 set 里唯一 line 的 tag 和地址的 t bits 是否一致，就可以确定是否缓存命中。

  命中之后根据 block offset 确定偏移量，因为需要读入一个 int，所以会读入 4 5 6 7 这四个字节（假设缓存是 8 个字节）。如果 tag 不匹配的话，这行会被扔掉并从内存中读取新的数据进来。

  - 直接映射缓存模拟：

    假设我们的寻址空间是 M=16 字节，也就是 4 位的地址，缓存的空间为8个字节，4个set，每个set 1块，每块2个字节。对应 B=2, S=4, E=1。

    <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210727115744456.png" alt="image-20210727115744456" style="zoom: 50%;" />

    对应就是有 4 个 set，所以需要 2 位的 set index，所以进行读入的时候，会根据中间两位来确定在哪个 set 中查找，**其中 8 和 0，因为中间两位相同，会产生冲突，导致连续 miss**，这个问题可以用多路映射来解决。

- 当 E 大于 1 时，也就是每组有 E 个路的时候，称之为 E 路组相联映射。

  读取一个short int（两个字节），首先根据地址中间的组索引set index确定是哪一行，再判断valid位，使用tag和这一组中的每个块（路）进行对比，找到 tag 相同的那个 line，最后再根据 block offset 确定要从 line 的第5位开始读取两个字节（即4和5）。

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210727120113782.png" alt="image-20210727120113782" style="zoom:67%;" />

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210727120859685.png" alt="image-20210727120859685" style="zoom:67%;" />

  这个情况下即使 8 和 0 的 set index 一致，一个 set 也可以容纳两个数据。如果没有命中，将缓存中**最近最少使用的块（LRU）**替换出去。

也就是说，直接映射和组相联映射都有组索引（处于地址的中间）

- 直接映射用组索引（地址的中间部分）找到对应的行，每行只有一路，也就是一个块，然后用地址的高位tag对该块进行比较
- k路组相联映射也是用组索引（地址的中间部分）找到对应的行，每行有K路，也就是K个块，然后用地址的高位tag对每一块进行比较

### Cache Write

在整个存储层级中，不同的层级可能会存放同一个数据的不同拷贝（如 L1, L2, L3, 主内存, 硬盘）。如果发生写入命中的时候（也就是要写入的地址在缓存中有），有两种策略：

- Write-through: 命中后更新缓存，同时写入到内存中。每次更新缓存都要同时更新内存，效率低下。
- Write-back: 减少更新内存的频率，直到这个缓存需要被置换出去，才写入到内存中（需要额外的 dirty bit 来表示缓存中的数据是否和内存中相同，因为可能在其他的时候内存中对应地址的数据已经更新，那么重复写入就会导致原有数据丢失）

在写入 miss 的时候，同样有两种方式：

- Write-allocate: 在缓存中创建一个新的块（可能会覆盖现有的块），写入数据，更新缓存（如果之后还需要对其操作，我们就可以直接在缓存中读取这个块）
- No-write-allocate: 直接写入到内存中，不载入到缓存

这四种策略通常的搭配是：

- Write-through + No-write-allocate
- **Write-back + Write-allocate**

其中第一种可以保证绝对的数据一致性，第二种效率会比较高（通常情况下）。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210727122906008.png" alt="image-20210727122906008" style="zoom: 67%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210727123558779.png" alt="image-20210727123558779" style="zoom:67%;" />


# CS144实验记录（三）：lab2

分清楚ByteStream和字节流的区别：ByteStream是指在此TCP_lab中的一个数据结构（实际上也是，而字节流则是指双方通信过程中的一个抽象的字节流。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210830152928395.png" alt="image-20210830152928395"  />

## Overview

![image-20210812173918122](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210812173918122.png)

在lab2中，我们将实现`TCPReceiver`。

`TCPReceiver`通过`segment_received()`方法从网络层接收**TCPsegments（即IP数据报的载荷部分）**。每接收到一个segment，就调用一次`STreamReassembler`的`push_substring()`方法。`push_substring`将该segment中的有效载荷切割、去重后加入StreamReassembler的等待重组的缓冲区中，segment中的有效载荷在`StreamReassembler` 中重组成有序的字节流。然后再调用ByteStream中的`write()`方法将该缓冲区中的所有可以加入ByteStream的子串（即与ByteStream连续的子串）全部加入ByteStream中。应用层通过运输层提供的层间接口socket从receiver的ByteStream中读取。

然后**TCPReceiver还负责向sender返回一个TCPsegment**，此TCPsegment包含两个信号，**这两个信号对TCP在不可靠的网络层上提供流量控制和可靠数据传递起到了关键的作用**：

- first unassembled的索引（我在lab1中的`_next_index`的值），即**ackno**。该值就是与ByteStream缓冲区中最后的字符连续的索引，也**就是TCPReceiver期望接收到的下一个segment的索引（以便将更多的segments重组送入ByteStream）**。该值告诉发送方下一个应该发送的segment是什么。
- first unassembled索引和first unacceptable索引的距离，即**window size**。也就是StreamReassembler已接收未重组的空间和空闲空间的大小。
  - ackno和window size一起描述了接收方的窗口`[first_unassembled, first_unacceptable)`：ackno就是接收方可以接收的最小的索引，ackno+window size-1是接收方可以接收的最大的索引。该窗口提供了流量控制，指明了TCPReceiver可以接收的范围，告诉了TCPSender允许发送的范围。

**至此，TCPReceiver接收一个segment的流程才结束，等待sender的下一个segment的到来**。

![](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210817235249307.png)

## 3.1 Translating between 64-bit indexes and 32-bit seqnos

lab1的流重组器重组的子串每个字节都有一个 64 位的**流索引**，流中的第一个字节的索引总是为0。一个 64 位的索引足够大，我们认为它永远不会溢出。但实际上，在 TCP 头部中，空间是非常宝贵的，流中每个字节的索引不是 64位表示的，而是用32位的“序列号”或“seqno”表示的。

这就带来了三个复杂性：

1. 我们的实现需要规划32位整数的循环使用：
   - 通过TCP发送的字节流可以任意长，而在TCPsegment中流的字节的索引是用32位表示的，仅 2^32 bytes = 4GB 可能不够用，所以要循环使用序列号。当一个字节的序列号到达了2^32-1时，下一个字节的序号就要从0开始
2. TCP 序号是从一个随机值开始的
   - 为了安全性，也为了避免被同一端点之间早期连接的旧的segments所混淆，TCP序列号从随机值开始，避免重复和被攻击者猜到。流中的第一个序列号是一个随机的32位数字，称为**初始序列号(ISN)**。这是代表SYN（流的起始）的序列号。后续字节的序号正常工作： (ISN + 1) mod 2^32、(ISN + 2) mod 2^32 ……
3. 字节流的逻辑开始和结束各占据一个序列号：
   - 除了确保接收到所有字节的数据外，TCP 还确保可靠地接收到流的开头和结尾。 因此，**在 TCP 中，SYN（流开始）和 FIN（流结束）控制标志被分配序列号**。 这些中的每一个都占用一个序列号（SYN标志占用的序列号就是ISN），流中的每个数据字节也占用一个序列号。SYN 和 FIN 不是流本身的一部分，也不是“字节”——它们代表字节流本身的开始和结束。

所以综上，我们有三种序列：

- seqno：在**TCP传输的TCPsegment中**的标志每个字节的序列号，从ISN开始，32位。算上了FIN和SYN；返回的ackno也是seqno
- absolute seqno：将seqno变为从0开始，64位。**通过wrap和unwrap与seqno相互转换**。
- stream indices：**实际接受的字节流中每个字节的序列号**，64位，即**真正传输的数据**的序列号（也就是我们在StreamReassembler中使用的索引，从0开始），**FIN和SYN不占序列号**

![image-20210818184635063](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210818184635063.png)

在Absolute seqno 和 Stream Indices 之间转换很简单，加一或减一就好了。

在 Seqno和 Absolute Seqno之间转换就比较困难了，我们将使用自定义类型`WrappingInt32`来表示seqno，并实现它与绝对序列号(`uint64_t`)之间的转换。`WrappingInt32`是一种包装类型：包含内部类型(`uint32_t`)但提供一组不同的函数/操作符。

在`wrapping_integers.cc`中实现 seqno和absolute seqno之间的转换：

```cpp
//从64位的absolute seqno转换成32位的seqno
WrappingInt32 wrap(uint64_t n, WrappingInt32 isn)；
    
//从32位的seqno转换成64位的absolute seqno
uint64 t unwrap(WrappingInt32 n, WrappingInt32 isn, uint64 t checkpoint)；
```

- `WrappingInt32 wrap(uint64_t n, WrappingInt32 isn)；`：给定一个absolute seqno（n）和一个isn，生成n对应的seqno。

  - 很显然，用absolute seqno除以2^32（**即取出n的低32位**）再加上ISN就得到seqno

    ```cpp
    WrappingInt32 wrap(uint64_t n, WrappingInt32 isn) {
        return WrappingInt32(static_cast<uint32_t>(n & 0x00000000ffffffff) + isn.raw_value());
    }
    ```

- `uint64 t unwrap(WrappingInt32 n, WrappingInt32 isn, uint64 t checkpoint)；`：给定一个seqno（n）和一个isn以及一个checkpoint，计算对应n、且**距离checkpoint最近**的absolute seqno。

  checkpoint的必要性：任意一个给定的seqno会有多个对应的absolute seqno。比如seqno17会对应absolute seqno 17、2^32 + 17, or 2^33 + 17, or 2^34 + 17等等。在本实现中需要将**最后一个重组的字节**的absolute seqno作为checkpoint（n转为absolute seqno后的值会在 checkpoint ± (2^32-1) 这个范围）

  - ![image-20210819235703136](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210819235703136.png)用刚写好的 wrap 函数把 checkpoint 变为 32位的seqno，然后求出这个转换后checkpoint 的 seqno 和n的距离，然后把这个步数加到 checkpoint 上。注意这里有一个特殊情况，因为我们有可能是往数轴的反方向走的，可能走完之后 res 的值是个负数，这时候需要在加上一个 2^32。

    ```cpp
    uint64_t unwrap(WrappingInt32 n, WrappingInt32 isn, uint64_t checkpoint) {
      int32_t tmp = n - wrap(checkpoint, isn);
      int64_t ans = checkpoint + tmp;
      return ans >= 0 ? ans : ans + (1ul << 32);
    }
    ```

我们可以从build目录使用`ctest -R wrap`命令来运行`WrappingInt32` 的测试。

![image-20210819234154162](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210819234154162.png)

## 3.2 Implementing the TCP receiver

所以TCPReceiver负责：

1. 从对方接受 TCPSegment，将TCPsegment转化为字节流（包括处理seqno、FIN和SYN标志位）
2. 使用`StreamReassembler`重新组装字节流
3. 计算本地的ackno 和 window size，它们最后被发送回对方

下图是TCPsegment的格式，是网络层的IP数据报的载荷部分。非灰色的字段是这个lab关注的部分

- 由TCPsender发送，被TCPReceiver接收的部分：
  - 序列号seqno；
  - SYN and FIN flags
  - Payload
- 由TCPReceiver发送，被TCPsender接收的部分：
  - ackno
  - window size

![image-20210821093250808](https://raw.githubusercontent.com/BoL0150/image2/master/mQHyoZegScCE1zi.png)



`TCPReceiver.hh`提供的接口：

```c++
class TCPReceiver {
    StreamReassembler _reassembler;

    //TCPReceiver将存储的最大字节数
    size_t _capacity;

  public:
    //构造一个 `TCPReceiver`，最多可以存储 `capacity` 字节，实际上就是STreamReassembler的大小
    TCPReceiver(const size_t capacity) : _reassembler(capacity), _capacity(capacity) {}

	//向远程TCPsender提供反馈
    
    //返回包含ackno的optional<WrappingInt32>，如果ISN还没有初始化（即没有收到SYN）则返回空。 ackno是接收窗口的开始，或者说接收者没有接收到的流中第一个字节的序列号 
    std::optional<WrappingInt32> ackno() const;

    //应该发送给对方的接收窗口大小
    //capacity减去 TCPReceiver 在其BYteStream中保存的字节数（那些已重组但被应用层读取的字节数）。
    //也相当于first unassembled索引（即ackno）和first unacceptable索引的距离
    size_t window_size() const;
    
    //!已存储但尚未重组的字节数
    size_t unassembled_bytes() const { return _reassembler.unassembled_bytes(); }

    //处理接收到的TCPsegment
    void segment_received(const TCPSegment &seg);

    //返回StreamReassembler中的已重组的字节流ByteStream
    ByteStream &stream_out() { return _reassembler.stream_out(); }
    const ByteStream &stream_out() const { return _reassembler.stream_out(); }
};

```



### 3.2.1 segment_received()

实现这个函数是本次实验的主要工作，每次从对方接收一个新的TCPsegment时，都会调用一次 `segment_received()`。该方法需要做的是：

- 初始化ISN。
  - 第一个携带SYN 标志到达的段的seqno就是 ISN，我们需要记录该值，以便能够使用unwarp函数将TCPsegment的32位seqno转换成64位的absolute seqno。（ SYN flag 值是TCP头部的一个标志位。同样的段也能够同时携带 FIN 标志位，所以 SYN 和 FIN 可能在一个段内一起到达）
- 把所有的数据和流结束标志（FIN）交给 StreamReassembler（调用`push_substring`方法）。
  - 如果TCP header 中的 FIN标志位被设置了，那么就意味着负载的最后一个字节就是整个流的最后一个字节。StreamReassembler 期望 stream indexes 从 0 开始，你需要 unwrap seqno 来生成 stream indexes

所以，`segment_received()`方法具体需要做的是：

- 得到`segment`的`SYN`字段，从而判断是否需要设置
- 获取`ISN`字段
- 得到`FIN`字段
- 调用unwarp函数将TCPsegment的32位seqno转换成64位的absolute seqno，checkpoint为流索引格式的ackno，也就是字节流中期望接收的索引号，因为期望接收到的seqno应该就在ackno附近
- 将absolute seqno减一转换成stream indices，和segment的data部分、以及fin一起调用`push_substring`方法

#### ackno()

ackno被包含在TCPsender发送的TCPsegment中发送给对方，所以是32位的。需要由first unassembled（stream indices）转换为absolute seqno，再warp成32位的seqno。

- 如果之前没收到syn，则返回空ackno

- 如果之前收到了syn，也收到了fin，而且fin还被读取进了ByteStream（也就是说此时`StreamReassembler`中未重组的字节数为0) 。则ackno等于stream indices加2（因为`first_unassembled`前面多出了syn和fin）
- 如果之前只收到了syn，则则ackno等于stream indices加1（因为`first_unassembled`前面多出了syn）

#### TCPReceiver在TCP连接的生命周期中的演变

![image-20210821140114743](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210821140114743.png)

`tcp_receiver.hh`：

```c++
class TCPReceiver {
    StreamReassembler _reassembler;

    size_t _capacity;
    bool _syn = false;
    bool _fin = false;
    size_t _isn = 0;
    uint64_t _checkpoint = 0;


  public:
    TCPReceiver(const size_t capacity) : _reassembler(capacity), _capacity(capacity) {}
    std::optional<WrappingInt32> ackno() const;
    size_t window_size() const;
    size_t unassembled_bytes() const { return _reassembler.unassembled_bytes(); }
    void segment_received(const TCPSegment &seg);
    ByteStream &stream_out() { return _reassembler.stream_out(); }
    const ByteStream &stream_out() const { return _reassembler.stream_out(); }
};

```

`tcp_receiver.cc`：

```c++
void TCPReceiver::segment_received(const TCPSegment &seg) {
    TCPHeader header = seg.header();
    //处于listen状态，只接收syn
    if(!header.syn && !_syn)return;
    //如果已经接收到了syn，就拒绝接收其他的syn
    if(header.syn && _syn) return;
    if(header.syn){
        _syn = true;
        _isn = header.seqno.raw_value();
    }
    if(_syn && header.fin)
        _fin = true;
    size_t absolute_seqno = unwrap(header.seqno,WrappingInt32( _isn ),_reassembler.first_unassembled());
    uint64_t stream_indices = header.syn ? 0 : absolute_seqno -1;
    _reassembler.push_substring(seg.payload().copy(),stream_indices,header.fin); 
}

optional<WrappingInt32> TCPReceiver::ackno() const { 
    //如果没有收到syn，就返回空
    if(!_syn)return {};
    //如果之前收到了syn，也收到了fin，同时fin被读取进ByteStream（也就是说此时
    //StreamReassembler中未重组的字节数为0) 。则ackno等于stream indices加2（因为
    //first_unassembled前面多出了syn和fin）
    if(_fin && _reassembler.unassembled_bytes() == 0)
        return wrap(_reassembler.first_unassembled() + 2,WrappingInt32(_isn));
    return wrap(_reassembler.first_unassembled() + 1,WrappingInt32(_isn));
}

size_t TCPReceiver::window_size() const { 
    return _capacity - stream_out().buffer_size();
}

```



![image-20210821184108421](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210821184108421.png)


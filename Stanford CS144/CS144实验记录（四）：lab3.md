# CS144实验记录（四）：lab3

![image-20210812173918122](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210812173918122.png)

在lab3中，我们需要实现`TCPsender`。`TCPsender`负责接收对方发送的`TCPsegment`中的ack号和接收窗口大小（first unassembled索引和first unacceptable索引的距离），应用层通过socket将字节流写入`TCPsender`中的`ByteStream`，`TCPsender`**根据接收到的`ackno`和window size，从`ByteStream`中读取出来，将`ByteStream`中的字节流转化为连续的`TCPsegment`，发送给对方**。

而**在接收方，`TCPReceiver`将收到的`TCPsegment`转化回原始的`BYteStream`中的字节流**，并发送`ackno`和window size给sender。

TCPsender和TCPReceiver各负责发送和接收TCPsegment的一部分：

- 由TCPsender发送，被TCPReceiver接收的部分：
  - 序列号seqno
  - SYN and FIN flags
  - Payload
- 由TCPReceiver发送，被TCPsender接收的部分：
  - ackno
  - window size

这是 TCPsegment的结构，红色部分突出显示的是TCPsender将读取的字段

![image-20210828231240114](https://raw.githubusercontent.com/BoL0150/image2/master/oCIMizV9hGYuPtS.png)

TCPSender’s responsibility：

- 追踪TCPReceiver的接收窗口（处理收到的ackno和windowsize）
- 通过从ByteStream读取字节流，创建TCPsegment（包括将流索引转化为seqno，处理SYN和FIN标志）并发送，**尽可能地填满接收窗口**。**只有当接收窗口已满或TCPsender的BYteStream为空时，TCPsender才能停止发送segments**。
- 追踪已发送但是没有收到ackno的segments（称为outstanding segments），超时之后重新发送这些segments。

`TCPsender`发送的`TCPsegment`每一个都由`ByteStream`字节流中的子串组成（也可能为空），此`segment`中还包含`seqno`，标识这个子串在字节流中的索引；以及SYN和FIN标志位，表示是否是字节流的开头和结尾。

TCPsender需要维护一个数据结构，存储已发送未确认的TCPsegment集合（也可以称为**发送窗口**），如果**最早发送的segment超时还没有被确认，就重传**。

![image-20210829213515415](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210829213515415.png)

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210830152928395.png" alt="image-20210830152928395"  />

至此，我们的lab都是单向通信。发送方只有一个`TCPsender`，接收方只有一个`TCPReceiver`，**由`TCPsender`发送的和`TCPReceiver`返回的都是`TCPsegment`格式的TCP报文段**，只不过由于单向通信，所以发送方的`TCPsegment`中只包含序列号`seqno`、 Payload、SYN 和 FIN，接收方返回的`TCPsegment`中只包含`ackno`和window size。

TCPReceiver返回的window size是TCPReceiver还可以接收多少字节的**载荷**，不包括FIN和SYN。而TCPsender用window size来限制发送窗口的字节长度，发送窗口中的字节长度是载荷加上FIN和SYN。

具体实现细节：

- 每隔几毫秒，TCPSender 的 tick 方法将被调用，并带有一个参数，该参数告诉它自上次调用该方法以来已经过去了多少毫秒。 **使用它来维护 TCPSender 一直存活的总毫秒数**。

- 当 TCPSender 被构造时，它被赋予一个参数，告诉它重传超时 (retransmission timeout，`RTO`) 的“初始值”。 RTO 是在重传之前等待的毫秒数。 RTO 的值会随时间变化，但“初始值”保持不变。 初始代码将 RTO 的“初始值”保存在名为`_initial_retransmission_timeout`的成员变量中
- 我们需要实现一个超时计时器，从某一个特定时间开始计时，当RTO时间过去后，该计时器重启。消耗的时间的概念来自于被调用的tick方法，而不是实际的时间。

- 每当一个**包含数据**的segment（长度不为0）被发送时，**如果计时器没有启动，就启动一个计时器，在RTO时间后“报警”**。**同时，发送窗口的后沿向后移动（即将该segment送入发送窗口）**。

- **当所有已发送未确认的segments都被确认，停止计时器**。

- 计时器超时后：

  - **重传已发送未确认的最早的segments**。

  - 如果window size不为0：

    - **跟踪连续重传的次数**，我们的 TCPConnection 将使用此信息来决定连接是否无望（连续重传过多）并需要中止
    - 将RTO的值翻倍，称作**指数避退**，它减少了糟糕网络上的重传，以避免进一步破坏网络状况。

    以上两项作用是进行拥塞控制。

  - 重启计时器，并在RTO时间后再次重复以上的过程（不要忘了将RTO翻倍）

- **TCPsender收到ackno**（ackno表示累积确认，即在此之前的（seqno小于ackno的）所有segments都已经收到）：

  - 在此lab中，我们认为只有能确认接收窗口中**一整段segment**的ackno才算数。

    如果ackno大于sendBase（发送窗口的前沿，即已发送未确认的最早的字节的序号，就表示sendBase及其之前的所有字节都已确认），并且ackno可以确认至少一整段segment：

    由于收到了合法的ackno，说明网络拥塞状态有所缓解，所以：

    - 将RTO重置为初始值
    - 将连续重传的次数变为0

    除此之外还需要：

    - 重启计时器（超时的值为重置的RTO）

    - 发送窗口的前沿sendBase向后移动到ackno所在的位置（即将发送窗口中seqno小于ackno的segments删除，在本lab中，是将发送窗口中被完全确认的segments删除）
    - 如果sender没有任何未确认的segments，关闭计时器



## 3.2 Implementing the TCP sender

```cpp
class TCPSender {
  private:
    //! our initial sequence number, the number for our SYN.
    WrappingInt32 _isn;
    //TCPsender已发送未确认的segments队列（即outstanding segments）
    std::queue<TCPSegment> _segments_out{};
    //重传计时器
    unsigned int _initial_retransmission_timeout;
    //可以发送的字节流
    ByteStream _stream;
    //要发送的下一个字节的absolute seqno
    uint64_t _next_seqno{0};

  public:
    //! Initialize a TCPSender
    TCPSender(const size_t capacity = TCPConfig::DEFAULT_CAPACITY,
              const uint16_t retx_timeout = TCPConfig::TIMEOUT_DFLT,
              const std::optional<WrappingInt32> fixed_isn = {});

    //!"Input" interface for the writer
    ByteStream &stream_in() { return _stream; }
    const ByteStream &stream_in() const { return _stream; }
    
    void ack_received(const WrappingInt32 ackno, const uint16_t window_size);

    //!生成一个负载为空的segment（用于创建空的 ACK segment）
    void send_empty_segment();

    //!创建并发送segments以尽可能多地填充接收窗口
    void fill_window();

    //!通知 TCPSender 时间的流逝
    void tick(const size_t ms_since_last_tick);
    //! 已发送未确认的字节数是多少，SYN和FIN也占一个字节
    size_t bytes_in_flight() const;

    //!连续重传次数
    unsigned int consecutive_retransmissions() const;

    //! \brief TCPSegments that the TCPSender has enqueued for transmission.
    //! \note These must be dequeued and sent by the TCPConnection,which will need to fill in the fields that are set by the TCPReceiver(ackno and window size) before sending.
    std::queue<TCPSegment> &segments_out() { return _segments_out; }
    
    //! \brief absolute seqno for the next byte to be sent
    uint64_t next_seqno_absolute() const { return _next_seqno; }

    //! \brief relative seqno for the next byte to be sent
    WrappingInt32 next_seqno() const { return wrap(_next_seqno, _isn); }
};
```

TCPsender的四个接口如下，每个接口对应一个TCPsender需要处理的重要事件，每个事件最终都需要发送一个TCPsegment：

- `void ack_received( const WrappingInt32 ackno,const uint16_t windows_size )`

  接收到从`TCPReceiver`返回的ack segments，提取出其中的ackno和window size，TCPsender需要将发送窗口中`seqno`小于`ackno`的segments删除，根据window size调整发送窗口的大小，**调用 `fill_windows()`继续传送**

- `void fill_windows()`

  TCPSender 被要求填充窗口：它从它的输入 `ByteStream` 中读取字节，然后构造成**TCPSegment **（加上SYN和FIN，并且占据序列号和空间）发送**尽可能多**的字节，只要ByteStream中有字节可以读取**并且**~~接收窗口有可用空间（window size> 0)~~**发送窗口有空位**。

  我们发送的每一个TCPsegment的**载荷**要尽可能地大，但是不要超过`TCPConfig::MAX_PAYLOAD_SIZE` (1452字节，链路层MTU减去IP数据报首部和UDP报文段首部和TCP报文段首部)

  我们可以使用`TCPSegment::length_in_sequence_space()`方法计算出一个segment所占据的序列号的长度，**如果segment中包括SYN和FIN标志位，也需要各占据一个序列号**。**segment存储在发送窗口**中，所以SYN和FIN在发送窗口也需要占据空间。

  - 注意，**如果TCPReceiver返回的ack segment中的window size为0，那么 `fill_windows()`方法将window size视为1**，TCPsender会发送一个被接收方拒绝（并且未确认）的字节，这样会促使接收方发送一个新的ack segment，从而表明此时的window size的大小。 **如果不这样，即使接收窗口有空闲的空间了，发送者也不知道它可以再次开始发送**。
    - ~~比如：TCPReceiver的ByteStream不停接收segment，应用层没有读取，最终ByteStream满了，size变为capacity，返回给TCPSender的window size为0 。TCPSender将收到的window size当成1，发送一个空segment，~~

- `void tick( const size_t ms_since_last_tick )`

  每隔几毫秒，TCPSender 的 tick 方法将自动被调用（不需要我们调用），并带有一个参数，该参数告诉它自上次调用该方法以来已经过去了多少毫秒， **使用它来维护 TCPSender 一直存活的总毫秒数**。**我们只需要在 tick 中实现，通过参数判断过去了多少时间，需要执行何种操作即可**

- `void send_empty_segment()`

  TCPsender发送一个序列号长度为0的TCPsegment，但seqno被正确的设置。当TCP连接想要发送一个空ACK segment时，可以使用此方法。

  - 注意，一个序列号长度为0的segment，不需要作为outstanding segments放入发送窗口中被追踪，也不需要重传。

我们需要在TCPsender中实现一个队列`_outstanding_segments`存储已发送未确认的segments，也就是以上所说的发送窗口。

**在本lab中，我们假设TCPsegment发送到`_segments_out`队列中就算发送出去了**，就TCPsender而言，一旦我们将TCPsegment推送到此队列，我们就认为它已经发送。很快，TCPsender的所有者（我们将在lab4中实现的TCPconnection）会对它进行pop（使用公共方法`segments_out()`访问该队列），并真正地发送它。

在从TCPReceiver获取ackno之前，TCPsender假设window size为一个字节。

如果接收到outstanding segments的部分ackno，实际的TCPsender可以对segments已确认的部分进行切割，而我们的TCPsender不要求实现此功能，**一个TCPsegment只有全部被ack才能被remove**。

### 具体实现

`tcp_sender.hh`

```cpp
#include "byte_stream.hh"
#include "tcp_config.hh"
#include "tcp_segment.hh"
#include "wrapping_integers.hh"
#include <functional>
#include <queue>
class TCPSender {
  private:
    //! our initial sequence number, the number for our SYN.
    WrappingInt32 _isn;
    //TCPsender已发送未确的segments队列（即outstanding segments）
    std::queue<TCPSegment> _segments_out{};
    //重传超时的初始值,在TCPsender被构造时会被初始化
    unsigned int _initial_retransmission_timeout;
    //重传超时,初始值等于_initial_retransmission_timeout
    uint32_t _RTO;
    //当计时器超时时，_RTO是否需要翻倍
    bool _back_off = true;
    //可以发送的字节流
    ByteStream _stream;
    //要发送的下一个字节的absolute seqno
    uint64_t _next_seqno{0};
    //记录发送窗口中的字节数量
    uint64_t _bytes_in_flight{0};
    //已发送未确认的segments
    std::queue<TCPSegment> _outstanding_segments{};
    //计时器是否启动
    bool _timer_running = false;
    //超时计时器，如果到达RTO，则重传最老的segment
    uint32_t _time_elipsed = 0;
    //连续重传的次数
    uint16_t _consecutive_retransmissions = 0;
    //保存TCPReceiver报文段携带的接收窗口大小,也就是发送窗口的最大值
    //初始值为1.如果接收到的值为0，也视为1.
    uint16_t _receiver_window_size = 1;
    //TCPReceiver接收窗口的空闲空间大小
    uint16_t _receiver_free_space = 0;
    //是否发送了syn
    bool _syn_sent = false;
    //是否发送了fin
    bool _fin_sent = false;
    //发送segment的辅助方法，封装所有发送segments都要做的操作
    void _send_segments(TCPSegment & seg);

  public:
    //! Initialize a TCPSender
    TCPSender(const size_t capacity = TCPConfig::DEFAULT_CAPACITY,
              const uint16_t retx_timeout = TCPConfig::TIMEOUT_DFLT,
              const std::optional<WrappingInt32> fixed_isn = {});
    //!"Input" interface for the writer
    ByteStream &stream_in() { return _stream; }
    const ByteStream &stream_in() const { return _stream; }
    
    //接收TCPReceiver返回的ackno和window size
    void ack_received(const WrappingInt32 ackno, const uint16_t window_size);
    //判断接收的absolute ack是否合法
    bool valid_ack(uint64_t abs_ack);
    //!生成一个负载为空的segment（用于创建空的 ACK segment）
    void send_empty_segment();
    //!创建并发送segments以尽可能多地填充接收窗口
    void fill_window();
    //!通知 TCPSender 时间的流逝
    void tick(const size_t ms_since_last_tick);
    //! 已发送未确认的字节数是多少，SYN和FIN也占一个字节
    size_t bytes_in_flight() const;
    //!连续重传次数
    unsigned int consecutive_retransmissions() const;

    //! \brief TCPSegments that the TCPSender has enqueued for transmission.
    //! \note These must be dequeued and sent by the TCPConnection,which will need to fill in the fields that are set by the TCPReceiver(ackno and window size) before sending.
    std::queue<TCPSegment> &segments_out() { return _segments_out; }
    
    //! \brief absolute seqno for the next byte to be sent
    uint64_t next_seqno_absolute() const { return _next_seqno; }

    //! \brief relative seqno for the next byte to be sent
    WrappingInt32 next_seqno() const { return wrap(_next_seqno, _isn); }
};
```

`tcp_sender.cc`

```cpp
#include "tcp_sender.hh"
#include "tcp_config.hh"
#include <random>
// Dummy implementation of a TCP sender

// For Lab 3, please replace with a real implementation that passes the
// automated checks run by `make check_lab3`.

template <typename... Targs>
void DUMMY_CODE(Targs &&... /* unused */) {}

using namespace std;

//! \param[in] capacity 初始化ByteStream，容量为capacity
//! \param[in] retx_timeout 重传超时的初始值
//! \param[in] fixed_isn the Initial Sequence Number to use, if set (otherwise uses a random ISN)
TCPSender::TCPSender(const size_t capacity, const uint16_t retx_timeout, const std::optional<WrappingInt32> fixed_isn)
    : _isn(fixed_isn.value_or(WrappingInt32{random_device()()}))
    , _initial_retransmission_timeout{retx_timeout}
    , _RTO(retx_timeout)
    , _stream(capacity){}

//已发送未确认的字节数量
uint64_t TCPSender::bytes_in_flight() const {
    return _bytes_in_flight ;
}

void TCPSender::fill_window() {
    //如果SYN没有发送，则发送SYN后返回
    if(!_syn_sent){
        TCPSegment seg;
        _syn_sent = true;
        seg.header().syn = true;
        _send_segments(seg);
        return;
    }
    //如果SYN发送了但是没有被确认，则返回，等待确认
    if(!_outstanding_segments.empty() && _outstanding_segments.front().header().syn)
        return;
    //如果发送了FIN，则返回
    if(_fin_sent)return;
   
    //计算之后_receiver_free_space的值可能为负，转换为uint16_t后就溢出了
    //实在想不出什么办法解决这个问题，只有出此下策
    _receiver_free_space = _receiver_window_size <= _bytes_in_flight ? 0 : _receiver_window_size - _bytes_in_flight;
    //如果字节流结束了，没有发送FIN，且发送窗口有空余空间，则发送FIN
    //注意，发送结束的标志并不只是ByteStream为空，还需要应用层结束输入
    if(_stream.eof() && _receiver_free_space >= 1){
        TCPSegment seg;
        seg.header().fin = true;
        _fin_sent = true;
        _send_segments(seg);
        return;
    }
    //接收窗口空余空间的大小
    _receiver_free_space = _receiver_window_size <= _bytes_in_flight ? 0 : _receiver_window_size - _bytes_in_flight;
    //只要ByteStream不为空且接收窗口有空余的空间,就可以继续发送segments
    while(!_stream.buffer_empty() && _receiver_free_space > 0){
        TCPSegment seg;
        //接收窗口空闲大小、ByteStream可以读取的字节数量以及 TCPConfig::MAX_PAYLOAD_SIZE三者对载荷的大小进行限制
        size_t temp = _receiver_free_space > TCPConfig::MAX_PAYLOAD_SIZE ? TCPConfig::MAX_PAYLOAD_SIZE : _receiver_free_space;
        size_t payload_size = _stream.buffer_size() > temp ?temp :_stream.buffer_size();
        seg.payload() = _stream.read(payload_size);
        
        //当发送窗口空间充足时，要求将FIN捎带在segment中
        if(_stream.eof() && _receiver_free_space >= 1 + seg.length_in_sequence_space()){
            seg.header().fin = true;
            _fin_sent = true;
        }
        _send_segments(seg);
        _receiver_free_space = _receiver_window_size <= _bytes_in_flight ? 0 : _receiver_window_size - _bytes_in_flight;
    }
    //由于我们直接将收到为0的window size视为1，所以此处省去了特殊判断
}
void TCPSender::_send_segments(TCPSegment &seg){
    seg.header().seqno = wrap(_next_seqno,_isn);
    //length_in_sequence_space计算segment的载荷部分和SYN和FIN的长度
    _next_seqno += seg.length_in_sequence_space();
    //更新发送窗口中的字节数量
    _bytes_in_flight += seg.length_in_sequence_space();
    //将segments发送出去，加入发送窗口
    _segments_out.push(seg);
    _outstanding_segments.push(seg);
    //当发送一个segment时，如果计时器没有启动，就启动一个计时器
    if(!_timer_running){
        _timer_running = true;
        _time_elipsed = 0;
    }
}

//! \param ackno The remote receiver's ackno (acknowledgment number)
//! \param window_size The remote receiver's advertised window size
void TCPSender::ack_received(const WrappingInt32 ackno, const uint16_t window_size) { 
    //收到的ackno是32位的seqno，而发送窗口中的是64位的absolute seqno，
    //所以需要将ackno进行unwrap
    //TCPReceiver对seqno进行unwrap的checkpoint是first unassembled，
    //因为TCPReceiver期望收到的seqno就是这个。
    //同理，TCPsender期望收到的ackno是_next_seqno，所以
    //对ackno进行unwrap的checkpoint是_next_seqno
    uint64_t absolute_ackno = unwrap(ackno,_isn,_next_seqno);
    //只接收部分ackno.此处的判断条件比较宽松,只要ackno在发送窗口范围中就可以
    if(!valid_ack(absolute_ackno)){
        return;
    }
    //根据TCPReceiver返回的信息更新TCPsender中保存的接收窗口大小
    this->_receiver_window_size = window_size ;
    //如果返回的window size为0，将其视为1，但是当计时器超时时，不需要进行指数退避
    if(window_size == 0){
        this->_receiver_window_size = 1;
        _back_off = false;
    }else
        _back_off = true;
    //将发送窗口中被确认的segment删除
    while(!_outstanding_segments.empty()){
        TCPSegment front_outstanding_segment = _outstanding_segments.front();
        uint64_t front_abs_seqno = unwrap(front_outstanding_segment.header().seqno,_isn,_next_seqno);

        //只有能够确认发送窗口中至少一整段segment（也就是说，ackno要大于等于发送窗口中
        //第二段segment的seqno，即第一段segment的seqno 加上 第一段segment的长度）的ackno，
        //才可以执行以下操作
        if(absolute_ackno >= front_abs_seqno + front_outstanding_segment.length_in_sequence_space()){
            
            //将重传超时RTO恢复为初始值
            _RTO = _initial_retransmission_timeout;
            //将连续重传的次数变为0
            _consecutive_retransmissions = 0;
            //将被确认的segments删除
            _outstanding_segments.pop();
            //更新发送窗口中的字节数量
            _bytes_in_flight -= front_outstanding_segment.length_in_sequence_space();
            //重置计时器
            _time_elipsed = 0;
        }else 
            break;
    }
    //如果发送窗口中没有任何未确认的segments，关闭计时器
    if(_outstanding_segments.empty()){
        _timer_running = false;
        _time_elipsed = 0;
    }
    //window size更新后，就可以继续发送新的segments,填充发送窗口
    fill_window();
}
//在此lab中，我们认为只有能确认发送窗口中一整段segment的ackno才算数
//此处的判断条件更加宽松一些，只要ackno大于第一段的seqno即可
bool TCPSender::valid_ack(uint64_t abs_ack){
    //获取发送窗口中最老的segment
    TCPSegment front_outstanding_segment = _outstanding_segments.front();
    //ackno需要小于等于_next_seqno（发送窗口的最后一个字节的下一个索引）,
    //并且大于第一段的seqno
    return abs_ack <= _next_seqno && 
        abs_ack >= unwrap( front_outstanding_segment.header().seqno ,_isn,_next_seqno) ; 
}

//! \param[in] ms_since_last_tick the number of milliseconds since the last call to this method
void TCPSender::tick(const size_t ms_since_last_tick) { 
    if(_timer_running){
        _time_elipsed += ms_since_last_tick;
        //计时器超时
        if(_time_elipsed >= _RTO){
            //重传已发送未确认的最早的segments
            _segments_out.push(_outstanding_segments.front());
            //如果window size不为0，跟踪连续重传的次数，将RTO的值翻倍
            if(_receiver_window_size != 0 && _back_off){
                _consecutive_retransmissions++;
                _RTO *= 2;
            }
            //重启计时器
            _time_elipsed = 0;
        }
    }
}

unsigned int TCPSender::consecutive_retransmissions() const { return _consecutive_retransmissions;}

void TCPSender::send_empty_segment() {
    TCPSegment seg;
    seg.header().seqno = wrap(_next_seqno,_isn);
    _segments_out.push(seg);
}
```

![image-20210901180854720](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210901180854720.png)



关于此lab的疑问：

- TCPReceiver返回的window size是TCPReceiver还可以接收多少字节的**载荷**，不包括FIN和SYN。而TCPsender用window size来限制发送窗口的字节长度，但是发送窗口中的字节长度却是**载荷加上FIN和SYN**。
- 为什么要用unsigned类型？本lab中几个unsigned的变量相互减，会出现溢出的情况，难以发觉。


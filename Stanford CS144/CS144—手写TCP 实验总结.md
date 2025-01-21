# CS144—手写TCP 实验总结

每个TCP连接是一个TCPConnection类，此类由两部分组成：

- TCPSender类：负责接收对方发送的TCPsegment中的ack号和接收窗口大小。TCPsender还负责产生下一个发送的TCPSegment的有效载荷、seqno（序列号），SYN和FIN标志位。
- TCPReceiver类：负责接收对方发送的TCPSegment（主要接收seqno（序列号）、有效载荷、FIN和SYN标志位）。由收到的上一个segment的序列号和载荷才能确定发送给对方的ack号和接收窗口的大小，所以TCPreceiver还负责产生下一个发送的TCPSegment的ackno和window_size（接收窗口）。

![image-20210812173918122](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210812173918122.png)

TCPSender和TCPReceiver内部各有一个发送和接收缓冲区`ByteStream`，分别用来缓存从应用层接收后准备发送出去的数据和准备交给应用层的数据。

TCPReceiver将接收到的数据存入TCPReceiver内部的ByteStream中，等待应用层读取。TCPSender的应用层将要发送的数据存入TCPSender内部的ByteStream中，等待TCPSender发送。

这个字节流类似于一个带容量的队列，从一头读，从另一头写。当流中的数据达到容量上限时，便无法再写入新的数据。特别的，读操作被分为了peek和pop两步。peek为从头部开始读取指定数量的字节，pop为弹出指定数量的字节。

- 在输入端写入的字节以相同的顺序从输出端读出来
- 字节流是有限的，发送方可以结束输入，然后就没有字节可以继续被写入了

- 当接收方读取到字节流的结束，会遇到EOF，然后就没有字节可以继续被读取了

数据结构：由于`queue`只能访问开头的节点，所以使用`deque<char>`实现。将输入的`string`中的char一个个push到deque中，接收端再从deque中读取相应长度的char。

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

## TCPReceiver

TCPReceiver接收端收到的是一个个的TCP数据段（segment），它们有可能并不按照发送端发出的顺序排列，还有可能发生丢失、重叠或者重复。而ByteStream在输入端写入的字节会以相同的顺序从输出端读出来，所以我们需要确保最终存入ByteStream的是正确的字节流。所以TCPReceiver中还需要一个`StreamReassembler`流重组器（stream reassembler），**可以将带索引的字节流碎片重组成有序的字节流，将收到的报文段按情况送入 ByteStream ，或丢弃，或暂存（在合适的时候重组送入 ByteStream）**。

- 对方发送的segment由receiver接收，**segment中的有效载荷在`StreamReassembler` 中重组成有序的字节流，再写入ByteStream**，然后应用层通过运输层提供的层间接口socket从receiver的ByteStream中读取。
- 应用层通过socket将字节流写入sender中的ByteStream，再由TCPsender从ByteStream中读取出来，发送给对方。

`StreamReassembler`将接收子字符串，由一串字节组成，以及该字符串在字节流中的第一个字节的索引。`StreamReassembler`最多可以存储capacity个字节，**capacity包括已重组的字节（即在ByteStream中的字节）和已接收但是未重组的字节（即在`StreamReassembler`中的字节）**。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210813180717383.png" alt="image-20210813180717383" style="zoom: 67%;" />

每个字节流碎片都通过索引（seqno）、长度、内容（segment中的有效载荷）三要素进行描述。接收到并且重组完的字节流应当被送入指定的字节流（byte stream）对象`_output`中。还有部分容量用于存储不与`_output`连续的子串，一旦连续自然也需要写入`_output`，也就是`StreamReassembler`。**以上的两部分容量合起来就是capacity。**也就是说，这两部分中任意一部分最大容量不超过capacity。

![image-20210817235249307](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210817235249307.png)

TCPReceiver接收到的TCPsegment在StreamReassembler中重组成有序的字节流后，就可以加入ByteStream中，等待应用层读取。

所以我们需要将收到的segment按照seqno从小到大进行排序，因此选择set作为StreamReassembler缓冲区的数据结构，将处理的segment封装成一个类，包含data、index、eof三个字段，按照报文段的 index 大小对比重载 < 运算符。

## TCPSender

`TCPsender`负责接收对方发送的`TCPsegment`中的ack号和接收窗口大小，应用层通过socket将字节流写入`TCPsender`中的`ByteStream`，`TCPsender`根据接收到的`ackno`和window size，从`ByteStream`中读取出来，将`ByteStream`中的字节流转化为连续的`TCPsegment`，发送给对方。

TCPSender’s responsibility：

- 追踪TCPReceiver的接收窗口（处理收到的ackno和windowsize）
- 通过从ByteStream读取，创建新的TCPsegment（如果需要的话，包括SYN和FIN标志）并发送，**尽可能地填满接收窗口**。**只有当接收窗口已满或TCPsender的BYteStream为空时，TCPsender才能停止发送segments**。
- 追踪已发送但是没有收到ackno的segments（称为outstanding segments），超时之后重新发送这些segments。

TCPsender需要维护一个数据结构，存储已发送未确认的TCPsegment集合（也可以称为**发送窗口**），如果**最早发送的segment超时还没有被确认，就重传**。

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



## TCPConnection

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210830152928395.png" alt="image-20210830152928395"  />

TCPConnection需要遵守的基本规则：

- 接收segments。

  当TCPConnection的`segment_received`方法被调用时，TCPConnection从网络中接收TCPsegments。此时，TCPConnection查看接收到的segment：

  - 如果设置了RST（reset）标志
    - 将输入流和输出流设置为error state并永久终止连接。

  - 否则：
    - 将segment传给`TCPReceiver`，以查看输入段上的字段：seqno、syn、有效载荷和fin。
    - 如果segment设置了ack标志，则需要将segment中的ackno和window size传给`TCPSender`。
  - 如果收到的segment占据了至少一个序列号，TCPConnection需要确保至少发送一个segment回复，以获取ackno和window size的反馈。

- 发送segments

  - 每当TCPSender将segment push到输出队列时，设置如下字段：seqno、syn、有效载荷和fin。
  - 在发送segment之前，TCPConnection查看TCPReceiver中是否有ackno和window size，如果有，则设置segment的ACK标志位为1 。                                                                                

- TCPConnection 的tick方法会被OS周期性调用，TCPConnection通过此方法得知时间的流逝。当被调用时，TCPConnection需要：

  - 告诉TCPSender过去的时间
  - 如果连续重传次数超过上限（`TCPConfig::MAX_RETX_ATTEMPTS`），就放弃连接，并发送一个reset segment给对方
  - 必要时结束connection

### TCP连接结束

TCPConnection的一个重要功能是决定TCP 连接什么时候完全结束。当TCP连接完全结束时，停止对任何接收到的segment回复ackno，并且`active()`方法返回false 。

TCP连接有两种关闭的方法：

- 不干净的关闭：**TCPConnection发送或接收到一个首部字段中的RST标志位被设置的segment** 。这种情况下，inbound和outbound的ByteStream都处于error state，并且`active()`方法可以马上返回false 。

- 干净的关闭：在没有error的情况下关闭（`active()`=false）。**这种情况可以尽可能地保证两个字节流都完全可靠地交付到接收对等方**。

  **由于两将军问题，不可能保证对等方都能完全干净关闭连接**，但是可以非常接近。

  从一个对等设备的角度来看，对其与“远程”对等设备的连接进行干净关闭有四个先决条件，**条件1保证了输入流被读取干净了，条件2和3保证了输出流被对等方读取干净了**。条件4也是关于输入流的，保证了输入流的正常关闭。

  1. 输入流被完全确认（`StreamReassembler`为空）并且结束（收到了FIN）

  2. 输出流被应用层结束（调用ByteStream的`end_input()`方法），并且被完全发送出去（`ByteStream`为空），首部字段包括FIN的segment也被发送出去。

  3. 输出流被对等方完全确认（对方的`StreamReassembler`为空，实际上要求本地的`_outstanding_segments`为空）

  4. 本地TCPConnection需要让对等方满足条件3。有两种方式：

     - 选择A：在两个流都已经结束后 linger 一段时间：

       本地TCPConnection确认了整个输入流，但是难以确认对等端是否知道自己确认了整个输入流，即对等端是否收到ack（因为对等端不会对本地发送的ack进行ack ）。**如果不能确保对等端收到ack，也就不能确保对等端的`_outstanding_segments`为空，那么对等端就有可能不停地重传无法得到本地确认的segment，输入流永远无法关闭**。

       我们可以让本地的TCPConnection等待一段时间，如果对等端没有重传任何东西，那么就可以相信对等端收到了ack。

       具体地，**当一个连接满足条件1到条件3，并且在收到对等端最后一个segment后，等待了至少 初始重传超时时间（`_cfg.rt_timeout`）的十倍，才能断开**。

       这意味着TCPConnection需要保持alive一段时间，保持对本地端口号的独占声明并可能发送 acks 以响应传入的segment，即使在 TCPSender 和 TCPReceiver 完全完成其工作并且两个流都已经结束了。

     - 选择B：被动关闭

       如果在TCPConnection发送FIN之前，TCPConnection的输入流就结束了（收到了FIN），那么这个TCPConnection在两个流结束后不需要 linger 。（因为FIN在发送ack之后，所以FIN的seqno大于之前发送的ack，所以对方对FIN的确认，就相当于确认了之前发送的所有ack）

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210925203406767.png" alt="image-20210925203406767" style="zoom: 50%;" />

先发送FIN的端在两个流结束后需要TIME_WAIT，后发送FIN的端不需要TIME_WAIT 。在实际实现中，TIME_WAIT为segment最大生命周期（MSL）的两倍时间。

### 具体实现

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220304150135298.png" alt="image-20220304150135298" style="zoom:50%;" />

开始时，发送方处于CLOSED状态，接收方处于LISTEN状态。

第一个SYN可以携带data，第二个SYN不能携带data，第三次握手可以携带data。

主动连接时，应用层调用`connect()`方法，调用`fill_window()`方法，向某一对等端发送包含SYN=1，ACK=0的包，进入SYN_SENT状态。

- 当从网络中收到segment时，调用`segment_received()`方法，此时只接收SYN=1的包
  - 如果有效载荷不为空，不符合SYN，丢弃
  - 如果ACK=1，那么正常接收，发送SYN=0，ACK=1的包，进入ESTABLISHED状态。
  - 如果ACK=0，可能是双方同时尝试建立连接，receiver正常接收，发送SYN=1，ACK=1且**有效载荷为空**的包

被动连接时，调用`segment_received()`方法，只接收SYN=1，ACK=0的包，发送SYN=1，ACK=1且**有效载荷为空**的包，进入SYN_RECV状态。后续再在`segment_received()`方法中接收到含有ACK的包，就进入ESTABLISHED状态。

连接建立后：

- 应用层可以调用 `write`方法，向发送缓冲区（ByteStream）中写入想要发送的内容，再调用`_send_segments`方法，将其发送。

- 当从网络中收到segments时，调用`segment_received()`方法。将包的接收窗口大小（window size）、ackno交给sender，将其余的部分（seqno、有效载荷、FIN和SYN）交给receiver，由它们各自处理。
- 由操作系统周期性地调用`tick()`方法，指示时间的流逝，以对已发送未确认的最早的包进行超时重传。还需要检查连续重传的次数是否超过上限，如果超过，则需要调用`_unclean_shutdown`强制关闭连接。还需要调用`_send_segments`发送segments 。
- 由tick()方法时不时地调用`_send_segments`方法，如果TCPSender的`_segments_out`队列中有可以发送的segment，就设置它的ackno，window size以及ACK标志位，然后将其真正地发送出去。每次调用`_send_segments`方法都会调用`_clean_shutdown()`来判断是否需要关闭连接。

应用层可以调用 `end_input_stream`方法，结束向TCPConnection中写入，表明应用层**想要**干净地主动结束连接。 `end_input_stream`方法中调用`fill_window()`方法，和`_send_segments`方法，将ByteStream中的值发送出去。如果接收窗口空间不够，无法一次性将ByteStream发送完，此时只能等`segment_received()`接收到对方的新的包，调用sender的`ack_received()`方法，更新window size，再调用`fill_window()`方法，将sender的发送缓冲区（ByteStream）变成空的，再调用`_send_segments`方法，才能发送FIN。最后调用`_clean_shutdown()`来判断是否需要关闭连接。

当TCP连接还处于active状态时，TCPConnection对象被析构，此时会导致TCPConnection的异常中断。在析构函数的内部调用`_unclean_shutdown`函数，该函数直接强制关闭连接：将输入流和输出流设置为错误状态，TCPConnection的状态马上变为false，然后向对等方发送包含RST的segment。

`tcp_connection.hh`

```cpp
#ifndef SPONGE_LIBSPONGE_TCP_FACTORED_HH
#define SPONGE_LIBSPONGE_TCP_FACTORED_HH

#include "tcp_config.hh"
#include "tcp_receiver.hh"
#include "tcp_sender.hh"
#include "tcp_state.hh"

//! \brief A complete endpoint of a TCP connection
class TCPConnection {
  private:
    TCPConfig _cfg;
    // 初始化TCPReceiver
    TCPReceiver _receiver{_cfg.recv_capacity};
    // 初始化TCPSender
    TCPSender _sender{_cfg.send_capacity, _cfg.rt_timeout, _cfg.fixed_isn};
    // TCPConnection准备发送的TCPsegment的输出队列
    // TCPSender中 _segments_out 中的segment不包括ackno和window size，不是TCPConnection真正发送出去的TCPsegment,
    // 需要与TCPReceiver的ackno()和window_size()的返回值结合在一起，再放入到TCPConnection 中的 _segments_out 中。
    // 我们可以认为放入此队列的segment已经发送出去了，但是实际上需要由拥有者或OS将该队列中的segment pop出，然后一一
    // 封装进更底层的IP数据报或UDP，此时才真正发送出去了
    std::queue<TCPSegment> _segments_out{};

    // TCPConnection是否alive
    bool _active{true};
    // TCPConnection是否应该在两个流结束后保持alive 10 * _cfg.rt_timeout的时间
    bool _linger_after_streams_finish{true};
    
    bool _time_since_last_segment_received{0};

    // 对TCPSender的 _segments_out中的segment设置首部的ackno和windowsize字段
    // 再加入到TCPConnection的 _segments_out，真正地将TCPsegment发送出去
    void send_sender_segments();

    void clean_shutdown();
    void unclean_shutdown();
  public:
    //! 提供给应用层writer的输入接口

    // 通过发送SYN segment 初始化连接
    void connect();

    // 应用层向输出流写入数据，如果有可能的话通过TCP发送
    // 返回实际写入的数据的字节数
    size_t write(const std::string &data);

    // 返回应用层可以向ByteStream写入的字节数,即ByteStream的空闲空间大小
    //! \returns the number of `bytes` that can be written right now.
    size_t remaining_outbound_capacity() const;

    // 结束向TCPConnection中写入，也就是关闭输出流（仍然允许读取输入的数据）
    //! \brief Shut down the outbound byte stream (still allows reading incoming data)
    void end_input_stream();

    // 提供给应用层reader的输出接口
    //! \name "Output" interface for the reader
    //!@{

    // 已经从对等端收到的输入字节流
    //! \brief The inbound byte stream received from the peer
    ByteStream &inbound_stream() { return _receiver.stream_out(); }
    //!@}

    //! \name Accessors used for testing

    //!@{
    //! \brief number of bytes sent and not yet acknowledged, counting SYN/FIN each as one byte
    size_t bytes_in_flight() const;
    //! \brief number of bytes not yet reassembled
    size_t unassembled_bytes() const;
    //! \brief Number of milliseconds since the last segment was received
    size_t time_since_last_segment_received() const;
    //!< \brief summarize the state of the sender, receiver, and the connection
    TCPState state() const { return {_sender, _receiver, active(), _linger_after_streams_finish}; };
    //!@}

    // 由拥有者或操作系统调用的方法
    //! \name Methods for the owner or operating system to call
    //!@{

    // 当从网络中接收到一个segment时调用
    //! Called when a new segment has been received from the network
    void segment_received(const TCPSegment &seg);

    // 当时间流逝时，周期性调用
    //! Called periodically when time elapses
    void tick(const size_t ms_since_last_tick);

    // 返回TCPConnection想要输出的TCPSegment队列。
    // 拥有者或OS会将此队列中的segment全部出队, 然后将每一个TCPSegment
    // 放入低层次的数据报中（通常是IP数据包，也可以是UDP）,并真正地发送出去
    std::queue<TCPSegment> &segments_out() { return _segments_out; }

    // 连接是否alive,返回true如果流还在运行，或者流结束后TCPConnection在linger
    bool active() const;
    //!@}

    // 从_cfg配置构造一个TCPConnection
    explicit TCPConnection(const TCPConfig &cfg) : _cfg{cfg} {}

    //! \name 构造函数和析构函数
    // 允许移动，不允许拷贝，不允许默认构造

    //!@{
    // 如果连接还存活析构函数就发送一个RST
    ~TCPConnection();  
    TCPConnection() = delete;
    TCPConnection(TCPConnection &&other) = default;
    TCPConnection &operator=(TCPConnection &&other) = default;
    TCPConnection(const TCPConnection &other) = delete;
    TCPConnection &operator=(const TCPConnection &other) = delete;
    //!@}
};

#endif  // SPONGE_LIBSPONGE_TCP_FACTORED_HH

```

`tcp_connection.cc`

```cpp
#include "tcp_connection.hh"

#include <iostream>

using namespace std;

size_t TCPConnection::remaining_outbound_capacity() const { return _sender.stream_in().remaining_capacity(); }

size_t TCPConnection::bytes_in_flight() const { return _sender.bytes_in_flight(); }

size_t TCPConnection::unassembled_bytes() const { return _receiver.unassembled_bytes(); }

size_t TCPConnection::time_since_last_segment_received() const { return _time_since_last_segment_received;}

bool TCPConnection::active() const { return _active; }

// 由操作系统调用，接收从UDP或IP数据报中的解封装的TCPsegment
void TCPConnection::segment_received(const TCPSegment &seg) { 
    // 如果连接断开了，不接收任何segment
    if(!_active){
        return;
    }
    // 接收到一个segment，重置计数
    _time_since_last_segment_received = 0;
    // 被动建立连接的一方可能处于的状态,处于listen状态
    // 没有收到过任何segment，也没有发送过任何segment,
    if(!_receiver.ackno().has_value() && _sender.next_seqno_absolute() == 0){
        // 只接收syn
        if(!seg.header().syn){
            return;
        }
        _receiver.segment_received(seg);
        // 收到对方的syn，就发送SYN与对方建立连接，处于SYN_RECV状态
        // 三次握手的阶段二
        connect();
        return;
    }
    // 主动建立连接的一方可能处于的状态,处于SYN_SENT状态，三次握手的阶段一
    // 发送出去的流没有得到确认，也没有收到过对方的segment。
    if(_sender.next_seqno_absolute() > 0 && _sender.bytes_in_flight() == _sender.next_seqno_absolute() && 
       !_receiver.ackno().has_value()){
        // 如果有效载荷不为0，不符合SYN，直接丢弃
        if(seg.payload().size() ){
            return;
        }
        // 如果ack等于0，则双方同时发起了建立连接
        if(!seg.header().ack){
            if(seg.header().syn){
                _receiver.segment_received(seg);
                // 发送空的segment，以返回ack
                _sender.send_empty_segment();
            }
            return;
        }
        // 如果syn=1，ack=1，rst=1，则关闭连接
        if(seg.header().rst){
            _receiver.stream_out().set_error();
            _sender.stream_in().set_error();
            _active = false;
            return;
        }
    }
    // 如果syn=1，ack=1，rst!=1，或者其他情况
    _receiver.segment_received(seg);
    _sender.ack_received(seg.header().ackno,seg.header().win);
    // 发送确认的报文，进入ESTABLISHED状态，连接建立。处于三次握手的第三阶段
    if (_sender.stream_in().buffer_empty() && seg.length_in_sequence_space())
        _sender.send_empty_segment();
    if (seg.header().rst) {
        _sender.send_empty_segment();
        unclean_shutdown();
        return;
    }
    send_sender_segments();
}

size_t TCPConnection::write(const string &data) {
    if(data.size() == 0){
        return 0;
    }
    // 向TCPSender的ByteStream中写入数据
    size_t write_size = _sender.stream_in().write(data);
    _sender.fill_window();
    // 对TCPSender中的segment设置ackno和windowsize,再发送给对等端
    send_sender_segments();
    return write_size;
}

// 此方法被OS周期性调用
void TCPConnection::tick(const size_t ms_since_last_tick) {
    if(!_active){
        return;
    }
    _time_since_last_segment_received += ms_since_last_tick;
    // 告知TCPSender过去的时间
    _sender.tick(ms_since_last_tick);
    // 如果连续重传的次数超过上限，则强制关闭连接
    if(_sender.consecutive_retransmissions() > TCPConfig::MAX_RETX_ATTEMPTS){
        unclean_shutdown();    
    }
    send_sender_segments();
}

// 结束向TCPConnection中写入，也就是关闭输出流（仍然允许读取输入的数据）
void TCPConnection::end_input_stream() {
    _sender.stream_in().end_input();
    // 发送fin，不能保证这一次能将fin发送出去，因为接收窗口有可能空间不够，ByteStream无法全部发送出去
    _sender.fill_window();
    send_sender_segments();
}

// 主动连接
void TCPConnection::connect() {
    _sender.fill_window();
    send_sender_segments();
}

// 对TCPSender的 _segments_out中的segment设置首部的ackno和windowsize字段，还有ACK标志位
// 再加入到TCPConnection的 _segments_out，真正地将TCPsegment发送出去
void TCPConnection::send_sender_segments(){
    // 此处必须要是引用类型，才能指向_sender中的同一个成员变量,才能对其进行操作
    // std::queue<TCPSegment>&sender_segs_out = _sender.segments_out();

    // 对TCPSender的 _segments_out进行遍历，将所有的segment的头部都加上ackno和windowsize
    // 再发送出去
    while(!_sender.segments_out().empty()){
        TCPSegment seg = _sender.segments_out().front();
        _sender.segments_out().pop();
        // 只有当ackno()的返回值非空时，才需要加上
        if(_receiver.ackno().has_value()){
            seg.header().ack = true;
            seg.header().ackno = _receiver.ackno().value();
            seg.header().win = _receiver.window_size();
        }
        // 将segment真正发送出去
        _segments_out.push(seg);
    }
    // 每次发送segment后，都需要判断是否需要干净关闭连接
    clean_shutdown();
    
}
// 不干净的关闭，直接强制关闭连接
// 将输入输出流设置为错误状态
// 将连接的active置为false，向对等方发送rst
void TCPConnection::unclean_shutdown(){
    _receiver.stream_out().set_error();
    _sender.stream_in().set_error();
    _active = false;
    TCPSegment seg = _sender.segments_out().front();
    _sender.segments_out().pop();
    seg.header().ack = true;
    if(_receiver.ackno().has_value()){
        seg.header().ackno = _receiver.ackno().value();
    }
    seg.header().win = _receiver.window_size();
    seg.header().rst = true;
    _segments_out.push(seg);

}
// 干净关闭连接，判断能否干净地关闭连接，
// 判断是否需要在两个流结束后linger一段时间
void TCPConnection::clean_shutdown(){
    // 如果receiver已经收到了对等端的fin，StreamReassembler为空
    if(_receiver.stream_out().input_ended()){
        // 如果sender的输出流还没有结束，即ByteStream不为空，fin还没有发送出去
        // 那么需要在两个流结束后linger一段时间
        if(!_sender.stream_in().eof()){
            _linger_after_streams_finish = false;
        // 如果sender发送了fin，且得到了确认
        }else if(_sender.bytes_in_flight() == 0){
            // 那么只有不需要linger或者linger了指定时间后，才能断开连接
            if(!_linger_after_streams_finish || time_since_last_segment_received() >= 10 * _cfg.rt_timeout){
                _active = false;
            }
        }
    }
}
TCPConnection::~TCPConnection() {
    try {
        if (active()) {
            cerr << "Warning: Unclean shutdown of TCPConnection\n";
            _sender.send_empty_segment();
            unclean_shutdown();
            // Your code here: need to send a RST segment to the peer
        }
    } catch (const exception &e) {
        std::cerr << "Exception destructing TCP FSM: " << e.what() << std::endl;
    }
}

```


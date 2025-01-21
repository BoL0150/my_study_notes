# CS144实验记录（五）：lab4

在lab 4中，我们将创建总体模块，称为TCP connection，该模块将TCPSender和TCPReceiver结合起来。

我们的TCP segment可以封装到用户(TCP-In-UDP)或IP(TCP/IP)数据报的有效载荷中。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210923164740632.png" alt="image-20210923164740632" style="zoom: 80%;" />

本lab提供了代码支持从用户数据包或IP数据报中读取或写入TCPsegment，还提供了`CS144TCPSocket`类，将我们的TCPConnection包装，使它表现得像一个普通的流套接字，就像在lab 0中用来实现webget的TCPSocket一样。

我们需要做的是将之前已经实现的`TCPSender`和`TCPReceiver`结合成一个对象（`TCPConnection`），并处理一些连接全局的管理任务。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210923191514721.png" alt="image-20210923191514721" style="zoom: 67%;" />

TCPsegment的报文格式：

- ACK标志位用于指示确认字段中的值是有效的，即该报文段包括对一个已被成功接收报文段的确认。在TCP中，只有第一次连接请求的ACK等于0，其余的ACK都为1.
- RST表示复位，表示TCP连接中出现异常必须**强制**断开连接。发送RST包关闭连接时，不必等缓冲区的包都发出去（不像上面的FIN包），直接就丢弃缓存区的包发送RST包。而接收端收到RST包后，也不必发送ACK包来确认。
- SYN同步： 表示开始会话请求，用来发起一个连接，建立连接。SYN为1表示希望建立连接，并在其序列号的字段进行序列号初始值的设定。（Synchronize本身有同步的意思。也就是意味着建立连接的双方，序列号和确认应答号要保持同步） 

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210923200351007.png" alt="image-20210923200351007" style="zoom:67%;" />

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

TCP连接是如何实际发送一个段？

- TCPConnection发送segment类似于TCPSender，将segment push到_segments_out队列中即可。**一旦我们将TCPsegment推送到此队列，我们就认为它已经发送**。很快，所有者会对它进行pop（使用公共方法`segments_out()`访问该队列），并真正地发送它。

当TCPConnection收到带有RST标志位的segment该做什么？

- 如果TCPConnection收到（或发送）一个RST标志位被置位的segment，表示连接的**立即终止**。我们应该设置inbound 和 outbound的ByteStream的error flag，并且**后续任何对`TCPConnection::active()`的调用都应该返回false** 。

遇到以下两种情况TCPConnection需要**发送**RST，即放弃整个连接：

- 连续重传次数超过上限（`TCPConfig::MAX_RETX_ATTEMPTS`）
- 在连接还是active时（`active()`函数返回true），TCPConnection的destructor被调用。

如何制作一个可以设置RST标志位的segment：

- 可以通过调用`send_empty_segment()`方法生成一个正确seqno的空segment，将其RST标志位置位即可。

ACK标志位的作用是什么？

- ACK标志位用于指示确认字段中的值（即ackno和window size）是有效的，即该报文段包括对一个已被成功接收报文段的确认。在TCP中，只有第一次连接请求的ACK等于0，其余的ACK都为1.
  - 对于TCPSender输出的segments，当TCPReceiver的`ackno()`方法返回的`std::optional<WrappingInt32>`非空时（使用 has_value()方法测试），就可以设置发送segment中的ackno和window size，并将ACK标志位置为1
  - 对于TCPConnection接收的segments，当ACK标志位为1时，我们可以查看ackno，并将ackno和window size传递给TCPSender

### 连接建立流程

TCPSender的接口：

```cpp
    //接收TCPReceiver返回的ackno和window size
    void ack_received(const WrappingInt32 ackno, const uint16_t window_size);
    //!生成一个负载为空的segment（用于创建空的 ACK segment）
    void send_empty_segment();
    //!创建并发送segments以尽可能多地填充接收窗口
    void fill_window();
    //!通知 TCPSender 时间的流逝
    void tick(const size_t ms_since_last_tick);
    //! 已发送未确认的字节数是多少，SYN和FIN也占一个字节
```

TCPReceiver的接口：

```cpp
	//向远程TCPsender提供反馈
    //返回包含ackno的optional<WrappingInt32>，如果ISN还没有初始化（即没有收到SYN）则返回空。 ackno是接收窗口的开始，或者说接收者没有接收到的流中第一个字节的序列号 
    std::optional<WrappingInt32> ackno() const;

    //应该发送给对方的接收窗口大小
    //capacity减去 TCPReceiver 在其BYteStream中保存的字节数（那些已重组但被应用层读取的字节数）。
    //也相当于first unassembled索引（即ackno）和first unacceptable索引的距离
    size_t window_size() const;

    //处理接收到的TCPsegment
    void segment_received(const TCPSegment &seg);
```

客户端：

- 客户端先调用TCPSender中的`fill_window()`方法，检查类内维护的变量`_syn_sent`，没有发送过SYN，所以首先生成一个SYN=1的segment，此segment的seqno为ISN（客户端的）。并且检测到计时器没有启动，还要启动计时器。

  ```cpp
  void TCPSender::fill_window() {
  //如果SYN没有发送，则发送SYN后返回
      if(!_syn_sent){
          TCPSegment seg;
          _syn_sent = true;
          seg.header().syn = true;
          _send_segments(seg);
          return;
      }
  }
  ```

- 再调用TCPReceiver中的`ackno()`方法，检查类内维护的变量`_syn`，没有收到过SYN，所以返回的`std::optional<WrappingInt32>`为空。

  ```cpp
  optional<WrappingInt32> TCPReceiver::ackno() const { 
      //如果没有收到syn，就返回空
      if(!_syn)return {};
  }
  ```

- 所以将segment中的ACK标志位置为0，ackno和window size都不需要填写。将该segment发送出去

服务器：

- 调用TCPReceiver中的`segment_received()`方法，接收segment。记录客户端的ISN，以及收到了SYN。
- 检测到ACK=0，所以不需要调用TCPSender中的`ack_received()`方法。
- 调用TCPSender中的`fill_window()`方法，检测到服务器端没有发送过SYN，所以生成一个SYN=1的segment，此segment的seqno为ISN（服务器端的）。并且检测到计时器没有启动，还要启动计时器。
- 再调用TCPReceiver中的`ackno()`方法，检查类内维护的变量`_syn`，收到过SYN，返回的`std::optional<WrappingInt32>`为非空。
- 所以将segment中的ACK标志位置为1，ackno和window size填写为调用TCPReceiver中的`ackno()`和`window_size()`方法的返回值。将该segment发送出去

收到segment时，将segment作为参数调用TCPReceiver中的`segment_received()`，检查segment中的ACK标志位是否为1，如果是，则将segment中的ackno和window_size提取出来，作为参数调用TCPSender中的`ack_received()` 。

发送segment时，调用TCPSender中的`fill_window()`方法，再与TCPReceiver中的`ackno()`和`window_size()`的方法的结果（如果收到过SYN，有结果的话）结合，生成完整的segment，发送出去。

### 连接结束流程

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210830152928395.png" alt="image-20210830152928395"  />

**客户端断开连接的流程**：

输出流的结束：

- 如果应用层决定将输出流结束（调用ByteStream的`end_input()`方法），并且等待输出的缓冲区为空，表明整个输出流的结束，即eof，此时才可以向对方发送FIN。

  ```cpp
  void TCPSender::fill_window() {
      //如果字节流结束了，没有发送FIN，且发送窗口有空余空间，则发送FIN
      //注意，发送结束的标志并不只是ByteStream为空，还需要应用层结束输入
      if(_stream.eof() && _receiver_free_space >= 1){
          TCPSegment seg;
          seg.header().fin = true;
          _fin_sent = true;
          _send_segments(seg);
          return;
      }
  }
  ```

输入流的结束：

输入流只能被动结束：

- 对TCPConnection来说，收到了FIN 并且 等待被重组的缓冲区为空表明**对等方的输入结束**（收到FIN并不代表输入结束，有可能发生乱序）

  ```cpp
  void StreamReassembler::push_substring(const string &data, const size_t index, const bool eof) {
      segment seg={index,data,eof};
      // 收到FIN且_segs_to_be_reassembled为空
      if(_segs_to_be_reassembled.empty() == true && _eof == true)
          // 对等方的输入结束
          _output.end_input();
     }
  ```

- 对应用层来说，对等方的输入结束 并且 等待被读取的缓冲区为空 表明**整个输入流的结束（eof）**。

  ```cpp
  //缓冲区为空并且输入结束代表eof，接收端才停止读取
  bool ByteStream::eof() const {
      return buffer_empty()&&input_ended();
  }
  ```

如果是本地端先结束连接，则应用层可以随便什么时候`end_input()`；如果是本地端先接收到对等端输入的结束（FIN进入`ByteStream`），则应用层需要结束输出流，调用`end_input()`，停止往ByteStream中写入内容，等待ByteStream空了之后，发送FIN。

**只要发送了FIN，就不再对对等端任何的非FIN的segment进行回复确认**，不管这个FIN对方有没有收到。



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

![image-20211114131357882](https://raw.githubusercontent.com/BoL0150/image2/master/image-20211114131357882.png)

#### 连接结束的实现总结

TCPConnection中有一个成员变量叫`_linger_after_streams_finish`，通过`state()`方法暴露于测试中。此变量的初始值为true。如果输入流先结束（收到了FIN，并且StreamReassembler为空），TCPConnection的输出流才达到EOF（应用层调用ByteStream的`end_input()`方法结束输出，并且ByteStream为空），此变量被设为false。

在满足先决条件1到3的任何时候，如果`_linger_after_streams_finish`为 false，则连接结束（并且 `active()` 应返回 false）。 否则，连接需要在收到最后一个segment后 `10 *_cfg.rt_timeout`时间过去后才结束。

### 具体实现

我实现的TCP由两个部分组成：发送方TCPSender，接收方TCPReceiver。

- TCPSender负责：
  - 处理接收到的segment的ackno和windowsize，追踪对方的接收窗口，实现流量控制
  - TCPSender内部维护一个队列作为缓冲区Bytestream，存放应用层写入但是还没有发送出去的字节流。TCPSender从Bytestream中读取数据，将数据在字节流中的64位索引转换成TCP头部的32位索引，加上SYN和FIN标志位，组成TCPSegment。
  - TCPSender内部还维护了一个队列，追踪已发送但是还没有收到ackno的segment。TCPSender采用累积确认的方式，当收到ackno时，就将这个队列中所有seqno小于这个ackno的segment移除。
  - 内部还有一个计时器，计算从上次接收到有效ackno到现在过了多长时间，超时之后重新发送队列中最老的segment。还需要跟踪连续重传的次数，每次连续重传，就将超时重传的时间间隔翻倍；连续重传超过一定次数后直接断开连接。当收到有效ackno后就重启计时器。
- TCPReceiver负责：
  - 处理接收到的segment，将TCP头部中32位的seqno转换成64位的流索引，处理SYN和FIN标志位
  - 内部维护一个队列作为接收缓冲区，存放有序的，等待应用层读取的字节流
  - 内部维护一个集合，作为流重组器，记录乱序到来的等待重组的segment。TCPReceiver将刚才收到的segment、FIN标志位和segment对应的流索引传入流重组器中。根据流索引将segment切割、去重、合并。然后再将流重组器中所有与Bytestream缓冲区连续的子串加入Bytestream中。
  - 根据本地的接收窗口，计算出本地的ackno和window size。

所以当TCPConnection从网络中收到segments时，调用`segment_received()`方法。将包的接收窗口大小（window size）、ackno交给sender，将其余的部分（seqno、有效载荷、FIN和SYN）交给receiver，由它们各自处理。

由操作系统周期性地调用`tick()`方法，告诉TCPSender过去的时间，以对已发送未确认的最早的包进行超时重传。定时器还需要检查连续重传的次数是否超过上限，如果超过，则需要调用`_unclean_shutdown`发送RST来强制关闭连接。

定时器还需要时不时地调用`_send_segments`方法，如果TCPSender的`_segments_out`队列中有可以发送的segment，就从receiver中获取ackno，window size以及ACK标志位，然后将其真正地发送出去。**每次调用`_send_segments`方法都会调用`_clean_shutdown()`来判断是否需要关闭连接**。

- 如果receiver收到了对方的fin，并且流重组器为空，则说明需要关闭连接：
  - 如果sender之前没有发送过fin，则说明自己发送fin后不需要TIME_WAIT
  - 如果sender已经发送了fin，那么只有不需要TIME_WAIT或者TIME_WAIT了指定时间后才能断开连接



开始时，发送方处于CLOSED状态，接收方处于LISTEN状态。

第一个SYN可以携带data，第二个SYN不能携带data，第三次握手可以携带data。

主动连接时，应用层调用`connect()`方法，调用`fill_window()`方法，向某一对等端发送包含SYN=1，ACK=0的包，进入SYN_SENT状态。

- 当从网络中收到segment时，调用`segment_received()`方法，此时只接收SYN=1的包
  - 如果有效载荷不为空，不符合SYN，丢弃
  - 如果ACK=1，那么正常接收，发送SYN=0，ACK=1的包，进入ESTABLISHED状态。
  - 如果ACK=0，可能是双方同时尝试建立连接，receiver正常接收，发送SYN=1，ACK=1且**有效载荷为空**的包

被动连接时，调用`segment_received()`方法，只接收SYN=1，ACK=0的包，发送SYN=1，ACK=1且**有效载荷为空**的包，进入SYN_RECV状态。后续再在`segment_received()`方法中接收到含有ACK的包，就进入ESTABLISHED状态。

> 连接建立后：
>
> - 应用层可以调用 `write`方法，向发送缓冲区（ByteStream）中写入想要发送的内容，再调用`_send_segments`方法，将其发送。
> - 当从网络中收到segments时，调用`segment_received()`方法。将包的接收窗口大小（window size）、ackno交给sender，将其余的部分（seqno、有效载荷、FIN和SYN）交给receiver，由它们各自处理。
> - 由操作系统周期性地调用`tick()`方法，指示时间的流逝，以对已发送未确认的最早的包进行超时重传。还需要检查连续重传的次数是否超过上限，如果超过，则需要调用`_unclean_shutdown`发送RST来强制关闭连接。还需要调用`_send_segments`发送segments 。
> - 由tick()方法时不时地调用`_send_segments`方法，如果TCPSender的`_segments_out`队列中有可以发送的segment，就设置它的ackno，window size以及ACK标志位，然后将其真正地发送出去。**每次调用`_send_segments`方法都会调用`_clean_shutdown()`来判断是否需要关闭连接**。
>
> 应用层可以调用 `end_input_stream`方法，结束向TCPConnection中写入，表明应用层**想要**干净地主动结束连接。 `end_input_stream`方法中调用`fill_window()`方法，和`_send_segments`方法，将ByteStream中的值发送出去。如果接收窗口空间不够，无法一次性将ByteStream发送完，此时只能等`segment_received()`接收到对方的新的包，调用sender的`ack_received()`方法，更新window size，再调用`fill_window()`方法，将sender的发送缓冲区（ByteStream）变成空的，再调用`_send_segments`方法，才能发送FIN。最后调用`_clean_shutdown()`来判断是否需要关闭连接。
>
> 当TCP连接还处于active状态时，TCPConnection对象被析构，此时会导致TCPConnection的异常中断。在析构函数的内部调用`_unclean_shutdown`函数，该函数直接强制关闭连接：将输入流和输出流设置为错误状态，TCPConnection的状态马上变为false，然后向对等方发送包含RST的segment。
>

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

## 同时打开：

两个应用程序同时彼此执行主动打开的情况是可能的，但是发生的可能性极小。每一方必须发送一个SYN，且这些SYN必须传递给对方。这需要每一方使用一个对方熟知的端口作为本地端口。这又称为同时打开（ simultaneous open）。两端必须几乎在同时启动，以便收到彼此的SYN。只要两端有较长的往返时间就能保证这一点。

TCP是特意设计为了可以处理同时打开，对于同时打开它**仅建立一条连接而不是两条连接**（其他的协议族，最突出的是OSI运输层，在这种情况下将建立两条连接而不是一条连接）出现同时打开的情况时，两端几乎在同时发送SYN，并进入 SYN_SENT状态。当每一端收到SYN时，状态变为SYN_RCVD，同时它们都再发SYN并对收到的SYN进行确认。当双方都收到SYN及相应的ACK时，状态都变迁为ESTABLISHED。

![image-20220307234317939](https://raw.githubusercontent.com/BoL0150/image2/master/image-20220307234317939.png)



同时打开的连接需要交换4个报文段，比正常的三次握手多一个。此外，要注意的是我们没有将任何一端称为客户或服务器，因为每一端既是客户又是服务器。

## 同时关闭

我们在以前讨论过一方（通常但不总是客户方）发送第一个 FIN执行主动关闭。双方都执行主动关闭也是可能的，TCP协议也允许这样的同时关闭（ simultaneous close）。

当应用层发出关闭命令时，两端均从ESTABLISHED变为FIN _ WAIT_1。
 这将导致双方各发送一个 FIN，两个FI N经过网络传送后分别到达另一端。收到FIN后，状态由F I N_WAIT_1变迁到 C L O S I N G，并发送最后的ACK。当收到最后的 ACK时，状态变化为TIME _ WAIT。

**同时关闭与正常关闭使用的段交换数目相同。**

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220307234343734.png" alt="image-20220307234343734" style="zoom:50%;" />


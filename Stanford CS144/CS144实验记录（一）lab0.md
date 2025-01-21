# CS144实验记录（一）lab0

## 环境搭建

按照官网要求搭建环境

## 2.1 Fetch a Web page

使用`Telnet`获取http://cs144.keithw.org/hello页面的内容

1. 在虚拟机中，输入 `telnet cs144.keithw.org http`，This tells the telnet program to open a reliable byte stream between your computer and another computer (named cs144.keithw.org), and with a particular service running on that computer: the “http” service
2. Type `GET /hello HTTP/1.1` . This tells the server the path part of the URL.
3. Type Host: `cs144.keithw.org` . This tells the server the host part of the URL.
4. Type `Connection: close` . This tells the server that you are finished making requests, and it should close the connection as soon as it finishes replying.
5. **Hit the Enter key one more time**: . This sends an empty line and tells the server that you are done with your HTTP request.

完成后，就可以在终端上看到与在浏览器中访问该页面一样的信息。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210809123436396.png" alt="image-20210809123436396" style="zoom:67%;" />

## 2.3 Listening and connecting

我们已经看到Telnet是什么：与在其他计算机上运行的程序建立连接的客户端程序。我们还可以使用`netcat`成为一个简单的服务器（即等待客户端连接到它的程序）

1. In one terminal window, run `netcat -v -l -p 9090` on your VM. You should see:
2. Leave netcat running. In another terminal window, run `telnet localhost 9090`
3. If all goes well, the netcat will have printed something like “Connection from localhost 53500 received!”.
4. 现在我们可以在任意一个终端中输入文字，按下回车后，另一个终端就将其会打印在屏幕上

![image-20210809130652558](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210809130652558.png)

## 3 Writing a network program using an OS stream socket

In this lab, you will simply use the operating system’s pre-existing support for the Transmission Control Protocol. **You’ll write a program called “webget” that creates a TCP stream socket, connects to a Web server, and fetches a page**—much as you did earlier in this lab.

In particular, we would like you to:

- Use the language documentation at https://en.cppreference.com as a resource
- Never use `malloc()` or `free()`.
- Never use `new` or `delete`.
- Essentially never use raw pointers (*), and use “smart” pointers (`unique_ptr` or `shared_ptr`) only when necessary. (You will not need to use these in CS144.)
- Avoid templates, threads, locks, and virtual functions. (You will not need to use these in CS144.)
- Avoid C-style strings (`char *str`) or string functions (`strlen()`, `strcpy()`). These are pretty error-prone. Use a `std::string` instead.
- Never use C-style casts (e.g., `(FILE *)x`). Use a C++ `static_cast` if you have to (you generally will not need this in CS144).
- Prefer passing function arguments by `const` reference (e.g.: `const Address & address`).
- Make every variable `const` unless it needs to be mutated.
- Make every method `const` unless it needs to mutate the object
- Avoid global variables, and give every variable the smallest scope possible. 
- Before handing in an assignment, please run `make format` to normalize the coding style.

### Writing webget

实现webget，这是一个使用操作系统的TCP支持和流套接字抽象在Internet上提取网页的程序，就像上面使用telnet进行的操作一样。要求使用提供的`TCPSocket`和`Address`类。

实际上就是创建一个TCP套接字然后与目标主机建立连接，然后构造http的GET请求，通过socket发送出去，socket再接收响应并打印出来。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210719132221521.png" alt="image-20210719132221521" style="zoom:200%;" />

```c
void get_URL(const string &host, const string &path) {
    // Your code here.

    //创建socket，与目标IP和端口建立连接
    TCPSocket sock;
    //这里的host是主机名或IP地址，如果是主机名，则自动执行nslookup从而找到IP地址
    sock.connect(Address(host,"80"));
    std::string send,recv;
    //构造http的GET请求
    send="GET "+path+" HTTP/1.1\r\n"+"Host:"+host+"\r\n"+"Connection:close\r\n\r\n";
    //发送GET请求
    sock.write(send);
    //获取响应并打印出来
    while(sock.eof()==false){
        sock.read(recv);
        std::cout<<recv;
    }
    //关闭socket
    sock.close();

}
```

## 4 An in-memory reliable byte stream

要求实现一个有序字节流类，使之支持读写、容量控制，**实际上就是类似于TCP的流量控制的缓冲区**。这个字节流类似于一个带容量的队列，从一头读，从另一头写。当流中的数据达到容量上限时，便无法再写入新的数据。特别的，读操作被分为了peek和pop两步。peek为从头部开始读取指定数量的字节，pop为弹出指定数量的字节。

要求：

- 在输入端写入的字节以相同的顺序从输出端读出来
- 字节流是有限的，发送方可以结束输入，然后就没有字节可以继续被写入了

- 当接收方读取到字节流的结束，会遇到EOF，然后就没有字节可以继续被读取了

数据结构：由于`queue`只能访问开头的节点，所以使用`deque<char>`实现。将输入的`string`中的char一个个push到deque中，接收端再从deque中读取相应长度的char。

`byte_stream.hh`

```c++
class ByteStream {
  private:
    bool _error=false;  //!< Flag indicating that the stream suffered an error.
    size_t _capacity = 0;
    std::deque<char>_buffer={};
    bool _input_ended_flag=false;
    size_t _write_count=0;
    size_t _read_count=0;
    
 public:
    //! Construct a stream with room for `capacity` bytes.
    ByteStream(const size_t capacity);

    //! \name "Input" interface for the writer
    //!@{

    //! Write a string of bytes into the stream. Write as many
    //! as will fit, and return how many were written.
    //! \returns the number of bytes accepted into the stream
    size_t write(const std::string &data);

    //! \returns the number of additional bytes that the stream has space for
    size_t remaining_capacity() const;

    //! Signal that the byte stream has reached its ending
    void end_input();

    //! Indicate that the stream suffered an error.
    void set_error() { _error = true; }
    //!@}

    //! \name "Output" interface for the reader
    //!@{

    //! Peek at next "len" bytes of the stream
    //! \returns a string
    std::string peek_output(const size_t len) const;

    //! Remove bytes from the buffer
    void pop_output(const size_t len);

    //! Read (i.e., copy and then pop) the next "len" bytes of the stream
    //! \returns a string
    std::string read(const size_t len);

    //! \returns `true` if the stream input has ended
    bool input_ended() const;

    //! \returns `true` if the stream has suffered an error
    bool error() const { return _error; }

    //! \returns the maximum amount that can currently be read from the stream
    size_t buffer_size() const;

    //! \returns `true` if the buffer is empty
    bool buffer_empty() const;

    //! \returns `true` if the output has reached the ending
    bool eof() const;
    //!@}

    //! \name General accounting
    //!@{

    //! Total number of bytes written
    size_t bytes_written() const;

    //! Total number of bytes popped
    size_t bytes_read() const;
    //!@}

}
```

`byte_stream.cc`

```c++
ByteStream::ByteStream(const size_t capacity):_capacity(capacity) { }

size_t ByteStream::write(const string &data) {
    size_t len=data.length();
    //输入的长度不能超过缓冲空闲空间的大小
    if(len>_capacity-_buffer.size())
        len=_capacity-_buffer.size();
    //记录输入的总字节数数
    _write_count+=len;
    //将string中的char一个个push到deque中
    for(size_t i=0;i<len;i++){
        _buffer.push_back(data[i]);
    }
    return len;

}

//! \param[in] len bytes will be copied from the output side of the buffer
string ByteStream::peek_output(const size_t len) const {
    size_t length=len;
    if(len>_buffer.size())
        length=_buffer.size();
    //将deque中的char构造成string对象再返回
    return string().assign(_buffer.begin(),_buffer.begin()+length);
}

//! \param[in] len bytes will be removed from the output side of the buffer
void ByteStream::pop_output(const size_t len) {
    size_t length=len;
    if(len>_buffer.size())
        length=_buffer.size();
    _read_count+=length;
    //将缓冲区中的内容一个个pop出去
    for(;length>0;length--)
        _buffer.pop_front();
    return;
}

//! Read (i.e., copy and then pop) the next "len" bytes of the stream
//! \param[in] len bytes will be popped and returned
//! \returns a string
std::string ByteStream::read(const size_t len) {
    //接收端读取实际上就是peek和pop的结合
   string s=peek_output(len);
   pop_output(len);
   return s;
}

void ByteStream::end_input() {
    _input_ended_flag=true;
}

bool ByteStream::input_ended() const { 
    return _input_ended_flag;
}

size_t ByteStream::buffer_size() const { 
    return _buffer.size();
}

bool ByteStream::buffer_empty() const { 
    return _buffer.empty();
}
//缓冲区为空并且输入结束代表eof，接收端才停止读取
bool ByteStream::eof() const {
    return buffer_empty()&&input_ended();
}

size_t ByteStream::bytes_written() const {
    return _write_count;
}

size_t ByteStream::bytes_read() const {
    return _read_count;
}

size_t ByteStream::remaining_capacity() const {
    return _capacity-_buffer.size();
}
```

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210812142433641.png" alt="image-20210812142433641" style="zoom: 67%;" />




# CMU15-213学习笔记（九）Network Programming

### IP 地址

我们会用一个叫做 IP address struct 的东西来存储IP地址。

我们知道不同主机可以有不同的主机字节顺序，常用的是小端法，而TCP/IP为任意整数定义了统一的**网络字节顺序（Network Byte Order）**，来规定IP地址的字节顺序（总是大端法），所以**IP 地址在内存中是以 network byte order（也就是大端）来进行存储的**。

```c
// Internet address structure
struct in_addr {
    uint32_t s_addr;    // 在IP地址结构中存放的地址总是以（大端法）网络字节顺序存放的
}
```

一般用下面的形式来进行表示：

IP 地址：`0x8002C2F2 = 128.2.194.242`

具体的转换可以使用 `getaddrinfo` 和 `getnameinfo` 函数

- Unix提供了以下函数来进行字节顺序转换

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210922114554430.png" alt="image-20210922114554430" style="zoom:50%;" />

  `htonl`函数是将32位整数由主机字节顺序转换为网络字节顺序；`ntohl`函数是将32位整数由网络字节顺序转换为主机字节顺序。

- IP地址通常用**点分十进制表示法**来表示，这里提供以下函数来实现IP地址和点分十进制字符串串之间的转换

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210922120544194.png" alt="image-20210922120544194" style="zoom:67%;" />

  `inet_pton`函数将一个点分十进制字符串`src`转换为网络字节顺序的IP地址，其中`AF_INET`表示32位的IPv4地址，而`AF_INET6`表示64位的IPv6地址。`inet_ntop`函数将网络字节顺序的IP地址`src`转换为对应的点分十进制字符串表示。

### 因特网连接

连接的两端分别是客户端套接字和服务器套接字，每个套接字都有相应的**套接字地址**，由IP地址和16位整数**端口**组成，表示为`地址:端口`。其中客户端套接字中的端口是由内核**自动分配**的**临时端口（Ephemeral Port）**，而服务器套接字的端口与服务器提供的服务相关，比如Web服务器就使用端口`80`、FTP服务器就是用端口`25`，可通过`/etc/services`查看。

**对Linux内核而言，套接字就是通信的一个端点。对应用而言，套接字就是一个可以让应用从网络中读写数据的文件描述符**（Linux中将所有资源都视为文件，套接字也不例外）。

**一个连接由它两端的套接字地址唯一确定**，称为**套接字对（Socket Pair）**，表示为`(cliaddr:cliport, servaddr:servport)`。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210922120843948.png" alt="image-20210922120843948" style="zoom:67%;" />

一个客户端可以同时和单一服务器上的不同端口通信，来获得该服务器的不同服务，但这需要建立不同的连接，避免相互干扰。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210922120952261.png" alt="image-20210922120952261" style="zoom:50%;" />

## 套接字接口

**套接字接口（Socket Interface）**是一组系统级的函数，他们和Unix I/O函数结合起来，用以创建网络应用。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210922121356029.png" alt="image-20210922121356029" style="zoom: 80%;" />

### 套接字地址结构

套接字的地址存放在下面的结构中，**在套接字地址结构中的IP地址和端口号的字节顺序总是以大端法存放的**。

- **泛型 socket address** ：

  ![image-20210922121925916](https://raw.githubusercontent.com/BoL0150/image2/master/image-20210922121925916.png)

- 具体协议 socket address：

  **对于采用套接字地址参数的函数，必须将 (`struct sockaddr_in *`) 强制转换为 (`struct sockaddr *`)**。	

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210922122042123.png" alt="image-20210922122042123" style="zoom:67%;" />

不同协议的套接字地址有自己不同的地址构造，比如IPv4就是`sockaddr_in`、IPv6就是`sockaddr_in6`等等，而`sockaddr`就是这些不同协议地址的抽象

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210922122626691.png" alt="image-20210922122626691" style="zoom:67%;" />

这样通过将`sockaddr_in`强制类型转换为`sockaddr`时，`sockaddr`的`sa_family`值就是 `sockaddr_in`的`sa_family`值，而`sockaddr`的`sa_data`值就是`sockaddr_in`中`sin_port`、`sin_addr`和`sin_zero`拼接起来的结果。

**由此通过将不同协议的地址强制类型转换为`sockaddr`类型，就能得到统一的套接字地址，而后的函数都是基于`sockaddr`类型的，方便兼容不同协议的地址**。

**注意！对于采用套接字地址参数的函数，必须将 (`struct sockaddr_in *`) 强制转换为 (`struct sockaddr *`)**。	

### 常用函数

首先，套接字作为一种文件，我们需要在服务器和客户端将其打开来得到**套接字描述符（Socket Descriptor）**

```c
#include <sys/types.h>
#include <sys/socket.h>
int socket(int domain, int type, int protocol);
```

为了使套接字作为连接的端点，可使用如下硬编码的参数来调用`socket`函数

```c
socket(AF_INET, SOCK_STREAM, 0); 
```

`AF_INET`表示是一个32位的IPv4地址，`SOCK_STREAM`表明连接是一个TCP连接。

此时就会返回套接字描述符，客户端套接字描述符写成`clientfd`，服务器套接字描述符写成`sockfd`。

客户端给clientfd随机分配一个端口号。而服务器端需要指明sockfd绑定的端口号。

#### 服务器

在服务器方面，接下来需要将服务器套接字地址和它的描述符`sockfd`联系起来

```c
#include <sys/socket.h>
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen); 
```

该内核函数会将服务器套接字描述符`sockfd`与服务器套接字地址`addr`关联起来，其中`addrlen=sizeof(sockaddr_in)`，此时`addr`中包含了端口号和服务器的IP地址。

由于`socket`函数创建的是**主动套接字描述符（Active Socket Descriptor）**，而服务器是接收请求方，所以需要将服务器套接字描述符转换为**监听套接字描述符（Listening Socket Descriptor）**

```c
#include <sys/socket.h>
int listen(int sockfd, int backlog);
```

该函数会将服务器套接字的描述符`sockfd`转换为监听套接字描述符`listenfd`，也相当于高速内核当前为服务器。接下来服务器就能监听等待来自客户端的连接请求

```c
#include <sys/socket.h>
int accept(int listenfd, struct sockaddr *addr, int *addrlen); 
```

该函数会等待来自客户端的连接请求到达监听描述符`listenfd`，然后将客户端的套接字地址填写到`addr`中，返回**已连接描述符（Connected Descriptor）** `connfd`，这样服务器就能通过`connfd`和Unix I/O来与客户端通信了。

服务器通常只创建一次监听套接字描述符`listenfd`，并存在于服务器的整个生命周期，作为客户端连接请求的一个端点，服务器可从`listenfd`得知是否有客户端发送连接请求 ，而后服务器每次接受连接请求时就会创建一个已连接描述符`connfd`来作为与客户端建立起连接的一个端点，只存在于服务器为一个客户端服务的过程，当客户端不需要该服务器的服务时，就会删掉该`connfd`。**这两种描述符的存在，使得我们可以构建并发的服务器来同时处理多个客户端连接，每当`accept`返回一个`connfd`时，就表示服务器接收了一个客户端的连接，此时就能通过`fork`函数创建一个子进程来通过该`connfd`与客户端通信，而父进程继续监听`listenfd`来查看是否有另一个客户端发送连接请求。**

**注意！服务器端创建的connfd与listenfd的端口号是相同的！只是文件描述符不同！**

#### 客户端

在客户端方面，在创建了客户端套接字描述符`clientfd`后，就能通过以下函数来建立与服务器的连接

```c
#include <sys/socket.h>
int connect(int clientfd, const struct sockaddr *addr, socklen_t addrlen); 
```

该函数会尝试与服务器套接字地址`addr`建立一个连接，其中`addrlen=sizeof(sockaddr_in)`。该函数会阻塞直到连接建立成功或发生错误，如果成功，客户端就能通过`clientfd`和Unix I/O与服务器进行通信了。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210922235623913.png" alt="image-20210922235623913" style="zoom:50%;" />

最好是使用getaddrinfo函数来自动生成以上这些函数的参数，这样代码就与协议无关了。

### `getaddrinfo`

`getaddrinfo`函数能将主机名、主机地址、服务名和端口号的字符串转换成套接字地址结构，能让我们**编写独立于任何特定版本的IP协议的网络程序**。

```c
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>

int getaddrinfo(const char *host, const char *service,
                const struct addrinfo *hints,
                const struct addrinfo **result);
void freeaddrinfo(struct addrinfo *result);
const char *gai_strerror(int errcode);
```

`getaddrinfo`函数能将主机名、主机地址、服务名和端口号的字符串转换成套接字地址结构。

- 其中，`host`参数可以是域名、点分十进制数字地址或NULL，
- `service`参数可以是服务名（如http）、十进制端口号或NULL，两者必须制定一个。
- 而`hits`和`result`参数分别是`addrinfo`结构的指针和链表

最终要调用`freeaddrinfo`函数来释放你得到的结果`result`链表，如果出错，可以调用`gai_strerror`函数来查看错误信息。

`addrinfo`：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210923094627023.png" alt="image-20210923094627023" style="zoom:67%;" />

`result`是通过`host`和`service`构建的一系列套接字地址结构，可以通过`ai_next`来一次遍历链表中的每个`addrinfo`结构

- 每个`addrinfo`结构的`ai_family`、`ai_socktype`和`ai_protocol`可以直接传递给`socket`函数
- `ai_addr`和`ai_addrlen`可以直接传递给`connect`和`bind`函数

使得我们只要传递`host`和`service`参数给`getaddrinfo`函数，就能编写独立于某个特定版本的IP协议的客户端和服务器。

一个域名可能会对应多个IP地址，通过这个`getaddrinfo`函数就能寻找域名对应的所有可用的IP地址。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20210923095023559.png" alt="image-20210923095023559" style="zoom:50%;" />

比如：Twitter有四个不同的IP地址，我们对`getaddrinfo`传入Twitter，实际上会返回一个有五个节点的链表，头结点是它的规范名，之后的四个节点中的每个节点都会含有一个IP地址。



### `getnameinfo`函数

```c
#include <sys/socket.h>
#include <netdb.h>
int getnameinfo(const struct sockaddr *sa, socklen_t salen,
                char *host, size_t hostlen,
                char *service, size_t servlen, int flags); 
```

该函数和`getaddrinfo`函数相反，通过传入套接字地址`sa`，能够返回对应的主机名字符串`host`和服务名字字符串`service`。其中`flags`可以修改该函数的默认行为：

- `NI_NUMERICHOST`：该函数默认返回`host`中的域名，通过设置这个标志位来返回数字地址字符串。
- `NI_NUMERICSERV`：该函数默认检查`/etc/service`并返回服务名，通过设置这个标志位来返回端口号。

一个IP地址可能会对应多个域名，通过这个`getnameinfo`函数就能寻找IP地址对应的所有可用的域名。


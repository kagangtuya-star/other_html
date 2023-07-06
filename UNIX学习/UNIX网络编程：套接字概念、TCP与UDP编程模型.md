# UNIX网络编程：套接字概念、TCP与UDP编程模型

UNIX网络编程 卷1: 套接字联网API (第3版) 第2、3、4、5、8章

第二章讲述了传输层的基础概念，第三章介绍了套接字的地址结构与操作函数，第4章给出了tcp编程模型需要的函数，第五章实现了一个基础的Tcp服务器客户端程序，第八章介绍了UDP编程模型的函数与一个范例程序。

**略去了SCTP的内容**

## 第2章 传输层：TCP、UDP

这里的介绍较为简略，更深层需要查阅之前的笔记

[`ComputerNetworkNotes`/计算机网络微课堂笔记第五章.md at main · `kagangtuya-star/ComputerNetworkNotes` · GitHub](https://github.com/kagangtuya-star/ComputerNetworkNotes/blob/main/计算机网络微课堂笔记第五章.md)

### 概述与总览

绝大多数客户/服务器网络应用使用TCP或UDP。

UDP是一个简单的、不可靠的数据报协议，而TCP是一个复杂、可靠的字节流协议。

TCP的某些特性一旦理解，就很容易编写健壮的客户和服务器程序，也很容易使用诸如netstat 等普遍可用的工具来调试客户和服务器程序。

- IPv4（通常称之为IP）自20世纪80年代早期以来一直是网际协议族的主力协议。它使用32位地址（见A.4节）。IPv4给TCP、UDP、SCTP、ICMP和IGMP提供分组递送服务。
- IPv6是在20世纪90年代中期作为IPv4的一个替代品设计的。其主要变化是使用128位更大地址（见A.5节）以应对20世纪90年代因特网的爆发性增长。IPv6给TCP、UDP、SCTP和ICMPv6提供分组递送服务。

TCP 传输控制协议 （Transmission Control Protocol）。TCP是一个面向连接的协议，为用户进程提供可靠的全双工字节流。TCP套接字是一种流套接字 （stream socket）。TCP关心确认、超时和重传之类的细节。大多数因特网应用程序使用TCP。注意，TCP既可以使用IPv4，也可以使用IPv6。

UDP 用户数据报协议 （User Datagram Protocol）。UDP是一个无连接协议。UDP套接字是一种数据报套接字 （datagram socket）。UDP数据报不能保证最终到达它们的目的地。与TCP一样，UDP既可以使用IPv4，也可以使用IPv6。

### UDP

UDP是一个简单的传输层协议。应用进程往一个UDP套接字写入一个消息，该消息随后被封装（encapsulating）到一个UDP数据报，该UDP数据报进而又被封装到一个IP数据报，然后发送到目的地。UDP不保证UDP数据报会到达其最终目的地，不保证各个数据报的先后顺序跨网络后保持不变，也不保证每个数据报只到达一次。

- UDP缺乏可靠性，无法自动重传丢失或错误的数据报。如果一个数据报到达了其最终目的地，但是校验和检测发现有错误，或者该数据报在网络传输途中被丢弃了，它就无法被投递给UDP套接字，也不会被源端自动重传。如果想要确保一个数据报到达其目的地，可以往应用程序中添置一大堆的特性：来自对端的确认、本端的超时与重传等。
- UDP数据报有长度，与数据一起传递给接收端应用进程。如果一个数据报正确地到达其目的地,那么该数据报的长度将随数据一道传递给接收端应用进程
- UDP是无连接的服务，一个UDP客户可以创建一个套接字并发送一个数据报给一个给定的服务器,然后立即用同一个套接字发送另一个数据报给另一个服务器。同样地,一个UDP服务器可以用同一个UDP套接字从若干个不同的客户接收数据报,每个客户一个数据报。

### TCP

TCP提供客户与服务器之间的连接 (connection)。TCP客户先与某个给定服务器建立一个连接,再跨该连接与那个服务器交换数据,然后终止这个连接。

- TCP提供了可靠性（数据的可靠递送或故障的可靠通知）。当TCP向另一端发送数据时,它要求对端返回一个确认。如果没有收到确认,TCP就自动重传数据并等待更长时间。在数次重传失败后,TCP才放弃。注意,TCP并不保证数据一定会被对方端点接收,因为这是不可能做到的。如果有可能,TCP就把数据递送到对方端点,否则就(通过放弃重传并中断连接这一手段)通知用户。
- TCP可以动态估算客户和服务器之间的往返时间`RTT`。
- TCP通过给其中每个字节关联一个序列号对所发送的数据进行排序 。
- TCP提供流量控制 (flow control)。TCP总是告知对端在任何时刻它一次能够从对端接收多少字节的数据。TCP提供流量控制，通过通告窗口告知对端在任何时刻它一次能够从对端接收多少字节的数据。通告窗口的大小时刻动态变化，指出接收缓冲区中当前可用的空间量，以确保发送端发送的数据不会使接收缓冲区溢出。当接收端应用从缓冲区中读取数据时，窗口大小增大，但窗口大小减小到0是有可能的，当TCP对应某个套接字的接收缓冲区已满导致它必须等待应用从该缓冲区读取数据时，才能从对端再接收数据。
- TCP连接是全双工的 (full-duplex)。这意味着在一个给定的连接上应用可以在任何时刻在进出两个方向上既发送数据又接收数据。因此,TCP必须为每个数据流方向跟踪诸如序列号和通告窗口大小等状态信息。

### TCP连接的建立和终止

#### TCP连接建立——三路握手

建立一个TCP连接时会发生下述情形。

1. 服务器必须准备好接受外来的连接。这通常通过调用socket 、bind 和listen 这3个函数来完成,我们称之为被动打开(passive open)。
2. 客户通过调用connect 发起主动打开 (active open)。这导致客户TCP发送一个SYN(同步)分节,它告诉服务器客户将在(待建立的)连接中发送的数据的初始序列号。通常SYN分节不携带数据,其所在IP数据报只含有一个IP首部、一个TCP首部及可能有的TCP选项(我们稍后讲解)。
3. 服务器必须确认(ACK)客户的SYN,同时自己也得发送一个SYN分节,它含有服务器将在同一连接中发送的数据的初始序列号。服务器在单个分节中发送SYN和对客户SYN的ACK(确认)。
4. 客户必须确认服务器的SYN。

这种交换至少需要3个分组,因此称之为TCP的三路握手 (three- way handshake)。

![image-20230706111605481](UNIX网络编程：套接字概念、TCP与UDP编程模型.assets/image-20230706111605481.png)

客户的初始序列号为J,服务器的初始序列号为K。ACK中的确认号是发送这个ACK的一端所期待的下一个序列号。因为SYN占据一个字节的序列号空间,所以每一个SYN的ACK中的确认号就是该SYN的初始序列号加1。类似地,每一个FIN(表示结束)的ACK中的确认号为该FIN的序列号加1。

建立TCP连接类似于打电话,`socket`函数相当于有电话可用,`bind`函数告诉别人你的电话号码,`listen`函数打开电话振铃,`connect`函数要求拨打对方电话,`accept`函数在连接建立后返回客户的标识,类似于电话的呼叫者ID功能部件。使用DNS提供电话簿服务,`getaddrinfo`类似于查找电话号码,`getnameinfo`则按照电话号码而不是用户名排序。

#### TCP选项

TCP的SYN可以含有多个选项,常用的包括MSS选项和窗口规模选项。MSS选项通告对端本连接的最大分节大小（它在本连接的每个TCP分节中愿意接受的最大数据量）,窗口规模选项指定通告窗口必须扩大的位数,以增加最大窗口大小。时间戳选项可防止数据损坏,但仅在高速网络连接中必要,不用考虑。

#### TCP连接终止——四路握手

TCP建立一个连接需3个分节,终止一个连接则需4个分节。

1. 某个应用进程首先调用close ,我们称该端执行主动关闭(active close)。该端的TCP于是发送一个FIN分节,表示数据发送完毕。
2. 接收到这个FIN的对端执行被动关闭 (passive close)。这个FIN由TCP确认。它的接收也作为一个文件结束符(end-of-file)传递给接收端应用进程(放在已排队等候该应用进程接收的任何其他数据之后),因为FIN的接收意味着接收端应用进程在相应连接上再无额外数据可接收。
3. 一段时间后,接收到这个文件结束符的应用进程将调用close 关闭它的套接字。这导致它的TCP也发送一个FIN。
4. 接收这个最终FIN的原发送端TCP(即执行主动关闭的那一端)确认这个FIN。

每个方向都需要一个FIN和一个ACK,因此通常需要4个分节。

![image-20230706114513736](UNIX网络编程：套接字概念、TCP与UDP编程模型.assets/image-20230706114513736.png)

类似SYN,一个FIN也占据1个字节的序列号空间。因此,每个FIN的ACK确认号就是这个FIN的序列号加1。

当套接字被关闭时,其所在端TCP各自发送了一个FIN。我们在图中指出,这是由应用进程调用close 而发生的,不过需认识到,当一个Unix进程无论自愿地(调用exit 或从main 函数返回)还是非自愿地(收到一个终止本进程的信号)终止时,所有打开的描述符都被关闭,这也导致仍然打开的任何TCP连接上也发出一个FIN。

无论是客户还是服务器,任何一端都可以执行主动关闭。通常情况是客户执行主动关闭,但是某些协议(譬如值得注意的HTTP/1.0)却由服务器执行主动关闭。

#### TCP状态总览

![](UNIX网络编程：套接字概念、TCP与UDP编程模型.assets/image-20230706115217518.png)

TCP为一个连接定义了11种状态,并且TCP规则规定如何基于当前状态及在该状态下所接收的分节从一个状态转换到另一个状态。举例来说,当某个应用进程在`CLOSED`状态下执行主动打开时,TCP将发送一个SYN,且新的状态是SYN_SENT。如果这个TCP接着接收到一个带ACK的SYN,它将发送一个ACK,且新的状态是`ESTABLISHED`。这个最终状态是绝大多数数据传送发生的状态。、

自`ESTABLISHED`状态引出的两个箭头处理连接的终止。如果某个应用进程在接收到一个FIN之前调用close (主动关闭),那就转换到`FIN_WAIT_1`状态。但如果某个应用进程在`ESTABLISHED`状态期间接收到一个FIN(被动关闭),那就转换到`CLOSE_WAIT`状态。

下面是一个完整的TCP连接所发生的实际分组交换情况，包括连接建立、数据传送和连接终止。

![image-20230706115502023](UNIX网络编程：套接字概念、TCP与UDP编程模型.assets/image-20230706115502023.png)

一旦建立TCP连接，客户端向服务器发送请求并收到应答。如果请求和应答都很小，可以在单个TCP分节中发送和接收。在TCP中，确认通常随着应答一起发送，这种做法称为捎带（piggybacking）。终止连接需要发送四个分节，并且主动关闭一端（例如客户端）会进入TIME_WAIT状态。 TCP的开销比UDP大，但TCP提供了可靠性和拥塞控制等重要功能。一些网络应用使用UDP，因为它避免了TCP连接建立和终止所需的开销，但是需要应用层处理可靠性和拥塞控制。

### TIME_WAIT状态

TCP中的TIME_WAIT状态是网络编程中最难理解的部分之一。当一端执行主动关闭时，它会进入TIME_WAIT状态，这个状态持续的时间是最长分节生命周期（`MSL`）的两倍，通常被称为2MSL。每个TCP实现必须选择一个`MSL`的值。RFC 1122建议的值是2分钟，但Berkeley的实现传统上使用30秒的值。因此，TIME_WAIT状态的持续时间在1分钟到4分钟之间。

`MSL`是任何IP数据报在因特网中存活的最长时间。每个数据报都包含一个8位字段，称为跳限或TTL（生存时间），它的最大值为255。尽管这是一个跳数限制而不是真正的时间限制，但我们假设：具有最大跳数限制的分组在网络中存在的时间不可能超过`MSL`秒。

分组在网络中“迷途”通常是路由异常的结果。如果路由器崩溃或两个路由器之间的链路断开，路由协议需要数秒钟到数分钟的时间来稳定并找到另一条通路。在这段时间内，可能会发生路由循环，导致我们关心的分组陷入其中。如果迷途的分组是一个TCP分节，发送端TCP可能会超时并重传该分节，而重传的分节可能通过某条候选路径到达最终目的地。但是在最多`MSL`秒之后，路由循环可能会修复，之前迷失在这个循环中的分组最终也被送到目的地。这个原来的分组称为迷途的重复分组或漫游的重复分组。TCP必须正确处理这些重复的分组，以保证数据的可靠传输。

### 端口号

在任何时候，多个进程可能同时使用TCP、UDP和SCTP这三种传输层协议之一。这三种协议都使用16位整数的端口号来区分这些进程。

当一个客户端想要与一个服务器建立连接时，它必须标识要与之通信的这个服务器。TCP、UDP和SCTP定义了一组众所周知的端口，用于标识众所周知的服务。例如，支持FTP的所有TCP/IP实现都将21这个众所周知的端口分配给FTP服务器。而分配给简化文件传输协议（TFTP）的是UDP端口号69。

另一方面，客户端通常使用短期存活的临时端口。这些端口号通常由传输层协议自动分配给客户端。客户端通常不关心其临时端口的具体值，只需确保该端口在所在主机中是唯一的即可。传输协议的代码会确保这种唯一性。

端口号的一般划分如下

![image-20230706131619213](UNIX网络编程：套接字概念、TCP与UDP编程模型.assets/image-20230706131619213.png)

端口号被划分成以下三段：

- 众所周知的端口为0~1023。这些端口由IANA分配和控制。如果可能的话，相同的端口号将被分配给TCP、UDP和SCTP的同一给定服务。例如，TCP和UDP都使用端口号80来标识Web服务器，尽管目前所有的实现都只使用TCP。

- 已登记的端口为1024~49151。这些端口不受IANA控制，但是IANA会登记并提供这些端口的使用情况清单，以方便整个群体。如果可能的话，相同的端口号也将被分配给TCP和UDP的同一给定服务。例如，端口号6000~6063被分配给TCP和UDP的X Window服务器，尽管目前所有的实现都只使用TCP。引入49151作为上限是为了给临时端口留出空间，而RFC 1700中列出的上限为65535。

- 49152~65535是动态或私有端口。IANA不控制这些端口。它们被称为临时端口。49152这个数字是65536的四分之三。

Unix系统有保留端口 (reserved port)的概念,指的是小于1024的任何端口。这些端口只能赋予特权用户进程的套接字。所有IANA 众所周知的端口都是保留端口,分配使用这些端口的服务器(例如FTP服务器)必须以超级用户特权启动。

#### 标记通信位置的一组值——套接字

套接字是用于标识网络上的通信端点的一组值，包括IP地址和端口号。在TCP连接中，套接字对是一个四元组，包括本地IP地址、本地TCP端口号、远程IP地址和远程TCP端口号，唯一标识每个连接。

对于SCTP，套接字对的概念与TCP相同，但是一个关联可能需要多个四元组标识，特别是在其中一个端点是多宿的情况下。每个端点由一组IP地址和一个端口号标识。

即使UDP是无连接的，我们仍然可以将套接字对的概念应用于它。在讲解套接字函数（如`bind`、`connect`、`getpeername`等）时，我们需要指定套接字对中的哪些值。例如，bind函数要求应用程序为TCP、UDP或SCTP套接字指定本地IP地址和本地端口号。

#### TCP端口号与并发服务器（多个进程/线程用一个端口）

TCP无法仅仅通过查看目的端口号来分离外来的分节到不同的端点。它必须查看套接字对的所有4个元素才能确定由哪个端点接收某个到达的分节。

![image-20230706140122080](UNIX网络编程：套接字概念、TCP与UDP编程模型.assets/image-20230706140122080.png)

图中，对于同一个本地端口(21) 存在3个套接字。如果一个分节来自206.168.112.219端口1500,目的地为12.106.32.254端口21,它就被递送给第一个子进程。如果一个分节来自206.168.112.219端口1501,目的地为12.106.32.254端口21,它就被递送给第二个子进程。所有目的端口为21的其他TCP分节都被递送给拥有监听套接字的最初那个服务器(父进程)。

#### 缓冲区大小以及限制

IP数据报大小存在着各种各样的最大大小，有的是标准规定的，如IPv4数据报，有的是硬件需求规定的，如MTU。

当一个IP数据报将从某个接口送出时,如果它的大小超过相应链路的MTU,IPv4和IPv6都将执行分片 (fragmentation)。这些片段在到达最终目的地之前通常不会被重组 (reassembling)。

IPv4和IPv6都定义了最小重组缓冲区大小 (minimum reassembly buffer size),它是IPv4或IPv6的任何实现都必须保证支持的最小数据报大小。

TCP有一个MSS(maximum segment size,最大分节大小),用于向对端TCP通告对端在每个分节中能发送的最大TCP数据量。

...

## 套接字的API

大多数套接字函数都需要一个指向套接字地址结构的指针作为参数。每个协议族都定义它自己的套接字地址结构。这些结构的名字均以`sockaddr_` 开头,并以对应每个协议族的唯一后缀结尾

#### IPv4套接字地址结构

Pv4套接字地址结构通常也称为“网际套接字地址结构”,它以`sockaddr_in` 命名,定义在`<netinet/in.h>` 头文件中。

```
struct in_addr {
    in_addr_t s_addr;  // 32-bit IPv4 address, network byte ordered
};

struct sockaddr_in {
    uint8_t sin_len;        // length of structure (16)
    sa_family_t sin_family; // AF_INET
    in_port_t sin_port;     // 16-bit TCP or UDP port number, network byte ordered
    struct in_addr sin_addr;// 32-bit IPv4 address, network byte ordered
    char sin_zero[8];       // unused
};
```

 `sin_len`

表示结构体的长度,通常设置为16。这是为增加对OSI协议的支持而随4.3BSD-Reno添加的。在此之前,第一个成员是sin_family,它是一个无符号短整数(unsigned short)。

并不是所有的厂家都支持套接字地址结构的长度字段,而且POSIX规范也不要求有这个成员。
该成员的数据类型uint8_t是典型的,符合POSIX的系统都提供这种形式的数据类型。
有了长度字段,才简化了长度可变套接字地址结构的处理。

即使有长度字段,我们也无须设置和检查它,除非涉及路由套接字。它是由处理来自不同协议族的套接字地址结构的例程(例如路由表处理代码)在内核中使用的。

`sin_family`

表示地址族,通常设置为 AF_INET,表示IPv4地址族。

![image-20230706145116426](UNIX网络编程：套接字概念、TCP与UDP编程模型.assets/image-20230706145116426.png)

`sin_port`

表示端口号,是一个16位的无符号整数,使用网络字节序(大端序)表示

 `in_addr` `sin_addr`

表示IP地址,是一个32位的无符号整数,同样使用网络字节序表示。32位IPv4地址存在两种不同的访问方法。如果`serv`定义为某个网际套接字地址结构,那么`serv.sin_addr`将按`in_addr`结构引用其中的32位IPv4地址,而`serv.sin_addr.s_addr`将按`in_addr_t`(通常是一个无符号的32位整数)引用同一个32位IPv4地址。

`in_addr` 

结构体表示一个32位的IPv4地址,包含一个无符号整数 `s_addr`,使用网络字节序(大端序)表示。`sockaddr_in` 结构体则表示一个IPv4套接字地址,包含了地址族、端口号和IP地址三个重要的信息

`sin_zero`[8]:是一个8字节的填充字段,在协议中没有实际用途,仅用于对齐。



必须正确地使用IPv4地址,尤其是在将它作为函数的参数时,因为编译器对传递结构和传递整数的处理是完全不同的。

POSIX规范只需要这个结构中的3个字段:`sin_family`、`sin_addr`和`sin_port`。对于符合POSIX的实现来说,定义额外的结构字段是可以接受的,这对于网际套接字地址结构来说也是正常的。几乎所有的实现都增加了sin_zero字段,所以所有的套接字地址结构大小都至少是16字节。

IPv4地址和TCP或UDP端口号在套接字地址结构中总是以网络字节序来存储。在使用这些字段时,我们必须牢记这一点。

**套接字地址结构仅在给定主机上使用:**虽然结构中的某些字段(例如IP地址和端口号)用在不同主机之间的通信中,但是结构本身并不在主机之间传递。

#### 通用套接字地址结构

其作用为对指向特定于协议的套接字地址结构的指针执行类型强制转换。

```
struct sockaddr{
    uint8_t        sa_len;
    sa_family_t    sa_family;   /*address family：AF_XXX value*/
    char           sa_data[14]; /*protocol-specific address*/
}
```

当作为一个参数传递进任何套接字函数时,套接字地址结构总是以**引用形式(也就是以指向该结构的指针)**来传递。

以这样的指针作为参数之一的任何套接字函数**必须处理来自所支持的任何协议族的套接字地址结构。**

在如何声明所传递指针的数据类型上存在一个问题。有了`ANSI C`后解决办法很简单:`void *`是通用的指针类型。然而套接字函数是在`ANSI C`之前定义的,采取的办法是在`<sys/socket.h>`头文件中定义一个通用的套接字地址结构。

下面是一个简单的示例

```
int bind(int,struct sockaddr *,socklen_t);

struct sockaddr_int serv;
bind(sockfd,(struct sockaddr*)&serv,sizeof(serv));
//  bind 函数调用将 sockfd 文件描述符绑定到一个IPv4套接字地址 serv 上,serv 是一个 struct sockaddr_in 结构体类型的变量,其中包含了要绑定的地址和端口信息。bind 函数的第二个参数是指向 struct sockaddr 结构体的指针,需要将 serv 的地址强制转换为 struct sockaddr 的指针类型,即 (struct sockaddr*)&serv,这样才能作为 bind 函数的参数传入。第三个参数是 serv 结构体的大小,可以使用 sizeof 运算符获取。
```

#### IPv6套接字地址结构

IPv6套接字地址结构是用来表示IPv6地址和端口号的数据结构,它定义在头文件`<netinet/in.h>`中。

```
struct in6_addr{
    unit8_t    s6_addr[16];    /*s128-bit IPv6 address*/
                               /*network byte ordered*/
};
 
#define SIN6_LEN    /*required for compile-time tests*/
 
struct sockaddr_in6{
    uint8_t         sin6_len;      /*length of this struct (28)*/
    sa_family_t     sin6_family;   /*AF_INET6*/
    in_port_t       sin6_port;     /*transport layer port*/
                                   /*network byte ordered*/
    uint32_t        sin6_flowinfo; /*flow information,undefined*/
    struct in6_addr sin6_addr;     /*IPv6 address*/
                                   /*network byte ordered*/
    uint32_t        sin6_scope_id; /*set of interfaces for a scope*/
};
```

`sin6_len`: 一个8位的字段,表示IPv6套接字地址结构的长度,单位为字节。在IPv6套接字地址结构中,`sin6_len`的值为28。
`sin6_family`: 一个16位的字段,表示地址族,它的值为AF_INET6,表示IPv6地址族。
`sin6_port`: 一个16位的字段,表示传输层端口号,它的值是网络字节序(big-endian)的。对于TCP或UDP套接字,该字段必须在调用bind()函数时设置,一般使用`htons`()函数将主机字节序转换为网络字节序。
`sin6_flowinfo`: 一个32位的字段,用于指定流信息,目前该字段尚未定义,应该设置为0。
`sin6_addr`: 一个128位的IPv6地址,存储在一个长度为16字节的数组s6_addr中,每个字节都是网络字节序的。IPv6地址的表示方法是8个16位的整数,每个整数使用16进制表示,各整数之间用冒号分隔,例如:2001:0db8:85a3:0000:0000:8a2e:0370:7334。
`sin6_scope_id`: 一个32位的字段,用于指定地址的范围,当使用链路本地地址或站点本地地址时,该字段必须设置为相应的接口索引号。对于全局地址,该字段应设置为0。

注意

- 如果系统支持套接字地址结构中的长度字段,那么`SIN6_LEN`常值必须定义。
- IPv6的地址族是AF_INET6,而IPv4的地址族是AF_INET。
- 结构中字段的先后顺序做过编排,使得如果`sockaddr_in6`结构本身是64位对齐的,那么128位的`sin6_addr`字段也是64位对齐的。在一些64位处理机上,如果64位数据存储在某个64位边界位置,那么对它的访问将得到优化处理。
- `sin6_flowinfo`字段分成两个字段:低序20位是流标(flow label);高序12位保留。
- 对于具备范围的地址(scoped `address`),`sin6_scope_id`字段标识其范围(scope),最常见的是链路局部地址(link-local address)的接口索引(interface index))。

#### 新的通用套接字地址结构

```
struct sockaddr_storage{
    uint8_t     ss_len;    /*length of thos struct (implementation dependent)*/
    sa_family_t ss_familt; /*address family;AF_XXX value*/
}
```

作为IPv6套接字API的一部分而定义的新的通用套接字地址结构克服了现有struct sockaddr 的一些缺点。不像struct sockaddr ,新的struct sockaddr_storage 足以容纳系统所支持的任何套接字地址结构。

- 如果系统支持的任何套接字地址结构有对齐需要,那么`sockaddr_storage`能够满足最苛刻的对齐要求。
- `sockaddr_storage`足够大,能够容纳系统支持的任何套接字地址结构。

除了`ss_family`和`ss_len`外(如果有的话),`sockaddr_storage`结构中的其他字段对用户来说是透明的。
`sockaddr_storage`结构必须类型强制转换成或复制到适合于`ss_family`字段所给出地址类型的套接字地址结构中,才能访问其他字段。

#### 套接字地址结构的比较

在套接字编程中,不同类型的套接字地址结构有不同的长度。为了处理长度可变的结构,我们需要在传递指向套接字地址结构的指针时,同时传递该结构的长度作为另一个参数给套接字函数。这个长度字段可以用一个单字节的字节表示,而地址族字段也占用一个字节。

IPv4和IPv6结构是固定长度的,所以在使用这些结构时,我们不需要传递额外的长度参数给套接字函数。但是,Unix域结构和数据链路结构是可变长度的,因此我们需要小心处理长度字段,包括套接字地址结构本身的长度字段(如果实现支持)以及作为参数传递给内核或从内核返回的长度。

虽然`sockaddr_un`结构本身的长度是固定的,但是其中的路径名长度是可变的。因此,在传递指向`sockaddr_un`结构的指针时,我们仍然需要传递该结构的长度作为另一个参数给套接字函数,以便正确处理路径名长度。

在4.3BSD Reno版本中,长度字段被引入到所有套接字地址结构中。如果套接字API的原始版本提供了长度字段,那么所有套接字函数就不再需要长度参数,例如`bind`和`connect`函数的第三个参数。相反,结构的大小可以包含在结构的长度字段中。

![image-20230706151132372](UNIX网络编程：套接字概念、TCP与UDP编程模型.assets/image-20230706151132372.png)

### 值—结果参数

当往一个套接字函数传递一个套接字地址结构时,该结构总是以引用形式来传递,也就是说传递的是指向该结构的一个指针。该结构的长度也作为一个参数来传递,不过其传递方式取决于该结构的传递方向:是从进程到内核,还是从内核到进程。

换句话说,当传递一个套接字地址给套接字函数时,实际上传递的是这个地址结构的指针,而不是整个结构本身。同时,为了确保内核和进程之间传递的数据是一致的,这个结构的长度也需要同时传递,并根据数据传递方向的不同采用不同的传递方式。

#### 从进程到内核传递套接字地址结构的函数

`bind`、`connect`和`sendto` 这些函数的一个参数是指向某个套接字地址结构的指针,另一个参数是该结构的整数大小。这些函数的一个参数是指向某个套接字地址结构的指针,另一个参数是该结构的整数大小。如下：

```
struct sockaddr_in serv;
/* fill in serv{} */
connect(sockfd, (SA *) &serv, sizeof(serv));
```

指针和指针所指内容的大小都传递给了内核,于是内核知道到底需从进程复制多少数据进来。

![image-20230706154926751](UNIX网络编程：套接字概念、TCP与UDP编程模型.assets/image-20230706154926751.png)

#### 从内核到进程传递套接字地址结构的函数

`accept`、`recvfrom`、`getsockname`和`getpeername`这4个函数的其中两个参数是指向某个套接字地址结构的指针和指向表示该结构大小的整数变量的指针。

```
struct sockaddr_un cli; /* Unix domain */
socklen_t len;
len = sizeof(cli); /* len is a value */
getpeername(unixfd, (SA *) &cli, &len);
/* len may have changed */
```

把套接字地址结构大小这个参数从一个整数改为指向某个整数变量的指针,其原因在于:

- 当函数被调用时,结构大小是一个值(value),它告诉内核该结构的大小,这样内核在写该结构时不至于越界;
- 当函数返回时,结构大小又是一个结果(result),它告诉进程内核在该结构中究竟存储了多少信息。

这种类型的参数称为值—结果(value-result)参数。

![image-20230706162121088](UNIX网络编程：套接字概念、TCP与UDP编程模型.assets/image-20230706162121088.png)

当使用值—结果参数作为套接字地址结构的长度时,如果套接字地址结构是固定长度的,那么从内核返回的值总是那个固定长度,例如IPv4的`sockaddr_in`长度是16,IPv6的`sockaddr_in6`长度是28。然而对于可变长度的套接字地址结构(例如Unix域的`sockaddr_un`),返回值可能小于该结构的最大长度。

在网络编程中,值—结果参数最常见的例子是所返回套接字地址结构的长度。还会碰到其他值—结果参数:

- `select`函数中间的3个参数
- `getsockopt`函数的长度参数
- 使用`recvmsg`函数时,`msghdr`结构中的`msg_namelen`和`msg_controllen`字段
- `ifconf`结构中的`ifc_len`字段
- `sysctl`函数两个长度参数中的第一个

### 字节排序函数

一个16位整数,它由2个字节组成。内存中存储这两个字节有两种方法：

- 将低序字节存储在起始地址,这称为小端(little-endian)字节序
- 将高序字节存储在起始地址,这称为大端(big-endian)字节序。

“小端”和“大端”表示多个字节值的哪一端(小端或大端)存储在该值的起始地址。把某个给定系统所用的字节序称为主机字节序(host byteorder)。

![image-20230706171231962](UNIX网络编程：套接字概念、TCP与UDP编程模型.assets/image-20230706171231962.png)

检验大小端测试:在一个短整数变量中存放2字节的值0x0102,然后查看它的两个连续字节c[0] (对应图地址A)和c[1] (对应图中的地址A+1),以此确定字节序。
字符串CPU_VENDOR_OS是由GNU的`autoconf`程序在配置本书中的软件时确定的,它标识CPU类型、厂家和操作系统版本。

```
int main(int argc, char **argv)
{
    union {
        short  s;
        char   c[sizeof(short)];
    } un;
 
    un.s = 0x0102;
    printf("%s: ", CPU_VENDOR_OS);
    if (sizeof(short) == 2) {
        if (un.c[0] == 1 && un.c[1] == 2)
            printf("big-endian\n");
        else if (un.c[0] == 2 && un.c[1] == 1)
            printf("little-endian\n");
        else
            printf("unknown\n");
    } else
        printf("sizeof(short) = %d\n", sizeof(short));
 
    exit(0);
}
```

#### 在主机字节序和网络字节序之间相互转换

由于历史的原因和POSIX规范的规定,套接字地址结构中的某些字段必须按照网络字节序进行维护。因此要关注如何在主机字节序和网络字节序之间相互转换。

`htons`和`htonl`函数将主机字节序的值转换为网络字节序的值,而`ntohs`和`ntohl`函数将网络字节序的值转换为主机字节序的值,以便在不同计算机之间进行正确的数据传输。

使用这些函数时,我们并不关心主机字节序和网络字节序的真实值(或为大端,或为小端)。我们所要做的只是调用适当的函数在主机和网络字节序之间转换某个给定值。在那些与网际协议所用字节序(大端)相同的系统中,这四个函数通常被定义为空宏。

```
#include<netinet/in.h>
uint16_t htons(uint16_t host16bitvalue);
uint32_t htonl(uint32_t host32bitvalue);  //均返回：网络字节序的值
uint16_t ntohs(uint16_t net16bitvalue);
uint32_t ntohl(uint32_t net32bitvalue);   //均返回：主机字节序的值
```

**参数**

`host16bitvalue`:一个16位的主机字节序的值。
`host32bitvalue`:一个32位的主机字节序的值。
`net16bitvalue` :一个16位的网络字节序的值。
`net32bitvalue` :一个32位的网络字节序的值。
**返回值**

`htons`和`htonl`函数返回一个网络字节序的值,该值是将主机字节序的16位或32位值转换为网络字节序的结果。
`ntohs`和`ntohl`函数返回一个主机字节序的值,该值是将网络字节序的16位或32位值转换为主机字节序的结果。

### 字节操纵函数

字节操纵函数用于处理以空字符结尾的C字符串，这些函数是由在<`string.h`>头文件中定义、名字以str(表示字符串)开头的函数处理的。

历史上，名字以b(表示字节)开头的第一组函数起源于4.2BSD,几乎所有现今支持套接字函数的系统仍然提供它们。名字以mem(表示内存)开头的第二组函数起源于ANSI C标准,支持ANSI C函数库的所有系统都提供它们。

```
#include <strings.h>
void bzero(void *dest, size_t nbytes);
void bcopy(const void *src, void *dest, size_t nbytes);
int bcmp(const void *ptr1, const void *ptr2, size_t nbytes); 

void *memset(void *dest, int c, size_t len);
void *memcpy(void *dest, const void *src, size_t nbytes);
int memcmp(const void *ptr1, const void *ptr2, size_t nbytes); 
```

##### `bzero`函数

**参数**

`dest`：指向要清零的内存区域的指针

`nbytes`：要清零的字节数

**返回值**

无返回值

**作用**

将 `dest` 指向的内存区域的前 nbytes 个字节清零，即设置为0。

memcmp比较两个任意的字节串，若相同则返回0，否则返回一个非0值，是大于0还是小于0则取决于第一个不等的字节：

- 如果ptr1所指字节串中的这个字节大于ptr2所指字节中的对应字节，那么大于0，否则小于0
- 比较操作是在假设两个不等的字节均为无符号字符（unsigned char）的前提下完成的。

**`bcopy`函数**

**参数**

`src`：指向源内存区域的指针

`dest`：指向目标内存区域的指针

`nbytes`：要复制的字节数

**返回值**

无返回值

**作用**

将 src 指向的内存区域的前 nbytes 个字节复制到 dest 指向的内存区域。



**`bcmp`函数**

参数：

`ptr1`：指向第一个内存区域的指针

`ptr2`：指向第二个内存区域的指针

`nbytes`：要比较的字节数

**返回值**

若两个内存区域的前 nbytes 个字节相等，则返回0；否则返回非0值。

**作用**

比较 ptr1 指向的内存区域和 ptr2 指向的内存区域的前 nbytes 个字节是否相等。



`memset`**函数**

参数：

`dest`：指向要填充的内存区域的指针

`c`：要填充的字节值

`len`：要填充的字节数

**返回值**

返回指向填充后的内存区域的指针。

**作用**

将 `dest` 指向的内存区域的前 len 个字节全部设置为 c。

记住memset最后两个参数顺序的方法之一是认识到所有ANSI C的memXXX函数都需要一个长度参数,而且它总是最后一个参数。

`memcpy`**函数**

**参数**

`dest`：指向目标内存区域的指针

`src`：指向源内存区域的指针

`nbytes`：要复制的字节数

**返回值**

返回指向目标内存区域的指针。

**作用**

将 src 指向的内存区域的前 nbytes 个字节复制到 dest 指向的内存区域。

memcpy类似bcopy,不过两个指针参数的顺序是相反的。

当源字节串与目标字节串重叠时,bcopy能够正确处理,但是memcpy的操作结果却不可知。这种情形下必须改用ANSI C的memmove函数。

记住`memcpy`两个指针参数顺序的方法之一是记着它们是按照与C中的赋值语句相同的顺序从左到右书写的:`dest = src;`

`memcmp`**函数**

**参数**

`ptr1`：指向第一个内存区域的指针

`ptr2`：指向第二个内存区域的指针

`nbytes`：要比较的字节数

**返回值**

若两个内存区域的前 nbytes 个字节相等，则返回0；若第一个内存区域的对应字节值小于第二个，则返回负值；否则返回正值。

**作用**

比较 ptr1 指向的内存区域和 ptr2 指向的内存区域的前 nbytes 个字节大小。

### 地址转换函数

在ASCII字符串(这是人们偏爱使用的格式)与网络字节序的二进制值 (这是存放在套接字地址结构中的值)之间转换网际地址。

#### 在点分十进制数串（例如“206.168.112.96 ”）与它长度为32位的网络字节序二进制值间转换IPv4地址——`inet_aton/inet_addr /inet_ntoa`函数（不推荐使用）

```
#include <arpa/inet.h>

int inet_aton(const char *strptr, struct in_addr *addrptr);
in_addr_t inet_addr(const char *strptr);
char *inet_ntoa(struct in_addr inaddr);

// 将一个IPv4地址字符串转换成网络字节序的二进制IP地址
// strptr - 要转换的IPv4地址字符串
// addrptr - 存储转换后的二进制IP地址的结构体指针
// 返回值 - 转换成功返回1，否则返回0
int inet_aton(const char *strptr, struct in_addr *addrptr);

// 将一个IPv4地址字符串转换成32位网络字节序的二进制IP地址
// strptr - 要转换的IPv4地址字符串
// 返回值 - 转换成功返回32位网络字节序的二进制IP地址，否则返回INADDR_NONE
in_addr_t inet_addr(const char *strptr);

// 将一个32位网络字节序的二进制IP地址转换成点分十进制数串
// inaddr - 要转换成点分十进制数串的二进制IP地址
// 返回值 - 指向点分十进制数串的指针
char *inet_ntoa(struct in_addr inaddr);
```

**`inet_aton`：**将strptr所指C字符串转换成一个32位的网络字节序二进制值，并通过指针addrptr来存储。若成功则返回1，否则返回0。

**`inet_addr`：**进行相同的转换，返回值为32位的网络字节序二进制值。该函数存在一个问题:所有232个可能的二进制值都是有效的IP地址 (从0.0.0.0到255255.255.255)，但是当出错时该数返回INADDR_NONE常值 (通常是一个32位均为1的值)。这意味着点分十进制数串255.255.255.255(这是IP4的有限广播地址)不能由该函数处理因为它的二进制值 被用来指示该函数失败。

**`inet_ntoa`：**函数将一个32位的网络字节序二进制IPv4地址转换成相应的点分十进制数串。

如今`inet_addr` 已被废弃,新的代码应该改用`inet_aton` 函数。但最好用下一节的新函数。

#### 在点分十进制数串（例如“206.168.112.96 ”）与网络字节序二进制值间互相转换IPv4或IPv6地址——`inet_pton/inet_ntop` 函数

函数名中p 和n 分别代表表达（presentation）和数值 （numeric）,地址的表达格式通常是ASCII字符串，数值格式则是存放到套接字地址结构中的二进制值。

```
include <arpa/inet.h>
int inet_pton(int family, const char *strptr, void *addrptr);

const char *inet_ntop(int family, const void *addrptr, char *strptr, size_t len);
```

**`inet_pton`函数**

**作用**

将字符串形式的IP地址转换为二进制形式，并存储到`addrptr`指向的内存中。

**参数**

- `family`：表示IP地址族，可取`AF_INET`或`AF_INET6`，分别表示IPv4和IPv6。
- `strptr`：指向字符串形式的IP地址的指针。
- `addrptr`：指向存储转换结果的内存的指针。

**返回值**

- 若成功则返回1。
- 若输入不是有效的表达格式则返回0。
- 若出错则返回-1。

---

**`inet_ntop`函数**

**作用**

将二进制形式的IP地址转换为字符串形式，并存储到`strptr`指向的内存中。

**参数**

- `family`：表示IP地址族，可取`AF_INET`或`AF_INET6`，分别表示IPv4和IPv6。
- `addrptr`：指向二进制形式的IP地址的指针。
- `strptr`：指向存储转换结果的内存的指针。
- `len`：表示存储转换结果的内存的大小，以免该函数溢出其调用者的缓冲区。为有助于指定这个大小，在`<netinet/in.h>`头文件中有如下定义

- ```
  #define INET_ADDRSTRLEN 16 /* for IPv4 dotted-decimal */
  #define INET6_ADDRSTRLEN 46 /* for IPv6 hex string */
  ```

**返回值**

- 若成功则返回指向结果的指针。
- 若出错则返回NULL。

![image-20230706175323825](UNIX网络编程：套接字概念、TCP与UDP编程模型.assets/image-20230706175323825.png)

### 获取套接字结构中指向某个二进制地址的指针——`sock_ntop` 和相关函数

`inet_ntop`的一个基本问题是:它要求调用者传递一个指向某个二进制地址的指针，而该地址通常包含在一个套接字地址结构中，这就要求调用者必须知道这个结构的格式和地址族。

为了使用这个函数，我们必须为编写如下代码

```
struct sockaddr_in addr;
inet_ntop(AF_INET, &addr.sin_addr, str, sizeof(str));// ipv4

struct sockaddr_in6 addr6;
inet_ntop(AF_INET6, &addr6.sin6_addr, str, sizeof(str)); // ipv6
```

为了解决这个问题，可以自行编写一个名为`sock_ntop` 的函数，它以指向某个套接字地址结构的指针为参数，查看该结构的内部，然后调用适当的函数返回该地址的表达格式

```
#include "unp.h"
char *sock_ntop(const struct sockaddr *sockaddr, socklen_t addrlen);
// 返回：若成功则为非空指针，若出错则为NULL
```

`sockaddr` 指向一个长度为`addrlen` 的套接字地址结构。本函数用它自己的静态缓冲区来保存结果，而指向该缓冲区的一个指针就是它的返回值。

```
int sock_bind_wild(int sockfd, int family);
int sock_cmp_addr(const struct sockaddr *sockaddr1, const struct sockaddr *sockaddr2, socklen_t addrlen);
int sock_cmp_port(const struct sockaddr *sockaddr1, const struct sockaddr *sockaddr2, socklen_t addrlen);
int sock_get_port(const struct sockaddr *sockaddr, socklen_t addrlen);
char *sock_ntop_host(const struct sockaddr *sockaddr, socklen_t addrlen);
void sock_set_addr(const struct sockaddr *sockaddr, socklen_t addrlen, void *ptr);
void sock_set_port(const struct sockaddr *sockaddr, socklen_t addrlen, int port);
void sock_set_wild(struct sockaddr *sockaddr, socklen_t addrlen);
```

**`sock_bind_wild`：**将通配地址和一个临时端口捆绑到一个套接字。

**`sock_cmp_addr`：**比较两个套接字地址结构的地址部分。

**`sock_cmp_port`：**则比较两个套接字地址结构的端口号部分。

**`sock_get_port`：**只返回端口号。

**`sock_ntop_host`：**把一个套接字地址结构中的主机部分转换成表达格式(不包括端口号)。

**`sock_set_addr`：**把一个套接字地址结构中的地址部分置为ptr指针所指的值;sock_set_port则只设置一个套接字地址结构的端口号部分。

**`sock_set_wild`：**把一个套接字地址结构中的地址部分置为通配地址。

跟本书所有函数一样，**我们也为那些返回值不是void的上述函数提供了包裹函数，它们的名字以s开头**，我们的程序通常调用这些包裹函数。

### `readn` 、`writen` 和`readline` 函数

字节流套接字(例如TCP套接字)上的read和write函数所表现的行为不同于通常的文件I/O。字节流套接字上调用read或write输入或输出的字节数可能比请求的数量少，然而这不是出错的状态。这个现象的原因在于内核中用于套接字的缓冲区可能已达到了极限。此时所需的是调用者再次调用read或write函数，以输入或输出剩余的字节。有些版本的Unix在往一个管道中写多于4096字节的数据时也会表现出这样的行为。这个现象在读一个字节流套接字时很常见，但是在写一个字节流套接字时只能在该套接字为非阻塞的前提下才出现。尽管如此，为预防万一，不让实现返回一个不足的字节计数值，我们总是改为调用writen函数来取代write函数。

**`readn`函数**

函数作用：从文件描述符`fd`指向的文件或套接字中读取`count`个字节，存放到缓冲区`buf`中。

参数：

- `fd`：文件描述符，可以是普通文件描述符、套接字描述符等。
- `buf`：存放读取数据的缓冲区。
- `count`：期望读取的字节数。

返回值：返回实际读取的字节数，如果返回值小于`count`，表示读取过程中遇到了文件结束或错误。

```
int readn(int fd, void *vptr, size_t n)
{
    size_t          nleft = n;           //readn函数还需要读的字节数
    ssize_t         nread = 0;           //read函数读到的字节数
    unsigned char   *ptr = (char *)vptr; //指向缓冲区的指针

    while (nleft > 0)
    {
        nread = read(fd, ptr, nleft);
        if (-1 == nread)
        {
            if (EINTR == errno)
                nread = 0;
            else
                return -1;
        }
        else if (0 == nread)
        {
            break;
        }
        nleft -= nread;
        ptr += nread;
    }
    return n - nleft;
}
```

`writen`函数

函数作用：将缓冲区`buf`中的`count`个字节写入到文件描述符`fd`指向的文件或套接字中。

参数：

- `fd`：文件描述符，可以是普通文件描述符、套接字描述符等。
- `buf`：存放待写入数据的缓冲区。
- `count`：待写入的字节数。

返回值：返回实际写入的字节数，如果返回值小于`count`，表示写入过程中遇到了错误。

```
int writen(int fd, const void *vptr, size_t n)
{
    size_t          nleft = n;  //writen函数还需要写的字节数
    ssize_t         nwrite = 0; //write函数本次向fd写的字节数
    const char*     ptr = vptr; //指向缓冲区的指针

    while (nleft > 0)
    {
        if ((nwrite = write(fd, ptr, nleft)) <= 0)
        {
            if (nwrite < 0 && EINTR == errno)
                nwrite = 0;
            else
                return -1;
        }
        nleft -= nwrite;
        ptr += nwrite;
    }
    return n;
}
```

`readline`函数

函数作用：从文件描述符`fd`指向的套接字中读取一行数据，存放到缓冲区`buf`中，最多读取`maxlen`个字节。

参数：

- `fd`：套接字描述符。
- `buf`：存放读取数据的缓冲区。
- `maxlen`：缓冲区的最大长度。

返回值：返回实际读取的字节数，如果返回值为0，表示对端已经关闭连接；如果返回值小于0，表示读取过程中遇到了错误。读取到的数据不包含行结束符（例如`'\n'`），需要自行添加。

```
static ssize_t readch(int fd, char *ptr)
{
    static int          count = 0;
    static char*        read_ptr = 0;
    static char         read_buf[1024*4] = {0};

    if (count <= 0)
    {
    again:
        count = read(fd, read_buf, sizeof(read_buf));
        if (-1 == count)
            if (EINTR == errno)
                goto again;
            else
                return -1;
        else if (0 == count)
            return 0;
        read_ptr = read_buf;
    }
    count--;
    *ptr = *read_ptr++;
    return 1;
}

ssize_t readline(int fd, void *vptr, size_t maxlen)
{
    ssize_t         i = 0;
    ssize_t         ret = 0;
    char            ch = '\0';
    char*           ptr = NULL;

    ptr = (char *)vptr;

    for (i = 1; i < maxlen; ++i)
    {
        ret = readch(fd, &ch);
        if (1 == ret)
        {
            *ptr++ = ch;
            if ('\n' == ch)
                break;
        }
        else if (0 == ret)
        {
            *ptr = 0;
            return i-1;
        }
        else
            return -1;
    }
    *ptr = 0;
    return i;
}
```


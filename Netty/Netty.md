## Netty

### Java I/O

#### I/O基础

##### Linux网络I/O模型简介

UNIX编程对I/O模型的分类，共有五种I/O模型：

1.阻塞I/O

最常用的I/o模型就是阻塞I/O，缺省情况下，所有文件操作都是阻塞的。在进程空间中调用recvfrom，其系统调用直到数据包到达且被复制到应用进程的缓冲区中或发生错误时才返回，在此期间一直等待，进程从调用recvfrom开始到它返回的整段时间内都是被阻塞的。

![image-20201218104505515](E:\studyRepository\img\image-20201218104505515.png)

2.非阻塞I/O

recvfrom从应用层到内核的时候，如果该缓冲区没有数据的话，就直接返回一个EWOULDBLOCK错误，一般对非阻塞I/O模型进行轮询检查这个状态。

![image-20201218104728217](E:\studyRepository\img\image-20201218104728217.png)

3.I/O复用模型

Linux提供Select/poll，进程通过将一个或多个fd(socket描述符)传递给select或poll系统调用，阻塞在select操作上，这样select/poll可以帮我们侦测多个fd是否处于就绪状态。select/poll是按顺序扫描fd是否就绪。而且支持的fd数量有限，因此它的使用受到了一些约束。Linux还提供一个epoll系统调用，epoll使用基于事件驱动方式代替顺序扫描，因此性能更高。当有fd就绪时，立即回调rollback函数

![image-20201218105503866](E:\studyRepository\img\image-20201218105503866.png)

4.信号驱动I/O模型

首先开启套接口信号驱动I/O功能，并通过系统调用sigaction执行一个信号处理函数（此系统调用立即返回，进程继续工作，是非阻塞的）。当数据准备就绪时，九尾该进程生成一个SIGIO信号，通过信号回调通知应用程序调用recvfrom来读取数据，并通知主循环函数处理数据。

![image-20201218105927051](E:\studyRepository\img\image-20201218105927051.png)

5.异步I/o

告知内核启动某个操作，并让内核在整个操作完成后（包括将数据从内核复制到用户自己的缓冲区）通知我们。与信号驱动模型的区别是：信号驱动是由内核告知我们何时可以开始一个I/O操作，异步I/O是由内核通知我们何时操作已完成。

![image-20201218110451297](E:\studyRepository\img\image-20201218110451297.png)

##### I/O多路复用

在I/O编程中，当需要同时处理多个客户端接入请求时，可以利用多线程或者I/O多路复用技术进行处理。I/O多路复用技术通过把多个I/O的阻塞复用到同一个select阻塞上，从而使得系统在单线程的情况下可以同时处理多个客户端请求。

与传统的多线程模型相比，I/O多路复用最大优势是系统开销小，系统不需要创建新的额外线程或进程，也不需要维护这些线程或进程的运行，降低了系统维护的工作量，节省系统资源，主要应用场景如下：

- 服务器需要同时处理多个处于监听状态或多个连接状态的socket
- 服务器需要同时处理多种网络协议的socket

目前支持I/O多路复用的系统调用有select，pselect，poll，epoll。以前一直使用select，现在改为epoll。epoll对于select的改进

1. 支持一个进程打开的socket描述符（FD）不受限制，仅限于操作系统的最大文件句柄数

2. I/O的效率不会随着FD的数目增加而线性下降

   select/poll每次调用都会线性扫描全部的集合，导致效率程线性下降。epoll不存在这个问题，它只会对活跃的socket进行操作--因为epoll是根据每个fd上面的callback函数实现的。

3. 使用mmap加速内核与用户空间的消息传递

   无论是select、poll、epoll都需要内核把FD消息通知给用户空间。epoll可以通过内核和用户空间mmap同一块内存来实现。

4. epoll的API更加简单

#### JAVA I/O的演进

在JDK1.4出现NIO之前，JAVA的所有Socket通信都是基于同步阻塞的BIO。

NIO主要类和接口如下：

- 进行异步I/O操作的ByteBuffer等
- 进行异步I/O操作的管道Pipe
- 进行各种I/O操作的（异步或同步）的Channel，包括ServerSocketChannel和SocketChannel
- 多种字符集的编码能力和解码能力
- 实现非阻塞I/O操作的duolufuyongselector
- 基于流行的Perl实现的正则表达式类库
- 文件通道FileChannel

不足之处：

- 没有统一的文件属性（如读写权限）
- API能力较弱，如目录的级联创建和递归遍历，需自己实现
- 底层存储系统的一些高级API无法使用
- 所有的文件操作都是同步阻塞调用，不支持异步文件读写操作

JDK1.7正式发布后的NIO2.0，对三个方面进行了改进：

- 提供能够批量获取文件属性的API。还提供了标准文件的SPI，供各个服务提供商扩展实现
- 提功AIO功能，支持基于文件的异步I/O操作和网络套接字的异步操作
- 完成JSR-51定义的通道功能，包括对配置和多播数据的支持

### NIO入门

#### NIO类库简介

##### 缓冲区Buffer

Buffer是一个对象，包含一些要写入或者要读出的数据。在面向流的I/O中，可以将数据直接写入或者将数据直接读到Stream对象中。

==在NIO库中，所有数据都是用缓冲区处理的==，在读取数据时，是直接读到缓冲区中的。在写入数据时，写入到缓冲区中。任何时候访问NIO中的数据，都是通过缓冲区进行操作。

==缓冲区实质是一个数组==。通常是一个字节数组，也可以是其他种类的数组。提供了对数据结构化访问以及维护读写位置等信息。

每一种Java基本类型对应的都有缓冲区：

- ByteBuffer 字节缓冲区
- CharBuffer 字符缓冲区
- ShortBuffer 短整型缓冲区
- IntBuffer 整形缓冲区
- LongBuffer 长整型缓冲区
- FloatBuffer 浮点型缓冲区
- DoubleBuffer 双精度浮点型缓冲区

每一个Buffer类都是Buffer接口的一个子实例。因为大多数标准I/O操作都使用ByteBuffer，所以它在具有一般缓冲区的操作之外还提供了一些特有的操作，以方便网络读写。除了ByteBuffer，每一个Buffer类都有完全一样的操作，只不过处理的数据类型不一样。

##### 通道Channel

==Channel是一个通道，网络数据通过Channel读取和写入。通道与流的不同之处，在于通道是双向的，流只是在一个方向上移动==，通道可以用于读、写或二者同时进行。

##### 多路复用器Selector

Selector会不断的轮询注册其上的Channel，如果某个Channel上发生读或者写事件，这个Channel就处于就绪状态，会被Selector轮询出来，然后通过SelectionKey可以获取就绪Channel集合，进行后续操作。

一个多路复用Selector可以同时轮询多个Channel，由于JDK使用了epoll（）代替传统的select，所以它并没有最大连接句柄1024/2048的限制。这也意味着只需要一个线程负责Selector的轮询，就可以接入成千上万个客户端。

![image-20201218150101052](E:\studyRepository\img\image-20201218150101052.png)

1. 打开ServerSocketChannel，用于监听客户端的连接，它是所有客户端连接的父管道

   ![image-20201218150414327](E:\studyRepository\img\image-20201218150414327.png)

2. 绑定监听端口，设置连接为非阻塞模式

   ![image-20201218150425799](E:\studyRepository\img\image-20201218150425799.png)

3. 创建Reactor线程，创建多路复用器并启动线程

    ![image-20201218150453086](E:\studyRepository\img\image-20201218150453086.png)

4. 将ServerSocketChannel注册到Reactor线程的多路复用器Selector上，监听ACCEPT事件![image-20201218150703094](E:\studyRepository\img\image-20201218150703094.png)

5. 多路复用器在线程run方法的无限循环体内轮询准备就绪的Key![image-20201218150801941](E:\studyRepository\img\image-20201218150801941.png)

6. 多路复用器监听到有新的客户端接入，处理新的接入请求，完成TCP三次握手，建立物理链路 ![image-20201218150910424](E:\studyRepository\img\image-20201218150910424.png)

7. 设置客户端链路为非阻塞模式![image-20201218150949756](E:\studyRepository\img\image-20201218150949756.png)

8. 将新接入的客户端连接注册到Reactor线程的多路复用器，监听读操作，读取客户端发送的网络信息![image-20201218151110374](E:\studyRepository\img\image-20201218151110374.png)

9. 异步读取客户端请求信息到缓冲区![image-20201218151139592](E:\studyRepository\img\image-20201218151139592.png)

10. 对ByteBuffer进行编解码，将解码成功的消息封装成Task，投递到业务线程池![image-20201218151248855](E:\studyRepository\img\image-20201218151248855.png)

11. 将POJO对象encode成ByteBuffer，调用SocketChannel的异步write接口，将消息异步发送给客户端

    ![image-20201218151358391](E:\studyRepository\img\image-20201218151358391.png)

如果发送区TCP缓冲区满，会导致写半包，此时，需要注册监听写操作位，循环写，知道整包消息写入TCP缓冲区。

![image-20201218151807221](E:\studyRepository\img\image-20201218151807221.png)

1. 打开SocketChannel绑定客户端本机地址![image-20201218151944089](E:\studyRepository\img\image-20201218151944089.png)
2. 设置SocketChannel为非阻塞模式，同时设置客户端连接的TCP参数，![image-20201218152046870](E:\studyRepository\img\image-20201218152046870.png)
3. 异步连接服务端![image-20201218152122291](E:\studyRepository\img\image-20201218152122291.png)
4. 判断是否连接成功，如果连接成功，直接注册读状态位到多路复用器，如果当前没有连接成功（异步连接，返回false，说明客户端已经发送sync包，服务端没有返回ack包，物理链路还没有建立）![image-20201218152426845](E:\studyRepository\img\image-20201218152426845.png)
5. 向Reactor线程的多路复用器注册OP_CONNECT状态位，监听服务端的TCP ACK应答![image-20201218152543195](E:\studyRepository\img\image-20201218152543195.png)
6. 创建Reactor线程，创建多路复用器并启动线程![image-20201218152630347](E:\studyRepository\img\image-20201218152630347.png)
7. 多路复用器在线程run方法的无限循环体内轮询准备就绪的Key![image-20201218152727947](E:\studyRepository\img\image-20201218152727947.png)
8. 接受connect事件进行处理![image-20201218152749026](E:\studyRepository\img\image-20201218152749026.png)
9. 判断连接结果，如果连接成功，注册读事件到多路复用器![image-20201218152848803](E:\studyRepository\img\image-20201218152848803.png)
10. 注册读事件到多路复用器![image-20201218152915634](E:\studyRepository\img\image-20201218152915634.png)
11. 异步读客户端请求消息到缓冲区![image-20201218152940326](E:\studyRepository\img\image-20201218152940326.png)
12. 对ByteBuffer进行编解码，将编码成功的消息封装成Task，投递到业务线程池![image-20201218153032446](E:\studyRepository\img\image-20201218153032446.png)
13. 将POJO对象encode成ByteBuffer，调用SocketChannel的异步write接口，将消息异步发送给客户端![image-20201218153123683](E:\studyRepository\img\image-20201218153123683.png)

#### AIO

异步通道提供以下两种方式来获取操作结果：

- 通过java.util.concurrent.Future类来表示异步操作的结果
- 在执行异步操作的时候传入一个java.nio.channels

CompletionHanlder接口的实现类作为操作完成的回调。

### TCP粘包拆包问题解决

#### TCP粘包拆包

![image-20201218161127593](E:\studyRepository\img\image-20201218161127593.png)

TCP粘包拆包的原因

原因有三个：

1. 应用程序write写入的字节大小大于套接口发送缓冲区大小
2. 碱性MSS大小的TCP分段
3. 以太网帧的payload大于MTU进行IP分片

业界的主流协议解决方案：

1. 消息定长，例如每个报文的大小固定长度200字节，如果不够，空位补空格
2. 在包尾增加回车换行符进行分割，例如FTP
3. 将消息分为消息头和消息体，消息头中包含表示消息总长度（或者消息体长度）的字段，通常的设计思路为消息头的第一个字段使用int32来表示消息的总长度
4. 更复杂的应用层协议

==TCP以流的方式进行数据传输，上层的应用协议为了对消息进行区分==，往往采用四种方式：

1. 消息长度固定，累计读取到长度总和为定长LEN的报文后，就认为读取带了一个完整的消息，将计数器置位，重新开始读取下一个数据报文
2. 将回车换行符作为消息结束符，例如FTP
3. 将特殊的分隔符作为消息结束的标志
4. 通过在消息头中定义长度字段来表示消息的总长度

Netty对上述四种应用做了统一的抽象，提供四种解码器来解决对应的问题。

==利用LineBasedFrameDecoder解决TCP粘包==

LineBasedFrameDecoder工作原理是依次遍历ByteBuf中的可读字节，判断是都有回车或换行符。如果有，以此位置为结束位置，从可读索引位置到结束位置区间的字节就组成了一行。==它是以换行符为结束标志的解码器，支持携带结束符或不携带结束符两种编码方式，并支持配置单行的最大长度==。如果连续读取到最大长度后仍没有发现换行符，则抛出异常。

==StringDecoder就是将接收到的对象转换成字符串==，然后继续调用后面的Handler。LineBasedFrameDecoder和StringDecoder组合就是按行切换的文本解码器。

==DelimiterBasedFrameDecoder可以指定特定的分隔符，可以自动完成以分隔符作为码流结束标识的消息的解码。==

==FixedLengthFrameDecoder是固定长度的解码器。能够对指定的长度对消息进行自动解码。==

### 编解码技术

java序列化缺点

1. 无法跨语言
2. 序列化后的码流太大
3. 序列化性能太低

#### 主流编解码框架

##### Google的Protobuf（使用二进制进行编码）

全称为Google Protocol Buffers。它将数据结构以.proto文件进行描述，通过代码生成工具，可以生成对应数据结构的POJO对象和ProtoBuf相关的方法和属性。

具有以下特点：

- 结构化数据存储格式（XML、JSON等）
- 高效的编解码性能
- 语言无关、平台无关、扩展性好
- 官方支持Java、c++、Python三种语言

Protobuf的数据描述文件和代码生成机制具有以下优点：

- 文本化的数据结构描述语言，可以实现语言和平台无关，特别适合异构系统间的集成
- 通过标识字段的顺序，可以实现协议的前向兼容
- 自动代码生成，不需要手工编译
- 方便后续的管理和维护，相比于代码，结构化的文档更容易管理和维护

##### Facebook的Thrift

支持多种程序语言。在不同语言之间通信，Thrift可以作为高性能的通信中间件使用，支持数据序列化和多种类型的RPC服务。适用于静态的数据交换，需要先确定好他的数据结构，当数据结构发生变化时，必须重新编辑IDL文件，生成代码和编译。

主要有五部分组成

1. 语言系统以及IDL编译器，负责由用户给定的IDL文件生成相应的语言的接口代码。
2. TProtocol：RPC协议层，可以选择多种不同的对象序列化方式，
3. TTransport：RPC的传输层，可以选择不同的传输层实现：如socket、NIO、MemoryBuffer
4. TProcessor：作为协议层和用户提供的服务实现之间的纽带，负责调用服务实现的接口
5. TServer：聚合TProtocol、TTransport和TProcessor等对象

TProtocol就是Thrift的编解码框架。通过IDL描述接口和数据结构定义，支持8种Java基本类型，Map、Set、和List，支持可选和必选定义。因为可以定义数据结构中字段的顺序，所以它也可以支持协议的前向兼容。支持的三种比较经典的编解码方式

1. 通用的二进制编解码
2. 压缩二进制编解码
3. 优化的可选字段压缩编解码

##### JBoss Marshalling

JBoss Marshalling是一个Java对象的序列化API包，修正了JDK自带的序列化包的很多问题，但又保持和java.io.Serializable接口的兼容。同时又增加了一些可调参数和附加特性。相对于传统的Java序列化机制，优点如下：

1. 可插拔的类解析器，提供更加便捷的类加载定制策略，通过一个接口可实现定制，
2. 可插拔的对象替换技术，不需要通过继承的方式
3. 可插拔的预定义类缓存表，可以减小序列化的字节数组长度，提升常用类型的对象序列化性能
4. 无需事先java.io.Serializable接口
5. 通过缓存技术提升对象的序列化性能

### MessagePack编解码

MessagePack是一个高效的二进制序列化框架。支持不同语言之间的数据交换。

#### MessagePack

特点：

1. 编解码高效，性能高
2. 序列化之后的码流较小
3. 支持跨语言

![image-20201223141923249](E:\studyRepository\img\image-20201223141923249.png)

### Google ProtoBuf编解码

ProtoBufDecoder仅仅负责解码，不支持读半包。因此，在ProtobufDecoder前面，一定要有能够处理半包的解码器。有以下三种方式供选择：

1. 使用Netty提供的ProtobufVarint32FrameDecoder
2. 集成Netty提供的通用半包解码器LengthFieldBasedFrameDecoder
3. 集成ByteToMessageDecoder类，自己处理半包消息

### HTTP协议开发应用

#### HTTP协议介绍

HTTP协议是建立在TCP传输协议之上的应用层协议。是一个属于应用层的面向对象的协议，简捷、快速，适用于分布式超媒体信息系统。

主要特点：

- 支持Client/Server模式
- 简单--客户向服务器请求服务时，只需指定服务URL，携带必要的请求参数或者消息体
- 灵活--HTTP允许传输任意类型的数据对象，传输的内容类型由HTTP消息头中的Content-Type加以标记
- 无状态--HTTP协议是无状态协议，无状态是指协议对于事务处理没有记忆能力。缺少状态意味着如果后续处理需要之前的信息，则必须重传，这样可能导致每次连接传送的数据量增大。另一方面，在服务器不需要先前信息时它的应答就较快，负载较轻

##### HTTP请求消息

请求由三部分组成：

1. HTTP请求行
2. HTTP消息头
3. HTTP请求正文

请求行以一个方法符开头，以空格分开，后面跟着请求的URI和协议的版本。格式为Method   

Request-URI   HTTP-Version CRLF。

Method表示请求方法，Request-URI是一个统一的资源标识符，HTTP-Version表示请求的HTTP协议的版本，CRLF表示回车和换行。

请求的方法有很多种，作用如下：

- GET：请求获取Request-URI标识的资源
- POST：在Request-URI所标识的资源后附加新的提交数据
- HEAD：请求获取由Request-URI所表示的资源的响应消息报头
- PUT：请求服务器存储一个资源，并用Request-URI作为其标识
- DELETE：请求服务器删除Request-URI所表示的资源
- TRACE：请求服务器回送收到的请求信息，主要用于测试或诊断
- CONNECT：保留将来使用
- OPTIONS：请求查询服务器的性能，或者查询与资源相关的选项和请求

GET一般用于获取/查询资源信息，而POST一般用于更新资源信息。GET和POST主要区别如下：

1. 根据HTTP规范，GET用于信息获取，而且应该是安全和幂等的，POST则表示可能改变服务器上资源的请求
2. GET提交，请求的数据会附在URL之后，就是把数据放置在请求行中，以？分割URL和传输数据，多个参数用&连接；而POST提交会把数据放置在HTTP消息的包体中，数据不会在地址栏中显示出来。
3. 传输数据的大小不同，特定浏览器和服务器对URL长度有限制。所以GET携带的参数长度会受到浏览器的限制，而POST由于不是通过ＵＲＬ传值，不会受到限制
4. 安全性。POST比GET安全性高.GET在URL终会暴露隐私数据,而且GET提交的数据还可能造成Cross-site request forgery 攻击.

![image-20201223160801548](E:\studyRepository\img\image-20201223160801548.png)

##### HTTP响应消息

HTTP相应也是由三个部分组成,分别是状态行、消息报头、响应正文.

状态行的格式为: HTTP-Version States-Code Reason-Phrase CRLF，States-Code表示服务器返回的响应状态代码。

状态代码由三位数字组成，第一个数字定义了响应的类别，由5种可能值。

1. 1xx：指示信息。表示请求已接收、继续处理。
2. 2xx：成功，表示已被成功接收、理解、接受
3. 3xx：重定向。要完成请求必须进行更进一步的操作
4. 4xx：客户端错误。请求有语法错误或无法实现
5. 5xx：服务器端错误。服务器未能处理请求

![image-20201223163604991](E:\studyRepository\img\image-20201223163604991.png)

##### JiBx框架（XML绑定框架）

JiBx是一款非常优秀的XML数据绑定框架。提功灵活的绑定映射文件，实现数据对象与XMl之间的转换，并且不需要修改既有的Java类。

优点：

1. 转换效率高
2. 配置绑定文件简单
3. 不需要操作xpath文件
4. 不需要写get/set方法
5. 对象的属性名与xml的element名可以不同

两个比较重要的概念：Unmarshal（数据分解）和Marshal（数据编排）。

Unmarshal是将XML文件转换为Java对象，而Marshal是将Java对象编排成xml文件。

### WebSocket协议

HTTP协议的弊端：

1. HTTP协议为半双工协议。半双工协议指数据可以在客户端和服务端两个方向传输，但是不能同时传输，意味着在同一时刻，只有一个方向上的数据传送。
2. HTTP消息冗长繁琐，HTTP消息包括消息头，消息体，换行符等。通常情况下采用文本传输，相比于其他的二进制通信协议，冗长而繁琐。
3. 针对服务器推送的黑客攻击。例如长时间轮询

WebSocket是一种浏览器与服务器全双工通信的技术。在WebSocket API中，浏览器和服务器只需要一个握手的动作，就形成了一条快速通道，两者就可以互传数据。WebSocket是基于TCP双向全双工进行消息传递。同一时刻，即可以发送消息，也可以接受消息。

特点：

1. 单一的TCP连接，采用全双工模式通信。
2. 对代理，防火墙和路由器透明
3. 无头部信息、Cookie和身份验证
4. 无安全开销
5. 通过ping/pong帧保持链路激活
6. 服务器可以主动传递消息给客户端，不再需要客户端轮询

![image-20201224093408040](E:\studyRepository\img\image-20201224093408040.png)

#### 生命周期

握手成功之后，服务端和客户端可以通过“messages”的方式进行通信，一个消息由一个或多个帧组成，WebSocket的消息并补一点给对应一个特定网络层的帧，它可以被分割成多个帧或者被合并。帧都有自己对应的类型，属于同一个消息的多个帧具有相同类型的数据。

### Netty私有协议

![image-20201224100341192](E:\studyRepository\img\image-20201224100341192.png)

#### Netty协议的编码

编码规范如下：

1. cecCode：java.nio.ByteBuffer.putInt(int value)，如果采用其他缓冲区实现，必须与其等价

2. length：java.nio.ByteBuffer.putInt(int value)，如果采用其他缓冲区实现，必须与其等价

3. sessionID：java.nio.ByteBuffer.putLong(long value)，如果采用其他缓冲区实现，必须与其等价

4. type：java.nio.ByteBuffer.put（byte b）如果采用其他缓冲区实现，必须与其等价

5. priority：java.nio.ByteBuffer.put（byte b）如果采用其他缓冲区实现，必须与其等价

6. attachment：它的编码规则为--如果attachment长度为0，表示没有可选附件，则长度编码设为0，java.nio.ByteBuffer.putInt(0)；如果大于0，说明有附件需要编码，具体编码队则如下：

   1.首先对附件个数进行编码，java.nio.ByteBuffer.putInt(attachment.size())

   2.然后对key进行编码，先编码长度，再将他转换成byte数组之后编码内容

7. body的编码，通过JBoss Marshalling 将其序列化为Byte数组，然后调用java.nio.ByteBuffer.put(byte[] src)将其写入Byte Buffer缓冲区

由于整个消息长度必须等全部字段都编码完成之后才能确认，所以最后需要更新消息头中的length字段，将其重新写入ByteBuffer中。

#### Netty协议的解码

1. cecCode：java.nio.ByteBuffer.getInt()获取验证码字段，如果采用其他缓冲区实现，必须与其等价
2. length：java.nio.ByteBuffer.getInt()获取Netty消息长度，如果采用其他缓冲区实现，必须与其等价
3. sessionID：java.nio.ByteBuffer.getLong()获取会话ID，如果采用其他缓冲区实现，必须与其等价
4. type：java.nio.ByteBuffer.get()获取消息类型，如果采用其他缓冲区实现，必须与其等价
5. priority：java.nio.ByteBuffer.get()获取消息优先级，如果采用其他缓冲区实现，必须与其等价
6. attachment：解码规则为：首先创建一个新的attachment对象，调用java.nio.ByteBuffer.getInt()获取附件长度，如果为0，说明附件为空，解码结束，如果非空，根据长度通过for循环进行解码
7. body：通过JBoss 的marshaller对其进行解码

#### 链路的建立

链路建立需要通过基于IP地址或者号段的和黑白名单安全认证机制。实际项目中，也可以通过密钥对用户名和密码进行安全认证。

## Netty源码

### 服务端创建

![image-20201224110523365](E:\studyRepository\img\image-20201224110523365.png)

1. 创建ServerBootstrap实例。ServerBootstrap是Netty服务端的启动辅助类，提供了一系列方法用于设置服务端的启动相关参数，底层通过门面模式对各种能力进行了抽象和封装。

2. 设置并绑定Reactor线程池。Netty的Reactor线程池是EventLoopGroup，实际上是一个EventLoop数组。EventLoop的职责是处理所有注册到本线程多路复用器Selector上的Channel，Selector的轮询操作由绑定的EventLoop线程run方法驱动，在一个循环体内循环执行。==而且EventLoop的指责不仅仅是处理网络I/O事件，用户自定义的Task和定时任务Task也统一由EventLoop负责。这样就实现了线程模型的统一。从调用层面看，EvevtLoop线程中不会再启动其他类型的线程，避免了多线程的并发操作和锁竞争==

   --源码中的逻辑

   将NioServerSocketChannel注册到Reactor线程时，首先判断是否是NioEventLoop自身发起的操作，如果是，不存在并发，直接执行Channel注册；如果是其他线程发起的，则封装成一个Task放入消息队列中异步执行。

3. 设置并绑定服务端的Channel。Netty的ServerBootstrap方法提供了channel方法用于指定服务器的channel类型。==作为NIO服务器，需要创建ServerSocketChannel，Netty对原生NIO类库进行了封装，对应的实现是NioServerSocketChannel。==

   --源码中的逻辑

   用户可以为启动辅助类和其父类分别指定Handler。两类的Handler的用途不同，子类中的Handler是NioServerSocketChannel对应的ChannelPipeline的Handler；父类的Handler是客户端新接入的连接SocketChannel对应的ChannelPipeline的Handler。

   本质的区别是：ServerBootstrap中的Handler是NioServerSocketChannel中使用的，所有的连接该监听端口的客户端都会使用；父类AbstractBootStrap中的Handler是个工厂类，为每个新接入的客户端都创建一个新的Handler。

4. 链路建立的时候创建并初始化ChannelPipeline。ChannelPipeline并不是NIO服务端必须的。它本质是一个负责处理网络事件的职责链，负责管理和执行ChannelHandler。网络事件以事件流的形式在ChannelPipeline中流转，由ChannelPipeline根据ChannelHandler的执行策略调度ChannelHandler的执行。

   典型的网络事件如：链路注册、链路激活、链路断开、接收到请求消息、请求消息接收并处理完毕、发送应答消息、链路发生异常、发生用户自定义事件

5. 初始化ChannelPipeline完成后，添加并设置ChannelHandler。ChannelHandler是Netty提供给用户定制和扩展的关键接口。利用ChannelHandler用户可以完成大多数的功能定制。如：消息编解码、心跳、安全认证、TSL/SSL认证、流量控制和流量整形等。![image-20201224112729330](E:\studyRepository\img\image-20201224112729330.png)

6. 绑定并设置监听端口。在绑定监听端口之前系统会做一系列的初始化和检测工作，完成之后，会启动监听端口，并将ServerSocketChannel注册到Selector上监听客户端连接

7. Selector轮询。由Reactor线程NioEventLoop负责调度和执行Selector轮询操作，选择准备就绪的Channel集合

8. 轮循到准备就绪的Channel后，就由Reactor线程NioEventLoop执行ChannelPipeline的相应方法，最终调度并执行ChannelHandler

9. 执行Netty系统ChannelHandler和用户添加定制的ChannelHandler。

### 客户端接入分析

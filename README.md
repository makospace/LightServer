## CppNetServer
> 本项目分为两个部分，首先是重构muduo库的小型网络库（姑且叫做miduo），基本实现了muduo库的全部功能，遵循了事件驱动，非阻塞IO复用，构建了内存池结构，采用Reactor模式；其次是利用miduo编写的一个Web服务器，解析了get、head请求，可处理静态资源，支持HTTP长连接，支持管线化请求，并实现了异步日志，记录服务器运行状态。

### 相关技术
1. 底层使用 Epoll + LT 模式的 I/O 复用模型，并且结合非阻塞 I/O 实现主从 Reactor 模型。
2. 采用「one loop per thread」线程模型，并向上封装线程池避免线程创建和销毁带来的性能开销。
3. 采用 eventfd 作为事件通知描述符，方便高效派发事件到其他线程执行异步任务。
4. 基于自实现的双缓冲区实现异步日志，由后端线程负责定时向磁盘写入前端日志信息，避免数据落盘时阻塞网络服务。
5. 基于红黑树实现定时器管理结构，内部使用 Linux 的 timerfd 通知到期任务，高效管理定时任务。 
6. 利用有限状态机解析 HTTP 请求报文。
7. 想着为了轻量级的目标，正在准备实现了内存池模块，以便于更好管理小块内存空间，减少内存碎片，可以遵循 RALL 手法使用智能指针管理内存，减小内存泄露风险。
8. 想着可以记录每个用户的特点进行配置设置，可以用数据库来实现，可以使用数据库连接池可以动态管理连接数量，及时生成或销毁连接，保证连接池性能。
9. 使用多线程充分利用多核CPU，并使用线程池避免线程频繁创建销毁的开销
10. 使用基于小根堆的定时器关闭超时请求
11. 主线程只负责accept请求，并以Round Robin的方式分发给其它IO线程(兼计算线程)，锁的争用只会出现在主线程和某一特定线程中
12. 使用状态机解析了HTTP请求,支持管线化，支持优雅关闭连接
    
### 主要类介绍：
1. EventLoop类：事件循环类，负责维护事件循环，处理IO事件、定时器事件和信号事件等。

2. Channel类：通道类，封装了一个文件描述符（socket或者文件），负责处理该文件描述符上的IO事件。

3. TcpConnection类：TCP连接类，封装了一个TCP连接，负责处理该连接上的IO事件和状态管理。

4. TcpServer类：TCP服务器类，负责监听和管理TCP连接，包括接受新连接、关闭连接等操作。

5. Buffer类：缓冲区类，封装了一个可变长度的字节数组，提供了读写操作和数据处理的接口。

6. TimerQueue类：定时器队列类，负责管理定时器事件，包括添加、删除、触发等操作。

7. Log类：日志类，提供了日志记录和输出的接口，支持多线程输出和异步日志。

8. Thread类：线程类，封装了一个线程对象，提供了线程的创建、启动、停止等操作。

9. ThreadPool类：线程池类，负责管理线程池，包括添加任务、线程池扩容等操作。

10. 该项目是一个C++实现的网络服务器，可以同时处理多个连接。它由几个类组成，每个类负责服务器功能的特定方面。
11. EventLoop类是服务器的核心，负责管理所有I/O事件。它使用epoll系统调用来有效地监视多个文件描述符的事件。
12. Channel类用于表示由事件循环监视的文件描述符。它提供了启用/禁用读写事件的方法，以及在发生事件时调用的回调函数
13. Socket类用于创建和操作套接字。它提供了创建、绑定、监听、接受和连接套接字的方法。
14. Acceptor类负责在指定的IP地址和端口号上接受传入的连接。它创建一个Socket对象，并将其绑定到IP地址和端口号上。然后，它使用套接字的文件描述符和传递的事件循环创建一个Channel对象。当套接字准备好读取时，设置回调函数AcceptConnection()将被调用。当接受新连接时，将调用AcceptConnection()。它首先检查套接字对象的Accept()方法是否返回RC_SUCCESS。如果是，则使用fcntl()将新连接的文件描述符设置为非阻塞模式。最后，如果设置了新连接回调函数，则将其调用，并将新连接的文件描述符作为参数传递。
15. Connection类表示与客户端的连接。它使用传递的文件描述符创建一个Socket对象。然后，它使用套接字的文件描述符和传递的事件循环创建一个Channel对象。当套接字准备好读取时，设置回调函数HandleRead()将被调用。当有数据可读时，将调用HandleRead()。它将数据读入缓冲区，并调用用户定义的回调函数，将数据作为参数传递。
16. Buffer类用于表示数据缓冲区。它提供了读取和写入数据到/从缓冲区的方法。总的来说，该项目提供了一个框架，用于构建可以同时处理多个连接的网络服务器。
### 采用的模式：Reactor模式
还有一些其他模式：
1. Proactor模式：与Reactor模式类似，但是在处理I/O操作时采用了异步的方式，即I/O操作完成后再通知应用程序。
2. Actor模式：将系统中的每个组件都看作是一个独立的Actor，通过消息传递来实现组件之间的通信和协作。
3. Half-sync/Half-async模式：将应用程序分为两部分，一部分异步处理I/O事件，另一部分同步处理业务逻辑。
4. Thread-pool模式：将一组线程预先创建好，当有任务到来时将任务分配给线程池中的线程来处理，避免了线程的频繁创建和销毁。
### Reactor模式的特点：
1. 采用事件驱动的方式处理I/O操作，提高了系统的可伸缩性和性能。
2. 将I/O操作和业务逻辑分离，使得系统的设计更加清晰和易于维护。
3. 通过回调函数的方式来处理I/O事件的结果，避免了阻塞操作，提高了系统的并发性。
4. 可以通过选择合适的设计模式来扩展Reactor模式，使得系统更加灵活和可扩展。
5. 事件驱动的IO和异步IO都是实现高效IO操作的方法，但是有一些异同点。
>事件驱动的IO是指程序通过监听事件来实现IO操作，当有事件发生时，程序才会进行相应的IO操作。常见的事件驱动模型包括Reactor模型和Proactor模型。异步IO是指程序在进行IO操作时，不需要等待IO操作完成，而是可以继续执行其他操作。异步IO通常需要使用回调函数来处理IO操作的结果。区别在于，事件驱动的IO是通过监听事件来实现IO操作，而异步IO是通过不等待IO操作完成来实现高效IO操作。事件驱动的IO通常需要使用事件循环来处理事件，而异步IO则需要使用回调函数来处理IO操作的结果。

*智能指针*：
>1. 普通指针：普通指针是C++中最基本的指针类型，它只是一个指向内存地址的变量，不具有自动内存管理的功能。需要手动分配和释放内存，容易出现内存泄漏和悬挂指针等问题。
>2. unique_ptr：unique_ptr是C++11引入的一种智能指针，它是一种独占式的智能指针，即一个unique_ptr对象只能有一个指向同一个对象的指针，不能进行复制或赋值。当unique_ptr超出作用域或被删除时，它所指向的对象也会被自动释放。unique_ptr实现原理是通过模板类和move语义实现的。
>3. shared_ptr：shared_ptr也是C++11引入的智能指针，它是一种共享式的智能指针，多个shared_ptr对象可以指向同一个对象，对象的引用计数会增加，只有当最后一个shared_ptr对象被销毁时，对象才会被释放。shared_ptr实现原理是通过引用计数实现的。
>4. weak_ptr：weak_ptr也是C++11引入的智能指针，它是一种弱引用的智能指针，它可以指向一个由shared_ptr管理的对象，但不会增加对象的引用计数，当最后一个shared_ptr被销毁后，weak_ptr也会自动失效。weak_ptr实现原理是通过一个弱引用计数器实现的。

特点：
>1. unique_ptr具有独占式的特点，能够防止多个指针同时访问同一个对象，从而避免了悬挂指针和内存泄漏等问题。
>2. shared_ptr可以实现多个指针共享同一个对象，可以避免内存泄漏和悬挂指针等问题。
>3. weak_ptr可以用于解决shared_ptr的循环引用问题，避免内存泄漏。
>4. 普通指针没有自动内存管理的功能，需要手动分配和释放内存，容易出现内存泄漏和悬挂指针等问题。

悬挂指针：指向已经被释放空间的指针

shared_ptr的循环引用：如果存在循环引用，即两个或多个对象相互持有对方的shared_ptr，那么它们都无法被释放，因为它们的引用计数永远不会变为0。这时候，可以将其中一个shared_ptr改为weak_ptr，这样它不会增加对象的引用计数

GCC是GUN的编译工具集，可以编译多种语言，gcc是c++的编译器（一般用大小写区分）

## Webserver介绍
使用自制的小型仿muduo库编写的Web服务器，实现解析get、head请求，可处理静态资源，支持HTTP长连接，实现异步日志。
### 项目特色
* 具有仿muduo库的全部特征
* 使用多线程分别处理请求，具体实现方式为创建一定容量的线程池
* 构建状态机解析HTTP请求
   
### HTTP(HyperText Transform Protocal)超文本链接传输协议详解
1. HTTP服务器：相当于一个中转站，在用户端和客户端之间传递信息，但是又有一些独特的功能：
   >负载均衡：当Web应用程序或API的访问量较大，单个服务器可能无法承受所有的请求。HTTP服务器可以通过负载均衡的方式将请求分发到多个服务器上，以提高系统可用性和性能。

   >缓存：HTTP服务器可以对请求进行缓存，以减少对Web应用程序或API的请求次数，提高响应速度和性能。

   >安全性：HTTP服务器可以对请求进行过滤、验证和授权，以确保请求的安全性和合法性。

   >日志记录：HTTP服务器可以记录请求和响应的日志，以便进行监控和分析。

   >网络拓扑：HTTP服务器可以作为Web用程序或API的前置节点，将请求从公网转发到内部网络，以保护内部网络的安全性。
2. HTTP版本
   >0.9：是最早的HTTP协议版本，只支持GET方法，没有请求头和响应头，也没有状态码。它的主要特点是简单、轻量级，但功能有限，已经很少使用。

   >1.0：是第一个正式的HTTP协议版本，支持多种HTTP方法，引入了请求头、响应头和状态码等概念。它的主要特点是简单、易于实现，但性能较差，因为每个请求都需要建立一个TCP连接。

   >1.1：是目前最广泛使用的HTTP协议版本，引入了持久连接、管线化、分块传输编码、缓存等特性，可以提高性能和效率。它的主要特点是复杂、功能强大，但存在一些安全和性能问题。
   
   >2.0：是HTTP/1.1的升级版本，引入了二进制分帧、多路复用、头部压缩等特性，可以进一步提高性能和效率。它的主要特点是复杂、功能强大，但需要支持TLS加密。
   
   >3.0：是HTTP/2的升级版本，使用QUIC协议作为传输层协议，可以提高性能和安全性。它的主要特点是快速、安全，但需要支持新的传输层协议。

   >HTTPS：是HTTPx版本加上SSL/TLS协议而成的。SSL（Secure Sockets Layer）和TLS（Transport Layer Security）协议是一种用于保护网络通信安全的协议。它们主要用于在客户端和服务器之间建立安全的加密连接，以保护数据的机密性、完整性和真实性。SSL协议最初由Netscape公司开发，用于保护Web浏览器和Web服务器之间的通信安全。后来，SSL协议被TLS协议所取代，TLS协议是SSL协议的升级版本，它在SSL协议的基础上进行了改进和扩展，包括支持更多的加密算法、更好的握手协议、更严的证书验证等。SSL/TLS协议的主要功能包括：1. 加密：使用公钥加密技术对数据进行加密，保数据传输的机密性。2. 认证：使用数字证书对服务器和客户端进行身份验证，保证数据传输的真实性和完整性。3. 握手协议：使用握手协议对加密算法、密钥等进行协商，确保双方使用相同的加密算法和密钥4. 会话管理：使用会话ID和会话票据等机制对会话进行管理，提高性能和效率。SSL/TLS协议广泛应用于Web浏览器、电子邮件、即时通讯、VPN等网络通信场景中，是保护网络通信安全的重要手段。
3. HTTP缺陷
   >在许多应用场景中，我们需要保持用户登录的状态或记录用户购物车中的商品。由于HTTP是无状态协议，所以必须引入一些技术来记录管理状态，例如Cookie。
4. 相关名词
   > URL(Uniform Resource Locator)统一资源定位符
5. 常见操作
   > GET：从服务器获取资源并将其返回给客户端。这是HTTP协议中最常见的操作。

   > POST：将数据发送到服务器以创建或更新资源。

   > PUT：将数据发送到服务器以替换现有资源。

   > DELETE：从服务器删除指定的资源。

   > HEAD：与GET类似，但不返回响应体，只返回头信息。通常用于获取资源的元数据而不是实际的内容。

   > OPTIONS：请求服务器提供可用的操作和参数。

   > TRACE：回显服务器收到的请求，用于调试和测试。

   > CONNECT：建立与服务器的网络连接，主要用于HTTPS代理。

   > POST和PUT是HTTP请求方法，它们都用于向服务器提交数据。它们之间的主要区别在于：POST用于创建新资源，而PUT用于更新已有资源。当客户端使用POST发送请求时，服务器通常会在请求的URI下创建一个新资源，并返回包含新资源URI的响应。而当客户端使用PUT发送请求时，服务器通常会将请求的主体内容存储在指定的URI下，如果该URI不存在，则创建一个新资源。使用POST时，服务器可能对接收到的数据进行处理，例如对请求进行验证或过滤。而使用PUT时，服务器不会对数据进行任何处理，只是简单地替换原始资源的内容。POST可以连续多次提交相同的数据，并且每次提交的结果可能不同。而PUT被视为幂等操作（即多次执行具有相同结果），因此多次提交相同的数据不会产生不同的结果。如果使用POST来更新一个资源，而不是使用PUT，那么每次执行该操作时，服务器将在原始资源的旁边创建一个新资源，而不是更新原始资源的内容。这样会导致资源URI和其它相关信息发生变化，从而给客户端带来不必要的复杂性和额外的工作量。此外，在RESTful架构中，PUT被视为幂等操作，即多次执行具有相同的结果。这意味着如果客户端发送相同的PUT请求多次，服务器只会更新一次资源，并返回相同的响应。而使用POST则不能保证这种幂等性，因为POST请求的结果可能取决于先前已经执行的操作。
### [常见问题](./muduo.md)
### 友情鸣谢
1. #### @linyacool | WebServer
2. #### 《MuduoManual》
3. #### 知乎 & 简书 | Reactor模式 
4. #### 掘金 | muduo实践脑图汇总
5. #### 知乎 | 深入理解HTTP协议 
6. #### 《Linux多线程服务端编程》
7. #### 《C++ Primer》
8. #### [muduo库代码详解](./muduo.md)
> https://zhuanlan.zhihu.com/p/45173862
>
> https://github.com/834810071/muduo_study/blob/master/book_study/%E9%97%AE%E9%A2%98%E8%AE%B0%E8%BD%BD/%E7%9B%B8%E5%85%B3%E9%9D%A2%E8%AF%95%E9%97%AE%E9%A2%98%E8%AE%B0%E8%BD%BD.md
> 
> https://blog.csdn.net/weixin_46272577/article/details/128295509
> 
> https://github.com/Shangyizhou/A-Tiny-Network-Library/blob/main/%E9%A1%B9%E7%9B%AE%E8%AE%B2%E8%A7%A3/

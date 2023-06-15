## 主要实现类
### Channel类
1. Channel类其实相当于一个文件描述符的保姆！

2. 在TCP网络编程中，想要IO多路复用监听某个文件描述符，就要把这个fd和该fd感兴趣的事件通过epoll_ctl注册到IO多路复用模块（我管它叫事件监听器）上。当事件监听器监听到该fd发生了某个事件。事件监听器返回 [发生事件的fd集合]以及[每个fd都发生了什么事件]

3. Channel类则封装了**一个** [fd] 和这个 [fd感兴趣事件] 以及事件监听器是否监听到 [该fd实际发生的事件]。同时Channel类还提供了设置该fd的感兴趣事件，以及将该fd及其感兴趣事件注册到事件监听器或从事件监听器上移除，以及保存了该fd的每种事件对应的处理函数。

#### Channel类重要的成员变量：
- int fd_这个Channel对象照看的文件描述符
- int events_代表fd**感兴趣的事件类型**集合
- int revents_代表事件监听器**实际监听到该fd发生的事件类型**集合，当事件监听器监听到一个fd发生了什么事件，通过Channel::set_revents()函数来设置revents值。
- EventLoop* loop 表示这个fd属于哪个EventLoop对象[1], read_callback_ 、write_callback_、close_callback_、error_callback_：这些是std::function类型，代表着这个Channel为这个文件描述符保存的各事件类型发生时的处理函数。比如这个fd发生了可读事件，需要执行可读事件处理函数，这时候Channel类都替你保管好了这些可调用函数，真是贴心啊，要用执行的时候直接管Channel要就可以了。
#### Channel类重要的成员函数
- 向Channel对象注册各类事件的处理函数
  
	```void setReadCallback(ReadEventCallback cb) {read_callback_ = std::move(cb);}```

	```void setWriteCallback(Eventcallback cb) {write_callback_ = std::move(cb);}```

	```void setCloseCallback(EventCallback cb) {close_callback_ = std::move(cb);}```

	```void setErrorCallback(EventCallback cb) {error_callback_ = std::move(cb);} ```

- 一个文件描述符会发生可读、可写、关闭、错误事件。当发生这些事件后，就需要调用相应的处理函数来处理。外部通过调用上面这四个函数可以将事件处理函数放进Channel类中，当需要调用的时候就可以直接拿出来调用了。
- 将Channel中的**文件描述符**及其感兴趣事件**注册**事件监听器上或从事件监听器上**移除**
  
	```void enableReading() {events_ |= kReadEvent; upadte();}```

	```void disableReading() {events_ &= ~kReadEvent; update();}```

	```void enableWriting() {events_ |= kWriteEvent; update();}```

	```void disableWriting() {events_ &= ~kWriteEvent; update();}```

	```void disableAll() {events_ |= kNonEvent; update();}```
- 外部通过这几个函数来告知Channel你所监管的文件描述符都对哪些事件类型感兴趣，并把这个文件描述符及其感兴趣事件注册到事件监听器（IO多路复用模块）上。这些函数里面都有一个update()私有成员方法，这个update其实本质上就是调用了epoll_ctl()。
  

- ```int set_revents(int revt) {revents_ = revt;}```
当事件监听器监听到某个文件描述符发生了什么事件，通过这个函数可以将这个文件描述符实际发生的事件封装进这个Channel中。这里显然是边沿触发

> fd感兴趣的事件：指的是文件描述符（fd）所关注的网络事件类型，比如可读、可写、异常等事件。在网络编程中，程序需要监听这些感兴趣的事件类型，以便及时处理发生的网络事件。因此，在创建Channel对象时，需要指定该fd所关注的网络事件类型。例如，如果一个Socket对象需要监听可读事件和异常事件，那么对应的Channel对象就需要设置该fd关注EPOLLIN和EPOLLPRI事件。在Linux中，常见的网络事件类型包括以下几种：
> - EPOLLIN：表示该文件描述符可读（有数据可以读取）。
> - EPOLLOUT：表示该文件描述符可写（能够发送数据或者缓冲区已经空闲）。
> - EPOLLRDHUP：表示对端关闭连接或者半关闭（仅限于使用TCP协议的套接字）。
> - EPOLLPRI：表示有紧急数据可读（带外数据）。
> - EPOLLERR：表示发生错误（例如连接重置等）。
> - EPOLLHUP：表示连接被挂起或者发生了某些异常情况。
> - EPOLLET：表示边缘触发模式（Edge Triggered），即只有当状态发生变化时才会触发事件。
> - EPOLLONESHOT：表示单次触发模式（One Shot），即一旦事件被触发就会自动移除该fd的监听。
> - 需要注意的是，不同的事件类型只能用于特定类型的文件描述符上，例如EPOLLPRI只用于Socket的带外数据，而EPOLLOUT只用于Socket的连接建立时。因此，在设置Channel对象的事件类型时，需要根据具体场景进行选择。

> ```int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);```
> 
> 其中，参数含义如下：
> - epfd：epoll实例的文件描述符。
> - op：表示要执行的操作类型，可以取值EPOLL_CTL_ADD（添加）、EPOLL_CTL_MOD（修改）或EPOLL_CTL_DEL（删除）。
> - fd：需要添加、修改或删除的文件描述符。
> - event：一个指向epoll_event结构体的指针，该结构体描述了要监听的事件类型和与其相关联的数据。
> - 如果调用成功，epoll_ctl函数返回0，否则返回-1，并设置errno变量以指示错误原因。一些常见的错误原因包括：epfd或fd无效、op不合法等。
> - 使用epoll_ctl函数，可以方便地控制epoll实例中的文件描述符和其对应的事件类型，从而实现高效的IO多路复用。

>epoll是什么：epoll是Linux内核提供的一种I/O多路复用机制，它可以通过同时监视多个文件描述符的就绪状态来提高系统效率和性能。相比于传统的select和poll等机制，epoll具有更高的效率和更强的扩展性。
> - 在使用epoll时，应用程序可以将需要监视的文件描述符添加到一个epoll句柄（epoll_create）中，并通过epoll_ctl函数向该句柄注册需要监视的事件类型和回调函数。然后，应用程序可以使用epoll_wait函数等待事件的发生，并处理返回的活跃事件列表。
> - epoll支持两种触发模式：边缘触发（Edge Triggered）和水平触发（Level Triggered）。在边缘触发模式下，只有当文件描述符状态从未就绪变为就绪时才会触发一次事件通知，而在水平触发模式下，只要文件描述符处于就绪状态，每次epoll_wait都会返回该文件描述符。
> - 相比于传统的select和poll等机制，epoll具有以下优势：
> 	- 没有最大文件描述符限制；
> 	- 可以按照需求只关注感兴趣的事件
> 	- 支持边缘触发和水平触发两种触发模式；
> 	- 内核态与用户态之间的数据拷贝次数更少，效率更高。
> - 内核态和用户态之间为什么会发生数据拷贝：
> 	- 内核态与用户态之间发生数据拷贝的原因是由于它们分别处于不同的地址空间，而且是由不同的特权级在访问。
> 	- 在Linux系统中，应用程序运行在用户态，而内核运行在特殊的内核态。因此，当应用程序需要进行系统调用时，必须切换到内核态才能执行相应的操作。
> 	- 系统调用的过程大致可以分为以下几个步骤：首先，应用程序通过系统调用号和参数向内核发送请求；然后，内核根据请求处理相应的操作，并返回结果给应用程序；最后，应用程序根据返回值进行相应的处理。在这个过程中，由于内核态和用户态的地址空间是相互独立的，因此数据需要在两个地址空间之间进行传输。在数据传输过程中，就会发生数据拷贝的情况，即将数据从用户态复制到内核态，在内核态进行相应的操作，然后再将结果从内核态复制回用户态，供应用程序使用。这种数据拷贝的操作会带来一定的性能开销，尤其在频繁的数据传输场景下更加明显。
> 	- 为了解决这个问题，Linux系统引入了零拷贝（Zero Copy）技术，通过直接在内存中共享数据来避免数据拷贝的过程，以提高系统效率和性能。而epoll也是基于这种技术来实现I/O多路复用机制的。
> - 水平触发（Level Triggered）模式是一种I/O多路复用的工作模式，与边缘触发（Edge Triggered）模式相对应。
> 	- 在水平触发模式下，只有当文件描述符的状态发生变化时才会触发I/O事件通知，并返回该文件描述符。
> 	- 在使用水平触发模式时，如果某个文件描述符处于就绪状态，那么每次调用epoll_wait函数都会返回该文件描述符，应用程序需要持续地进行处理并读取所有可用数据，以充分利用I/O多路复用机制提高系统效率和性能。
> 	- 相比之下，在边缘触发模式下，只有当文件描述符的状态从未就绪变为就绪时才会触发I/O事件通知。这样可以避免因为读取不完整的数据、部分数据已经被处理而重复触发事件通知。但是，在使用边缘触发模式时，应用程序需要确保及时处理已经就绪的事件，否则可能会导致数据丢失或者延迟处理。
> 	- 因此，选择水平触发模式还是边缘触发模式，需要根据具体的应用场景和需求进行选择。通常来说，对于数据量小、响应速度快的场景，选择水平触发模式更加合适；对于数据量大、处理时间长的场景，选择边缘触发模式会更好一些。

## 执行流程
1. 首先创建一个主线程MainReactor，这个线程中有一个EventLoop(one loop per thread)，然后构建自己的IP地址等，封装为Addr类，创建一个TCPServer对象，用Addr，loop，fd等初始化该TCPServer
2. 一开始MainReactor会创建一个所谓的线程池，用map来记录新创建的线程，采用懒创建方式，我们并不是一开始就创建好了线程，只是给出一个可以存在的线程上限，当需要创建SubReactor时，才会对应创建线程，对应的线程函数可以记为TCPConnectionx，到后面调用某个SubReactor来执行
3. 初始化Server时，在类里实例化一个Acceptor分派器对象，这个类负责将监听到的连接均衡分配给一个SubReactor，具体执行函数为HandleRead
4. 实例化Acceptor就是在类里实例化一个Channel，将传进来的fd封装起来，感兴趣的事件为是否可读（证明是否已经建立连接），将该Channel绑定到MainReactor的EventLoop里，作为其中一个fd
5. 现在MainReactor执行loop操作，loop会循环调用epoll（其实就是listen）函数，负责监听注册到loop上的fd，当检测到可读信号，返回vector<Channel*>，这里的Channel就是可以激活的Channel，在Channel已经绑定好了要执行的回调函数，就是Acceptor分配操作，这时候，会将对方的Socket，本地fd，对方传来的回调函数（希望建立连接后执行的函数）和Channel作为参数，建立SubReactor线程去处理连接完成后的操作
6. 连接完成后的操作，依旧是SubReactor创建一个Loop，只不过这时候的注册fd只有一个，因为一个线程只检测一个连接，这个连接已经建立，fd就是本地文件，且回调函数不是MainReactor默认的Acceptor函数，而是由客户端注册的回调函数，对方在建立连接后，可以通过Loop来更改感兴趣的事件和相应的回调函数，Loop会调用set或者是remove函数去修改注册到它上面的Channel，Channel其实就是一个封装了一个fd的操作类，它会执行Loop给与的修改感兴趣事件和对应回调函数的操作，Loop一直循环，就和MainReactor一样，循环一次之后，给出一个vector<Channel*>，代表当前激活的fd和相应的事件与回调函数，对应事件和回调函数已经被注册到fd里，与MainReactor不同的是，这里的Channel集合中不会有多个fd（因为只有一个连接好的fd），但是会有多个事件。
7. 我们的服务器并不主动关闭连接，因为如果读此作，我们要自顶向下去找到最底层的对象，然后逐步删除，但这样会有安全隐患，因为可能在你关闭服务器的时候，其他线程正在处理TcpConnection的发送消息任务，这个时候你应该等它发完才释放TcpConnection对象的内存才对，否则写入的时候会出现异常，一般是客户机主动关闭，我们的SubReactor的readv函数在一段时间内未检测到数据，就可以认为对方关闭，或者对方发来fin请求（我们认为这是一个事件，就相当于客户机要将此事件注册到Loop上，且此事件认为一直活跃，就是说在注册成功的一瞬间，我们就认为该链接可以关闭，可以执行默认的Close函数）TcpServer::~TcpServer()函数开始把所有TcpConnection对象都删掉。那么其他线程还在使用这个TcpConnection对象，如果你把它的内存空间都释放了，其他线程访问了非法内存，会直接崩溃。

## 常见问题
1. 在muduo库中，如何实现TCP连接的数据压缩和加密传输？

2. muduo库中的EpollPoller类是如何进行事件管理和处理的？

3. 在muduo库中，如何实现TCP连接的断点续传和文件传输？

4. muduo库中的TcpServer类是如何进行负载均衡算法的优化和扩展？

5. 在muduo库中，如何实现TCP连接的消息队列和异步通信？

6. muduo库中的Buffer类是如何进行高效内存管理和回收的？

7. 在muduo库中，如何实现TCP连接的流量控制和拥塞控制算法？

8. muduo库中的EventLoopThreadPool类是如何管理线程池中的事件循环的？

9. 在muduo库中，如何实现TCP连接的快速重传和拥塞避免策略？

10. muduo库中的TimerId类是如何管理定时器ID的生成和回收的？

11. 在muduo库中，如何实现TCP连接的多路复用和同步/异步模式切换？

12. muduo库中的ThreadLocal类是如何进行线程局部变量的管理的？
    
13. 在muduo库中，如何实现TCP连接的多路径数据传输和负载均衡选择？

14. muduo库中的Atomic类是如何进行高效的原子操作的？

15. 在muduo库中，如何实现TCP连接的零拷贝和高效数据传输？
    
16. muduo库中的ProcessInfo类是如何获取进程信息和系统资源使用情况的？
    
17. 在muduo库中，如何实现TCP连接的流式传输和缓存管理？
    
18. muduo库中的LogFile类是如何进行日志文件的创建和管理的？
    
19. 在muduo库中，如何实现TCP连接的多路复用和调度策略？
    
20. muduo库中的LogStream类是如何进行日志输出和格式化的？
    
21. 在muduo库中，如何实现TCP连接的断线重连和恢复机制？
    
22. muduo库中的TcpClient类是如何进行连接管理和维护的？
    
23. 在muduo库中，如何实现TCP连接的优雅关闭和半关闭状态？
    
24. muduo库中的Exception类是如何进行异常处理和错误报告的？
    
25. 在muduo库中，如何实现TCP连接的数据采样和流量统计？
    
26. muduo库中的ThreadPool类是如何进行线程池任务调度和管理的？
    
27. 在muduo库中，如何实现TCP连接的异步读写和消息处理？
    
28. muduo库中的FileUtil类是如何进行文件读写和文件描述符管理的？
    
29. 在muduo库中，如何实现TCP连接的质量控制和优化？
    
30. muduo库中的RpcChannel类是如何进行远程过程调用和消息编解码的？
    
31. muduo库中的Reactor模式是什么？它有什么优点？
    
32. 在muduo库中，如何实现多线程并发处理网络请求？
    
33. muduo库中的Buffer类是什么？它的作用是什么？
34. muduo库中的TimerQueue是什么？它是如何实现定时器功能的？
35. 在muduo库中，如何进行日志记录？它的设计思路是什么？
36. muduo库中的Acceptor和TcpServer类分别是什么？它们的作用是什么？
37. 在muduo库中，如何实现TCP粘包和拆包？
38. muduo库中的EventLoop类是什么？它的作用是什么？
39. 在muduo库中，如何实现异步日志记录？
40. muduo库中的ThreadPool类是什么？它的作用是什么？
41. muduo库中的Channel类是什么？它的作用是什么？
42. 在muduo库中，如何实现TCP连接的生命周期管理？
43. muduo库中的TcpConnection类是什么？它的作用是什么？
44. 在muduo库中，如何进行跨线程的异步调用？
45. muduo库中的Buffer类是如何实现零拷贝的？
46. 在muduo库中，如何进行TCP数据的发送和接收？
47. muduo库中的TcpServer类是如何处理新连接的？
48. muduo库中的EventLoopThreadPool类是什么？它的作用是什么？
49. 在muduo库中，如何实现定时器的精度？
50. muduo库中的TcpClient类是什么？它的作用是什么？
51. 在muduo库中，如何进行TCP连接的优雅关闭？
52. muduo库中的InetAddress类是什么？它的作用是什么？
53. muduo库中的Buffer类是如何进行内存管理的？
54. 在muduo库中，如何处理TCP连接断开后的资源回收？
55. muduo库中的Connector类是什么？它的作用是什么？
在muduo库中，如何处理TCP连接的超时与心跳？
muduo库中的Poller类是什么？它的作用是什么？
在muduo库中，如何进行多线程间的数据共享？
muduo库中的TcpClient类是如何处理连接失败的情况？
在muduo库中，如何进行CPU亲和性的设置？
在muduo库中，如何处理TCP连接的半关闭状态？
muduo库中的TcpServer类是如何进行负载均衡的？
muduo库中的Timestamp类是什么？它的作用是什么？
在muduo库中，如何对TCP连接进行性能优化？
muduo库中的RpcChannel类是什么？它的作用是什么？
在muduo库中，如何进行TCP连接的优化以提高网络吞吐量？
muduo库中的Socket类是什么？它的作用是什么？
在muduo库中，如何进行多线程间的同步与互斥？
muduo库中的TcpClient类是如何处理连接重试的情况？
在muduo库中，如何进行内存泄漏检测与排查？
在muduo库中，如何处理TCP连接的错误与异常？
muduo库中的SocketOps类是什么？它的作用是什么？
muduo库中的ThreadLocal类是什么？它的作用是什么？
在muduo库中，如何进行TCP连接的限流与防止DDoS攻击？
muduo库中的TcpClient类是如何处理连接超时的情况？
在muduo库中，如何进行内存池管理以提高性能？
muduo库中的TimerId类是什么？它的作用是什么？
在muduo库中，如何处理TCP连接的Keep-Alive机制？
muduo库中的SocketsOps类是如何进行网络字节序和主机字节序的转换的？
在muduo库中，如何进行TCP连接的优先级设置？
muduo库中的EventLoopThread类是什么？它的作用是什么？
在muduo库中，如何进行TCP连接的负载均衡以及故障恢复？
muduo库中的EpollPoller类是什么？它的作用是什么？
在muduo库中，如何进行TCP连接的数据加密与解密？
muduo库中的Atomic类是什么？它的作用是什么？
在muduo库中，如何进行TCP连接的Nagle算法优化？
muduo库中的Thread类是什么？它的作用是什么？
在muduo库中，如何避免网络拥塞导致TCP连接的性能下降？
muduo库中的EventLoopThread类是如何进行线程池管理的？
在muduo库中，如何进行TCP连接的QoS质量保证？
muduo库中的LogStream类是什么？它的作用是什么？
在muduo库中，如何实现TCP连接的自适应流控制？
muduo库中的ProcessInfo类是什么？它的作用是什么？
在muduo库中，如何进行TCP连接的动态调整窗口大小以适应网络状况？
muduo库中的AsyncLogging类是什么？它的作用是什么？
在muduo库中，如何进行TCP连接的网络诊断与故障排除？
muduo库中的LogFile类是什么？它的作用是什么？
在muduo库中，如何进行TCP连接的错误恢复和重试？
muduo库中的FileUtil类是什么？它的作用是什么？
在muduo库中，如何进行TCP连接的协议栈优化以提高性能？
在muduo库中，如何进行TCP连接的网络拓扑发现和优化？
muduo库中的TcpConnection类是如何处理TCP连接的流量控制和拥塞控制？
muduo库中的EventLoop类是如何处理事件循环中的定时器任务？
在muduo库中，如何进行TCP连接的丢包处理和重传机制？
muduo库中的Buffer类是如何进行数据压缩和解压缩的？
在muduo库中，如何进行TCP连接的路由选择和多路径探测？
muduo库中的TcpServer类是如何进行服务注册和发现？
在muduo库中，如何进行TCP连接的带宽分配和调度？
muduo库中的TimerQueue类是如何管理定时器队列的？
在muduo库中，如何进行TCP连接的多级队列调度以提高性能？
muduo库中的Timestamp类是如何进行时间戳的转换和计算的？
在muduo库中，如何进行TCP连接的质量评估和监控？
muduo库中的Atomic类是如何进行原子操作的？
在muduo库中，如何进行TCP连接的链路质量估计和优化？
muduo库中的Exception类是什么？它的作用是什么？
在muduo库中，如何进行TCP连接的多路径选择和切换？
muduo库中的ThreadPool类是如何处理线程池中的任务调度的？
在muduo库中，如何进行TCP连接的拥塞控制参数的动态优化？
muduo库中的Logger类是什么？它的作用是什么？
在muduo库中，如何进行TCP连接的可扩展性设计以支持大规模并发连接？
muduo库中的TcpClient类是如何进行连接管理和维护的？
在muduo库中，如何进行TCP连接的流式传输和数据分片？
muduo库中的Callback类是什么？它的作用是什么？
在muduo库中，如何进行TCP连接的负载均衡调度算法的优化？
muduo库中的InetAddress类是如何进行IP地址的转换和解析的？
在muduo库中，如何进行TCP连接的可靠传输和数据完整性检测？
muduo库中的Connector类是如何进行连接管理和维护的？
在muduo库中，如何进行TCP连接的安全性设计以防止网络攻击和恶意操作？
muduo库中的Acceptor类是什么？它的作用是什么？
在muduo库中，如何进行TCP连接的服务质量保证和性能优化？
## muduo库中的所有类
AcceptSocket：封装了 TCP 服务器的监听套接字。

Acceptor：连接请求接受类，封装了 socket 监听套接字的创建、绑定、监听等操作。

Any：类型不安全的任意值类，用于实现类似 C++17 中的 std::any 的功能。

Atomic：原子变量类，封装了多线程下的原子操作。

BlockingQueue：阻塞队列类，支持多个线程之间的同步操作。

Buffer：缓冲区类，封装了字节序列的读写操作，支持自动扩展空间。

Callbacks：回调函数类，封装了一组回调函数，用于处理 TcpConnection 的各种事件。

Channel：事件通道类，封装了文件描述符及其对应的事件，与 EventLoop 配合使用。

Condition：条件变量类，用于线程间的同步。

CountDownLatch：倒计时闩类，用于线程间的同步。

CurrentThread：当前线程信息类，提供了获取当前线程 ID 和栈回溯信息的功能。

Date：日期时间类，提供了日期时间的格式化和解析等功能。

Duration：时间间隔类，表示一个时间段。

EPollPoller：基于 epoll 的 I/O 多路复用类。

Entry：文件系统条目类，表示一个文件或目录。

EventLoop：事件循环类，封装了 epoll 的操作，处理 I/O 事件和定时器事件。

EventLoopThread：事件循环线程类，用于创建一个新的事件循环并在其中运行指定的任务。

EventLoopThreadPool：事件循环线程池类，管理一组事件循环线程。

InetAddress：IPv4 地址类，封装了 IPv4 地址和端口号，用于表示网络地址。

InetAddress6：IPv6 地址类，封装了 IPv6 地址和端口号，用于表示网络地址。

LogFile：日志文件类，负责将日志记录到磁盘上的文件中。

Logger：日志类，提供了日志记录的功能，支持配置不同级别的日志输出。

LogStream：日志流类，用于格式化日志记录。

McpThread：MCP 线程类，用于启动子线程并等待其完成。

Mutex：互斥量类，用于线程间的同步。

noncopyable：禁止拷贝类，用于实现对某个类的拷贝构造和赋值运算符进行禁用。

Poller：I/O 多路复用类，负责管理事件通道和 EventLoop 的交互。

ProcessInfo：进程信息类，提供了获取当前进程信息的功能。

Socket：socket 类，封装了 socket 文件描述符的创建、绑定、监听、接收连接等操作。

SocketsOps：socket 操作类，提供了一组封装了 socket 系统调用的函数。

TcpClient：TCP 客户端类，封装了 TCP 客户端的相关操作，包括连接建立、数据读写等。

TcpConnection：TCP 连接类，封装了 TCP 连接的状态和操作，包括连接建立、数据读写等。

TcpServer：TCP 服务器类，封装了服务器的启动、监听、连接管理等功能。

Thread：线程类，封装了线程的创建和执行操作。

ThreadLocal：线程局部存储类，封装了线程局部存储的操作，支持在多个线程之间共享数据。

ThreadPool：线程池类，管理一组线程，用于并发地执行任务。

ThreadpoolThread：线程池线程类，封装了线程池中每个线程的行为。

Timer：定时器类，封装了定时器的相关操作，如设置定时器时间、取消定时器等功能。

TimerId：定时器 ID 类，用于标识一个定时器。

TimerQueue：定时器队列类，管理所有定时器，并安排它们在适当的时候触发。

TimeZone：时区类，提供了时区相关的操作和转换。

Timestamp：时间戳类，封装了一个时间，支持比较、转换等操作。

Types：类型类，定义了一些 muduo 库中用到的基本类型和常量。

UdpClient：UDP 客户端类，封装了 UDP 客户端的相关操作，包括数据读写等。

UdpServer：UDP 服务器类，封装了 UDP 服务器的相关操作，包括数据读写等。

Version：版本类，提供了库的版本信息。

WeakCallback：弱引用回调函数类，用于避免 TcpConnection 对象被过早地析构。

Zookeeper：Zookeeper 客户端类，支持连接 Zookeeper 服务器并进行数据操作。
## 参考
- [muduo核心代码概述](https://zhuanlan.zhihu.com/p/495016351)
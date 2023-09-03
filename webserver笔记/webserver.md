[TOC]

## 零拷贝技术

DMA 技术：在进行 I/O 设备和内存的数据传输的时候，数据搬运的工作全部交给 DMA 控制器，而 CPU 不再参与任何与数据搬运相关的事情，这样 CPU 就可以去处理别的事务。



在没有 DMA 技术前，I/O 的过程是这样的：

- CPU 发出对应的指令给磁盘控制器，然后返回；
- 磁盘控制器收到指令后，于是就开始准备数据，会把数据放入到磁盘控制器的内部缓冲区中，然后产生一个**中断**；
- CPU 收到中断信号后，停下手头的工作，接着把磁盘控制器的缓冲区的数据一次一个字节地读进自己的寄存器，然后再把寄存器里的数据写入到内存，而在数据传输的期间 CPU 是无法执行其他任务的。

![img](./webserver.assets/I_O 中断.png)





有了 DMA 技术之后，访问方式如下：

![img](./webserver.assets/DRM I_O 过程.png)

**CPU 不再参与「将数据从磁盘控制器缓冲区搬运到内核空间」的工作，这部分工作全程由 DMA 完成**。但是 CPU 在这个过程中也是必不可少的，因为传输什么数据，从哪里传输到哪里，都需要 CPU 来告诉 DMA 控制器。





- 对于网络传输，普通的文件传输需要4次数据复制。

![img](./webserver.assets/传统文件传输.png)





- 要以零拷贝的方式实现传输，有3种
  - mmap + write
  - sendfile
  - 硬件级别支持
- mmap + write 实现：
- ![img](./webserver.assets/mmap %2B write 零拷贝.png)

- `mmap()` 系统调用函数会直接把内核缓冲区里的数据「**映射**」到用户空间，这样，操作系统内核与用户空间就不需要再进行任何的数据拷贝操作。
- 这种方式可以减少一次数据复制。



- sendfile：
- ![img](./webserver.assets/senfile-3次拷贝.png)

- 减少一次上下文切换的开销。



- 如果是硬件级别支持，比如网络传输的过程中，网卡的 DMA 控制器可以直接从内核缓存区将数据复制至网卡缓冲区，再减少一次数据复制（减少了 内核缓冲区 -> socket 缓冲区）



## 总：项目介绍

### 1. 项目的整体流程

这个项目是我在学习计算机网络、多线程编程、socket 通信等内容时，结合这些知识点综合设计出的一个内容。服务器的网络模型是 模拟 proactor 模式 + 线程池的模式，IO 处理方面，使用了 非阻塞 IO 和 IO 多路复用技术，具备处理多个客户端 HTTP 请求的功能。



### 2. 项目中的优化

1. 对于程序本身的话，使用了 非阻塞 IO + 多路复用减少程序 IO 等待；
1. 在 epoll 的处理中，使用了 `EPOLLONESHOT` 避免一个 socket 事件被触发多次；
2. 用了线程池来避免频繁地申请和释放内存，并用单例模式来维护它；
3. 对于文件的发送，使用了 `sendfile` 系统调用，避免数据的频繁拷贝；
4. 使用 writev 一次写多个文件，避免频繁地系统调用开销。



### 3. 详细流程

首先是有关 socket 初始化方面的内容：

- 创建了单例模式下的线程池

- 初始化 socket 链接，获取文件描述符：

  - 初始化 socket，得到文件描述符
  - 调用 bind，绑定 IP 和端口
  - 调用 setsockopt，做到断开链接后端口复用，提升调试效率
  - 调用 listen 进行监听

- 调用 `epoll_create` 创建 epoll 对象，获取 epoll_fd；创建 epoll_event 结构体数组，用于保存事件

- 将监听描述符绑定到 epoll 树上，设置监听方法为 `EPOLLIN` 和 `EPOLLRDHUP` （读事件 和 用于检测对方异常断开）

- 然后进入主程序，一个大循环：使用 `epoll_wait` 阻塞当前运行，检测创建的 epoll 实例中有没有就绪的文件描述符。该函数会返回检测到发生变化的文件描述符的个数。

  ```cpp
  int number = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1);
  /*
  events：传出参数, 这是一个结构体数组的地址, 里边存储了已就绪的文件描述符的信息
  maxevents：修饰第二个参数, 结构体数组的容量（元素个数）
  timeout：如果检测的epoll实例中没有已就绪的文件描述符，该函数阻塞的时长, 单位ms 毫秒
  0：函数不阻塞，不管epoll实例中有没有就绪的文件描述符，函数被调用后都直接返回
  大于0：如果epoll实例中没有已就绪的文件描述符，函数阻塞对应的毫秒数再返回
  -1：函数一直阻塞，直到epoll实例中有已就绪的文件描述符之后才解除阻塞
  */
  ```

- 然后依次遍历本次事件中触发的文件描述符，进行判断。如果是 socket 文件描述符发生了变化，说明有新连接建立了，调用 accept 返回新的文件描述符，并挂载回 epoll 树上；如果是 `EPOLL_IN` 或者 `EPOLL_OUT`，则在主线程中处理读写，将fd中的数据储存到读写缓冲区中。然后将回写发送的任务添加到线程池中。



- 本项目中，线程池主要负责处理将 HTTP 报文回送给客户端的工作。工作流程是这样的：
  - 主要是维护了一个 HTTP 状态机，用状态机的状态来维护 HTTP 报文解析的阶段，状态分为：解析请求行、解析请求头、解析请求体、解析结束；
  - **请求行** 主要是分析出解析方法、请求路径、HTTP 版本；
  - **请求头** 主要是解析了 Content-Length 字段。
  - **请求体** 主要是解析了 POST 信息。
- 在子线程解析的过程中，HTTP 状态机根据当前状态进行解析，解析完毕后保存相应的解析结果，并转移状态。
- 当 HTTP 报文的请求全部解析完毕后，进行回传工作：
  - 判断请求的路径是文件 or 目录 (`S_ISDIR`)，如果是文件就直接获取该文件的 fd，用 `mmap` 系统调用将文件映射到内存；如果是目录就遍历整个目录中的文件，回写一个网页。
  - 回写，实际上就是生成响应：根据解析 HTTP 请求报文得到的结果，进行相应处理。如果请求报文里得到的结果不正常 （INTERNAL_ERROR、BAD_REQUEST、NO_RESOURCE 等等），则回写一个简单的响应报文；
  - 如果正常（200 OK），则将要回写的请求头 和 文件数据，储存在 缓冲区中。准备好后，将 fd 挂载到 epoll 树上。
  - 然后，epoll 树会检测到这个写事件，使用 writev 系统调用将数据写入 fd 中。



### 4. 线程池

- 本项目中，**主线程**用于维护 epoll 数据结构，在 epoll 监听的文件描述符发生变化后，将数据从 fd 读入缓冲区，或将数据写入 fd。
- **子线程**用于处理 HTTP 请求报文，即，从读缓冲区中解析文本，根据请求的方式、路径，找到相应的文件或目录，写入读缓冲区。
- 线程池本质上是一个生产者 消费者模型，生产者就是主线程，它会将报文数据加入到任务队列中。消费者就是各个子线程，从任务队列里取出任务，进行业务逻辑的处理。

分工：

- 工作的线程（任务队列任务的消费者） ，N个
  - 如果任务队列为空, 工作的线程将会被阻塞 (使用条件变量阻塞)
  - 如果阻塞之后有了新的任务, 由生产者将阻塞解除, 工作线程开始工作
  - 子线程工作完后，如果队列为空，则重新回到阻塞等待的状态；否则这个线程会重新和其他线程竞争资源
- 主线程，1个
  - 将 epoll 接收到的 socket 通知处理后，往任务队列里添加任务，并使用条件变量发出通知，解除子线程的阻塞



### 5. HTTP 连接类

每一个新建立的连接都封装成了一个 HTTP 连接类，其中主要的成员有：

- socket 的文件描述符
- 读写缓冲区和偏移量
- 当前解析的状态

HTTP 连接类，在项目里的设计，主要是业务逻辑的处理。主线程会将报文数据从 fd 中保存至该类的缓冲区中，然后开启子线程来解析报文。解析的流程是这样的：

- 状态机的状态分别为：解析请求行、解析请求头、解析请求体、解析结束。根据这些状态，依次解析报文的内容，每解析一次就储存得到的关键信息（比如请求的路径、是 GET / POST、HTTP 版本、Content-Length 等）。
- 报文全部解析完毕后，准备好要回送的数据（根据请求路径，判断要发送的是文件还是目录），并按照格式写好响应报文
- 将该 fd 设置为 `EPOLL_OUT`，挂载到 epoll 树上，等待下一轮询触发，将数据写出。



### -1 项目中遇到的困难，如何解决？

#### 问题定位

像普通的，代码写错了、语义方面的普通错误，我就不多说了。我想说一个因为疏忽导致的 HTTP 报文收发不全的问题，顺便讲解一下我排查的心路历程。

在处理完项目的大致流程，可以正常收发 HTML 纯文字页面后，我就开始尝试在页面里添加其它的资源文件，比如图片等。然后发现如果客户端的请求如果带图片，图片就收不到了。

然后就开始分析这个过程，主要考虑三个方面：

- 检查服务器程序是否有问题（重点）；
- 通信连接是否正常；
- 会不会有什么前端知识我不了解的。



首先是检查代码，这一段主要是做了两项工作：

- 按照 epoll 处理连接，维护文件描述符的逻辑，详细地梳理了一遍代码，也打了很多 log，没找到问题；
- 检查线程池是否有问题，我还新建了一个项目，把线程池的代码拉出来单独测试，也没找到问题；



然后就怀疑是 TCP 连接是不是有问题了；

- 我的项目是在 Windows 下的 Linux 虚拟机子系统开发的，尝试把它移植到了真机上，还是有问题；
- 还尝试用了 Wireshark 抓包，没发现有什么明显问题；
- 然后还把返回的响应报文换成了固定的 html 内容，F12 打开浏览器的开发者工具，发现没问题；但是！在这里我发现，如果浏览器请求的页面，里面包含了多个资源（比如多个图片），浏览器会发送多条 HTTP 请求，而不是一条，在这里我发现，除了第一条以外，其它的图片请求都没有返回给浏览器。
- 然后推测是，由于访问的资源没有全部获取完，所以浏览器不会立即渲染文字部分，导致整个页面都无法被加载出来了。

**最终定位到问题：浏览器没有获取到全部的请求资源，导致页面无法被正确的加载。**



#### 问题原因思考

然后在这里就联想到了 HTTP/1.1 的长连接，里面的请求头有一个字段 Connection: keep-alive 使得浏览器访问网页时，会保持这个 TCP 连接不断开。

然后就感觉是第二个连接及以后的内容服务器都没有收到或者没有正确处理。然后以这个角度去看服务器的代码，发现第二个 HTTP 请求报文根本没收到。

这个时候再回顾一下 `epoll` 的工作流程：我的 epoll 采用的是 ET 边沿触发 + `EPOLLONESHOT` 避免通信的 socket 文件描述符被多个线程操作。使用 `EPOLLONESHOT` 的 socket，一旦被某个线程处理完毕后，这个线程要立刻重置这个 socket 上面的监听事件，确保下一次能够被触发。

所以问题最终就是我应该把这个 socket 文件描述符重新挂载到 epoll 树上。



### -2 并发性问题：如果同时1000个客户端进⾏访问请求，线程数不多，怎么能及时响应处理每⼀个呢？

这种问法就相当于问服务器如何处理⾼并发的问题。

直接体现就是项⽬中使⽤了 I/O 多路复⽤技术，让主线程在不阻塞的情况下接收多个 I/O 请求，然后将业务逻辑分发至不通的线程进行处理。如果并发量比较大，可以满足在同一时刻，每一个线程都在处理业务逻辑的请求的一个状态。

> 因为我是采用的 模拟 Proactor 模式，对缓冲区的读写实际上是在主线程上完成的，要说效率的话应该不算高，可能采用 Reactor 模式，让子线程去处理 socket 事件会效率更高一点。



### -3 如果一个客户请求要占用线程很久的时间，会不会影响接下来的请求？有什么好的策略？

按照我项目的设计，应该有两种情况：一个是主线程占用，一个是子线程占用。

如果是在主线程中耗时比较久，这说明是接收数据从 socket 复制到缓冲区，或者发送数据，把缓冲区的内容复制到 socket，而这些内容比较多。优化的方法，可以考虑用类似 Reactor 的模式结合，将 socket 的处理也分发到子线程中，可以一定程度上减少主线程的占用。然后在子线程中，开发类似断点续传的功能，设置响应报⽂的⼤⼩上限，当响应报⽂超出上限时，可以记录已经发送的位置，之后可以选择继续由该线程进⾏发送，也可以转交给其他线程进⾏发送。

如果是子线程占用的话，一般是在 HTTP 报文中，业务逻辑的处理遇上了问题。这个感觉好像不太会有这种情况，HTTP 报文的解析，一般来说只要那几个头部就行了吧。



## 一、IO 多路复用

### 1. 什么是多路复用

IO多路复用是一种网络通信的手段，可以同时监测多个文件描述符，并且这个过程是阻塞的，一旦检测到有文件描述符就绪（ 可以读数据或者可以写数据）程序的阻塞就会被解除，之后就可以基于这些（一个或多个）就绪的文件描述符进行通信了。通过这种方式在单线程/进程的场景下也可以在服务器端实现并发。

常见的IO多路转接方式有：select、poll、epoll。



**工作过程：**

- IO多路复用函数：委托内核检测服务器端所有的文件描述符
- 这个检测过程会导致进程/线程的阻塞，如果检测到已就绪的文件描述符阻塞解除，并将这些已就绪的文件描述符传出
- 根据类型对传出的所有已就绪文件描述符进行判断，并做出不同的处理
  - 监听的文件描述符：和客户端建立连接。此时调用accept()是不会导致程序阻塞的，因为监听的文件描述符是已就绪的（有新请求）
  - 通信的文件描述符：调用通信函数和已建立连接的客户端通信
- 对这些文件描述符进行下一轮检测



**和多进程相比的好处：**

I/O多路复用技术的最大优势是系统开销小，系统不必创建进程/线程，也不必维护这些进程/线程，从而大大减小了系统的开销。



### 2 事件处理模式：Proactor、Reactor 等

#### 2.0 问答：Reactor、Proactor 的区别

1. Reactor 是⾮阻塞同步⽹络模式，感知的是就绪可读写事件。在每次感知到有事件发⽣（⽐如可读就绪事件）后，就需要应⽤进程主动调⽤ read ⽅法来完成数据的读取，也就是要应⽤进程主动将 socket 接收缓存中的数据读到应⽤进程内存中，这个过程是同步的，读取完数据后应⽤进程才能处理数据。
2. Proactor 是异步⽹络模式， 感知的是已完成的读写事件。在发起异步读写请求时，需要传⼊数据缓冲区的地址（⽤来存放结果数据）等信息，这样系统内核才可以⾃动帮我们把数据的读写⼯作完成，这⾥的读写⼯作全程由操作系统来做，并不需要像 Reactor 那样还需要应⽤进程主动发起 read/write 来读写数据，操作系统完成读写⼯作后，就会通知应⽤进程直接处理数据。

**在项目中不采用 Proactor 的原因是，**在 Linux 下的异步 I/O 是不完善的， aio 系列函数不是真正的操作系统级别⽀持的，⽽是在⽤户空间模拟出来的异步，并且仅仅⽀持基于本地⽂件的 aio 异步操作，⽹络编程中的 socket 是不⽀持的。



#### 2.1 Proactor 模式

Proactor 模式将所有 I/O 操作都交给主线程和内核来处理（进行读、写），工作线程仅仅负责业务逻辑。一般是使用异步 I/O 模型（aio_read 和 aio_write 等）实现。工作流程通常是：

1. 主线程调用 aio_read 函数向内核注册 socket 上的读完成事件，并告诉内核用户读缓冲区的位置，以及读操作完成时如何通知应用程序（比如用信号）。
2. 当 socket 上的数据被读入用户缓冲区后，内核向应用程序发送一个信号，以通知应用程序数据已经可用。
3. 应用程序预先定义好的信号处理函数选择一个工作线程来处理客户请求。工作线程处理完客户请求后，调用 aio_write 函数向内核注册 socket 上的写完成事件，并告诉内核用户写缓冲区的位置，以及写操作完成时如何通知应用程序。
4. 当用户缓冲区的数据被写入 socket 之后，内核将向应用程序发送一个信号，以通知应用程序数据已经发送完毕。
7. 应用程序预先定义好的信号处理函数选择一个工作线程来做善后处理，比如决定是否关闭 socket。



#### 2.2 模拟 Proactor 模式

原理是：主线程执行数据读写操作，读写完成之后，主线程向工作线程通知这一“完成事件”。那么从工作线程的角度来看，它们就直接获得了数据读写的结果，接下来要做的只是对读写的结果进行逻辑处理。

工作流程如下：

1. 主线程往 epoll 内核事件表中注册 socket 上的读就绪事件。
2. 主线程调用 epoll_wait 等待 socket 上有数据可读。
3. 当 socket 上有数据可读时，epoll_wait 通知主线程。主线程从 socket 循环读取数据，直到没有更多数据可读，然后将读取到的数据封装成一个请求对象并插入请求队列。
4. 睡眠在请求队列上的某个工作线程被唤醒，它获得请求对象并处理客户请求，然后往 epoll 内核事件表中注册 socket 上的写就绪事件。
5. 主线程调用 epoll_wait 等待 socket 可写。
6. 当 socket 可写时，epoll_wait 通知主线程。主线程往 socket 上写入服务器处理客户请求的结果。



#### 2.3 Reactor 模式

主线程（I/O处理单元）只负责监听文件描述符上是否有事件发生，有的话就立即将该事件通知工作线程（逻辑单元），将 socket 可读可写事件放入请求队列，交给工作线程处理。除此之外，主线程不做任何其他实质性的工作。读写数据，接受新的连接，以及处理客户请求均在工作线程中完成。

工作流程是：

1. 主线程往 epoll 内核事件表中注册 socket 上的读就绪事件。
2. 主线程调用 epoll_wait 等待 socket 上有数据可读。
3. 当 socket 上有数据可读时， epoll_wait 通知主线程。主线程则将 socket 可读事件放入请求队列。
4. 睡眠在请求队列上的某个工作线程被唤醒，它从 socket 读取数据，并处理客户请求，然后往 epoll内核事件表中注册该 socket 上的写就绪事件。
5. 当主线程调用 epoll_wait 等待 socket 可写。
6. 当 socket 可写时，epoll_wait 通知主线程。主线程将 socket 可写事件放入请求队列。
7. 睡眠在请求队列上的某个工作线程被唤醒，它往 socket 上写入服务器处理客户请求结果。





### 3 三个复用函数

当监测的 fd 数量较小，且大多数 fd 都比较活跃的情况下，使用 select 和 poll 效果更好。当监听的 fd 数量较多，且一般的时间段中只有部分 fd 活跃的情况下，使用 epoll 性能更好。



#### 3.1 select

**==Brief==**

select 使⽤线性表描述⽂件描述符集合，⽂件描述符有上限。所有⽂件描述符都是在⽤户态被加⼊其⽂件描述符集合的，每次调⽤都需要将整个集合拷⻉到内核态。

select 和 poll 的最⼤开销来⾃内核判断是否有⽂件描述符就绪这⼀过程：每次执⾏ select 或 poll 调⽤时，它们会采⽤遍历的⽅式，遍历整个⽂件描述符集合去判断各个⽂件描述符是否有活动。



`int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval * timeout)`;

- 该函数跨平台
- 把读、写、异常的文件描述符，整合成三个集合
  - 当文件描述符对应的读缓冲区有数据，该读文件描述符就绪
  - 当文件描述符对应的写缓冲区不为 full，该写文件描述符就绪
  - 读写异常：当文件描述符对应的读写缓冲区有异常，该文件描述符就绪
- 委托检测的文件描述符被遍历检测完毕之后，已就绪的这些满足条件的文件描述符会通过select()的参数分3个集合传出。
- 最后一个参数是超时时长，用来强制解除select()函数的阻塞的。



**处理流程**

1. 创建监听的套接字 `lfd = socket()`;

2. 将监听的套接字和本地的IP和端口绑定 `bind()`
3. 给监听的套接字设置监听 `listen()`
4. 创建一个文件描述符集合 `fd_set`，用于存储需要检测读事件的所有的文件描述符
   - 通过 `FD_ZERO()` 初始化
   - 通过 `FD_SET()` 将监听的文件描述符放入检测的读集合中
5. 循环调用 `select()`，周期性的对所有的文件描述符进行检测
6. `select()` 解除阻塞返回，得到内核传出的满足条件的就绪的文件描述符集合
   - 通过 `FD_ISSET()` 判断集合中的标志位是否为 1
     - 如果这个文件描述符是监听的文件描述符，调用 `accept()` 和客户端建立连接
       - 将得到的新的通信的文件描述符，通过 `FD_SET()` 放入到检测集合中
   - 如果这个文件描述符是通信的文件描述符，调用通信函数和客户端通信
     - 如果客户端和服务器断开了连接，使用 `FD_CLR()` 将这个文件描述符从检测集合中删除
     - 如果没有断开连接，正常通信即可
7. 重复第6步



**局限性**

- 待检测集合（第2、3、4个参数）需要频繁的在用户区和内核区之间进行数据的拷贝，效率低

- 内核对于select传递进来的待检测集合的检测方式是线性的
  - 如果集合内待检测的文件描述符很多，检测效率会比较低
  - 如果集合内待检测的文件描述符相对较少，检测效率会比较高
- 使用select能够检测的最大文件描述符个数有上限，默认是1024，这是在内核中被写死了的。



### 3.2 poll

**==Brief==**

poll使⽤链表来描述，没有最大文件描述符数量的限制。



poll的机制与select类似，与select在本质上没有多大差别，使用方法也类似，下面的是对于二者的对比：

- 内核对应文件描述符的检测也是以线性的方式进行轮询，根据描述符的状态进行处理
- poll和select检测的文件描述符集合会在检测过程中频繁的进行用户区和内核区的拷贝，它的开销随着文件描述符数量的增加而线性增大，从而效率也会越来越低。
- select检测的文件描述符个数上限是1024，poll没有最大文件描述符数量的限制
- select可以跨平台使用，poll只能在Linux平台使用

函数原型：

```c
#include <poll.h>
// 每个委托poll检测的fd都对应这样一个结构体
struct pollfd {
    int   fd;         /* 委托内核检测的文件描述符 */
    short events;     /* 委托内核检测文件描述符的什么事件 */
    short revents;    /* 文件描述符实际发生的事件 -> 传出 */
};

struct pollfd myfd[100];
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

**函数参数：**

- fds: 这是一个 `struct pollfd` 类型的数组, 里边存储了待检测的文件描述符的信息，这个数组中有三个成员：

  - fd：委托内核检测的文件描述符
  - events：委托内核检测的fd事件（输入、输出、错误），每一个事件有多个取值
  - revents：这是一个传出参数，数据由内核写入，存储内核检测之后的结果
- nfds: 这是第一个参数数组中最后一个有效元素的下标 + 1（也可以指定参数1数组的元素总个数）



**与 select 比较**

- `poll`和`select`进行IO多路复用的处理思路是完全相同的，但是使用poll编写的代码看起来会更直观一些
- `select`使用的位图的方式来标记要委托内核检测的文件描述符（每个比特位对应一个唯一的文件描述符），并且对这个`fd_set`类型的位图变量进行读写还需要借助一系列的宏函数，操作比较麻烦
- 而poll直接将要检测的文件描述符的相关信息封装到了一个结构体`struct pollfd`中，可以直接读写这个结构体变量。



### 3.3 epoll

epoll 全称 eventpoll，是select和poll的升级版，改进了工作方式，因此它更加高效。

- 对于待检测集合`select`和`poll`是基于线性方式处理的，`epoll`是基于红黑树来管理待检测集合的。

- `select`和`poll`每次都会线性扫描整个待检测集合，集合越大速度越慢，`epoll`使用的是回调机制，效率高，处理效率也不会随着检测集合的变大而下降
- `select`和`poll`工作过程中存在内核/用户空间数据的频繁拷贝问题，在`epoll`中内核和用户区使用的是共享内存（基于mmap内存映射区实现），省去了不必要的内存拷贝。
- 程序猿需要对`select`和`poll`返回的集合进行判断才能知道哪些文件描述符是就绪的，通过`epoll`可以直接得到已就绪的文件描述符集合，无需再次检测

当多路复用的文件数量庞大、IO流量频繁的时候，一般不太适合使用select()和poll()，这种情况下select()和poll()表现较差，推荐使用epoll()。



```c
#include <sys/epoll.h>
// 创建epoll实例，通过一棵红黑树管理待检测集合
int epoll_create(int size);
// 管理红黑树上的文件描述符(添加、修改、删除)
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
// 检测epoll树中是否有就绪的文件描述符
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```



**为什么 select 和 poll 相对低效？** 

select/poll低效的原因之一是将“添加/维护待检测任务”和“阻塞进程/线程”两个步骤合二为一。每次调用select都需要这两步操作，然而大多数应用场景中，需要监视的socket个数相对固定，并不需要每次都修改。epoll将这两个操作分开，先用`epoll_ctl()`维护等待队列，再调用`epoll_wait()`阻塞进程（解耦）。



**调用流程**

`epoll_create()`：创建一个红黑树模型的实例，用于管理待检测的文件描述符的集合。



`epoll_ctl()`：管理红黑树实例上的节点，可以进行添加、删除、修改操作。

```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);

// 联合体, 多个变量共用同一块内存        
typedef union epoll_data {
 	void        *ptr;
	int          fd;	// 通常情况下使用这个成员, 和epoll_ctl的第三个参数相同即可
	uint32_t     u32;
	uint64_t     u64;
} epoll_data_t;

struct epoll_event {
	uint32_t     events;      /* Epoll events */
	epoll_data_t data;        /* User data variable */
};
```

- epfd：epoll_create() 函数的返回值，通过这个参数找到epoll实例

- op：这是一个枚举值，控制通过该函数执行什么操作
  - `EPOLL_CTL_ADD`：往epoll模型中添加新的节点
  - `EPOLL_CTL_MOD`：修改epoll模型中已经存在的节点
  - `EPOLL_CTL_DEL`：删除epoll模型中的指定的节点
- fd：文件描述符，即要添加/修改/删除的文件描述符
- event：epoll事件，用来修饰第三个参数对应的文件描述符的，指定检测这个文件描述符的什么事件
  - events：委托epoll检测的事件
    - `EPOLLIN`：读事件, 接收数据, 检测读缓冲区，如果有数据该文件描述符就绪
    - `EPOLLOUT`：写事件, 发送数据, 检测写缓冲区，如果可写该文件描述符就绪
    - `EPOLLERR`：异常事件
  - data：用户数据变量，这是一个联合体类型，通常情况下使用里边的fd成员，用于存储待检测的文件描述符的值，在调用epoll_wait()函数的时候这个值会被传出



`epoll_wait()`：检测创建的epoll实例中有没有就绪的文件描述符。

```c
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```





### 3.4 epoll 的工作模式

#### 3.4.1 水平模式 (LT)

LT：⽔平触发模式，只要内核缓冲区有数据就⼀直通知，只要 socket 处于可读状态或可写状态，就会⼀直返回 socket fd；是默认的⼯作模式，⽀持阻塞 IO 和⾮阻塞 IO。

具体来说，

读事件：如果文件描述符对应的缓冲区内还有数据，读事件就会触发，epoll_wait 解除阻塞。比如，接收到的数据的大小大于缓冲区时，读事件会被反复触发，直到数据被全部读出。

写事件：如果文件描述符对应的写缓冲区可写，写事件就会被触发，epoll_wait 解除阻塞。如果缓冲区没有写满，会被一直触发。**因为写数据是主动的，并且写缓冲区一般情况下都是可写的（缓冲区不满），因此对于写事件的检测不是必须的。**



#### 3.4.2 边沿模式 (ET)

ET：边沿触发模式，只有状态发⽣变化才通知并且这个状态只会通知⼀次，只有当socket由不可写到可写或由不可读到可读，才会返回其 sockfd；只⽀持⾮阻塞IO。

具体来说，

读事件：**当读缓冲区有新的数据进入，读事件被触发一次，没有新数据不会触发该事件。**因此，如果接收的数据长度大于缓冲区，它不会反复读出。

写事件：**当写缓冲区状态可写，写事件只会触发一次。**

- 如果写缓冲区被检测到可写，写事件被触发，epoll_wait() 解除阻塞

- 写事件被触发，就可以通过调用 write () / send() 函数，将数据写入到写缓冲区中
  - 写缓冲区从不满到被写满，期间写事件只会被触发一次
  - 写缓冲区从满到不满，状态变为可写，写事件只会被触发一次

综上所述：epoll 的边沿模式下 epoll_wait() 检测到文件描述符有新事件才会通知，如果不是新的事件就不通知，通知的次数比水平模式少，效率比水平模式要高。



#### 3.4.3 EPOLLONESHOT

一个 socket 上的某个事件可能在并发程序中被触发多次：比如一个线程在读取完某个 socket 上的数据后开始处理这些数据，而在数据的处理过程中该 socket 上又有新数据可读（EPOLLIN 再次被触发），此时另外一个线程被唤醒来读取这些新的数据，于
是就出现了两个线程同时操作一个 socket 的局面。这样显然是不科学的。

使用 `EPOLLONESHOT` 可以规避这个问题。注册了 `EPOLLONESHOT` 的文件描述符，操作系统最多触发其上注册的一个可读、可写或者异常事件。被触发后，如果需要再次触发，需要使用 `epoll_ctl` 重新注册事件。



#### 3.4.4 为什么 ET 模式不可以文件描述符阻塞，而 LT 可以

- 因为ET模式是当fd有可读事件时，epoll_wait()只会通知⼀次，如果没有⼀次把数据读完，那么要到下⼀次fd有可读事件epoll才会通知。⽽且在ET模式下，在触发可读事件后，需要循环读取信息，直到把数据读完。如果把这个fd设置成阻塞，数据读完以后read()就阻塞在那了。⽆法进⾏后续请求的处理。
- LT模式不需要每次读完数据，只要有数据可读，epoll_wait()就会⼀直通知。所以 LT模式下去读的话，内核缓冲区肯定是有数据可以读的，不会造成没有数据读⽽阻塞的情况。



## 二、多线程知识

- 线程的生命周期会随着主线程，主线程退出后，会销毁进程的全部地址空间，这意味着会把子线程一起销毁。使用 pthread_exit 可以在子线程中退出。
- 主线程和子线程之间的栈空间可以互相访问。
- 使用 pthread_detach 可以分离子线程，调用这个函数之后指定的子线程和主线程分离，当子线程退出的时候，其占用的内核资源就被系统的其他进程接管并回收了。线程分离之后在主线程中使用 pthread_join 就回收不到子线程资源了。





C++ with pthread 实现：https://subingwen.cn/linux/threadpool-cpp/





## 三、Linux 上的五种 I/O 模型

### 1. 阻塞

调用者调用了某个函数，等待这个函数返回，期间什么也不做，不停的去检查这个函数有没有返回，必须等这个函数返回才能进行下一步动作。

### 2. 非阻塞

非阻塞等待，应用进程在发起 IO 系统调用后会立刻返回，借助这个设计，可以让应用程序循环地轮询发起这个 IO 系统调用，根据内核返回的不同状态码，来进行相应的处理。比如 read，大于 0，就是读到了数据，EAGAIN 就是数据读取完毕等等。

### 3. IO 复用

Linux 用 select/poll/epoll 函数实现 IO 复用模型，这些函数也会使进程阻塞，但是和阻塞IO所不同的是这些函数可以同时阻塞多个IO操作。而且可以同时对多个读操作、写操作的IO函数进行检测。直到有数据可读或可写时，才真正调用IO操作函数。（检测多个文件描述符，并统一处理）

### 4. 信号驱动

应用进程需向内核注册一个信号处理程序，该操作并立即返回。当内核中有数据准备好，会发送一个信号给应用进程，应用进程便可以在信号处理程序中发起 IO 系统调用，来完成数据读取了。

### 5. 异步

应用进程发起 IO 系统调用后，会立即返回。当内核中数据准备好，并且复制到了用户空间，会产生一个信号来通知应用进程，也就是执行程序指定的信号处理函数。



## 附录 线程池实现

pool.h

```cpp
#ifndef __POOL_H__
#define __POOL_H__

#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <queue>
#include <vector>
// using namespace std;

#define NUMBER 5

// 任务结构体
struct Task {
    void (*function)(void* arg);
    void* arg;

    Task() { }

    Task(void (*_function)(void*), void* _arg) : function(_function), arg(_arg) { } 
};

// 线程池结构体
struct ThreadPool
{
    // 任务队列
    std::queue<Task*> taskQueue;

    int queueCapacity;  // 容量

    pthread_t managerID;    // 管理者线程ID
    // pthread_t *threadIDs;   // 工作的线程ID
    std::vector<pthread_t> threadIDs;   // 工作的线程ID
    int minNum;             // 最小线程数量
    int maxNum;             // 最大线程数量
    int busyNum;            // 忙的线程的个数
    int liveNum;            // 存活的线程的个数
    int exitNum;            // 要销毁的线程个数
    pthread_mutex_t mutexPool;  // 锁整个的线程池
    pthread_mutex_t mutexBusy;  // 锁busyNum变量
    pthread_cond_t notFull;     // 任务队列是不是满了
    pthread_cond_t notEmpty;    // 任务队列是不是空了

    int shutdown;           // 是不是要销毁线程池, 销毁为1, 不销毁为0
};



typedef struct ThreadPool ThreadPool;
// 创建线程池并初始化
ThreadPool *threadPoolCreate(int min, int max, int queueSize);

// 销毁线程池
int threadPoolDestroy(ThreadPool* pool);

// 给线程池添加任务
void threadPoolAdd(ThreadPool* pool, void(*func)(void*), void* arg);

//////////////////////
// 工作的线程(消费者线程)任务函数
void* worker(void* arg);

// 管理者线程任务函数
void* manager(void* arg);


// 单个线程退出
void threadExit(ThreadPool* pool);
#endif  // _THREADPOOL_H
```





pool.cpp

```cpp
#include "pool.h"
#include <string.h>
#include <unistd.h>


ThreadPool* threadPoolCreate(int min, int max, int queueSize)
{
    ThreadPool* pool = new ThreadPool();
    do 
    {
        if (pool == NULL)
        {
            printf("malloc threadpool fail...\n");
            break;
        }

        pool->threadIDs.resize(max);

        pool->minNum = min;
        pool->maxNum = max;
        pool->busyNum = 0;
        pool->liveNum = min;    // 和最小个数相等
        pool->exitNum = 0;

        if (pthread_mutex_init(&pool->mutexPool, NULL) != 0 ||
            pthread_mutex_init(&pool->mutexBusy, NULL) != 0 ||
            pthread_cond_init(&pool->notEmpty, NULL) != 0 ||
            pthread_cond_init(&pool->notFull, NULL) != 0)
        {
            printf("mutex or condition init fail...\n");
            break;
        }

        // 任务队列
        pool->queueCapacity = queueSize;

        pool->shutdown = 0;

        // 创建主线程
        // pthread_create(&pool->managerID, NULL, manager, pool);

        // 创建子线程
        for (int i = 0; i < min; ++i)
        {
            pthread_create(&pool->threadIDs[i], NULL, worker, pool);
        }
        return pool;
    } while (0);

    // 释放资源
    if (pool) delete pool;

    return NULL;
}

int threadPoolDestroy(ThreadPool* pool)
{
    if (pool == NULL)
    {
        return -1;
    }

    // 关闭线程池
    pool->shutdown = 1;
    // 阻塞回收管理者线程
    pthread_join(pool->managerID, NULL);
    // 唤醒阻塞的消费者线程
    for (int i = 0; i < pool->liveNum; ++i)
    {
        pthread_cond_signal(&pool->notEmpty);
    }

    pthread_mutex_destroy(&pool->mutexPool);
    pthread_mutex_destroy(&pool->mutexBusy);
    pthread_cond_destroy(&pool->notEmpty);
    pthread_cond_destroy(&pool->notFull);

    delete pool;
    pool = NULL;

    return 0;
}


void threadPoolAdd(ThreadPool* pool, void(*func)(void*), void* arg)
{
    pthread_mutex_lock(&pool->mutexPool);
    while (pool->taskQueue.size() == pool->queueCapacity && !pool->shutdown)
    {
        // 阻塞生产者线程
        pthread_cond_wait(&pool->notFull, &pool->mutexPool);
    }
    if (pool->shutdown)
    {
        pthread_mutex_unlock(&pool->mutexPool);
        return;
    }

    // 添加任务
    pool->taskQueue.push(new Task(func, arg));

    pthread_cond_signal(&pool->notEmpty);
    pthread_mutex_unlock(&pool->mutexPool);
}


void* worker(void* arg)
{
    ThreadPool* pool = (ThreadPool*)arg;

    while (1)
    {
        pthread_mutex_lock(&pool->mutexPool);
        // 当前任务队列是否为空
        while (pool->taskQueue.empty() && !pool->shutdown)
        {
            // 阻塞工作线程
            pthread_cond_wait(&pool->notEmpty, &pool->mutexPool);

            // 判断是不是要销毁线程
            if (pool->exitNum > 0)
            {
                pool->exitNum--;
                if (pool->liveNum > pool->minNum)
                {
                    pool->liveNum--;
                    pthread_mutex_unlock(&pool->mutexPool);
                    threadExit(pool);
                }
            }
        }

        // 判断线程池是否被关闭了
        if (pool->shutdown)
        {
            pthread_mutex_unlock(&pool->mutexPool);
            threadExit(pool);
        }

        // 从任务队列中取出一个任务
        Task* task = pool->taskQueue.front();
        pool->taskQueue.pop();

        // 解锁
        pthread_cond_signal(&pool->notFull);
        pthread_mutex_unlock(&pool->mutexPool);

        printf("thread %ld start working...\n", pthread_self());
        pthread_mutex_lock(&pool->mutexBusy);
        pool->busyNum++;
        pthread_mutex_unlock(&pool->mutexBusy);

        task->function(task->arg);
        delete (int*)task->arg;
        task->arg = NULL;

        delete task;

        printf("thread %ld end working...\n", pthread_self());
        pthread_mutex_lock(&pool->mutexBusy);
        pool->busyNum--;
        pthread_mutex_unlock(&pool->mutexBusy);
    }
    return NULL;
}

void* manager(void* arg)
{
    ThreadPool* pool = (ThreadPool*)arg;
    while (!pool->shutdown)
    {
        // 每隔3s检测一次
        sleep(3);

        // 取出线程池中任务的数量和当前线程的数量
        pthread_mutex_lock(&pool->mutexPool);
        int queueSize = pool->taskQueue.size();
        int liveNum = pool->liveNum;
        pthread_mutex_unlock(&pool->mutexPool);

        // 取出忙的线程的数量
        pthread_mutex_lock(&pool->mutexBusy);
        int busyNum = pool->busyNum;
        pthread_mutex_unlock(&pool->mutexBusy);

        // 添加线程
        // 任务的个数>存活的线程个数 && 存活的线程数<最大线程数
        if (queueSize > liveNum && liveNum < pool->maxNum)
        {
            pthread_mutex_lock(&pool->mutexPool);
            int counter = 0;
            for (int i = 0; i < pool->maxNum && counter < NUMBER
                && pool->liveNum < pool->maxNum; ++i)
            {
                if (pool->threadIDs[i] == 0)
                {
                    pthread_create(&pool->threadIDs[i], NULL, worker, pool);
                    counter++;
                    pool->liveNum++;
                }
            }
            pthread_mutex_unlock(&pool->mutexPool);
        }
        // 销毁线程
        // 忙的线程*2 < 存活的线程数 && 存活的线程>最小线程数
        if (busyNum * 2 < liveNum && liveNum > pool->minNum)
        {
            pthread_mutex_lock(&pool->mutexPool);
            pool->exitNum = NUMBER;
            pthread_mutex_unlock(&pool->mutexPool);
            // 让工作的线程自杀
            for (int i = 0; i < NUMBER; ++i)
            {
                pthread_cond_signal(&pool->notEmpty);
            }
        }
    }
    return NULL;
}

void threadExit(ThreadPool* pool)
{
    pthread_t tid = pthread_self();
    for (int i = 0; i < pool->maxNum; ++i)
    {
        if (pool->threadIDs[i] == tid)
        {
            pool->threadIDs[i] = 0;
            printf("threadExit() called, %ld exiting...\n", tid);
            break;
        }
    }
    pthread_exit(NULL);
}

```





poolmain.cpp

```cpp
#include "pool.h"
#include <unistd.h>


void taskFunc(void* arg)
{
    int num = *(int*)arg;
    printf("thread %ld is working, number = %d\n",
        pthread_self(), num);
    sleep(1);
}

int main()
{
    // 创建线程池
    ThreadPool* pool = threadPoolCreate(3, 10, 100);
    for (int i = 0; i < 100; ++i)
    {
        int* num = new int(100 + i);
        threadPoolAdd(pool, taskFunc, num);
    }

    sleep(30);

    threadPoolDestroy(pool);
    return 0;
}
```







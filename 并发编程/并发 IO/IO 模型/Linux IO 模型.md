# IO 类型

按照《Unix 网络编程》的划分，IO 模型可以分为：阻塞 IO、非阻塞 IO、IO 复用、信号驱动 IO 和异步 IO，按照 POSIX 标准来划分只分为两类：同步 IO 和异步 IO。

首先一个 IO 操作(read/write 系统调用)其实分成了两个步骤：1)发起 IO 请求和 2)实际的 IO 读写(内核态与用户态的数据拷贝)阻塞 IO 和非阻塞 IO 的区别在于第一步，发起 IO 请求的进程是否会被阻塞，如果阻塞直到 IO 操作完成才返回那么就是传统的阻塞 IO，如果不阻塞，那么就是非阻塞 IO。同步 IO 和异步 IO 的区别就在于第二步，实际的 IO 读写(内核态与用户态的数据拷贝)是否需要进程参与，如果需要进程参与则是同步 IO，如果不需要进程参与就是异步 IO。如果实际的 IO 读写需要请求进程参与，那么就是同步 IO。因此阻塞 IO、非阻塞 IO、IO 复用、信号驱动 IO 都是同步 IO，在编程上，这种非阻塞 IO 一般都采用 IO 状态事件+回调方法的方式来处理 IO 操作。如果是同步 IO，则状态事件为读写就绪。此时的数据仍在内核态中，但是已经准备就绪，可以进行 IO 读写操作。如果是异步 IO，则状态事件为读写完成。此时的数据已经存在于应用进程的地址空间（用户态）中。

在 IO 类型划分中，最重要的就是阻塞/非阻塞、同步/异步这两组概念。阻塞模式的 IO 会造成应用程序等待，直到 IO 完成。同时操作系统也支持将 IO 操作设置为非阻塞模式，这时应用程序的调用将可能在没有拿到真正数据时就立即返回了，为此应用程序需要多次调用才能确认 IO 操作完全完成。而同步与异步，更多地是在应用层面的概念，如果做阻塞 IO 调用，应用程序等待调用的完成的过程就是一种同步状况。相反，IO 为非阻塞模式时，应用程序则是异步的。不过值得一提的是，在 Linux 的 IO 模型中，非阻塞与异步是不同的 IO 模型，这里的异步强调的是异步通知触发。

从应用的调度看，IO 还可以根据以下维度进行划分：

- 大/小块 IO：这个数值指的是控制器指令中给出的连续读出扇区数目的多少。如果数目较多，如 64，128 等，我们可以认为是大块 IO；反之，如果很小，比如 4，8，我们就会认为是小块 IO，实际上，在大块和小块 IO 之间，没有明确的界限。

- 连续/随机 IO：连续 IO 指的是本次 IO 给出的初始扇区地址和上一次 IO 的结束扇区地址是完全连续或者相隔不多的。反之，如果相差很大，则算作一次随机 IO。连续 IO 比随机 IO 效率高的原因是：在做连续 IO 的时候，磁头几乎不用换道，或者换道的时间很短；而对于随机 IO，如果这个 IO 很多的话，会导致磁头不停地换道，造成效率的极大降低。

- 顺序/并发 IO：从概念上讲，并发 IO 就是指向一块磁盘发出一条 IO 指令后，不必等待它回应，接着向另外一块磁盘发 IO 指令。对于具有条带性的 RAID（LUN），对其进行的 IO 操作是并发的，例如：raid 0+1(1+0),raid5 等。反之则为顺序 IO。

# Linux IO 模型

在整个请求过程中，IO 设备数据输入至内核 buffer 需要时间，而从内核 buffer 复制数据至进程 Buffer 也需要时间。因此根据在这两段时间内等待方式的不同，IO 动作可以分为以下五种模式：

- 阻塞 IO (Blocking IO)
- 非阻塞 IO (Non-Blocking IO)
- IO 复用（IO Multiplexing)
- 信号驱动的 IO (Signal Driven IO)
- 异步 IO (Asynchrnous IO)

![IO 模型](https://i.postimg.cc/wvr0DwLQ/image.png)

前四个模型之间的主要区别是第一阶段，四个模型的第二阶段是一样的，过程受阻在调用 recvfrom 当数据从内核拷贝到用户缓冲区。然而，异步 IO 处理两个阶段，与前四个不同。

## 阻塞 IO (Blocking IO)

当用户进程调用了 recvfrom 这个系统调用，内核就开始了 IO 的第一个阶段：等待数据准备。对于 network io 来说，很多时候数据在一开始还没有到达（比如，还没有收到一个完整的 UDP 包），这个时候内核就要等待足够的数据到来。而在用户进程这边，整个进程会被阻塞。当内核一直等到数据准备好了，它就会将数据从内核中拷贝到用户内存，然后内核返回结果，用户进程才解除 block 的状态，重新运行起来。所以，blocking IO 的特点就是在 IO 执行的两个阶段都被 block 了。（整个过程一直是阻塞的）

![Blocking IO](https://i.postimg.cc/SxZTyQRP/image.png)

![Blocking IO 时序图](https://i.postimg.cc/0yJCQP6y/image.png)

## 非阻塞 IO (Non-Blocking IO)

linux 下，可以通过设置 socket 使其变为 non-blocking。当对一个 non-blocking socket 执行读操作时，流程是这个样子：

![non-blocking](https://i.postimg.cc/bJ192qYp/image.png)

当用户进程调用 recvfrom 时，系统不会阻塞用户进程，而是立刻返回一个 ewouldblock 错误，从用户进程角度讲，并不需要等待，而是马上就得到了一个结果（这个结果就是 ewouldblock）。用户进程判断标志是 ewouldblock 时，就知道数据还没准备好，于是它就可以去做其他的事了，于是它可以再次发送 recvfrom，一旦内核中的数据准备好了。并且又再次收到了用户进程的 system call，那么它马上就将数据拷贝到了用户内存，然后返回。当一个应用程序在一个循环里对一个非阻塞调用 recvfrom，我们称为轮询。应用程序不断轮询内核，看看是否已经准备好了某些操作。这通常是浪费 CPU 时间。

## IO 复用（IO Multiplexing)

我们都知道，select/epoll 的好处就在于单个 process 就可以同时处理多个网络连接的 IO。它的基本原理就是 select/epoll 这个 function 会不断的轮询所负责的所有 socket，当某个 socket 有数据到达了，就通知用户进程。它的流程如图：

![IO Multiplexing](https://i.postimg.cc/52vyJys4/image.png)

Linux 提供 select/epoll，进程通过将一个或者多个 fd 传递给 select 或者 poll 系统调用，阻塞在 select 操作上，这样 select/poll 可以帮我们侦测多个 fd 是否处于就绪状态。select/poll 是顺序扫描 fd 是否就绪，而且支持的 fd 数量有限，因此它的使用受到一定的限制。Linux 还提供了一个 epoll 系统调用，epoll 使用基于事件驱动的方式代替顺序扫描，因此性能更高一些。

![epoll，进程通过将一个或者多个](https://i.postimg.cc/TwTjCbtK/image.png)

IO 复用模型具体流程：用户进程调用了 select，那么整个进程会被 block，而同时，内核会“监视”所有 select 负责的 socket，当任何一个 socket 中的数据准备好了，select 就会返回。这个时候用户进程再调用 read 操作，将数据从内核拷贝到用户进程。这个图和 blocking IO 的图其实并没有太大的不同，事实上，还更差一些。因为这里需要使用两个
system call (select 和 recvfrom)，而 blocking IO 只调用了一个 system call (recvfrom)。但是，用 select 的优势在于它可以同时处理多个 connection。

## 信号驱动的 IO (Signal Driven IO)

首先用户进程建立 SIGIO 信号处理程序，并通过系统调用 sigaction 执行一个信号处理函数，这时用户进程便可以做其他的事了，一旦数据准备好，系统便为该进程生成一个 SIGIO 信号，去通知它数据已经准备好了，于是用户进程便调用 recvfrom 把数据从内核拷贝出来，并返回结果。

![Signal Driven IO](https://i.postimg.cc/pLNVsxRg/image.png)

## 异步 IO

一般来说，这些函数通过告诉内核启动操作并在整个操作（包括内核的数据到缓冲区的副本）完成时通知我们。这个模型和前面的信号驱动 IO 模型的主要区别是，在信号驱动的 IO 中，内核告诉我们何时可以启动 IO 操作，但是异步 IO 时，内核告诉我们何时 IO 操作完成。

![](https://i.postimg.cc/GtywD889/image.png)

当用户进程向内核发起某个操作后，会立刻得到返回，并把所有的任务都交给内核去完成（包括将数据从内核拷贝到用户自己的缓冲区），内核完成之后，只需返回一个信号告诉用户进程已经完成就可以了。

# 非阻塞与异步

在传统的网络服务器的构建中，IO 模式会按照 Blocking/Non-Blocking、Synchronous/Asynchronous 这两个标准进行分类，其中 Blocking 与 Synchronous 大同小异，而 NIO 与 Async 的区别在于 NIO 强调的是轮询（Polling），而 Async 强调的是通知（Notification）。

譬如在一个典型的单进程单线程 Socket 接口中，阻塞型的接口必须在上一个 Socket 连接关闭之后才能接入下一个 Socket 连接。而对于 NIO 的 Socket 而言，服务端应用会从内核获取到一个特殊的 "Would Block" 错误信息，但是并不会阻塞到等待发起请求的 Socket 客户端停止。

![](https://i.postimg.cc/wx4t0D8f/image.png)

一般来说，在 Linux 系统中可以通过调用独立的 `select` 或者 `epoll` 方法来遍历所有读取好的数据，并且进行写操作。而对于异步 Socket 而言(譬如 Windows 中的 Sockets 或者 .Net 中实现的 Sockets 模型)，服务端应用会告诉 IO Framework 去读取某个 Socket 数据，在数据读取完毕之后 IO Framework 会自动地调用你的回调(也就是通知应用程序本身数据已经准备好了)。以 IO 多路复用中的 Reactor 与 Proactor 模型为例，非阻塞的模型是需要应用程序本身处理 IO 的，而异步模型则是由 Kernel 或者 Framework 将数据准备好读入缓冲区中，应用程序直接从缓冲区读取数据。

- 同步阻塞：在此种方式下，用户进程在发起一个 IO 操作以后，必须等待 IO 操作的完成，只有当真正完成了 IO 操作以后，用户进程才能运行。

- 同步非阻塞：在此种方式下，用户进程发起一个 IO 操作以后边可返回做其它事情，但是用户进程需要时不时的询问 IO 操作是否就绪，这就要求用户进程不停的去询问，从而引入不必要的 CPU 资源浪费。

- 异步非阻塞：在此种模式下，用户进程只需要发起一个 IO 操作然后立即返回，等 IO 操作真正的完成以后，应用程序会得到 IO 操作完成的通知，此时用户进程只需要对数据进行处理就好了，不需要进行实际的 IO 读写操作，因为真正的 IO 读取或者写入操作已经由内核完成了。

而在并发 IO 的问题中，较常见的就是所谓的 C10K 问题，即有 10000 个客户端需要连上一个服务器并保持 TCP 连接，客户端会不定时的发送请求给服务器，服务器收到请求后需及时处理并返回结果。

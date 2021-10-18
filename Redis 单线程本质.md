## Redis 单线程本质**

Redis 的单线程，指的是 Redis 的网络 IO 和键值对读写由一个线程完成，这是 Redis 对外提供键值存储服务的主要流程，但是 Redis 的其他功能，比如持久化、异步删除、集群数据同步都是由额外的线程执行。

### Redis 为什么用单线程

我们先从多线程开销讲起。

#### 多线程开销

一个多线程系统，在合理分配资源的前提下可以增加系统中处理请求操作的资源实体，进而提升系统能够同时处理的请求数，即吞吐率。

但是如果多线程系统中没有良好的系统设计，随着线程数增加而增长的系统吞吐率到一定瓶颈后增长迟缓甚至下降。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ts4QibG8CPiba8H1rj1ibmkd9SCSZ44fWIoHSUwZyuKicGj9jzsjntypUrnNhpExzDYiavuwz1NkjtBYHBZtYR3CiaEw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)线程数与系统吞吐率

出现这个现象的原因是，系统中存在被多线程同时访问的共享资源，在多个线程要修改这个共享资源时候，要增加额外的机制来保证共享资源正确性，额外的机制带来额外的开销。

假设 Redis 采用多线程设计，如下图所示，现在有两个线程 A 和 B ，线程 A 对一个 List 做 LPUSH 操作， 并对队列长度加 1。同时，线程 B 对该 List 执行 LPOP 操作，并对队列长度减 1。为了保证队列长度的正确性，Redis 需要让线程 A 和 B 的 LPUSH 和 LPOP 串行执行， 只有这样，Redis 可以保证正确记录它们对 List 长度的修改。否则，我们可能就会得到错误的长度结果。这就是多线程编程模式面临的共享资源的并发访问控制问题。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Ts4QibG8CPiba8H1rj1ibmkd9SCSZ44fWIowVNp9wIjhWYy1WNPzmiboiauzWhO7YU9g9TwnXabFbmJXELdRjHiaqXoA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)多线程并发访问 Redis

并发访问控制是多线程开发的难点问题，如果只是对共享资源简单采用粗粒度互斥锁，就可能会出现即使增加线程，大部分线程只是在等待获取访问共享资源的互斥锁，并行变串行，系统吞吐率并没有随着线程增加而增加。

而且，采用多线程开发一般会引入同步原语来保护共享资源的并发访问，这也会降低系统 代码的易调试性和可维护性。为了避免这些问题，Redis 直接采用了单线程模式。

### 单线程 Redis 为什么快

通常来说单线程处理能力比多线程差很大，但是 Redis 用单线程模型达到每秒数十万操作指令处理能力，这是 Redis 多方面设计作用下的综合结果。

一方面，Redis 操作键值对都在内存中完成，并且它还有高效的数据结构，比如哈希表和跳表。

另一方面，网络请求解析，Redis 采用多路复用机制，在网络 IO 中能并发处理大量客户端请求，实现高吞吐量。

### 基本 IO 模型和阻塞点

Redis 的基本执行流程就是依次执行如下操作：

以 Get 请求为例，Redis 为了处理一个 Get 请求，需要监听客户端请求 （ bind/listen ） ，和客户端建立连接（accept） ，从 socket 中读取请求（recv），解析客户端发送请求（parse） ，根据请求类型读取键值数据（get）  最后给客户端返回结果，即向 socket 中写回数据（send）。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

bind/listen、accept、recv、parse 和 send 属于网络 IO 处理，而 get 属于键值数据操作。

在网络 IO 操作的 accept 和 recv 处理上，可能会出现阻塞。

当 Redis 监听到一个客户端的连接请求，但是未能成功建立连接就会阻塞在 accept 函数里，导致其他客户端无法和 Redis 建立连接。同样的，Redis 通过 recv 从客户端读取数据时，如果数据一直没有到达，也会一直阻塞在 recv 函数。

这就阻塞 Redis 整个工作线程，无法处理其他客户端请求，为了解决这个问题，要运用 socket 网络模型的非阻塞模式。






___
# 概述
什么是Reactor模式？
Reactor模式是事件驱动架构的一种具体实现方法，简而言之，就是一个单线程进行循环监听就绪IO事件，并将就绪IO事件分发给对应的回调函数。

___
# 组成
Reactor模式由什么组成的？
Reactor模式分为两个重要组成部分,Reactor和Handler
Reactor(反应器)：循环监听就绪IO事件，并分发给回调函数。
Handler(回调函数)：执行对应IO事件的实际业务逻辑。

___
# 模型演进

## 单线程模型
IO事件轮询，分发(accept)和IO事件执行(read,decode,compute,encode,send)都在一个线程中完成，如下图所示：
![[Pasted image 20230524115052.png]]

在单线程模型下，不仅IO操作在Reactor线程上，而非IO操作(handlder中process()方法)也在Reactor线程上执行了，当非IO操作执行慢的话，这会大大延迟IO请求响应。

### 优点
模型简单，没有多线程

### 缺点：
-   单线程模型是串行的，可以理解成等于每个请求都是串行执行，当其中某个handler阻塞时， 其他client的handler得不到执行，无法接收新的client请求。
-   单线程模型不能充分利用多核资源。

所以应该**把非IO操作拆出来，来加速Reactor线程对IO请求响应，就出现多线程模型**。

### 场景
redis 等操作小且快的应用

## 多线程模型
与单线程模型不同的是添加一个**业务线程池**，将非IO操作(**业务逻辑处理**)交给业务线程池来处理，提高Reactor线程的IO响应，如图所示：
![[Pasted image 20230524115042.png]]

在多线程模型下，虽然将非IO操作拆出去了，但是**所有IO操作都在Reactor单线程中完成的**。在高负载、高并发场景下，也会成为瓶颈，于是对Reactor单线程进行了优化，出现了主从线程模型。

主从线程模型： 相比多线程模型而言，对于多核cpu，为了充分利用资源，将Reactor拆分成了mainReactor 和 subReactor，但是，主从线程模型也有弊端，不适合大量数据传输。 mainReactor：负责监听接收(accpet)新连接，将新连接后续操作交给subReactor来处理，通常由一个线程处理。 subReactor： 负责处理IO的读写操作，通常由多个线程处理。 非IO操作依然由业务线程池来处理。

## 主从线程模型
单Reactor多线程模型解决了 Handler 单线程的性能问题，但是 Reactor 还是单线程的，对于高并发场景还是会有性能瓶颈，所以需要将 Reactor 调整为多线程模式，也就是接下来要介绍的主从 Reactor 多线程模型。主从 Reactor 多线程模型将 Reactor 分成两部分：
（1）MainReactor：只负责处理连接建立事件，通过 select 监听 server socket，将建立的 socketChannel 指定注册给 subReactor，通常一个线程就可以了
（2）SubReactor：负责读写事件，维护自己的 selector，基于 MainReactor 注册的 SocketChannel 进行多路分离 IO 读写事件，读写网络数据，并将业务处理交由 worker 线程池来完成。SubReactor 的个数一般和 CPU 个数相同
![[Pasted image 20230524115025.png]]




# HashMap 的底层数据结构是怎样的
[[HashMap#数据结构]]

# HashMap 的扩容机制
[[HashMap#扩容机制]]

# Java 怎么获取多线程的返回值
[[FutureTask]]

创建线程的三个方法是什么？


# 说一下泛型原理，并举例说明
[[泛型]]

# Java 中实现多态的机制是什么？
[[多态]]

如何将一个 Java 对象序列化到文件里

# 说说你对 Java 反射的理解
[[反射]]

谈谈你对 Java 中 String 的了解

String 为什么要设计成不可变的

用过 ConcurrentHashMap，讲一下他和 HashTable 的不同之处

HashMap 结构，线程不安全举个例子？

Java 进程间的几种通信方式

乐观锁和悲观锁的理解及如何实现，有哪些实现方式

ArrayList 和 LinkedList 的区别在哪里

数组和链表的区别？当数组内存过大时会出现什么问题？链表增删过多会出现的什么问题？

final、finally、finallize？finally 是在 return 之前执行还是之后？finally 块里的代码一定会执行吗？



在生产环境 Linux 服务器上，发现某台运行 Java 服务的服务器的CPU100%，不借助任何可视化工具，怎么进行问题的定位？
⚫ top 找出进程 CPU 比较高 PID

⚫ top -Hp PID 打印 该 PID 进程下哪条线程的 CPU 占用比较高 tid

⚫ printf “%x\n” tid 将该 id 进行 16 进制转换 tidhex

⚫ jstack PID |grep tidhex 打印线程的堆栈信息


# `Synchronized`和`Reentrantlock`的区别
[[ReentrantLock#synchronized 和 ReentrantLock 的比较]]

# 线程的状态有哪些
[[Thread#线程状态]]

# 线程池的状态有哪些
[[ThreadPoolExecutor#线程池状态]]


# 线程池的`submit`和`execute`有什么区别
[[ThreadPoolExecutor#execute 和 submit 比较]]

# Bean 的声明周期

java是值传递还是引用传递，为什么
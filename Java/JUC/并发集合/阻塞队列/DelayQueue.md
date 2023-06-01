___
# 概述
具有延迟获取功能无界阻塞队列
内部通过维护 [[PriorityQueue]] 实现优先级
只有当元素到达指定时间后，才能被取出消费

## 数据结构
阻塞
[[PriorityQueue#数据结构]]
是最小堆
+
是不公平的
 
## 原理
插入的元素，必须实现`Delayed`接口，而`Delayed`接口又继承自`Comparable`接口。
通过实现`Comparable`接口的`compareTo`方法来进行堆排序。
通过实现`Delayed`接口的`getDelay`方法来获取剩余时间，为0或负数则可以取出了。
所以，我们实现的`compareTo`方法必须正确实现，保证剩余时间最短的元素会在堆的前面位置，否则将不满足延迟队列要求。

一把`ReentrantLock`独占锁`lock`来保证线程操作队列时同步安全。
一个信号量`Condition`是`available`来做阻塞操作。当使用阻塞的获取方法`take`获取元素时，队列没有元素或者当前线程不是`leader`则会被阻塞。

### Leader-Follower 线程模型
在Leader-follower线程模型中每个线程有三种模式：

-   leader：只有一个线程成为leader，如DelayQueue如果有一个线程在等待元素到期，则其他线程就会阻塞等待
-   follower：会一直尝试争抢leader，抢到leader之后才开始干活
-   processing：处理中的线程

DelayQueue队列中有一个leader属性：private Thread leader = null;用到的就是Leader-Follower线程模型。  
当有一个线程持有锁，设置了leader属性，正在等待元素到期时，则成为了leader，其他线程就直接阻塞。

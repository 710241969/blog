___
# 概述
具有优先级的无界阻塞队列

## 数据结构
阻塞
基于数组实现的最小二叉堆算法，堆就是一棵完全二叉树
最大容量是`Integer.MAX_VALUE - 8`
默认长度11
不允许 null 元素

## 原理
元素必须实现`Compare`接口或在`PriorityBlockingQueue`构造函数中传入`Comparator`接口实现
一把`ReentrantLock`独占锁`lock`来保证线程同步安全。
一个信号量`Condition`是`notEmpty`来做当队列为空时的阻塞操作。当使用阻塞的获取方法`take`获取元素时，队列没有元素则会被`notEmpty`条件原语阻塞。

___







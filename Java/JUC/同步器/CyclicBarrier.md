___
# 概述
回环栅栏、循环屏障。可以被重用。两个构造方法，public CyclicBarrier(int parties, Runnable barrierAction) {}、public CyclicBarrier(int parties) {}。传入的 parties 是指一共需要等待多少个线程达到 barrier 状态（就是调用 CyclicBarrier 的 await() 方法进行等待），当指定数量的线程达到 barrier 状态，那么指定的 parties 数量的线程就会同时被唤醒。至于另外一个参数是 barrierAction 是所有线程达到 barrier 状态时会执行的额外内容（会从由几个线程中最先被唤起的一个去执行这个 Runnable）
可重用是指，当 parties 个线程完成后，如果后面又有 parties 个线程去执行相同逻辑，结果是一样的。比如说每5个1循环，每 5 个 1 循环，而 CountDownLatch 是只能用一次，计数器到0了就用完了

___
# 原理
一把`ReentrantLock`独占锁`lock`来保证线程安全。
一个信号量`Condition`是`trip`来做阻塞操作。
通过线程加锁方式来操作`count`字段，一个进程获取锁进来进行`await`操作，就进行减1操作，然后通过`Condition`进行`await`，当`count`字段减到0就唤醒全部`await`的线程，通过`Condition`的`signalAll`唤醒。

___
# 源码分析

## 成员变量
```java
/**  
* 通过独占 lock 保证线程安全，只有获取锁的线程能够操作 count 并进行等待  
*/  
private final ReentrantLock lock = new ReentrantLock(); 

/**  
* 通过信号量实现线程的阻塞和唤醒  
*/  
private final Condition trip = lock.newCondition(); 

/**  
* 记录栅栏的数量，再次创建 Generation 时会用到  
*/  
private final int parties;  

/**  
* 每 parties 个线程是一代，新进入的线程就在新的 Generation 中  
*/  
private Generation generation = new Generation();  

/**  
* 作为回环计数器，每进入一个线程 count - 1  
* 当 count = 0 唤醒当前阻塞的线程，然后赋值回 parties
*/  
private int count;
```

## 回环
为什么可循环使用？计数器的值为 0 时，唤醒所有线程的同时，会重新设置计数器的值为 parties
```JAVA
/**  
* 为下一代回环重置回环状态  
*/  
private void nextGeneration() {  
	// 唤醒所有回环等待的线程  
	trip.signalAll();  
	// 重置 countcount = parties;  
	generation = new Generation();  
}
```




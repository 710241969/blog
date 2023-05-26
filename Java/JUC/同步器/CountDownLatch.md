CountDownLatch
只有一个构造器 public CountDownLatch(int count) {  };，传入一个 int 。实现了一个线程计数器的功能，比如一个线程A，他要等待其他5个线程执行完之后才开始执行，就可以使用 CountDownLatch ，设置值为 5 。线程通过调用 CountDownLatch 的 await() 方法进行等待，其余线程执行完，调用 CountDownLatch 的 countDown() 方法，使内部计数器数值减一，计数器为 0 时，被 await() 的线程将被唤醒。

___
# 原理
一个成员变量 `private final Sync sync;`，自己实现`AbstractQueuedSynchronizer`的`tryAcquireShared`和`tryReleaseShared`。
`countDown()`函数是由`tryReleaseShared()`通过 CAS 实现进行对`state`的减少，当减少为0时，通过`LockSupport.unpark(s.thread);`实现唤醒。
`await()`函数是由`tryAcquireShared()`通过循环判断`state`是否为0，通过`LockSupport.park(this);`进行阻塞。



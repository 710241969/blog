

不需要获取锁


## park()
直接休眠当前线程

##  park(Object blocker)
```java
public static void park(Object blocker) {  
	Thread t = Thread.currentThread();  
	setBlocker(t, blocker);  
	UNSAFE.park(false, 0L);  
	setBlocker(t, null);  
}
```

## unpark(Thread thread)
唤醒指定线程

# 底层实现原理
在Linux系统下，是用的Posix线程库pthread中的mutex（互斥量），condition（条件变量）来实现的。
mutex和condition保护了一个_counter的变量，简单点说：当park时，这个变量被设置为0，当unpark时，这个变量被设置为1。

LockSupport的park和unpark方法相比于Synchronize的wait和notify，notifyAll方法：

1.更简单，**不需要获取锁**，能直接阻塞线程。

2.更直观，以thread为操作对象更符合阻塞线程的直观定义；

3.更精确，可以准确地唤醒某一个线程（notify随机唤醒一个线程，notifyAll唤醒所有等待的线程）；

4.更灵活 ，unpark方法可以在park方法前调用。那么等于没有休眠过程，没有阻塞

除此之外，LockSupport.unpark(Thread t)相当于是为线程t提供一个运行许可，并且这个许可是不可重叠的。举例子就是如果线程t2连续三次调用了LockSupport.unpark(Thread t)，然后线程t调用LockSupport.park()一次就会将许可证用光，若线程t再次调用LockSupport.park()照样会阻塞。






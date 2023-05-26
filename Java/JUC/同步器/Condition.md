___
# 概述
Condition  作为信号量的接口，定义了阻塞和唤醒方法

其中最常用到的实现是
`java.util.concurrent.locks.AbstractQueuedSynchronizer.ConditionObject`

___
# 原理

## 数据结构
通过双向链表，维护线程队列

维护了头结点和尾结点

___
# 源码分析

## 成员变量
```java
/**  
* 信号量等待队列的头结点  
*/  
private transient Node firstWaiter;  

/**  
* 信号量等待队列的尾结点  
*/  
private transient Node lastWaiter;
```

## 阻塞
```java
    public final void await() throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        // 当前线程加入到信号量队列尾部
        Node node = addConditionWaiter();
        /*
        当前线程释放资源，如果当前线程没有获取锁，会抛出 IllegalMonitorStateException 异常
            */
        int savedState = fullyRelease(node);
        int interruptMode = 0;
        while (!isOnSyncQueue(node)) {
            // 进入阻塞
            LockSupport.park(this);
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
        }
        // 唤醒后会进行一次锁的获取
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        if (node.nextWaiter != null) // clean up if cancelled
            unlinkCancelledWaiters();
        if (interruptMode != 0)
            reportInterruptAfterWait(interruptMode);
    }
    
    /**
     * 将当前线程获得当前锁的 state 字段清零，比如进入了可重入锁的次数
     * @param node
     * @return
     */
    final long fullyRelease(Node node) {
        boolean failed = true;
        try {
            // 获取当前线程获取的 state
            long savedState = getState();
            // 释放全部 state，会判断当前线程是否锁的持有者
            if (release(savedState)) {
                failed = false;
                return savedState;
            } else {
                // 如果当前线程根本不是当前锁的持有者会判断
                throw new IllegalMonitorStateException();
            }
        } finally {
            // 如果失败，会将当前节点的状态改为 CANCELLED
            if (failed)
                node.waitStatus = Node.CANCELLED;
        }
    }
```

## 唤醒

### signal
唤醒第一个状态是`Node.CONDITION`的线程

### signalAll
遍历队列，唤醒全部状态是`Node.CONDITION`的线程

Condition 它更强大的地方在于：能够更加精细的控制多线程的休眠与唤醒。对于同一个锁，我们可以创建多个Condition，在不同的情况下使用不同的Condition。
例如，假如多线程读/写同一个缓冲区：当向缓冲区中写入数据之后，唤醒"读线程"；当从缓冲区读出数据之后，唤醒"写线程"；并且当缓冲区满的时候，"写线程"需要等待；当缓冲区为空时，"读线程"需要等待。         如果采用Object类中的wait(), notify(), notifyAll()实现该缓冲区，当向缓冲区写入数据之后需要唤醒"读线程"时，不可能通过notify()或notifyAll()明确的指定唤醒"读线程"，而只能通过notifyAll唤醒所有线程(但是notifyAll无法区分唤醒的线程是读线程，还是写线程)。  但是，通过Condition，就能明确的指定唤醒读线程。

通过 LockSupport.park(this); 阻塞
通过 LockSupport.unpark(node.thread); 唤醒
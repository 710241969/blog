# 概述
`AbstractQueuedSychronizer`简称 AQS ，翻译过来应该是抽象队列同步器。

如果说`java.util.concurrent`的基础是 CAS 乐观锁机制的话，那么 AQS 就是整个 Java 并发包的核心了，`ReentrantLock`、`CountDownLatch`、`Semaphore`等等都用到了它。

AQS 实际上以双向队列的形式连接所有的`Entry`，维护了一个 CLH 队列 (Craig, Landin, and Hagersten) ，是一个非阻塞的 FIFO 队列，队列元素是当前等待锁的线程，通过自旋锁和 CAS 保证节点插入和移除的原子性。比方说 ReentrantLock ，所有等待的线程都被放在一个 Entry 中并连成双向队列

AQS 定义了对双向队列所有的操作，而只开放了`tryLock`和`tryRelease`方法给开发者使用，开发者可以根据自己的实现重写`tryLock`和`tryRelease`方法，以实现自己的并发功能。

___
# 源码分析
```Java
  /**
     * Wait queue node class.
     *
     * <p>The wait queue is a variant of a "CLH" (Craig, Landin, and
     * Hagersten) lock queue. CLH locks are normally used for
     * spinlocks.  We instead use them for blocking synchronizers, but
     * use the same basic tactic of holding some of the control
     * information about a thread in the predecessor of its node.  A
     * "status" field in each node keeps track of whether a thread
     * should block.  A node is signalled when its predecessor
     * releases.  Each node of the queue otherwise serves as a
     * specific-notification-style monitor holding a single waiting
     * thread. The status field does NOT control whether threads are
     * granted locks etc though.  A thread may try to acquire if it is
     * first in the queue. But being first does not guarantee success;
     * it only gives the right to contend.  So the currently released
     * contender thread may need to rewait.
     *
     * <p>To enqueue into a CLH lock, you atomically splice it in as new
     * tail. To dequeue, you just set the head field.
     * <pre>
     *      +------+  prev +-----+       +-----+
     * head |      | <---- |     | <---- |     |  tail
     *      +------+       +-----+       +-----+
     * </pre>
     *
     * <p>Insertion into a CLH queue requires only a single atomic
     * operation on "tail", so there is a simple atomic point of
     * demarcation from unqueued to queued. Similarly, dequeuing
     * involves only updating the "head". However, it takes a bit
     * more work for nodes to determine who their successors are,
     * in part to deal with possible cancellation due to timeouts
     * and interrupts.
     *
     * <p>The "prev" links (not used in original CLH locks), are mainly
     * needed to handle cancellation. If a node is cancelled, its
     * successor is (normally) relinked to a non-cancelled
     * predecessor. For explanation of similar mechanics in the case
     * of spin locks, see the papers by Scott and Scherer at
     * http://www.cs.rochester.edu/u/scott/synchronization/
     *
     * <p>We also use "next" links to implement blocking mechanics.
     * The thread id for each node is kept in its own node, so a
     * predecessor signals the next node to wake up by traversing
     * next link to determine which thread it is.  Determination of
     * successor must avoid races with newly queued nodes to set
     * the "next" fields of their predecessors.  This is solved
     * when necessary by checking backwards from the atomically
     * updated "tail" when a node's successor appears to be null.
     * (Or, said differently, the next-links are an optimization
     * so that we don't usually need a backward scan.)
     *
     * <p>Cancellation introduces some conservatism to the basic
     * algorithms.  Since we must poll for cancellation of other
     * nodes, we can miss noticing whether a cancelled node is
     * ahead or behind us. This is dealt with by always unparking
     * successors upon cancellation, allowing them to stabilize on
     * a new predecessor, unless we can identify an uncancelled
     * predecessor who will carry this responsibility.
     *
     * <p>CLH queues need a dummy header node to get started. But
     * we don't create them on construction, because it would be wasted
     * effort if there is never contention. Instead, the node
     * is constructed and head and tail pointers are set upon first
     * contention.
     *
     * <p>Threads waiting on Conditions use the same nodes, but
     * use an additional link. Conditions only need to link nodes
     * in simple (non-concurrent) linked queues because they are
     * only accessed when exclusively held.  Upon await, a node is
     * inserted into a condition queue.  Upon signal, the node is
     * transferred to the main queue.  A special value of status
     * field is used to mark which queue a node is on.
     *
     * <p>Thanks go to Dave Dice, Mark Moir, Victor Luchangco, Bill
     * Scherer and Michael Scott, along with members of JSR-166
     * expert group, for helpful ideas, discussions, and critiques
     * on the design of this class.
     */
    static final class Node {

    }
```

```java
/**  
* Head of the wait queue, lazily initialized. Except for  
* initialization, it is modified only via method setHead. Note:  
* If head exists, its waitStatus is guaranteed not to be  
* CANCELLED.  
*/  
private transient volatile Node head;  
  
/**  
* Tail of the wait queue, lazily initialized. Modified only via  
* method enq to add new wait node.  
*/  
private transient volatile Node tail;  
  
/**  
* The synchronization state.  
*/  
private volatile int state;
```

## 模版方法模式
```java
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }
    
    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }
    
    protected int tryAcquireShared(int arg) {
        throw new UnsupportedOperationException();
    }

    protected boolean tryReleaseShared(int arg) {
        throw new UnsupportedOperationException();
    }

    protected boolean isHeldExclusively() {
        throw new UnsupportedOperationException();
    }
```
通过模版方法模式，由子类实现对`state`的修改

## 总结
公平锁和非公平锁就是基于这个队列而言的：
公平锁，是按照通过 CLH 等待线程按照队列 FIFO 的规则，公平的获取锁；而非公平锁，则当线程要获取锁时，它会无视CLH等待队列而直接获取锁。

通过 CAS 修改 state 的值来实现线程安全

通过 LockSupport.park(this); 来阻塞

通过 LockSupport.unpark(s.thread); 来唤醒 CLH 的第一个等待线程

___
# 实现的子类
Sync in ReentrantLock
FairSync in ReentrantLock
NonfairSync in ReentrantLock

Sync in ReentrantReadWriteLock
FairSync in ReentrantReadWriteLock
NonfairSync in ReentrantReadWriteLock

FairSync in Semaphore
NonfairSync in Semaphore

Sync in CountDownLatch

Sync in LimitLatch

Sync in Semaphore

Worker in ThreadPoolExecutor





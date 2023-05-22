___
# 概述
LinkedBlockingQueue 单向链表实现的阻塞队列，可指定大小，该队列按 FIFO（先进先出）排序元素。

## 数据结构
单向链表实现
有界队列
可以指定容量大小，该队列按 FIFO（先进先出）排序元素
默认容量是`Integer.MAX_VALUE`，但我们依然说他是有界队列，因为插入可能会阻塞，**是否有界，取决于插入操作是否会阻塞，而不是说容量的实际大小**
阻塞队列

## 原理
两把`ReentrantLock`独占锁来保证线程安全，`putLock`对插入操作进行同步，`takeLock`对取出操作进行同步。
两个信号量`Condition`分别是`notEmpty`和`notFull`来做插入和取出操作的阻塞操作。

___
# 源码分析

## 成员变量
```java
    /**
     * Head of linked list.
     * Invariant: head.item == null
     */
    transient Node<E> head;

    /**
     * Tail of linked list.
     * Invariant: last.next == null
     */
    private transient Node<E> last;

    /** Lock held by take, poll, etc */
    private final ReentrantLock takeLock = new ReentrantLock();

    /** Wait queue for waiting takes */
    private final Condition notEmpty = takeLock.newCondition();

    /** Lock held by put, offer, etc */
    private final ReentrantLock putLock = new ReentrantLock();

    /** Wait queue for waiting puts */
    private final Condition notFull = putLock.newCondition();
```

## 构造函数
可以指定容量，也可以不指定容量，默认是`Integer.MAX_VALUE`，可以认为默认就是无界的
```java
    /**
     * Creates a {@code LinkedBlockingQueue} with a capacity of
     * {@link Integer#MAX_VALUE}.
     */
    public LinkedBlockingQueue() {
        this(Integer.MAX_VALUE);
    }

    /**
     * Creates a {@code LinkedBlockingQueue} with the given (fixed) capacity.
     *
     * @param capacity the capacity of this queue
     * @throws IllegalArgumentException if {@code capacity} is not greater
     *         than zero
     */
    public LinkedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
        last = head = new Node<E>(null);
    }

    /**
     * Creates a {@code LinkedBlockingQueue} with a capacity of
     * {@link Integer#MAX_VALUE}, initially containing the elements of the
     * given collection,
     * added in traversal order of the collection's iterator.
     *
     * @param c the collection of elements to initially contain
     * @throws NullPointerException if the specified collection or any
     *         of its elements are null
     */
    public LinkedBlockingQueue(Collection<? extends E> c) {
        this(Integer.MAX_VALUE);
        final ReentrantLock putLock = this.putLock;
        putLock.lock(); // Never contended, but necessary for visibility
        try {
            int n = 0;
            for (E e : c) {
                if (e == null)
                    throw new NullPointerException();
                if (n == capacity)
                    throw new IllegalStateException("Queue full");
                enqueue(new Node<E>(e));
                ++n;
            }
            count.set(n);
        } finally {
            putLock.unlock();
        }
    }
```






___
# 概述

## 数据结构
单向链表实现
无界队列，该队列按 FIFO（先进先出）排序元素
非阻塞，所以没有用到条件原语

## 原理
使用 CAS 原子指令来处理对数据的并发访问，这是非阻塞算法得以实现的基础。
head/tail 节点都允许滞后，也就是说它们并非总是指向队列的头/尾节点，这是因为并不是每次操作队列都更新 head/tail，和 LinkedTransferQueue 一样，使用了一个“松弛阀值（2）”， 当前指针距离
head/tail 节点大于2时才会更新 head/tail，这也是一种优化方式，减少了 CAS 指令的执行次数。
由于队列有时会处于不一致状态。为此，ConcurrentLinkedQueue 对节点使用了“不变性”和“可变性”来约束非阻塞算法的正确性（后面会详细说明）。
使用“自链接”方式管理出队节点，这样一个自链接节点意味着需要从head向后推进

从tail节点向后自旋查找 next 为 null 的节点，也就是最后一个节点（因为 tail 节点并不是每次都更新，所以我们取到的 tail 节点有可能并不是最后一个节点），然后CAS插入新增节点

上面我们提到过：并不是每次操作都会更新 head/tail 节点，而是使用了一个“松弛阀值”，这个“松弛阀值”就体现在上面源码中if (p != t)这一行，p初始是等于tail的，如果向后查找了一次以上才找到最后一个节点，再加上新增的节点，说明tail已经跳跃了两个（或以上）节点，此时才会CAS更新tail，这里也算是一种编程技巧

___
# 源码分析

## 成员变量
只维护了头结点和尾结点，没有维护长度
```java
    /**
     * A node from which the first live (non-deleted) node (if any)
     * can be reached in O(1) time.
     * Invariants:
     * - all live nodes are reachable from head via succ()
     * - head != null
     * - (tmp = head).next != tmp || tmp != head
     * Non-invariants:
     * - head.item may or may not be null.
     * - it is permitted for tail to lag behind head, that is, for tail
     *   to not be reachable from head!
     */
    private transient volatile Node<E> head;

    /**
     * A node from which the last node on list (that is, the unique
     * node with node.next == null) can be reached in O(1) time.
     * Invariants:
     * - the last node is always reachable from tail via succ()
     * - tail != null
     * Non-invariants:
     * - tail.item may or may not be null.
     * - it is permitted for tail to lag behind head, that is, for tail
     *   to not be reachable from head!
     * - tail.next may or may not be self-pointing to tail.
     */
    private transient volatile Node<E> tail;
```

## 构造函数
没有指定容量的地方，是无界的
```java
    /**
     * Creates a {@code ConcurrentLinkedQueue} that is initially empty.
     */
    public ConcurrentLinkedQueue() {
        head = tail = new Node<E>(null);
    }

    /**
     * Creates a {@code ConcurrentLinkedQueue}
     * initially containing the elements of the given collection,
     * added in traversal order of the collection's iterator.
     *
     * @param c the collection of elements to initially contain
     * @throws NullPointerException if the specified collection or any
     *         of its elements are null
     */
    public ConcurrentLinkedQueue(Collection<? extends E> c) {
        Node<E> h = null, t = null;
        for (E e : c) {
            checkNotNull(e);
            Node<E> newNode = new Node<E>(e);
            if (h == null)
                h = t = newNode;
            else {
                t.lazySetNext(newNode);
                t = newNode;
            }
        }
        if (h == null)
            h = t = new Node<E>(null);
        head = h;
        tail = t;
    }
```





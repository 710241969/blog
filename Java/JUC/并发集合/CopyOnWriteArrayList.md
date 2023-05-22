___
# 概述
CopyOnWriteArrayList 相当于线程安全的 ArrayList
还有一种线程安全的 ArrayList 叫做 [[Vector]]
支持单个插入删除，也支持多个插入删除，每次只变化操作数量的容量，非常紧凑，不会浪费空间。
只有增删操作加锁，读取不加锁，所以读取到的是可能脏数据，包括`size`，`contains`等读取的方法可能都会得到旧数组的数据

## 数据结构
一维数组
非阻塞，所以没有用到条件原语

## 原理
顾名思义，当数组有增删变化时，从旧的数组复制出来，重新建立一个新的数组。通过一把 ReentrantLock 锁，对增删操作加锁。
使用场景：因为每次写操作都会加锁和复制数组，所以适合读多写少场景。

___
# 源码分析

## 成员变量
```java
    /** The array, accessed only via getArray/setArray. */
    private transient volatile Object[] array;
```

## 构造函数
```java
    /**
     * Creates an empty list.
     */
    public CopyOnWriteArrayList() {
        setArray(new Object[0]);
    }

    /**
     * Creates a list containing the elements of the specified
     * collection, in the order they are returned by the collection's
     * iterator.
     *
     * @param c the collection of initially held elements
     * @throws NullPointerException if the specified collection is null
     */
    public CopyOnWriteArrayList(Collection<? extends E> c) {
        Object[] elements;
        if (c.getClass() == CopyOnWriteArrayList.class)
            elements = ((CopyOnWriteArrayList<?>)c).getArray();
        else {
            elements = c.toArray();
            if (c.getClass() != ArrayList.class)
                elements = Arrays.copyOf(elements, elements.length, Object[].class);
        }
        setArray(elements);
    }

    /**
     * Creates a list holding a copy of the given array.
     *
     * @param toCopyIn the array (a copy of this array is used as the
     *        internal array)
     * @throws NullPointerException if the specified array is null
     */
    public CopyOnWriteArrayList(E[] toCopyIn) {
        setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
    }
```

## 增加单个元素
每次只加一个长度
```java
    /**
     * 在数组末尾加入元素
     * @param e 待插入元素
     * @return 其实一定会返回 true
     */
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        // 加锁
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            // 复制一个长度 +1 的新数组
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            // 将新增元素插入新数组末尾
            newElements[len] = e;
            // 将新数组作为当前使用的数组
            setArray(newElements);
            return true;
        } finally {
            // 解锁
            lock.unlock();
        }
    }

    /**
     * 在指定下标位置插入元素
     * @param index 指定的下标
     * @param element 待插入元素
     */
    public void add(int index, E element) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            if (index > len || index < 0)
                throw new IndexOutOfBoundsException("Index: "+index+
                                                    ", Size: "+len);
            Object[] newElements;
            int numMoved = len - index;
            if (numMoved == 0)
                newElements = Arrays.copyOf(elements, len + 1);
            else {
                newElements = new Object[len + 1];
                System.arraycopy(elements, 0, newElements, 0, index);
                System.arraycopy(elements, index, newElements, index + 1,
                                 numMoved);
            }
            newElements[index] = element;
            setArray(newElements);
        } finally {
            lock.unlock();
        }
    }
```

## 读取元素
无锁，直接读
```java
    /**
     * 读取指定下标的元素
     * @param index 期望获取元素的下标
     * @return 返回指定下标的元素
     */
    public E get(int index) {
        return get(getArray(), index);
    }

        /**
     * Gets the array.  Non-private so as to also be accessible
     * from CopyOnWriteArraySet class.
     */
    final Object[] getArray() {
        return array;
    }

    @SuppressWarnings("unchecked")
    private E get(Object[] a, int index) {
        return (E) a[index];
    }
```

___
# CopyOnWriteArrayList 和 Vector 对比

## 相同点
增减操作都会进行 copy 生成新的数组，变化多少长度就变化多少，非常紧凑，没有空间浪费

## 不同点
Vector 所有操作加锁
CopyOnWriteArrayList 只有修改的操作加锁，读不加锁


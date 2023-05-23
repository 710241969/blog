___
# 概述

___
# 对象定义
```Java
public class ThreadLocal<T> {
    /**
     * ThreadLocalMap is a customized hash map suitable only for
     * maintaining thread local values. No operations are exported
     * outside of the ThreadLocal class. The class is package private to
     * allow declaration of fields in class Thread.  To help deal with
     * very large and long-lived usages, the hash table entries use
     * WeakReferences for keys. However, since reference queues are not
     * used, stale entries are guaranteed to be removed only when
     * the table starts running out of space.
     */
    static class ThreadLocalMap {
        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }

        }
        

        /**
         * The initial capacity -- MUST be a power of two.
         */
        private static final int INITIAL_CAPACITY = 16;

        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         */
        private Entry[] table;
    }
}
```
ThreadLocal 类内部定义了静态类 ThreadLocalMap ， ThreadLocalMap 内部中定义了静态类 Entry ，以 K-V 结构形式存储数据 Key 是 ThreadLocal ， Value 是对应的值，而且 Key 是 WeakReference 弱引用
ThreadLocalMap 定了了叫 table 的 Entry 列表成员变量，作为实际数据的存储
数据实际上是保存在 ThreadLocalMap 的 Entry 中，Key 是 ThreadLocal，Value 是对应的值

# 数据结构
数据结构是一维数组
ThreadLocalMap 是 ThreadLocal 的内部类，没有实现 Map 接口，用独立的方式实现了 Map 的功能，其内部的 Entry 也独立实现，非常简单，初始为 16 长度的数组，扩容直接扩大一倍
和 HashMap 的最大的不同在于，ThreadLocalMap 结构非常简单，没有 next 引用，也就是说 ThreadLocalMap 中解决 Hash 冲突的方式并非链表的方式，而是采用线性探测的方式，所谓线性探测，就是根据初始 key 的 hashcode 值确定元素在 table 数组中的位置，如果发现这个位置上已经有其他 key 值的元素被占用，则利用固定的算法寻找一定步长的下个位置，依次判断，直至找到能够存放的位置。
ThreadLocalMap 解决 Hash 冲突的方式就是简单的步长加 1 或减 1，寻找下一个相邻的位置

# 组织关系
```java
public class Thread implements Runnable {
	/* ThreadLocal values pertaining to this thread. This map is maintained  
	* by the ThreadLocal class. */  
	ThreadLocal.ThreadLocalMap threadLocals = null;  
}
```
Thread 类中定义了默认权限的成员变量  `threadLocals`
正是 threadLocals 用来记录当前线程用到的 ThreadLocal 数据
```java
    /**
     * Returns the value in the current thread's copy of this
     * thread-local variable.  If the variable has no value for the
     * current thread, it is first initialized to the value returned
     * by an invocation of the {@link #initialValue} method.
     *
     * @return the current thread's value of this thread-local
     */
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```
所以其实 ThreadLocal.get 方法，就是获取当前线程对象，再通过线程对象来读取 ThreadLocalMap ，访问其中的 ThreadLocal Key

# 为什么使用弱引用
如上面所说，`ThreadLocalMap`是真正用来存放对象的，`ThreadLocalMap`本身并没有为外界提供取出和存放数据的 API，我们所能获得数据的方式只有通过`ThreadLocal`类提供的 API 来间接的从`ThreadLocalMap`取出数据，所以，当我们用不了 key（ThreadLocal对象）的 API 也就无法从`ThreadLocalMap`里取出指定的数据
假设`ThreadLocal`是函数内 new 的，函数退出，对象被回收了，这些 get 和 set 方法也访问不到了，也就没法从`ThreadLocalMap`里取出数据了。没法利用 API 取出数据，那这个 Entry 对象还有用吗？所以最好的方法是在`ThreadLocal`对象被回收后，系统自动回收对应的`Entry`对象
所以，让key（threadLocal对象）为弱引用，自动被垃圾回收，`key`就变为`null`了，后面就可以通过`Entry`不为`null`，而`key`为`null`来判断该`Entry`对象该被清理掉了

# 那为什么不把 Enrty 或 Value 设计成弱引用
假设 ThreadLocal 对象是 static 的，那么强引用一直存在
如果 Enrty 没有被强引用，如果是弱引用，一次 GC 后就变成 null，就导致我们查询不到值
同理 Value 如果是在一个函数内 new 出来的，函数退出，强引用就没了，一次 GC 后 Value 也为 null，导致我们查不到值

# 内存泄漏
由于 ThreadLocalMap 的 key 是弱引用，而Value是强引用。这就导致了一个问题，ThreadLocal 在没有外部对象强引用时，发生 GC 时弱引用 Key 会被回收，而 Value 不会回收，如果创建 ThreadLocal 的线程一直持续运行，那么这个 Entry 对象中的 value 就有可能一直得不到回收，发生内存泄露。

既然 Key 是弱引用，那么我们要做的事，就是在调用 ThreadLocal 的 get()、set() 方法时完成后，确定当前线程不再使用该数据时，调用 remove 方法，将 Entry 节点和 Map 的引用关系移除，这样整个 Entry 对象在 GC Roots 分析后就变成不可达了，下次GC的时候就可以被回收。

如果使用 ThreadLocal 的 set 方法之后，没有显示的调用remove方法，就有可能发生内存泄露，所以养成良好的编程习惯十分重要，使用完ThreadLocal之后，记得调用remove方法

# 脏数据
ThreadLocalMap 定义了 expungeStaleEntry 在该方法里，除了清理指定下标 staleSlot 的 entry 外，还会历整个 entry table, 当发现有 key 为 nul 时，就会发 rehash 压缩整个 table 以达到清理的作用
expungeStaleEntry 方法是被安插到了 ThreadLocal.ThreadLocalMap 中的 get ， set ， remove 等方法中,并被 ThreadLocal 的 get，set，remove 方法间接调用，必须显式得调用这些方法, 才能主动式地清理空间

我们通常会将 ThreadLocal 定义为 static final ，这样其实就不存在 threadlocal 被回收的情况了， WeakReference 机制也将效用有限
如果某些 threadloca 设置的 value 是大对象,而所涉及的 thread 却没来得及在 threadlocal 被 GC 前 remove ，再加上之后也没有什么其他 threadlocal 去作 get 、 set 操作，那这些大对象是没机会被回收的,这将造成严重的内存泄露甚至是 OOM

所以线程池复用也会产生脏数据。由于线程池会重用 Thread 对象 ，那么与 Thread 绑定的类的静态属性 ThreadLocal 变量也会被重用。如果在实现的线程 run() 方法体中不显式地调用 remove() 清理与线程相关的 TbreadLocal 信息，那么倘若下一个线程不调用 set() 设置初始值，就可能 get() 到重用的线程信息，包括 ThreadLocal 所关联的线程对象的 value 值

# 使用建议
1. 设计为 static 的，被 class 对象给强引用，线程存活期间就不会被回收，也不用remove，完全不用担心内存泄漏
2. 设计为非 static 的长对象（比如被 spring 管理的对象）的内部，也不会被回收
3. 方法内创建使用 ThreadLocal 要谨记一点：用完主动 remove ，主动释放内存,而且是放在 finally 块里面 remove，以确保执行
```Java
try {
    // 逻辑
} finally {
    threadLocal.remove();
}
```
当然，我们不要在方法内创建 ThreadLocal 对象，也就不存在上述问题
只要让ThreadLocal具有线程的生命周期，就完全没必要使用remove方法，也完全不用担心内存泄漏的问题

# 应用场景
使用ThreadLocal的典型场景正如上面的数据库连接管理，线程会话管理等场景，只适用于独立变量副本的情况，如果变量为全局共享的，则不适用在高并发下使用

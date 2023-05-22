___
# 概述
## 数据结构
一维数组+链表+红黑树

___
# 类图
![[Pasted image 20230513194931.png]]

___
# 源码分析
`HashMap`的具体数据存放的成员变量是`table`，数据结构是一个一维数组`Node<K,V>[]`
其中的`Node<K,V>`是单向链表数据结构实现
`TreeNode<K,V>` 是红黑树实现，是`Node<K,V>`的一个子类实现
![[Pasted image 20230513195943.png]]

所以`HashMap`的数据结构是**数组+链表+红黑树**
其中链表到红黑树还有一套规则，参考[[#扩容机制]]
使用红黑树，是为了用空间换时间，将查询的时间复杂度由`O(n)`降为`O(logn)`
为什么不一开始就使用红黑树呢？
因为红黑树的维护更加复杂，当链表很短的时候，直接遍历比红黑树的调整更有效率，而当链表长度到一定长度再使用红黑树，查找效率高于树的调整才更有意义，所以设计者为此订下一些数值，将 8 作为转换临界值
```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
    ...
    /**
     * The table, initialized on first use, and resized as
     * necessary. When allocated, length is always a power of two.
     * (We also tolerate length zero in some operations to allow
     * bootstrapping mechanics that are currently not needed.)
     */
    transient Node<K,V>[] table;
    ...
    ...
    /**
     * Basic hash bin node, used for most entries.  (See below for
     * TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
     */
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
        ...
    }
    ...
    ...
    /**
     * Entry for Tree bins. Extends LinkedHashMap.Entry (which in turn
     * extends Node) so can be used as extension of either regular or
     * linked node.
     */
    static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }

        /**
         * Returns root of tree containing this node.
         */
        final TreeNode<K,V> root() {
            for (TreeNode<K,V> r = this, p;;) {
                if ((p = r.parent) == null)
                    return r;
                r = p;
            }
        }
    }
    ...
}
```

# 扩容机制
默认构造函数得到的`HashMap`的`tabld`是`null`的，作为延迟加载，会在第一次`put`操作的时候才初始化，初始化默认长度为 16 的数组
当数组使用达到长度×扩容因子，会再次进行扩容，扩容长度为一倍
当 hash 冲突导致链表长度等于 8 的时候，会进行一次树化判断，如果数组长度小于 64 ，会进行扩容，而不是树化，当数组长度大于等于 64 ，会将当前的链表转换为红黑树
```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
    ...

        /**
     * Constructs an empty <tt>HashMap</tt> with the default initial capacity
     * (16) and the default load factor (0.75).
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
    ...
    ...
        /**
     * Implements Map.put and related methods.
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
    ...

```

# 红黑树退化
## 红黑树退化一
进行扩容`resize()`时，红黑树拆分成的树的结点数小于等于临界值6个，则退化成链表
`HashMap`类常量定义了树退化的个数临界值
```java
    /**
     * 定义了红黑树退化的临界值，在 resize() 方法执行扩容过程中，如果链表是红黑树，
     * 会将红黑树进行拆分，如果拆分后节点数量小于等于 6 ，会退化成链表
     */
    static final int UNTREEIFY_THRESHOLD = 6;
```
扩容过程对红黑树结构进行拆分
```java
    final Node<K,V>[] resize() {
        ...
        else if (e instanceof TreeNode)
            // 如果当前的节点是红黑树结构，则进行红黑树的拆分，可能会将红黑树退化为链表
            ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
        ...
    }
```
扩容时如果是红黑树结构会执行红黑树的 split( ) 方法
split 方法中会初始化生成 loHead 和 hiHead 两个红黑树的头结点（之后会用 low 和 high 表示）
lc 和 hc 分别为 low 和 high 的元素个数，初始化为0
把红黑树中的结点依次添加到 low 和 high 两颗红黑树中，依靠 (e.hash & bit) == 0 的位运算来判断属于哪颗树，bit是传过来的旧数组下标。每颗树元素的增加都会使对应的 lc 或 hc 自增1。
生成完 low 和 high 两颗红黑树后，如果对应的元素个数 lc，hc 小于等于6，退化成链表，否则维持红黑树形态。
low树插入到新数组 tab[index] 的位置上，index是当前红黑树所在旧数组坐标；high树插入到新数组 tab[index + bit] 的位置上，bit是旧数组长度。
```java
    /**
     * Splits nodes in a tree bin into lower and upper tree bins,
     * or untreeifies if now too small. Called only from resize;
     * see above discussion about split bits and indices.
     *
     * @param map the map
     * @param tab the table for recording bin heads
     * @param index the index of the table being split
     * @param bit the bit of hash to split on
     */
    final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
        TreeNode<K,V> b = this;
        // Relink into lo and hi lists, preserving order
        TreeNode<K,V> loHead = null, loTail = null;
        TreeNode<K,V> hiHead = null, hiTail = null;
        int lc = 0, hc = 0;
        for (TreeNode<K,V> e = b, next; e != null; e = next) {
            next = (TreeNode<K,V>)e.next;
            e.next = null;
            if ((e.hash & bit) == 0) {
                if ((e.prev = loTail) == null)
                    loHead = e;
                else
                    loTail.next = e;
                loTail = e;
                ++lc;
            }
            else {
                if ((e.prev = hiTail) == null)
                    hiHead = e;
                else
                    hiTail.next = e;
                hiTail = e;
                ++hc;
            }
        }

        if (loHead != null) {
            if (lc <= UNTREEIFY_THRESHOLD)
                tab[index] = loHead.untreeify(map);
            else {
                tab[index] = loHead;
                if (hiHead != null) // (else is already treeified)
                    loHead.treeify(tab);
            }
        }
        if (hiHead != null) {
            if (hc <= UNTREEIFY_THRESHOLD)
                tab[index + bit] = hiHead.untreeify(map);
            else {
                tab[index + bit] = hiHead;
                if (loHead != null)
                    hiHead.treeify(tab);
            }
        }
    }
```
## 红黑树退化二
移除元素 `remove()` 时，在`removeTreeNode()`方法会检查红黑树是否满足退化条件，与结点数无关，满足以下三项之一
1. 如果红黑树根 root 为空
2. 或者 root 的左子树/右子树为空
3. root.left.left 根的左子树的左子树为空
都会发生红黑树退化成链表
```java
final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab,
                                  boolean movable) {
    ...
    if (root == null
    || (movable
        && (root.right == null
            || (rl = root.left) == null
            || rl.left == null))) {
        tab[index] = first.untreeify(map);  // too small
        return;
    }
    ...
}
```



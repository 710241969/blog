___
# 概述

## 数据结构
双向链表实现的无界列表

___
# 源码分析

## 获取元素
```java
    /**
     * Returns the element at the specified position in this list.
     *
     * @param index index of the element to return
     * @return the element at the specified position in this list
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public E get(int index) {
        checkElementIndex(index);
        return node(index).item;
    }
    
    private void checkElementIndex(int index) {
        if (!isElementIndex(index))
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
    
    /**
     * Tells if the argument is the index of an existing element.
     */
    private boolean isElementIndex(int index) {
        return index >= 0 && index < size;
    }

    /**
     * Returns the (non-null) Node at the specified element index.
     */
    Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```
查找元素时，会先比较 index 和双向链表长度的 1/2 （即代码的 size >> 1 ）；若前者大，则从链表头开始往后查找，直到location位置；否则，从链表末尾开始先前查找，直到location位置

所以，不要采用 get() 访问的方式去遍历`LinkedList`
```java
for (int i=0, size=list.size(); i<size; i++) {
    list.get(i);
}
```

正确的做法是
1. 使用迭代器遍历
```java
    LinkedList<String> list = new LinkedList<>();
    Iterator<String> iterator = list.iterator();
    while (iterator.hasNext()){
        System.out.println(iterator.next());
    }
```
2. 使用foreach遍历
```java
    LinkedList<String> list = new LinkedList<>();
    for (String s:list){
        System.out.println(s);
    }
```
但其实使用foreach遍历和使用迭代器遍历是一样的，使用foreach遍历，代码编译的时候也会转变成迭代器遍历：
```java
    LinkedList<String> list = new LinkedList();
    Iterator var3 = list.iterator();
    while(var3.hasNext()) {
        String s = (String)var3.next();
        System.out.println(s);
    }    
```






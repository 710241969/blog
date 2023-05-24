
___
# ==
java 是[[值传递]]的，== 号比较的就是两个值是否相等

对象类型是的值是对象在堆中的地址，此时比较两个对象 == 就是比较这两个地址的值是否相等


___
# equals 和 ==
java.lang.Object#equals
```java
    public boolean equals(Object obj) {
        return (this == obj);
    }
```
默认的 euqals 实现的就是 ==

___
# 为什么重写 equals
当我们比较两个对象的时候，默认在比较这两个对象的地址的值

在业务中，两个对象的成员变量值完全相等，我们希望他们能够被看作是一样的对象，就需要重写 equals 了

___
# hashCode
java.lang.Object#hashCode
```java
    public native int hashCode();
```
hashcode 是保存在对象头里面的。
是个 native 调用。

___
# 为什么重写 hashCode
正如[[#为什么重写 equals]]说的，我们希望两个对象被当做是相等，equals 是可以的。

但是我们在使用 java 一些集合类的时候，存入操作判断是否重复是通过 hashCode 来判断的，为了能够让这些集合正确工作，我们需要重写 hashCode ，让我们希望相等的对象能够返回相同的 hashCode

___
# 总结
在Effective Java一书中，提到了一点：**在每个覆盖了equals方法的类中，都必须覆盖hashCode方法**





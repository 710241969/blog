# 概述
ConcurrentSkipListSet 是线程安全的有序的集合，相当于线程安全的TreeSet

通过`ConcurrentSkipListMap`实现，只用到了`ConcurrentSkipListMap`中的`key`，`value` 是`Boolean.TRUE`

详情看[[ConcurrentSkipListMap]]
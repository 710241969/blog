LRU，Least Recently Used
最近最少使用算法

# 数据结构
利用双向链表+Map即可，JAVA 可直接利用 LinkedHashMap 来做
如果 key 存在，则删除 key ，再重新插入
如果 key 不存在
* 如果缓存已经满了，就删除掉第一个位置的 key
* 如果缓存未满，则直接插入
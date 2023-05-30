
___
# 能不能使用 join 语句
如果可以使用 Index Nested-Loop Join 算法，也就是说可以用上被驱动表上的索引，其实是没问题的；

如果使用 Block Nested-Loop Join 算法，扫描行数就会过多。尤其是在大表上的 join 操作，这样可能要扫描被驱动表很多次，会占用大量的系统资源。所以这种 join 尽量不要用。所以你在判断要不要使用 join 语句时，**就是看 explain 结果里面，Extra 字段里面有没有出现“Block Nested Loop”字样**。

所以你在判断要不要使用 join 语句时，就是看 explain 结果里面，Extra 字段里面有没有出现“Block Nested Loop”字样。

___
# 如果要使用 join，应该选择大表做驱动表还是选择小表做驱动表？
如果是 Index Nested-Loop Join 算法，应该选择小表做驱动表；
如果是 Block Nested-Loop Join 算法：在 join_buffer_size 足够大的时候，是一样的；在 join_buffer_size 不够大的时候（这种情况更常见），应该选择小表做驱动表。

所以，这个问题的结论就是，**总是应该使用小表做驱动表**。

___
# 如何选择驱动表？什么是小表？
**在决定哪个表做驱动表的时候，应该是两个表按照各自的条件过滤，过滤完成之后，计算参与 join 的各个字段的总数据量，数据量小的那个表，就是“小表”，应该作为驱动表。**



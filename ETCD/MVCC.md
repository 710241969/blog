etcd v3 支持 MVCC，可以保存一个键值对的多个历史版本

日志应用到MVCC模块，实现真正的存储。

MVCC模块包含treeIndex和boltdb，treeIndex在内存中
treeIndex 由 B 树实现，维护版本号和用户key的映射关系
boltdb 底层由 B+ 树实现，真正的数据存储在boltdb中，启动时会加载到内存

在 treeIndex 通过 key 找版本，再到存储引擎找到 KV

# 读写机制
etcd 是串行写，并发读。  

读写都先从 treeIndex 中根据查询的 key 从 B-tree 查找得到的是一个 keyIndex 对象，里面包含了 Revision 等全局版本号信息

写发生在内存 TreeIndex 中
读则直接查询 BoltDB 中的数据
事务提交后，数据由 Backend 刷盘

数据持久化的操作由 Backend 的协程来完成，以此提高写的性能和吞吐量。协程通过事务批量提交，将 BoltDB 内存中的数据持久化存储磁盘中。

事务开启的时候，读数据读的是B+树中，通过mmap映射的磁盘数据，因为事务开启期间的数据还未落盘，所以读取的是事务开启之前的数据。如果不采用这种方式，读数据就可能出现脏读的问题。  

为什么写是串行的而不是并发的呢？如果写是并发的，事务就很难实现了。目前boltdb就是这种做法，**写是先写内存，读却是读的磁盘映射到内存中的数据**，这就不会涉及到加锁问题。事务提交之后，数据会刷盘。  

如果没有开启事务，读写都是走的B树。

## 读
![[Pasted image 20230522130253.png]]
从 treeIndex 中获取 key hello 的版本号，再以版本号作为 boltdb 的 key，从 boltdb 中获取其 value 信息

RANGE(查找)
1.**开启一个读事务，并获取当前系统最新的版本 ID(main) currRev**。
2.根据 key 和 currRev 从 treeIndex 中**查找 key 中所有版本号中第一个小于等于 currRev 的 revision**。 比如 main=3, 那么查找到的 revision 就是(2,3)。
3.**根据 revision, 生成 bbolt key(2_3)值，并从 bbolt 中获取 keyValue 值**。


## TreeIndex：内存索引模块 treeIndex，保存 key 的历史版本号信息
B 树实现，treeIndex 模块是基于 Google 开源的内存版 btree 库实现的

`key`为业务的`key`，`value`为`keyIndex`
treeIndex 模块只会保存用户的 key 和相关版本号信息，用户 key 的 value 数据存储在 boltdb 里面，相比 ZooKeeper 和 etcd v2 全内存存储，etcd v3 对内存要求更低。

### keyIndex 
结构体定义如下所示：
```go
type keyIndex struct {
   key         []byte //用户的key名称，比如我们案例中的"hello"
   modified    revision //最后一次修改key时的etcd版本号,比如我们案例中的刚写入hello为world1时的，版本号为2
   generations []generation //generation保存了一个key若干代版本号信息，每代中包含对key的多次修改的版本号列表
}
```
在 ETCD 中，一个 KEY 由始至终都只有同一个`keyIndex`对象

结构体中，key为原始的业务key, modified为该key值最后一次修改对应的revision, generations为一个数组，其中每一个元素表示该key的一个生命周期(从创建到删除为一个生命周期)，因为同一个key，可能会删除之后又创建了，那么会在数组append一个新的generation信息

`[]generation`这个数组维护了一个 Key 的全部生命周期，比如: [新建，修改，修改，修改，删除，新建，修改，修改，删除]

### generations
其中 generations 的结构体定义如下：
```go
// generation contains multiple revisions of a key.
type generation struct {
   ver     int64    //表示此key的修改次数
   created revision //表示generation结构创建时的版本号
   revs    []revision //每次修改key时的revision追加到此数组
}
```

### Revision
结构体的定义如下：
```go
type revision struct {
   main int64    // 一个全局递增的主版本号，随put/txn/delete事务递增，一个事务内的key main版本号是一致的
   sub int64    // 一个事务内的子版本号，从0开始随事务内put/delete操作递增
}
```
对于revision可以理解为唯一并且递增的序列。
Revision 中定义了一个**全局递增**的主版本号事务 ID main，发生 put、txn、del 操作会递增，一个事务内的 main 版本号是唯一的；事务内的子版本号定义为sub，事务发生 put 和 del 操作时，从 0 开始递增。sub在每个事务中都从0开始。

## Buffer
在获取到版本号信息后，就可从 boltdb 模块中获取用户的 key-value 数据了。不过有一点你要注意，并不是所有请求都一定要从 boltdb 获取数据。etcd 出于数据一致性、性能等考虑，在访问 boltdb 前，首先会从一个内存读事务 buffer 中，二分查找你要访问 key 是否在 buffer 里面，若命中则直接返回。

**并发读特性的核心原理是创建读事务对象时，它会全量拷贝当前写事务未提交的 buffer 数据，并发的读写事务不再阻塞在一个 buffer 资源锁上，实现了全并发读。**


## boltdb：持久化存储 key-value 数据
boltdb若 buffer 未命中，此时就真正需要向 boltdb 模块查询数据了。

BoltDB 是一个基于 B+ 树的 KV 存储数据库，具有如下特性：
- 使用mmap技术，避免IO操作，简单来讲就是：一般情况下进程读取文件内容，需要将文件内容复制到内核空间，再从内核空间复制到用户空间，而mmap技术可以直接通过指针直接读取该段内存，底层的操作系统能自动将数据写会到对应的文件中，提高了文件读写的效率
- 使用Copy-On-Write技术，提高读操作的并发，简单来讲就是复制一个文件时并不会把原先的文件复制一份到内存，而是在内存中做一个映射指向原始文件，只有当对文件有修改时才会复制更新后的文件到内存并且修改映射到新的地址
- 内部使用B+树实现
- 使用golang语言开发
- 支持完全可序列化ACID事务

**boltdb 里每个 bucket 类似对应 MySQL 一个表，用户的 key 数据存放的 bucket 的名字是 「key」，etcd MVCC 元数据存放的 bucket 是 「meta」。**

BoltDB中的B+树存储，B+树的非叶子节点存储的是[[#Revision]]，叶子节点存储的是才是值[[#KeyValue]]
ETCD 的 v3 版本存放了 KEY 的历史数据

### KeyValue
```go
type KeyValue struct {
	// 键
	Key []byte `protobuf:"bytes,1,opt,name=key,proto3" json:"key,omitempty"`
	// 创建时的版本号
	CreateRevision int64 `protobuf:"varint,2,opt,name=create_revision,json=createRevision,proto3" json:"create_revision,omitempty"`
	// 最后一次修改的版本号
	ModRevision int64 `protobuf:"varint,3,opt,name=mod_revision,json=modRevision,proto3" json:"mod_revision,omitempty"`
	// 表示 key 的修改次数，删除 key 会重置为 0，key 的更新会导致 version 增加
	Version int64 `protobuf:"varint,4,opt,name=version,proto3" json:"version,omitempty"`
	// 值
	Value []byte `protobuf:"bytes,5,opt,name=value,proto3" json:"value,omitempty"`
	// 键值对绑定的租约 LeaseId，0 表示未绑定
	Lease int64 `protobuf:"varint,6,opt,name=lease,proto3" json:"lease,omitempty"`
}
```
### 文件
- Boltdb文件指的是etcd数据目录下的member/snap/db的文件，etcd的keyvalue、lease、meta、member、cluster、auth等所有数据存储在其中。
- etcd启动的时候，会通过mmap机制将db文件映射到内存，后续可从内存中快速读取文件中的数据
- 写请求通过fwrite和fdatasync来写入、持久化数据到磁盘
- 文件的内容由若干个page组成，一般情况下page size为4KB
- page按照功能可分为元数据页(meta page)、B+ tree索引节点页(branch page)、B+ tree叶子节点页(leaf page)、空闲页管理页(freelist page)、空闲页(free page)
- 文件最开头的两个page是固定的db元数据meta page
- 空闲页管理页记录了db中哪些页是空闲、可使用的
- 索引节点页保存了B+ tree的内部节点，如图中右边部分所示，它们记录了key值
- **叶子节点页记录了B+ tree中的key-value和bucket数据**
- boltdb逻辑上通过B+ tree来管理branch/leaf page，实现快速查找、写入key-value数据

与其他的 KV 存储组件使用存放数据的键作为 key 不同，etcd 存储以数据的 Revision 作为 key，键值、创建时的版本号、最后修改的版本号等作为 value 保存到数据库。etcd 对于每一个键值对都维护了一个全局的 Revision 版本号，键值对的每一次变化都会被记录。获取某一个 key 对应的值时，需要先获取该 key 对应的 Revision，再通过它找到对应的值。

### 总结
当你未带版本号查询 key 时，etcd 返回的是 key 最新版本数据。当你指定版本号读取数据时，etcd 实际上返回的是版本号生成那个时间点的快照数据。

## 写
![[Pasted image 20230522133631.png]]


事务提交的过程，包含 B+tree 的平衡、分裂，将 boltdb 的脏数据（dirty page）、元数据信息刷新到磁盘，因此事务提交的开销是昂贵的。如果我们每次更新都提交事务，etcd 写性能就会较差。那么解决的办法是什么呢？etcd 的解决方案是合并再合并。首先 boltdb key 是版本号，put/delete 操作时，都会基于当前版本号递增生成新的版本号，因此属于顺序写入，可以调整 boltdb 的 bucket.FillPercent 参数，使每个 page 填充更多数据，减少 page 的分裂次数并降低 db 空间。其次 etcd 通过合并多个写事务请求，通常情况下，是异步机制定时（默认每隔 100ms）将批量事务一次性提交（pending 事务过多才会触发同步提交）， 从而大大提高吞吐量

这优化又引发了另外的一个问题， 因为事务未提交，读请求可能无法从 boltdb 获取到最新数据。为了解决这个问题，etcd 引入了一个 bucket buffer 来保存暂未提交的事务数据。在更新 boltdb 的时候，etcd 也会同步数据到 bucket buffer。因此 etcd 处理读请求的时候会优先从 bucket buffer 里面读取，其次再从 boltdb 读，通过 bucket buffer 实现读写性能提升，同时保证数据一致性

### 总结
删除一个数据时，etcd 并未真正删除它，而是基于 lazy delete 实现的异步删除。删除原理本质上与更新操作类似，只不过 boltdb 的 key 会打上删除标记，keyIndex 索引中追加空的 generation。真正删除 key 是通过 etcd 的压缩组件去异步实现的，在后面的课程里我会继续和你深入介绍。
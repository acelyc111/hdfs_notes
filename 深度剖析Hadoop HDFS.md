# 第一部分 核心设计篇

## 第1章 HDFS的数据存储

HDFS的数据存储包括两块：一块是HDFS内存存储，另一块是HDFS异构存储。

- HDFS内存存储是一种十分特殊的存储方式，将会对集群数据的读写带来不小的性能提升
- HDFS异构存储则能帮助我们更加合理地把数据存到应该存的地方

### 1.1 HDFS内存存储

HDFS内存存储策略：LAZY_PERSIST 直接将内存作为数据存放的载体

可以这么理解，此时节点的内存也充当了一块“磁盘”。只要将文件设置为内存存储方式，最终会将其存储在节点的内存中。

#### 1.1.1 HDFS内存存储原理

![lazy_persist](./lazy_persist.png)

异步存储的大体步骤：

1. 对目标文件目录设置StoragePolicy为LAZY_PERSIST的内存存储策略
2. 客户端进程向NameNode发起创建/写文件的请求
3. 客户端请求到具体的DataNode后DataNode会把这些数据块写入RAM内存中，同时启动异步线程服务将内存数据持久化写到磁盘上。

#### 1.1.2 Linux 虚拟内存盘

虚拟内存盘（RAM disk）

这是一种模拟的盘，实际数据都是存放在内存中的。虚拟内存盘可以在某些特定的内存式存储文件系统下结合使用，比如tmpfs、ramfs。

通过此项技术，我们就可以将机器内存利用起来，作为一块独立的虚拟盘供DataNode使用了。

#### 1.1.3 HDFS的内存存储流程分析

##### 1. HDFS文件内存存储策略设置

设置存储策略的方法目前有以下3种：

- 通过命令行的方式，调用如下命令。这种方式比较方便、快速。

```
hdfs storagepolicies -setStoragePolicy -path <path> -policy LAZY_PERSIST
```

- 调用对应的程序方法，比如调用暴露在外部的create文件方法，但是得带上参数`CreateFlag.LAZY_PERSIST`

```
    FSDataOutputStream fos =
        fs.create(
            path,
            FsPermission.getFileDefault(),
            EnumSet.of(CreateFlag.CREATE, CreateFlag.LAZY_PERSIST),
            bufferLength,
            replicationFactor,
            blockSize,
            null);
```

- 通过FileSystem的setStoragePolicy方法（2.8.0+）

```
fs.setStoragePolicy(path, "LAZY_PERSIST");
```

##### 2. LAZY_PERSIST内存存储

![FsDatasetImpl](./FsDatasetImpl.png)

- RamDiskAsyncLazyPersistService：

  异步持久化线程服务，针对每一个磁盘块设置一个对应的线程池，需要持久化到给定磁盘的数据块会被提交到对应的线程池中去。每个线程池的最大线程数为1。

- LazyWriter：

  这是一个线程服务，此线程会不断地从数据块列表中取出数据块，将数据块加入到异步持久化线程池RamDiskAsyncLazyPersistService中去执行。

- RamDiskReplicaLruTracker：

  是副本块跟踪类，此类中维护了所有已持久化、未持久化的副本以及总副本数据信息。所以当一个副本被最终存储到内存中后，相应地会有副本所属队列信息的变更。当节点内存不足时，会将最近最少被访问的副本块移除。

  （代码解析略）

#### 1.1.4 LAZY_PERSIST内存存储的使用

1. 配置虚拟内存盘
2. 将机器中已经完成好的虚拟内存盘配置到dfs.datanode.data.dir中，并带上RAM_DISK标签
3. 设置具体的文件策略类型

### 1.2 HDFS异构存储

异构存储可以根据各个存储介质读写特性的不同发挥各自的优势，如冷热分离。

#### 1.2.1 异构存储类型

RAM_DISK：内存存储（LAZY_PERSIST）

SSD：固态硬盘存储

DISK：机械盘存储（默认）

ARCHIVE：主要指的是高密度存储介质，用于解决数据扩容的问题

HDFS并没有自动检测识别的功能，需要在配置属性时主动声明。

配置属性dfs.datanode.data.dir可以对本地对应存储目录进行设置，同时带上一个存储类型标签：

```
[SSD]file:///grid/dn/ssd0
```

#### 1.2.2 异构存储原理

HDFS异构存储可总结为以下三点：

- DataNode通过心跳汇报自身数据存储目录的StorageType给NameNode。
- 随后NameNode进行汇总并更新集群内各个节点的存储类型情况。
- 待复制文件根据自身设定的存储策略信息向NameNode请求拥有此类型存储介质的DataNode作为候选节点。

（代码解析略）

#### 1.2.3 块存储类型选择策略

当前存储类型不可用的时候，退一级选择使用的存储类型

#### 1.2.4 块存储策略集合

根据冷热数据的角度区分：

- HOT
- COLD
- WARM

根据存放盘的性质区分：

- ALL_SSD
- ONE_SSD
- LAZY_PERSIST

#### 1.2.5 块存储策略的调用

HDFS的默认策略是 `HOT`，HDFS把集群中的数据都看成是经常访问的数据。

DN上存储的策略ID从何而来：

- 通过RPC接口主动设置
- 没有主动设置的ID会继承父目录的策略
- 如果父目录还是没有设置策略，则会设置ID_UNSPECIFIED，继而会用DEFAULT（默认）存储策略进行替代

#### 1.2.6 HDFS异构存储策略的不足之处

目前HDFS上还不能对文件目录存储策略变更做出自动的数据迁移。这里需要用户额外执行`hdfs -mover`命令做文件目录的扫描。在mover命令扫描的过程中，如果发现文件目录的实际存储类型与其所设置的storagePolicy策略不同，将会进行数据块的迁移，将数据迁移到相对应的存储介质中。

#### 1.2.7 HDFS存储策略的使用

```
hdfs storagepolicies
- setStoragePolicy
- listPolicies
- getStoragePolicy
```

最简单的使用方法是：

1. 事先划分好冷热数据的存储目录，设置好对应的存储策略
2. 使用相应的程序在对应分类目录下写数据，自动继承父目录的存储策略

在较新版的Hadoop发布版本中增加了数据迁移工具。此工具的重要用途在于它会：

```
hdfs mover
```

1. 扫描HDFS上的文件，判断文件是否满足其内部设置的存储策略
2. 如果不满足，就会重新迁移数据到目标存储类型节点上

## 第2章 HDFS的数据管理与策略选择

![cacheProcedure](./cacheProcedure.png)

### 2.1 HDFS缓存与缓存块

HDFS缓存的出现可以大大提高用户读取文件的速度，因为它是缓存在DataNode内存中的，此过程无需进行读取磁盘的操作。

在HDFS中缓存的对象是数据块，需要缓存的目标数据块称为CacheBlock，不需要缓存的数据块称为UnCacheBlock。

#### 2.1.1 HDFS物理层面缓存块

利用mmap、mlock这样的系统调用将块数据锁入内存，以此达到在DataNode上缓存数据的效果。

- mmap：mmap系统调用，它是一个内存映射调用。mmap主要作用是将一个文件或者其他对象映射进内存。

#### 2.1.2 缓存块的生命周期状态

在HDFS的缓存过程中有这四类缓存状态，并可以切换。

- CACHING：表示块正在被缓存
- CACHING_CANCELLED：正在被缓存的块已处于被取消的状态
- CACHED：表明数据块已被缓存
- UNCACHING：表明缓存块正处于取消缓存的状态

#### 2.1.3 CacheBlock、UnCacheBlock场景触发

##### 1. CacheBlock动作

- 此方法最终来自于NameNode心跳处理的方法

![cacheBlock](./cacheBlock.png)

##### 2. UnCacheBlock动作

- 当块执行append写操作时

  因为对块继续执行了写动作，数据必然发生改变，原有的缓存块需要重新更新

- 当把块处理为无效块时

  当把块处理为无效块的时候，接着会被NameNode从系统中清除，缓存块自然而言就没有存在的必要了

- 上层NameNode发送uncache回复命令时

![unCacheBlock](./unCacheBlock.png)

#### 2.1.4 CacheBlock、UnCacheBlock缓存块的确定

NameNode中的CacheReplicationMonitor自身持有一个系统中的标准缓存块列表，通过自身内部的缓存规则，进行缓存块的添加和移除，然后对应更新到之前提到过的pendingCache和pendingUncache列表中，随后这些信息会被NameNode拿来放入回复命令中。

##### CacheReplicationMonitor内部缓存规则

- 任何少于标准副本块个数的副本应该被缓存到新的节点上
- 过量副本数的缓存块应该从节点上进行移除

#### 2.1.5 系统持有的缓存块列表如何更新

因为缓存块列表是系统全局持有的，会存在反馈上报的过程，相关逻辑位于心跳处理部分。

缓存块的更新形成一个闭环。

#### 2.1.6 缓存块的使用

#### 2.1.7 HDFS缓存相关配置

```
    <property>
      <name>dfs.datanode.max.locked.memory</name>
      <value>0</value>
      <description>
      DataNode用来缓存块的最大内存空间大小，单位用字节表示。系统变量 RLIMIT_MEMLOCK 至少需要设置得比此配置要大，否则DataNode会出现启动失败的现象。在默认情况下，此配置值为0，表名默认关闭内存缓存的功能。
      </description>
    </property>
```

其他配置：

- `dfs.datanode.fsdatasetcache.max.threads.per.volume	：用于缓存块数据的最大线程数，这个线程数是针对每个存储目录而言，默认值为4
- `dfs.cachereport.intervalMsec`：缓存块上报间隔，默认10秒

注意：

- 此配置项会受系统最大可使用内存大小（RLIMIT_MEMLOCK）的影响，造成启动DataNode失败的现象

  可以通过`ulimit -l <value>`命令对此进行调整

- 此配置项并不是HDFS缓存机制所独有的，它与HDFS的LAZY_PERSIST策略会共享`dfs.datanode.max.locked.memory`配置

### 2.2 HDFS中心缓存管理

HDFS中心缓存管理机制主要依赖于中心缓存管理器（CacheManager）以及缓存块监控服务（CacheReplicationMonitor）。通过二者的协作，来控制集群缓存块的状态。

#### 2.2.1 HDFS缓存适用场景

- 缓存HDFS中的热点公共资源文件

  如：依赖资源 jar 包，或是一些算法学习依赖的 .so 文件等

- 缓存短期临时的热点数据文件

  如：集群中每天运行统计的报表数据，需要读取前一天的或是最近一周的数据做离线分析

#### 2.2.2 HDFS缓存的结构设计

![cacheManager](./cacheManager.png)

在HDFS中，最终缓存的本质是数据文件。但是在逻辑上，引入了下面几个概念。

##### 1. CacheDirective

CacheDirective是缓存的基本单元，但是这里CacheDirective不一定针对的是一个目录，也可以是一个文件。

##### 2. CachePool

缓存池中维护了一个缓存单元列表。同时这些缓存池被CacheManager所掌管，CacheManager在这里就好比一个总管理者的角色。

#### 2.2.3 HDFS缓存管理机制分析

CacheManager通过:

- id到CacheDirective

- 路径到CacheDirective列表

  对同一个缓存路径是可以被多次缓存的

- 名称到CachePool

的多个映射关系，使得原本逻辑上的父子关系结构平级化了，方便了多条件地灵活查询。

*比如说我们通过id去找对应的缓存对象，就不需要重新遍历查找了。*

##### 1. CacheAdmin CLI命令在CacheManager的实现

CacheAdmin是HDFS中缓存块的管理命令。在CacheAdmin中的每个操作命令，最后通过RPC调用都会对应到CacheManager中的一个具体操作方法。

##### 2. CacheReplicationMonitor缓存监控服务

（略）

![rescan](./rescan.png)

缓存副本块的监控服务，循环执行扫描、统计、重排等逻辑

1. resetStatistics重置统计变量计数值

   因为要进行完全新一轮的缓存过程，所以CachePool以及其所包含的CacheDirective都要重新计数

2. rescanCacheDirectives

   扫描之前保存在CacheManager中的那些CacheDirectives

3. rescanCachedBlockMap

#### 2.2.4 HDFS中心缓存疑问点

两个JIRA，略

#### 2.2.5 HDFS CacheAdmin命令使用

![cacheCLI](./cacheCLI.png)

### 2.3 HDFS快照管理

Snapshot

#### 2.3.1 快照概念

快照不是数据的简单拷贝，快照只做差异的记录

因为不保存实际的数据，所以快照的生成往往非常迅速

对于大多不变的数据，你所看到的数据其实是当前物理路径所指的内容，而发生变更的INode数据才会被快照额外拷贝，也就是前面所说的差异拷贝。

#### 2.3.2 HDFS中的快照相关命令

```
$ hadoop fs￼
  Usage: hadoop fs [generic options]￼
      [-createSnapshot <snapshotDir> [<snapshotName>]]     // 在指定快照目录下创建快照￼
      [-deleteSnapshot <snapshotDir> <snapshotName>]       // 在指定快照目录下删除快照￼
      [-renameSnapshot <snapshotDir> <oldName> <newName>]  // 在指定快照目录下重命名某快照

$ hdfs￼
  Usage: hdfs [--config confdir] [--loglevel loglevel] COMMAND￼
      where COMMAND is one of:￼
          snapshotDiff           // 比较两个快照之间的不同或是比较当前内容与某快照之间的不同￼
          lsSnapshottableDir     // 列出所属当前用户的所有的快照目录
```

一个快照目录下可以有多个快照文件，快照目录可以创建、删除自身目录下的快照文件，同时快照目录本身又被快照目录管理器所管理。

#### 2.3.3 HDFS内部的快照管理机制

##### 1. 快照结构关系

- 快照管理器管理多个快照目录
- 一个快照目录拥有多个快照文件

##### 2. 快照调用流程

SnapshotManager负责接收快照操作请求，继而调用相关类进行处理

![snapshot](./snapshot.png)

##### 3. 快照原理实现分析

创建快照之前，需要对目标目录执行allowSnapshot操作，使得对目录能够有创建快照的权限

会在快照目录下的隐藏目录 ./snapshot 下创建目标快照

注意：不允许创建出网状关系（NestedSnapshots）的快照目录，就是目标快照目录的子目录和父目录不能够同样为快照目录

计数：

- 每次新增快照时，Counter计数会加1，然后做计数判断，这里的MaxSnapshotID指的是上限值：1<<24 - 1
- 在每个目录下又会有快照总数的限制：1<<16

获取快照的数据：

- 如果当前快照id不是Snapshot.CURRENT_STATE_ID，则从对应的快照中获取结果，否则从当前的目录中获取结果

最终的孩子列表是通过将diff发生过变更的INode信息与原目录节点信息进行结合，然后将一个新的子节点信息作为最终结果返回。diff中保留的INode就是当时快照创建时的INode信息。

- HDFS中只为每个快照保存相对当时快照创建时间点发生过变更的INode信息，只是“存不同”
- 获取快照信息时，根据快照Id和当前没发生过变更的INode信息，进行对应恢复

快照之间的比差异功能对于使用者来说是非常实用的功能。因为通过比较不同时间点创建的快照，我们可以知道在此期间到底哪些文件目录被修改、创建或删除，甚至还能通过这些差异数据做元数据同步。

（diff的代码实现，略）

#### 2.3.4 HDFS的快照使用

- 丢失数据的恢复

  这里丢失数据指的是相对于创建快照时间点之后丢失的数据

- 元数据的差异比较

  HDFS的快照能够提供diff比较的功能。比较的结果会展示相对于源端快照，目标快照中发生的文件目录的变更记录。这个差异结果可以用于数据的同步，比如快照在DistCp命令中的使用。比如用于实现集群数据间的同步。

### 2.4 HDFS副本放置策略(BlockPlacementPolicy)

一个文件块的落盘过程：

1. 存储类型策略

   -> 筛选一批符合存储类型要求的存储位置列表

2. 副本放置策略

#### 2.4.1 副本放置策略概念与方法

##### BlockPlacementPolicyDefault策略

1. 如果写请求方所在机器是其中一个DataNode，则直接存放在本地，否则随机在集群中选择一个DataNode
2. 第二个副本存放于不同于第一个副本所在的机架
3. 第三个副本存放于第二个副本所在的机架，但是属于不同的节点

![block_placement_policy_default](./block_placement_policy_default.png)

#### 2.4.2 副本放置策略的有效前提

如果机架感知功能关闭并不会导致副本放置策略的失败，但是副本放置策略在这种情况下会失效。

```
<property>￼  <name>net.topology.script.file.name</name>￼  <value>/path/to/rackAware.py</value>￼</property>
```

#### 2.4.3 默认副本放置策略的分析

##### 策略核心方法chooseTargets

favoredNodes：使得方法在选择节点时候优先选取偏爱节点参数中的节点

1. 初始化操作

2. 选择目标节点

3. 对目标节点列表进行排序，形成Pipeline

   对已经选择好的目标节点存放位置进行排序，然后形成Pipeline进行返回。

   就是从writer所在节点开始，总是寻找相对路径最短的目标节点，最终形成Pipeline。（TSP旅行商问题）

   *就是选出一个源节点，根据这个节点，遍历当前可选的下一个目标节点，找出一个最短距离的节点，作为下一轮选举的源节点。这样每两个节点之间的距离总是最近的，于是整个Pipeline节点间的距离和就能保证是足够小的了*

###### HDFS中节点间的距离

LCA（最近公共祖先算法）

![rack_nodes](./rack_nodes.png)

##### chooseTarget方法主逻辑

1. 首节点的选择

   - 如果writer请求方本身位于集群中的一个DataNode之上，则第一个副本的位置就在本地节点上
   - 如果result中还是没有任何节点，则会从集群中随机挑选一个节点作为第一个节点

2. 三副本位置的选取

   如上的pipeline过程

3. 策略函数

   优先级递减：

   chooseLocalStorage

   chooseLocalRack

   chooseRemoteRack

   chooseRandom

#### 2.4.4 目标存储好坏的判断

最后一关：对于存储的验证

条件：

- 存储的存储类型必须是请求的存储类型
- 存储不能是READ_ONLY（只读）
- 存储不能是坏的
- 存储所在节点不应该是已下线或下线中的节点
- 存储所在节点不应该是消息落后的节点，实际指的是一段时间内没有更新心跳的节点
- 节点内保证有足够的剩余空间能满足写块所要求的大小
- 要考虑节点的IO负载繁忙程度
- 要满足同机架内最大副本数的限制

#### 2.4.5 chooseTargets的调用

略

#### 2.4.6 BlockPlacementPolicyWithNodeGroup继承类

在Rack机架层下还多了Node-Group层

![NodeGroup](./NodeGroup.png)

#### 2.4.7 副本放置策略的结果验证

我们需要有一种方式能够检测块当前的详细位置，这样我们才能判断是否满足HDFS的副本放置策略

```
 hdfs fsck <path> -files -blocks -locations
```

### 2.5 HDFS内部的认证机制

#### 2.5.1 BlockToken认证

BlockToken认证是基于令牌的块级别粒度的验证

略

#### 2.5.2 HDFS的Sasl认证

Sasl 是一套公开的认证机制，全称是Simple Authentication andSecurity Layer，中文翻译为“简单认证与安全层”，是一种用来扩充C/S模式验证能力的机制。

##### 1. SaslClient与SaslServer的握手

##### 2. DoSaslHandshake

进入真正握手阶段的方法中，会提前一步进行用户、密码的构造过程

![sasl_handshake](./sasl_handshake.png)

##### 3. SaslInputStream和SaslOutputStream的“多余处理”

#### 2.5.3 BlockToken认证与HDFS的Sasl认证对比

##### 共同点

- 没有空间局部的限制，都是数据全局的认证
- 都会对数据读写效率造成一定程度的影响

##### 不同点

- 认证维度不同。

  BlockToken认证的粒度较细，是针对块级别的认证，会对每次的块操作做认证。Sasl则是针对每次数据传输操作做认证。

- 复杂性不同。

  BlockToken的认证过程相对简单、清晰。而Sasl认证体系则复杂一些，会经过握手阶段，而且中间还可以配置相关的认证防护级别（Qop）的参数。论完整度而言，Sasl比BlockToken更加完整化、体系化一些。

### 2.6 HDFS内部的磁盘目录服务

HDFS在DataNode所在的节点中启动了多种磁盘目录的检测服务，来保证数据的完整性与一致性

#### 2.6.1 HDFS的三大磁盘目录检测扫描服务

##### DiskChecker（磁盘）

**坏盘检测服务**。检测的级别是每个**磁盘**，检测的对象是FsVolume, FsVolume对应一个存储数据的磁盘。通过检测文件目录的访问权限以及目录是否可创建来判断目录所属磁盘的好坏，如果是坏盘，则此块盘将会被移除，上面的所有块都将被重新复制。

##### DirectoryScanner（目录）

目录扫描服务，对每块盘上的目录做扫描，使之**与内存中维护的块信息同步**。比如存储在磁盘上的块已经没有了，则内存中的块信息也应该被移除。

##### VolumeScanner（块）

磁盘目录扫描服务。从名称上来看，VolumeScanner与DirectoryScanner比较类似，但是VolumeScanner才是真正意义上的**块检查服务**。它会对已发现的“可疑块”做检查，判断此块是否为损坏块，如果是，则会将其汇报给NameNode。

#### 2.6.2 DiskChecker：坏盘检测服务

DiskChecker服务并不是一个周期性的定时任务，它只会在可能有坏盘出现的场景中被启动，然后执行

##### 1. DiskChecker何时被调用

![disk_cheker](./disk_cheker.png)

##### 2. DiskChecker坏盘检测原理

不同的BlockPool在每个盘上的存储是以BP打头的目录做区分的，类似格式如下，其中xx.xx. xx.xx代表的是当时做格式化操作的NameNode的ip地址：

```
BP-805037254-xx.xx.xx.xx-1460537955319
```

检测3类目录：

- finalizedDir目录，已经完成后的块文件存储目录，层级不止一层，子目录下还存在子目录，所以在此处需要递归地检查。
- tmpDir临时目录，存储临时副本的目录。
- rbwDir目录，正在写操作的文件会存放于此目录，写完成之后，会被移入到finalizedDir目录中。

检测过程：

- 创建目录的检测。在这里会通过执行mkdir的方法来判断是否能够创建出目录。
- 访问权限的检测。检测的逻辑是判断目录的是否能够进行读、写和执行。

结果：

- 坏盘被DiskChecker检测出来之后，会在NameNode的50070端口页面中显示出来，集群管理人员看到了可以做后续的处理工作。

##### 3. DiskChecker注意点

- 坏盘检测的误判

- 大量的坏盘导致DataNode启动的失败

  配置：dfs.datanode.failed.volumes.tolerated

#### 2.6.3 DirectoryScanner：目录扫描服务

阶段性扫描块以及块的元数据文件，使之与DataNode内存中维护的数据趋向一致。

##### 1. scan生成diff差异报告

diff差异报告的生成需要同时获取磁盘上的块信息报告和内存中的块信息报告，然后做具体维度的比较

- 内存中的块丢失，而磁盘中的块还在
- 磁盘中的块丢失，而内存中的块还在
- 块元数据文件存在，而块文件不存在
- 块元数据文件中的版本值或文件长度不一致
- 块文件对象不一致

##### 2. DataNode的更新操作

比较完毕之后，就要进行相应数据的更新趋同操作了：checkAndUpdate

DirectoryScanner是一项周期性的服务，默认间隔执行时间6小时。DirectoryScanner像是一个“园丁”的角色，将这段时间内出现的一些异常的数据处理掉，维护内存中的数据与磁盘上块数据的完整性与一致性。

![directory_scanner](./directory_scanner.png)

#### 2.6.4 VolumeScanner：磁盘目录扫描服务

VolumeScanner是专门针对每块磁盘做块扫描的服务。块扫描类似于一次健康状况的检查。每个VolumeScanner扫描一块盘，并且每个VolumeScanner有属于它自己的独立线程。

##### 可疑块

在读数据时，如果发生了IO异常，则会进行可疑块的标记

- BlockSender在扫描块的时候，特意对其进行了**限流**，防止其对DataNode正常读写的影响

- 根据BlockSender读块时是否抛出IO异常来作为块好坏的评判标准

### 2.7 小结


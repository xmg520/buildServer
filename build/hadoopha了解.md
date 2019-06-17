# Hadoop HA机制
在搭建HA之前我们需要弄明白的是他的原理和机制。

# Hadoop1.X架构与2.X架构
在Hadoop1.X版本时，官方架构图如下：
![](https://upload-images.jianshu.io/upload_images/7013389-48623133ed631e5f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

hadoop1.X架构图

从1.X的官方架构图可以看出，整个集群通过一个Namenode进行管理，可以想象，当集群中的Namenode节点挂掉是整个集群将不可用。

2.X架构和1.X架构又有什么区别呢。我们来看2.X的架构：
![](https://upload-images.jianshu.io/upload_images/7013389-e7e1c250738b443d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Hadoop2.X架构图

2.X版本中，HDFS架构解决了单点故障的问题，即引入双Namenode机制，并且借助共享存储系统来进行元数据的同步，共享存储系统类型一般有几类，如：Shared NAS+NFS、BookKeeper、BackupNode 和 Quorum Journal Manager(QJM)，上图即用的QJM方式进行元数据同步，接下来介绍元数据相关内容。

# 元数据
Hadoop的元数据主要用来维护HDFS文件系统中的文件和目录相关信息。元数据的存储形式主要有3类：内存镜像、磁盘镜像(FSImage)、日志(EditLog)。在Namenode启动时，会加载磁盘镜像到内存中以进行元数据的管理，存储在NameNode内存；磁盘镜像是某一时刻HDFS的元数据信息的快照，包含所有相关Datanode节点文件块映射关系和命名空间(Namespace)信息，存储在NameNode本地文件系统；日志文件记录client发起的每一次操作信息，即保存所有对文件系统的修改操作，用于定期和磁盘镜像合并成最新镜像，保证NameNode元数据信息的完整，存储在NameNode本地和共享存储系统(QJM)中。

如下所示为NameNode本地的EditLog和FSImage文件格式，EditLog文件有两种状态： inprocess和finalized, inprocess表示正在写的日志文件，文件名形式:editsinprocess[start-txid]，finalized表示已经写完的日志文件,文件名形式：edits[start-txid][end-txid]； FSImage文件也有两种状态, finalized和checkpoint， finalized表示已经持久化磁盘的文件，文件名形式: fsimage_[end-txid], checkpoint表示合并中的fsimage, 2.x版本checkpoint过程在Standby Namenode(SNN)上进行，SNN会定期将本地FSImage和从QJM上拉回的ANN的EditLog进行合并，合并完后再通过RPC传回ANN。
```
data/hbase/runtime/namespace
├── current
│ ├── VERSION
│ ├── edits_0000000003619794209-0000000003619813881
│ ├── edits_0000000003619813882-0000000003619831665
│ ├── edits_0000000003619831666-0000000003619852153
│ ├── edits_0000000003619852154-0000000003619871027
│ ├── edits_0000000003619871028-0000000003619880765
│ ├── edits_0000000003619880766-0000000003620060869
│ ├── edits_inprogress_0000000003620060870
│ ├── fsimage_0000000003618370058
│ ├── fsimage_0000000003618370058.md5
│ ├── fsimage_0000000003620060869
│ ├── fsimage_0000000003620060869.md5
│ └── seen_txid
└── in_use.lock
```
上面所示的还有一个很重要的文件就是seen_txid,保存的是一个事务ID，这个事务ID是EditLog最新的一个结束事务id，当NameNode重启时，会顺序遍历从edits_0000000000000000001到seen_txid所记录的txid所在的日志文件，进行元数据恢复，如果该文件丢失或记录的事务ID有问题，会造成数据块信息的丢失。

HA其本质上就是要保证主备NN元数据是保持一致的，即保证fsimage和editlog在备NN上也是完整的。元数据的同步很大程度取决于EditLog的同步，而这步骤的关键就是共享文件系统，下面开始介绍一下关于QJM共享存储机制。

# QJM原理
## QJM介绍
QJM全称是Quorum Journal Manager, 由JournalNode（JN）组成，一般是奇数点结点组成。每个JournalNode对外有一个简易的RPC接口，以供NameNode读写EditLog到JN本地磁盘。当写EditLog时，NameNode会同时向所有JournalNode并行写文件，只要有N/2+1结点写成功则认为此次写操作成功，遵循Paxos协议。其内部实现框架如下：
![](https://upload-images.jianshu.io/upload_images/7013389-594e491b7b10a676.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
QJM内部实现

从图中可看出，主要是涉及EditLog的不同管理对象和输出流对象，每种对象发挥着各自不同作用：

- FSEditLog：所有EditLog操作的入口
- JournalSet: 集成本地磁盘和JournalNode集群上EditLog的相关操作
- FileJournalManager: 实现本地磁盘上 EditLog 操作
- QuorumJournalManager: 实现JournalNode 集群EditLog操作
- AsyncLoggerSet: 实现JournalNode 集群 EditLog 的写操作集合
- AsyncLogger：发起RPC请求到JN，执行具体的日志同步功能
- JournalNodeRpcServer：运行在 JournalNode 节点进程中的 RPC 服务，接收 - NameNode 端的 AsyncLogger 的 RPC 请求。
- JournalNodeHttpServer：运行在 JournalNode 节点进程中的 Http 服务，用于接收处于 Standby 状态的 NameNode 和其它 JournalNode 的同步 EditLog 文件流的请求。 下面具体分析下QJM的读写过程。
# QJM 写过程分析
上面提到EditLog，NameNode会把EditLog同时写到本地和JournalNode。写本地由配置中参数dfs.namenode.name.dir控制，写JN由参数dfs.namenode.shared.edits.dir控制，在写EditLog时会由两个不同的输出流来控制日志的写过程，分别为：EditLogFileOutputStream(本地输出流)和QuorumOutputStream(JN输出流)。写EditLog也不是直接写到磁盘中，为保证高吞吐，NameNode会分别为EditLogFileOutputStream和QuorumOutputStream定义两个同等大小的Buffer，大小大概是512KB，一个写Buffer(buffCurrent)，一个同步Buffer(buffReady)，这样可以一边写一边同步，所以EditLog是一个异步写过程，同时也是一个批量同步的过程，避免每写一笔就同步一次日志。

这个是怎么实现边写边同步的呢，这中间其实是有一个缓冲区交换的过程，即bufferCurrent和buffReady在达到条件时会触发交换，如bufferCurrent在达到阈值同时bufferReady的数据又同步完时，bufferReady数据会清空，同时会将bufferCurrent指针指向bufferReady以满足继续写，另外会将bufferReady指针指向bufferCurrent以提供继续同步EditLog。上面过程用流程图就是表示如下：
![](https://upload-images.jianshu.io/upload_images/7013389-ea30355e57ab0411.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

EditLog输出流程图

# 主备切换机制
![Failover流程图](https://upload-images.jianshu.io/upload_images/7013389-36f940edc3b91893.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从图中可以看出，整个切换过程是由ZKFC来控制的，具体又可分为HealthMonitor、ZKFailoverController和ActiveStandbyElector三个组件。

- ZKFailoverController: 是HealthMontior和ActiveStandbyElector的母体，执行具体的切换操作
- HealthMonitor: 监控NameNode健康状态，若状态异常会触发回调ZKFailoverController进行自动主备切换
- ActiveStandbyElector: 通知ZK执行主备选举，若ZK完成变更，会回调ZKFailoverController相应方法进行主备状态切换
在故障切换期间，ZooKeeper主要是发挥什么作用呢，有以下几点：

- 失败保护：集群中每一个NameNode都会在ZooKeeper维护一个持久的session,机器一旦挂掉，session就会过期，故障迁移就会触发

- Active NameNode选择：ZooKeeper有一个选择ActiveNN的机制，一旦现有的ANN宕机，其他NameNode可以向ZooKeeper申请排他成为下一个Active节点

- 防脑裂： ZK本身是强一致和高可用的，可以用它来保证同一时刻只有一个活动节点

那在哪些场景会触发自动切换呢，从HDFS-2185中归纳了以下几个场景：

- ActiveNN JVM崩溃：ANN上HealthMonitor状态上报会有连接超时异常，HealthMonitor会触发状态迁移至SERVICE_NOT_RESPONDING, 然后ANN上的ZKFC会退出选举，SNN上的ZKFC会获得- - - Active Lock, 作相应隔离后成为Active结点。
- ActiveNN JVM冻结：这个是JVM没崩溃，但也无法响应，同崩溃一样，会触发自动切换。
- ActiveNN 机器宕机：此时ActiveStandbyElector会失去同ZK的心跳，会话超时，SNN上的ZKFC会通知ZK删除ANN的活动锁，作相应隔离后完成主备切换。 ActiveNN 健康状态异常： 此时HealthMonitor会收到一个HealthCheckFailedException，并触发自动切换。
- Active ZKFC崩溃：虽然ZKFC是一个独立的进程，但因设计简单也容易出问题，一旦ZKFC进程挂掉，虽然此时NameNode是OK的，但系统也认为需要切换，此时SNN会发一个请求到ANN要求ANN放弃主结点位置，ANN收到请求后，会触发完成自动切换。
- ZooKeeper崩溃：如果ZK崩溃了，主备NN上的ZKFC都会感知断连，此时主备NN会进入一个NeutralMode模式，同时不改变主备NN的状态，继续发挥作用，只不过此时，如果ANN也故障了，那集群无法发挥Failover, 也就不可用了，所以对于此种场景，ZK一般是不允许挂掉到多台，至少要有N/2+1台保持服务才算是安全的。
HA机制总结
# 上面介绍了下关于HadoopHA机制，归纳起来主要是两块：元数据同步和主备选举。元数据同步依赖于QJM共享存储，主备选举依赖于ZKFC和Zookeeper。整个过程还是比较复杂的，如果能理解Paxos协议，那也能更好的理解这个。
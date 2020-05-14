# HDFS 分布式文件系统详解
## 1.HDFS 介绍
HDFS 是 Hadoop Distribute File System 的简称，意为：Hadoop 分布式文件系统。是 Hadoop 核心组件之一，作为最底层的分布式存储服务而存在。分布式文件系统解决的问题就是大数据存储，他们是横跨在多台机器上的存储系统。分布式文件系统在大数据时代有着广泛的应用前景，他们为存储和处理超大规模数据提供所需的扩展能力
##  2. HDFS 特性
- hdfs是一个分布式的文件系统，用于存储文件，通过统一的命名空间目录树来定位文件;
- 采用  master/slave（主从）架构。有一个 namenode 和多个 datanode 组成，各司其职;
- 分块存储，默认大小在Hadoop2.x版本中是128M；
- namenode 元数据管理，负责维护整个hdfs文件系统的目录树结构，以及每个文件所对应的 block 块信息（block 的 id，及所在的 datanode 服务器）。
- DataNode 数据存储  文件的 block 具体存储由 datanode承担，datanode 定时向 namenode 汇报自己持有的 block 信息
- 副本机制，为了容错，文件的 所有block 都会有副本
- HDFS 的设计为适应一次写入，多次读取，且不支持文件的修改。

## 3.  HDFS 架构
1、	NameNode是一个中心服务器，单一节点（简化系统的设计和实现），负责管理文件系统的名字空间（namespace）以及客户端对文件的访问
2、	文件操作，namenode是负责文件元数据的操作，datanode负责处理文件内容的读写请求，跟文件内容相关的数据流不经过Namenode，只询问它跟哪个dataNode联系，否则NameNode会成为系统的瓶颈
3、	副本存放在哪些Datanode上由NameNode来控制，根据全局情况作出块放置决定，读取文件时NameNode尽量让用户先读取最近的副本，降低读取网络开销和读取延时
4、	NameNode全权管理数据库的复制，它周期性的从集群中的每个DataNode接收心跳信合和状态报告，接收到心跳信号意味着DataNode节点工作正常，块状态报告包含了一个该DataNode上所有的数据列表

## 4. HDFS 中的FsImage以及edits和secondaryNamenode的作用
>  客户端对 hdfs 进行写文件时会首先被记录在 edits 文件中，edits 修改时元数据也会更新。每次 hdfs 更新时 edits 先更新后客户端
>  **fsimage**：NameNode 中关于元数据的镜像, 一般称为检查点, fsimage 存放了一份比较完整的 元数据信息
>  **edits**：edits 存放了客户端近一段时间的操作日志 ，客户端对 HDFS 进行写文件时会首先被记录在 edits 文件中 ，edits 修改时元数据也会更新 
>  一般开始时对namenode的操作都放在edits中，为什么不放在fsimage中呢？
>  因为fsimage是namenode的完整的镜像，内容很大，如果每次都加载到内存的话生成树状拓扑结构，这是非常耗内存和CPU。
>  fsimage内容包含了namenode管理下的所有datanode中文件及文件block及block所在的datanode的元数据信息。随着edits内容增大，就需要在一定时间点和fsimage合并。
>  **secondaryNamenode**：定期检查edits文件，一旦edits文件触发合并条件(时间长短  比如 1个小时，文件大小 ：64M)
1. Secondarynamenode 通知 NameNode 切换 edits文件
2. Secondarynamenode 从 NameNode 中获得 fsimage 和 edits文件
3. Secondarynamenode 将 fsimage载入内存，然后开始合并 edits 文件，合并之后成为新的 fsimage
4. Secondarynamenode 将新的 fsimage 发回给 NameNode
5. NameNode 用新的 fsimage替换旧的 fsimage
## 5.  HDFS 的副本机制和机架感知 
HDFS分布式文件系统的内部有一个副本存放策略：以默认的副本数=3为例：
1、第一个副本块存本机
2、第二个副本块存跟本机同机架内的其他服务器节点
3、第三个副本块存不同机架的一个服务器节点上
## 6. HDFS 读写过程
* ==hdfs 文件写入过程==
1. Client 发起文件上传请求，通过 RPC与 NameNode 建立通讯，NameNode 检查目标文件是否已经存在，父目录是否存在，返回是否可以上传
2. Client 请求 第一个 block 该传输到哪些 DataNode 服务器上
3. NameNode 根据配置文件中指定的备份数量及机架感知原理进行文件分配，返回可用的 DataNode 的地址如：A,B,C（Hadoop 在设计时考虑到数据的安全与高效，数据文件默认在HDFS上存放三份，存储策略为本地一份，同机架内其他某一节点上一份，不同机架的某一节点上存一份）
4. Client 请求 3 台 DataNode 中的一台 A 上传数据（本质上是一个rpc 调用，建立pipeline），A 收到请求会继续调用 B，然后 B 调用 C，将整个 pipeline 建立完成，后逐级返回 client
5. Client 开始往 A 上传第一个 block（先从磁盘读取数据放到一个本地内存缓存），以 packet 为单位（默认64k），A收到一个packet就会传给 B，B传给C，A 每传一个packet 会放入一个应答队列等待应答
6. 数据被分割成一个个 packet 数据包在 pipeline 上依次传输，在 pipeline 反方向上，逐个发送ack（命令正确应答），最终由pipeline中第一个 DataNode 节点 A 将 pipeline 发送给 Client
7. 当一个 block 传输完成之后，Client 再次请求 NameNode 上传第二个到服务器
* ==hdfs 文件读取过程==
8. Client向NameNode发起rpc请求，来确定请求文件block所在的位置;
9. NameNode 会视情况返回文件的部分或者全部block列表，对于每个block，NameNode 都会返回含有该 block 副本的DataNode地址；这些返回的DN地址，会按照集群拓扑机构得出 DataNode 与客户端的距离，然后进行排序，排序两个规则：网络拓扑结构中距离 Client近的排靠前；心跳机制中超时汇报的 DN 状态为 STATE,这样的排靠后；
10. Client选取排序靠前的 DataNode 来读取block，如果客户端本身就是 DataNode，那么将从本地直接获取数据（短路读取特性）
11. 底层上本质是建立 Socket Stream（FSDataInputStream），重复的调用父类DataInputStream 的 read 方法，直到这个块上的数据读取完毕；
12. 当读完列表的block后，若文件读取还没有结束，客户端会继续向NameNode 获取下一批的block列表；
13. 读取完一个block都会进行 checksum 验证，如果读取 DataNode 时出现错误，客户端会通知 NameNode ，然后再从下一个拥有block副本的Datanode继续读。
14. read 方法是并行的读取block信息，不是一块一块的读取；NameNode 只是返回 Client 请求包含块的DataNode地址，并不是返回请求块的数据；
15. 最终读取来所有的block 会合并成一个完整的最终文件。

---

# MapReduce 分布式计算框架概述

## 1. MapReduce 介绍

- 核心思想：分而治之
- Map 负责”分“，把复杂的任务分解为若干个”简单的任务“来并行处理。可以拆分的前提是这些小任务可以并行计算，彼此之间几乎没有任何依赖关系
- Reduce 负责”合“，即对 map 阶段的结果进行全局汇总
  这两个阶段合起来就是MapReduce 思想的体现。



## 2. MapReduce 的设计构思

- 面对海量的数据划分计算任务或数据以便同时进行计算。
- 构建抽象模型：Map和Reduce，为程序员提供了清晰的操作接口抽象描述
- 统一架构，隐藏系统层细节，只需要关心其应用层的具体计算问题，仅需编写少量的代码即可解决问题
  ​

## 3. MapReduce 框架架构

1. appMaster：负责整个程序的过程调度及状态协调
2. mapTask：负责 map 阶段的整个数据处理流程
3. reduceTask：负责 reduce 阶段的整个数据处理流程

## 4. MapReduce 的编程模型（天龙八部）

> 你能点进来说明我俩缘分到了，再加上我看你骨骼精奇，今日就将其传授于你

**map 阶段 2 个步骤**

1. 输入：设置 inputFormat 类，将我们的数据切分成 key，value 对，输入到第二步
2. map阶段（k1,v1-> k2,v2）：自定义 map 逻辑，处理我们第一步的输入数据，然后转换成新的 key，value 对进行输出

**shuffle 阶段4个步骤**

3. 分区：对输出的key，value对进行分区
4. 排序：对不同分区的数据按照相同key进行排序
5. 规约：对分组后的数据进行规约（combine 操作），降低数据的网络拷贝（可选步骤）
6. 分组：对排序后的数据进行分组，分组的过程中，将相同 key 的 value 放到一个集合当中

**reduce 阶段 2 个步骤**

7. reduce（k2,v2 -> k3,v3）：对多个 map 的任务进行合并，排序，写 reduce 函数自己的逻辑，对输入的 key，value 对进行处理，转换成新的key，value 对进行输出
8. 输出：设置 outputFormat 将输出的 key，value 对数据进行保存到文件中

# Yarn 资源调度框架

## 1. 介绍

yarn 是 Hadoop 集群当中的资源管理系统模块，自 Hadoop2.x 引入来管理集群中的资源（主要指的是各种硬件资源，包括 CPU，内存，磁盘，网络io等）以及运行在 yarn 上的各种任务  <!--主要职责：调度资源，管理任务等。-->

## 2.Yarn 中的各个主要组件的介绍

Resourcemanager：yarn 集群的主节点，主要用于接收客户端提交的任务，并对任务进行分配。

NodeManager：yarn 集群的从节点，主要用于任务的计算

ApplicationMaster：当有新的任务提交到 ResourceManager 的时候，ResourceManager 会在某个从节点 nodemanager 上启动一个ApplicationMaster进程，负责这个任务执行资源的分配，任务生命周期的监控等。

Container：资源分配的单位。

JobHistoryServer：这是 yarn 提供的一个查看已经完成的任务的历史日志记录服务，可以启动他来查看详细日志信息

## 3. Yarn 中各个主要组件的作用

resource Manager：

- 处理客户端请求
- 启动/监控 ApplicationMaster
- 监控 NodeManager
- 资源分配与调度

Nodemanager：

- 单个节点上的资源管理和任务管理
- 接收并处理来自 resourceManager 的命令
- 接收并处理来自 ApplicationMaster 的命令
- 管理抽象容器 container
- 定时向 RM 汇报本节点资源使用情况和各个 container 的运行状态

ApplicationMaster：

- 数据切分
- 为应用程序申请资源
- 任务监控与容错
- 负责协调来自 ResourceManager 的资源，开通监视容器的执行和资源使用

Container ：

- 对任务运行环境的抽象
- 任务运行资源
- 任务启动命令
- 任务运行环境

## 4. Yarn 架构

![1589461021824](C:\Users\86198\AppData\Local\Temp\1589461021824.png)

## 5. Yarn 调度器

1. FIFO Scheduler（队列调度器）
2. capacity scheduler（容量调度器，apache 默认使用的调度器）
3. Fair Scheduler（公平调度器，CDH 版本的Hadoop默认使用的调度器）
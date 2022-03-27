# Lecture 3 - GFS

![Lecture 3 - GFS](https://yulan-img-work.oss-cn-beijing.aliyuncs.com/img/202203270935644.png)

目录：

- [Lecture 3 - GFS](#lecture-3---gfs)
  - [分布式存储系统的难点](#分布式存储系统的难点)
  - [设计目标与特征](#设计目标与特征)
    - [目标](#目标)
    - [特征](#特征)
  - [Master 节点](#master-节点)
    - [架构](#架构)
    - [保存的数据内容](#保存的数据内容)
  - [读文件](#读文件)
  - [写文件](#写文件)
  - [一致性](#一致性)

## 分布式存储系统的难点

人们设计大型分布式系统或大型存储系统的出发点通常是追求更好的性能，但最后出于一致性的考虑，又会付出相应的性能代价。

* 高性能 -> 分片，将数据分割放到大量的服务器上，这样就可以并行的从多台服务器读取数据。
* 分片 -> 故障，服务器数量过多后故障会常态化。
* 故障 -> 容错，由于故障常态化，需要一个自动的容错系统。
* 容错 -> 复制，实现容错的最有用的一个方法是复制，即维护多个副本，其中一个故障后，可以切换至另外一个。
* 复制 -> 一致性，拥有多个副本后，需要考虑到多个副本间的数据一致性。
* 一致性 -> 低性能，为了实现一致性，需要额外的网络交互，但这样的交互又会降低性能。

## 设计目标与特征

### 目标

* 构建大型的，快速的文件系统（Big、Fast）
  * Google 有大量的数据，需要大量的磁盘来存储这些数据，同时需要快速的并行访问这些海量数据。
* 全局有效（Global）
  * 不用针对某个特定的应用程序构建特定的存储系统，而是全局通用，各个应用程序使用同一个存储系统。
* 文件分割（Sharding）
  * 含了数据的文件会被 GFS 自动的分割并存放在多个服务器之上，这样读写操作自然就会变得很快。因为可以从多个服务器上同时读取同一个文件，进而获得更高的聚合吞吐量。
  * 可以存储比单个磁盘还要大的文件。
* 自动修复（Automatic recovery）

### 特征

* 单数据中心（Single data center）
* Google 内部使用（Internal use）
* 大型的顺序文件读写（Big sequential access）
  * GFS是为 TB 级别的文件而生。
  * GFS 只会顺序处理，不支持随机访问。
  * GFS 并没有花费过多的精力来降低延迟，它的关注点在于巨大的吞吐量上，所以单次操作都涉及到 MB 级别的数据。

## Master 节点

### 架构

* 单 Master
  * 管理文件和 Chunk 的信息
* 多 Chunk
  * 存储实际的数据
* 多 客户端

![GFS 架构](https://yulan-img-work.oss-cn-beijing.aliyuncs.com/img/202203261026394.png)

### 保存的数据内容

* 文件名到 Chunk ID 或者 Chunk Handle 数组的映射。（NV，非易失，需要写入磁盘）
* Chunk ID 到 Chunk 数据的映射，该映射又包括：
  * 每个 Chunk 存储在哪些服务器上，即 Chunk 服务器的列表；（V，易失，不用写入磁盘，Master 重启后通过轮询 Chunk 获取）
  * 每个 Chunk 当前的版本号；（NV）
  * 对于 Chunk 的写操作都必须在主 Chunk（Primary Chunk）上顺序处理，主 Chunk 是 Chunk 的多个副本之一。所以，Master 节点必须记住哪个 Chunk 服务器持有主 Chunk。（V）
  * 并且，主 Chunk 只能在特定的租约时间内担任主 Chunk，所以，Master节点要记住主 Chunk 的租约过期时间。（V）

## 读文件

1. Client 发送 文件名 和 偏移量 给 Master；

2. Master 返回 Chunk Handle 和 服务器列表 给 Client；

    * 通过偏移量除以 64MB 就可以从数组中得到对应的 Chunk ID。
    * 再从 Chunk 表单中获取到存有 Chunk 的服务器列表。

3. Client 发送 Chunk Handle 和 偏移量 给 网络上最近的 Chunk 服务器；

    * 客户端可能会连续多次读取同一个 Chunk 的不同位置。所以，客户端会缓存Chunk和服务器的对应关系

4. Chunk 服务器 返回 数据 给 Client。

    * Chunk 服务器会在本地的硬盘上，将每个Chunk存储成独立的 Linux 文件，并通过 Chunk Handle，找到对应的文件。

## 写文件

1. Master 选择 Primary Chunk 服务器，并告知 Client

    * 如果没有指定 Primary Chunk 服务器，Master 会首先找出所有存有 Chunk 最新副本的 Chunk 服务器（与 Master 中记录的版本号一致），然后选择其中一个作为 Primary，其他作为 Secondary。
    * 之后 Master 会增加版本号，并将版本号写入磁盘，这样就算故障了也不会丢失这个数据。
    * Master 节点会向 Primary 和 Secondary 副本对应的服务器发送消息并告诉它们，谁是 Primary，谁是 Secondary，Chunk 的新版本是什么。Primary 和 Secondary 服务器都会将版本号存储在本地的磁盘中。这样，当它们因为电源故障或者其他原因重启时，它们可以向 Master 报告本地保存的 Chunk 的实际版本号。

2. Client 发送 数据 给 Primary 和 Chunk 服务器
3. Primary 和 Chunk 服务器 通知 Client 已收到数据
4. Client 通知 Primary 开始追加数据
5. Primary 将 Client 要追加的数据写入 Chunk 的末尾。同时通知所有的 Secondary 服务器也将 Client 要追加的数据写入自己存储的 Chunk末尾
6. Primary 仅当所有 Secondary 服务器均返回成功后，返回 Client 成功，否则失败
7. 如果 Client 从 Primary 收到写入失败，那么客户端应该重新发起整个追加过程

## 一致性

GFS 不是强一致性，GFS 允许多副本不一致（出于简化设计和性能的考虑）。

GFS 可能会返回错误或者过时的数据，需要应用程序对这类错误的有一定的接受能力，如：搜索引擎，搜索出大量的结果，但其中丢失了一条或者排序结果错误，网络上的投票数量。

如果应用程序不能容忍乱序，应用程序可以：

* 通过在文件中写入序列号，这样读取的时候能自己识别顺序；
* 对于特定的文件不要并发写入。例如，对于电影文件。

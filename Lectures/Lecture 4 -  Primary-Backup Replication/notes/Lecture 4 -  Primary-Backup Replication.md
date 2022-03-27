# Lecture 4 -  Primary-Backup Replication

![Lecture 4 -  Primary-Backup Replication](https://yulan-img-work.oss-cn-beijing.aliyuncs.com/img/202203271634816.png)

目录：

- [Lecture 4 -  Primary-Backup Replication](#lecture-4----primary-backup-replication)
  - [复制与容错](#复制与容错)
  - [复制的方法](#复制的方法)
    - [状态转移（State Transfer）](#状态转移state-transfer)
    - [复制状态机（Replicated State Machine）](#复制状态机replicated-state-machine)
  - [VMware FT 特点](#vmware-ft-特点)
  - [VMware FT 工作原理](#vmware-ft-工作原理)
  - [非确定性事件](#非确定性事件)
  - [异常处理](#异常处理)
    - [输出控制](#输出控制)
    - [重复输出](#重复输出)
  - [Test-and-Set 服务](#test-and-set-服务)

## 复制与容错
		
容错本身是为了提供高可用性，而复制则是实现容错的一个工具。

> 复制：维护一个服务器的多个副本，并让它们保持同步。

复制能处理单台计算机的 fail-stop 故障，但不能处理软件中的 bug 和硬件设计中的缺陷。

> fail-stop 故障：指若出现故障，仅会停止运行，而不是运算出错误结果，如：机器断电、网络中断等。

## 复制的方法

### 状态转移（State Transfer）

Primary 将自己完整状态，比如说内存中的内容，拷贝并发送给 Backup。Backup 会保存收到的最近一次状态，并拥有所有的数据。

### 复制状态机（Replicated State Machine）

Primary 只会将外部事件，例如外部的输入，发送给 Backup。

基于的前提：如果有两台计算机，如果它们从相同的状态开始，并且它们以相同的顺序，在相同的时间，看到了相同的输入，那么它们会一直互为副本，并且一直保持一致。

**复制状态机优缺点：**

* 优点：相较于状态转移，数据量通常较小。
* 缺点：更复杂，并且对于计算机的运行做了更多的假设。

## VMware FT 特点

VMware FT 的特点是从机器级别实现复制。

**机器级复制：** 它会复制机器的完整状态，这包括了所有的内存，所有的寄存器，可以在 VMware FT 管理的机器上运行任何软件，只要该软件可以运行在 VMware FT 支持的微处理器上。

* 优点：可以将任何现有的软件（需支持相应 CPU 架构），运行在 VMware FT 的这套方案上，使其具备容错性。
* 缺点：不够高效。

**应用级复制：** 如：GFS，仅复制 Chunk 等相关内容，不会复制其他东西，现有的大多数系统，都是采用的这种应用程序级别的状态复制。

* 优点：粒度更精细，更加高效。
* 缺点：复制这个行为，必须构建在应用程序内部。即应用程序本身需要参与并实现复制的这个过程。

## VMware FT 工作原理

需要两个物理服务器分别运行 Primary 和 Backup 虚拟机。这两个物理服务器上的 VMM 会为每个虚拟机分配一段内存，在 VMware FT 里，多副本服务没有使用本地盘，而是使用了一些 Disk Server（远程盘）。

目标：使 Primary 和 Backup 的内存镜像完全一致。

**正常流程：**

1. Client 发送请求（网络数据包形式） 给 Primary
2. 网络数据包会产生一个中断，中断被送到了 VMM，VMM 收到请求后：
   * 在虚拟机的 guest 操作系统中，模拟网络数据包到达的中断，将相应的数据送给应用程序的 Primary 副本。
   * 同时 VMM 会将网络数据拷贝一份，通过网络送给 Backup 虚拟机所在的 VMM，Backup 虚拟机收到后，也会执行同 Primary 虚拟机一样的操作（模拟、发送给应用程序）。
   * Primary 和 Backup 拥有了相同的输入，并进行相同的处理，最终保持同步。
   * 虚机内的服务会回复客户端的请求。在 Primary 虚机里面，服务会生成一个回复报文，并通过 VMM 在虚机内模拟的虚拟网卡发出。之后 VMM 可以看到这个报文，它会实际的将这个报文发送给客户端。
   * Backup 虚机运行了相同顺序的指令，它也会生成一个回复报文给客户端，并将这个报文通过它的 VMM 模拟出来的虚拟网卡发出。但是它的 VMM 知道这是 Backup 虚机，会丢弃这里的回复报文。

Primary 发往 Backup 之间同步的数据流的通道称之为 Log Channel。
Primary 发往 Backup 的事件被称为 Log Channel 上的 Log Event/Entry。

**FT（Fault-Tolerance）流程：**

* Backup 每秒会收到很多条 Log，其中就包括 Primary 的定时器中断，一旦 Backup 超过一定时间没有收到消息，便会认为 Primary 出现故障，Backup 将会上线（Go Alive）
* Backup 不会再等待来自于 Primary 的 Log Channel 的事件，Backup 的VMM 会让 Backup 自由执行，而不是受来自于 Primary 的事件驱动。
* Backup 的 VMM 会在网络中做一些处理（猜测是发 GARP），让后续的客户端请求发往 Backup 虚机，而不是 Primary 虚机。同时，Backup 的 VMM 不再丢弃 Backup 虚机的输出。
* 自此，Backup 虚机接管了整个服务，成为了 Primary。

## 非确定性事件

* 客户端输入
  * 提示网络数据包送达了的中断，对于 Primary 和 Backup 而言，最好要在相同的时间，相同的位置触发，否则执行过程就是不一样的，进而会导致它们的状态产生偏差。
  * 解决方法 - Bounce Buffer 机制：
    * 物理服务器的网卡会将网络数据包拷贝给 VMM 的内存，而不是直接拷贝到 Primary 虚拟机中，以防止失去对 Primary 虚机的时许控制（因为无法预知 Primary 会什么时候收到网络数据包）。
    * 然后，VMM 会暂停 Primary 虚机，记住当前的指令序号，将整个网络数据包拷贝给 Primary 虚机的内存，之后模拟一个网卡中断发送给 Primary 虚机。
    * 同时，Primary 的 VMM 会将网络数据包和指令序号发送给 Backup。Backup 虚机的 VMM 也会在对应的指令序号暂停 Backup 虚机，将网络数据包拷贝给 Backup 虚机，之后在相同的指令序号位置模拟一个网卡中断发送给 Backup 虚机。
* 怪异指令
  * 随机数生成器
  * 获取当前时间的指令
  * 获取计算机的唯一 ID
  * 解决方法：
    * VMware FT 的设计者认为他们找到了所有类似的怪异指令，Backup 的 VMM 会探测到每一个这样的指令，拦截并且不执行它们。
    * VMM 会让 Backup 虚机等待来自 Log Channel 的有关这些指令的指示，比如随机数生成器这样的指令，之后 VMM 会将 Primary 生成的随机数发送给 Backup。
* 多 CPU 的并发
  * 指令在不同的CPU上会交织在一起运行，进而产生的指令顺序是不可预期的。（例如：两个核同时向同一份数据请求锁，但 Primary 和 Backup 不同的线程获得了锁）
  * VMware FT 并没有讨论它

## 异常处理

### 输出控制

> Primary 收到客户端修改请求后，修改自身数据，并返回给客户端成功，但发送给 Backup 的 Log 条目丢包了，导致 Backup 虚机因为没有看到客户端请求。在后续 Backup 接管服务后，会出现数据前后不一致。

解决方法：输出控制。

核心思想：确保在 Client 看到对于请求的响应时，Backup 虚机一定也看到了对应的请求，即，Primary 的 VMM 会等到之前的 Log 条目都被 Backup 虚机确认收到了才将输出转发给 Client。

由于这里的同步等待使得 Primary 不能超前 Backup 太多，在某个时间点，Primary 必须要停下来等待 Backup，这对性能影响十分巨大。

### 重复输出

> 回复报文已经从 VMM 发往客户端了，并且客户端收到了回复，但是这时 Primary 虚机崩溃了。而在 Backup 侧，客户端请求还堆积在 Backup 对应的 VMM 的 Log 等待缓冲区，也就是说客户端请求还没有真正发送到 Backup 虚机中。当 Primary 崩溃之后，Backup 接管服务，Backup 首先需要消费所有在等待缓冲区中的 Log，以保持与 Primay 在相同的状态，这样 Backup 才能以与 Primary 相同的状态接管服务。假设最后一条 Log 条目对应来自客户端的请求，那么 Backup 会在处理完客户端请求对应的中断之后，再上线接管服务。这意味着，Backup 会生成一个重复的输出报文（同之前 Primary 生成的）。因为这时，Backup 已经上线接管服务，它生成的输出报文会被它的 VMM 发往客户端。这样客户端会收到两个重复的报文。

解决方法：由于客户端通过 TCP 与服务器进行交互，Backup 的状态与 Primary 完全相同，它所生成的回复报文与之前 Primary 生成报文的 TCP 序列号是一样的，这样客户端的 TCP 栈会发现并丢弃这个重复的报文，不会暴露给用户层。

## Test-and-Set 服务

作为仲裁官，决定了两个副本中哪一个应该上线，避免脑裂问题（即，对于同一个服务，Primary 和 Backup 同时在线并与客户端通信）。

Test-and-Set 服务需要运行在独立于 Primary 和 Backup 的物理服务器上。同时，基于容错考虑 Test-and-Set 服务，也应该是复制的。

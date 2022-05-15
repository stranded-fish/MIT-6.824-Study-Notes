# MIT 6.824 - Lab 3: Fault-tolerant Key/Value Service

实验地址：<http://nil.csail.mit.edu/6.824/2021/labs/lab-kvraft.html>

目录：

- [MIT 6.824 - Lab 3: Fault-tolerant Key/Value Service](#mit-6824---lab-3-fault-tolerant-keyvalue-service)
  - [实验内容](#实验内容)
  - [实验思路](#实验思路)
    - [强一致性实现](#强一致性实现)
    - [系统交互流程](#系统交互流程)
    - [client 核心代码](#client-核心代码)
    - [server 核心代码](#server-核心代码)
  - [实验结果](#实验结果)
  - [参考链接](#参考链接)

## 实验内容

本实验主要基于 Lab 2 中实现的 Raft 库构建一个可以容错的 key/value 存储系统。该存储系统将由多个使用 Raft 进行复制的 kv server 组成。只要集群中大多数服务器处于活动状态并且可以通信，该存储系统就应该能正确处理客户端请求。

该 key/value 存储系统需要支持三种操作：

* `Put(key, value)`
  * 替换 key 的值为 value
* `Append(key, arg)`
  * 向 key 键值追加 arg，没有则新建
* `Get(key)`
  * 获取 key 的值，如果 key 不存在则返回空字符串

实验要求对客户端 `Get/Put/Append` 方法的调用提供强一致性，即整个系统在客户端看来表现得像是只有一个副本，每次读写都能感知到系统之前的最新修改。

## 实验思路

### 强一致性实现

**针对写操作：**

因为 Raft 的写操作只能是交由 Leader 处理，并且需要在整个集群中达成共识（实现多数复制），故可以保证不会出现写冲突。

但仍需要明确客户端写操作的幂等性无法保证线性化语义，举例：假设有两个客户端 A、B，先后向一个 kv server 发起 `Put` 请求，A 先执行 `Put(x, 1)`，B 后执行 `Put(x, 2)`，但是 A 收到的响应超时了，B 正常收到了响应，当 B 又发起 `Get(x)` 请求时，A 因为超时又重新发起了 `Put(x, 1)`，最终导致 B 收到了 `x = 1`，不符合线性语义。

解决方法：

Raft 的强一致性实现，在其 [论文](https://raft.github.io/raft.pdf) 中的第 8 章，已经提出了相关解决方法：

> The solution is for clients to assign unique serial numbers to every command. Then, the state machine tracks the latest serial number processed for each client, along with the associated response. If it receives a command whose serial number has already been executed, it responds immediately without re-executing the request.

即客户端对于每一条指令都赋予一个唯一的序列号。然后，状态机跟踪每条指令最新的序列号和相应的响应。如果接收到一条指令，它的序列号已经被执行了，那么就立即返回结果，而不重新执行指令。

**针对读操作：**

客户端读操作，如果 server 层不进行任何处理，可能会有读取到脏数据的风险，举例：客户端发送 `Get(x)` 请求到一个 server，但该 server 还不知道集群中已经选出了新的 Leader，仍然认为自己是 Leader 于是响应客户端请求。

解决方法：

同样在 Raft 论文中的第 8 章，介绍了这个问题的解决方法：

> Raft handles this by having the leader exchange heartbeat messages with a majority of the cluster before responding to read-only requests. Alternatively, the leader could rely on the heartbeat mechanism to provide a form of lease [9], but this would rely on timing for safety (it assumes bounded clock skew).

* 方法一：让 Leader 在响应客户端读请求前，先与集群中的大多数节点交换一次心跳，以确认自己的 Leader 身份。
* 方法二：Leader 可以依赖心跳机制来实现一种租约，但该方法依赖时间保证安全性。

### 系统交互流程

整个系统分为：Client、KV server、Raft 三层，Client 向 KV server 发起请求，然后 KV server 通过其绑定的 Raft 实现日志复制与一致性，然后再让状态机执行，执行完毕后响应客户端。其各个模块交互图如下所示（来自 [Lab 3 官方文档](http://nil.csail.mit.edu/6.824/2021/labs/lab-kvraft.html)）：

![系统交互图](https://yulan-img-work.oss-cn-beijing.aliyuncs.com/img/202205142047381.png)

**Put/Append 请求流程：**

![Put/Append 流程 - 函数调用图](https://yulan-img-work.oss-cn-beijing.aliyuncs.com/img/202205142335402.png)

* client Put/Append 统一调用 `PutAppend` 方法，在正式发起 RPC 请求前 client 会自增自己的请求标识 id，然后附带上自身唯一的标识符，以唯一标识该请求。
* client 发起 RPC，调用 KV server 的 `PutAppend` 方法，收到 RPC 请求的 KV server 会首先检查自己的活性以及是否为 Leader。
  * 如果否，则返回 client false 以及最新心跳所获取到 leader id。
  * 如果是，则构造 operation 并调用 Raft `start` 方法进行复制。同时使用该 Raft `start` 方法返回的 log index 作为 key 值映射对应的 `channel`，之后 `PutAppend` 等待该 `channel` 解除阻塞或是直到超时。
* Raft 收到来自 KV server 的请求后，执行日志复制流程，并在 commit 后，`*Cond.Broadcast()` 唤醒后台协程 `FSMDoCommit` 将已经 commit 但还未 apply 的 command 发送到 `apply channel` 以通知应用层 KV server。
* KV server 的后台协程 `onApply` 会持续监听 `apply channel`，当监测到新的 apply 消息后，会根据消息内容，执行 apply 或者 安装快照。
  * 如果是 安装快照 则解析快照信息，替换自己维护的状态机（KV DB）。
  * 如果是 apply 则进一步执行 `doApply` 解析，并调用状态机的 `doPut` 或 `doAppend` 执行实际的任务。
    * 执行完成后，检查当前日志大小，判断是否需要生成快照，如果是则进一步调用 Raft 层的 `Snapshot` 接口生成快照。
    * 最终调用 `done` 方法，表示状态机处理完成，将消息写入 `channel` 通知 KV server 的 `PutAppend` 请求。
    * KV server 解除阻塞后，会检查该请求是否匹配（根据 client 发起请求时的唯一标识 client id 和 request id），如果匹配，则返回 client OK，否则返回 error。
* 客户端在收到 KV server 响应 false 或者请求超时后，会不断重试，直至成功。注意：重试时不会更改 request id，以避免状态机重复 Apply。

**Get 请求流程：**

本实验的 `Get` 请求实现出于强一致性的考虑，也将其作为了一条日志交由 Raft 进行复制（有待优化，参考 [SOFAJRaft 实现](https://www.sofastack.tech/blog/sofa-jraft-linear-consistent-read-implementation/)），故其整体流程与 `Put/Append` 基本相同。

### client 核心代码

**1. `Clerk` 结构体定义**

```go
type Clerk struct {
    servers        []*labrpc.ClientEnd // kv server 集群信息
    clientId       int64               // 客户端 id
    requestId      int                 // 请求 id - 单调递增
    latestLeaderId int                 // 最新 leader id
}
```

**2. `PutAppend`**

```go
func (ck *Clerk) PutAppend(key string, value string, operation string) {
    // 请求 id + 1，将该请求 id 与 client id 结合以唯一标识该请求
    // 从而避免 kv server 重复 apply 相同请求
    ck.requestId++
    requestId := ck.requestId
    server := ck.latestLeaderId

    // 不断尝试直至成功
    for {
        request := PutAppendRequest{Key: key, Value: value, Operation: operation, ClientId: ck.clientId, RequestId: requestId}
        response := PutAppendResponse{}
        ok := ck.servers[server].Call("KVServer.PutAppend", &request, &response)
        if ok && response.msg == OK {
            // 记录当前 Leader
            ck.latestLeaderId = server
            return
        } else if response.msg == ErrWrongLeader {
            // 切换指定 Leader 重试
            server = response.LeaderId 
        } else {
            // 无法确定当前 Leader，遍历所有 server 重试
            server = (server + 1) % len(ck.servers)
        }
    }
}
```

**3. `Get`**

```go
func (ck *Clerk) Get(key string) string {
    // 请求 id + 1，将该请求 id 与 client id 结合以唯一标识该请求
    // 从而避免 kv server 重复 apply 相同请求
    ck.requestId++
    requestId := ck.requestId
    server := ck.latestLeaderId

    // 不断尝试直至成功
    for {
        request := GetRequest{Key: key, ClientId: ck.clientId, RequestId: requestId}
        response := GetResponse{}
        
        // 出于线性一致读的要求，读请求同样走 Raft 协议（有待优化）
        ok := ck.servers[server].Call("KVServer.Get", &request, &response)
        if ok && response.msg == OK {
            // 记录当前 Leader
            ck.latestLeaderId = server
            return response.value
        } else if response.msg == NoKey {
            return ""
        } else if response.msg == ErrWrongLeader {
            // 切换指定 Leader 重试
            server = response.LeaderId 
        } else {
            // 无法确定当前 Leader，遍历所有 server 重试
            server = (server + 1) % len(ck.servers)
        }
    }
}
```

### server 核心代码

**1. `doApply` 状态机进行 apply**

```go
func (kv *KVServer) doApply(message raft.ApplyMsg) {
    op := message.Command.(Op)
    if message.CommandIndex <= kv.lastIncludeIndex {
        return
    }

    // 判断当前请求是否已经 apply 过，若是则跳过，避免重复执行
    if !kv.isDone(op.ClientId, op.RequestId) {
        if op.Operation == "put" {
            kv.doPut(op)
        } else if op.Operation == "append" {
            kv.doAppend(op)
        }
    }
    
    // 判断是否需要生成快照
    if kv.maxraftstate != -1 {
        kv.needGenerateSnapshot(message.CommandIndex)
    }

    kv.done(op, message.CommandIndex)
}
```

**2. `doPut` 状态机执行 Put 操作**

```go
func (kv *KVServer) doPut(op Op) {
    kv.mu.Lock()
    defer kv.mu.Unlock()
    kv.kvDB[op.Key] = op.Value

    // 记录当前 client apply 的最新的 request id 以避免重复 apply
    kv.lastRequestId[op.ClientId] = op.RequestId
}
```

**3. `doAppend` 状态机执行 Append 操作**

```go
func (kv *KVServer) doAppend(op Op) {
    kv.mu.Lock()
    defer kv.mu.Unlock()
    value, exist := kv.kvDB[op.Key]
    if exist {
        kv.kvDB[op.Key] = value + op.Value
    } else {
        kv.kvDB[op.Key] = op.Value
    }

    // 记录当前 client apply 的最新的 request id 以避免重复 apply
    kv.lastRequestId[op.ClientId] = op.RequestId
}
```

**4. `done` 状态机执行完毕，通知 KV server**

```go
func (kv *KVServer) done(op Op, logIndex int) bool {
    kv.mu.Lock()
    defer kv.mu.Unlock()

    // 根据 logIndex 获取到指定 channel 并写入消息解除其阻塞
    ch, exist := kv.waitApplyCh[logIndex]
    if exist {
        ch <- op
    }
    return exist
}
```

## 实验结果

完成所有实验后，可通过 `go test -race` 命令，同时测试 Lab 3A - 3B，测试结果如下：

![实验结果](https://yulan-img-work.oss-cn-beijing.aliyuncs.com/img/202205142152785.png)

## 参考链接

* [6.824 Lab 3: Fault-tolerant Key/Value Service](http://nil.csail.mit.edu/6.824/2021/labs/lab-kvraft.html)
* [In Search of an Understandable Consensus Algorithm (Extended Version)](https://raft.github.io/raft.pdf)
* [OneSizeFitsQuorum/MIT6.824-2021-lab3.md](https://github.com/OneSizeFitsQuorum/MIT6.824-2021/blob/master/docs/lab3.md)
* [raft在处理用户请求超时的时候，如何避免重试的请求被多次应用？](https://www.zhihu.com/question/278551592)
* [braft-docs](https://github.com/kasshu/braft-docs)
* [baidu/braft](https://github.com/baidu/braft)

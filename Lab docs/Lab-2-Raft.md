# MIT 6.824 - Lab 2: Raft

实验地址：<http://nil.csail.mit.edu/6.824/2021/labs/lab-raft.html>

目录：

- [MIT 6.824 - Lab 2: Raft](#mit-6824---lab-2-raft)
  - [实验内容](#实验内容)
  - [实验思路](#实验思路)
    - [数据结构](#数据结构)
    - [节点启动流程](#节点启动流程)
    - [Part 2A - Leader 选举](#part-2a---leader-选举)
    - [Part 2B - 日志复制](#part-2b---日志复制)
    - [Part 2C - 持久化](#part-2c---持久化)
    - [Part 2D - 日志压缩](#part-2d---日志压缩)
  - [实验结果](#实验结果)
  - [参考链接](#参考链接)

## 实验内容

本实验将实现 Raft 共识协议，实验整体大致分为了四个模块：

* Part 2A: leader election
  * Leader 选举。实现 Raft leader 选举和心跳检测功能。
* Part 2B: log
  * 日志复制。实现 Leader 与 Follower append entry 追加日志功能。
* Part 2C: persistence
  * 持久化。实现 Raft 持久状态落盘。
* Part 2D: log compaction
  * 日志压缩。实现日志的快照压缩、安装快照等功能。

## 实验思路

### 数据结构

TODO Raft 节点数据结构定义说明

### 节点启动流程

Raft 应用层通过 `Make()` 接口新建 Raft server。其函数接口定义如下：

```go
func Make(peers []*labrpc.ClientEnd, me int,
	persister *Persister, applyCh chan ApplyMsg) *Raft {...}
```

* peers: 包含 Raft 集群的全部端口号
* me: 该节点端口号
* persister: 用于数据持久化
* applyCh: 用于 Raft 通知应用层进行 apply

整个节点启动流程如下：

* 初始化 Raft Node 结构体数据
* 通过 persister 读取之前持久化的数据
* 随机化选举超时时间，并启动选举超时计时器
* 启动 apply 监听线程，用于监听状态机 commit 时机，并通知上层应用 apply。

### Part 2A - Leader 选举

TODO 函数调用图

* `electionTicker`：
  * Raft 节点会在后台，运行选举超时计时器，当该计时器超时后，将会调用 `handleElectionTimeout` 方法，触发选举流程。
* `handleElectionTimeout`：
  * 触发选举超时的节点，将首先判断自己非 leader，然后将自己状态变为 candidate，增加自身 term 同时投票给自己，然后持久化该元数据。
  * 然后将自身的 term、id、lastLogIndex、lastLogTerm 封装到 request 中，并发起 RPC。

```go
// 处理 election timeout 选举超时 - 发起选举
func (rf *Raft) handleElectionTimeout() {
	rf.mu.Lock()
	defer rf.mu.Unlock()
	if rf.currentState != StateLeader {
		rf.currentState = StateCandidate // 变为 candidate 状态
		rf.currentTerm++                 // 自身 term + 1
		rf.votedFor = rf.me              // 投票给自己
		rf.voteCounts = 1                // 计票
		rf.persist()
		DPrintf("[%d]---In %d term try to elect", rf.me, rf.currentTerm)
		request := RequestVoteRequest{
			Term:         rf.currentTerm,
			CandidateId:  rf.me,
			LastLogIndex: rf.lastLogIndex(),
			LastLogTerm:  rf.lastLogTerm(),
		}
		for server := range rf.peers {
			if server == rf.me {
				continue
			}
			go func(server int, currentTerm int, request RequestVoteRequest) {
				var response RequestVoteResponse
				ok := rf.sendRequestVote(server, &request, &response)
				if ok {
					rf.HandleRequestVoteResponse(currentTerm, response)
				}
			}(server, rf.currentTerm, request)
		}
	}
}
```

* `HandleRequestVoteRequest`
  * 其他节点在收到来自 candidate 的请求投票请求后，会首先进行以下检查：
    * 判断 request 中的 term 是否 < 自身 term
    * 判断收到 request 中的 term == 自身 term，但该 term 已经投过票了（一个 term 只能投一次票）
    * case 3.1 收到 request 中的 term > 自身 term，执行 step_down
    * case 3.2 收到 request 中的日志 >= （不落后）于自身的日志，且自身还未投票，投票给这个节点，并更新自己的选举计时器，同时持久化状态。

```go
// HandleRequestVoteRequest 处理来自 candidate 的拉票请求
func (rf *Raft) HandleRequestVoteRequest(request *RequestVoteRequest, response *RequestVoteResponse) {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	DPrintf("[%d]---Handle RequestVote, CandidatesId[%d] Term%d CurrentTerm%d LastLogIndex%d LastLogTerm%d votedFor[%d]",
		rf.me, request.CandidateId, request.Term, rf.currentTerm, request.LastLogIndex, request.LastLogTerm, rf.votedFor)
	defer DPrintf("[%d]---Return RequestVote, CandidatesId[%d] VoteGranted %v ", rf.me, request.CandidateId, response.VoteGranted)

	// case 1. 收到 request 中的 term <  自身 term
	if request.Term < rf.currentTerm {
		return
	}

	// case 2. 收到 request 中的 term == 自身 term，但该 term 已经投过票了（一个 term 只能投一次票）
	if request.Term == rf.currentTerm && rf.votedFor != -1 && rf.votedFor != request.CandidateId {
		return
	}

	// case 3.1 收到 request 中的 term > 自身 term
	if request.Term > rf.currentTerm {
		rf.stepDown(request.Term)
	}

	// case 3.2 收到 request 中的日志 >= （不落后）于自身的日志，且自身还未投票
	logIsOk := request.LastLogTerm > rf.lastLogTerm() || (request.LastLogTerm == rf.lastLogTerm() && request.LastLogIndex >= rf.lastLogIndex())
	if logIsOk && rf.votedFor == -1 {
		rf.votedFor = request.CandidateId
		rf.persist()
		rf.resetElectionTimer()
	}

	response.Term = rf.currentTerm
	response.VoteGranted = rf.currentTerm == request.Term && rf.votedFor == request.CandidateId
}
```

* `stepDown`

```go
// 将节点状态回退为 follower，并更新相关状态
func (rf *Raft) stepDown(term int) {
	rf.currentState = StateFollower
	rf.currentTerm = term
	rf.votedFor = -1
	rf.persist()
	rf.resetElectionTimer()
}
```

* `HandleRequestVoteResponse`
  * candidate 在收到用户请求后，需要进行以下检查：
    * 判断自身状态是否仍是 candidate
    * 确保响应未过期
    * 收到 response 中的 term > 自身 term - 进行状态回退
    * 统计选票，并判断是否获得超过半数选票

```go
func (rf *Raft) HandleRequestVoteResponse(term int, response RequestVoteResponse) {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	// 确保当前节点状态为 candidate
	// 可能情况：当前节点已经当选成功，节点成为了 leader 或 回退为 follower
	if rf.currentState != StateCandidate {
		return
	}

	// 确保 RequestVoteResponse 未过期
	// 可能情况：网络拥塞，收到了旧任期时的 Response
	if term != rf.currentTerm {
		return
	}

	// 收到 response 中的 term > 自身 term - 进行状态回退
	if response.Term > rf.currentTerm {
		rf.stepDown(response.Term)
		return
	}

	// 统计选票
	if response.VoteGranted {
		rf.voteCounts++
		if rf.voteCounts >= len(rf.peers)/2+1 {
			rf.becomeLeader()
		}
	}
}
```

* `becomeLeader`
  * 再判断获得超过半数选票之后，当选 leader，当选 leader 后首先重置计时器，然后广播通知其他节点。

### Part 2B - 日志复制

TODO 函数调用图

TODO 关键代码

* `boardCastEntries`
  * 遍历所有节点，检查是否...
* `HandleAppendEntriesRequest`
  * follower 处理 append entries RPC 请求

```go
// HandleAppendEntriesRequest follower 处理 append entries RPC 请求
func (rf *Raft) HandleAppendEntriesRequest(request *AppendEntriesRequest, response *AppendEntriesResponse) {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	// request.Term < 自身 term - 直接忽略
	if request.Term < rf.currentTerm {
		response.Term = rf.currentTerm
		response.Success = false
		return
	}

	// 判断是否需要 step down
	rf.checkStepDown(request.Term, request.LeaderId)

	// 出现脑裂错误
	if rf.votedFor != request.LeaderId {
		// 通过使自身 term + 1 让发生脑裂的 leader step down，以最小化脑裂损失
		rf.stepDown(request.Term + 1)
		response.Success = false
		response.Term = request.Term + 1
		return
	}

	// 日志不匹配
	if request.PrevLogIndex > rf.lastLogIndex() || // follower 日志落后于 leader 且存在 gap
		request.PrevLogIndex < rf.lastSSPointIndex || // 当前 index 涉及已执行快照部分
		(request.PrevLogTerm != 0 && request.PrevLogTerm != rf.getLogTermWithIndex(request.PrevLogIndex)) {
		// 告知 leader 自身 commit index，请求 leader 以该 index 为起点发送
		response.FollowerCommitedIndex = rf.commitIndex
		response.Term = rf.currentTerm
		response.Success = false
		rf.resetElectionTimer()
		return
	}

	// 判断是否为心跳
	if len(request.Entries) == 0 {
		rf.advanceFollowerCommittedIndex(request.LeaderCommit)
		response.FollowerCommitedIndex = rf.commitIndex
		response.Term = rf.currentTerm
		response.Success = true
		rf.resetElectionTimer()
		return
	}

	// 检查日志冲突并解决
	if !rf.checkAndResolveConflict(request) {
		response.Success = false
		response.Term = rf.currentTerm
		response.FollowerCommitedIndex = rf.commitIndex
		return
	}

	// 尝试推进 committed Index
	rf.advanceFollowerCommittedIndex(request.LeaderCommit)
	response.FollowerCommitedIndex = rf.commitIndex
	response.Term = rf.currentTerm
	response.Success = true
	rf.resetElectionTimer()
}
```

* `checkStepDown`
  * 判断是否需要 step down

```go
// in lock - 检查的 term 与 state 以判断是否需要状态回退
func (rf *Raft) checkStepDown(requestTerm int, leaderId int) {
	if requestTerm > rf.currentTerm {
		// Raft node 接收到来自拥有更高 term 的新 leader 的信息
		rf.stepDown(requestTerm)
	} else if rf.currentState != StateFollower {
		// candidate 收到来自同一 term 的新 leader 的信息
		rf.stepDown(requestTerm)
	} else if rf.votedFor == -1 {
		// follower 收到来自同一 term 的新 leader 的信息
		rf.stepDown(requestTerm)
	}

	// 保存当前 leader
	if rf.votedFor == -1 {
		rf.votedFor = leaderId
	}
}
```

* `checkAndResolveConflict`
  * 啊手动阀
* `advanceFollowerCommittedIndex`
  * 奥斯丁激发
* `handleAppendEntriesResponse`

```go
// Leader 处理 append entry RPC 响应
func (rf *Raft) handleAppendEntriesResponse(server int, request *AppendEntriesRequest, response *AppendEntriesResponse) {
	rf.mu.Lock()
	defer rf.mu.Unlock()

	// 对方比我任期大了，立即转为follower
	if response.Term > rf.currentTerm {
		rf.stepDown(response.Term)
		return
	}

	// case 2. 收到了过期的RPC回复
	if request.Term != rf.currentTerm || response.Term != rf.currentTerm {
		return
	}

	if !response.Success {
		// prev_log_index and prev_log_term doesn't match
		// The peer contains less logs than leader
		// 直接从follower的commitIndex发
		rf.nextIndex[server] = response.FollowerCommitedIndex + 1
	} else {
		// 收到了正确的回答,修改对应 matchIndex 和 nextIndex
		prev := rf.matchIndex[server]
		rf.matchIndex[server] = Max(rf.matchIndex[server], request.PrevLogIndex+len(request.Entries))
		now := rf.matchIndex[server]
		rf.nextIndex[server] = Max(request.PrevLogIndex+len(request.Entries)+1, rf.matchIndex[server])
		INFO("[%d]---Received true from [%d], Now matchIndex = %d, nextIndex = %d---<handleAppendEntriesResponse> ", rf.me, server, rf.matchIndex[server], rf.nextIndex[server])
		if prev != now {

			// 判断commit是否可以向前推，排序，取中位数和现在的commitIndex对比
			sortMatchIndex := make([]int, 0)
			sortMatchIndex = append(sortMatchIndex, rf.lastLogIndex())
			for i, value := range rf.matchIndex {
				if i == rf.me {
					continue
				}
				sortMatchIndex = append(sortMatchIndex, value)
			}
			sort.Ints(sortMatchIndex)
			midIndex := len(rf.peers) / 2
			if len(rf.peers)%2 == 0 {
				midIndex--
			}
			newCommitIndex := sortMatchIndex[midIndex]

			// 注意：只能提交自己的任期的日志，解决了论文Fig 8的问题
			if newCommitIndex > rf.commitIndex && rf.getLogTermWithIndex(newCommitIndex) == rf.currentTerm {
				rf.commitIndex = newCommitIndex
				DPrintf("[%d]---Leader commitIndex change to %d", rf.me, rf.commitIndex)
				rf.applyCond.Broadcast()
			}
		}
	}
}
```

TODO 流程说明

### Part 2C - 持久化

TODO 函数调用图

TODO 关键代码

```C++
// 持久化常规数据(无需加锁，外部有锁)
func (rf *Raft) persist() {
	w := new(bytes.Buffer)
	e := labgob.NewEncoder(w) // TODO 注释
	e.Encode(rf.currentTerm)
	e.Encode(rf.votedFor)
	e.Encode(rf.commitIndex)
	e.Encode(rf.lastSSPointIndex)
	e.Encode(rf.lastSSPointTerm)
	e.Encode(rf.logs)

	data := w.Bytes()
	INFO("[%d]---Persisted      currentTerm %d voteFor %d the size of logs %d", rf.me, rf.currentTerm, rf.votedFor, len(rf.logs))
	rf.persister.SaveRaftState(data)
}
```

TODO 流程说明

### Part 2D - 日志压缩

TODO 函数调用图

TODO 关键代码

```C++

```

TODO 流程说明

## 实验结果

## 参考链接

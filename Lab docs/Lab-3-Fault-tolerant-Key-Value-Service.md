# MIT 6.824 - Lab 3: Fault-tolerant Key/Value Service

实验地址：<http://nil.csail.mit.edu/6.824/2021/labs/lab-kvraft.html>

目录：

- [MIT 6.824 - Lab 3: Fault-tolerant Key/Value Service](#mit-6824---lab-3-fault-tolerant-keyvalue-service)
  - [实验内容](#实验内容)
  - [实验思路](#实验思路)
    - [强一致性实现](#强一致性实现)
    - [系统交互流程](#系统交互流程)
    - [client](#client)
    - [server](#server)
  - [实验结果](#实验结果)
  - [参考链接](#参考链接)

## 实验内容

本实验主要基于 Lab 2 中实现的 Raft 库构建一个可以容错的 key/value 存储系统。该存储系统将由多个使用 Raft 进行复制的 kv server 组成。只要集群中大多数服务器处于活动状态并且可以通信，该存储系统就应该能正确处理客户端请求。

该 key/value 存储系统需要支持三种操作：

* `Put(key, value)`
  * 替换 key 的值为 value
* `Append(key, arg)`
  * 新增 key 的值为 arg
* `Get(key)`
  * 获取 key 的值，如果 key 不存在则返回空字符串

实验要求对客户端 `Get/Put/Append` 方法的调用提供强一致性，即整个系统在客户端看来表现得像是只有一个副本，每次读写都能感知到系统的最新修改。

## 实验思路

### 强一致性实现

### 系统交互流程

### client

### server

## 实验结果

## 参考链接

* [6.824 Lab 3: Fault-tolerant Key/Value Service](http://nil.csail.mit.edu/6.824/2021/labs/lab-kvraft.html)
* [In Search of an Understandable Consensus Algorithm (Extended Version)](https://raft.github.io/raft.pdf)

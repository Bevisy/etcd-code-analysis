# Raft 协议

## Leader 选举

## 日志复制

## 网络分区


## 日志压缩及快照
创建快照文件时，会将整个节点的状态进行序列化，然后写入稳定的持久化存储。

多个快照文件存在的意义？如果旧的日志已经被压缩，应该只有最新的快照才能恢复完整数据

## linearizablity(线性一致性)语义
客户端对于每个请求都产生一个唯一的序列号，然后 由服务端为每个客户端维护一个 Session， 并对每个请求进行去重。当服务端接收到一个请求时， 会检测其序列号，如果该请求 己经被执行过了，那么就立即返回结果，而不会重新执行。

## 只读请求
1. 网络分区后，少数派节点读请求，请求失败
etcd 默认使用 ReadIndex 机制来实现线性一致性读。当 etcd 节点处理读请求时，节点必须先向 leader 查询 commitIndex，少数派无法联系 leader，导致读取失败。如果开启非线性一致性读请求，则读请求不会失败。

非线性一致性读开启方式：
- SDK 访问需要配置 WithSerializable 选项（默认并不开启）
- etcdctl 访问需要配置 --consistency=s 选项

2. 网络分区后，旧的 leader 节点处于少数派，读取请求到旧 leader 节点，读取失败
leader 节点处理只读请求前，必须检查集群是否存在最新的 Leader 节点。Raft 协议中，通过让 Leader 节点在处理只读请求之前，先和集群中的半数以上的节点交换一次心跳消息来解决这个问题。如果该 Leader 节点可以与集群中半数以上的节点交换一次心跳，则表示该 Leader 节点依然为该集群最新的 Leader 节点 。这样， readlndex 值也就是整个集群中所有节点所能看到的最大提交位置（ commitlndex ） 。

3. 网络分区后，新的 leader 节点处理读请求
TODO
Leader 节点必须有关于己提交日志的最新信息，虽然在节点刚刚赢得选举成为 Leader 时 ，拥有所有己经被提交的日志记录，但是在其任期刚开始时，它可能不知道哪些是己经被提交的日志。为了明确这些信息，它会在其任期开始时提交一条空日志记录，这样上一个任期中的所有日志都会被提交。
Leader 节点会记录该只读请求对应的编号作为 readlndex，当 Leader 节点的提交位置(commitlnd巳x ） 达到或是超过该位置之后， 即 可响应该只读请求。

## ReadIndex
基本原理：Leader节点在处理读请求时，首先需要与集群多数节点确认自己依然是Leader，然后读取已经被应用到应用状态机的最新数据。

基本原理包含两方面：
- Leader首先通过某种机制确认自己依然是Leader；
- Leader需要给客户端返回最近已应用的数据：即最新被应用到状态机的数据。




## PreVote 状态
Raft 协议为了优化无意义的选举，给节点添加了一个 PreVote 的状态：当某个节点要发起选举之前，需要先进入 PreVote 的状态。在 PreVot巳状态下的节点会先尝试连接集群中的其他节点，如果能够成功连接到半数以上的节点，才能真正切换成 Candidate 状态并发起新一轮的选举。

# etcd-raft 模块

## Config 结构体
代码路径：etcd/raft/raft.go

Config 包含启动 raft 需要的参数

## Storage 接口
代码路径：etcd/raft/storage.go

存储当前节点的 Entry 记录；节点快照创建、查询、删除等操作

## unstable 结构体
代码路径：etcd/raft/log_unstable.go
除了 Storage 存储 Entry 外，unstable 结构体也会存储 Entry 记录。
unstable 使用内存数组维护所有的 Entry 记录。对于 Leader 节点而言，它维护了客户端请求对应的 Entry 记录；对于 Follower 节点而言，它维护的是从 Leader 节点复制过来的 Entry 记录。无论是 Leader 节点还是 Follower 节点，对于刚刚接收到的 Entry 记录首先都会被存储在unstable 中。然后按照 Raft 协议将 unstable 中缓存的这些 Entry 记录交给上层模块进行处理，上层模块会将这些 Entry 记录发送到集群其他节点或进行保存（写入 Storage 中）。之后，上层模块会调用 Advance（）方法通知底层的 etcd-raft 模块将 unstable 中对应的 Entry 记录删除（因为己经保存到了 Storage 中）。正因为 unstable 中保存的 Entry 记录并未进行持久化，可能会因节点故障而意外丢失，所以被称为 unstable。

unstable 提供的方法很多和 Storage 类似。raftLog 中，很多方法都是先尝试调用 unstable 方法，在其失败后返回 (0, false) 表示失败，在尝试 Storage 的对应方法。

## raftLog 结构体
代码路径：etcd/raft/log.go

Raft 协议中日志复制部分的核心就是在集群中各个节点之间完成日志的复制，因此在 etcd-raft 模块的实现中使用 raftLog 结构来管理节点上的日志，就依赖于 Storage 接口和 unstable 结构体。

raftLog 核心字段含义和功能：


## raft
代码路径：etcd/raft/raft.go

etcd-raft 模块核心实现


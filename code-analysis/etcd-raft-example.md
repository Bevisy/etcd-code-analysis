# raft-example 示例
1. 将客户端发来的请求传递给底层 etcd-raft 组件处理
2. 从 node.readyc 获取 Ready 实例，并处理其封装的数据
3. 管理 WAL 日志文件
4. 管理快照数据
5. 管理逻辑时钟
6. 将 etcd-raft 模块返回的待发送消息，通过网络组件发送到指定的节点


## WAL struct
```go
// WAL is a logical representation of the stable storage.
// WAL is either in read mode or append mode but not both.
// A newly created WAL is in append mode, and ready for appending records.
// A just opened WAL is in read mode, and ready for reading records.
// The WAL will be ready for appending after reading out all the previous records.
type WAL struct {
	lg *zap.Logger

	dir string // the living directory of the underlay files

	// dirFile is a fd for the wal directory for syncing on Rename
	dirFile *os.File

	metadata []byte           // metadata recorded at the head of each WAL
	state    raftpb.HardState // hardstate recorded at the head of WAL

	start     walpb.Snapshot // snapshot to start reading
	decoder   *decoder       // decoder to decode records
	readClose func() error   // closer for decode reader

	unsafeNoSync bool // if set, do not fsync

	mu      sync.Mutex
	enti    uint64   // index of the last entry saved to the wal
	encoder *encoder // encoder to encode records

	locks []*fileutil.LockedFile // the locked files the WAL holds (the name is increasing)
	fp    *filePipeline
}
```

### 循环冗余检验 CRC（Cyclic redundancy check）

### wal 文件最大容量
sequence number- last raft index 均为64位，形如：000000000000011c-0000000002366d20.wal
每个 wal 固定容量上限为 64MB，大约是index数为：集群抽样数据 337416 340366 334617 ~= 340000
结果：2^63÷340000 ~= 222222222222 = 2222亿
longrun环境 69天 sequence增加至 440
换算下来就是： 222222222222/440*69/365 ~= 64亿年

### wal 包文档
包 wal 提供了 etcd 预写式日志的实现。
WAL 被创建在指定的目录，有许多分段的WAL文件组成。在每个文件的内部，使用 Save 方法将 raft 状态和 entries 附加到该文件中：
```go
	metadata := []byte{}
	w, err := wal.Create(zap.NewExample(), "/var/lib/etcd", metadata)
	...
	err := w.Save(s, ents)
```
将 raft 快照保存到磁盘后，应调用 SaveSnapshot 方法进行记录。因此，WAL 可以在重新启动时与保存的快照匹配。
```go
	err := w.SaveSnapshot(walpb.Snapshot{Index: 10, Term: 2})
```
用户使用完 WAL 后，必须将其关闭：
```go
	w.Close()
```
每个WAL文件都是WAL记录的流。 WAL记录是一个 length field 和一个 wal record protobuf。 recorf protobuf 包含CRC，类型和数据有效载荷。 length field 是一个64位打包结构，该结构保留其余逻辑记录数据的低56位的长度以及其物理填充的最高有效字节的前三位。 每个记录都是8字节对齐的，因此长度字段永远不会被撕裂。 CRC包含当前记录之前的所有记录协议的CRC32值。

WAL文件以以下格式放置在目录中：
$seq-$index.wal

要创建的第一个WAL文件将是0000000000000000-0000000000000000.wal，指示初始序列0和初始raft索引0。写入WAL的第一个条目务必具有raft索引0。

如果WAL的大小超过64MB，WAL将剪切其当前的尾部 wal 文件。 这将增加一个内部序列号，并导致创建一个新文件。 如果保存的最后一个raft索引是0x20，并且这是在该WAL上第一次被调用，则序列将从0x0递增到0x1。 新文件将是：0000000000000001-0000000000000021.wal。 如果第二次剪切稍后发布带有增量索引的0x10条目，则该文件将被称为：0000000000000002-0000000000000031.wal。

以后可以在特定快照中打开WAL。 如果没有快照，则应传递一个空快照。
```go
	w, err := wal.Open("/var/lib/etcd", walpb.Snapshot{Index: 10, Term: 2})
	...
```
快照必须已经被写入 WAL。

其他项目无法保存到这个WAL直到所有从给定的快照到WAL结束的项目首先被读取：
```go
	metadata, state, ents, err := w.ReadAll()
```
这将为您提供元数据、日志中的最后一个raft.State和raft.Entry项的片段。

### wal 文件物理格式
日志项具有多种类型：
```go
	metadataType int64 = iota + 1 // 特殊的日志项，被写在每个wal文件的头部
	entryType                     // 应用的更新数据，也是日志中存储的最关键数据
	stateType                     // 代表日志项中存储的内容是快照
	crcType                       // 前一个wal文件里面的数据的crc，也是wal文件的第一个记录项
	snapshotType                  // 当前快照的索引{term, index}，即当前的快照位于哪个日志记录，不同于stateType，这里只记录快照的索引，而非快照的数据
```

每个日志项由以下四个部分组成：
- type:     日志项类型
- crc:      校验和
- data:     根据日志项类型存储的实际数据也不尽相同，如 snapshotType 类型的日志项存储的是快照的日志索引，crcType 类型的日志项中则无数据项，其crc字段充当了数据项 TODO: 找出实际的数据，并打印
- padding:  为了保持日志8字节对齐而填充的数据

### wal 文件初始化


### txn 多事务请求
```sh
$ bin/etcdctl txn -i  
compares:

success requests (get, put, del):
put foo f
put foo2 f2
put foo3 f3

failure requests (get, put, del):

SUCCESS

OK

OK

OK

$ etcd-dump-logs infra1.etcd/
Snapshot:
empty
Start dupmping log entries from snapshot.
WAL metadata:
nodeID=8211f1d0f64f3269 clusterID=7230e3513973170f term=2 commitIndex=7 vote=8211f1d0f64f3269
WAL entries:
lastIndex=7
term         index      type    data
   1             1      conf    method=ConfChangeAddNode id=8211f1d0f64f3269
   2             2      norm
   2             3      norm    method=PUT path="/0/members/8211f1d0f64f3269/attributes" val="{\"name\":\"infra1\",\"clientURLs\":[\"http://127.0.0.1:2379\"]}"
   2             4      norm    method=PUT path="/0/version" val="3.4.0"
   2             5      norm    header:<ID:3632568352834133765 > put:<key:"foo" value:"bar" > 
   2             6      norm    header:<ID:3632568352834133771 > txn:<compare:<target:VALUE key:"foo" value:"bar" > success:<request_put:<key:"foo2" value:"bar2" > > failure:<request_range:<key:"foo2" > > > 
   2             7      norm    header:<ID:3632568352834133774 > txn:<success:<request_put:<key:"foo" value:"f" > > success:<request_put:<key:"foo2" value:"f2" > > success:<request_put:<key:"foo3" value:"f3" > > > 

Entry types () count is : 7%
```

### db
```sh
$ etcd-dump-db list-bucket infra1.etcd               
alarm
auth
authRoles
authUsers
cluster
key
lease
members
members_removed
meta

$ etcd-dump-db iterate-bucket infra1.etcd members
key="8211f1d0f64f3269", value="{\"id\":9372538179322589801,\"peerURLs\":[\"http://127.0.0.1:12380\"],\"name\":\"infra1\",\"clientURLs\":[\"http://127.0.0.1:2379\"]}"

$ etcd-dump-db iterate-bucket infra1.etcd cluster
key="clusterVersion", value="3.4.0"

$ etcd-dump-db iterate-bucket infra1.etcd key  
# 第三次提交： txn <<<'
# put foo "f"
# put foo2 "f2"
# put foo3 "f3"
# '
# 同一事务内
key="\x00\x00\x00\x00\x00\x00\x00\x04_\x00\x00\x00\x00\x00\x00\x00\x02", value="\n\x04foo3\x10\x04\x18\x04 \x01*\x02f3"
key="\x00\x00\x00\x00\x00\x00\x00\x04_\x00\x00\x00\x00\x00\x00\x00\x01", value="\n\x04foo2\x10\x03\x18\x04 \x02*\x02f2"
key="\x00\x00\x00\x00\x00\x00\x00\x04_\x00\x00\x00\x00\x00\x00\x00\x00", value="\n\x03foo\x10\x02\x18\x04 \x02*\x01f"
# 第二次提交： put foo2 "bar2"
key="\x00\x00\x00\x00\x00\x00\x00\x03_\x00\x00\x00\x00\x00\x00\x00\x00", value="\n\x04foo2\x10\x03\x18\x03 \x01*\x04bar2"
# 初始提交： put foo "bar"
key="\x00\x00\x00\x00\x00\x00\x00\x02_\x00\x00\x00\x00\x00\x00\x00\x00", value="\n\x03foo\x10\x02\x18\x02 \x01*\x03bar"
```
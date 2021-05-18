# etcd storage

## backend
接口 Backend
```go
// https://sourcegraph.com/github.com/etcd-io/etcd@d19fbe5/-/blob/mvcc/backend/backend.go#L51-69
type Backend interface {
	// ReadTx returns a read transaction. It is replaced by ConcurrentReadTx in the main data path, see #10523.
	ReadTx() ReadTx
	BatchTx() BatchTx
	// ConcurrentReadTx returns a non-blocking read transaction.
	ConcurrentReadTx() ReadTx

	Snapshot() Snapshot
	Hash(ignores map[IgnoreKey]struct{}) (uint32, error)
	// Size returns the current size of the backend physically allocated.
	// The backend can hold DB space that is not utilized at the moment,
	// since it can conduct pre-allocation or spare unused space for recycling.
	// Use SizeInUse() instead for the actual DB size.
	Size() int64
	// SizeInUse returns the current size of the backend logically in use.
	// Since the backend can manage free space in a non-byte unit such as
	// number of pages, the returned value can be not exactly accurate in bytes.
	SizeInUse() int64
	// OpenReadTxN returns the number of currently open read transactions in the backend.
	OpenReadTxN() int64
	Defrag() error
	ForceCommit()
	Close() error
}
```

实现接口的结构体 backend
```go
// https://sourcegraph.com/github.com/etcd-io/etcd@d19fbe5/-/blob/mvcc/backend/backend.go#L85:6
type backend struct {
	// size and commits are used with atomic operations so they must be
	// 64-bit aligned, otherwise 32-bit tests will crash

	// size is the number of bytes allocated in the backend
	size int64
	// sizeInUse is the number of bytes actually used in the backend
	sizeInUse int64
	// commits counts number of commits since start
	commits int64
	// openReadTxN is the number of currently open read transactions in the backend
	openReadTxN int64

	mu sync.RWMutex
	db *bolt.DB

	batchInterval time.Duration
	batchLimit    int
	batchTx       *batchTxBuffered

	readTx *readTx

	stopc chan struct{}
	donec chan struct{}

	lg *zap.Logger
}
```
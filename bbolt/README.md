(原始项目的README改为了README_ORIGIN.md)
(在本文中BoltDB简称为bolt)

# bolt 在磁盘中是什么形式
下面描述一下 bolt 的文件在磁盘中是什么形式存储的, 也就是真正在磁盘中的时候是什么样子的

Page 类型
- meta page
- freelist page
- branch page
- leaf page


## Head
位于 page.go 中的 type page struct{}
```go
┌────┬───────┬───────┬──────────┐
│ id │ flags │ count │ overflow │ (8+2+2+4)bytes
└────┴───────┴───────┴──────────┘
type pgid uint64
type page struct {
    // 每个页的唯一id
	id       pgid
    // 类型
	flags    uint16
    // 页中元素的数量
	count    uint16
    // 数据是否有溢出(主要是freelist page中有用)
	overflow uint32
}
```


## meta page 的内容
位于 db.go 中的 type meta struct{}
```go
type meta struct {
    // 魔数
	magic    uint32

    // 版本号(固定为2)
	version  uint32

    // 页大小(4KB)
	pageSize uint32

    // 
	flags    uint32

    // 
	root     bucket

    // 空闲列表页的 page id
	freelist pgid

    // 
	pgid     pgid

    // 数据库事务操作序号
	txid     txid

    // data数据的hash摘要, 用于判断data是否损坏
	checksum uint64
}
```

## freelist page 的内容
位于 freelist.go 中的 type freelist struct{}
```go
┌────┬───────┬─────────────────┬──────────┐
│ id │ flags │ count(< 0xffff) │ overflow │ (8+2+2+4)bytes
├────┴─┬─────┴┬──────┬──────┬──┴───┬──────┤
│ pgid │ pgid │ pgid │ .... │ pgid │ pgid │ (8*n)bytes
└──────┴──────┴──────┴──────┴──────┴──────┘

┌────┬───────┬──────────────────┬──────────┐
│ id │ flags │ count(>= 0xffff) │ overflow │ (8+2+2+4)bytes
├────┴──┬────┴─┬──────┬──────┬──┴───┬──────┤
│ COUNT │ pgid │ pgid │ .... │ pgid │ pgid │ (8*n)bytes
└───────┴──────┴──────┴──────┴──────┴──────┘
```

## branch page 的内容
```go
// page.go
type branchPageElement struct {
	pos   uint32
	ksize uint32
	pgid  pgid
}

a-->┌────┬───────┬───────┬──────────┐
    │ id │ flags │ count │ overflow │ (8+2+2+4)bytes
b-->├────┴─┬─────┴──┬────┴──┬───────┘
    │ pos1 │ ksize1 │ pgid1 │ (4+4+8)bytes
c-->├──────┼────────┼───────┤
    │ pos2 │ ksize2 │ pgid2 │ (4+4+8)bytes
    ├──────┴────────┴───────┤
    │ ..................... │ 
    ├──────┬────────┬───────┤
    │ posn │ ksizen │ pgidn │ (4+4+8)bytes
d-->├──────┼────────┴───────┘
    │ key1 │ 
e-->├──────┤
    │ key2 │
    ├──────┤
    │ .... │ 
    ├──────┤
    │ keyn │
    └──────┘
```
pos 实际上就是 key 相对于 head 结尾的偏移量

如图:
```
a = 0
b = 16*8
c = 16*8 + 16*8
pos1 = d-b
pos2 = e-c
```

## leaf page 的内容
```go
// page.go
type leafPageElement struct {
    // 标识当前的节点是否是 bucket 类型
    // page.go 定义了 bucketLeafFlag = 0x01
    // 0 不是 bucket 类型
    // 1 是 bucket 类型
	flags uint32
	pos   uint32
	ksize uint32
	vsize uint32
}

a-->┌────┬───────┬───────┬──────────┐
    │ id │ flags │ count │ overflow │ (8+2+2+4)bytes
b-->├────┴──┬────┴─┬─────┴──┬───────┴┐
    │ flags │ pos1 │ ksize1 │ vsize1 │ (4+4+4+4)bytes
c-->├───────┼──────┼────────┼────────┤
    │ flags │ pos2 │ ksize2 │ vsize2 │ (4+4+4+4)bytes
    ├───────┴──────┴────────┴────────┤
    │ .............................. │ 
    ├───────┬──────┬────────┬────────┤
    │ flags │ posn │ ksizen │ vsizen │ (4+4+4+4)bytes
d-->├──────┬┴──────┴┬───────┴────────┘
    │ key1 │ value1 │ 
e-->├──────┼────────┤
    │ key2 │ value2 │ 
    ├──────┴────────┤
    │ ............. │ 
    ├──────┬────────┤
    │ keyn │ valuen │ 
    └──────┴────────┘
```

pos 实际上就是 kv 相对于 head 结尾的偏移量
如图:
```
a = 0
b = 16*8
c = 16*8 + 16*8
pos1 = d-b
pos2 = e-c
```


# bolt 在内存中是什么形式
page 是物理存储的基本单位, 那么 node 就是在逻辑存储的基本单位. 也是在内存中存储的单位

位于 node.go 下的 type node struct{}

node 只是描述了 branch/leaf page 在内存中的格式

而 meta/freelist page 在内存中是在 DB 结构体中, 他们也有专门的结构体 type meta struct{} 以及 type freelist struct{}. 分别位于 db.go 与 freelist.go 中

```go
type nodes []*node
type inode struct {
    // branch page OR leaf page
	flags uint32
    // 如果类型是 branch page 时
    // 表示的是 子节点的 page id
	pgid  pgid
	key   []byte
    // 如果类型是 leaf page 时
    // 表示的是 kv 对中的值(value)
	value []byte
}
type inodes []inode

type node struct {
    // 该 node 节点位于哪个 bucket
	bucket     *Bucket
    // 是否是叶子节点
    // 对应到 page header 中的 flags 字段
	isLeaf     bool
    // 是否平衡
	unbalanced bool
    // 是否需要分裂
	spilled    bool
    // 节点最小的 key
	key        []byte
    // 节点对应的 page id
	pgid       pgid
    // 当前节点的父节点
	parent     *node
    // 当前节点的孩子节点
    // 在 spill rebalance 过程中使用
	children   nodes
    // 节点中存储的数据
    // 广义的 kv 数据(v 可能是子节点)
    // page header 中的 count 可以通过 len(inodes) 获得
    // branch/leaf 的数据都在 inodes 中可以体现到
	inodes     inodes
}
```

## node 以及 page 的转换(branch/leaf page)

在这里可以清楚的看到branch/leaf的磁盘内存的转换过程

```go
// node.go

// 读入
func (n *node) read(p *page)
// 落盘
func (n *node) write(p *page)
```

## node 以及 page 的转换(meta page)
```go
// 内存中的 meta 结构
type meta struct {
	magic    uint32
	version  uint32
	pageSize uint32
	flags    uint32
    // 对应一个 root bucret
	root     bucket
	freelist pgid
	pgid     pgid
	txid     txid
	checksum uint64
}

// 读入
// db启动的时候就会先读入
func (db *DB) mmap(minsz int) (err error) {
    // ....
    db.meta0 = db.page(0).meta()
    db.meta1 = db.page(1).meta()
    // ...
}
func (p *page) meta() *meta {
	return (*meta)(unsafeAdd(unsafe.Pointer(p), unsafe.Sizeof(*p)))
}

// 落盘
// 每一次事务 commit 的时候都会调用
// Note: read only不会改变 meta, 只有 read-write 会改变 meta 信息
// (tx *Tx)Commit -> (tx *Tx)writeMeta -> (m *meta)write
// db.go
func (m *meta) write(p *page) 
```




## node 以及 page 的转换(freelist page)
```go
// 读入 freelist.go
func (f *freelist) read(p *page) 
// 落盘 freelist.go
func (f *freelist) write(p *page) error 
```

# bolt 如何组织数据(Bucket)

每一个 Bucket 对应一棵 B+ 树

先看一下 Bucket 的定义
```go
type bucket struct {
	root     pgid   // page id of the bucket's root-level page
	sequence uint64 // monotonically incrementing, used by NextSequence()
}

type Bucket struct {
	*bucket
    // 当前 Bucket 关联的事务
	tx       *Tx                // the associated transaction
    // 子 Bucket
	buckets  map[string]*Bucket // subbucket cache
    // 关联的 page
	page     *page              // inline page reference
    // Bucket 管理的树的根节点
	rootNode *node              // materialized node for the root page.
    // 缓存
	nodes    map[pgid]*node     // node cache

	// Sets the threshold for filling nodes when they split. By default,
	// the bucket will fill to 50% but it can be useful to increase this
	// amount if you know that your write workloads are mostly append-only.
	//
	// This is non-persisted across transactions so it must be set in every Tx.
    // 填充率
    // 与B+树的分裂相关
	FillPercent float64
}
```

## Bucket 中重要的工具 Cursor 游标

bolt 中没有使用传统的 B+ 树中将叶子节点使用链表串起来的方式, 而是使用的 cursor游标, 通过路径回溯的方式, 来支持范围查询. 

所以 cursor 对于 bucket 中 curd 还是听重要的. 
```go
type elemRef struct {
    // 当前位置对应的 page
	page  *page
    // 当前位置对应的 node
    // 可能是没有被反序列化的
    // 但是没关系, 使用page可以序列化
	node  *node
    // 在第几个 kv 对
	index int
}
type Cursor struct {
	bucket *Bucket
	stack  []elemRef
}
```


# 事务
```go
type Tx struct {
	writable       bool
	managed        bool
	db             *DB
	meta           *meta
	root           Bucket
	pages          map[pgid]*page
	stats          TxStats
	commitHandlers []func()

	// WriteFlag specifies the flag for write-related methods like WriteTo().
	// Tx opens the database file with the specified flag to copy the data.
	//
	// By default, the flag is unset, which works well for mostly in-memory
	// workloads. For databases that are much larger than available RAM,
	// set the flag to syscall.O_DIRECT to avoid trashing the page cache.
	WriteFlag int
}
```


# 感谢
- [自底向上分析 BoltDB 源码](https://www.bookstack.cn/books/jaydenwen123-boltdb_book)
- 微信公众号: 小徐先生的编程世界
- 微信公众号: 数据小冰
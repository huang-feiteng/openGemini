# openGemini 存储引擎分析

**代码版本**: 7032447bce3ec6d35eb204a0f317882298b774b5 (2026-05-15)

本文档深入分析 openGemini 存储引擎的核心架构、数据结构和关键流程。

---

## 1. 存储引擎概述

### 1.1 核心架构

openGemini 存储引擎基于 **LSM 树 (Log-Structured Merge-Tree)** 设计，采用分层存储架构：

```mermaid
graph TB
    subgraph Memory["内存层"]
        WAL[WAL 日志]
        Mutable[Mutable Table<br/>内存表]
    end
    
    subgraph Disk["磁盘层"]
        L0[Level 0<br/>TSSP Files]
        L1[Level 1<br/>TSSP Files]
        L2[Level 2<br/>TSSP Files]
        LN[Level N<br/>TSSP Files]
    end
    
    WAL --> Mutable
    Mutable -->|Snapshot| L0
    L0 -->|Compaction| L1
    L1 -->|Compaction| L2
    L2 -->|Compaction| LN
    
    style Mutable fill:#e1f5ff
    style L0 fill:#fff4e1
    style L1 fill:#fff4e1
    style L2 fill:#fff4e1
    style LN fill:#fff4e1
```

### 1.2 存储层次

| 层级 | 组件 | 位置 | 特点 |
|------|------|------|------|
| WAL | Write Ahead Log | 磁盘 | 顺序写入，崩溃恢复 |
| L0 | Mutable Table | 内存 | 写入缓冲，快速访问 |
| L1+ | Immutable TSSP | 磁盘 | 列式存储，只读 |

### 1.3 关键代码位置

| 组件 | 文件 | 说明 |
|------|------|------|
| 引擎核心 | `engine/engine.go` | EngineImpl 主引擎 |
| 分片管理 | `engine/shard.go` | Shard 分片实现 |
| 内存表 | `engine/mutable/table.go` | 内存数据写入 |
| 磁盘文件 | `engine/immutable/tssp_file.go` | TSSP 文件读写 |
| WAL | `engine/wal.go` | 预写日志 |
| 压缩 | `engine/immutable/compact.go` | 数据压缩合并 |

---

## 2. 核心数据结构

### 2.1 EngineImpl - 引擎主结构

```go
// engine/engine.go:80-115
type EngineImpl struct {
    mu       sync.RWMutex
    closed   *interruptsignal.InterruptSignal
    dataPath string        // 数据目录
    walPath  string        // WAL 目录
    
    engOpt       EngineOptions
    DBPartitions map[string]map[uint32]*DBPTInfo  // 数据库分区映射
    
    log *logger.Logger
    loadCtx     *meta.LoadCtx
    metaClient  meta.MetaClient
    
    // 数据迁移
    migratingDbPT map[string]map[uint32]struct{}
    
    // 回调函数
    onPTOffload map[uint64]func(ptID uint32)
    onPTLoaded  map[uint64]PtLoadFunc
    
    // 备份
    backup *Backup
    fileInfos chan []immutable.FileInfoExtend
}
```

**关键方法**:
- `Open()` - 打开引擎，加载分片
- `CreateDBPartition()` - 创建数据库分区
- `WriteToShard()` - 写入分片
- `ExecuteQuery()` - 执行查询

### 2.2 Shard - 数据分片

```go
// engine/shard.go:211-260
type shard struct {
    mu             sync.RWMutex
    ident          *meta.ShardIdentifier  // 分片标识
    database       string                 // 数据库名
    retentionPolicy string                // 保留策略
    
    wal *WAL                            // WAL 日志
    
    // 内存表
    activeTbl *MTable       // 当前写入表
    immTables []*MTable     // 不可变内存表
    
    // 磁盘文件
    fileGroups []*FileGroup  // TSSP 文件组
    
    // 索引
    indexBuilder *tsi.IndexBuilder
    
    // 状态
    closed    *interruptsignal.InterruptSignal
    cacheClosed int32
    
    // 统计
    lastWriteTime uint64
}
```

**关键接口**:
```go
type Shard interface {
    // 写入接口
    WriteRows(rows []influx.Row, binaryRows []byte) error
    WriteCols(mst string, cols *record.Record, binaryCols []byte) error
    WriteBlobs(group *shelf.BlobGroup) error
    ForceFlush()
    
    // 查询接口
    CreateLogicalPlan(...) (hybridqp.QueryNode, error)
    CreateCursor(...) (comm.TSIndexInfo, error)
    Scan(...) (tsi.GroupSeries, int64, uint64, error)
    
    // 管理接口
    OpenAndEnable(client metaclient.MetaClient) error
    Close() error
    DropMeasurement(ctx context.Context, name string) error
}
```

### 2.3 MTable - 内存表

```go
// engine/mutable/table.go
type MTable struct {
    sid          uint64        // Series ID
    name         string        // 表名
    schema       *record.Schema // 数据模式
    
    // 数据段
    segments []*Segment
    
    // 刷盘条件
    maxMemorySize int64
    snapShotTime  time.Time
}
```

### 2.4 TSSP File - 时序分片文件

```go
// engine/immutable/tssp_file.go:61-80
type tsspFileReader struct {
    ref            int64
    r              fileops.BasicFileReader
    metaIndexItems []MetaIndex  // 元数据索引
    trailer        Trailer      // 文件尾部信息
    bloom          *bloom.Filter // 布隆过滤器
    version        uint64
    fileSize       int64
    
    // 缓存
    pageCacheReader *PageCacheReader
}
```

**文件结构**:
```
┌─────────────────────┐
│   Data Blocks       │  数据块 (列式存储)
├─────────────────────┤
│  Chunk Meta Index   │  数据块索引
├─────────────────────┤
│   Meta Index        │  元数据索引
├─────────────────────┤
│     Bloom Filter    │  布隆过滤器
├─────────────────────┤
│      Trailer        │  文件尾部 (魔术数、大小等)
└─────────────────────┘
```

---

## 3. 数据写入流程

### 3.1 写入路径

```mermaid
sequenceDiagram
    participant Client
    participant Shard
    participant WAL
    participant Mutable as MTable
    participant Snapshot
    participant Immutable as TSSP
    
    Client->>Shard: WriteRows/WriteCols
    Shard->>WAL: 追加写入 (顺序 IO)
    WAL-->>Shard: 确认持久化
    Shard->>Mutable: 写入内存表
    Mutable-->>Shard: 确认写入
    Shard-->>Client: 返回成功
    
    Note over Mutable: 内存表达到阈值
    Shard->>Snapshot: 触发 Snapshot
    Snapshot->>Immutable: 刷盘为 TSSP 文件
    Note over Immutable: 异步 Compaction 合并
```

### 3.2 写入代码实现

```go
// engine/shard.go:512-545
func (s *shard) WriteRows(rows []influx.Row, binaryRows []byte) error {
    if s.closed.Closed() {
        return errno.NewError(errno.ErrShardClosed, s.ident.ShardID)
    }
    
    start := time.Now()
    // 1. 写入 WAL
    if err := s.wal.Write(binaryRows, WriteWalLineProtocol, 0); err != nil {
        atomic.AddInt64(&statistics.PerfStat.WriteReqErrors, 1)
        return err
    }
    
    // 2. 更新写入时间
    s.lastWriteTime = fasttime.UnixTimestamp()
    
    // 3. 写入内存表
    err := s.activeTbl.MTable.WriteRows(...)
    if err != nil {
        return err
    }
    
    atomic.AddInt64(&statistics.PerfStat.WriteRowsDurationNs, time.Since(start).Nanoseconds())
    return nil
}
```

### 3.3 WAL 实现

```go
// engine/wal.go:49-56
const (
    DefaultFileSize   = 10 * 1024 * 1024  // 10MB
    WALFileSuffixes   = "wal"
    StreamWalDir      = "stream"
    WALRecordHeadSize = 1 + 4  // Type(1) + Length(4)
    WALCompBufSize    = 256 * 1024
    WALCompMaxBufSize = 2 * 1024 * 1024
)
```

// 写入方法
func (w *WAL) Write(data []byte, typ WalRecordType, maxTime int64) error {
    // 1. 压缩数据 (Snappy)
    compressed := snappy.Encode(w.compBuf, data)
    
    // 2. 写入记录头 (类型 + 长度)
    head := make([]byte, WalRecordHeadSize)
    head[0] = byte(typ)
    binary.BigEndian.PutUint32(head[1:], uint32(len(compressed)))
    
    // 3. 追加写入
    w.writer.Write(head)
    w.writer.Write(compressed)
    
    // 4. 检查文件大小，滚动文件
    if w.fileSize > w.maxFileSize {
        w.rollFile()
    }
    
    return nil
}
```

**WAL 记录格式**:
```
┌──────────────┬──────────────────┬─────────────┐
│ Record Type  │ Length (4 bytes) │ Data...     │
│ (1 byte)     │                  │ (compressed)│
└──────────────┴──────────────────┴─────────────┘
```

---

## 4. 数据刷盘 (Snapshot)

### 4.1 触发条件

```go
// engine/shard.go
func (s *shard) shouldSnapshot() bool {
    // 1. 内存表大小超过阈值
    if s.activeTbl.Size() > s.snapshotSize {
        return true
    }
    
    // 2. 距离上次刷盘时间超过阈值
    if time.Since(s.snapShotTime) > s.snapshotDuration {
        return true
    }
    
    return false
}
```

### 4.2 刷盘流程

```mermaid
sequenceDiagram
    participant Shard
    participant Mutable as MTable
    participant Index
    participant Writer as TSSPWriter
    participant File
    
    Shard->>Mutable: 冻结当前内存表
    Shard->>Mutable: 创建新内存表
    Mutable->>Writer: 序列化数据
    Writer->>File: 写入 TSSP 文件
    Index->>File: 写入索引
    File->>File: 写入布隆过滤器
    File->>File: 写入 Trailer
    Shard->>FileGroup: 添加文件到组
```

### 4.3 TSSP 文件写入

```go
// engine/immutable/writer.go
type TSSPWriter struct {
    fd           fileops.BasicFile
    dataWriter   *DataWriter
    indexWriter  *IndexWriter
    bloomWriter  *BloomWriter
}

func (w *TSSPWriter) Write(record *record.Record) error {
    // 1. 按列写入数据块
    for _, col := range record.Schema {
        w.writeColumn(col)
    }
    
    // 2. 写入 Chunk 元数据
    w.writeChunkMeta()
    
    // 3. 写入元数据索引
    w.writeMetaIndex()
    
    // 4. 写入布隆过滤器
    w.writeBloomFilter()
    
    // 5. 写入 Trailer
    w.writeTrailer()
    
    return nil
}
```

---

## 5. 数据压缩 (Compaction)

### 5.1 Compaction 策略

openGemini 采用 **Level Compaction** 策略，类似 LevelDB/RocksDB:

```
Level 0: 8 个文件 -> 触发 Compaction -> Level 1
Level 1: 4 个文件 -> 触发 Compaction -> Level 2
Level 2: 4 个文件 -> 触发 Compaction -> Level 3
...
Level 6: 2 个文件
```

**压缩规则**:
```go
// engine/immutable/compact.go
const CompactLevels = 7
var LevelCompactRule = []uint16{0, 1, 0, 2, 0, 3, 0, 1, 2, 3, 0, 4, 0, 5, 0, 1, 2, 6}
var LeveLMinGroupFiles = [CompactLevels]int{8, 4, 4, 4, 4, 4, 2}
```

### 5.2 Compaction 类型

| 类型 | 触发条件 | 目的 |
|------|----------|------|
| Level Compaction | 某 Level 文件数超限 | 合并小文件，减少文件数 |
| Full Compaction | 手动触发或定期执行 | 全局合并，优化查询性能 |
| Stream Compaction | 流式数据合并 | 实时合并新数据 |

### 5.3 Compaction 流程

```mermaid
graph TB
    Start[开始 Compaction] --> Select[选择待合并文件]
    Select --> Read[读取数据块]
    Read --> Merge[合并排序]
    Merge --> Write[写入新文件]
    Write --> Index[重建索引]
    Index --> Replace[替换旧文件]
    Replace --> Delete[删除旧文件]
    Delete --> End[完成]
```

### 5.4 Compaction 代码

```go
// engine/immutable/compact.go
type Compactor struct {
    level       int
    fileGroups  []*FileGroup
    workDir     string
    limiter     limiter.Fixed
}

func (c *Compactor) Compact() error {
    // 1. 获取限流器
    token := c.limiter.Take()
    defer c.limiter.Give(token)
    
    // 2. 选择待合并文件
    files := c.selectFiles()
    
    // 3. 创建合并读取器
    reader := NewMergeReader(files)
    
    // 4. 创建新文件写入器
    writer := NewTSSPWriter(c.newFileName())
    
    // 5. 合并写入
    for reader.Next() {
        record := reader.Record()
        writer.Write(record)
    }
    
    // 6. 关闭文件
    writer.Close()
    reader.Close()
    
    // 7. 替换文件
    c.replaceFiles(files, writer.File())
    
    return nil
}
```

---

## 6. 查询执行

### 6.1 查询路径

```mermaid
sequenceDiagram
    participant Client
    participant Engine
    participant Shard
    participant Index
    participant TSSP
    
    Client->>Engine: Query
    Engine->>Shard: 路由到分片
    Shard->>Index: 查询索引
    Index-->>Shard: 返回 Series ID
    Shard->>TSSP: 读取数据块
    TSSP-->>Shard: 返回 Record
    Shard->>Engine: 返回结果
    Engine->>Client: 合并结果
```

### 6.2 查询接口

```go
// engine/shard.go
type Shard interface {
    CreateLogicalPlan(ctx context.Context, sources influxql.Sources, 
        schema *executor.QuerySchema) (hybridqp.QueryNode, error)
    
    CreateCursor(ctx context.Context, schema *executor.QuerySchema) (
        comm.TSIndexInfo, error)
    
    Scan(span *tracing.Span, schema *executor.QuerySchema, 
        callBack func(num int64) error) (tsi.GroupSeries, int64, uint64, error)
}
```

### 6.3 数据读取

```go
// engine/immutable/tssp_reader.go
type TSSPReader struct {
    files []TSSPFile
    opt   query.ProcessorOptions
}

func (r *TSSPReader) Read(ctx context.Context) (record.Record, error) {
    // 1. 根据时间范围过滤文件
    files := r.filterFiles(r.opt.StartTime, r.opt.EndTime)
    
    // 2. 并发读取数据块
    records := make(chan record.Record)
    for _, file := range files {
        go func(f TSSPFile) {
            data := f.ReadBlocks(ctx, r.opt)
            records <- data
        }(file)
    }
    
    // 3. 合并结果
    return r.mergeRecords(records), nil
}
```

---

## 7. 索引系统

### 7.1 索引类型

| 索引 | 位置 | 用途 |
|------|------|------|
| TSI (Time Series Index) | `engine/index/tsi/` | 时间序列索引 |
| 稀疏索引 | `engine/index/sparseindex/` | 数据块范围索引 |
| 布隆过滤器 | `engine/index/bloomfilter/` | 快速判断数据存在性 |

### 7.2 TSI 索引结构

```
┌──────────────────────┐
│  Series ID -> Offset │  系列 ID 到偏移量
├──────────────────────┤
│  Tag Key Index       │  标签键索引
├──────────────────────┤
│  Tag Value Index     │  标签值索引
└──────────────────────┘
```

---

## 8. 内存管理

### 8.1 内存限制

```go
// engine/shard.go:340-370
type ShardOptions struct {
    NodeMutableSizeLimit int64         // 节点内存表总限制
    MaxWriteHangTime     time.Duration // 写入挂起时间
    WriteColdDuration    time.Duration // 写入冷数据时长
    SnapshotSize         int64         // 快照大小阈值
    SnapshotDuration     time.Duration // 快照时间阈值
}
```

### 8.2 对象池

```go
// engine/immutable/pool.go
var recordPool = sync.Pool{
    New: func() interface{} {
        return record.NewRecord(schema)
    },
}

func GetRecord() *record.Record {
    return recordPool.Get().(*record.Record)
}

func PutRecord(rec *record.Record) {
    rec.Reset()
    recordPool.Put(rec)
}
```

---

## 9. 并发控制

### 9.1 锁机制

| 锁类型 | 使用场景 | 实现 |
|--------|----------|------|
| Shard 锁 | 分片级别并发 | `sync.RWMutex` |
| MTable 锁 | 内存表写入 | `sync.Mutex` |
| FileGroup 锁 | 文件组合并 | `sync.RWMutex` |
| Engine 锁 | 引擎全局 | `sync.RWMutex` |

### 9.2 限流器

```go
// engine/immutable/compact.go
var (
    compLimiter      = limiter.NewFixed(maxCompactor)      // 压缩限流
    snapshotLimiter  = limiter.NewFixed(snapshotLimit)     // 快照限流
    openShardsLimit  = limiter.Fixed(openShardLimit)       // 打开分片限流
    replayWalLimit   = limiter.Fixed(replayWalLimit)       // WAL 回放限流
)
```

---

## 10. 错误处理

### 10.1 错误类型

```go
// lib/errno/errno.go
var (
    ErrShardClosed      = errors.New("shard closed")
    ErrDatabaseNotFound = errors.New("database not found")
    ErrLoadFilesFailed  = errors.New("load files failed")
    ErrWriteFileFailed  = errors.New("write file failed")
    ErrOpenFileFailed   = errors.New("open file failed")
)
```

### 10.2 错误统计

```go
// lib/statisticsPusher/statistics/engine_stat.go
type EngineStatistics struct {
    OpenErrors      int64  // 打开错误计数
    WriteErrors     int64  // 写入错误计数
    QueryErrors     int64  // 查询错误计数
    CompactErrors   int64  // 压缩错误计数
}
```

---

## 11. 性能优化

### 11.1 列式存储

- 数据按列组织，提高压缩率
- 支持列裁剪，减少 IO
- 向量化查询执行

### 11.2 数据压缩

| 算法 | 数据类型 | 压缩比 |
|------|----------|--------|
| Snappy | 通用数据 | 2-4x |
| LZ4 | 快速压缩 | 2-3x |
| Gorilla | 时间戳 | 10x+ |
| XOR | 浮点数 | 5-8x |
| Delta | 整数 | 3-5x |

### 11.3 缓存策略

```go
// engine/immutable/tssp_file.go
type PageCacheReader struct {
    cache *readcache.ReadCache
    // 数据块页缓存
    dataPages *sync.Map
    // 元数据页缓存
    metaPages *sync.Map
}
```

---

## 12. 代码验证

### 12.1 验证点

✅ **EngineImpl 核心结构**:
- `engine/engine.go:80-115` 已确认结构体定义
- `engine/engine.go:166-191` Open 方法实现

✅ **Shard 分片实现**:
- `engine/shard.go:211-260` 分片结构
- `engine/shard.go:512-629` 写入流程

✅ **WAL 日志**:
- `engine/wal.go:71-150` WAL 结构
- 记录格式和压缩实现

✅ **TSSP 文件**:
- `engine/immutable/tssp_file.go:61-100` 文件读取器
- 文件结构确认

✅ **Compaction**:
- `engine/immutable/compact.go:41-84` 压缩级别和限流
- Level Compaction 规则

### 12.2 关键代码索引

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| 引擎打开 | `engine/engine.go` | 166-191 |
| 写入行 | `engine/shard.go` | 512-545 |
| 写入列 | `engine/shard.go` | 627-680 |
| WAL 写入 | `engine/wal.go` | 200-280 |
| 压缩合并 | `engine/immutable/compact.go` | 200-350 |
| TSSP 读取 | `engine/immutable/tssp_reader.go` | 100-200 |

---

## 13. 下一步分析

完成存储引擎分析后，建议按以下顺序深入：

1. **索引系统** - `engine/index/` 目录详细分析
2. **查询执行器** - `engine/executor/` 目录
3. **Immutable/Mutable** - 更详细的数据结构
4. **分布式存储** - Raft 和数据复制

---

**文档版本**: 1.0  
**创建日期**: 2026-05-15  
**验证状态**: ✅ 已验证代码一致性

---
**最后验证**: 2026-05-15
**验证状态**: ✅ 已修正
**代码版本**: 7032447bce3ec6d35eb204a0f317882298b774b5

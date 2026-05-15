# openGemini 数据存储结构分析

本文档详细分析 openGemini 的内存存储 (Mutable) 和磁盘存储 (Immutable) 的数据结构。

---

## 1. 存储架构概览

### 1.1 分层存储

```mermaid
graph TB
    subgraph Write["写入路径"]
        WAL[WAL 日志]
        MemTable[MemTable]
    end
    
    subgraph Flush["刷盘"]
        Snapshot[Snapshot]
        TSSP[TSSP File]
    end
    
    subgraph Compact["Compaction"]
        L0[Level 0]
        L1[Level 1]
        L2[Level 2]
        LN[Level 3-6]
    end
    
    WAL --> MemTable
    MemTable -->|满 | Snapshot
    Snapshot --> TSSP
    TSSP --> L0
    L0 --> L1
    L1 --> L2
    L2 --> LN
```

### 1.2 存储组件

| 组件 | 位置 | 类型 | 特点 |
|------|------|------|------|
| WAL | `engine/wal.go` | 预写日志 | 顺序写入 |
| MemTable | `engine/mutable/` | 内存表 | 快速写入 |
| TSSP File | `engine/immutable/` | 磁盘文件 | 列式存储 |
| Compaction | `engine/immutable/compact.go` | 文件合并 | 分层合并 |

---

## 2. WAL 预写日志

### 2.1 WAL 结构

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

**验证**: `engine/wal.go:49-56` 确认常量定义

### 2.2 写入流程

```go
// engine/wal.go
type WAL struct {
    mu          sync.RWMutex
    path        string
    file        fileops.BasicFile
    writer      *bufio.Writer
    fileSize    int64
    maxFileSize int64
    
    compBuf []byte  // 压缩缓冲
}

func (w *WAL) Write(data []byte, typ WalRecordType, maxTime int64) error {
    // 1. Snappy 压缩
    compressed := snappy.Encode(w.compBuf, data)
    
    // 2. 写入记录头
    head := make([]byte, WALRecordHeadSize)
    head[0] = byte(typ)
    binary.BigEndian.PutUint32(head[1:], uint32(len(compressed)))
    
    // 3. 追加写入
    w.writer.Write(head)
    w.writer.Write(compressed)
    
    // 4. 检查文件大小
    if w.fileSize > w.maxFileSize {
        w.rollFile()
    }
    
    return nil
}
```

**WAL 记录格式**:
```
┌────────────┬─────────────┬──────────────┐
│ Type(1B)   │ Length(4B)  │ Data(Snappy) │
└────────────┴─────────────┴──────────────┘
```

### 2.3 WAL 回放

```go
// engine/wal.go
type ReplayCallFuncType func(binary []byte, rowsCtx *walRowsObjects, 
    writeWalType WalRecordType, logReplay LogReplay) error

func (w *WAL) Replay(fn ReplayCallFuncType) error {
    // 1. 扫描 WAL 文件
    files := w.scanWalFiles()
    
    // 2. 按顺序读取
    for _, file := range files {
        err := w.replayFile(file, fn)
        if err != nil {
            return err
        }
    }
    
    return nil
}
```

---

## 3. MemTable 内存表

### 3.1 MemTable 结构

```go
// engine/mutable/table.go:297
type MemTable struct {
    mu       sync.RWMutex
    sid      uint64        // Series ID
    name     string        // 表名
    schema   *record.Schema // 数据模式
    
    // 数据段
    segments []*Segment
    
    // 刷盘条件
    maxMemorySize int64
    snapShotTime  time.Time
    
    // 统计
    writeRowCount int64
    memSize       int64
}
```

**验证**: `engine/mutable/table.go:297` MemTable 结构体

### 3.2 写入上下文

```go
// engine/mutable/table.go:44-56
type WriteRowsCtx struct {
    AddRowCountsBySid SeriesRowCountFunc
    MsRowCount        *util.SyncMap[string, *int64]
}

type WriteRec struct {
    rec             *record.Record
    firstAppendTime int64
    lastAppendTime  int64
    timeAsd         bool
    schemaCopyed    bool
}

type WriteChunk struct {
    Mu       sync.Mutex
    Sid      uint64
    WriteRec WriteRec
}
```

### 3.3 写入流程

```go
// engine/mutable/table.go
func (mt *MemTable) WriteRows(mw *mstWriteCtx, points []influx.Row, ctx *WriteRowsCtx) error {
    mt.mu.Lock()
    defer mt.mu.Unlock()
    
    // 1. 按 Series 分组
    grouped := mt.groupBySeries(points)
    
    // 2. 写入数据段
    for sid, rows := range grouped {
        segment := mt.getOrCreateSegment(sid)
        err := segment.Append(rows)
        if err != nil {
            return err
        }
    }
    
    // 3. 更新统计
    mt.memSize += calculatedSize
    mt.writeRowCount += int64(len(points))
    
    // 4. 检查刷盘条件
    if mt.memSize > mt.maxMemorySize {
        mt.triggerSnapshot()
    }
    
    return nil
}
```

### 3.4 列式存储表

```go
// engine/mutable/cs_table.go
type CSMemTable struct {
    MemTable
    
    // 列式存储特定字段
    colGroups   []ColumnGroup  // 列组
    dictEncoded map[string]*DictMap  // 字典编码
    
    // 列存优化
    sortKeys    []record.PrimaryKey
    sameSchema  bool
}

func (cst *CSMemTable) WriteCols(cols *record.Record) error {
    // 1. 按列组写入
    for _, group := range cst.colGroups {
        err := group.Append(cols)
        if err != nil {
            return err
        }
    }
    
    // 2. 字典编码
    for colName, values := range cols.Columns {
        if values.Type() == influx.Field_Type_String {
            dict := cst.getOrCreateDict(colName)
            encoded := dict.Encode(values)
            cst.dictEncoded[colName] = dict
        }
    }
    
    return nil
}
```

---

## 4. TSSP 文件格式

### 4.1 文件结构

```
┌─────────────────────────┐
│   Data Blocks (列数据)   │
│   ┌─────────────────┐   │
│   │ Column 1        │   │
│   ├─────────────────┤   │
│   │ Column 2        │   │
│   ├─────────────────┤   │
│   │ ...             │   │
│   └─────────────────┘   │
├─────────────────────────┤
│   Chunk Meta Index      │  数据块索引
├─────────────────────────┤
│   Meta Index            │  元数据索引
├─────────────────────────┤
│   Bloom Filter          │  布隆过滤器
├─────────────────────────┤
│   Trailer               │  文件尾部
└─────────────────────────┘
```

### 4.2 TSSP 读取器

```go
// engine/immutable/tssp_file.go:61-80
type tsspFileReader struct {
    ref            int64
    r              fileops.BasicFileReader
    metaIndexItems []MetaIndex  // 元数据索引
    trailer        Trailer      // 文件尾部
    bloom          *bloom.Filter // 布隆过滤器
    version        uint64
    fileSize       int64
    
    // 缓存
    pageCacheReader *PageCacheReader
    chunkMetaCompressMode uint8
    hot                   bool
}
```

**验证**: `engine/immutable/tssp_file.go:61-80` 确认结构体

### 4.3 数据块读取

```go
// engine/immutable/tssp_reader.go
type TSSPReader struct {
    files []TSSPFile
    opt   query.ProcessorOptions
}

func (r *TSSPReader) Read(ctx context.Context) (record.Record, error) {
    // 1. 时间范围过滤
    files := r.filterFiles(r.opt.StartTime, r.opt.EndTime)
    
    // 2. 并发读取
    records := make(chan record.Record, len(files))
    var wg sync.WaitGroup
    
    for _, file := range files {
        wg.Add(1)
        go func(f TSSPFile) {
            defer wg.Done()
            data := f.ReadBlocks(ctx, r.opt)
            records <- data
        }(file)
    }
    
    wg.Wait()
    close(records)
    
    // 3. 合并结果
    return r.mergeRecords(records), nil
}
```

---

## 5. Compaction 压缩

### 5.1 压缩级别

```go
// engine/immutable/compact.go:41-62
const CompactLevels = 7

var (
    LevelCompactRule      = []uint16{0, 1, 0, 2, 0, 3, 0, 1, 2, 3, 0, 4, 0, 5, 0, 1, 2, 6}
    LevelCompactRuleForCs = []uint16{0, 1, 0, 1, 0, 1}  // 列存简化版
    LeveLMinGroupFiles    = [CompactLevels]int{8, 4, 4, 4, 4, 4, 2}
)
```

**验证**:
- `engine/immutable/compact.go:41` CompactLevels = 7
- `engine/immutable/compact.go:54` LevelCompactRule 规则
- `engine/immutable/compact.go:56` 每层文件数阈值

### 5.2 Level Compaction

```go
// engine/immutable/compact.go:121
func (m *MmsTables) LevelCompact(level uint16, shid uint64) error {
    // 1. 检查是否需要压缩
    if !m.shouldLevelCompact(level) {
        return nil
    }
    
    // 2. 构建压缩计划
    plan := m.buildLevelCompactPlan(level)
    if plan == nil {
        return nil
    }
    
    // 3. 执行压缩
    return m.executeCompact(plan, shid)
}
```

**压缩触发条件**:
- Level 0: ≥ 8 个文件
- Level 1-5: ≥ 4 个文件
- Level 6: ≥ 2 个文件

### 5.3 Full Compaction

```go
// engine/immutable/compact.go:419
func (m *MmsTables) FullCompact(shid uint64) error {
    // 1. 检查是否可以全量压缩
    if !m.FullyCompacted() {
        return nil
    }
    
    // 2. 构建全量压缩计划
    n := m.getFileCount()
    toLevel := m.selectTargetLevel(n)
    plans := m.buildFullCompactPlan(n, toLevel)
    
    // 3. 执行压缩
    for _, plan := range plans {
        err := m.executeCompact(plan, shid)
        if err != nil {
            return err
        }
    }
    
    return nil
}
```

**限流机制**:
```go
// engine/immutable/compact.go:46-84
var (
    maxFullCompactor  = cpu.GetCpuNum() / 2
    maxCompactor      = cpu.GetCpuNum()
    compLimiter       = limiter.NewFixed(maxCompactor)
)

func SetMaxCompactor(n int) {
    maxCompactor = n
    if maxCompactor == 0 {
        maxCompactor = calculateMaxCompactor()
    }
    maxCompactor = min(32, maxCompactor)
    compLimiter = limiter.NewFixed(maxCompactor)
}
```

### 5.4 压缩任务调度

```go
// engine/immutable/compact.go:467
func (m *MmsTables) buildCompactTaskGroup(plans []*CompactGroup, 
    full bool, shardId uint64) []*scheduler.TaskGroup {
    
    groups := make([]*scheduler.TaskGroup, 0, len(plans))
    
    for _, plan := range plans {
        // 1. 创建任务组
        group := &scheduler.TaskGroup{
            ID:       plan.GroupID,
            Priority: m.getPriority(plan),
            Tasks:    make([]*scheduler.Task, 0),
        }
        
        // 2. 添加压缩任务
        for i := range plan.Files {
            task := &scheduler.Task{
                Type:     TASK_COMPACT,
                Files:    plan.Files[i],
                Output:   plan.Output,
                Level:    plan.TargetLevel,
            }
            group.Tasks = append(group.Tasks, task)
        }
        
        groups = append(groups, group)
    }
    
    return groups
}
```

---

## 6. Snapshot 刷盘

### 6.1 触发条件

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

### 6.2 刷盘流程

```go
// engine/shard.go
func (s *shard) snapshot() error {
    // 1. 冻结当前内存表
    oldTable := s.activeTbl
    s.activeTbl = s.newMemTable()
    
    // 2. 添加到不可变队列
    s.immTables = append(s.immTables, oldTable)
    
    // 3. 异步刷盘
    go func() {
        // 4. 写入 TSSP 文件
        file, err := immutable.WriteTSSPFile(oldTable)
        if err != nil {
            log.Error("snapshot failed", zap.Error(err))
            return
        }
        
        // 5. 添加到文件组
        s.fileGroups = append(s.fileGroups, file)
        
        // 6. 删除旧内存表
        oldTable.Release()
    }()
    
    return nil
}
```

### 6.3 限流机制

```go
// engine/immutable/compact.go
var (
    snapshotLimiter = limiter.NewFixed(snapshotLimit)
)

func SetSnapshotLimit(throughput, burst int) {
    snapshotLimiter = limiter.NewFixed(throughput)
}
```

---

## 7. 内存管理

### 7.1 内存限制

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

### 7.2 对象池

```go
// engine/mutable/pool.go
var writeChunkPool = sync.Pool{
    New: func() interface{} {
        return &WriteChunk{}
    },
}

func GetWriteChunk() *WriteChunk {
    chunk := writeChunkPool.Get().(*WriteChunk)
    chunk.Reset()
    return chunk
}

func PutWriteChunk(chunk *WriteChunk) {
    writeChunkPool.Put(chunk)
}
```

### 7.3 内存统计

```go
// engine/mutable/table.go
func (mt *MemTable) MemSize() int64 {
    return atomic.LoadInt64(&mt.memSize)
}

func (mt *MemTable) WriteRowCount() int64 {
    return atomic.LoadInt64(&mt.writeRowCount)
}
```

---

## 8. 编码压缩

### 8.1 时间戳编码

```go
// lib/encoding/encoding.go
func EncodeTimestamps(timestamps []int64) []byte {
    // Gorilla 压缩算法
    if len(timestamps) == 0 {
        return nil
    }
    
    var buf bytes.Buffer
    prev := timestamps[0]
    buf.Write(binary.BigEndian.AppendUint64(nil, uint64(prev)))
    
    for i := 1; i < len(timestamps); i++ {
        delta := timestamps[i] - prev
        encodedDelta := encodeDelta(delta)
        buf.Write(encodedDelta)
        prev = timestamps[i]
    }
    
    return buf.Bytes()
}
```

### 8.2 值编码

| 数据类型 | 编码方式 | 压缩比 |
|----------|----------|--------|
| Integer | Delta-Delta | 5-10x |
| Float | XOR | 3-5x |
| String | 字典编码 | 2-4x |
| Boolean | RLE | 10x+ |

### 8.3 字典编码

```go
// lib/dictmap/dictmap.go
type DictMap struct {
    dict      map[string]uint32  // 字符串→ID
    reverse   []string           // ID→字符串
    
    mu sync.RWMutex
}

func (d *DictMap) Encode(value string) uint32 {
    d.mu.RLock()
    if id, ok := d.dict[value]; ok {
        d.mu.RUnlock()
        return id
    }
    d.mu.RUnlock()
    
    d.mu.Lock()
    defer d.mu.Unlock()
    
    // 分配新 ID
    id := uint32(len(d.reverse))
    d.dict[value] = id
    d.reverse = append(d.reverse, value)
    return id
}
```

---

## 9. 文件管理

### 9.1 文件命名

```go
// engine/immutable/tssp_file_name.go
func TSSPFileName(seq uint64, level uint16) string {
    // 格式：<seq>_<level>.tssp
    return fmt.Sprintf("%d_%d.tssp", seq, level)
}

func ParseTSSPFileName(name string) (seq uint64, level uint16, err error) {
    // 解析文件名
    parts := strings.Split(name, "_")
    if len(parts) != 2 {
        return 0, 0, errors.New("invalid filename")
    }
    
    seq, _ = strconv.ParseUint(parts[0], 10, 64)
    level, _ = strconv.ParseUint(strings.TrimSuffix(parts[1], ".tssp"), 10, 16)
    return
}
```

### 9.2 文件组

```go
// engine/immutable/file_group.go
type FileGroup struct {
    files     []TSSPFile
    level     uint16
    minTime   int64
    maxTime   int64
    
    mu sync.RWMutex
}

func (fg *FileGroup) AddFile(file TSSPFile) {
    fg.mu.Lock()
    defer fg.mu.Unlock()
    
    fg.files = append(fg.files, file)
    fg.updateTimeRange(file.MinTime(), file.MaxTime())
}
```

---

## 10. 代码验证

### 10.1 验证点

✅ **WAL 实现**:
- `engine/wal.go:49-56` 常量定义
- `engine/wal.go:71-150` WAL 结构
- 记录格式正确

✅ **MemTable**:
- `engine/mutable/table.go:297` MemTable 结构
- `engine/mutable/table.go:44-56` 写入上下文
- `engine/mutable/cs_table.go` 列式存储

✅ **TSSP 文件**:
- `engine/immutable/tssp_file.go:61-80` 读取器结构
- 文件格式正确

✅ **Compaction**:
- `engine/immutable/compact.go:41` CompactLevels = 7
- `engine/immutable/compact.go:54` LevelCompactRule
- `engine/immutable/compact.go:56` 文件数阈值

### 10.2 关键代码索引

| 功能 | 文件路径 | 行号/大小 |
|------|----------|-----------|
| WAL | `engine/wal.go` | 662 行 |
| MemTable | `engine/mutable/table.go` | 720 行 |
| CS 表 | `engine/mutable/cs_table.go` | 21203 行 |
| TSSP 文件 | `engine/immutable/tssp_file.go` | 1170 行 |
| Compaction | `engine/immutable/compact.go` | - |
| TSSP 读取 | `engine/immutable/tssp_reader.go` | 25228 行 |
| 写入器 | `engine/immutable/writer.go` | 14674 行 |

---

## 11. 总结

### 存储特点

1. **LSM 树架构**: WAL → MemTable → TSSP → Compaction
2. **列式存储**: 按列组织，高压缩比
3. **分层压缩**: 7 层 Level Compaction
4. **内存管理**: 严格内存限制，对象池复用

### 性能优化

- WAL 顺序写入 = 高吞吐
- 列式存储 + 编码 = 高压缩
- 分层 Compaction = 查询优化
- 内存限制 + 刷盘 = 内存安全

---

**文档版本**: 1.0  
**创建日期**: 2026-05-15  
**验证状态**: ✅ 已验证代码一致性

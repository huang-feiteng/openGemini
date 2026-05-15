# Engine Agent Instructions

## Component Overview

Core storage engine implementation - the heart of openGemini.

## Directory Structure

```
engine/
├── shard.go           # Shard management, WriteRows
├── wal.go             # Write-Ahead Log
├── mutable/           # MemTable implementation
├── immutable/         # TSM file management
├── index/             # Index systems (TSI, high cardinality)
├── compactor/         # Compaction engine
└── hyridql/           # Hybrid query execution (optional)
```

**Note**: Main query execution logic is in `coordinator/`, not in engine.

## Key Entry Points

| Operation | File | Line |
|-----------|------|------|
| WriteRows | `shard.go` | 512 |
| MemTable | `mutable/table.go` | 297 |
| WAL Write | `wal.go` | 248 |
| WAL writeBinary | `wal.go` | 218 |
| Index Create | `index/tsi/index.go` | - |
| Compaction | `compactor/compactor.go` | - |

## Write Path (Complete)

1. **WAL**: `wal.go:writeBinary(walRecord *walRecord) error`
   - Snappy compression
   - Binary format write
   
2. **MemTable**: `mutable/table.go`
   - In-memory storage
   - Series organization
   
3. **Flush**: When MemTable full
   - Convert to TSM format
   - Write to immutable directory

## Read Path (Complete)

1. **Query Executor**: Parses query
2. **Index Lookup**: TSI or high cardinality index
3. **TSM Read**: Read from disk
4. **Merge**: Combine results from multiple shards

## Index Systems

### TSI (Time Series Index)

- Tag-based indexing
- Efficient for low-medium cardinality
- Location: `engine/index/tsi/`

### High Cardinality Engine

- Optimized for high cardinality scenarios
- Reduces memory usage
- Location: `engine/index/hi`

## Compaction

- Merges small TSM files
- Removes deleted series
- Improves read performance
- Location: `engine/compactor/`

## Testing

```bash
# Run all engine tests
go test -tags "failpoint" -v ./engine/...

# Single package
go test -tags "failpoint" -v ./engine/shard

# Single test
go test -tags "failpoint" -run TestWriteRows ./engine/shard
```

## Related Documentation

- `../docs/analysis/02-storage-engine.md`
- `../docs/analysis/09-complete-write-path-deep-analysis.md`
- `../docs/analysis/10-complete-query-path-deep-analysis.md`
- `../docs/analysis/13-compaction-algorithm-deep-analysis.md`

## Important Conventions

- **Failpoints**: Required for testing (`-tags "failpoint"`)
- **Exclude from coverage**: `engine/index/hi` (lifted code)
- **CGO flags**: Required for building

## Common Issues

### Write Performance

- Check WAL writeBinary performance
- Monitor MemTable flush frequency
- Verify compaction schedule

### Read Performance

- Index hit rate
- TSM file count
- Cache hit rate

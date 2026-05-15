# ts-store Agent Instructions

## Component Overview

Storage engine with LSM-tree implementation for time series data.

## Entry Point

- **Main**: `app/ts-store/main.go`
- **Package**: `app/ts-store/storage`

## Key Responsibilities

- Data ingestion (write path)
- LSM-tree storage management
- Index management (TSI, high cardinality)
- Compaction and merging
- Query execution (read path)

## Running ts-store

```bash
# Build only ts-store
cd app/ts-store && go build -o ../../build/ts-store

# Run with config
./build/ts-store -config ../../config/openGemini.conf
```

## Testing

```bash
# Run store tests
go test -tags "failpoint" -v ./app/ts-store/...

# Single test
go test -tags "failpoint" -run TestWriteRows ./app/ts-store/storage
```

## Architecture Notes

- LSM-tree with MemTable + TSM files
- Write-Ahead Log (WAL) for durability
- TSI (Time Series Index) for tag indexing
- High cardinality storage engine support
- Snappy compression for data blocks

## Related Documentation

- `../../docs/analysis/02-storage-engine.md`
- `../../docs/analysis/03-index-system.md`
- `../../docs/analysis/05-data-storage.md`
- `../../docs/analysis/09-complete-write-path-deep-analysis.md`
- `../../docs/analysis/13-compaction-algorithm-deep-analysis.md`

## Write Path

1. WAL writeBinary
2. MemTable insert
3. MemTable flush → TSM
4. Compaction

## Read Path

1. Query parsing
2. Index lookup
3. TSM file read
4. Result aggregation

## Key Files

| File | Description |
|------|-------------|
| `engine/shard.go:512` | WriteRows entry point |
| `engine/mutable/table.go:297` | MemTable implementation |
| `engine/wal.go:218` | WAL writeBinary |
| `engine/index/tsi/` | Tag index implementation |

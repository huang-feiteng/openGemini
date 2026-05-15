# ts-meta Agent Instructions

## Component Overview

Metadata service for distributed consensus and cluster state management.

## Entry Point

- **Main**: `app/ts-meta/main.go`
- **Package**: `app/ts-meta/meta`

## Key Responsibilities

- Cluster metadata storage
- Raft consensus implementation
- Node discovery and health monitoring
- Shard allocation and rebalancing

## Running ts-meta

```bash
# Build only ts-meta
cd app/ts-meta && go build -o ../../build/ts-meta

# Run with config
./build/ts-meta -config ../../config/openGemini.conf
```

## Testing

```bash
# Run meta tests
go test -tags "failpoint" -v ./app/ts-meta/...

# Single test
go test -tags "failpoint" -run TestMetaService ./app/ts-meta/meta
```

## Architecture Notes

- Uses HashiCorp Raft for consensus
- Stores metadata in BoltDB
- Communicates with ts-store via memberlist
- Handles DDL operations (CREATE DATABASE, RETENTION POLICY, etc.)

## Related Documentation

- `../../docs/analysis/11-distributed-consensus-deep-analysis.md`
- `../../docs/analysis/01-architecture-overview.md`

## Common Operations

| Operation | Location |
|-----------|----------|
| Leader election | `meta/raft.go` |
| Shard assignment | `meta/shard_mapper.go` |
| Node heartbeat | `meta/node.go` |
| Metadata snapshot | `meta/snapshot.go` |

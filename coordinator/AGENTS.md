# Coordinator Agent Instructions

## Component Overview

Distributed coordination layer - bridges SQL and storage.

## Directory Structure

```
coordinator/
├── query_executor.go      # Query execution coordinator
├── shard_mapper.go        # Shard routing
├── points_writer.go       # Write routing
└── meta_client.go         # Metadata interaction
```

## Key Responsibilities

- Query routing to shards
- Write distribution across nodes
- Metadata interaction (DDL)
- Result aggregation

## Query Flow

1. **Receive Query** from ts-sql
2. **Map Shards** via ShardMapper
3. **Route Subqueries** to ts-store nodes
4. **Aggregate Results**
5. **Return** to ts-sql

## Write Flow

1. **Receive Points** from client
2. **Determine Shards** based on shard key
3. **Route** to appropriate ts-store nodes
4. **Confirm** write success

## Testing

```bash
# Run coordinator tests
go test -tags "failpoint" -v ./coordinator/...

# Single test
go test -tags "failpoint" -run TestQueryExecutor ./coordinator
```

## Related Documentation

- `../docs/analysis/04-query-executor.md`
- `../docs/analysis/10-complete-query-path-deep-analysis.md`

## Key Interfaces

| Interface | Purpose |
|-----------|---------|
| QueryExecutor | Execute distributed queries |
| ShardMapper | Map queries to shards |
| PointsWriter | Route writes to nodes |
| MetaClient | Interact with ts-meta |

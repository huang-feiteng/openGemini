# ts-sql Agent Instructions

## Component Overview

SQL parsing and query execution engine.

## Entry Point

- **Main**: `app/ts-sql/main.go`
- **Package**: `app/ts-sql/sql`

## Key Responsibilities

- SQL parsing (InfluxQL, PromQL)
- Query planning and optimization
- Distributed query execution
- Result aggregation

## Running ts-sql

```bash
# Build only ts-sql
cd app/ts-sql && go build -o ../../build/ts-sql

# Run with config
./build/ts-sql -config ../../config/openGemini.conf
```

## Testing

```bash
# Run SQL tests
go test -tags "failpoint" -v ./app/ts-sql/...

# Single test
go test -tags "failpoint" -run TestQueryExecutor ./app/ts-sql/sql
```

## Architecture Notes

- Parses InfluxQL and PromQL
- Rewrites queries for distributed execution
- Routes subqueries to ts-store nodes
- Implements streaming aggregation

## Related Documentation

- `../../docs/analysis/04-query-executor.md`
- `../../docs/analysis/10-complete-query-path-deep-analysis.md`

## Query Flow

1. Parse SQL → AST
2. Semantic analysis
3. Plan generation
4. Distributed routing
5. Result merge

## Key Types

| Type | Location |
|------|----------|
| QueryExecutor | `coordinator/query_executor.go` |
| StatementExecutor | `coordinator/statement_executor.go` |
| ShardMapper | `coordinator/shard_mapper.go` |

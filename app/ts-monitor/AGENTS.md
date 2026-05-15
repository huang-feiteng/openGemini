# ts-monitor Agent Instructions

## Component Overview

System monitoring and metrics collection service.

## Entry Point

- **Main**: `app/ts-monitor/main.go`
- **Package**: `app/ts-monitor/collector`

## Key Responsibilities

- System metrics collection (CPU, memory, disk, network)
- openGemini process metrics
- Prometheus-compatible metrics exposure
- Health check endpoints

## Running ts-monitor

```bash
# Build only ts-monitor
cd app/ts-monitor && go build -o ../../build/ts-monitor

# Run with config
./build/ts-monitor -config ../../config/openGemini.conf
```

## Testing

```bash
# Run monitor tests
go test -tags "failpoint" -v ./app/ts-monitor/...
```

## Architecture Notes

- Collects metrics from all components
- Exposes Prometheus `/metrics` endpoint
- Supports custom metric collectors
- Integrated with openGemini statistics system

## Related Documentation

- `../../docs/analysis/06-services-libraries.md`

## Key Metrics

- System: CPU, memory, disk I/O, network I/O
- Database: writes, queries, cache hit rate
- Engine: compaction, WAL, index operations

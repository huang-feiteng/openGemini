# Lib Agent Instructions

## Component Overview

Common utilities, configuration, and shared libraries.

## Directory Structure

```
lib/
├── config/              # Configuration parsing
├── logging/             # Logging utilities
├── net/                 # Network utilities
├── stat/                # Statistics collection
├── util/                # General utilities
└── util/lifted/         # Lifted code (VictoriaMetrics, InfluxDB)
```

## Key Modules

### Configuration

- **File**: `lib/config/`
- **Purpose**: Parse openGemini configuration files
- **Key Types**: `Config`, `Meta`, `Sql`, `Store`

### Logging

- **File**: `lib/logging/`
- **Purpose**: Unified logging interface
- **Backend**: zap + lumberjack (rotation)

### Statistics

- **File**: `lib/statisticsPusher/`
- **Purpose**: Metrics collection and push
- **Format**: Prometheus-compatible

### Lifted Code

- **VictoriaMetrics**: Storage optimizations
- **InfluxDB**: Protocol compatibility
- **HashiCorp**: Raft, memberlist
- **Location**: `lib/util/lifted/`

## Testing

```bash
# Note: lifted code excluded from tests
go test -tags "failpoint" -v ./lib/... | grep -v lifted
```

## Related Documentation

- `../docs/analysis/06-services-libraries.md`
- `../docs/analysis/07-application-config.md`

## Important Notes

- **Lifted code**: Do not modify unless fixing critical bugs
- **Config changes**: Update all related components
- **Logging**: Use unified interface, not direct zap calls

## Module Count

- **Total**: 69 modules (as of verification 2026-05-15)
- **Lifted**: Excluded from coverage stats

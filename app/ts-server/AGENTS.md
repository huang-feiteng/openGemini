# ts-server Agent Instructions

## Component Overview

Unified server entry point that combines ts-meta, ts-sql, and ts-store.

## Entry Point

- **Main**: `app/ts-server/main.go`

## Key Responsibilities

- Single-process deployment mode
- Component lifecycle management
- Configuration loading
- Service orchestration

## Running ts-server

```bash
# Build only ts-server
cd app/ts-server && go build -o ../../build/ts-server

# Run with config (standalone mode)
./build/ts-server -config ../../config/openGemini.conf
```

## Architecture Notes

- Embeds meta, sql, and store components
- Suitable for single-node or testing deployments
- Simplifies cluster setup for development

## Testing

```bash
# Build and run
cd app/ts-server && go build -o ../../build/ts-server
../../build/ts-server -config ../../config/openGemini.conf
```

## Related Documentation

- `../../docs/analysis/01-architecture-overview.md`
- `../../docs/analysis/07-application-config.md`

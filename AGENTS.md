# Agent Instructions for openGemini

## Quick Start

```bash
# Build all binaries
python build.py --clean

# Run tests
make gotest

# Run integration tests (requires cluster running)
make integration-test

# Static checks before PR
make static-check go-version-check style-check go-vet-check
```

## Build & Test Commands

| Task | Command |
|------|---------|
| Build binaries | `python build.py --clean` or `make go-build` |
| Run unit tests | `make gotest` |
| Run integration test | `make integration-test` |
| Static analysis | `make static-check` |
| Code formatting | `go fmt ./...` |
| Go vet | `make go-vet-check` |
| Style check | `make style-check` |
| Version check | `make go-version-check` |
| Generate code | `make go-generate` |

### Build Output

Binaries are placed in `./build/`:
- `ts-meta` - metadata service
- `ts-sql` - SQL query service
- `ts-store` - storage engine
- `ts-server` - unified server
- `ts-monitor` - monitoring service
- `ts-data` - data utility
- `ts-recover` - recovery utility

## Test Execution

### Single Test Package

```bash
go test -tags "failpoint" -failfast -short -v -count 1 -p 1 -timeout 10m ./engine/...
```

### Single Test Function

```bash
go test -tags "failpoint" -run TestFunctionName -v ./package/path
```

### Enable Failpoints

```bash
make failpoint-enable
# ... run tests ...
make failpoint-disable
```

## Architecture Overview

### Core Components

| Component | Path | Description |
|-----------|------|-------------|
| ts-meta | `app/ts-meta/` | Metadata management, distributed consensus |
| ts-sql | `app/ts-sql/` | SQL parsing and query execution |
| ts-store | `app/ts-store/` | Storage engine, LSM-tree implementation |
| ts-server | `app/ts-server/` | Unified server entry point |
| ts-monitor | `app/ts-monitor/` | System monitoring and metrics |

### Key Modules

| Module | Path | Description |
|--------|------|-------------|
| Engine | `engine/` | Storage engine, index systems, compaction |
| Coordinator | `coordinator/` | Request routing, distributed coordination |
| Services | `services/` | Background services (CQ, downsample) |
| Lib | `lib/` | Common utilities, config, logging |
| Tests | `tests/` | Integration and functional tests |

## Important Conventions

### Go Environment

- **Go version**: 1.24+ (see `go.mod`)
- **Module mode**: `GO111MODULE=on`
- **Sumdb disabled**: `GONOSUMDB=*`, `GOSUMDB=off`
- **Timezone**: `TZ=Asia/Shanghai`

### Code Style

- Run `go fmt ./...` before commit
- Use `goimports-reviser` for import organization: `make style-check`
- Follow conventional commits for PR titles
- Sign commits with `-s` flag (DCO requirement)

### Package Organization

- **Exclude from tests**: Packages containing `tests` or `lifted` are excluded from test coverage
- **License headers**: Required on all `.go` files (except lifted code)
- **Build tags**: Tests require `-tags "failpoint"` for failpoint support

## CI/CD Pipeline

Pre-merge checks (in order):
1. Commit lint (conventional commits)
2. License header check
3. Line lint (linelint)
4. Static check (`staticcheck`)
5. Go version check
6. Code generation check (`go generate`)
7. Style check (`goimports-reviser`)
8. Go vet check
9. Unit tests
10. Integration tests

Run full local CI simulation:
```bash
bash scripts/ci/go_vet.sh
make static-check
make style-check
make gotest
```

## Development Gotchas

### Compilation Flags

CGO flags are required for building:
```bash
export CGO_LDFLAGS="-Wl,-z,now -Wl,-z,relro -Wl,-z,noexecstack -fPIE -ftrapv"
export CGO_CFLAGS="-fstack-protector-strong -D_FORTIFY_SOURCE=2 -O2"
```

### Test Timeout

Default test timeout is 10 minutes. For long-running tests:
```bash
go test -timeout 30m ./...
```

### Failpoint Testing

Failpoints are compile-time enabled:
- Must use `-tags "failpoint"` when running tests
- Enable failpoints before running failpoint tests
- Always disable after testing

### Cross-Platform Builds

To verify cross-platform compatibility:
```bash
make build-check
# Builds for: linux/amd64, darwin/amd64, darwin/arm64, windows/amd64
```

## Cluster Deployment

### Install Cluster

```bash
bash scripts/install_cluster.sh
```

### Health Check Scripts

- Meta node: `scripts/ha_check_meta.sh`
- SQL node: `scripts/ha_check_sql.sh`
- Store node: `scripts/ha_check_store.sh`
- Monitor: `scripts/ha_monitor.sh`

## Version Management

Current release branch pattern: `v{MAJOR}.{MINOR}.{PATCH}-release_{FEATURE}`

Example: `v1.5.2-release_qwen3.6-plus`

**Code Version for Documentation**:
- Commit: `7032447bce3ec6d35eb204a0f317882298b774b5`
- Verified: 2026-05-15

## File Structure Quick Reference

```
openGemini/
├── app/           # Component entry points
├── engine/        # Storage engine core
├── coordinator/   # Distributed coordination
├── services/      # Background services
├── lib/           # Common libraries
├── tests/         # Integration tests
├── scripts/ci/    # CI scripts
├── config/        # Configuration files
└── docs/          # Documentation
```

## Documentation Analysis Files

Analysis documentation is located in `docs/analysis/`:
- `00-analysis-plan.md` - Analysis scope and plan
- `01-08-*-overview.md` - Architecture overviews
- `09-13-*-deep-analysis.md` - Deep dive analyses
- `README-analysis-summary.md` - Summary index
- `VERIFICATION-FINAL-REPORT.md` - Verification status
- `CORRECTION-LOG.md` - Correction history

All docs include code version (commit hash) and verification date in footer.

## External Dependencies

### Major Dependencies

- **VictoriaMetrics**: Lifted for storage optimizations
- **InfluxDB**: Lifted for protocol compatibility
- **HashiCorp**: Raft consensus, memberlist
- **Prometheus**: Metrics and monitoring
- **Arrow**: Columnar data format

### Protocol Buffers

Version 3.20 required for code generation:
```bash
make install-protoc
```

## Testing Prerequisites

### Integration Tests

Before running integration tests:
1. Start subscriber: `make start-subscriber`
2. Build binaries: `make go-build`
3. Install cluster: `bash scripts/install_cluster.sh`
4. Verify processes: `ps -ef | grep ts-`
5. Run tests: `make integration-test`
6. Cleanup: `make stop-subscriber`

### Test Coverage

Coverage reports are generated per test package:
- Files: `coverage_1.txt`, `coverage_2.txt`, etc.
- Uploaded to Codecov in CI

## Common Issues

### Go Module Download Errors

```bash
go mod download
# If fails, check GONOSUMDB and GOSUMDB settings
```

### Staticcheck Not Found

```bash
make install-staticcheck
```

### License Check Failures

```bash
make license-check
# Adds copyright headers to files missing them
```

### Code Formatting Issues

```bash
go fmt ./...
# Or use goimports-reviser
make style-check
```

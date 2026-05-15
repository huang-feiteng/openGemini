# Tests Agent Instructions

## Component Overview

Integration and functional tests for openGemini.

## Directory Structure

```
tests/
├── integration/           # Integration test cases
├── functional/            # Functional test cases
└── fixtures/              # Test data and configs
```

## Test Types

### Integration Tests

- Full cluster tests
- Requires running openGemini cluster
- Location: `tests/integration/`

### Functional Tests

- Feature-specific tests
- Can run in isolation
- Location: `tests/functional/`

## Running Tests

### Integration Test Suite

```bash
# Prerequisites
make start-subscriber
make go-build
bash scripts/install_cluster.sh

# Run tests
make integration-test

# Cleanup
make stop-subscriber
```

### Manual Test Execution

```bash
# Run integration tests
GOCACHE=off URL=http://127.0.0.1:8086 \
  go test -mod=mod -test.parallel 1 -timeout 10m ./tests -v

# Single test case
go test -run TestWriteQuery ./tests/integration
```

## Test Coverage

- Coverage reports: `coverage_*.txt`
- Uploaded to Codecov in CI
- Excluded packages: `tests`, `lifted`

## Related Documentation

- `../docs/analysis/08-code-review-report.md`

## CI Integration

Integration tests run in CI after:
1. Static checks pass
2. Unit tests pass
3. Cluster is deployed

## Common Issues

### Cluster Not Starting

Check process status:
```bash
ps -ef | grep ts-
```

Verify logs in `/tmp/openGemini/`

### Test Timeout

Default: 10 minutes
Increase if needed:
```bash
go test -timeout 30m ./tests
```

# Services Agent Instructions

## Component Overview

Background services running in openGemini.

## Directory Structure

```
services/
├── continuousquery/     # Continuous query service
├── downsample/          # Multi-level downsampling
└── subscriber/          # Data subscription service
```

## Services

### Continuous Query (CQ)

- Periodic query execution
- Automatic result storage
- Location: `services/continuousquery/`

### Downsample

- Multi-level data aggregation
- Configurable retention policies
- Location: `services/downsample/`

### Subscriber

- Real-time data streaming
- Kafka integration
- Location: `services/subscriber/`

## Enabling Services

Edit `config/openGemini.conf`:

```toml
[subscriber]
enabled = true
```

## Testing

```bash
# Run service tests
go test -tags "failpoint" -v ./services/...

# Single service
go test -tags "failpoint" -v ./services/continuousquery
```

## Related Documentation

- `../docs/analysis/06-services-libraries.md`

## Integration Test Prerequisites

Before running integration tests:

```bash
# Enable subscriber
make start-subscriber

# Build binaries
make go-build

# Install cluster
bash scripts/install_cluster.sh

# Run tests
make integration-test

# Cleanup
make stop-subscriber
```

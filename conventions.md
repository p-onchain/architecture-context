# Conventions & Shared Patterns

> Coding standards, naming conventions, and shared patterns across BlackSwan services.

## Language & Stack

- **Primary language:** Go (1.25+)
- **Exceptions:** onchain (TypeScript), heimdall (Rust), clients (TypeScript/Vue/React Native)
- **Standard toolkit:** `p-blackswan/corekit` — every Go service uses this
- **Observability:** `p-blackswan/observer` — shared OTLP + Pyroscope setup
- **Proto contracts:** `p-blackswan/proto-hub` — central protobuf definitions

## Project Structure (Go Services)

```
service-name/
├── .cd/
│   ├── docker/Dockerfile     # Standard Dockerfile
│   └── helm/                 # Helm chart (values per env: test/values.yaml, prod/values.yaml)
├── cmd/
│   └── main.go               # Entrypoint
├── internal/
│   ├── config/               # Env-based config (corekit/envcfg)
│   ├── domain/               # Business logic, entities
│   ├── application/          # Use cases, orchestration
│   └── adapters/             # gRPC handlers, Kafka consumers, DB repos
├── go.mod
├── go.sum
├── Makefile
├── lefthook.yml              # Git hooks (lint, test)
└── README.md
```

## Corekit Packages

Every Go service typically imports:

| Package | Usage |
|---------|-------|
| `corekit/envcfg` | Environment-based configuration |
| `corekit/database/postgres` | PostgreSQL connection + migrations |
| `corekit/database/redis` | Redis client setup |
| `corekit/messaging/kafka` | Kafka consumer/producer (confluent-kafka-go/v2) |
| `corekit/grpc` | gRPC server setup |
| `corekit/grpc/gclient` | gRPC client with interceptors |
| `corekit/grpc/interceptor` | Standard interceptors (logging, tracing, recovery) |
| `corekit/apperrors` | Standard error types |
| `corekit/xlogger` | Structured logging (zerolog) |
| `corekit/logctx` | Context-aware logging |
| `corekit/worker` | Background worker patterns |
| `corekit/xjobs` | Distributed job scheduling (pg/redis store) |
| `corekit/retry` | Retry with backoff |
| `corekit/httptransport` | HTTP server/client setup |
| `corekit/masking` | Sensitive data masking for logs |

## Naming Conventions

### Services
- Lowercase, hyphenated: `order-api`, `wallet-ledger-sink`, `commission-update-worker`
- CQRS pattern: `{domain}-projection`, `{domain}-query`, `{domain}-worker`
- BFF pattern: `bff-api` (web), `bff-client` (mobile)

### Kafka Topics
- Match events: `order.events.match.{market}` (e.g., `order.events.match.btc-try`)
- Status events: `order.events.status.{market}`
- Orderbook: `orderbook.state`, `orderbook.match_price`
- Ledger: `ledger-logs`, `external.ledger-logs`
- Domain events: `{domain}-events` (e.g., `user-commission-events`, `config-service-events`)

### Protobuf
- Package: `proto.{domain}.v1` (e.g., `proto.wallet.v1`, `proto.order.v1`)
- Go package: `github.com/p-blackswan/proto-hub/gen/go/{domain}/v1;{domain}_v1`
- Service names: `{Domain}Service` (e.g., `WalletService`, `OrderService`)
- Event names: `{entity}_{action}_event` (e.g., `crypto_deposit_created_event`)

### Git
- Commit format: `type: [JIRA-ID] description` (e.g., `feat: [EXCH-1234] add fast cancel`)
- Branch naming: `feat/EXCH-1234-short-description`, `fix/EXCH-5678-bug-description`
- Main branch: `main` (not `master`)

### Environment Variables
- Uppercase, underscore-separated
- Service addresses: `{SERVICE}_API_URL` (gRPC) or `{SERVICE}_ADDR` (e.g., `WALLET_SERVICE_ADDR=wallet:50058`)
- Database: `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_DATABASE`, `DB_SSLMODE`
- Kafka: `KAFKA_BROKERS`, `KAFKA_{TOPIC_NAME}_TOPIC`, `KAFKA_CONSUMER_GROUP`
- Redis: `REDIS_HOST`, `REDIS_PORT`, `REDIS_DB`, `REDIS_PASSWORD`, `REDIS_TLS_ENABLED`
- Observability: `OTEL_*` (standard OTEL env vars), `PYROSCOPE_SERVER_ADDRESS`

## Communication Patterns

### gRPC (Synchronous)
- All inter-service sync calls use gRPC with Protobuf
- Proto definitions in proto-hub, generated Go code imported as dependency
- Standard interceptors from corekit (logging, tracing, recovery, auth metadata)
- User ID propagated via gRPC metadata (originally from KrakenD X-User-Id header)

### Kafka (Asynchronous)
- Redpanda (Kafka-compatible) for internal messaging
- MSK for external bridge (legacy pikachu-exchange consumers)
- Protobuf serialization with Schema Registry
- Consumer groups follow pattern: `{service-name}-{topic-suffix}`
- redpanda-connect mirrors select topics from internal Redpanda to MSK

### Schema Registry
- All Kafka messages use Protobuf schemas registered in Redpanda Schema Registry
- `AUTO_REGISTER_SCHEMAS=false` in production (schemas pre-registered via CI)
- `USE_LATEST_VERSION=true` for forward compatibility

## Decimal Handling

- All financial values are **strings** in protobuf (no float/double)
- Use `fixedpoint` library (p-blackswan/fixedpoint) for arithmetic
- Never use floating-point for money calculations

## User IDs

- UUIDv7 format throughout the system
- Passed as strings in protobuf messages
- System accounts: user_id `42` (trade commissions), `43` (withdraw commissions)

## Observability

- **Tracing:** OpenTelemetry (OTLP) → SigNoz
- **Metrics:** OpenTelemetry (OTLP) → VictoriaMetrics → Grafana
- **Logs:** Structured JSON (zerolog) → SigNoz
- **Profiling:** Pyroscope (continuous profiling)
- Every service sets `OTEL_SERVICE_NAME` matching the deployment name
- Trace context propagated via gRPC interceptors and Kafka headers

## CI/CD

- GitHub Actions with shared workflows (p-blackswan/shared-workflows)
- Docker build → push to ECR
- Helm chart per service (in `.cd/helm/`)
- ArgoCD watches gitops repo (blackswan-gitops) for deployments
- Feature flags via flagd (OpenFeature), definitions in feature-flags repo

## Testing

- Unit tests: `go test ./...`
- Integration tests: Docker Compose (platform repo for full-system tests)
- Load tests: k6 (bswn-k6-suite)
- E2E automation: exchange-automation-tests (TypeScript, tests order→wallet→match flow)
- Linting: `golangci-lint`, pre-commit hooks via `lefthook`

## Helm / Deployment

- Each service has `.cd/helm/` with a Helm chart
- Values per environment: `test/values.yaml`, `staging/values.yaml`, `prod/values.yaml`
- blackswan-helm-base provides the shared base chart
- ArgoCD ApplicationSets in blackswan-gitops generate one Application per service
- Match engines use a special ApplicationSet that generates one Deployment per market YAML in market-configs

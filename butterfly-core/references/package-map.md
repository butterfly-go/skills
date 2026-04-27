# Butterfly Core Consumer Map

Use this reference when working in a service repository that imports `butterfly.orx.me/core`.

## Service-Facing Packages

- `app`: Primary bootstrap package. `app.Config` controls service name, config decoding, HTTP route registration, gRPC registration, and post-framework initialization hooks.
- `config`: Read raw configuration content through the active framework provider when the service needs it explicitly.
- `log`: Logging bootstrap and logger helpers based on `log/slog`.
- `mod`: Framework-owned config structs for core sections such as `store`, `log`, and `otel`.
- `store/redis`, `store/mongo`, `store/sqldb`, and `store/s3`: Access framework-managed clients from service code.
- `store/gorm`: Use `NewDB` when service code needs to build a GORM handle from a DSN. Do not rely on `GetDB` for initialized framework-managed state.
- `observe/otel`: Observability helpers exposed for service usage.
- `utils/httputils`: Shared transport helpers.

## What Butterfly Initializes For You

- Config provider selection from environment variables
- Service config decode into your custom config struct
- Core config decode into framework config
- Logging
- Metrics
- Tracing
- Store clients

Your service-specific dependency graph should be layered on top of that work in `InitFunc`.

## Configuration Shape

Expect the config source to contain both service-specific fields and Butterfly core sections:

- service-owned fields for business logic
- `store` for Redis, Mongo, DB, and S3
- `log` for level and format
- `otel` for tracing-related config

The framework reads config by service key or `namespace/service`.

## Debugging Checklist

- Wrong config loaded: check `Service`, `Namespace`, and the resolved config key first.
- Config struct does not compile: make sure the service config type implements Butterfly's `Print()` method.
- Store client missing: check the corresponding `store.*` YAML section before changing service code.
- Startup panic before app logic: inspect config source, logging, telemetry, and store initialization order.
- HTTP behavior mismatch: remember Butterfly owns Gin recovery, disables default Gin access logs, and injects OpenTelemetry middleware.

## When Not to Use This Skill

If the task is about modifying Butterfly internals such as `internal/config`, startup order, or store implementation details inside the framework repository, inspect `butterfly-go/core` directly instead of relying on this consumer-oriented skill.

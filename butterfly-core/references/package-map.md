# Butterfly Core Consumer Map

Use this reference when working in a service repository that imports `butterfly.orx.me/core`.

## Service-Facing Packages

- `app`: Primary bootstrap package. `app.Config` controls service name, config decoding, HTTP route registration, gRPC registration, and post-framework initialization hooks.
- `config`: Read raw configuration content through the active framework provider when the service needs it explicitly.
- `log`: Logging bootstrap and logger helpers based on `log/slog`. `log.Init` honors `level`, `format`, and `add_source`; `FromContext` currently returns the default slog logger.
- `mod`: Framework-owned config structs for core sections such as `store`, `log`, and `otel`.
- `store/redis`, `store/mongo`, `store/sqldb`, and `store/s3`: Access framework-managed clients from service code.
- `store/gorm`: Use `NewDB` when service code needs to build a MySQL GORM handle from a DSN. Do not rely on `GetDB` for initialized framework-managed state.
- `observe/otel`: Observability helpers exposed for service usage, including the Prometheus registry used by the framework metrics server.
- `utils/httputils`: Shared transport helpers, including `RegisterTwirpHandler` for mounting Twirp servers on Gin.

## What Butterfly Initializes For You

- Config provider selection from environment variables
- Service config decode into your custom config struct
- Core config decode into framework config
- Logging
- Metrics and `:2223/metrics`
- Tracing unless disabled
- Store clients for Redis, SQL, Mongo, and S3

Your service-specific dependency graph should be layered on top of that work in `InitFunc`.

## Configuration Shape

Expect the config source to contain both service-specific fields and Butterfly core sections:

- service-owned fields for business logic
- `store` for Redis, Mongo, DB, and S3
- `log` for level, format, and source-location behavior
- `otel` for tracing-related config

The framework reads config by service key or `namespace/service`.

Environment variables are derived by replacing `.` and `-` with `_`, uppercasing, and adding `BUTTERFLY_`. Current core uses these runtime keys most often:

- `BUTTERFLY_CONFIG_TYPE`
- `BUTTERFLY_CONFIG_FILE_PATH`
- `BUTTERFLY_CONFIG_CONSUL_ADDRESS`
- `BUTTERFLY_CONFIG_CONSUL_NAMESPACE`
- `BUTTERFLY_TRACING_ENDPOINT`
- `BUTTERFLY_TRACING_PROVIDER`
- `BUTTERFLY_TRACING_DISABLE`
- `BUTTERFLY_PROMETHEUS_PUSH_ENDPOINT`

## Store Notes

- `store.redis.<name>` creates a Redis client and pings it during startup.
- `store.mongo.<name>` creates a MongoDB v2 client from `uri`.
- `store.db.<name>` creates a `database/sql` handle. `postgres` and `postgresql` use the `pgx` driver; other driver values default to MySQL.
- `store.s3.<name>` creates an AWS SDK v2 S3 client and stores its configured bucket. `ak`/`sk` are accepted as shorthand fallback fields for `access_key_id`/`secret_access_key`.
- `store/gorm.NewDB` is separate from framework-managed `store.db` and currently opens MySQL with GORM tracing enabled.

## Debugging Checklist

- Wrong config loaded: check `Service`, `Namespace`, and the resolved config key first.
- Config struct does not compile: make sure the service config type implements Butterfly's `Print()` method.
- Store client missing: check the corresponding `store.*` YAML section before changing service code.
- Startup panic before app logic: inspect config source, logging, telemetry, and store initialization order.
- HTTP behavior mismatch: remember Butterfly owns Gin recovery, disables default Gin access logs, and injects OpenTelemetry middleware.
- Trace exporter errors: check `BUTTERFLY_TRACING_ENDPOINT`, `BUTTERFLY_TRACING_PROVIDER`, or temporarily set `BUTTERFLY_TRACING_DISABLE=true`.
- S3 endpoint mismatch: verify whether the endpoint includes a scheme and whether `use_ssl`/`use_path_style` match the provider.

## When Not to Use This Skill

If the task is about modifying Butterfly internals such as `internal/config`, startup order, or store implementation details inside the framework repository, inspect `butterfly-go/core` directly instead of relying on this consumer-oriented skill.

---
name: butterfly-core
description: Use when building or debugging a Go service that uses butterfly.orx.me/core. This skill covers service bootstrap with app.Config, framework-managed configuration and observability, store client wiring, and the common integration patterns a service repository should follow.
---

# Butterfly Core

## Overview

`butterfly.orx.me/core` is a service framework for Go microservices. Use this skill when a service repository needs help with:

- bootstrapping an HTTP or gRPC service with `app.Config`
- structuring service-specific config alongside Butterfly core config
- reading Redis, Mongo, SQL, or S3 clients from the framework
- configuring logging, Prometheus metrics, and OpenTelemetry tracing
- understanding what the framework initializes automatically before app code runs

If the task is about changing the framework itself, inspect the core repository separately. This skill is for service consumers first.

## Service Bootstrap

Start from `app.Config` and let the framework own process startup:

```go
type MyConfig struct {
    APIKey string `yaml:"api_key"`
}

func (c *MyConfig) Print() {}

cfg := &app.Config{
    Service: "user-service",
    Namespace: "prod",
    Config: &MyConfig{},
    Router: registerHTTPRoutes,
    GRPCRegister: registerGRPC,
    InitFunc: []func() error{
        initDomainDeps,
    },
}

app.New(cfg).Run()
```

Rules to preserve:

- `Service` is required in practice because it sets the runtime service name and the default config lookup key.
- `Namespace` changes the lookup key from `service` to `namespace/service`.
- `Config` must be a pointer to the service config struct and that type must satisfy Butterfly's `config.AppConfig` contract by implementing `Print()`.
- `Router` is optional. When present, Butterfly creates a Gin engine, installs recovery, disables default Gin access logs, and adds OpenTelemetry middleware.
- `GRPCRegister` is optional. When present, Butterfly starts a gRPC server on port `9090`.
- `InitFunc` is where service-owned dependency wiring belongs after framework config, logging, telemetry, and store setup complete.
- `TeardownFunc` exists on `app.Config`, but current core startup does not call it from `Run()`.

## Initialization Order

Butterfly runs a fixed startup sequence before your service logic:

1. Select config source
2. Load service config into `app.Config.Config`
3. Load core framework config
4. Initialize logging
5. Initialize metrics
6. Initialize tracing
7. Initialize stores
8. Run service `InitFunc`

Use that order when debugging startup failures. If a dependency needs config values or framework-owned clients, initialize it in `InitFunc`, not before `app.New(...).Run()`.

## Configuration Model

Two configuration layers are loaded from the same YAML source:

- service config: your custom struct passed through `app.Config.Config`
- core config: Butterfly-managed `store`, `log`, and `otel` sections

Keep that split clean:

- Put business settings like API keys, feature flags, and retry limits in the service config struct.
- Put Redis, Mongo, database, S3, log, and telemetry settings in the framework-owned YAML sections.
- Keep examples and generated config aligned with Butterfly's environment key conversion: keys are uppercased, `.` and `-` become `_`, and `BUTTERFLY_` is prepended. Common variables are `BUTTERFLY_CONFIG_TYPE`, `BUTTERFLY_CONFIG_FILE_PATH`, `BUTTERFLY_CONFIG_CONSUL_ADDRESS`, `BUTTERFLY_CONFIG_CONSUL_NAMESPACE`, `BUTTERFLY_TRACING_ENDPOINT`, `BUTTERFLY_TRACING_PROVIDER`, `BUTTERFLY_TRACING_DISABLE`, and `BUTTERFLY_PROMETHEUS_PUSH_ENDPOINT`.

For file config, Butterfly reads `BUTTERFLY_CONFIG_FILE_PATH` and ignores the config key. For Consul config, Butterfly reads KV by `Service` or `Namespace/Service`.

Core YAML shape:

```yaml
store:
  redis:
    cache:
      addr: localhost:6379
      password: ""
      db: 0
  mongo:
    primary:
      uri: mongodb://localhost:27017
  db:
    main:
      driver: postgres # postgres/postgresql use pgx; anything else uses mysql
      host: localhost
      port: 5432
      user: app
      password: secret
      db_name: app
      ssl_mode: disable
  s3:
    assets:
      endpoint: localhost:9000
      ak: minioadmin
      sk: minioadmin
      region: us-east-1
      bucket: assets
      use_ssl: false
      use_path_style: true
log:
  level: info
  format: text
  add_source: false
otel: {}
```

## Common Tasks

### Add a new service endpoint

1. Register HTTP routes in `Router` or gRPC services in `GRPCRegister`.
2. Put business logic in your service packages, not in the bootstrap file.
3. Let Butterfly keep ownership of server startup, middleware, and tracing wiring.

### Add a service dependency

1. Add service-specific config fields to your custom config struct and keep the struct implementing `Print()`.
2. Read framework-managed clients from the public packages under `store/` when the backing config already lives in Butterfly core config.
3. Prefer `store/sqldb.GetDB` for initialized SQL handles. Treat `store/gorm.NewDB` as a constructor for ad hoc GORM setup, not as a getter for framework-managed runtime state.
4. Build higher-level repositories or clients inside `InitFunc`.
5. When using S3-compatible storage, use `store/s3.GetClient(name)` and `store/s3.GetBucket(name)`. The config accepts `access_key_id`/`secret_access_key` or shorthand `ak`/`sk`; custom endpoints without a scheme get `http://` or `https://` from `use_ssl`.

### Debug startup or config issues

1. Verify the service name and optional namespace first because they control the config key.
2. Check whether the failure is in service config decode or framework config decode.
3. Confirm the expected YAML shape under `store`, `log`, and `otel`.
4. Check environment variables before changing application code.

## Store and Observability Usage

- Use public packages such as `store/redis`, `store/mongo`, `store/sqldb`, and `store/s3` from service code instead of reaching into framework internals.
- Use `store/gorm.NewDB` only when you need to build a new MySQL GORM handle from a DSN in service code; it installs the OpenTelemetry tracing plugin. Do not assume `store/gorm.GetDB` returns an initialized framework-managed connection.
- SQL config under `store.db` creates `database/sql` handles. `driver: postgres` or `driver: postgresql` uses `pgx`; other values default to MySQL DSNs.
- Assume the framework has already initialized logging and telemetry before `InitFunc`.
- Treat `log`, `config`, and `observe` packages as service-facing helpers, not extension points for framework rewrites.
- Metrics are served on `:2223/metrics`; use `observe/otel.PrometheusRegistry()` to register custom Prometheus collectors.
- Tracing defaults to OTLP gRPC unless `BUTTERFLY_TRACING_PROVIDER=http`; set `BUTTERFLY_TRACING_DISABLE=true` or `1` to skip tracing initialization.
- Logging is configured from `log.level`, `log.format`, and `log.add_source`. Unknown levels default to `info`; unknown formats default to text.

## Validation

- For service repository changes, run the service's tests and any startup smoke checks that exercise config loading and dependency initialization.
- When adding or changing Butterfly usage, verify the service still boots with realistic env vars or config fixtures.
- Re-read `references/package-map.md` when you need a quick map of the consumer-facing packages.

## References

- Consumer package map and usage notes: `references/package-map.md`

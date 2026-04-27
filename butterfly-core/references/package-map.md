# Butterfly Core Package Map

Use this reference when a task needs package-specific context without loading the whole repository into the main skill.

## Public Entry Points

- `app`: Main service bootstrap API. `app.Config` wires service identity, config loading, optional HTTP routes, optional gRPC registration, and user init/teardown hooks.
- `config`: Thin public wrapper around the active config provider.
- `log`: Public logging bootstrap and logger helpers built on `log/slog`.
- `mod`: Framework-owned configuration structs such as `CoreConfig`, store config, and log config.
- `store/gorm`, `store/mongo`, `store/redis`, `store/s3`, `store/sqldb`: Public store access packages backed by `internal/store`.
- `observe/otel`: Public observability helpers.
- `utils/httputils`: Shared transport helpers.

## Internal Packages

- `internal/arg`: Reads framework environment variables and normalizes argument names like `config.type`.
- `internal/config`: Selects the config backend, loads framework config, and initializes logging from config.
- `internal/log`: Internal logger naming helpers used during framework startup.
- `internal/observe/metric`: Metrics initialization.
- `internal/observe/tracing`: OpenTelemetry initialization.
- `internal/runtime`: Global service name and config-key state used across subsystems.
- `internal/store`: Shared store initialization and client registries.

## Behavioral Notes

- `(*app.App).Run()` is the canonical startup path. Changes to initialization order should be deliberate and tested.
- Framework config is loaded twice from the same config source: once into the app-specific config struct, then into `mod.CoreConfig`.
- Gin HTTP middleware is installed by the framework, including recovery and OpenTelemetry instrumentation.
- gRPC startup is optional and currently listens on port `9090`.
- Public packages are intentionally shallow; prefer putting implementation detail in `internal/*` unless consumers must call it directly.

## Typical Edit Targets

- New bootstrap option: `app/app.go` plus tests in `app/`.
- Config backend or config schema change: `internal/config/`, `mod/config.go`, and docs in `doc.md`.
- Store integration: matching files in `internal/store/` and public accessors under `store/`.
- Logging or telemetry: `log/`, `internal/log/`, `internal/observe/metric/`, or `internal/observe/tracing/`.

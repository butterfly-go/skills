---
name: butterfly-core
description: Use when working in the butterfly-go/core repository or when building or debugging services that depend on butterfly.orx.me/core. This skill covers the app bootstrap flow, configuration loading, storage and observability initialization, and the public package layout so changes match the framework's existing patterns.
---

# Butterfly Core

## Overview

`butterfly.orx.me/core` is a Go microservice framework. Most work falls into one of two paths:

- framework changes inside this repository
- service bootstrapping and integration changes in code that imports this repository

Start by locating the request in the package map under `references/package-map.md`, then follow the rules below before editing.

## Working Rules

- Treat `app.Config` in `app/app.go` as the public entrypoint. New framework behavior should usually hang off `app.Config`, the init chain, or a public package under `store/`, `config/`, `log/`, or `observe/`.
- Preserve the startup order in `(*App).Run()`. Core initialization currently runs as: config source, app config decode, core config decode, logging, metrics, tracing, storage, then user `InitFunc`.
- Keep public packages thin. The repository exposes wrappers like `config`, `log`, and `store/*` while most implementation lives under `internal/`.
- Assume configuration is YAML loaded by service key or namespace/service key. New config fields should be added to `mod.CoreConfig` or the service config struct passed through `app.Config.Config`.
- Follow the existing framework split: shared runtime state under `internal/runtime`, configuration providers under `internal/config`, stores under `internal/store`, and user-facing helpers under top-level packages.

## Common Tasks

### Add or change startup behavior

1. Inspect `app/app.go` and decide whether the behavior belongs in the core init chain, HTTP server setup, gRPC server setup, or user-supplied hooks.
2. If the feature is framework-owned and must run for every service, add it through the init chain or built-in server setup.
3. If services should opt in, expose it via `app.Config` instead of hard-coding it.
4. Update or add tests around the changed package. Prefer package-level tests near the affected code.

### Change configuration behavior

1. Check whether the value is framework-level or app-specific.
2. Framework-level settings belong in `mod.CoreConfig` and `internal/config`.
3. App-specific settings should remain in the consumer's `app.Config.Config` struct and be decoded in `InitAppConfig()`.
4. Keep environment-variable assumptions aligned with `doc.md` and the current `internal/arg` usage.

### Change storage or observability integrations

1. Update internal initialization first.
2. Keep public accessors in `store/*` or `observe/*` minimal and stable.
3. Make sure startup still succeeds when a subsystem is not configured, if that is the existing behavior for that subsystem.
4. Verify telemetry middleware or client instrumentation still attaches to the service name from `internal/runtime`.

## Service Integration Pattern

When creating an example or validating consumer code, use this shape:

```go
cfg := &app.Config{
    Service: "user-service",
    Config:  &MyConfig{},
    Router:  registerHTTPRoutes,
    GRPCRegister: registerGRPC,
    InitFunc: []func() error{
        initDomainDeps,
    },
}

app.New(cfg).Run()
```

Important constraints:

- `Service` is effectively required because it drives runtime service name and default config key.
- `Namespace` changes the config lookup key to `namespace/service`.
- `Config` must be a pointer compatible with YAML unmarshal and satisfy the framework's `Print()` contract if the consumer uses it.
- HTTP uses Gin with recovery, discarded default access logs, and OpenTelemetry middleware added by the framework.

## Validation

- Run targeted Go tests for touched packages first, then broader `go test ./...` if the change is wide enough.
- Re-read `doc.md` when altering config names, startup sequence, or public usage examples so the skill and docs stay aligned.
- If behavior changes a public package, confirm the top-level package still presents a small API and does not leak `internal` details.

## References

- Package map and edit guidance: `references/package-map.md`

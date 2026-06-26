# AGENTS.md

F5 NGINX Agent v3 — a Go application that manages NGINX instances remotely via gRPC (Management Plane Interface) and ships metrics/logs through an embedded OpenTelemetry collector.

- **Module:** `github.com/nginx/agent/v3` · Go 1.26 (toolchain 1.26.4)
- **Entrypoint:** `cmd/agent/main.go` → `internal.NewApp()` → `internal.App.Run()`
- **Config:** YAML (`nginx-agent.conf`), loaded via viper; runtime env vars prefixed `NGINX_AGENT_` (e.g. `NGINX_AGENT_COLLECTOR_CONFIG_PATH`)

## Common commands

```bash
make install-tools     # install golangci-lint, gofumpt, buf, counterfeiter, nfpm, etc. (run once)
make build             # build build/nginx-agent (PGO + ldflags inject version/commit/date)
make lint              # go vet ./... + golangci-lint v2 + buf generate (api/grpc)
make format            # gofumpt -l -w -extra .
make unit-test         # CGO_ENABLED=0, covers ./internal ./api ./cmd ./pkg
make race-condition-test   # CGO_ENABLED=1 -race, only ./internal ./api ./cmd (NOT ./pkg)
make generate          # weaver metadata gen + buf generate (api/grpc) + go generate ./...
make dev               # run agent locally with debug collector env vars
make run-mock-management-grpc-server   # mock management plane for local testing
```

Run a single package's tests: `go test ./internal/nginx/...`
Run a single test: `go test ./internal/nginx -run TestName -v`

## Pre-push hook (lefthook) — blocking, ordered

Pushing runs these in sequence and stops on first failure (`lefthook.yml`):
1. `make generate`
2. `make lint`
3. `make format`
4. `go mod tidy`
5. `make no-local-changes` — **fails if anything is uncommitted after steps 1–4**

Implication: after editing, always run `make generate && make lint && make format && go mod tidy`, then **commit the generated/formatted changes before pushing**. Generated diff left uncommitted will block the push.

## Required copyright header (goheader linter — hard fail)

Every `.go` file must start with exactly this header (the `goheader` linter enforces it):

```go
// Copyright (c) F5, Inc.
//
// This source code is licensed under the Apache License, Version 2.0 license found in the
// LICENSE file in the root directory of this source tree.
```

## Code generation — three independent generators

| What | Tool | Where | Trigger |
|---|---|---|---|
| Protobuf/gRPC stubs + validation + docs | `buf generate` | `api/grpc/` | `make generate` (runs `cd api/grpc && buf generate`) |
| Mocks (fakes) | `counterfeiter` v6.11.2 | `//go:generate` in interface files across `internal/`, `pkg/`, `api/grpc/mpi/` | `go generate ./...` |
| OTel receiver metadata | `weaver` (docker) → `mdatagen` | `internal/collector/{nginx,nginxplus,containermetrics}receiver/doc.go` | `make nginx-metadata-gen` / `make nginxplus-metadata-gen`, then `mdatagen` via `go generate` |

When you change a proto file, a Go interface with a `//go:generate counterfeiter` directive, or a receiver's `metadata.yaml`, regenerate with `make generate` and commit the output.

**Generated files are excluded from lint and coverage:** `*.pb.go`, `*.pb.validate.go`, `*gen.go`, `generated_*.go`, `*fakes*`. Do not hand-edit these.

## Linter rules that bite (golangci-lint v2, `.golangci.yml`)

- **importas:** `go.opentelemetry.io/otel/sdk/metric` MUST be imported as `metricSdk` (no-unaliased).
- **gci import order:** standard → default → `prefix(github.com/nginx/agent)` → blank → dot.
- **lll:** max line length 120 (tab-width 4).
- **exhaustive:** all switch statements on enums must be exhaustive.
- **gosec:** `//nosec` suppressions are disabled (`nosec: false`) — do not try to silence with nosec.
- **nolintlint:** any `//nolint` requires a specific linter name AND an explanation.
- **ireturn:** returning interfaces is restricted (allowed: `error`, `empty`, `stdlib`, grpc/testcontainers types). Avoid returning interfaces from public funcs.
- **interfacebloat:** max 10 methods per interface.
- **complexity:** cyclop ≤12, gocognit ≤20, nestif ≤5.
- **argument-limit:** ≤5 args; **function-result-limit:** ≤3 returns.
- **imports-blocklist:** `crypto/md5` and `crypto/sha1` are banned.
- **enforce-map-style:** use `make` (not `map{}` composite literals).
- **tagalign:** struct tags ordered `json, yaml, yml, toml, mapstructure, binding, validate`.
- Formatter is **gofumpt with extra rules** (`-extra`), module path `github.com/nginx/agent`. `gofmt` alone is not sufficient — always use `make format`.

## Testing conventions

- **Table-driven tests** with numbered case names: `"Test 1: ..."`, `"Test 2: ..."` (enforced by convention; see `CONTRIBUTING.md`).
- Unit tests run with `CGO_ENABLED=0`; race tests with `CGO_ENABLED=1 -race`. Race tests do **not** cover `./pkg/...` — if you add concurrency to `pkg/`, run `go test -race ./pkg/...` manually.
- Integration tests (`make integration-test`) require Docker and a pre-built OS package (`make local-deb-package` / `local-rpm-package` / `local-apk-package`). They spin containers per OS matrix. Not run in normal local dev.

## Architecture notes

- **Message bus + plugin pattern:** `internal.App.Run` loads config, creates `bus.NewMessagePipe`, registers `plugin.LoadPlugins(ctx, config)`, then runs the pipe. Plugins communicate over the message pipe (`internal/bus`), not via direct imports.
- **Collector integration:** `internal/collector` embeds an OpenTelemetry collector (`otelcol`) with custom receivers/processors (`nginxreceiver`, `nginxplusreceiver`, `containermetricsreceiver`, `logsgzipprocessor`, `securityviolations*`). These follow OTel component conventions; `internal/collector/factories.go` registers them.
- **gRPC MPI:** `api/grpc/mpi/` holds the protobuf Management Plane Interface. The agent connects to a management/command server; `test/mock/grpc/` is the reference mock server.
- **`pkg/`** is public reusable library code (`config`, `files`, `host`, `id`, `nginxprocess`, `tls`) — importable by external projects. **`internal/`** is private application code.

## Build quirks

- **PGO is on:** `default.pgo` is committed and passed via `-pgo=default.pgo` to `go build`. Do not delete it. Regenerate with `make generate-pgo-profile` (heavy, runs integration tests + benchmarks).
- **No workspace / no vendor:** `go.work`, `go.work.sum`, and `vendor/` are gitignored. Use plain `go mod` (no workspace).
- `make build` injects `main.version`, `main.commit`, `main.date` via `-ldflags`. `VERSION` defaults to `git describe --match "v[0-9]*"`.

## Commit / PR conventions

- Conventional Commits format preferred; present-tense imperative subject ≤72 chars.
- F5 CLA required for external contributors (bot prompts on PR).
- CODEOWNERS: `@nginx/nginx-agent` owns everything.

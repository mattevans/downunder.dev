# bad-tysm — control plane

> Repo: [ethpandaops/bad-tysm](https://github.com/ethpandaops/bad-tysm) · branch: `master`

## TL;DR

bad-tysm is a control plane and observability system for **tysm** fleets. tysm's hook system (observe / validate / mutate) injects behavior — blob corruption, peer sharding, telemetry forwarding — into Prysm. bad-tysm gives operators two integrated functions:

1. **observability** — ClickHouse-backed dashboards, paginated events, live SSE event stream, per-pod hook config snapshots.
2. **control plane** — register tysm endpoints, create TTL-bound hook activations, apply multi-node scenarios, audit every change in Postgres.

These two halves meet at the **audit layer**: when an activation ends, bad-tysm queries ClickHouse for events that fired during its live window and denormalises a summary onto the audit row.

One Go binary with an embedded React frontend.

| area | state |
|---|---|
| Postgres persistence | <span class="pill pill-done">done</span> · migrations live |
| Push-discovery from tysm | <span class="pill pill-done">done</span> |
| Docker label scan discovery | <span class="pill pill-done">done</span> · kurtosis-aware |
| Stale heartbeat reaper | <span class="pill pill-done">done</span> |
| Hook activations + audit | <span class="pill pill-done">done</span> |
| Scenarios (multi-node templates) | <span class="pill pill-done">done</span> |
| React frontend | <span class="pill pill-done">done</span> · 8 pages |
| OIDC auth | <span class="pill pill-done">done</span> · dex for local dev |

## walkthrough

Short tour of the bad-tysm UI driving a TYSM fleet — fleet view, registering / discovering nodes, creating an activation, watching events stream in from ClickHouse.

<div class="video-embed">
  <iframe src="https://www.youtube.com/embed/KTDmFp0U5ps" title="bad-tysm walkthrough" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
</div>

## HTTP API surface

Three flavours: observability (read ClickHouse), control (read/write Postgres + proxy tysm), discovery (the inbound side for tysm push-registration).

### observability — `/api/hooks/*`

OIDC-protected.

| method + path | purpose |
|---|---|
| `GET /api/hooks/namespaces` | distinct namespaces in the query window |
| `GET /api/hooks/pods` | distinct pods |
| `GET /api/hooks/nodes` | distinct node names |
| `GET /api/hooks/events` | cursor-paginated hook execution events |
| `GET /api/hooks/stream` | SSE live event stream |
| `GET /api/hooks/summary` | aggregated stats (totals, error rate, events/sec, per-hook) |
| `GET /api/hooks/timeline` | time-bucketed event counts for charting |

### control plane — `/api/control/*`

OIDC-protected. Proxies tysm for hook + activation operations.

| method + path | purpose |
|---|---|
| `GET / POST /api/control/nodes` | list / create |
| `GET / PATCH / DELETE /api/control/nodes/{id}` | single-node CRUD |
| `GET /api/control/nodes/{id}/info` | cached tysm `/info` |
| `GET /api/control/nodes/{id}/health` | last healthcheck result |
| `GET /api/control/nodes/{id}/hooks` | proxies `GET /tysm/v1/hooks` |
| `GET /api/control/nodes/{id}/hooks/{name}` | single hook detail |
| `POST /api/control/nodes/{id}/activations` | dispatch a new activation (creates pending audit row, calls tysm, updates row) |
| `GET /api/control/activations` | cross-fleet activation list (filters: nodeId, hook, status, ns, pod, time range, limit, offset) |
| `GET /api/control/activations/{id}` | one activation |
| `GET /api/control/activations/{id}/events` | ClickHouse events that fired during `[started_at, ended_at]` for this node's `(ns, pod)` matching this hook |
| `DELETE /api/control/activations/{id}` | cancel |
| `GET /api/control/config` | fleet-wide fanout of `GET /tysm/v1/hooks` to every node (returns partial results: `[{node, snapshot, error}, ...]`) |
| `GET / POST /api/control/scenarios` | list / create |
| `GET / PATCH / DELETE /api/control/scenarios/{id}` | CRUD |
| `POST /api/control/scenarios/{id}/apply` | apply to nodes (body `{nodeIds: [...]}`; returns `{created, errors: [{nodeId, error}]}`) |

### discovery — `/api/discovery/*`

**Bypasses OIDC.** Each request validates a shared bearer token from `discovery.sharedTokenRef` in `config.yaml` (typically a `${VAR}` env reference like `${BAD_TYSM_AUTH_REF_TOKEN}`; compared constant-time).

| method + path | purpose |
|---|---|
| `POST /api/discovery/register` | tysm announces itself on startup |
| `POST /api/discovery/heartbeat` | tysm bumps liveness (clears `last_error`, bumps timestamps) |
| `DELETE /api/discovery/register/{id}` | graceful shutdown — marks row stale (`last_error = "deregistered…"`), row not deleted |

### auth — `/api/auth/*`

| method + path | purpose |
|---|---|
| `GET /api/auth/me` | current user info |
| `GET /api/auth/oidc` | OIDC callback |
| `POST /api/auth/logout` | session logout |

## discovery + reaping

Two modes plus the reaper:

### push registration (`internal/discovery/discovery.go`)

- tysm pods POST to `/api/discovery/register` and heartbeat every ~30s.
- Shared bearer token configured via `discovery.sharedTokenRef` in `config.yaml` — typically `${BAD_TYSM_AUTH_REF_TOKEN}`, supplied at boot.
- Handler calls `nodes.UpsertDiscovered()` with `managed_by='discovery'`, `discovery_source='push'`.
- Refuses to overwrite rows owned by `ui` or `config` — returns 409 conflict when `(namespace, pod)` collides.

### docker label scan (`internal/discovery/docker.go`)

- Activated by `discovery.docker.enabled: true` in `config.yaml`.
- Background goroutine polls docker every 60s for containers matching `^ethpandaops/tysm(:|$)`.
- Reads kurtosis labels (`kurtosis_service_name`, `kurtosis_enclave_uuid`) to derive `(namespace, pod)` — **same key as ClickHouse**.
- In **host mode** (`discovery.docker.hostMode: true`), spawns `alpine/socat` sidecars (labelled `bad-tysm.tysm-forward=1`) that forward each container's API port to a stable host port starting at 18555. This is how kurtosis-on-docker setups get reachable URLs.
- Same upsert path: `managed_by='discovery'`, `discovery_source='docker'`.

### stale-heartbeat reaper (`internal/discovery/reaper.go`)

- Goroutine sweeps every 30s (default `StaleAfter` = 90s).
- Marks any discovery row with `last_heartbeat_at < now - StaleAfter` with `last_error = "stale: no heartbeat for 90s"`.
- **Rows are never deleted** — FK from activations would break, and the audit history is sacred. Stale rows are skipped by the healthcheck + reconciliation loops via `IsDiscoveryStale()`.

## build + run

### config.yaml

bad-tysm reads a single YAML config file pointed at by `BADTYSM_CONFIG_FILE`. Local dev config:

```yaml
listenAddr: ":8666"
postgresUrl: "postgres://badtysm:badtysm@localhost:5433/badtysm?sslmode=disable"
tokenEncryptionKey: "X+2eobs7kRbBDjIuZ816lE5kgOE7GhPiU8gjprMgezE="
clickhouseAddr: "localhost:9000"
clickhouseUser: ""
clickhousePass: ""
source: external
defaultNamespace: ""

oidc:
  enabled: false
  issuerUrl: ""
  clientId: ""
  clientSecret: ""
  scopes: []
  allowedGroups: []

discovery:
  sharedTokenRef: "${BAD_TYSM_AUTH_REF_TOKEN}"
  docker:
    enabled: true
    hostMode: true
```

Notes:
- `clickhouseAddr` is the **native TCP port** (`9000`), not the HTTP port (`8123`).
- `postgresUrl` points at the local Postgres brought up by `docker compose` (port `5433` to avoid clashing with a host Postgres on 5432).
- `tokenEncryptionKey` above is a throwaway dev key — generate your own with `openssl rand -base64 32` for anything non-local.
- `discovery.sharedTokenRef` uses `${VAR}` interpolation; the env var supplies the actual token at boot.
- `discovery.docker.hostMode: true` enables the `alpine/socat` sidecar mechanic that publishes each TYSM container's runtime API to a stable host port (`18555+`).

### local dev

```bash
# terminal 1 — Go server
BADTYSM_CONFIG_FILE=./config.yaml \
BAD_TYSM_AUTH_REF_TOKEN=dev \
go run ./cmd/badtysm

# terminal 2 — frontend dev server (Vite, proxies /api/* to :8666)
cd frontend && pnpm dev
```

Open <http://localhost:5173> for the UI (Vite). The Go server is on `:8666`; in production it serves the embedded `frontend/dist` directly from the same port.

Postgres needs to be running first — there's a `docker-compose.yml` in the repo for it, exposing port `5433`. Migrations apply on startup via goose.

### deps

- Go 1.26.1+
- PostgreSQL 16+ (via `docker-compose.yml`)
- ClickHouse (external; shipped by the log pipeline from tysm pods via xatu — see [kurtosis page](#/kurtosis))
- Node 22+ + pnpm for the frontend


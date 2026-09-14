# Fork-local Compose overrides + bring-up notes

Fork-local add-ons on top of `truefoundry/trueforge` `main`:

- `docker-compose.override.example.yml` — tracked example. Copy it to
  `docker-compose.override.yml` (gitignored, auto-merged by
  `docker compose up`) to run the stack with **host-networked
  Postgres/Redis**, so processes on the host (and other LAN harnesses) can
  reach them on `localhost:5432` / `localhost:6379`.
- This document: the minimum bring-up we verified end-to-end on this fork,
  plus the mapping open-swarm needs to treat TrueForge as a **remote
  harness** (`docs/REMOTE_HARNESSES.md` contract: health / list / send).

Nothing here is upstream; the tracked example + this doc are the only
fork-local files.

## Minimum `.env`

`packages/trueforge/.env` (gitignored; compose `env_file` requires it to
exist). The only hard-required value in distributed mode is
`TRUEFORGE_API_KEY`; Postgres credential defaults already match the app:

```bash
TRUEFORGE_API_KEY=change-me-to-a-real-random-value
POSTGRES_USER=trueforge
POSTGRES_PASSWORD=trueforge
POSTGRES_DB=trueforge
```

LLM inference is **not** env-configured: model providers live in Postgres and
are managed over the settings API (below), so no `OPENAI_*` style vars exist.

## Bring-up (verified)

```bash
cp packages/trueforge/.env.example packages/trueforge/.env   # then edit per above
cp docker-compose.override.example.yml docker-compose.override.yml
docker compose up --build --wait
# healthz + UI on http://localhost:8791
```

Under the override the server listens on host port **8791** (PORT inside the
container is set to 8791 so URLs match the base file's bridge-mode mapping
and `pnpm dev`'s 8790 stays free). Host-networked Postgres/Redis bind
`:5432`/`:6379` — stop `pnpm dev:infra` first; the two stacks collide.

## Model provider (OpenAI-compatible, LAN proxy)

Configure once via the settings API (no auth gate without OIDC on localhost):

```bash
curl -X PUT http://localhost:8791/api/v1/settings/model-providers \
  -H 'Content-Type: application/json' -d '{
  "manifest": {
    "type": "custom",
    "name": "orch",
    "base_url": "http://10.0.0.30:8000/v1",
    "auth": { "api_key": "sk-local-noauth" },
    "models": [
      { "model_id": "orchestration", "name": "orchestration",
        "properties": { "context_length": 131072, "max_output_tokens": 16384 } }
    ]
  }
}'
```

Notes learned the hard way:

- PUT takes `{ "manifest": ... }` only — the provider name lives *inside* the
  manifest for `type: "custom"`.
- Models are referenced elsewhere as FQN `orch/orchestration`
  (`provider-name/model-name`).
- The endpoint must support OpenAI SSE streaming; turn playback is streamed.

## Agent + session + turn (verified)

```bash
# Named, reusable agent (registry):
curl -X POST http://localhost:8791/api/v1/agents -H 'Content-Type: application/json' -d '{
  "name": "orchestrator",
  "description": "Showcase agent",
  "manifest": {
    "model": { "name": "orch/orchestration", "params": { "max_tokens": 8192 } },
    "instructions": "You are Orchestrator...",
    "config": { "iteration_limit": 40,
                "sandbox": { "enabled": false, "file_downloads": true },
                "dynamic_sub_agents": { "enabled": true } }
  }
}'

# Inline session (one-off spec) — or {"agent":{"name":"orchestrator"}}:
curl -X POST http://localhost:8791/api/v1/sessions -H 'Content-Type: application/json' -d '{
  "agent": { "spec": { "model": { "name": "orch/orchestration" } } }
}'

# Streamed turn (SSE) — input items are discriminated on "type":
curl -N -X POST "http://localhost:8791/api/v1/sessions/$SID/turns" \
  -H 'Content-Type: application/json' -d '{
  "input": [ { "type": "user.message", "content": "hello" } ],
  "stream": true
}'
```

Event stream observed on a healthy turn: `turn.created` →
`model.message.delta`×N → `model.message` → `turn.done`. Session create
accepts either `agent: { "name": ... }` (named registry agent) or
`agent: { "spec": {...} }` (inline one-off).

## open-swarm remote-harness mapping

open-swarm's `RemoteHarness` contract (health / list / send) maps onto
TrueForge's HTTP API with no code changes on either side:

| open-swarm verb | TrueForge call | Notes |
| --- | --- | --- |
| **health** | `GET /healthz` | `{"status":"ok","version":...}`; 200 = UP |
| **list** | `GET /api/v1/agents` (+ `GET /api/v1/sessions`) | agents = "listable workers"; paging via `limit`/cursor |
| **send** | `POST /api/v1/sessions` then `POST .../turns` (`stream: false`) | sessions are cheap; open-swarm caches `session_id` per conversation |

Send flow for open-swarm (public `/api/v1` only, verified end-to-end):

1. `POST /api/v1/sessions` with
   `agent: { "name": "orchestrator" }` (named agent) or `{ "spec": {...} }`
   (inline) and `metadata: { "swarm_conversation_id": ... }` → store the
   returned `session_id` keyed by the open-swarm conversation id.
   (A `/api/internal/sessions/get-or-create-by-external-id` route exists for
   schedule dispatch but is internal API — do not build on it.)
2. `POST /api/v1/sessions/{session_id}/turns` with
   `input: [{ "type": "user.message", "content": ... }]` and
   `stream: false` → returns the running turn immediately.
3. Poll `GET /api/v1/sessions/{session_id}/turns/{turn_id}` until
   `state` is `done` / `error` / `cancelled` (or subscribe to the SSE stream
   via `POST .../turns` with `stream: true`).
4. Read the final assistant message from
   `GET /api/v1/sessions/{session_id}/turns/{turn_id}/events`
   (last `model.message` event).

Auth for a LAN-exposed deployment: enable OIDC (then open-swarm holds a
bearer from the login flow) or keep the stack loopback-only and front it with
your existing gateway. Without OIDC the API has a fixed local admin identity —
never expose 8791 unauthenticated.

## Caveats

- `!reset []` (used to drop the base `ports:` under host networking) needs
  Compose v2.24+.
- The override intentionally does **not** set `POSTGRES_HOST`/`REDIS_URL`;
  base-file compose DNS names do not resolve under host networking.
- `pnpm smoke` still exercises the upstream bridge-mode path unchanged.

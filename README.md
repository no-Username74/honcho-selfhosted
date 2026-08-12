# Honcho Self-Hosted — Dokploy Stack (internal-only)

Prebuilt, hardened Honcho deployment for the Dokploy "Agent Tools" project.
No public ports, no domains — reachable only inside `dokploy-network` at
`http://honcho-api:8000`.

## Stack

| Service  | Image                                        | Reachable from           |
|----------|----------------------------------------------|--------------------------|
| api      | ghcr.io/plastic-labs/honcho@sha256:790e186a… | dokploy-network:8000     |
| deriver  | ghcr.io/plastic-labs/honcho@sha256:790e186a… | internal only            |
| database | pgvector/pgvector:pg15                       | internal only            |
| redis    | redis:8.2                                    | internal only            |

**Image pin:** `latest` pinned by multi-arch index digest. ghcr's semver tags
stop at `v2.0.3` (old v2 schema — no `docker/entrypoint.sh`, no
`MODEL_CONFIG__*` vars). To update the pin:

```bash
TOKEN=$(curl -s "https://ghcr.io/token?scope=repository:plastic-labs/honcho:pull" | python3 -c "import sys,json;print(json.load(sys.stdin)['token'])")
curl -sI -H "Authorization: Bearer $TOKEN" -H "Accept: application/vnd.oci.image.index.v1+json" \
  https://ghcr.io/v2/plastic-labs/honcho/manifests/latest | grep -i docker-content-digest
```

## Hardening vs upstream `docker-compose.yml.example`

- `AUTH_USE_AUTH=true` + 64-char hex `AUTH_JWT_SECRET` — every request needs a scoped JWT.
- Strong `POSTGRES_PASSWORD`; upstream's `POSTGRES_HOST_AUTH_METHOD=trust` removed.
- Zero `ports:` blocks (upstream binds 8000/5432/6379 to localhost).
- `api` is the only service on the external `dokploy-network` (alias `honcho-api`).
- Telemetry disabled.

## LLM routing (OpenRouter, OpenAI-compatible transport)

Single key (`OPENROUTER_API_KEY` → `LLM_OPENAI_API_KEY`) covers text + embeddings.

| Feature                    | Model                          | Why                |
|----------------------------|--------------------------------|--------------------|
| deriver / summary / dream  | google/gemini-2.5-flash-lite   | cheapest, tools    |
| dialectic minimal / low    | google/gemini-2.5-flash-lite   | cheapest, tools    |
| dialectic medium/high/max  | google/gemini-2.5-flash        | stronger reasoning |
| embeddings                 | openai/text-embedding-3-small  | 1536 dims, cheap   |

`openrouter/auto` was rejected: dynamic pricing, can route to expensive models.

## Dokploy setup

1. Dokploy project → "Agent Tools" environment → create compose service from this repo
   (`sourceType=git`, `customGitUrl=https://github.com/no-Username74/honcho-selfhosted`,
   branch `main`, `composePath=docker-compose.yml`).
2. Set env field (Dokploy → compose → Environment):

   ```
   POSTGRES_PASSWORD=<48-char hex, secrets.token_hex(24)>
   AUTH_JWT_SECRET=<64-char hex, secrets.token_hex(32)>
   OPENROUTER_API_KEY=sk-or-...
   ```

3. Deploy.

## Auth model (JWT claims — HS256, key = AUTH_JWT_SECRET)

Claims (all optional except scope rules):
- `ad: true` — admin (full access, can create workspaces/keys)
- `w` — workspace name
- `p` — peer name (requires `w`)
- `s` — session name (requires `w`)
- `exp` — ISO timestamp, optional

Mint tokens locally with PyJWT, e.g. workspace token:

```python
import jwt  # PyJWT
token = jwt.encode({"t": "", "w": "hermes-kiarash"},
                   AUTH_JWT_SECRET.encode(), algorithm="HS256")
```

## Workspaces

One workspace per agent (hard isolation). Naming: `hermes-<agent>` (e.g. `hermes-kiarash`).
Create via admin JWT:

```bash
curl -X POST http://honcho-api:8000/v3/workspaces \
  -H "Authorization: Bearer <admin-jwt>" -H "Content-Type: application/json" \
  -d '{"name": "hermes-kiarash"}'
```

## Verify

```bash
curl http://honcho-api:8000/health        # from any container on dokploy-network
curl http://honcho-api:8000/v3/workspaces # expect 401/403 without token
```

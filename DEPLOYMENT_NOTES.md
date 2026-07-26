# Deployment Notes (DigitalOcean droplet)

Personal deployment notes for this instance. Not part of upstream LibreChat docs —
kept here as a runbook for future me (or future agent) when something in this
stack breaks again.

## Stack

- Compose file: `deploy-compose.yml` (not the default `docker-compose.yml`)
- Reverse proxy: **Caddy** (`Caddyfile`, root of repo) — replaced the stock
  `nginx` `client` service, which is commented out in `deploy-compose.yml`.
  Caddy handles automatic HTTPS via Let's Encrypt.
- Domains: `lorcanium.net` → `api` (LibreChat), `admin.lorcanium.net` → `admin-panel`.
- `librechat.yaml` is gitignored — not tracked here, lives only on the droplet/local
  checkout. Custom endpoint (`OpenRouter`) and `modelSpecs`/`interface` config live there.
- Droplet: `ubuntu-s-2vcpu-2gb-nyc1` — **2GB RAM**, tight for this full stack
  (Mongo + Meilisearch + pgvector + rag_api + api + admin-panel + Caddy). Watch
  for OOM kills if containers disappear from `docker compose ps` unexpectedly
  (check `free -h` and `dmesg -T | grep -i oom`).

## Incidents & fixes (2026-07-26 Caddy migration)

Migrating from nginx → Caddy broke everything at once. Root causes, in the
order they were found — each one only became visible after fixing the
previous layer:

1. **Duplicate `restart:` key in the `caddy` compose service.** Made the whole
   `deploy-compose.yml` fail to parse (`docker compose config` caught it
   immediately). Nothing could start until this was fixed.

2. **Caddy on its own Docker network.** A custom `librechat-net` network was
   added and only `caddy` was placed on it; every other service (implicitly)
   stayed on Compose's `default` network. Caddy couldn't reach the API
   container at all. Fix: removed the custom network — everything now shares
   the implicit `default` network.

3. **Docker embedded DNS case-sensitivity bug.** `reverse_proxy LibreChat-API:3080`
   (mixed-case `container_name`) intermittently failed with
   `dial tcp: lookup LibreChat-API on 127.0.0.11:53: server misbehaving`. Fix:
   always address containers by their **lowercase Compose service name**
   (`api`, not `LibreChat-API`) in the Caddyfile — this is a known class of bug
   with Docker's embedded resolver (127.0.0.11) and strict Go DNS clients.

4. **Missing DNS record for `admin.lorcanium.net`.** Let's Encrypt's ACME
   challenge failed (`NXDOMAIN`) because the A record didn't exist yet. Not a
   Docker/Caddy issue — every hostname in the Caddyfile needs a real, public
   DNS record before Caddy can get a cert for it.

5. **`librechat.yaml` crash-looping the API — unrelated to Caddy.** A
   `modelSpecs` preset had `endpoint: "custom"` + `endpointType: "OpenRouter"`,
   but `endpointType` is a strict built-in enum
   (`azureOpenAI|openAI|google|anthropic|assistants|azureAssistants|agents|custom|bedrock`)
   — it can't hold a custom endpoint's name. Zod validation rejected the
   config on every boot, so `api` crash-looped every ~10s and never had
   anything listening on 3080. Caddy's `connection refused` was just a
   symptom — the real signal was in `docker compose logs api`, not Caddy's logs.

   **Lesson**: for a custom endpoint, `preset.endpoint` is the custom
   endpoint's own `name` directly (e.g. `endpoint: "OpenRouter"`). There is no
   separate `endpointType` needed for custom endpoints.

## Model selection / `modelSpecs` gotchas

- Defining any `modelSpecs.list` **hides** `modelSelect`, `parameters`, and
  `presets` in the UI unless explicitly re-enabled via an `interface:` block.
  Needed `interface: { modelSelect: true, parameters: true, presets: true }`
  to get the endpoint/model dropdown back.
- `modelSpecs.addedEndpoints` is **not additive** — once non-empty, it becomes
  an *allowlist* for every endpoint, built-in or custom
  (`client/src/hooks/Endpoint/useEndpoints.ts`). Adding `anthropic` to make it
  selectable silently hid the custom `OpenRouter` endpoint from the dropdown
  until `OpenRouter` was also added to the same list.
- A `modelSpec` with no `group` field shows as its own top-level dropdown
  entry, separate from the raw endpoint of the same name — confusing when
  they're both named "OpenRouter"/"Open Router". Setting
  `group: "OpenRouter"` on the spec nests it inside the `OpenRouter` endpoint's
  own submenu instead, alongside the full model list.
- A spec with `default: true` is a **hard admin default**
  (`client/src/utils/endpoints.ts:getDefaultModelSpec`) — it wins over
  `prioritize` and forces re-selection of that spec on every *new*
  conversation, regardless of other settings.
- `icon URL` for a modelSpec must point at an actual image file, not a
  webpage — a broken image load renders a red alert-circle badge on the spec
  icon (`client/src/components/Endpoints/URLIcon.tsx`).

## General troubleshooting method (useful beyond this incident)

1. `docker compose -f deploy-compose.yml config` before anything else — catches
   YAML/parse errors instantly.
2. Read the exact error message — "server misbehaving" (Docker DNS) vs
   "connection refused" (nothing listening) vs "NXDOMAIN" (public DNS/ACME)
   are three different failure domains, not the same bug.
3. Work outside-in: does the edge (Caddy) start → resolve the upstream →
   connect to the upstream → does the upstream itself run without crashing?
   Each question points to a different service's logs.
4. When a reverse proxy says "can't reach upstream," it doesn't know why —
   go read the actual backend service's own logs
   (`docker compose logs <service>`) next.
5. Small VPS + many containers = watch for OOM. `docker compose ps -a` +
   `free -h` + `dmesg -T | grep -i oom` if containers vanish.

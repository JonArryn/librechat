# Deployment Notes

## Stack

- `deploy-compose.yml` + Caddy (`Caddyfile`) — replaces nginx for automatic HTTPS.
- Domains: `lorcanium.net` → api, `admin.lorcanium.net` → admin-panel.
- Droplet: 2GB RAM — watch for OOM if containers vanish from `docker compose ps`.

## Deployment

**Problem:** Caddy couldn't reach the API container.
**Solution:** Don't put Caddy on its own custom Docker network — every service must share one network (Compose's implicit `default` is fine; just don't declare `networks:` at all).

**Problem:** Caddy failed to resolve `LibreChat-API` (`server misbehaving`).
**Solution:** Reverse-proxy to the lowercase Compose **service name** (`api`), never `container_name`. Docker's embedded DNS is unreliable with mixed-case names.

**Problem:** `docker compose` silently breaks after editing the compose file.
**Solution:** Run `docker compose -f deploy-compose.yml config` after any manual edit — catches YAML errors (e.g. duplicate keys) instantly.

**Problem:** Caddy cert request failed with `NXDOMAIN`.
**Solution:** Every hostname in the Caddyfile needs a real DNS A record pointing at the droplet before Caddy can issue a cert for it.

**Problem:** API crash-looped on boot; Caddy showed `connection refused`.
**Solution:** The proxy error is a symptom, not the cause — check `docker compose logs api` directly instead of Caddy's logs.

## librechat.yaml

**Problem:** Custom endpoint config rejected at boot (`ZodError` on `endpointType`).
**Solution:** Custom endpoints use `preset.endpoint: "<name>"` directly — no separate `endpointType` field.

**Problem:** Model/endpoint dropdown only showed one modelSpec, nothing else selectable.
**Solution:** `modelSpecs.list` hides `modelSelect`/`parameters`/`presets` unless re-enabled via `interface: { modelSelect: true, parameters: true, presets: true }`.

**Problem:** Adding `anthropic` to `addedEndpoints` made the custom `OpenRouter` endpoint disappear.
**Solution:** `modelSpecs.addedEndpoints` is an allowlist for *all* endpoints once set — list every endpoint (built-in and custom) you want visible.

**Problem:** Duplicate-looking "OpenRouter" entries in the model dropdown.
**Solution:** Set `group: "<endpoint name>"` on the modelSpec to nest it inside that endpoint's own submenu instead.

**Problem:** Red error badge on a model spec icon.
**Solution:** `iconURL` must point directly at an image file, not a webpage.

## Git workflow

- `origin` = `https://github.com/JonArryn/librechat.git` (my fork — push configs here).
- `upstream` = `https://github.com/danny-avila/LibreChat.git` (pull app updates only, never push).
- Pull config changes to droplet: `git pull origin main`.
- Pull app updates: `git fetch upstream && git merge upstream/main && git push origin main`.
- `librechat.yaml` is tracked here (unignored) because it only uses `${VAR}` env interpolation, no literal secrets. `.env` stays untracked — scp it manually.
- Keep `.env*` wildcarded in `.gitignore`, not just `.env`, so variants like `.env.local` stay ignored.

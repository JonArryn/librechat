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

**Problem:** Cloudflare 521 (origin down); `caddy` was missing entirely from `docker compose ps`, and `chat-mongodb`/`chat-meilisearch`/`LibreChat-API` were all crash-looping with permission errors (`EACCES`, WiredTiger metadata read failures).
**Solution:** A bare `docker compose up`/`down` (no `-f`) had been run after uploading a new `.env`. Without `-f`, Compose silently falls back to LibreChat's bundled `docker-compose.yml` + `docker-compose.override.yml` — a completely different stack that has no `caddy` service and runs `mongodb`/`meilisearch` as `user: "${UID}:${GID}"` (host user, e.g. `1000:1000`), which doesn't match the root/`999`-owned data directories the production stack expects. Always target the deploy file explicitly (`docker compose -f deploy-compose.yml ...`), or use the `npm run start:deployed` / `stop:deployed` / `rebuild:deployed` scripts, which hardcode `-f deploy-compose.yml` so this can't happen again.

## Cloudflare + firewall (only accept Cloudflare IPs)

**Problem:** After restricting the DO Cloud Firewall to Cloudflare's published IP ranges (ports 80/443), the site became unreachable in browsers (`ERR_CONNECTION_RESET`), even though the DO firewall rules and DNS proxy status (orange cloud) were both correct.
**Solution:** Check Cloudflare SSL/TLS mode first. Caddy here is configured with a **Cloudflare Origin CA certificate** (`tls /etc/ssl/cloudflare/origin.pem ...`), which is only used when Cloudflare connects to the origin over HTTPS. That requires SSL/TLS mode **Full** or **Full (strict)** — not Flexible. Flexible connects to the origin over plain HTTP, clashing with Caddy's automatic HTTP→HTTPS redirect and producing broken/reset connections that look like a firewall problem but aren't. Prefer **Full (strict)** since it actually validates the origin cert.

**Problem:** Caddy sees Cloudflare's edge IP as the client for every request (breaks real-IP-based logging/rate-limiting).
**Solution:** Add a global `trusted_proxies` option so Caddy unwraps the real visitor IP from `X-Forwarded-For`/`CF-Connecting-IP`. There is no built-in `cloudflare` source module — hardcode Cloudflare's current CIDR ranges (https://www.cloudflare.com/ips/) with `static`, and re-sync periodically since they change occasionally:
```
{
    servers {
        trusted_proxies static <cloudflare CIDR ranges...>
        trusted_proxies_strict
    }
}
```

**Problem:** After a Caddyfile/compose change, `docker compose exec caddy caddy reload` doesn't pick up a new **volume mount** added to `deploy-compose.yml` (e.g. mounting the origin cert directory).
**Solution:** `reload` only re-reads the Caddyfile inside the already-running container. A changed volume mount requires recreating the container: `docker compose -f deploy-compose.yml up -d caddy`.

**Problem:** Looked like the browser couldn't reach the site at all, unrelated to the above.
**Solution:** Turned out to be stale DNS caching at a layer between the browser and Cloudflare's public DNS (home router, and separately a VPN's DNS resolver) — both were serving a cached pre-Cloudflare A record pointing straight at the origin, which the firewall correctly rejected. Confirmed via `dig +short <domain> @1.1.1.1` (correct) vs. plain `dig +short <domain>` (stale/wrong) from the affected device. Fix: flush the OS DNS cache (`sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder` on macOS), or bypass the stale resolver entirely by setting the device/router DNS to `1.1.1.1`/`8.8.8.8`. Not a firewall or Caddy issue — don't waste time re-checking those once public-resolver DNS is confirmed correct.

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

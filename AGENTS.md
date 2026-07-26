See CLAUDE.md.

When adding or changing code that mutates user documents, invalidate the auth user document cache for affected users. This includes single-user updates and bulk role/user mutations; otherwise OpenID JWT request burst caching can serve a stale `req.user` until its TTL expires.

For this instance's droplet deployment (Caddy, `deploy-compose.yml`, `librechat.yaml` model/endpoint config), see DEPLOYMENT_NOTES.md before troubleshooting infra/deployment issues.

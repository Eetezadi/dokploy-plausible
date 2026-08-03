# dokploy-plausible

Deployment-only repo (no application source) that runs Plausible Analytics Community Edition on [Dokploy](https://dokploy.com). Dokploy's Compose service for this project is **git-linked**: it pulls `docker-compose.yml` and this whole repo tree fresh on every deploy.

## Layout

- `docker-compose.yml` — the three services: `plausible_db` (Postgres), `plausible_events_db` (ClickHouse), `plausible` (the app, `env_file: .env`).
- `clickhouse/*.xml` — bind-mounted ClickHouse config. Files under `config.d/` are server-level config; anything under `<profiles>` (per-query limits like `max_threads`, `max_memory_usage`) **must** be mounted into `users.d/`, not `config.d/` — ClickHouse silently ignores `<profiles>` blocks placed in `config.d/`. See `clickhouse/default-profile-low-resources-overrides.xml`.
- `.env.example` — vars a Dokploy deploy needs set in its UI (`BASE_URL`, `SECRET_KEY_BASE`, `TOTP_VAULT_KEY` required; SMTP/Google OAuth/geolocation optional).
- `.github/dependabot.yml` — auto-bumps `postgres`/`clickhouse/clickhouse-server`/`plausible` images, with major-version pins on the DB images.

## Version policy

- **Plausible CE image**: track upstream releases (`ghcr.io/plausible/community-edition`), but always read the release notes before bumping — some releases change files in this repo too (e.g. v3.2.0 changed the ClickHouse profile layout).
- **ClickHouse**: hold deliberately, upgrade one minor at a time with a backup first. **Data directories cannot be downgraded.** Dependabot is configured to skip ClickHouse minor/major bumps for this reason — don't let it auto-merge them.
- **Postgres**: pinned to major v18; Dependabot handles minor/patch bumps.
- Check what Plausible's own CI actually tests against (`plausible/analytics` repo, `.github/workflows/elixir.yml`) before assuming a newer DB version is safe — upstream's own example `compose.yml` in `plausible/community-edition` often lags behind what's actually validated.

## Dokploy specifics

- Compose service is git-linked — files checked into this repo (like `clickhouse/*.xml`) are pulled automatically on deploy. No need to duplicate their content into Dokploy's "Volumes" UI panel; that panel is only for content managed outside of git.
- Postgres password is hardcoded (`postgres`/`postgres`) and only reachable on the internal Docker network (no host port published). Changing `POSTGRES_PASSWORD` after first boot is a no-op without an in-database `ALTER USER` plus a matching `DATABASE_URL`.
- Domain routing is entirely delegated to Dokploy's Traefik integration via the UI (set a domain on port `8000`) — no Traefik labels are defined in the compose file itself.

## Workflow

- **Always work on a feature branch and open a PR** — never commit directly to `main`.

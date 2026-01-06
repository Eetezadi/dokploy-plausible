# Plan: Port Updates from Plausible Community Edition to Dokploy Implementation

## Executive Summary

Successfully cloned and analyzed the official Plausible community-edition repository at `/workspaces/dokploy-plausible/tmp/community-edition`. Identified key updates to port.

**Current State:**
- Your implementation: PostgreSQL 16-alpine, ClickHouse 25.11-alpine (main) / 25.8-alpine (v2)
- Upstream latest: PostgreSQL 16-alpine, ClickHouse 24.12-alpine (with pending updates to 18/25.11)
- Your folder structure: `files/clickhouse/`
- Upstream folder structure: `clickhouse/` (at root)

**Goal:** Modernize implementation with PostgreSQL 18, standardize folder structure, add health checks, and improve documentation.

---

## Implementation Overview

### Priority Order
1. PostgreSQL 18 upgrade + Dependabot pinning
2. ClickHouse config additions + folder restructure + Dependabot LTS pinning
3. Health checks
4. Documentation enhancements

### Excluded
- ❌ Section 6 (Let's Encrypt/Reverse Proxy) - Not needed for Dokploy

---

## Detailed Changes

### 1. PostgreSQL 18 Upgrade + Dependabot Configuration

**PostgreSQL: 16-alpine → 18-alpine**
- **File:** [docker-compose.yml](docker-compose.yml)
- **Change:** `image: postgres:16-alpine` → `image: postgres:18-alpine`
- **Impact:** Performance, security, better query optimization
- **Note:** Postgres 18 was released recently and is production-ready

**Dependabot: Pin to PostgreSQL 18**
- **File:** [.github/dependabot.yml](.github/dependabot.yml)
- **Current:** Monitors all docker images (no version constraints)
- **Change:** Add version constraints to keep postgres on v18.x
- **Why:** Prevent automatic upgrades to v19+ until tested

**Implementation:**
```yaml
# Add to dependabot.yml
version: 2
updates:
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
    assignees:
      - "Eetezadi"
    commit-message:
      prefix: "chore"
      include: "scope"
    labels:
      - "dependencies"
      - "docker"
    ignore:
      # Pin PostgreSQL to v18 major version
      - dependency-name: "postgres"
        update-types: ["version-update:semver-major"]
        versions: [">=19"]
```

### 2. ClickHouse Configuration Updates

**2a. Folder Restructure**
- **Current:** `files/clickhouse/clickhouse-config.xml` and `clickhouse-user-config.xml`
- **Target:** `clickhouse/` folder at root (matching upstream)
- **Actions:**
  1. Move `files/clickhouse/` → `clickhouse/`
  2. Delete old `files/` directory
  3. Update docker-compose.yml volume mounts from `./files/clickhouse/` → `./clickhouse/`

**2b. Add Missing Config Files from Upstream**

**Add: `clickhouse/ipv4-only.xml`** (from upstream)
```xml
<clickhouse>
    <listen_host>0.0.0.0</listen_host>
</clickhouse>
```
- **Why:** Prevents IPv6 warnings in Docker bridge networks
- **Impact:** Cleaner logs

**Add: `clickhouse/low-resources.xml`** (from upstream)
```xml
<!-- https://clickhouse.com/docs/en/operations/tips#using-less-than-16gb-of-ram -->
<clickhouse>
    <mark_cache_size>524288000</mark_cache_size>
    <profile>
        <default>
            <max_threads>1</max_threads>
            <max_block_size>8192</max_block_size>
            <max_download_threads>1</max_download_threads>
            <input_format_parallel_parsing>0</input_format_parallel_parsing>
            <output_format_parallel_formatting>0</output_format_parallel_formatting>
        </default>
    </profile>
</clickhouse>
```
- **Why:** Optimizes ClickHouse for systems with <16GB RAM
- **Impact:** Lower memory usage, better for small Dokploy deployments

**Update: `clickhouse/logs.xml`** (replace current clickhouse-config.xml)
- Use upstream version with 30-day TTL query logging
- Better balance between debugging visibility and disk usage

**Remove:**
- `clickhouse-config.xml` (replaced by logs.xml)
- `clickhouse-user-config.xml` (functionality merged into upstream configs)

**2c. Dependabot: Pin to ClickHouse LTS**
- **Current:** No version constraints
- **Target:** Stay on latest LTS releases only
- **Add to dependabot.yml:**
```yaml
ignore:
  # Pin ClickHouse to LTS releases (avoid interim versions)
  - dependency-name: "clickhouse/clickhouse-server"
    update-types: ["version-update:semver-minor"]
```
- **Note:** ClickHouse uses date-based versioning (YYYY.MM), LTS releases are recommended for production

### 3. Docker Compose Improvements

**3a. Add Plausible Application Volume**
- **Current:** No app volume
- **Add:**
```yaml
volumes:
  - plausible-data:/var/lib/plausible
```
- **Add to volumes section:** `plausible-data:`
- **Why:** Persistent temp files, prevents issues on container restart

**3b. Add Plausible ulimits**
- **Current:** Only ClickHouse has ulimits
- **Add to plausible service:**
```yaml
ulimits:
  nofile:
    soft: 65535
    hard: 65535
```
- **Why:** Prevents file descriptor exhaustion under load

**3c. Set TMPDIR Environment Variable**
- **Add to plausible environment:**
```yaml
environment:
  - TMPDIR=/var/lib/plausible/tmp
```
- **Why:** Use persistent volume for temp files instead of ephemeral container filesystem

### 4. Health Checks

**Add to plausible_db:**
```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U postgres"]
  start_period: 1m
```

**Add to plausible_events_db:**
```yaml
healthcheck:
  test: ["CMD-SHELL", "wget --no-verbose --tries=1 -O - http://127.0.0.1:8123/ping || exit 1"]
  start_period: 1m
```

**Update plausible depends_on:**
```yaml
depends_on:
  plausible_db:
    condition: service_healthy
  plausible_events_db:
    condition: service_healthy
```

**Impact:** Ensures databases are ready before Plausible starts, eliminates the manual 10-second sleep

### 5. Documentation Enhancements

**Create Comprehensive .env.example**
- **What:** Detailed example environment file
- **Current:** You have `.env.example` with basic variables
- **Upstream:** No .env.example, but README documents all variables
- **Recommendation:** Enhance your existing `.env.example` with:
  - SMTP/Email configuration options
  - Google OAuth variables
  - Registration control flags (DISABLE_REGISTRATION, ENABLE_EMAIL_VERIFICATION)
  - GeoIP configuration (MAXMIND_LICENSE_KEY, IP_GEOLOCATION_DB)
  - Erlang flags (ERL_FLAGS)
  - Port configuration (HTTP_PORT, HTTPS_PORT) for Let's Encrypt
- **Impact:** Better user experience for Dokploy users
- **Risk:** None - documentation only
- **Files to modify:** [.env.example](.env.example)

**Update README with New Features**
- **What:** Document additional configuration options
- **Current:** Your README focuses on Dokploy deployment
- **Upstream:** README covers many advanced features
- **Additions to consider:**
  - Email provider options (SMTP, Postmark, Mailgun, Sendgrid, Mandrill)
  - Google Analytics import capability
  - GeoIP/geolocation setup
  - Registration and email verification controls
  - Let's Encrypt automatic TLS
  - Resource requirements (2GB RAM minimum, SSE 4.2/NEON CPU)
- **Impact:** Users discover more features
- **Risk:** None - documentation only
- **Files to modify:** [README.md](README.md)

---

## Implementation Plan

### Step 1: PostgreSQL 18 Upgrade + Dependabot Pinning

**File:** [docker-compose.yml](docker-compose.yml)
- Change `postgres:16-alpine` → `postgres:18-alpine`

**File:** [.github/dependabot.yml](.github/dependabot.yml)
- Change `package-ecosystem: "docker-compose"` → `package-ecosystem: "docker"`
- Add `ignore` block to pin PostgreSQL to v18

```yaml
version: 2
updates:
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
    assignees:
      - "Eetezadi"
    commit-message:
      prefix: "chore"
      include: "scope"
    labels:
      - "dependencies"
      - "docker"
    ignore:
      # Pin PostgreSQL to v18 major version
      - dependency-name: "postgres"
        update-types: ["version-update:semver-major"]
        versions: [">=19"]
```

### Step 2: ClickHouse Configuration + Folder Restructure + Dependabot LTS

**2a. Move folder structure:**
- Rename `files/clickhouse/` → `clickhouse/`
- Delete empty `files/` directory

**2b. Copy files from upstream:**
- Copy `/workspaces/dokploy-plausible/tmp/community-edition/clickhouse/ipv4-only.xml` → `clickhouse/ipv4-only.xml`
- Copy `/workspaces/dokploy-plausible/tmp/community-edition/clickhouse/low-resources.xml` → `clickhouse/low-resources.xml`
- Copy `/workspaces/dokploy-plausible/tmp/community-edition/clickhouse/logs.xml` → `clickhouse/logs.xml`

**2c. Remove old files:**
- Delete `clickhouse/clickhouse-config.xml`
- Delete `clickhouse/clickhouse-user-config.xml`

**2d. Update docker-compose.yml:**
- Change all volume mounts from `./files/clickhouse/` → `./clickhouse/`
- Add new mounts for ipv4-only.xml and low-resources.xml
- Change clickhouse-config.xml mount to logs.xml

**Before:**
```yaml
volumes:
  - ./files/clickhouse/clickhouse-config.xml:/etc/clickhouse-server/config.d/logging.xml:ro
  - ./files/clickhouse/clickhouse-user-config.xml:/etc/clickhouse-server/users.d/logging.xml:ro
```

**After:**
```yaml
volumes:
  - ./clickhouse/logs.xml:/etc/clickhouse-server/config.d/logs.xml:ro
  - ./clickhouse/ipv4-only.xml:/etc/clickhouse-server/config.d/ipv4-only.xml:ro
  - ./clickhouse/low-resources.xml:/etc/clickhouse-server/config.d/low-resources.xml:ro
```

**2e. Update dependabot.yml:**
- Add ClickHouse LTS pinning to ignore block

```yaml
ignore:
  # Pin PostgreSQL to v18 major version
  - dependency-name: "postgres"
    update-types: ["version-update:semver-major"]
    versions: [">=19"]
  # Pin ClickHouse to LTS releases (stay on current LTS)
  - dependency-name: "clickhouse/clickhouse-server"
    update-types: ["version-update:semver-minor"]
```

### Step 3: Add Plausible Improvements to docker-compose.yml

**3a. Add volume to plausible service:**
```yaml
volumes:
  - plausible-data:/var/lib/plausible
```

**3b. Add ulimits to plausible service:**
```yaml
ulimits:
  nofile:
    soft: 65535
    hard: 65535
```

**3c. Add TMPDIR to plausible environment (via env_file .env):**
- Document in .env.example: `TMPDIR=/var/lib/plausible/tmp`

**3d. Add to volumes section:**
```yaml
volumes:
  db-data:
  event-data:
  event-logs:
  plausible-data:  # NEW
```

### Step 4: Add Health Checks to docker-compose.yml

**4a. Add to plausible_db service:**
```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U postgres"]
  start_period: 1m
```

**4b. Add to plausible_events_db service:**
```yaml
healthcheck:
  test: ["CMD-SHELL", "wget --no-verbose --tries=1 -O - http://127.0.0.1:8123/ping || exit 1"]
  start_period: 1m
```

**4c. Update plausible service depends_on:**
```yaml
depends_on:
  plausible_db:
    condition: service_healthy
  plausible_events_db:
    condition: service_healthy
```

**4d. Remove manual sleep from command:**
- Change: `sh -c "sleep 10 && /entrypoint.sh db createdb && /entrypoint.sh db migrate && /entrypoint.sh run"`
- To: `sh -c "/entrypoint.sh db createdb && /entrypoint.sh db migrate && /entrypoint.sh run"`

### Step 5: Documentation Enhancements

**5a. Update .env.example** - Add comprehensive documentation:
- ERL_FLAGS optimization (optional)
- TMPDIR=/var/lib/plausible/tmp
- All SMTP/email provider options
- Google OAuth variables
- GeoIP configuration
- Registration controls

**5b. Update README.md** - Add sections:
- Advanced features (email, OAuth, GeoIP)
- Performance optimization (ERL_FLAGS, TMPDIR)
- Resource requirements (2GB RAM, CPU requirements)
- Troubleshooting tips
- Link to PostgreSQL migration notes

---

## Files to Create/Modify

### Created
1. `clickhouse/ipv4-only.xml` - Copied from upstream
2. `clickhouse/low-resources.xml` - Copied from upstream
3. `clickhouse/logs.xml` - Copied from upstream

### Modified
1. [docker-compose.yml](docker-compose.yml) - PostgreSQL 18, folder paths, volumes, ulimits, health checks
2. [.github/dependabot.yml](.github/dependabot.yml) - Package ecosystem + version pinning
3. [.env.example](.env.example) - Comprehensive documentation
4. [README.md](README.md) - Advanced features documentation

### Deleted
1. `files/clickhouse/clickhouse-config.xml` - Replaced by logs.xml
2. `files/clickhouse/clickhouse-user-config.xml` - Functionality in upstream configs
3. `files/` directory - Empty after move

---

## ERL_FLAGS Explanation

**What is ERL_FLAGS=+sbwt none +sbwtdcpu none +sbwtdio none?**

Plausible is built with Elixir (runs on Erlang VM). By default, Erlang uses "busy-waiting" - scheduler threads spin in a loop checking for work instead of sleeping. This reduces latency but wastes CPU even when idle.

**The flags disable busy-waiting:**
- `+sbwt none` - Disables busy-waiting for schedulers
- `+sbwtdcpu none` - Disables for dirty CPU schedulers
- `+sbwtdio none` - Disables for dirty I/O schedulers

**Impact:** Idle CPU usage drops from 5-10% to <1%, adds only microseconds latency (negligible for analytics)

**Recommendation:** Document in .env.example as optional. Perfect for shared Dokploy hosting.

---

## Testing Notes

**Health Checks:** With health checks added, the manual 10-second sleep is no longer needed. Plausible will wait for actual database readiness.

**PostgreSQL 18:** Compatible upgrade path from 16. Existing data migrates automatically on first start.

**ClickHouse Config:** The new configs from upstream provide better logging balance (30-day TTL) vs. the current approach (no logging).

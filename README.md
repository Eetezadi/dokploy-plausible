# Plausible Analytics Deployment

Self-hosted Plausible Analytics deployment for Dokploy.

## Setup

1. Clone this repository
2. Copy `.env.example` to `.env`
3. Fill in your actual values in `.env` (get from existing Dokploy deployment)
4. In Dokploy, connect this repo to your existing Plausible app

## Important Notes

- **Never commit `.env` file** - it contains secrets
- `SECRET_KEY_BASE` and `TOTP_VAULT_KEY` must match your existing deployment
- Regenerating secrets will break access to encrypted data

## Deployment

### Connect to Dokploy

1. Go to your existing Plausible app in Dokploy
2. Navigate to **Source** tab
3. Select **Git Provider** → GitHub
4. Choose this repository
5. Select branch (e.g., `main`)
6. Deploy

### Domain Configuration

Configure domain in Dokploy UI:
- Service: `plausible`
- Port: `8000`
- Host: `pa.eetezadi.com`

### Backups

Volume backups in Dokploy cover:
- `db-data` - PostgreSQL database
- `event-data` - ClickHouse analytics data
- `event-logs` - ClickHouse logs

## Updates

To update Plausible version:
1. Edit `docker-compose.yml`
2. Change image tag: `ghcr.io/plausible/community-edition:v2.1.6`
3. Commit and push
4. Dokploy will redeploy automatically (if enabled)

## Structure
```
.
├── docker-compose.yml              # Main compose file
├── .env.example                    # Environment template
├── .env                           # Actual secrets (gitignored)
├── files/
│   └── clickhouse/                # ClickHouse configuration
│       ├── clickhouse-config.xml
│       └── clickhouse-user-config.xml
└── README.md
```
# Plausible Analytics for Dokploy

Self-hosted Plausible Community Edition deployment for Dokploy.

## Quick Start

### Fork This Repository

1. Fork this repository to your GitHub account
2. In Dokploy, create a new `Compose` service from your forked repository
3. In your service settings, add environment variables from `.env.example`. The `docker-compose.yml` automatically loads these variables
4. Set a domain on port `8000` 

## Configuration

### Required Variables

- **BASE_URL**: Domain where Plausible will be accessible (e.g., `https://analytics.example.com`)
- **SECRET_KEY_BASE**: Encryption key (generate with: `openssl rand -base64 48`)
- **TOTP_VAULT_KEY**: 2FA vault key (generate with: `openssl rand -base64 24`)

> ⚠️ **Important**: If migrating from an existing deployment, use your existing `SECRET_KEY_BASE` and `TOTP_VAULT_KEY` values. Regenerating these will break encrypted data and disable 2FA.

### Optional Variables

For additional features, see `.env.example` for configuration options:
- Email/SMTP (send notifications and password resets)
- Google OAuth (allow users to sign in with Google)
- Geolocation (GeoIP database setup)
- Performance tuning (Erlang VM optimization)

For comprehensive documentation on all configuration options, visit the [official Plausible Community Edition (CE)](https://github.com/plausible/community-edition/).

## Updates

To update the Plausible version:

1. Edit `docker-compose.yml`
2. Update the image tag: `ghcr.io/plausible/community-edition:vX.X.X`
3. Commit and push
4. Dokploy will redeploy automatically (if webhook enabled)
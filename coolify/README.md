# Coolify production deployment

This directory is the production deployment contract for Hapuppy. It uses the
official Sub2API release image and keeps PostgreSQL and Redis private to the
Coolify-managed Docker network.

## Coolify resource

- Repository: `hapuppy-ai/sub2api`
- Branch: `main` after the deployment pull request is merged
- Build pack: `Docker Compose`
- Compose location: `/coolify/docker-compose.yml`
- Public service: `sub2api`, internal port `8080`

Set the Sub2API domain in Coolify with the internal port suffix, for example
`https://ops.hapuppy.com:8080`. Coolify terminates public HTTPS on port 443.
Do not publish PostgreSQL or Redis domains or ports.

## Resource profile

The production Compose file is tuned for the current dedicated 4 vCPU / 8 GB
VPS. Container limits are ceilings rather than reservations:

- Sub2API: 2.5 vCPU / 3500 MB, 80 database connections, 512 Redis connections
- PostgreSQL: 1 vCPU / 2 GB
- Redis: 0.5 vCPU / 768 MB

The remaining memory is reserved for Ubuntu, Coolify, the reverse proxy, and
filesystem cache. Image generation has a separate eight-request concurrency
limit because large request and response bodies have a different memory profile
from text streaming. Re-run staged load tests before raising these limits or
moving the stack to a larger VPS.

## Environment

Set `ADMIN_EMAIL` in Coolify before the first deployment. Coolify generates the
remaining credentials from the magic variables declared in the Compose file:

- `SERVICE_PASSWORD_64_POSTGRES`
- `SERVICE_PASSWORD_64_REDIS`
- `SERVICE_PASSWORD_ADMIN`
- `SERVICE_HEX_64_JWT`
- `SERVICE_HEX_64_TOTP`

Never replace these with committed credentials. PostgreSQL credentials are
applied only when the data volume is first initialized.

## Release updates

`.github/workflows/update-sub2api-production.yml` checks the official upstream
release every six hours. A new stable version creates a pull request that only
changes the pinned Sub2API image. Merge that pull request to trigger Coolify's
Git-based deployment.

The production stack intentionally does not use `latest`. Before merging an
upgrade, read the upstream release notes and confirm that a current PostgreSQL
backup exists. After deployment, verify `/health`, admin login, channel status,
and a low-cost API request.

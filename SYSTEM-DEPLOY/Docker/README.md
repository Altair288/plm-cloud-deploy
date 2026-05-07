# Docker Compose Production Startup

## Included services

- `postgres`
- `redis`
- `flyway`
- `auth-service`
- `attribute-service`
- `gateway`
- `frontend`

`flyway` runs automatically on every full startup. On a brand-new PostgreSQL volume it will initialize the database schema from `plm-infrastructure/src/main/resources/db/migration`, and on subsequent starts it will only apply missing migrations.

## First-time setup

1. Copy `.env.example` to `.env` and replace all secrets.
2. Log in to GHCR before pulling images.

```bash
docker login ghcr.io
```

For private GHCR images, use a GitHub account or token with `read:packages`.

## Start the stack

Run these commands from `system-deploy/Docker`:

```bash
docker compose --env-file .env pull
docker compose --env-file .env up -d
```

## Stop the stack

```bash
docker compose --env-file .env down
```

To delete PostgreSQL and Redis persisted data as well:

```bash
docker compose --env-file .env down -v
```

## Exposed ports

- Frontend: `${FRONTEND_PORT}` default `3000`
- Gateway: `${GATEWAY_PORT}` default `8080`

`auth-service`, `attribute-service`, `postgres`, and `redis` stay on the internal Docker network by default.

## Important runtime parameters

These values are derived from the current service configuration:

- Auth service uses `PLM_DATASOURCE_*` and `PLM_AUTH_REDIS_*`
- Attribute service uses `PLM_DATASOURCE_*`
- Gateway overrides route URIs with Spring environment properties
- Frontend uses `NEXT_PUBLIC_API_BASE_URL=http://gateway:8080` so browser requests hit the frontend and are proxied internally to gateway

## Notes

- The compose file assumes this repository layout remains unchanged so Flyway can mount migrations from `../../plm-cloud-platform/plm-infrastructure/src/main/resources/db/migration`.
- If you publish a non-`latest` image, update `BACKEND_IMAGE_TAG` and `FRONTEND_IMAGE_TAG` in `.env` before running `docker compose pull`.
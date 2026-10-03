# Getting started

This guide shows how to run the project locally. Docker is the supported way to run it: the application expects the other services (`postgres`, `redis`, `jaeger`) to be reachable by their Docker Compose service names.

## Prerequisites

| Tool                                                  | Version     | Needed for                                        |
| ----------------------------------------------------- | ----------- | ------------------------------------------------- |
| [Docker](https://www.docker.com/) + Docker Compose v2 | recent      | running the stack                                 |
| `make`                                                | any         | the shortcuts in the `Makefile` (optional)        |
| [Node.js](https://nodejs.org)                         | 22.x        | running tooling outside Docker (optional)         |
| [pnpm](https://pnpm.io)                               | **10.34.6** | installing dependencies outside Docker (optional) |
| [Stripe CLI](https://docs.stripe.com/stripe-cli)      | recent      | testing webhooks locally (optional)               |

The pnpm version is pinned in the `packageManager` field of `package.json`. If you install dependencies outside Docker, run `corepack enable` so the right version is picked automatically. Newer pnpm versions (11+) block the Prisma install scripts and fail with `ERR_PNPM_IGNORED_BUILDS`.

## 1. Configure the environment

```bash
cp .env.example .env
```

Fill in the values. The ones you must change to boot the stack locally:

- every `MAPPED_PORT_*` variable, with free ports on your machine (for example `MAPPED_PORT_NGINX=8080`, `MAPPED_PORT_DB=5432`, `MAPPED_PORT_JAEGER_UI=16686`, `MAPPED_PORT_GRAFANA_UI=3001`);
- `EMAIL_PORT`, with a number (for example `587`).

Keep `PORT_API=3003`: Nginx, Prometheus and the OpenTelemetry collector point to `api:3003`.

All variables are described in [Configuration](configuration.md). `shell/check_env_vars.sh` runs before every `make` target and aborts if a required variable is missing.

## 2. Start the development environment

```bash
make run_development_docker
```

or, without `make`:

```bash
docker compose -f docker/composes/docker-compose.dev.yml --env-file .env up -d --build
```

When the `api` container starts it:

1. waits for Postgres to be healthy;
2. runs `prisma migrate dev`, which applies pending migrations and regenerates the Prisma Client;
3. starts Nest in watch mode (`pnpm run start:dev`).

Follow the logs with:

```bash
docker logs -f api-dev
```

The API is ready when you see `Nest application successfully started`.

## 3. Check that it works

| URL                                            | What it is                                |
| ---------------------------------------------- | ----------------------------------------- |
| `http://localhost:<MAPPED_PORT_NGINX>/`        | Landing page                              |
| `http://localhost:<MAPPED_PORT_NGINX>/docs`    | Swagger UI                                |
| `http://localhost:<MAPPED_PORT_NGINX>/metrics` | Prometheus metrics                        |
| `http://localhost:<MAPPED_PORT_JAEGER_UI>`     | Jaeger (traces)                           |
| `http://localhost:<MAPPED_PORT_GRAFANA_UI>`    | Grafana (default login `admin` / `admin`) |
| `localhost:<MAPPED_PORT_DB>`                   | PostgreSQL, for your database client      |

The API container is not published directly: every request goes through Nginx.

Quick smoke test:

```bash
curl "http://localhost:<MAPPED_PORT_NGINX>/products/list-many/single?page=1&pageSize=5"
# {"data":[],"total":0,"page":1,"pageSize":5,"totalPages":0}
```

To get an admin user to play with, run the seed (see [Database](database.md#seed)):

```bash
docker exec api-dev pnpm run seed
```

## Environments

|                     | Development              | Test                      | Production                      |
| ------------------- | ------------------------ | ------------------------- | ------------------------------- |
| Make target         | `run_development_docker` | `run_test_docker`         | `run_production_docker`         |
| Compose file        | `docker-compose.dev.yml` | `docker-compose.test.yml` | `docker-compose.prod.yml`       |
| Dockerfile          | `Dockerfile.dev`         | `Dockerfile.test`         | `Dockerfile.prod`               |
| Migrations on start | `prisma migrate dev`     | `prisma migrate dev`      | `prisma migrate deploy`         |
| Command             | `start:dev` (watch)      | `start:dev` (watch)       | `start:prod` (`node dist/main`) |
| Mounted folders     | `src`, `prisma`, `test`  | `src`, `prisma`, `test`   | none                            |
| Container names     | `*-dev`                  | `*-test`                  | `*-prod`                        |

`make run_test_docker` runs `docker compose down -v` first, so the test environment always starts with an empty database. See [Testing](testing.md) for how to run the e2e suite.

`shell/run-docker.sh` is an alternative entry point that starts the development or production stack based on the `ENVIRONMENT` variable.

> **Warning:** the three compose files live in the same folder, so Docker Compose gives them the same project name (`composes`). They share volumes and the image tag, which means `make run_test_docker` also **wipes the development database**. Until this is fixed, pass a project name when you need to keep data, for example `docker compose -p saas-dev -f docker/composes/docker-compose.dev.yml --env-file .env up -d`.

## Why only some folders are mounted

In development and test, only `src`, `prisma` and `test` are mounted into the container. The dependencies and the generated Prisma Client live in the image. Mounting the project root (as the compose files did before) hides the image's `node_modules`, and `npx prisma` then downloads the latest Prisma CLI, which is not compatible with this project.

If you change `package.json`, rebuild the image:

```bash
docker compose -f docker/composes/docker-compose.dev.yml --env-file .env up -d --build
```

## Common tasks

```bash
# Create a migration after editing prisma/schema.prisma
docker exec -it api-dev pnpm exec prisma migrate dev --name <migration_name>

# Open a shell in the API container
docker exec -it api-dev sh

# Check that the code compiles
docker exec api-dev pnpm run build

# Stop the stack (keeps the volumes)
docker compose -f docker/composes/docker-compose.dev.yml --env-file .env down

# Stop the stack and delete its volumes (database, redis, grafana, prometheus)
docker compose -f docker/composes/docker-compose.dev.yml --env-file .env down -v
```

## Stripe webhooks locally

To receive Stripe events on your machine, forward them to the webhook endpoint through Nginx:

```bash
stripe listen --forward-to http://localhost:<MAPPED_PORT_NGINX>/billing/webhook
```

Copy the `whsec_...` secret printed by the CLI into `STRIPE_WEBHOOK_SECRET` and restart the API. See [Billing](modules/billing.md) for the whole payment flow.

## Running without Docker

This is not supported yet. Redis (`redis:6379`) and the tracing exporter (`http://jaeger:4317`) are hard-coded in the source (see [Configuration](configuration.md#hard-coded-settings)), so the app only runs where those host names resolve.

## Windows notes

- **Line endings:** `.gitattributes` enforces LF endings. If an existing clone has CRLF line endings, run `git add --renormalize .` or convert shell scripts with `sed -i 's/\r$//' shell/*.sh`. See [Troubleshooting](troubleshooting.md).
- **Hot reload:** file changes made on a Windows folder reach the container, but the watcher inside it does not get notified, so Nest does not recompile. Cloning the repository inside the WSL 2 file system (for example `~/projects`) usually solves it; otherwise restart the container (`docker restart api-dev`) after your changes.

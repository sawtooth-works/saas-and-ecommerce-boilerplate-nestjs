# Troubleshooting

## Docker and Prisma

### `ERR_PNPM_IGNORED_BUILDS` during `pnpm install`

```text
Error: ERR_PNPM_IGNORED_BUILDS
  Ignored build scripts: @prisma/client@6.4.1, @prisma/engines@6.4.1, prisma@6.4.1, ...
```

A pnpm version newer than the one pinned in `package.json` is being used. pnpm 11+ refuses to install when dependency build scripts are blocked. Use pnpm **10.34.6**: `corepack enable` outside Docker, `npm install -g pnpm@10.34.6` in a Dockerfile. The allowed build scripts are listed in `pnpm.onlyBuiltDependencies` in `package.json`. Add a package there if it needs its install script.

### `P1012 The datasource property url is no longer supported in schema files`

```text
npm warn exec The following package was not found and will be installed: prisma@7.x
Error code: P1012
error: The datasource property `url` is no longer supported in schema files.
```

The command ran `npx prisma` without a local Prisma CLI, so npm downloaded Prisma 7, which is not compatible with this project. Common causes:

- the project root is mounted over the container's working directory, hiding `node_modules`. Mount only `src`, `prisma` and `test` (as the compose files do now) and rebuild the image;
- `prisma` was removed from the installed dependencies.

Always use `pnpm exec prisma ...`.

### `@prisma/client did not initialize yet. Please run "prisma generate"`

The Prisma Client was not generated in the `node_modules` the app is using. Rebuild the image (`docker compose ... up -d --build`) or run `docker exec api-dev pnpm exec prisma generate`. Do not reuse a `node_modules` folder installed on your host: it was generated for your OS, not for the container.

### `P1000 Authentication failed against database server`

Check `DATABASE_URL`. Older versions of `.env.example` had a typo (`{$POSTGRES_USER}` instead of `${POSTGRES_USER}`), which makes the user name `{generic_user}`. If you changed the Postgres credentials after the volume was created, the old ones are still in use: remove the volume with `docker compose ... down -v`.

### `P1001 Can't reach database server at postgres:5432`

- Inside Docker the host is `postgres`. From your machine, use `localhost:<MAPPED_PORT_DB>`.
- Check that the Postgres container is healthy: `docker ps --filter name=postgres`.

### My development database disappeared

`make run_test_docker` runs `docker compose down -v`, and the three compose files share the project name `composes`, so they share volumes. Use a project name (`docker compose -p <name> ...`) for the stack whose data you want to keep.

### `Bind for 0.0.0.0:<port> failed: port is already allocated`

Another process uses one of the `MAPPED_PORT_*` ports. Change it in `.env`.

## Shell scripts and line endings

### `make: ./shell/check_env_vars.sh: No such file or directory` or `$'\r': command not found`

The repository defines `.gitattributes` to ensure LF line endings across all platforms. If files in an existing clone were checked out with CRLF endings, renormalize them:

```bash
git add --renormalize .
```

Alternatively, convert the shell scripts directly:

```bash
sed -i 's/\r$//' shell/check_env_vars.sh shell/run-docker.sh
```

### Prettier reports every file

The same cause: with CRLF checkouts, Prettier (`endOfLine: lf` by default) flags every line. Running `git add --renormalize .` will normalize the line endings to LF.

## Application

### Nest does not recompile after I change a file (Windows)

File change events from a Windows folder do not reach the watcher inside the container. Clone the repository inside the WSL 2 file system, or restart the container (`docker restart api-dev`).

### E-mails are not delivered

1. Check the `EMAIL_*` variables.
2. Read the failure reason of the job:

   ```bash
   docker exec redis-dev redis-cli --scan --pattern 'bull:SEND_EMAIL_QUEUE:[0-9]*'
   docker exec redis-dev redis-cli hget bull:SEND_EMAIL_QUEUE:<id> failedReason
   ```

3. In the production image, `ENOENT ... templates/pt_br/...` means the templates were removed with `src/`. This is a known issue, see [Email](modules/email.md#known-limitations).

### `POST /auth/recovery-password` returns `500`

If the token is valid, this is a known bug: recovery only works when `SECRET_RECOVERY_PASSWORD_TOKEN_KEY` equals `SECRET_REFRESH_TOKEN_KEY`. See [Auth](modules/auth.md#known-limitations).

### The Stripe webhook returns `500`

- `No signatures found matching the expected signature`: `STRIPE_WEBHOOK_SECRET` does not match the endpoint. With the Stripe CLI, use the secret printed by `stripe listen`.
- `PrismaClientValidationError` after `checkout.session.completed`: purchases cannot be saved yet. See [Billing](modules/billing.md#what-is-missing).

### `pnpm run lint` fails

The error says that ESLint could not find an `eslint.config.(js|mjs|cjs)` file. The project uses ESLint 9 with a legacy `.eslintrc.js`, which ESLint 9 no longer reads by default. Until the config is migrated to `eslint.config.mjs`, you can force the legacy format:

```bash
docker exec -e ESLINT_USE_FLAT_CONFIG=false api-dev pnpm exec eslint "{src,test}/**/*.ts"
```

With LF line endings, this currently reports 10 existing errors, mostly unused imports. Do not add new ones.

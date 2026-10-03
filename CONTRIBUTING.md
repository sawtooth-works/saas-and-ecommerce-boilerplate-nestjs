# Contributing

Thank you for helping! This project is maintained by the [Sawtooth Works](https://github.com/sawtooth-works) community, and contributions of every size are welcome: bug fixes, features, tests, documentation and reviews.

Questions and ideas can be discussed on [Discord](https://discord.gg/W6sKEvXvtv) or in a GitHub issue.

## Where to start

- [Project status & roadmap](docs/project-status.md) lists what is unfinished and the known bugs.
- The [billing module](docs/modules/billing.md#suggested-plan-to-finish-the-flow) has a step-by-step plan to finish the payment flow, where help is most needed.
- Issues labeled [`good first issue`](https://github.com/sawtooth-works/saas-and-ecommerce-boilerplate-nestjs/labels/good%20first%20issue) and [`help wanted`](https://github.com/sawtooth-works/saas-and-ecommerce-boilerplate-nestjs/labels/help%20wanted) are a good entry point.

Before working on something that is not a small fix, open an issue (or comment on an existing one) describing what you plan to do. This avoids duplicated work and lets the maintainers confirm the approach.

## Setting up the project

Follow [Getting started](docs/getting-started.md). In short:

```bash
git clone https://github.com/<your-user>/saas-and-ecommerce-boilerplate-nestjs.git
cd saas-and-ecommerce-boilerplate-nestjs
cp .env.example .env   # fill in the ports and secrets
make run_development_docker
```

On Windows, line endings are automatically maintained as LF via `.gitattributes` (see [Troubleshooting](docs/troubleshooting.md#shell-scripts-and-line-endings)).

## Workflow

1. **Fork** the repository and clone your fork.
2. **Create a branch** from `main` named `<type>/<your-github-user>/<short-description>`, for example `fix/jane/purchase-relation-names` or `feat/jane/subscription-webhooks`.

   | Type       | Use for                                  |
   | ---------- | ---------------------------------------- |
   | `feat`     | a new feature                            |
   | `fix`      | a bug fix                                |
   | `refactor` | code changes that do not change behavior |
   | `test`     | tests only                               |
   | `docs`     | documentation only                       |
   | `devops`   | Docker, CI, infrastructure               |
   | `chore`    | dependencies, tooling, housekeeping      |

3. **Commit** using [Conventional Commits](https://www.conventionalcommits.org/): `<type>(optional scope): <description>`, in English and in the imperative mood.

   ```text
   fix(billing): use Prisma relation names when saving purchases
   feat(auth): add logout route that revokes the refresh token
   docs: explain how to run the e2e suite
   ```

4. **Check your change** (see the next section).
5. **Open a pull request** against `main` and fill in the description (see [Pull requests](#pull-requests)).

## Checking your change

Run these inside the development container:

```bash
# formatting
docker exec api-dev pnpm exec prettier --check "src/**/*.ts" "test/**/*.ts"
# fix formatting
docker exec api-dev pnpm run format

# type check and build
docker exec api-dev pnpm run build

# lint (legacy config, see the note below)
docker exec -e ESLINT_USE_FLAT_CONFIG=false api-dev pnpm exec eslint "{src,test}/**/*.ts"
```

Then run the e2e suite on a fresh database, as described in [Testing](docs/testing.md):

```bash
make run_test_docker
docker exec api-test pnpm run seed
docker exec api-test pnpm run test:e2e
```

`pnpm run lint` is currently broken because ESLint 9 does not read `.eslintrc.js` (see [Troubleshooting](docs/troubleshooting.md#pnpm-run-lint-fails)). The command above works around it and reports 10 existing errors: make sure your change does not add new ones. Migrating the config is a welcome contribution.

## Code conventions

The full description is in [Architecture](docs/architecture.md). The essentials:

- **Layers:** keep the `domain` / `application` / `infrastructure` / `interface` split of each module. Business rules live in use cases, not in controllers or gateways.
- **Dependency injection:** depend on interfaces. Register each implementation under a string token with the interface name (`'ISingleProductsRepository'`) and inject it with `@Inject('ISingleProductsRepository')`.
- **One use case per class**, with a single public `execute()` method.
- **Naming:** `snake_case` file names with a role suffix (`create_product.use_case.ts`), `PascalCase` classes, `I`-prefixed interfaces, `_camelCase` injected properties, `kebab-case` routes.
- **Language:** code, identifiers, comments, commit messages and documentation in English.
- **Validation:** every request body, query and param has a DTO with `class-validator` decorators and `@ApiProperty` for Swagger.
- **Access control:** routes are private by default. Use `@IsPublicRoute()` only when needed, `@Roles(...)` for role checks and CASL rules for per-field permissions.
- **Configuration:** new settings go to `.env.example`, `EnvService` and, if required, `shell/check_env_vars.sh`. Document them in [Configuration](docs/configuration.md).
- **Database:** change `prisma/schema.prisma` and create a named migration (`pnpm exec prisma migrate dev --name <name>`). Never edit a merged migration.
- **Style:** Prettier (`singleQuote`, `trailingComma: all`) is the source of truth for formatting.

## Tests

- Every bug fix and feature should come with e2e tests that cover the success path, validation errors and access control.
- New case files must be imported in `test/index.e2e-spec.ts` after the files that create the data they depend on.
- If you fix a behavior that a test documents as wrong (for example a `500` that should be `401`), update the test.

## Documentation

Update the docs in the same pull request when you change behavior, routes, configuration or setup:

- routes → [API reference](docs/api-reference.md) and the module page in `docs/modules/`;
- environment variables → [Configuration](docs/configuration.md);
- a fixed known issue → remove it from [Project status](docs/project-status.md).

## Pull requests

A good pull request:

- does one thing. Split unrelated changes;
- links the issue it solves (`Closes #123`);
- explains **what** changed and **why**, and lists anything reviewers should pay attention to;
- describes **how it was tested** (commands and results);
- keeps the build, formatting and e2e suite passing.

A maintainer reviews every pull request before it is merged. Be ready to iterate on feedback: reviews are about the code, never about the person.

## Reporting bugs

Open an issue with:

- what you did (request, command or steps);
- what you expected and what happened (status code, response body, logs);
- environment: OS, Docker version, which stack (dev/test/prod) and the commit you are on.

## Security

Do not open public issues for vulnerabilities that can be exploited. Contact the maintainers privately (for example on Discord) and give them time to fix it before disclosing.

## License

The project is licensed under the [MIT License](LICENSE). By contributing, you agree that your contributions are licensed under the same license.

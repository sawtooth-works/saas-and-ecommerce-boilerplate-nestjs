# Project status & roadmap

Last reviewed: September 2026.

Legend: ✅ done · 🚧 partial · ❌ not started · 🐞 known bug

## Feature status

| Area           | Feature                                                         | Status | Notes                                                           |
| -------------- | --------------------------------------------------------------- | ------ | --------------------------------------------------------------- |
| Auth           | Sign-up with password / password-less                           | ✅     |                                                                 |
| Auth           | Account verification by e-mail code                             | ✅     |                                                                 |
| Auth           | Sign-in with password, magic link, OTP                          | ✅     | OTP edge cases: [#20]                                           |
| Auth           | Refresh token                                                   | ✅     | No rotation or revocation: [#32]                                |
| Auth           | Password recovery                                               | 🐞     | Fails when the recovery and refresh secrets differ: [#14]       |
| Auth           | Logout / token revocation                                       | ❌     | [#32]                                                           |
| Auth           | Social login (OAuth)                                            | ❌     |                                                                 |
| Access control | RBAC (`@Roles`)                                                 | ✅     |                                                                 |
| Access control | ABAC (CASL)                                                     | 🚧     | Only `ADMIN` → `Products` rules exist                           |
| Users          | Profile routes (get/update me, change password, delete account) | ❌     | `UserModule` only has `findOneByPublicId`                       |
| Users          | Admin user management                                           | ❌     |                                                                 |
| Products       | CRUD for single and subscription products                       | ✅     |                                                                 |
| Products       | Stripe catalog sync                                             | ❌     | `price_id` is copied by hand                                    |
| Billing        | Stripe Checkout (one-time and subscription)                     | ✅     |                                                                 |
| Billing        | Webhook signature verification                                  | ✅     |                                                                 |
| Billing        | Save purchases after payment                                    | 🐞     | Prisma validation error, nothing is saved: [#13]                |
| Billing        | Payment confirmation e-mail                                     | ❌     | Event emitted, no listener: [#21]                               |
| Billing        | Subscription lifecycle (renewal, cancel, failure)               | ❌     | [#21]                                                           |
| Billing        | Purchase history / "my subscriptions" routes                    | ❌     | [#21]                                                           |
| Billing        | Entitlements (access based on purchases)                        | ❌     | [#21]                                                           |
| Billing        | Tests                                                           | ❌     | [#21]                                                           |
| E-mail         | Queue + templates (`pt_br`)                                     | ✅     |                                                                 |
| E-mail         | Works in the production image                                   | 🐞     | Templates are deleted with `src/`: [#15]                        |
| E-mail         | Other languages                                                 | ❌     |                                                                 |
| Observability  | Traces (OpenTelemetry → Jaeger)                                 | ✅     |                                                                 |
| Observability  | Metrics (Prometheus)                                            | 🚧     | Default metrics only; collector target only valid in dev: [#26] |
| Observability  | Grafana dashboards                                              | ❌     | Nothing provisioned                                             |
| Observability  | Health check endpoint                                           | ❌     |                                                                 |
| Infrastructure | Docker (dev, test, prod)                                        | ✅     | Prisma setup fixed in [#12]                                     |
| Infrastructure | CI pipeline                                                     | ❌     | [#27]                                                           |
| Infrastructure | Dev container                                                   | 🐞     | Points to a `docker-compose.yml` that does not exist: [#25]     |
| Tests          | e2e: auth, root, products                                       | ✅     | 64 tests                                                        |
| Tests          | Unit tests                                                      | ❌     | [#28]                                                           |
| Repository     | License                                                         | ✅     | [MIT](../LICENSE)                                               |

The billing module has its own detailed breakdown and a plan to finish it: [Billing — what is missing](modules/billing.md#what-is-missing), tracked in the epic [#21].

## Known issues

Confirmed problems, grouped by area, with the issue that tracks each one. Comment on the issue before starting, and follow the [contributing guide](../CONTRIBUTING.md).

### Billing

- Purchases are never saved: the repositories use relation names that do not exist in the Prisma schema, and the webhook answers `500`. [#13]
- Payment confirmation e-mail, subscription lifecycle, idempotency, payment history, Stripe customer id, purchase routes, entitlements and billing tests are missing. [#21]
- Unknown product in checkout and invalid webhook signature return `500`. [#19]

### Auth

- Password recovery verifies the token with the refresh secret instead of the recovery secret. [#14]
- Invalid or expired tokens return `500` instead of `401` (refresh and recovery), and so do verifying an account with no pending code and signing in with an OTP that was never requested. [#19]
- Requesting a second OTP before using the first fails, and an OTP validated from the cache can be used a second time. [#20]
- Cache TTLs are written in seconds but `cache-manager` expects milliseconds, and `/auth/sign-in-one-time-password` uses the wrong DTO. [#33]
- E-mail links are hard-coded to `http://example.com`. [#29]

### Security

- `POST /email/email-sender` is public: anyone can send e-mails through the configured SMTP account. [#17]
- `.env` is not in `.dockerignore`, so it is copied into the production image with every secret. [#16]
- No rate limiting, no security headers, user enumeration through `forgot-password`, refresh tokens that cannot be revoked, no logout. [#32]

### E-mail

- Templates are read from `src/` at runtime and the production image deletes `src/`, so every e-mail fails in production. Failed jobs are not retried or logged. [#15]
- Sender name hard-coded as `Seu Nome`. [#29]

### Infrastructure

- The dev, test and prod compose files share the project name `composes`: `make run_test_docker` deletes the development database, and the three environments overwrite the same image tag.
- The base image `node:22.14-bullseye-slim` runs on Debian 11, which is out of support: `apt-get` fails with `404`.
- `.devcontainer/devcontainer.json` references `../docker-compose.yml`, which does not exist.
- The `Makefile` calls `docker-compose`, which is missing on installations that only have the `docker compose` plugin.
- Redis host, tracing endpoint and CORS origins are hard-coded (see [Configuration](configuration.md#hard-coded-settings)).
- Hot reload does not work on Windows when the repository is on the Windows file system.

### Repository and tooling

- `pnpm run lint` fails: ESLint 9 does not read the legacy `.eslintrc.js`. [#22]
- No CI: lint, build and tests are not run on pull requests. [#27]
- No unit tests, so `pnpm test` fails with `No tests found`. [#28]
- `Injectable();` typo, `package.json` still named `auth-boilerplate-nestjs`. [#33]

### Products

- A `USER` trying to write gets `401` instead of `403`, deleting a product deletes its purchase history, and a duplicate `slug`/`price_id` on update returns `500`. [#33]

## Roadmap ideas

Not planned or prioritized yet. Open a discussion or an issue before starting one of these:

- **Finish billing** following the [suggested plan](modules/billing.md#suggested-plan-to-finish-the-flow) ([#21]).
- **SaaS building blocks:** organizations/tenants, members and invitations, plans and feature entitlements.
- **E-commerce building blocks:** cart, orders, stock, coupons, shipping and taxes.
- **User self-service:** profile, password change, account deletion and data export (LGPD/GDPR).
- **Operations:** `/health` endpoint, CI pipeline, provisioned Grafana dashboards, structured logs.
- **Developer experience:** unit test setup, working ESLint config, working dev container, running without Docker.
- **Internationalization** of e-mails and API messages.

[#12]: https://github.com/sawtooth-works/saas-and-ecommerce-boilerplate-nestjs/issues/12
[#13]: https://github.com/sawtooth-works/saas-and-ecommerce-boilerplate-nestjs/issues/13
[#14]: https://github.com/sawtooth-works/saas-and-ecommerce-boilerplate-nestjs/issues/14
[#15]: https://github.com/sawtooth-works/saas-and-ecommerce-boilerplate-nestjs/issues/15
[#16]: https://github.com/sawtooth-works/saas-and-ecommerce-boilerplate-nestjs/issues/16
[#17]: https://github.com/sawtooth-works/saas-and-ecommerce-boilerplate-nestjs/issues/17
[#18]: https://github.com/sawtooth-works/saas-and-ecommerce-boilerplate-nestjs/issues/18
[#19]: https://github.com/sawtooth-works/saas-and-ecommerce-boilerplate-nestjs/issues/19
[#20]: https://github.com/sawtooth-works/saas-and-ecommerce-boilerplate-nestjs/issues/20
[#21]: https://github.com/sawtooth-works/saas-and-ecommerce-boilerplate-nestjs/issues/21
[#22]: https://github.com/sawtooth-works/saas-and-ecommerce-boilerplate-nestjs/issues/22
[#23]: https://github.com/sawtooth-works/saas-and-ecommerce-boilerplate-nestjs/issues/23
[#24]: https://github.com/sawtooth-works/saas-and-ecommerce-boilerplate-nestjs/issues/24
[#25]: https://github.com/sawtooth-works/saas-and-ecommerce-boilerplate-nestjs/issues/25
[#26]: https://github.com/sawtooth-works/saas-and-ecommerce-boilerplate-nestjs/issues/26
[#27]: https://github.com/sawtooth-works/saas-and-ecommerce-boilerplate-nestjs/issues/27
[#28]: https://github.com/sawtooth-works/saas-and-ecommerce-boilerplate-nestjs/issues/28
[#29]: https://github.com/sawtooth-works/saas-and-ecommerce-boilerplate-nestjs/issues/29
[#30]: https://github.com/sawtooth-works/saas-and-ecommerce-boilerplate-nestjs/issues/30
[#32]: https://github.com/sawtooth-works/saas-and-ecommerce-boilerplate-nestjs/issues/32
[#33]: https://github.com/sawtooth-works/saas-and-ecommerce-boilerplate-nestjs/issues/33

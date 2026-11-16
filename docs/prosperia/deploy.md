# Prosperia deploy runbook (Chatwoot on Railway)

Replaces Respond.io as the sales team inbox. Later also hosts the support inbox (Evolution API) and the support bot (`Prosperia-App/prosperia-support-bot`). Fork of `chatwoot/chatwoot`, deployed from branch `prosperia` only.

## Pinned upstream

- Upstream: `chatwoot/chatwoot`
- Release: `v4.19.1` (commit `caab07a8f00032d5205b5a5d7e2eb0ba2f3d2d82`, recorded 2026-11-13). Release-tag image availability on Docker Hub is not yet verified.
- `Dockerfile.prosperia` is `FROM chatwoot/chatwoot:v4.19.1`: the running code is exactly that release. Branch `prosperia` = that tag + the files in this list: `Dockerfile.prosperia`, `railway/`, `docs/prosperia/`.
- Code patches: none. If one is ever needed, switch both services to `dockerfilePath: docker/Dockerfile` (source build) in a reviewed PR and record it here.

## Railway project

- Project: `chatwoot`, workspace `Prosperia` (separate from production; do not add services there).
- Public hostname: `chat.prosperiaapp.com` (Cloudflare DNS, SSL mode Full).
- Postgres: pgvector image, database `chatwoot`. Reserved on the same instance for later: `evolution`, `support_bot`.
- Redis: dedicated to Chatwoot. The support bot gets its own Redis later (BullMQ needs `noeviction`).

## Services

| Service | Config path | Start command | Restart policy | Public domain |
|---|---|---|---|---|
| `web` | `railway/web.json` | `rails db:chatwoot_prepare && rails s` | ON_FAILURE, max 5 | yes, port 3000 |
| `worker` | `railway/worker.json` | `sidekiq -C config/sidekiq.yml` | ALWAYS | no |

Migrations run only on `web`. Health: `GET https://chat.prosperiaapp.com/api` returns the version and `queue_services` / `data_services` = `ok`.

## Environment variables (names only; values in the password manager under `Chatwoot / Railway` and `Chatwoot / AWS`)

`Shared` = Railway shared variable used by `web` and `worker`.

| Variable | Scope | Value / note |
|---|---|---|
| `SECRET_KEY_BASE` | Shared | `openssl rand -hex 64` |
| `FRONTEND_URL` | Shared | `https://chat.prosperiaapp.com` |
| `FORCE_SSL` | Shared | `true` |
| `ENABLE_ACCOUNT_SIGNUP` | Shared | `false` |
| `DEFAULT_LOCALE` | Shared | `es` |
| `RAILS_ENV` / `NODE_ENV` | Shared | `production` |
| `INSTALLATION_ENV` | Shared | `docker` |
| `RAILS_LOG_TO_STDOUT` | Shared | `true` |
| `LOG_LEVEL` | Shared | `info` |
| `POSTGRES_HOST` | Shared | `${{Postgres.PGHOST}}` |
| `POSTGRES_PORT` | Shared | `${{Postgres.PGPORT}}` |
| `POSTGRES_USERNAME` | Shared | `${{Postgres.PGUSER}}` |
| `POSTGRES_PASSWORD` | Shared | `${{Postgres.PGPASSWORD}}` |
| `POSTGRES_DATABASE` | Shared | `chatwoot` |
| `REDIS_URL` | Shared | `${{Redis.REDIS_URL}}` |
| `MAILER_SENDER_EMAIL` | Shared | `Prosperia <facturas@prosperiaapp.com>`. SES email-address identity, verified; same sender as OpenReply. |
| `MAILER_REPLY_TO` | Shared | `soporte@prosperiaapp.com` (Google Workspace mailbox). Read by the fork patch below. |
| `SMTP_ADDRESS` | Shared | `email-smtp.<AWS_REGION>.amazonaws.com` |
| `SMTP_PORT` | Shared | `587` |
| `SMTP_DOMAIN` | Shared | `prosperiaapp.com` |
| `SMTP_USERNAME` / `SMTP_PASSWORD` | Shared | SES SMTP credentials of IAM user `ses-smtp-chatwoot`. Policy: `ses:SendRawEmail` only, `ses:FromAddress` = `facturas@prosperiaapp.com`. Access key and SMTP password to be created in console, stored under `Chatwoot / AWS`. |
| `SMTP_AUTHENTICATION` | Shared | `login` |
| `SMTP_ENABLE_STARTTLS_AUTO` | Shared | `true` |
| `ACTIVE_STORAGE_SERVICE` | Shared | `amazon` |
| `S3_BUCKET_NAME` | Shared | `prosperia-chatwoot-prod` |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | Shared | IAM user `chatwoot-storage` |
| `AWS_REGION` | Shared | same region as SES |
| `ENABLE_PUSH_RELAY_SERVER` | Shared | `true` (mobile app push through Chatwoot's relay) |
| `RAILS_MAX_THREADS` | Shared | `5` |
| `SIDEKIQ_CONCURRENCY` | `worker` | `10` |
| `PORT` | `web` | `3000` |

## Fork patch: Reply-To

Patch in branch `prosperia`: `app/mailers/application_mailer.rb` sets `default reply_to: ENV.fetch('MAILER_REPLY_TO', nil)`. Upstream sets only `from`. Conversation-reply mail passes its own `reply_to` explicitly and is unaffected. Transactional mail (invites, Devise, notifications) replies to `MAILER_REPLY_TO`. Not yet tested: needs a Ruby/Rails run (`bundle exec rspec` and `bundle exec rubocop` on the changed file).

## Account setup

- Account: `Prosperia`. Super admin: listed in the password manager.
- Integration user: `CRM Prosperia` (administrator). Its access token is `CHATWOOT_API_TOKEN` in the CRM service. Never use a person's token.
- Team: `Ventas`.
- Contact custom attributes (created by the CRM's `scripts/chatwoot-bootstrap.js`): `etapa` (list), `facturas`, `sistema`, `empleados`, `clientes`.
- Account webhook → CRM `/webhooks/chatwoot?key=…` (created by the same script). Events: `message_created`, `conversation_created`, `conversation_updated`, `contact_created`, `contact_updated`.

## Meta app (WhatsApp Cloud API)

- App: `Prosperia Inbox` (type Business, Prosperia business portfolio, Live mode). App ID: record here once created (not a secret).
- System user: `chatwoot-inbox`, assets: the sales WABA (full control) and the app. Permanent token with `whatsapp_business_messaging` + `whatsapp_business_management`, stored under `Chatwoot / Meta`.
- Webhook callback: the URL shown in the Chatwoot inbox settings (`https://chat.prosperiaapp.com/webhooks/whatsapp/+<number>`), field `messages`.
- `POST /{WABA_ID}/subscribed_apps` done with the system user token.

## Inboxes

| Inbox | Number | Channel | Bot | Inbox id |
|---|---|---|---|---|
| `Ventas TEST` | Meta test number | WhatsApp Cloud | no | record |
| `Ventas WhatsApp` | sales number | WhatsApp Cloud | no | record |
| `Soporte` (later) | support number | API inbox via Evolution | support bot (later) | — |

## Rollback (sales number)

1. CRM: `INBOX_PROVIDER=respond`, `RESPOND_POLL` unset (on), redeploy.
2. Respond.io: reconnect the WhatsApp channel to the same number.
3. Meta: `DELETE /{WABA_ID}/subscribed_apps` with the `Prosperia Inbox` system user token.
4. Record the failure here; debug from `web` and `worker` logs and the CRM's `crm_dashboard_raw_webhooks` (`source='chatwoot'`).

Respond.io stays paid until 7 clean days after cutover **and** until calls have a replacement.

## Upgrades

Monthly: read the release notes of every Chatwoot release after the pinned one (watch for breaking env changes and Postgres/Redis version requirements). In a PR to `prosperia`: rebase the three Prosperia paths onto the new tag (`git checkout -b upgrade-<tag> <tag>` and copy `Dockerfile.prosperia`, `railway/`, `docs/prosperia/`), update the `FROM` tag and *Pinned upstream* above. Merge, watch the `web` deploy (migrations), then check `/api` version, send and receive one WhatsApp message.

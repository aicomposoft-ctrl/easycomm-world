# Completion — easycomm-world

> Phase 5 SPARC. Deployment & Operations.
> Опирается на технологические решения, зафиксированные в Architecture.md
> (Distributed Monolith Monorepo, Next.js + Fastify + Prisma + BullMQ,
> PostgreSQL + Redis + ClickHouse, VPS + Docker Compose, MCP servers).

---

## 1. Pre-Deployment Checklist

Чек-лист — gating перед переходом dev → staging → prod. Phase, в которой
проверка обязательна, помечена в третьей колонке.

| # | Проверка | Stage | Команда / артефакт |
|---|---|---|---|
| 1 | `pnpm lint` — 0 ошибок | dev/staging/prod | `pnpm -w lint` |
| 2 | `pnpm typecheck` — 0 ошибок | dev/staging/prod | `pnpm -w typecheck` |
| 3 | `pnpm test:unit` — coverage ≥ targets из Refinement.md §2.1 | dev/staging/prod | CI report |
| 4 | `pnpm test:integration` — все зелёные | staging/prod | testcontainers CI job |
| 5 | `pnpm test:e2e` против ephemeral compose | staging/prod | Playwright report |
| 6 | `pnpm test:contract` — все маркетплейс-фикстуры | staging/prod | Adapter reports |
| 7 | `pnpm build` — все пакеты | dev/staging/prod | `dist/` или `.next/` |
| 8 | Trivy scan docker images — 0 high/critical | staging/prod | SBOM в artifact |
| 9 | Snyk scan deps — 0 high/critical (или waivers с обоснованием) | staging/prod | Snyk PR check |
| 10 | OWASP ZAP baseline — 0 high | staging/prod | nightly job |
| 11 | DB migration plan reviewed + rollback plan | staging/prod | PR description must include "Migration plan" |
| 12 | Feature flags для рискованных изменений | prod | LaunchDarkly / Unleash |
| 13 | Runbook ссылается на новую функцию (если есть on-call риск) | prod | `docs/runbooks/*.md` |
| 14 | Performance budget — нет регрессии >10% p95 | staging/prod | k6 baseline diff |
| 15 | Аудит-логирование критических действий — проверено | staging/prod | Code review checklist |
| 16 | i18n: нет голых строк ru/en в новом UI | prod | ESLint правило `no-literal-jsx-string` |
| 17 | Lighthouse Web Vitals: LCP <2.5s, CLS <0.1 на staging | prod | Lighthouse CI |
| 18 | Манифест Chrome Extension — review (permissions diff) | prod | Manual review |
| 19 | Privacy Policy + ToS актуальны (если API менялись) | prod | Legal sign-off |
| 20 | Backup verified within last 30 days (restore drill) | prod | Backup report |

---

## 2. Environments

### 2.1 local — Docker Compose

`docker-compose.yml` поднимает: postgres, redis, clickhouse, minio (S3
эмулятор), maildev (SMTP), wiremock (marketplace stubs), а также
Fastify API + Next.js. Используется для dev и для CI e2e.

| Сервис | Порт | Notes |
|---|---|---|
| web | 3000 | Next.js dev server |
| api | 4000 | Fastify |
| postgres | 5432 | tag `16-alpine` |
| redis | 6379 | tag `7-alpine` |
| clickhouse | 8123/9000 | tag `24-alpine` |
| minio | 9000/9001 | S3-совместимый |
| maildev | 1080 | UI на 1080, SMTP 1025 |
| wiremock | 8080 | marketplace stubs |

### 2.2 dev — shared VPS

- 1 VPS (AdminVPS), 4 vCPU / 8 GB RAM / 80 GB NVMe.
- Хостит: Postgres, Redis, CH, API, Web, BullMQ workers (все в compose).
- Деплой: автоматический при push в `develop`.
- Назначение: внутренние демо, ручные тесты, dogfooding.

### 2.3 staging — mirror prod

- 2 VPS (HOSTKEY), идентичная топология prod на минимум-1.
- Поднимается из тех же образов, что prod (теги `:staging`).
- Полные backup-restore drills тут же.
- Деплой: после merge в `main`, до prod (мануальный promote).

### 2.4 prod — RU jurisdiction

| Узел | Сервис | Спеки |
|---|---|---|
| `vps-app-1` (HOSTKEY MSK) | API, Web (compose) | 8 vCPU / 16 GB / 200 GB NVMe |
| `vps-app-2` (HOSTKEY MSK) | API, Web (compose) — hot standby | 8 vCPU / 16 GB |
| `vps-worker-1` (HOSTKEY MSK) | BullMQ workers | 4 vCPU / 8 GB |
| `vps-data` (HOSTKEY MSK) | PostgreSQL primary | 8 vCPU / 32 GB / 500 GB NVMe |
| `vps-data-replica` (HOSTKEY SPB) | Postgres streaming replica | 4 vCPU / 16 GB |
| `vps-ch` (HOSTKEY MSK) | ClickHouse single-node | 4 vCPU / 16 GB / 1 TB SSD |
| `vps-redis` (HOSTKEY MSK) | Redis + Sentinel | 2 vCPU / 4 GB |
| `vps-backup` (AdminVPS, другая зона) | Cold-standby + backups receiver | 2 vCPU / 8 GB / 2 TB |
| S3 | Yandex Cloud Object Storage (RU) | versioning + Object Lock |
| CDN | Yandex Cloud CDN | для статики и images |

**Все ПДн** хранятся на VPS в РФ-юрисдикции (152-ФЗ).

---

## 3. Deployment Sequence

Атомарный compose-swap с пред-/постпроверками.

1. **Preflight**
   - `gh workflow run preflight.yml` — повторно прогоняет lint + tests на target tag.
   - Проверка backup < 24h: `ssh vps-data 'cat /var/log/pg_backup.last'`.
   - Maintenance window декларируется в `#engineering` (Slack/Discord).

2. **Pull artifacts**
   - `ssh vps-app-1 'docker pull ghcr.io/easycomm/api:<sha> && docker pull ghcr.io/easycomm/web:<sha>'`
   - То же для `vps-app-2`, `vps-worker-1`.
   - **Rollback point R1:** если pull fails — abort, no impact.

3. **DB migrations (expand-only)**
   - `ssh vps-app-1 'docker run --rm --network easycomm_net -e DATABASE_URL=$URL ghcr.io/easycomm/api:<sha> npx prisma migrate deploy'`
   - Только expand-операции (ADD COLUMN nullable, CREATE INDEX CONCURRENTLY).
   - **Rollback point R2:** прямой `prisma migrate resolve --rolled-back <name>`; expand миграции откатываются drop column.

4. **Health gates pre-swap**
   - Smoke `GET /healthz` против старого compose — должно быть 200.
   - Подготовить новый compose файл `docker-compose.<sha>.yml` рядом со старым.

5. **Blue-green swap на vps-app-2** (резервный)
   - `docker compose -p easycomm-green -f docker-compose.<sha>.yml up -d`
   - Wait healthcheck (HEALTHCHECK в Dockerfile): `until docker inspect ... | jq '.[0].State.Health.Status' = '"healthy"'; do sleep 2; done` — таймаут 180 c.
   - Переключить nginx upstream на green (`reload`).
   - 5 минут наблюдения метрик и логов.
   - **Rollback point R3:** обратно nginx upstream на blue; `docker compose -p easycomm-green down`.

6. **Promote на vps-app-1**
   - Тот же blue-green swap.
   - Wait healthcheck.

7. **Workers** (vps-worker-1)
   - Drain режим: `docker exec ... node scripts/worker-drain.js` (перестают брать новые джобы).
   - Ждём пока `bullmq.active === 0` или таймаут 60 c.
   - Swap.

8. **Contract migrations (if any)**
   - DROP COLUMN, NOT NULL, и пр. — только после успешного swap и 24 ч наблюдения.
   - Запускаются отдельным workflow `contract-migrations.yml`.
   - **Rollback point R4:** если требуется откат после contract — restore из бэкапа точечного.

9. **Post-deploy verification**
   - Smoke E2E: login + dashboard + 1 SKU fetch.
   - Sentry release marker.
   - OTel deployment marker.
   - Telegram-канал ops: "deploy <sha> ok".

---

## 4. Rollback Procedure

### Уровни отката

| Уровень | Сценарий | Команда |
|---|---|---|
| L1: nginx upstream | swap green→blue | `nginx -s reload` (config с blue) |
| L2: compose revert | поднять предыдущий tag | `docker compose -f docker-compose.<prev-sha>.yml up -d` |
| L3: DB migration revert (expand) | drop добавленные columns/indexes | `prisma migrate resolve --rolled-back ...` + manual SQL |
| L4: DB restore (contract) | PITR из WAL | `pg_basebackup` + replay до timestamp |
| L5: full DR | потеря узла | см. DR-runbook §11 |

### Per-component rollback

- **Web**: только nginx-swap; данных нет.
- **API**: nginx-swap + проверить BullMQ consumer compatibility (job schema).
- **Workers**: rolling restart старой версии; убедиться нет orphaned locks в Redis.
- **DB migration**: только если expand-only; иначе — стопаем, debugging, PITR в крайнем случае.

### Irreversible migrations

Принцип: **не пишем irreversible миграции без feature flag**.

Если необходимо (например, drop колонки):
1. Шаг 1 (релиз N): expand — добавить новую колонку, начать писать в обе, чтение из старой.
2. Шаг 2 (релиз N+1): переключить чтение на новую.
3. Шаг 3 (релиз N+2, через ≥7 дней наблюдения): contract — drop старой.

Это требует 3 деплоя, но даёт rollback на любом шаге.

---

## 5. CI/CD Configuration

### 5.1 `.github/workflows/ci.yml`

```yaml
name: CI

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main, develop]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

env:
  NODE_VERSION: "22"
  PNPM_VERSION: "9"

jobs:
  setup:
    name: Setup
    runs-on: ubuntu-latest
    outputs:
      pnpm-store: ${{ steps.pnpm.outputs.STORE_PATH }}
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: pnpm
      - id: pnpm
        run: echo "STORE_PATH=$(pnpm store path --silent)" >> $GITHUB_OUTPUT
      - run: pnpm install --frozen-lockfile

  lint:
    name: Lint
    needs: setup
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm -w lint

  typecheck:
    name: Typecheck
    needs: setup
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm -w typecheck

  unit:
    name: Unit Tests
    needs: setup
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm -w test:unit -- --coverage
      - uses: actions/upload-artifact@v4
        with:
          name: coverage-unit
          path: coverage/

  integration:
    name: Integration Tests
    needs: setup
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: ci
          POSTGRES_DB: easycomm_test
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U postgres"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      clickhouse:
        image: clickhouse/clickhouse-server:24-alpine
        ports: ["8123:8123", "9000:9000"]
        options: >-
          --health-cmd "wget --no-verbose --tries=1 --spider http://localhost:8123/ping || exit 1"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 10
    env:
      DATABASE_URL: postgres://postgres:ci@localhost:5432/easycomm_test
      REDIS_URL: redis://localhost:6379
      CLICKHOUSE_URL: http://localhost:8123
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm -w prisma migrate deploy
      - run: pnpm -w test:integration

  e2e:
    name: E2E (Playwright)
    needs: [lint, typecheck, unit]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: docker compose -f docker-compose.e2e.yml up -d --wait
      - run: pnpm -w exec playwright install --with-deps chromium
      - run: pnpm -w test:e2e
      - if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
      - if: always()
        run: docker compose -f docker-compose.e2e.yml down -v

  security:
    name: Security Scan
    needs: setup
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

  build:
    name: Build Images
    needs: [lint, typecheck, unit, integration]
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build & push API
        uses: docker/build-push-action@v6
        with:
          context: .
          file: apps/api/Dockerfile
          push: true
          tags: |
            ghcr.io/${{ github.repository }}/api:${{ github.sha }}
            ghcr.io/${{ github.repository }}/api:${{ github.ref_name }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      - name: Build & push Web
        uses: docker/build-push-action@v6
        with:
          context: .
          file: apps/web/Dockerfile
          push: true
          tags: |
            ghcr.io/${{ github.repository }}/web:${{ github.sha }}
            ghcr.io/${{ github.repository }}/web:${{ github.ref_name }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      - name: Trivy scan API
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}/api:${{ github.sha }}
          severity: HIGH,CRITICAL
          exit-code: "1"
```

### 5.2 `.github/workflows/deploy.yml`

```yaml
name: Deploy

on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Target environment"
        type: choice
        options: [dev, staging, prod]
        required: true
      sha:
        description: "Commit SHA to deploy"
        required: true
  push:
    branches: [develop]

concurrency:
  group: deploy-${{ inputs.environment || 'dev' }}
  cancel-in-progress: false

jobs:
  deploy:
    name: Deploy to ${{ inputs.environment || 'dev' }}
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment || 'dev' }}
    env:
      DEPLOY_SHA: ${{ inputs.sha || github.sha }}
      TARGET_ENV: ${{ inputs.environment || 'dev' }}
    steps:
      - uses: actions/checkout@v4

      - name: Setup SSH key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.DEPLOY_SSH_KEY }}" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          ssh-keyscan -H ${{ secrets.DEPLOY_HOST }} >> ~/.ssh/known_hosts

      - name: Verify image exists
        run: |
          docker manifest inspect ghcr.io/${{ github.repository }}/api:${DEPLOY_SHA}
          docker manifest inspect ghcr.io/${{ github.repository }}/web:${DEPLOY_SHA}

      - name: Render compose
        run: |
          envsubst < deploy/docker-compose.template.yml > docker-compose.${DEPLOY_SHA}.yml

      - name: Copy compose to host
        run: |
          scp docker-compose.${DEPLOY_SHA}.yml \
            ${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_HOST }}:/opt/easycomm/

      - name: Pull images on host
        run: |
          ssh ${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_HOST }} \
            "cd /opt/easycomm && \
             docker pull ghcr.io/${{ github.repository }}/api:${DEPLOY_SHA} && \
             docker pull ghcr.io/${{ github.repository }}/web:${DEPLOY_SHA}"

      - name: Run DB migrations (expand-only)
        run: |
          ssh ${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_HOST }} \
            "cd /opt/easycomm && \
             docker run --rm --network easycomm_net \
               --env-file .env.${TARGET_ENV} \
               ghcr.io/${{ github.repository }}/api:${DEPLOY_SHA} \
               npx prisma migrate deploy"

      - name: Atomic compose swap
        run: |
          ssh ${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_HOST }} \
            "cd /opt/easycomm && \
             docker compose -p easycomm-green \
               -f docker-compose.${DEPLOY_SHA}.yml up -d && \
             /opt/easycomm/scripts/wait-healthy.sh easycomm-green 180 && \
             /opt/easycomm/scripts/nginx-swap.sh green && \
             sleep 30 && \
             docker compose -p easycomm-blue down || true && \
             mv easycomm-green easycomm-blue"

      - name: Post-deploy smoke
        run: |
          curl -fsS https://${{ secrets.DEPLOY_PUBLIC_HOST }}/healthz
          curl -fsS https://${{ secrets.DEPLOY_PUBLIC_HOST }}/api/v1/version | jq -e ".sha == \"${DEPLOY_SHA}\""

      - name: Notify Telegram ops
        if: always()
        run: |
          STATUS="${{ job.status }}"
          curl -fsS -X POST https://api.telegram.org/bot${{ secrets.TG_BOT_TOKEN }}/sendMessage \
            -d chat_id=${{ secrets.TG_OPS_CHAT }} \
            -d text="Deploy ${TARGET_ENV} ${DEPLOY_SHA:0:7}: ${STATUS}"
```

---

## 6. Database Migration Strategy

### 6.1 Prisma migrate deploy в CI

- Команда: `npx prisma migrate deploy`.
- Запускается в deploy.yml ПОСЛЕ pull-а образа, ДО compose-swap.
- Если миграция падает — deploy abort, swap не происходит.

### 6.2 Online migration patterns (expand-migrate-contract)

| Шаг | Действие | Деплоев |
|---|---|---|
| E (expand) | Добавить новое (nullable column, index CONCURRENTLY) | 1 |
| M (migrate) | Двойная запись/чтение; backfill через джоб | 1+ |
| C (contract) | Удалить старое после ≥7 дней | 1 |

**Пример** — переименование `sku.price → sku.list_price`:
1. Expand: `ADD COLUMN list_price NUMERIC NULL`.
2. Migrate: code пишет в обе; backfill `UPDATE sku SET list_price = price`; read из list_price с fallback.
3. Contract: `DROP COLUMN price`; убрать fallback.

### 6.3 Rollback для irreversible

- Запрещены без feature flag.
- Если необходимо — restore из бэкапа в режиме PITR + manual reconcile.
- Контракт: irreversible миграция требует sign-off DevOps lead и записи в `docs/migrations/manual-review.md`.

### 6.4 CONCURRENTLY и lock-аware

- Все CREATE INDEX — с `CONCURRENTLY`.
- ALTER TABLE с проверкой lock_timeout = 5s и `SET statement_timeout = 30s`.
- Большие backfill — батчами по 1000 строк через BullMQ job, не одной транзакцией.

---

## 7. Monitoring & Alerting

15+ метрик. Алерт-каналы: PagerDuty (severe), Telegram ops (warning),
Email digest (info). SLO в последней колонке.

| # | Метрика | Сервис | Тип | Severity | Алерт-порог | SLO |
|---|---|---|---|---|---|---|
| 1 | `http_requests_total` (rate) | API/Web | RED-Rate | info | n/a | n/a |
| 2 | `http_requests_failed_total{code=~"5.."}` | API/Web | RED-Errors | severe | >1% за 5 мин | <0.1% |
| 3 | `http_request_duration_seconds{quantile="0.95"}` | API | RED-Duration | warning | >500 ms за 5 мин | <300 ms |
| 4 | `http_request_duration_seconds{quantile="0.95",route="/api/dashboard/*"}` | API | RED | warning | >800 ms | <500 ms |
| 5 | `bullmq_active_jobs` | Workers | Saturation | warning | >500 sustained 10 мин | n/a |
| 6 | `bullmq_failed_total` (rate) | Workers | Errors | severe | >10/min | <1/min |
| 7 | `marketplace_429_total{mp}` | API | Errors | warning | >100/h | <50/h |
| 8 | `mcp_failover_total` | API | Errors | warning | >5/h | <2/h |
| 9 | `vault_unlock_failed_total` | Web | Security | warning | >50/min globally | n/a |
| 10 | `repricer_oscillation_blocked` | Workers | Quality | info | n/a | n/a |
| 11 | `dau` (daily active users) | Business | Growth | info | day-over-day < -20% | n/a |
| 12 | `mrr_rub` | Billing | Business | info | week-over-week < -5% | growth target |
| 13 | `churn_monthly_pct` | Billing | Business | warning | >7% | <5% |
| 14 | `ai_cost_per_user_daily_rub` | AI | Business | warning | >50 ₽ | <30 ₽ |
| 15 | `activation_rate_first_24h_pct` | Onboarding | Growth | warning | <40% | >55% |
| 16 | `pg_connections` / `max_connections` | DB | Saturation | warning | >70% | <60% |
| 17 | `pg_replication_lag_seconds` | DB | Health | severe | >30 s | <5 s |
| 18 | `redis_used_memory_bytes` / `maxmemory` | Redis | Saturation | warning | >80% | <70% |
| 19 | `clickhouse_parts_count` | CH | Saturation | warning | >1000 | <500 |
| 20 | `clickhouse_merge_seconds` (p99) | CH | Health | warning | >300 s | <120 s |
| 21 | `node_filesystem_free_bytes` < 10% | Infra | Saturation | severe | <10% free | >20% free |
| 22 | `cert_expiry_days_remaining` | Infra | Health | warning | <14 days | >30 days |
| 23 | `backup_age_hours` | Infra | Health | severe | >36 h | <24 h |
| 24 | `audit_log_chain_valid` | Security | Integrity | severe | =0 | =1 |
| 25 | `availability_30d_pct` | API | SLO | n/a | n/a | ≥99.5% |

**Стек:** Prometheus (scrape) + Grafana (дашборды) + Alertmanager →
PagerDuty + Telegram. Логи отдельно через Loki.

### Грэйфана-дашборды:
- "Golden Signals" (latency/traffic/errors/saturation на API)
- "Marketplaces health" (RPS, errors, 429 per adapter)
- "Business" (DAU, MRR, churn, activation)
- "Infra" (cpu/mem/disk на каждом VPS)
- "BullMQ" (queues, waiting, active, failed)
- "Database" (connections, replication, slow queries)

---

## 8. Logging Strategy

### Формат

- Structured JSON через **Pino** во всех Node-сервисах.
- Поля обязательные: `ts, level, msg, service, env, sha, traceId, spanId, tenantId?, userId?, requestId?`.
- Frontend клиентские ошибки — через Sentry (отдельный канал, не сырые логи).

### PII scrubbing rules

Pino redact paths (применяется до сериализации):

```
[
  "req.headers.authorization",
  "req.headers.cookie",
  "req.body.password",
  "req.body.masterPassword",
  "req.body.apiKey",
  "req.body.token",
  "*.email",
  "*.phone",
  "*.ipAddress",
  "vault.*",
  "user.passwordHash"
]
```

Email и phone, если необходимы для debugging — хэш `sha256(value)` идёт в логи, plaintext — никогда.

### Retention

| Слой | Storage | Retention |
|---|---|---|
| Hot (Loki cluster, NVMe) | local | 30 дней |
| Cold (S3 Glacier-class) | Yandex Cloud Object Storage | 365 дней |
| Audit log (PG append-only) | PG | 7 лет (152-ФЗ) |
| Audit log hash receipts | S3 Object Lock | 7 лет |

---

## 9. Tracing

- **OpenTelemetry SDK** в API, Workers, Web SSR.
- Бэкенд: **Tempo** (или Jaeger на dev).
- Frontend RUM: OTel JS instrumentation → Tempo через OTLP/HTTP.
- **Sampling:** 1% baseline (head-based) + 100% на ошибках (tail-based через OTel collector tail_sampling processor).
- **Trace propagation:** W3C Trace Context во всех заголовках; BullMQ jobs несут traceId в payload.
- **Chrome Extension** генерирует traceId и пробрасывает в API через `traceparent` header.

---

## 10. Backup Strategy

| Резерв | Источник | Частота | Retention | Storage | Verify |
|---|---|---|---|---|---|
| PG logical (`pg_dump`) | primary | daily 03:00 MSK | 30 дней | S3 (RU) | weekly restore drill в staging |
| PG WAL archiving | primary | непрерывно | 7 дней | S3 (RU) | хранит PITR |
| PG base backup | primary | weekly | 4 weeks | S3 (RU) | PITR base |
| CH snapshot (`BACKUP`) | CH node | daily 04:00 | 14 дней | S3 (RU) | monthly restore drill |
| Redis RDB | Redis | hourly | 24 hours | local + S3 daily | n/a (ephemeral cache) |
| Compose env files & secrets vault | vps-* | on change | 90 дней | encrypted S3 | manual review monthly |
| Audit log dumps | PG audit_log | daily | 7 лет | S3 Object Lock | hash chain verify daily |
| Application code | Git | — | forever | GitHub + mirror на Gitea (RU) | n/a |

**S3 lifecycle:** versioning ON; >30d → cold; >180d → archive class.
**Restore tested monthly:** randomly chosen day → restore до staging → smoke E2E.

---

## 11. Disaster Recovery Plan

**Targets:**
- **RTO (Recovery Time Objective):** 4 ч от P0 инцидента до восстановления prod.
- **RPO (Recovery Point Objective):** 1 ч (т.е. не более 1 ч данных потеряется).

### Сценарии

| Сценарий | Действия |
|---|---|
| Падение vps-app-1 | Trafik автоматически на vps-app-2 (active-active за nginx); 0 downtime |
| Падение vps-data (primary) | Promote vps-data-replica; ~5 мин downtime + reconfig string; replication обратно при возврате primary |
| Падение vps-ch | CH single-node — read-only до восстановления; ingestion пауза, BullMQ копит; <2 ч RTO |
| Падение S3 region | Failover на backup S3 (другой провайдер, holds last 7d) — degraded mode без статики |
| Утечка ключей | Ротация всех secrets; force re-login всех users; force vault re-encrypt; notify в течение 24 ч |
| Ransomware на app VPS | Восстановление из неизменяемых образов; данные — из PITR; immutable receipts checked |
| Полная потеря prod | Спин-ап из IaC (Terraform/Ansible playbook); restore из последнего backup в DR-регион |

### Runbook outline

```
1. Declare incident (#incident-<date>; severity P0/P1/P2)
2. Page on-call (PagerDuty)
3. Diagnose: dashboards Grafana + recent deploys + Sentry releases
4. Containment: rollback / failover / disable feature flag
5. Mitigation: per scenario actions
6. Communication: status page + Telegram support
7. Eradication: root cause analysis
8. Recovery: full prod health check (top-8 E2E)
9. Post-mortem within 5 дней (blameless)
```

---

## 12. Cost Model (ориентировочно, ₽/мес)

| Категория | MVP (0–100 users) | 1k users |
|---|---|---|
| VPS (HOSTKEY) — 3–8 узлов | 18 000 | 75 000 |
| S3 storage (Yandex Object Storage) | 500 (50 GB) | 8 000 (1 TB) |
| Yandex Cloud CDN | 200 | 5 000 |
| Egress / network | 500 | 10 000 |
| AI MCP tokens (YandexGPT + резерв OpenAI) | 3 000 | 60 000 (≈ 60 ₽/user/mo) |
| Email (Unisender) | 1 000 | 5 000 |
| ЮKassa fees (3.5%) | n/a | ≈ 105 000 (3.5% от 3M MRR) |
| Sentry / Grafana Cloud (low-tier) | 0 (self-hosted) | 5 000 (расширение нагрузки) |
| Snyk subscription | 0 (free tier) | 12 000 |
| Domain & TLS | 200 | 200 |
| **Итого / мес** | **≈ 23 400 ₽** | **≈ 285 000 ₽** |

**Per-user стоимость (excl. fees):**
- MVP: ≈ 234 ₽/user (но fixed-cost-heavy)
- 1k users: ≈ 180 ₽/user (без учёта ЮKassa fees)

При Pro-tier 2 990 ₽ valid LTV/CAC модель достижима после ~300 платящих.

---

## 13. Handoff Checklists

### Dev → QA

- [ ] PR имеет описание + screenshots/recording для UI-изменений
- [ ] Unit + integration тесты прошли
- [ ] Локально воспроизводится в `docker-compose.yml`
- [ ] Feature flag создан (если рискованно)
- [ ] Test plan приложен (или ссылка на Gherkin)

### QA → Ops (release)

- [ ] E2E зелёные на staging
- [ ] Performance baseline без регрессии
- [ ] Security scan чистый
- [ ] Release notes готовы
- [ ] Runbook обновлён (если on-call риск)

### Ops → Customer Success

- [ ] Notification customer support о новой функции
- [ ] Help center статья (или ссылка на черновик)
- [ ] Известные ограничения зафиксированы
- [ ] Канал для feedback готов (Telegram)

### Customer Success → Product

- [ ] Сводка обращений в support за 7/30 дней
- [ ] NPS / CSAT для затронутой функции
- [ ] Roadmap-кандидаты на основе feedback

---

## 14. Runbooks — outlines

### 14.1 `runbooks/vault-corruption-recovery.md`
1. Симптомы: пользователь жалуется "не могу разблокировать"; Sentry events `vault_corruption`.
2. Не можем восстановить vault (zero-knowledge), но можем помочь пользователю переподключить ключи.
3. Шаги: identify user → confirm corruption (Sentry) → guide через UI reset → log incident.
4. Постмортем — если >5 incidents/неделю.

### 14.2 `runbooks/marketplace-api-outage.md`
1. Симптомы: `marketplace_429_total` + `marketplace_5xx_total` спайк; circuit breaker open.
2. Action: проверить status page маркетплейса; если выяснено, что они лежат — disable adapter, показать UI banner.
3. BullMQ jobs накапливаются — пауза worker-а; resume через 5 мин после восстановления.
4. После recovery — backfill catch-up job.

### 14.3 `runbooks/ai-mcp-outage.md`
1. Симптомы: `mcp_failover_total` спайк; AI запросы фейлятся.
2. Failover chain: YandexGPT → OpenAI → Claude. Если все падают — disable AI features flag, banner юзерам.
3. Не списывать квоту за failed запросы.

### 14.4 `runbooks/payment-incident.md`
1. Симптомы: ЮKassa webhook не приходит ИЛИ массовые 3DS failures.
2. Action: проверить ЮKassa status; включить fallback на ProdamusGate; manual reconciliation в конце дня.
3. Уведомить affected users в течение 24 ч.

### 14.5 `runbooks/full-restore.md`
1. Триггер: полная потеря prod (см. DR §11).
2. Step 1: спин-ап нового VPS-кластера через Ansible.
3. Step 2: restore PG из последнего base + WAL до RPO.
4. Step 3: restore CH из снапшота.
5. Step 4: regenerate JWT signing keys; force user re-login.
6. Step 5: smoke E2E top-8.
7. Step 6: communicate status; analyze logs для понимания причины.

---

## 15. Compliance Audit Trail

### 152-ФЗ

- Reference: см. Refinement.md §8.
- Audit checklist выполняется ежеквартально, результат в `docs/compliance/152fz-q<N>.md`.
- РКН-уведомление подано до production launch; копия в `docs/compliance/rkn-notification.pdf`.

### ToS с маркетплейсами

| МП | ToS-требование | Наш статус |
|---|---|---|
| Wildberries | Использование Seller API ограничено зарегистрированным продавцом; не скрейпинг | ✅ только официальный API |
| Ozon | Аналогично; rate limits соблюдаются | ✅ |
| Яндекс.Маркет | Partner API с явным согласием продавца | ✅ |
| Megamarket | Partner API | ✅ Phase 2 |

### Data subject rights (152-ФЗ + GDPR-style для будущего)

| Право | Endpoint / процесс | SLA |
|---|---|---|
| Доступ к своим данным | `GET /settings/privacy/export` | мгновенно (job <5 мин) |
| Исправление | `PATCH /api/user/profile` | мгновенно |
| Удаление | `DELETE /settings/account` | 30 дней soft-delete → hard-delete |
| Ограничение обработки | Support request | 30 дней |
| Возражение | Support request | 30 дней |
| Переносимость | `GET /settings/privacy/export` (JSON) | мгновенно |
| Отзыв согласия | Toggle в settings | мгновенно |

Все обращения логируются в `dsr_log` таблицу с tracking.

---

## 16. Summary

- **20-item pre-deploy checklist** с gating per-stage.
- **4 environments** с чёткими spec-ами VPS.
- **9-шаговая deployment sequence** с 4 rollback points.
- **5 rollback levels** + irreversible migration protocol.
- **Полные `ci.yml` и `deploy.yml`** (валидный YAML, готовы к копированию).
- **Expand-Migrate-Contract** как основа DB migration discipline.
- **25 мониторинг-метрик** с алерт-порогами и SLO.
- **Structured logging** через Pino с PII redaction и 30d/365d retention.
- **OTel + Tempo** с 1%/100% adaptive sampling.
- **Бэкап-стратегия** PG (logical + WAL + PITR), CH snapshots, S3 versioning.
- **DR plan** RTO 4 ч / RPO 1 ч с 7 сценариями.
- **Cost model**: ≈23k ₽/мес MVP → ≈285k ₽/мес @ 1k users.
- **4 handoff checklists** (Dev/QA/Ops/CS).
- **5 runbook outlines**.
- **152-ФЗ compliance trail** + ToS-маркетплейсы + DSR процессы.

Завершает SPARC-пакет вместе с Final_Summary.md.

# Refinement — easycomm-world

> Phase 4 SPARC (Refinement). Edge cases, testing, optimization, security.
> Документ описывает, что может пойти не так, как мы это ловим тестами,
> где режем латентность и как защищаемся.

---

## 1. Edge Cases Matrix

Матрица охватывает 8 функциональных областей. Колонки: триггер →
ожидаемое поведение → детектор → мера в проде. Минимум 25 строк.

| # | Область | Edge case | Триггер | Ожидаемое поведение | Детектор / тест | Мера в проде |
|---|---|---|---|---|---|---|
| 1 | Vault | Забыт мастер-пароль (нет recovery) | User инициирует "восстановить" | UI: явно сообщить "восстановления нет, переподключите API-ключи"; обнулить vault и потребовать reconnect маркетплейсов | E2E: `vault.forgot-password.spec.ts` | `metric: vault_reset_total` алерт >5/час |
| 2 | Vault | Кража устройства | Сессия > 15 мин без активности | Auto-lock vault; refresh token revoke на сервере при следующем запросе | Unit: `autoLock.test.ts` (fake timers) | `metric: vault_autolock_total` |
| 3 | Vault | Cross-device sync | User логинится на 2-м устройстве | Vault не синхронизирован — требуется ввод мастер-пароля повторно (zero-knowledge by design) | E2E: `vault.cross-device.spec.ts` | Документировано в onboarding |
| 4 | Vault | Corruption IndexedDB | AES-GCM auth tag mismatch | UI: "хранилище повреждено, переподключите ключи"; не падать; послать sentry event без plaintext | Property-based: `vault.tampered.fuzz.ts` | Sentry tag `vault_corruption` |
| 5 | Vault | Tampering detection | Браузерное расширение/malware пытается прочитать IDB | Web Crypto держит non-extractable CryptoKey; даже plaintext API key никогда не лежит в памяти > 100 мс | Unit: `cryptoKey.nonExtractable.test.ts` | CSP report-only + SRI |
| 6 | Vault | Auto-lock во время текущего AI-запроса | Таймер сработал в момент стрима | Запрос завершить (gracefully); следующий — потребует unlock; не терять полученный контент | Integration: `vault.lock-midstream.test.ts` | OTel span атрибут `interrupted_by_lock=true` |
| 7 | Marketplace | API key rotation | User обновил ключ WB | Старый ключ deprecated в vault; jobs на старом ключе переходят на новый при следующем poll; нет дубликатов | Integration testcontainer + WB stub | `metric: api_key_rotations_total` |
| 8 | Marketplace | Rate limit (429) | WB вернул 429 с Retry-After | BullMQ retry-with-backoff (jitter); SLA-окно 5 мин; уведомление если >10 мин подряд | Contract: `wb-adapter.rate-limit.spec.ts` | `metric: marketplace_429_total{mp}` |
| 9 | Marketplace | 5xx от маркетплейса | 50% запросов фейлятся за окно 1 мин | Circuit breaker (5-min open + half-open); UI badge "WB недоступен" | Integration: `circuitBreaker.test.ts` | Алерт PagerDuty при breaker open >15 мин |
| 10 | Marketplace | Partial response | WB вернул 80 SKU из 100 | Job помечен `partial=true`, ретрай по диффу; UI показывает "обновлено N из M" | Integration: `partialSync.test.ts` | `metric: sync_partial_total` |
| 11 | Marketplace | Timezone mismatch | Ozon отдаёт UTC, WB — MSK | Все timestamps нормализуются в UTC при ingestion; UI рендерит в TZ юзера (по profile) | Unit: `tzNormalize.test.ts` | Audit log fix-up daily |
| 12 | Marketplace | Удалённый SKU на маркетплейсе | SKU исчез из выгрузки | Soft-delete (status=archived); 30-дневный карантин; не удаляем history | Integration: `skuDeleted.test.ts` | `metric: sku_archived_total` |
| 13 | Marketplace | Flash sale price changes | Цена меняется 50 раз/час | Throttle ingestion price-history до 1 запись/15 мин; spike-detection отдельным потоком | Performance: `priceFlashSale.k6.js` | CH MV `price_changes_hourly` |
| 14 | Repricer | Rule conflict | 2 правила матчат один SKU | Приоритет по `priority` field (DESC), tie-break — дата создания DESC; warning в UI | Unit: `ruleResolver.test.ts` (table-driven) | UI badge "конфликт правил" |
| 15 | Repricer | Concurrent updates | Цена меняется user-ом и repricer-ом одновременно | Optimistic lock (version column); проигравший — retry с fresh state; max 3 попытки | Integration: `concurrentRepricer.test.ts` | `metric: repricer_conflict_total` |
| 16 | Repricer | Price floor violation | Расчёт ниже floor | Применить floor, лог `floor_clipped=true`; никогда не уходить ниже | Property-based: `repricer.floor.fuzz.ts` | Алерт >100 floor-clip/час |
| 17 | Repricer | Competitor disappears | Конкурент удалил карточку | Перейти на next-best competitor; если ни одного — fallback к target margin rule | Integration: `competitorGone.test.ts` | `metric: competitor_lost_total` |
| 18 | Repricer | Oscillation prevention | Цена прыгает между 2 значениями | Hysteresis ±2% и min-hold 30 мин; debounce price-change events | Unit: `oscillationGuard.test.ts` | `metric: repricer_oscillation_blocked` |
| 19 | Repricer | Dry-run vs apply | User запустил dry-run, потом apply | Dry-run пишет в `repricer_simulation`; apply — отдельный path, требует подтверждения; нет автогенерации side-effects из dry-run | E2E: `repricer.dryRun.spec.ts` | Audit log delta dry→apply |
| 20 | AI | MCP server outage | YandexGPT/OpenAI MCP недоступен | Fallback chain: yandex → openai → claude; если все — UI "AI временно недоступен", без ошибки 500 | Integration: `mcpFallback.test.ts` | `metric: mcp_failover_total` |
| 21 | AI | Timeout | MCP не ответил за 30 с | Cancel + cleanup; не списывать AI-квоту; вернуть friendly error | Integration: `mcpTimeout.test.ts` | `metric: mcp_timeout_total` |
| 22 | AI | Quota exceeded | User исчерпал месячный лимит | Soft-block с upsell modal; админ может выдать grace 100 запросов | Unit: `quotaGuard.test.ts` | `metric: quota_blocked_total` |
| 23 | AI | Hallucination detection | Генерация содержит несуществующий артикул/бренд | Post-validation regex + сравнение с DB; если нерелевантно — retry с tighter prompt | Property-based: `hallucinationGuard.test.ts` | `metric: ai_hallucination_caught` |
| 24 | AI | Off-topic completion | User спросил рецепт пиццы | Pre-prompt guardrail + topic classifier; ответ "я помогаю только с маркетплейсами" | Unit: `topicGuard.test.ts` | Sample audit 1% prompts |
| 25 | Multichannel | SKU sync conflict | Один SKU отредактирован в WB и в нашем UI одновременно | Last-write-wins по `updated_at` + warning bell; UI показывает diff | Integration: `multichannelConflict.test.ts` | `metric: mc_conflict_total` |
| 26 | Multichannel | Image format incompatibility | Ozon требует JPG, WB допускает WebP | Pipeline: ingest → нормализация в JPG 2000×2000 (sharp) → per-channel upload | Unit: `imageNormalize.test.ts` | `metric: image_convert_total` |
| 27 | Multichannel | Attribute mapping gap | У ЯМ есть поле, которого нет в WB | UI диалог "заполнить ЯМ-only"; нельзя падать на push | Integration: `attrMapping.test.ts` | `metric: attr_gap_total` |
| 28 | Multichannel | Concurrent edit | 2 user-а в одном tenant правят SKU | Pessimistic lock с TTL 5 мин; UI badge "редактирует Маша" | Integration: `concurrentEdit.test.ts` | `metric: edit_lock_acquired` |
| 29 | Agency | Client revokes access | Клиент отозвал доступ агентству | Все active sessions агента — кикнуть; jobs in-flight — finish & freeze; audit-лог "revoke by client" | E2E: `agency.revoke.spec.ts` | `metric: agency_revoke_total` |
| 30 | Agency | Orphan permissions | Удалили клиента, но permissions остались | Daily cron `permission-reaper`; cascade soft-delete | Integration: `orphanPerms.test.ts` | Алерт >0 orphans |
| 31 | Agency | Cross-tenant data leak risk | Plaintext SQL bypassing RLS | Все queries через Prisma middleware c `tenantId`-инъекцией; запрещён raw SQL без явной обёртки | Lint rule + unit: `tenantGuard.test.ts` | DAST scanner еженедельно |
| 32 | Billing | Payment failed | ЮKassa вернула 3DS reject | Retry через 24/72 ч; на 4-й fail — downgrade на Free + email | Integration: `billing.fail.test.ts` | `metric: billing_fail_total` |
| 33 | Billing | Refund | User инициирует возврат в 14-дневном окне | Prorate; revert subscription; не удалять данные | Integration: `billing.refund.test.ts` | Audit `refund_issued` |
| 34 | Billing | Partial period | Upgrade mid-cycle | Prorate billing; неиспользованный остаток зачесть | Unit: `prorate.test.ts` | n/a |
| 35 | Billing | Downgrade с quota overrun | Pro → Free, у юзера 100 SKU | UI warning; grace period 14 дней; затем заморозка excess SKU (read-only) | Integration: `downgradeOverrun.test.ts` | `metric: downgrade_overrun_total` |
| 36 | Performance | Cold cache | Холодный старт после deploy | Pre-warm hot keys (top-100 tenants); SLA <2 с на первый запрос после restart | Performance: `coldStart.k6.js` | `metric: cache_miss_ratio` |
| 37 | Performance | Database connection storm | Sudden 1000 reqs/s | Connection pool max=50, queue=200; reject 503 с retry-after | Performance: `connStorm.k6.js` | Алерт pool saturation >80% |
| 38 | Performance | ClickHouse merge backlog | Parts > 1000 | Pause ingestion (BullMQ rate-limit) до merge cooldown; алерт ops | Integration: `chBacklog.test.ts` | `metric: ch_parts_count` |

**Итого: 38 edge cases** (требование: ≥25).

---

## 2. Testing Strategy

### 2.1 Unit (Vitest)

| Модуль | Coverage target | Особенности |
|---|---|---|
| `packages/core` (crypto, repricer engine, quota) | **≥ 90%** | Property-based для crypto и floor-логики |
| `packages/marketplaces/*` (адаптеры) | **≥ 85%** | Recorded fixtures (см. contract tests) |
| `packages/api` (Fastify routes) | **≥ 80%** | Smoke + edge cases |
| `apps/web` (Next.js) | **≥ 70%** | UI-логика; визуал — Storybook + Chromatic |
| `apps/extension` (Chrome MV3) | **≥ 70%** | Mocked `chrome.*` API |
| `packages/billing` | **≥ 90%** | Prorate и финансовая логика — критично |

**Запуск:** `pnpm test:unit` локально; в CI с `--coverage` и threshold gating.

### 2.2 Integration (Vitest + Testcontainers)

Поднимаем реальные PostgreSQL 16, Redis 7, ClickHouse 24 через
`@testcontainers/postgresql` и т.д. Один контейнер на test suite, чистка
схемы между тестами через `TRUNCATE ... RESTART IDENTITY CASCADE`.

| Сценарий | Контейнеры | Время |
|---|---|---|
| API e2e на ручках | PG + Redis | <60 c suite |
| BullMQ jobs | PG + Redis | <60 c |
| ClickHouse ingestion | PG + CH | <90 c |
| Marketplace adapter against stub | PG + Redis + WireMock | <45 c |

### 2.3 E2E (Playwright) — топ-8 user journeys

1. **Signup → email confirm → создание workspace → unlock vault**
2. **Подключение WB через API-ключ → импорт первых 100 SKU → "aha moment" дашборд**
3. **Создание repricer rule → dry-run → apply → откат за 24 ч**
4. **AI: сгенерировать описание карточки → отредактировать → опубликовать в WB**
5. **Multichannel: завести карточку → push в WB + Ozon одним кликом**
6. **Agency: invite клиента → клиент grants access → агент видит данные → revoke**
7. **Billing: upgrade Free → Pro → ЮKassa 3DS → активация Pro**
8. **Chrome Extension: установить → открыть карточку WB → увидеть оверлей метрик**

**Запуск:** `pnpm test:e2e` против эфемерного `docker-compose -f docker-compose.e2e.yml up -d`.

### 2.4 Contract tests для marketplace-адаптеров

- Рекорды реальных ответов WB/Ozon/ЯМ — в `packages/marketplaces/__fixtures__`.
- Тест `verify` проверяет, что наш парсер всё ещё извлекает все обязательные поля.
- Обновление фикстур: `pnpm test:contract:record` (ручной запуск с реальным test-аккаунтом).
- CI запускает только `verify` mode (детерминированный).

### 2.5 Property-based tests (fast-check)

- **Vault round-trip:** `encrypt(plain, key) → decrypt(...) === plain` для произвольных байтовых строк длиной 1..10MB.
- **Repricer floor:** для любых rule + market price, итоговая цена ≥ floor.
- **Prorate:** для любых дат cycle и upgrade, сумма prorate монотонна.
- **Tenant isolation:** для любых SQL queries, при подмене tenantId fixture, не возвращается ничего чужого.

### 2.6 Performance tests (k6)

| Endpoint family | RPS target | p95 latency | Профиль |
|---|---|---|---|
| `GET /api/sku/:id` | 500 | <100 ms | Steady |
| `GET /api/dashboard/overview` | 200 | <250 ms | Steady |
| `POST /api/repricer/run` | 50 | <400 ms | Spike |
| `GET /api/clickhouse/timeseries` | 100 | <500 ms | Steady |
| AI `POST /api/ai/generate` | 20 | <8 s (streamed) | Steady |
| Webhook `POST /api/webhooks/ozon` | 200 | <80 ms | Burst |

**Профили:** `ramp-up 2 min → steady 10 min → cool-down 2 min`.
Запускаются nightly против staging.

### 2.7 Security tests

- **OWASP ZAP baseline:** ежедневно, public endpoints.
- **Custom CSP test:** Playwright проверяет, что `script-src` блокирует inline.
- **SSRF test:** image-proxy refuses `127.0.0.1`, `169.254.169.254`, internal CIDR (10/8, 192.168/16, 172.16/12).
- **JWT replay:** просроченный token не принимается даже при rolling secret.
- **Audit log integrity:** snapshot теста проверяет, что hash-chain consistent.

---

## 3. Test Cases (Gherkin) — 14 сценариев

### Vault

```gherkin
Feature: Vault unlock

  Scenario: Happy path unlock
    Given user signed in and vault is locked
    When user enters correct master password
    Then vault becomes unlocked
    And API keys are available for marketplace requests
    And vault_unlock_total counter increments

  Scenario: Wrong password
    Given user signed in and vault is locked
    When user enters wrong master password 5 times within 60s
    Then vault remains locked
    And user is rate-limited for 5 minutes
    And vault_unlock_failed_total increments

  Scenario: Auto-lock after 15 min idle
    Given vault is unlocked at 10:00
    And user has no activity for 15 minutes
    When clock advances to 10:15:01
    Then vault is auto-locked
    And next protected request prompts master password
```

### Marketplace

```gherkin
Feature: Connect Wildberries and import SKUs

  Scenario: Connect WB and import 100 SKUs
    Given user has Pro subscription and unlocked vault
    When user pastes a valid WB API key into Settings > Integrations
    Then key is encrypted and stored in IndexedDB
    And initial-sync job is enqueued
    And within 5 minutes 100 SKUs appear in catalog
    And sync_partial_total = 0

  Scenario: Rate limit recovery
    Given Wildberries returns 429 with Retry-After 60
    When the marketplace adapter handles the response
    Then the job is rescheduled with backoff + jitter
    And the user sees status "обновляется, маркетплейс ограничил скорость"
    And no data is lost
```

### Repricer

```gherkin
Feature: Repricer

  Scenario: Dry-run shows projected price changes
    Given 50 SKUs and a "match lowest competitor - 1 ruble" rule
    When user clicks "Запустить dry-run"
    Then repricer_simulation table has 50 rows
    And no SKU on WB has its price changed

  Scenario: Apply respects price floor
    Given a SKU with floor 500 ₽ and competitor at 480 ₽
    And a "match lowest competitor - 1 ruble" rule
    When repricer applies
    Then the SKU price is set to 500 ₽
    And the action is logged with floor_clipped=true

  Scenario: Rollback within 24h
    Given a repricer apply ran at 12:00 changing 30 prices
    When user clicks "Откатить" at 18:00
    Then all 30 SKU prices revert to their pre-apply values
    And an audit entry "repricer_rollback" is created
```

### AI

```gherkin
Feature: AI description generation

  Scenario Outline: Generate descriptions for 3 brand voices
    Given a SKU with title and 5 attribute fields
    And brand voice "<voice>" is selected
    When user requests AI description generation
    Then a description of 600-1200 characters is returned within 8 seconds
    And the description matches tone profile "<voice>"
    And quota_remaining decrements by 1

    Examples:
      | voice       |
      | premium     |
      | playful     |
      | technical   |
```

### Agency

```gherkin
Feature: Agency invite and revoke

  Scenario: Invite client to agency workspace
    Given agency user with Agency tier
    When agency invites a client via email
    And the client accepts and grants access to their WB store
    Then agency dashboard shows the client's SKUs
    And RLS scopes all queries to the client tenant only

  Scenario: Client revokes agency access
    Given an active agency-client link
    When the client clicks "Отозвать доступ"
    Then agency loses read/write to client data immediately
    And any in-flight jobs are frozen with status=revoked
```

### Billing

```gherkin
Feature: Subscription lifecycle

  Scenario: Upgrade Free to Pro
    Given user on Free tier with 8 SKUs
    When user pays 2 990 ₽ via ЮKassa 3DS
    Then subscription becomes Pro within 30 seconds
    And SKU limit is now 200
    And first invoice is stored in S3

  Scenario: Downgrade Pro to Free with quota overflow
    Given user on Pro tier with 80 SKUs
    When user downgrades to Free
    Then a 14-day grace period starts
    And UI shows warning "уберите 70 SKU"
    And after 14 days excess SKUs become read-only
```

**Итого: 14 сценариев** (требование: ≥12).

---

## 4. Performance Optimizations

### 4.1 Caching strategy

**Слои:**
1. **Browser cache:** статика — `Cache-Control: public, max-age=31536000, immutable` для `_next/static/*`.
2. **Redis L1:** TTL 60 с — 5 мин для горячих API.
3. **CDN:** Yandex Cloud CDN перед Next.js (для public pages и static).

**Cache key conventions:**

```
v1:tenant:{tenantId}:sku:{skuId}                 TTL 60s,  inv: on sku.update
v1:tenant:{tenantId}:dashboard:overview          TTL 300s, inv: on any sku/order change
v1:mp:{mp}:competitors:{categoryId}              TTL 1h,   inv: on cron refresh
v1:user:{userId}:profile                         TTL 1h,   inv: on profile.update
v1:ai:prompt:{hash(prompt+model+voice)}          TTL 24h,  inv: never (immutable)
v1:repricer:rules:{tenantId}                     TTL 600s, inv: on rule.update
```

**Инвалидация:**
- Pub/sub channel `cache-invalidate` — каждый mutation публикует key glob.
- Stale-while-revalidate в frontend через TanStack Query (`staleTime: 30s, gcTime: 5min`).

### 4.2 DB indexing strategy (PostgreSQL)

| Таблица | Индексы |
|---|---|
| `tenant` | `pk(id)`, `unique(slug)`, `idx(owner_user_id)` |
| `user` | `pk`, `unique(email)`, `idx(tg_id) WHERE tg_id IS NOT NULL` |
| `sku` | `pk`, `unique(tenant_id, marketplace, external_id)`, `idx(tenant_id, status)`, `gin(attributes)` |
| `order` | `pk`, `idx(tenant_id, created_at DESC)`, `idx(tenant_id, marketplace, status)`, partial `idx(tenant_id) WHERE status='active'` |
| `repricer_rule` | `pk`, `idx(tenant_id, priority DESC)`, `idx(sku_id) WHERE active` |
| `repricer_log` | `pk`, `idx(sku_id, applied_at DESC)`, `BRIN(applied_at)` |
| `vault_metadata` | `pk`, `unique(user_id, key_handle)` |
| `audit_log` | `pk`, `idx(tenant_id, created_at DESC)`, `BRIN(created_at)` |
| `subscription` | `pk`, `unique(tenant_id) WHERE active`, `idx(stripe_customer_id)` |
| `agency_link` | `pk`, `unique(agency_tenant_id, client_tenant_id)` |

**Партиционирование:** `audit_log` и `repricer_log` — partitioned by `RANGE(created_at)` помесячно, drop партиций старше 12 мес.

### 4.3 ClickHouse materialized views

Цель — для горячих агрегатов читать с CH вместо PG.

| MV | Источник | Гранулярность | Назначение |
|---|---|---|---|
| `sku_metrics_hourly` | `sku_events` | 1 ч × sku | Дашборд "продажи последний час" |
| `sku_metrics_daily` | `sku_metrics_hourly` | 1 д × sku | Тренды 7/30/90 дней |
| `price_history_daily` | `price_events` | 1 д × sku | Графики цен |
| `competitor_positions_hourly` | `position_events` | 1 ч × sku × competitor | Buy-Box-like логика |
| `ai_usage_daily` | `ai_events` | 1 д × tenant × model | Билинг и квоты |

**Движок:** `AggregatingMergeTree` с `SimpleAggregateFunction(sum, ...)`.

### 4.4 Bundle splitting

- **Route-based code split:** Next.js `app/` дробит автоматически.
- **Vendor chunk:** TanStack/React/Zustand в отдельный chunk; ожидаемый размер <120 KB gz.
- **Lazy load:** Recharts, AI panel, Repricer modal — `next/dynamic({ ssr: false })`.
- **Bundle budget:** ошибка билда если `_app` > 200 KB gz, `route chunk` > 80 KB gz.

### 4.5 Image CDN + responsive формы

- Источник: `images.yandex.cloud/easycomm-prod/...`.
- На лету: `?w=200&h=200&fmt=webp&q=85`.
- В `<Image>` Next.js — `srcSet` для 320/640/1280, AVIF fallback WebP fallback JPG.

### 4.6 Background prefetch (Chrome Extension)

- При открытии товара на WB — extension в background fetcher тянет `/api/sku/lookup?wb_id=...` и кладёт в `chrome.storage.session` на 5 мин.
- Если юзер кликает "Открыть в easycomm" — данные уже есть, переход без spinner.

---

## 5. Security Hardening

### 5.1 Input validation

- Все API ручки валидируют входы через **Zod** перед бизнес-логикой.
- На фронте — `react-hook-form + zodResolver`.
- HTML в описаниях карточек — sanitization через **DOMPurify** с allowlist тегов (`p, ul, ol, li, strong, em, br`).
- Имена файлов — нормализация до `[a-z0-9_-]+\.\w{1,5}`, max 255.

### 5.2 Rate limiting

| Скоуп | Лимит | Implementation |
|---|---|---|
| Per-IP unauthenticated | 60 req/min | `@fastify/rate-limit` с Redis store |
| Per-user authenticated | 600 req/min | tenant+user key |
| Per-tenant write API | 60 req/min | защита от мульти-аккаунтного abuse |
| Auth endpoints (login, reset) | 10 req/min per IP | sliding window |
| AI endpoints | per-tier quota + 5 req/s burst | quota guard + token bucket |

### 5.3 Browser security headers

```
Content-Security-Policy: default-src 'self'; script-src 'self' 'wasm-unsafe-eval';
  style-src 'self' 'unsafe-inline'; img-src 'self' data: https://images.yandex.cloud;
  connect-src 'self' https://api.easycomm.world wss://api.easycomm.world;
  frame-ancestors 'none'; base-uri 'self'; form-action 'self'
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), microphone=(), camera=(), payment=(self)
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```

### 5.4 SSRF prevention (image proxy)

- DNS resolve до запроса; reject если IP в `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `127.0.0.0/8`, `169.254.0.0/16`, IPv6 link-local.
- Только https, max redirect = 3, timeout 5 c, max response 10 MB.
- Content-Type whitelist: `image/jpeg|png|webp|avif`.

### 5.5 Audit log integrity

- Append-only таблица `audit_log` (`REVOKE UPDATE, DELETE FROM api_user`).
- Каждая запись содержит `prev_hash` + `payload_hash` (SHA-256); первая запись tenant-а — genesis.
- Cron-job ежедневно валидирует цепочку и пишет signed receipt в S3 Object Lock.
- Чтение audit — только через signed view с `tenantId` фильтром.

### 5.6 Secrets rotation

| Секрет | Период ротации | Механизм |
|---|---|---|
| JWT signing key | 30 дней | Rotating key set (kid); старый валиден 7 дней |
| DB passwords | 90 дней | Vault → docker-compose env reload |
| S3 access keys | 90 дней | IAM dual-key |
| ЮKassa shop secret | 180 дней или при инциденте | Ручная ротация |
| MCP API keys | 90 дней | env reload |
| WB/Ozon API keys пользователей | По действию user-а | Vault re-encrypt |

### 5.7 Dependency scanning

- **Snyk** в PR-checks (high+critical блокируют merge).
- **Trivy** для docker images (CI: после build, до push).
- **npm audit --omit=dev** в CI как secondary check.
- Renovate bot еженедельно поднимает PR на patch-апдейты.

---

## 6. Accessibility (WCAG 2.1 AA)

### 6.1 Keyboard navigation

- Все интерактивные элементы достижимы Tab/Shift+Tab в логическом порядке.
- `Escape` закрывает модалки и поповеры.
- `/` фокусирует глобальный search.
- `g d` — go dashboard, `g s` — go SKUs, `g r` — repricer (gmail-style shortcuts).
- Focus ring обязателен (`outline: 2px solid var(--ring)`), не скрывать через `outline:none`.

### 6.2 Color contrast

- Основной текст: контраст ≥ 4.5:1 (проверяется через `@axe-core/playwright` в e2e).
- Графики: не полагаться только на цвет — добавлять паттерн/иконку.
- Темная и светлая темы — обе проходят axe scan.

### 6.3 aria-labels

- Все icon-only кнопки имеют `aria-label`.
- Live regions для toast (`role="status"`, `aria-live="polite"`).
- Таблицы: `<th scope="col">`, `aria-sort` на сортируемых колонках.
- Modal: `role="dialog"`, `aria-modal="true"`, focus trap.

### 6.4 Screen reader checklist

- [ ] NVDA + Firefox, JAWS + Chrome, VoiceOver + Safari — все top-8 journeys
- [ ] Поля формы озвучивают label, описание, ошибку
- [ ] Динамические обновления (новые SKU прилетели) озвучиваются через live region
- [ ] Таблицы навигируются по строке/колонке
- [ ] Графики имеют `<desc>` с краткой текстовой сводкой

---

## 7. Technical Debt — initial register

| # | Title | Severity | Создан | Owner | Plan |
|---|---|---|---|---|---|
| TD-001 | Repricer logic в монолите — выделить в отдельный пакет с публичным API | medium | Phase 3 | core team | После 1k DAU |
| TD-002 | Chrome Extension manifest v3 service worker лимиты — миграция на offscreen API | high | Phase 3 | extension team | До v1 |
| TD-003 | ClickHouse cluster — однонодовый MVP; миграция на shard+replica при 50M rows | high | Phase 3 | data team | При 30M rows |
| TD-004 | Marketplace adapters: дублирование retry-логики между WB/Ozon/ЯМ → общий middleware | low | Phase 3 | core team | Q1 после MVP |
| TD-005 | i18n инфраструктура: hardcoded ru-RU; не готовы под en-EN/uz-UZ | low | Phase 3 | frontend | По спросу |
| TD-006 | Тесты: моки fetch напрямую вместо MSW — заменить | medium | Phase 3 | QA | Sprint 2 после MVP |
| TD-007 | Audit log hash chain валидирует cron ежедневно — добавить on-write проверку | medium | Phase 3 | security | После аудита |
| TD-008 | Биллинг: реализован только ЮKassa, ProdamusGate резерв не подключён | high | Phase 3 | billing | При первом ЮKassa инциденте |
| TD-009 | Telegram-бот живёт в монолите — выделить как worker | low | Phase 3 | core team | При 5k DAU |
| TD-010 | AI prompt versions: захардкожены в коде — вынести в DB с A/B | medium | Phase 3 | AI team | После 1k AI users |
| TD-011 | Нет распределённого трейсинга в Chrome Extension → API | medium | Phase 3 | platform | До Q2 |
| TD-012 | Migrations: нет автоматического smoke-теста после `migrate deploy` | high | Phase 3 | DevOps | Sprint 1 после MVP |

---

## 8. 152-ФЗ Compliance Checklist

| # | Требование | Статус | Реализация |
|---|---|---|---|
| 1 | Хранение ПДн в РФ (юрисдикция) | ✅ обязательно | Yandex Cloud RU / Selectel RU; явное согласие в Terms |
| 2 | Согласие на обработку ПДн | ✅ | Checkbox при регистрации + версионирование (audit) |
| 3 | Уведомление РКН об обработке | ⚠ требуется | Подать форму до запуска prod |
| 4 | Право на доступ к своим ПДн | ✅ | `/settings/privacy/export` — JSON dump |
| 5 | Право на удаление | ✅ | `/settings/privacy/delete-account` — soft-delete 30д, потом hard-delete |
| 6 | Право на исправление | ✅ | Профиль editable |
| 7 | Уведомление о субподрядчиках | ✅ | Privacy Policy перечисляет: ЮKassa, Unisender, MCP providers (если применимо) |
| 8 | Защита от НСД | ✅ | TLS 1.3, AES-GCM в vault, RBAC, MFA опционально |
| 9 | Журналирование действий с ПДн | ✅ | `audit_log` append-only с hash chain |
| 10 | Уведомление об инциденте за 24/72 ч | ✅ | Runbook + dedicated email security@ |
| 11 | DPO (опционально для нашего масштаба) | ⏸ Phase 2 | Назначить при достижении 100k users или enterprise клиента |
| 12 | Cross-border transfer | ⚠ | Если используем OpenAI MCP — требуется отдельное согласие |
| 13 | Возраст обработки несовершеннолетних | n/a | B2B — only adults |
| 14 | Кооперация с РКН при проверке | ✅ | Контакт security@; чек-лист в `docs/legal/rkn-readiness.md` |

---

## 9. Anti-Patterns to Avoid (project-specific)

| Anti-Pattern | Почему плохо |
|---|---|
| Отправлять plaintext API-ключ маркетплейса на сервер для "удобства" | Нарушает core security promise (client-side vault); discovery — нулевая ценность продукта |
| Хранить мастер-пароль в localStorage / cookie | Плейн-доступ из любого XSS; ломает zero-knowledge |
| Использовать `eval` или `new Function(...)` для динамических repricer-правил | RCE по pattern; правила должны быть данными (DSL → AST) |
| Один большой `pages/api/*.ts` под всё | Tight coupling; невозможно тестировать; разнесите по features в Fastify routes |
| Direct SQL через `prisma.$queryRawUnsafe` без обёртки с `tenantId` | Cross-tenant leak; ESLint правило блокирует |
| Background job без idempotency key | Дубли при ретраях; биллинговые ошибки |
| Skipping migrations review (auto-merge Renovate для prisma/*) | Может уронить prod; миграции всегда manually reviewed |
| AI completion без guardrails + повторный показ на UI без sanitize | Prompt injection → XSS; rendering без DOMPurify запрещён |
| Логировать API key в трассировке (даже на dev) | Утечка через Sentry/OTel; mask-rule в pino redactor обязательно |
| Использовать `@ts-ignore` без TODO + issue link | Технический долг без owner-а |
| Хранить JWT в `localStorage` | XSS-уязвимо; используем httpOnly cookie + SameSite=Lax + CSRF token |
| Запросы в маркетплейс из foreground (UI thread) с ключом из vault | Vault unlocked видим в network tab; всегда — через service worker + signed request |
| Тестировать на prod-маркетплейсе живыми ключами | Влияет на цены реальных юзеров; для contract — sandbox + recorded |

---

## 10. Summary

- **38 edge cases** покрыты по 8 областям.
- **6 уровней тестов**: unit, integration, e2e, contract, property-based, performance + security.
- **14 Gherkin сценариев** для критических флоу.
- **5 слоёв кэширования** (browser, TanStack, CDN, Redis, CH MV).
- **10 пунктов security hardening** (input, RL, headers, SSRF, audit, rotation, deps, ...).
- **WCAG 2.1 AA** через axe + ручные SR-проверки.
- **12 known TD items** с owner-ами и планом.
- **14-пунктный 152-ФЗ чек-лист**.
- **13 проектно-специфичных anti-patterns** для code review.

Следующий шаг — Phase 5 (Completion): развернуть всё это в эксплуатацию.

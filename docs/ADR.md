# Architecture Decision Records — easycomm-world

> SPARC Phase: **Architecture** companion. Каждый ADR фиксирует:
> ID, Title, Status, Context, Decision, Consequences (+ pros / − cons),
> Alternatives Considered. ADR-формат — упрощённый Michael Nygard, расширен
> разделом Alternatives.
> Source documents referenced: `docs/Product_Discovery_Brief.md`,
> `docs/research/predecessor-analysis.md`, `docs/Architecture.md`,
> `docs/C4_Diagrams.md`.

---

## ADR-001 — Distributed Monolith vs Microservices vs Full Monolith

- **Status**: Accepted
- **Date**: 2026-05-14

### Context

Команда стартапа (3–6 инженеров) строит next-generation платформу
с 10 доменными модулями (auth, vault-proxy, connections, products,
analytics, repricer, ai, multichannel, agency, billing, audit).
Constraint: deploy на VPS через Docker Compose, без managed K8s.
Нагрузка MVP: ≤ 500 активных tenant'ов, ≤ 200 q/s peak.

Прародители категории (Helium 10, Sellics, Rithum) исторически
архитектурно эволюционировали "монолит → service-oriented" по мере
роста. На стадии MVP они все были монолитами.

### Decision

Реализуем **Distributed Monolith в монорепозитории**:

- Один deployable бэкенд (`core-service`) с модульной декомпозицией.
- Отдельные worker-контейнеры (BullMQ consumers), делящие кодовую
  базу с core.
- Отдельные spec-сервисы (`mcp-proxy`, `telegram-bot`, `api-gateway`)
  для изолированных fault-domain'ов и независимого scaling.

### Consequences

**Pros (+)**:
- Один CI/CD pipeline, одна миграция БД, единый rollback.
- Shared types между frontend и backend через монорепу
  (`@easycomm/shared-types`).
- Минимальный operational overhead — 1 операционная схема для всего.
- Лёгкий future migration path на микросервисы (модули уже изолированы
  через `exposedService` interfaces).
- Воркеры независимо масштабируются — критично для долгих ingest/AI задач.

**Cons (−)**:
- Один deploy core-service ↓ deploy frequency у крупных команд.
- Soft module boundaries — высокий риск coupling при недисциплине.
- На > 5k tenant понадобится разрыв (см. Architecture §9.4).

### Alternatives Considered

| Альтернатива | Почему отклонена |
|---|---|
| **Full monolith** (всё в одном процессе) | Long-running парсинг и AI-задачи блокируют HTTP-цикл; OOM в воркере положит UI. |
| **Микросервисы** (10+ сервисов с дня 1) | Operational overhead неприемлем для команды 3–6; межсервисная сеть на VPS без mesh — отдельный класс проблем. |
| **Serverless** (Lambda / YC Functions) | Нарушает constraint VPS + Docker; long-running парсинг дорог в FaaS; cold-start неприемлем для seller-UX. |
| **Service-oriented (3–4 сервиса)** | Прежневременная декомпозиция; модули **должны** созреть прежде, чем отделяться. |

---

## ADR-002 — Next.js vs Remix vs Nuxt for Frontend

- **Status**: Accepted
- **Date**: 2026-05-14

### Context

Frontend — главное seller-приложение с dashboard, analytics,
repricer UI, agency console. Требования:
- SSR/RSC для быстрого TTFB на аналитических страницах.
- TypeScript first-class.
- Огромное число форм, таблиц, dashboard'ов — нужна зрелая
  экосистема UI.
- Web Crypto API для client-side vault.
- TanStack Query + Zustand state (зафиксировано в Phase 0).

Russian SaaS-сообщество 2025 преимущественно использует Next.js
(70%+ новых проектов).

### Decision

Используем **Next.js 15 с App Router (React Server Components)**.

### Consequences

**Pros (+)**:
- RSC снижает client bundle и TTFB на analytics-страницах.
- App Router + server actions упрощают form-handling.
- Огромная экосистема + shadcn/ui официально поддерживает Next.js.
- Большой пул разработчиков на российском рынке.
- Vercel-style deploy при необходимости (но мы делаем self-host через standalone build).

**Cons (−)**:
- App Router всё ещё имеет шероховатости в edge-cases
  (caching, streaming).
- "Magic" реактивности RSC требует обучения команды.
- Vendor влияние Vercel на roadmap (но open-source MIT — не блокирует).

### Alternatives Considered

| Альтернатива | Почему отклонена |
|---|---|
| **Remix** | Меньше экосистема UI-kit'ов; в РФ найм сложнее; RSC pattern у Next.js считается передовым. |
| **Nuxt 3** (Vue) | Vue-разработчиков на seller-tools меньше; vault-крипта и сложные dashboards проще на React; shadcn/ui — React-only. |
| **SvelteKit** | Маленькая команда, low risk-tolerance — Svelte ecosystem пока меньше для enterprise-uses. |
| **Vanilla React SPA + Vite** | Нет SSR → плохой TTFB на analytics, плохой SEO для marketing landing. |

---

## ADR-003 — Fastify vs NestJS vs Hono for Backend

- **Status**: Accepted
- **Date**: 2026-05-14

### Context

Backend — Fastify-modular monolith с 10 модулями, Prisma ORM,
BullMQ producer. Нужны:
- High throughput (target 2000 req/s на 4 vCPU).
- Schema-driven validation для API contract'ов.
- TypeScript first-class.
- Plugin-ecosystem (JWT, rate-limit, CORS, OpenAPI).

### Decision

Используем **Fastify 4** с zod для schema validation и
`fastify-type-provider-zod` для type-safe routes.

### Consequences

**Pros (+)**:
- Быстрее Express в 2–3 раза, на уровне Hono на Node-runtime.
- Plugin-architecture естественно ложится на 10 доменных модулей.
- Schema-driven validation (JSON Schema + zod) сразу даёт OpenAPI.
- Полная экосистема (`@fastify/*`) — JWT, rate-limit, CORS, multipart,
  websocket, autoload — всё первоклассно.
- Стабильность и production-tested (Walmart, MAERSK).

**Cons (−)**:
- Меньше "магии" чем NestJS — нет встроенных DI-decorators (но мы
  предпочитаем явный DI через construct-time wiring).
- Кривая обучения для тех, кто пришёл с Express.

### Alternatives Considered

| Альтернатива | Почему отклонена |
|---|---|
| **NestJS** | Heavyweight DI + decorators; "magic" затрудняет debugging; runtime overhead 20–30%; opinionated архитектура мешает гибкости в модулях. |
| **Express** | Slower; нет встроенной schema validation; legacy callbacks; меньше TS-friendly. |
| **Hono** | Edge-focused; Node-on-Hono plugins пока не production-grade; ecosystem не покрывает наших нужд (BullMQ, Prisma integrate-points). |
| **Koa** | Тонкий, но без plugin ecosystem уровня Fastify — пришлось бы собирать всё руками. |

---

## ADR-004 — PostgreSQL + ClickHouse vs Single-DB (TimescaleDB)

- **Status**: Accepted
- **Date**: 2026-05-14

### Context

Данные делятся на два класса:
- **OLTP**: tenant'ы, users, products, policies, subscriptions —
  транзакционные, ACID, FK, не очень большие (≤ 100 GB на 5k tenant).
- **Time-series**: snapshots цен/позиций/продаж по SKU × marketplace × день.
  Один tenant с 1000 SKU × 4 marketplace × 365 дней = 1.4M точек/год.
  На 5000 tenant'ов = 7B точек/год. Read-heavy для dashboard'ов.

Прародители категории — Helium 10 Profits Dashboard, Jungle Scout AccuSales —
все имеют выделенный time-series store.

### Decision

Используем **two-store polyglot**:
- **PostgreSQL 16** для OLTP (с RLS, pgvector).
- **ClickHouse 24** для time-series snapshots и analytics aggregates.

### Consequences

**Pros (+)**:
- ClickHouse column-store даёт **10×** compression vs PG для snapshot-данных.
- Analytics-запросы на млрд строк — секунды вместо минут.
- PG остаётся "узким" и быстрым для OLTP — нет аналитической нагрузки на primary.
- Independent scaling: CH cluster vs PG read-replicas.

**Cons (−)**:
- Два backup pipeline, два monitoring stack, два DBA-skill.
- ETL/sync между PG и CH (через workers) — дополнительная сложность.
- Нет cross-DB join'ов (но они и не нужны — analytics не join'ит OLTP).

### Alternatives Considered

| Альтернатива | Почему отклонена |
|---|---|
| **TimescaleDB** (PG extension) | Кратно дороже storage чем CH на тех же объёмах; hypertable INSERT slower при > 100k rows/s; одна точка отказа (БД), нет независимого scaling. |
| **Только PostgreSQL** (партиционирование) | Не справится с time-series объёмами на > 1k tenant; dashboards будут медленные. |
| **InfluxDB** | Меньшая community, более слабый SQL-интерфейс; cardinality limits на series keys. |
| **DuckDB embedded** | Read-only, нет distributed scenarios, нет multi-writer. |

---

## ADR-005 — Client-side Vault via Web Crypto API (non-negotiable)

- **Status**: Accepted (constraint from Phase 0)
- **Date**: 2026-05-14

### Context

API-ключи маркетплейсов (WB, Ozon, ЯМ, Megamarket) — самый чувствительный
актив пользователя. Утечка ключа = доступ ко всему магазину, включая
изменение цен, удаление карточек, экспорт заказов.

Ни один российский конкурент (MPSTATS, Moneyplace, MarketGuru, Easy
Commerce CAT) не предлагает client-side encryption. У всех ключи живут
в БД сервиса в plaintext или с обратимым шифрованием. Это — Blue Ocean
дифференциация (ELIMINATE pillar в Phase 0).

Phase 0 constraint:
```
storage: "Encrypted IndexedDB (AES-GCM 256-bit) на клиенте"
key_derivation: "PBKDF2 от мастер-пароля пользователя (100k+ итераций)"
server_side: "API ключи маркетплейсов НИКОГДА не уходят на бэкенд"
```

### Decision

Реализуем **client-side vault через Web Crypto API**:
- Мастер-пароль вводится пользователем при login.
- `vault_key = PBKDF2-SHA256(master_password, user_salt, 100k iters)`.
- API-ключи: `ciphertext = AES-GCM-256(vault_key, iv, plaintext_json)`.
- Сервер хранит только `(ciphertext, iv, version)` как opaque BYTEA.
- Auto-lock 15 минут неактивности.
- Для server-side операций (repricer worker) браузер выдаёт worker'у
  **ephemeral credential ≤ 60 сек** через signed JWT-style token.

### Consequences

**Pros (+)**:
- Утечка БД сервера → ноль скомпрометированных ключей маркетплейсов.
- Маркетинговое преимущество: единственный игрок в РФ с этой моделью.
- Снижает compliance scope для нас (не храним секреты — нет требований к KMS).

**Cons (−)**:
- UX: пользователь должен помнить master password; забыл = vault lost (документировано в onboarding).
- Backup-сценарий сложнее: vault мигрирует между устройствами через шифрованный blob, но key — только у юзера.
- Server-side repricer требует unlocked vault в активной сессии → background-only-режим невозможен без явного "agent mode" с saved credentials (Phase 2, отдельный consent flow).
- Web Crypto API throttling в некоторых браузерах при больших объёмах — mitigated через Web Worker.

### Alternatives Considered

| Альтернатива | Почему отклонена |
|---|---|
| **Server-side encryption with KMS** | Сервер всё равно имеет доступ к ключам → не решает проблему "утечка БД = утечка ключей". |
| **OAuth flow с marketplaces** | WB и Ozon API на 2026 не поддерживают full OAuth для seller-tools — только Bearer-token (API key). |
| **Hardware-backed (WebAuthn для key wrap)** | Не у всех пользователей есть аппаратный ключ; UX-overhead высокий. |
| **End-to-end через passkey** | Зрелость технологии в РФ на 2026 ещё низкая; добавим в Phase 2 как опцию. |

---

## ADR-006 — BullMQ vs RabbitMQ vs Kafka for Background Work

- **Status**: Accepted
- **Date**: 2026-05-14

### Context

Background-нагрузка состоит из:
- Marketplace ingest (cron-trigger, per-tenant, rate-limited).
- Repricer runs (user-initiated, per-tenant, priority-aware).
- AI dispatch (per-task, latency-sensitive).
- Alert digest (cron + reactive).
- Multichannel sync (user-initiated, fan-out).

Объём на MVP: ≤ 100k jobs/day, peak 500 jobs/min.

### Decision

Используем **BullMQ (Redis-backed)** для всех очередей.

### Consequences

**Pros (+)**:
- Redis уже в стеке (cache + rate-limit) — нет нового инфра-зверя.
- TypeScript-first API, отличное DX.
- Поддержка delayed jobs, cron, priority, repeatable jobs, flow patterns.
- Bull Board UI для observability из коробки.
- Active maintenance.

**Cons (−)**:
- Не подходит для > 50k jobs/sec (Redis bandwidth bound). Для нас это
  на 100× выше MVP threshold.
- При сбое Redis теряются in-flight jobs (mitigated AOF + RDB persistence).
- Нет first-class stream-processing — для AccuSales-style analytics
  потребуется добавить отдельный поток (CH ingest напрямую без BullMQ).

### Alternatives Considered

| Альтернатива | Почему отклонена |
|---|---|
| **RabbitMQ** | +1 инфра-зверь (отдельный broker, кластеризация Erlang); overkill для нашего объёма. |
| **Kafka** | Stream-platform, не job queue. ZooKeeper + Connect — operational complexity, не оправдан для MVP. |
| **AWS SQS / YC Message Queue** | Vendor lock-in; latency хуже local Redis; egress costs. |
| **Temporal** | Workflow engine, не очередь; добавляет ту же сложность что Kafka. |
| **In-DB queue (PG SELECT FOR UPDATE SKIP LOCKED)** | Конкуренция с OLTP-нагрузкой; нет встроенных delayed/priority semantics. |

---

## ADR-007 — MCP for AI Integration vs Direct API Calls

- **Status**: Accepted
- **Date**: 2026-05-14

### Context

Phase 0 constraint:
```
ai_integration: "MCP servers"
ai_mcp_servers:
  - openai-mcp (gpt-4o for content generation)
  - anthropic-mcp (claude for analytics insights)
  - yandexgpt-mcp (для русскоязычной генерации)
```

Требования: tool-use (вызов наших функций из LLM), cost-tracking,
rate-limit per tenant, fallback при provider-outage, semantic cache.

Jungle Scout анонсировал "Speed to Insight" с AI-обзорами рынка в 2025
(см. predecessor-analysis §3). Это — догоняющий tech-vector.

### Decision

Используем **MCP (Model Context Protocol)** через выделенный
`mcp-proxy` контейнер. Все AI-вызовы проходят через него.

### Consequences

**Pros (+)**:
- Unified tool protocol — switchable providers одним конфигом.
- Tool-use first-class (LLM может вызывать наш `getProductAnalytics()`).
- Централизованный cost ledger, semantic cache, rate-limit.
- Provider fallback chain (openai → anthropic, ratp-limit aware).
- Изолированный fault-domain — AI-outage не валит core-service.

**Cons (−)**:
- MCP на 2026 — relatively new (2024+ standard); меньше прод-документации.
- +1 hop = +20–50 мс latency vs direct call.
- Не все провайдеры одинаково well-supported (YandexGPT MCP wrapper —
  community-maintained на момент 2026, требует мониторинга).

### Alternatives Considered

| Альтернатива | Почему отклонена |
|---|---|
| **Direct SDK calls** (openai-node, anthropic-sdk) | Нет unified abstraction; tool-use каждый провайдер реализует по-разному; cost-tracking растягивается по коду. |
| **LangChain / LlamaIndex orchestration** | Heavyweight, opinionated, частые breaking changes; tool-use не "first-class". |
| **OpenRouter** (commercial proxy) | Vendor lock-in; нет yandexgpt; нет contrl над семантическим кэшем. |
| **Build own protocol** | Изобретение MCP, NIH. |

---

## ADR-008 — Docker Compose on VPS vs K8s vs Managed Cloud

- **Status**: Accepted (constraint from Phase 0)
- **Date**: 2026-05-14

### Context

Phase 0 constraint:
```
infrastructure: "VPS (AdminVPS / HOSTKEY)"
deploy: "Docker Compose direct deploy"
```

Российский регуляторный контекст (152-ФЗ "О персональных данных"):
данные должны храниться на серверах в РФ. Yandex Cloud — opt; AWS/GCP — нет.

Команда MVP — 3–6 инженеров, без выделенного DevOps в первые 6 месяцев.

### Decision

Используем **Docker Compose на VPS** (AdminVPS / HOSTKEY primary).
Blue-green deploy через два compose-файла + nginx upstream switch.

### Consequences

**Pros (+)**:
- Минимальный operational overhead.
- Прозрачный deploy: `docker compose pull && docker compose up -d`.
- Лёгкий rollback (один SIGHUP nginx'у).
- VPS дёшев (₽ 2-5k/мес на старте) vs managed cloud (₽ 30–80k/мес для эквивалента).
- 100% контроль над данными — критично для 152-ФЗ.

**Cons (−)**:
- Single-node = single point of failure (mitigated в Phase 2: multi-node).
- Нет автомасштабирования.
- Логи / метрики надо настраивать руками (prometheus + grafana в Phase 2).

### Alternatives Considered

| Альтернатива | Почему отклонена |
|---|---|
| **Self-hosted K8s (k3s / rke2)** | Operational complexity (etcd, control plane upgrades, networking); 6+ месяцев продуктивной разработки уйдёт в ops. |
| **Yandex Cloud Managed K8s** | Существенный бюджет (₽ 50–100k/мес для prod-сетапа); К8s ноды + control plane + балансировщик; не оправдан для 0–500 tenant. |
| **AWS / GCP** | Нарушает 152-ФЗ для данных российских граждан. |
| **Serverless (Lambda/CF)** | Нарушает constraint; long-running парсинг неприемлем. |
| **Nomad** | Дешевле K8s по сложности, но всё ещё +1 ops уровень без бизнес-выгоды на MVP. |

---

## ADR-009 — Multitenancy via RLS in Single PG vs Schema-per-Tenant vs DB-per-Tenant

- **Status**: Accepted
- **Date**: 2026-05-14

### Context

Multitenancy с двумя уровнями (tenant = продавец/бренд; workspace =
клиент агентства внутри tenant=agency) — наследие Sellics-модели.
Прогноз 5000+ tenant на горизонте 24 месяца.

Open question Q4 из Phase 0: "RLS в Postgres или физическая изоляция
по schema?"

### Decision

Используем **single PostgreSQL DB + Row-Level Security (RLS)** с
`tenant_id` колонкой в каждой tenant-scoped таблице.

### Consequences

**Pros (+)**:
- Один schema, одна миграция — простой ops.
- Cross-tenant analytics (для нас admin) тривиальны.
- Storage-эффективно: shared TOAST, shared indexes per table.
- Connection pooling простой.
- RLS — enforced на уровне PG, защита даже от SQL injection.

**Cons (−)**:
- "Noisy neighbor" — heavy-tenant query может замедлить others (mitigated:
  query_timeout + per-tenant query budget).
- Migrations затрагивают всех — long-running migration блокирует всех (mitigated: expand-then-contract).
- При компрометации БД — все tenant'ы exposed.
- Limit ~5k tenant до необходимости sharding (см. Architecture §9.2).

### Alternatives Considered

| Альтернатива | Почему отклонена |
|---|---|
| **Schema-per-tenant** | N schemas × M tables = быстро упирается в PG catalog limits (~10k tables); миграции — N циклов; pg_dump растёт линейно. |
| **DB-per-tenant** | Operational explosion; backups × N; connection pool fragmentation; нет cross-tenant query. |
| **Hybrid (PG для small tenant + dedicated DB для enterprise)** | Possible Phase 3 эволюция, но overkill для MVP. |
| **Application-level tenant filtering без RLS** | Bug в любом query = data leak. RLS даёт defense-in-depth. |

---

## ADR-010 — Chrome Extension MV3 as Distribution Wedge

- **Status**: Accepted
- **Date**: 2026-05-14

### Context

Phase 0 growth strategy:
```
1. Chrome Extension (как Jungle Scout и MPSTATS) — wedge для нового
   пользователя: free analytics через расширение → upgrade в web app.
```

Jungle Scout (2015) и Helium 10 — отрасль начала именно с Chrome
extension'ов. В РФ MPSTATS и Маяк используют ту же модель.
Chrome MV2 deprecated с 2024 — обязательный MV3.

### Decision

Реализуем **Chrome Extension MV3** как primary distribution channel
для free-tier пользователей. Featureset: overlay с метриками ниши
на страницах WB и Ozon, "import to easycomm-world" CTA.

### Consequences

**Pros (+)**:
- Низкий activation friction (1-click install vs регистрация в SaaS).
- Естественно ведёт к upgrade (overlay показывает, что доступно после signup).
- Распространение через Chrome Web Store + Яндекс.Браузер addons store.
- Прецедент успеха в категории (Jungle Scout, MPSTATS).

**Cons (−)**:
- MV3 service-worker архитектура ограничивает background-tasks (≤ 30 сек).
- Forced auto-update Chrome Web Store: breaking changes API нужны migration plan.
- Scraping DOM WB/Ozon хрупкий — изменение разметки ломает extension (mitigated: feature flag, version pinning, daily integration tests).
- Yandex Browser ↔ Chrome Web Store: дополнительный канал дистрибуции, но extension API совместим.

### Alternatives Considered

| Альтернатива | Почему отклонена |
|---|---|
| **Только web app** | Высокий activation cost; теряем category-standard wedge. |
| **Firefox extension вместо Chrome** | < 5% share в РФ. |
| **Bookmarklet** | Limited API; нет background workers; UX хуже. |
| **PWA вместо extension** | PWA не может инжектить контент в чужие страницы (WB/Ozon dashboard). |

---

## ADR-011 — Telegram for Notifications (Primary) vs Email

- **Status**: Accepted
- **Date**: 2026-05-14

### Context

Phase 0 retention loop:
```
Daily: Telegram-бот → push о position drop / margin shrink → возврат
Weekly: email-дайджест "что упустил конкурент" → CTR обратно в app
```

В РФ Telegram — основной коммуникационный канал для seller'ов: WB-
комьюнити, новости маркетплейсов, отзывы, поддержка. Email open-rates
в e-commerce-сегменте РФ упали до 12–18% (vs Telegram message
delivery >95% + read >70%).

### Decision

Используем **Telegram Bot API как primary канал** для daily-frequency
уведомлений (alerts, digests). Email — secondary для weekly digest и
transactional (password reset, invoice, receipt).

### Consequences

**Pros (+)**:
- Доставка близка к 100%, открываемость 70%+.
- Бесплатно для нас (Telegram Bot API — free tier большая).
- Поддержка inline keyboards = быстрые actions ("approve repricer suggestion") без перехода в app.
- Mini-app embed (Telegram Web App) — для quick views.
- Полезный effect на retention loop #1 (наследие Sellics weekly).

**Cons (−)**:
- Требует от пользователя связать Telegram-аккаунт (one-time consent).
- Telegram заблокирован в некоторых юрисдикциях/корп. сетях (mitigated: email fallback всегда доступен).
- Зависимость от Telegram Bot API — если поменяется политика, нужен план Б.

### Alternatives Considered

| Альтернатива | Почему отклонена |
|---|---|
| **Email primary** | Open-rate 12–18% — недостаточно для retention loop'а. |
| **SMS** | Дорого (₽ 2–4 за SMS × 1000 daily digests = ₽ 60k/мес на 1k users). |
| **Push (web push)** | Нет UX-параллели с inline keyboards; opt-in conversion plохой. |
| **WhatsApp Cloud API** | Недоступен российским юр.лицам без VPN/intermediary; rate-limits жёстче. |
| **VK Notify / VK сообщения** | VK активно у части аудитории, но Telegram доминирует в B2B-seller сегменте. |

---

## ADR-012 — ЮKassa as Primary Billing, Prodamus as Fallback

- **Status**: Accepted
- **Date**: 2026-05-14

### Context

Российский payment landscape 2026:
- **ЮKassa** (от Юmoney/Сбер) — самое широкое покрытие юр.лиц; принимает
  карты МИР + Visa/MC, СБП, ЮMoney.
- **Prodamus** (Prodamus Gate, ПродаМус) — лучший acquirer для ИП и
  самозанятых; быстрый онбординг; meaningful tier для seller-сегмента
  (наши customer'ы в массе ИП).
- **Robokassa** — старый игрок, fees выше, API legacy.
- **CloudPayments** — хороший API, но fees для подписочной модели выше.

Phase 0 monetization:
```
SaaS subscription (Free / Pro 2990₽ / Team 9990₽ / Agency 24990₽)
```

### Decision

**ЮKassa primary** для всех auto-renewing подписок. **Prodamus
fallback** при отказе ЮKassa в обслуживании tenant'а (некоторые ИП
получают отказ от ЮKassa due to compliance).

### Consequences

**Pros (+)**:
- ЮKassa поддерживает recurring billing (саб-подписка через rebilling token).
- Покрытие 95%+ потенциальных customers одним acquirer.
- Prodamus fallback ловит remaining 5% (преимущественно ИП с короткой кредитной историей).
- Webhook-driven architecture — async обработка через workers.

**Cons (−)**:
- Два provider integration = два webhook handler'а, две reconciliation pipeline.
- Fee structures разные (ЮKassa 3.5% + ₽ 30 transaction; Prodamus ≈ 3%).
- При смене tier'а tenant остаётся на своём acquirer'е до renewal — fragmentation in reporting.

### Alternatives Considered

| Альтернатива | Почему отклонена |
|---|---|
| **Stripe** | Не работает с российскими юр.лицами и картами МИР с 2022. |
| **CloudPayments only** | Fees выше; меньшая аудитория знакома с UI. |
| **Только Prodamus** | Корпоративные клиенты (агентства, бренды) предпочитают ЮKassa due to enterprise compliance. |
| **Robokassa** | Legacy API; UX комиссионных страниц устаревший. |
| **Build own merchant integration with банк-эквайером** | 6 месяцев compliance work — не оправдан. |

---

## ADR-013 — Russian-only MVP vs RU+EN Day One

- **Status**: Accepted
- **Date**: 2026-05-14

### Context

Target customer segments (Phase 0):
- Mid-size sellers WB/Ozon (>99% русскоязычные).
- Brands (>95% русскоязычные).
- Marketplace agencies (100% русскоязычные).

Маркетплейсы (WB, Ozon, ЯМ, Megamarket) — russian-only API documentation,
russian customer support, russian seller agreements.

i18n из коробки — 2-3× больше работы по UI текстам, тестам, screenshots.

### Decision

**Russian-only MVP**. Архитектурно — i18n-ready (все строки через
`t('key')`, не hardcoded), но единственный locale на launch — `ru-RU`.

### Consequences

**Pros (+)**:
- Sharper focus на target market.
- Существенно меньше работы UI/content team.
- Marketing-материалы и documentation — одна версия.
- Никаких "пустых переводов" в EN, которые ухудшают user trust.

**Cons (−)**:
- Если в Phase 3 захочется выйти на Kazakhstan/Belarus (тоже WB+Ozon), будет нужна локализация (но архитектура готова).
- Иностранные инвесторы / press могут voice concern (mitigated: marketing landing в `/en` для investor presentations).

### Alternatives Considered

| Альтернатива | Почему отклонена |
|---|---|
| **RU + EN day one** | Удваивает content overhead без бизнес-выгоды; никто из core customers не nuжен EN. |
| **RU + Казахский** | Казахстанские seller'ы используют тот же WB/Ozon, но интерфейс на русском их устраивает. |
| **Multi-locale framework с RU как default** | Architecturally что мы и делаем (i18n-ready); просто не включаем других locale на старте. |

---

## ADR-014 — Free Tier Strategy (10 SKU, 1 Marketplace) vs No Free Tier

- **Status**: Accepted
- **Date**: 2026-05-14

### Context

Phase 0 pricing tiers:
```
Free: 0 ₽ / 10 SKU / 1 marketplace / 50 AI requests/мес / 0 agency seats
Pro:  2 990 ₽ / 200 SKU / до 3 marketplace / 500 AI / 0 agency
Team: 9 990 ₽ / 2 000 SKU / все marketplaces / 5k AI / 3 agency
Agency: 24 990 ₽ / 20k SKU / все / 50k AI / 15 + clients
```

Distribution strategy (Phase 0):
```
1. Chrome Extension — wedge для нового пользователя: free analytics →
   upgrade в web app.
```

Прародители: Helium 10 — Starter $29 (нет полного free); Jungle Scout —
Starter $29 (нет полного free, но 7-day trial). Из competitors в РФ:
MPSTATS — trial, не free; Moneyplace — trial.

### Decision

**Yes, free tier with strict limits**: 10 SKU, 1 marketplace, 50 AI requests/мес.

### Consequences

**Pros (+)**:
- Chrome extension activation funnel замкнут на ноль friction (signup → free → "I want more SKU").
- Дифференциация от MPSTATS/Moneyplace (у которых только trial).
- Долгосрочный organic growth pool: free users рассказывают о продукте.
- 10 SKU достаточно для оценки product fit, но недостаточно для серьёзного бизнеса → естественный upgrade trigger.

**Cons (−)**:
- Infrastructure cost (≥ 50 free users на 1 active VPS workload).
- Risk of abuse (multiple emails per single business) — mitigated через phone-verification на free tier.
- AI requests дорогие — 50/мес лимит критичен; превышение либо graceful degradation, либо force-upgrade prompt.

### Alternatives Considered

| Альтернатива | Почему отклонена |
|---|---|
| **No free tier, 14-day trial** | Уничтожает Chrome extension wedge; противоречит growth-strategy Phase 0. |
| **Freemium с unlimited SKU но без AI** | Infrastructure стоимость взлетит при бесконечных ingest+repricer для free users. |
| **Free tier с 100 SKU** | Слишком ёмко — нет естественного upgrade trigger'а. |
| **Free tier только для Chrome ext (без web app access)** | Меньше insight в фичи; теряется path to upgrade. |

---

## Summary Table

| ADR | Title | Status |
|---|---|---|
| ADR-001 | Distributed Monolith vs Microservices vs Full Monolith | Accepted |
| ADR-002 | Next.js vs Remix vs Nuxt for Frontend | Accepted |
| ADR-003 | Fastify vs NestJS vs Hono for Backend | Accepted |
| ADR-004 | PostgreSQL + ClickHouse vs Single-DB (TimescaleDB) | Accepted |
| ADR-005 | Client-side Vault via Web Crypto API | Accepted (constraint) |
| ADR-006 | BullMQ vs RabbitMQ vs Kafka for Background Work | Accepted |
| ADR-007 | MCP for AI Integration vs Direct API Calls | Accepted |
| ADR-008 | Docker Compose on VPS vs K8s vs Managed Cloud | Accepted (constraint) |
| ADR-009 | Multitenancy via RLS in Single PG vs Schema-per-Tenant vs DB-per-Tenant | Accepted |
| ADR-010 | Chrome Extension MV3 as Distribution Wedge | Accepted |
| ADR-011 | Telegram for Notifications (Primary) vs Email | Accepted |
| ADR-012 | ЮKassa as Primary Billing, Prodamus as Fallback | Accepted |
| ADR-013 | Russian-only MVP vs RU+EN Day One | Accepted |
| ADR-014 | Free Tier Strategy (10 SKU, 1 Marketplace) vs No Free Tier | Accepted |

---

*Конец ADR.md*

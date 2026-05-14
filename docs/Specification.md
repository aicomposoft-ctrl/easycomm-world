# Specification — easycomm-world

> SPARC Phase 1 artifact. Source of truth for functional and non-functional
> requirements. Derived from `docs/Product_Discovery_Brief.md` and
> `docs/research/predecessor-analysis.md`. Consumed by Architecture.md,
> Pseudocode.md, Refinement.md, and Completion.md.

---

## 1. Functional Requirements

Format: `FR-NNN` | Title | Description | Priority (MUST / SHOULD / COULD) |
Source JTBD (Job-1..4 from Brief) | Depends-on.

### 1.1 Identity, Onboarding, Tenancy

| ID | Title | Description | Priority | JTBD | Depends-on |
|----|-------|-------------|----------|------|-----------|
| FR-001 | Email/password signup | User can register via email + master password (min 12 chars, zxcvbn score ≥ 3). Email confirmation via signed token (15 min TTL). | MUST | Job-1..4 | — |
| FR-002 | Sign-in with rate limiting | Login throttled to 5 attempts / 15 min per IP+email tuple; lockout 30 min after 10 failed. | MUST | Job-1..4 | FR-001 |
| FR-003 | Password reset | Recovery via signed email link. **WARNING** banner: reset re-derives a fresh vault key — historical vault is destroyed unless user supplied the old master password. | MUST | Job-1..4 | FR-001 |
| FR-004 | Tenant provisioning | On signup a `tenant` is created. Mode is `solo` by default; can be upgraded to `agency` on subscription change. | MUST | Job-1, Job-4 | FR-001 |
| FR-005 | Tenant member roles | Roles: `owner`, `admin`, `analyst`, `operator`, `client` (agency mode only). RBAC matrix in Architecture.md. | MUST | Job-4 | FR-004 |
| FR-006 | Master-password change | User can rotate master password; client re-encrypts vault locally, server stores only PBKDF2 verifier salt + iteration count. | SHOULD | Job-1..4 | FR-001 |
| FR-007 | Session revocation | Owner can invalidate active sessions for any tenant member. | MUST | Job-4 | FR-005 |
| FR-008 | 2FA TOTP | Optional TOTP (RFC 6238) bound per user; mandatory for `owner` of `agency` tenants. | SHOULD | Job-4 | FR-001 |

### 1.2 Encrypted Vault (Client-Side)

| ID | Title | Description | Priority | JTBD | Depends-on |
|----|-------|-------------|----------|------|-----------|
| FR-010 | Vault creation | First-time vault initialised on signup with PBKDF2-SHA256(masterPwd, salt, 100_000 iter) → AES-GCM 256 key. | MUST | Job-1..4 | FR-001 |
| FR-011 | Vault unlock | UI prompts master password; key kept in memory only. | MUST | Job-1..4 | FR-010 |
| FR-012 | Auto-lock | Vault auto-locks after 15 min of UI inactivity OR explicit lock button OR tab close. | MUST | Job-1..4 | FR-011 |
| FR-013 | Encrypted secret storage | API keys (WB/Ozon/ЯМ tokens, ЮKassa keys, MCP creds) are stored as AES-GCM ciphertext in IndexedDB and (encrypted blob mirror) in PostgreSQL `vault_blob` table for cross-device sync. | MUST | Job-1..4 | FR-010 |
| FR-014 | Vault export | User can export encrypted blob + display the recovery phrase. | SHOULD | Job-1..4 | FR-013 |
| FR-015 | Vault integrity | HMAC-SHA256 over blob; server rejects blobs with non-monotonic version counter (prevents rollback). | MUST | Job-1..4 | FR-013 |
| FR-016 | Server zero-knowledge guarantee | Backend stores ciphertext + salt + iter count; never plaintext key, never master password. Documented in `/security` page. | MUST | Job-1..4 | FR-013 |

### 1.3 Marketplace Connections

| ID | Title | Description | Priority | JTBD | Depends-on |
|----|-------|-------------|----------|------|-----------|
| FR-020 | Connect Wildberries | User pastes Seller API token; client validates against WB `GET /ping` (suppliers-api.wildberries.ru), stores in vault, registers connection. | MUST | Job-1..3 | FR-011 |
| FR-021 | Connect Ozon | User pastes `Client-Id` + `Api-Key`; validated against Ozon Partner API `POST /v1/category/tree`. | MUST | Job-1..3 | FR-011 |
| FR-022 | Connect Яндекс.Маркет | OAuth 2.0 flow with Yandex; campaign-id selection. | SHOULD | Job-1..3 | FR-011 |
| FR-023 | Connect Megamarket | Token-based; validated by `GET /api/merchantIntegration/v1/oauth/info`. | COULD | Job-1..3 | FR-011 |
| FR-024 | Multiple connections per marketplace | A tenant can register N tokens per marketplace (sub-stores, dropship accounts). | SHOULD | Job-1, Job-4 | FR-020 |
| FR-025 | Connection health monitor | Every 5 min cron pings each token's `/ping` endpoint; status surfaced in `/connections`. | MUST | Job-1..3 | FR-020..23 |
| FR-026 | Token rotation | UI flow: paste new token, validate, atomically swap, audit-log event. | SHOULD | Job-1..4 | FR-020..23 |

### 1.4 Product Catalog & Ingestion

| ID | Title | Description | Priority | JTBD | Depends-on |
|----|-------|-------------|----------|------|-----------|
| FR-030 | Initial catalog import | On connection, schedule full SKU pull. WB: `POST /content/v2/get/cards/list`; Ozon: `POST /v2/product/list`. | MUST | Job-1 | FR-020..23 |
| FR-031 | Incremental sync | Cron pulls updated cards every 1h (Pro) or 15 min (Team/Agency) using cursor-based `POST /content/v2/get/cards/list` (WB) and Ozon `updated_at` filter. | MUST | Job-1, Job-3 | FR-030 |
| FR-032 | Sales/orders ingestion | WB `POST /api/v5/supplier/sales`, Ozon `POST /v3/posting/fbo/list` + `POST /v3/posting/fbs/list`. ClickHouse time-series. | MUST | Job-1..3 | FR-030 |
| FR-033 | Stock ingestion | WB `POST /api/v3/stocks/{warehouseId}`, Ozon `POST /v3/product/info/stocks`. | MUST | Job-3 | FR-030 |
| FR-034 | Pricing ingestion | WB `POST /api/v2/list/goods/filter`, Ozon `POST /v4/product/info/prices`. | MUST | Job-3 | FR-030 |
| FR-035 | Card content ingestion | WB `POST /content/v2/get/cards/list` (full attributes), Ozon `POST /v3/product/info/list`. | MUST | Job-2 | FR-030 |
| FR-036 | Master SKU registry | Tenant-wide canonical SKU model linking marketplace-specific listings. | MUST | Job-1, Job-3 | FR-030 |
| FR-037 | Catalog search | Full-text + faceted (brand, category, marketplace, price band, status). | MUST | Job-1..3 | FR-036 |

### 1.5 Analytics Dashboard (CAT analog)

| ID | Title | Description | Priority | JTBD | Depends-on |
|----|-------|-------------|----------|------|-----------|
| FR-040 | Seller overview KPIs | Revenue, GMV, profit, ad-spend, ROAS, refund rate, ACoS — daily/weekly/monthly. | MUST | Job-1 | FR-032 |
| FR-041 | Per-SKU detail | Sales curve, price history, position history, review velocity, buy-box ratio. | MUST | Job-1, Job-3 | FR-032..35 |
| FR-042 | Competitor tracking | Add competitor SKUs by URL; nightly snapshot of price/rating/reviews/position. | MUST | Job-2 | FR-035 |
| FR-043 | Niche overview | Aggregate stats by category × marketplace (median price, top-10 share, entry barriers). | SHOULD | Job-2 | FR-042 |
| FR-044 | Custom reports | User-defined metric × dimension matrices, CSV/XLSX export. | SHOULD | Job-1, Job-4 | FR-040 |
| FR-045 | Profitability drill-down | Cost-of-goods + commissions + logistics + ads → unit-economy per SKU per day. | MUST | Job-1, Job-2 | FR-040 |

### 1.6 Research Tools (Helium-10 analog)

| ID | Title | Description | Priority | JTBD | Depends-on |
|----|-------|-------------|----------|------|-----------|
| FR-050 | Black-box product search | Filter the entire WB+Ozon catalog by ≥ 10 facets (category, price range, monthly revenue est., rating, review count, age). | MUST | Job-2 | FR-032 |
| FR-051 | Sales estimator | Given a SKU URL, return estimated monthly units sold using SalesEstimation algorithm (Pseudocode §3.3). | MUST | Job-2 | FR-042 |
| FR-052 | Reverse-SKU keyword extraction (Cerebro analog) | Given target SKU URL(s), return keywords ranked by where SKU surfaces in WB/Ozon search (positional weighting). | MUST | Job-2 | FR-035 |
| FR-053 | Keyword research (Magnet analog) | Seed-keyword expansion via search-suggestion API + co-occurrence in top-100 cards. | SHOULD | Job-2 | FR-052 |
| FR-054 | Trend detector | Time-series of category-level demand; surface rising niches WoW. | COULD | Job-2 | FR-043 |

### 1.7 AI Tools (MCP-powered)

| ID | Title | Description | Priority | JTBD | Depends-on |
|----|-------|-------------|----------|------|-----------|
| FR-060 | Card description generator | Given SKU + keywords + audience, produce Russian description draft with HTML formatting allowed by WB/Ozon. | MUST | Job-2 | FR-052 |
| FR-061 | Bullet-point optimiser | Suggest 5 selling points, ranked by keyword coverage and emotional triggers. | SHOULD | Job-2 | FR-060 |
| FR-062 | Photo-brief generator | Brief for photographer: shot list, props, model angle, 360° rotations. | COULD | Job-2 | FR-060 |
| FR-063 | Niche AI insights | Natural-language summary "what's changing in your niche this week" via MCP. | SHOULD | Job-1, Job-2 | FR-040, FR-043 |
| FR-064 | AI quota enforcement | Per-tier monthly request counter; soft-warn at 80 %, hard-block at 100 % (allow on overage with upsell modal). | MUST | Job-1..4 | FR-060..63 |
| FR-065 | MCP server registry | Admin UI lists registered MCP servers (openai, anthropic, yandexgpt); per-tenant enable/disable. | SHOULD | Job-4 | FR-060 |
| FR-066 | AI cost attribution | Persist `model`, `input_tokens`, `output_tokens`, `cost_₽` per call in `ai_tool_call` for billing reconciliation. | MUST | Job-4 | FR-064 |

### 1.8 Repricer

| ID | Title | Description | Priority | JTBD | Depends-on |
|----|-------|-------------|----------|------|-----------|
| FR-070 | Repricer rule creation | User defines rule: target (price-match competitor, margin floor, dynamic discount). | MUST | Job-3 | FR-034 |
| FR-071 | Repricer dry-run | Preview new prices over last-7-days simulation; no writes. | MUST | Job-3 | FR-070 |
| FR-072 | Repricer activation | Schedule rule on cron interval (5 / 15 / 60 min). Issues WB `POST /api/v2/upload/task` / Ozon `POST /v1/product/import/prices`. | MUST | Job-3 | FR-070 |
| FR-073 | Repricer safety guards | Hard floor (margin %), hard ceiling, change-rate cap (max ±N % per hour). | MUST | Job-3 | FR-072 |
| FR-074 | Repricer audit trail | Every price change recorded in `repricer_action` with reason + competitors snapshot. | MUST | Job-3, Job-4 | FR-072 |
| FR-075 | Repricer notifications | Telegram/email on each batch result; opt-in. | SHOULD | Job-3 | FR-072 |

### 1.9 Multichannel SKU Sync (Rithum analog)

| ID | Title | Description | Priority | JTBD | Depends-on |
|----|-------|-------------|----------|------|-----------|
| FR-080 | Master-SKU editor | Edit canonical attributes (title, descr, attributes, media) once. | MUST | Job-1 | FR-036 |
| FR-081 | Channel projection | Generate marketplace-specific listing payloads from master SKU; preview before push. | MUST | Job-1, Job-3 | FR-080 |
| FR-082 | Push to marketplace | WB `POST /content/v2/cards/upload`, Ozon `POST /v3/product/import`. Track task ID. | MUST | Job-1, Job-3 | FR-081 |
| FR-083 | Stock projection rules | Per-channel stock allocation (e.g., 60 % WB, 40 % Ozon; or shared pool with reserve). | MUST | Job-3 | FR-033 |
| FR-084 | Cross-channel deduplication | Detect when 2 listings on different marketplaces describe the same physical SKU. | SHOULD | Job-1 | FR-036 |

### 1.10 Agency Mode

| ID | Title | Description | Priority | JTBD | Depends-on |
|----|-------|-------------|----------|------|-----------|
| FR-090 | Client workspace | Each client has an isolated workspace (data segregation via Postgres RLS, see NFR-020). | MUST | Job-4 | FR-004 |
| FR-091 | Invite client | Agency owner invites client email; client joins with restricted role. | MUST | Job-4 | FR-005, FR-090 |
| FR-092 | Permission scopes | Per-feature toggles: `analytics:read`, `repricer:write`, `cards:write`, `billing:read`. | MUST | Job-4 | FR-005 |
| FR-093 | Activity report for client | Auto-generated weekly PDF of actions taken on the client's workspace. | SHOULD | Job-4 | FR-090 |
| FR-094 | Workspace switcher | UI dropdown to switch between tenant's own data and N client workspaces. | MUST | Job-4 | FR-090 |
| FR-095 | Agency-level billing rollup | Agency owner sees aggregated AI/MCP usage across all client workspaces. | SHOULD | Job-4 | FR-066 |

### 1.11 Channels: Telegram Bot, Chrome Extension, Email

| ID | Title | Description | Priority | JTBD | Depends-on |
|----|-------|-------------|----------|------|-----------|
| FR-100 | Telegram bot link | User links Telegram via deep-link `t.me/easycommBot?start=<token>`. | MUST | Job-1, Job-3 | FR-001 |
| FR-101 | Daily digest | 09:00 МСК Telegram message: revenue Δ, top mover, position drops, refund spikes. | MUST | Job-1 | FR-100, FR-040 |
| FR-102 | Real-time alerts | Push Telegram message on: position drop > 10 lines, out-of-stock, repricer rule triggered, AI quota threshold. | MUST | Job-3 | FR-100 |
| FR-103 | Chrome extension v1 | MV3 extension; injects badge on WB/Ozon catalog pages with price/sales/keyword data. | MUST | Job-2 | FR-040 |
| FR-104 | Chrome extension export | One-click "Save SKU to easycomm-world" → adds to competitor tracker. | SHOULD | Job-2 | FR-042, FR-103 |
| FR-105 | Email digest (weekly) | Sunday 20:00 МСК email digest, fallback if Telegram not linked. | SHOULD | Job-1 | FR-001 |

### 1.12 Billing & Subscription

| ID | Title | Description | Priority | JTBD | Depends-on |
|----|-------|-------------|----------|------|-----------|
| FR-110 | Tier selection | Free / Pro 2 990 ₽ / Team 9 990 ₽ / Agency 24 990 ₽ per month. | MUST | Job-1..4 | FR-001 |
| FR-111 | ЮKassa checkout | Hosted-page redirect; receive webhook `payment.succeeded`. | MUST | Job-1..4 | FR-110 |
| FR-112 | Recurring auto-charge | Saved payment method recurring 1× per month per ЮKassa Recurring API. | MUST | Job-1..4 | FR-111 |
| FR-113 | Upgrade / downgrade | Pro-rated changes; new tier active immediately on upgrade, end-of-cycle on downgrade. | SHOULD | Job-1..4 | FR-111 |
| FR-114 | Invoice download | PDF (УПД-формат) per billing cycle. | MUST | Job-1, Job-4 | FR-111 |
| FR-115 | Backup payment provider | ProdamusGate fallback when ЮKassa unavailable. | COULD | Job-1..4 | FR-111 |

### 1.13 Audit & Admin

| ID | Title | Description | Priority | JTBD | Depends-on |
|----|-------|-------------|----------|------|-----------|
| FR-120 | Audit log | Every mutation event recorded with `actor`, `tenant_id`, `workspace_id`, `action`, `target`, `metadata`, `created_at`. Retention 12 months. | MUST | Job-4 | FR-005 |
| FR-121 | Audit log viewer | Filter by actor / action / date range; export CSV. | MUST | Job-4 | FR-120 |
| FR-122 | Admin console | Internal-only; user search, refund, manual subscription override. | SHOULD | — | FR-110 |
| FR-123 | Feature flags | Server-side flags per tenant for canary releases. | SHOULD | — | — |

---

## 2. Non-Functional Requirements

### 2.1 Performance

| ID | Requirement | Threshold |
|----|-------------|-----------|
| NFR-001 | Web app FCP | ≤ 1.5 s p75 on 4G Moscow |
| NFR-002 | Dashboard endpoint p99 | ≤ 800 ms (`GET /api/analytics/overview`) |
| NFR-003 | Catalog list p99 | ≤ 500 ms for 200 SKUs page |
| NFR-004 | Black-box search p99 | ≤ 2 s with 10 facets across ClickHouse |
| NFR-005 | Repricer batch | Apply 1 000 price changes in ≤ 60 s (limited by WB rate-limit 100 req/min) |
| NFR-006 | AI tool roundtrip p99 | ≤ 12 s end-to-end (incl. MCP server latency); UI shows streaming progress |
| NFR-007 | SSE realtime price drop | server-to-client latency ≤ 2 s after detection |
| NFR-008 | Ingestion throughput | 100 000 SKU full pull ≤ 30 min per tenant |
| NFR-009 | API RPS | Sustain 500 RPS / app instance (HTTP layer) |
| NFR-010 | Batch size — multichannel push | up to 1 000 SKUs per push request |

### 2.2 Security & Compliance (152-ФЗ ready)

| ID | Requirement |
|----|-------------|
| NFR-020 | All personal data of Russian users stored in datacentres physically located in RF (Yandex Cloud RF / Selectel RF). Cross-border transfer disabled. |
| NFR-021 | Client-side vault: AES-GCM 256-bit; key derived via PBKDF2-SHA256, min 100 000 iterations, per-user random 16-byte salt. |
| NFR-022 | Auto-lock: master key purged from in-memory after 15 minutes of UI inactivity. |
| NFR-023 | TLS 1.3 mandatory; HSTS `max-age=63072000; includeSubDomains; preload`. |
| NFR-024 | Server-side secrets in HashiCorp Vault or Docker secrets; rotated every 90 days. |
| NFR-025 | Authentication tokens: short-lived JWT (15 min) + rotating refresh token (30 days); both `HttpOnly; SameSite=Lax; Secure`. |
| NFR-026 | Row-Level Security (Postgres RLS) on every multi-tenant table; policies keyed by `tenant_id` and (when agency mode) `workspace_id`. |
| NFR-027 | Audit log immutable: APPEND-only table, weekly archival to S3 with object-lock 12 months. |
| NFR-028 | PII redaction in application logs (emails partial-masked, no API keys, no tokens). |
| NFR-029 | Bcrypt (cost 12) for password hashes server-side (separately from PBKDF2-derived vault key). |
| NFR-030 | CSP `default-src 'self'`; no inline scripts; SRI for 3rd-party CDN. |
| NFR-031 | Dependency CVE scan on every PR (npm audit + Snyk). Block merge on Critical. |
| NFR-032 | Penetration test once per year; remediation SLO 30 days for High. |
| NFR-033 | Data Processing Agreement (DPA) template signed with each Agency tenant. |
| NFR-034 | Right-to-erasure: account deletion purges PII within 30 days, audit log anonymised in place. |
| NFR-035 | Roskomnadzor notification (Notification of personal data processing) filed before commercial launch. |

### 2.3 Scalability

| ID | Target |
|----|--------|
| NFR-040 | 10 000 DAU on a 3-node deployment (≤ 4 vCPU + 8 GB RAM per node). |
| NFR-041 | 100 000 SKUs per tenant supported in catalog and analytics queries. |
| NFR-042 | 1 000 000 time-series points / day ingested per tenant (ClickHouse). |
| NFR-043 | Horizontally scalable app tier behind a load balancer (Nginx). Stateless app pods. |
| NFR-044 | BullMQ workers horizontally scalable; queue sharding by tenant_id hash. |
| NFR-045 | Postgres read-replica supported (failover hot-standby). |

### 2.4 Reliability

| ID | Requirement |
|----|-------------|
| NFR-050 | Uptime: 99.5 % monthly during MVP, raised to 99.9 % by month 12. |
| NFR-051 | RTO ≤ 4 h. |
| NFR-052 | RPO ≤ 1 h (Postgres WAL ship to S3 every 15 min; daily logical dumps). |
| NFR-053 | Idempotency keys on every state-mutating API endpoint (`Idempotency-Key` header). |
| NFR-054 | Circuit breaker on marketplace API integrations; fall back to last-good cache. |
| NFR-055 | Auto-retry with exponential backoff (base 500 ms, max 30 s, jitter 25 %) for marketplace 429/5xx. |

### 2.5 Observability

| ID | Requirement |
|----|-------------|
| NFR-060 | Structured JSON logs with `trace_id`, `tenant_id`, `user_id`, `action`. |
| NFR-061 | OpenTelemetry traces; collector → ClickHouse (Tempo-compatible) for 30-day retention. |
| NFR-062 | RED metrics (Rate / Errors / Duration) per HTTP route, queue, marketplace endpoint. Prometheus + Grafana. |
| NFR-063 | Synthetic monitoring (cron probes from external location) covering 5 critical user journeys. |
| NFR-064 | Alertmanager rules: p99 > 2× target → page; error rate > 1 % → page. |
| NFR-065 | Sentry for frontend + backend error capture; PII scrubbed. |

### 2.6 Localization

| ID | Requirement |
|----|-------------|
| NFR-070 | Primary language Russian (ru-RU); date format dd.mm.yyyy; decimal comma; ₽ currency. |
| NFR-071 | All marketplace integrations natively Cyrillic-safe (UTF-8 throughout). |
| NFR-072 | English locale (en-US) shipped after MVP+6 months; translation via i18next; no hard-coded strings. |
| NFR-073 | Timezone configurable per tenant; default Europe/Moscow. |

### 2.7 Accessibility

| ID | Requirement |
|----|-------------|
| NFR-080 | WCAG 2.1 AA compliance on web app. |
| NFR-081 | Keyboard navigation full-coverage; visible focus rings. |
| NFR-082 | Colour contrast ratio ≥ 4.5:1 for body text. |
| NFR-083 | Screen-reader labels on all interactive controls; ARIA-live for SSE alerts. |
| NFR-084 | No keyboard trap; Esc closes all overlays. |
| NFR-085 | Lighthouse a11y score ≥ 95 on production dashboards. |

---

## 3. User Stories (Gherkin)

12 highest-priority features → 40+ acceptance scenarios.

### Feature: User Onboarding + Vault Setup

```gherkin
US-001  As a new seller, I want to create an account so that I can start using the platform.
  Given I am on /signup
  When I enter a valid email "olga@shop.ru" and master password "S3cret-Passw0rd!#"
   And I accept the 152-ФЗ consent
   And I submit the form
  Then a tenant in "solo" mode is created
   And a confirmation email is sent
   And I am redirected to /onboarding/vault

US-002  As a new user, I want the vault to be initialised so that secrets stay client-side.
  Given my account exists and email is confirmed
   And I am on /onboarding/vault
  When the page loads
  Then the browser derives an AES-GCM key via PBKDF2(masterPwd, salt, 100000)
   And an empty encrypted blob is written to IndexedDB
   And a sync record is created server-side with HMAC and version=1
   And the master password is NEVER transmitted

US-003  As a returning user, I want the vault to auto-lock on inactivity.
  Given the vault is unlocked
   And I have been idle for 15 minutes
  When the inactivity timer fires
  Then the in-memory key is purged
   And the UI shows the unlock modal
   And any background queries requiring the vault are paused

US-004  As a user who forgot their master password, I want a clear warning about data loss.
  Given I am on /reset-password
  When I request a recovery link
  Then the next page shows a banner explaining vault destruction
   And only after I tick "I accept loss of stored secrets" can I proceed
```

### Feature: Connect Marketplace

```gherkin
US-005  As a seller, I want to connect my Wildberries account by API token.
  Given the vault is unlocked
   And I am on /connections/new?marketplace=wb
  When I paste a token starting with "ey..."
   And I click "Validate"
  Then the client calls WB `GET /ping` with the token
   And on 200 the token is encrypted and saved to vault and server-mirror
   And the connection appears in /connections with status="active"

US-006  As a seller, I want to connect Ozon with Client-Id + Api-Key.
  Given the vault is unlocked
  When I enter "Client-Id: 12345" and a valid Api-Key
   And I click "Validate"
  Then a `POST /v1/category/tree` call is made to Ozon
   And on 200 the credentials are encrypted into vault
   And initial catalog import is queued

US-007  As a seller, I want to see when a token has expired.
  Given a connection exists
   And the marketplace returns 401 on next health probe
  When I open /connections
  Then the status is "expired"
   And a CTA "Re-enter token" is shown
   And no further sync is attempted until I rotate the token
```

### Feature: Seller Dashboard (Analytics Overview)

```gherkin
US-008  As a seller, I want a one-screen daily overview.
  Given I am authenticated
   And my tenant has ≥ 1 active connection
   And ≥ 1 day of sales data ingested
  When I open /dashboard
  Then I see KPIs: today's revenue, week-over-week Δ, top mover SKU, anomalies
   And the p99 server response is ≤ 800 ms

US-009  As a seller, I want a per-SKU detail view.
  Given I clicked a SKU in /catalog
  When the detail page loads
  Then I see price history, position history, sales curve, review velocity
   And I can pivot timeframe (7d / 30d / 90d / 365d)
   And data older than 365d is loaded lazily on demand
```

### Feature: Black-Box Product Search

```gherkin
US-010  As a researcher, I want to filter the marketplace catalog by 10+ facets.
  Given I am on /research/black-box
  When I set: marketplace=WB, category="детская одежда", price 500–2000 ₽, revenue est ≥ 100k ₽/month, reviews 50–500, age ≤ 6 months
   And I click "Search"
  Then results stream from ClickHouse
   And the first 50 rows appear within 2 s
   And each row shows: image, title, price, est. monthly revenue, reviews, rating

US-011  As a researcher, I want to save a search to track over time.
  Given I have a search with results
  When I click "Save niche"
  Then a `saved_search` row is created
   And weekly digest will include WoW deltas of the saved search
```

### Feature: Reverse-SKU Keyword Extraction (Cerebro analog)

```gherkin
US-012  As a seller, I want to extract keywords driving a competitor's SKU.
  Given I am on /research/reverse-sku
  When I paste "https://www.wildberries.ru/catalog/12345/detail.aspx"
   And I click "Extract"
  Then the system crawls WB search results for top-100 candidate queries
   And computes positional weighting per query (KeywordReverseLookup algorithm)
   And returns ≥ 50 keywords with position, frequency, relevance score
   And I can export as CSV

US-013  As a seller, I want to combine reverse-SKU lookup of 5 competitors.
  Given I have selected 5 competitor SKUs
  When I click "Compare keywords"
  Then I see a matrix: keywords × competitors with each cell = avg position
   And keywords I'm missing in my own SKU are highlighted in red
```

### Feature: AI-Generated Card Description

```gherkin
US-014  As a seller, I want AI to draft a card description.
  Given I am on /catalog/<sku>/edit/description
   And the vault is unlocked
  When I click "Generate with AI"
   And I select keywords to emphasise
   And I select audience "молодые мамы 25-35"
  Then a MCP call is dispatched (AIToolDispatch algorithm)
   And streaming tokens appear within 2 s
   And the result is shown side-by-side with the current description
   And I can accept, reject, or edit

US-015  As a seller, I want AI usage to count against my tier quota.
  Given my tier is Pro (500 AI requests/month)
   And I have used 499
  When I trigger one more generation
  Then it succeeds
   And the usage counter shows 500/500
   And a soft-warning toast appears

US-016  As a seller, I want my AI usage to be blocked when quota is exhausted (unless I opt-in to overage).
  Given my tier is Pro and I have used 500/500
  When I trigger generation
  Then I see an upsell modal with "Upgrade to Team" and "Continue with overage"
   And no API call is made unless I confirm overage
```

### Feature: Repricer Rules + Dry-Run

```gherkin
US-017  As a seller, I want to define a price-match rule.
  Given I am on /repricer/new
  When I set: target = "match cheapest of 3 selected competitors − 1 ₽"
   And I set margin floor = 18 %
   And I set max change rate = 10 % / hour
   And I select 25 SKUs
   And I click "Save (inactive)"
  Then a `price_rule` row is created with status="dry"

US-018  As a seller, I want to dry-run a repricer rule.
  Given a dry-run rule exists
  When I click "Simulate last 7 days"
  Then the system replays competitor history and shows: # changes, avg Δ, est. revenue Δ
   And no marketplace write occurs

US-019  As a seller, I want to activate the rule after reviewing dry-run.
  Given the dry-run looks acceptable
  When I click "Activate"
  Then status moves to "active"
   And cron job is scheduled at the chosen interval
   And next run will issue WB `POST /api/v2/upload/task`
```

### Feature: Multichannel SKU Sync

```gherkin
US-020  As a multichannel seller, I want to edit a master SKU once.
  Given a master SKU exists with WB + Ozon listings
  When I edit title/description/photos in /catalog/master/<id>
   And I click "Save & project"
  Then channel projections are recomputed
   And I see two preview tabs (WB, Ozon) with red/green diffs

US-021  As a seller, I want to push the new content to both marketplaces.
  Given the projections are reviewed
  When I click "Push to all"
  Then WB `POST /content/v2/cards/upload` and Ozon `POST /v3/product/import` are queued
   And task IDs are stored
   And UI polls task status every 30 s and shows progress

US-022  As a seller, I want stock allocation across channels.
  Given total stock = 100 units
   And channel rule = 60 % WB / 40 % Ozon
  When the sync runs
  Then WB stock = 60, Ozon stock = 40, total available = 100 (anti-oversell)
   And on order from WB, stock is decremented atomically
```

### Feature: Agency Mode — Invite & Scope

```gherkin
US-023  As an agency owner, I want to invite a client.
  Given my tenant is in "agency" mode
   And I have an open seat
  When I enter client email and click "Invite"
  Then a `workspace` row is created
   And the client receives an email with a magic-link
   And the workspace appears in /workspaces with status="pending"

US-024  As an agency owner, I want to scope client permissions.
  Given the client has accepted the invite
  When I open /workspaces/<id>/permissions
   And I uncheck "billing:read" and "repricer:write"
   And save
  Then the client sees no billing data and the repricer page is read-only

US-025  As a client of an agency, I want to see what the agency did on my behalf.
  Given I am a "client" role in the workspace
  When I open /workspace/activity
  Then I see an audit log filtered to the workspace
   And I can export as PDF for monthly review
```

### Feature: Telegram Bot Daily Digest

```gherkin
US-026  As a seller, I want to link my Telegram.
  Given I am on /settings/notifications
  When I click "Link Telegram"
  Then a deep-link `t.me/easycommBot?start=<token>` opens
   And after the user presses Start, the bot stores tg-user-id
   And the UI shows "Linked as @olga_shop"

US-027  As a seller, I want a daily digest at 09:00 MSK.
  Given my tenant has digest enabled
  When the time is 09:00 Europe/Moscow
  Then the AlertDigestCompose algorithm runs for my tenant
   And a Telegram message is sent: revenue Δ, top mover, position drops, refund spikes
   And the message is logged in `notification_log`

US-028  As a seller, I want to mute Telegram for the weekend.
  Given digest is enabled
  When I toggle "Mute weekends"
  Then no Telegram messages are sent Saturday/Sunday
   And realtime alerts respect mute schedule
```

### Feature: Chrome Extension Instant Analytics

```gherkin
US-029  As a researcher browsing Wildberries, I want instant analytics on category pages.
  Given I have the easycomm-world Chrome extension installed and logged in
  When I open a WB category page (e.g., /catalog/zhenshchinam/odezhda)
  Then the extension reads page DOM
   And injects a sidebar with per-SKU: monthly revenue est., review velocity, age
   And clicking a row deep-links to /catalog/<sku> in the web app

US-030  As a researcher, I want to save a competitor SKU from the extension.
  Given I am on a WB SKU page
  When I click the "Save to easycomm-world" button injected by the extension
  Then the SKU is added to the tenant's competitor tracker (FR-042)
   And a toast confirms success
```

### Feature: Subscription Upgrade + Billing

```gherkin
US-031  As a Free-tier user hitting limits, I want to upgrade to Pro.
  Given my tier is Free and I tried to add an 11th SKU
  When the limit modal appears and I click "Upgrade to Pro"
  Then I am redirected to ЮKassa checkout for 2 990 ₽
   And on `payment.succeeded` webhook my tier becomes Pro immediately

US-032  As a paid user, I want recurring auto-charge with predictable receipts.
  Given my subscription is active
  When the billing cycle ends
  Then ЮKassa Recurring API is invoked
   And on success a new `invoice` is created
   And an УПД-format PDF is generated and emailed

US-033  As a paid user, I want to downgrade at end of cycle.
  Given my tier is Team
  When I click "Downgrade to Pro" on /billing
  Then status="downgrade_scheduled" with target tier Pro and effective date = end_of_cycle
   And tier flips automatically at that date without payment retry

US-034  As an agency owner, I want consolidated billing across client workspaces.
  Given my tier is Agency and I have 8 client workspaces
  When I open /billing
  Then I see AI usage breakdown per workspace
   And one single invoice for 24 990 ₽
   And no separate charge per workspace
```

### Additional cross-cutting stories

```gherkin
US-035  As a tenant owner, I want to revoke a session immediately.
  Given another device has an active session
  When I click "Revoke" in /settings/security
  Then the refresh token is rotated server-side
   And the other device receives 401 on next refresh attempt

US-036  As a 152-ФЗ-compliant operator, I want all PII stored in RF.
  Given a Russian user signs up
  When their PII is persisted
  Then the storage region for `users` and `audit_log` tables is RF (Yandex Cloud RF)
   And no cross-border replica exists

US-037  As an admin, I want to feature-flag a risky release for one tenant.
  Given a feature flag "new-repricer" exists
  When I enable it for tenant_id=42
  Then only that tenant sees the new repricer UI
   And rollback is one toggle away

US-038  As any user, I want SSE-pushed price drop alerts in the UI.
  Given I have an open browser tab with /dashboard
   And a competitor drops price > 10 %
  When the detector emits an event
  Then a banner appears in the UI within 2 s (NFR-007)
   And clicking it deep-links to the repricer modal

US-039  As a user, I want my account erased on demand (152-ФЗ).
  Given I request erasure in /settings/account
  When I confirm with master password
  Then within 30 days all PII is purged
   And audit_log entries are anonymised in place (replaced by tenant_id only)

US-040  As an analyst role, I want read-only access to analytics.
  Given my role is "analyst"
  When I open /repricer
  Then I see analytics but all "Save" and "Activate" buttons are disabled
   And API attempts return 403
```

---

## 4. Feature Matrix

Effort estimate scale: S (≤ 2 d), M (3–5 d), L (1–2 w), XL (≥ 3 w).

| FR ID range | Feature group | MVP | v1 (Q+1) | v2 (Q+2) | Effort |
|----|----|----|----|----|----|
| FR-001..008 | Identity / RBAC | ✓ (FR-001..005,007) | FR-006, FR-008 | — | L |
| FR-010..016 | Encrypted vault | ✓ | — | — | L |
| FR-020..026 | Marketplace connections | ✓ WB, Ozon | ЯМ | Megamarket | L |
| FR-030..037 | Catalog & ingestion | ✓ | refinement | full-text faceted | XL |
| FR-040..045 | Analytics dashboard | ✓ (FR-040,041,045) | FR-042, FR-044 | FR-043 | XL |
| FR-050..054 | Research tools | FR-050, FR-051 | FR-052, FR-053 | FR-054 | XL |
| FR-060..066 | AI / MCP | FR-060, FR-064, FR-066 | FR-061, FR-063, FR-065 | FR-062 | L |
| FR-070..075 | Repricer | ✓ (FR-070..074) | FR-075 | — | L |
| FR-080..084 | Multichannel sync | FR-080..082 | FR-083 | FR-084 | XL |
| FR-090..095 | Agency mode | — | ✓ all | — | L |
| FR-100..105 | Telegram / Chrome / Email | FR-100..102 | FR-103, FR-105 | FR-104 | M |
| FR-110..115 | Billing | FR-110..114 | — | FR-115 | M |
| FR-120..123 | Audit & admin | FR-120, FR-121 | FR-122 | FR-123 | M |

---

## 5. Success Metrics

| Metric | Target (MVP+3 mo) | Target (MVP+12 mo) |
|--------|------------------|--------------------|
| Signups / month | 1 000 | 5 000 |
| Activation rate (signup → vault unlocked → 1st connection) | ≥ 35 % | ≥ 50 % |
| Pro tier conversion | ≥ 5 % of activated | ≥ 12 % |
| Churn (Pro+) | ≤ 8 % / mo | ≤ 5 % / mo |
| MRR | 1 M ₽ | 8 M ₽ |
| ARR | 12 M ₽ | 80 M ₽ |
| CAC payback | ≤ 4 mo | ≤ 2.5 mo |
| LTV / CAC | ≥ 4 | ≥ 10 |
| Daily active sellers (DAU) | 500 | 3 500 |
| AI requests / paid user / month | 50 | 300 |
| Repricer-active SKUs | 10 k | 200 k |
| Chrome ext installs | 2 000 | 30 000 |
| p99 dashboard | ≤ 1 200 ms | ≤ 800 ms |
| Uptime | 99.5 % | 99.9 % |
| WCAG 2.1 AA Lighthouse score | ≥ 90 | ≥ 95 |

---

## 6. Compliance Requirements

### 6.1 152-ФЗ "О персональных данных"

- Notify Roskomnadzor before launch (form Уведомление о намерении).
- Store PII of RF residents in RF-located datacentres (Yandex Cloud RF preferred; Selectel RF fallback).
- Consent screen on signup with explicit checkboxes per processing purpose.
- DPA template available; appoint Data Protection Officer (DPO) before reaching 5 000 active users.
- Right-to-access, right-to-rectification, right-to-erasure UI flows.
- Breach-notification procedure (≤ 24 h to Roskomnadzor, ≤ 72 h to affected users).

### 6.2 Wildberries Seller API ToS

- Token usage tied to the account that issued it; no token-sharing.
- Respect rate limits: 100 req/min default, 300 req/min for some endpoints; observe `X-Ratelimit-Remaining` header.
- Do not scrape closed-portal pages with the seller token; only documented endpoints.
- Persist audit trail of writes (price/stock/card updates).

### 6.3 Ozon Partner ToS

- `Client-Id` + `Api-Key` pair must match the registered partner account.
- Honour `X-O3-App-Name` + `X-O3-App-Version` headers (we register `easycomm-world/<semver>`).
- Rate limits per endpoint family (varies 5–120 req/min); back-off on 429.
- Do not modify another seller's listings (multichannel sync only for own SKUs).

### 6.4 Яндекс.Маркет Partner ToS

- OAuth 2.0 only; no static API tokens.
- Scope minimization principle: request only `cards:read`, `cards:write`, `orders:read`, `prices:write`.

### 6.5 Marketplace ToS — general

- No automated bidding outside documented Promotion APIs.
- No off-platform competitor scraping that violates each marketplace's `robots.txt` or rate-limit headers.
- Display "Powered by easycomm-world" disclosure on Chrome extension overlays.

### 6.6 Payment Compliance

- ЮKassa integration follows PCI-DSS scope reduction (hosted page; no PAN ever touches our servers).
- 54-ФЗ (online cash register / чеки) — fiscal receipts issued by ЮKassa as fiscal-agent.
- Invoices follow УПД format (Постановление №1137).

---

## 7. Out of Scope (Explicit non-MVP)

- ❌ Native mobile apps (iOS/Android) — Telegram WebApp + responsive web only.
- ❌ Megamarket and Lamoda integrations — post-MVP.
- ❌ FBO / FBS logistics orchestration (creating supply orders).
- ❌ Photo studio booking, copywriter marketplace, ad-buying brokerage (managed-services upsell — handled by parent agency, not platform).
- ❌ Cross-border export advisory (US/EU Amazon).
- ❌ Custom-branded white-label deployments for agencies (post-MVP+12).
- ❌ Public REST API for third parties (post-MVP+12).
- ❌ On-prem / private-cloud deployment.
- ❌ Bookkeeping / 1С integration (post-MVP+6).
- ❌ ML-driven demand forecasting (post-MVP+6; MVP ships with linear-baseline only).
- ❌ Built-in PPC bid management for WB/Ozon retail-media (post-MVP+12; quoted by PARENT brand Sellics but parked here).
- ❌ Review-management workflows (post-MVP+6).

---

End of Specification.md.

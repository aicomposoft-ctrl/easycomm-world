# Product Requirements Document — easycomm-world

| Поле | Значение |
|---|---|
| **Product Name** | easycomm-world |
| **Version** | 0.1 PRD |
| **Date** | 2026-05-14 |
| **Status** | DRAFT for SPARC (Phase 1 output, идёт на Phase 2 validation) |
| **Owner** | Product / Founding team |
| **Input documents** | `docs/Product_Discovery_Brief.md`, `docs/Research_Findings.md`, `docs/Solution_Strategy.md` |
| **Next artifacts** | `docs/Specification.md`, `docs/Architecture.md`, `docs/Pseudocode.md` |

---

## 1. Vision Statement

**easycomm-world — единая платформа для российских marketplace-продавцов,
которая превращает аналитику в действие за один клик.** Мы строим
next-generation реплику западной категории «marketplace seller tools»,
структурно унаследовав гибридную модель **Sellics** (agency + SaaS) и
функциональную глубину **Helium 10**, адаптируем её под Wildberries,
Ozon, Яндекс.Маркет и Megamarket, и **впервые в категории** реализуем
client-side зашифрованный vault для API-ключей маркетплейсов плюс
AI-помощник через MCP-серверы.

Долгосрочная цель — стать **default tool** для продавцов с оборотом
1-20 млн ₽/мес и для marketplace-агентств, обслуживающих этих
продавцов. Мы не строим «ещё одну MPSTATS» — мы убираем разрыв между
данными и действиями, который сейчас вынуждает 100% продавцов держать
2-3 разных сервиса параллельно с кабинетами WB и Ozon.

---

## 2. Goals (OKR-style)

### O1. Захватить нишу «hybrid agency+SaaS» в России

| KR | Target | Deadline |
|---|---|---|
| KR-1.1 | 1 000 платящих Pro+ пользователей | M+12 |
| KR-1.2 | 50 платящих Agency-tier клиентов | M+18 |
| KR-1.3 | 30-80 млн ₽ ARR (SOM из Research_Findings) | M+24 |

### O2. Закрыть time-to-first-action < 5 минут

| KR | Target | Deadline |
|---|---|---|
| KR-2.1 | Median TTFA от регистрации до первого write-API действия | < 5 мин на M+6 |
| KR-2.2 | 7-day activation rate (≥ 1 действие после регистрации) | ≥ 40% на M+6 |

### O3. Обеспечить security-moat (encrypted vault)

| KR | Target | Deadline |
|---|---|---|
| KR-3.1 | 100% API-ключей маркетплейсов хранятся client-side (никогда не уходят на бэкенд) | С дня 1 |
| KR-3.2 | Пройти security-audit от независимой фирмы (PT Security или Solar appScreener) | M+9 |

### O4. Multichannel-coverage от MVP

| KR | Target | Deadline |
|---|---|---|
| KR-4.1 | Поддержка WB + Ozon в MVP | M+3 |
| KR-4.2 | + Яндекс.Маркет + Megamarket | M+9 |
| KR-4.3 | 60% пользователей подключают ≥ 2 МП в течение 30 дней регистрации | M+12 |

### O5. AI-foundation через MCP

| KR | Target | Deadline |
|---|---|---|
| KR-5.1 | ≥ 3 продуктовых модуля имеют AI-companion mode | M+9 |
| KR-5.2 | Median cost-per-AI-action не выше 8 ₽ при сохранении perceived quality | M+12 |

---

## 3. Non-Goals (out of scope)

Явно **не** делаем в горизонте 18 месяцев:

1. **Enterprise tier (>Agency 24 990 ₽)** — крупные ритейлеры
   (Сбер.Мегамаркет 1P, корпорации) требуют контрактных продаж, SLA и
   compliance-обвязки, которые радикально ломают unit economics на
   ранней стадии.
2. **Поддержка не-российских маркетплейсов** (Amazon, eBay, AliExpress
   Global) — потребует Amazon SP-API сертификации, других платёжных
   рельс, английского интерфейса. Возвращаемся при выходе на СНГ-рынок.
3. **Собственный платёжный шлюз для продаж B2C-конечникам продавца** —
   мы не Stripe и не Yookassa. Только tooling layer.
4. **Полностью автономный AI-агент, действующий без подтверждения
   человека** — все write-операции требуют явного approve пользователем
   (регуляторика + risk § Solution_Strategy SO-2).
5. **Mobile-native приложение iOS/Android** до достижения 2 000 MAU
   на web. PWA-режим — да; native — нет.

---

## 4. Personas

### Persona 1 — Mid-size Seller «Анна»

| Параметр | Значение |
|---|---|
| **Возраст** | 34 |
| **Локация** | Краснодар |
| **Образование** | Высшее экономическое |
| **Роль** | Founder + sole operator интернет-магазина (бижутерия, аксессуары) |
| **Оборот магазина** | 6 млн ₽/мес GMV, marge ~25% |
| **Каналы** | Wildberries (70%), Ozon (25%), ЯМ (5%) |
| **SKU** | 80 активных, ассортимент обновляется на 20% за квартал |
| **Команда** | Анна + 1 ассистент-фрилансер на контенте |
| **Tools today** | Excel, MPSTATS Pro (8 990 ₽/мес), Маяк free, кабинеты WB/Ozon, телега-чат продавцов |

**Поведение:**
- Заходит в MPSTATS 3-4 раза в неделю, в WB-кабинет — ежедневно.
- Принимает решения о цене на основе intuition + Excel-таблицы.
- Раз в 2 недели заказывает фотосессию у фрилансера; контент готовит сама.

**Pain:**
- «MPSTATS показывает динамику, но чтобы поменять цену — иду в WB-
  кабинет; в WB-кабинете не видно prognoz и конкурентов».
- «При 80 SKU вручную репрайс невозможен; репрайсер от MPSTATS работает
  только по их правилам — не настроишь».
- «Хочу подключить Яндекс.Маркет, но не понимаю unit-экономики там».

**Tools today friction:**
- Дублирующая подписка MPSTATS + Маяк = ~10 тыс ₽/мес, и всё равно
  половину времени в Excel.

**JTBD:**
> *«Когда я веду магазин на нескольких маркетплейсах, я хочу видеть
> единую картину продаж, прибыли и позиций конкурентов, чтобы
> принимать решения о ассортименте, цене и рекламе без открытия 5
> разных кабинетов».*

**Acquisition channel:** Chrome extension → free analytics на странице
WB-конкурента → регистрация → upgrade на Pro через 14 дней trial.

---

### Persona 2 — Brand Manager «Дмитрий»

| Параметр | Значение |
|---|---|
| **Возраст** | 41 |
| **Локация** | Москва |
| **Роль** | E-commerce manager бренда домашнего текстиля |
| **Магазин** | Собственное производство, 320 SKU |
| **Оборот** | 35 млн ₽/мес GMV, marge ~35% |
| **Каналы** | WB, Ozon, ЯМ, Megamarket, AliExpress Россия — все 5 |
| **Команда** | Дмитрий + контент-менеджер + менеджер по рекламе + 2 менеджера МП |

**Поведение:**
- В кабинете WB/Ozon ежедневно; в аналитических tools — еженедельно.
- Делает расширенный analysis раз в квартал по всему ассортименту.
- Контентный pipeline — 30+ карточек в месяц.

**Pain:**
- «Каждая площадка — свой формат, контент-команда тратит 60% времени
  на адаптацию одной карточки в 5 форматах».
- «Не вижу единого P&L по бренду — собираю Excel из 5 экспортов».
- «Команда работает в 5 кабинетах, потери в коммуникации».

**Tools today:** MPSTATS Enterprise, Excel/Power BI, Notion для пайплайна
контента, кабинеты МП.

**JTBD:**
> *«Когда у меня сотни SKU и 5 каналов, я хочу управлять контентом,
> остатками и рекламой из одного места, чтобы команда работала
> синхронно и я видел consolidated P&L».*

**Acquisition channel:** Контент-маркетинг (blog/YouTube о
multichannel ops) → demo-call → Team-tier подписка.

---

### Persona 3 — Agency Lead «Сергей»

| Параметр | Значение |
|---|---|
| **Возраст** | 38 |
| **Локация** | СПб |
| **Роль** | Co-founder marketplace-агентства полного цикла |
| **Команда агентства** | 18 человек: 5 account-менеджеров, 4 контентщика, 2 рекламщика, 7 операционка |
| **Клиенты** | 22 активных магазина (от 2 до 80 млн ₽/мес GMV каждый) |
| **Revenue агентства** | ~12 млн ₽/мес |
| **Tools today** | MPSTATS Enterprise + индивидуальные подписки на клиентов, Bitrix24 CRM, Notion, кабинеты клиентов |

**Поведение:**
- Сам глубоко в продукте не сидит — управляет account-менеджерами.
- Раз в неделю — синхронизация по всем клиентам.
- Постоянно ищет, чем дифференцироваться от конкурентов-агентств.

**Pain:**
- «MPSTATS не даёт multi-tenant — для каждого клиента отдельная
  подписка, дорого и неудобно».
- «Менеджер должен переключаться между 5-10 кабинетами клиентов».
- «Не могу показать клиенту прозрачный отчёт о действиях команды без
  ручной сборки».
- «При onboarding нового клиента — неделя на интеграции и настройки».

**Tools today friction:** Платит ~80 тыс ₽/мес за фрагментированный
стек, всё равно собирает Excel-отчёты вручную.

**JTBD:**
> *«Когда я веду 20+ клиентских магазинов одновременно, я хочу
> единый workspace, где видны метрики всех клиентов, разграничены
> права команды и автоматически собирается отчёт о наших действиях».*

**Acquisition channel:** Agency-referral + direct sales + sales-call с
демонстрацией Agency Mode. Conversion в Agency-tier 24 990 ₽ за 30 дней.

---

## 5. User Journey — Persona 1 «Анна»

### Стадия 1 — Discovery (Day -7 до Day 0)

| Этап | Действие | Touchpoint | Friction |
|---|---|---|---|
| Triggering event | Анна заметила, что MPSTATS дорогой, а половину функций не использует | Telegram-чат продавцов | – |
| Поиск альтернатив | Гуглит «MPSTATS аналоги», читает обзоры | Yandex search, блог easycomm-world | Конкуренты в выдаче перетягивают clicks |
| Первый контакт | Кликает на Chrome extension в рекомендациях блога | Chrome Web Store страница | Установка занимает 30 сек, нужны permissions |
| Free analytics | Открывает любую карточку WB, видит popup с данными о товаре (продажи, marge proxy, тренд) | Chrome extension widget | – |

**Success metric:** Activation rate Chrome-ext → web sign-up = ≥ 25%.

### Стадия 2 — Activation (Day 0 до Day 7)

| Этап | Действие | Touchpoint | Friction |
|---|---|---|---|
| Sign-up | Регистрация email + мастер-пароль | Web app onboarding | Мастер-пароль — новая концепция, нужно объяснить через 1-screen tutorial |
| Vault unlock | Создаёт мастер-пароль, видит explainer «ключи не уходят на сервер» | Onboarding step 2 | Mistake: забудет пароль → ключи теряются. Mitigation: recovery hint (не recovery email!) |
| Connect 1st marketplace | Вводит WB API-ключ (read-only сначала) | Settings > Integrations | Найти API-ключ в WB-кабинете — 5 минут; tutorial с скриншотами |
| First insight | Видит Profit Dashboard с её реальными SKU | Dashboard | – |
| First action | Меняет цену на одном SKU прямо из Dashboard | One-click action | Это aha-moment. Target: < 5 минут от sign-up |

**Success metric:** 7-day activation = ≥ 40% (KR-2.2).

### Стадия 3 — Habit (Day 7 до Day 30)

| Этап | Действие | Touchpoint |
|---|---|---|
| Daily digest | Получает Telegram-бот daily push: «3 SKU упали в позиции, marge на X снизилась на Y%» | Telegram |
| Weekly review | Заходит в web app по push в Telegram, проверяет dashboard | Web app |
| Repricer trial | Включает auto-repricer на 5 SKU (Pro feature, в free-tier недоступен) | Repricer module |
| Upgrade trigger | Видит, что 5 SKU repricer-ом дали +12% marge → готова платить Pro | Pricing page |

**Success metric:** Free → Pro conversion 14-day = ≥ 8%.

### Стадия 4 — Upgrade & Expansion (Day 30+)

| Этап | Действие | Touchpoint |
|---|---|---|
| Pro upgrade | Подписывается на Pro 2 990 ₽/мес | ЮKassa checkout |
| Add Ozon | Подключает Ozon API-ключ (read+write) | Settings |
| Use AI generation | Регенерирует описание карточки через MCP-помощник | Card editor |
| Team upgrade (опционально) | Если нанимает ассистента — upgrade на Team 9 990 ₽ | Pricing page |

**Success metric:** Pro → Team conversion 6-month = ≥ 5%.

---

## 6. Feature List (grouped by release + priority)

Маркировка приоритета:
- **P0** — без этого нельзя релизить MVP.
- **P1** — целевая v1 (M+6 до M+9).
- **P2** — v2 (M+12+).

### MVP (M+0 до M+3-6) — стартовый release

| ID | Feature | Priority | Owner module |
|---|---|---|---|
| F-001 | Email + master-password регистрация и login | **P0** | Auth |
| F-002 | Client-side encrypted vault (AES-GCM + PBKDF2 + IndexedDB) для API-ключей | **P0** | Vault |
| F-003 | Подключение Wildberries Seller API (read-only начально, write по подтверждению) | **P0** | MarketplaceAdapter:WB |
| F-004 | Подключение Ozon Seller API (read+write) | **P0** | MarketplaceAdapter:Ozon |
| F-005 | Profit Dashboard (GMV, marge, ROI на рекламу, динамика за 7/30/90 дней) | **P0** | Analytics |
| F-006 | One-click change цены SKU из Dashboard | **P0** | Actions |
| F-007 | Reverse-карточка: ввод URL карточки конкурента → ключевые слова, продажи proxy, тренд | **P0** | Reverse-card |
| F-008 | Keyword research для WB+Ozon (поиск по seed-слову → частотность + конкуренция) | **P0** | Keywords |
| F-009 | AI Card Generator: regenerate описание карточки через MCP (YandexGPT default) | **P0** | AI / MCP |
| F-010 | Competitor tracking: pin до 10 карточек, daily snapshot позиции и цены | **P0** | Tracker |
| F-011 | Chrome MV3 extension: free analytics на странице WB/Ozon товара | **P0** | Chrome ext |
| F-012 | Telegram bot: connect, daily digest о SKU метриках | **P0** | Notifications |
| F-013 | Pricing tiers + ЮKassa billing (Free/Pro/Team/Agency) | **P0** | Billing |
| F-014 | Account settings, API-key rotation, vault lock/unlock UI | **P0** | Settings |

### v1 (M+6 до M+9)

| ID | Feature | Priority |
|---|---|---|
| F-101 | Подключение Яндекс.Маркет Partner API | P1 |
| F-102 | Подключение Megamarket Partner API | P1 |
| F-103 | Repricer engine: правила на основе 10+ параметров (цена конкурента, остаток, marge floor) | P1 |
| F-104 | Stock Manager: alerts о low-stock, прогноз out-of-stock на основе sales velocity | P1 |
| F-105 | Niche Finder: поиск товарных ниш по фильтрам (продажи, конкуренция, marge) | P1 |
| F-106 | SEO optimisation: AI-проверка карточки на ключевые слова, recommendations | P1 |
| F-107 | Agency Mode: multi-tenant workspace, права (admin/manager/viewer), client switching | P1 |
| F-108 | RLS-isolation для Agency tenants на Postgres + ClickHouse | P1 |
| F-109 | Audit log всех write-действий пользователя и AI-агентов | P1 |
| F-110 | Bulk operations: batch update цены / описания на 10-100 SKU | P1 |
| F-111 | PWA-режим web-app (offline read-cache, push notifications) | P1 |

### v2 (M+12+)

| ID | Feature | Priority |
|---|---|---|
| F-201 | Order management (просмотр и обработка заказов из одного UI) | P2 |
| F-202 | PPC automation: правила управления ставками во внутр. рекламе WB/Ozon | P2 |
| F-203 | Review management: AI-генерация ответов на отзывы, queue для approve | P2 |
| F-204 | Photo studio integration: API партнёрской фото-студии для заказа 360°-съёмки | P2 |
| F-205 | Speed-to-Insight: AI-обзоры рынка (еженедельный AI-отчёт по нише) | P2 |
| F-206 | Sales forecast: ML-прогноз продаж SKU на 30/60/90 дней | P2 |
| F-207 | Loss recovery (Refund Genie аналог): поиск списанных потерь, формирование заявок | P2 |
| F-208 | White-label режим для Agency-tier (custom domain, branded reports) | P2 |
| F-209 | Managed services upsell flow (заказать ведение магазина у партнёров) | P2 |

---

## 7. Success Metrics

### North Star

**Number of Active Sellers Performing ≥ 1 write-action per week** (WAS-1w).

Эта метрика отражает основной FP-1 (data → action) и FP-2 (action latency
определяет ARPU). Цель: 600 WAS-1w на M+12, 2 500 на M+24.

### Activation Funnel

| Метрика | Definition | Target M+6 | Target M+12 |
|---|---|---|---|
| Chrome-ext install rate | DAU extension / DAU блога | ≥ 8% | ≥ 12% |
| Sign-up rate | Sign-ups / extension users | ≥ 25% | ≥ 30% |
| TTFA (median) | Время от sign-up до первого write-action | < 5 мин | < 4 мин |
| 7-day activation | Users с ≥ 1 write-action в первые 7 дней / sign-ups | ≥ 40% | ≥ 50% |

### Retention

| Метрика | Target M+6 | Target M+12 |
|---|---|---|
| W4 retention (write-active) | ≥ 35% | ≥ 45% |
| Monthly churn (Pro+) | ≤ 7% | ≤ 5% |
| Net Revenue Retention | ≥ 100% | ≥ 110% |

### Revenue

| Метрика | Target M+12 | Target M+24 |
|---|---|---|
| ARR | 5-10 млн ₽ | 30-80 млн ₽ |
| Paying users (Pro+) | 500 | 2 000 |
| Agency-tier clients | 15 | 50 |
| ARPU (Pro+) | ≥ 3 500 ₽/мес | ≥ 4 200 ₽/мес |
| LTV / CAC | ≥ 5 | ≥ 10 |
| Payback period | ≤ 4 мес | ≤ 2.5 мес |

---

## 8. Pricing Tiers

| Параметр | **Free** | **Pro** | **Team** | **Agency** |
|---|---|---|---|---|
| **Цена/мес** | 0 ₽ | 2 990 ₽ | 9 990 ₽ | 24 990 ₽ |
| **Цена/год (-15%)** | – | 30 500 ₽ | 101 900 ₽ | 254 900 ₽ |
| SKU limit | 10 | 200 | 2 000 | 20 000 |
| Маркетплейсы | 1 | до 3 | Все 4 | Все 4 |
| Chrome extension | ✓ | ✓ | ✓ | ✓ |
| Profit Dashboard | базовый | полный | полный | полный |
| Reverse-карточка | 3/день | 50/день | 500/день | unlimited |
| Keyword research | 10 запросов/день | 200/день | 2 000/день | unlimited |
| Competitor tracker | 3 SKU | 30 SKU | 300 SKU | unlimited |
| AI requests (MCP) | 50/мес | 500/мес | 5 000/мес | 50 000/мес |
| AI Card Generator | ✗ | ✓ (Y.GPT) | ✓ (Y.GPT+Claude) | ✓ (Y.GPT+Claude+GPT-4o) |
| Repricer | ✗ | до 50 SKU | до 500 SKU | unlimited |
| Stock Manager | ✗ | ✓ | ✓ | ✓ |
| Niche Finder | ✗ | ✓ | ✓ | ✓ |
| Agency Mode (multi-tenant) | ✗ | ✗ | ✗ | ✓ |
| Team seats | 1 | 1 | 3 | 15 + unlimited clients |
| Audit log retention | 7 дней | 30 дней | 90 дней | 365 дней |
| Telegram daily digest | ✗ | ✓ | ✓ | ✓ |
| White-label reports | ✗ | ✗ | ✗ | ✓ (v2) |
| Priority support | ✗ | email | email + chat | dedicated CSM |

**Гипотезы для validation (Phase 2):**
- 2 990 ₽ — порог входа для mid-size sellers; ниже — не отбивает marketing CAC ≤ 6 000 ₽.
- 24 990 ₽ — против ~80 тыс ₽ фрагментированного стека агентства (Persona Сергей) даёт 70% saving.
- Free-tier 10 SKU — достаточно для solo-seller wedge, мало для перехода на Pro без upgrade.

---

## 9. Go-to-Market (summary)

Полный детальный план — в отдельном GTM-документе. Здесь — каналы и приоритеты.

| # | Канал | Приоритет | Target persona | Ожидаемая доля acquisition |
|---|---|---|---|---|
| 1 | **Chrome MV3 extension** + Chrome Web Store SEO | ★★★★★ | Solo / Mid-size | 35% |
| 2 | **Telegram-сообщества WB/Ozon продавцов** (organic + paid) | ★★★★★ | Mid-size | 25% |
| 3 | **Telegram-бот**: daily digest как retention/referral loop | ★★★★ | All | (retention, не acquisition) |
| 4 | **Контент-маркетинг**: блог + YouTube + Дзен | ★★★★ | Brand / Mid-size | 15% |
| 5 | **Agency referral programme** (партнёрские агентства) | ★★★ | Mid-size приведённый | 10% |
| 6 | **Direct sales для Agency-tier** | ★★★ | Agency | 10% |
| 7 | **SEO органический** (поисковая выдача «MPSTATS аналог» и т.п.) | ★★ | All | 5% |

**Launch cadence:**
- **M-2 до M+0**: closed beta 50 пользователей из Telegram-сообществ продавцов.
- **M+0**: public launch (web + Chrome ext) с Free + Pro tiers.
- **M+3**: Team-tier + ЯМ интеграция.
- **M+6**: Agency-tier + Megamarket интеграция + agency referral programme.

---

## 10. Constraints

### Архитектурные (из Product_Discovery_Brief, не подлежат изменению без Architecture review)

```yaml
pattern: "Distributed Monolith (Monorepo)"
containers: "Docker + Docker Compose"
infrastructure: "VPS (AdminVPS / HOSTKEY)"
deploy: "Docker Compose direct deploy (SSH или GitHub Actions runner)"
ai_integration: "MCP servers"

frontend:
  framework: "Next.js (React, TypeScript)"
  state: "TanStack Query + Zustand"
  ui: "shadcn/ui + Tailwind"
  extension: "Chrome MV3 (Manifest V3)"

backend:
  runtime: "Node.js 22 LTS + TypeScript"
  api: "Fastify"
  orm: "Prisma"
  background: "BullMQ (Redis-backed)"

data:
  primary: "PostgreSQL"
  timeseries: "ClickHouse"
  cache: "Redis"
  blob: "S3-совместимое (Selectel / Yandex Cloud Object Storage)"

security:
  api_keys_input: "UI Settings > Integrations"
  storage: "Encrypted IndexedDB (AES-GCM 256-bit) на клиенте"
  key_derivation: "PBKDF2 от мастер-пароля (100k+ итераций)"
  server_side: "API ключи маркетплейсов НИКОГДА не уходят на бэкенд"
  vault_pattern: "Web Crypto API + auto-lock 15 min"
```

### Регуляторные

| Закон / стандарт | Требование | Артефакт compliance |
|---|---|---|
| **152-ФЗ «О персональных данных»** | Хранение PII граждан РФ — только в РФ-инфраструктуре | Yandex Cloud (Москва) / Selectel; роль DPO; уведомление РКН |
| **АИ-disclosure (превентивно)** | Watermark AI-сгенерированного контента, audit log AI-действий | F-109 (audit log), AI watermark в metadata карточек |
| **ToS WB/Ozon** | Не нарушать условия Seller API; scraping только публичных страниц с robots.txt-compliance | Юридический аудит ежеквартально |
| **ФЗ «О рекламе»** | Маркировка рекламы при PPC-автоматизации (v2) | Откладывается на F-202 phase |
| **Защита прав потребителей** | AI-генерация описаний не должна вводить покупателя в заблуждение | Human-in-the-loop approve на каждое описание |

### Рыночные

- Pricing валидируется на месяцах 1-6; правка тиров — только после ≥ 30
  customer-interviews и анализа conversion-метрик.
- Free-tier лимиты — баланс между acquisition и cannibalisation Pro;
  пересмотр ежеквартально по cost-per-AI-action и server-cost-per-free-user.
- Multi-marketplace интеграция через `MarketplaceAdapter` — каждый
  адаптер изолирован, breaking-changes API одной площадки **не должны**
  ломать остальные.

---

## 11. Open Questions (для Phase 2 validation)

Перенесено из `Product_Discovery_Brief.md` § "Открытые вопросы".

| # | Вопрос | Owner | Deadline для resolution |
|---|---|---|---|
| **OQ-1** | **152-ФЗ хостинг**: Yandex Cloud (Москва) vs Selectel (СПб) vs hybrid? Где разместить Postgres+ClickHouse с PII? Какова процедура уведомления РКН? | CTO + юрист | До конца Phase 2 |
| **OQ-2** | **Парсинг vs API**: при каких условиях допустимо парсить публичные страницы WB/Ozon с т.з. их ToS? Какой residential-proxy provider compliant с 149-ФЗ? | Юрист + Backend lead | До MVP launch |
| **OQ-3** | **MCP-сервера для AI**: какие операторы доступны российским пользователям без VPN? Anthropic через прокси Cloudflare работает? YandexGPT-4 API лимиты? Сколько стоит per-token при ожидаемом нагрузочном профиле? | AI lead | До F-009 implementation |
| **OQ-4** | **Agency Mode multi-tenancy**: Postgres RLS достаточно для compliance, или нужна физическая изоляция по schema? Performance impact RLS при 100+ tenants? | Backend lead | До F-107 implementation |
| **OQ-5** | **Pricing валидация**: 2 990 / 9 990 / 24 990 ₽ — гипотезы; нужно ≥ 30 customer interviews для confirmation. Какие сегменты готовы платить 9 990? Где правильный price ceiling для Agency? | Product + Sales | M+6 (по результатам closed beta) |

Дополнительные открытые вопросы, всплывшие в Phase 1:

| # | Вопрос | Owner | Deadline |
|---|---|---|---|
| **OQ-6** | Recovery flow при потере мастер-пароля — что хранить, что нет? Tradeoff: «true zero-knowledge» vs «забыл пароль = потерял подписки». | Security + UX | До F-002 implementation |
| **OQ-7** | Telegram bot vs Email digest — оба или один? Российская специфика говорит за Telegram, но Brand Manager Persona ожидает email-отчётов. | Growth | M+3 |
| **OQ-8** | [GAP: внешний эксперт] Стоимость WB Seller API premium-доступа (если такой существует) — нет публичных данных. Контакт нужен. | Founder + BD | Pre-MVP |

---

## 12. Appendix

### Document graph

```
Product_Discovery_Brief.md
        ↓
Research_Findings.md ←──┐
        ↓               │
Solution_Strategy.md  ──┤  (читают друг друга)
        ↓               │
PRD.md (этот документ)──┘
        ↓
Specification.md (next, Phase 2)
        ↓
Architecture.md → Pseudocode.md → Refinement.md → Completion.md
```

### Ссылки

- `docs/Research_Findings.md` — § 5 Competitive Landscape, § 6 Functional Map (источник feature-list), § 8 User Insights (источник pain points в personas).
- `docs/Solution_Strategy.md` — § 7 Recommended Approach (источник vision), § 8 Risk Assessment (источник constraints), § 10 Hand-off table (жёсткие MUST).
- `docs/Product_Discovery_Brief.md` — § Architecture Constraints, § Product Context, § Открытые вопросы.

### Glossary

| Термин | Значение |
|---|---|
| **МП** | Маркетплейс (WB / Ozon / ЯМ / Megamarket) |
| **TTFA** | Time To First Action (от sign-up до первого write-API действия) |
| **WAS-1w** | Weekly Active Sellers performing ≥ 1 write-action |
| **MCP** | Model Context Protocol (Anthropic-стандарт для AI-инструментов) |
| **Vault** | Client-side зашифрованное хранилище API-ключей в IndexedDB |
| **Adapter** | `MarketplaceAdapter` интерфейс, абстрагирующий различия API маркетплейсов |
| **RLS** | Postgres Row-Level Security — механизм tenant isolation |
| **JTBD** | Jobs To Be Done — формат описания мотивации персоны |

---

*Last updated: 2026-05-14. Status: DRAFT for SPARC Phase 2 validation
via `requirements-validator` skill. Next review after validation report.*

# Product Discovery Brief — "Easycomm-World"

> Phase 0 output reverse-engineering-unicorn (QUICK mode, modules M2–M5).
> Передаётся в Phase 1 (sparc-prd-mini) как pre-filled context.
> Источник: `docs/research/easycomm-analysis.md` + `docs/research/predecessor-analysis.md`.

## 0. Контекст и цель проекта

Воспроизвести **next-generation версию Easy Commerce** (easycomm.ru),
опираясь на верифицированно определённый зарубежный прародитель —
**Sellics** (Берлин, 2014 → Perpetua, 2022), с функциональной начинкой
уровня **Helium 10** (Amazon analytics market leader) и
**Rithum/ChannelAdvisor** (multichannel commerce ops), но
**адаптированный под российские маркетплейсы 2025-2026 г.**

Кодовое имя продукта: **easycomm-world** (от названия репозитория).

## M2 — Product & Customers

### JTBD (Jobs To Be Done) — иерархия

**Job-1 (стратегический):**
> *"Когда я веду магазин на нескольких маркетплейсах, я хочу видеть единую
> картину продаж, прибыли и позиций конкурентов, чтобы принимать решения
> о ассортименте, цене и рекламе без открытия 5 разных кабинетов."*

**Job-2 (тактический):**
> *"Когда я запускаю новый товар, я хочу быстро понять размер ниши, цены
> конкурентов и ключевые слова, чтобы рассчитать unit-экономику до закупки
> товара."*

**Job-3 (операционный):**
> *"Когда у меня сотни SKU, я хочу автоматически пересчитывать цены и
> отслеживать остатки, чтобы не терять Buy Box / витрину и не уходить в
> out-of-stock."*

**Job-4 (агентский / для B2B-клиента):**
> *"Когда я производитель, я хочу делегировать ведение магазина команде
> экспертов, имея прозрачный отчёт по каждому действию."*

### Value Proposition

> *Единая платформа для российских маркетплейс-продавцов: аналитика на
> уровне Helium 10, мультиканал на уровне Rithum, агентский слой Sellics,
> AI-помощник на MCP — всё через зашифрованный браузерный кабинет.*

### Customer Segments (приоритет → MVP)

| # | Сегмент | Объём в РФ (порядок) | MVP-приоритет |
|---|---|---|---|
| 1 | **Mid-size sellers** (1–10 SKU, оборот 1–20 млн ₽/мес на WB+Ozon) | 100–200 тыс. | ★★★ Core MVP |
| 2 | **Brands и производители** (своё производство, 50+ SKU) | 5–20 тыс. | ★★ Phase 2 |
| 3 | **Marketplace-агентства** (как сам Easy Commerce) | 200–500 | ★★ Phase 2 |
| 4 | **Solo sellers** / новички (1–3 SKU, < 0,5 млн ₽/мес) | 500+ тыс. | ★ Free tier |
| 5 | **Enterprise** (Сбер.Мегамаркет 1P, корпорации) | < 100 | ✗ вне MVP |

### Critical pain points (валидируется Specification.md)

- *"Кабинет WB/Ozon показывает не то, что нужно для бизнес-решений"*
- *"Внешние аналитические сервисы (MPSTATS, Moneyplace) дают данные, но
  не действия — переключаешься в кабинет, теряешь контекст"*
- *"Управление ценой в Excel — медленно и ошибочно при 100+ SKU"*
- *"Подбор ключей через перебор — часы вручную, нет reverse-ASIN-стиль
  инструмента для российских площадок"*
- *"Нет единого реестра карточек по всем маркетплейсам"*

## M3 — Market & Competition

### TAM / SAM / SOM (порядки величин, для MVP-фрейминга)

- **TAM** — российский рынок e-commerce SaaS-инструментов для продавцов
  на маркетплейсах: ≈ **15–25 млрд ₽/год** (оценка по аналогии с
  американским ≈ $2-3B рынка Amazon-seller-tools, скорректированная на
  объём GMV WB+Ozon ≈ $50–60B vs Amazon ≈ $700B).
- **SAM** (mid-size sellers + brands в B2B SaaS-подписке) — **3–5 млрд ₽/год**.
- **SOM** (реалистичная доля при выходе на рынок в 2026 году) —
  **30–80 млн ₽ ARR** на 18–24 месяце.

### Конкурентная матрица — Blue Ocean lens

| Игрок | Аналитика | Multichannel | AI-помощник | Agency layer | Encrypted vault | Российские МП |
|---|---|---|---|---|---|---|
| MPSTATS | ★★★★★ | ★★ | ✗ | ✗ | ✗ | ✓ |
| Moneyplace | ★★★★ | ★★ | ✗ | ✗ | ✗ | ✓ |
| MarketGuru | ★★★ | ★ | ✗ | ✗ | ✗ | ✓ |
| Маяк | ★★ | ✗ | ✗ | ✗ | ✗ | ✓ |
| Easy Commerce (CAT) | ★★★★ | ★★★ (E-commerce Tool) | ✗ | ★★★★★ | ✗ | ✓ |
| Helium 10 (US) | ★★★★★ | ✗ (только Amazon) | ★★ | ✗ | ✗ | ✗ |
| **easycomm-world (new)** | **★★★★★** | **★★★★★** | **★★★★★ (MCP)** | **★★★★** | **★★★★★** | **✓** |

### Blue Ocean differentiation (4 действия по Kim & Mauborgne)

| Действие | Решение для easycomm-world |
|---|---|
| **ELIMINATE** (что убрать у конкурентов) | Серверное хранение API-ключей и личных данных без шифрования — все ключи AES-GCM в IndexedDB |
| **REDUCE** (что снизить) | Latency перехода "данные → действие": репрайсинг и обновление карточки запускаются прямо из аналитического экрана |
| **RAISE** (что усилить) | Глубина AI-помощи через MCP-серверы: семантический поиск ниш, генерация контента, прогноз тренда |
| **CREATE** (что создать, чего нет ни у кого) | **Agency Mode** — workspace для агентств с multi-tenant управлением клиентскими магазинами (наследие Sellics-модели, не реализовано в РФ ни у кого) |

## M4 — Business & Finance (Unit Economics, гипотеза)

### Бизнес-модель

| Поток | Описание | Доля выручки (целевая) |
|---|---|---|
| **SaaS subscription** (3 tier) | Free / Pro 2 990 ₽/мес / Team 9 990 ₽/мес | 60% |
| **Agency Mode subscription** | 24 990 ₽/мес (multi-client workspaces) | 25% |
| **Managed services upsell** | Под управление магазинов клиентов (как сам Easy Commerce) | 15% |

### Pricing tiers (исходные гипотезы для Specification.md)

| Tier | Цена/мес | SKU limit | Маркетплейсы | AI requests | Agency seats |
|---|---|---|---|---|---|
| **Free** | 0 ₽ | 10 | 1 | 50/мес | 0 |
| **Pro** | 2 990 ₽ | 200 | До 3 | 500/мес | 0 |
| **Team** | 9 990 ₽ | 2 000 | Все | 5 000/мес | 3 |
| **Agency** | 24 990 ₽ | 20 000 | Все | 50 000/мес | 15 + clients |

(Бенчмарк: Helium 10 — $29/99/279, Jungle Scout — $29/49/+ Cobalt enterprise.
Российский pricing с поправкой на покупательную способность: ÷3-4.)

### Unit Economics (целевые, валидируется на 6-м месяце)

- **CAC** (acquisition cost on Pro tier): ≤ 6 000 ₽
- **MRR per Pro user**: 2 990 ₽
- **Churn (mid-size sellers)**: ≤ 5%/мес (LTV ≈ 60 000 ₽ при 20 мес)
- **LTV/CAC**: ≥ 10
- **Payback period**: ≤ 2.5 мес

## M5 — Growth Engine

### Distribution channels (приоритет)

1. **Chrome Extension** (как Jungle Scout и MPSTATS) — wedge для
   нового пользователя: free analytics через расширение → upgrade в
   web app.
2. **Telegram-бот** с дневным дайджестом (российская специфика —
   замена email-маркетинга).
3. **Контент-маркетинг через блог** (как Easy Commerce делает уже сейчас
   с тысячами просмотров на тренды-2024).
4. **Telegram-сообщества WB/Ozon продавцов** (нишевый organic).
5. **YouTube/Дзен туториалы**.
6. **Agency referral** — агентства приводят своих клиентов на B2C-тиры
   (как Sellics через своих agency-partners).

### Activation funnel (контракт для Pseudocode.md)

```
Anonymous → Chrome Ext install (1-click free analytics)
         → Account create (email + master password)
         → Connect 1st marketplace (read-only API key, encrypted vault)
         → "Aha moment": first profit metric vs competitor
         → Upgrade to Pro (на 5–10 SKU)
```

### Retention loops

- **Daily**: Telegram-бот → push о position drop / margin shrink → возврат
- **Weekly**: email-дайджест "что упустил конкурент" → CTR обратно в app
- **Monthly**: "Опубликовать карточки в новом маркетплейсе одним кликом"
  upsell на multichannel модуль

## Architecture Constraints (passed to Phase 1)

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
  timeseries: "ClickHouse"  # для исторических метрик карточек
  cache: "Redis"
  blob: "S3-совместимое (Selectel / Yandex Cloud Object Storage)"

integrations:
  marketplaces: ["Wildberries Seller API", "Ozon Seller API",
                 "Яндекс.Маркет Partner API", "Megamarket Partner API"]
  ai_mcp_servers:
    - "openai-mcp (gpt-4o for content generation)"
    - "anthropic-mcp (claude for analytics insights)"
    - "yandexgpt-mcp (для русскоязычной генерации)"
  payments: "ЮKassa + ProdamusGate (резерв)"
  email: "Unisender / SendPulse"
  telegram: "Bot API + Telegram Web Apps"

security:
  api_keys_input: "UI Settings > Integrations"
  storage: "Encrypted IndexedDB (AES-GCM 256-bit) на клиенте"
  key_derivation: "PBKDF2 от мастер-пароля пользователя (100k+ итераций)"
  server_side: "API ключи маркетплейсов НИКОГДА не уходят на бэкенд"
  vault_pattern: "Web Crypto API + auto-lock 15 min"
```

## Product Context (passed to Phase 1)

```yaml
target_segments:
  - "Mid-size sellers (1-10 SKU, оборот 1-20 млн ₽/мес)"
  - "Brands/manufacturers (50+ SKU)"
  - "Marketplace agencies"

key_competitors:
  direct: ["MPSTATS", "Moneyplace", "MarketGuru", "Easy Commerce CAT"]
  foreign_predecessors:
    primary: "Sellics (business model)"
    feature_set: "Helium 10"
    multichannel: "ChannelAdvisor / Rithum"
    category_pioneer: "Jungle Scout"

differentiation:
  - "Agency Mode (multi-tenant) — нет ни у одного российского конкурента"
  - "AI через MCP — глубже, чем у западных лидеров (Speed to Insight)"
  - "Client-side encrypted vault — ни у одного российского игрока"
  - "Multichannel native — в архитектуре с дня 1, не post-MVP"

monetization:
  primary: "SaaS subscription (Free / Pro 2990₽ / Team 9990₽ / Agency 24990₽)"
  secondary: "Managed services upsell (Sellics-style)"
  free_tier_strategy: "Chrome extension + 1 marketplace + 10 SKU"
```

## Открытые вопросы (для Phase 1 — sparc-prd-mini уточнит)

1. **Регуляторика**: 152-ФЗ "О персональных данных" — где хранятся
   персональные данные продавцов? (Yandex Cloud в РФ? Selectel?)
2. **Парсинг vs API**: WB API ограничен; в каких случаях допустим
   парсинг публичных страниц с т.з. ToS?
3. **MCP-сервера для AI**: какие конкретно операторы доступны
   российским пользователям без VPN (YandexGPT vs OpenAI через прокси)?
4. **Agency Mode**: multi-tenant — RLS в Postgres или физическая
   изоляция по schema?
5. **Pricing валидация**: 2990 ₽ / 9990 ₽ — гипотезы, нужен customer
   interview раунд (но для MVP — закладываем эти значения).

## Sign-off Phase 0

- ✅ Прародитель идентифицирован: **Sellics + Helium 10 + Rithum** (гибрид)
- ✅ Customer segments определены и приоритизированы
- ✅ Конкурентная матрица построена (Blue Ocean дифференциация ясна)
- ✅ Бизнес-модель и pricing tiers зафиксированы как гипотезы
- ✅ Growth engine описан (channels + activation funnel + retention loops)
- ✅ Architecture constraints + product context подготовлены для Phase 1

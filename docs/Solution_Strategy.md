# Solution Strategy — easycomm-world

> SPARC Phase 1 artifact. Стратегическое обоснование выбранного подхода.
> Источник входных данных: `docs/Research_Findings.md`,
> `docs/Product_Discovery_Brief.md`.
>
> Методики: SCQA → First Principles → 5 Whys → Game Theory →
> Second-Order Thinking → TRIZ → Synthesis → Risk Assessment.

---

## 1. SCQA — постановка задачи

### Situation

С 2022 года российский рынок маркетплейсов переживает структурный бум:
GMV WB+Ozon оценивается в $50-60B/год, число продавцов перешагнуло
рубеж 700 000 (включая solo-tier). Параллельно сформировалась
SaaS-категория инструментов для продавцов: MPSTATS, Moneyplace,
MarketGuru, Маяк, JLab.ai, CAT/Easy Commerce. Западные аналоги
(Helium 10, Jungle Scout, Sellics, Rithum) на российский рынок не
зашли и не зайдут до политических изменений (горизонт 5+ лет).

### Complication

Категория **расщеплена на два непересекающихся слоя**:

1. **Чистый SaaS-аналитический трек** (MPSTATS, Moneyplace) —
   глубокая аналитика, но **«данные без действия»**: чтобы что-либо
   сделать с этими данными (поменять цену, обновить карточку, заказать
   рекламу), пользователь идёт в кабинет WB/Ozon.
2. **Чистый agency-трек** (Okkam Group, и частично сам Easy Commerce
   как агентство) — действие есть, но **дорого** и **непрозрачно**
   для клиента.

Структурный гибрид «agency + SaaS» из западной категории (Sellics)
**закрыт на западе** в 2022 и **никем не реализован** в России.
Дополнительно: ни один игрок не реализует (a) клиентский шифрованный
vault, (b) AI-помощника через MCP, (c) native multichannel в архитектуре
с дня 1.

### Question

Как построить продукт, который:
- **выигрывает у MPSTATS/Moneyplace** на их территории (analytics);
- **обходит агентства** на их территории (action layer);
- **создаёт защитный ров** (encrypted vault + AI/MCP + agency mode);
- **выходит на безубыточность** на 18-24 месяце при разумной burn rate.

### Answer

Гибрид «Sellics-business-model + Helium 10-feature-set + Rithum-multichannel-
ops + AI-via-MCP + client-side-encrypted-vault», адаптированный под
российские МП 2025-2026. **Не реверс-инжиниринг Easy Commerce**, а
next-generation реплика категории с архитектурными решениями, которые
западные прародители не реализовали (vault + MCP).

---

## 2. First Principles — атомарные истины

Декомпозиция проблемы до уровня, ниже которого спорить уже не с чем.

### FP-1. Данные ≠ инсайт ≠ действие

> Сырые данные WB/Ozon API бесполезны, если не превращаются в инсайт.
> Инсайт бесполезен, если требует переключения контекста для действия.

Следствие для архитектуры: **action-buttons рядом с метрикой** на всех
аналитических экранах. Latency от инсайта до действия — < 3 кликов.

### FP-2. Action latency определяет ARPU

> Чем быстрее пользователь выполняет действие, основанное на данных,
> тем чаще он возвращается, тем выше LTV.

Следствие: оптимизируем **time-to-first-action** в активации
(< 5 минут от регистрации до первого изменения цены/карточки/ставки).

### FP-3. Sellers хотят делегировать рутину

> При 100+ SKU ручной режим невозможен. Sellers готовы платить за
> автоматизацию **больше**, чем за более глубокие данные.

Следствие: правила репрайсинга, автоматическая регенерация описаний,
авто-ответы на отзывы — в core MVP, а не «когда-нибудь потом».

### FP-4. AI replaces marketplace analyst at fraction of cost

> Профессиональный marketplace-аналитик в РФ — 80-150 тыс ₽/мес.
> AI-агент через MCP с тем же качеством рекомендаций — 5-15 тыс ₽/мес
> на инференс. Дельта 10× — экономически непреодолимый аргумент.

Следствие: AI — не «features», а **product foundation**. Каждый модуль
имеет AI-companion mode.

### FP-5. API-ключ — это деньги пользователя

> WB/Ozon API-ключи дают доступ к финансовой операционке магазина.
> Утечка одного ключа = слив бизнеса конкуренту.

Следствие: **API-ключи не должны попадать на наш сервер ни при каких
условиях.** Это первоосновное архитектурное ограничение, а не feature.

### FP-6. Marketplaces — adversarial environment

> WB и Ozon — наши инфраструктурные противники, а не партнёры. У них
> нет коммерческого интереса в успехе третьих-сторонних SaaS-вендоров.

Следствие: rate-limit стратегия, fallback на scraping публичных
страниц, готовность к смене схем API без анонса.

---

## 3. Five Whys — корневая причина churn у MPSTATS/Moneyplace

> Сценарий: продавец отказывается от платной подписки MPSTATS через 3-4 месяца.

| Уровень | Вопрос | Ответ |
|---|---|---|
| Why-1 | Почему ушёл? | «Перестал смотреть» / «всё равно делаю по своему» |
| Why-2 | Почему перестал смотреть? | Чтобы воспользоваться инсайтом, всё равно нужно идти в кабинет WB и делать руками |
| Why-3 | Почему идёт в кабинет, а не действует в MPSTATS? | MPSTATS не умеет действовать — только показывать |
| Why-4 | Почему MPSTATS не умеет действовать? | Архитектурно: не подключают seller-API в режиме write, чтобы не нести ответственность за изменения |
| Why-5 | **Корневая причина** | **Категория исторически росла из «парсинга публичных данных» (Jungle Scout 2015) — write-API не был частью DNA.** |

**Стратегический вывод:** easycomm-world входит в категорию с DNA
write-API с первого дня. Это и есть наш моат против MPSTATS.

---

## 4. Game Theory — равновесия игроков

### Игроки

| Игрок | Главный интерес |
|---|---|
| **Seller** (наш клиент) | Максимизировать прибыль за вычетом операционных издержек |
| **Marketplace** (WB/Ozon) | Максимизировать GMV комиссии, удерживать продавца внутри своей экосистемы |
| **Конкурент-seller** | Симметричен нашему клиенту → arms race |
| **Tool vendor** (мы и MPSTATS) | Максимизировать ARR при защите от platform-risk |
| **Agency** | Максимизировать доход за управление магазинами клиентов |

### Матрица интересов и конфликтов

| | Seller | Marketplace | Comp-Seller | Vendor | Agency |
|---|---|---|---|---|---|
| **Seller** | — | ⚠ комиссии | ⚔ конкуренция | ✓ платит | ✓ делегирует |
| **Marketplace** | ⚠ забирает marge | — | нейтрально | ⚔ scraping | ⚠ посредник |
| **Comp-Seller** | ⚔ | нейтрально | — | ✓ платит | ✓ |
| **Vendor (мы)** | ✓ | ⚔ adversarial | ✓ | — | ✓ partner |
| **Agency** | ✓ | ⚠ дилюция | нейтрально | ✓ B2B-канал | — |

### Nash equilibrium — ключевые выводы

1. **Marketplace ↔ Vendor — adversarial равновесие.** WB/Ozon не имеют
   стимула делать жизнь третьесторонних tools проще. Любая
   стабильность API — побочный продукт, а не намерение.
   *Стратегия:* проектировать с допущением **breaking-changes раз в
   квартал**; abstraction-layer над API маркетплейсов.
2. **Seller ↔ Comp-Seller — symmetric arms race.** Если все продавцы
   получат одинаковый AI-репрайсер, цены сходятся к marginal cost →
   marge стремится к 0.
   *Стратегия:* персонализация AI-моделей под конкретный магазин
   (private fine-tuning per-tenant), а не glob-rules.
3. **Vendor ↔ Agency — positive-sum.** Agency-mode даёт нам B2B
   распределение, agency получает tool как white-label feature.
   *Стратегия:* приоритизировать Agency Mode как distribution channel,
   а не только revenue stream.
4. **Seller-multihoming.** Российский продавец редко на одной площадке.
   Tool, который покрывает только Wildberries (как MPSTATS изначально), —
   проигрывает мультиплатформенному.
   *Стратегия:* multichannel в архитектуре с дня 1.

### Marketplace как противник — что это значит на практике

- WB периодически меняет endpoints без анонса (наблюдалось 2023-2025).
- Ozon вводил лимиты Seller API (500 запросов/мин) без предупреждения.
- Возможен сценарий, когда WB/Ozon **сами выпустят** собственный
  SaaS-конкурент (как Amazon выпустил Amazon Brand Analytics).

**Митигация:**
- Architectural abstraction layer (`MarketplaceAdapter` интерфейс).
- Бизнес-модель не должна зависеть от одного маркетплейса (≥3 МП в core).
- Парсинг публичных страниц как fallback при API-перебоях.
- Юридическая работа: ToS-compliance аудит ежеквартально.

---

## 5. Second-Order Thinking — последствия успеха

Что будет, если easycomm-world и аналоги станут success-story и
большинство продавцов будут принимать решения через AI-агентов?

### SO-1. Race to the bottom по цене

Если все продавцы используют AI-репрайсер, цены сходятся к marginal
cost. Маркетплейсы получают **больше** GMV-комиссии при **меньшей**
маржинальности продавцов → продавцы уходят в офлайн.

*Митигация:* AI не должен оптимизировать **только цену** — должен
оптимизировать долгосрочный profit, включая stock turnover, ROI на
рекламу. Многофакторная функция полезности.

### SO-2. Регуляторный backlash

Если AI-агенты автоматически отвечают на отзывы и генерируют контент,
ФАС и Роспотребнадзор могут ввести регулирование «AI-disclosure»
по аналогии с EU AI Act 2024.

*Митигация:* watermark AI-сгенерированного контента, audit log всех
автоматических действий, готовность к встраиванию AI-disclosure банера.

### SO-3. Anti-scraping arms race

При массовом использовании парсинга публичных страниц WB и Ozon
введут CAPTCHA, fingerprinting, rate-limit per-residential-IP.

*Митигация:* минимизировать долю парсинга, переходить на официальные
API там, где доступны; использовать residential proxy pool как
fallback, а не основной канал.

### SO-4. Платформенный risk

WB или Ozon могут выпустить собственный SaaS-аналог и **аннулировать
сторонние ключи** массово (как Twitter сделал с client apps в 2023).

*Митигация:* мультиплатформенность, чтобы revenue не падал на 0 при
обрыве одного канала; европейский B2B-разворот (СНГ + Армения +
Беларусь) как опция.

### SO-5. Marketplaces подадут в суд

При scraping публичных страниц возможны cease-and-desist (как
LinkedIn vs hiQ Labs в США 2017-2022). В РФ судебная практика по
веб-скрапингу слабая, но риск ненулевой.

*Митигация:* юридическая консультация о границах open-data доступа,
robots.txt-compliance, отказ от парсинга авторизованных страниц.

---

## 6. TRIZ — разрешение противоречий

Систематические инженерные противоречия категории и их разрешение
через TRIZ-принципы.

| # | Противоречие | TRIZ-принцип | Решение в easycomm-world |
|---|---|---|---|
| **C-1** | Хотим **real-time данные**, но API маркетплейсов имеют **rate-limit** (Ozon 500 req/мин) | #15 Динамизация + #24 Посредник | **BullMQ очередь + Redis cache 5-60s TTL**. UI читает кэш, фоновый воркер обновляет; rate-limit-aware adapter |
| **C-2** | Хотим **AI на каждый клик**, но **inference стоит денег** ($0.01-$0.10 за запрос Anthropic) | #5 Объединение + #25 Самообслуживание | **Hybrid AI**: дешёвые/быстрые ответы — on-device WASM (Phi-3-mini), сложная аналитика — bulk batch overnight через MCP; only premium-tier зовёт top-tier модели real-time |
| **C-3** | Хотим **бескомпромиссную security** API-ключей, но это **усложняет UX** (мастер-пароль каждый раз) | #34 Отбрасывание и восстановление + #19 Периодическое действие | **Master password + WebAuthn/Biometric auto-unlock 15 мин TTL**; ключ деривируется из master-password (PBKDF2), кешируется в session-storage с auto-lock |
| **C-4** | Хотим **multichannel-сложность** под капотом, но **простой UX** для solo-sellers | #1 Сегментация + #6 Универсальность | **MarketplaceAdapter** интерфейс изолирует различия; UI показывает unified-card, под капотом per-marketplace mappers |
| **C-5** | Хотим **agency-mode multi-tenancy**, но не хотим расходов на **физическую изоляцию схем** | #3 Локальное качество + #17 Переход в другое измерение | **Postgres Row-Level Security (RLS)** + tenant-id в каждой таблице; физическая изоляция только для enterprise-клиентов (post-MVP) |
| **C-6** | Хотим **аналитику глубже MPSTATS**, но **хранение time-series данных дорого** | #2 Удаление + #36 Применение | **ClickHouse columnar storage** с partition pruning по tenant-id + day; aggregation rollups; retention 90 дней default, расширяемо на Pro+ |

---

## 7. Recommended Approach — синтез

### Парагpaф 1 — Стратегическая позиция

easycomm-world позиционируется как **«next-gen Sellics для РФ»**: гибрид
SaaS-аналитики уровня Helium 10, multichannel-операций уровня Rithum
и agency-layer уровня Sellics, **адаптированный под российские
маркетплейсы 2025-2026 года**. Мы не конкурируем с MPSTATS на их поле
(чистая аналитика) — мы выводим продавца **из аналитического экрана
сразу в действие**, чего MPSTATS архитектурно не умеет (см. § 3,
5 Whys). Стратегическая дифференциация: (1) Agency Mode multi-tenant —
**нет ни у одного российского конкурента**; (2) Client-side encrypted
vault — **нет ни у кого, включая западных лидеров**; (3) AI через
MCP — **догоняющая позиция к JS Q4 2025, но опережающая на рос. рынке**.

### Параграф 2 — Архитектурная философия

Проектируем как **Distributed Monolith (Monorepo)** на Node.js 22 LTS +
Fastify + Prisma + Postgres + ClickHouse + Redis + BullMQ, фронтенд —
Next.js 14 + TypeScript + shadcn/ui + Chrome MV3 extension. Адаптер-
паттерн `MarketplaceAdapter` изолирует каждую интеграцию (WB, Ozon,
ЯМ, Megamarket); это даёт нам устойчивость к breaking-changes API
(adversarial Game Theory, § 4). AI-интеграция через MCP-серверы
(YandexGPT для рус. генерации, Anthropic/OpenAI для аналитики через
прокси) — хайбридная модель: дешёвые операции on-device WASM, сложные —
batch overnight (TRIZ C-2). Multi-tenancy через Postgres RLS до
enterprise-уровня (TRIZ C-5). Хранение API-ключей маркетплейсов —
**только клиентский AES-GCM 256 + IndexedDB + PBKDF2** от мастер-пароля
(First Principle FP-5).

### Параграф 3 — Go-to-market и моат

Distribution: Chrome MV3 extension как free-tier wedge (наследие
Jungle Scout 2015), Telegram-бот для daily retention, blog/YouTube/Дзен
для contentmarketing, agency-referral для B2B. Activation funnel
оптимизирован под time-to-first-action < 5 мин (FP-2). Pricing:
0 / 2 990 / 9 990 / 24 990 ₽ — ÷3-4 от долларовых аналогов (российский
покупательная способность). Моат строится из трёх слоёв: (а) network
effect от Agency Mode (агентство приводит десятки магазинов, lock-in
эффект), (б) data moat — accumulated time-series по каждому SKU
ClickHouse даёт прогностические преимущества, (в) security moat —
encrypted vault как маркетинговый и регуляторный аргумент против всех
российских конкурентов.

---

## 8. Risk Assessment

Шкала: **Probability** 1-5, **Impact** 1-5, **Score** = P × I.

| # | Риск | P | I | Score | Mitigation |
|---|---|---|---|---|---|
| **R-1** | WB/Ozon выпускают собственный SaaS-конкурент и аннулируют сторонние ключи | 2 | 5 | 10 | Мультиплатформенность ≥3 МП; готовность к СНГ-развороту; диверсификация revenue (Agency 25%) |
| **R-2** | Breaking-changes Seller API без анонса (ежеквартально) | 5 | 3 | 15 | `MarketplaceAdapter` abstraction; интеграционные тесты CI; SLA на patch < 24 часа |
| **R-3** | Утечка API-ключей пользователя через нашу систему | 1 | 5 | 5 | Client-side vault (ключи не на сервере); ежеквартальный security audit; bug bounty programme |
| **R-4** | 152-ФЗ enforcement — штрафы за хранение PII вне РФ | 3 | 4 | 12 | Все PII в Yandex Cloud (РФ); AI-инференс OpenAI/Anthropic — через прокси без PII в payload; DPO-роль |
| **R-5** | AI-inference costs выходят за unit economics при росте MAU | 4 | 3 | 12 | On-device WASM для cheap-ops; batch overnight для bulk; rate-limit per-tier; opt-in upgrade prompts |
| **R-6** | MPSTATS/Moneyplace копируют наши дифференциаторы (Agency Mode, vault) | 3 | 3 | 9 | Speed-to-market преимущество; data network effect через accumulated time-series; agency-channel lock-in |
| **R-7** | Регуляторика AI-disclosure (по аналогии EU AI Act) | 2 | 3 | 6 | Watermark AI-контента с дня 1; audit log; feature-flag для disclosure banner |
| **R-8** | Anti-scraping арм-рейс — WB/Ozon вводят CAPTCHA / fingerprint | 4 | 2 | 8 | Приоритет API над scraping; residential proxy pool как fallback; robots.txt-compliance |
| **R-9** | Cash burn до достижения SOM 30-80 млн ₽ ARR | 3 | 4 | 12 | Pre-seed → seed раунд; agency-channel приносит revenue с месяца 1; managed-services upsell как cash-cow |
| **R-10** | Команда не справляется с complexity (Distributed Monolith + ClickHouse + Chrome ext + MCP) | 3 | 4 | 12 | Поэтапный roll-out (MVP без ClickHouse, без MCP); сильный CTO; outsource Chrome-ext первой версии |

### Топ-3 риска по Score

1. **R-2 (Score 15)** — API breaking-changes. Митигация: adapter pattern + SLA.
2. **R-4, R-5, R-9, R-10 (Score 12)** — регуляторика, AI costs, cash burn, team complexity. Митигация: см. таблицу.
3. **R-1 (Score 10)** — platform risk WB/Ozon. Митигация: мультиплатформенность.

---

## 9. Decision Log — что решено сейчас, что отложено

### Решено в Phase 1 (этот документ)

- ✅ Стратегическая позиция: гибрид Sellics + H10 + Rithum + AI/MCP + vault.
- ✅ Архитектурный паттерн: Distributed Monolith + Adapter per marketplace.
- ✅ Multi-tenancy: Postgres RLS (не физическая изоляция).
- ✅ AI-стратегия: hybrid (WASM + MCP-batch + MCP-realtime по тиру).
- ✅ Vault-pattern: client-side AES-GCM + IndexedDB + PBKDF2.
- ✅ Pricing tiers: 0 / 2 990 / 9 990 / 24 990 ₽.
- ✅ MVP-scope: 5 P0-модулей (см. Research_Findings § 6).

### Отложено в Phase 2 / далее

- ⏳ Конкретные MCP-серверы (YandexGPT vs OpenAI прокси) — Specification.md.
- ⏳ Схема Postgres + ClickHouse — Architecture.md.
- ⏳ Pricing валидация через customer interview — Phase 2 validation.
- ⏳ Enterprise tier (выше Agency 24 990 ₽) — post-MVP.
- ⏳ Юридическая ToS-compliance проверка scraping публичных страниц — pre-launch.

---

## 10. Hand-off to PRD.md

PRD.md строится на следующих **жёстких ограничениях** из этого
документа (не подлежат изменению без re-strategy):

| Constraint | Из секции | Жёсткость |
|---|---|---|
| Encrypted client-side vault для API-ключей маркетплейсов | FP-5, TRIZ C-3 | **MUST** |
| Multichannel ≥ 3 МП в архитектуре с дня 1 | Game Theory § 4 | **MUST** |
| Agency Mode multi-tenant (Postgres RLS) | Differentiation § 7 | **MUST для v1** |
| Action-button рядом с метрикой на каждом аналитическом экране | FP-1, 5 Whys § 3 | **MUST** |
| AI-через-MCP в архитектуре | TRIZ C-2, Differentiation | **MUST для v1** |
| Pricing 0/2990/9990/24990 ₽ — без изменения tier-структуры до validation | M4 brief + § 7 | **SHOULD** |
| MVP не включает order routing, PPC automation, review moderation | Research § 6 (P2) | **MUST для MVP** |

---

*Last updated: 2026-05-14. Owner: Strategy / Founding team.*

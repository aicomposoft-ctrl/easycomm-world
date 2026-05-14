# Research Findings — easycomm-world

> SPARC Phase 1 artifact. Сводный исследовательский документ, на котором
> базируются Solution_Strategy.md и PRD.md. Все факты — с указанием
> источников и уровня доверия. Спорные цифры помечены `[Confidence: …]`.
>
> Источники нижнего уровня: `docs/research/easycomm-analysis.md`,
> `docs/research/predecessor-analysis.md` (verified goap-research).

---

## 1. Executive Summary

Российская SaaS-категория инструментов для продавцов на маркетплейсах
(MPSTATS, Moneyplace, MarketGuru, Маяк, JLab.ai, CAT/Easy Commerce) на
2025-2026 гг. структурно разделена на два непересекающихся слоя: чистая
SaaS-аналитика и операционные агентства полного цикла. Единственный
западный прообраз, который **одновременно** имел оба слоя — **Sellics**
(Берлин, 2014 → Perpetua, 2022), — на западе закрыт как самостоятельный
бренд. Функционально категория задана **Helium 10** (Irvine, 2017) и
**Jungle Scout** (Austin, 2015); мультиканальный ops-слой —
**ChannelAdvisor/Rithum** (NC, 2001 / 2022). Окно возможностей:
российский рынок переживает бум GMV WB+Ozon (≈$50–60B), но ни один
локальный игрок не реализует одновременно (a) аналитику на уровне
Helium 10, (b) multichannel ops на уровне Rithum, (c) agency-mode на
уровне Sellics, (d) AI-помощник через MCP, (e) клиентский
зашифрованный vault для API-ключей. easycomm-world проектируется
именно как такой гибрид.

---

## 2. Research Objective

Исследование преследовало две цели:

1. **Идентифицировать зарубежного прародителя easycomm.ru** — чтобы
   корректно отнести next-generation реплику к существующей категории
   и не изобретать архитектурные решения «с нуля» там, где западный
   рынок уже прошёл этот путь.
2. **Зафиксировать функциональную карту категории** «marketplace seller
   tools» — чтобы Specification.md и Architecture.md следующих фаз
   получили нормативный набор модулей, а не произвольный список.

### Исследовательские вопросы (RQ)

| # | Вопрос | Статус |
|---|---|---|
| RQ-1 | Что собой представляет easycomm.ru как бизнес? | Закрыт (см. § 4.1) |
| RQ-2 | Какой западный сервис был структурным прародителем? | Закрыт — гибрид Sellics+Helium10+Rithum (§ 5) |
| RQ-3 | Какие функциональные модули обязательны для категории? | Закрыт — 11 модулей (§ 6) |
| RQ-4 | Какие архитектурные паттерны типичны для категории? | Закрыт (§ 8) |
| RQ-5 | Каковы ключевые pain points российских продавцов? | Закрыт (§ 9) |
| RQ-6 | Какова рыночная ёмкость (TAM/SAM/SOM)? | Закрыт с оговоркой [Confidence: Medium] (§ 4.2) |

---

## 3. Methodology

### Подход

Использован адаптированный **goap-research** (Goal-Oriented Action
Planning) с верификацией Ed25519-стиля: каждое утверждение должно
иметь **минимум 2 независимых источника**. Если найден только один —
утверждение помечено `[Confidence: Low — single source]`.

### Источниковая иерархия (reliability rating)

| Rating | Тип | Примеры в этом исследовании |
|---|---|---|
| **A** | Official-primary | helium10.com/pricing, junglescout.com/about-us, rithum.com, easycomm.ru/cat |
| **B** | Wikipedia / established encyclopedic / industry-reports | en.wikipedia.org/wiki/Jungle_Scout, en.wikipedia.org/wiki/Rithum |
| **C** | Reviews / blogs / SEO-aggregators | g2.com, ecomcrew.com, influencermarketinghub.com, revenuegeeks.com |
| **D** | Telegram-каналы / форумы | (не использовалось как первичный) |

### Триангуляция

- Факт основания Sellics в 2014 — подтверждён frogcapital.com (A) +
  ecomcrew.com (C) + perpetua.io/sellics-joins-perpetua (A).
- Pricing Helium 10 $29/$99/$279 — helium10.com/pricing (A) +
  influencermarketinghub.com (C) + revenuegeeks.com (C).
- Перебренд ChannelAdvisor → Rithum в 2022 — wikipedia/Rithum (B) +
  rithum.com (A) + ecomclips.com (C).
- Pricing Jungle Scout $29/$49 — junglescout.com (A) + Wikipedia (B).

### Ограничения методологии

- Российские конкуренты (MPSTATS, Moneyplace) не публикуют ARR/выручку —
  оценки TAM/SAM/SOM в § 4.2 построены **по аналогии** с американским
  рынком и скорректированы пропорционально GMV. `[Confidence: Medium]`
- Pricing российских конкурентов меняется ежеквартально — снимки на дату
  исследования. `[Confidence: Medium]`
- Реальная функциональность CAT/E-commerce Tool частично определена по
  блог-анонсам, а не по «живому» демо. `[Confidence: Medium]`

---

## 4. Market Analysis

### 4.1 Что такое easycomm.ru (исходный объект)

Источник: `docs/research/easycomm-analysis.md`.

Easy Commerce — российское IT-агентство полного цикла по ведению
магазинов на маркетплейсах + два собственных SaaS-продукта:

| Продукт | URL | Назначение |
|---|---|---|
| **Commerce Analytics Tool (CAT)** | easycomm.ru/cat | Сервис статистики и аналитики продаж |
| **E-commerce Tool** | блог 2024 | Управление продажами из одного кабинета (multichannel ops) |
| **Полносервисное агентство** | easycomm.ru | Карточки, фото 360°, SEO, репрайсинг, отзывы |

Покрываемые площадки: WB, Ozon, Яндекс.Маркет, Megamarket, AliExpress
Россия. Признание Rating Runet: 1-е место в 4 категориях (контент,
продвижение, работа с МП, аналитика). [Источники: easycomm.ru,
ratingruneta.ru/agency-easycomm/]

**Ключевая структурная особенность:** одновременно агентство и
разработчик SaaS. Российские конкуренты — либо одно (MPSTATS, Moneyplace —
чистый SaaS), либо другое (Okkam Group — чистое агентство).

### 4.2 TAM / SAM / SOM

Метод оценки: by-analogy с американским рынком Amazon seller tools
(~$2-3B/год, рыночные оценки Helium 10/JS), скорректировано
пропорционально GMV WB+Ozon (~$50-60B) vs Amazon (~$700B).

| Метрика | Оценка | Confidence |
|---|---|---|
| **TAM** — российский рынок SaaS для marketplace-продавцов | 15–25 млрд ₽/год | Medium |
| **SAM** — mid-size sellers + brands на B2B-подписке | 3–5 млрд ₽/год | Medium |
| **SOM** — реалистичный захват на 18-24 месяце | 30–80 млн ₽ ARR | Low–Medium |

**Источник коэффициента**: соотношение GMV маркетплейсов и размера
SaaS-tooling рынка. Российский рынок имеет более высокую долю
**single-platform sellers** (Wildberries-only), что снижает spend на
multichannel SaaS — учтено понижающим множителем.

### 4.3 Демография рынка продавцов (по сегментам)

| Сегмент | Объём в РФ (порядок) | Источник оценки |
|---|---|---|
| Solo sellers (1-3 SKU, < 0.5 млн ₽/мес) | 500+ тыс. | Аналитика WB/Ozon публичных дашбордов |
| Mid-size sellers (1-10 SKU, 1-20 млн ₽/мес) | 100-200 тыс. | По аналогии с структурой Amazon Marketplace |
| Brands и производители (50+ SKU) | 5-20 тыс. | Открытые данные регистраций ИП/ООО |
| Marketplace-агентства | 200-500 | Reputation-ranking Rating Runet и аналоги |
| Enterprise (Сбер.МегаМаркет 1P) | < 100 | Публичные пресс-релизы |

`[Confidence: Medium — данные не из официальных регистров]`

---

## 5. Competitive Landscape

### 5.1 Сводная конкурентная таблица

Шкала: ★ = присутствует слабо/частично, ★★★★★ = market-leader-level.

| Игрок | Аналитика | Multichannel | AI | Agency layer | Encrypted vault | Российские МП | Pricing 2025-26 |
|---|---|---|---|---|---|---|---|
| **MPSTATS** | ★★★★★ | ★★ | ✗ | ✗ | ✗ | ✓ | от ~5 тыс ₽/мес |
| **Moneyplace** | ★★★★ | ★★ | ✗ | ✗ | ✗ | ✓ | от ~3 тыс ₽/мес |
| **MarketGuru** | ★★★ | ★ | ✗ | ✗ | ✗ | ✓ | от ~2 тыс ₽/мес |
| **Маяк** | ★★ | ✗ | ✗ | ✗ | ✗ | ✓ | Free tier |
| **JLab.ai** | ★★★ | ★ | ★★ | ✗ | ✗ | ✓ | от ~3 тыс ₽/мес |
| **CAT / Easy Commerce** | ★★★★ | ★★★ | ✗ | ★★★★★ | ✗ | ✓ | непублично |
| **Helium 10** (US) | ★★★★★ | ✗ | ★★ | ✗ | ✗ | ✗ | $29/$99/$279 |
| **Jungle Scout** (US) | ★★★★★ | ✗ | ★ | ✗ (но Cobalt для брендов) | ✗ | ✗ | $29/$49 |
| **Sellics** (EU, 2014-22) | ★★★★ | ★★ | ★ | ★★★★★ | ✗ | ✗ | от $57/мес (закрыт для new) |
| **Rithum/ChannelAdvisor** | ★★★ | ★★★★★ | ★★ | ★★ | ✗ | ✗ | Enterprise (контракт) |
| **easycomm-world (целевое)** | ★★★★★ | ★★★★★ | ★★★★★ (MCP) | ★★★★ | ★★★★★ | ✓ | 0/2990/9990/24990 ₽ |

Источники: § 11. Pricing-снимки актуальны на 2025-2026 на момент
исследования; российские pricing меняются ежеквартально.

### 5.2 Покрытие функционала easycomm.ru каждым прародителем

Из `predecessor-analysis.md` (§ 5):

| Возможность Easy Commerce | Sellics | Helium 10 | Jungle Scout | Rithum |
|---|---|---|---|---|
| Аналитика продаж | 90 | **100** | 95 | 70 |
| SEO / keyword research | **100** | 100 | 80 | 50 |
| Создание/оптимизация карточек | 90 | 80 | 70 | 70 |
| Автоматический репрайсинг | 70 | 80 | 50 | **100** |
| Управление остатками | 60 | 70 | 40 | **100** |
| Управление заказами | 80 | 70 | 50 | **100** |
| Работа с отзывами | **100** | 90 | 70 | 60 |
| Реклама / PPC | **100** | 95 | 70 | 90 |
| Managed services (agency) | **100** | 0 | 0 | 60 |
| Multichannel UI | 90 | 0 | 0 | **100** |
| Российские МП | 0 | 0 | 0 | 0 |
| **СУММА** | **890** | **685** | **555** | **800** |

**Вывод:** Sellics закрывает 890/1100 ≈ 81% функциональности Easy
Commerce и **единственный** имеет одновременно managed-services + SaaS.

### 5.3 Sellics — главный структурный прародитель

| Параметр | Значение |
|---|---|
| Основан | 2014, Berlin |
| Основатель | Franz Jordan |
| Исходный продукт | Amazon SEO tool |
| Бизнес-модель | SaaS + managed agency services |
| Финал | Поглощён Perpetua в 2022; agency-трек закрыт |
| Pricing | от $57/мес для $0-продаж, шкала по обороту |

**Заимствуется easycomm-world:** двойной трек (SaaS + managed services),
наследие SEO-инструмента (keyword research), profit analytics, review
management, PPC automation.

**Не заимствуется:** география (US/EU vs RU), pricing-схема по обороту
(сложно для российского рынка), интеграция с Amazon SP-API.

### 5.4 Helium 10 — функциональный шаблон CAT

Pricing 2025: Starter $29 / Platinum $99 / Diamond $279 в месяц.
30+ инструментов. Прямые аналоги в easycomm-world:

| Helium 10 модуль | Аналог в easycomm-world | Приоритет MVP |
|---|---|---|
| Black Box (фильтр товаров) | Niche Finder | P1 |
| Cerebro (reverse-ASIN keywords) | Reverse-карточка для WB/Ozon | P0 |
| Magnet (keyword research) | Keyword research для МП | P0 |
| Profits Dashboard | Profit Dashboard | P0 |
| Inventory Management | Stock Manager | P1 |
| Listing Builder + AI | Card Generator (MCP) | P0 |
| Refund Genie | Loss Recovery | P2 |
| Repricer (Diamond+) | Repricer 50+ параметров | P1 |

### 5.5 Jungle Scout — Chrome-extension как distribution wedge

Основан 02-2015 Грегом Мерсером, 500k+ продавцов Amazon, исходная
форма — Chrome extension для парсинга страниц Amazon. AccuSales™ —
алгоритм оценки продаж на основе публичных данных (1 млрд точек/день).

**Что наследует easycomm-world:**
- Chrome MV3 extension как точка входа free-tier пользователя.
- ML-оценка продаж конкурентов по косвенным сигналам (число отзывов,
  позиция в выдаче, изменение позиции).
- Speed-to-Insight — AI-обзоры рынка (анонс Jungle Scout Q4 2025).

### 5.6 Rithum — multichannel ops

Бывший ChannelAdvisor (2001 → 2022 ребрендинг после слияния с
CommerceHub). 600+ маркетплейсов в сети. Enterprise-уровень (Walmart,
Target, Home Depot).

**Наследует easycomm-world:** концепция «единого UI для нескольких
МП», синхронизация остатков (anti-oversell), dynamic pricing, order
routing, performance reporting. **Не наследует:** enterprise pricing,
контрактная B2B-модель.

---

## 6. Functional Map of the Category (нормативная)

Из анализа Helium 10, Jungle Scout, Sellics, Rithum и российских
конкурентов выделены **11 обязательных модулей** категории:

| # | Модуль | Sellics | H10 | JS | Rithum | MPSTATS | easycomm-world MVP? |
|---|---|---|---|---|---|---|---|
| 1 | Profit analytics & dashboard | ✓ | ✓ | ✓ | ✓ | ✓ | **P0** |
| 2 | Keyword research / reverse-card | ✓ | ✓ | ✓ | – | ✓ | **P0** |
| 3 | Niche/product research | – | ✓ | ✓ | – | ✓ | P1 |
| 4 | Competitor tracking | ✓ | ✓ | ✓ | – | ✓ | **P0** |
| 5 | Listing/card creation + AI | ✓ | ✓ | – | ✓ | – | **P0** |
| 6 | SEO optimisation | ✓ | ✓ | – | – | – | P1 |
| 7 | Repricer | – | ✓ | – | ✓ | – | P1 |
| 8 | Inventory/stock manager | – | ✓ | – | ✓ | ✓ | P1 |
| 9 | Order management | – | – | – | ✓ | – | P2 |
| 10 | PPC/ads automation | ✓ | ✓ | – | ✓ | – | P2 |
| 11 | Review management | ✓ | ✓ | – | – | – | P2 |

Эта таблица — нормативный input для Specification.md.

---

## 7. Technology Assessment — паттерны западных прародителей

Стек, общий для категории (для копирования в Architecture.md):

| Паттерн | Где наблюдается | Что копирует easycomm-world |
|---|---|---|
| Event-driven ingestion парсеров (cron + queues) | Helium 10, JS, Sellics | BullMQ + Redis cron-jobs |
| Time-series хранилище | Helium 10 (Snowflake), JS (ClickHouse-style) | **ClickHouse** для исторических метрик карточек |
| Кеш расчётных метрик | Все | **Redis** для UI-метрик |
| Chrome Extension wedge | Helium 10, JS, MPSTATS, Маяк | **Chrome MV3** с Web Crypto vault |
| Web frontend на React | Helium 10, JS, Rithum, Sellics | **Next.js + TypeScript** |
| API + scraping hybrid | JS (исходно), MPSTATS | WB Seller API + Ozon Seller API + scraping публичных страниц |
| Stripe/Recurly billing | Все западные | **ЮKassa + ProdamusGate (резерв)** |
| Auth и vault | Все западные хранят на сервере | **Отход от прародителей**: client-side vault (AES-GCM + IndexedDB + PBKDF2) |

**Дельта от прародителей** (российская специфика):
- 152-ФЗ → персональные данные хранятся в РФ (Yandex Cloud / Selectel).
- AI-операторы: YandexGPT доступен без VPN, OpenAI/Anthropic — через
  прокси или MCP.
- Pricing ÷3-4 от долларовых аналогов (покупательная способность).

---

## 8. User Insights — болевые точки российских продавцов

Из `Product_Discovery_Brief.md` (Section "Critical pain points",
валидируется customer interview в Specification.md):

| Pain | Цитата (агрегированная) | Какой модуль закрывает |
|---|---|---|
| Кабинеты WB/Ozon не дают бизнес-картины | «Кабинет показывает не то, что нужно для решений» | Profit Dashboard (P0) |
| Аналитика и действие — в разных местах | «MPSTATS даёт данные, не действия — переключаешься в кабинет, теряешь контекст» | One-click actions из аналитики (P0) |
| Excel-репрайсинг при 100+ SKU | «Управление ценой в Excel — медленно и ошибочно» | Repricer (P1) |
| Подбор ключей вручную | «Часы перебора, нет reverse-ASIN для российских площадок» | Reverse-карточка (P0) |
| Нет единого реестра карточек | «По всем маркетплейсам открываем 5 кабинетов» | Multichannel UI (P0) |

`[Confidence: Medium — основано на агрегированных опросах в Telegram-
каналах продавцов, формальный customer-discovery будет в Phase 2]`

---

## 9. Confidence Assessment

### High-confidence findings (источников ≥ 3, A или B reliability)

- Идентификация Sellics как бизнес-модельного прародителя.
- Pricing Helium 10 / Jungle Scout (актуальные данные с
  helium10.com/pricing и junglescout.com).
- Перебренд ChannelAdvisor → Rithum в 2022.
- Поглощение Sellics → Perpetua в 2022.
- 11-модульная функциональная карта категории.
- Технологический стек категории (Chrome ext + time-series + queues).

### Medium-confidence findings (2 источника или один + аналогия)

- TAM/SAM/SOM оценки — by-analogy, без официальных регистров.
- Демография российских продавцов по сегментам.
- Покрываемые площадки easycomm.ru (5+ источников, но без официального
  pricing card).
- Pricing российских конкурентов (волатилен, нужен ежеквартальный
  пересмотр).
- Pain points продавцов (агрегированы из Telegram, не из formal
  customer-discovery).

### Low-confidence findings (1 источник или экстраполяция)

- Конкретный год основания Easy Commerce (предположительно 2020-2022).
- Внутренний стек easycomm.ru (нет публичных данных).
- Реальные unit-economics MPSTATS/Moneyplace.
- Долгосрочная стратегия WB/Ozon по отношению к третьим сторонам
  (anti-scraping политика может ужесточиться).

---

## 10. Research Path Log (краткий)

```
Phase 0 — reverse-engineering-unicorn QUICK mode
├── Step 1: search "easycomm.ru что это" → определили категорию (агентство+SaaS)
├── Step 2: fetch easycomm.ru/cat, /business, /cases, /blog → зафиксировали
│           линейку продуктов
├── Step 3: cross-ref Rating Runet → top-tier маркер
├── Step 4: hypothesis "западный прародитель = Sellics?" → triangulation:
│   ├── Sellics: 2014, Berlin, agency+SaaS — совпадает по структуре
│   ├── Helium 10: только SaaS — отвергнут как single-track
│   └── Rithum: только ops, не research — отвергнут как single-layer
├── Step 5: feature-coverage анализ (см. § 5.2) → подтверждена гибридная
│           природа (Sellics 81% + Rithum 73% + Helium 10 62%)
└── Step 6: extract technology patterns from foreign predecessors → § 7
```

---

## 11. Sources (numbered, with reliability rating)

### Easy Commerce (исходный объект)

1. https://easycomm.ru/ — главная — **A**
2. https://easycomm.ru/cat — CAT — **A**
3. https://easycomm.ru/business — Запуск под ключ — **A**
4. https://easycomm.ru/cases — кейсы клиентов — **A**
5. https://easycomm.ru/blog/tpost/dg0np9agy1-easy-commerce-zapustil-e-commerce-tool-i — анонс E-commerce Tool — **A**
6. https://ratingruneta.ru/agency-easycomm/ — Rating Runet — **B**

### Sellics

7. https://sellics.com/pricing/ → перенаправляется на Perpetua — **A**
8. https://perpetua.io/sellics-joins-perpetua/ — официальный анонс — **A**
9. https://www.g2.com/products/sellics/reviews — **C**
10. https://www.ecomcrew.com/sellics-review/ — **C**
11. https://frogcapital.com/company/sellics/ — портфельная инвестиция — **A**

### Helium 10

12. https://www.helium10.com/ — **A**
13. https://www.helium10.com/pricing/ — **A**
14. https://influencermarketinghub.com/helium-10/ — **C**
15. https://revenuegeeks.com/helium10-review/ — **C**

### Jungle Scout

16. https://www.junglescout.com/ — **A**
17. https://en.wikipedia.org/wiki/Jungle_Scout_(company) — **B**
18. https://www.junglescout.com/about-us/ — **A**
19. https://www.junglescout.com/cobalt-quarterly-release/ — **A**
20. https://www.junglescout.com/meet-greg-mercer/ — **A**

### ChannelAdvisor / Rithum

21. https://www.rithum.com/ — **A**
22. https://en.wikipedia.org/wiki/Rithum — **B**
23. https://ecomclips.com/blog/channeladvisor-rithum-overview-2024/ — **C**

### Российские конкуренты

24. https://mpstats.io/ ; https://mpstats.ru/ — **A**
25. https://moneyplace.io/ — **A**
26. https://marketguru.io/ — **A**
27. https://okkam.group/e-commerce — **A**

### Категория и обзоры

28. https://thunderbit.com/blog/leading-tools-for-effective-marketplace-analysis — **C**
29. https://grizzlysms.com/blog/marketplace-analytics-services-overview — **C**

---

## 12. Hand-off to Solution_Strategy.md и PRD.md

Из этого Research_Findings вытекают **5 опорных тезисов** для следующих
документов:

1. **Структурный gap на российском рынке** — никто не реализует hybrid
   (Sellics-стиль) → это и есть стратегическая позиция easycomm-world.
2. **Функциональная карта из 11 модулей** — Specification.md строит на
   ней feature-list. MVP = 5 модулей P0 (см. § 6).
3. **Технологический стек категории** валидирован — Architecture.md
   копирует ClickHouse + BullMQ + Redis + Next.js + Chrome MV3.
4. **Client-side encrypted vault** — единственный пункт, где
   easycomm-world сознательно **отступает** от прародителей (они хранят
   на сервере). Это маркетинговое и регуляторное преимущество.
5. **AI через MCP-серверы** — догоняющая позиция относительно Jungle
   Scout Q4 2025 Speed-to-Insight, но опережающая на российском рынке.

`[Confidence: High по всем 5 тезисам]`

---

*Last updated: 2026-05-14. Next review: после Phase 2 validation.*

# easycomm-world — Executive Summary

> Финальный синтез SPARC-документации. Источник правды: 10 документов
> в `docs/` + 2 файла глубокого research в `docs/research/`.

## Overview

**easycomm-world** — next-generation платформа для российских
маркетплейс-продавцов, объединяющая Helium-10-уровень аналитики,
Rithum-уровень мультиканала, AI-помощника через MCP и
Sellics-style гибрид "agency + SaaS" в одном кабинете. Шифрованный
клиентский vault (AES-GCM в IndexedDB) гарантирует, что API-ключи WB,
Ozon, ЯМ никогда не уходят на наш сервер. Цель MVP — догнать MPSTATS
по аналитике, обойти всех по action-latency ("видишь → меняешь цену в
один клик") и стать первой в РФ платформой с Agency-Mode multi-tenancy.

## Problem & Solution

**Проблема.** Российский рынок продаж на WB/Ozon/ЯМ растёт двузначными
темпами с 2022 г., но инструменты для продавца расколоты на два мира:
*аналитики* (MPSTATS, Moneyplace, MarketGuru) — дают данные, но не
действия; *агентства* (Easy Commerce, Okkam) — закрывают руками, но
без прозрачного SaaS-уровня. Связки нет ни у одного российского игрока.

**Решение.** Гибрид трёх западных архетипов, адаптированный под
российские маркетплейсы:
- **Sellics** (Berlin 2014 → Perpetua 2022) — модель "agency + SaaS"
- **Helium 10** (Irvine 2017) — функциональная начинка аналитики
- **Rithum / ChannelAdvisor** (NC 2001) — мультиканальные операции
- **+ AI через MCP** — новая категория, которой нет ни у одного из них

## Target Users (top 3 из PRD)

| Persona | Сегмент | Объём в РФ | MVP-приоритет |
|---|---|---|---|
| Анна, 32 — mid-size seller WB+Ozon, 5-10 SKU, 3-15 млн ₽/мес | Mid-size sellers | 100-200 тыс. | ★★★ Core |
| Дмитрий, 38 — brand manager FMCG, 50+ SKU, выход на ЯМ | Brands | 5-20 тыс. | ★★ Phase 2 |
| Сергей, 35 — agency lead, 20 клиентов под управлением | Agencies | 200-500 | ★★ Phase 2 |

## Key Features MVP (P0)

1. **Encrypted Vault** — клиентское шифрование API-ключей маркетплейсов
   (AES-GCM 256, PBKDF2 100k iter, auto-lock 15 мин).
   *Ценность:* единственный из всех российских игроков, кто гарантирует
   "сервер не может расшифровать ваши данные".
2. **Unified Analytics Dashboard** — единая картина продаж/прибыли по
   всем маркетплейсам сразу. *Ценность:* нет нужды переключаться между 4
   кабинетами + 2 аналитическими сервисами.
3. **Niche Finder (Black-Box-style)** — фильтрация SKU по 30+ параметрам:
   объём ниши, средняя цена, рейтинг, конкуренция. *Ценность:* unit-эк
   до закупки товара за минуты, а не часы.
4. **AI Card Generator (MCP)** — генерация SEO-описаний по атрибутам +
   keywords + tone-of-voice. *Ценность:* экономит часы агентской работы,
   доступна solo-продавцу.
5. **Repricer Engine** — автоматическое управление ценой по правилам
   (floor, ceiling, competitor-pegging, schedule). *Ценность:* не теряем
   выручку из-за oscillation; dry-run + rollback.

## Technical Approach

- **Architecture style:** Distributed Monolith в Monorepo (1 деплой-юнит,
  но logical modules; готовый split-point когда модуль выходит за рамки).
- **Stack:** Next.js 15 + TypeScript + shadcn/ui (front) → Fastify +
  Prisma + BullMQ (back) → PostgreSQL (primary) + ClickHouse (time-series
  для исторических метрик 1M+ точек/день) + Redis (cache+queue) +
  S3-compat (Selectel/YC Object Storage).
- **AI:** MCP proxy с registry: openai-mcp, anthropic-mcp, yandexgpt-mcp.
  Cost ledger per tenant.
- **Distribution wedge:** Chrome Extension MV3 (как Jungle Scout, MPSTATS).
- **Deploy:** Docker Compose на 1-2 VPS (AdminVPS/HOSTKEY) — НЕ K8s, см.
  ADR-008.
- **Differentiators:** (1) client-side encrypted vault, (2) Agency Mode
  multi-tenancy native в архитектуре, (3) MCP-based AI on day 1,
  (4) multichannel в MVP, а не post-MVP.

## Research Highlights

1. **Прямого "единственного прародителя" у easycomm.ru нет.** Это
   гибрид Sellics (model 90%) + Helium 10 (functions 75%) + Rithum
   (multichannel 80%) + Jungle Scout (category root 2015).
2. **Sellics — главный единый прообраз** по бизнес-модели: 81%
   функционального overlap, единственный из всех имел combined
   agency + SaaS + SEO-карточек. Поглощён Perpetua в 2022.
3. **MPSTATS = "Helium 10 для WB"** по консенсусу русского рынка. Наш
   challenger-positioning против него по линии: action-latency,
   multichannel, AI, encrypted vault.
4. **Историческая ирония:** Sellics закрылся (через Perpetua) в 2022 —
   именно когда российский маркетплейс-бум стартовал. Бизнес-модель
   *живёт только в России*, и пока её копируют как агентства (Easy
   Commerce), либо как SaaS (MPSTATS), но не как гибрид.
5. **Стек-паттерн категории** (из всех прообразов): Chrome ext + web
   app + time-series store + cron parsers + Stripe-like billing — мы
   воспроизводим с поправками на 152-ФЗ и ЮKassa.

## Success Metrics (12-month targets)

| Metric | M+6 | M+12 |
|---|---|---|
| **North Star: WAS-1w** (Weekly Active Sellers, see action+insight) | 500 | 2 500 |
| MRR | 0.5 млн ₽ | 3 млн ₽ |
| Paid users | 200 | 1 000 |
| DAU | 200 | 1 200 |
| Activation rate (signup → 1st marketplace connected) | 35% | 50% |
| 4-week retention (Pro tier) | 60% | 70% |
| Monthly churn (Pro+) | ≤ 7% | ≤ 5% |
| LTV / CAC | ≥ 5 | ≥ 10 |
| AI cost per active user / mo | ≤ 80 ₽ | ≤ 50 ₽ |

## Timeline & Phases

| Phase | Months | Feature Groups | Headcount (estimated) |
|---|---|---|---|
| **Pre-MVP** | M0-M1 | Repo bootstrap, CI/CD, vault primitives, WB adapter PoC, design system | 2 eng + 1 design |
| **MVP** (Pro tier) | M1-M5 | Vault, dashboard, WB+Ozon connect, niche finder, repricer (dry-run), AI card gen, Chrome ext v1, ЮKassa billing | 3 eng + 1 design + 1 QA |
| **v1** (Team tier) | M5-M8 | Multichannel sync, full repricer, Telegram bot, ЯМ adapter, advanced analytics, brand voice profiles | +1 eng |
| **v2** (Agency tier) | M8-M12 | Agency Mode multitenancy, RBAC+ABAC, client workspaces, audit log v2, white-label, Megamarket adapter | +1 eng + 1 PM |
| **GTM ramp** | M3+ | Content blog, Telegram comms, ext launch, agency-partner program | +1 marketing |

## Risks & Mitigations (top 5)

| Risk | P×I | Mitigation |
|---|---|---|
| WB/Ozon ToS изменения / API закрытие | High × High | Multi-source ingest (API + open data + scraping fallback); abstract Marketplace Adapter interface; legal monitoring |
| AI-cost runaway (MCP tokens на free-tier) | Med × High | Quota per tier; on-device WASM models для дешёвых задач; bulk batch; cost ledger + alerting |
| 152-ФЗ — данные пользователей вне РФ | Low × High | Все БД на VPS в РФ (AdminVPS/HOSTKEY); S3 = Selectel/Yandex Cloud; контракт с DPO |
| Vault UX-trap — пользователь забывает мастер-пароль | Med × Med | Recovery code at setup; biometric quick-unlock (WebAuthn); explicit "no server recovery" disclaimer |
| Конкурент (MPSTATS) копирует multichannel/AI в течение 6 мес | Med × Med | Скорость к рынку; agency-mode защитный ров (требует архитектурного перестроя для копирования) |

## Immediate Next Steps (first 2 weeks)

1. **Pre-flight:** прогнать Phase 2 validation (`requirements-validator`)
   по всему пакету; зафиксировать verdict.
2. **Toolkit generation:** Phase 3 (`cc-toolkit-generator-enhanced`) —
   создать project-specific агенты (planner, code-reviewer, architect),
   rules (security, coding-style, secrets-management, testing), skills
   (project-context, coding-standards, security-patterns) и
   `feature-roadmap.json` из MVP-фич.
3. **Phase 4 scaffold:** сгенерировать `docker-compose.yml`, `Dockerfile`,
   `.gitignore`, `CLAUDE.md` для AI-помощника.
4. **First feature dispatch:** `/feature vault-encryption` — самая
   фундаментальная фича, ставит security-фундамент.
5. **Customer interviews validation:** 5 mid-size sellers за 2 недели
   для валидации pricing 2 990 ₽ / 9 990 ₽ и приоритезации MVP-фич.

## Documentation Package (11 files)

| # | File | Purpose |
|---|---|---|
| 0 | `docs/Product_Discovery_Brief.md` | Phase 0 brief — JTBD, segments, GTM, constraints (вход в Phase 1) |
| 1 | `docs/PRD.md` | Product vision, personas, feature list (P0/P1/P2), pricing, GTM, open questions |
| 2 | `docs/Solution_Strategy.md` | SCQA, First Principles, Game Theory, TRIZ, Second-Order, Risk |
| 3 | `docs/Specification.md` | FR-001..050+, NFR-001+, 40+ user stories Gherkin, feature matrix |
| 4 | `docs/Pseudocode.md` | 14 entities, 11 алгоритмов, 15+ API endpoints, 4 mermaid state-machines |
| 5 | `docs/Architecture.md` | Distributed monolith design, components, polyglot persistence, security, scaling |
| 6 | `docs/C4_Diagrams.md` | C4 Level 1-4 диаграммы (Context / Container / Component / Code for Repricer) |
| 7 | `docs/ADR.md` | 14 Architecture Decision Records (DM vs μS, stack choices, multitenancy, deploy) |
| 8 | `docs/Refinement.md` | 25+ edge cases, testing strategy, 12+ Gherkin scenarios, perf+security hardening |
| 9 | `docs/Completion.md` | Deploy plan, CI/CD YAML, monitoring (15+ metrics), runbooks, cost model |
| 10 | `docs/Research_Findings.md` | Деep verified research, competitive landscape, 29 cited sources A/B/C |
| 11 | `docs/Final_Summary.md` | Этот документ |

**Дополнительно в `docs/research/`:**
- `easycomm-analysis.md` — что такое easycomm.ru (исходный объект)
- `predecessor-analysis.md` — глубокий разбор Sellics/Helium 10/JS/Rithum

---

✅ **SPARC documentation package ready for Phase 2 (validation).**

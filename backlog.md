# Бэклог экосистемы pinout (верхнеуровневый)

Платформенный план. Детальные пер-сервисные бэклоги — в `docs/design/<slice>/` каждого репозитория (скилл
`program-design`). Здесь — **эпики, распределённые по компонентам как независимые конвейеры**, зависимости
и порядок. Каждый эпик = отдельный харнес-прогон в своём репо; сходятся через общий формат отчёта и общую
модель контракта.

Парадигма — [`docs/CONCEPT.md`](docs/CONCEPT.md). Она **заменяет** раннюю формулировку «чистая функция
сравнения двух полных спек» на актуальную (ниже).

## 🔴 Приоритет №1 — долг: форк канона отчёта

**[`debt/report-canon-fork.md`](debt/report-canon-fork.md)** — два валидатора заморозили несовместимые
выходные контракты под одним `schema_version: "1.0"`, а netlist по дизайну ждёт третью форму. Решение
оператора — аддитивный канон `1.1` (вариант A). **Блокирует E0 и E2**; внутри долга фаза P0 (написать
канон) блокирует всё остальное. Идёт вперёд очереди эпиков.

## Согласованная модель (общая для всех валидаторов)

- **Источник истины — спека ПОСТАВЩИКА** (OpenAPI 3.x / AsyncAPI 3.0);
- **ожидание потребителя — `consumed-contract`**, авто-извлечённый из его компонентных заглушек+тестов
  (проекция «вызываемые операции/каналы × шлёт/читает поля/payload»), со штампом **provenance**
  (`provider-version@capture + hash`);
- **стаб + `consumed-contract` генерятся ИЗ спеки поставщика** обязательным шагом конвейера потребителя
  (контроль дисциплины — механический гейт);
- **FORWARD** (валидаторы): `consumed-contract` ↳ схема поставщика, **schema-vs-schema поле-в-поле**
  (request/send контравар., response/receive ковар.); глубина сравнения — даром от валидации схемы;
- **REVERSE** (netlist): breaking-change во времени — диф спеки поставщика v_old→v_new (`oasdiff`) + граф.

## Карта зависимостей / независимые конвейеры

```
E-harness  скилл component-tests: стаб+consumed-contract из спеки + гейт  ── методология, параллельно
           │ (даёт вход consumed-contract для валидаторов)
           ▼
E1 pinout-openapi  (forward, sync)  ─┐  🔴 канон docs/report-format.md (владелец E1) ─┐
E0 pinout-asyncapi (forward, async) ─┼──► общий формат отчёта ────────────────────────┼─► E2 pinout-netlist (граф + provenance + reverse)
                                     └──────────────────────────────────────────────────► E3 pinout-cli (единый фронт)
```

**Порядок:** `E-harness` (даёт consumed-contract) ∥ `E1`/`E0` (валидаторы forward, фиксируют общий отчёт) →
`E2` (граф + reverse) → `E3` (фронт). E1 и E0 — симметричные независимые конвейеры на одной модели.
**Блокер:** канон `docs/report-format.md` (владелец E1) — предпосылка для отчётной части E0 и для E2.
Не просто «снесён файл»: контракт форкнулся под одной версией — разбор и план в
[`debt/report-canon-fork.md`](debt/report-canon-fork.md).

---

## E-harness — скилл `component-tests`: генерация `consumed-contract` + механический гейт

**Статус:** 📋 методология, параллельно. **Репозиторий:** [rationaldev-ai-sdlc-skills](https://github.com/codemonstersteam/rationaldev-ai-sdlc-skills/) (харнес izi).
**Цель:** у потребителя стаб **и** `consumed-contract` выводятся из master-спеки поставщика — по построению, с provenance, под контролем гейта.

- [ ] Доработать скилл `component-tests` (проектирование+реализация): из спеки поставщика + `contract-tests.yaml`
  (перечень используемых операций/каналов = объём) генерить **contract-true стаб** (HTTP-стаб / брокер-стаб) —
  happy path + сценарий на каждый обещанный контрактом режим отказа.
- [ ] Ввести артефакт **`consumed-contract`** (schema-shaped проекция читаемого/шлемого) + штамп **provenance**
  (`provider-version@capture + hash`), авто-извлекаемый из заглушек+тестов.
- [ ] **Механический гейт** `validate-consumed-contract` (харнес): стаб+consumed-contract сгенерены-из-спеки,
  provenance есть, версия совпала — иначе блок (в `wirth-tester` consequent + `@fagan` DoD). Не рекомендация — гейт.
- [ ] Изоляция фикстур по slice сохраняется (без скрытой связности).

**DoD:** стаб+`consumed-contract` генерятся из спеки поставщика, гейт блокирует рукопись/дрейф; согласовано с моделью §CONCEPT.

## E1 — pinout-openapi: forward-валидатор (sync)

**Статус:** ✅ слайс `slice-01-validate` реализован и смержен (`83e8655`), юнит-тесты зелёные, CI на PR настроен.
Остаток — 🔴 канон отчёта (долг) и перевод отчёта на `1.1`.
**Цель:** статическая сверка `consumed-contract` потребителя ↳ схема master-OpenAPI поставщика (bi-directional), симметрично async по конфигу и выходу.
**Репозиторий:** `../pinout-openapi`. Дизайн-пакет: `docs/design/slice-01-validate/` (module-tree, contracts, C4, use-case, 5 ADR, 17 тикетов).
Замороженные контракты: `api-specification/{config,report}.schema.json`.

> Алгоритм сверки **зафиксирован песочницей** [`../pinout-openapi/sandbox/`](../pinout-openapi/sandbox/) (`ALGORITHM.md` +
> доказательство `EXPERIMENT.md`, 5/5): ядро — `requires ⊆ sends` (запрос) + `reads ⊆ provides` (ответ) + типы.
> E1 переносит его на Go, а не переизобретает.

- [ ] 🔴 **P0 долга — канон отчёта `docs/report-format.md`** (`1.1`). Владелец канона — pinout-openapi.
  **БЛОКИРУЕТ E0 и E2.** Тикет-вход: `../pinout-openapi/debt/01-report-canon-doc.md`.
  Разбор и решение (вариант A) — [`debt/report-canon-fork.md`](debt/report-canon-fork.md).
- [ ] 🔴 **P1a долга — выход валидатора на `1.1`:** `report.schema.json` (+`validator`/`interaction`/`consumer.name`/`generated_at`/`errors[].subject`), шов часов, фикстуры компонентных тестов.
  Тикет-вход: `../pinout-openapi/debt/02-report-schema-1.1.md`.
- [x] Парсер OpenAPI 3.x — **`kin-openapi`** (honest reuse): parse + `$ref`-резолв + валидация (`internal/validate/provider/loader.go`, ADR-0001).
- [x] Конфиг (`api-specification/config.schema.json`: consumer, provider `spec_url|spec_path` = master, перечень операций), симметричный async.
- [x] Вход **`consumed-contract`** потребителя (из E-harness); резолв операций у поставщика (нет → `OP_NOT_IN_PROVIDER`).
- [x] **Сверка schema-vs-schema** per операция (перенос алгоритма песочницы на Go, `internal/validate/compare/`): request контравар. (поставщик принимает то, что потребитель шлёт), response ковар. (потребитель читает лишь то, что поставщик отдаёт).
- [x] Единый `Violation` + `Report` → JSON; CLI `validate <config>` + exit `0/1/2/3` (`x-exit-codes` в `report.schema.json`).

**DoD:** пара совместима/несовместима определяется корректно (вкл. type-drift/вложенность) — ✅; компонентные тесты CLI зелёные; `docs/report-format.md` написан и отчёт в формате `1.1` — 🔴 долг; README по скиллу `documentation` (с pipe-описанием) — ✅.

## E0 — pinout-asyncapi: forward-валидатор (async) + отчёт под netlist

**Статус:** 📋 **greenfield** — репозиторий сброшен под харнес (`208c6ce`, clean slate), **Go-кода нет**.
Дизайн-пакет заведён: `TASK.md`, `docs/concept.md`, песочница `sandbox/EMULATION.md` (17 сценариев),
замороженные контракты `api-specification/{config,consumed-contract,report}.schema.json` (`x-frozen: 2026-07-25`).
**Репозиторий:** `../pinout-asyncapi`. Строится сразу на согласованной модели (consumed-contract, провайдер-как-истина).

> **Поправка к прежней записи.** Здесь стояло «✅ инструмент работает», «E0 = только слой отчёта, алгоритм
> не трогаем» и ссылка на план `docs/integration-netlist.md`. Всё три устарели: инструмент снесён вместе с
> кодом при clean slate, `integration-netlist.md` не существует и **не восстанавливается — поглощён каноном
> отчёта** (решение оператора), а под-эпик `E0-model` растворён: репозиторий строится на новой модели с нуля,
> выравнивать нечего.

**P1b долга — до старта харнес-прогона E0** (сейчас правится только спека; после старта та же правка задевает написанный код и тесты). Тикет-вход: `../pinout-asyncapi/debt/01-report-canon-1.1.md`:
- [ ] 🔴 `api-specification/report.schema.json` → `1.1` зеркально sync-близнецу; `uncovered_channels` остаётся.
- [ ] 🔴 Снять `x-canon-note` → ссылка на канон; актуализировать `docs/concept.md:330`, `TASK.md:102`, `TASK.md:198`.
- [ ] 🔴 Проверить `consumer.name` в `config.schema.json`, добавить при отсутствии.

**E0 (реализация валидатора):** зависит от P0 долга (канон) и P1b.
- [ ] Прогон харнеса izi по `TASK.md`: слайсы → use case → дизайн-пакет → тикеты → реализация.
- [ ] Сверка каналов/сообщений: `consumed-contract` потребителя ↳ схема payload поставщика (send контравар. / receive ковар.), provenance, коды R1–R9.
- [ ] Отчёт в формате `1.1` (`schema_version` совпал с openapi); CLI `validate <config>` + exit `0/1/2/3`.
- [ ] README по скиллу `documentation` со ссылкой на канон отчёта.

**DoD (E0):** async-валидатор реализован на согласованной модели; отчёт в каноне `1.1`, `schema_version` совпал с sync-близнецом; exit-коды симметричны; компонентные тесты и CI зелёные.

## E2 — pinout-netlist: граф, provenance, reverse (breaking-change во времени)

**Статус:** 📋 каркас + пакет проектирования; реализация после E1/E0.
**Цель:** граф consumer↔provider с версиями/provenance; приём отчётов валидаторов; **reverse** — «кого сломает изменение поставщика».
**Репозиторий:** `../pinout-netlist`.

**P2 долга — docs-only, сейчас бесплатно (кода нет, спека не заморожена, E2 не стартовал).** Тикет-вход: `../pinout-netlist/debt/01-report-canon-1.1.md`:
- [ ] 🔴 Починить 8 битых ссылок на `report-format.md` (`README` ×2, `AGENTS`, `docs/design/intent.md`, `api-specification/openapi.yml:65`, `docs/design/backlog.md`, `docs/design/messages.md` ×2).
- [ ] 🔴 Переписать `ValidatorReport` в `api-specification/openapi.yml` под канон `1.1` — сейчас он требует `verdicts[]`/`provider{}`, которых валидаторы не печатают.
- [ ] 🔴 `docs/design/messages.md`: `VerdictRecord.At ← generated_at`, `Edge.Subject ← errors[].subject`, `Edge.Interaction ← interaction`, `Edge.Consumer ← consumer.name`.

- [ ] Модель данных: сервис, спека+версия/коммит, ребро consumer→provider с **provenance** consumed-contract, запись вердикта.
- [ ] Приём отчётов обоих валидаторов (общий формат).
- [ ] **Reverse:** диф спеки поставщика v_old→v_new (**`oasdiff`**, honest reuse) → затронутые операции → по графу живущие потребители, чей `consumed-contract` задет.
- [ ] Запрос «кто сломается, если поставщик так изменит контракт»; авто-перепроверка forward по provenance-свежести.
- [ ] (Позже) визуализация графа.

**DoD:** netlist принимает отчёты обоих валидаторов, строит граф пар; `oasdiff`-диф двух версий поставщика помечает затронутых потребителей.

## E3 — pinout-cli: единый фронт

**Статус:** 📋 позже. **Зависит от:** E1, E2.

- [ ] Делегирование `validate` нужному валидатору по типу взаимодействия (sync/async).
- [ ] Команды netlist (push отчёта, запрос графа/impact).

---

## Чек-лист этапов

- [x] Этап 0 — критический разбор идеи + **исправление модели** (bi-directional, consumed-contract) + концепт (`README.md`/`docs/CONCEPT.md`), пруф.
- [x] Этап 1 — верхнеуровневый бэклог экосистемы (этот файл) на согласованной модели.
- [x] Этап 2 — BR/TASK в компоненты: `pinout-openapi/TASK.md` ✅, `pinout-asyncapi/TASK.md` ✅; доработка скилла `component-tests` (E-harness) под модель — 📋 в работе.
- [ ] Этап 3 — запуск независимых конвейеров разработки по компонентам (харнес izi), сходятся к связному pinout. E1 прошёл первый слайс; **E0/E2 ждут закрытия долга** [`debt/report-canon-fork.md`](debt/report-canon-fork.md).

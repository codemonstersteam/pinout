# Бэклог экосистемы pinout (верхнеуровневый)

Платформенный план. Детальные пер-сервисные бэклоги — в `docs/design/<slice>/` каждого репозитория (скилл
`program-design`). Здесь — **эпики, распределённые по компонентам как независимые конвейеры**, зависимости
и порядок. Каждый эпик = отдельный харнес-прогон в своём репо; сходятся через общий формат отчёта и общую
модель контракта.

Парадигма — [`docs/CONCEPT.md`](docs/CONCEPT.md). Она **заменяет** раннюю формулировку «чистая функция
сравнения двух полных спек» на актуальную (ниже).

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
**Блокер:** канон `docs/report-format.md` (в E1) — предпосылка для отчётной части E0 и для E2; сейчас снесён (битые ссылки).

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

**Статус:** 📋 модель исправлена (см. `../pinout-openapi/TASK.md` + `docs/CONCEPT.md`); старый дизайн-пакет («две полные спеки») пересматривается.
**Цель:** статическая сверка `consumed-contract` потребителя ↳ схема master-OpenAPI поставщика (bi-directional), симметрично async по конфигу и выходу.
**Репозиторий:** `../pinout-openapi`.

> Алгоритм сверки **зафиксирован песочницей** [`../pinout-openapi/sandbox/`](../pinout-openapi/sandbox/) (`ALGORITHM.md` +
> доказательство `EXPERIMENT.md`, 5/5): ядро — `requires ⊆ sends` (запрос) + `reads ⊆ provides` (ответ) + типы.
> E1 переносит его на Go, а не переизобретает.

- [ ] 🔴 **Канон отчёта — `docs/report-format.md`** (validator / interaction / consumer / provider / `verdicts[].subject` / `errors[]` / provenance / `schema_version`). Владелец канона — pinout-openapi. **БЛОКИРУЕТ E0 и E2:** файл снесён при reset, на него висят 10 битых ссылок (async `integration-netlist.md` ×2; netlist `README`/`AGENTS`/`openapi.yml`/`docs/design` ×6). Восстановить/написать **до старта E0/E2**.
- [ ] Парсер OpenAPI 3.x — **`kin-openapi`** (honest reuse): parse + `$ref`-резолв + валидация; instance/schema-валидация (`openapi3filter`).
- [ ] Конфиг `contract-tests.yaml` (consumer, provider `spec_url|spec_path` = master, перечень операций), симметричный async.
- [ ] Вход **`consumed-contract`** потребителя (из E-harness); резолв операций у поставщика (нет → `OP_NOT_IN_PROVIDER`).
- [ ] **Сверка schema-vs-schema** per операция (перенос алгоритма песочницы на Go): request контравар. (поставщик принимает то, что потребитель шлёт), response ковар. (потребитель читает лишь то, что поставщик отдаёт). Глубина — от валидации схемы.
- [ ] Единый `ValidationError` + `Report` → канон-JSON (см. пункт «Канон отчёта» выше); CLI `validate <config>` + exit `0/1/2/3`.

**DoD:** пара совместима/несовместима определяется корректно (вкл. type-drift/вложенность); компонентные тесты CLI зелёные; `docs/report-format.md` написан, отчёт в этом формате; README по скиллу `documentation` (с pipe-описанием).

## E0 — pinout-asyncapi: forward-валидатор (async) + отчёт под netlist

**Статус:** ✅ инструмент работает; доработка в два под-скоупа (scope зафиксирован).
**Репозиторий:** `../pinout-asyncapi`. План отчёта: [`pinout-asyncapi/docs/integration-netlist.md`](../pinout-asyncapi/docs/integration-netlist.md).

> **Scope (решение):** E0 **сейчас = ТОЛЬКО слой отчёта** — алгоритм валидации каналов/сообщений НЕ трогаем
> (так и записано в `integration-netlist.md`). Модель-алайнмент (consumed-contract, README на новую модель) —
> **отдельный под-эпик `E0-model`, ПОСЛЕ канона отчёта** (E1 `docs/report-format.md`). Так дешевле: инструмент
> уже работает, а перевод модели — самостоятельный заход.

**E0 (сейчас — отчёт под netlist):** зависит от E1 `docs/report-format.md`.
- [ ] Привести `compatibility_report.json` к общему канон-формату (см. E1 «Канон отчёта») + provenance; маппер + тест.
- [ ] Команда/флаг выгрузки отчёта для netlist; не ломать текущий CLI/exit codes.
- [ ] Отметить в README async раздел «формат отчёта» со ссылкой на канон.

**DoD (E0):** async-валидатор отдаёт отчёт в канон-формате (`schema_version` совпал с openapi); текущий CLI/exit codes целы; зелёный CI; маппинг покрыт тестами.

**E0-model (позже — выравнивание модели):** отдельный заход после E0.
- [ ] Завести **`pinout-asyncapi/TASK.md`** (БТ, симметрично `openapi/TASK.md`) — вход для харнеса izi.
- [ ] Выровнять по согласованной модели: `consumed-contract` каналов/сообщений потребителя ↳ схема payload поставщика (send контравар. / receive ковар.), provenance.
- [ ] Привести README async с двух-спек модели («спека потребителя ↔ спека поставщика») на новую (consumed-contract, провайдер-как-истина).

**DoD (E0-model):** async на общей модели (`consumed-contract`); README отражает новую модель; согласовано с §CONCEPT; зелёный CI.

## E2 — pinout-netlist: граф, provenance, reverse (breaking-change во времени)

**Статус:** 📋 каркас + пакет проектирования; реализация после E1/E0.
**Цель:** граф consumer↔provider с версиями/provenance; приём отчётов валидаторов; **reverse** — «кого сломает изменение поставщика».
**Репозиторий:** `../pinout-netlist`.

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
- [ ] Этап 2 — BR/TASK в компоненты (E1/E0/E2) + доработка скилла `component-tests` (E-harness) под модель.
- [ ] Этап 3 — запуск независимых конвейеров разработки по компонентам (харнес izi), сходятся к связному pinout.

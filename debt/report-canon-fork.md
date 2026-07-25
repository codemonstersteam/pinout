# Долг: форк канона отчёта — два замороженных контракта под версией `1.0`

**Статус:** 🔴 приоритет №1, блокирует E0 и E2. **Владелец канона:** `pinout-openapi` (эпик E1).
**Решение оператора:** вариант **A** (аддитивная версия `1.1`), 2026-07-25.
**Родительский бэклог:** [`../backlog.md`](../backlog.md).

## Симптом, с которого начали

`pinout-openapi/docs/report-format.md` удалён коммитом `2e0c232` («clean slate for harness run»).
На него ссылаются другие репозитории; в бэклоге это было записано как «снесён файл, 10 битых ссылок».

## Диагноз: сломан не файл, а версионирование контракта

Пока канон отсутствовал, **оба валидатора заморозили собственный выходной контракт** — и он
структурно отличается от удалённого канона, но носит **тот же номер версии**.

| | Канон (удалён, `2e0c232^`) | Реально заморожено |
|---|---|---|
| `schema_version` | `"1.0"` | `"1.0"` ← **тот же** |
| форма ошибок | `verdicts[]{subject, compatible, errors[]}` | плоский `errors[]` |
| идентичность сторон | `validator`, `interaction`, `consumer{}`, `provider{}` | **нет** |
| время | `generated_at` | **нет** |
| происхождение | нет | `provenance{provider, provider_version, captured_hash}` |

Заморожены:

- `pinout-openapi/api-specification/report.schema.json` — `x-frozen: 2026-07-16`, **реализован**
  (`internal/validate/report/build.go`, `internal/validate/domain/domain.go`), тесты зелёные;
- `pinout-asyncapi/api-specification/report.schema.json` — `x-frozen: 2026-07-25`, `x-twin` первого,
  реализации ещё нет (репозиторий на clean slate, `208c6ce`).

### Чем это ломает E2

`pinout-netlist/api-specification/openapi.yml` объявляет `ValidatorReport` с
`required: [schema_version, validator, interaction, consumer, provider, compatible, verdicts]` —
то есть **netlist по дизайну не может распарсить то, что валидаторы реально печатают**.

- `docs/design/messages.md` берёт `VerdictRecord.At` из `report.generated_at` — поля нет;
- `Edge{Consumer, Provider, Subject, Interaction}` требует субъект и сторону взаимодействия —
  субъект спрятан в прозе `location`, `interaction` отсутствует;
- **имени потребителя в отчёте нет вообще** — netlist строит граф рёбер consumer↔provider и
  физически не знает, кто консьюмер. Это функциональный провал, а не косметика.

### Почему нельзя просто восстановить файл

`git show 2e0c232^:docs/report-format.md > docs/report-format.md` — **вредное действие**: воскресит
спеку, противоречащую двум замороженным контрактам под тем же `1.0`. Долг закрывается не откатом,
а согласованием версии вперёд.

## Уточнение инвентаря (запись в бэклоге была неточной)

- Реально битых ссылок — **8, все в `pinout-netlist`**: `README.md` ×2, `AGENTS.md` ×1,
  `docs/design/intent.md`, `api-specification/openapi.yml:65` (комментарий),
  `docs/design/backlog.md`, `docs/design/messages.md` ×2.
- 4 упоминания в `pinout-asyncapi` (`docs/concept.md:330`, `TASK.md:102`, `TASK.md:198`,
  `api-specification/report.schema.json` → `x-canon-note`) — **не битые**: это корректно оформленный
  долг («канон удалён, замораживаем фактическую форму, миграция при восстановлении»).
  Их надо не чинить, а актуализировать после закрытия долга.
- **Пропущено в исходной записи:** `pinout-asyncapi/docs/integration-netlist.md` тоже отсутствует —
  а на него ссылался бэклог как на «План отчёта» для E0. **Решение оператора: не восстанавливать,
  поглощён каноном**; ссылка снята.

## Решение: канон `1.1` — аддитивная надстройка (вариант A)

Надстройка поверх **фактической** плоской формы; `additionalProperties: false` сохраняется.

| поле | статус | источник значения |
|---|---|---|
| `schema_version` | `const "1.0"` → `const "1.1"` | — |
| `validator` | required, enum `pinout-openapi` \| `pinout-asyncapi` | константа сборки |
| `interaction` | required, enum `sync` \| `async` | константа сборки |
| `consumer.name` | required | `Config.ConsumerName` |
| `consumer.version` | optional, печатается только если известна | источника пока нет — не эмитим |
| `generated_at` | required, RFC3339 | **инжектится вызывающим** (порт часов) |
| `errors[].subject` | required | уже вычисляется как префикс `location` |

**Идентичность поставщика не дублируется:** канон объявляет `provenance{provider, provider_version,
captured_hash}` единственным её источником — она честнее старого `provider{}`, потому что несёт хеш
захвата. Это записывается в канон явно, иначе netlist будет искать `provider{}` из старой спеки.

**Почему A, а не перегруппировка в `verdicts[]` (вариант B):** `errors[].subject` даёт netlist ту же
субъектную гранулярность (одно ребро на операцию/канал) **без** реструктуризации отчёта — обе
реализации уже вычисляют ровно эту строку. B — это major на обоих валидаторах и переписывание
`FoldReport` + фикстур ради формы, которую netlist всё равно разложит обратно в плоский `Errors[]`.

## Риски, названные до старта

1. **`generated_at` — единственная неаддитивная часть.** Нужен шов часов в обоих валидаторах, иначе
   компонентные тесты станут недетерминированными.
2. **`consumer.name` в asyncapi не проверен.** В `pinout-openapi` поле есть (`Config.ConsumerName`);
   в асинке — первый шаг `change-intake`, до тикетов. Если его нет — добавка в его
   `config.schema.json`.
3. **Асимметрия зрелости репозиториев.** `pinout-openapi` — работающий код с зелёными тестами:
   правка задевает реализацию и фикстуры. `pinout-asyncapi` — greenfield с уже замороженными
   контрактами, но без кода: правка сводится к редактированию спеки **до** старта реализации.
   Окно дешевизны закроется, как только по E0 пойдёт харнес-прогон.

## План работ

> Вес и маршрут каждой фазы определяет харнес (`wirth-triage` → `route=`), здесь не задаются.
> Тикеты-входы разложены по репозиториям в их `debt/`.

### P0 — канон (`pinout-openapi`) — блокирует всё остальное

Тикет-вход: `pinout-openapi/debt/01-report-canon-doc.md`. Это спека, против которой пишутся
остальные фазы.

- [ ] Написать `pinout-openapi/docs/report-format.md` **заново от фактической формы**, не
      восстанавливая из git: схема `1.1`, таблица `x-exit-codes` (уже единая у обоих валидаторов),
      словари кодов sync/async, инвариант `compatible ⇔ errors == []`, `provenance` как источник
      идентичности поставщика, политика версионирования.
- [ ] В шапке — `superseded: docs/report-format.md@2e0c232^` с пометкой, что тот `1.0` никогда не был
      реализован. Закрывает археологию для будущих агентов.

### P1a — `pinout-openapi`: перевод выхода на `1.1`

Тикет-вход: `pinout-openapi/debt/02-report-schema-1.1.md`.

- [ ] `api-specification/report.schema.json` → `1.1` (поля по таблице выше).
- [ ] `domain.Report`, `FoldReport` — новые поля; шов часов для `generated_at`.
- [ ] `errors[].subject` выделяется из вычисления `location` (не дублируется руками).
- [ ] Обновить фикстуры компонентных тестов — обновление ожидаемо, регрессией не считать.

### P1b — `pinout-asyncapi`: правка замороженных контрактов **до старта реализации**

Тикет-вход: `pinout-asyncapi/debt/01-report-canon-1.1.md`.
Кода в репозитории нет — это редактирование дизайн-пакета greenfield-репозитория.

- [ ] `api-specification/report.schema.json` → `1.1` зеркально; `uncovered_channels` остаётся.
- [ ] Снять `x-canon-note` → заменить ссылкой на канон.
- [ ] Проверить наличие `consumer.name` в `config.schema.json`, добавить при отсутствии.
- [ ] Актуализировать `docs/concept.md:330`, `TASK.md:102`, `TASK.md:198`.

**Дедлайн: до запуска харнес-прогона E0.** После старта та же правка будет задевать уже написанный
код и тесты — цена вырастет.

### P2 — `pinout-netlist` + концепт-репо (docs-only)

Тикет-вход: `pinout-netlist/debt/01-report-canon-1.1.md`. Кода в netlist нет, спека не заморожена.

- [ ] Починить 8 битых ссылок (перечень выше).
- [ ] Переписать `ValidatorReport` в `api-specification/openapi.yml` под `1.1`. Спека не заморожена и
      E2 не стартовал — **сейчас бесплатно, после старта дорого**.
- [ ] `docs/design/messages.md`: `VerdictRecord.At ← generated_at`, `Edge.Subject ← errors[].subject`,
      `Edge.Interaction ← interaction`, `Edge.Consumer ← consumer.name`.
- [ ] `pinout/backlog.md`: снять 🔴 с E1 по завершении долга.

### P3 — снятие блокировки

E0 и E2 идут по бэклогу без изменений.

## Порядок и параллельность

`P0` → далее `P1a` ∥ `P1b` ∥ `P2` (три независимых PR-потока в трёх репозиториях).
Исполнители изолированы по worktree; git-операции на общем рабочем дереве не выполняются, пока
активен фоновый агент.

## DoD долга

- `pinout-openapi/docs/report-format.md` существует и описывает `1.1`;
- оба `report.schema.json` — `schema_version: "1.1"`, поля идентичности и `errors[].subject` есть;
- `pinout-openapi` эмитит `1.1`, юнит- и компонентные тесты зелёные;
- `pinout-netlist/api-specification/openapi.yml` парсит фактический выход обоих валидаторов;
- битых ссылок на `report-format.md` в экосистеме — ноль;
- 🔴 снят с E1 в [`../backlog.md`](../backlog.md).

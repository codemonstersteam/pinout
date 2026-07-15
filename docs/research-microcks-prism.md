# Исследование: Microcks / Prism в экосистеме pinout

> Как использовать рантайм-инструменты **Microcks** и **Stoplight Prism** внутри существующего
> проекта — для **стабов (заглушек) в компонентных тестах, поднимаемых в docker-compose**, и для
> **conformance одного сервиса** по OpenAPI/AsyncAPI. Контекст: пример компонентных тестов — сервис
> `../pinout-asyncapi`. Опорные документы: [`CONCEPT.md`](./CONCEPT.md) (§2 несущий инвариант,
> §7 дисциплина, §8 industry rationale, §10 границы), [`../backlog.md`](../backlog.md) (E4 — скилл
> `component-tests` харнеса izi).

---

## 0. TL;DR (выводы)

1. **Microcks/Prism — не альтернатива `pinout`, а слой под ним.** `pinout` — *статическая* сверка
   schema-vs-schema (без запуска сервисов, без реплея). Microcks/Prism — *рантайм*: поднимают живой
   стаб из спеки и/или валидируют живой трафик против спеки. Они закрывают ровно то, что `pinout`
   явно выносит за свои границы (`CONCEPT.md` §10: «не проверяет конформность сервиса своей спеке —
   это его компонентные тесты»). То есть они и есть реализация «его компонентных тестов».
2. **Они попадают в две разные ячейки модели pinout, не путать их:**
   - **Роль A — стаб зависимости (mock)** в компонентных тестах *потребителя*. Реализует требование
     §2: «HTTP-провайдер → HTTP-стаб; брокер-провайдер → **брокер-стаб (реальный транспорт, не in-code
     мок)**».
   - **Роль B — self-conformance** сервиса своей спеке (несущий инвариант §2; обязательный гейт §7).
3. **Для async (`pinout-asyncapi`) — Microcks практически безальтернативен.** Только Microcks из этой
   пары умеет **реальный брокерный транспорт** (Kafka/Red Panda, AMQP, MQTT, WS, NATS…) — публикует
   настоящие сообщения в настоящий брокер, что и требует §2 («не in-code мок»). Prism — **только HTTP**,
   для брокера не годится вообще.
4. **Для sync (`pinout-openapi`) — Prism легче, Microcks универсальнее.** Prism = один контейнер, ноль
   конфигурации, HTTP-mock и HTTP-conformance-proxy прямо из OpenAPI. Microcks = один инструмент на
   sync+async, но тяжелее (несколько контейнеров + MongoDB).
5. **Они НЕ отменяют `pinout`.** Contract-true стаб «из коробки» (Microcks/Prism поднимают его прямо
   из master-спеки поставщика) закрывает *транспорт* заглушки, но **не** даёт `consumed-contract` +
   `provenance` (§7) — это по-прежнему извлекает конвейер pinout. Роли комплементарны.

---

## 1. Профили инструментов (что реально умеет каждый)

### Stoplight Prism

| Свойство | Значение |
|---|---|
| Вход | OpenAPI 2/3 (+ Postman Collection) |
| Протокол | **только HTTP** |
| Режим `mock` | поднимает HTTP-сервер-заглушку из спеки; отвечает примерами/сгенерированными телами; умеет `Prefer`-заголовки для выбора примера/кода |
| Режим `proxy` (**validation proxy**) | пропускает реальный трафик к живому сервису и **валидирует запрос+ответ против OpenAPI**, репортит расхождения — это и есть conformance живого сервиса |
| Docker | `stoplight/prism:*`, один контейнер; в контейнере запускать с `-h 0.0.0.0` |
| Async | ❌ нет |

Prism — точный минималистичный инструмент «спека ↔ HTTP». Две команды: `mock` (Роль A для sync),
`proxy` (Роль B для sync).

### Microcks (CNCF)

| Свойство | Значение |
|---|---|
| Вход | OpenAPI, **AsyncAPI**, Postman, SoapUI, gRPC, GraphQL |
| Протоколы (async) | Kafka, **AMQP/RabbitMQ**, MQTT, WebSocket, NATS, Google PubSub, Amazon SQS/SNS |
| Режим mock (sync) | HTTP-заглушка из OpenAPI, как Prism |
| Режим mock (**async**) | заводит топик для версии API на **подключённом реальном брокере** и **сам публикует mock-сообщения** из examples спеки — «реальный транспорт», как требует §2 |
| Режим contract-testing (**conformance**) | подключается к топику/эндпойнту сервиса, слушает/дёргает, **валидирует реальные сообщения/ответы против схемы** его спеки → это Роль B |
| AsyncAPI 3.0 | ✅ поддержан с **1.9.0** (14.03.2024); все 8 протоколов v2 доступны и для v3; `$ref` мультифайловый, параметризованные адреса каналов, JSON/Avro |
| Docker | docker-compose: `docker-compose.yml` (полный), `docker-compose-devmode.yml` (лёгкий, с встроенным **Red Panda** брокером, без Keycloak — удобен для CI компонентных тестов) |
| Testcontainers | есть `microcks-testcontainers-java`/`-go` — встраивание throwaway-инстанса прямо в тесты |
| Async-компонент | `microcks-async-minion` — публикация mock-сообщений и тестирование async-эндпойнтов |

Microcks — «швейцарский нож»: **один инструмент покрывает и стаб-зависимости, и self-conformance,
и sync, и async**, ценой веса (Mongo + minion + сам Microcks).

---

## 2. Где это в модели pinout — карта ролей

```
                        pinout (СТАТИКА, schema-vs-schema, без запуска)
   ┌───────────────────────────────────────────────────────────────────────────┐
   │  FORWARD: consumed-contract потребителя ↳ master-спека поставщика           │
   │  REVERSE: netlist — breaking-change во времени (oasdiff)                    │
   └───────────────────────────────────────────────────────────────────────────┘
                                   ▲ опирается на инвариант
   ────────────────────────────────┼───────────────────────────────────────────
                                    │  НЕСУЩИЙ ИНВАРИАНТ (§2): каждый сервис
                                    │  конформен своей спеке — доказано его
                                    │  компонентными тестами  →  Microcks/Prism
   ┌────────────────────────────────┴───────────────────────────────────────────┐
   │  Роль A: СТАБ ЗАВИСИМОСТИ (mock)          Роль B: SELF-CONFORMANCE            │
   │  в компонентных тестах потребителя        сервиса-под-тестом своей спеке      │
   │  — стаб из master-спеки поставщика         — валидируем живой трафик/сообщения │
   │    (contract-true by construction)           против собственной спеки сервиса │
   │  sync → Prism mock / Microcks              sync → Prism proxy                 │
   │  async → Microcks (реальный брокер)        async → Microcks contract-test     │
   └─────────────────────────────────────────────────────────────────────────────┘
```

Обе роли — **рантайм и внутри одного сервиса**. `pinout` их не заменяет и не дублирует: он берёт их
*результат как предпосылку* (§2: «совместимость спек-пары ⇒ реальная совместимость сервисов» держится
только если каждая спека правдива — а правдивость доказывают именно эти компонентные тесты).

---

## 3. Матрица выбора по типу взаимодействия

| Что нужно | Тип | Рекомендация | Почему |
|---|---|---|---|
| Стаб REST-зависимости в compose | sync | **Prism `mock`** (или Microcks) | 1 контейнер, ноль конфига; стаб прямо из OpenAPI поставщика |
| Стаб брокер-зависимости (Kafka/AMQP/MQTT) | async | **Microcks** (безальтернативно) | единственный даёт реальный брокерный транспорт, как требует §2 |
| Self-conformance REST-сервиса | sync | **Prism `proxy`** | funnel живого трафика → валидация против своей OpenAPI, ноль изменений в коде |
| Self-conformance async-сервиса | async | **Microcks contract-test** | слушает топик сервиса, валидирует реальные сообщения против AsyncAPI |
| Один инструмент на всё | оба | **Microcks** | покрывает 4 клетки сразу, ценой веса |

**Практический вывод для двух репозиториев:**
- `pinout-asyncapi` (async) → **Microcks** для Роли A (брокер-стаб) и Роли B (conformance).
- `pinout-openapi` (sync) → **Prism** как лёгкий HTTP-стаб + `prism proxy` для conformance; либо тот же
  Microcks, если хочется единый стек на экосистему.

---

## 4. Топология docker-compose компонентных тестов

### 4.1. Async (целевой пример для `pinout-asyncapi`)

Сейчас в `../pinout-asyncapi` компонентных тестов с docker-compose **нет** — там Go-валидатор против
testdata-спек. «Пример компонентных тестов» здесь — это **целевой паттерн, который надо построить**
(относится к E4, скилл `component-tests` харнеса izi, а не к инструменту pinout).

Целевая топология (чёрный ящик сервиса-под-тестом + реальный брокер + Microcks-стабы зависимостей):

```yaml
# docker-compose.component-tests.yml  (эскиз, async)
services:
  broker:                      # реальный транспорт — Red Panda / Kafka
    image: redpandadata/redpanda
    # ...

  microcks:                    # брокер-стаб ЗАВИСИМОСТЕЙ сервиса + conformance-раннер
    # проще всего взять готовый docker-compose-devmode.yml Microcks
    # (встроенный Red Panda + async-minion, без Keycloak — лёгкий режим для CI)
    # импорт: master-AsyncAPI КАЖДОГО поставщика, от которого зависит сервис-под-тестом
    #  → Microcks публикует mock-сообщения поставщиков в топики брокера

  service-under-test:          # наш сервис как чёрный ящик
    build: .
    environment:
      BROKER_URL: broker:9092  # ходит в брокер, где Microcks играет поставщиков
    depends_on: [broker, microcks]

  # прогон компонентных тестов сервиса:
  #  - Роль A: зависимости-поставщики отвечают mock-сообщениями из своих master-спек
  #  - Роль B: Microcks contract-test слушает топик, куда пишет сервис,
  #            и валидирует его сообщения против ЕГО AsyncAPI (self-conformance)
```

Ключевое соответствие §2: mock-сообщения идут через **настоящий брокер**, сервис под тестом не знает,
что на другом конце Microcks — «реальный транспорт, не in-code мок».

### 4.2. Sync (для `pinout-openapi`)

```yaml
# docker-compose.component-tests.yml  (эскиз, sync)
services:
  provider-stub:               # HTTP-стаб поставщика из его master-OpenAPI
    image: stoplight/prism
    command: mock -h 0.0.0.0 /specs/provider-openapi.yml   # contract-true стаб
    volumes: [./specs:/specs]

  service-under-test:
    build: .
    environment:
      PROVIDER_URL: http://provider-stub:4010
    depends_on: [provider-stub]

  # self-conformance (Роль B) — отдельным шагом:
  #   prism proxy /specs/own-openapi.yml http://service-under-test:8080
  #   гоняем сценарии через proxy → расхождения ответов с собственной спекой = провал
```

---

## 5. Граница с pinout: что эти инструменты НЕ дают (и почему pinout остаётся)

Соблазн «Microcks поднимает стаб из master-спеки поставщика ⇒ contract-true ⇒ pinout не нужен» —
**ложный**. Разграничение:

| Аспект | Microcks/Prism (рантайм) | pinout (статика) |
|---|---|---|
| Стаб contract-true? | ✅ да, by construction (из master-спеки) | — (потребляет, не поднимает) |
| **Свежесть/provenance** (§7: версия@capture + hash) | ❌ не штампует | ✅ трекает через netlist |
| **`consumed-contract`** (какие поля потребитель реально *шлёт/читает*) | ❌ не извлекает | ✅ авто-извлечение из заглушек+тестов |
| Ковариантность response поле-в-поле, pre-merge, без запуска | ❌ (рантайм, только на прогнанных сценариях = coverage-дыры) | ✅ schema-vs-schema, вся поверхность |
| Self-conformance сервиса своей спеке | ✅ (Роль B) | ❌ (вне границ, §10) |

Итог: **Microcks/Prism закрывают транспорт стаба и self-conformance; pinout закрывает статическую
форматную сверку пары и provenance.** Именно из живого стаба, который поднимает Microcks/Prism, скилл
`component-tests` (E4) извлекает `consumed-contract` со штампом provenance — и дальше его статически
сверяет `pinout`. Они стоят в конвейере последовательно, а не вместо друг друга.

Это ровно табличка `CONCEPT.md` §8: `Mock/conformance (Microcks/Prism) → несущий инвариант`,
`Bi-directional (pinout forward) → contract потребителя ↳ OpenAPI статически`.

---

## 6. Рекомендация

**Гибрид, привязанный к протоколу:**

1. **`pinout-asyncapi` → Microcks** (devmode-compose со встроенным Red Panda):
   - Роль A: импортировать master-AsyncAPI поставщиков → брокер-стабы зависимостей сервиса;
   - Роль B: `microcks contract-test` как гейт self-conformance сервиса своей AsyncAPI.
2. **`pinout-openapi` → Prism**: `prism mock` как HTTP-стаб поставщика в compose; `prism proxy` как
   шаг self-conformance. (Если позже захочется единый стек — заменить на Microcks, паттерн тот же.)
3. **Формализовать в E4** (скилл `component-tests` харнеса izi, `../backlog.md`): конвенция «стаб
   поднимается инструментом (Microcks/Prism) из master-спеки поставщика, `consumed-contract`+provenance
   извлекаются поверх него». Это не инструмент pinout — это методология, но именно она делает стаб
   contract-true *и* pinout-совместимым.
4. **Не тащить Microcks в статический контур pinout.** Инструменты живут в docker-compose компонентных
   тестов сервиса, а `pinout` остаётся чистой функцией над спеками и `consumed-contract`.

**Следующий конкретный шаг:** собрать PoC `docker-compose.component-tests.yml` для одного канала
`pinout-asyncapi` (напр. `restGetBalanceRequest` из `contract-tests.yaml`): Red Panda + Microcks
(стаб `wallet-balance-service` из его AsyncAPI) + сервис-под-тестом; прогнать happy-path и один
режим отказа; убедиться, что Microcks contract-test ловит несоответствие сообщения схеме.

---

## 7. Открытые вопросы / риски (проверить на PoC)

1. **AsyncAPI 3.0 в Microcks — ✅ СНЯТ (проверено, с пруфом).** Концепт опирается на **AsyncAPI 3.0**
   (`CONCEPT.md` §3). Microcks поддерживает AsyncAPI **v3 начиная с релиза 1.9.0 (14 марта 2024)** —
   он был первым инструментом с поддержкой v3 для мокинга и тестирования EDA, спустя ~3 месяца после
   выхода спеки. При этом **все 8 async-протоколов** v2 доступны и для v3; поддержаны `$ref` (в т.ч.
   мультифайловые, абсолютные/относительные URL), **параметризованные адреса каналов** v3 (динамические
   топики/очереди из payload), схемы JSON и Avro + Schema Registry. Актуальная линейка Microcks — 1.15.x,
   так что для PoC достаточно `>= 1.9.0`, но брать стоит свежий релиз.
   → Минимальное требование к образу Microcks: **1.9.0+** (рекомендуется последний 1.15.x).
2. **Вес Microcks в CI — измерено.** `docker-compose-devmode.yml` = **5 контейнеров** (`app` ~312 MiB,
   `async-minion` ~243 MiB, `kafka`=Red Panda ~136 MiB при лимите `--memory 1G`, `mongo:4.4.29` ~133 MiB,
   `postman-runtime` ~38 MiB) → **~860 MiB RAM в покое**, первый pull образов **~1.5–2 GB**, холодный
   старт ~30–60 с. Без Keycloak; Red Panda и Mongo встроены. Укладывается в стандартный CI-раннер
   (4–7 GB), но **на порядок тяжелее Prism** (~40–60 MiB, 1 контейнер, старт <2 с) — это плата за
   реальный брокерный транспорт. Облегчение: (а) убрать `postman` из compose, если Postman-тесты не
   нужны; (б) `microcks-testcontainers-go` (uber-дистрибутив, **in-memory MongoDB**, throwaway-инстанс
   в тесте) — подходит, т.к. `pinout-asyncapi` на Go; (в) держать sync-стабы на Prism, а Microcks
   поднимать только под async (гибрид §6).
3. **Provenance-штамп.** Где и как фиксируется `provider-version@capture + hash` при подъёме стаба —
   решается в E4, инструменты его не дают.
4. **Единый стек vs два.** Prism (sync) + Microcks (async) = два инструмента, но каждый минимален по
   своей роли. Microcks-на-всё = один инструмент, но sync-стаб тяжелее нужного. Решение — по вкусу
   команды к операционной простоте vs единообразию.
5. **Coverage-дыры conformance.** Рантайм-conformance (Роль B) проверяет только *прогнанные* сценарии;
   полноту self-conformance держат сами компонентные тесты (happy + каждый обещанный контрактом режим
   отказа, §7 / E4), а не инструмент.

---

## Источники

- Microcks — AsyncAPI/Kafka mock: <https://microcks.io/documentation/tutorials/first-asyncapi-mock/>
- Microcks — docker-compose установка: <https://microcks.io/documentation/guides/installation/docker-compose/>
- Microcks — Kafka mocking & testing: <https://microcks.io/blog/apache-kafka-mocking-testing/>
- Microcks — AsyncAPI adoption (part 2): <https://www.asyncapi.com/blog/microcks-asyncapi-part2>
- Microcks — Testcontainers (Java): <https://github.com/microcks/microcks-testcontainers-java>
- Microcks 1.9.0 release (первая поддержка AsyncAPI v3, 14.03.2024): <https://microcks.io/blog/microcks-1.9.0-release/>
- Prism — репозиторий: <https://github.com/stoplightio/prism>
- Prism — validation proxy (conformance): <https://github.com/stoplightio/prism/blob/master/docs/guides/03-validation-proxy.md>
- Prism — Docker image: <https://hub.docker.com/r/stoplight/prism>

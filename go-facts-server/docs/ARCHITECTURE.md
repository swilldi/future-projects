# Архитектура

## Коротко

Раскладка из сокращалки: `cmd/factsd`, `cmd/factsctl`, `internal/{config,facts,postgres,server}`. `facts` это домен: `Fact`, `Store` с методами чтения по дате, дельты и записи, `Service` с валидацией. `postgres` реализует `Store` с триггером ревизий. `server` добавляет ETag-middleware для чтения по дате, авторизацию и ограничение частоты для записи. `factsctl` импортирует. На стороне Swift в `tearoff-calendar` добавляются `RemoteFactRepository`, `FactSyncService` и `GRDBFactStore`.

## Почему так

Скелет уже есть, и цель увидеть, что в нём было общим. Новые вещи локализованы: ревизия в базе триггером, потому что это инвариант данных, а не приложения; ETag в middleware, потому что это HTTP, а не домен; синхронизация на клиенте за существующим протоколом `FactRepository`, потому что календарь был спроектирован с этой границей, и это проверка той архитектуры.

## Структура папок

```
facts-server/
├── cmd/
│   ├── factsd/main.go
│   └── factsctl/main.go                   import
├── internal/
│   ├── config/
│   ├── facts/
│   │   ├── fact.go                        Fact, ошибки, валидация даты
│   │   ├── store.go                       Store interface
│   │   ├── service.go                     ByDate, Random, Delta, Create, Update, Delete
│   │   ├── etag.go                        ETag(maxRevision, count) и разбор
│   │   └── service_test.go
│   ├── memstore/                          для тестов обработчиков
│   ├── postgres/
│   │   ├── store.go
│   │   ├── migrations/
│   │   │   ├── 0001_facts.sql             таблица, последовательность, триггер ревизии
│   │   │   └── 0002_indexes.sql
│   │   ├── importer.go                    upsert с условием изменения
│   │   └── store_integration_test.go
│   └── server/
│       ├── server.go
│       ├── handlers.go
│       ├── auth.go                        Bearer, ConstantTimeCompare, лимит попыток
│       ├── etag.go                        middleware: If-None-Match → 304
│       ├── ratelimit.go
│       └── handlers_test.go
├── Dockerfile, compose.yaml, Makefile, .golangci.yml, README.md

tearoff-calendar (дополнение)
├── Packages/CalendarCore/Sources/CalendarCore/
│   ├── Facts/
│   │   ├── RemoteFactRepository.swift     FactRepository поверх GRDBFactStore
│   │   ├── GRDBFactStore.swift            факты и ревизия локально
│   │   ├── FactsAPI.swift                 GET /v1/facts?since=
│   │   ├── FactSyncService.swift          дельта, слияние, hasMore
│   │   └── BundleFallbackRepository.swift обёртка: локально, иначе бандл
│   └── Tests: FactSyncServiceTests с фикстурами дельт
```

## Компоненты

**facts.Store.** `ByDate(ctx, monthDay) ([]Fact, maxRev, error)`, `Random(ctx, monthDay)`, `Delta(ctx, since, limit) (facts, deletedIDs, maxRev, hasMore, error)`, `Create`, `Update`, `Delete` с надгробием, `CurrentRevision(ctx)`.

**facts.Service.** Валидация даты и текста, вызовы стора, `ETag` из ревизии и количества.

**postgres.Store.** Запросы с параметрами. `Delta` через `WHERE revision > $1 ORDER BY revision LIMIT $2+1` для `hasMore`. Триггер в миграции присваивает ревизию.

**postgres.Importer.** Транзакция, `INSERT ... ON CONFLICT (id) DO UPDATE SET ... WHERE facts.text IS DISTINCT FROM EXCLUDED.text OR ...`, счётчики по `xmax` или через `RETURNING (xmax = 0)`. Идемпотентен.

**server.etag middleware.** Для маршрута по дате: обработчик кладёт ETag в контекст ответа, middleware сравнивает с `If-None-Match`, отдаёт 304.

**server.auth.** Bearer, `subtle.ConstantTimeCompare`, счётчик неудач по IP в памяти с истечением.

**RemoteFactRepository (Swift).** Реализует `facts(for:)` из `GRDBFactStore`. `FactSyncService.sync()`: читает локальную ревизию, цикл `while hasMore`, слияние в транзакции, сохранение ревизии. Вызывается из приложения в `Task` при запуске, не блокирует показ.

**BundleFallbackRepository.** Если локальная база пуста, отдаёт из бандла.

## Поток данных

```
факты.json ──▶ factsctl import ──▶ upsert с условием ──▶ триггер ревизии ──▶ facts

GET /v1/facts/09-03 ──▶ etag middleware ──▶ handler ──▶ service.ByDate ──▶ store
                                              └──▶ ETag "42-3" ──▶ совпал? 304 : 200 + ETag

календарь при старте ──▶ FactSyncService.sync
                            ├──▶ GET /v1/facts?since=40
                            ├──▶ merge: upsert по id и revision, delete по deleted
                            └──▶ revision = 42 ──▶ GRDBFactStore
FactPicker ──▶ RemoteFactRepository.facts(for:) ──▶ GRDB
```

## Правила

- Ревизия только триггером. В коде Go нет присвоения `revision`.
- Удаление только надгробием.
- ETag только из ревизии и количества.
- Обработчики не знают о Postgres, тесты на `memstore`.
- Токен и лимит попыток в `auth.go`, больше нигде.
- На клиенте слияние в одной транзакции на страницу дельты, ревизия сохраняется после успешного слияния.
- `FactRepository` протокол календаря не меняется. Если пришлось, это замечание к архитектуре календаря.

## Что здесь тестируемо

`facts.Service` с `memstore`. Обработчики через `httptest`: ETag и 304, дельта с `hasMore`, 401 и 429, надгробие в дельте. `postgres.Store` интеграционно: триггер ревизии, монотонность под сотней горутин, импорт дважды. `FactSyncService` на фикстурах дельт: первая синхронизация, обновление, удаление, обрыв посреди `hasMore`.

## Где ожидать боль

- `BIGSERIAL` и параллельные транзакции: ревизии могут зафиксироваться не по порядку, и клиент с `since=41` может пропустить 40, зафиксированную позже. Решение: клиент берёт `since` как максимальную ревизию из ответа минус запас, или сервер использует `pg_advisory_lock` на запись. Выбрать блокировку: проще и запись редкая. Записать.
- `ON CONFLICT DO UPDATE WHERE`: без условия каждая строка получает новую ревизию при импорте, и дельта раздувается.
- ETag с кавычками и `W/`: разбирать аккуратно, сравнивать строки.
- Лимит попыток в памяти сбрасывается при рестарте: принято.
- На клиенте: миграция календаря с бандла на GRDB это второй сторедж в проекте, который задумывался с одним. Записать, как это ощущается.
- `hasMore` и оборвавшаяся сеть: ревизия сохраняется после каждой страницы, не в конце.

# Архитектура

## Коротко

Стандартная раскладка небольшого Go-сервиса без лишних слоёв. `cmd/shortener` собирает всё и запускает. `internal/link` это домен: типы, ошибки, интерфейс хранилища, сервис с бизнес-правилами, генератор кодов. `internal/server` это HTTP: роутер, обработчики, middleware, JSON. `internal/postgres` это реализация хранилища и миграции. `internal/memstore` для тестов. `internal/config` читает окружение.

## Почему так

В Go принято делить по пакетам с понятной ответственностью, а не по слоям на каждый чих. Интерфейс `Store` лежит рядом с тем, кто его использует, в `link`, а не рядом с реализацией: так пакет `link` ничего не знает о Postgres, и обработчики тестируются с памятью. Сервис в `link` нужен, чтобы правила валидации и генерации кода не размазались по обработчикам.

## Структура папок

```
shortener/
├── cmd/
│   └── shortener/
│       └── main.go                  конфиг, база, миграции, сервер, сигналы
├── internal/
│   ├── config/
│   │   └── config.go                Load() из окружения, ошибки на обязательных
│   ├── link/
│   │   ├── link.go                  Link, ошибки
│   │   ├── store.go                 интерфейс Store
│   │   ├── service.go               Create, Resolve, Stats, List, Delete
│   │   ├── code.go                  генератор, валидация alias и URL
│   │   └── service_test.go
│   ├── memstore/
│   │   ├── memstore.go              map под мьютексом
│   │   └── memstore_test.go
│   ├── postgres/
│   │   ├── store.go
│   │   ├── migrations/
│   │   │   └── 0001_links.sql
│   │   ├── migrate.go               goose из embed.FS
│   │   └── store_integration_test.go   //go:build integration
│   └── server/
│       ├── server.go                New(service, logger) *http.Server, роуты
│       ├── handlers.go
│       ├── middleware.go            логи, request id, recover, лимит тела
│       ├── respond.go               JSON, ошибки
│       └── handlers_test.go         httptest + memstore
├── Dockerfile
├── compose.yaml
├── .golangci.yml
├── Makefile                         test, lint, run, compose-up
└── README.md                        curl-примеры
```

## Компоненты

**config.Load.** Читает переменные, возвращает структуру или ошибку с именем недостающей переменной. Без библиотек.

**link.Service.** Принимает `Store` и генератор кодов через конструктор. `Create(ctx, url, alias)` валидирует, выбирает код или alias, повторяет при коллизии, зовёт `store.Create`. `Resolve(ctx, code)` возвращает URL и инкрементирует счётчик. Ошибки домена: `var ErrNotFound = errors.New("link not found")` и подобные.

**link.CodeGenerator.** Интерфейс с одной функцией, реализация на `crypto/rand`. В тестах подменяется детерминированной.

**memstore.Store.** `map[string]link.Link` под `sync.RWMutex`. Полностью реализует интерфейс. Используется в тестах обработчиков и сервиса.

**postgres.Store.** `*sql.DB` через `pgx/stdlib`. Каждый метод один запрос с параметрами. Конфликт уникальности переводится в `link.ErrAliasTaken` через проверку кода ошибки Postgres `23505`.

**postgres.Migrate.** `goose.SetBaseFS(embedMigrations)`, `goose.Up`. Вызывается из `main` при флаге или из сервиса `migrate` в compose.

**server.New.** Собирает `http.ServeMux` с шаблонами методов, оборачивает в middleware, возвращает `*http.Server` с таймаутами чтения и записи. Обработчики это методы структуры с сервисом и логгером.

**server middleware.** Цепочка: recover с 500, request id, лог запроса, лимит тела через `http.MaxBytesReader`.

**main.** Конфиг, база с `PingContext`, миграции, сервер в горутине, `signal.NotifyContext`, `Shutdown` с таймаутом.

## Поток данных

```
POST /api/links ──▶ middleware ──▶ handler.create
                                      ├──▶ decode JSON, лимит тела
                                      ├──▶ service.Create(ctx, url, alias)
                                      │        ├──▶ validate, generate code
                                      │        └──▶ store.Create(ctx, link)   ──▶ postgres | memstore
                                      └──▶ respond 201 JSON | errors.Is → 400/409/500

GET /{code} ──▶ handler.redirect ──▶ service.Resolve ──▶ store.Get + store.IncrementClicks ──▶ 302
```

## Правила

- `internal/link` не импортирует ни `net/http`, ни `database/sql`, ни `postgres`.
- `internal/server` не импортирует `postgres`. Только `link`.
- Все методы `Store` и `Service` принимают `context.Context` первым.
- Ошибки наружу из пакета только объявленные значения или обёрнутые через `%w`.
- Обработчики не пишут логи об ошибках сами: middleware логирует статус, обработчик кладёт ошибку в контекст ответа или возвращает через `respond.Error`.
- Тесты обработчиков только с `memstore`. Тесты `postgres` только под тегом `integration`.

## Что здесь тестируемо

`link.Service` с `memstore` и детерминированным генератором: валидация, коллизии, alias. `server` через `httptest.NewRecorder` на каждую ручку и каждую ошибку. `memstore` на конкурентность с `-race`. `postgres.Store` интеграционно: те же сценарии, что у `memstore`, плюс сто горутин на инкремент.

## Где ожидать боль

- `internal` и видимость: заглавная буква экспортирует, и это единственный механизм.
- Ошибки: `if err != nil` на каждой строке. Это норма, не бороться. Оборачивать с контекстом.
- `context`: передавать везде, даже когда кажется лишним. Отмена запроса клиентом должна доходить до базы.
- `database/sql` пул: `SetMaxOpenConns`, иначе под нагрузкой закончатся соединения.
- Goose и compose: сервис `migrate` должен завершиться до старта `app`, `depends_on` с `condition: service_completed_successfully`.
- `scratch` без сертификатов и часовых поясов: для этого сервиса не нужно, но помнить.
- Тесты с `t.Parallel()` и общий `memstore`: каждому тесту свой экземпляр.

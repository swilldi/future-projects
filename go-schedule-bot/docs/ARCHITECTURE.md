# Архитектура

## Коротко

Четыре долгоживущих компонента и домен между ними. `telegram.Poller` получает обновления и передаёт команды в `bot.Handler`. `schedule.Cache` держит события и обновляет их через `Source` по тикеру. `reminder.Scheduler` раз в 30 секунд сверяет события с подписчиками и шлёт через `Sender`. `store` это SQLite. Всё запускается из `main` под одним контекстом и `errgroup`.

## Почему так

Долгоживущие горутины это то, ради чего проект. Каждая делает одно: опрашивает, обновляет, планирует. Они не зовут друг друга напрямую: кэш отдаёт снимок событий, планировщик его читает, отправка через интерфейс. Так каждую можно остановить и протестировать отдельно, а гонки ловит `-race`, потому что общие данные только под мьютексом кэша.

## Структура папок

```
schedule-bot/
├── cmd/bot/main.go                     конфиг, база, компоненты, errgroup, сигналы
├── internal/
│   ├── config/config.go
│   ├── clock/clock.go                  interface Clock { Now() time.Time }, Real, Fake
│   ├── schedule/
│   │   ├── event.go                    Event
│   │   ├── source.go                   Source
│   │   ├── cache.go                    Cache: RWMutex, Events(), Run(ctx) с тикером и ретраями
│   │   └── cache_test.go
│   ├── sources/
│   │   ├── splatoon/                   клиент, разбор JSON, фикстуры, тесты
│   │   └── f1/
│   ├── store/
│   │   ├── store.go                    Open, миграции, Users, Sent
│   │   └── store_test.go               на временном файле
│   ├── reminder/
│   │   ├── scheduler.go                Run(ctx), tick(now), Sender interface
│   │   └── scheduler_test.go           FakeClock, FakeSender
│   ├── bot/
│   │   ├── handler.go                  команды → ответы, зависит от cache, store, clock
│   │   ├── format.go                   тексты, пояса
│   │   └── handler_test.go
│   └── telegram/
│       ├── poller.go                   long polling через библиотеку, передаёт в handler
│       └── sender.go                   реализация reminder.Sender
├── Dockerfile
├── compose.yaml                        бот, том для SQLite
├── .golangci.yml
├── Makefile
└── README.md
```

## Компоненты

**schedule.Cache.** `events []Event` под `sync.RWMutex`, `updatedAt`, `lastErr`. `Run(ctx)`: `Fetch` сразу, потом по тикеру источника; при ошибке ретраи с ростом до интервала. `Events() ([]Event, time.Time, error)` отдаёт копию. Тест: фальшивый источник, ошибка не затирает события.

**reminder.Scheduler.** Зависит от `Cache`, `store`, `Sender`, `Clock`. `Run(ctx)` тикер 30 секунд, каждый тик `tick(now)`. `tick`: подписчики из базы, события из кэша, для каждой пары проверка окна и `sent`, отправка, запись в `sent`. Пропущенные по правилу из `SPEC.md`. Тесты: фальшивые часы двигаются вручную, фальшивый отправитель записывает.

**bot.Handler.** `Handle(ctx, chatID, text) (reply string, err error)`. Разбор команды, обращение к кэшу и базе, форматирование в поясе пользователя. Без Telegram-типов. Тесты таблицей команд.

**telegram.Poller.** Обёртка над библиотекой: `Run(ctx)` получает обновления, для каждого сообщения зовёт `handler.Handle` и отвечает. Ошибки отправки логируются. Единственное место, где есть типы библиотеки, кроме `Sender`.

**telegram.Sender.** `Send(ctx, chatID, text) error`. Ошибка «бот заблокирован» распознаётся и возвращается типизированной, планировщик отключает подписку.

**store.** `database/sql` с `modernc.org/sqlite`. `busy_timeout`, `journal_mode=WAL`. Методы с контекстом. Миграции списком версионированных SQL.

**main.** Конфиг, база, источник по `SOURCE`, компоненты, `errgroup.WithContext`, `signal.NotifyContext`, ожидание, закрытие.

## Поток данных

```
Telegram ──▶ Poller.Run ──▶ handler.Handle(chatID, "/next")
                                ├──▶ cache.Events()
                                ├──▶ store.User(chatID) для пояса
                                └──▶ reply ──▶ Poller отвечает

источник ──▶ Cache.Run: тикер ──▶ source.Fetch ──▶ events под мьютексом

Scheduler.Run: тикер 30с ──▶ tick(now)
                               ├──▶ store.Subscribers()
                               ├──▶ cache.Events()
                               ├──▶ окно и sent?
                               └──▶ sender.Send ──▶ store.MarkSent
```

## Правила

- `schedule`, `reminder`, `bot` не импортируют библиотеку Telegram и `net/http`.
- Все `Run(ctx)` завершаются по отмене, проверяется тестом с отменой контекста.
- `time.Now()` только в `clock.Real`. Везде часы через интерфейс.
- Общие данные только в `Cache` под мьютексом и в базе.
- Источники тестируются на записанных JSON, сеть только в их клиентах.
- Токен из окружения, в логах маскируется.

## Что здесь тестируемо

`Cache` с фальшивым источником: обновление, ошибка, отмена. `Scheduler` с фальшивыми часами и отправителем: окно, дедупликация, пропущенные, заблокировавший пользователь. `Handler` таблицей команд с фальшивыми кэшем и базой. Источники на фикстурах. `store` на временном файле. `Poller` руками с реальным ботом.

## Где ожидать боль

- Библиотека Telegram и контексты: убедиться, что long polling останавливается по отмене, а не висит до таймаута.
- Тикеры и тесты: без интерфейса часов тест планировщика ждёт 30 секунд. С интерфейсом `tick(now)` вызывается напрямую.
- `-race` и кэш: копию среза отдавать, не сам срез.
- SQLite и параллельные записи: `busy_timeout` и одна `*sql.DB` с `SetMaxOpenConns(1)` для записи.
- Часовые пояса: `time.LoadLocation` требует базу поясов в образе; `scratch` её не содержит, нужен `tzdata` или пакет `time/tzdata`.
- splatoon3.ink может быть недоступен или закрыт: адаптер F1 как запасной, и оба через одну фикстуру-структуру.
- `errgroup`: первая ошибка отменяет всех. Ошибка источника не должна быть фатальной для группы.

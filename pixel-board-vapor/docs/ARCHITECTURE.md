# Архитектура

## Коротко

Монорепозиторий с тремя частями. `PixelShared`: пакет с сообщениями и константами. `Server`: Vapor-приложение, где `BoardState` актор держит доску и версию, `LiveHub` актор держит соединения и задержки, `EventLog` актор копит события и пишет в Fluent пачками, контроллеры тонкие. `Client`: iOS-приложение с `LiveConnection` актором над `URLSessionWebSocketTask`, `BoardModel` на главном акторе и `CanvasView`.

## Почему так

Три источника одновременности на сервере: HTTP-запросы, WebSocket-сообщения от сотен клиентов, таймеры записи. Акторы дают каждому состоянию одного владельца, и рассылка становится циклом по соединениям внутри хаба. Общий пакет это главная проверка гипотезы «Swift на сервере окупается общими моделями»: если сообщения меняются в одном месте и оба конца компилируются, гипотеза подтверждена.

## Структура папок

```
pixel-board/
├── Package.swift                          workspace-стиль: три пакета в папках
├── PixelShared/
│   ├── Package.swift
│   ├── Sources/PixelShared/
│   │   ├── Messages.swift
│   │   ├── Palette.swift
│   │   └── Board.swift                    размер, индекс (x, y) → offset, валидация
│   └── Tests/
├── Server/
│   ├── Package.swift                      Vapor, Fluent, FluentPostgresDriver, PixelShared
│   ├── Sources/App/
│   │   ├── configure.swift                база, миграции, маршруты, хаб
│   │   ├── routes.swift
│   │   ├── Controllers/
│   │   │   ├── BoardController.swift      GET /board
│   │   │   └── LiveController.swift       WS /live, декодирование, передача в хаб
│   │   ├── Core/
│   │   │   ├── BoardState.swift           актор: пиксели, версия, place, snapshot
│   │   │   ├── LiveHub.swift              актор: соединения, задержки, рассылка, пинг
│   │   │   └── EventLog.swift             актор: буфер, запись пачками, снимки
│   │   ├── Models/
│   │   │   ├── PixelEvent.swift           Fluent
│   │   │   └── BoardSnapshot.swift
│   │   ├── Migrations/
│   │   └── Config.swift                   окружение
│   ├── Sources/Run/main.swift
│   ├── Tests/AppTests/                    XCTVapor
│   ├── Dockerfile
│   └── compose.yaml
├── Client/
│   └── PixelBoard.xcodeproj
│       ├── App/
│       ├── Networking/
│       │   ├── LiveConnection.swift       актор: WebSocket, переподключение, поток сообщений
│       │   └── BoardAPI.swift             GET /board
│       ├── Model/
│       │   ├── BoardModel.swift           @Observable @MainActor: пиксели, версия, оптимизм
│       │   └── Cooldown.swift
│       ├── Features/
│       │   ├── Canvas/                    CanvasView, зум, панорама, тап → координаты
│       │   ├── Palette/
│       │   └── Settings/
│       └── PixelBoardTests/               BoardModel: оптимизм, откат, версия
└── LoadTest/
    └── Sources/loadtest/main.swift        сто клиентов на URLSessionWebSocketTask
```

## Компоненты

**Board (shared).** `struct Board { var pixels: [UInt8]; func index(x, y) }`, `validate(x, y, color)`. Используется и сервером, и клиентом.

**BoardState.** Актор. `place(x, y, color) -> Int` возвращает новую версию. `snapshot() -> (version, Data)`. При старте загружается из последнего снимка плюс события после него.

**LiveHub.** Актор. `connections: [UUID: WebSocket]`, `lastPlaced: [UUID: Date]`. `join(ws, clientID)`, `leave`, `handle(message, from)`: проверка задержки, `boardState.place`, `eventLog.append`, рассылка `placed` всем. Пинг по таймеру, удаление мёртвых. Рассылка это цикл по словарю с `send`, ошибки отправки ведут к удалению.

**EventLog.** Актор. `append(event)` в буфер. Таймер раз в секунду: `PixelEvent.create` пачкой в транзакции. Таймер раз в минуту: снимок, если версия изменилась. Ограничение буфера при недоступной базе.

**LiveController.** `app.webSocket("live")`: на подключение ждёт `hello`, зовёт `hub.join`, дальше `onText` декодирует `ClientMessage` и передаёт в хаб, `onClose` зовёт `leave`. Ничего не решает.

**BoardController.** `GET /board` через `boardState.snapshot`.

**LiveConnection (клиент).** Актор над `URLSessionWebSocketTask`. `connect()`, цикл `receive`, декодирование в `AsyncStream<ServerMessage>`. Переподключение с задержкой при обрыве, `hello` заново. `send(ClientMessage)`.

**BoardModel (клиент).** `@Observable @MainActor`. `pixels`, `version`, `pending: [(x, y, previousColor)]`. `place(x, y)`: оптимистично, `pending`, `connection.send`. На `placed`: применить, убрать из `pending` при совпадении. На `rejected`: откатить. На `snapshot`: заменить всё, `pending` очистить. `cooldownUntil`.

**CanvasView.** `Canvas` рисует пиксели из `BoardModel` с учётом масштаба и смещения, `MagnifyGesture` и `DragGesture`, тап переводит точку в координаты пикселя.

**LoadTest.** Исполняемый пакет: сто задач, каждая подключается, шлёт `place` каждые полсекунды, считает полученные `placed` и `rejected`. Печатает итог.

## Поток данных

```
тап ──▶ CanvasView ──▶ boardModel.place(x, y)
                          ├──▶ pixels[i] = color (оптимизм), pending
                          └──▶ connection.send(.place)
                                    │ WebSocket
                          LiveController ──▶ hub.handle(.place, from: id)
                                                ├──▶ задержка? ──▶ ws.send(.rejected)
                                                ├──▶ boardState.place ──▶ version
                                                ├──▶ eventLog.append
                                                └──▶ для всех: ws.send(.placed)
                                                          │
все клиенты ◀── LiveConnection ──▶ BoardModel.apply(.placed) ◀┘
```

## Правила

- Сообщения только из `PixelShared`.
- Обработчики Vapor тонкие: декодировать, передать в актор, ответить.
- Состояние только в акторах.
- Рассылка не ждёт медленных клиентов: `send` без `await` результата или с таймаутом, ошибки удаляют соединение.
- Запись в базу пачками из `EventLog`, контроллеры базу не трогают.
- Клиент никогда не доверяет своему состоянию после разрыва: снимок заново.

## Что здесь тестируемо

`PixelShared`: кодирование сообщений в обе стороны, валидация. `BoardState`: версия, снимок, загрузка из событий. `LiveHub` с фейковыми соединениями: рассылка всем, задержка, удаление мёртвых. `EventLog` с фейковым хранилищем: пачки, снимок раз в минуту, лимит буфера. Маршруты через XCTVapor: `/board`, `/healthz`, WebSocket-рукопожатие. Клиент: `BoardModel` оптимизм и откат. Нагрузка скриптом.

## Где ожидать боль

- Vapor и Swift Concurrency: часть API на `EventLoopFuture`, часть async. Держаться async-вариантов, `get()` там, где иначе нельзя.
- `WebSocket.send` в Vapor не `Sendable`-дружелюбен; хранить соединения в акторе и звать через `Task`.
- Рассылка ста клиентам синхронно в цикле блокирует актор на время отправки. Отправлять через `Task` на соединение или собирать в группу.
- Fluent миграции при старте в compose: отдельная команда `migrate` до запуска.
- Linux: Foundation другой, `JSONEncoder` есть, `Data` base64 есть. Клиентский код в общий пакет не тащить.
- `URLSessionWebSocketTask` на клиенте не умеет автоматический пинг-понг с сервером Vapor без настройки; проверить, что пинг сервера доходит и `receive` не зависает.
- Снимок 10 000 байт в base64 в JSON это 13 килобайт на каждое подключение: терпимо, бинарный протокол в стретче.

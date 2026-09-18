# Архитектура

## Коротко

Загрузкой управляет актор `ImageLoader`. Он принимает список, держит ограничитель, запускает задачи и шлёт события в `AsyncStream`. `LoaderModel` на главном акторе слушает события и держит состояние для интерфейса. `Downloader` умеет скачать один URL с прогрессом и отменой. `Thumbnailer` уменьшает. Интерфейс на SwiftUI читает `LoaderModel`.

## Почему так

Разделение на «кто решает, что грузить» (актор), «кто грузит один файл» (чистая функция с отменой) и «кто показывает» (главный актор) позволяет тестировать ограничитель и отмену без сети и без интерфейса. `AsyncStream` как единственный канал от фона к интерфейсу убирает вопрос «из какого потока я это трогаю».

## Структура папок

```
ImageLoader/
├── App/
│   ├── ImageLoaderApp.swift
│   └── AppDependencies.swift
├── Core/
│   ├── ImageItem.swift
│   ├── LoaderEvent.swift
│   ├── Downloading.swift            протокол: download(url, onProgress) async throws -> Data
│   ├── URLSessionDownloader.swift   URLSession.bytes, прогресс по Content-Length
│   ├── Thumbnailer.swift            CGImageSource, без UIKit
│   ├── ConcurrencyLimiter.swift     актор: acquire/release, лимит меняется на лету
│   └── ImageLoader.swift            актор: оркестрация, события
├── Model/
│   └── LoaderModel.swift            @Observable @MainActor, слушает события
├── Features/
│   ├── Grid/                        GridView, TileView, StatsView
│   └── Lab/                         LabView, RaceDemo, ActorDemo, ReentrancyDemo
└── ImageLoaderTests/
    ├── ConcurrencyLimiterTests.swift
    ├── ImageLoaderTests.swift       с FakeDownloader
    └── Doubles/FakeDownloader.swift
```

## Компоненты

**ConcurrencyLimiter.** Актор с `limit`, `active`, очередью продолжений. `acquire()` ждёт, пока `active < limit`, `release()` будит следующего. `setLimit(_:)` меняет на лету: если увеличили, будит столько, сколько влезло. Тестируется без сети: сто задач с фейком, который считает одновременных.

**ImageLoader.** Актор. `start(items:)`: создаёт `TaskGroup`, для каждого элемента добавляет задачу: `acquire`, событие `started`, `downloader.download` с прогрессом, `Thumbnailer.make`, событие `finished`, `release` в `defer`. Отмена всего: отмена группы. Отмена одной: словарь `id → Task`? В `TaskGroup` отдельные задачи не отменить, поэтому одна из двух схем, и выбор это урок: либо неструктурные `Task` в словаре под актором, либо `TaskGroup` плюс флаг «отменён» через `withTaskCancellationHandler`. Реализовать первую, записать, почему `TaskGroup` для этого не подошёл, потом попробовать вторую в стретче.

**Downloading и URLSessionDownloader.** `URLSession.bytes(for:)` даёт `AsyncBytes`, читаем кусками, копим `Data`, зовём `onProgress` по `expectedContentLength`. Отмена задачи прерывает `for await` сама. Конфигурация без кэша.

**Thumbnailer.** Статическая функция `make(from data: Data, maxPixel: Int) throws -> CGImage`. Синхронная, тяжёлая. Вызывается из задачи внутри актора? Нет: из задачи `TaskGroup`, которая не изолирована, и это правильно: кооперативный пул её и переварит. Записать, почему не `Task.detached`.

**LoaderModel.** `@Observable @MainActor`. `items: [ImageItem]`, статистика. В `start()` запускает `for await event in loader.events` и обновляет состояние. Тап по плитке зовёт `loader.cancel(id:)`.

**Lab.** Три демо с `nonisolated(unsafe)` там, где надо показать гонку. Изолированы в папке, в остальном проекте запрещены.

## Поток данных

```
Старт ──▶ LoaderModel.start() ──▶ ImageLoader.start(items)
                                        │
                                        ├──▶ TaskGroup: для каждого
                                        │       limiter.acquire()
                                        │       events.yield(.started)
                                        │       downloader.download ──▶ events.yield(.progress)
                                        │       Thumbnailer.make
                                        │       events.yield(.finished)
                                        │       defer limiter.release()
                                        │
LoaderModel ◀── for await events ◀──────┘
     └──▶ items[id].state = ...  (главный актор)
```

## Правила

- Единственный канал из фона в интерфейс: `events`. Прямых вызовов `LoaderModel` из задач нет.
- `ConcurrencyLimiter` не знает о картинках. Он про числа.
- `Downloading` протокол, чтобы тесты подставили фейк с управляемой задержкой.
- `defer { release() }` в каждой задаче: отмена и ошибка не должны «съедать» слот.
- Проверка отмены перед тяжёлой обработкой: не уменьшать картинку, если задача уже отменена.

## Что здесь тестируемо

`ConcurrencyLimiter`: пик не превышает лимит, увеличение лимита будит, уменьшение не рвёт активные. `ImageLoader` с `FakeDownloader`: события идут в правильном порядке, отмена всего даёт `cancelled` для всех ожидающих, отмена одной не трогает остальные, повтор упавших грузит только их. Без сети, без интерфейса, детерминированно через управляемые продолжения в фейке.

## Где ожидать боль

- Отмена одной задачи внутри `TaskGroup`. Это главный узел проекта, на него уйдёт вечер.
- Актор с `await` внутри: между `acquire` и `release` состояние актора может измениться. Реентерабельность.
- `AsyncStream` без потребителя копит события. Буфер ограничить или убедиться, что потребитель есть.
- `Sendable`: `CGImage` не `Sendable`. Передавать `Data` или помечать с обоснованием.
- Прогресс: `expectedContentLength` бывает `-1`. Тогда неопределённый индикатор.
- Тесты с фейком и задержками: без управляемых продолжений тесты станут гонками сами.

# Архитектура

## Коротко

SwiftUI с MVVM. `LocationTracker` оборачивает `CLLocationManager` и отдаёт поток точек. `WalkRecorder` актор принимает точки, фильтрует, считает через `Stats/`, копит пачку и пишет в `WalkRepository` на GRDB, публикует снимок состояния. `LiveActivityController` обновляет плашку по снимку с ограничением частоты. `HealthStore` оборачивает HealthKit. Экраны читают репозиторий через `ValueObservation` и снимок рекордера.

## Почему так

Точки приходят из фона в любой момент, и всё, что с ними происходит, должно быть быстрым и не на главном акторе: фильтр, расчёт, запись. Актор-рекордер это одно место, где есть состояние прогулки. Расчёты вынесены в чистые функции, чтобы тестировать на синтетических треках: реальную прогулку в тест не положишь. GRDB, потому что тысячи точек с запросами по интервалу это база, а `ValueObservation` даёт живые экраны без ручной подписки.

## Структура папок

```
Walks/
├── App/
│   ├── WalksApp.swift
│   ├── AppDependencies.swift
│   └── Info.plist                       фоновый режим location, тексты разрешений
├── Domain/
│   ├── Walk.swift, TrackPoint.swift
│   └── WalkSnapshot.swift               состояние для экранов и плашки
├── Stats/                               чистые функции
│   ├── PointFilter.swift                точность, возраст, скачки
│   ├── DistanceCalculator.swift
│   ├── ElevationCalculator.swift        гистерезис
│   ├── PaceCalculator.swift             окно и среднее
│   └── SeriesBuilder.swift              ряды для графиков
├── Tracking/
│   ├── LocationTracker.swift            CLLocationManager → AsyncStream
│   ├── WalkRecorder.swift               актор
│   ├── PedometerSource.swift            CMPedometer
│   └── RecoveryService.swift            незавершённые прогулки при старте
├── Persistence/
│   ├── Database.swift                   DatabasePool, миграции
│   ├── WalkRepository.swift             запись пачками, запросы, наблюдение
│   └── Records/                         WalkRecord, TrackPointRecord
├── Health/
│   ├── HealthStore.swift                авторизация, шаги, тренировка
│   └── WorkoutExporter.swift            Walk → HKWorkout с маршрутом
├── LiveActivity/
│   ├── WalkActivityAttributes.swift     общий с виджет-таргетом
│   ├── LiveActivityController.swift     старт, троттлинг, стоп
│   └── PauseWalkIntent.swift            AppIntent
├── WalkWidget/                          таргет расширения: вью плашки и острова
├── Features/
│   ├── Walk/                            WalkView, WalkViewModel, MetricsView
│   ├── History/                         HistoryView, WalkDetailView, RouteMap, PaceChart, ElevationChart
│   ├── Stats/                           WeeklyStatsView
│   └── Settings/
└── WalksTests/
    ├── Stats/                           фильтр, дистанция, высота, темп на синтетике
    ├── WalkRecorderTests.swift          с фейковым потоком точек и репозиторием в памяти
    └── RepositoryTests.swift            GRDB в памяти
```

## Компоненты

**LocationTracker.** Класс с делегатом `CLLocationManager`, конфигурация из `SPEC.md`. Точки в `AsyncStream<CLLocation>`. Статус разрешения как `@Published` или отдельный поток. `start()`, `stop()`. Единственное место с CoreLocation.

**Stats.** `PointFilter.accept(point, previous) -> Bool`. `DistanceCalculator.total(points)` и инкрементально. `ElevationCalculator` с гистерезисом. `PaceCalculator.current(points, window: 30s)` и `average`. Все над `TrackPoint`, без `CLLocation`. Тесты на синтетических треках: прямая линия 1 км, зигзаг с шумом, подъём и спуск.

**WalkRecorder.** Актор. `start()`: создаёт `Walk` в репозитории, подписывается на трекер, стартует плашку. На каждую точку: фильтр, обновление накопленных значений, добавление в пачку, снимок в `AsyncStream<WalkSnapshot>`. Таймер 5 секунд: сброс пачки в репозиторий. `pause()`, `resume()`, `stop()`: финализация, экспорт в Здоровье по настройке. Восстановление из незавершённой.

**WalkRepository.** GRDB `DatabasePool`. `insertPoints(batch)` в транзакции. `walks()`, `points(for:)`, `activeWalk()`. `ValueObservation` для истории и статистики в `AsyncStream`.

**LiveActivityController.** `Activity<WalkActivityAttributes>.request` при старте. Подписан на снимки, обновляет не чаще раза в 15 секунд, немедленно на паузу и стоп. `end` с итоговым состоянием. `PauseWalkIntent` вызывает `WalkRecorder` через общую точку доступа.

**HealthStore.** `HKHealthStore`, запрос разрешений на шаги и тренировки, `steps(from:to:)`, `saveWorkout(walk, points)` через билдеры. `isAvailable` для iPad.

**Экраны.** `WalkView` подписан на снимки рекордера, `Map` с `MapPolyline` по точкам из снимка (последние N плюс упрощение). `WalkDetailView` читает точки из репозитория, строит ряды через `SeriesBuilder`, графики через Swift Charts. `WeeklyStatsView` через наблюдение репозитория.

## Поток данных

```
GPS ──▶ LocationTracker ──▶ AsyncStream<CLLocation>
                                  │
WalkRecorder (актор): for await ──▶ PointFilter ──▶ Distance/Elevation/Pace ──▶ пачка
                                  │                                          │ каждые 5 с
                                  ├──▶ snapshots.yield(WalkSnapshot)          └──▶ repository.insertPoints
                                  │
WalkViewModel ◀── снимки ──▶ карта, показатели
LiveActivityController ◀── снимки (троттлинг 15 с) ──▶ Activity.update
стоп ──▶ recorder.stop ──▶ repository.finish ──▶ HealthStore.saveWorkout
```

## Правила

- CoreLocation только в `LocationTracker`. HealthKit только в `Health/`. ActivityKit только в `LiveActivity/`.
- `Stats/` без импортов платформы.
- Запись точек пачками в транзакции, не на главном акторе.
- Плашка обновляется по правилу частоты, проверяется тестом контроллера с фейковым `Activity`.
- Разрешения по действию.
- Снимок для экрана содержит упрощённый маршрут, не все точки: `SeriesBuilder.simplify` по расстоянию.

## Что здесь тестируемо

`Stats/` полностью на синтетике. `WalkRecorder` с фейковым потоком точек и репозиторием в памяти: пачки, пауза не пишет, стоп финализирует, восстановление. `WalkRepository` на GRDB в памяти. `LiveActivityController` с фейком: частота. Фон, батарея, Здоровье руками.

## Где ожидать боль

- Фоновая геолокация: без `UIBackgroundModes: location` и `allowsBackgroundLocationUpdates` обновления прекращаются через секунды после блокировки. И синяя полоса статуса это норма.
- Первые точки с точностью 65 метров и больше: без фильтра дистанция сразу 200 метров.
- `pausesLocationUpdatesAutomatically = true` по умолчанию: система остановит обновления при остановке и не возобновит. Выключить.
- Батарея: `Best` точность дорогая. Попробовать `NearestTenMeters` и сравнить трек и расход.
- Live Activity: бюджет обновлений ограничен, частые обновления откладываются. 15 секунд с запасом.
- `AppIntent` из плашки запускается в процессе расширения или приложения в зависимости от настройки; доступ к рекордеру через общее хранилище состояния или `openAppWhenRun`.
- HealthKit недоступен на iPad: проверка `isHealthDataAvailable`.
- Убитое системой приложение: `WalkRecorder` не переживёт, но точки в базе. Восстановление по `activeWalk`. Геолокация в фоне после убийства не возобновится без «Всегда» и мониторинга значимых изменений. Записать как ограничение.

# Архитектура

## Коротко

MVVM на SwiftUI с Observation, без DI-контейнера и без сторонних библиотек. Вся логика, не зависящая от UI, живёт в локальном Swift-пакете `CalendarCore`, который подключают и приложение, и виджет. Пакет тестируется без симулятора.

## Почему так

- Проект на один экран. Тяжёлые архитектуры тут были бы ради архитектуры. MVVM даёт ровно одно: логика экрана отделена от вёрстки и тестируется.
- Локальный пакет, а не папка: единственный честный способ поделить код с виджетом без дублирования. Плюс граница «ядро не знает про SwiftUI» проверяется компилятором, потому что в пакете нет `import SwiftUI`.
- Без DI-контейнера: зависимости передаются через `init`. Их три штуки, контейнер не окупится.

## Слои

```
┌──────────────────────────────────────────────┐
│ App (SwiftUI)             Widget (WidgetKit) │  Presentation
│  Views + ViewModels        Provider + Views  │
├──────────────────────────────────────────────┤
│ Services (в App)                             │  Platform
│  NotificationScheduler, DayChangeObserver    │
├──────────────────────────────────────────────┤
│ CalendarCore (Swift Package)                 │  Domain + Data
│  Models, DateEngine, FactRepository,         │
│  FactPicker, PageStore, SettingsStore        │
└──────────────────────────────────────────────┘
```

Зависимости направлены только вниз. `CalendarCore` не импортирует ни SwiftUI, ни UIKit, ни WidgetKit. Services знают про платформенные фреймворки (UserNotifications) и про Core, но не про Views.

## Структура папок

```
TearoffCalendar/
├── TearoffCalendar.xcodeproj
├── .swift-format
├── Packages/
│   └── CalendarCore/
│       ├── Package.swift
│       ├── Sources/CalendarCore/
│       │   ├── Models/              Fact, CalendarPage, Settings
│       │   ├── DateEngine.swift     следующий и предыдущий день, «сегодня», ключ даты
│       │   ├── Facts/               FactRepository (протокол), BundleFactRepository, FactPicker
│       │   └── Storage/             PageStore, SettingsStore (протоколы и реализации на UserDefaults)
│       └── Tests/CalendarCoreTests/
├── App/
│   ├── TearoffCalendarApp.swift     точка входа, сборка зависимостей
│   ├── Features/
│   │   ├── Calendar/
│   │   │   ├── CalendarView.swift
│   │   │   ├── CalendarViewModel.swift
│   │   │   ├── PageView.swift       одна страница, чистая вёрстка
│   │   │   └── TearGesture.swift    жест и анимация отрыва
│   │   └── Settings/
│   │       ├── SettingsView.swift
│   │       └── SettingsViewModel.swift
│   ├── Services/
│   │   ├── NotificationScheduler.swift
│   │   └── DayChangeObserver.swift
│   └── Resources/
│       ├── facts.json
│       └── Assets.xcassets
├── Widget/
│   ├── CalendarWidget.swift
│   ├── CalendarTimelineProvider.swift
│   └── CalendarWidgetView.swift
└── AppTests/                        тесты ViewModel-ов
```

## Компоненты

**DateEngine.** Чистые функции над `Calendar`: `next(after:)`, `previous(before:)`, `startOfToday()`, `monthDayKey(for:)`. Принимает `Calendar` в init, чтобы тесты подсовывали фиксированный часовой пояс.

**FactRepository.** Протокол `facts(for monthDay: String) -> [Fact]`. Реализация `BundleFactRepository` читает `facts.json` один раз и держит словарь. В тестах `InMemoryFactRepository`.

**FactPicker.** Выбирает факт для страницы. Если в `PageStore` уже есть id для этой даты, берёт его. Иначе случайный из репозитория и сохраняет. Принимает генератор случайных чисел в init, чтобы тесты были детерминированными.

**PageStore, SettingsStore.** Протоколы с реализациями поверх `UserDefaults(suiteName:)`. Единственное место, где живёт строка идентификатора App Group.

**CalendarViewModel.** `@Observable`, `@MainActor`. Состояние: текущая страница, следующая страница для подложки, флаг «не сегодня», прогресс жеста. Действия: `tearOff()`, `restore()`, `jump(to:)`, `goToday()`, `dayDidChange()`. Не знает про Views, импортирует `Observation`, а не SwiftUI.

**CalendarView.** Вёрстка и привязка жестов. Логики нет: жест вызывает действие ViewModel, анимация читает состояние.

**TearGesture.** Отдельный файл, потому что это самое сложное место вёрстки. Прогресс перетаскивания от 0 до 1, порог срыва, spring-анимация после отпускания. Прерываемость: прогресс хранится в ViewModel, а не в `@GestureState`, иначе при перебивании жеста он теряется.

**NotificationScheduler.** Запрос разрешения, постановка ежегодного `UNCalendarNotificationTrigger` с `repeats: true` на компонентах month, day, hour, minute. Удаление по идентификатору. Единственная точка входа для уведомлений.

**DayChangeObserver.** Подписка на `NSCalendarDayChanged` и на возврат из фона. Дёргает `dayDidChange()`.

**CalendarTimelineProvider.** Собирает две записи, сегодня и завтра, политика `.after(начало завтра)`. Использует тот же `FactPicker`, чтобы факт совпал с приложением.

## Поток данных

```
жест ──▶ CalendarView ──▶ CalendarViewModel.tearOff()
                                │
                                ├──▶ DateEngine.next(after:)
                                ├──▶ FactPicker.fact(for:) ──▶ FactRepository
                                │                          └──▶ PageStore (запись)
                                └──▶ state.currentPage = ...
                                          │
CalendarView ◀── Observation ◀────────────┘
```

Одно направление: события вниз, состояние вверх через Observation. Views не пишут в сторы напрямую.

## Паттерны, которые здесь есть

- **MVVM.** Ради тестируемости логики экрана.
- **Repository.** `FactRepository` прячет источник фактов. Сейчас бандл, потом сервер, вёрстка не заметит.
- **Внедрение зависимостей через init.** Без контейнера. Точка сборки одна: `TearoffCalendarApp`.
- **Протоколы для сторов.** Только ради тестов и виджета. Не плодить протоколы там, где вторая реализация не предвидится.

Чего здесь нет намеренно: Coordinator (один экран), Clean-слои с use case-ами (три действия, не окупится), Combine (Observation закрывает всё).

## Грабли

- **Даты.** `Date` это момент времени, не день. Всё, что «день», хранить как `startOfDay` через `Calendar`. Сравнивать через `calendar.isDate(_:inSameDayAs:)`.
- **App Group.** Виджет не видит `UserDefaults.standard`. Забыть suiteName значит виджет и приложение разойдутся молча.
- **Ежегодный триггер.** Повтор работает только если в `DateComponents` нет года. С годом уведомление сработает один раз.
- **Разрешение на пуши.** Запрос при запуске злит и часто получает отказ. Только по действию пользователя.
- **`@GestureState` сбрасывается** по окончании жеста, поэтому для прерываемой анимации не подходит.
- **Виджет и тяжёлая работа.** Провайдер должен отработать быстро. Парсинг JSON там допустим только пока файл маленький. При росте базы перейти на заранее индексированный формат.
- **Strict concurrency.** `UNUserNotificationCenter` и `UserDefaults` будут спорить с изоляцией. Решать через `@MainActor` на сервисах, а не через `nonisolated(unsafe)`.

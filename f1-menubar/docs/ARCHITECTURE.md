# Архитектура

## Коротко

AppKit-приложение без сториборда. `AppDelegate` это композиция: создаёт сервисы и связывает их через Combine. `ScheduleModel` владеет данными и кэшем, публикует состояние. `StatusItemController` рисует строку и меню по состоянию. `CountdownTicker` тикает раз в минуту. `NotificationPlanner` планирует по состоянию и настройкам. `Preferences` хранит настройки и публикует изменения. `SettingsWindowController` показывает SwiftUI-окно.

## Почему так

Менюбар-утилита это несколько независимых реакций на одно состояние: текст, меню, уведомления. Если состояние одно и публикуется, каждая реакция это подписка, и они не знают друг о друге. Combine здесь удобнее Observation, потому что AppKit не перерисовывает сам, и явные подписки читаются лучше. Окно настроек на SwiftUI, потому что формы там в три раза короче.

## Структура папок

```
F1Menubar/
├── App/
│   ├── AppDelegate.swift               композиция, жизненный цикл, сон
│   ├── Info.plist                      LSUIElement
│   └── F1Menubar.entitlements          сеть, sandbox
├── Model/
│   ├── ScheduleModel.swift             @Published state, обновление, кэш
│   ├── ScheduleState.swift             races, lastResults, nextSession, lastUpdated, error
│   ├── ScheduleCache.swift             JSON в Application Support
│   └── RefreshScheduler.swift          когда обновлять: старт, сутки, пробуждение
├── StatusItem/
│   ├── StatusItemController.swift      NSStatusItem, меню
│   ├── StatusTextFormatter.swift       чистая: состояние + сейчас → текст
│   ├── MenuBuilder.swift               чистая: состояние → [NSMenuItem]
│   └── CountdownTicker.swift           таймер на границу минуты
├── Notifications/
│   ├── NotificationPlanner.swift       планирование по состоянию и настройкам
│   └── NotificationIDs.swift           season-round-kind
├── Preferences/
│   ├── Preferences.swift               @Published поля поверх UserDefaults
│   ├── LaunchAtLogin.swift             SMAppService
│   ├── SettingsWindowController.swift  NSWindow + NSHostingController
│   └── SettingsView.swift              SwiftUI
└── F1MenubarTests/
    ├── StatusTextFormatterTests.swift
    ├── MenuBuilderTests.swift
    ├── RefreshSchedulerTests.swift
    └── NotificationPlannerTests.swift
```

## Компоненты

**ScheduleModel.** `@Published private(set) var state: ScheduleState`. `refresh()` через `F1Client`: расписание и результаты, запись в кэш, публикация. При старте читает кэш. Ошибки в `state.error`, данные не затираются. Сеть только здесь.

**RefreshScheduler.** Чистая логика «пора ли обновлять» по `lastUpdated` и `now`, плюс расчёт следующего момента (04:00, два часа после гонки). Таймер на следующий момент. Подписан на пробуждение через замыкание из `AppDelegate`.

**StatusTextFormatter.** `func text(for state: ScheduleState, now: Date, calendar: Calendar) -> String`. Чистая, все правила из `SPEC.md`, тесты таблицей.

**MenuBuilder.** `func items(for state: ScheduleState, now: Date) -> [MenuEntry]`, где `MenuEntry` это своя структура: заголовок, стиль, действие. `StatusItemController` превращает в `NSMenuItem`. Так меню тестируется без AppKit.

**StatusItemController.** Подписан на `state` и на тик. Обновляет `button.title`, пересобирает меню в `menuWillOpen`. Действия меню зовут замыкания, переданные из `AppDelegate`.

**CountdownTicker.** `Timer` на ближайшую границу минуты, потом каждую минуту. Публикует `now`. После пробуждения переустанавливается.

**NotificationPlanner.** Подписан на `state` и `Preferences`. При изменении: снять все свои уведомления по префиксу, поставить новые для будущих сессий по правилам. Разрешение запрашивает только когда напоминания включают. Тестируется с фальшивым центром уведомлений.

**Preferences.** `@Published` поля с записью в `UserDefaults`. `LaunchAtLogin` через `SMAppService.mainApp.register()` и `unregister()`, статус читается при открытии настроек.

**SettingsWindowController.** Одно окно, `NSHostingController(rootView: SettingsView(preferences:))`. Повторный вызов делает `makeKeyAndOrderFront`.

## Поток данных

```
старт ──▶ AppDelegate
            ├──▶ ScheduleModel.loadCache(), RefreshScheduler.start()
            ├──▶ StatusItemController ◀── model.$state, ticker.$now
            ├──▶ NotificationPlanner ◀── model.$state, preferences.$…
            └──▶ NSWorkspace.didWake ──▶ RefreshScheduler.wake(), CountdownTicker.reset()

тик ──▶ StatusTextFormatter.text(state, now) ──▶ button.title
клик ──▶ MenuBuilder.items(state, now) ──▶ NSMenu
обновление ──▶ F1Client ──▶ state ──▶ все подписчики
```

## Правила

- `StatusTextFormatter`, `MenuBuilder`, `RefreshScheduler` чистые: без AppKit, без сети, время параметром.
- `ScheduleModel` единственный, кто зовёт F1Kit и пишет кэш.
- `NotificationPlanner` единственный, кто трогает `UNUserNotificationCenter`.
- Связи между компонентами только в `AppDelegate`.
- Ни одного `Timer` вне `CountdownTicker` и `RefreshScheduler`.
- Меню собирается при открытии, не по каждому тику.

## Что здесь тестируемо

Форматтер текста таблицей на все правила. Сборщик меню: порядок, серые прошедшие, жирная ближайшая, пустые состояния. Планировщик обновлений: «пора ли» и «когда следующий». Планировщик уведомлений с фальшивым центром: количество, идентификаторы, только гонка, перепланирование без дублей. `ScheduleModel` с `RecordingHTTPClient` из F1Kit: кэш при ошибке сохраняется.

## Где ожидать боль

- `LSUIElement` и окно настроек: окно без приложения в доке не активируется само, нужен `NSApp.activate(ignoringOtherApps: true)`.
- `SMAppService` требует, чтобы приложение лежало в Applications или было подписано; в отладке статус может быть `requiresApproval`. Показать это в настройках.
- Sandbox: `com.apple.security.network.client`, иначе F1Kit молчит.
- Таймер и сон: `Timer` не срабатывает во сне, после пробуждения все расчёты «сколько осталось» устарели. Отсюда сброс тикера по `didWake`.
- Уведомления на macOS показываются только если приложение не активно или в настройках разрешено. Проверить руками.
- `menuWillOpen` вызывается на главном потоке синхронно: сборка меню должна быть быстрой, без сети.
- Часовой пояс: `Calendar.current` при старте может устареть после перелёта. Слушать `NSSystemTimeZoneDidChange`.

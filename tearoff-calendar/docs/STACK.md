# Стек

Версии на момент написания, сентябрь 2026. При старте проверить и поправить.

## Основа

| Что | Зачем здесь | Что хочу понять |
|---|---|---|
| Swift 6, strict concurrency | Чтобы компилятор ловил гонки с самого начала | Как выглядит код, который проходит проверку, и где ставить `@MainActor` |
| Актуальный Xcode, минимум iOS 26 | Observation, интерактивные виджеты и SwiftUI-шейдеры доступны | Ничего специального |
| SwiftUI | Весь интерфейс | Жесты с состоянием, прерываемые spring-анимации, `rotation3DEffect`, переходы |
| Observation | Состояние экрана | Чем `@Observable` отличается от `ObservableObject`, когда вью перерисовывается |
| Локальный Swift Package | Ядро, общее для приложения и виджета | Как устроен `Package.swift`, как гонять тесты пакета без симулятора |
| Swift Testing | Тесты ядра и ViewModel-ов | `@Test`, `#expect`, параметризованные тесты, чем лучше XCTest |
| swift-format | Единый стиль без споров | Настроить конфиг один раз |

## Платформенные фреймворки

| Что | Зачем здесь | Что хочу понять |
|---|---|---|
| Foundation `Calendar` | Вся арифметика дат | Почему нельзя прибавлять 86400, как работают `DateComponents` |
| UserNotifications | Пуш третьего сентября | Календарный триггер с повтором, запрос разрешения, обработка тапа |
| WidgetKit | Виджет с сегодняшней страницей | Таймлайн, политика обновления, лимит времени провайдера |
| App Groups | Общий `UserDefaults` для приложения и виджета | Как настроить capability и suiteName |
| `sensoryFeedback` | Отклик при отрыве | Хватит ли SwiftUI-модификатора или нужен CoreHaptics |

## Стретч

| Что | Зачем | Что хочу понять |
|---|---|---|
| SwiftUI и Metal-шейдер: `distortionEffect`, `layerEffect` | Эффект загибающейся бумаги при отрыве | Как выглядит шейдер на MSL, как SwiftUI передаёт в него параметры. Первое касание Metal перед песочницей |
| `ImageRenderer` | Шаринг страницы картинкой | Как отрендерить вью в картинку вне экрана |

## Чего в стеке нет и почему

- Сторонних библиотек. Проект маленький, всё есть в системе, и цель как раз пощупать системное.
- Combine. Observation закрывает потребности.
- CoreData и SwiftData. Хранить нужно три значения, это `UserDefaults`.

## Что почитать и посмотреть

Не всё сразу, а перед соответствующей фазой плана.

**Перед фазой 1, ядро:**
- Документация Foundation: `Calendar`, `DateComponents`.
- Документация Swift Testing, раздел Getting started.

**Перед фазой 3, жесты:**
- WWDC23 «Wind your way through advanced animations in SwiftUI».
- WWDC23 «Discover Observation in SwiftUI».
- Документация SwiftUI: «Adding interactivity with gestures».

**Перед фазой 5, пуши:**
- Документация UserNotifications: «Scheduling a notification locally from your app», раздел про `UNCalendarNotificationTrigger`.

**Перед фазой 6, виджет:**
- WWDC23 «Bring widgets to life».
- Документация WidgetKit: «Creating a widget extension», «Keeping a widget up to date».
- Human Interface Guidelines, раздел Widgets.

**Перед стретчем:**
- WWDC24 «Create custom visual effects with SwiftUI», часть про шейдеры.

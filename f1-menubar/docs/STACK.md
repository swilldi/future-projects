# Стек

Версии на момент написания, сентябрь 2026. При старте проверить и поправить.

## Основа

| Что | Зачем здесь | Что хочу понять |
|---|---|---|
| AppKit без сториборда | Приложение | `NSApplication`, `AppDelegate`, `LSUIElement` |
| `NSStatusItem`, `NSMenu` | Строка меню | Кнопка, заголовок, меню, `NSMenuDelegate`, длина текста |
| Combine | Связь компонентов | Как в полигоне AppKit |
| F1Kit | Данные | Подключение пакета по версии, что делать, когда API не хватает |
| `Timer`, `NSWorkspace` | Тик и сон | Границы минуты, `didWakeNotification`, дрейф |
| UserNotifications на macOS | Напоминания | Разрешение, `UNCalendarNotificationTrigger` или интервальный, идентификаторы, удаление |
| `SMAppService` | Автозапуск | Регистрация, статус, `requiresApproval` |
| `NSHostingController` | Настройки | SwiftUI-форма внутри AppKit-окна, передача модели |
| App Sandbox | Сеть | Entitlement |
| Swift Testing | Чистые компоненты | Таблицы для форматтера и меню |

## Чего нет и почему

- Сторонних библиотек для менюбара. `NSStatusItem` хватает.
- Виджета. Не в объёме.

## Что почитать и посмотреть

**Перед фазой 0:**
- Документация: `NSStatusItem`, `NSStatusBar`, «Configuring the user notification framework» не нужно пока.
- Документация: `LSUIElement` в Information Property List.

**Перед фазой 2:**
- Документация: `NSMenu`, `NSMenuItem`, `NSMenuDelegate`.

**Перед фазой 4:**
- Документация: `UNUserNotificationCenter` на macOS, «Asking permission to use notifications».

**Перед фазой 5:**
- Документация: `SMAppService`, WWDC22 «What's new in privacy» раздел про Login Items.
- Документация: `NSHostingController`, «Using SwiftUI with AppKit».

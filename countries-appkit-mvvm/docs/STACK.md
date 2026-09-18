# Стек

Версии на момент написания, сентябрь 2026. При старте проверить и поправить.

## Общее для всех полигонов

| Что | Зачем здесь | Что хочу понять |
|---|---|---|
| Swift 6, strict concurrency | Гонки ловит компилятор | Где ставить `@MainActor`, как передавать данные между акторами |
| Актуальный Xcode, минимум macOS 26 | Свежие API | Ничего специального |
| URLSession и Codable | Сеть | Декодирование вложенных структур API, `keyDecodingStrategy`, ошибки декодирования |
| FileManager и JSON | Кэш и избранное | Разница Caches и Documents, атомарная запись |
| Swift Testing | Тесты | `@Test`, `#expect`, параметризованные тесты, `confirmation` для асинхронных событий |
| swift-format | Стиль | Один конфиг на все полигоны |
| AppKit программно | Окно и панели | `NSSplitViewController`, `NSTableView` с diffable data source, `NSToolbar`, `NSSearchToolbarItem`, меню и responder chain |

## Специфично для этой архитектуры

| Что | Зачем здесь | Что хочу понять |
|---|---|---|
| AppKit программно | Окно и панели | `NSWindow`, `NSWindowController`, `NSSplitViewController`, `NSSplitViewItem` |
| `NSTableView` | Списки | Diffable data source на macOS, view-based ячейки, source list для сайдбара |
| `NSToolbar` и меню | Команды | `NSSearchToolbarItem`, `NSMenu` кодом, responder chain, `validateMenuItem` |
| App Sandbox | Сеть | Entitlement для исходящих соединений |
| Combine | Биндинги | Как в третьем полигоне, чтобы менялась только платформа |

## Чего нет и почему

- Сторонних библиотек. Всё есть в системе, и цель как раз посмотреть, сколько архитектуры можно построить без них.
- CoreData и SwiftData: два JSON-файла закрывают потребность, хранилища это другой полигон.
- Кэша картинок: не предмет эксперимента.

## Что почитать и посмотреть

Перед соответствующей фазой плана, не всё сразу.

**Перед фазой 0:**
- Документация: «App Sandbox», раздел про сетевые entitlement.

**Перед фазой 2:**
- Документация: `NSSplitViewController`, `NSTableView`, `NSTableViewDiffableDataSource`.
- WWDC20 «Adopt the new look of macOS», про сайдбар и тулбар.

**Перед фазой 4:**
- Документация: «Event Handling Guide», глава про responder chain. Старый, но актуальный.
- Документация: `NSMenu`, `validateMenuItem`.

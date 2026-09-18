# Стек

Версии на момент написания, сентябрь 2026. При старте проверить и поправить.

## Основа

| Что | Зачем здесь | Что хочу понять |
|---|---|---|
| CoreData | Хранилище | Модель со связями, правила удаления, контексты, `perform`, слияние, `undoManager`, версии |
| `@FetchRequest`, `NSPredicate`, `NSSortDescriptor` | Списки | Предикаты по связям, сортировка, динамическое обновление |
| `NavigationSplitView` | Три колонки | Поведение в портрете, выбор, компактный режим |
| `List` с `onMove`, `swipeActions` | Юниты | Перестановка и поле порядка |
| `NSUndoManager` | Отмена | Группировка, названия действий, связь с контекстом |
| `fileImporter`, `fileExporter` | JSON | Типы, безопасный доступ к URL |
| `ImageRenderer`, `UIGraphicsPDFRenderer` | PDF | Рендер вью в PDF-контекст, пагинация |
| Swift Testing с контейнером в памяти | Тесты хранилища | `NSInMemoryStoreType` и его отличия от SQLite |
| Swift Charts | Стретч | Разбивка очков по типам |

## Чего нет и почему

- SwiftData. Здесь намеренно CoreData: связи, правила удаления и отмена в нём взрослее.
- CloudKit. Синхронизация не в объёме.

## Что почитать и посмотреть

**Перед фазой 1:**
- Документация: «Core Data Programming Guide», разделы про модель, связи, правила удаления.
- WWDC19 «Making Apps with Core Data».
- WWDC20 «Core Data: Sundries and maxims».

**Перед фазой 3:**
- Документация: `NSUndoManager`, «Undo architecture». Раздел про CoreData и undo.

**Перед фазой 4:**
- WWDC22 «Meet the SwiftUI navigation cookbook» повторно, раздел про три колонки.
- Документация: `draggable`, `dropDestination`, `onMove`.

**Перед фазой 6:**
- Документация: `ImageRenderer`, `UIGraphicsPDFRenderer`.

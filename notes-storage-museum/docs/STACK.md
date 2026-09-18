# Стек

Версии на момент написания, сентябрь 2026. При старте проверить и поправить.

## Основа

| Что | Зачем здесь | Что хочу понять |
|---|---|---|
| Swift 6, SwiftUI, Observation | Экраны | Ничего нового |
| Акторы | Изоляция хранилищ | `@ModelActor`, свой актор для файлов, как отдавать результат на главный |
| `AsyncStream` | Сигнал изменений | Как сделать из уведомлений, из `ValueObservation`, из своего вызова |
| `ContinuousClock` | Замеры | Чем отличается от `Date`, как мерить медиану |
| Swift Testing параметризованный | Контракт | `@Test(arguments:)` с фабриками, `@Suite` |

## Хранилища

| Что | Зачем здесь | Что хочу понять |
|---|---|---|
| `FileManager`, `JSONEncoder` | Файлы | Атомарная запись, чтение папки, стоимость тысячи файлов |
| `UserDefaults` | Антипример | Что он делает при каждой записи и почему это не база |
| CoreData | Классика | Стек, контексты, `perform`, `NSFetchRequest`, предикаты, лёгкая миграция, версии модели |
| SwiftData | Новая обёртка | `@Model`, `ModelContainer`, `ModelContext`, `#Predicate`, `VersionedSchema`, где течёт CoreData |
| GRDB | SQLite напрямую | `DatabaseQueue`, записи, миграции, `ValueObservation`, FTS5, транзакции, свой SQL |

## Чего нет и почему

- Realm и подобных. Пять хватает, и все пять либо системные, либо тонкие.
- Синхронизации. Это другой проект.

## Что почитать и посмотреть

**Перед фазой 3, CoreData:**
- Документация: «Setting up a Core Data stack», «Using Core Data in the background», «Lightweight migrations».
- WWDC19 «Making Apps with Core Data».

**Перед фазой 4, SwiftData:**
- WWDC23 «Meet SwiftData», «Model your schema with SwiftData», «Dive deeper into SwiftData».
- Документация: `VersionedSchema`, `SchemaMigrationPlan`.

**Перед фазой 5, GRDB:**
- README GRDB целиком, он хороший. Разделы про миграции, `ValueObservation`, FTS.
- Документация SQLite: «FTS5 Extension», первые разделы.

**Перед фазой 6:**
- Документация: `ContinuousClock`, `Duration`.

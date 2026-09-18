# Архитектура

## Коротко

SwiftUI с `NavigationSplitView`. Слой хранения на CoreData с одним `NSPersistentContainer`, `viewContext` для чтения экранами через `@FetchRequest`, фоновые контексты для импорта. Мутации списка через `RosterService`, который работает с `viewContext` и группирует отмену. Правила очков и предупреждений в `Rules/` над структурами-снимками, без CoreData. Экспорт в `Export/`.

## Почему так

CoreData здесь предмет, поэтому его родные механизмы используются по назначению: связи вместо ручных идентификаторов, правила удаления вместо ручной чистки, отмена через контекст вместо своей. Но правила игры не должны зависеть от хранилища: снимок списка в структуры и чистые функции. Это граница, которую легко нарушить, и она в правилах ревью.

## Структура папок

```
Roster/
├── App/
│   ├── RosterApp.swift
│   └── AppDependencies.swift            контейнер, сервисы
├── Persistence/
│   ├── Roster.xcdatamodeld              версии
│   ├── PersistenceController.swift      контейнер, контексты, undo
│   ├── Entities/                        расширения NSManagedObject
│   ├── Snapshot/
│   │   ├── RosterSnapshot.swift         структуры для правил и экспорта
│   │   └── Snapshotting.swift           NSManagedObject → снимок
│   ├── RosterService.swift              мутации с отменой
│   └── CatalogImporter.swift            JSON → сущности в фоне
├── Rules/
│   ├── PointsCalculator.swift           чистые функции
│   ├── RosterValidator.swift            предупреждения
│   └── CatalogFormat.swift              Codable для JSON справочника
├── Export/
│   ├── PDFExporter.swift
│   └── RosterJSON.swift
├── Features/
│   ├── Rosters/                         RostersListView
│   ├── RosterDetail/                    RosterView, EntryRow, WarningsView
│   ├── Entry/                           EntryEditorView, WargearPicker
│   ├── DatasheetPicker/
│   └── Catalog/                         импорт, демо
└── RosterTests/
    ├── PointsCalculatorTests.swift
    ├── RosterValidatorTests.swift
    ├── CatalogImporterTests.swift       на контейнере в памяти
    ├── RosterServiceTests.swift         отмена
    └── MigrationTests.swift
```

## Компоненты

**PersistenceController.** Контейнер, `viewContext` с `automaticallyMergesChangesFromParent`, `undoManager` на `viewContext`, фабрика фоновых контекстов. Вариант в памяти для тестов и превью.

**Snapshotting.** `RosterSnapshot(from: Roster)` внутри контекста: имя, лимит, юниты с размером, выбранными опциями, замороженными очками. Всё, что нужно правилам и экспорту, и ничего из CoreData.

**PointsCalculator и RosterValidator.** Чистые функции над снимком. Тесты без хранилища.

**RosterService.** `@MainActor`. `addEntry(datasheet:to:)`, `remove`, `duplicate`, `move`, `setModels`, `toggleOption`. Каждая: `undoManager.beginUndoGrouping`, `setActionName`, изменения, `endUndoGrouping`, `save`. Отмена и повтор через `undoManager`. Проверяет ограничения опций и заморозку.

**CatalogImporter.** Декодирует JSON в `CatalogFormat`, в фоновом контексте создаёт или обновляет сущности пачками по 50 с `save`, сообщает прогресс. Режимы «заменить» и «дополнить». При замене отряды, на которые ссылаются юниты, не удаляются, а помечаются.

**Экраны.** `RostersListView` с `@FetchRequest`. `RosterView` наблюдает список, строит снимок, зовёт правила, показывает предупреждения. Перетаскивание через `onMove` в `List`. `EntryEditorView` с `WargearPicker` по группам. `DatasheetPickerView` с поиском через `@FetchRequest` и предикатом.

**PDFExporter.** `UIGraphicsPDFRenderer` с ручной вёрсткой таблицы или `ImageRenderer` SwiftUI-вью в PDF-контекст. Выбрать второе как проще, первое в стретче. Пагинация по количеству юнитов.

## Поток данных

```
тап «добавить» ──▶ DatasheetPicker ──▶ rosterService.addEntry
                                          ├──▶ undo grouping «Добавить юнит»
                                          ├──▶ RosterEntry в viewContext, save
                                          └──▶ @FetchRequest обновляет RosterView
                                                  └──▶ snapshot ──▶ PointsCalculator, RosterValidator ──▶ UI

импорт ──▶ CatalogImporter (фон) ──▶ save пачками ──▶ merge в viewContext ──▶ экраны
```

## Правила

- `NSManagedObject` только в `Persistence/` и во вью через `@FetchRequest` для отображения. В `Rules/` и `Export/` только снимки.
- Все мутации через `RosterService`. Прямое изменение сущности из вью это блокер.
- Импорт только в фоновом контексте.
- Правила удаления в модели, ручной чистки связей нет.
- Отмена только через `undoManager` контекста, своих стеков нет.
- Версия модели добавляется, старая не меняется.

## Что здесь тестируемо

`Rules/` целиком без CoreData. `CatalogImporter` на контейнере в памяти: 300 отрядов, замена, дополнение, битый JSON. `RosterService`: добавить, отменить, повторить, дублировать, переместить, на контейнере в памяти. Миграция: база старой версии в фикстурах открывается новой. Экраны и PDF руками.

## Где ожидать боль

- `@FetchRequest` и отмена: после `undo` контекст меняется, вью обновляется, но порядок юнитов по `order` надо пересчитывать в сервисе.
- Перетаскивание в `List` с `onMove` даёт индексы, а порядок хранится в поле. Пересчитать `order` всех юнитов в одной группе отмены.
- Фоновый импорт и `viewContext`: без `automaticallyMergesChangesFromParent` экраны не увидят. С ним при 300 отрядах будет всплеск обновлений: импорт пачками.
- Nullify при удалении отряда из справочника: юнит теряет ссылку, и тогда нужны замороженные имя и очки. Заморозка при создании юнита, не при удалении.
- PDF через `ImageRenderer`: длинный список не влезает в страницу. Разбить вью на страницы руками.
- Три колонки на iPad в портрете: третья колонка прячется, выбор отряда становится листом. Проверить обе ориентации.

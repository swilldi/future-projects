# Архитектура

## Коротко

MVVM на SwiftUI. Экраны знают только протокол `NotesRepository`. Пять реализаций в папке `Storage/`, каждая в своей подпапке со своими моделями. `RepositoryFactory` строит реализацию по ключу из настроек. `Benchmark` гоняет операции через тот же протокол. Один набор контрактных тестов проверяет все пять.

## Почему так

Паттерн Repository здесь не абстракция ради абстракции, а сам предмет изучения: пять реализаций одного контракта показывают, что у хранилищ общего и что разного. Контрактные тесты гарантируют, что сравнение честное: все пять ведут себя одинаково снаружи, различается только внутри и по скорости.

## Структура папок

```
NotesMuseum/
├── App/
│   ├── NotesMuseumApp.swift
│   ├── AppDependencies.swift
│   └── RepositoryFactory.swift        ключ → реализация
├── Domain/
│   ├── Note.swift
│   ├── NotesRepository.swift          протокол
│   └── NoteSeeder.swift               генератор тысячи заметок
├── Storage/
│   ├── Files/
│   │   └── FileNotesRepository.swift
│   ├── Defaults/
│   │   └── UserDefaultsNotesRepository.swift
│   ├── CoreData/
│   │   ├── NotesModel.xcdatamodeld    две версии после миграции
│   │   ├── NoteEntity+Mapping.swift
│   │   └── CoreDataNotesRepository.swift
│   ├── SwiftData/
│   │   ├── NoteModel.swift            @Model
│   │   ├── NotesSchema.swift          VersionedSchema, план миграции
│   │   └── SwiftDataNotesRepository.swift
│   └── GRDB/
│       ├── NoteRecord.swift
│       ├── Migrations.swift
│       └── GRDBNotesRepository.swift
├── Benchmark/
│   ├── BenchmarkRunner.swift          операции через протокол, ContinuousClock
│   └── BenchmarkResult.swift
├── Features/
│   ├── NotesList/                     View, ViewModel
│   ├── NoteEditor/
│   ├── Settings/
│   └── Benchmark/
└── NotesMuseumTests/
    ├── Contract/
    │   └── NotesRepositoryContractTests.swift   параметризован по пяти реализациям
    ├── Migration/
    └── Domain/
```

## Компоненты

**NotesRepository.** Протокол из `SPEC.md`. Все методы `async throws`. `changes` это `AsyncStream<Void>`: сигнал «перечитай», без данных. Проще, чем поток заметок, и одинаково реализуемо во всех пяти.

**FileNotesRepository.** Папка с файлами `<uuid>.json`. `all()` читает все файлы. Поиск в памяти. `changes` через собственный `AsyncStream.Continuation`, дёргается после записи. Актор.

**UserDefaultsNotesRepository.** Весь массив как `Data` под одним ключом. Каждая запись перекодирует всё. Актор. Существует, чтобы замер показал, почему так не делают.

**CoreDataNotesRepository.** `NSPersistentContainer`, `viewContext` для чтения, `newBackgroundContext()` для записи с `perform`. Маппинг `NoteEntity` в `Note` внутри контекста. `changes` через `NSManagedObjectContextDidSave` или `NSFetchedResultsController`. `NSManagedObject` не покидает класс.

**SwiftDataNotesRepository.** `ModelContainer` в init, `ModelContext` на запись создаётся в `@ModelActor`. `NoteModel` с `@Model`, маппинг в `Note`. Поиск через `#Predicate`. `changes` через уведомления контейнера или собственный сигнал после записи.

**GRDBNotesRepository.** `DatabaseQueue`, `NoteRecord: Codable, FetchableRecord, PersistableRecord`. Теги как JSON-столбец или отдельная таблица: выбрать отдельную, это и есть урок про нормализацию. FTS5 виртуальная таблица для поиска. `ValueObservation` в `AsyncStream`.

**RepositoryFactory.** По ключу строит реализацию. Держит кэш построенных, чтобы переключение туда-обратно не пересоздавало контейнеры.

**BenchmarkRunner.** Принимает репозиторий и `NoteSeeder`. Для каждой операции: подготовить данные, замерить `ContinuousClock`, записать. Три прогона, медиана.

**ViewModel-ы.** Как в полигоне MVVM. `NotesListViewModel` слушает `changes` и перечитывает.

## Поток данных

```
редактор ──▶ NoteEditorViewModel.save() ──▶ repository.save(note)
                                                 │
                                                 └──▶ changes.yield()
                                                          │
NotesListViewModel ◀── for await in changes ◀─────────────┘
       └──▶ repository.search(query, tag) ──▶ rows
```

## Правила

- `Note` единственная модель, которую видят экраны и ViewModel.
- Каждая реализация в своей папке, свои модели не экспортируются.
- Запись не на главном акторе. Все пять реализации либо акторы, либо используют свои механизмы фоновой работы.
- Контрактные тесты один файл, параметризованы по фабрике реализаций. Тест, специфичный для одной реализации, лежит рядом с ней и помечен.
- Замеры только через протокол. Специальных быстрых путей для отдельных хранилищ нет.

## Что здесь тестируемо

Контракт: одни тесты на пять реализаций. `NoteSeeder` и логика тегов. Миграции: для каждого хранилища тест «данные старой версии читаются новой». Замеры не тестируются, это измерение.

## Где ожидать боль

- CoreData: `NSManagedObject` в другом контексте, `objectID`, «context was never set». Правило: маппинг внутри `perform`.
- SwiftData с CloudKit не нужен, но `@Model` требует, чтобы все свойства были совместимы. Массив строк как тег работает, отношения нет.
- GRDB: FTS5 требует отдельную таблицу и триггеры синхронизации. Сначала без FTS, потом добавить и сравнить время поиска.
- Файлы: тысяча файлов читается медленно, и это правильный результат.
- Миграция SwiftData: `VersionedSchema` многословен. CoreData лёгкая миграция срабатывает сама, если поле с default.
- `changes` из пяти источников ведут себя по-разному по частоте. Схлопывать дубликаты в ViewModel.

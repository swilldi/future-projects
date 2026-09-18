# Архитектура

## Коротко

SwiftUI с MVVM там, где есть логика, как в полигоне. SwiftData с одним контейнером, настроенным на CloudKit. `ISBN` как тип-значение в `Domain/`. `MetadataProvider` протокол с двумя реализациями и объединяющим `MetadataService`. Сканер как `UIViewControllerRepresentable` только для iOS. Один таргет на iOS и macOS.

## Почему так

SwiftData с CloudKit это самый короткий путь к синхронизации, если модель с самого начала уважает ограничения CloudKit. Поэтому модель здесь спроектирована под них, а не под удобство. Метаданные из двух источников за одним протоколом, чтобы источники менялись без правки экранов. Платформенные ветки сведены к минимуму, чтобы увидеть, сколько кода на самом деле общего.

## Структура папок

```
Bookshelf/
├── App/
│   ├── BookshelfApp.swift              ModelContainer с CloudKit, окружение
│   └── AppDependencies.swift
├── Domain/
│   ├── ISBN.swift                      валидация, конвертация
│   ├── BookMetadata.swift              структура из источников
│   └── BookStatus.swift
├── Data/
│   ├── Book.swift                      @Model
│   ├── MetadataProvider.swift          протокол
│   ├── OpenLibraryProvider.swift
│   ├── GoogleBooksProvider.swift
│   ├── MetadataService.swift           цепочка, уменьшение обложки
│   └── BookRepository.swift            запросы: по ISBN, поиск, полки
├── Features/
│   ├── Shelves/                        ShelvesView, BookRow, ShelfFilter
│   ├── AddBook/
│   │   ├── AddBookView.swift           выбор способа, превью, «на полку»
│   │   ├── AddBookViewModel.swift
│   │   ├── ISBNEntryView.swift
│   │   └── ScannerView.swift           #if os(iOS), DataScannerViewController
│   ├── BookDetail/                     BookDetailView, без ViewModel
│   └── Settings/                       статус iCloud, о приложении
├── Platform/
│   └── PlatformLayout.swift            тулбар и навигация по платформе
└── BookshelfTests/
    ├── ISBNTests.swift
    ├── MetadataMappingTests.swift      на фикстурах
    ├── MetadataServiceTests.swift      цепочка с фейками
    └── BookRepositoryTests.swift       контейнер в памяти
```

## Компоненты

**ISBN.** `struct ISBN { let isbn13: String; init?(_ raw: String) }`. Чистит, проверяет суммы, конвертирует 10 в 13. Тесты на известные номера, на X, на мусор.

**Book.** `@Model` по `SPEC.md`. Все значения по умолчанию. Вычисляемые свойства для отображения.

**MetadataProvider.** `func metadata(for isbn: ISBN) async throws -> BookMetadata?`. `OpenLibraryProvider` делает до трёх запросов: книга, авторы, обложка. `GoogleBooksProvider` один запрос. Оба принимают `URLSessionConfiguration` для тестов.

**MetadataService.** Цепочка провайдеров, первый непустой результат. Уменьшает обложку до 600 пикселей по большей стороне через `CGImageSource`. Не на главном акторе.

**BookRepository.** Обёртка над `ModelContext` для запросов: `book(isbn:)`, `search(query:shelf:sort:)`. `#Predicate` для фильтра. Тесты на контейнере в памяти без CloudKit.

**AddBookViewModel.** `@Observable @MainActor`. Состояния: выбор способа, сканирование, ввод, загрузка, превью, ручная форма, дубликат. `addToShelf(status:)` создаёт `Book` в контексте.

**ScannerView.** `UIViewControllerRepresentable` над `DataScannerViewController` с `recognizedDataTypes: [.barcode(symbologies: [.ean13])]`. Координатор как делегат, отдаёт строку, ViewModel парсит в `ISBN`. Только iOS, проверка `isSupported` и `isAvailable`.

**ShelvesView.** `NavigationSplitView` с боковой панелью полок на Mac и iPad, `TabView` на iPhone через `PlatformLayout`. `@Query` с предикатом и сортировкой из состояния.

**BookDetailView.** Без ViewModel: `@Bindable var book`, правки идут в модель напрямую, SwiftData сохраняет.

## Поток данных

```
скан ──▶ ScannerView ──▶ viewModel.didScan("9785...")
                              ├──▶ ISBN(raw) ──▶ nil? подсказка
                              ├──▶ repository.book(isbn:) ──▶ есть? дубликат
                              └──▶ metadataService.metadata(isbn)  (фон)
                                        ├──▶ OpenLibrary ──▶ пусто? ──▶ GoogleBooks
                                        └──▶ BookMetadata + обложка
                                                  │
                              превью ◀────────────┘
«на полку» ──▶ Book(...) ──▶ modelContext.insert ──▶ SwiftData ──▶ CloudKit ──▶ другие устройства
```

## Правила

- Модель `Book` под CloudKit: значения по умолчанию, без `.unique`, без обязательных связей.
- `ISBN` единственный способ получить `isbn13` для хранения.
- Сеть только в провайдерах, не на главном акторе.
- `#if os` только в `ScannerView`, `PlatformLayout` и кнопке добавления.
- Дубликаты проверяются запросом до вставки.
- Обложка уменьшается до вставки, лимит размера в константе.

## Что здесь тестируемо

`ISBN` полностью. Маппинг обоих провайдеров на записанных JSON. `MetadataService` с фейками: первый пустой, второй отвечает. `BookRepository` на контейнере в памяти: поиск, полки, дубликат. Синхронизация и сканер руками на двух устройствах.

## Где ожидать боль

- CloudKit требует, чтобы контейнер был создан в консоли и схема развёрнута. Первый запуск с синхронизацией в отладке создаёт схему в Development, для TestFlight нужно Deploy to Production.
- SwiftData молчит о статусе синхронизации. Нет индикатора «синхронизировано». Проверять руками.
- `@Attribute(.externalStorage)` и CloudKit: работает, но обложки идут как ассеты, медленнее.
- `DataScannerViewController` требует `NSCameraUsageDescription` и не работает в симуляторе.
- Open Library бывает медленным и отдаёт разные форматы для книг и изданий: `/isbn/` редиректит на `/books/`. Следовать редиректу, маппить оба.
- `NavigationSplitView` на iPhone превращается в стек, но выбор полки ведёт себя иначе. Отсюда `PlatformLayout`.
- Один таргет на две платформы: на Mac нет `UIImage`, обложка через `Data` и `Image(data:)` своей обёрткой.

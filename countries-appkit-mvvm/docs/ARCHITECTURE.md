# Архитектура

## Коротко

Одно окно, `MainWindowController` владеет `NSSplitViewController` с тремя панелями: `SidebarViewController` (Все, Избранное), `CountriesListViewController`, `CountryDetailViewController`. У каждой панели ViewModel с `@Published` состоянием, как в третьем полигоне. Оконный контроллер связывает панели: выбор в сайдбаре меняет режим списка, выбор в списке задаёт страну деталям, поиск из тулбара идёт в список.

## Почему так

В iOS три экрана это три контроллера в навигации, и связь между ними это push. В macOS три панели живут одновременно в одном окне, и кто-то должен их связывать. В AppKit это традиционно оконный контроллер. Так MVVM получает координатора, которого в iOS-версии не было, и это первое, что стоит заметить. Combine оставлен тем же, чтобы менялась только платформа.

## Общая часть, не предмет эксперимента

Сервисы одинаковы во всех полигонах и живут в `Services/`. Если это не первый полигон, скопировать из предыдущего и только прогнать тесты.

- `CountriesAPI`: `func fetchAll() async throws -> [CountryDTO]`. URLSession, Codable, одна функция, параметр `fields` зашит.
- `CountryCache`: `func load() -> [CountryDTO]?`, `func save(_:)`. JSON-файл в Caches.
- `FavoritesStore`: `func load() -> Set<String>`, `func save(_:)`, `var changes: AsyncStream<Set<String>>`. JSON-файл в Documents. Поток изменений нужен, чтобы экраны узнавали об избранном, не опрашивая файл.
- `CountryDTO` и маппинг в `Country`.

Как эти сервисы подключаются к экранам, уже предмет эксперимента. Это описано ниже.

## Структура папок

```
Countries/
├── App/
│   ├── AppDelegate.swift                меню, создание MainWindowController
│   ├── AppDependencies.swift
│   └── MainWindowController.swift       окно, split view, тулбар, связь панелей
├── Models/
├── Services/
├── Views/
│   ├── CountryCellView.swift            NSTableCellView
│   ├── CountryDetailView.swift          NSView с лейблами
│   └── StateView.swift
├── Screens/
│   ├── Sidebar/
│   │   ├── SidebarViewController.swift
│   │   └── SidebarViewModel.swift       режим: все, избранное
│   ├── CountriesList/
│   │   ├── CountriesListState.swift
│   │   ├── CountriesListViewModel.swift режим + поиск + кэш + сеть
│   │   └── CountriesListViewController.swift
│   └── CountryDetail/
│       ├── CountryDetailViewModel.swift
│       └── CountryDetailViewController.swift
└── CountriesTests/
    └── ViewModels/
```

## Компоненты

**MainWindowController.** Создаёт окно и `NSSplitViewController` с тремя `NSSplitViewItem`: sidebar, content, detail. Ставит `NSToolbar` с `NSSearchToolbarItem` и кнопкой обновления. Подписывается на `sidebarViewModel.$mode` и передаёт в `listViewModel.mode`; на `listViewModel.$selectedCountry` и передаёт в `detailViewModel.country`; текст поиска из тулбара в `listViewModel.searchText`. Это координатор, и он тут единственный.

**CountriesListViewModel.** Как в третьем полигоне плюс `@Published var mode: Mode` (все или избранное) и `@Published private(set) var selectedCountry: Country?`. Экран избранного отдельным контроллером не нужен: это режим списка. Записать, что структура экранов подстроилась под платформу.

**CountriesListViewController.** `NSTableView` с `NSTableViewDiffableDataSource`, подписка на `$state`, `render`. Выбор строки пишет в `viewModel.select(row:)`. Реализует `@objc func refresh(_:)` для меню: responder chain доставит.

**SidebarViewController.** `NSTableView` в стиле source list с двумя строками. Выбор пишет в `SidebarViewModel.mode`.

**CountryDetailViewController.** Подписка на `$country` и `$isFavorite`, кнопка избранного в тулбаре или в самой панели.

**Меню.** `AppDelegate` строит главное меню кодом: пункт «Обновить» с ⌘R и `action: #selector(refresh(_:))`, `target: nil`. Кто первый в responder chain ответит, тот и обновит. Это AppKit-специфика, которой нет в iOS.

## Поток данных

```
сайдбар ──▶ SidebarViewModel.mode ──▶ MainWindowController ──▶ listViewModel.mode
                                                                      │
CountriesListViewController ◀── $state ◀── фильтрация по режиму и поиску ◀──┘

выбор строки ──▶ listViewModel.select(row:) ──▶ $selectedCountry ──▶ MainWindowController ──▶ detailViewModel.country
                                                                                                     │
CountryDetailViewController ◀── $country ◀───────────────────────────────────────────────────────────┘

⌘R ──▶ меню ──▶ responder chain ──▶ CountriesListViewController.refresh ──▶ listViewModel.refresh()
```

## Правила

- ViewModel без `import AppKit`. Foundation и Combine.
- Оконный контроллер единственный, кто связывает ViewModel панелей между собой. Панели друг о друге не знают.
- Контроллеры панелей только подписываются, рисуют и пересылают события.
- Одно `@Published state` на панель для выхода.
- Меню и тулбар строятся кодом в одном месте: `AppDelegate` для меню, `MainWindowController` для тулбара.
- Программный AppKit без сториборда.

## Что здесь тестируемо

Три ViewModel, как в третьем полигоне. Плюс `MainWindowController` в части связывания: можно проверить без окна, если вынести связывание в отдельный объект. Решить, делать ли это, и записать.

## Где ожидать боль

- `NSTableView` многословнее `UITableView`: колонки, идентификаторы ячеек, `makeView(withIdentifier:)`.
- `NSSplitViewController` схлопывает панели по-своему, минимальные ширины надо задавать явно.
- Responder chain: ⌘R не работает, пока не разберёшься, кто first responder.
- Без entitlement «Outgoing Connections» в App Sandbox сеть молча падает. Первые полчаса уйдут на это, если не знать.
- Нет `AsyncImage` и нет `UIImage(data:)`, есть `NSImage`, и в ячейке его тоже надо грузить через `Task` с отменой.
- Окно можно закрыть, и приложение продолжит жить. Решить, что делать при закрытии.

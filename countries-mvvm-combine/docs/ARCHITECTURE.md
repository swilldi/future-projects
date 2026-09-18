# Архитектура

## Коротко

Экран это контроллер плюс ViewModel. ViewModel публикует `@Published private(set) var state` и принимает входы: методы и `@Published var searchText`. Контроллер подписывается на `$state` и превращает его в UI. ViewModel не знает о контроллере, не импортирует UIKit, не командует.

## Почему так

В MVP презентер помнит, что показать и что спрятать. В MVVM он просто описывает состояние, а view сама выводит разницу. Это снимает класс ошибок «забыл спрятать загрузку» и убирает протокол view. Плата: Combine, который надо освоить, и дисциплина «одно состояние, а не пять флагов».

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
│   ├── AppDelegate.swift
│   ├── SceneDelegate.swift
│   └── AppDependencies.swift
├── Models/
├── Services/
├── Views/                               CountryCell, CountryDetailView, StateView
├── Screens/
│   ├── CountriesList/
│   │   ├── CountriesListState.swift     структура состояния и CountryRow
│   │   ├── CountriesListViewModel.swift
│   │   └── CountriesListViewController.swift
│   ├── CountryDetail/
│   │   ├── CountryDetailViewModel.swift
│   │   ├── CountryDetailViewController.swift
│   │   └── CountryDetailView.swift      бонус: SwiftUI-версия
│   └── Favorites/
│       ├── FavoritesViewModel.swift
│       └── FavoritesViewController.swift
└── CountriesTests/
    └── ViewModels/
```

## Компоненты

**CountriesListState.** Структура: `rows: [CountryRow]`, `isLoading`, `errorMessage: String?`, `isOffline`, `isSearchEmpty`. Одно значение описывает весь экран. Не enum с кейсами, потому что офлайн и список сосуществуют.

**CountriesListViewModel.** `@Published private(set) var state`, `@Published var searchText`. В init подписывается на `$searchText.debounce.removeDuplicates` и пересчитывает `rows`. Методы `load()`, `refresh()`, `retry()`. Слушает `favoritesStore.changes` через `Task` и обновляет состояние. Отдаёт `country(at:)` для навигации. Не импортирует UIKit.

**CountriesListViewController.** В `viewDidLoad` подписывается: `viewModel.$state.receive(on: DispatchQueue.main).sink { [weak self] in self?.render($0) }`. `render` применяет снапшот и показывает `StateView` по состоянию. Пересылает ввод: `searchController` пишет в `viewModel.searchText`, `refreshControl` вызывает `refresh()`. По тапу берёт `country(at:)` и пушит детали.

**CountryDetailViewModel.** `@Published private(set) var isFavorite`, `toggleFavorite()`, слушает `changes`. Это `ObservableObject`, потому что бонус подключит его к SwiftUI.

**Навигация.** Контроллер пушит следующий контроллер, создавая ViewModel через `AppDependencies`. Координатора нет намеренно: записать, где он был бы уместен.

## Поток данных

```
ввод ──▶ ViewController ──▶ viewModel.searchText = text
                                    │
                                    ├──▶ debounce, фильтрация
                                    └──▶ state = State(rows: ...)
                                              │
ViewController ◀── $state.sink ◀──────────────┘
      └──▶ render(state): snapshot, StateView

ViewModel публикует, контроллер наблюдает. Никто никому не командует.
```

## Правила

- ViewModel не импортирует UIKit. Импортирует Foundation и Combine.
- Контроллер только подписывается и пересылает. Логики нет. В `sink` только вызов `render`.
- Одно `@Published state` на экран для выхода. Входы отдельно.
- Каждая подписка хранится в `cancellables`. Каждое замыкание с `[weak self]`.
- `receive(on: DispatchQueue.main)` в контроллере, не в ViewModel. ViewModel не знает о потоках UI.
- Никаких протоколов view. Их заменило состояние.

## Что здесь тестируемо

ViewModel целиком: подать входы, собрать значения `$state`, сравнить. Без шпионов: проверяется состояние, а не последовательность вызовов. Сравнить с MVP по хрупкости.

## Где ожидать боль

- `[weak self]` и `cancellables` везде. Забыть одно из двух значит утечка или молчание.
- Мост `AsyncStream` в Combine: приходится держать `Task` рядом с `cancellables`, два механизма отмены.
- Соблазн завести пять `@Published` вместо одного состояния. Тогда экран мигает промежуточными состояниями.
- Тесты Combine с асинхронностью: `debounce` в тестах требует управляемого планировщика или ожидания.

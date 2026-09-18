# Архитектура

## Коротко

Четыре фичи: `AppFeature` с вкладками, `CountriesListFeature`, `CountryDetailFeature`, `FavoritesFeature`. Сервисы обёрнуты в клиенты-зависимости: `CountriesClient`, `CountryCacheClient`, `FavoritesClient`. Список ведёт на детали через `@Presents` или стек. Избранное общее: попробовать `@Shared`, если не зайдёт, держать в родителе.

## Почему так

Redux руками показал, что архитектура хорошая, а обвязки много. TCA обещает убрать обвязку и добавить то, что руками писать долго: отмену эффектов, тестирование эффектов, зависимости с тестовыми значениями, навигацию как состояние. Это надо проверить на той же задаче, чтобы разница была видна на строках, а не на ощущениях. Версию закрепить: библиотека меняется быстро.

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
│   └── CountriesApp.swift               Store(initialState:) { AppFeature() }
├── Models/
├── Services/                            сырые реализации, как везде
├── Dependencies/
│   ├── CountriesClient.swift            struct с замыканиями, DependencyKey, live/test
│   ├── CountryCacheClient.swift
│   └── FavoritesClient.swift
├── Features/
│   ├── AppFeature.swift                 вкладки, Scope на дочерние
│   ├── CountriesList/
│   │   ├── CountriesListFeature.swift
│   │   └── CountriesListView.swift
│   ├── CountryDetail/
│   │   ├── CountryDetailFeature.swift
│   │   └── CountryDetailView.swift
│   ├── Favorites/
│   │   ├── FavoritesFeature.swift
│   │   └── FavoritesView.swift
│   └── Shared/                          CountryRowView, StateView
└── CountriesTests/
    └── Features/                        TestStore на каждую фичу
```

## Компоненты

**Клиенты-зависимости.** `struct CountriesClient { var fetchAll: @Sendable () async throws -> [Country] }` с `DependencyKey`: `liveValue` оборачивает `CountriesAPI`, `testValue` падает с `unimplemented`, чтобы тест, забывший подменить, упал явно. Маппинг DTO внутри `liveValue`.

**CountriesListFeature.** `@ObservableState struct State { var countries, searchText, isLoading, errorMessage, isOffline; @Presents var detail: CountryDetailFeature.State? }`. `enum Action { case task, searchChanged(String), refresh, countriesLoaded(Result<[Country], Error>, fromCache: Bool), rowTapped(Country), detail(PresentationAction<...>) }`. В `body`: `Reduce` с `.run` для кэша и сети, `.ifLet(\.$detail, action: \.detail)`. Вычисляемое `filteredCountries` в `State`.

**CountryDetailFeature.** `State` с `country` и `isFavorite`. `toggleTapped` через `FavoritesClient`.

**Избранное.** Вариант А: `@Shared(.fileStorage(url)) var favorites: Set<String>` в состоянии каждой фичи, библиотека сама синхронизирует и сохраняет. Вариант Б: коды в `AppFeature.State`, дочерние читают через родителя. Попробовать А, записать впечатление, при проблемах откатиться на Б.

**AppFeature.** `State` с двумя дочерними и выбранной вкладкой. `Scope` на каждую. Здесь видно композицию.

**Вью.** `@Bindable var store: StoreOf<CountriesListFeature>`. `.searchable(text: $store.searchText.sending(\.searchChanged))`. `.task { await store.send(.task).finish() }`. Навигация через `.sheet(item:)` или `.navigationDestination(item:)` на `$store.scope(state: \.detail, action: \.detail)`.

## Поток данных

```
ввод ──▶ View ──▶ store.send(.searchChanged(text))
                        │
                        └──▶ Reduce: state.searchText = text; return .none
                                  │
View ◀── @ObservableState ◀── filteredCountries ◀──┘

старт ──▶ send(.task)
             └──▶ Reduce: return .run { send in
                        await send(.countriesLoaded(cache, fromCache: true))
                        await send(.countriesLoaded(Result { try await client.fetchAll() }, fromCache: false))
                  }.cancellable(id: CancelID.load)
```

## Правила

- Вся логика в редьюсерах. Вью только `send` и чтение состояния.
- Сервисы только через клиенты-зависимости с `@Dependency`. Прямой вызов `CountriesAPI` из редьюсера это нарушение.
- У каждого клиента есть `testValue` с `unimplemented`.
- Эффекты, которые могут пережить экран, помечены `.cancellable` и отменяются в `task`-действии или при уходе.
- Тесты через `TestStore`, исчерпывающие. Неисчерпывающий режим только с записью причины в `NOTES.md`.
- Версия библиотеки закреплена точно. Обновлять только между фазами и с записью.

## Что здесь тестируемо

Каждая фича через `TestStore`: отправить действие, описать каждое изменение состояния, каждое полученное действие от эффекта. Зависимости подменяются через `withDependencies`. Сравнить с тестами Redux руками: что стало строже, что длиннее.

## Где ожидать боль

- Первый день уходит на чтение документации, а не на код. Это честная цена.
- Макросы и время компиляции. Засечь чистую сборку.
- Исчерпывающие тесты требуют описать всё. Пропущенное изменение состояния это упавший тест. Это и хорошо, и утомительно.
- Навигация как состояние непривычна после `NavigationStack` с путём во вью.
- Обновление библиотеки может сломать код. Поэтому версия закреплена.
- `@Shared` это своя магия. Если непонятно, как оно синхронизирует, лучше вариант Б.

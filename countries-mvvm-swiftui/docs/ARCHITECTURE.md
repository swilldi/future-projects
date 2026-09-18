# Архитектура

## Коротко

Экран это SwiftUI-вью плюс `@Observable` ViewModel там, где есть логика. Список и избранное с ViewModel, детали без: там нечего тестировать, кроме кнопки. Избранное живёт в одном `@Observable` объекте в окружении, который читают все экраны. ViewModel не импортирует SwiftUI.

## Почему так

SwiftUI перерисовывает вью по изменению наблюдаемых свойств, поэтому весь Combine-слой из прошлого полигона исчезает. Остаётся вопрос, что ViewModel даёт кроме тестируемости, и ответ: ничего, поэтому он нужен только там, где логика есть. Общий объект избранного в окружении это способ SwiftUI решить задачу «экран А изменил, экран Б увидел», и его надо попробовать вместо трёх подписок на `changes`.

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
│   ├── CountriesApp.swift               точка входа, TabView, окружение
│   └── AppDependencies.swift
├── Models/
├── Services/
├── Features/
│   ├── CountriesList/
│   │   ├── CountriesListView.swift
│   │   └── CountriesListViewModel.swift
│   ├── CountryDetail/
│   │   └── CountryDetailView.swift      без ViewModel, намеренно
│   ├── Favorites/
│   │   ├── FavoritesView.swift
│   │   └── FavoritesViewModel.swift
│   └── Shared/
│       ├── FavoritesModel.swift         @Observable поверх FavoritesStore, в окружении
│       ├── CountryRowView.swift
│       └── StateView.swift
└── CountriesTests/
    └── ViewModels/
```

## Компоненты

**CountriesListViewModel.** `@Observable @MainActor final class`. `var searchText`, `private(set) var countries`, `isLoading`, `errorMessage`, `isOffline`. Вычисляемое `rows` фильтрует по `searchText`. Методы `load()`, `refresh()`. Не импортирует SwiftUI, импортирует Observation. Debounce не нужен: фильтрация 250 строк дешевле, чем задержка. Записать это как отличие от Combine-версии.

**CountriesListView.** `List` по `rows`, `.searchable(text: $viewModel.searchText)`, `.refreshable { await viewModel.refresh() }`, `.task { await viewModel.load() }`, `.navigationDestination(for: Country.self)`. Состояния через `overlay` с `StateView`.

**FavoritesModel.** `@Observable` обёртка над `FavoritesStore`: `private(set) var codes`, `toggle(_:)`, `isFavorite(_:)`. Создаётся один раз в `CountriesApp`, кладётся в `.environment`. Все экраны читают его напрямую. Подписок на `changes` нет: единственный писатель это сам объект.

**CountryDetailView.** Получает `Country`, читает `FavoritesModel` из окружения, кнопка вызывает `toggle`. ViewModel нет: логики нет. Это решение записать: «ViewModel только где есть что тестировать».

**FavoritesViewModel.** Соединяет `FavoritesModel.codes` с кэшем стран, отдаёт `rows`. Тонкий, но логика сортировки и объединения есть, значит тестируется.

**Навигация.** `NavigationStack` в каждой вкладке, `navigationDestination(for: Country.self)`. Координатора нет. Записать, где он был бы нужен, если бы экранов стало десять.

## Поток данных

```
ввод ──▶ CountriesListView ──▶ viewModel.searchText = text
                                       │
                                       └──▶ rows пересчитывается
                                                │
CountriesListView ◀── Observation ◀─────────────┘

кнопка ──▶ CountryDetailView ──▶ favoritesModel.toggle(code)
                                       │
все вью, читающие favoritesModel ◀─────┘
```

## Правила

- ViewModel импортирует Observation и Foundation, не SwiftUI.
- Вью не вызывают сервисы напрямую. Только ViewModel или `FavoritesModel`.
- Зависимости через init или окружение, не синглтоны.
- ViewModel есть только у экранов с логикой. У деталей нет, и это правило, а не лень.
- `ObservableObject` и `@Published` запрещены. Только Observation.
- Общее состояние избранного в одном объекте окружения. Подписок на `changes` в этом полигоне нет.

## Что здесь тестируемо

`CountriesListViewModel`, `FavoritesViewModel`, `FavoritesModel`. Обычные классы, тесты без UI и без Combine. Проверить, что `@Observable` не мешает тестам: он и не должен.

## Где ожидать боль

- Соблазн завести ViewModel деталям «для единообразия». Не заводить.
- Окружение против init: где граница. Правило: через окружение только то, что нужно многим экранам.
- `.task` отменяется при уходе с экрана. Если `load()` не готов к отмене, будет частичное состояние.
- Observation отслеживает только прочитанные свойства. Если вью читает `countries.count` в одном месте и `countries` в другом, перерисовки могут удивить.
- `.searchable` и `.refreshable` ведут себя по-разному на iOS и macOS. Это тема бонуса.

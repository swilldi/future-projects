# Архитектура

## Коротко

`Store` это `@Observable` объект с `state: AppState` и `dispatch(_:)`. `AppState` содержит состояние стран, избранного и поиска. `AppAction` перечисление с вложенными перечислениями по фичам. Редьюсер `(inout AppState, AppAction) -> Void` чистый. Middleware `(AppState, AppAction, dispatch) async` делает сеть и файлы и диспатчит результат. Вью получают стор из окружения.

## Почему так

В MVVM три ViewModel и общий объект избранного, и синхронизация между ними это отдельная забота. В Redux состояние одно, поэтому синхронизации нет как класса задач. Плата: каждое изменение UI это действие, и действий много. Полигон нужен, чтобы понять, стоит ли это того, и где именно захочется библиотеку.

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
│   ├── CountriesApp.swift               создаёт стор, кладёт в окружение
│   └── AppDependencies.swift
├── Models/
├── Services/
├── Redux/
│   ├── Store.swift                      @Observable, dispatch, цепочка middleware
│   ├── Reducer.swift                    typealias и композиция
│   └── Middleware.swift                 typealias
├── State/
│   ├── AppState.swift
│   ├── CountriesState.swift             countries, isLoading, error, isOffline, searchText
│   └── FavoritesState.swift             codes
├── Actions/
│   └── AppAction.swift                  enum с вложенными countries(...), favorites(...)
├── Reducers/
│   ├── appReducer.swift                 комбинирует
│   ├── countriesReducer.swift
│   └── favoritesReducer.swift
├── Middlewares/
│   ├── countriesMiddleware.swift        кэш, сеть
│   ├── favoritesMiddleware.swift        запись файла, подписка на changes
│   └── loggingMiddleware.swift          печатает действия
├── Selectors/
│   └── CountriesSelectors.swift         filteredCountries(state), favoriteCountries(state)
├── Features/
│   ├── CountriesList/CountriesListView.swift
│   ├── CountryDetail/CountryDetailView.swift
│   ├── Favorites/FavoritesView.swift
│   └── Shared/                          CountryRowView, StateView
└── CountriesTests/
    ├── Reducers/
    └── Middlewares/
```

## Компоненты

**Store.** `@Observable @MainActor final class Store`. `private(set) var state`. `dispatch(action)`: прогоняет через редьюсер, потом отдаёт действие каждому middleware. Middleware получают `dispatch`, чтобы отправлять результаты асинхронно.

**AppState.** Значимый тип, `Equatable`. Вложенные `CountriesState` и `FavoritesState`. Навигация в состоянии не хранится: `NavigationStack` держит путь сам. Записать почему: путь это состояние вью, а не приложения.

**AppAction.** `enum AppAction { case countries(CountriesAction); case favorites(FavoritesAction) }`. Внутри: `load`, `loaded([Country], fromCache: Bool)`, `failed(String)`, `searchChanged(String)`, `refresh`, `toggle(code)`, `changed(Set<String>)`.

**Редьюсеры.** Чистые. Ничего не знают о сервисах и асинхронности. `countriesReducer` на `loaded` кладёт список и сбрасывает ошибку, на `failed` ставит ошибку только если списка нет.

**Middleware.** `countriesMiddleware` на `load` читает кэш, диспатчит `loaded(fromCache: true)`, идёт в сеть, диспатчит `loaded(fromCache: false)` или `failed`. `favoritesMiddleware` на `toggle` пишет файл; при старте подписывается на `changes` и диспатчит `changed`. `loggingMiddleware` печатает каждое действие: бесплатная история.

**Селекторы.** Чистые функции над состоянием: `filteredCountries(state)` учитывает поиск, `favoriteCountries(state)` объединяет коды со списком. Вью зовут селекторы, а не считают сами.

**Вью.** `@Environment(Store.self)`. Читают через селекторы, отправляют действия. Локальное состояние только для того, что не влияет ни на что: раскрытый лист, фокус.

## Поток данных

```
ввод ──▶ CountriesListView ──▶ store.dispatch(.countries(.searchChanged(text)))
                                        │
                                        ├──▶ reducer: state.countries.searchText = text
                                        └──▶ middlewares: ничего для этого действия
                                                  │
CountriesListView ◀── Observation ◀── filteredCountries(state) ◀──┘

старт ──▶ dispatch(.countries(.load))
              ├──▶ reducer: isLoading = кэша нет
              └──▶ countriesMiddleware ──▶ кэш ──▶ dispatch(.loaded(fromCache: true))
                                       ──▶ сеть ──▶ dispatch(.loaded(fromCache: false)) | .failed
```

## Правила

- Редьюсер чистый: без `async`, без сервисов, без `Date()` и случайных чисел. Только `inout` состояние и действие.
- Побочные эффекты только в middleware. Сеть или файл в редьюсере или во вью это нарушение.
- Вью только читают состояние через селекторы и диспатчат действия. Условие над данными во вью это нарушение.
- Состояние значимый тип и `Equatable`, чтобы тесты сравнивали целиком.
- Один стор. Второй стор это другая архитектура.
- Путь навигации не в сторе. Это решение, его можно оспорить в `NOTES.md`, но не в коде.

## Что здесь тестируемо

Редьюсеры: таблица «состояние, действие, ожидаемое состояние», без моков вообще. Селекторы: чистые функции. Middleware: с моками сервисов и шпионом `dispatch`, проверять, какие действия были отправлены. Лучшая тестируемость в серии по соотношению тестов к обвязке. Проверить.

## Где ожидать боль

- Действий много, и на каждое два места: перечисление и редьюсер. Добавить флаг «офлайн» это три файла.
- Всё через `dispatch`, даже тап по кнопке. Стек вызовов в отладчике неинформативен.
- Что в глобальном состоянии, а что локально: `searchText` в сторе, а раскрытие листа нет. Граница спорная, записать критерий.
- Middleware с `async` и `dispatch` из другого потока: стор на главном акторе, поэтому `dispatch` тоже. Проверить компилятором.
- Хочется генерировать код для действий и редьюсеров. Это и есть запрос к библиотеке.

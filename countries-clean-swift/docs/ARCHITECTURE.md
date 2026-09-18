# Архитектура

## Коротко

Сцена: `ViewController` (DisplayLogic), `Interactor` (BusinessLogic и DataStore), `Presenter` (PresentationLogic), `Router` (RoutingLogic и DataPassing), `Models` (вложенные перечисления сценариев с тройками структур), `Configurator` (сборка). Сервисы обёрнуты в `Worker`, общие для сцен. Строго по кругу: VC → Interactor → Presenter → VC.

## Почему так

VIPER даёт презентеру две роли: принимать от view и принимать от интерактора. Clean Swift разрывает это: интерактор не возвращает результат презентеру-хабу, а передаёт дальше по кругу. Меньше протоколов, поток читается сверху вниз. Плата: три структуры на каждый сценарий и необычный способ передавать данные между сценами через DataStore. Проверить оба свойства на практике.

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
├── Models/                              Country, CountryDTO
├── Services/
├── Workers/
│   ├── CountriesWorker.swift            кэш + сеть, стратегия «кэш, потом сеть»
│   └── FavoritesWorker.swift            избранное и поток изменений
├── Views/                               CountryCell, CountryDetailView, StateView
├── Scenes/
│   ├── CountriesList/
│   │   ├── CountriesListModels.swift      enum CountriesList { enum Fetch, Search, Select ... }
│   │   ├── CountriesListViewController.swift
│   │   ├── CountriesListInteractor.swift
│   │   ├── CountriesListPresenter.swift
│   │   ├── CountriesListRouter.swift
│   │   └── CountriesListConfigurator.swift
│   ├── CountryDetail/                     те же шесть
│   └── Favorites/                         те же шесть
└── CountriesTests/
    ├── Interactors/
    └── Presenters/
```

## Компоненты

**Models.** `enum CountriesList { enum FetchCountries { struct Request {}; struct Response { let countries: [Country]; let isOffline: Bool }; struct ViewModel { let rows: [Row]; let offlineText: String? } } }`. И так для `Search`, `Refresh`, `SelectCountry`. Тройка на сценарий, имена вложенные.

**ViewController.** Реализует `CountriesListDisplayLogic`: `displayCountries(viewModel:)`, `displayError(viewModel:)`. Владеет `interactor: CountriesListBusinessLogic` и `router`. На событие создаёт `Request` и зовёт интерактор. Пассивный.

**Interactor.** Реализует `BusinessLogic` и `DataStore` (`var selectedCountry: Country?`). Зовёт воркеры, формирует `Response`, передаёт `presenter.presentCountries(response:)`. Не знает view. Слушает `changes` через `FavoritesWorker`.

**Presenter.** Реализует `PresentationLogic`. Превращает `Response` в `ViewModel` со строками и зовёт `viewController.displayCountries(viewModel:)`. `weak var viewController`.

**Router.** Реализует `RoutingLogic` и `DataPassing` (`var dataStore: CountriesListDataStore?`). `routeToDetail()` берёт `selectedCountry` из своего DataStore, кладёт в DataStore сцены деталей, пушит. Данные между сценами ходят через DataStore, не через init.

**Configurator.** Связывает шесть объектов, соблюдая владение: VC владеет интерактором и роутером, интерактор владеет презентером, презентер слабо держит VC.

**Workers.** `CountriesWorker.fetch(useCache:)` инкапсулирует «сначала кэш, потом сеть». Сцены переиспользуют, интеракторы тонкие.

## Поток данных

```
событие ──▶ ViewController ──▶ interactor.fetchCountries(request)
                                        │
                                        ├──▶ CountriesWorker
                                        └──▶ presenter.presentCountries(response)
                                                     │
ViewController ◀── displayCountries(viewModel) ◀─────┘

Строго по кругу. Стрелка назад запрещена.

навигация ──▶ ViewController ──▶ router.routeToDetail()
                                    ├──▶ dataStore.selectedCountry
                                    └──▶ detail.dataStore.country = ...; push
```

## Правила

- Цикл в одну сторону: VC → Interactor → Presenter → VC. Interactor не знает VC, Presenter не знает Interactor.
- Interactor и Presenter без `import UIKit`. Presenter может импортировать только для форматирования, и это записать как спорное место.
- Каждый сценарий это тройка `Request`, `Response`, `ViewModel` во вложенном перечислении. Передавать `Country` напрямую в `display` нельзя.
- Данные между сценами только через DataStore и роутер. Параметры в init сцены запрещены.
- Сервисы известны только воркерам. Интеракторы зовут воркеры.
- Три сцены структурно одинаковы.

## Что здесь тестируемо

Интерактор: шпион презентера, моки воркеров. Презентер: шпион VC. VC: шпион интерактора, проверка что событие создаёт правильный `Request`. Три слоя. Сравнить объём с VIPER.

## Где ожидать боль

- `CountriesList.FetchCountries.Response` и подобные имена в каждой сигнатуре.
- Три структуры на сценарий, даже когда `Request` пустой.
- DataStore выглядит как магия: данные появляются в сцене через свойство, а не через init. Легко забыть заполнить.
- VC одновременно вход и выход цикла, у него два протокола, и это спорно.
- Презентеру иногда нужен `UIKit` ради `NSAttributedString` или цвета. Решить, разрешать ли.

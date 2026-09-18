# Архитектура

## Коротко

Три пакета. `Domain` без зависимостей: `Country`, протоколы `CountriesRepository` и `FavoritesRepository`, use case-ы `GetCountries`, `RefreshCountries`, `ToggleFavorite`, `ObserveFavorites`. `Data` зависит от Domain: `CountriesAPI`, `CountryCache`, `FavoritesStore`, DTO, маппинги, реализации репозиториев. `Presentation` зависит от Domain: вью и ViewModel, которые знают только use case-ы. Приложение зависит от всех и собирает граф в `AppContainer`.

## Почему так

Слои на папках держатся на честном слове. Слои на пакетах держатся на компиляторе: `Domain` физически не может импортировать `Data`. Это единственный способ прочувствовать правило зависимостей, а не прочитать о нём. MVVM сверху потому, что Clean Architecture не говорит, как устроен экран, и MVVM из четвёртого полигона уже знаком: меняется только то, откуда ViewModel берёт данные.

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
├── Countries.xcodeproj
├── Packages/
│   ├── Domain/
│   │   ├── Package.swift                зависимостей нет
│   │   ├── Sources/Domain/
│   │   │   ├── Entities/Country.swift
│   │   │   ├── Repositories/            CountriesRepository, FavoritesRepository (протоколы)
│   │   │   └── UseCases/                GetCountriesUseCase, RefreshCountriesUseCase,
│   │   │                                ToggleFavoriteUseCase, ObserveFavoritesUseCase
│   │   └── Tests/DomainTests/
│   ├── Data/
│   │   ├── Package.swift                зависит от Domain
│   │   ├── Sources/Data/
│   │   │   ├── Network/                 CountriesAPI, CountryDTO
│   │   │   ├── Persistence/             CountryCache, FavoritesStore
│   │   │   ├── Mappers/                 CountryDTO+Domain
│   │   │   └── Repositories/            CountriesRepositoryImpl, FavoritesRepositoryImpl
│   │   └── Tests/DataTests/
│   └── Presentation/
│       ├── Package.swift                зависит от Domain
│       ├── Sources/Presentation/
│       │   ├── Features/CountriesList/  View, ViewModel
│       │   ├── Features/CountryDetail/
│       │   ├── Features/Favorites/
│       │   └── Shared/                  CountryRowView, StateView
│       └── Tests/PresentationTests/
└── App/
    ├── CountriesApp.swift
    └── AppContainer.swift               composition root
```

## Компоненты

**Domain.** `Country` как чистая структура. Протоколы репозиториев: `CountriesRepository { func cached() -> [Country]?; func fetch() async throws -> [Country] }`, `FavoritesRepository { func codes() -> Set<String>; func toggle(_:); var changes: AsyncStream<Set<String>> }`. Use case-ы как протокол плюс реализация, каждый с одним методом `execute`. `GetCountriesUseCase` отдаёт кэш, `RefreshCountriesUseCase` ходит в сеть и просит репозиторий сохранить. Разделены намеренно: ViewModel сам решает «показать кэш, потом обновить». Записать, что это спорно: кто-то положил бы стратегию в репозиторий.

**Data.** Три сервиса из общей части, теперь они детали реализации. `CountriesRepositoryImpl` собирает API и кэш. Маппинг `CountryDTO → Country` только здесь. `Domain` никогда не видит DTO.

**Presentation.** ViewModel из четвёртого полигона, но вместо сервисов принимает use case-ы. `FavoritesModel` из окружения превращается в ViewModel, который слушает `ObserveFavoritesUseCase`. Вью не изменились: это надо заметить.

**AppContainer.** Единственное место, где `Data` встречается с `Presentation`. Создаёт репозитории, use case-ы, ViewModel-фабрики. Без DI-библиотеки, руками. Растёт с каждым use case-ом, и это тоже наблюдение.

## Поток данных

```
CountriesListView ──▶ ViewModel.load()
                            ├──▶ GetCountriesUseCase.execute()      ──▶ CountriesRepository (протокол в Domain)
                            │                                              └──▶ CountriesRepositoryImpl (Data) ──▶ CountryCache
                            └──▶ RefreshCountriesUseCase.execute()  ──▶ ... ──▶ CountriesAPI, маппинг, CountryCache
                                       │
CountriesListView ◀── Observation ◀────┘

Зависимости: Presentation → Domain ◀── Data. Стрелки к домену, никогда от него.
```

## Правила

- `Domain/Package.swift` без зависимостей. `Data` и `Presentation` зависят только от `Domain`. Приложение от всех трёх.
- В `Domain` нет `import SwiftUI`, `UIKit`, `Combine`. `Foundation` только ради `URL` и `Date`, и это записать как компромисс.
- DTO не выходит за пределы `Data`.
- ViewModel знает только use case-ы. Репозиторий в ViewModel это нарушение.
- Use case делает одно. Если он только пересылает вызов репозиторию, это тоже нормально: записать, сколько таких.
- Сборка графа только в `AppContainer`.

## Что здесь тестируемо

Три слоя независимо. Use case-ы с моками репозиториев. Репозитории с моками API и файлов. ViewModel с моками use case-ов. Каждый пакет тестируется командой `swift test` без симулятора, кроме `Presentation`.

## Где ожидать боль

- Типов много: на список это `Country`, DTO, два протокола репозитория, два impl, два use case-а с протоколами, ViewModel, вью. Посчитать.
- Use case-ы, которые пересылают вызов. Хочется их убрать, и это правильный вопрос: записать ответ.
- `AppContainer` разрастается и превращается в то, ради чего люди берут DI-библиотеки.
- `Foundation` в домене: без него нет `URL`. Пурист вынес бы `flagURL` в строку.
- Маппинг DTO в сущность это код, который не делает ничего видимого. Каждое новое поле трогает три места.

# Архитектура

## Коротко

Модуль экрана: `View` (контроллер, пассивный), `Presenter` (принимает события view, готовит данные к показу, дёргает интерактор и роутер), `Interactor` (логика и сервисы, сообщает результат презентеру через протокол вывода), `Router` (строит и показывает следующий модуль), `Entity` (модели, общие для всех), `Assembly` (собирает модуль). Шесть файлов на модуль, три модуля.

## Почему так

VIPER доводит идею MVP до конца: презентер больше не ходит в сервисы (это интерактор) и не навигирует (это роутер). Каждый объект делает одно. Плата очевидна: файлы и протоколы. Полигон нужен, чтобы её измерить, а не чтобы прочитать о ней. И заодно проверить гипотезу, что такой шаблонный код это идеальная задача для агента.

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
├── Entities/                            Country, CountryDTO
├── Services/
├── Views/                               CountryCell, CountryDetailView, StateView
├── Modules/
│   ├── CountriesList/
│   │   ├── CountriesListProtocols.swift   ViewInput, ViewOutput, InteractorInput, InteractorOutput, RouterInput
│   │   ├── CountriesListViewController.swift
│   │   ├── CountriesListPresenter.swift
│   │   ├── CountriesListInteractor.swift
│   │   ├── CountriesListRouter.swift
│   │   └── CountriesListAssembly.swift
│   ├── CountryDetail/                     те же шесть файлов
│   └── Favorites/                         те же шесть файлов
└── CountriesTests/
    ├── Presenters/
    └── Interactors/
```

## Компоненты

**Protocols.** Пять на модуль. `ViewInput` (что презентер может показать во view), `ViewOutput` (что view сообщает презентеру), `InteractorInput` (что презентер просит у интерактора), `InteractorOutput` (что интерактор сообщает презентеру), `RouterInput` (куда презентер просит перейти).

**ViewController.** Реализует `ViewInput`, владеет `output: ViewOutput` (презентер). Пассивный, как в MVP.

**Presenter.** Реализует `ViewOutput` и `InteractorOutput`. `weak var view: ViewInput`, `interactor: InteractorInput`, `router: RouterInput`. Превращает результат интерактора в строки для view. Не ходит в сервисы, не знает UIKit.

**Interactor.** Реализует `InteractorInput`. `weak var output: InteractorOutput`. Владеет сервисами: кэш, сеть, избранное, подписка на `changes`. Вся логика данных здесь. Не знает UIKit и не знает о view.

**Router.** Реализует `RouterInput`. `weak var viewController`. Строит следующий модуль через его `Assembly` и пушит. Единственный, кто знает о `UINavigationController`.

**Assembly.** `static func build(dependencies:) -> UIViewController`. Создаёт все пять, связывает, соблюдая правила владения: view владеет презентером, презентер владеет интерактором и роутером, обратные ссылки слабые.

**Синхронизация избранного.** Интерактор слушает `changes`, сообщает презентеру через `InteractorOutput.favoritesDidChange`, презентер обновляет view. Три модуля, три подписки, зато каждая в своём интеракторе.

## Поток данных

```
событие ──▶ View ──▶ Presenter (ViewOutput)
                          ├──▶ Interactor (InteractorInput) ──▶ сервисы
                          │         └──▶ Presenter (InteractorOutput)
                          │                    └──▶ View (ViewInput)
                          └──▶ Router (RouterInput) ──▶ Assembly следующего модуля ──▶ push

Презентер в центре, но сам ничего не делает: только переводит.
```

## Правила

- Любое общение между сущностями модуля только через протокол из `Protocols.swift`.
- Interactor и Presenter без `import UIKit`. Router и View с ним.
- Сервисы известны только интерактору. Презентер, увидевший `CountriesAPI`, это нарушение.
- Роутер единственный, кто пушит и кто вызывает чужие `Assembly`.
- Владение: View → Presenter → Interactor, Router. Обратные ссылки `weak`. Проверять в `Assembly`.
- Три модуля устроены одинаково. Отличие в структуре модуля это нарушение, даже если так удобнее.

## Что здесь тестируемо

Презентер: моки view, интерактора, роутера. Интерактор: моки сервисов и шпион вывода. Два слоя тестов, и это больше, чем где-либо до сих пор. Измерить, окупает ли второй слой свою обвязку.

## Где ожидать боль

- Восемнадцать файлов на три экрана, пять протоколов на модуль, имена вроде `CountriesListInteractorOutput`.
- Цикл ссылок ловится только вниманием в `Assembly`. Проверить через `deinit` с print.
- Передача данных между модулями идёт через роутер и `Assembly`: параметры init плодятся.
- Простейшее действие проходит четыре объекта. Отладка через точки останова утомляет.
- Модули отличаются на одну строку, но копировать шесть файлов руками тоскливо. Здесь и нужен агент.

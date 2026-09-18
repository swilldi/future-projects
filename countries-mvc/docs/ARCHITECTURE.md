# Архитектура

## Коротко

Классический MVC от Apple. Контроллер экрана владеет всем: получает данные из сервисов, фильтрует, собирает снапшот таблицы, обрабатывает ошибки, переходит на следующий экран. Модели и сервисы чистые, views тупые, вся сложность в контроллерах.

## Почему так

Это не рекомендация, а базовая линия. Все остальные полигоны решают проблемы, которые здесь должны проявиться сами. Поэтому здесь запрещено «немного улучшить»: не выносить логику в помощники, не заводить ViewModel под другим именем. Честный MVC, чтобы честно было с чем сравнивать.

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
│   ├── SceneDelegate.swift              собирает UITabBarController
│   └── AppDependencies.swift            создаёт сервисы, отдаёт контроллерам
├── Models/
│   ├── Country.swift
│   └── CountryDTO.swift                 DTO и маппинг
├── Services/
│   ├── CountriesAPI.swift
│   ├── CountryCache.swift
│   └── FavoritesStore.swift
├── Views/
│   ├── CountryCell.swift                ячейка списка
│   ├── CountryDetailView.swift          UIView с лейблами деталей
│   └── StateView.swift                  загрузка, ошибка, пусто
├── Controllers/
│   ├── CountriesListViewController.swift
│   ├── CountryDetailViewController.swift
│   └── FavoritesViewController.swift
└── CountriesTests/
```

## Компоненты

**CountriesListViewController.** Держит массив стран, строку поиска, ссылки на три сервиса. В `viewDidLoad` читает кэш, показывает, запускает загрузку в `Task`. Реализует `UISearchResultsUpdating`, собирает `NSDiffableDataSourceSnapshot`, обрабатывает pull-to-refresh, показывает `StateView`, подписывается на `favoritesStore.changes`, по тапу создаёт `CountryDetailViewController` и пушит. Это и есть эксперимент: всё в одном месте.

**CountryDetailViewController.** Получает `Country` и `FavoritesStore`. Показывает `CountryDetailView`, по кнопке пишет в стор, слушает `changes`, чтобы кнопка отражала актуальное состояние.

**FavoritesViewController.** Читает кэш и избранное, фильтрует, показывает тот же `CountryCell`. Подписан на `changes`. Половина кода совпадает со списком, и это намеренно: копипаста тоже наблюдение.

**Views.** `CountryCell`, `CountryDetailView`, `StateView` не знают о `Country`: принимают строки и URL через `configure(...)`. Это единственная граница, которую MVC держит.

**AppDependencies.** Создаёт сервисы один раз и передаёт контроллерам через init. Без этого контроллеры создавали бы сервисы сами, и тестировать было бы нечего.

## Поток данных

```
тап / ввод ──▶ CountriesListViewController
                    ├──▶ CountriesAPI, CountryCache, FavoritesStore
                    ├──▶ фильтрация, сборка снапшота
                    ├──▶ dataSource.apply(snapshot)
                    └──▶ navigationController.push(Detail)

Все стрелки выходят из контроллера и возвращаются в него.
```

## Правила

- Views не импортируют ничего, кроме UIKit, и не знают о `Country`: только `configure(title:subtitle:flagURL:)`.
- Сервисы не знают о UIKit.
- Контроллеры могут всё: сервисы, модели, views, навигация.
- Запрещено заводить промежуточные объекты между контроллером и сервисами. Если рука тянется, записать в `NOTES.md`, что именно захотелось вынести: это и есть результат.
- Каждый контроллер сам подписывается на `changes` и сам управляет своим `Task`. Общего наблюдаемого объекта нет, это уже другая архитектура.

## Что здесь тестируемо

Сервисы и маппинг. Логика фильтрации и состояний живёт в контроллере, и чтобы её проверить, нужно создать контроллер, загрузить view и подсунуть моки сервисов. Сделать один такой тест, чтобы прочувствовать цену.

## Где ожидать боль

- `CountriesListViewController` перевалит за 250 строк. Не бороться, засечь.
- Фильтрация, обработка ошибок и сборка снапшота перемешаны в одном методе.
- `FavoritesViewController` дублирует половину списка.
- Три контроллера, три подписки на `changes`, три места, где можно забыть отменить `Task`.
- Тест контроллера требует `loadViewIfNeeded()` и знания о жизненном цикле.

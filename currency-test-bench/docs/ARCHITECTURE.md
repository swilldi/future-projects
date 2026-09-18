# Архитектура

## Коротко

MVVM на SwiftUI с Observation, как в полигоне `countries-mvvm-swiftui`. Отличие в том, что каждая граница проведена так, чтобы её можно было тестировать отдельно: логика без Foundation-сети, сеть за протоколом с подменой на уровне `URLProtocol`, ViewModel с фальшивым репозиторием, интерфейс со снапшотами и UI-тестами на фикстуре.

## Почему так

Тестируемость это свойство архитектуры, а не тестов. Если логика конвертации знает про сеть, её нельзя проверить без сети. Если ViewModel сам создаёт URLSession, его нельзя проверить без сервера. Поэтому границы здесь важнее, чем в обычном приложении такого размера, и они оправданы целью проекта.

## Структура папок

```
CurrencyBench/
├── App/
│   ├── CurrencyBenchApp.swift
│   ├── AppDependencies.swift        сборка графа, учитывает аргументы запуска
│   └── LaunchArguments.swift        единственное место, где читаются -uiTesting
├── Domain/
│   ├── Currency.swift
│   ├── Rates.swift
│   └── ConversionEngine.swift       чистая логика на Decimal
├── Data/
│   ├── RatesClient.swift            протокол
│   ├── FrankfurterClient.swift      URLSession, декодирование
│   ├── RatesCache.swift             JSON-файл
│   ├── RatesRepository.swift        клиент + кэш, политика обновления
│   └── FixtureRatesClient.swift     для -uiTesting, читает фикстуру из бандла
├── Features/
│   └── Converter/
│       ├── ConverterView.swift
│       ├── ConverterViewModel.swift
│       └── CurrencyPickerView.swift
├── Resources/
│   └── rates-fixture.json
├── UnitTests/                        Swift Testing
│   ├── ConversionEngineTests.swift
│   ├── RatesRepositoryTests.swift
│   ├── ConverterViewModelTests.swift
│   ├── Doubles/                      FakeRatesClient, InMemoryRatesCache
│   └── Network/                      URLProtocolStub, FrankfurterClientTests
├── SnapshotTests/                    XCTest + swift-snapshot-testing
│   └── ConverterViewSnapshotTests.swift
├── UITests/                          XCTest, XCUIApplication
│   └── ConverterFlowTests.swift
└── PerformanceTests/                 XCTest measure
    └── ConversionEnginePerformanceTests.swift
```

## Компоненты

**ConversionEngine.** Структура с одной функцией `convert(amount:from:to:rates:) throws -> Decimal`. Без зависимостей. Ошибки: `missingRate(code)`. Тестируется таблицей.

**RatesClient.** Протокол `func latest() async throws -> Rates`. `FrankfurterClient` реализует через URLSession, принимает `URLSessionConfiguration` в init, чтобы тесты подставили конфигурацию с `URLProtocolStub`. `FixtureRatesClient` читает `rates-fixture.json` для UI-тестов.

**RatesRepository.** Принимает клиент, кэш и часы (`Clock` или замыкание `now`). Политика: отдать кэш, если свежее суток; иначе загрузить и сохранить; при ошибке сети отдать кэш с флагом «устарело». Тестируется с фальшивым клиентом и кэшем в памяти. Часы подменяются, чтобы проверять «сутки прошли».

**ConverterViewModel.** `@Observable`. Входы: `amountText`, `from`, `to`. Выход: `result: String?`, `ratesDate: String?`, `error: String?`, `isLoading`. Форматирование чисел здесь, через `NumberFormatter` с локалью из init, чтобы тест зафиксировал локаль.

**ConverterView.** Только вёрстка и биндинги. Три состояния для снапшотов: обычное, «курсы устарели», «курсы недоступны».

**AppDependencies.** Читает `LaunchArguments`. С `-uiTesting` собирает `FixtureRatesClient` и кэш в памяти. Без него настоящие. Это единственная ветка «для тестов» в приложении.

## Виды тестовых дублёров, которые здесь есть

- **Stub:** `URLProtocolStub` отдаёт заранее заданный ответ. Проверяет, что клиент правильно строит запрос и декодирует ответ.
- **Fake:** `InMemoryRatesCache` работает по-настоящему, но в памяти. Для репозитория.
- **Spy:** не нужен. Если захотелось, записать почему.
- **Mock:** намеренно не используется. Записать разницу с остальными.

## Поток данных

```
ввод ──▶ ConverterView ──▶ viewModel.amountText = "1,5"
                                │
                                ├──▶ ConversionEngine.convert(...)  ◀── rates из RatesRepository
                                └──▶ result = "1,62 USD"
                                          │
ConverterView ◀── Observation ◀───────────┘

старт ──▶ viewModel.load() ──▶ RatesRepository.rates() ──▶ кэш свежий? отдать : RatesClient.latest() ──▶ кэш.save
```

## Правила

- `Domain` без `import Foundation`-сети и без SwiftUI. `Decimal` из Foundation допустим.
- `ConversionEngine` не знает о кэше и сети.
- ViewModel принимает репозиторий через init, локаль через init.
- Аргументы запуска читаются в одном файле. Проверка `ProcessInfo` в другом месте это нарушение.
- Тесты не спят и не ждут по таймеру. Асинхронность через `await` и `confirmation`.
- Снапшоты записываются один раз и коммитятся. Перезапись только осознанно.

## Что здесь тестируемо

Всё, и в этом суть. `ConversionEngine` таблицей. `FrankfurterClient` через `URLProtocolStub`. `RatesRepository` с фейками и подменённым временем. `ConverterViewModel` с фальшивым репозиторием. `ConverterView` снапшотами. Сценарий целиком UI-тестом на фикстуре. Производительность движка через `measure`.

## Где ожидать боль

- `URLProtocol` регистрируется глобально или через конфигурацию сессии. Через конфигурацию правильнее, глобально проще. Выбрать конфигурацию.
- Снапшоты зависят от устройства и версии ОС. Закрепить одно устройство в тестах и записать в `NOTES.md`.
- UI-тесты медленные и падают от анимаций. Отключить анимации через аргумент запуска, ждать элементы через `waitForExistence`.
- `Decimal` и локали: «1,5» на русской локали и «1.5» на английской. Парсинг через `NumberFormatter` с локалью из окружения.
- CI на macOS-раннере медленный и платный по минутам. Кэшировать зависимости, гонять UI-тесты только на main.

# Стек

Версии на момент написания, сентябрь 2026. При старте проверить и поправить.

## Основа

| Что | Зачем здесь | Что хочу понять |
|---|---|---|
| Swift 6, SwiftUI, Observation | Приложение | Ничего нового, всё из полигона MVVM |
| `Decimal` и `NumberFormatter` | Деньги | Почему не `Double`, как форматировать по локали и парсить обратно |
| Swift Testing | Юнит и интеграционные тесты | `@Test(arguments:)`, `#expect(throws:)`, `confirmation`, теги, параллельность |
| XCTest | UI-тесты и замеры | `XCUIApplication`, `launchArguments`, `waitForExistence`, `measure` |
| `URLProtocol` | Подмена сети | Почему это надёжнее, чем мок клиента, и как передать через `URLSessionConfiguration` |
| swift-snapshot-testing | Снапшоты | Запись, сравнение, стратегия `.image` с точностью, где хранить эталоны |
| GitHub Actions | CI | Раннер macOS, выбор Xcode, `xcodebuild test`, кэш, разделение юнитов и UI-тестов |
| Покрытие в Xcode | Метрика | Как включить, как читать, почему 100 процентов интерфейса не цель |

## Чего нет и почему

- Моков через библиотеки. Дублёры пишутся руками, чтобы понять, что они делают.
- Сторонних сетевых библиотек. URLSession хватает.

## Что почитать и посмотреть

**Перед фазой 1:**
- Документация Swift Testing целиком, она короткая: «Defining test functions», «Parameterized testing», «Expectations».
- WWDC24 «Meet Swift Testing» и «Go further with Swift Testing».

**Перед фазой 2:**
- Документация: `URLProtocol`. Статья на objc.io «Testing Network Requests» или аналог, суть одна.

**Перед фазой 4:**
- README swift-snapshot-testing, раздел про стратегии и точность.

**Перед фазой 5:**
- WWDC15 «UI Testing in Xcode», старая, но база не менялась.
- Документация: «Testing your apps in Xcode», «Running tests and interpreting results».

**Перед фазой 6:**
- Документация GitHub Actions: раннеры macOS, действие `maxim-lobanov/setup-xcode` или выбор через `xcode-select`.

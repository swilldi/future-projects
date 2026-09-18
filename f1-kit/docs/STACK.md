# Стек

Версии на момент написания, сентябрь 2026. При старте проверить и поправить.

## Основа

| Что | Зачем здесь | Что хочу понять |
|---|---|---|
| Swift Package с тремя таргетами | Библиотека, тесты, команда | `Package.swift`: продукты, зависимости таргетов, ресурсы для фикстур |
| Дженерики и `Endpoint<Response>` | Связь пути и типа | Дженерик-структура с замыканием декодера, вывод типов |
| Протоколы `HTTPClient`, `Sleeper`, `Interceptor` | Границы для тестов | `Sendable` протоколы, `any` против `some` |
| Акторы | Ограничение частоты | Скользящее окно под изоляцией |
| `Duration`, `Date`, `ISO8601FormatStyle` | Время | Разбор дат в UTC без локали, арифметика |
| `URLProtocol` | Тесты клиента | Как подсунуть через `URLSessionConfiguration`, ограничения на Linux |
| DocC | Документация | Каталог `.docc`, символы, туториал, сборка из терминала |
| Семантические версии, теги | Публикация | Что ломающее, что нет; `from:` против `exact:` у потребителя |
| swift-argument-parser | Команда | Подкоманды, опции, справка |
| Docker `swift:6` | Linux | `swift test` в контейнере, что из Foundation отсутствует |
| GitHub Actions | CI | Матрица macOS и Linux |

## Чего нет и почему

- Alamofire и подобных. Транспорт это предмет.
- Кэша. Забота потребителя.

## Что почитать и посмотреть

**Перед фазой 0:**
- Документация: «Creating a standalone Swift package with Xcode», «Package.swift» справочник.
- Jolpica, README и описание эндпоинтов. Записать лимиты.

**Перед фазой 1:**
- Документация: `JSONDecoder`, `DecodingError`, `codingPath`.

**Перед фазой 2:**
- WWDC22 «Meet Swift Async Algorithms» не нужен; нужен WWDC21 «Explore structured concurrency» ещё раз, раздел про отмену в повторах.
- Документация: `Duration`, `Clock`.

**Перед фазой 5:**
- WWDC21 «Meet DocC documentation in Xcode», WWDC22 «What's new in Swift-DocC».
- Semantic Versioning, semver.org, одна страница.

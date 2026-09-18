# Стек

Версии на момент написания, сентябрь 2026. При старте проверить и поправить.

## Основа

| Что | Зачем здесь | Что хочу понять |
|---|---|---|
| Swift 6, strict concurrency | Всё | Компилятор как учитель: каждое предупреждение это вопрос |
| `TaskGroup` | Структурная параллельность | Отмена группы, `next()`, почему нельзя отменить одного ребёнка |
| Акторы | Ограничитель и загрузчик | Изоляция, реентерабельность, `nonisolated` |
| `AsyncStream` | Шина событий | Продолжение, буферизация, завершение, один потребитель |
| `URLSession.bytes` | Прогресс | `AsyncBytes`, чтение кусками, отмена |
| `Task.checkCancellation`, `withTaskCancellationHandler` | Отмена | Кооперативность: кто и когда проверяет |
| Приоритеты задач | Обработка | `TaskPriority`, наследование, инверсия |
| `Sendable` | Границы | Что можно передать через `await`, что нет, и почему `CGImage` нельзя |
| Thread Sanitizer | Лаборатория | Как включить, как читать отчёт |
| SwiftUI `LazyVGrid` | Сетка | Ничего нового |

## Чего нет и почему

- GCD целиком. Цель проекта прожить без него.
- Библиотек для картинок. Загрузка это и есть предмет.

## Что почитать и посмотреть

**Перед фазой 1:**
- WWDC21 «Explore structured concurrency in Swift». Обязательно, дважды.
- WWDC21 «Protect mutable state with Swift actors».
- The Swift Programming Language, глава «Concurrency».

**Перед фазой 3:**
- WWDC21 «Meet AsyncSequence».
- Документация: `AsyncStream`, `URLSession.bytes(for:)`.

**Перед фазой 6:**
- WWDC22 «Eliminate data races using Swift Concurrency».
- WWDC24 «Migrate your app to Swift 6».
- Документация Xcode: «Diagnosing memory, thread, and crash issues early», раздел Thread Sanitizer.

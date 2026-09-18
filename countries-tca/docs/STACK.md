# Стек

Версии на момент написания, сентябрь 2026. При старте проверить и поправить.

## Общее для всех полигонов

| Что | Зачем здесь | Что хочу понять |
|---|---|---|
| Swift 6, strict concurrency | Гонки ловит компилятор | Где ставить `@MainActor`, как передавать данные между акторами |
| Актуальный Xcode, минимум iOS 26 | Свежие API | Ничего специального |
| URLSession и Codable | Сеть | Декодирование вложенных структур API, `keyDecodingStrategy`, ошибки декодирования |
| FileManager и JSON | Кэш и избранное | Разница Caches и Documents, атомарная запись |
| Swift Testing | Тесты | `@Test`, `#expect`, параметризованные тесты, `confirmation` для асинхронных событий |
| swift-format | Стиль | Один конфиг на все полигоны |
| SwiftUI | Экраны | `NavigationStack`, `List`, `.searchable`, `.refreshable`, `.task`, `AsyncImage`, `TabView` |

## Специфично для этой архитектуры

| Что | Зачем здесь | Что хочу понять |
|---|---|---|
| swift-composable-architecture | Вся архитектура | `@Reducer`, `@ObservableState`, `Effect`, `@Dependency`, `@Presents`, `@Shared`, `TestStore` |
| swift-dependencies | Зависимости | Идёт вместе с TCA. `DependencyKey`, `withDependencies`, `unimplemented` |
| Точное закрепление версии в SPM | Стабильность | Как закрепить `exact` и почему для TCA это важнее, чем обычно |

## Чего нет и почему

- Сторонних библиотек. Кроме одной: swift-composable-architecture. Это единственный полигон с зависимостью, и в этом его смысл.
- CoreData и SwiftData: два JSON-файла закрывают потребность, хранилища это другой полигон.
- Кэша картинок: не предмет эксперимента.

## Что почитать и посмотреть

Перед соответствующей фазой плана, не всё сразу.

**Перед фазой 0:**
- README swift-composable-architecture, раздел Getting started. Закрепить версию, которая там указана как текущая.

**Перед фазой 2:**
- Документация TCA: «Meet the Composable Architecture» и статья про `@Reducer`.
- Point-Free, бесплатный тур по TCA, первые эпизоды.

**Перед фазой 3:**
- Документация TCA: «Navigation», раздел про tree-based и stack-based. Документация про `@Shared`.

**Перед фазой 5:**
- Документация TCA: «Testing».

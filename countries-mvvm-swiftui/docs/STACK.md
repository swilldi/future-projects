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
| Observation | Состояние | `@Observable`, `@Bindable`, что именно вызывает перерисовку |
| `NavigationStack` | Навигация | `navigationDestination(for:)`, `NavigationPath`, где хранить путь |
| `.environment` с объектом | Общее состояние | Когда окружение уместнее init |
| `NavigationSplitView` | Бонус macOS | Что меняется в навигации на большом экране |

## Чего нет и почему

- Сторонних библиотек. Всё есть в системе, и цель как раз посмотреть, сколько архитектуры можно построить без них.
- CoreData и SwiftData: два JSON-файла закрывают потребность, хранилища это другой полигон.
- Кэша картинок: не предмет эксперимента.

## Что почитать и посмотреть

Перед соответствующей фазой плана, не всё сразу.

**Перед фазой 2:**
- WWDC23 «Discover Observation in SwiftUI».
- WWDC22 «The SwiftUI cookbook for navigation».
- Документация: «Migrating from the Observable Object protocol to the Observable macro».

**Перед фазой 3:**
- Документация: «Managing model data in your app», раздел про окружение.

**Перед бонусом:**
- Документация: `NavigationSplitView`.
- WWDC22 «What's new in SwiftUI» про многоплатформенность, кратко.

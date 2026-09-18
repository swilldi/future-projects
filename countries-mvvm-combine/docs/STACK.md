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
| UIKit программно | Экраны | `UITableView`, diffable data source, `UISearchController`, `UIRefreshControl`, `UITabBarController`, `UINavigationController` |
| `AsyncImage` нет | Флаги | Как грузить картинку в ячейку через `Task` и не получить чужой флаг при переиспользовании ячейки |

## Специфично для этой архитектуры

| Что | Зачем здесь | Что хочу понять |
|---|---|---|
| Combine | Биндинги | `@Published`, `sink`, `debounce`, `removeDuplicates`, `receive(on:)`, `AnyCancellable`, память |
| `.values` на Publisher | Тесты | Как читать Combine как `AsyncSequence` и не ждать вручную |
| `UIHostingController` | Бонус | Как встроить SwiftUI-экран в UIKit-навигацию и отдать ему тот же ViewModel |

## Чего нет и почему

- Сторонних библиотек. Всё есть в системе, и цель как раз посмотреть, сколько архитектуры можно построить без них.
- CoreData и SwiftData: два JSON-файла закрывают потребность, хранилища это другой полигон.
- Кэша картинок: не предмет эксперимента.

## Что почитать и посмотреть

Перед соответствующей фазой плана, не всё сразу.

**Перед фазой 2:**
- WWDC19 «Introducing Combine» и «Combine in Practice».
- Документация: «Receiving and Handling Events with Combine».

**Перед фазой 5:**
- Документация Combine: `Publisher.values`.

**Перед бонусом:**
- Документация: `UIHostingController`, «Using SwiftUI with UIKit».

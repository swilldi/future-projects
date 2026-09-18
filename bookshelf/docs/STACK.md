# Стек

Версии на момент написания, сентябрь 2026. При старте проверить и поправить.

## Основа

| Что | Зачем здесь | Что хочу понять |
|---|---|---|
| SwiftData с CloudKit | Хранилище и синк | `ModelConfiguration(cloudKitDatabase:)`, ограничения модели, `externalStorage`, что происходит при конфликте |
| CloudKit Dashboard | Схема | Контейнер, Development и Production, деплой схемы |
| VisionKit `DataScannerViewController` | Скан | Символологии, `isSupported`, делегат, подсветка |
| Open Library, Google Books | Метаданные | Форматы, редиректы, обложки, отсутствие ключей |
| Многоплатформенный таргет | iOS и macOS | `#if os`, что общее, `NavigationSplitView` против `TabView` |
| `@Query`, `#Predicate` | Списки | Динамические предикаты, сортировка |
| `CGImageSource` | Обложки | Уменьшение без UIKit |
| Swift Testing с контейнером в памяти | Тесты | `isStoredInMemoryOnly` |

## Чего нет и почему

- Своего сервера. iCloud закрывает задачу для личной библиотеки.
- Библиотек для штрихкодов. VisionKit умеет.

## Что почитать и посмотреть

**Перед фазой 1:**
- Документация: «Syncing model data across a person's devices» в SwiftData. Раздел про ограничения модели читать первым.
- WWDC23 «Meet SwiftData», «Model your schema with SwiftData».

**Перед фазой 2:**
- Документация Open Library API: Books API, Covers API.

**Перед фазой 4:**
- WWDC22 «Capture machine-readable codes and text with VisionKit».
- Документация: `DataScannerViewController`.

**Перед фазой 5:**
- Документация CloudKit: «Enabling CloudKit in your app», «Deploying an iCloud container's schema».

**Перед фазой 7:**
- WWDC22 «What's new in SwiftUI» про многоплатформенность, документация «Building a multiplatform app».

# Стек

Версии на момент написания, сентябрь 2026. При старте проверить и поправить.

## Основа

| Что | Зачем здесь | Что хочу понять |
|---|---|---|
| CoreLocation | Маршрут | Разрешения, фоновый режим, точность, фильтры, делегат, `activityType`, батарея |
| CoreMotion `CMPedometer` | Шаги вживую | Запрос за интервал и живые обновления |
| HealthKit | Шаги и тренировка | Типы, авторизация, `HKStatisticsQuery`, `HKWorkoutBuilder`, `HKWorkoutRouteBuilder` |
| ActivityKit | Плашка | `ActivityAttributes`, `ContentState`, запрос, обновление, завершение, бюджет |
| App Intents | Кнопка паузы | `LiveActivityIntent`, где выполняется |
| WidgetKit | Вью плашки и острова | Таргет расширения, `ActivityConfiguration`, `DynamicIsland` |
| MapKit в SwiftUI | Карта | `Map`, `MapPolyline`, `MapCameraPosition`, `UserAnnotation` |
| Swift Charts | Графики | `LineMark`, `AreaMark`, `BarMark`, оси, интерполяция |
| GRDB | Хранилище | `DatabasePool`, миграции, транзакции, индексы, `ValueObservation` |
| Акторы и `AsyncStream` | Рекордер | Из полигона многопоточности |

## Чего нет и почему

- Сервера. Личный трекер.
- Готовых SDK трекинга. Цель системные API.

## Что почитать и посмотреть

**Перед фазой 2:**
- Документация: «Configuring your app to use location services», «Handling location updates in the background», `CLLocationManager`.
- WWDC19 «What's New in Core Location», WWDC23 «Discover streamlined location updates» про `CLLocationUpdate` как альтернативу делегату.

**Перед фазой 4:**
- Документация: «Creating and updating Live Activities», «Displaying live data with Live Activities».
- WWDC23 «Meet ActivityKit», «Update Live Activities with push notifications» не нужно.

**Перед фазой 5:**
- Документация: «Setting up HealthKit», «Reading data from HealthKit», «Creating a workout session» для iOS.

**Перед фазой 6:**
- WWDC22 «Hello Swift Charts», «Swift Charts: Raise the bar».
- Документация MapKit: «Map», «MapPolyline».

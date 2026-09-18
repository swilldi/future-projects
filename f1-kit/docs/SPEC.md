# Спецификация

## Одним абзацем

Пакет с библиотекой `F1Kit` и командой `f1`. Библиотека отдаёт расписание, результаты и зачёты из Jolpica через async-методы, работает на iOS и macOS, ошибки понятные, повторы и частота под контролем. Команда показывает, что библиотека живая.

## Публичный API

```
public struct F1Client: Sendable {
  public init(httpClient: any HTTPClient = URLSessionHTTPClient(),
              configuration: Configuration = .default)

  public func schedule(season: Int) async throws -> [Race]
  public func results(season: Int, round: Int) async throws -> [RaceResult]
  public func driverStandings(season: Int) async throws -> [DriverStanding]
  public func constructorStandings(season: Int) async throws -> [ConstructorStanding]
  public func nextSession(after date: Date, season: Int) async throws -> Session?
}

public struct Race: Sendable, Hashable { round, name, circuit, country, sessions: [Session] }
public struct Session: Sendable, Hashable { kind: SessionKind, start: Date }   // start в UTC
public enum SessionKind: String, Sendable { fp1, fp2, fp3, sprintQualifying, sprint, qualifying, race }
public struct RaceResult: Sendable, Hashable { position, driver, constructor, points, status, time? }
public struct DriverStanding ... ; public struct ConstructorStanding ...

public enum F1Error: Error, Sendable {
  case network(underlying: any Error)
  case http(status: Int, body: String?)
  case rateLimited(retryAfter: Duration?)
  case decoding(path: String, underlying: any Error)
  case notFound
}

public protocol HTTPClient: Sendable {
  func send(_ request: URLRequest) async throws -> (Data, HTTPURLResponse)
}

public struct Configuration: Sendable {
  public var baseURL: URL            // по умолчанию Jolpica
  public var retry: RetryPolicy      // попытки, база задержки, множитель
  public var rateLimit: RateLimit    // запросов в секунду, всплеск
  public var interceptors: [any Interceptor]
  public var sleeper: any Sleeper    // по умолчанию Task.sleep
}
```

## Источник данных

Jolpica, совместимый с Ergast: `https://api.jolpi.ca/ergast/f1/`. Эндпоинты: `/{season}.json` расписание, `/{season}/{round}/results.json`, `/{season}/driverStandings.json`, `/{season}/constructorStandings.json`. Формат: `MRData.RaceTable.Races[]` с полями `FirstPractice`, `Qualifying`, `Sprint` и датой-временем в UTC.

Лимиты Jolpica на момент написания: около четырёх запросов в секунду и несколько сотен в час без ключа. Точные числа проверить и записать в `NOTES.md`; `RateLimit.default` выставить с запасом.

Запасной источник: OpenF1, `https://api.openf1.org/v1/sessions?year=2026`. Другой формат. В первой версии не реализуется, но `HTTPClient` и `Endpoint` спроектированы так, чтобы второй провайдер добавлялся без изменения публичных моделей.

## Поведение

- Повторы: на 5xx и 429 и на сетевые ошибки, до трёх попыток, задержка 0,5 секунды с удвоением. При 429 с `Retry-After` ждать его. На 4xx кроме 429 не повторять.
- Ограничение частоты: перехватчик держит скользящее окно и задерживает запрос, если лимит исчерпан.
- Логирование: перехватчик пишет метод, URL, статус, длительность через переданное замыкание. По умолчанию выключен.
- `nextSession`: берёт расписание, собирает все сессии всех гонок, возвращает первую с `start > date`. Нет ни одной: `nil`.
- Даты: строки Ergast `2026-09-20` и `13:00:00Z` собираются в `Date` в UTC. Локальное время это забота клиента библиотеки.

## Команда `f1`

```
f1 next [--season 2026]            следующая сессия в локальном времени и через сколько
f1 schedule [--season 2026]        таблица уикендов
f1 standings [--season 2026] [--constructors]
```

Ошибки печатаются человеческим языком, код выхода 1.

## Границы

- Сезон ещё не начался или уже закончился: `nextSession` даёт `nil`, команда печатает «сезон окончен».
- Гонка без спринта: массив сессий короче, порядок по времени.
- Ответ с неожиданным полем: игнорируется. С отсутствующим обязательным: `decoding` с путём.
- 429 без `Retry-After`: экспоненциальная задержка.
- Сеть недоступна: `network` после трёх попыток.

## Не делаем

- Кэш ответов. Это забота приложения.
- Живой тайминг.
- Аутентификацию.
- Исторические данные сверх того, что отдаёт API по тем же эндпоинтам.

# Архитектура

## Коротко

Три слоя внутри библиотеки. Транспорт: `HTTPClient`, `URLSessionHTTPClient`, `Interceptor`, `RetryPolicy`, `RateLimit`, `Sleeper`. Описание: `Endpoint<Response>` с путём, параметрами и декодером. Домен: DTO формата Ergast, маппинг в публичные модели, `F1Client` с методами. Исполняемый таргет зависит от библиотеки и печатает.

## Почему так

Транспорт не знает об F1: его можно вынести в отдельный пакет для других проектов, и это проверка, что границы честные. `Endpoint` с дженерик-ответом это единственное место, где путь и тип результата связаны, поэтому новый метод клиента это одна структура и одна строка. Протоколы для сети и сна нужны не ради абстракции, а ради тестов, которые проходят за миллисекунды и без сети.

## Структура папок

```
f1-kit/
├── Package.swift                     targets: F1Kit, F1KitTests, f1
├── Sources/
│   ├── F1Kit/
│   │   ├── F1Kit.docc/               Overview.md, туториал
│   │   ├── Transport/
│   │   │   ├── HTTPClient.swift      протокол
│   │   │   ├── URLSessionHTTPClient.swift
│   │   │   ├── Interceptor.swift     протокол: prepare(request), didReceive(response)
│   │   │   ├── LoggingInterceptor.swift
│   │   │   ├── RateLimitInterceptor.swift
│   │   │   ├── RetryPolicy.swift
│   │   │   ├── Sleeper.swift         протокол и TaskSleeper
│   │   │   └── Transport.swift       собирает: перехватчики, повторы, отправка
│   │   ├── Endpoints/
│   │   │   ├── Endpoint.swift        struct Endpoint<Response: Decodable>
│   │   │   └── ErgastEndpoints.swift schedule, results, standings
│   │   ├── DTO/
│   │   │   ├── ErgastEnvelope.swift  MRData
│   │   │   ├── RaceDTO.swift
│   │   │   ├── ResultDTO.swift
│   │   │   └── StandingsDTO.swift
│   │   ├── Models/
│   │   │   ├── Race.swift, Session.swift, RaceResult.swift, Standings.swift
│   │   │   └── F1Error.swift
│   │   ├── Mapping/
│   │   │   └── ErgastMapper.swift    DTO → модели, разбор дат
│   │   ├── Configuration.swift
│   │   └── F1Client.swift
│   └── f1/
│       ├── F1Command.swift           ArgumentParser, подкоманды
│       └── Formatting.swift          таблицы, локальное время
├── Tests/
│   └── F1KitTests/
│       ├── Fixtures/                 записанные JSON: schedule-2025.json и другие
│       ├── Support/
│       │   ├── URLProtocolStub.swift
│       │   ├── RecordingHTTPClient.swift
│       │   └── InstantSleeper.swift
│       ├── TransportTests.swift      повторы, 429, перехватчики, лимит
│       ├── MappingTests.swift        даты, спринт, отсутствие полей
│       ├── F1ClientTests.swift       методы на фикстурах через URLProtocol
│       └── NextSessionTests.swift
├── Dockerfile.test                   swift:6 образ, swift test
└── .github/workflows/ci.yml
```

## Компоненты

**Endpoint.** `struct Endpoint<Response: Decodable & Sendable> { path: String; query: [URLQueryItem]; decode: (Data) throws -> Response }`. Фабрики в `ErgastEndpoints`: `static func schedule(season:) -> Endpoint<ErgastEnvelope<RaceTableDTO>>`.

**Transport.** Внутренний актор или структура: строит `URLRequest` из `Endpoint` и `baseURL`, прогоняет через `interceptors.prepare`, отправляет через `HTTPClient` с повторами по `RetryPolicy` и `Sleeper`, прогоняет ответ через `interceptors.didReceive`, переводит статусы в `F1Error`, декодирует. Единственное место с циклом повторов.

**RateLimitInterceptor.** Актор со скользящим окном временных меток. `prepare` ждёт через `Sleeper`, если окно полное. Тестируется с `InstantSleeper`, который записывает запрошенные задержки.

**RetryPolicy.** Структура: `maxAttempts`, `baseDelay`, `multiplier`, `func delay(attempt:retryAfter:) -> Duration?`. Чистая, тестируется таблицей.

**Sleeper.** `protocol Sleeper: Sendable { func sleep(for: Duration) async throws }`. `TaskSleeper` по умолчанию, `InstantSleeper` в тестах.

**ErgastMapper.** DTO в модели. Разбор дат через `ISO8601` без локали. Отсутствующие сессии пропускаются. Порядок сессий по времени.

**F1Client.** Публичная точка. Каждый метод: `try await transport.send(ErgastEndpoints.x(...))`, затем маппинг. `nextSession` через `schedule` и фильтр.

**URLSessionHTTPClient.** Принимает `URLSessionConfiguration` в init, чтобы тесты подставили `URLProtocolStub`. Проверяет, что ответ `HTTPURLResponse`.

**Команда f1.** Три подкоманды, форматирование в локальное время через `Date.FormatStyle`, таблица выравниванием.

## Поток данных

```
client.schedule(2026)
   ──▶ Endpoint.schedule(2026)
   ──▶ Transport.send
         ├──▶ interceptors.prepare (rateLimit ждёт при необходимости)
         ├──▶ httpClient.send ──▶ (Data, HTTPURLResponse)
         ├──▶ статус: 2xx → декодировать; 429/5xx → sleeper.sleep(delay) → повтор; 4xx → F1Error.http
         └──▶ interceptors.didReceive
   ──▶ ErgastMapper.races(from: envelope)
   ──▶ [Race]
```

## Правила

- `Transport/` и `Endpoints/` не знают о F1: ни одного упоминания гонок. Проверять на ревью.
- `public` только в `Models/`, `F1Client`, `Configuration`, `HTTPClient`, `Interceptor`, `Sleeper`, `F1Error`. DTO и `Transport` внутренние.
- Ни одного `Task.sleep`, `URLSession.shared`, `Date()` в библиотеке. Время приходит параметром, сон через протокол, сеть через протокол.
- Модели `Sendable`, `Hashable`, без классов.
- Ошибка декодирования несёт путь ключа из `DecodingError.Context.codingPath`.
- Добавление метода клиента не меняет существующие сигнатуры. Ломающее изменение это новая мажорная версия.

## Что здесь тестируемо

Всё, кроме `URLSessionHTTPClient` вживую. `RetryPolicy` таблицей. `Transport` с `RecordingHTTPClient`, который отдаёт очередь ответов, и `InstantSleeper`: три попытки на 500, `Retry-After` соблюдён, 404 без повтора. `RateLimitInterceptor` с записью задержек. `ErgastMapper` на фикстурах: сезон со спринтами, без, гонка без времени. `F1Client` через `URLProtocolStub` целиком. `nextSession` с фиксированной датой. Команда `f1` не тестируется юнитами, только дымом в CI с сетью, и это записано.

## Где ожидать боль

- Формат Ergast: строки в числах, вложенность `MRData.RaceTable.Races`, необязательные `Sprint`. Записать фикстуры с реального API до написания DTO.
- Linux: `URLProtocol` в swift-corelibs-foundation работает не полностью. Если тесты с ним падают на Linux, оставить их только на macOS с `#if canImport(Darwin)` и тестировать транспорт через `RecordingHTTPClient` везде. Записать.
- `Duration` и `Date`: два типа времени. Задержки в `Duration`, моменты в `Date`.
- Актор для лимита и `Sendable` перехватчики: цепочка перехватчиков должна быть `[any Interceptor]` с `Sendable`.
- Лимиты Jolpica меняются. Значения по умолчанию с запасом и возможность переопределить.
- DocC собирается долго и ругается на каждый недокументированный `public`. Это и нужно.

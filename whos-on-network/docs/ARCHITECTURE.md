# Архитектура

## Коротко

AppKit с MVVM на Combine, как в полигоне. `BonjourScanner` это актор, владеющий браузерами и соединениями, отдаёт `AsyncStream<[Device]>`. `DeviceMerger` и `ExpiryPolicy` чистые: собирают сервисы в устройства и решают, кто устарел. `ServiceCatalog` знает типы и значки. `AddressResolver` разрешает адреса по запросу. `MainWindowController` связывает список и детали.

## Почему так

Сеть асинхронна и шумна: одно устройство объявляет десять сервисов, каждый приходит отдельным событием. Если интерфейс слушает браузеры напрямую, он утонет. Поэтому один актор собирает всё в снимок и отдаёт целиком раз в полсекунды, а интерфейс рисует снимок. Слияние и истечение отделены от сети, чтобы их тестировать на записанных событиях.

## Структура папок

```
WhosOnNetwork/
├── App/
│   ├── AppDelegate.swift
│   ├── MainWindowController.swift        split view, тулбар, связь панелей
│   ├── Info.plist                        NSLocalNetworkUsageDescription, NSBonjourServices
│   └── WhosOnNetwork.entitlements        network.client
├── Discovery/
│   ├── ServiceCatalog.swift              типы, имена, значки, эвристика
│   ├── DiscoveredService.swift           name, type, domain, txt, seenAt
│   ├── Device.swift                      key, name, model, services, addresses, lastSeen, status
│   ├── DeviceMerger.swift                [DiscoveredService] → [Device]
│   ├── ExpiryPolicy.swift                статусы по времени
│   ├── AddressResolver.swift             NWConnection → адреса
│   └── BonjourScanner.swift              актор: браузеры, снимки, поток
├── Screens/
│   ├── DeviceList/
│   │   ├── DeviceListViewController.swift
│   │   └── DeviceListViewModel.swift
│   └── DeviceDetail/
│       ├── DeviceDetailViewController.swift
│       └── DeviceDetailViewModel.swift
├── Views/
│   └── DeviceCellView.swift
└── WhosOnNetworkTests/
    ├── DeviceMergerTests.swift
    ├── ExpiryPolicyTests.swift
    └── ServiceCatalogTests.swift
```

## Компоненты

**BonjourScanner.** Актор. `start()`: мета-браузер типов и браузеры по известному списку. На каждое событие браузера обновляет словарь сервисов с `seenAt`. Раз в 500 мс: `DeviceMerger.merge`, `ExpiryPolicy.apply`, `continuation.yield(devices)`. Для новых устройств запрашивает `AddressResolver` не чаще раза в минуту. `stop()`: отмена всех. Состояние разрешения из `NWBrowser.State.waiting` с ошибкой политики.

**DeviceMerger.** Чистая: группирует сервисы по ключу, вычисляет имя, модель из TXT, набор типов. Тесты на записанных наборах.

**ExpiryPolicy.** Чистая: по `lastSeen` и `now` ставит `active`, `stale`, `gone`. Тесты.

**AddressResolver.** `NWConnection` к endpoint сервиса, ждёт `ready`, читает `currentPath?.remoteEndpoint`, отменяет. Таймаут пять секунд. Возвращает адреса или пусто.

**ServiceCatalog.** Словарь тип → имя и значок, эвристика устройства по набору типов. Тесты на эвристику.

**DeviceListViewModel.** Подписан на поток снимков через `Task`, `@Published state` с отфильтрованным списком, `filterText`, `selectedKey`. Кнопка обновить зовёт `scanner.restart()`.

**DeviceDetailViewModel.** `@Published device`, получает из оконного контроллера по выбору.

**MainWindowController.** Как в полигоне: связывает выбор в списке с деталями, фильтр из тулбара с моделью списка. Закрытие окна зовёт `scanner.stop()`.

## Поток данных

```
NWBrowser события ──▶ BonjourScanner (актор) ──▶ словарь сервисов
                                                     │ каждые 500 мс
                                                     ├──▶ DeviceMerger.merge
                                                     ├──▶ ExpiryPolicy.apply
                                                     └──▶ stream.yield([Device])
                                                              │
DeviceListViewModel ◀── for await ◀──────────────────────────┘
      └──▶ $state ──▶ DeviceListViewController
выбор ──▶ MainWindowController ──▶ DeviceDetailViewModel.device
```

## Правила

- `NWBrowser` и `NWConnection` только внутри `Discovery/BonjourScanner` и `AddressResolver`.
- Интерфейс получает только снимки `[Device]`.
- `DeviceMerger`, `ExpiryPolicy`, `ServiceCatalog` без `import Network`.
- Каждый созданный браузер и соединение имеет путь к `cancel()`, проверяется в `stop()` и в `deinit` с print на время разработки.
- Разрешение адресов не чаще раза в минуту на устройство, не для всех сразу.
- Типы сервисов и их имена только в `ServiceCatalog`.

## Что здесь тестируемо

`DeviceMerger` на записанных событиях: десять сервисов одного устройства дают одно устройство; два с одинаковым именем и разными хостами дают два. `ExpiryPolicy` по времени. `ServiceCatalog` эвристика. `BonjourScanner` руками в сети, и это записано.

## Где ожидать боль

- `NSBonjourServices` в Info.plist: без перечисления типов браузер получает `waiting` с ошибкой политики и молчит. Мета-запрос находит типы, которых нет в списке, и они не будут разрешены. Записать это как ограничение.
- Разрешение на локальную сеть на macOS появляется при первом браузере. Отказ виден только по состоянию `waiting`.
- Имя хоста: `NWBrowser` отдаёт имя сервиса, не хоста. Хост виден после разрешения через соединение. До этого ключ по имени сервиса, потом слияние по хосту, и устройство может «переехать» между ключами. `DeviceMerger` должен уметь объединять по любому совпадению.
- TXT-записи приходят в `NWBrowser.Result.Metadata.bonjour`, не всегда.
- Bluetooth и AWDL интерфейсы дают дубли. Ограничить браузер параметрами `requiredInterfaceType = .wifi` или фильтровать по интерфейсу пути.
- `NWConnection` к `_sleep-proxy._udp` не имеет смысла. Разрешать через TCP-сервисы.

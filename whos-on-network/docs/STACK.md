# Стек

Версии на момент написания, сентябрь 2026. При старте проверить и поправить.

## Основа

| Что | Зачем здесь | Что хочу понять |
|---|---|---|
| Network.framework | Обнаружение | `NWBrowser`, дескрипторы, состояния, `browseResultsChangedHandler`, параметры интерфейса |
| Bonjour, DNS-SD | Протокол | Типы сервисов, мета-запрос, TXT-записи, что объявляют устройства Apple |
| `NWConnection` | Адреса | Путь, удалённый endpoint, отмена, таймаут |
| Приватность локальной сети | Разрешение | `NSLocalNetworkUsageDescription`, `NSBonjourServices`, состояние `waiting` |
| AppKit, `NSSplitViewController` | Окно | Как в полигоне |
| Combine и `AsyncStream` | Связь | Актор отдаёт поток, ViewModel публикует |
| Акторы | Изоляция сети | Актор, владеющий колбэками из своей очереди |

## Чего нет и почему

- Сканирования портов. Принципиально.
- `NetService` и `NetServiceBrowser`: устарели, Network.framework вместо них.

## Что почитать и посмотреть

**Перед фазой 1:**
- WWDC18 «Introducing Network.framework», WWDC19 «Advances in Networking, Part 2» про Bonjour.
- Документация: `NWBrowser`, «Discovering services with Bonjour» в Network.framework.
- Документация: «Bonjour Overview», раздел про типы сервисов и TXT.

**Перед фазой 2:**
- Документация: «Local network privacy», `NSBonjourServices`.

**Перед фазой 3:**
- Документация: `NWConnection`, `NWPath.remoteEndpoint`.

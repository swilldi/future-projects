# Стек

Версии на момент написания, сентябрь 2026. При старте проверить и поправить.

## Основа

| Что | Зачем здесь | Что хочу понять |
|---|---|---|
| Go 1.23 или новее | Язык | Пакеты, интерфейсы, ошибки, `defer`, горутины, каналы на уровне «прочитал tour и пишу» |
| `net/http` с шаблонами | Роутинг | `GET /path/{id}`, `r.PathValue`, middleware как функции, таймауты сервера |
| `database/sql` и `pgx/stdlib` | Postgres | Пул, `QueryRowContext`, `ExecContext`, сканирование, ошибки драйвера |
| `goose` | Миграции | SQL-файлы из `embed.FS`, `Up`, версии |
| `embed` | Миграции в бинарнике | Директива `//go:embed` |
| `log/slog` | Логи | JSON-обработчик, атрибуты, уровни, логгер в контексте |
| `context` | Отмена | Дерево контекстов, `WithTimeout`, отмена от клиента до базы |
| `os/signal` | Завершение | `NotifyContext`, `Shutdown` |
| `testing`, `httptest` | Тесты | Табличные тесты, `t.Run`, `t.Parallel`, `NewRecorder`, теги сборки |
| `go vet`, `golangci-lint` | Качество | Что ловит каждый, конфиг с разумным набором |
| Docker многоэтапный | Образ | `CGO_ENABLED=0`, `-ldflags="-s -w"`, `distroless`, пользователь |
| Docker Compose | Окружение | healthcheck, `depends_on` с условиями, сети, тома |
| GitHub Actions | CI | `setup-go`, кэш модулей, сервис Postgres в job для интеграционных |

## Чего нет и почему

- Роутеров и фреймворков: стандартная библиотека умеет достаточно.
- ORM: SQL руками, чтобы видеть запросы.
- Логгеров сверх `slog`: он в стандартной библиотеке с 1.21.

## Что почитать и посмотреть

**До фазы 0:**
- A Tour of Go целиком. Один вечер.
- Effective Go. Прочитать, вернуться через неделю.

**Перед фазой 1:**
- Go blog: «Error handling and Go», «Working with Errors in Go 1.13».
- go.dev/doc/modules/layout, про раскладку модуля.

**Перед фазой 2:**
- Go blog: «Routing Enhancements for Go 1.22».
- Документация `net/http`, раздел про `ServeMux` и шаблоны.

**Перед фазой 3:**
- README `pgx`, раздел про `stdlib`. README `goose`.
- Документация `database/sql`, статья «Accessing a relational database» на go.dev.

**Перед фазой 6:**
- Документация Docker: «Multi-stage builds». Документация Compose: `depends_on` с `condition`.

**Перед фазой 7:**
- Документация `golangci-lint`, выбрать линтеры. Go blog про `go vet` и `-race`.

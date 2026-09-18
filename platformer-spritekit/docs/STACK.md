# Стек

Версии на момент написания, сентябрь 2026. При старте проверить и поправить.

## Основа

| Что | Зачем здесь | Что хочу понять |
|---|---|---|
| SpriteKit | Сцена | Узлы, `update`, физика, категории и маски, `SKTileMapNode`, камера, `SKAction`, звуки |
| GameplayKit | Структура | `GKEntity`, `GKComponent`, `GKStateMachine`, зачем ECS |
| GameController | Геймпад | Уведомления подключения, `extendedGamepad`, обработчики |
| `SpriteView` | В SwiftUI | Опции, пауза, размер |
| Математика платформера | Ощущение | Переменный прыжок, coyote time, jump buffer, разная гравитация |
| Свободные ассеты | Графика и звук | Kenney или аналог, лицензия CC0, атласы |
| Swift Testing | Чистые типы | Таблицы для движения |

## Чего нет и почему

- Unity, Godot. Цель системные фреймворки.
- Физики SpriteKit для движения героя. Только контакты.

## Что почитать и посмотреть

**Перед фазой 1:**
- Документация: «SpriteKit», «Building your scene», `SKTileMapNode`.
- WWDC14 «Best Practices for Building SpriteKit Games», старая, но база.

**Перед фазой 2:**
- Статья «Platformer controls: how to avoid limpness and rigidity» или любой разбор coyote time и jump buffering. GDC-доклад «Math for Game Programmers: Building a Better Jump».

**Перед фазой 3:**
- WWDC15 «Introducing GameplayKit», раздел про сущности, компоненты и машины состояний.

**Перед фазой 7:**
- Документация: «GameController», «Supporting game controllers».

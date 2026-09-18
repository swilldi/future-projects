# Архитектура

## Коротко

`GameScene` на SpriteKit владеет узлами и физикой. Сущности и компоненты через GameplayKit: герой это `GKEntity` с компонентами движения, ввода, анимации, здоровья. `LevelLoader` превращает текст в модель уровня без SpriteKit. `InputController` сводит экранные кнопки и геймпад в `InputState`. `PlayerMovement` это чистая математика прыжка и бега над `InputState` и `deltaTime`. `GameStateMachine` на `GKStateMachine`. SwiftUI-обёртка через `SpriteView`.

## Почему так

Игровой цикл тянет всё в одну сцену на тысячу строк. Компоненты режут это по ответственности, а чистые типы для уровня, ввода и движения позволяют тестировать «ощущение прыжка» числами: сколько кадров coyote time, какая высота при отпускании. Физика SpriteKit используется для столкновений и контактов, а не для движения героя: движение считается руками, иначе платформер ощущается как желе.

## Структура папок

```
Platformer/
├── App/
│   ├── PlatformerApp.swift
│   └── GameView.swift                  SpriteView, экран победы, кнопки
├── Level/
│   ├── LevelModel.swift                тайлы, объекты, размеры
│   ├── LevelLoader.swift               текст → модель, ошибки
│   └── Resources/level1.txt
├── Input/
│   ├── InputState.swift
│   ├── InputController.swift           экранные кнопки + GameController → InputState
│   └── TouchControlsView.swift
├── Gameplay/
│   ├── PhysicsCategory.swift           OptionSet
│   ├── PlayerMovement.swift            чистая математика бега и прыжка
│   ├── Entities/
│   │   ├── PlayerEntity.swift
│   │   ├── EnemyEntity.swift
│   │   └── PlatformEntity.swift
│   ├── Components/
│   │   ├── SpriteComponent.swift
│   │   ├── PlayerControlComponent.swift  применяет PlayerMovement к узлу
│   │   ├── PatrolComponent.swift
│   │   └── MovingPlatformComponent.swift
│   └── GameStateMachine.swift          GKStateMachine и состояния
├── Scene/
│   ├── GameScene.swift                 сборка из LevelModel, update, контакты
│   ├── CameraController.swift
│   └── HUD.swift
├── Assets/                             спрайты, звуки, лицензии
└── PlatformerTests/
    ├── LevelLoaderTests.swift
    ├── PlayerMovementTests.swift
    ├── InputControllerTests.swift
    └── GameStateMachineTests.swift
```

## Компоненты

**LevelLoader.** Строки в `LevelModel`: сетка тайлов, список объектов с координатами, старт, флаг, размер. Ошибки: разная длина строк, нет старта, неизвестный символ. Чистый.

**InputController.** Держит `InputState`. Экранные кнопки пишут в него через замыкания. `GCController` подключается по уведомлениям, читает `extendedGamepad` через `valueChangedHandler`. `jumpJustPressed` живёт один кадр: сбрасывается в `update`.

**PlayerMovement.** `struct` с параметрами и `func step(state: inout PlayerState, input: InputState, dt: TimeInterval) -> PlayerState`, где состояние это позиция, скорость, `isGrounded`, таймеры coyote и буфера. Вся математика прыжка здесь. Тестируется: высота прыжка при удержании и без, coyote срабатывает на 5 кадров, буфер срабатывает.

**PlayerControlComponent.** Каждый кадр берёт `InputState`, зовёт `PlayerMovement.step`, применяет позицию к узлу. `isGrounded` обновляется по контактам с землёй через счётчик касаний.

**PhysicsCategory.** `player`, `ground`, `platform`, `movingPlatform`, `enemy`, `coin`, `flag`, `hazard`. Маски контактов заданы в одном месте при сборке.

**GameScene.** `didMove`: строит узлы из `LevelModel`, `SKTileMapNode` для земли, сущности для объектов. `update`: `deltaTime` с ограничением, обновляет компоненты, камеру, сбрасывает `jumpJustPressed`. `didBegin(contact)`: разбирает пары категорий и зовёт обработчики: монета, враг сверху или сбоку, флаг, пропасть. Пропасть это узел-сенсор под уровнем.

**CameraController.** Мёртвая зона, сглаживание, ограничение по границам уровня.

**GameStateMachine.** `GKStateMachine` с `Playing`, `Dying`, `Respawning`, `Won`, `Paused`. Переходы разрешены явно. Сцена спрашивает состояние, прежде чем обрабатывать ввод.

**MovingPlatformComponent.** `SKAction` туда-сюда между границами; герой, стоящий на платформе, получает её смещение за кадр.

## Поток данных

```
кнопка/геймпад ──▶ InputController.state
                         │
update(dt) ──▶ PlayerControlComponent ──▶ PlayerMovement.step ──▶ node.position
           ──▶ PatrolComponent, MovingPlatformComponent
           ──▶ CameraController.follow(player)
           ──▶ input.endFrame()

didBegin(contact) ──▶ по категориям ──▶ coin: +1, remove
                                     ──▶ enemy: сверху? remove : stateMachine.enter(Dying)
                                     ──▶ flag: enter(Won)
                                     ──▶ hazard: enter(Dying)
```

## Правила

- `Level/`, `Input/InputState`, `PlayerMovement`, `GameStateMachine` без `import SpriteKit`. GameplayKit допустим в машине состояний.
- Движение героя считается `PlayerMovement`, не физическим телом. Тело героя кинематическое, только для контактов.
- Ссылки на узлы держатся в компонентах, `childNode(withName:)` в `update` запрещён.
- `deltaTime` ограничен 1/30, чтобы после паузы герой не улетал.
- Маски категорий только из `PhysicsCategory`.
- Контакты обрабатываются в `didBegin`, изменения узлов там же или отложенно, но не в физическом шаге.

## Что здесь тестируемо

`LevelLoader` на валидных и битых картах. `PlayerMovement` числами: высота при удержании, при отпускании, coyote, буфер, торможение. `InputController` со стабом геймпада: `jumpJustPressed` один кадр. `GameStateMachine`: разрешённые и запрещённые переходы. Сцена, камера и контакты руками.

## Где ожидать боль

- `isGrounded` по контактам: `didEnd` иногда приходит раньше `didBegin` для соседнего тайла. Счётчик касаний вместо флага.
- Платформа `=`, проходимая снизу: маска столкновения меняется в зависимости от направления скорости героя.
- Движущаяся платформа и кинематическое тело: без ручного переноса смещения герой скользит.
- Геймпад: `valueChangedHandler` зовётся на своём потоке. Записывать в состояние на главном.
- Звук через `SKAction.playSoundFileNamed` создаёт задержку при первом воспроизведении. Прогреть при старте.
- `SpriteView` в SwiftUI и пауза: `isPaused` при уходе в фон, иначе физика скачет при возврате.
- Ассеты: размер тайла и размер спрайта должны совпадать, иначе `SKTileMapNode` ведёт себя странно.

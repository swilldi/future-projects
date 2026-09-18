# Архитектура

## Коротко

`MetalView` оборачивает `MTKView` в SwiftUI. `Renderer` реализует `MTKViewDelegate`: владеет устройством, очередью команд, конвейерами, буферами и в `draw(in:)` собирает кадр. `WaveSimulator` и `ParticleSystem` в `Simulation/` считают состояние на CPU, чистые, тестируются. `Scene` держит параметры со слайдеров и точки ряби. Шейдеры в `Shaders.metal`, структуры в `ShaderTypes.h`.

## Почему так

Metal это про то, чтобы CPU готовил данные, а GPU рисовал, и граница между ними должна быть видна в коде. Симуляция отдельно от рендера, чтобы её тестировать и чтобы потом перенести в шейдер, не трогая остальное. Тройная буферизация это не оптимизация, а необходимость: без неё CPU перезапишет буфер, который GPU ещё читает.

## Структура папок

```
WaveSandbox/
├── App/
│   └── WaveSandboxApp.swift
├── Shaders/
│   ├── ShaderTypes.h                общие структуры
│   ├── Lines.metal                  вершинный и фрагментный для линий
│   └── Particles.metal              инстансинг
├── Rendering/
│   ├── MetalView.swift              UIViewRepresentable
│   ├── Renderer.swift               MTKViewDelegate, кадр
│   ├── Pipelines.swift              сборка двух конвейеров
│   ├── FrameBuffers.swift           три набора буферов, семафор
│   └── GPUStats.swift               время кадра, FPS
├── Simulation/                      без import Metal
│   ├── WaveSimulator.swift          y(x, line, time), рябь
│   ├── ParticleSystem.swift         update(dt), respawn
│   └── ParticleStyle.swift
├── Scene/
│   └── SceneState.swift             @Observable, параметры, касания
├── Features/
│   └── ControlsView.swift
└── WaveSandboxTests/
    ├── WaveSimulatorTests.swift
    └── ParticleSystemTests.swift
```

## Компоненты

**Renderer.** Создаётся один раз с `MTLDevice`. В init: очередь команд, конвейеры через `Pipelines`, буферы через `FrameBuffers`. В `draw(in:)`: `semaphore.wait`, взять текущий набор буферов, обновить юниформы, попросить `ParticleSystem.update(dt)` и записать экземпляры в буфер, в первой версии попросить `WaveSimulator` вершины линий и записать, создать `commandBuffer`, `renderEncoder` с дескриптором из вью, два `draw`, `addCompletedHandler { semaphore.signal() }`, `present`, `commit`. Индекс набора по кругу.

**FrameBuffers.** Три набора: буфер юниформов, буфер вершин линий, буфер экземпляров частиц. Размер под максимум: 8 линий по 256 сегментов, 5000 частиц. `storageMode` shared.

**Pipelines.** Два `MTLRenderPipelineState` из функций библиотеки по умолчанию. MSAA согласован с вью. Смешивание для частиц: альфа.

**WaveSimulator.** `func y(x: Float, line: Int, time: Float, params) -> Float` и `func ripple(...)`. Чистая математика. Во второй версии то же самое переписывается на MSL в вершинном шейдере, и Swift-версия остаётся для тестов и сравнения.

**ParticleSystem.** Массив структур, `update(dt:style:bounds:)`, возрождение. Детерминирован при заданном генераторе. Стретч: перенос в вычислительный шейдер.

**SceneState.** `@Observable`. Параметры слайдеров, стиль, до четырёх точек ряби с временем. `Renderer` читает его на главном потоке в начале `draw`: `MTKView` по умолчанию зовёт делегата на главном потоке, поэтому изоляция сходится.

**GPUStats.** `commandBuffer.gpuStartTime` и `gpuEndTime` в обработчике завершения, скользящее среднее. FPS по `CACurrentMediaTime` между кадрами.

## Поток данных

```
слайдер ──▶ SceneState.amplitude
касание ──▶ SceneState.ripples.append(...)
                    │
draw(in:) ──▶ semaphore.wait
          ──▶ uniforms[i] = Uniforms(из SceneState, time)
          ──▶ ParticleSystem.update(dt) ──▶ instances[i]
          ──▶ (v1) WaveSimulator ──▶ lineVertices[i]
          ──▶ encoder.draw(lines), encoder.draw(particles, instanceCount)
          ──▶ completed { semaphore.signal }
          ──▶ present, commit
```

## Правила

- `Simulation/` без `import Metal` и `MetalKit`.
- Структуры для GPU только из `ShaderTypes.h`.
- В `draw(in:)` нет аллокаций: буферы готовы, массивы не создаются, `memcpy` в `contents()`.
- Семафор на три, ожидание строго до записи в буферы, сигнал строго в обработчике завершения.
- Один `MTLDevice`, одна очередь, конвейеры собраны в init.
- Изменение числа частиц не пересоздаёт буферы: буфер под максимум, рисуется `instanceCount`.

## Что здесь тестируемо

`WaveSimulator`: форма периодична по времени, рябь затухает до нуля за секунду, амплитуда ограничена. `ParticleSystem`: частицы остаются в границах после возрождения, возраст растёт, стиль влияет на направление. `Renderer` не тестируется юнитами: проверяется захватом кадра и счётчиками, и это записано.

## Где ожидать боль

- Выравнивание структур: `float2` в MSL выровнен на 8, `float4` на 16. Swift-структура с теми же полями может иметь другую раскладку. Поэтому общий заголовок.
- Семафор: перепутать порядок ожидания и сигнала значит либо гонка, либо вечное ожидание.
- Система координат: NDC от -1 до 1, ось Y вверх. Первые линии окажутся вверх ногами.
- Толстые линии: нужны нормали к сегментам, на изгибах стыки. Достаточно miter без сглаживания углов.
- MSAA: дескриптор конвейера, вью и проход должны совпадать по количеству сэмплов, иначе краш при запуске.
- 120 Гц: `preferredFramesPerSecond` это просьба, и в Info.plist нужен `CADisableMinimumFrameDurationOnPhone`. Без него 60.
- Захват кадра работает только на устройстве с включённым в схеме GPU Frame Capture.

# Стек

Версии на момент написания, сентябрь 2026. При старте проверить и поправить.

## Основа

| Что | Зачем здесь | Что хочу понять |
|---|---|---|
| Metal, MetalKit | Рендер | Устройство, очередь, командный буфер, энкодер, конвейер, дескриптор прохода |
| MSL | Шейдеры | Вершинный и фрагментный, атрибуты, `[[vertex_id]]`, `[[instance_id]]`, `[[stage_in]]`, буферы по индексам |
| Bridging header | Общие структуры | Как Swift и MSL видят одну структуру, выравнивание |
| Инстансинг | Частицы | `instanceCount`, буфер экземпляров, почему не 5000 вызовов draw |
| Тройная буферизация | Кадры в полёте | Семафор, три набора буферов, обработчик завершения |
| MSAA | Гладкие линии | Согласование сэмплов между вью, конвейером и проходом |
| GPU Frame Capture, счётчики | Отладка | Что видно в захвате, как читать время прохода |
| `UIViewRepresentable` | MTKView в SwiftUI | Координатор как делегат, обновление параметров |
| Вычислительный шейдер | Стретч | `[[thread_position_in_grid]]`, обновление частиц на GPU |

## Чего нет и почему

- SpriteKit, SceneKit, RealityKit. Цель ниже уровнем.
- Текстур. Не нужны для линий и точек, и это упрощает первый проход.

## Что почитать и посмотреть

**Перед фазой 0 и 1:**
- Документация Apple: «Using Metal to Draw a View's Contents», «Using a Render Pipeline to Render Primitives». Два примера, с них начинается всё.
- Warren Moore, «Metal by Example», первые главы: устройство, буферы, конвейер.

**Перед фазой 3:**
- Документация Apple: «Creating and Sampling Textures» не нужна, а вот «Managing groups of resources with argument buffers» пока тоже нет. Читать «Synchronizing CPU and GPU Work», это про тройную буферизацию.

**Перед фазой 5:**
- Документация: «Rendering many instances with instanced drawing» или аналогичный раздел про `instanceCount`.

**Перед фазой 7:**
- Документация Xcode: «Capturing a Metal workload in Xcode», «Analyzing your Metal workload».
- WWDC20 «Harness Apple GPUs with Metal», про тайловую архитектуру, чтобы понимать счётчики.

**Перед стретчем:**
- Документация: «Processing a texture in a compute function», как образец вычислительного шейдера, адаптировать под буфер частиц.

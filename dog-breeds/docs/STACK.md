# Стек

Версии на момент написания, сентябрь 2026. При старте проверить и поправить.

## Основа

| Что | Зачем здесь | Что хочу понять |
|---|---|---|
| CreateML фреймворк | Обучение скриптом | `MLImageClassifier`, параметры, аугментации, оценка, метаданные модели |
| CoreML | Модель в приложении | Загрузка, компиляция, `MLModelConfiguration`, что такое `.mlmodelc` |
| Vision | Распознавание | `VNCoreMLRequest`, `VNImageRequestHandler`, `VNClassificationObservation`, обрезка и ориентация |
| AVFoundation | Камера | Сессия, устройства, `AVCaptureVideoDataOutput`, очередь делегата, превью-слой, разрешение |
| `AsyncStream` с буферизацией | Кадры | `bufferingNewest`, почему старые кадры выбрасываются |
| SwiftData, `externalStorage` | Коллекция | Картинки вне базы |
| `PhotosPicker` | Галерея | Загрузка `Data` из выбора |
| `ProcessInfo.thermalState` | Нагрев | Реакция на тепловое состояние |
| Stanford Dogs, открытые наборы | Данные | Структура, лицензия, разбивка |

## Чего нет и почему

- Готовых моделей из интернета. Цель обучить самому.
- Детекции объектов. Классификации достаточно для первой версии.

## Что почитать и посмотреть

**Перед фазой 1:**
- Документация: «Creating an Image Classifier Model» в Create ML, «MLImageClassifier».
- WWDC19 «Training Object Detection Models in Create ML» не нужно; нужно WWDC18 «Introducing Create ML» для основ и WWDC21 «Build dynamic iOS apps with the Create ML framework» для скриптов.

**Перед фазой 3:**
- Документация: «Classifying Images with Vision and Core ML», пример Apple с той же задачей.

**Перед фазой 4:**
- Документация: «Setting Up a Capture Session», «Capturing Video Frames», `AVCaptureVideoDataOutputSampleBufferDelegate`.
- WWDC19 «Understanding Images in Vision Framework», про ориентацию.

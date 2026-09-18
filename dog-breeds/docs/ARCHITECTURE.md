# Архитектура

## Коротко

Две части. `Training/`: исполняемый Swift-пакет для macOS с двумя командами: подготовить данные и обучить, использует `CreateML`. Приложение: `CameraSession` владеет AVFoundation и отдаёт кадры, `Classifier` владеет моделью и Vision-запросом и отдаёт результаты, `Verdict` превращает сырые оценки в решение по порогам, `ClassificationSmoother` сглаживает, SwiftData хранит снимки, SwiftUI показывает.

## Почему так

Обучение это код, а не клики, иначе через месяц модель не воспроизвести. В приложении камера и модель это два независимых источника асинхронности на своих потоках, и между ними нужен узкий канал: кадр туда, результат обратно. Решение по порогам и сглаживание отделены от Vision, чтобы тестировать их числами.

## Структура папок

```
DogBreeds/
├── Training/                              отдельный Swift-пакет, macOS
│   ├── Package.swift
│   ├── Sources/train/
│   │   ├── Train.swift                    команды prepare и train
│   │   ├── Breeds.swift                   список 30 пород
│   │   ├── DatasetPreparer.swift          скачивание, отбор, разбивка
│   │   └── Trainer.swift                  MLImageClassifier, оценка, экспорт
│   ├── Tests/
│   │   └── DatasetPreparerTests.swift     разбивка детерминирована, по 100 на породу
│   └── data/                              в .gitignore
├── DogBreeds/
│   ├── App/
│   │   ├── DogBreedsApp.swift
│   │   └── AppDependencies.swift
│   ├── Resources/
│   │   └── DogBreeds.mlmodel              из Training
│   ├── Capture/
│   │   ├── CameraSession.swift            AVCaptureSession, вывод кадров в AsyncStream
│   │   ├── CameraPreview.swift            UIViewRepresentable с AVCaptureVideoPreviewLayer
│   │   └── FrameThrottle.swift            не чаще N мс, пропуск пока занято
│   ├── Classification/
│   │   ├── Classifier.swift               VNCoreMLRequest, один экземпляр
│   │   ├── Verdict.swift                  пороги → решение
│   │   ├── ClassificationSmoother.swift   два подряд
│   │   └── Thresholds.swift               константы
│   ├── Data/
│   │   ├── Sighting.swift                 @Model
│   │   └── SightingStore.swift
│   ├── Features/
│   │   ├── Camera/                        CameraView, CameraViewModel, ResultOverlay
│   │   ├── Collection/                    CollectionView, SightingDetailView
│   │   ├── Breeds/                        BreedsView
│   │   └── Gallery/                       PhotoClassifyView
│   └── DogBreedsTests/
│       ├── VerdictTests.swift
│       ├── SmootherTests.swift
│       ├── FrameThrottleTests.swift
│       └── ClassifierTests.swift          на пяти тестовых картинках из бандла тестов
```

## Компоненты

**DatasetPreparer.** Скачивает архив, распаковывает, выбирает породы по списку, берёт первые 100 по имени файла после перемешивания с сидом, раскладывает в `train/<breed>/` и `test/<breed>/`. Добавляет класс `other`. Детерминирован.

**Trainer.** `MLImageClassifier(trainingData: .labeledDirectories(at:), parameters: ...)` с аугментациями и сидом. `evaluation(on:)` для проверки. Печатает точность, матрицу ошибок для худших пар, пишет `.mlmodel` с метаданными: версия, дата, точность.

**CameraSession.** Класс, владеющий `AVCaptureSession`, входом и `AVCaptureVideoDataOutput`. Делегат на последовательной очереди складывает `CVPixelBuffer` с ориентацией в `AsyncStream` с политикой `bufferingNewest(1)`: старые кадры выбрасываются. `start()` и `stop()` с сессией на своей очереди. Разрешение запрашивает.

**FrameThrottle.** Чистая: по времени последней обработки и флагу «занято» решает, брать ли кадр. Тесты.

**Classifier.** Загружает `VNCoreMLModel` один раз, держит один `VNCoreMLRequest` с `imageCropAndScaleOption = .centerCrop`. `classify(pixelBuffer, orientation) async throws -> [BreedScore]` через `VNImageRequestHandler`. Работает вне главного актора. Второй метод для `CGImage` из галереи.

**Verdict.** Чистая: по оценкам и `Thresholds` отдаёт `notADog`, `unsure`, `breeds`. Тесты на границах.

**ClassificationSmoother.** Чистая: хранит предыдущую верхнюю породу, отдаёт результат только при совпадении два раза. Тесты.

**CameraViewModel.** `@Observable @MainActor`. Запускает цикл: `for await frame in session.frames`, `throttle.shouldProcess`, `classifier.classify`, `Verdict`, `smoother`, обновляет состояние. Следит за `ProcessInfo.thermalState`. `capture()` берёт последний кадр, конвертирует в JPEG, сохраняет `Sighting`.

**SightingStore.** SwiftData, `externalStorage` для картинки, запросы по породе и по дате.

## Поток данных

```
камера ──▶ AVCaptureVideoDataOutput (очередь делегата) ──▶ frames.yield(buffer)   bufferingNewest(1)
                                                                  │
CameraViewModel: for await frame ──▶ throttle? ──▶ classifier.classify (фон)
                                                        │
                                          [BreedScore] ──▶ Verdict ──▶ Smoother ──▶ state.overlay
снимок ──▶ последний кадр ──▶ JPEG ──▶ Sighting ──▶ SwiftData
```

## Правила

- `DispatchQueue` только в `CameraSession` для делегата и сессии, с комментарием.
- `Classifier` один на приложение, запрос создаётся один раз.
- `Verdict`, `Smoother`, `FrameThrottle` без Vision и AVFoundation.
- Ориентация кадра передаётся в Vision явно из ориентации устройства.
- В коллекцию сохраняется уменьшенный JPEG, не сырой буфер.
- Модель обучается только скриптом, файл модели с метаданными о версии.

## Что здесь тестируемо

`Training`: разбивка детерминирована и по 100 на породу. `Verdict`, `Smoother`, `FrameThrottle` таблицами. `Classifier` дымом: пять картинок в тестовом бандле дают ожидаемую верхнюю породу и «другое» для кота. `CameraSession` и живой цикл руками.

## Где ожидать боль

- Датасет большой, скачивание и распаковка долгие. Скрипт должен уметь пропускать готовые шаги.
- Обучение на 3000 снимках с аугментациями это десятки минут на ноутбуке. Сначала прогнать на 5 породах по 20.
- «Не собака» без класса «другое» не работает: модель обязана выбрать породу. Класс «другое» обязателен, и его данные должны быть разнообразными.
- Ориентация: `AVCaptureVideoDataOutput` отдаёт кадры в ландшафте. Без `CGImagePropertyOrientation` Vision видит собаку боком.
- `bufferingNewest(1)` обязателен: иначе очередь кадров растёт, память и задержка тоже.
- `CVPixelBuffer` не `Sendable`. Передавать через поток с обоснованием или конвертировать раньше.
- Нагрев: десять минут распознавания греют телефон. Реакция на `thermalState`.
- Лицензия Stanford Dogs некоммерческая. Приложение личное, не в стор.

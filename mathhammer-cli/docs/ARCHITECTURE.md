# Архитектура

## Коротко

Библиотека `MathhammerCore`: тип `Distribution` (вероятности по целым значениям) с операциями, парсер `DiceExpression`, стадии расчёта как функции от распределения к распределению, правила в отдельной папке, симулятор Монте-Карло, отчёт. Исполняемый таргет разбирает флаги, строит `Profile`, зовёт `Calculator`, печатает.

## Почему так

Вся сложность в математике, и она чистая. Если каждая стадия это функция `Distribution -> Distribution`, конвейер читается как правила игры, а тесты пишутся на каждую стадию отдельно. Монте-Карло не для скорости, а как независимая проверка точного расчёта: две реализации одного правила должны сойтись.

## Структура папок

```
mathhammer/
├── Package.swift                      MathhammerCore, mathhammer, тесты
├── Sources/
│   ├── MathhammerCore/
│   │   ├── Probability/
│   │   │   ├── Distribution.swift     [Double] по значениям 0…n, convolve, scale, expected, percentile
│   │   │   ├── Binomial.swift         биномиальное с параметрами n, p
│   │   │   └── Dice.swift             DiceExpression, парсер, распределение
│   │   ├── Rules/
│   │   │   ├── HitRules.swift         порог, модификаторы, перебросы, криты, Lethal, Sustained, Torrent
│   │   │   ├── WoundRules.swift       таблица S/T, Devastating
│   │   │   ├── SaveRules.swift        AP, инвульн, FNP
│   │   │   ├── DamageRules.swift      выражение урона, убитые модели
│   │   │   └── Blast.swift
│   │   ├── Profile.swift              атакующий и цель, Codable
│   │   ├── Calculator.swift           конвейер стадий → Report
│   │   ├── Simulator.swift            Монте-Карло с RandomNumberGenerator
│   │   └── Report.swift               ожидания, распределения, процентили
│   └── mathhammer/
│       ├── MathhammerCommand.swift    ArgumentParser
│       ├── ProfileLoader.swift        JSON и флаги
│       └── TableRenderer.swift
├── Tests/MathhammerCoreTests/
│   ├── DistributionTests.swift
│   ├── DiceTests.swift
│   ├── HitRulesTests.swift, WoundRulesTests.swift, SaveRulesTests.swift, DamageRulesTests.swift
│   ├── CalculatorTests.swift          десять эталонных профилей
│   └── SimulatorAgreementTests.swift  свойство сходимости
├── Dockerfile.test
└── .github/workflows/ci.yml
```

## Компоненты

**Distribution.** `struct Distribution { var probabilities: [Double] }`, индекс это значение. `convolve(_:)` для суммы независимых, `binomialCompound(trials: Distribution, successProbability: Double)` для «каждое из N событий успешно с p», `expected`, `probability(atLeast:)`, `percentile`. Инвариант: сумма единица с допуском, проверяется в тестах.

**DiceExpression.** `enum { constant(Int), dice(count: Int, sides: Int, plus: Int) }`, парсер из строки, `distribution` через свёртку.

**HitRules.** `func hitProbability(profile) -> HitOutcome`, где исход это распределение по трём классам: промах, обычное попадание, критическое. Перебросы меняют вероятности до модификатора по правилам. Sustained добавляет дополнительные попадания через свёртку. Lethal переводит криты в «автоматические ранения» отдельным потоком.

**WoundRules.** Порог по таблице, распределение обычных и критических ранений из обычных попаданий; автоматические ранения от Lethal добавляются. Devastating переводит критические ранения в «урон без сейва».

**SaveRules.** Вероятность провала сейва по AP и инвульну. FNP как вероятность пройти каждую единицу урона.

**DamageRules.** Из числа непрошедших ранений и выражения урона: распределение суммарного урона через свёртку. Убитые модели: марковская цепь по состоянию «убито k, у текущей осталось r ран», шаг на каждое ранение с распределением урона. Это самый сложный кусок.

**Calculator.** Собирает стадии по `Profile`, отдаёт `Report` с распределениями на каждой стадии.

**Simulator.** Та же последовательность, но бросками через переданный `RandomNumberGenerator` с сидом. Считает частоты. Независимая реализация правил: не переиспользует функции стадий, иначе проверка бессмысленна.

**Команда.** ArgumentParser с валидацией флагов, `ProfileLoader` сливает JSON и флаги, `TableRenderer` выравнивает.

## Поток данных

```
флаги/JSON ──▶ Profile
                 ──▶ HitRules ──▶ распределение {miss, hit, crit} × attacks
                 ──▶ WoundRules ──▶ {fail, wound, critWound} + lethalAutoWounds
                 ──▶ SaveRules ──▶ unsaved + devastatingDamage
                 ──▶ DamageRules ──▶ damage, modelsKilled
                 ──▶ Report
                 ──▶ (опц.) Simulator ──▶ сравнение
                 ──▶ TableRenderer | JSON
```

## Правила

- Стадии не знают друг о друге: вход и выход это распределения и числа.
- Правила игры только в `Rules/`, каждая функция с комментарием-ссылкой на правило.
- `Simulator` не зовёт `Rules/`. Дублирование намеренное.
- `Double` сравнивается через допуск `1e-9` в библиотеке и `1e-2` в тестах сходимости.
- Генератор случайных чисел передаётся, `SystemRandomNumberGenerator` только в команде.

## Что здесь тестируемо

Всё. `Distribution`: свёртка двух кубиков даёт известное распределение суммы, ожидание, инварианты. `Dice`: парсер на валидные и невалидные строки, распределения. Каждое правило: таблица «профиль, ожидаемое значение» из ручного расчёта. `Calculator`: десять эталонных профилей из сообщества с известными ответами. `Simulator`: свойство «на 100 000 прогонов отклонение меньше процента» для пяти профилей с сидом.

## Где ожидать боль

- Перебросы с модификаторами: переброс до модификатора, и «неудачные» это по немодифицированному броску. Записать правило до кода.
- Sustained и Lethal вместе: крит даёт и дополнительные попадания, и авторанение. Дополнительные попадания не критические.
- Убитые модели: наивная свёртка урона врёт, потому что лишний урон пропадает. Марковская цепь обязательна.
- Размер распределений: 20 атак с Sustained 2 и `2D6` урона это сотни значений. Обрезать хвосты ниже `1e-12`.
- Linux: `Foundation` там другой. `JSONDecoder` есть, `NumberFormatter` частично. Форматирование чисел руками.

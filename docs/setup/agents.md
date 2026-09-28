# Где в коде читается пробег (mileage)

Инвентаризация всех мест чтения mileage по коду репозитория. Поиск по `mileage`, `mileage_km`, `max_mileage_km`, `пробег` (регистронезависимо) плюс проверка косвенных цепочек: Http → AssessmentService → Validator → Repository и FormData → NUMERIC_FIELDS → fetch. Дата: 2026-09-28.

## 1. Чтения пробега в коде

| Файл | Стр. | Что и откуда читается |
|---|---|---|
| backend/src/Domain/ApplicationValidator.php | 43 | значение пробега читается из входящего payload HTTP-запроса: `$mileage = (int) ($payload['mileage'] ?? -1)` |
| backend/src/Domain/ApplicationValidator.php | 44 | лимит `max_mileage_km` читается из конфига правил (`$this->rules['vehicle']['max_mileage_km']`) для сравнения со значением пробега |
| backend/src/Domain/ApplicationValidator.php | 45 | лимит `max_mileage_km` повторно читается из конфига — подставляется в текст ошибки валидации |
| backend/src/Domain/ApplicationValidator.php | 78 | прочитанное значение пробега (`$mileage` из стр. 43) помещается в нормализованный массив входных данных, который уходит дальше в сервис и репозиторий |
| backend/src/Repository/ApplicationRepository.php | 45 | значение `mileage` читается из массива входных данных (`$input['mileage']`) для подстановки в SQL-параметр `:mileage` |
| backend/src/Repository/ApplicationRepository.php | 68 | колонка `v.mileage_km` читается из БД в SELECT метода `find()` (строка результата fetched на стр. 77 и возвращается вызывающему коду целиком на стр. 79) |
| frontend/app.js | 13 | значения всех полей формы, включая `mileage`, читаются из DOM через `new FormData(form).forEach(...)` |
| frontend/app.js | 14 | значение поля `mileage` (ключ входит в `NUMERIC_FIELDS`, стр. 8) читается из FormData и преобразуется в `Number` — точка извлечения числового пробега из формы |
| frontend/app.js | 49 | собранный payload (включая пробег) сериализуется через `JSON.stringify(collectPayload())` и уходит в тело HTTP-запроса |

## 2. Сопутствующие места (не чтения)

| Файл | Стр. | Что там |
|---|---|---|
| backend/config/rules.php | 23 | объявление лимита `'max_mileage_km' => 500000` в конфиг-массиве; адресное чтение лимита — только в ApplicationValidator (стр. 44–45) |
| backend/src/AppFactory.php | 27, 32 | конфиг rules.php загружается целиком (`require`) и передаётся в ApplicationValidator; ключ `max_mileage_km` адресно здесь не читается |
| backend/src/Domain/ApplicationValidator.php | 22 | PHPDoc-аннотация типа возврата с полем `mileage:int` — описание типа, не исполняемое чтение |
| backend/src/Repository/ApplicationRepository.php | 19 | PHPDoc-аннотация входного массива с `mileage:int` — описание типа, не исполняемое чтение |
| backend/src/Repository/ApplicationRepository.php | 38–39 | SQL-запись пробега в БД (`INSERT INTO vehicles (… mileage_km …)`); значение для параметра читается на стр. 45 |
| db/schema.sql | 22 | объявление колонки `mileage_km INT UNSIGNED` в таблице `vehicles` |
| db/seed.sql | 31 (значения в 32–55) | сид-данные: `INSERT INTO vehicles (… mileage_km …)` — запись синтетических пробегов в БД |
| frontend/index.html | 30–31 | разметка поля формы: label «Пробег, км» и `input#mileage` со значением по умолчанию 84000 |
| frontend/app.js | 8 | объявление константы `NUMERIC_FIELDS` со строкой `'mileage'`; чтение значения по этому списку — стр. 13–14 |
| frontend/app.js | 37–39 | цикл по объекту `errors` выводит тексты ошибок валидации; значение пробега здесь не читается, в DOM попадает только ключ `mileage` и текст ошибки |
| tests/Unit/ApplicationValidatorTest.php | 34 | фикстура `'mileage' => 84000` в `validPayload()` — тест передаёт значение, читает его продакшн-валидатор |
| tests/Unit/ApplicationValidatorTest.php | 19 | setUp загружает весь конфиг rules.php и передаёт его в ApplicationValidator; адресного чтения лимита в тесте нет |
| tests/Unit/AssessmentServiceTest.php | 38 | фикстура `'mileage' => 96000` в `payload()` — тест передаёт значение, читает его продакшн-код |
| tests/Unit/AssessmentServiceTest.php | 21 | setUp загружает весь конфиг rules.php и передаёт его в ApplicationValidator; адресного чтения лимита нет |
| README.md | 47 | пример curl с `"mileage":84000` — документация |
| docs/ (README, spec, plan, intent, hw1, setup, sources) | — | упоминания MILEAGE/пробега — документация и материалы клиента, не код |

## 3. Где пробег НЕ читается (проверено)

- **backend/src/Http/ApplicationController.php** — payload берётся целиком (`getParsedBody()`, стр. 25 и 56) и передаётся в `assess()` (стр. 28 и 59) без адресации ключа `mileage`; явно из payload читается только `applicant_ref` (стр. 33); `show()` (стр. 75, 78) возвращает строку репозитория как есть, не трогая `mileage_km`; `index()` (стр. 84–87) с пробегом не связан.
- **backend/src/Domain/AssessmentService.php** — payload передаётся валидатору (стр. 30); из `$input` явно читаются только `requested_amount`, `market_value`, `year` (стр. 32, 36, 39); `mileage` не адресуется, весь `$input` уходит в `'input'` (стр. 40).
- **backend/src/Domain/** — LtvCalculator.php, DecisionEngine.php, VehicleAge.php, VinValidator.php, ValidationException.php: пробег не упоминается и не используется ни в каком виде.
- **backend/src/Support/Json.php, backend/src/Database.php** — универсальные хелперы (JSON-ответ, подключение PDO), пробег не читается.
- **backend/public/index.php, backend/public/router.php** — бутстрап приложения и раздача статики frontend/; пробег не читается.
- **tests/Unit/** — VinValidatorTest.php, LtvCalculatorTest.php, DecisionEngineTest.php: фикстуры без mileage; **tests/Feature/** — только README.md, исполняемых тестов нет.
- **mocks/, scripts/, .githooks/** — совпадений по mileage/пробег нет (проверено полным поиском по репозиторию).

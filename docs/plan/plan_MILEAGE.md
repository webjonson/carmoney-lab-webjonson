# План: проверка пробега в решении по заявке (MILEAGE)

Статус: черновик, ждёт ответов на вопросы из последнего раздела.

## Контекст

- Требование (день 1): «пробег авто не больше 400 000 км, иначе решение
  `review`» (задание дня; `docs/README.md`, стр. 4).
- Сегодня решение считается только по LTV: `DecisionEngine::decide()`
  (`backend/src/Domain/DecisionEngine.php`, стр. 30–41) — LTV < 60 →
  approve, ≤ 85 → review, иначе reject.
- Пробег сейчас участвует только в валидации входа: `ApplicationValidator`
  (стр. 43–46) отвергает заявки с пробегом < 0 или > `max_mileage_km` = 500 000
  (`backend/config/rules.php`, стр. 23) — 422, `ValidationException`.
  Валидный пробег 0–500 000 на решение не влияет.
- Инвентаризация всех чтений пробега — `docs/setup/agents.md`. Файла
  `docs/setup/code_map.md` в репозитории нет; отдельный поиск не требуется —
  инвентаризация уже закрывает вопрос «где пробег читается».
- Выбранная семантика (обоснование — в «Рисках», открытые развилки — в
  «Вопросах»): при пробеге > 400 000 решение **понижается** с approve до
  review; reject не смягчается; заявки с невалидным пробегом (отсутствует,
  < 0, > 500 000) по-прежнему дают 422 и до нового правила не доходят.

## Файлы

- `backend/config/rules.php` — в секцию `vehicle` (после стр. 23
  `max_mileage_km`) добавить ключ `'review_mileage_km' => 400000` с
  комментарием, что выше этого пробега approve понижается до review.
- `backend/src/Domain/AssessmentService.php` — новый 5-й параметр конструктора
  `int $reviewMileageKm` (стр. 16–22); в `assess()` после стр. 33 (`decide()`)
  и до стр. 39 (`approved_limit`) — понижение approve → review при пробеге
  выше порога; обновить docblock класса (стр. 7–13). Публичная сигнатура
  `assess()` не меняется.
- `backend/src/AppFactory.php` — в сборке `AssessmentService` (стр. 30–39)
  передать `(int) $rules['vehicle']['review_mileage_km']` пятым аргументом.
- `tests/Unit/AssessmentServiceTest.php` — setUp (стр. 24–29): 5-й аргумент
  сервиса; хелпер `payload()` (стр. 33–43): параметр `int $mileage = 96000`;
  +4 новых теста из раздела «Тесты».
- `tests/Unit/ApplicationValidatorTest.php` — +1 регрессионный тест на
  отсутствующий пробег из раздела «Тесты».

Всё, чего нет в списке, при реализации трогать нельзя. Явно не трогаем:
`DecisionEngine.php`, `ApplicationValidator.php`, `ApplicationController.php`,
`ApplicationRepository.php`, `db/schema.sql`, `db/seed.sql`, `frontend/*`,
`README.md`.

Единственная меняющаяся сигнатура — конструктор `AssessmentService`:

```php
public function __construct(
    ApplicationValidator $validator,
    LtvCalculator $ltvCalculator,
    DecisionEngine $decisionEngine,
    VehicleAge $vehicleAge,
    int $reviewMileageKm,
)
```

## Шаги

1. `backend/config/rules.php`: добавить `'review_mileage_km' => 400000` в
   секцию `vehicle`. Порог живёт в конфиге, не в коде (AGENTS.md:
   бизнес-числа не хардкодим).
2. `backend/src/Domain/AssessmentService.php`: добавить параметр
   `int $reviewMileageKm` в конструктор; в `assess()` после
   `$decision = $this->decisionEngine->decide($ltv)` (стр. 33) понижать
   `$decision` с `DecisionEngine::APPROVE` до `DecisionEngine::REVIEW`, если
   `$input['mileage'] > $this->reviewMileageKm`. Понижение — строго до
   формирования массива возврата: тогда `approved_limit` (стр. 39) у
   пониженной заявки автоматически станет 0.
3. `backend/src/AppFactory.php`: передать порог из правил в `AssessmentService`.
4. `tests/Unit/AssessmentServiceTest.php`: обновить `setUp()` (5-й аргумент —
   `(int) $rules['vehicle']['review_mileage_km']`) и хелпер `payload()`
   (параметр `int $mileage = 96000`), добавить тесты из раздела «Тесты».
5. `tests/Unit/ApplicationValidatorTest.php`: добавить регрессионный тест на
   отсутствующий пробег.
6. Прогнать `make test` и `make lint` (по AGENTS.md работают и локально без
   Docker). При поднятом окружении — `curl` на `/api/ltv` с пробегом 400001:
   в ответе `decision: review`, `approved_limit: 0`.

## Тесты

Новые тесты в `tests/Unit/AssessmentServiceTest.php`; базовая фикстура —
approve по LTV: `payload(450000, 900000)`, LTV 50%.

- **399999** — `testKeepsApproveWhenMileageJustBelowReviewThreshold`:
  пробег 399999 → `decision` approve, `approved_limit` 450000 — порог
  не сработал.
- **400000** — `testKeepsApproveWhenMileageExactlyOnReviewThreshold`:
  пробег 400000 → approve, `approved_limit` 450000 — «не больше 400 000»
  значит граница включительна, правило не срабатывает.
- **400001** — `testDowngradesApproveToReviewWhenMileageAboveReviewThreshold`:
  пробег 400001 → `decision` review, `approved_limit` 0 — правило
  сработало, лимит обнулился.
- **reject не смягчается** — `testKeepsRejectWhenMileageAboveReviewThreshold`:
  фикстура reject по LTV (`payload(855000, 900000)`, LTV 95%) + пробег
  400001 → `decision` reject, `approved_limit` 0. Фиксирует выбранную
  семантику «только понижение»; при другом ответе на вопрос В1 тест
  переворачивается вместе с условием из шага 2.

Регрессионный тест в `tests/Unit/ApplicationValidatorTest.php`:

- **Пустой пробег** — `testRejectsMissingMileage`: payload из
  `validPayload()` без ключа `mileage` → `ValidationException`, в
  `errors()` есть ключ `mileage`. Существующее поведение: заявка без
  пробега не доходит до решения и `review` по новому правилу не получает.

Все тесты — AAA, заканчиваются assert'ом (конвенция AGENTS.md).

## Риски

- **Взаимодействие с `max_mileage_km` (главный риск).** Новое правило работает
  только в валидном диапазоне: 400 001–500 000 → review, > 500 000 →
  по-прежнему 422 валидации. Если заказчик ждёт «любой пробег > 400 000 →
  review», валидационный потолок 500 000 этому противоречит (вопрос В2).
  Менять `max_mileage_km` и текст его ошибки в этой задаче не входит.
- **Двусмысленность требования.** «Иначе решение review» можно прочитать как
  полную замену решения. Выбрано понижение approve → review: полная замена
  смягчала бы reject до review — ухудшение кредитной политики. При другом
  ответе на В1 меняются одно условие шага 2 и один тест.
- **`approved_limit`.** Если понижение сделать после вычисления лимита, у
  пониженной заявки лимит останется равным запрошенной сумме. Порядок
  зафиксирован в шаге 2 (понижение до массива возврата); тест на 400001
  проверяет `approved_limit` 0.
- **Смена сигнатуры конструктора `AssessmentService`.** Ломает сборку в
  `AppFactory` и тестах — оба файла в списке «Файлы»; других мест создания
  `AssessmentService` в репозитории нет (проверено по
  `docs/setup/agents.md`). `ApplicationController` не затронут: публичная
  сигнатура `assess()` и структура ответа не меняются.
- **«Пустой пробег» неоднороден.** Отсутствующий ключ и `null` → 422
  (тест выше), но пустая строка `''` в `(int)` превращается в 0 и проходит
  валидацию — существующий вар `ApplicationValidator`, в этой задаче не
  чиним (вопрос В3). Фронтенд закрыт `required` у `input#mileage`.
- **Хранение и ответ API.** `decision` пишется в таблицу `decisions`, ENUM
  уже содержит `review` — схема БД не меняется; `mileage_km INT UNSIGNED`
  вмещает 400001. Контракт `/api/ltv` и `/api/applications` не меняется.
- **Существующие фикстуры** (84000, 96000 в тестах; сиды `db/seed.sql`) ниже
  порога — их решения не меняются.

## Не входит

- Изменение `max_mileage_km` (500 000) и текста его ошибки валидации.
- Фронтенд: подсказки, клиентская валидация порога, показ причины review.
- Схема и сиды БД, миграции.
- LOAN-12 (лимит по `ltv_by_age`), CASE-08 (скидки каталога за пробег).
- Feature-тесты HTTP-эндпоинтов (в `tests/Feature` сейчас только README).
- Обновление `README.md` и остальных docs — отдельный шаг задания.

## Вопросы, на которые без заказчика не ответить

1. **В1 — reject при большом пробеге.** Заявка с LTV-решением reject и
   пробегом 400 001: оставляем reject (выбрано в плане) или «иначе review»
   означает замену решения на review?
2. **В2 — потолок валидации.** Верна ли связка «400 001–500 000 → review,
   > 500 000 → 422»? Или пробег выше 400 000 всегда должен доходить до
   решения — тогда `max_mileage_km` надо поднять или убрать, и это отдельное
   решение по существующей проверке.
3. **В3 — пустая строка как пробег.** `mileage: ""` сегодня превращается
   в 0 и проходит валидацию. Это баг, который чинить в этой задаче, или
   допустимое поведение?
4. **В4 — причина review в ответе.** Ответ API содержит только
   `decision`/`ltv`/`approved_limit`. Нужен ли признак «review из-за
   пробега» (например, поле `reason`), или достаточно самого решения?

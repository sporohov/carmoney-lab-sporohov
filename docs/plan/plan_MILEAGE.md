# План MILEAGE: пробег > 400 000 км → понижение approve до review

Статус: **готов к реализации** (черновик закрыт). Приведён в соответствие с
docs/spec/spec_MILEAGE.md (REQ-MILEAGE-01…04, AC-MILEAGE-01…08). Все вопросы
черновика закрыты — раздел 6; данных, которых не хватало бы для реализации,
нет — все факты сверены чтением кода (см. Контекст).

## Контекст (сверено с кодом и spec, а не с черновиком)

- Решение сегодня считается только по LTV: `DecisionEngine::decide(float $ltv)`
  (DecisionEngine.php:30), пороги `approve_max = 60.0` / `review_max = 85.0`
  (rules.php:44–45), проводка — AppFactory.php:37 (`new DecisionEngine($rules['ltv'])`).
- Пробег участвует только в валидации: диапазон 0…`max_mileage_km = 500000`
  (ApplicationValidator.php:43–46, rules.php:23). При отсутствии поля `mileage`
  валидатор подставляет `-1` (ApplicationValidator.php:43) — заявка падает с
  `ValidationException`. Ошибки валидации HTTP-слой отдаёт как 422
  (ApplicationController.php:30,61), успех — 200 (`/api/ltv`) / 201
  (`/api/applications`, ApplicationController.php:50).
- Пробег уже лежит нормализованным в `$input['mileage']` внутри
  `AssessmentService::assess()` (ApplicationValidator.php:78) — точка передачи
  в движок: AssessmentService.php:33.
- Фактически новое правило сработает только в диапазоне
  400 000 < пробег <= 500 000: выше 500 000 заявку отсекает валидация.
- Лимит привязан к решению: `approved_limit` = запрошенная сумма при approve,
  иначе 0 (AssessmentService.php:39). Логика не меняется, фиксируется тестами.
- Известное расхождение кода и докблока `DecisionEngine` на границе 60.0
  (строгая `<` в коде, DecisionEngine.php:32, против `<=` в докблоке) — вне
  задачи (spec §1), не трогаем.
- Конвенции (AGENTS.md §4): PHP 8.3, `declare(strict_types=1)`, `final`,
  свойства `readonly` через конструктор, namespace `CarMoneyLab\`, бизнес-числа
  только из `backend/config/rules.php`, тесты AAA с именами, описывающими
  поведение.

**Тестовая база (проверено по файлам, исправляет ошибки черновика):**

- `tests/Unit/DecisionEngineTest.php` — **существующий** файл (черновик ошибочно
  называл его «новым»): setUp строит движок как
  `new DecisionEngine(['approve_max' => 60.0, 'review_max' => 85.0])` (:17),
  DataProvider-тест `testDecidesByLtv` вызывает `decide($ltv)` **одним
  аргументом** (:23), 6 кейсов LTV→решение. Любое расширение сигнатур
  конструктора и `decide()` затрагивает этот файл.
- `tests/Unit/AssessmentServiceTest.php` — существующий файл, **тоже
  конструирует `DecisionEngine` напрямую** (:27, `new DecisionEngine($rules['ltv'])`)
  на реальном rules.php (:21). Фикстуры: mileage 96 000 (:38), LTV 50.0 / 75.0 /
  95.0 (суммы 450 000 / 675000 / 855 000 при стоимости 900 000) — не ломать.
- `tests/Unit/ApplicationValidatorTest.php` — существующий; в базовой фикстуре
  mileage 84 000 (:34). Пробег за 500 000 и отсутствие поля `mileage` тестами
  не покрыты — оба кейса новые (AC-05, AC-08).
- Каталога `tests/Feature/` **нет** (проверено); testsuite "feature" на
  tests/Feature уже объявлен в phpunit.xml:13–16 с комментарием «нужна поднятая
  база или HTTP-запрос». Инфраструктуры фичей нет — в этой задаче не создаём.
- phpunit.xml: `failOnWarning="true"`, `failOnRisky="true"` — влияет на картину
  красной фазы: обращение к несуществующему ключу массива (Warning) делает тест
  упавшим.

**Принятая семантика** (все открытые вопросы закрыты — spec §5): порог
400 000 км; при `mileage > 400 000` решение не может быть approve — оно
понижается до review; ровно 400 000 проходит без понижения (граница
включительная); `reject` и `review` по LTV высокий пробег не меняет.

## Принятые проектные решения

**D1. Сигнатура `decide()` — вариант A: обязательный второй аргумент.**

`public function decide(float $ltv, int $mileage): string`

- Вариант B (`int $mileage = 0` по умолчанию) отвергнут: вызыватель, забывший
  пробег, молча получил бы решение только по LTV — ровно тот класс ошибки,
  который задача устраняет (approve для авто с пробегом 450 001). Дефолт 0
  к тому же неотличим от легитимного «нулевого пробега».
- С вариантом A забытый аргумент — громкий `ArgumentCountError` при первом же
  `make test`, а не тихий неверный ответ.
- Цена варианта A — правка существующих тестов в красной фазе, где ДЗ правку
  тестов разрешает: меняются только вызовы и конструкторы, семантика тестов
  сохраняется (раздел 3). В зелёной фазе тесты неприкосновенны.
- Полный список затронутых мест (проверено поиском по репозиторию — других
  вызовов `decide()` / `DecisionEngine` нет):
  - production: AssessmentService.php:33 (зелёная), AppFactory.php:37 (зелёная);
  - тесты: DecisionEngineTest.php:17,23 (красная), AssessmentServiceTest.php:27
    (красная — место, которого нет в разведке черновика).

**D2. Ключ в rules.php и способ проводки (закрывает Q5).**

- Имя ключа: `review_mileage_km => 400000`, секция `vehicle` (rules.php:20–24),
  рядом с валидационным `max_mileage_km` (:23), с комментарием «порог решения,
  а не валидации». Отдельная секция `mileage_decision` отвергнута: она
  фрагментировала бы конфиг; соседство с `max_mileage_km` и «review» в имени
  разводят два порога.
- В `DecisionEngine` прокидывается **значение** (`int`), а не вся секция
  `vehicle`: второй обязательный параметр конструктора
  `public function __construct(array $thresholds, int $reviewMileageKm)`
  (форма `$thresholds` — `array{approve_max:float,review_max:float}` — не
  меняется; расширять этот массив пробегом и склеивать массивы в AppFactory
  отвергнуто как менее явное). Движку нужны ровно две вещи — пороги LTV и порог
  пробега; вся секция vehicle потянула бы в него `min_year` / `max_age_years` /
  `max_mileage_km`, которые движку не нужны.
- Проводка: AppFactory.php:37 →
  `new DecisionEngine($rules['ltv'], $rules['vehicle']['review_mileage_km'])`.
  При отсутствии ключа — громкий сбой на старте (Warning + TypeError по int),
  а не тихий дефолт; дефолт порога в коде не изобретаем.

**D3. AC с HTTP-кодами (AC-04/05/08) закрываются доменным уровнем.**

- HTTP-слой тонкий и этой задачей не меняется: контроллер ловит
  `ValidationException` → 422 (ApplicationController.php:30,61), иначе отдаёт
  результат `assess()` со статусом 200 (`/api/ltv`) / 201 (`/api/applications`).
  Меняется только строка `decision`, которую возвращает `assess()`.
- Spec сам предписывает уровни тестов (spec §2): юнит `DecisionEngine` +
  интеграционный сценарий «валидация → LTV → пробег → решение» через
  `AssessmentService::assess()`. Фиче-инфраструктуры нет (каталога tests/Feature
  нет), создавать её — объём сверх spec.
- Итог: AC-04 — интеграционный тест «`assess()` не бросил `ValidationException`
  и вернул review» (доменный эквивалент «HTTP 200, не 422»); AC-05 — тест
  валидатора «`ValidationException` с ошибкой поля mileage» (эквивалент 422);
  AC-08 — тест валидатора + тест сервиса. Фиксируем явно: **422/200/201 —
  существующее поведение контроллера, этой задачей не меняется и автоматическими
  тестами не проверяется**; HTTP-аспект после зелёной фазы закрывается ручной
  дымовой проверкой (шаг зелёной фазы 7).

**D4. Процесс — красная → зелёная, два отдельных коммита.**

- Красная фаза: пишутся только тесты на все AC (+ минимальная правка вызовов
  существующих тестов, в этой фазе разрешённая); `make test` обязан упасть;
  отдельный коммит тестов ДО кода.
- Зелёная фаза: только реализация по разделу 2; тесты не трогаются и не
  ослабляются; `make test` зелёный; отдельный коммит кода.

## 1. Файлы

| Файл | Что делаем | Фаза |
|---|---|---|
| backend/config/rules.php | в секцию `vehicle` добавить `review_mileage_km => 400000` с комментарием «порог решения (пробег выше — approve понижается до review); валидационный `max_mileage_km` = 500000 не меняется» | зелёная |
| backend/src/Domain/DecisionEngine.php | сигнатуры конструктора и `decide()` (D1/D2), логика понижения, докблок | зелёная |
| backend/src/Domain/AssessmentService.php | :33 — передать `$input['mileage']` вторым аргументом в `decide()` | зелёная |
| backend/src/AppFactory.php | :37 — проводка значения порога (D2) | зелёная |
| tests/Unit/DecisionEngineTest.php | существующий файл: обновить setUp и вызов `testDecidesByLtv`, добавить 5 тестов | красная |
| tests/Unit/AssessmentServiceTest.php | существующий файл: обновить setUp (:27) и хелпер `payload()`, добавить 5 тестов | красная |
| tests/Unit/ApplicationValidatorTest.php | существующий файл: добавить 2 теста | красная |
| docs/setup/code_map.md | раздел «Гипотетическое правило» заменить описанием реализованного | зелёная |

Не трогаем (всё, чего нет в таблице): `ApplicationController.php` (статусы
200/201/422 — существующее поведение), `ApplicationValidator.php`,
`ValidationException.php`, `LtvCalculator.php`, `VehicleAge.php`,
`VinValidator.php`, `phpunit.xml`, `frontend/`, `db/`, `tests/Feature/`
(не создаём), `mocks/`, `scripts/`.

## 2. Шаги реализации

### Фаза «красная» — только тесты, коммит № 1 (до кода)

1. `DecisionEngineTest`: setUp —
   `new DecisionEngine(['approve_max' => 60.0, 'review_max' => 85.0], 400000)`
   (значения зеркалят rules.php, как и сейчас :17); в существующем
   `testDecidesByLtv` вызов становится `decide($ltv, 96000)` — 96 000 ниже
   порога, тест по-прежнему проверяет только LTV-ветвление (семантика не
   меняется). Добавить 5 тестов AC-01/02/03/06/07 — карта в разделе 3.
2. `AssessmentServiceTest`: setUp :27 —
   `new DecisionEngine($rules['ltv'], $rules['vehicle']['review_mileage_km'])`
   (реальный rules.php — интеграционная честность; в красной фазе ключа ещё
   нет, тесты файла падают на setUp — это ожидаемое красное поведение, а не
   ослабление). Хелпер получает третий параметр
   `payload(int $amount, int $marketValue, int $mileage = 96000)` — дефолт
   сохраняет существующие вызовы неизменными. Добавить 5 тестов
   AC-02e2e / AC-03+04 / AC-06e2e / AC-07e2e / AC-08e2e.
3. `ApplicationValidatorTest`: добавить 2 теста AC-05 и AC-08 (валидатор);
   «payload без mileage» — убрать ключ `mileage` из базовой фикстуры
   (unset после `validPayload()`).
4. Прогон: `make test` — обязан упасть (ожидаемая картина — раздел 3);
   `make lint` — чисто (синтаксис тестов валиден).
5. Коммит тестов, например: «MILEAGE: красные тесты — учёт пробега в решении».

### Фаза «зелёная» — только код, коммит № 2 (тесты не трогаем)

1. `rules.php`: `review_mileage_km => 400000` в секции `vehicle` + комментарий
   из таблицы раздела 1.
2. `DecisionEngine`: конструктор — второй обязательный `int $reviewMileageKm`;
   `decide(float $ltv, int $mileage): string`; логика: вычислить LTV-решение
   как сейчас (границу 60.0 — строгую `<` — не трогать), затем если
   LTV-решение = approve и `mileage > $reviewMileageKm` — вернуть REVIEW,
   иначе LTV-решение. Числа только из конструктора, ничего не хардкодить.
   Докблок: дополнить правилом пробега; существующее расхождение `<`/`<=` в
   описании LTV-границы оставить как есть.
3. `AssessmentService.php:33`: `decide($ltv, $input['mileage'])`.
4. `AppFactory.php:37`: `new DecisionEngine($rules['ltv'], $rules['vehicle']['review_mileage_km'])`.
5. Прогон: `make test` — зелёный (все AC + существующие тесты без правок),
   `make lint` — чисто. Если красно — править код, не тесты.
6. `docs/setup/code_map.md`: раздел «Гипотетическое правило „пробег > 400 000 →
   review"» заменить описанием реализованного: ключ `review_mileage_km`,
   проводка через AppFactory, приоритет (понижается только approve), фактический
   диапазон действия 400 000 < x <= 500 000; в блок-схеме mermaid отметить, что
   `decide` теперь принимает и пробег.
7. Ручная дымовая проверка HTTP-поведения (закрывает коды 200/422 из
   AC-04/05/08, см. D3): `make up`, затем POST `/api/ltv` — mileage 400 001 →
   200 + `decision: review`; mileage 500 001 → 422; payload без `mileage` → 422.
8. Коммит кода, например: «MILEAGE: понижение approve до review при пробеге > 400000».

## 3. Тесты

Имена — по конвенции репозитория: `test*`, camelCase, имя описывает поведение
(как у существующих тестов; snake-case из черновика конвенции не соответствует);
структура AAA. Уровни: юнит движка, интеграция через `assess()`
(валидация → LTV → пробег → решение), юнит валидатора (AC-05/08 —
регрессионные пины неизменного поведения). Тексты сообщений валидатора не
ассертим (тексты вне задачи — intent §5), проверяем ключ `'mileage'` в
`errors()`.

### Карта AC → тест (каждый REQ и каждый AC покрыты)

| REQ / AC | Тест-метод | Файл | Уровень | Суть |
|---|---|---|---|---|
| REQ-01 / AC-01 | `testKeepsApproveWhenMileageBelowReviewThreshold` | tests/Unit/DecisionEngineTest.php | юнит | `decide(50.0, 399999)` → approve |
| REQ-01 / AC-02 | `testKeepsApproveWhenMileageAtReviewThreshold` | tests/Unit/DecisionEngineTest.php | юнит | `decide(50.0, 400000)` → approve (граница включительна) |
| REQ-01 / AC-02 (e2e) | `testApprovesBoundaryMileageAndSetsLimitToRequestedAmount` | tests/Unit/AssessmentServiceTest.php | интеграция | payload LTV 50.0, mileage 400 000 → approve, `approved_limit` 450 000 |
| REQ-01 / AC-03 | `testDowngradesApproveToReviewWhenMileageExceedsReviewThreshold` | tests/Unit/DecisionEngineTest.php | юнит | `decide(50.0, 400001)` → review |
| REQ-01, REQ-02 / AC-03 + AC-04 | `testSendsHighMileageToReviewWithZeroLimit` | tests/Unit/AssessmentServiceTest.php | интеграция | payload LTV 50.0, mileage 400 001 → без `ValidationException`, review, `approved_limit` 0 |
| REQ-02 / AC-05 | `testRejectsMileageAboveValidationLimit` | tests/Unit/ApplicationValidatorTest.php | юнит валидатора | mileage 500 001 → `ValidationException`, ключ `'mileage'` в errors |
| REQ-03 / AC-06 | `testKeepsRejectWhenMileageExceedsReviewThreshold` | tests/Unit/DecisionEngineTest.php | юнит | `decide(95.0, 400001)` → reject |
| REQ-03 / AC-06 (e2e) | `testKeepsRejectForHighLtvWithHighMileage` | tests/Unit/AssessmentServiceTest.php | интеграция | payload LTV 95.0, mileage 400 001 → reject, `approved_limit` 0 |
| REQ-03 / AC-07 | `testKeepsReviewWhenMileageExceedsReviewThreshold` | tests/Unit/DecisionEngineTest.php | юнит | `decide(75.0, 400001)` → review |
| REQ-03 / AC-07 (e2e) | `testKeepsReviewForMiddleLtvWithHighMileage` | tests/Unit/AssessmentServiceTest.php | интеграция | payload LTV 75.0, mileage 400 001 → review, `approved_limit` 0 |
| REQ-04 / AC-08 | `testRejectsMissingMileageField` | tests/Unit/ApplicationValidatorTest.php | юнит валидатора | payload без `mileage` → `ValidationException`, ключ `'mileage'` в errors |
| REQ-04 / AC-08 (e2e) | `testDoesNotAssessApplicationWithoutMileage` | tests/Unit/AssessmentServiceTest.php | интеграция | `assess()` на payload без `mileage` бросает `ValidationException` — до решения не доходит |

### Граничные значения (все числа — из spec §4)

- 399 999 — последнее значение без понижения (AC-01);
- 400 000 — граница включительно, без понижения (AC-02);
- 400 001 — первое значение с понижением (AC-03/04/06/07);
- 500 001 — за валидационным пределом → ошибка валидации (AC-05);
- отсутствие поля `mileage` → ошибка валидации (AC-08);
- LTV-фикстуры зон 50.0 / 75.0 / 95.0 — как в AssessmentServiceTest (суммы
  450 000 / 675 000 / 855 000 при стоимости 900 000);
- существующие фикстуры пробега 96 000 (AssessmentServiceTest:38) и
  84 000 (ApplicationValidatorTest:34) не ломать.

### Обновление существующих тестов (только красная фаза, семантика сохраняется)

- `DecisionEngineTest::testDecidesByLtv`: 6 LTV-кейсов не меняются; вызов
  `decide($ltv, 96000)` — пробег фиксирован ниже порога, тест по-прежнему
  проверяет LTV-ветвление.
- `AssessmentServiceTest`: setUp :27 читает порог из реального rules.php; до
  зелёной фазы ключа нет, поэтому в красной фазе падают все тесты файла на
  setUp. После появления ключа (зелёная фаза) они проходят без правок.
- Существующие approve/review/reject-тесты по LTV не переписываются:
  approve-кейс (mileage 96 000) остаётся зелёной охраной «низкий пробег не
  понижает approve, лимит равен запрошенной сумме».

### Ожидаемая картина красной фазы

`make test` обязан упасть. Падают:

- `testDowngradesApproveToReviewWhenMileageExceedsReviewThreshold` — движок
  ещё не учитывает пробег, review не возвращается;
- все тесты `AssessmentServiceTest` — setUp обращается к ещё не
  существующему ключу `rules['vehicle']['review_mileage_km']`
  (`failOnWarning="true"` делает Warning ошибкой теста).

Тесты AC-01/02/05/06/07/08 в красной фазе могут проходить: они фиксируют
поведение, существующее и до правила (границы 399 999/400 000, приоритет для
зон review/reject при пока не учитываемом пробеге, валидация 500 001 и
отсутствие поля). Их назначение — охрана от регрессий в зелёной фазе:
реализация «заменять любое решение на review» сломает AC-06, «понижать при
>= 400 000» сломает AC-02, «пропускать 500 001 в решение» сломает AC-05.
Общее падение сюиты гарантировано первыми двумя пунктами; требование «каждый
новый тест падает» не выдвигается.

## 4. Риски

- **Перепутать пороги**: валидационный `max_mileage_km = 500000` (rules.php:23)
  и новый порог решения `review_mileage_km = 400000` близки. Если правило
  случайно вписать в валидатор, заявки с 400 001+ начнут падать с 422 вместо
  review. Митигация: «review» в имени ключа, комментарий в rules.php, тесты
  AC-04 (не ошибка валидации) и AC-05 (422-эквивалент за 500 001) разводят
  пороги.
- **Сломать существующие тесты**: два существующих файла конструируют
  `DecisionEngine` напрямую (DecisionEngineTest:17, AssessmentServiceTest:27 —
  второй в разведке черновика не был учтён). Оба обновляются только в красной
  фазе и только по сигнатуре (семантика — LTV-ветвление при пробеге ниже
  порога). В зелёной фазе тесты не трогаем: если `make test` красный — править
  код, не тесты; ослабление или удаление тестов запрещено.
- **«Заодно» поменять границу 60.0**: строгая `<` в DecisionEngine.php:32 против
  `<=` в докблоке — существующее расхождение (spec §1). Значение LTV ровно
  60.0 существующими тестами не покрыто, регрессия не будет поймана
  автоматически. Запрет на уровне плана и ревью: LTV-ветвление не менять,
  докблок LTV-части не переписывать (только дополнить правилом пробега).
- **Приоритет правил — проектное решение**: «reject не смягчается» зафиксирован
  spec §5 как рабочее для ДЗ и, по пометке spec §5.2, подлежит подтверждению
  заказчиком до выпуска. Тесты AC-06/07 охраняют выбранную семантику.
- **Проводка AppFactory не покрыта тестами** (как и сегодня для ltv-порогов):
  AssessmentServiceTest собирает сервис сам, минуя AppFactory; опечатка в ключе
  даст громкий сбой на старте приложения, а не в юнитах. Митигация: точное имя
  ключа в плане (D2) + дымовая проверка шага 7 зелёной фазы.
- **`AppFactory::create(?array $rules)` с кастомными правилами**: внешний вызов
  с урезанным rules без `vehicle.review_mileage_km` упадёт на старте (громко,
  без тихого дефолта — это осознанное решение D2). В репозитории таких
  вызовов нет (проверено); реальный rules.php ключ получит.

## 5. Не входит (сверено с spec §2)

- Валидатор и его диапазон: `max_mileage_km = 500000`, тексты сообщений —
  код валидатора не меняется (REQ-MILEAGE-02 фиксирует поведение тестами).
- Пороги решения по LTV 60.0 / 85.0 и расхождение кода/докблока на границе 60.0.
- Логика `approved_limit` (существующее поведение, AssessmentService.php:39) —
  только фиксируется тестами: review от понижения → лимит 0.
- HTTP-слой: контроллер, статусы 200/201/422, тексты ответов (D3).
- Тестовая инфраструктура `tests/Feature` — каталог и фреймворк фичей не
  создаём; testsuite в phpunit.xml остаётся декларацией на будущее.
- Фронтенд (поле mileage уже есть), база данных и миграции (`mileage_km`
  INT UNSIGNED), задача LOAN-12 (`ltv_by_age`).

## 6. Вопросы и решения (все закрыты, открытых нет)

| # | Вопрос из черновика | Решение | Источник |
|---|---|---|---|
| Q1 | Приоритет: reject + высокий пробег; review + высокий пробег | Закрыт: понижается только approve; reject остаётся reject, review остаётся review. До выпуска подтверждается заказчиком (для ДЗ — рабочее) | spec §5, вопрос intent №2 |
| Q2 | Граница: проходит ли ровно 400 000 | Закрыт: 400 000 включительно проходит, 400 001 → review | spec §5, вопрос intent №1 (задание ДЗ.1: заголовок PR «Пробег ≤ 400000», ручная проверка «400001 даёт review») |
| Q3 | Пустой пробег: менять ли текущее поведение | Закрыт: не менять — отсутствие `mileage` = ошибка валидации (422), не review; фиксируется тестами AC-08 | spec §5, вопрос intent №3; REQ-MILEAGE-04 |
| Q4 | Пробел 400–500 тыс.: ужесточать ли `max_mileage_km` | Закрыт: валидация не меняется, диапазон 0…500 000 остаётся; заявки 400 001…500 000 валидны и уходят в review — это и есть суть REQ-MILEAGE-02 | intent §5 «Не входит», spec §2 |
| Q5 | Имя ключа rules.php и способ проводки | Решено в плане (D2): `review_mileage_km` в секции `vehicle`; в `DecisionEngine` прокидывается значение (int) вторым аргументом конструктора через AppFactory.php:37; отдельная секция не создаётся | план D2 |

Чего не хватает для реализации: ничего — все факты сверены чтением кода
(Контекст). Единственная пометка вне плана работ: семантика приоритета (Q1)
по spec §5.2 подлежит подтверждению заказчиком до выпуска; блокером для
реализации по ДЗ не является.
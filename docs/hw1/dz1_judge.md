# Отчёт независимого аудита реализации фичи MILEAGE

**Дата аудита:** 30 сентября 2026 г.  
**Аудитор:** Независимый агент-судья (модель: `aitunnel/gemini-3.8-flash`)  
**Ветка аудита:** `hw1/dz1-sporohov`  
**Целевой коммит:** `2b95fe5`  
**Статус сюита тестов:** OK (38 tests, 51 assertions), lint OK  

---

## 1. Уровень ★: Проверка покрытия и отсутствия лишних требований

### 1.1. Анализ отсутствия лишних требований (scope creep) в `docs/spec/spec_MILEAGE.md`
Проведён детальный построчный анализ требований `spec_MILEAGE.md` против замысла и ограничений `docs/intent/intent_MILEAGE.md`:

* **REQ-MILEAGE-01 (Понижение approve до review при пробеге > 400 000 км):**
  * *Соответствие Intent:* Полное. Прямо следует из Intent §1 («пробег не больше 400 000 км, иначе решение review») и Intent §3 («порог 400 000 км»).
  * *Лишние требования:* Отсутствуют. Включение граничного значения 400 000 км в approve прямо закрывает Open question №1 из Intent на основании условий домашнего задания ДЗ.1 и семантики плана.
* **REQ-MILEAGE-02 (Правило живёт в решении, а не в валидации):**
  * *Соответствие Intent:* Полное. В Intent §3 явно зафиксировано: «проверка входит именно в расчёт решения по заявке, а не в проверку входных данных», а диапазон 0…500 000 км не меняется.
  * *Лишние требования:* Отсутствуют. Требование явно предостерегает от переноса правила в валидатор, что сломало бы архитектуру.
* **REQ-MILEAGE-03 (Приоритет перед решением по LTV):**
  * *Соответствие Intent:* Полное. Закрывает Open question №2 из Intent §4. Зафиксировано, что высокий пробег снижает только `approve` и не превращает `reject` в `review` и не переопределяет существующий `review`.
  * *Лишние требования:* Отсутствуют. Это необходимое разрешение неоднозначности постановки, зафиксированное как проектное решение с явным указанием источника.
* **REQ-MILEAGE-04 (Заявка без поля пробега — поведение не меняется):**
  * *Соответствие Intent:* Полное. Закрывает Open question №3 из Intent §4 («сохранять ли текущее поведение — ошибка валидации»). В Intent §3 отмечено, что сегодня отсутствие поля вызывает ошибку валидации.
  * *Лишние требования:* Отсутствуют. Требование фиксирует статус-кво и защищает от скрытой деградации функционала.

**Вывод:** В спецификации `docs/spec/spec_MILEAGE.md` лишних требований (scope creep) нет. Все 4 REQ строго соответствуют исходному `intent_MILEAGE.md`, а открытые вопросы закрыты со ссылками на объективные источники (ТЗ ДЗ.1, конвенции репозитория и текущий код).

---

### 1.2. Таблица трассируемости: Покрытие REQ и AC автоматическими тестами

Все 4 REQ и 8 AC спецификации покрыты модульными и компонентными интеграционными тестами без пропусков:

| REQ | AC | Суть критерия приёмки | Метод теста | Файл теста |
|---|---|---|---|---|
| **REQ-MILEAGE-01** | AC-MILEAGE-01 | LTV=50.0, пробег **399 999** км → `decision = approve` | `testKeepsApproveWhenMileageBelowReviewThreshold` | `tests/Unit/DecisionEngineTest.php` |
| **REQ-MILEAGE-01** | AC-MILEAGE-02 | LTV=50.0, пробег **400 000** км → `decision = approve` | `testKeepsApproveWhenMileageAtReviewThreshold`<br>`testApprovesBoundaryMileageAndSetsLimitToRequestedAmount` | `tests/Unit/DecisionEngineTest.php`<br>`tests/Unit/AssessmentServiceTest.php` |
| **REQ-MILEAGE-01** | AC-MILEAGE-03 | LTV=50.0, пробег **400 001** км → `decision = review`, `approved_limit = 0` | `testDowngradesApproveToReviewWhenMileageExceedsReviewThreshold`<br>`testSendsHighMileageToReviewWithZeroLimit` | `tests/Unit/DecisionEngineTest.php`<br>`tests/Unit/AssessmentServiceTest.php` |
| **REQ-MILEAGE-02** | AC-MILEAGE-04 | Пробег 400 001 км проходит валидацию, отдаёт `review` (не 422) | `testSendsHighMileageToReviewWithZeroLimit` | `tests/Unit/AssessmentServiceTest.php` |
| **REQ-MILEAGE-02** | AC-MILEAGE-05 | Пробег **500 001** км → ошибка валидации `mileage` (HTTP 422) | `testRejectsMileageAboveValidationLimit` | `tests/Unit/ApplicationValidatorTest.php` |
| **REQ-MILEAGE-03** | AC-MILEAGE-06 | LTV=95.0 (reject), пробег 400 001 км → `decision = reject` (не `review`) | `testKeepsRejectWhenMileageExceedsReviewThreshold`<br>`testKeepsRejectForHighLtvWithHighMileage` | `tests/Unit/DecisionEngineTest.php`<br>`tests/Unit/AssessmentServiceTest.php` |
| **REQ-MILEAGE-03** | AC-MILEAGE-07 | LTV=75.0 (review), пробег 400 001 км → `decision = review` | `testKeepsReviewWhenMileageExceedsReviewThreshold`<br>`testKeepsReviewForMiddleLtvWithHighMileage` | `tests/Unit/DecisionEngineTest.php`<br>`tests/Unit/AssessmentServiceTest.php` |
| **REQ-MILEAGE-04** | AC-MILEAGE-08 | Отсутствие поля `mileage` → `ValidationException` (HTTP 422), решение не считается | `testRejectsMissingMileageField`<br>`testDoesNotAssessApplicationWithoutMileage` | `tests/Unit/ApplicationValidatorTest.php`<br>`tests/Unit/AssessmentServiceTest.php` |

---

### 1.3. Проверка граничных значений и пустого пробега
* **399 999 км:** проверено в `DecisionEngineTest::testKeepsApproveWhenMileageBelowReviewThreshold` (возвращает `approve`).
* **400 000 км:** проверено на уровне движка (`DecisionEngineTest::testKeepsApproveWhenMileageAtReviewThreshold`) и на уровне сервиса (`AssessmentServiceTest::testApprovesBoundaryMileageAndSetsLimitToRequestedAmount`) — строго `approve` и выдача запрошенного лимита.
* **400 001 км:** проверено на понижение `approve` → `review` с обнулением лимита на обоих уровнях (`DecisionEngineTest::testDowngradesApproveToReviewWhenMileageExceedsReviewThreshold`, `AssessmentServiceTest::testSendsHighMileageToReviewWithZeroLimit`), а также на сохранение `reject` и `review` (AC-06, AC-07).
* **500 001 км:** проверено в `ApplicationValidatorTest::testRejectsMileageAboveValidationLimit` — отсекается валидатором, до движка решений не доходит.
* **Пустой пробег (отсутствие ключа `mileage`):** проверено как в `ApplicationValidatorTest::testRejectsMissingMileageField`, так и в `AssessmentServiceTest::testDoesNotAssessApplicationWithoutMileage` — выбрасывается `ValidationException`, расчёт решения прерывается.

---

### 1.4. Проверка соблюдения TDD-процесса (по коммитам Git)
Анализ журнала репозитория (`git log --oneline`):
1. `f9b9f33` — *MILEAGE: spec (REQ-MILEAGE-01…04, AC-MILEAGE-01…08) и план, готовый к реализации* (зафиксированы спецификация и план).
2. `2fdcca1` — *MILEAGE: красные тесты — учёт пробега в решении* (написаны и закоммичены **только тесты** в `tests/Unit/`, продакшн-код в `backend/` не изменялся).
3. `2b95fe5` — *MILEAGE: реализация правила пробега в DecisionEngine и обновление code_map* (написан продакшн-код и обновлена документация, тесты не модифицировались).

**Вывод:** TDD-процесс соблюдён безукоризненно:
- Тесты написаны и зафиксированы в git строго **ДО** написания кода реализации.
- На этапе коммита `2fdcca1` сюит падал с ожидаемыми ошибками (2 failures, 1 warning из-за отсутствия ключа в конфиге).
- При переходе к коммиту `2b95fe5` файлы тестов не модифицировались ради подгонки под зелёный статус (`git diff 2fdcca1..2b95fe5 -- tests/` пуст).

---

### 1.5. Итог уровня ★
**СТАТУС: ПРОЙДЕН ПОЛНОСТЬЮ (PASS).**  
Требования не раздуты, все критерии приёмки и граничные условия покрыты тестами на двух уровнях, TDD-дисциплина подтверждена историей коммитов.

---

## 2. Уровень ★★: Где агент срезал углы и что сделал не по плану

Проведён критический анализ всей цепочки артефактов (`spec_MILEAGE.md`, `plan_MILEAGE.md`, коммит `2fdcca1` с тестами, коммит `2b95fe5` с кодом) и хода выполнения сессии.

### 2.1. Особенности и скрытые предположения в тестах
1. **Скрытый дефолт пробега в `AssessmentServiceTest`:**
   В методе `AssessmentServiceTest::payload()` сигнатура была изменена с `payload(int $amount, int $marketValue)` на `payload(int $amount, int $marketValue, int $mileage = 96000)`.
   *Анализ:* С одной стороны, это обеспечило обратную совместимость старых тестов LTV без переписывания каждого вызова. С другой стороны, значение `96000` стало скрытым допущением («магическим числом») для старых сценариев.
2. **Проверка `mileage` в существующих тестах `DecisionEngineTest`:**
   В существующем тесте `testDecidesByLtv` был добавлен аргумент `96000` (`$this->engine->decide($ltv, 96000)`). Это было неизбежно из-за отказа от опциональности второго аргумента в `decide()`, но формально изменило вызов в старом тесте ещё до реализации.

### 2.2. Особенности реализации в `DecisionEngine.php` и `rules.php`
1. **Самовольный рефакторинг модификаторов доступа (`readonly`):**
   В `DecisionEngine.php` агент не просто добавил `private readonly int $reviewMileageKm`, но попутно изменил уже существовавшие поля `$approveMax` и `$reviewMax` с `private float` на `private readonly float`.
   *Оценка:* Хотя это соответствует общим конвенциям проекта (`AGENTS.md §4: свойства через конструктор private readonly`), в плане задачи MILEAGE рефакторинг старых полей объявлен не был. Агент «прибрался» попутно.
2. **Строгая проверка `<` vs `<=` на границе 60.0:**
   Агент строго следовал указанию плана и спецификации не трогать известное расхождение на границе `60.0`:
   ```php
   if ($ltv < $this->approveMax) {
       return $mileage > $this->reviewMileageKm ? self::REVIEW : self::APPROVE;
   }
   ```
   Угол здесь **не был срезан** — агент удержался от соблазна исправить `<` на `<=`.
3. **Способ прокидывания зависимостей:**
   В `AppFactory.php` и `AssessmentServiceTest` порог передаётся явно скаляром `int`:
   `new DecisionEngine($rules['ltv'], $rules['vehicle']['review_mileage_km'])`.
   Это полностью соответствует плану D2 и изолирует `DecisionEngine` от прямого знания о структуре массива `$rules['vehicle']`.

### 2.3. Процессные моменты и сбои инфраструктуры
1. **Падение LLM-провайдеров и бюджета:**
   - По регламенту ДЗ код должен был писаться на модели `MiniMax M3`.
   - При первой попытке запуска агента произошёл отказ: исчерпание тренировочного баланса провайдера `stg-proxy` (`$5.016 > $5.00`).
   - При второй попытке через `kilo` был получен ответ `403 Forbidden` (модель заблокирована для шлюза).
   - Агент успешно адаптировался и запустил реализацию на той же модели `MiniMax M3`, но через провайдер `aitunnel/minimax-m3`. Требование ДЗ по модели было соблюдено, несмотря на инфраструктурный сбой.
2. **Локальное окружение:**
   - На машине хоста отсутствовали локально установленные `make`, `php` и `composer`.
   - Агент самостоятельно поднял демон Docker Desktop и перенаправил прогон тестов через Docker-контейнер (`docker compose run --rm backend ...`), сохранив валидность TDD-цикла.

### 2.4. Уровень закрытия HTTP-требований (доменные тесты vs `tests/Feature`)
1. **Отсутствие автоматических Feature-тестов:**
   В спецификации есть требования AC-04, AC-05, AC-08, упоминающие HTTP-статусы (200 и 422). В плане было принято решение: каталог `tests/Feature/` отсутствует в проекте (хотя секция в `phpunit.xml` заведена), поэтому автоматические тесты пишутся на уровне доменных классов (`AssessmentServiceTest`, `ApplicationValidatorTest`), а HTTP-уровень проверяется ручной дымовой проверкой через `curl`.
   *Критическая оценка:* Агент здесь формально следовал утверждённому плану (D3), однако с точки зрения абсолютной строгости требований спецификации («HTTP 200 на /api/ltv (не 422)», «HTTP 422») автоматический тест на `Slim App` через вызов `AppFactory::create()->handle(...)` был бы надёжнее ручного прогона curl. Это компромисс («срез угла»), зафиксированный ещё на этапе планирования.

### 2.5. Сводка расхождений между планом и фактом
* **План:** Добавить в `DecisionEngine` порог и метод `decide(float $ltv, int $mileage)`.  
  **Факт:** Выполнено, плюс добавлены модификаторы `readonly` к старым свойствам `$approveMax`, `$reviewMax`.
* **План:** Покрыть все AC автоматическими тестами.  
  **Факт:** Выполнено. В красной фазе тесты полностью отразили запланированную картину (2 assertion failures + warning).
* **План:** Обновить документацию `docs/setup/code_map.md`.  
  **Факт:** Выполнено в коммите зелёной фазы `2b95fe5` (обновлена диаграмма mermaid и убран подраздел о гипотетическом правиле).

---

### 2.6. Итог уровня ★★
**СТАТУС: ПРОЙДЕН (PASS).**  
Все расхождения носят конструктивный характер, не нарушают контрактов и архитектурных границ. Срез угла по Feature-тестам был осознанно задекларирован в плане и компенсирован полноценной верификацией на уровне оркестратора `AssessmentService` и ручной дымовой проверкой API в Docker-окружении. Инфраструктурные сбои (stg-proxy, kilo 403) были преодолены без отклонения от назначенной модели `MiniMax M3`.

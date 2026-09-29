** Dispatch Instructions **

If
User ask scout, search, find, найди, ищи, изучи in any language or similar words use subagent scout.md

If
User asks plan, planning, planner, планируй, запланируй in any language or similar words - use subagents planner.md

Ответ Scout:
Разведка по mileage завершена. Найдено 23 места в коде + 8 в docs/setup/code_map.md.

Сводная карта чтений пробега
Backend
backend/src/Domain/ApplicationValidator.php — единственная точка входа пробега в домен

L43 — чтение из payload: $mileage = (int) ($payload['mileage'] ?? -1);
L44–45 — валидация диапазона против rules['vehicle']['max_mileage_km'] и запись ошибки $errors['mileage']
L78 — запись нормализованного значения в возвращаемый массив
backend/src/Repository/ApplicationRepository.php

L38–39, L45 — INSERT в vehicles.mileage_km с плейсхолдером :mileage, привязка $input['mileage']
L68 — SELECT v.mileage_km в find()
backend/config/rules.php — L23: 'max_mileage_km' => 500000

AssessmentService / LtvCalculator / DecisionEngine — 0 обращений к mileage. В assess() значение лежит в $input['mileage'], но не передаётся дальше (см. замечание в code_map.md:78).

ApplicationController — 0 прямых обращений, работает через сервис/репозиторий.

Тесты — только фикстуры
tests/Unit/ApplicationValidatorTest.php:34 — 'mileage' => 84000
tests/Unit/AssessmentServiceTest.php:38 — 'mileage' => 96000
Отдельных тестов на валидацию/правила для пробега нет.
БД
db/schema.sql:22 — mileage_km INT UNSIGNED NOT NULL
db/seed.sql:31 — синтетические значения в INSERT
Frontend
frontend/index.html:30–31 — <input name="mileage">
frontend/app.js:8 — mileage в NUMERIC_FIELDS (приведение через Number())
Документация
README.md:63 — пример curl с "mileage":84000
docs/setup/code_map.md:26,31,65,70,78,81,95 — пояснения: где сейчас валидация, что пробег не доходит до DecisionEngine, и обсуждение порога 400 000 для решения (ещё не реализован)
Ключевые наблюдения
Односторонний поток: пробег читается → валидируется → пишется в БД, но не участвует ни в LTV, ни в decide() — DecisionEngine::decide(float $ltv) принимает только LTV.
Расхождение имён: mileage в PHP/JSON ↔ mileage_km в MySQL.
Разные пороги: валидационный max_mileage_km = 500 000 (в rules.php) vs. обсуждаемый порог 400 000 для решения (только в code_map.md, не реализован).
Если это запрос на план изменений (использовать пробег в решении, добавить правило 400 000, развести имена и т. п.) — уточни ID задачи, я составлю план в docs/plan/.
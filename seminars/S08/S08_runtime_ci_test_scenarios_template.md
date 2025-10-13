# S08 - Runtime CI Test Scenarios (template)

> Цель: подтвердить, что **CI запускается по push/PR**, корректно выполняет **init DB → pytest → upload artifacts**, а также что настроены **кэш** и **безопасная работа с секретами**.

## Как пользоваться

1. Отметьте чекбоксы у сценариев (минимум **5 базовых** + опциональные улучшения).
2. По каждому сценарию выполните шаги и **положите артефакт** в `EVIDENCE/S08/` (имя файла указано).
3. В `GRADING/DV.md` перечислите точные относительные пути к артефактам и кратко «что было → что сделали».

---

## Минимальный набор (чек-лист)

* [ ] **CI-01** Триггер на `push/PR` с URL рана (`ci-run.txt`)
* [ ] **CI-02** Шаги job выполнены в нужном порядке (`ci-log.txt`)
* [ ] **CI-03** Артефакт загружен и содержит JUnit (`artifacts.txt`, `test-report.xml`)
* [ ] **CI-04** JUnit валиден и не пустой (`junit-validate.txt`)
* [ ] **CI-05** Кэш pip сработал на повторном ране (`cache-proof.txt`)
* [ ] **SEC-01** В логах нет секретов (`secrets-proof.txt`)

*(Для ★★ добавьте ещё ≥2 из блока «Улучшения».)*

---

## Сценарии (GWT + шаги + артефакты)

### CI-01 - Триггер на push/PR

**Given** включён GitHub Actions, `ci.yml` в репозитории.
**When** делаем коммит и пушим (или открываем PR).
**Then** на вкладке Actions появился ран, есть URL.

**Шаги**

1. Сделайте небольшой коммит (например, правка README).
2. Откройте **Actions → последний ран**, скопируйте URL.

**Артефакт →** `EVIDENCE/S08/ci-run.txt`
(внутри одна строка - URL рана)

---

### CI-02 - Порядок и успех шагов job

**Given** ран завершён (зелёный или красный по тестам - допустимо).
**When** проверяем лог job.
**Then** присутствуют шаги: *Checkout → Setup Python → Cache pip → Install deps → Init DB → Run tests → Upload artifacts*.

**Шаги**

* Скопируйте из лога по абзацу на каждый шаг в текстовый файл (или сделайте выписку команд/результатов).

**Артефакт →** `EVIDENCE/S08/ci-log.txt`
(краткая выписка с названиями шагов и статусами)

---

### CI-03 - Артефакт загружен и содержит JUnit

**Given** шаг *Upload artifacts* есть в логе.
**When** скачиваем артефакт с Actions.
**Then** внутри есть `EVIDENCE/S08/test-report.xml`.

**Шаги**

1. На странице рана скачайте артефакт (например, `evidence-s08`).
2. Распакуйте локально и сохраните список файлов:

   ```bash
   unzip -l evidence-s08.zip > EVIDENCE/S08/artifacts.txt
   ```

3. Скопируйте из распакованного архива файл `EVIDENCE/S08/test-report.xml` в репозиторий **(можно как подтверждение; оригинал остаётся в артефакте).**

**Артефакты →**

* `EVIDENCE/S08/artifacts.txt`
* `EVIDENCE/S08/test-report.xml`

---

### CI-04 - Валидация JUnit

**Given** есть `test-report.xml`.
**When** проверяем структуру/ненулевой размер.
**Then** файл содержит `<testsuite`/`<testcase>`.

**Шаги (любой из вариантов)**

```bash
# Проверка размера
stat -c%s EVIDENCE/S08/test-report.xml > EVIDENCE/S08/junit-validate.txt 2>&1 || ^
# Grep по ключам (Linux/macOS)
( grep -q "<testsuite" EVIDENCE/S08/test-report.xml && echo OK ) >> EVIDENCE/S08/junit-validate.txt
```

**Артефакт →** `EVIDENCE/S08/junit-validate.txt`

---

### CI-05 - Кэш pip сработал (повторный ран быстрее)

**Given** настроен `actions/cache@v4`.
**When** запускаем второй ран без изменения `requirements.txt`.
**Then** в логе видно восстановление кэша, а длительность шага `Install deps` меньше.

**Шаги**

1. Сделайте ещё один пустой коммит (например, правка пробела в README).
2. Зафиксируйте длительность шага `Install deps` в обоих ранах и строки `Cache restored`/`Cache not found`.

**Формат артефакта (ручная заметка)**

```bash
run1: 2025-10-07T12:34Z  Install deps: 38s  Cache: not found
run2: 2025-10-07T12:40Z  Install deps: 9s   Cache: restored
```

**Артефакт →** `EVIDENCE/S08/cache-proof.txt`

---

### SEC-01 - Гигиена секретов в CI (DV2)

**Given** секреты (если нужны) заведены через Actions Secrets/Variables.
**When** просматриваем лог job.
**Then** значений секретов в явном виде **нет** (маскированы `***` или не логируются).

**Шаги**

* Скопируйте 3-5 строк лога, где могли бы печататься секреты (и где видно маскирование/отсутствие).
* Если секретов нет - кратко напишите «секреты не используются, .env.example в репо».

**Артефакт →** `EVIDENCE/S08/secrets-proof.txt`

---

## Улучшения (выберите ≥2 для ★★)

### CI-06 - Concurrency: отмена старых ранов по ветке

**Given** включён блок `concurrency` в workflow.
**When** пушим 2 коммита подряд.
**Then** первый ран отменён, второй идёт.

**Артефакт →** `EVIDENCE/S08/concurrency.txt`
(вставьте URL обоих ранов и строку *Cancelled* первого)

---

### CI-07 - Paths-ignore: «док-коммиты» не триггерят CI

**Given** в `on.push.paths-ignore` указаны `EVIDENCE/**`, `README.md`.
**When** коммитим правку только `README.md`.
**Then** CI не стартует.

**Артефакт →** `EVIDENCE/S08/paths-ignore.txt`
(ссылка на коммит + заметка: «Actions не запустился»)

---

### CI-08 - Красный ран «по делу»

**Given** искусственно «роняем» один тест (например, временный `assert False`).
**When** CI становится красным.
**Then** в DV пояснение «почему красный», в артефактах - JUnit с проваленным тестом.

**Артефакты →** `EVIDENCE/S08/test-report.xml` (с провалом), `ci-log.txt` (фрагмент ошибки)

---

### CI-09 - Покрытие тестов

**Given** добавлен `--cov=app --cov-report xml:coverage.xml`.
**When** CI загружает `coverage.xml` как артефакт.
**Then** файл присутствует.

**Артефакт →** `EVIDENCE/S08/coverage.xml` *(или сохраните в артефакт и зафиксируйте в `artifacts.txt`)*

---

### CI-10 - Матрица Python-версий

**Given** job с matrix (напр., 3.10/3.11).
**When** оба ран завершились.
**Then** в логе видны две конфигурации.

**Артефакт →** `EVIDENCE/S08/matrix-proof.txt`
(выписка из лога с обеими версиями)

---

## Индекс артефактов (сошлитесь в `GRADING/DV.md`)

* `EVIDENCE/S08/ci-run.txt`
* `EVIDENCE/S08/ci-log.txt`
* `EVIDENCE/S08/artifacts.txt`
* `EVIDENCE/S08/test-report.xml`
* `EVIDENCE/S08/junit-validate.txt`
* `EVIDENCE/S08/cache-proof.txt`
* *(улучшения)* `EVIDENCE/S08/concurrency.txt`, `paths-ignore.txt`, `coverage.xml`, `matrix-proof.txt`, *(при красном ране)* обновлённый `test-report.xml`

---

## Как это положить в `GRADING/DV.md`

* **DV2 (секреты):** ссылка на `secrets-proof.txt` + где хранятся Secrets/Variables.
* **DV4 (артефакты/логи):** перечислите все файлы из индекса выше + URL рана.
* **DV5 (инструкции):** 3-5 строк в README «Как устроен CI» (когда триггерится, основные шаги, где смотреть артефакты).

---

## Definition of Done (минимум / проектный)

**Минимум (★1):**

* [ ] Есть `ci-run.txt`, `ci-log.txt`, `test-report.xml`, `artifacts.txt`, `junit-validate.txt`.
* [ ] Кэш подтверждён: `cache-proof.txt`.
* [ ] Секреты не утекли: `secrets-proof.txt`.
* [ ] DV обновлён (DV2/DV4/DV5) и README дописан.

**Проектный (★★2):**

* [ ] Добавлены ≥2 улучшения (concurrency/paths-ignore/coverage/matrix).
* [ ] Все артефакты доступны по ссылкам; логи аккуратно оформлены.

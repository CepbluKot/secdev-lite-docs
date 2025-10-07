# S08 - CI Checklist (template)

> Цель: включить и зафиксировать **минимальный CI** (build+test → artifacts) для вашего репозитория из seed v2, так чтобы проверяющий мог воспроизвести шаги и найти все пруфы.

## Как пользоваться этим файлом

1. Идите сверху вниз, ставьте галочки **[ ] → [x]**, дописывайте недостающее.
2. Все **артефакты CI** и ключевые ссылки кладите/индексируйте в `EVIDENCE/S08/`.
3. В конце заполните «Сводку для `GRADING/DV.md`» и перенесите туда краткие формулировки.

---

## Паспорт выполнения (заполнить)

* Репозиторий: **ТУТ НУЖНА ТВОЯ ВСТАВКА**
* Платформа CI: **GitHub Actions**
* Версия Python (раннер): **3.11** *(или своя)*
* Ссылка на последний удачный ран: `EVIDENCE/S08/ci-run.txt`
* Основной JUnit-отчёт: `EVIDENCE/S08/test-report.xml` *(из артефактов)*

---

## A. Триггеры и раннеры

* [ ] **on: push** включён (по умолчанию - на все ветки).
* [ ] **on: pull_request** включён (минимум для `main`).
* [ ] (Опц.) `paths-ignore` настроен, чтобы не гонять CI на чисто документных коммитах (см. сниппет ниже).
* [ ] Раннер: `ubuntu-latest`.

**Сниппет (добавьте/проверьте):**

```yaml
on:
  push:
    paths-ignore:
      - 'EVIDENCE/**'
      - 'README.md'
  pull_request:
    paths-ignore:
      - 'EVIDENCE/**'
      - 'README.md'
```

---

## B. Python & cache pip

* [ ] `actions/setup-python@v5` с вашей версией Python.
* [ ] Кэш pip включён и **сработал на втором ране** (видно по логу).

**Сниппет:**

```yaml
- uses: actions/setup-python@v5
  with:
    python-version: '3.11'

- uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
    restore-keys: ${{ runner.os }}-pip-
```

---

## C. Build → Init DB → Pytest

* [ ] Установка зависимостей: `pip install -r requirements.txt`.
* [ ] Инициализация БД: `python scripts/init_db.py`.
* [ ] Тесты: `pytest -q --junitxml=EVIDENCE/S08/test-report.xml`.
* [ ] (Опц.) Покрытие: `--cov=app --cov-report xml:coverage.xml`.

**Сниппет:**

```yaml
- name: Install deps
  run: pip install -r requirements.txt

- name: Init DB
  run: python scripts/init_db.py

- name: Run tests
  run: pytest -q --junitxml=EVIDENCE/S08/test-report.xml
```

---

## D. Артефакты CI

* [ ] Загрузка артефактов **всегда** (`if: always()`), даже при падении тестов.
* [ ] В артефактах есть `EVIDENCE/S08/test-report.xml`.
* [ ] (Опц.) Добавлен `coverage.xml` или zip с HTML.

**Сниппет:**

```yaml
- name: Upload artifacts
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: evidence-s08
    path: |
      EVIDENCE/S08/**
      coverage.xml
    if-no-files-found: warn
```

---

## E. Гигиена секретов (DV2)

* [ ] **Секреты не захардкожены** в YAML/логах.
* [ ] Все значения, требующие приватности, заведены через **Actions → Secrets/Variables**.
* [ ] Лог ранa проверен: секреты **не печатаются** (нет эха значений).
* [ ] `.env` не коммитим; в репозитории только `.env.example`.

**Памятка:** для передачи в шаг используйте

```yaml
env:
  MY_TOKEN: ${{ secrets.MY_TOKEN }}
```

и не логируйте `${{ secrets.* }}`.

---

## F. Ускорители и защита от гонок (рекомендуем)

* [ ] **concurrency** включает отмену старых ранов по ветке:

```yaml
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
```

* [ ] (Опц.) Матрица Python-версий (если хотите проверить 3.10/3.11/3.12).
* [ ] (Опц.) `fail-fast: false` в матрице, если важно собрать все результаты.
* [ ] (Опц.) `timeout-minutes` для джоба (напр., `15`).

---

## G. Проверка и сбор пруфов (что положить в `EVIDENCE/S08/`)

* [ ] `ci-run.txt` - **URL** последнего рана (скопируйте руками).
* [ ] `test-report.xml` - JUnit-отчёт скачан из артефакта.
* [ ] *(опц.)* `ci-log.txt` - выписка ключевых строк лога (Checkout/Setup/Cache/Init/pytest/Upload).
* [ ] *(опц.)* `artifacts.txt` - список файлов внутри артефакта.
* [ ] *(опц.)* `coverage.xml` - отчёт покрытия, если включали.

**Минимум для зачёта S08:** `ci-run.txt` **+** `test-report.xml` (или `ci-log.txt`).

---

## H. README и прозрачность (DV5)

* [ ] Добавлен раздел **«Как устроен CI»** (3-5 строк):
  *когда триггерится, какие шаги, где смотреть артефакты и логи.*
* [ ] Кратко указано, как воспроизвести локально (ссылка на one-liner S06).
* [ ] (Опц.) Отметка про кэш: «второй ран быстрее за счёт pip-cache».

---

## I. Сводка для переноса в `GRADING/DV.md` (заполнить)

* **DV2 (секреты):** *ТУТ НУЖНА ТВОЯ ВСТАВКА* - где храним секреты в Actions; подтверждение, что YAML/лог не содержит значений.
* **DV4 (артефакты/логи):** ссылки и файлы:

  * URL рана → `EVIDENCE/S08/ci-run.txt`
  * JUnit → `EVIDENCE/S08/test-report.xml`
  * *(опц.)* `ci-log.txt`, `artifacts.txt`, `coverage.xml`
* **DV5 (инструкции):** 3-5 строк в README «Как устроен CI» + где смотреть артефакты.

---

## Definition of Done (минимум / проектный)

**Минимум (★1):**

* [ ] Workflow триггерится на push/PR и проходит install → init DB → pytest.
* [ ] Артефакты доступны; в репозитории лежит **индекс** (`EVIDENCE/S08/ci-run.txt` + `test-report.xml`).
* [ ] `GRADING/DV.md` и README обновлены (DV2/DV4/DV5).

**Проектный (★★2):**

* [ ] Кэш pip заметно ускоряет повторный ран (отмечено в DV).
* [ ] Включены `concurrency` и `paths-ignore`.
* [ ] Подключено покрытие (и, опц., матрица Python).

---

### Приложение: Мини-скелет `.github/workflows/ci.yml` (для сверки)

```yaml
name: ci
on: [push, pull_request]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.11' }
      - uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
          restore-keys: ${{ runner.os }}-pip-
      - name: Install deps
        run: pip install -r requirements.txt
      - name: Init DB
        run: python scripts/init_db.py
      - name: Run tests
        run: pytest -q --junitxml=EVIDENCE/S08/test-report.xml
      - name: Upload artifacts
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: evidence-s08
          path: EVIDENCE/S08/**
          if-no-files-found: warn
```

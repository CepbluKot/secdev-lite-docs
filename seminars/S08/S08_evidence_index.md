# S08 - Evidence Index (манифест артефактов CI)

> Цель: чтобы проверяющий за 1-2 клика нашёл все подтверждения работы CI.
> Все пути ниже **относительные к корню репозитория команды**.

## Где лежит

```text
EVIDENCE/
└── S08/
    ├── ci-run.txt            # URL последнего (или эталонного) рана
    ├── ci-log.txt            # короткая выписка ключевых шагов
    ├── artifacts.txt         # список файлов внутри CI-артефакта
    ├── test-report.xml       # JUnit из артефакта CI
    ├── junit-validate.txt    # проверка, что JUnit валидный/не пустой
    ├── cache-proof.txt       # доказ-во, что pip-cache сработал (2-й ран)
    ├── coverage.xml          # (опц.) отчёт покрытия
    ├── concurrency.txt       # (опц.) отмена старых ранов по ветке
    ├── paths-ignore.txt      # (опц.) док-коммит не триггерит CI
    └── matrix-proof.txt      # (опц.) доказ-во работы матрицы версий
```

---

## Обязательные артефакты (минимум)

**1) URL рана CI**

* **Как получить:** откройте вкладку *Actions* → нужный ран → скопируйте URL.
* **Сохранить в файл:**

  ```bash
  echo "https://github.com/<org>/<repo>/actions/runs/<ID>" > EVIDENCE/S08/ci-run.txt
  ```

* **Файл:** `EVIDENCE/S08/ci-run.txt`

**2) JUnit-отчёт тестов**

* **Как получить:** скачайте CI-артефакт (например, `evidence-s08`) со страницы рана и извлеките оттуда файл:

  * `EVIDENCE/S08/test-report.xml` - положите прямо в репозиторий как «пруф».
* **Файл:** `EVIDENCE/S08/test-report.xml`

**3) Краткая выписка лога job (ключевые шаги)**

* **Что включить:** названия шагов и статусы: *Checkout → Setup Python → Cache pip → Install deps → Init DB → Run tests → Upload artifacts*.
* **Файл:** `EVIDENCE/S08/ci-log.txt`

**4) Индекс содержимого артефакта**

* **Как получить (локально):**

  ```bash
  unzip -l evidence-s08.zip > EVIDENCE/S08/artifacts.txt
  ```

* **Файл:** `EVIDENCE/S08/artifacts.txt`

**5) Валидация JUnit (не пустой, есть теги)**

* **Как проверить (любой способ):**

  ```bash
  # размер
  stat -c%s EVIDENCE/S08/test-report.xml > EVIDENCE/S08/junit-validate.txt 2>&1 || ^
  # ключевые теги (Linux/macOS)
  ( grep -q "<testsuite" EVIDENCE/S08/test-report.xml && echo OK ) >> EVIDENCE/S08/junit-validate.txt
  ```

* **Файл:** `EVIDENCE/S08/junit-validate.txt`

---

## Рекомендуемые (для ★★ или по желанию)

**A) Доказательство кэша pip (2-й ран быстрее)**

* **Что записать:** две строки вида
  `run1: <ts> Install deps: 38s  Cache: not found`
  `run2: <ts> Install deps: 9s   Cache: restored`
* **Файл:** `EVIDENCE/S08/cache-proof.txt`

**B) Покрытие**

* **Если в CI добавлен `--cov=app --cov-report xml:coverage.xml`:**
  заберите `coverage.xml` из артефакта.
* **Файл:** `EVIDENCE/S08/coverage.xml`

**C) Concurrency (отмена старых ранов)**

* **Что записать:** URL двух ранов; первый в статусе *Cancelled*.
* **Файл:** `EVIDENCE/S08/concurrency.txt`

**D) Paths-ignore (док-коммиты не стартуют CI)**

* **Что записать:** ссылка на коммит (например, правка `README.md`) + заметка «Actions не запустился».
* **Файл:** `EVIDENCE/S08/paths-ignore.txt`

**E) Матрица Python-версий**

* **Что записать:** выписка из лога с обеими версиями (напр., 3.10 и 3.11).
* **Файл:** `EVIDENCE/S08/matrix-proof.txt`

---

## Быстрые команды и подсказки

**Скачать и индексировать артефакт:**

```bash
# На странице рана скачайте artifact zip → положите рядом
unzip -l evidence-s08.zip > EVIDENCE/S08/artifacts.txt
# Извлеките test-report.xml и скопируйте в EVIDENCE/S08/
```

**Мини-проверки JUnit:**

```bash
grep -q "<testcase" EVIDENCE/S08/test-report.xml && echo "OK" >> EVIDENCE/S08/junit-validate.txt
```

**Замечания по секретам (DV2):**

* Секреты и приватные значения не должны появляться в `ci-log.txt`.
* Если секретов нет - явно напишите в конце файла `ci-log.txt` одну строку:
  `No secrets used. .env.example only.`

---

## Что указать в `GRADING/DV.md`

* **DV2 (гигиена секретов):** где хранятся Secrets/Variables; ссылка на `EVIDENCE/S08/ci-log.txt` (нет утечек).
* **DV4 (артефакты/логи):** перечислите **точно по именам**:
  `EVIDENCE/S08/ci-run.txt`, `test-report.xml`, `artifacts.txt`, `ci-log.txt`, `junit-validate.txt`
  *(и, если есть)* `cache-proof.txt`, `coverage.xml`, `concurrency.txt`, `paths-ignore.txt`, `matrix-proof.txt`.
* **DV5 (инструкции):** в README 3-5 строк «Как устроен CI» (когда триггерится, какие шаги, где лежат артефакты и как их скачать).

---

## Правила именования (чтобы не было каши)

* Только **ASCII** в именах; без пробелов.
* Логи - `.txt`, отчёты - `.xml`, архивы - `.zip`.
* Для нескольких прогонов допускайте суффиксы `-v2`, `-v3` (укажите в DV, какой смотреть).

---

## Definition of Done (S08 Evidence)

* [ ] Присутствуют обязательные файлы: `ci-run.txt`, `test-report.xml`, `ci-log.txt`, `artifacts.txt`, `junit-validate.txt`.
* [ ] Секреты не утекли (упоминание/пруф в `ci-log.txt`).
* [ ] (Для ★★) добавлены ≥2 рекомендованных: `cache-proof.txt` / `coverage.xml` / `concurrency.txt` / `paths-ignore.txt` / `matrix-proof.txt`.
* [ ] `GRADING/DV.md` обновлён ссылками на **точные** пути из списка выше; README пополнен разделом «Как устроен CI».

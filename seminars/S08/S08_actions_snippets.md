# S08 - Actions Snippets (готовые блоки YAML)

> Копируйте нужные куски в `.github/workflows/ci.yml`. Все пути и имена артефактов согласованы с `EVIDENCE/S08/`.

## Как использовать

1. Откройте свой `ci.yml` и вставьте выбранные блоки в соответствующие места.
2. Проверьте выравнивание отступов YAML.
3. Сделайте маленький коммит → убедитесь, что ран запустился и зелёный.

---

## 1) Триггеры (push/PR) + игнор документных правок

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
  workflow_dispatch: {}
```

---

## 2) Concurrency (отмена старых ранов по ветке)

```yaml
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
```

---

## 3) Минимальные permissions (рекомендуется)

```yaml
permissions:
  contents: read
  actions: read
  # packages: write   # только если публикуете образы/пакеты (опционально)
```

---

## 4) Матрица версий Python (опционально)

```yaml
jobs:
  build-test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        python-version: ['3.11', '3.12']
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: ${{ matrix.python-version }} }
      # …дальше обычные шаги
```

---

## 5) Кэш pip (ускорение повторных ранов)

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
    restore-keys: ${{ runner.os }}-pip-
```

---

## 6) Установка зависимостей, init DB, pytest → JUnit в `EVIDENCE/S08/`

```yaml
- name: Install deps
  run: pip install -r requirements.txt

- name: Init DB
  run: python scripts/init_db.py

- name: Run tests
  run: pytest -q --junitxml=EVIDENCE/S08/test-report.xml
```

---

## 7) Загрузка артефактов (всегда, даже если тесты упали)

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
    retention-days: 14
```

---

## 8) Покрытие (опционально)

```yaml
- name: Run tests w/ coverage
  run: pytest -q --junitxml=EVIDENCE/S08/test-report.xml --cov=app --cov-report=xml:coverage.xml

# (добавьте coverage.xml в блок Upload artifacts - см. выше)
```

---

## 9) Таймаут job и среда

```yaml
jobs:
  build-test:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    env:
      PYTHONUNBUFFERED: "1"
```

---

## 10) Условные шаги (только для PR / только для default branch)

```yaml
# Только на pull_request:
- name: Lint on PR
  if: ${{ github.event_name == 'pull_request' }}
  run: echo "lint placeholder"

# Только на default-ветке (обычно main):
- name: Extra checks on default branch
  if: ${{ github.ref == 'refs/heads/main' }}
  run: echo "extra checks"
```

---

## 11) Публикация HTML-отчёта как артефакта (если нужен)

```yaml
- name: Package HTML report
  run: |
    mkdir -p EVIDENCE/S08
    zip -r EVIDENCE/S08/html-report.zip htmlcov || true

# уже попадёт в Upload artifacts по маске EVIDENCE/S08/**
```

---

## 12) Docker build (локальная проверка в CI, без публикации) - *опционально*

```yaml
- name: Docker build (no push)
  run: |
    docker build -t secdev-seed:ci . | tee EVIDENCE/S08/build.log
    docker image ls secdev-seed:ci --format "{{.Repository}} {{.Tag}} {{.Size}}" > EVIDENCE/S08/image-size.txt
```

---

## 13) Login в GHCR и push образа - *НЕ обязательно для лайт-трека*

> Подходит тем, кто хочет ★★ и понимает риски. Нужны `GHCR_TOKEN` (Classic token с scope `write:packages`) и `GHCR_USER`.

```yaml
- name: Login to GHCR
  if: ${{ github.ref == 'refs/heads/main' }}
  run: echo "${{ secrets.GHCR_TOKEN }}" | docker login ghcr.io -u "${{ secrets.GHCR_USER }}" --password-stdin

- name: Build & push image
  if: ${{ github.ref == 'refs/heads/main' }}
  run: |
    IMAGE=ghcr.io/${{ github.repository_owner }}/secdev-seed:${{ github.sha }}
    docker build -t "$IMAGE" .
    docker push "$IMAGE"
```

---

## 14) Секреты безопасно (DV2)

```yaml
# Пример передачи секрета в шаг:
- name: Use secret safely
  env:
    API_TOKEN: ${{ secrets.API_TOKEN }}
  run: |
    python -c "import os; print('token length:', len(os.getenv('API_TOKEN','')))"
# Никогда не печатайте значения секретов!
```

---

## 15) Готовый минимальный каркас (соберите пазл)

```yaml
name: ci
on:
  push:
    paths-ignore: ['EVIDENCE/**','README.md']
  pull_request:
    paths-ignore: ['EVIDENCE/**','README.md']
  workflow_dispatch: {}

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read
  actions: read

jobs:
  build-test:
    runs-on: ubuntu-latest
    timeout-minutes: 15
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
          path: |
            EVIDENCE/S08/**
          if-no-files-found: warn
          retention-days: 14
```

---

## Подсказки и анти-паттерны

* ❌ Не хардкодьте секреты в YAML/коде; используйте `secrets.*`.
* ❌ Не печатайте секреты в лог (даже частично).
* ✔️ Для DV4 достаточно JUnit + индекс ссылок на ран/артефакты.
* ✔️ Для DV5 добавьте в README 3-5 строк «Как устроен CI» (триггеры, шаги, где артефакты).

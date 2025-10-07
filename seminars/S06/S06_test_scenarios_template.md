# S06 - Test Scenarios Template

> Цель: быстро оформить 4-6 автотестов, которые подтверждают ваши фиксы из `S06_patch_cards.md`, а также **пишут отчёт** в `EVIDENCE/S06/test-report.xml` для DV.

## Как этим пользоваться

1. Выберите **минимум 2 карточки** из `S06_patch_cards.md`.
2. На каждую карточку добавьте **≥1 автотест** (лучше 2: позитив + негатив).
3. Прогоняйте локально одной командой (**one-liner** для DV1).
4. Складывайте отчёты и логи в `EVIDENCE/S06/` и дайте ссылки в `GRADING/DV.md`.

---

## Нотация сценариев

Выберите удобную форму - **Given-When-Then** или **pytest-сценарий**. Внутри репозитория тесты реализуются на pytest.

**Шаблон GWT (заполняйте под себя):**

```Gherkin
Сценарий: <краткое имя>
Given:   <исходное состояние/данные/пользователь>
When:    <действие/запрос>
Then:    <ожидаемый результат/код/фрагмент ответа/заголовки>
Artifacts: EVIDENCE/S06/<путь к отчёту/скрину/логу>
Связано с: DV1/DV3/DV4 (указать релевантные)
```

**Шаблон pytest (скелет):**

```python
import pytest
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_<name>():
    # Arrange / Given
    # ТУТ НУЖНА ТВОЯ ВСТАВКА (инициализация/данные)
    # Act / When
    resp = client.get("/ТУТ НУЖНА ТВОЯ ВСТАВКА")
    # Assert / Then
    assert resp.status_code == 200
    # ТУТ НУЖНА ТВОЯ ВСТАВКА (конкретные проверки)
```

---

## Обязательный минимум (под seed FastAPI + SQLite + Jinja2)

Добавьте тесты так, чтобы они подтверждали фиксы. Ниже - заготовки под каждую карточку.

### 1) SQL Injection (login) - обязателен, если брали S06-01

**GWT**

```Gherkin
Сценарий: Логин не уязвим к SQLi-комментарию
Given:   Пользователь "admin" в БД
When:    POST /login с username="admin'-- " и любым password
Then:    401 Unauthorized
Artifacts: EVIDENCE/S06/test-report.xml
Связано с: DV3, DV4
```

**pytest**

```python
def test_login_sql_injection_blocked():
    payload = {"username": "admin'-- ", "password": "x"}
    resp = client.post("/login", json=payload)
    assert resp.status_code == 401
```

### 2) SQL Injection (search LIKE) - если брали S06-02

**GWT**

```Gherkin
Сценарий: Инъекция в LIKE не возвращает все записи
Given:   Предзаполненная таблица items
When:    GET /search?q=' OR '1'='1
Then:    Количество результатов не больше, чем для "шумового" запроса ("zzzzzz")
Artifacts: EVIDENCE/S06/test-report.xml
Связано с: DV3, DV4
```

**pytest**

```python
def test_search_like_injection_blocked():
    noise = client.get("/search", params={"q": "zzzzzz"}).json()
    inj   = client.get("/search", params={"q": "' OR '1'='1"}).json()
    assert len(inj["items"]) <= len(noise["items"])
```

### 3) XSS (экранирование вывода) - если брали S06-03

**GWT**

```Gherkin
Сценарий: Отражённая XSS не исполняется
Given:   Запуск /echo
When:    GET /echo?msg=<script>alert(1)</script>
Then:    В ответе нет тега <script>, контент экранирован
Artifacts: EVIDENCE/S06/test-report.xml; при желании - скрин ответа
Связано с: DV3, DV4
```

**pytest**

```python
def test_echo_escapes_script_tags():
    resp = client.get("/echo", params={"msg": "<script>alert(1)</script>"})
    assert "<script>" not in resp.text
```

### 4) Валидация ввода - если брали S06-04

**GWT**

```Gherkin
Сценарий: Слишком длинный логин отклоняется
Given:   Ограничения модели (например 3..48 символов)
When:    POST /login с username длиной > 200
Then:    422 Unprocessable Entity
Artifacts: EVIDENCE/S06/test-report.xml
Связано с: DV3, DV4
```

**pytest**

```python
def test_login_username_too_long():
    long_user = "u" * 256
    payload = {"username": long_user, "password": "x"}
    resp = client.post("/login", json=payload)
    assert resp.status_code in (400, 422)
```

### 5) Ошибки и логи без утечек - если брали S06-05

**GWT**

```Gherkin
Сценарий: Внутренняя ошибка не раскрывает детали
Given:   Обработчик ошибок включён
When:    Инициируем исключение (спровоцировать контролируемо)
Then:    Код 500 и "безопасное" тело без стек-трейса/секретов
Artifacts: EVIDENCE/S06/logs/app.log (если есть), test-report.xml
Связано с: DV3, DV4, DV5
```

**pytest (набросок)**

```python
def test_internal_error_is_sanitized(monkeypatch):
    from app import main
    def boom(*a, **kw): raise RuntimeError("boom")
    monkeypatch.setattr(main, "query", boom, raising=True)
    resp = client.get("/search", params={"q":"apple"})
    assert resp.status_code in (500, 200)  # Зависит от вашей реализации
    # ТУТ НУЖНА ТВОЯ ВСТАВКА: проверка "безопасного" тела/логов
```

### 6) Безопасные заголовки - если брали S06-06

**GWT**

```Gherkin
Сценарий: Ответы содержат минимальный набор security-заголовков
Given:   Middleware для заголовков включено
When:    GET /
Then:    Заголовки: X-Content-Type-Options=nosniff; X-Frame-Options=DENY; Referrer-Policy=no-referrer; CSP=default-src 'self'
Artifacts: EVIDENCE/S06/test-report.xml
Связано с: DV3, DV4
```

**pytest**

```python
def test_security_headers_present():
    resp = client.get("/")
    h = resp.headers
    assert h.get("X-Content-Type-Options") == "nosniff"
    assert h.get("X-Frame-Options") == "DENY"
    assert h.get("Referrer-Policy") == "no-referrer"
    assert "default-src 'self'" in (h.get("Content-Security-Policy") or "")
```

### 7) Гигиена секретов/конфигов - если брали S06-07

**GWT**

```Gherkin
Сценарий: Приложение стартует без .env и читает конфиг из env
Given:   Нет файла .env в репо
When:    Устанавливаем переменную окружения и запускаем app
Then:    Значение доступно в приложении; .env не требуется
Artifacts: EVIDENCE/S06/test-report.xml; снимок .git status (без секретов)
Связано с: DV2, DV5
```

### 8) Anti-bruteforce (лайт) - если брали S06-08

**GWT**

```Gherkin
Сценарий: Лимит попыток логина
Given:   Лимитер включён (например, 5 попыток/мин)
When:    6 неуспешных логинов подряд
Then:    429 Too Many Attempts
Artifacts: EVIDENCE/S06/test-report.xml
Связано с: DV3, DV4
```

**pytest (набросок)**

```python
def test_login_rate_limit():
    for _ in range(5):
        r = client.post("/login", json={"username":"admin","password":"bad"})
        assert r.status_code in (401, 429)
    r6 = client.post("/login", json={"username":"admin","password":"bad"})
    assert r6.status_code == 429
```

---

## Как запускать и куда писать отчёт

**Рекомендуемая команда (пример для DV1):**

```bash
pytest -q --junitxml=EVIDENCE/S06/test-report.xml
```

* ТУТ НУЖНА ТВОЯ ВСТАВКА: вы можете упаковать это в `make ci`/скрипт и использовать как **one-liner**.
* При необходимости скриншоты и логи кладите в:

  * `EVIDENCE/S06/screenshots/…`
  * `EVIDENCE/S06/logs/…`
  * `EVIDENCE/S06/patches/*.diff`

---

## Что сослать в GRADING/DV.md

* **DV1 (one-liner):** точную команду (или `make ci`) + ОС/версию языка.
* **DV3 (фиксы кода):** список карточек, коротко «что было → что сделали → где тест».
* **DV4 (артефакты):** пути к `EVIDENCE/S06/test-report.xml`, `logs/*`, `screenshots/*`.
* **DV5 (инструкции):** 2-5 строк «Локальный запуск» в `README` с ссылкой на one-liner.

---

## Definition of Done (чек перед сдачей)

* [ ] Есть **≥4 теста** (лучше 6), минимум по выбранным карточкам.
* [ ] Тесты **зелёные** после фиксов.
* [ ] Отчёт JUnit лежит в `EVIDENCE/S06/test-report.xml`.
* [ ] В `GRADING/DV.md` - ссылки на артефакты и краткие комментарии по фиксам.
* [ ] Секреты не закоммичены; `.env.example` присутствует.

---

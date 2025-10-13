# S06 - Patch Cards (карточки патчей)

> Выберите **минимум 2** карточки. По каждой: реализуйте фикс → добавьте/обновите тесты → положите отчёты/скрины/логи в `EVIDENCE/S06/` → отразите в `GRADING/DV.md` (DV1/DV3/DV4/DV5).

**Как отмечать прогресс:** поставьте `✔` в чекбокс и дайте короткий комментарий «что было → что сделали → где тест/пруф».

---

## 🟦 S06-01 - SQL Injection (login)

* [ ] **Проблема:** `POST /login` собирает SQL через f-строку → байпас `username="admin'-- "`.
* **Где:** `app/main.py` (`login`), `app/db.py`.
* **Исправление (шаги):**

  1. В `app/db.py` добавьте параметризированный вызов (простейший вариант):

     ```python
     def query_one_params(sql: str, params: tuple):
         with get_conn() as conn:
             row = conn.execute(sql, params).fetchone()
             return dict(row) if row else None
     ```

  2. В `login` замените строку на параметризованный запрос:

     ```python
     sql = "SELECT id, username FROM users WHERE username = ? AND password = ?"
     row = query_one_params(sql, (payload.username, payload.password))
     ```

  3. (Опционально) Добавьте ограничение длины/формата логина в модели (см. S06-04).
* **Тесты:** существующий `tests/test_login_sql_injection.py` должен стать зелёным; добавьте позитивный кейс «правильный логин/пароль → 200».
* **Пруфы:** `EVIDENCE/S06/test-report.*`, `EVIDENCE/S06/patches/login_sqli.diff` (опц.), скрин до/после.
* **DV:** DV3 (фиксы кода), DV4 (артефакты тестов), DV5 (2-5 строк в README «Локальный запуск»), DV1 (one-liner).

---

## 🟦 S06-02 - SQL Injection (search LIKE)

* [ ] **Проблема:** `/search` подставляет `q` в `LIKE '%{q}%'` → инъекция `' OR '1'='1`.
* **Где:** `app/main.py` (`search`).
* **Исправление (шаги):**

  1. Поменяйте на параметризованный запрос:

     ```python
     sql = "SELECT id, name, description FROM items WHERE name LIKE ?"
     pattern = f"%{q}%"
     items = query("SELECT id, name, description FROM items WHERE name LIKE ?", (pattern,))
     ```

     Если оставляете отдельный helper - добавьте `query_params(sql, params)` в `db.py`.
  2. (Опц.) Ограничьте `q` по длине (см. S06-04), чтобы не «взрывать» план.
* **Тесты:** `tests/test_search_sql_like.py` станет зелёным; добавьте тест «длинный q → 422/400».
* **Пруфы:** отчёт тестов, diff, скрин «инъекция → не все записи».
* **DV:** DV3, DV4, DV1.

---

## 🟦 S06-03 - XSS (экранирование вывода)

* [ ] **Проблема:** `templates/index.html` рендерит `{{ message|safe }}` → отражённая XSS через `/echo?msg=...`.
* **Где:** `app/templates/index.html`.
* **Исправление (шаги):**

  1. Уберите `|safe`, положитесь на автоэкранирование Jinja2:

     ```jinja2
     <div id="message">{{ message }}</div>
     ```

  2. (Опц.) Примите только текст в обработчиках (`str`), отбросьте HTML.
* **Тесты:** `tests/test_xss_escaping.py` должен стать зелёным; добавьте негатив «`<img onerror=...>` не исполняется».
* **Пруфы:** отчёт тестов; до/после HTML-фрагменты.
* **DV:** DV3, DV4.

---

## 🟦 S06-04 - Валидация ввода (строгость и ограничения)

* [ ] **Проблема:** слишком «широкие» поля (логин/пароль/поиск). Риск DoS/инъекций/ошибок.
* **Где:** `app/models.py` (`LoginRequest`), `app/main.py` (`search`).
* **Исправление (шаги):**

  1. Ужесточите `LoginRequest`:

     ```python
     from pydantic import BaseModel, constr
     Username = constr(strip_whitespace=True, min_length=3, max_length=48, pattern=r"^[a-zA-Z0-9_.-]+$")
     Password = constr(min_length=3, max_length=128)
     class LoginRequest(BaseModel):
         username: Username
         password: Password
     ```

  2. Ограничьте `q` в обработчике:

     ```python
     from fastapi import Query
     def search(q: str | None = Query(default=None, min_length=1, max_length=32)):
         ...
     ```

* **Тесты:** добавьте «слишком длинный/плохой логин → 422», «слишком длинный q → 422».
* **Пруфы:** отчёт, скрин 422, ссылку на commit/дифф.
* **DV:** DV3, DV4.

---

## 🟦 S06-05 - Ошибки и логи без утечек

* [ ] **Проблема:** стек-трейсы/SQL-ошибки могут утечь в ответы/логи; секреты могут попасть в логи.
* **Где:** `app/main.py` (общий), корень проекта (логгер).
* **Исправление (шаги):**

  1. Добавьте обработчик ошибок:

     ```python
     from fastapi.responses import JSONResponse
     from fastapi import Request
     @app.exception_handler(Exception)
     async def _unhandled(request: Request, exc: Exception):
         # логируем кратко, без чувствительных данных
         return JSONResponse({"detail": "Internal error"}, status_code=500)
     ```

  2. Настройте минимальный логгер: INFO, без дампов запросов целиком; маскируйте потенциальные секреты.
* **Тесты:** «искусственная ошибка → 500 + безопасное тело», «в логах нет пароля/токена» (простейшая проверка по строке).
* **Пруфы:** фрагмент лога в `EVIDENCE/S06/logs/`, отчёт тестов.
* **DV:** DV3, DV4, DV5.

---

## 🟦 S06-06 - Безопасные заголовки (минимальный набор)

* [ ] **Проблема:** по умолчанию нет заголовков, снижающих риск XSS/Clickjacking/утечек.
* **Где:** `app/main.py`.
* **Исправление (шаги):**

  1. Добавьте middleware, выставляющий заголовки:

     ```python
     from starlette.middleware.base import BaseHTTPMiddleware
     @app.middleware("http")
     async def sec_headers(request, call_next):
         resp = await call_next(request)
         resp.headers["X-Content-Type-Options"] = "nosniff"
         resp.headers["X-Frame-Options"] = "DENY"
         resp.headers["Referrer-Policy"] = "no-referrer"
         resp.headers["Content-Security-Policy"] = "default-src 'self'"
         return resp
     ```

* **Тесты:** проверьте наличие заголовков в ответе `/` (200 + все ключи).
* **Пруфы:** отчёт тестов, скрин ответа с заголовками.
* **DV:** DV3, DV4.

---

## 🟦 S06-07 - Гигиена секретов и конфигов

* [ ] **Проблема:** риск случайного коммита секретов; жёстко зашитые значения.
* **Где:** `.env.example`, `README.md`, (опц.) `app/main.py`.
* **Исправление (шаги):**

  1. Загружайте значения из окружения (`os.getenv(...)`), а **не** из кода.
  2. Оставьте **только** `.env.example`; `.env` и секреты - не коммитим (уже в `.gitignore`).
  3. В `README.md` допишите раздел «Конфиги/переменные окружения».
* **Тесты:** (легко) smoke-тест «приложение стартует без .env», «значение берётся из env».
* **Пруфы:** скрин `.git status` (секретов нет), фрагмент README.
* **DV:** DV2, DV5 (+ DV1 по one-liner, если используете переменные).

---

## 🟦 S06-08 - Утяжеление доступа к /login (anti-bruteforce, лайт)

* [ ] **Проблема:** можно без ограничений брутить пароль.
* **Где:** `app/main.py` (`login`).
* **Исправление (шаги):**

  1. Добавьте простейший in-memory лимитер на IP/логин:

     ```python
     from time import time
     attempts = {}
     WINDOW, LIMIT = 60, 5  # 5 попыток в минуту
     def too_many(key):
         now = time()
         bucket = [t for t in attempts.get(key, []) if now - t < WINDOW]
         if len(bucket) >= LIMIT: return True
         bucket.append(now); attempts[key] = bucket; return False
     # в login():
     key = f"{payload.username}:{request.client.host}"
     if too_many(key):
         raise HTTPException(status_code=429, detail="Too Many Attempts")
     ```

     > Это учебная заглушка (процесс-локальная, без распределённости).
* **Тесты:** три неуспешных логина подряд → 401; >5 → 429.
* **Пруфы:** отчёт тестов, скрин 429.
* **DV:** DV3, DV4.

---

## Как это положить в DV

* **DV1 (Воспроизводимость):** one-liner запускает окружение и тесты, пишет отчёт в `EVIDENCE/S06/`.
* **DV2 (Гигиена секретов):** `.env.example`, отсутствие секретов в репо, использование env.
* **DV3 (Минимальная безопасность кода):** выбраны ≥2 карточки, описаны фиксы и ссылки на соответствующие тесты/коммиты.
* **DV4 (Артефакты/логи):** `EVIDENCE/S06/test-report.*`, `logs/*`, `screenshots/*`, при необходимости - `patches/*.diff`.
* **DV5 (Инструкции):** 2-5 строк в `README` про локальный запуск + отсылка к one-liner и к `EVIDENCE/S06/`.

---

## Быстрый чек перед сдачей

* [ ] Выбрано **минимум 2** карточки и доведены до **зелёных тестов**.
* [ ] В `GRADING/DV.md` есть краткие описания фиксов + ссылки на файлы из `EVIDENCE/S06/`.
* [ ] One-liner работает на чистой машине (без ручного ввода) и пишет отчёт.
* [ ] В репозитории **нет** секретов; есть `.env.example`.

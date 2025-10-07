# S07 - Troubleshooting (лайт)

> Используйте как «шпаргалку первой помощи». Для каждого кейса: примените фикс и сохраните пруфы в `EVIDENCE/S07/` (см. блок *Evidence*).

---

## 1) Контейнер поднялся, но `/` не отвечает `200`

**Симптомы**

* `curl http://127.0.0.1:8000/` даёт `000`/timeout, браузер не открывает.
  **Причины**
* Приложение в контейнере слушает `127.0.0.1`, а не `0.0.0.0`.
* Порт занят на хосте (`8000`).
* Контейнер упал сразу после старта.

**Фиксы**

1. Убедитесь, что сервер слушает на `0.0.0.0:8000` (в seed уже так):
   `uvicorn app.main:app --host 0.0.0.0 --port 8000`
2. Проверьте, что порт свободен или пробросите другой:
   `docker run -p 18000:8000 ...` и зайдите на `http://127.0.0.1:18000/`.
3. Посмотрите логи:
   `docker logs seed | tee EVIDENCE/S07/run.log`

**Evidence**

* `EVIDENCE/S07/http_root_code.txt` (ожидаем `200`)
* `EVIDENCE/S07/run.log` (или `compose-up.log`)

---

## 2) `docker build` падает (зависимости, сеть, кэш)

**Симптомы**

* Ошибки установки `pip`, таймауты сети, нехватка прав.

**Фиксы**

1. Повторите сборку с логами:
   `docker build -t secdev-seed:latest . | tee EVIDENCE/S07/build.log`
2. Убедитесь, что в `requirements.txt` адекватные версии и интернет доступен (в кампусе - проверьте прокси).
3. Если база - `slim`, не ставьте системные тулзы без `apt-get update`. Пример:

   ```Dockerfile
   RUN apt-get update && apt-get install -y --no-install-recommends curl \
       && rm -rf /var/lib/apt/lists/*
   ```

**Evidence**

* `EVIDENCE/S07/build.log`
* `EVIDENCE/S07/image-size.txt`

---

## 3) Конфликт порта `8000`

**Симптомы**

* `Error starting userland proxy: Bind for 0.0.0.0:8000 failed: port is already allocated`.

**Фиксы**

* Найдите процесс и освободите порт **или** смените проброс:
  `docker run -p 18000:8000 ...` → обновите README и положите код ответа для нового порта.

**Evidence**

* `EVIDENCE/S07/http_root_code.txt` (для выбранного порта)
* Команда запуска в `GRADING/DV.md`

---

## 4) Healthcheck всегда `unhealthy`

**Симптомы**

* `docker inspect ... | jq '.[0].State.Health'` показывает `unhealthy`.

**Причины**

* Проверка бьётся в неправильный URL/порт.
* Сервис поднимается дольше, чем `start_period`.

**Фиксы**

* В Dockerfile/Compose используйте проверку на `http://localhost:8000/` внутри контейнера (не `127.0.0.1:18000`).
* Увеличьте `start_period`/`retries`:

  ```yaml
  healthcheck:
    test: ["CMD", "python", "-c", "import urllib.request as u; u.urlopen('http://localhost:8000/')"]
    interval: 10s
    timeout: 3s
    retries: 5
    start_period: 10s
  ```

**Evidence**

* `EVIDENCE/S07/health.json` или скрин `inspect` с `Status: "healthy"`

---

## 5) Non-root ломает запись в `app.db`

**Симптомы**

* Приложение стартует, но падает с `Permission denied` при работе с SQLite.

**Причины**

* Права у пользователя в контейнере не накатаны на `/app` (или volume).

**Фиксы**

* В Dockerfile после копирования кода:

  ```Dockerfile
  RUN useradd -m -u 10001 appuser && chown -R appuser:appuser /app
  USER appuser
  ```

* В Compose при read-only FS - дайте writable-точку под БД:

  ```yaml
  read_only: true
  volumes:
    - appdb:/app/app.db
  ```

**Evidence**

* `EVIDENCE/S07/non-root.txt` (`id -u` ≠ 0), `EVIDENCE/S07/rofs_code.txt` (если включали ROFS)

---

## 6) Нет `curl`/`jq` в образе/на хосте

**Симптомы**

* Команды из методички не работают.

**Фиксы**

* Везде можно заменить на Python, который уже есть:

  ```bash
  # HTTP-код без curl
  python - <<'PY' > EVIDENCE/S07/http_root_code.txt
  import urllib.request; print(urllib.request.urlopen("[http://127.0.0.1:8000/](http://127.0.0.1:8000/)", timeout=5).getcode())
  ```

* Вместо `jq` сохраните полный `inspect` и приложите скрин/поиск строки.

**Evidence**

* `EVIDENCE/S07/http_root_code.txt`, `EVIDENCE/S07/inspect_web.json`

---

## 7) `docker compose` ругается на версию/ключи

**Симптомы**

* Ошибки вида `Additional property deploy is not allowed`/`version: "3"` не распознана.

**Причины**

* Устаревший Compose или смешение синтаксиса V2/V3.

**Фиксы**

* Обновите Docker Desktop/CLI. Для локальной практики не используйте блок `deploy:` (он для Swarm).  
* Уберите лишние поля, оставьте базовый манифест (build/ports/environment/healthcheck).

**Evidence**

* `EVIDENCE/S07/compose-up.log` (успешный запуск), актуальный `docker-compose.yml` в репо

---

## 8) Windows/WSL: проблемы путей и прав

**Симптомы**

* Том не монтируется, `Permission denied`, странные пути.

**Фиксы**

* На Windows используйте WSL2 и запускайте команды из Linux-шелла.  
* Избегайте пробелов/кириллицы в путях к проекту.  
* Если нужны скрипты, добавьте `.bat`/PowerShell альтернативы или запускайте через `python -m`.

**Evidence**

* Фото/скрин `compose-up.log` и рабочие команды в README (раздел Windows)

---

## 9) «Залипшие» контейнеры/образы/тома

**Симптомы**

* «Ничего не меняется», хотя правки внесены.

**Фиксы (осторожно!)**

* Остановить и удалить сервисы:  
`docker compose down -v` *(удалит и тома проекта)*  
* Удалить одиночный контейнер/образ:  
`docker rm -f seed; docker rmi secdev-seed:latest`  
* Собрать заново с `--no-cache`:  
`docker build --no-cache -t secdev-seed:latest .`

**Evidence**

* `EVIDENCE/S07/build.log` (новый билд), обновлённый `image-size.txt`

---

## 10) БД не инициализируется в контейнере

**Симптомы**

* Пустые таблицы, 500 при запросах.

**Причины**

* Не вызван `scripts/init_db.py` в CMD/entrypoint.

**Фикс**

* В Dockerfile CMD (как в seed):  
`CMD python scripts/init_db.py && uvicorn app.main:app --host 0.0.0.0 --port 8000`

**Evidence**

* `EVIDENCE/S07/run.log` (видно «DB initialized …»)

---

## 11) Timeout на слабых машинах/в аудитории

**Симптомы**

* Сборка/запуск долго висят, healthcheck падает.

**Фиксы**

* Увеличьте `start_period` и `retries` healthcheck.  
* Временно выключите тяжёлые шаги (линт/сканеры) - это S08/DS, не S07.  
* Используйте `docker compose up --wait` и дайте сервису 10-20 секунд.

**Evidence**

* `EVIDENCE/S07/health.json` с `Status: "healthy"`

---

## Быстрый чек-лист «всё работает?»

* [ ] `docker build` успешен → `build.log`, `image-size.txt`  
* [ ] Контейнер запущен → `run.log`/`compose-up.log`, `ps.txt`  
* [ ] `/` отвечает `200` → `http_root_code.txt`  
* [ ] `inspect`/`health` сохранены → `inspect_web.json`, `health.json`  
* [ ] (Если делали) non-root/ROFS → `non-root.txt` / `rofs_code.txt`  
* [ ] README обновлён: команды запуска/проверок, примечания по платформе

---

## Что указать в `GRADING/DV.md` (после починки)

* Коротко: «что было → что сделали → где пруфы».  
* Ссылки на файлы из `EVIDENCE/S07/…` (точные относительные пути).  
* Для DV2 - явно зафиксируйте, что `.env` не коммитится; где храните значения (env/secrets).  
* Для DV5 - 3-5 строк в README: `build/run`, проверка `/`, health, типичные примечания (порт, Windows/WSL).

---

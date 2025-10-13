# S07 - Compose Template

> Цель: стандартизировать локальный запуск контейнера для seed-проекта (FastAPI на :8000), собрать пруфы в `EVIDENCE/S07/` и обновить DV (DV2/DV4/DV5).

## Как использовать

1. Скопируйте YAML ниже в `docker-compose.yml` в корне вашего командного репозитория.
2. Запустите:

   ```bash
   docker compose up --build --wait | tee EVIDENCE/S07/compose-up.log
   ```

3. Соберите пруфы (`inspect`, `health`, размеры) и добавьте ссылки на них в `GRADING/DV.md`.

---

## `docker-compose.yml` (шаблон)

```yaml
# Версия файла: Compose V2 (Docker CLI >= 20.10)
# Файлы .env (значения) НЕ коммитим. Пример - .env.example (без секретов).

services:
  web:
    # Вариант 1: собираем из Dockerfile текущего проекта
    build:
      context: .
      dockerfile: Dockerfile
      # args:        # если понадобятся build-аргументы
      #   ТУТ НУЖНА ТВОЯ ВСТАВКА: value
    # Вариант 2 (альтернативно): используем уже собранный имидж
    # image: ТУТ НУЖНА ТВОЯ ВСТАВКА # напр., ghcr.io/org/seed:1.0.0

    container_name: ТУТ НУЖНА ТВОЯ ВСТАВКА # напр., seed-web
    ports:
      - "8000:8000"     # host:container
    environment:
      # Значения берем из env/секретов платформы. Здесь - примерные дефолты.
      APP_NAME: ${APP_NAME:-secdev-seed}
      DEBUG: ${DEBUG:-true}
      # SECRET_KEY: ${SECRET_KEY:?set in your local env or CI secrets}
    # Если используются локальные .env, можно подключить:
    # env_file:
    #   - .env   # НЕ коммитить

    # ===== Минимальный healthcheck без curl (python есть в образе) =====
    healthcheck:
      test: ["CMD-SHELL", "python -c \"import urllib.request as u; u.urlopen('http://127.0.0.1:8000/').read()\""]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 5s

    # ===== Опциональные усиления рантайма (см. S07_hardening_cards) =====
    # user: "10001:10001"         # если добавите non-root user в Dockerfile
    # read_only: true             # включить только после настройки writable-точек
    # cap_drop: ["ALL"]
    # security_opt:
    #   - no-new-privileges:true

    # ===== Persist/readonly точки =====
    # volumes:
    #   - appdb:/app/app.db       # если нужно сохранять SQLite между перезапусками
    #   - ./EVIDENCE:/app/EVIDENCE:rw   # осторожно: монтируйте осознанно

    restart: unless-stopped

# ===== Опциональные named volumes =====
# volumes:
#   appdb:
#     name: ТУТ НУЖНА ТВОЯ ВСТАВКА
```

---

## Мини-проверки и артефакты (`EVIDENCE/S07/`)

Рекомендуемые команды (примерные):

```bash
# HTTP-код главной страницы
curl -sS http://127.0.0.1:8000/ -o /dev/null -w "%{http_code}\n" > EVIDENCE/S07/http_root_code.txt

# Метаданные контейнера
docker inspect ТУТ_ИМЯ_КОНТЕЙНЕРА > EVIDENCE/S07/inspect_web.json

# Health (если включён)
docker inspect ТУТ_ИМЯ_КОНТЕЙНЕРА | jq '.[0].State.Health' > EVIDENCE/S07/health.json || true

# Размер образа
docker image ls ТУТ_ИМЯ_ОБРАЗА:ТЕГ --format "{{.Repository}} {{.Tag}} {{.Size}}" > EVIDENCE/S07/image-size.txt
```

Перечень файлов, на которые сослаться в `GRADING/DV.md (DV4)`:

* `EVIDENCE/S07/compose-up.log`
* `EVIDENCE/S07/http_root_code.txt`
* `EVIDENCE/S07/inspect_web.json`
* `EVIDENCE/S07/health.json` *(если есть healthcheck)*
* `EVIDENCE/S07/image-size.txt`

---

## Памятка по DV

* **DV2:** секреты - только через переменные окружения/секрет-хранилище; `.env` не коммитим; `.env.example` - ок.
* **DV4:** приложены артефакты сборки/запуска/проверок из `EVIDENCE/S07/`.
* **DV5:** в `README` есть 3-5 строк «Запуск в контейнере» с командами и примечаниями.

# S07 - Hardening Cards (лайт)

> Выберите **минимум 1-2** карточки. Для каждой: примените фикс → проверьте → сложите пруфы в `EVIDENCE/S07/` → коротко опишите в `GRADING/DV.md` (DV2/DV4/DV5).

Формат карточки: **Что/Зачем → Как (Dockerfile/Compose) → Проверка → Пруфы → DV → Подводные камни.**

---

## 🟩 S07-01 - Pinned base image (фиксированный базовый образ)

**Что/Зачем.** Детерминизм сборки и предсказуемость уязвимостей.
**Как (Dockerfile).**

```Dockerfile
FROM python:3.11-slim@sha256:TУТ_НУЖЕН_КОНКРЕТНЫЙ_SHA   # или python:3.11.9-slim
```

**Проверка.**

```bash
docker build -t secdev-seed:pin . | tee EVIDENCE/S07/build.log
docker image ls secdev-seed:pin --format "{{.Repository}} {{.Tag}} {{.Size}}" > EVIDENCE/S07/image-size.txt
```

**Пруфы.** `build.log`, `image-size.txt` (+ ссылку на Dockerfile-коммит).
**DV.** DV4 (артефакты), DV5 (README «какой образ и почему»).
**Подводные камни.** Не переходите на `latest`.

---

## 🟩 S07-02 - Multi-stage / slimming (уменьшаем образ)

**Что/Зачем.** Меньше поверхность атаки и быстрее доставка.
**Как (эскиз).**

```Dockerfile
FROM python:3.11-slim AS runtime
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app ./app
COPY scripts ./scripts
CMD python scripts/init_db.py && uvicorn app.main:app --host 0.0.0.0 --port 8000
```

*(Если нужны сборочные тулзы - отдельный `builder` и перенос только `site-packages`.)*
**Проверка.** Зафиксируйте размер до/после в `image-size.txt`.
**Пруфы.** `build.log`, `image-size.txt`, diff Dockerfile.
**DV.** DV4, DV5 (комментарий про размер/слои).
**Подводные камни.** Не оставляйте кэш менеджера пакетов.

---

## 🟩 S07-3 - Non-root user (запуск не от root)

**Что/Зачем.** Снижаем ущерб при компрометации.
**Как (Dockerfile).**

```Dockerfile
# ...после установки зависимостей
RUN useradd -m -u 10001 appuser && chown -R appuser:appuser /app
USER appuser
```

**Проверка.**

```bash
docker run --rm --name seed -p 8000:8000 secdev-seed:latest &
docker exec seed id -u > EVIDENCE/S07/non-root.txt
```

`non-root.txt` должен содержать не `0`.
**Пруфы.** `non-root.txt`, `inspect_web.json`.
**DV.** DV4.
**Подводные камни.** Убедитесь, что у `appuser` есть права на запись туда, где создаётся `app.db`.

---

## 🟩 S07-4 - Read-only root FS (только чтение) + writable-точка

**Что/Зачем.** Блокируем неожиданные записи в контейнере.
**Как (Compose).**

```yaml
services:
  web:
    # ...
    read_only: true
    volumes:
      - appdb:/app/app.db   # точка записи только для БД
volumes:
  appdb:
```

**Проверка.**

```bash
docker exec seed sh -c "touch /etc/should_fail" ; echo $? > EVIDENCE/S07/rofs_code.txt
```

Код должен быть `1` (ошибка записи).
**Пруфы.** `rofs_code.txt`, `compose-up.log`.
**DV.** DV4.
**Подводные камни.** Наше приложение пишет `app.db` в `/app/` при старте - без тома запись упадёт.

---

## 🟩 S07-5 - Drop Linux capabilities + no-new-privileges

**Что/Зачем.** Режем привилегии процесса.
**Как (Compose).**

```yaml
services:
  web:
    # ...
    cap_drop: ["ALL"]
    security_opt:
      - no-new-privileges:true
```

**Проверка.** Снимите `inspect` и сохраните:
`EVIDENCE/S07/inspect_web.json` (поля `CapAdd: null`, `CapDrop: ["ALL"]`).
**Пруфы.** `inspect_web.json`.
**DV.** DV4.
**Подводные камни.** Некоторые инструменты требуют капабилити - для нашего FastAPI не нужно.

---

## 🟩 S07-6 - Healthcheck (жив ли сервис)

**Что/Зачем.** Автоматически детектим деградации.
**Как (Dockerfile вариант).**

```Dockerfile
HEALTHCHECK --interval=10s --timeout=3s --retries=5 CMD python -c "import urllib.request as u; u.urlopen('http://127.0.0.1:8000/').read()" || exit 1
```

**Или (Compose).**

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8000/"]
  interval: 10s
  timeout: 3s
  retries: 5
```

**Проверка.**

```bash
docker inspect seed | jq '.[0].State.Health' > EVIDENCE/S07/health.json
```

**Пруфы.** `health.json` (или скрин).
**DV.** DV4, DV5 (README: как смотреть статус).
**Подводные камни.** Не ставьте слишком частые проверки на слабых машинах.

---

## 🟩 S07-7 - Secrets & env-гигиена (контейнерный запуск)

**Что/Зачем.** Исключить утечки значений в образ/репо.
**Как.**

* В репо - только `.env.example`.
* Значения передавать окружением/секрет-хранилищем CI/CD.
* В Compose не хранить секреты в YAML (используйте placeholder’ы / `env_file` локально).
  **Проверка.** `git status` чистый; в образе нет `.env`.
  **Пруфы.** `EVIDENCE/S07/screenshots/git_status.png`, `build.log` (без секретов).
  **DV.** DV2, DV5.
  **Подводные камни.** Не кладите `.env` в образ (`COPY . .` без `.dockerignore` - антипаттерн).

---

## 🟩 S07-8 - Минимизация сети (поверхность)

**Что/Зачем.** Открыт только нужный порт и интерфейс.
**Как.**

* В контейнере - слушайте `0.0.0.0:8000` (для проброса), наружу - публикуйте только нужный порт.
* Не используйте `--network host`.
  **Проверка.**

```bash
docker exec seed ss -ltnp > EVIDENCE/S07/listen.txt
```

**Пруфы.** `listen.txt`, `inspect_web.json` (порт/экспозиция).
**DV.** DV4.
**Подводные камни.** Не пробрасывайте «дикий» диапазон портов.

---

## 🟩 S07-9 - Чистка слоёв и кэша (apt/pip)

**Что/Зачем.** Снижаем размер и мусор.
**Как (если ставите системные пакеты).**

```Dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends curl \
 && rm -rf /var/lib/apt/lists/*
```

**pip** уже с `--no-cache-dir` в нашем Dockerfile.
**Проверка.** Сравните `image-size.txt` до/после.
**Пруфы.** `image-size.txt`, diff Dockerfile.
**DV.** DV4, DV5 (комментарий).
**Подводные камни.** Не объединяйте лишнее в один RUN, если это ломает кеш-слои разумности.

---

## 🟩 S07-10 - Лимиты ресурсов контейнера (лайт)

**Что/Зачем.** Сдерживаем runaway-процессы.
**Как.**

* `docker run --memory=512m --cpus="1.0" ...`
* или (Compose, для локальной практики):

  ```yaml
  deploy:
    resources:
      limits:
        cpus: '1.0'
        memory: 512M
  ```

  *(вне Swarm это скорее документация; для реального лимита используйте `docker run`)*
  **Проверка.** Сохраните команду запуска и `inspect_web.json` (параметры HostConfig).
  **Пруфы.** `run.log`/`compose-up.log`, `inspect_web.json`.
  **DV.** DV4, DV5.
  **Подводные камни.** Убедитесь, что лимиты не мешают старту.

---

## 🟩 S07-11 - Линт Dockerfile (`hadolint`) *(опционально)*

**Что/Зачем.** Быстрая проверка антипаттернов.
**Как.**

```bash
hadolint Dockerfile | tee EVIDENCE/S07/hadolint.txt
```

**Пруфы.** `hadolint.txt`.
**DV.** DV4 (артефакт), DV5 (краткий вывод «пофиксили X/Y»).
**Подводные камни.** Не тратьте всё время на идеальный счёт - цель «лайт».

---

## Как это положить в DV

* **DV2 (гигиена секретов):** карточка S07-7 → опишите, как значения передаются окружением; приложите `git status`/`.env.example`.
* **DV4 (артефакты/логи):** для каждой выбранной карточки приложите пруфы из списка (`build.log`, `inspect_web.json`, `health.json`, `non-root.txt`, `image-size.txt`, и т. д.).
* **DV5 (инструкции):** в `README` добавьте 3-5 строк «Запуск в контейнере» + перечислите выбранные меры (короткими пунктами).

---

## Быстрый чек перед сдачей

* [ ] Выбраны **≥1-2** карточки, изменения применены.
* [ ] Сервис поднимается и отдаёт `200` на `/`.
* [ ] В `EVIDENCE/S07/` лежат пруфы по выбранным карточкам.
* [ ] `GRADING/DV.md` и `README` обновлены (DV2/DV4/DV5).

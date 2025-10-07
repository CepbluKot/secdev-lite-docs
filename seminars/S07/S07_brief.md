# S07 - Containerization (Docker/Compose)

**Задача занятия (70 мин):**
перенести локальный запуск из S06 в контейнер: **собрать образ → запустить сервис → пройти healthcheck → собрать артефакты в `EVIDENCE/S07/`**. Минимально затронуть харденинг (non-root/healthcheck/`.dockerignore`). Основано на seed-проекте `secdev-seed-s06-s08 v2`.

## Что получится к концу S07

* Собран **Docker-образ** и запущен контейнер (Docker или Compose).
* **Healthcheck OK** (200 на `/` и/или явный health-status).
* **Evidence** в `EVIDENCE/S07/` (логи сборки/запуска, inspect/health, размер образа).
* **DV обновлён:**

  * **DV2** - гигиена секретов в контейнерном запуске (env-переменные, без `.env` в репо).
  * **DV4** - артефакты/логи контейнерного запуска.
  * **DV5** - короткая инструкция «Запуск в контейнере» в `README`.

  > **DV1 (one-liner)** остаётся из S06 (локальный запуск), его не переносим в Docker - контейнеризация оценивается отдельно.

## Предварительные требования

* Seed `v2` и **рабочий one-liner S06**.
* Установлены Docker + (опц.) Docker Compose V2.

---

## План занятия (тайминг)

**0-10 мин. Подготовка**

1. Проверьте, что проект собирается локально (из S06).
2. Убедитесь, что в корне есть `Dockerfile`, `.dockerignore`, *(опц.)* `docker-compose.yml`.

**10-25 мин. Сборка образа**

* Соберите образ (любой из вариантов ниже).
* Зафиксируйте логи/размер:

  ```bash
  docker build -t secdev-seed:latest . | tee EVIDENCE/S07/build.log
  docker image ls secdev-seed:latest --format "{{.Repository}} {{.Tag}} {{.Size}}" > EVIDENCE/S07/image-size.txt
  ```

**25-40 мин. Запуск и healthcheck**

* Запуск **без Compose**:

  ```bash
  docker run --rm -p 8000:8000 --name seed secdev-seed:latest > EVIDENCE/S07/run.log 2>&1 &
  ```

* Или **через Compose**:

  ```bash
  docker compose up --build --wait | tee EVIDENCE/S07/compose-up.log
  ```

* Проверьте доступность:

  ```bash
  curl -sS http://127.0.0.1:8000/ -o /dev/null -w "%{http_code}\n" > EVIDENCE/S07/http_root_code.txt
  ```

* Снимите метаданные:

  ```bash
  docker inspect seed > EVIDENCE/S07/inspect_web.json  # если без compose
  ```

**40-60 мин. Мини-харденинг (лайт) и пруфы**
Выберите **минимум 1 карточку** из `S07_hardening_cards.md` (например, **non-root** или **read-only FS**) и примените в своём репозитории:

* Проверьте, что контейнер работает **не от root**:

  ```bash
  docker exec -it seed id -u > EVIDENCE/S07/non-root.txt
  ```

* Снимите health-status (если есть):

  ```bash
  docker inspect seed | jq '.[0].State.Health' > EVIDENCE/S07/health.json
  ```

*(Если нет `jq`, сохраните скрин/сырой `inspect`.)*

**60-70 мин. Обновляем DV/README**

* В `GRADING/DV.md` добавьте раздел S07 (см. «Что сдаём»).
* В `README` - 3-5 строк «Запуск в контейнере» (команды `build/run` или `compose up`).
* Закоммитьте `EVIDENCE/S07/*`.

---

## Что сдаём и куда складываем

В **репозитории команды**:

* `GRADING/DV.md` - дополнить:

  * **DV2:** описать, как передаются конфиги/секреты (через env), сослаться на `.env.example`; убедиться, что **секреты не попали в образ**.
  * **DV4:** перечислить артефакты S07 (см. следующий пункт).
  * **DV5:** ссылка на раздел README «Запуск в контейнере».
* `EVIDENCE/S07/` - создать и положить:

  * `build.log` - вывод `docker build`.
  * `image-size.txt` - размер образа.
  * `compose-up.log` **или** `run.log` - логи запуска.
  * `inspect_web.json` - результат `docker inspect` для контейнера.
  * `health.json` **или** `health.png` - статус healthcheck (если есть).
  * `non-root.txt` - результат `id -u` внутри контейнера (если делали non-root).
  * *(опц.)* `hadolint.txt` - если запускали линтер Dockerfile.

---

## Рекомендованные команды (примеры)

**Билд и запуск (Docker):**

```bash
docker build -t secdev-seed:latest .
docker run --rm -p 8000:8000 --name seed secdev-seed:latest
```

**Билд и запуск (Compose):**

```bash
docker compose up --build
```

**Проверка / сбор артефактов:**

```bash
curl -sS http://127.0.0.1:8000/ -o /dev/null -w "%{http_code}\n"
docker inspect seed | tee EVIDENCE/S07/inspect_web.json > /dev/null
```

> **Важно:** не храните `.env` в репозитории; используйте **`.env.example`**. Передавайте реальные значения через переменные среды/секреты платформы.

---

## Критерии «готово»

**Минимум (★1, базовый):**

* Образ собирается; контейнер стартует и отдаёт 200 на `/`.
* В `EVIDENCE/S07/` лежат `build.log`, лог запуска и хотя бы один из: `inspect_web.json` / `health.json`.
* `GRADING/DV.md` обновлён (DV2/DV4/DV5), README дописан.

**Проектный уровень (★★2):**

* Добавлены **1-2 харденинг-патча** (например, non-root + healthcheck), пруфы приложены.
* Размер образа обоснован (комментарий в DV), лишние слои/кэш вычищены.
* (Опц.) `hadolint`/линтинг Dockerfile, отчёт приложен.

---

## Частые ошибки

* Секреты/`.env` попали в образ или в репозиторий.
* Контейнер работает от root без необходимости.
* Нет доказательств (логи/inspect/health) в `EVIDENCE/S07/`.
* Порты заняты/не проброшены (`-p 8000:8000`).
* Не обновлён `GRADING/DV.md` и README.

---

## Связь с S06 и S08

* **Из S06** берём рабочий one-liner и логику запуска.
* **Для S08** этот же контейнер и команды лягут в CI-джобу (build → test; артефакты → `EVIDENCE/S08/`).

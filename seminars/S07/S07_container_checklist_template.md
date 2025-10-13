# S07 - Containerization Checklist (template)

> Цель: аккуратно «упаковать» one-liner из S06 в контейнер (Docker/Compose), собрать проверяемые артефакты и обновить DV.

## Как пользоваться

1. Пройдитесь по разделам, ставьте галочки и добавляйте **точные ссылки** на артефакты из `EVIDENCE/S07/`.
2. По завершении перенесите краткую сводку в `GRADING/DV.md` (DV2/DV4/DV5).
3. **Не коммитьте секреты.** Для примеров используйте `.env.example`.

---

## Паспорт выполнения (заполнить)

* Команда/репозиторий: **ТУТ НУЖНА ТВОЯ ВСТАВКА**
* Тег образа: **secdev-seed:ТУТ НУЖНА ТВОЯ ВСТАВКА**
* Вариант запуска: ☐ Docker  ☐ Compose
* One-liner из S06 (для справки): **ТУТ НУЖНА ТВОЯ ВСТАВКА**
* Ссылки на артефакты S07: см. «Индекс пруфов».

---

## A. Образ (build)

* [ ] **Pinned base image** (например, `python:3.11-slim`), зафиксирован тег/дистрибутив.
  *Комментарий:* **ТУТ НУЖНА ТВОЯ ВСТАВКА**
* [ ] **`.dockerignore`** присутствует и исключает мусор (`__pycache__/`, `.venv/`, `tests/`, `EVIDENCE/`, `.git/`, `.env`).
* [ ] **WORKDIR, COPY, RUN**: зависимости ставятся из `requirements.txt` с `--no-cache-dir`; копируются только нужные файлы (app/, scripts/, requirements.txt).
* [ ] **Сборка воспроизводима**: командой

  ```bash
  ТУТ НУЖНА ТВОЯ ВСТАВКА  # пример: docker build -t secdev-seed:latest .
  ```

* [ ] **Лог сборки сохранён** → `EVIDENCE/S07/build.log`.
* [ ] **Размер образа зафиксирован** → `EVIDENCE/S07/image-size.txt`.
* [ ] (Опц.) **Multi-stage**/очистка слоёв - если применимо, описать.
  *Комментарий:* **ТУТ НУЖНА ТВОЯ ВСТАВКА**

**DV:** DV4 (артефакты); DV5 (описание сборки в README).

---

## B. Запуск (run)

* [ ] Порт проброшен (`-p 8000:8000`) или описан в Compose.
* [ ] Контейнер поднимается командой:

  ```bash
  ТУТ НУЖНА ТВОЯ ВСТАВКА  # docker run ... ИЛИ docker compose up --build
  ```

* [ ] **Лог запуска сохранён** → `EVIDENCE/S07/run.log` или `EVIDENCE/S07/compose-up.log`.
* [ ] **Доступность проверена** (`GET /` → 200) → `EVIDENCE/S07/http_root_code.txt`.
* [ ] **inspect контейнера сохранён** → `EVIDENCE/S07/inspect_web.json`.
* [ ] (Если есть healthcheck) **статус сохранён** → `EVIDENCE/S07/health.json` (или скрин).

**DV:** DV4 (артефакты); DV5 (README: раздел «Запуск в контейнере»).

---

## C. Конфиги/секреты (гигиена)

* [ ] **`.env` не в репозитории**, только `.env.example` (образец значений).
* [ ] Значения конфигов читаются из **переменных окружения**; секреты не попадают в образ/логи.
* [ ] В Compose/CI используются секреты/vars платформы (не хранить значения в YAML).
* [ ] В `README` коротко описано: **какие переменные** нужны для запуска.

**DV:** DV2 (гигиена секретов); DV5 (README).

---

## D. Рантайм-безопасность (лайт-харденинг)

Выберите **минимум 1-2** пункта (для ★★ лучше 2):

* [ ] **Non-root user** в контейнере (пример для Debian slim):

  ```Dockerfile
  # пример, адаптируйте под свою базу
  RUN useradd -m -u 10001 appuser
  USER appuser
  ```

  Доказательство: `docker exec seed id -u` → `EVIDENCE/S07/non-root.txt`.
* [ ] **Healthcheck** (в Dockerfile или Compose). Пруф: `health.json`.
* [ ] **Read-only FS** и явный writable-каталог, если нужно (опц.).
* [ ] **Минимизация поверхности**: укажите, что именно исключили/не устанавливали.
* [ ] (Опц.) Линт Dockerfile (`hadolint`) → `EVIDENCE/S07/hadolint.txt`.

**DV:** DV4 (артефакты/логи); комментарий в DV5.

---

## E. Compose (если используется)

* [ ] Сервис `web` с `build: .`/`image:`, `ports: "8000:8000"`.
* [ ] `environment:` - только неспецифичные/несекретные значения; реальные - через секреты/vars.
* [ ] `healthcheck:` задан (test/interval/timeout/retries).
* [ ] (Опц.) `volumes:` для `app.db`/кэшей - если нужна локальная сохранность.
* [ ] `restart:` политика задана.
* [ ] Лог `compose up` сохранён → `EVIDENCE/S07/compose-up.log`.

**DV:** DV4 (артефакты); DV5 (README: пример `docker compose up --build`).

---

## Индекс пруфов (указывать точные пути)

* [ ] `EVIDENCE/S07/build.log`
* [ ] `EVIDENCE/S07/image-size.txt`
* [ ] `EVIDENCE/S07/run.log` **или** `EVIDENCE/S07/compose-up.log`
* [ ] `EVIDENCE/S07/http_root_code.txt`
* [ ] `EVIDENCE/S07/inspect_web.json`
* [ ] `EVIDENCE/S07/health.json` *(если есть healthcheck)*
* [ ] `EVIDENCE/S07/non-root.txt` *(если делали non-root)*
* [ ] `EVIDENCE/S07/hadolint.txt` *(если запускали linт)*

---

## Сводка для переноса в `GRADING/DV.md`

* **DV2 (гигиена секретов):** `.env` не коммитим; используем `.env.example`; секреты передаются окружением/секрет-хранилищем.
* **DV4 (артефакты/логи):** перечислите файлы из `EVIDENCE/S07/` (см. индекс выше).
* **DV5 (инструкции):** в `README` добавлен раздел «Запуск в контейнере» с командами:

  ```bash
  docker build -t secdev-seed:latest .
  docker run --rm -p 8000:8000 --name seed secdev-seed:latest
  # или:
  docker compose up --build
  ```

> **Примечание:** **DV1** (one-liner локальной сборки/тестов) остался из S06 и в Docker **не переносится**.

---

## Definition of Done (минимум/проектный)

**Минимум (★1):**

* [ ] Образ собирается; контейнер отвечает `200` на `/`.
* [ ] В `EVIDENCE/S07/` есть build/run/inspect (+ health при наличии).
* [ ] `GRADING/DV.md` и `README` обновлены (DV2/DV4/DV5).

**Проектный (★★2):**

* [ ] Применены ≥2 пункта харденинга (например, non-root + healthcheck).
* [ ] Обоснован размер образа (коммент в DV) и/или снижено число слоёв.
* [ ] (Опц.) Пройден линт Dockerfile; приложены отчёты.

---

**Подсказка:** если порт 8000 занят - временно используйте `-p 18000:8000` и зафиксируйте это в `README` и `EVIDENCE/S07/http_root_code.txt`.

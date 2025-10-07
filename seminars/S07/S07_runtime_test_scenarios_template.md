# S07 - Runtime Test Scenarios (template)

> Цель: подтвердить, что **образ собирается**, **контейнер поднимается**, эндпоинт `/` отвечает **200**, а выбранные меры харденинга действительно работают.
> Все артефакты - в `EVIDENCE/S07/` с понятными именами файлов.

## Как пользоваться

1. Отметьте чекбокс у сценариев, которые выполняете (минимум 5 базовых + то, что соответствует выбранным карточкам из `S07_hardening_cards.md`).
2. Запускайте **или** через `docker run`, **или** через `docker compose up --build --wait`.
3. Каждая проверка должна дать **артефакт** (файл/лог/скрин) с сохранённым выводом.
4. В `GRADING/DV.md` в разделе S07 перечислите **точные относительные пути** к этим файлам.

> Подсказка: если нет `curl/jq`, используйте `python -c` на хосте, как ниже.

---

## Минимальный набор (чек-лист)

* [ ] A1. Сборка образа зафиксирована (`build.log`, `image-size.txt`)
* [ ] B1. Контейнер запустился, есть лог (`run.log` или `compose-up.log`)
* [ ] B2. `/` отдаёт **HTTP 200** (`http_root_code.txt`)
* [ ] B3. `docker inspect` сохранён (`inspect_web.json`)
* [ ] C1. Env/секреты не утекли в образ/репо (скрин/заметка)
* [ ] D1+. **Харденинг**: ≥1-2 проверки (например, **non-root** и/или **healthcheck**/ **read-only FS**)

---

## Сценарии (GWT + команды + артефакты)

### A1 - Build: образ собирается, размер зафиксирован

**Given** Dockerfile в корне, `.dockerignore` присутствует
**When** выполняем сборку
**Then** сборка успешна, размер записан
**Commands (Linux/macOS):**

```bash
docker build -t secdev-seed:latest . | tee EVIDENCE/S07/build.log
docker image ls secdev-seed:latest --format "{{.Repository}} {{.Tag}} {{.Size}}" > EVIDENCE/S07/image-size.txt
```

**Artifacts:** `EVIDENCE/S07/build.log`, `EVIDENCE/S07/image-size.txt`
**DV:** DV4 (артефакты), DV5 (README: «как собираем»)

---

### B1 - Run: контейнер поднялся (Docker **или** Compose)

**Given** собранный образ
**When** запускаем сервис
**Then** контейнер в состоянии Running
**Commands (Docker):**

```bash
docker run --rm -p 8000:8000 --name seed secdev-seed:latest > EVIDENCE/S07/run.log 2>&1 &
sleep 2
docker ps --filter "name=seed" --format "{{.Names}} {{.Status}}" > EVIDENCE/S07/ps.txt
```

**Commands (Compose):**

```bash
docker compose up --build --wait | tee EVIDENCE/S07/compose-up.log
docker compose ps > EVIDENCE/S07/ps.txt
```

**Artifacts:** `run.log` **или** `compose-up.log`, `ps.txt`
**DV:** DV4

---

### B2 - HTTP 200 на `/`

**Given** запущенный контейнер на `:8000`
**When** обращаемся к `/`
**Then** статус-код 200
**Commands (кроссплатформенно без curl):**

```bash
python - <<'PY' > EVIDENCE/S07/http_root_code.txt
import urllib.request, sys
try:
    with urllib.request.urlopen("http://127.0.0.1:8000/", timeout=5) as r:
        print(r.getcode())
except Exception as e:
    print("ERR:", e, file=sys.stderr)
    print(0)
PY
```

*(Windows PowerShell альтернатива)*:

```powershell
(Invoke-WebRequest http://127.0.0.1:8000/ -UseBasicParsing).StatusCode `
  | Out-File EVIDENCE/S07/http_root_code.txt
```

**Expected:** файл содержит `200`
**DV:** DV4

---

### B3 - Inspect: метаданные контейнера

**Given** контейнер `seed` (или имя из Compose)
**When** снимаем `docker inspect`
**Then** JSON сохранён
**Commands:**

```bash
docker inspect seed > EVIDENCE/S07/inspect_web.json 2>&1 || true
```

*(Если Compose - подставьте имя контейнера из `docker compose ps`.)*
**DV:** DV4

---

### D1 - Non-root user (если применяли карточку S07-3)

**Given** в образе настроен пользователь non-root
**When** проверяем uid процесса
**Then** `id -u != 0`
**Commands:**

```bash
docker exec seed sh -c "id -u" > EVIDENCE/S07/non-root.txt
```

**Expected:** файл содержит число **не** `0`
**DV:** DV4

---

### D2 - Healthcheck (если включён в Dockerfile/Compose)

**Given** описан healthcheck
**When** запрашиваем состояние
**Then** `Status: "healthy"`
**Commands (с jq):**

```bash
docker inspect seed | jq '.[0].State.Health' > EVIDENCE/S07/health.json
```

**Без jq (сырой вывод):**

```bash
docker inspect seed > EVIDENCE/S07/inspect_web.json
# затем в GRADING/DV.md укажите строку/скрин, где виден .State.Health
```

**DV:** DV4, DV5 (README: «как проверять health»)

---

### D3 - Read-only root FS (если включали)

**Given** включён `read_only: true` + настроена writable-точка
**When** пытаемся писать в системный каталог
**Then** запись запрещена
**Commands:**

```bash
docker exec seed sh -c "touch /etc/should_fail; echo $?" > EVIDENCE/S07/rofs_code.txt
```

**Expected:** значение `1` (ошибка)
**DV:** DV4

---

### C1 - Env/секреты: гигиена конфигов

**Given** `.env` не коммитим; есть `.env.example`
**When** сервис читает конфиг из окружения
**Then** секреты не попали в образ/репо
**Steps/Artifacts:**

* Скрин `git status` → `EVIDENCE/S07/screenshots/git_status.png` (или `.txt`)
* Снимок `build.log`: нет строк с секретными значениями
* В `README` коротко перечислены нужные переменные (`APP_NAME`, `DEBUG`, …)
  **DV:** DV2, DV5

---

### D4 - Минимизация сети (поверхность)

**Given** публикуем только `:8000`
**When** смотрим открытые сокеты внутри
**Then** слушается только нужный порт
**Commands (может не быть ss/netstat в образе - допускается host-проверка):**

```bash
docker port seed > EVIDENCE/S07/ports.txt
```

**DV:** DV4

---

### D5 - Размер и слои (если делали slim/multi-stage)

**Given** оптимизирован Dockerfile
**When** сравниваем размер до/после
**Then** новый образ меньше/обоснован
**Artifacts:** `image-size.txt` (старый/новый), diff Dockerfile
**DV:** DV4, DV5 (комментарий «зачем так»)

---

## Индекс артефактов (сошлитесь в `GRADING/DV.md`)

* `EVIDENCE/S07/build.log`
* `EVIDENCE/S07/image-size.txt`
* `EVIDENCE/S07/run.log` **или** `EVIDENCE/S07/compose-up.log`
* `EVIDENCE/S07/ps.txt`
* `EVIDENCE/S07/http_root_code.txt`
* `EVIDENCE/S07/inspect_web.json`
* *(если есть)* `EVIDENCE/S07/health.json`
* *(если делали)* `EVIDENCE/S07/non-root.txt`, `EVIDENCE/S07/rofs_code.txt`, `EVIDENCE/S07/ports.txt`, `EVIDENCE/S07/screenshots/git_status.png`

---

## Как это положить в DV

* **DV2 (гигиена секретов):** приложите `git status`/`.env.example` и короткое описание в README.
* **DV4 (артефакты/логи):** перечислите все файлы из `EVIDENCE/S07/` (см. индекс).
* **DV5 (инструкции):** добавьте в README 3-5 строк «Запуск в контейнере» и «Проверки/health».

---

## Definition of Done (минимум)

* [ ] Образ собран, размер зафиксирован.
* [ ] Контейнер отвечает `200` на `/`.
* [ ] `inspect` (и, при наличии, `health`) сохранены.
* [ ] Выполнено ≥1-2 харденинг-проверки с пруфами.
* [ ] DV обновлён: DV2/DV4/DV5 + ссылки на артефакты.

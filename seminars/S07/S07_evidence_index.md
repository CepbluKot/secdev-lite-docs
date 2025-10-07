# S07 - Evidence Index (манифест артефактов)

> Цель: чтобы проверяющий за 1-2 клика нашёл все доказательства по S07.
> Все пути ниже **относительные к корню репозитория команды**.

## Где лежит

```text
EVIDENCE/
└── S07/
    ├── build.log
    ├── image-size.txt
    ├── run.log                # ИЛИ compose-up.log
    ├── ps.txt
    ├── http_root_code.txt
    ├── inspect_web.json
    ├── health.json            # если есть healthcheck
    ├── non-root.txt           # если делали non-root
    ├── rofs_code.txt          # если включали read_only
    ├── ports.txt              # если фиксировали сеть
    ├── hadolint.txt           # если линтили Dockerfile
    └── screenshots/
        └── git_status.png     # опционально, про гигиену секретов
```

---

## Обязательные артефакты (минимум)

**1) Лог сборки образа**

```bash
docker build -t secdev-seed:latest . | tee EVIDENCE/S07/build.log
```

Файл: `EVIDENCE/S07/build.log`

**2) Размер образа**

```bash
docker image ls secdev-seed:latest --format "{{.Repository}} {{.Tag}} {{.Size}}" > EVIDENCE/S07/image-size.txt
```

Файл: `EVIDENCE/S07/image-size.txt`

**3) Логи запуска контейнера**
- Docker:

```bash
docker run --rm -p 8000:8000 --name seed secdev-seed:latest > EVIDENCE/S07/run.log 2>&1 &
```

- Compose:

```bash
docker compose up --build --wait | tee EVIDENCE/S07/compose-up.log
```

Файл: `EVIDENCE/S07/run.log` **или** `EVIDENCE/S07/compose-up.log`

**4) Список запущенных (статус)**

```bash
docker ps --filter "name=seed" --format "{{.Names}} {{.Status}}" > EVIDENCE/S07/ps.txt
# или:
docker compose ps > EVIDENCE/S07/ps.txt
```

Файл: `EVIDENCE/S07/ps.txt`

**5) HTTP-код главной страницы /**

```bash
python - <<'PY' > EVIDENCE/S07/http_root_code.txt
import urllib.request
print(urllib.request.urlopen("http://127.0.0.1:8000/", timeout=5).getcode())
PY
```

Файл: `EVIDENCE/S07/http_root_code.txt` (ожидаем `200`)

**6) Инспект контейнера**

```bash
docker inspect seed > EVIDENCE/S07/inspect_web.json 2>&1 || true
```

Файл: `EVIDENCE/S07/inspect_web.json`

---

## Условные/по выбранным карточкам

**A) Healthcheck (если включён)**

```bash
docker inspect seed | jq '.[0].State.Health' > EVIDENCE/S07/health.json || true
```

Файл: `EVIDENCE/S07/health.json` *(или скрин в `screenshots/`)*

**B) Non-root user**

```bash
docker exec seed sh -c "id -u" > EVIDENCE/S07/non-root.txt
```

Файл: `EVIDENCE/S07/non-root.txt` *(значение НЕ должно быть `0`)*

**C) Read-only root FS**

```bash
docker exec seed sh -c "touch /etc/should_fail; echo $?" > EVIDENCE/S07/rofs_code.txt
```

Файл: `EVIDENCE/S07/rofs_code.txt` *(ожидаем `1`)*

**D) Минимизация сети**

```bash
docker port seed > EVIDENCE/S07/ports.txt
```

Файл: `EVIDENCE/S07/ports.txt`

**E) Линт Dockerfile (hadolint)**

```bash
hadolint Dockerfile | tee EVIDENCE/S07/hadolint.txt
```

Файл: `EVIDENCE/S07/hadolint.txt`

**F) Гигиена секретов (скрин/заметка)**

* `EVIDENCE/S07/screenshots/git_status.png` *(нет `.env`/секретов в репо)*
* `.env.example` лежит в корне (а **.env - нет**)

---

## Что указать в `GRADING/DV.md`

* **DV2 (гигиена секретов):** коротко как передаются значения (ENV/Secrets), ссылка на `.env.example`, скрин `git status`:
  `EVIDENCE/S07/screenshots/git_status.png`
* **DV4 (артефакты/логи):** перечислите **точно по именам**:
  `EVIDENCE/S07/build.log`, `image-size.txt`, `run.log`/`compose-up.log`, `ps.txt`, `http_root_code.txt`, `inspect_web.json`, а также условные (`health.json`, `non-root.txt`, …).
* **DV5 (инструкции):** 3-5 строк в `README` - как собрать/запустить в контейнере (Docker и/или Compose) + как проверить (HTTP 200, health).

---

## Правила именования (чтобы не было каши)

* Только **ASCII** в именах файлов; пробелы не используем.
* Логи - `.log`, статусы/коды - `.txt`, структуры - `.json`, скрины - `.png`.
* Для нескольких прогонов допускаются суффиксы `-v2`, `-v3` (фиксируйте в DV, какой смотреть).

---

## Definition of Done (S07 Evidence)

* [ ] Все **обязательные** файлы присутствуют и открываются.
* [ ] Если выбран харденинг - приложены соответствующие пруфы (A-F).
* [ ] В `GRADING/DV.md` даны **точные пути** к файлам из списка выше.
* [ ] README содержит краткую инструкцию «Запуск в контейнере» и «Проверки/health».

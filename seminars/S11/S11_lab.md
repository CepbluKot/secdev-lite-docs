# S11_lab.md - Пошаговый мини-лаб (20-30 минут)

## 0) Подготовка

- Создайте приватный репозиторий из шаблона **`secdev-seed-s09-s12`** (Use this template).
- Ничего добавлять не нужно: workflow `S11 - DAST (ZAP)` уже есть.

## 1) Запуск workflow «S11 - DAST (ZAP)»

1. Откройте **Actions** → *S11 - DAST (ZAP)* → **Run workflow** (или сделайте `git push`).
2. Дождитесь статуса **green**.

Что делает job:

- Ставит Python, поднимает FastAPI-приложение (`uvicorn`) на `:8080` и проверяет `/healthz`.
- Запускает **OWASP ZAP baseline** (`zap-baseline.py`) против `http://localhost:8080`.
- Сохраняет результаты → `EVIDENCE/S11/zap_baseline.html` и `EVIDENCE/S11/zap_baseline.json`.
- Публикует их как **artifacts**.

## 2) Проверка артефактов

- В репозитории откройте `EVIDENCE/S11/` - видны `zap_baseline.html/json`.
- В Actions скачайте artifacts и убедитесь, что отчёты совпадают.

## 3) Короткая запись в DS.md

- Откройте `GRADING/DS.md` и добавьте 3-6 строк по S11 (см. шаблон в **S11_brief.md**).
- Обязательно вставьте **ссылку на успешный job**.
- (1-2 строки) Кратко отметьте, что сделали с предупреждениями: *исправлено/принято риск/ложное срабатывание*.

## (Опционально) Полное сканирование локально

Полный активный скан может занимать время и «шуметь». Если хотите, запустите локально:

```bash
docker run --rm --network host -v $PWD:/zap/wrk owasp/zap2docker-stable   zap-full-scan.py -t http://localhost:8080 -r zap_full.html -J zap_full.json -d
mv zap_full.* EVIDENCE/S11/
```

> В CI оставляем только baseline.

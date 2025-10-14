# S10_lab.md - Пошаговый мини-лаб (15-25 минут)

## 0) Подготовка

- Создайте приватный репозиторий из шаблона **`secdev-seed-s09-s12`** (Use this template).
- Workflow и конфиги уже включены.

## 1) Запуск workflow «S10 - SAST & Secrets»

1. В вашем репозитории откройте **Actions** → *S10 - SAST & Secrets* → **Run workflow** (или сделайте `git push`).
2. Дождитесь завершения job (статус **green**).

Что делает job:

- **Semgrep** прогоняет готовый профиль `p/ci` и сохраняет отчёт в **SARIF** → `EVIDENCE/S10/semgrep.sarif`.
- **Gitleaks** ищет секреты в истории и рабочем дереве → `EVIDENCE/S10/gitleaks.json`.
- Папка `EVIDENCE/S10/` упаковывается как artifacts.

## 2) Проверка артефактов

- В репозитории убедитесь, что появились файлы в `EVIDENCE/S10/` (`semgrep.sarif`, `gitleaks.json`).
- В Actions скачайте artifacts и проверьте, что они совпадают.

## 3) Краткая запись в DS.md

- Откройте `GRADING/DS.md` и добавьте 3-6 строк по S10 (см. шаблон в **S10_brief.md**).
- Обязательно вставьте **ссылку на успешный job**.

## (Опционально) Локальный запуск через Docker

```bash
# Semgrep → SARIF
docker run --rm -v $PWD:/src returntocorp/semgrep:latest   semgrep ci --config p/ci --sarif --output /src/EVIDENCE/S10/semgrep.sarif --metrics=off || true

# Gitleaks → JSON
docker run --rm -v $PWD:/repo zricethezav/gitleaks:latest   detect --source=/repo --report-format=json --report-path=/repo/EVIDENCE/S10/gitleaks.json || true
```

> *Локалка не обязательна - достаточно Actions.*

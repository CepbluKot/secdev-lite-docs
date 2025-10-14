# S09_lab.md - Пошаговый мини‑лаб

## 0) Подготовка

- Зайдите в `secdev-seed-s09-s12` → **Use this template** → создайте свой приватный репозиторий команды.
- Ничего настраивать не нужно: workflow и конфиги уже внутри.

## 1) Запуск workflow «S09 - SBOM & SCA»

1. В вашем репозитории откройте **Actions** → *S09 - SBOM & SCA* → **Run workflow** (или просто сделайте `git push`).
2. Дождитесь завершения job (зелёный статус).

Что делает job:

- Генерирует **SBOM** (Syft, формат CycloneDX) → `EVIDENCE/S09/sbom.json`.
- Прогоняет **SCA** (Grype) → `EVIDENCE/S09/sca_report.json`.
- Делает короткую сводку → `EVIDENCE/S09/sca_summary.md`.
- Публикует содержимое `EVIDENCE/S09/` как **artifacts** Actions.

## 2) Проверка артефактов

- В репозитории убедитесь, что появились файлы в `EVIDENCE/S09/` (`sbom.json`, `sca_report.json`, `sca_summary.md`).
- В Actions скачайте artifacts и убедитесь, что в них те же файлы.

## 3) Краткая запись в DS.md

- Откройте свой `GRADING/DS.md` и добавьте 3-6 строк по S09 (см. шаблон в **S09_brief.md**).
- Обязательно вставьте **ссылку на успешный job**.

## (Опционально) Локальный запуск тех же действий

Если хотите повторить локально без ожидания Actions, используйте Docker‑образы как в CI:

```bash
# SBOM (CycloneDX)
docker run --rm -v $PWD:/work -w /work anchore/syft:latest packages dir:. -o cyclonedx-json > EVIDENCE/S09/sbom.json

# SCA (Grype), отчёт JSON + сводка
docker run --rm -v $PWD:/work -w /work anchore/grype:latest sbom:/work/EVIDENCE/S09/sbom.json -o json > EVIDENCE/S09/sca_report.json
echo "# SCA summary" > EVIDENCE/S09/sca_summary.md
jq '[.matches[].vulnerability.severity] | group_by(.) | map({(.[0]): length}) | add' EVIDENCE/S09/sca_report.json >> EVIDENCE/S09/sca_summary.md || true
```

> *Если нет Docker/jq - используйте только Actions.*

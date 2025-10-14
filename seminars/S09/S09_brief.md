# S09_brief.md - SBOM & SCA (Syft + Grype/Trivy)

**Цель:** научиться автоматически получать **SBOM** и проводить **SCA**‑скан зависимостей с фиксацией артефактов в репозитории и GitHub Actions.

**К чему привязано:** тренажёр для разделов DS.md про *SBOM/Dependencies (SCA)*. Студент **обязан** кратко зафиксировать результат в своём `GRADING/DS.md` (3-6 строк + ссылка на успешный job).

**Что делаем:**

- создаём репозиторий из шаблона **`secdev-seed-s09-s12`** (`Use this template`);
- запускаем workflow **`S09 - SBOM & SCA`**;
- получаем артефакты: `EVIDENCE/S09/sbom.json`, `sca_report.json`, `sca_summary.md` (+ artifacts в Actions);
- добавляем краткую запись в `GRADING/DS.md` (см. ниже «Вставка»).

**Ожидаемый минимальный результат:**

- job **зелёный**; артефакты **на месте**;
- в `GRADING/DS.md` есть короткая запись по S09 с **ссылкой на job**.

---

## Вставка в DS.md (мини‑шаблон)

> *Вставьте в специально отведённое место DS.md, 3-6 строк достаточно.*

```text
S09 - SBOM & SCA: сгенерирован SBOM (CycloneDX), выполнен SCA. 
Артефакты: EVIDENCE/S09/sbom.json, EVIDENCE/S09/sca_report.json. 
Сводка: EVIDENCE/S09/sca_summary.md. Actions: <ссылка на успешный job>.
```

---

## Ресурсы

- seed‑репозиторий: `secdev-seed-s09-s12` (в нём уже есть готовый workflow и конфиги)
- сканеры: [Syft] SBOM, [Grype] SCA, [Trivy] (альтернатива для SCA)

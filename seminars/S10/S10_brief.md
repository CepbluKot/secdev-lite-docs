# S10_brief.md - SAST & Secrets (Semgrep + Gitleaks)

**Цель:** отработать запуск **SAST** (Semgrep, формат вывода - SARIF) и **поиск секретов** (Gitleaks, JSON) с фиксацией артефактов и короткой записью в `DS.md`.

**К чему привязано:** разделы DS.md про *SAST* и *secrets scanning*. Студент **обязан** кратко зафиксировать результат в своём `GRADING/DS.md` (3-6 строк + ссылка на успешный job).

**Что делаем:**

- создаём репозиторий из шаблона **`secdev-seed-s09-s12`** (`Use this template`);
- запускаем workflow **`S10 - SAST & Secrets`** (Semgrep + Gitleaks);
- получаем артефакты: `EVIDENCE/S10/semgrep.sarif`, `gitleaks.json` (+ artifacts в Actions);
- добавляем краткую запись в `GRADING/DS.md` (см. «Вставка» ниже).

**Ожидаемый минимальный результат:**

- job **зелёный**; артефакты **на месте**;
- в `GRADING/DS.md` есть краткая запись по S10 с **ссылкой на job**.

---

## Вставка в DS.md (мини-шаблон)

> *Вставьте в отведённое место DS.md, 3-6 строк достаточно.*

```text
S10 - SAST & Secrets: выполнены Semgrep (SARIF) и Gitleaks (JSON).
Артефакты: EVIDENCE/S10/semgrep.sarif, EVIDENCE/S10/gitleaks.json.
Actions: <ссылка на успешный job>. Примечание: FP рассмотрены, при необходимости добавлены исключения.
```

---

## Ресурсы

- seed-репозиторий: `secdev-seed-s09-s12` (есть готовый workflow и конфиги)
- сканеры: [Semgrep] правила p/ci (быстрый старт для CI), [Gitleaks] поиск секретов
- SARIF полезен для последующей интеграции с code scanning

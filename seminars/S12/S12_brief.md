# S12_brief.md - IaC & Container Security (Hadolint, Checkov, Trivy; опц. подпись/аттестация)

**Цель:** проверить базовые артефакты безопасности для контейнеров и IaC:

- Dockerfile - **Hadolint** (lint как политика);
- `iac/` (Terraform/K8s) - **Checkov**;
- образ - **Trivy** (uвязвимости/конфиги);
- (опционально) подпись/аттестация образа.

**К чему привязано:** разделы DS.md про *контейнерный/IaC-харднинг* и «policy-as-code». Студент **обязан** кратко зафиксировать результат в `GRADING/DS.md` (3-6 строк + ссылка на успешный job) и перечислить **как минимум две применённые меры харднинга**.

**Что делаем:**

- создаём репозиторий из шаблона **`secdev-seed-s09-s12`** (`Use this template`);
- запускаем workflow **`S12 - IaC & Container Security`**;
- получаем артефакты: `EVIDENCE/S12/hadolint.json`, `checkov.json`, `trivy.json` (+ artifacts в Actions);
- фиксируем в `DS.md` результат + ≥2 меры харднинга.

**Минимальный результат:**

- job **зелёный**; артефакты **на месте**;
- в `DS.md` есть 3-6 строк по S12, **ссылка на job** и список **≥2 мер харднинга**.

---

## Вставка в DS.md (мини-шаблон)

> *Вставьте в отведённое место DS.md; 3-6 строк достаточно.*

```text
S12 - IaC & Container Security: Hadolint (Dockerfile), Checkov (iac/), Trivy (image).
Артефакты: EVIDENCE/S12/hadolint.json, checkov.json, trivy.json. Actions: <ссылка на успешный job>.
Применён харднинг (≥2 меры): 1) отказ от root; 2) фиксированный tag вместо latest; (пример) + добавлены limits в K8s.
```

---

## Примеры валидных мер харднинга

- **Dockerfile:** non-root пользователь; `HEALTHCHECK`; минимальный базовый образ; pin версий; drop лишних пакетов.
- **K8s:** `runAsNonRoot`, `readOnlyRootFilesystem`, ресурсы `limits/requests`, drop capabilities.
- **Образ:** отказ от `latest`; регулярный Trivy; (опц.) подпись/аттестация.

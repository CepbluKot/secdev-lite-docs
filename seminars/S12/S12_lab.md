# S12_lab.md - Пошаговый мини-лаб (20-30 минут)

## 0) Подготовка

- Создайте приватный репозиторий из шаблона **`secdev-seed-s09-s12`** (Use this template). Внутри уже есть Dockerfile, `iac/` и workflow.

## 1) Запуск workflow «S12 - IaC & Container Security»

1. Откройте **Actions** → *S12 - IaC & Container Security* → **Run workflow** (или сделайте `git push`).
2. Дождитесь статуса **green**.

Что делает job:

- `docker build` образ `s09s12-app:ci`;
- **Hadolint** проверяет Dockerfile → `EVIDENCE/S12/hadolint.json`;
- **Checkov** сканирует `iac/` (Terraform/K8s) → `EVIDENCE/S12/checkov.json`;
- **Trivy** сканирует образ → `EVIDENCE/S12/trivy.json`;
- публикует папку `EVIDENCE/S12/` как artifacts.

## 2) Разбор результатов и быстрый харднинг

- Откройте `EVIDENCE/S12/*.json`; выберите **≥2 простые меры** и внесите изменения (пример):  
  - Dockerfile: перейти на non-root, добавить `HEALTHCHECK`, выбрать фиксированный базовый образ вместо `latest`.
  - K8s: убрать `runAsUser: 0`, добавить `resources: limits/requests`, сделать FS `readOnly`.
- Коммит → перезапустите workflow → убедитесь, что замечаний стало меньше/другие.

## 3) Краткая запись в DS.md

- В `GRADING/DS.md` добавьте 3-6 строк по S12 (см. шаблон в **S12_brief.md**), **ссылку на job** и перечислите **≥2 мер харднинга**.
- (1-2 строки) Укажите, какие предупреждения остались и почему (FP/accept/дальнейший план).

## (Опционально) Политики/пороги и подпись

- Можно ввести «порог» (quality gate): например, падать на critical в Trivy/Hadolint/Checkov (сделайте это в форке/ветке).
- Подпись/аттестация образа (`cosign`) - вне рамок обязательной части, но можно упомянуть в DS.md как план.

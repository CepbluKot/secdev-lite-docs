# S08 - Secrets в CI (политика и практики)

> Цель: сделать так, чтобы **ни один секрет не оказался в репозитории, образе, артефакте или логе CI**. Всё конфигурируется через GitHub Actions **Secrets/Variables** и аккуратные шаги.

## 1) Где живут секреты (и где НЕ живут)

* ✅ **GitHub Actions → Settings → Secrets and variables:**

  * **Repository secrets** - достаточно для этого курса.
  * *(опц.)* **Environment secrets** - если используете среду с правилами защиты.
  * *(опц.)* **Organization secrets** - если один секрет общий для нескольких репо.
* ✅ **Variables** - для НЕсекретных значений (флаги, имена).
* ❌ **Никогда не коммитим**: `.env`, ключи, токены, пароли, cookie.
* ❌ **Не кладём секреты в Docker образ** (ни через `ARG`, ни через `COPY . .` с `.env`).

## 2) Как передавать секрет в job (без утечек)

Минимальный и безопасный способ:

```yaml
# .github/workflows/ci.yml
permissions:
  contents: read  # минимум прав GITHUB_TOKEN

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Use secret safely
        env:
          API_TOKEN: ${{ secrets.API_TOKEN }}   # добавить в Repo Secrets
        run: |
          python - <<'PY'
          import os
          token = os.getenv("API_TOKEN", "")
          print("token length:", len(token))  # не печатать сам токен!
          PY
```

Правила:

* Не логируйте значения секретов. Если очень нужно подтвердить «что-то пришло» - печатайте **только длину/хеш**.
* Не используйте `set -x`/трассировку команд - она печатает аргументы.
* Не пишите секреты в файлы из `EVIDENCE/` и не загружайте их как артефакты.

## 3) Контейнеры и секреты (S07/S08)

* **Запуск**: передавайте секреты на **рантайме** (`docker run -e`, `compose: environment:`) - но *в CI лайт-трека это необязательно*.
* **Сборка образа**: **не** используйте `ARG SECRET=...` - попадёт в слои. Если очень нужно (вне лайт-трека), используйте BuildKit `--secret` (за рамками курса).

## 4) Как доказать DV2/DV5

Положите в репозиторий **ровно то, что проверяющему нужно**:

* `EVIDENCE/S08/secrets-proof.txt` - 3-6 строк:

  ```text
  Secrets stored in: Actions → Repo Secrets (list of names only)
  No secrets in YAML/logs. Checked steps: Checkout, Setup, Cache, Tests.
  .env not committed. Only .env.example present.
  ```

* В `GRADING/DV.md` коротко: где секреты хранятся, где **их нет**, ссылка на `secrets-proof.txt`.
* В README («Как устроен CI») одной строкой: «секреты - через Actions Secrets; в лог не печатаются».

## 5) Частые опасности (и как избежать)

* ❌ **Печать секретов в лог**: не делайте `echo $API_TOKEN`.
  ✅ Печатайте `len($API_TOKEN)` или используйте сообщение «secret present: yes/no».
* ❌ **`.env` в репо/образе**: добавьте в `.gitignore` и `.dockerignore`; используйте `.env.example`.
* ❌ **Secrets в артефактах**: не записывайте секреты в файлы, которые собирает `actions/upload-artifact`.
* ❌ **Завышенные permissions**: не давайте `write`/`packages: write`, если не публикуете артефакты в реестры.

## 6) Мини-чеклист (поставьте галочки)

* [ ] В репозитории **нет** `.env`/секретов (есть `/.env.example`).
* [ ] Все приватные значения заведены в **Actions Secrets** (перечислите **имена** в `secrets-proof.txt`).
* [ ] В YAML нет «захардкоженных» секретов/токенов.
* [ ] В логах нет значений секретов (только длина/«present: yes»).
* [ ] Артефакты не содержат секретов.
* [ ] `permissions:` минимальны.

## 7) Сниппеты «под ключ»

**Минимальные права и маскирование:**

```yaml
permissions:
  contents: read
  actions: read

- name: Mask any incidental value (подстраховка)
  run: echo "::add-mask::${{ secrets.API_TOKEN }}"
```

**Отдельный шаг-пруф для DV2 (не печатает секрет):**

```yaml
- name: Secrets proof (DV2)
  env:
    API_TOKEN: ${{ secrets.API_TOKEN }}
  run: |
    if [ -z "${API_TOKEN}" ]; then echo "secret present: no"; exit 1; fi
    echo "secret present: yes" > EVIDENCE/S08/secrets-proof.txt
```

## 8) Что положить в `EVIDENCE/S08/`

* `secrets-proof.txt` - как выше.
* *(опц.)* `ci-log.txt` - фрагменты лога, где **секретов нет** (пары строк из шагов).
* *(опц.)* `git-status-no-env.txt` - вывод `git ls-files | grep -E '(^|/)\.env$' || echo 'no .env'`.

## 9) Definition of Done (секреты в CI)

* [ ] Repo чистый: `.env` не закоммичен; есть `.env.example`.
* [ ] Secrets - **только** в Actions Secrets (имена перечислены).
* [ ] Логи без значений секретов (есть `secrets-proof.txt`).
* [ ] `GRADING/DV.md` и README обновлены (DV2/DV5).

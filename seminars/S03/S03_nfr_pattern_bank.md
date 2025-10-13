# S03 - Банк NFR (Given-When-Then)

Этот банк помогает быстро выбрать и **адаптировать** проверяемые NFR под ваши User Stories, не изобретая велосипед.
Формат каждого шаблона: **Requirement** → **Acceptance (G-W-T)** → краткое **Rationale** → идеи для **Evidence** (что подтвердит выполнение).

## Как пользоваться банком

1. Выберите 6-8 категорий, релевантных вашим User Stories.
2. Адаптируйте числа/пороги/эндпойнты/контекст.
3. Для каждого требования заполните **G-W-T** так, чтобы оно реально проверялось тестом/логом/сканером/политикой.
4. Добавьте **Trace (issue/link)** и **Owner** в свой реестр.

## Условные обозначения (замените на свои значения)

* `<endpoint>` - конкретный маршрут API, например `/api/items`.
* `<minutes>`, `<ms>`, `<RPS>` - пороги (минуты, миллисекунды, запросы/с).
* `<tenant>` - идентификатор арендатора/организации.
* `<role>` - роль (admin/manager/viewer и т. п.).
* `<limit>` - числовой лимит (размер, количество, скорость).
* `<status>` - код ответа (200, 401, 403, 413, 429, 5xx и т. п.).

---

## 1) Performance (`Performance`)

**Кейс:** время отклика и ошибки на целевых нагрузках.

**Requirement A:** P95 латентность для `<endpoint>` ≤ **<ms>** при **<RPS>** в течение **<minutes>**.
**Acceptance (G-W-T):**

```Gherkin
Given сервис развернут и здоров
When на <endpoint> подается <RPS> RPS в течение <minutes> минут
Then P95 ≤ <ms> и доля ошибок ≤ 1%
```

**Rationale:** UX/SLO.
**Evidence (идеи):** отчёт нагрузочного теста; метрика `http_server_requests_seconds` (квантили); лог ошибок.

---

## 2) Scalability (`Scalability`)

**Кейс:** двукратный рост нагрузки без ручных действий.

**Requirement:** При росте нагрузки 1×→2× автоскейлинг удерживает P95 ≤ **1.2×** базового.
**Acceptance (G-W-T):**

```Gherkin
Given базовая нагрузка <RPS_base> RPS и P95=<ms_base>
When нагрузка увеличена до 2× (<RPS_base>*2) в течение <minutes>
Then P95 ≤ 1.2 * <ms_base> и не выше 1% 5xx
```

**Rationale:** устойчивость к всплескам.
**Evidence:** отчёт стресс-теста; график автоскейлинга.

---

## 3) Rate Limiting / Throttling (`RateLimiting`)

**Кейс:** ограничения скорости на токен/аккаунт/IP.

**Requirement:** На `<endpoint>` действует лимит **<limit> req/min** на токен; превышение → **429 + Retry-After**.
**Acceptance (G-W-T):**

```Gherkin
Given активный токен
When выполняется <limit>+N запросов к <endpoint> за 60 секунд
Then лишние запросы получают 429 и корректный заголовок Retry-After
```

**Rationale:** защита от DoS/абьюза.
**Evidence:** e2e-тест лимита; логи с 429 и `Retry-After`.

---

## 4) Timeouts / Retry / Circuit Breaker (`Timeouts/Retry/CircuitBreaker`)

**Кейс:** внешние вызовы не «висят» бесконечно.

**Requirement:** Исходящие вызовы к `<ext_service>`: timeout ≤ **2s**, retry ≤ **3** с джиттером; circuit-breaker при ошибках **≥50%** за **1 мин**.
**Acceptance (G-W-T):**

```Gherkin
Given недоступность <ext_service>
When сервис вызывает <ext_service> при обращении к <endpoint>
Then суммарное ожидание ≤ 6s; выполняются не более 3 retry с джиттером; после порога ошибок включается circuit breaker
```

**Rationale:** устойчивость к зависимостям.
**Evidence:** конфиг клиента; интеграционный тест отказов; логи breaker.

---

## 5) API Contract / Errors (`API-Contract/Errors`, RFC 7807)

**Кейс:** единый формат ошибок, без утечек.

**Requirement:** Ошибки отдаются в формате **RFC 7807** (`application/problem+json`) без стэктрейсов.
**Acceptance (G-W-T):**

```Gherkin
Given серверная ошибка при GET <endpoint>
When клиент получает ответ
Then Content-Type=application/problem+json, поля type/title/status/detail присутствуют, стэктрейсов нет, есть correlation_id
```

**Rationale:** предсказуемость и безопасность.
**Evidence:** e2e-пример ответа; контракт-тест.

---

## 6) Security - Authentication (`Security-AuthN`)

**Кейс:** защита write-операций JWT/сессией.

**Requirement:** Все write-эндпойнты требуют валидный токен; истёкший/некорректный → **401**.
**Acceptance (G-W-T):**

```Gherkin
Given истекший JWT
When POST <endpoint_write>
Then 401 с телом в RFC 7807
```

**Rationale:** базовая линия безопасности.
**Evidence:** интеграционный тест; пример ответа 401.

---

## 7) Security - Authorization / RBAC / Tenant Isolation (`Security-AuthZ/RBAC`)

**Кейс:** доступ строго по ролям и арендаторам.

**Requirement:** Пользователь `<tenant=A>` не может читать/менять ресурсы `<tenant=B>`; роль `<role>` ограничивает действия.
**Acceptance (G-W-T):**

```Gherkin
Given пользователь с ролью <role> из tenant A
When он запрашивает ресурс tenant B
Then ответ 404 (или 403 по политике) без утечки данных
```

**Rationale:** изоляция данных, наименьшие привилегии.
**Evidence:** тесты изоляции; policy-описание.

---

## 8) Security - Input Validation (`Security-InputValidation`)

**Кейс:** размер/тип/схема запросов.

**Requirement:** Для `<endpoint>`: размер тела ≤ **<limit> MiB**, MIME только из allowlist, **extra поля запрещены**.
**Acceptance (G-W-T):**

```Gherkin
Given тело 2× больше <limit> MiB
When POST <endpoint>
Then 413 и тело ошибки в RFC 7807; неизвестные поля приводят к 400
```

**Rationale:** защита от DoS/инъекций/грязных данных.
**Evidence:** e2e-тест; схема DTO/валидатор.

---

## 9) Security - Secrets Management (`Security-Secrets`)

**Кейс:** секреты не хранятся в коде, управляются централизованно.

**Requirement:** В репозитории **0** секретов; рантайм - только из защищённого хранилища/переменных окружения.
**Acceptance (G-W-T):**

```Gherkin
Given репозиторий и его история
When выполняется скан секретов
Then findings = 0; конфигурация сервиса читает секреты из env/KMS/manager
```

**Rationale:** предотвращение утечек.
**Evidence:** отчёт сканера секретов; фрагмент настроек рантайма.

---

## 10) Data Integrity / Canonicalization (`Data-Integrity`)

**Кейс:** параметризация и канонизация данных.

**Requirement:** Все SQL-запросы параметризованы; строки/даты/номера приводятся к канонической форме.
**Acceptance (G-W-T):**

```Gherkin
Given входные данные с нестандартными форматами
When выполняется запись в БД
Then используется параметризация; значения нормализованы (NFC/E.164/UTC/Decimal)
```

**Rationale:** защита от инъекций, целостность.
**Evidence:** ссылочный пример кода/ORM-настройки; тест канонизации.

---

## 11) Container / Runtime Baseline (`Container/Runtime`)

**Кейс:** безопасный образ и запуск.

**Requirement:** Контейнер запускается **не от root**; root FS **read-only**; задан `HEALTHCHECK`.
**Acceptance (G-W-T):**

```Gherkin
Given контейнер запущен
When выполняется проверка параметров рантайма
Then user != root, файловая система read-only, healthcheck проходит
```

**Rationale:** снижение ущерба при компрометации.
**Evidence:** вывод инспекции контейнера; фрагмент Dockerfile/Compose.

---

## 12) Observability / Logging (`Observability/Logging`)

**Кейс:** структурные логи и сквозной `correlation_id`.

**Requirement:** Все запросы логируются в **JSON** и содержат `correlation_id` на каждом этапе.
**Acceptance (G-W-T):**

```Gherkin
Given запрос с заголовком X-Correlation-ID=<id>
When он проходит через сервис
Then во всех логах появляется correlation_id=<id> и ключевые поля запроса/ответа
```

**Rationale:** трассировка и разбор инцидентов.
**Evidence:** паттерн логов; скрин/запрос поиска по id.

---

## 13) Auditability (`Auditability`)

**Кейс:** неизменяемый журнал админ-действий.

**Requirement:** Для административных операций создаётся **audit-событие** (actor, action, target, time, result).
**Acceptance (G-W-T):**

```Gherkin
Given администратор удаляет пользователя
When операция завершается
Then создается audit-запись с actor, target, временем и результатом; запись неизменяема
```

**Rationale:** соответствие политикам, расследование.
**Evidence:** пример audit-записи; политика хранения.

---

## 14) Privacy / PII (`Privacy/PII`)

**Кейс:** маскирование PII в логах и сроки хранения.

**Requirement:** PII не попадает в логи; сырые PII хранятся не дольше **<days>** дней.
**Acceptance (G-W-T):**

```Gherkin
Given DTO с персональными данными
When происходит логирование
Then поля PII маскированы/удалены; план удаления/ретенции существует и применяется в срок <days> дней
```

**Rationale:** приватность и комплаенс.
**Evidence:** пример логов с маскировкой; регламент ретенции.

---

## 15) Availability / Health (`Availability/Health`)

**Кейс:** готовность сервиса после старта.

**Requirement:** Через **<seconds>** секунд после запуска `/health/ready` отвечает **200**.
**Acceptance (G-W-T):**

```Gherkin
Given старт контейнера
When проходит <seconds> секунд
Then GET /health/ready возвращает 200 и зависимости отмечены healthy
```

**Rationale:** предсказуемость деплоя и проверок.
**Evidence:** конфиг readiness/startup probes; журнал healthcheck.

---

## 16) Maintainability / Feature Flags (`Maintainability/FeatureFlags`)

**Кейс:** управление функционалом без релиза.

**Requirement:** Флаги фич переключаются **без перезапуска**; смена состояния логируется.
**Acceptance (G-W-T):**

```Gherkin
Given фича защищена флагом <flag_name>
When флаг переключается в runtime
Then поведение меняется без релиза; событие зафиксировано в логах с actor и временем
```

**Rationale:** безопасные релизы и быстрые откаты.
**Evidence:** конфигурация флагов; лог переключения.

---

### Замечания по адаптации

* В **Requirement** всегда указывайте **конкретные числа/пороги** и **границы действия** (на уровне эндпойнта/ролей/тенанта).
* В **G-W-T** избегайте расплывчатых утверждений - фиксируйте **наблюдаемые факты** (код ответа, квантиль, наличие заголовка, запись в журнале).
* **Evidence** - это подсказка, чем подтвердите выполнение (тест, лог, скан, политика). Где хранить доказательства - решается позже в TM/DV/DS.

> Готово. Выбирайте категории, копируйте шаблоны в свой реестр (`S03_register.*`) и адаптируйте под ваши User Stories.

# S05 — Банк паттернов для ADR (облегчённый)

> Как пользоваться:
>
> 1. Найдите паттерн под ваш **Risk-ID** из S04.
> 2. Адаптируйте **Decision/Defaults** под свой контекст.
> 3. Перенесите в `S05_ADR_<slug>.md` (шаблон ADR) и свяжите с **NFR-ID** из S03.

---

## Содержание

1. JWT TTL + Refresh + Rotation
2. RBAC + Tenant Isolation
3. Input Validation & Size Limits
4. API Errors: RFC7807 + correlation_id
5. Rate Limiting + 429 + Retry-After
6. Timeouts + Retry (jitter) + Circuit Breaker
7. Secrets Management (no secrets in repo)
8. PII Masking + Data Retention
9. Observability: Structured JSON Logs
10. Audit Log for Admin Actions
11. Idempotency Keys for Writes
12. File Uploads: MIME Allowlist + Limits
13. Container Baseline (non-root, ro FS, healthcheck)
14. DB Safety: Parameterization + Canonicalization
15. Feature Flags (runtime toggles)
16. External API: Async Fallback (Queue)

---

## 1) JWT TTL + Refresh + Rotation

**Когда:** угрозы S (краденый/повторно используемый токен), публичный периметр.
**Decision:**

* Access token TTL = **<15–30m>**, Refresh TTL = **<7–30d>**.
* Key rotation (kid, JWKS), clock skew **<±60s>**.
* Logout/blacklist refresh по событию.
  **Defaults:** short TTL, single audience, HTTPS only, secure/httponly cookies (если cookie-based).
  **Альтернативы:** MTLS для машинных клиентов; OAuth Device Flow.
  **Consequences:** +снижение окна атаки; −сложнее управление сессиями.
  **DoD (G-W-T):**
  Given истёкший access token
  When POST `<endpoint_write>`
  Then 401 в RFC7807; при обмене refresh → новый access с новым `exp`.
  **NFR link:** Security-AuthN, RateLimiting, API-Contract/Errors.

---

## 2) RBAC + Tenant Isolation

**Когда:** угрозы E (повышение привилегий), I (утечка между арендаторами).
**Decision:**

* Политика deny-by-default; проверка `tenant_id` и `role` на каждом DAO/handler.
* Ошибка при межтенантном доступе → **404/403** (без утечки).
  **Defaults:** роли `admin/manager/viewer`; атрибуты на уровне записи.
  **Альтернативы:** ABAC/OPA policy.
  **Consequences:** +изоляция; −требует дисциплины на всех слоях.
  **DoD:**
  Given пользователь из tenant A
  When читает ресурс tenant B
  Then 404/403, отсутствует data leakage.
  **NFR link:** Security-AuthZ/RBAC, Privacy/PII.

---

## 3) Input Validation & Size Limits

**Когда:** T (tampering), D (DoS), грязный ввод.
**Decision:**

* Body size ≤ **<MiB>**, fields only from schema, MIME allowlist.
* Reject extra fields → **400**, oversize → **413**.
  **Defaults:** DTO-schema (server-side), unified error format.
  **Альтернативы:** API Gateway schema validation.
  **Consequences:** +предсказуемость; −жёстче к клиентам.
  **DoD:**
  Given body 2×limit и неизвестные поля
  When POST `<endpoint>`
  Then 413/400 + RFC7807.
  **NFR link:** Security-InputValidation.

---

## 4) API Errors: RFC7807 + correlation_id

**Когда:** R (repudiation), I (утечки в ошибках).
**Decision:**

* Все ошибки → `application/problem+json`, без стэктрейсов.
* Включать `correlation_id` в ответ.
  **Defaults:** common error middleware.
  **Альтернативы:** gRPC status + rich details.
  **Consequences:** +диагностика; −первичная настройка.
  **DoD:**
  Given internal error
  When GET `<endpoint>`
  Then 500 с полями type/title/status/detail и `correlation_id`.
  **NFR link:** API-Contract/Errors, Observability/Logging.

---

## 5) Rate Limiting + 429 + Retry-After

**Когда:** D (DoS), brute-force, abuse.
**Decision:**

* `<limit>` req/min per token/IP; превышение → **429** + `Retry-After`.
  **Defaults:** скользящее окно 60s, burst-bucket.
  **Альтернативы:** тарифные квоты per plan.
  **Consequences:** +защита; −риск ложных срабатываний.
  **DoD:**
  Given `<limit>+N` запросов за 60s
  When вызываем `<endpoint>`
  Then часть получает 429 с корректным `Retry-After`.
  **NFR link:** RateLimiting/Throttling.

---

## 6) Timeouts + Retry (jitter) + Circuit Breaker

**Когда:** D (залипание на внешних), T (частичные сбои).
**Decision:**

* Outbound timeout ≤ **<2s>**, retries ≤ **<3>** с джиттером, CB при error-rate **≥<50%>** / 1 мин.
  **Defaults:** exponential backoff, hedging off.
  **Альтернативы:** async queue (см. 16).
  **Consequences:** +устойчивость; −возможны спайки retry.
  **DoD:**
  Given зависание `<ext_service>`
  When вызов `<endpoint>`
  Then суммарно ≤ **<6s>**, retry ≤ **3**, CB активируется.
  **NFR link:** Timeouts/Retry/CircuitBreaker.

---

## 7) Secrets Management (no secrets in repo)

**Когда:** I (утечка секретов).
**Decision:**

* Секреты только из env/KMS/secret-manager; скан истории репо → findings = 0.
  **Defaults:** rotate on schedule, access least-privilege.
  **Альтернативы:** sealed-secrets/CSI.
  **Consequences:** +снижение риска утечки; −чуть сложнее локалка.
  **DoD:**
  Given скан по истории
  When запускаем secret-scan
  Then 0 находок; рантайм читает из manager.
  **NFR link:** Security-Secrets.

---

## 8) PII Masking + Data Retention

**Когда:** I (приватность в логах/дампах).
**Decision:**

* Маскируем PII в логах; retention для raw PII ≤ **<30d>**.
  **Defaults:** allowlist логируемых полей.
  **Альтернативы:** псевдонимизация/токенизация.
  **Consequences:** +комплаенс; −меньше деталей в отладке.
  **DoD:**
  Given DTO с PII
  When лог пишется
  Then PII отсутствует/маскировано; ретенция соблюдается.
  **NFR link:** Privacy/PII.

---

## 9) Observability: Structured JSON Logs

**Когда:** R/D/I (диагностика/корреляция).
**Decision:**

* JSON-логи, `correlation_id` сквозной, ключевые поля запроса/ответа.
  **Defaults:** парсинг в лог-хранилище, sample-rate.
  **Альтернативы:** трассировка (OpenTelemetry).
  **Consequences:** +разбор инцидентов; −стоимость хранения.
  **DoD:**
  Given `X-Correlation-ID=<id>`
  When запрос проходит через сервис
  Then все логи содержат `correlation_id=<id>`.
  **NFR link:** Observability/Logging.

---

## 10) Audit Log for Admin Actions

**Когда:** R (repudiation), E (опасные операции).
**Decision:**

* Фиксировать `actor, action, target, time, result`; неизменяемость.
  **Defaults:** отдельное хранилище/stream.
  **Альтернативы:** SIEM корелляция.
  **Consequences:** +разбор инцидентов; −доп. хранение.
  **DoD:**
  Given admin удаляет пользователя
  When операция завершена
  Then создана audit-запись с полями выше.
  **NFR link:** Auditability.

---

## 11) Idempotency Keys for Writes

**Когда:** T/D (повторы запросов, сетевые сбои).
**Decision:**

* Для POST/PUT — обязательный `Idempotency-Key`; хранить окна **<24h>**.
  **Defaults:** ключ в заголовке, hash тела.
  **Альтернативы:** серверное дедуплирование по контенту.
  **Consequences:** +отсутствие дублей; −хранилище ключей.
  **DoD:**
  Given повтор с тем же ключом
  When POST `<endpoint>`
  Then 200/409 детерминированно, без дублей.
  **NFR link:** Idempotency, Data-Integrity.

---

## 12) File Uploads: MIME Allowlist + Limits

**Когда:** I/D (файлы/аватарки/импорты).
**Decision:**

* Allowlist MIME, размер ≤ **<MiB>**, скан метаданных, хранение под UUID.
  **Defaults:** вирус-скан опционально (если доступно).
  **Альтернативы:** прямой аплоад в объектное хранилище + pre-signed URL.
  **Consequences:** +сafety; −сложнее UX при жёстких лимитах.
  **DoD:**
  Given запрещённый тип
  When загрузка файла
  Then 415/400 и RFC7807.
  **NFR link:** InputValidation, Storage-Quotas.

---

## 13) Container Baseline (non-root, read-only FS, healthcheck)

**Когда:** E/D (снижение ущерба при компрометации).
**Decision:**

* `USER` ≠ root, FS read-only (исключения в `/tmp`), `HEALTHCHECK` обязателен.
  **Defaults:** урезанные capabilities.
  **Альтернативы:** sandbox/sekcomp профили.
  **Consequences:** +минимизация урона; −иногда нужны исключения.
  **DoD:**
  Given контейнер запущен
  When проверяем рантайм
  Then user≠root, ro-FS, healthcheck OK.
  **NFR link:** Container/Runtime.

---

## 14) DB Safety: Parameterization + Canonicalization

**Когда:** T/I (инъекции, неканоничные данные).
**Decision:**

* Только параметризованные запросы; нормализация строк/дат/телефонов.
  **Defaults:** ORM с параметрами по умолчанию.
  **Альтернативы:** stored procedures only.
  **Consequences:** +целостность; −проверки миграций.
  **DoD:**
  Given «грязный» ввод
  When запись в БД
  Then инъекции нет; данные каноничны.
  **NFR link:** Data-Integrity.

---

## 15) Feature Flags (runtime toggles)

**Когда:** управляемый rollout/rollback.
**Decision:**

* Флаги переключаемы **без релиза**, логировать смену состояния.
  **Defaults:** процентный rollout, kill-switch.
  **Альтернативы:** конфиги с hot-reload.
  **Consequences:** +безопасные релизы; −управление матрицей флагов.
  **DoD:**
  Given включение `<flag>`
  When переключаем в runtime
  Then поведение меняется без деплоя; лог содержит событие.
  **NFR link:** Maintainability/FeatureFlags.

---

## 16) External API: Async Fallback (Queue)

**Когда:** частые таймауты/ограничения внешних.
**Decision:**

* Перевести операции в фоновые job через очередь; клиент получает 202 + локацию статуса.
  **Defaults:** саги/компенсации при сбоях.
  **Альтернативы:** sync + CB (см. паттерн 6).
  **Consequences:** +устойчивость, −сложнее UX/статусы.
  **DoD:**
  Given деградация внешнего API
  When запрос выполняется
  Then ответ 202 с URL статуса; job завершается с ретраями и логируется.
  **NFR link:** Timeouts/Retry, Observability/Logging.

---

### Примечания по адаптации

* Всегда указывайте **границы действия** решений (endpoints/roles/tenants).
* Для чисел используйте реалистичные **дефолты**, затем уточняйте по замерам.
* В **DoD/Acceptance** фиксируйте наблюдаемые факты (коды, заголовки, поля, метрики), чтобы их можно было проверить тестом/логом/сканом.

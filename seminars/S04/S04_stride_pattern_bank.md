# S04 - STRIDE Pattern Bank (готовые варианты для копирования)

> Использование: выберите **релевантные** угрозы для элементов вашей DFD и вставьте строки в `S04_stride_matrix.md`. Элементы и рёбра названы так же, как в шаблоне DFD (Internet→API, API/Controller, Service, DB, Service→External, *(опц.)* Queue). Приоритизацию делайте отдельно в `S04_risk_scoring.md`.
> Легенда NFR (пример): `NFR-AuthN`, `NFR-RateLimit`, `NFR-API-Contract/Errors`, `NFR-Privacy/PII`, `NFR-Timeouts/Retry/CB`, `NFR-Observability/Logging`, `NFR-Audit`, `NFR-AuthZ/RBAC`, `NFR-TenantIsolation`, `NFR-Config/Secrets`, `NFR-Data-Integrity`.

---

## A) Edge: **Internet → API** (публичная поверхность)

* **S** - JWT replay/forgery на публичных эндпоинтах → NFR-AuthN, NFR-RateLimit → *Mitigation:* короткий **JWT TTL + refresh**, проверка `iss/aud/exp/nbf`, clock-skew, rate limit `/auth/*`.
* **S** - Отсутствует mTLS/привязка клиента для интеграционных ключей → NFR-AuthN → *Mitigation:* mTLS для машинных клиентов, привязка IP/клиентских ключей.
* **T** - Подмена параметров/JSON-инъекция в невалидированном теле запроса → NFR-InputValidation → *Mitigation:* Pydantic-схемы, строгое `content-type`, лимиты размера тела.
* **T** - Directory traversal при загрузке файлов → NFR-InputValidation → *Mitigation:* запрет относительных путей, генерация имён, антивирус/типизация MIME.
* **R** - Нет traceability (нет `correlation_id`, логов логинов/ошибок) → NFR-Observability/Logging → *Mitigation:* корелляция запросов, структурные логи, уровни.
* **I** - Подробные ошибки/stacktrace наружу → NFR-API-Contract/Errors → *Mitigation:* RFC7807-ответы без чувствительных деталей.
* **I** - Слабый CORS/утечка через `Referrer` → NFR-Privacy/PII, NFR-API-Contract → *Mitigation:* строгий CORS (origin, методы, заголовки), `Referrer-Policy: no-referrer`.
* **D** - Нет rate-limit/body-limit/timeout → NFR-RateLimit → *Mitigation:* лимиты RPS/burst, ограничение размеров, server-timeouts.
* **E** - IDOR/недостаточные проверки авторизации → NFR-AuthZ/RBAC, NFR-TenantIsolation → *Mitigation:* проверка владения ресурсом, мандатный tenant-scoping.

---

## B) Node: **API / Controller**

* **S** - Проброс доверия внутренним сервисам без проверки токена сервиса → NFR-AuthN (service-to-service) → *Mitigation:* отдельные сервисные токены/mTLS.
* **T** - Header/host injection при проксировании → NFR-InputValidation → *Mitigation:* нормализация/блок-лист опасных заголовков, фиксированный upstream host.
* **R** - Логи без user/tenant/route → NFR-Observability/Logging → *Mitigation:* шаблон логов (user_id, tenant_id, route, status, latency).
* **I** - PII в логах/метриках → NFR-Privacy/PII → *Mitigation:* маскирование, запрет PII в trace, data-classification.
* **D** - Безграничный пул/воркеры → NFR-DoS/Resilience → *Mitigation:* лимиты concurrency/queue, back-pressure.
* **E** - Debug/админ ручки без RBAC → NFR-AuthZ/RBAC → *Mitigation:* feature-flag + RBAC, защита от сканирования маршрутов.

---

## C) Node: **Service / Бизнес-логика**

* **S** - Отсутствует взаимная аутентификация к DB/внешним → NFR-AuthN → *Mitigation:* выделенные учётки, ротация, минимальные права.
* **T** - Command/OS-инъекция при вызове shell-утилит → NFR-InputValidation → *Mitigation:* `subprocess` без shell, whitelisting аргументов.
* **T** - SQL-конкатенация (динамические фильтры) → NFR-Data-Integrity → *Mitigation:* **параметризация**, ограничители списков/диапазонов.
* **R** - Нет бизнес-аудита ключевых действий → NFR-Audit → *Mitigation:* аудит-топики/таблицы, неизменяемые записи.
* **I** - Секреты в коде/конфиге → NFR-Config/Secrets → *Mitigation:* env-переменные, secret-store, `.env.example`, ревью на утечки.
* **D** - Нет timeout/retry/circuit breaker при внешних вызовах → NFR-Timeouts/Retry/CB → *Mitigation:* timeouts ≤2s, retry≤3 с джиттером, CB.
* **E** - Обход бизнес-контролей (feature-flags/параметры) → NFR-AuthZ/RBAC → *Mitigation:* серверные гварды, неизменяемые флаги на сервере.

---

## D) Node: **DB / Хранилище**

* **S** - Учётка приложения с избыточными правами → NFR-AuthZ/RBAC → *Mitigation:* отдельный **least-privilege** пользователь, запрет `DDL/DROP`.
* **T** - Tampering данных/схемы без целостности → NFR-Data-Integrity → *Mitigation:* транзакции, foreign keys, миграции с проверками, checksum.
* **R** - Нет аудит-логов доступа/админ-операций → NFR-Audit → *Mitigation:* включить AUDIT, хранить отдельно/неизменяемо.
* **I** - PII без маскирования/шифрования/бэкапы открыты → NFR-Privacy/PII → *Mitigation:* колоночное шифрование/маскирование, контроль доступа к бэкапам.
* **D** - Долгие запросы/нет индексов → NFR-DoS/Resilience → *Mitigation:* индексы, лимиты на `LIKE %...%`, пагинация.
* **E** - Обход row-level security/тенант-изоляции → NFR-TenantIsolation → *Mitigation:* RLS-политики/tenant-scoping в запросах.

---

## E) Edge: **Service → External API**

* **S** - Доверие внешнему без проверки (pinning/issuer) → NFR-AuthN → *Mitigation:* TLS-pinning (где возможно), проверка CA, валидация ответов.
* **T** - Подмена/искажение ответа при слабом TLS/валидации → NFR-API-Contract → *Mitigation:* строгая проверка схем/подписей, idempotency keys.
* **R** - Нет отзыв-идентификаторов/idempotency для платёжных/критичных вызовов → NFR-Audit → *Mitigation:* idempotency-key, лог-журнал.
* **I** - Передача PII/секретов в query/логи → NFR-Privacy/PII → *Mitigation:* только в заголовках/теле, redaction логов.
* **D** - Отсутствуют retry-параметры/CB/лимиты → NFR-Timeouts/Retry/CB → *Mitigation:* разумные retry с лимитом, circuit breaker, бюджет времени.
* **E** - Ошибки в OAuth-потоке (over-scoped токен) → NFR-AuthZ/RBAC → *Mitigation:* минимальные scope, ротация/ревокация.

---

## F) *(Опционально)* Edge: **Events / Queue**

* **S** - Спуфинг продюсера/консюмеров → NFR-AuthN → *Mitigation:* аутентификация клиентов, ACL на топики/очереди.
* **T** - Тамперинг сообщений (без подписи/валидации схем) → NFR-Data-Integrity → *Mitigation:* схема-регистры, подпись/хеш сообщений.
* **R** - Нет аудита публикаций/доставки → NFR-Audit → *Mitigation:* лог поставки/неудач, DLQ мониторинг.
* **I** - PII в событиях без маскирования → NFR-Privacy/PII → *Mitigation:* редактирование/классификация, полиси ретенции.
* **D** - Back-pressure/переполнение очереди → NFR-DoS/Resilience → *Mitigation:* лимиты продюсеров, квоты, алерты на длину очереди.
* **E** - Доступ к админ-топикам/операциям → NFR-AuthZ/RBAC → *Mitigation:* разделение ролей, запрет админ-операций из приложений.

---

## G) Общие паттерны **на границах доверия**

* **S** - Отсутствует чёткая граница доверия (всё «в одном мешке») → *Mitigation:* выделить Internet/Service/External, разграничить учётки/ключи.
* **R** - Нет сквозного `correlation_id` через границы → *Mitigation:* прокидывать header, логировать на всех хопах.
* **I** - «Шумные» логи на границах содержат PII → *Mitigation:* маскирование на ingress/egress.
* **D** - Нет back-pressure/лимитов между зонами → *Mitigation:* rate-limit, очереди, таймауты/CB на выход за границу.

---

## Как переносить в ваши файлы S04

1. В `S04_stride_matrix.md` копируйте только **релевантные** вашей DFD строки из банка:
   `Element: <узел/ребро> | Data/Boundary: <тип/граница> | Threat: <S/T/R/I/D/E> | Description: <1-2 предложения> | NFR link: <ID> | Mitigation idea: <коротко>`

2. В `S04_risk_scoring.md` **объедините дубли**, поставьте **L** и **I** (1-5), посчитайте `Score = L×I` и выберите **Top-5** под ADR S05.

3. Держите DFD компактной (3-5 узлов) и явными границами доверия - так проще проходить STRIDE per element без воды.  

---

## Мини-самопроверка

* [ ] Для **каждого элемента** DFD выбран максимум 0-2 релевантные угрозы на букву STRIDE.
* [ ] У каждой угрозы указан **NFR-ID** (*или* `need NFR`).
* [ ] У каждой угрозы есть **Mitigation idea**, готовая стать ADR на S05.
* [ ] В risk scoring объединены дубли, посчитан **Score = L×I**, выделен **Top-5**.

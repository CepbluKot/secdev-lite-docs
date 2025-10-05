# S03 — Примерный перечень User Stories

> Формат ниже:
> **ID**, **Роль–Цель–Ценность** («Как [роль] я хочу [цель], чтобы [ценность]»), **Кратко**, **API/Endpoints (пример)**, **NFR hooks (tags)**.
> Эти истории — ориентиры. Выберите 2–3 и адаптируйте под свой контекст.

---

## US-001 — Регистрация пользователя

* **Роль–Цель–Ценность:** Как гость, я хочу зарегистрировать учётную запись, чтобы получить доступ к функционалу приложения.
* **Кратко:** Регистрация с подтверждением e-mail.
* **API/Endpoints (пример):** `POST /api/auth/register`, `POST /api/auth/verify-email`
* **NFR hooks:** `Security-AuthN`, `InputValidation`, `RateLimiting`, `API-Contract/Errors`, `Privacy/PII`

## US-002 — Вход в систему (Login)

* **Роль–Цель–Ценность:** Как пользователь, я хочу входить в систему по логину/паролю, чтобы управлять своими данными.
* **Кратко:** Вход, выдача JWT/сессии, защита от перебора.
* **API/Endpoints:** `POST /api/auth/login`
* **NFR hooks:** `Security-AuthN`, `RateLimiting`, `Timeouts/Retry`, `Observability/Logging`

## US-003 — Сброс пароля (Password Reset)

* **Роль–Цель–Ценность:** Как пользователь, я хочу сбросить пароль по ссылке, чтобы восстановить доступ.
* **Кратко:** Генерация токена, письмо, установка нового пароля.
* **API/Endpoints:** `POST /api/auth/forgot`, `POST /api/auth/reset`
* **NFR hooks:** `Security-Secrets`, `Privacy/PII`, `API-Contract/Errors`, `Auditability`

## US-004 — Редактирование профиля

* **Роль–Цель–Ценность:** Как пользователь, я хочу редактировать профиль, чтобы поддерживать актуальные данные.
* **Кратко:** Изменение имени, контактов; маскирование PII в логах.
* **API/Endpoints:** `GET/PUT /api/profile`
* **NFR hooks:** `Privacy/PII`, `Data-Integrity`, `Security-AuthZ/RBAC`, `Observability/Logging`

## US-005 — Загрузка аватара/файла

* **Роль–Цель–Ценность:** Как пользователь, я хочу загрузить аватар, чтобы персонализировать профиль.
* **Кратко:** Проверка типа/размера, хранение под UUID.
* **API/Endpoints:** `POST /api/files/avatar`
* **NFR hooks:** `InputValidation`, `RateLimiting`, `Storage-Quotas`, `API-Contract/Errors`

## US-006 — Просмотр списка и пагинация

* **Роль–Цель–Ценность:** Как пользователь, я хочу видеть список элементов с пагинацией, чтобы быстро находить нужное.
* **Кратко:** Сортировка/фильтры, стабильная пагинация.
* **API/Endpoints:** `GET /api/items?limit=&offset=`
* **NFR hooks:** `Performance`, `Scalability`, `API-Contract/Errors`

## US-007 — Поиск по каталогу

* **Роль–Цель–Ценность:** Как пользователь, я хочу искать по полю/тегам, чтобы находить релевантные результаты.
* **Кратко:** Полнотекст/префикс, ограничение запроса.
* **API/Endpoints:** `GET /api/search?q=`
* **NFR hooks:** `Performance`, `RateLimiting`, `Timeouts/Retry`, `Observability/Logging`

## US-008 — Оформление заказа/покупки

* **Роль–Цель–Ценность:** Как покупатель, я хочу оформить заказ и оплатить его, чтобы получить товар/услугу.
* **Кратко:** Корзина → заказ → платёжная сессия → вебхук статуса.
* **API/Endpoints:** `POST /api/orders`, `POST /api/payments/session`, `POST /api/payments/webhook`
* **NFR hooks:** `Security-AuthZ/RBAC`, `Data-Integrity`, `Auditability`, `Timeouts/Retry`, `Idempotency`

## US-009 — 2FA / подтверждение устройства

* **Роль–Цель–Ценность:** Как пользователь, я хочу включить 2FA, чтобы повысить безопасность аккаунта.
* **Кратко:** TOTP/SMS/e-mail код, доверенные устройства.
* **API/Endpoints:** `POST /api/auth/2fa/setup`, `POST /api/auth/2fa/verify`
* **NFR hooks:** `Security-AuthN`, `Secrets-Management`, `RateLimiting`

## US-010 — Управление API-токенами

* **Роль–Цель–Ценность:** Как разработчик, я хочу выпускать и отзывать персональные API-токены, чтобы интегрироваться с сервисом.
* **Кратко:** Список токенов, маскирование, отзыв.
* **API/Endpoints:** `GET/POST/DELETE /api/tokens`
* **NFR hooks:** `Security-Secrets`, `Auditability`, `RateLimiting`, `API-Contract/Errors`

## US-011 — Роли и права (RBAC, tenant isolation)

* **Роль–Цель–Ценность:** Как администратор организации, я хочу назначать роли/права, чтобы сотрудники видели только данные своего тенанта.
* **Кратко:** Роли: admin, manager, viewer; изоляция данных.
* **API/Endpoints:** `POST /api/org/users/{id}/role`, `GET /api/org/resources`
* **NFR hooks:** `Security-AuthZ/RBAC`, `Privacy/PII`, `Auditability`

## US-012 — Уведомления (email/webhook)

* **Роль–Цель–Ценность:** Как пользователь, я хочу получать уведомления о важных событиях, чтобы не пропускать изменения.
* **Кратко:** Настройки каналов, отправка, ретраи вебхуков.
* **API/Endpoints:** `POST /api/notifications/send`, `POST /api/webhooks/{id}`
* **NFR hooks:** `Timeouts/Retry`, `Observability/Logging`, `RateLimiting`

## US-013 — Импорт CSV

* **Роль–Цель–Ценность:** Как менеджер, я хочу загружать CSV с данными, чтобы быстро обновлять систему.
* **Кратко:** Валидация схемы/размера, фоновые задания.
* **API/Endpoints:** `POST /api/import/csv`
* **NFR hooks:** `InputValidation`, `Data-Integrity`, `Scalability`, `Observability/Logging`

## US-014 — Экспорт данных (CSV/JSON)

* **Роль–Цель–Ценность:** Как пользователь, я хочу экспортировать свои данные, чтобы анализировать их вне системы.
* **Кратко:** Экспорт по фильтрам, контроль приватности, агрегации.
* **API/Endpoints:** `GET /api/export?format=csv|json`
* **NFR hooks:** `Privacy/PII`, `RateLimiting`, `Performance`, `API-Contract/Errors`

## US-015 — История действий (Audit Log)

* **Роль–Цель–Ценность:** Как аудитор, я хочу видеть журнал действий, чтобы разбирать инциденты и соответствие политикам.
* **Кратко:** Неподменный лог, фильтры по актору/объекту/дате.
* **API/Endpoints:** `GET /api/audit?actor=&from=&to=`
* **NFR hooks:** `Auditability`, `Observability/Logging`, `Security-AuthZ/RBAC`

## US-016 — Удаление аккаунта (Right to Erasure)

* **Роль–Цель–Ценность:** Как пользователь, я хочу удалить аккаунт, чтобы прекратить обработку моих данных.
* **Кратко:** Запрос удаления, окна удержания, маскирование/анонимизация.
* **API/Endpoints:** `DELETE /api/account`
* **NFR hooks:** `Privacy/PII`, `Data-Integrity`, `Auditability`

## US-017 — Массовые операции (Bulk Update)

* **Роль–Цель–Ценность:** Как менеджер, я хочу массово обновлять записи по фильтру, чтобы ускорить операционные задачи.
* **Кратко:** Планирование job, idempotency, лимиты.
* **API/Endpoints:** `POST /api/bulk/update`
* **NFR hooks:** `Idempotency`, `RateLimiting`, `Timeouts/Retry`, `Observability/Logging`

## US-018 — Интеграция с внешним API

* **Роль–Цель–Ценность:** Как система, я хочу запрашивать данные у внешнего API, чтобы обогащать информацию.
* **Кратко:** Пулы соединений, таймауты, кэш, деградации.
* **API/Endpoints:** `GET /api/integrations/vendor-x/*`
* **NFR hooks:** `Timeouts/Retry`, `CircuitBreaker`, `Performance`, `Observability/Logging`

## US-019 — Загрузка больших файлов (chunked/multipart)

* **Роль–Цель–Ценность:** Как пользователь, я хочу загружать большие файлы по частям, чтобы не терять прогресс при сбоях.
* **Кратко:** Чанк-аплоад, контроль целостности, ограничение типов.
* **API/Endpoints:** `POST /api/uploads/init`, `PUT /api/uploads/{id}/chunk`, `POST /api/uploads/{id}/complete`
* **NFR hooks:** `InputValidation`, `RateLimiting`, `Data-Integrity`, `Performance`

## US-020 — Управление тарифом/планом (Billing)

* **Роль–Цель–Ценность:** Как пользователь, я хочу менять тарифный план, чтобы оптимизировать стоимость.
* **Кратко:** Переключение плана, пропорциональное списание, лимиты фич.
* **API/Endpoints:** `GET/POST /api/billing/plan`
* **NFR hooks:** `Security-AuthZ/RBAC`, `Data-Integrity`, `Auditability`, `Observability/Logging`

---

### Подсказки по адаптации

* Уточняйте **контекст** (доменные термины, конкретные эндпойнты).
* Думайте про **измеримость**: время (мс), частота (RPS), размер (MiB), коды ответа, CVSS, лимиты.
* Сразу держите в голове будущие **Evidence** (test/log/scan/policy) и **Trace** (issue/link) — они понадобятся в S03-реестре и позже в `TM.md`.

# Миграция SMM Studio в платформу YANGRAI

**Дата:** 18.11.2026  
**Статус:** план реализации  
**Решение:** интегрировать существующий Node-продукт, не переписывать на .NET ([D-07](2026-11-18-решения-по-биллингу.md))  
**Связанные документы:** [архитектура](2026-11-18-архитектура-YANGRAI.md), [решения по биллингу](2026-11-18-решения-по-биллингу.md), [аудит Identity/Billing](2026-11-15-аудит-Identity-Billing-план.md)

---

## 1. Цель и рамки

**Цель:** SMM Studio становится первым продуктом платформы: один аккаунт, организация, роли, подписка, лимиты. Продукт работает за BFF, данные изолированы по `org_id`, оплата управляется Core.

**В рамках:**

- `org_id` в `projects`, `channels`, `posts`;
- аутентификация через BFF + OIDC;
- изоляция тенантов во всех запросах, планировщике и подключении каналов;
- entitlements и paywall;
- usage-события;
- шифрование токенов каналов;
- тесты изоляции и регрессии.

**Вне рамок:**

- переписывание на .NET;
- аналитика, медиатека, Instagram/MAX;
- командные роли внутри продукта (кроме org-ролей);
- биллинг-UI (он в Core/кабинете);
- мультирегион.

**Оценка:** 2–3 недели, 1 Node-разработчик + QA. Зависимости: Core API entitlements, BFF, Keycloak.

---

## 2. Текущее состояние

| Компонент | Факт | Проблема для платформы |
|---|---|---|
| БД | 3 таблицы: `projects`, `channels`, `posts` (`server/schema.sql`) | Нет `org_id`, нет пользователей |
| Авторизация | Нет | Любой запрос видит всё |
| CORS | `app.use(cors())` — `Access-Control-Allow-Origin: *` (`server/index.js:21`) | Cookie-сессия с `*` не работает; небезопасно |
| Каналы | `platform text not null unique` (`server/schema.sql:14`) | Один Telegram/VK на всю платформу, а не на организацию |
| Подключение Telegram | Сессия в памяти без `userId`/`orgId` (`server/connect.js:34`) | Нельзя привязать канал к организации |
| Подключение Telegram | `on conflict (platform)` (`server/connect.js:64-74`) | Перезаписывает канал другой организации |
| VK OAuth | `pending` в памяти, state без `orgId` (`server/oauth-vk.js:32`) | Канал привязывается к «нику», а не к тенанту |
| VK OAuth | `on conflict (platform)` (`server/oauth-vk.js:91-102`) | То же |
| Планировщик | Выбирает все `scheduled` посты (`server/scheduler.js:10-12`) | Публикует без org-контекста и без проверки лимита |
| Публикация | Каналы по `id = any(...)` без org (`server/publish.js:6-8`) | IDOR: можно опубликовать в чужой канал |
| Схема | Применяется `schema.sql` при старте (`server/db.js:24-29`) | Нет миграций `ALTER` |
| Сессии подключения | In-memory (`server/sessions.js`) | При нескольких инстансах ломается |
| Токены каналов | Открытый текст в `channels.creds` | 152-ФЗ, компрометация |
| Фронт | `fetch` без `credentials` (`web/src/api.js`) | Cookie-сессия не отправляется; нет 401/402 |
| Медиа | `/media` — публичная статика (`server/index.js:23`) | Картинки доступны по ссылке без проверки org |

---

## 3. Целевая схема

```
┌──────────────────────────┐        ┌──────────────────────────┐
│ Браузер                  │        │ Keycloak                 │
│ app.yangrai.ru           │───────►│ OIDC                     │
└────────────┬─────────────┘        └──────────────────────────┘
             │ cookie
┌────────────▼─────────────┐        ┌──────────────────────────┐
│ BFF (ASP.NET Core)       │───────►│ YANGRAI Core             │
│ cookie-сессия, OIDC      │        │ orgs, billing,           │
│ X-User-Id, X-Org-Id      │        │ entitlements, usage      │
└────────────┬─────────────┘        └──────────────────────────┘
             │ private network
┌────────────▼─────────────┐
│ SMM Studio (Node)        │
│ projects/channels/posts  │
│ + org_id                 │
└──────────────────────────┘
```

Ключевые решения:

1. **BFF — единственная точка входа.** Продукт не публикуется в интернет напрямую; BFF передаёт `X-User-Id`, `X-Org-Id`, `X-Request-Id` и удаляет клиентские значения этих заголовков.
2. **`org_id` — во всех таблицах продукта.** Ни один запрос не выполняется без `org_id`.
3. **Каналы — per organization.** `unique(org_id, platform)`.
4. **Entitlements — через Core API** с кэшем 30–60 с.
5. **Планировщик проверяет лимит** перед публикацией, а не только UI.
6. **Токены каналов шифруются** AES-256-GCM, ключ — в секретах.

---

## 4. Этапы

### Этап 0. Подготовка (1–2 дня)

- Бэкап БД и медиа; проверить восстановление.
- Staging с копией прод-данных (обезличить).
- Зафиксировать baseline тестов: `scripts/smoke.mjs`, `scripts/api-test.mjs`, `scripts/scheduler-test.mjs`.
- Feature flag `AUTH_ENABLED` и `ENTITLEMENTS_ENABLED`.
- Создать первую организацию в Core, получить `DEFAULT_ORG_ID`.

**Готово, когда:** staging поднят, бэкап восстанавливается, тесты зелёные.

### Этап 1. Схема и миграции (2–3 дня)

- Добавить `db/migrations/` и раннер с таблицей `schema_migrations`.
- Остановить применение `schema.sql` на каждом старте (оставить только для чистой БД).
- Добавить `org_id` nullable, backfill, затем `not null`.
- Заменить `unique(platform)` на `unique(org_id, platform)`.
- Индексы: `posts(org_id, status, scheduled_at)`, `channels(org_id)`, `projects(org_id)`.
- Таблица `usage_outbox` (если продукт не пишет напрямую в Core).

**Готово, когда:** миграция применяется на копии прода, откатывается, данные на месте.

### Этап 2. Auth и BFF (3–4 дня)

- BFF: OIDC code+PKCE, cookie-сессия, `/bff/session`, `/bff/login`, `/bff/logout`.
- BFF проксирует `/api/*` в продукт, добавляет `X-User-Id`, `X-Org-Id`, `X-Request-Id`.
- Продукт: middleware `requireUser`, `requireOrg`; проверка, что заголовки пришли от BFF (private network / shared secret).
- Убрать `app.use(cors())`; same-origin через BFF. Если продукт на отдельном домене — allowlist точного origin + `credentials: true`.
- Фронт: `fetch` с `credentials: 'include'`; обработка 401 → редирект на BFF login; 403 → «нет доступа».
- Убрать демо-доступ к `/api/state` без сессии.

**Готово, когда:** без сессии `/api/state` → 401; с сессией — только данные своей org.

### Этап 3. Изоляция тенантов (3–4 дня)

- Все запросы `projects`, `channels`, `posts`, `generate`, `publish` — с `org_id`.
- `loadProject`, `loadPost` — принимают `orgId`; 404 для чужих.
- `/api/state` — фильтр по `org_id`.
- `connect.js`: `startTelegramConnect({ userId, orgId })`, сессия хранит `orgId`; `saveChannel(..., orgId)`; `on conflict (org_id, platform)`.
- `oauth-vk.js`: `state`/`pending` хранит `orgId`; `on conflict (org_id, platform)`.
- `scheduler.js`: выборка с `org_id`; для каждого поста — проверка entitlement; при превышении — `status='blocked'`, причина в `results`, без публикации.
- `publish.js`: каналы только `where id = any(...) and org_id = $org`; пост — тоже.
- Медиа: закрыть `/media` авторизованным маршрутом с проверкой org (или подписанные URL).

**Готово, когда:** автотест: две организации, пользователь A не видит и не может изменить/опубликовать данные B.

### Этап 4. Entitlements и paywall (2–3 дня)

- Клиент Core: `GET /internal/v1/orgs/{orgId}/entitlements`, service token, кэш 30–60 с, ETag.
- Проверки: `smm.projects.max` при создании проекта; `smm.channels.max` при подключении; `smm.posts.month` при планировании/публикации; `smm.ai_text.month` / `smm.ai_images.month` при генерации.
- 402 с `code`, `feature`, `limit`, `used`, `upgrade_url`.
- Фронт: paywall-модалка, баннер лимита, кнопка «Сменить тариф» → кабинет.
- При недоступности Core: кэш; для оплаченных — grace 24 ч; для новых платных действий — fail-closed.

**Готово, когда:** на исчерпанном лимите API возвращает 402, публикация не уходит, UI показывает paywall.

### Этап 5. Usage и шифрование (2–3 дня)

- `POST /internal/v1/orgs/{orgId}/usage-events` при успешной публикации и генерации; idempotency key.
- Локальный `usage_outbox` на случай недоступности Core; ретраи.
- Шифрование `channels.creds`: AES-256-GCM, ключ в секретах (Lockbox/Vault), версия ключа в записи; миграция существующих токенов.
- Маскирование `creds` в API (уже есть `stripChannel`, сохранить).

**Готово, когда:** usage не теряется при падении Core; старые токены перешифрованы; в API нет открытых токенов.

### Этап 6. Тесты и выкат (2–3 дня)

- `scripts/isolation-test.mjs` — две org, cross-tenant 404/403.
- `scripts/auth-test.mjs` — 401/403/402.
- `scripts/paywall-test.mjs` — лимиты постов, проектов, каналов, генераций.
- Обновить `smoke.mjs`, `api-test.mjs`, `scheduler-test.mjs` — авторизация и org.
- Playwright: вход, переключение org, создание проекта, подключение канала, публикация, paywall.
- Canary: 10% трафика / один тестовый аккаунт; затем 100%.
- Мониторинг: 401/402/403, ошибки публикаций, лаг планировщика.

**Готово, когда:** все тесты зелёные, canary 24 часа без инцидентов, rollback проверен.

---

## 5. Изменения по файлам

| Файл | Что менять |
|---|---|
| `server/schema.sql` | Оставить для чистой БД; добавить `org_id`, `unique(org_id, platform)` |
| `server/db.js` | Раннер миграций, `schema_migrations`, убрать применение схемы на каждом старте |
| `server/index.js` | Убрать `cors()`; middleware auth/org; `org_id` во всех запросах; 401/402; медиа через auth; проверки лимитов |
| `server/connect.js` | `startTelegramConnect({ userId, orgId })`; `saveChannel(..., orgId)`; `on conflict (org_id, platform)`; сессия хранит `orgId` |
| `server/oauth-vk.js` | `state` и `pending` хранят `orgId`; `on conflict (org_id, platform)`; привязка к org |
| `server/scheduler.js` | Выборка с `org_id`; проверка entitlement; `blocked` вместо публикации при лимите |
| `server/publish.js` | Каналы и пост — с `org_id`; 404 при чужом канале |
| `server/sessions.js` | Добавить `orgId`, `userId`; для multi-instance — Redis (позже) |
| `server/text.js`, `image.js` | Проверка лимита до вызова провайдера; usage после успеха |
| `web/src/api.js` | `credentials: 'include'`; обработка 401/402/403; `X-Request-Id` |
| `web/src/App.jsx` | Guard, paywall, переключатель org (если встроен в кабинет) |
| `scripts/*.mjs` | Авторизация, org, тесты изоляции и лимитов |

---

## 6. SQL-миграция

```sql
-- 001_org.sql
begin;

alter table projects add column org_id uuid;
alter table channels add column org_id uuid;
alter table posts    add column org_id uuid;

-- backfill: одна организация на все существующие данные
update projects set org_id = :'default_org' where org_id is null;
update channels set org_id = :'default_org' where org_id is null;
update posts    set org_id = :'default_org' where org_id is null;

alter table projects alter column org_id set not null;
alter table channels alter column org_id set not null;
alter table posts    alter column org_id set not null;

-- было: platform text not null unique
alter table channels drop constraint if exists channels_platform_key;
create unique index channels_org_platform_uq on channels (org_id, platform);

create index projects_org_idx on projects (org_id, created_at desc);
create index channels_org_idx on channels (org_id);
create index posts_org_due_idx on posts (org_id, status, scheduled_at);
create index posts_org_created_idx on posts (org_id, created_at desc);

commit;
```

Примечания:

- `:'default_org'` — UUID первой организации из Core; передаётся в миграцию, не хардкодится.
- Backfill выполнять в транзакции при остановленном продукте или в два шага (nullable → backfill → not null) без долгого lock.
- Старый unique удалять только после проверки, что все каналы имеют `org_id`.
- Для больших таблиц — `CREATE INDEX CONCURRENTLY` вне транзакции.

---

## 7. Контракт с Core

### 7.1. Entitlements

```http
GET /internal/v1/orgs/{orgId}/entitlements
Authorization: Bearer <service-token>
X-Request-Id: ...
```

```json
{
  "org_id": "…",
  "etag": "v42",
  "valid_until": "2026-12-01T00:00:00Z",
  "features": {
    "smm.projects.max": { "enabled": true, "limit": 3 },
    "smm.channels.max": { "enabled": true, "limit": 10 },
    "smm.posts.month": { "enabled": true, "limit": 150, "used": 42 },
    "smm.ai_text.month": { "enabled": true, "limit": 500, "used": 120 },
    "smm.ai_images.month": { "enabled": true, "limit": 100, "used": 7 },
    "smm.seats.max": { "enabled": true, "limit": 3 }
  }
}
```

Правила:

- Кэш 30–60 с; `If-None-Match` по ETag.
- При 5xx: последний известный ответ; для оплаченных — grace 24 ч; новые платные действия — fail-closed.
- Лимит `smm.posts.month` проверяется при планировании и публикации; `used` инкрементируется после успеха.

### 7.2. Usage

```http
POST /internal/v1/orgs/{orgId}/usage-events
Authorization: Bearer <service-token>
Content-Type: application/json

{
  "feature": "smm.posts",
  "quantity": 1,
  "idempotency_key": "post:<post-id>",
  "occurred_at": "2026-11-18T12:00:00Z",
  "meta": { "channels": 2, "platforms": ["telegram", "vk"] }
}
```

- 202 Accepted; продукт не ждёт.
- При недоступности — `usage_outbox` и ретраи.
- Core дедуплицирует по `idempotency_key`.

---

## 8. Тесты и критерии готовности

| Тест | Что проверяет |
|---|---|
| `isolation-test.mjs` | A не видит проекты/каналы/посты B; публикация в чужой канал → 404 |
| `auth-test.mjs` | Без сессии 401; чужая org 403 |
| `paywall-test.mjs` | 402 на лимитах проектов, каналов, постов, генераций |
| `scheduler-test.mjs` | Пост с исчерпанным лимитом не публикуется, статус `blocked` |
| `smoke.mjs` | Полный сценарий: вход, проект, канал, генерация, публикация |
| Playwright | UI: login → org → проект → paywall → оплата в кабинете |

Критерии:

1. Ни один API-запрос не выполняется без `org_id`.
2. Публикация в чужой канал невозможна.
3. Лимит проверяется на сервере, не только в UI.
4. Токены каналов не отдаются в API и зашифрованы в БД.
5. 401/402/403 обрабатываются фронтом.
6. Планировщик не публикует посты организаций с истёкшей подпиской.

---

## 9. Риски и откат

| Риск | Мера |
|---|---|
| Backfill испортит данные | Бэкап, staging-прогон, транзакция, проверка counts |
| Auth «запрёт» пользователей | Feature flag, тестовый аккаунт, canary, rollback деплоя |
| Планировщик начнёт дублировать публикации | Идемпотентность по `post_id`, блокировка `FOR UPDATE SKIP LOCKED` |
| Core недоступен | Кэш entitlements, grace 24 ч, `usage_outbox` |
| Медиа останется публичным | Закрыть `/media` auth-маршрутом или подписанными URL |
| Несколько инстансов ломают сессии | Redis для `sessions`, либо один инстанс на MVP |
| VK OAuth state подделан | Привязка state к `userId`/`orgId`, TTL, one-time use |

Откат:

- Миграции аддитивные; `org_id` можно оставить, старый unique вернуть при необходимости.
- Feature flag `AUTH_ENABLED=false` — только для аварийного доступа через VPN, не для прода.
- Деплой — blue-green; предыдущая версия готова.

---

## 10. Оценка

| Этап | Дни |
|---|---|
| 0. Подготовка | 1–2 |
| 1. Схема и миграции | 2–3 |
| 2. Auth и BFF | 3–4 |
| 3. Изоляция тенантов | 3–4 |
| 4. Entitlements и paywall | 2–3 |
| 5. Usage и шифрование | 2–3 |
| 6. Тесты и выкат | 2–3 |
| **Итого** | **15–22 рабочих дня** |

С буфером — 3 недели. Зависимости: Core API, BFF, Keycloak, первая организация.

После миграции продукт готов к продажам по тарифам «Старт/Про/Агентство». Переписывание на .NET — отдельный проект после первых платящих, только если появится сильная причина.

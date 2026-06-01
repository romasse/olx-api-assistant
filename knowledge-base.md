# OLX Partner API — Knowledge Base

> **Призначення:** knowledge base для AI-асистента з інтеграції OLX Partner API v2.0.  
> **Джерела:** офіційна OpenAPI специфікація `partner_api.yaml`, портал розробників `https://developer.olx.ua/api/doc`  
> **Версія API:** 2.0 | **Базовий URL (UA):** `https://www.olx.ua/api/partner` | **Дата:** травень 2026

---

## §1 — Швидкий старт

### §1.1 — Обов'язкові заголовки кожного запиту

```
Authorization: Bearer <access_token>
Version: 2.0
Accept-Language: uk
Content-Type: application/json
```

Відсутність `Version` → 400 `Missing required 'Version' header!`

### §1.2 — Базові поняття

| Термін | Значення |
|---|---|
| `client_id`, `client_secret` | Креденшіали додатку, видаються OLX вручну |
| `access_token` | Живе **86400 сек (24 год)** |
| `refresh_token` | Живе **2592000 сек (30 днів)** |
| `code` | Одноразовий код авторизації, живе **10 хвилин** |
| `scope` | Стандартно `v2 read write` |
| `external_id` | Ваш внутрішній ID у CRM — ключ для ідемпотентності |
| `external_url` | В Україні **НЕДОСТУПНЕ** (тільки PL/Jobs) |

---

## §2 — Автентифікація (OAuth 2.0)

### §2.1 — Три типи grant types

| Grant type | Контекст | Що дозволяє |
|---|---|---|
| `authorization_code` | Від імені користувача | Постити оголошення, читати повідомлення |
| `client_credentials` | Тільки додаток | Тільки читання довідників |
| `refresh_token` | Будь-який | Оновлення `access_token` |

### §2.2 — Повний OAuth-потік для SaaS

**Крок 1.** Redirect на авторизацію:
```
https://www.olx.ua/oauth/authorize/
  ?client_id=<your_client_id>
  &response_type=code
  &state=<random_hash>
  &scope=read+write+v2
  &redirect_uri=https://yourapp.com/auth/olx/callback
```

**Крок 4.** Обмін `code` → токени:
```http
POST https://www.olx.ua/api/open/oauth/token
Content-Type: application/json

{
  "grant_type": "authorization_code",
  "client_id": "<your_client_id>",
  "client_secret": "<your_client_secret>",
  "code": "<code_from_callback>",
  "scope": "v2 read write",
  "redirect_uri": "https://yourapp.com/auth/olx/callback"
}
```

Відповідь: `access_token`, `expires_in: 86400`, `refresh_token`

### §2.3 — Оновлення токена

```http
POST https://www.olx.ua/api/open/oauth/token

{
  "grant_type": "refresh_token",
  "client_id": "<your_client_id>",
  "client_secret": "<your_client_secret>",
  "refresh_token": "<saved_refresh_token>"
}
```

У відповіді — **нові** обидва токени. Обов'язково оновити в БД.

### §2.4 — Client credentials (для довідників)

```http
POST https://www.olx.ua/api/open/oauth/token

{
  "grant_type": "client_credentials",
  "client_id": "<your_client_id>",
  "client_secret": "<your_client_secret>",
  "scope": "v2 read"
}
```

Цей токен **не має прив'язки до користувача**. На `/users/me` або `/adverts` → `Invalid owner in token`.

---

## §3 — Каталог ендпоінтів

Базовий URL: `https://www.olx.ua/api/partner`

### §3.1 — Користувачі

| Метод | Шлях | Scope |
|---|---|---|
| GET | `/users/me` | read |
| GET | `/users/me/account-balance` | read |
| GET | `/users/me/payment-methods` | read |
| GET | `/users-business/me` | read |

### §3.2 — Географія

| Метод | Шлях |
|---|---|
| GET | `/regions` |
| GET | `/cities` (params: `region_id`, `query`, `limit`) |
| GET | `/cities/{cityId}/districts` |
| GET | `/locations?latitude=&longitude=` |

> Для великих міст (Київ, Дніпро, Харків, Одеса, Львів) `district_id` — **обов'язковий**.

### §3.3 — Довідники

| Метод | Шлях |
|---|---|
| GET | `/categories` |
| GET | `/categories/{id}` (поле `is_leaf`, `photos_limit`) |
| GET | `/categories/{id}/attributes` |
| GET | `/categories/suggestion?title=` |
| GET | `/currencies` |
| GET | `/languages` |

### §3.4 — Оголошення

| Метод | Шлях | Scope |
|---|---|---|
| GET | `/adverts` (params: `external_id`, `category_ids`, `offset`, `limit`) | read |
| POST | `/adverts` | write |
| PUT | `/adverts/{id}` | write |
| DELETE | `/adverts/{id}` (тільки якщо не active; throttling cost = 5) | write |
| POST | `/adverts/{id}/commands` | write |
| GET | `/adverts/{id}/statistics` | read |
| GET | `/adverts/{id}/moderation-reason` | read |

### §3.5 — Повідомлення (чат)

| Метод | Шлях |
|---|---|
| GET | `/threads` (params: `advert_id`, `interlocutor_id`, `offset`, `limit`) |
| GET | `/threads/{id}/messages` |
| POST | `/threads/{id}/messages` |
| POST | `/threads/{id}/commands` |

> ⚠️ **Вебхуків немає.** Потрібен поллінг `/threads`.

### §3.6 — Пакети та платні функції

| Метод | Шлях |
|---|---|
| GET | `/packets?category_id=` |
| POST | `/users/me/packets` |
| POST | `/adverts/{id}/packets` |
| GET | `/paid-features` |
| POST | `/adverts/{id}/paid-features` |
| GET | `/users/me/packets` |

---

## §4 — Lifecycle оголошення

### §4.1 — Статуси

```
new        → створено, очікує модерації
active     → опубліковано, видиме всім
limited    → НЕ опубліковано, потрібен пакет
removed_by_user → видалено
```

### §4.2 — Сценарій 1: без пакету

```
POST /adverts → status: "new" → (модерація) → status: "active" ✓
```

### §4.3 — Сценарій 2: категорія з лімітом

```
POST /adverts → status: "limited"
  ↓
GET /packets?category_id=<cat>
POST /users/me/packets
POST /adverts/{id}/packets
POST /adverts/{id}/commands { "command": "activate" }
  ↓ (модерація)
status: "active" ✓
```

### §4.4 — Деактивація

```http
POST /adverts/{id}/commands
{ "command": "deactivate", "is_success": true }
```

`is_success: true` — продано; `false` — інші причини.

### §4.5 — Видалення (двоетапне)

```
POST /adverts/{id}/commands { "command": "deactivate", "is_success": false }
DELETE /adverts/{id}
```

DELETE на active → 400 `Invalid status`.

### §4.6 — Команди

| Команда | Обмеження |
|---|---|
| `activate` | Потрібен пакет, якщо категорія лімітована |
| `deactivate` | Поле `is_success` обов'язкове |
| `finish` | — |
| `extend` | **В Україні НЕДОСТУПНА** |
| refresh (бамп) | Не частіше ніж раз на **14 днів** |

---

## §5 — Правила контенту OLX

### §5.1 — Заголовок і опис

- Не більше **50% символів у верхньому регістрі**
- Заборонені email, www, телефонні номери в тексті
- Не більше **2 підряд** однакових знаків пунктуації

### §5.2 — Категорія

- `category_id` має бути **листовою** (`is_leaf: true`)

### §5.3 — Атрибути

```json
"attributes": [
  { "code": "rooms", "value": "2" },
  { "code": "total_area", "value": "65" }
]
```

Тільки `code` + `value` або `values`. Зайві ключі → `This value is not valid: attributes`.

### §5.4 — Локація

- `district_id` — обов'язковий для великих міст
- `latitude`/`longitude` мають відповідати `district_id`

### §5.5 — Фото

- Ліміт: поле `photos_limit` у `GET /categories/{id}`
- URL має бути публічним (OLX скачує фото до себе)

### §5.6 — Контакти

- `phone`: формат `+380XXXXXXXXX`

### §5.7 — Ціна

```json
"price": {
  "value": 50000,
  "currency": "USD",
  "negotiable": true,
  "trade": false,
  "budget": false
}
```

---

## §6 — Шаблон POST /adverts для нерухомості

```http
POST https://www.olx.ua/api/partner/adverts
Authorization: Bearer <access_token>
Version: 2.0
Content-Type: application/json

{
  "title": "2-кімнатна квартира на Перемозі, 65 м²",
  "description": "Простора 2-кімнатна квартира з ремонтом, окрема кухня, балкон засклений.",
  "category_id": 1147,
  "advertiser_type": "business",
  "external_id": "R-2026-0531",
  "contact": {
    "name": "Роман",
    "phone": "+380501234567"
  },
  "location": {
    "city_id": 9,
    "district_id": 145,
    "latitude": 48.4647,
    "longitude": 35.0462
  },
  "images": [
    { "url": "https://cdn.yourcrm.com/photos/R-2026-0531/01.jpg" },
    { "url": "https://cdn.yourcrm.com/photos/R-2026-0531/02.jpg" }
  ],
  "price": {
    "value": 58000,
    "currency": "USD",
    "negotiable": true,
    "trade": false,
    "budget": false
  },
  "attributes": [
    { "code": "rooms", "value": "2" },
    { "code": "total_area", "value": "65" },
    { "code": "floor", "value": "3" },
    { "code": "floors_in_building", "value": "9" }
  ],
  "auto_extend_enabled": true
}
```

> ⚠️ `category_id`, `district_id`, коди атрибутів — приклади. Реальні значення через `GET /categories`, `/districts`, `/attributes`.

---

## §7 — Каталог помилок

### §7.1 — HTTP-коди

| Код | Значення |
|---|---|
| 400 | Валідація не пройшла (дивитися `validation` масив) |
| 401 | Невалідний/відсутній токен або scope |
| 403 | Виконання заборонене |
| 404 | Ресурс не знайдено |
| 429 | Too Many Requests — throttling |

### §7.2 — Помилки авторизації

| Помилка | Причина | Рішення |
|---|---|---|
| `Authorization code doesn't exist or is invalid` | code протух (>10 хв) або вже використаний | Restart OAuth |
| `Invalid refresh token` | refresh_token >30 днів | Re-auth користувача |
| `Insufficient scope` | Не вистачає привілеїв | Запросити `v2 read write` |
| `Invalid owner in token` | `client_credentials` замість `authorization_code` | Змінити grant type |
| `The access token provided is invalid` | Токен невалідний | Refresh через `refresh_token` |
| `Client is not active` | Невірний `client_id` | Перевірити + написати в OLX |

### §7.3 — Помилки публікації

| Помилка | Рішення |
|---|---|
| `Fix the category` | Категорія не листова — потрібна `is_leaf: true` |
| `Your coordinates are too far from picked location` | Перевірити `GET /locations?latitude=&longitude=` |
| `Too many capital letters` | >50% капсу в заголовку/описі |
| `Field is not valid. Phone numbers are not allowed` | Видалити телефон з тексту |
| `Field is not valid. Emails and www addresses are not allowed` | Видалити email/www |
| `Field contains to much punctuation` | Не більше 2 однакових знаків підряд |
| `This value is not valid: attributes` | Тільки `code` + `value`/`values` |
| `Image error: Remote file not exists` | URL фото недоступний |
| `Image error: Image limit exceeded` | Перевищено `photos_limit` категорії |
| `Invalid status` (delete) | Спочатку deactivate, потім DELETE |
| `You cannot refresh ad more often than once in 14 days` | Чекати 14 днів |
| `Missing required 'Version' header!` | Додати `Version: 2.0` |
| `Invalid phone format` | Формат `+380XXXXXXXXX` |

### §7.4 — Помилки оплати

| Помилка | Рішення |
|---|---|
| `No possibility to buy the packet` | Перевірити `GET /packets?category_id={id}` |
| `Not enough credits` | Поповнити баланс |
| `Payment method 'postpaid' is not activated` | Звернутись до OLX UA |

---

## §8 — Архітектура multi-tenant SaaS

### §8.1 — Зберігання токенів

Шифрування at-rest (KMS / Vault / pgcrypto). Структура таблиці:

```sql
olx_credentials (
  user_id           UUID,
  olx_user_id       INTEGER,
  access_token      TEXT,   -- зашифровано
  refresh_token     TEXT,   -- зашифровано
  access_expires_at TIMESTAMP,
  refresh_expires_at TIMESTAMP,
  scope             TEXT
)
```

### §8.2 — Refresh-воркер

Раз на годину сканувати токени з `expires_in` < 2 год. При протухлому refresh → `needs_reauth`, нотифікувати користувача.

### §8.3 — Throttling і черга

- Всі запити через чергу (Bull / Sidekiq / Celery) з retry
- На 429 — exponential backoff: 1→2→4→8→16с → fail
- DELETE = throttling cost 5; враховувати окремо

### §8.4 — Ідемпотентність через external_id

```
GET /adverts?external_id=R-2026-0531
  → якщо є advertId → PUT /adverts/{id}
  → якщо немає     → POST /adverts
```

### §8.5 — Pre-flight content validator

Перед відправкою в OLX перевірити:
- Капс >50% в заголовку/описі
- Email, www, телефони в тексті
- >2 однакових знаків пунктуації підряд
- `category_id` — листова
- `district_id` відповідає `city_id`
- Координати в межах міста
- URL фото повертають 200 OK
- Кількість фото ≤ `photos_limit`

**LLM-нормалізатор:** Claude/GPT прибирає капс, контакти, нормалізує текст ріелтора перед публікацією.

### §8.6 — Поллінг повідомлень

- Активні клієнти: кожні 1–2 хв
- Неактивні: кожні 15–30 хв
- Зберігати `last_message_id` щоб не обробляти двічі

### §8.7 — Логування

Кожен запит до OLX: request ID, user_id, метод + шлях, статус відповіді, час, (для помилок) повний body.

---

## §9 — Специфіка нерухомості (Україна)

### §9.1 — Ієрархія категорій

> Точні `category_id` — через `GET /categories`.

- Нерухомість → Квартири → **Продаж квартир** / **Довгострокова оренда** / **Подобова оренда**
- Нерухомість → Будинки → Продаж / Оренда
- Нерухомість → Комерційна нерухомість
- Нерухомість → Земля
- Нерухомість → Гаражі і паркінги

### §9.2 — Типові атрибути

| Код (приклад) | Значення |
|---|---|
| `rooms` | Кількість кімнат |
| `total_area` | Загальна площа (м²) |
| `living_area` | Житлова площа |
| `kitchen_area` | Площа кухні |
| `floor` | Поверх |
| `floors_in_building` | Поверховість |
| `building_type` | Тип будинку (цегла, панель, моноліт) |
| `condition` | Стан (євроремонт, з ремонтом, потребує ремонту) |
| `heating` | Опалення |
| `furniture` | Меблі |
| `parking` | Паркінг |

### §9.3 — Ціна для нерухомості

- Продаж → `USD`, `negotiable: true`
- Довгострокова оренда → `UAH`, місячна ціна
- Подобова → `UAH`, за добу

### §9.4 — Платні функції

- **VIP / Топ** — підняття у топ списку
- **Виділення кольором** — виділення в стрічці
- **Бамп (refresh)** — підняття на перші позиції (раз на 14 днів)

### §9.5 — Регуляторика

- При обробці персональних даних (телефони з threads) — Закон України «Про захист персональних даних»
- AI-чатбот у threads — публічно інформувати, що відповідає AI

---

## §10 — Чекліст готовності

**До першого запиту:**
- [ ] Отримано `client_id` та `client_secret` від OLX
- [ ] Зареєстровано всі `redirect_uri`
- [ ] Заголовок `Version: 2.0` у всіх запитах
- [ ] Шифрування токенів налаштовано

**До першої публікації:**
- [ ] Закешовано довідник категорій (оновлення раз на добу)
- [ ] Завантажено міста, райони, регіони
- [ ] Закешовано схеми атрибутів для потрібних категорій
- [ ] Pre-flight content validator налаштовано
- [ ] Ідемпотентність через `external_id` працює

**До продакшену:**
- [ ] Черга запитів з retry та backoff на 429
- [ ] Refresh-воркер запущено
- [ ] Сповіщення `needs_reauth` налаштовано
- [ ] Поллінг threads запущено
- [ ] Логування всіх запитів працює
- [ ] Моніторинг 4xx/5xx від OLX (Sentry/Datadog)
- [ ] Є окремий sandbox-акаунт для тестів

---

## §11 — Шаблони запитів

### Upsert оголошення по external_id

```http
GET /adverts?external_id=R-2026-0531
→ якщо є advertId → PUT /adverts/{id}  (те саме тіло, що для POST)
→ якщо немає     → POST /adverts
```

### Купити пакет і активувати

```http
GET /packets?category_id=1147
POST /users/me/packets        { "packet_id": ..., "payment_method": "... }
POST /adverts/{id}/packets    { "packet_id": ... }
POST /adverts/{id}/commands   { "command": "activate" }
```

### Відповісти в чаті

```http
GET /threads?advert_id={advertId}
GET /threads/{threadId}/messages
POST /threads/{threadId}/messages
{ "text": "Доброго дня! Дякую за звернення..." }
```

### Статистика для звіту

```http
GET /adverts?limit=100&offset=0
→ для кожного →
GET /adverts/{id}/statistics
→ поля: advert_views, phone_views, users_observing
```

---

## §12 — Глосарій

| Термін | Значення |
|---|---|
| **Advert** | Оголошення на OLX |
| **Thread** | Діалог у внутрішніх повідомленнях OLX |
| **Packet** | Пакет публікацій (передплачена кількість оголошень у категорії) |
| **Paid feature** | Платна функція просування (топ, виділення) |
| **Leaf category** | Кінцева категорія в дереві, в яку можна публікувати |
| **External ID** | Ваш внутрішній ID оголошення для ідемпотентної синхронізації |
| **Throttling cost** | «Вартість» запиту в системі обмежень OLX |
| **Scope** | Дозволи токена: `read`, `write`, `v2` |

---

## Корисні посилання

- Документація: https://developer.olx.ua/api/doc
- Postman колекція: https://app.getpostman.com/run-collection/3bf2713b8a249cdd917c
- OAuth: https://www.oauth.com/oauth2-servers/definitions/

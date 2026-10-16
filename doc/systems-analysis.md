## 1. Обзор архитектуры

Система состоит из трёх независимых микросервисов, которые общаются синхронно (REST) или асинхронно (RabbitMQ) и не разделяют базы данных.

| Сервис               | Порт | База данных        | Redis                                 | RabbitMQ  |
| -------------------- | ---- | ------------------ | ------------------------------------- | --------- |
| EventCatalog.Service | 5001 | event\_catalog\_db | кэш ответов + инициализация счётчиков | —         |
| Booking.Service      | 5002 | booking\_db        | атомарные DECRBY/INCRBY счётчиков     | publisher |
| Notification.Service | 5003 | notification\_db   | —                                     | consumer  |

### Потоки данных

**Синхронные (REST):**

- Клиент → EventCatalog.Service: просмотр каталога, управление мероприятиями
- Клиент → Booking.Service: бронирование, отмена, отчёт
- Booking.Service → EventCatalog.Service: HTTP-вызов для валидации события и получения цены категории
- Клиент → Notification.Service: просмотр истории уведомлений

**Асинхронные (RabbitMQ):**

- Booking.Service публикует `BookingCreated` / `BookingCancelled`
- Notification.Service потребляет оба типа событий и сохраняет уведомления

### Общий Redis

Оба сервиса используют один Redis-инстанс с разными namespace'ами ключей. EventCatalog инициализирует счётчики при создании/изменении мероприятия; Booking атомарно изменяет их при бронировании и отмене.

## 2. Сервисы

### EventCatalog.Service

**Ответственность:** CRUD мероприятий и категорий билетов; кэширование; инициализация Redis-счётчиков.

| Параметр | Значение |
| --- | --- |
| ORM | EF Core (code-first, миграции) |
| БД | PostgreSQL — event\_catalog\_db |
| Кэш | Redis — JSON-ответы с TTL 5 мин; инвалидация при изменении |
| Логирование | Serilog, Console + File sink, enricher: ServiceName=EventCatalog |

Внешние зависимости: Redis (чтение/запись), EventCatalog DB (чтение/запись).

---

### Booking.Service

**Ответственность:** создание и отмена броней; атомарное управление остатками; публикация событий в очередь.

| Параметр | Значение |
| --- | --- |
| ORM | Dapper (raw SQL, без абстракций) |
| БД | PostgreSQL — booking\_db |
| Кэш | Redis — только счётчики (DECRBY/INCRBY) |
| Очередь | RabbitMQ — publisher |
| Логирование | Serilog, Console + File sink, enricher: ServiceName=Booking |

Внешние зависимости: Redis (чтение/запись счётчиков), Booking DB, RabbitMQ, HTTP-вызов в EventCatalog.Service.

---

### Notification.Service

**Ответственность:** приём событий из RabbitMQ; сохранение уведомлений; предоставление истории по userId.

| Параметр | Значение |
| --- | --- |
| ORM | EF Core (code-first, миграции) |
| БД | PostgreSQL — notification\_db |
| Очередь | RabbitMQ — consumer |
| Логирование | Serilog, Console + File sink, enricher: ServiceName=Notification |

Внешние зависимости: Notification DB, RabbitMQ.

---

### Correlation ID

Все три сервиса читают заголовок `X-Correlation-ID` из входящего запроса (или генерируют новый GUID если отсутствует), добавляют его в Serilog LogContext и пробрасывают в исходящих HTTP-запросах к другим сервисам.

## 3. API контракты

Общие заголовки для всех запросов:

| Заголовок | Тип | Обязателен | Описание |
| --- | --- | --- | --- |
| `X-User-Id` | UUID | Да (все запросы Booking; POST/PUT/DELETE EventCatalog) | Идентификатор пользователя вместо JWT |
| `X-Correlation-ID` | UUID | Нет | Сквозной ID трейсинга; генерируется сервисом если отсутствует |

---

### EventCatalog.Service (порт 5001)

**GET /events** Возвращает постранично будущие мероприятия.

Query-параметры: `page` (int, default 1), `pageSize` (int, default 20, max 100), `dateFrom` (ISO 8601), `dateTo` (ISO 8601).

Response 200:

```json
{
  "items": [
    { "id": "uuid", "title": "...", "venue": "...", "date": "2026-12-01T19:00:00Z",
      "status": "Active", "minPrice": 500.00 }
  ],
  "page": 1, "pageSize": 20, "totalCount": 42
}
```

*Кэш: ключ `ec:events:list:{hash}`, TTL 5 мин. Инвалидируется при любом изменении мероприятия.*

---

**GET /events/{id}** Полные данные мероприятия с категориями.

Response 200:

```json
{
  "id": "uuid", "title": "...", "description": "...",
  "venue": "...", "date": "2026-12-01T19:00:00Z",
  "status": "Active", "ownerId": "uuid",
  "categories": [
    { "id": "uuid", "name": "Standard", "price": 500.00,
      "capacity": 200, "available": 143 }
  ]
}
```

Possible errors: 404 (не найдено). *Кэш: ключ `event:cache:{id}`, TTL 5 мин. `available` добавляется динамически из tc:counter:{categoryId} при каждом ответе; кэшируемый JSON поля available не содержит.*

---

**POST /events** Создаёт мероприятие и инициализирует Redis-счётчики. Заголовок: `X-User-Id` (становится `ownerId`).

Request body:

```json
{
  "title": "Rock Night", "description": "...",
  "venue": "СК Металлург", "date": "2026-12-01T19:00:00Z",
  "categories": [
    { "name": "Standard", "price": 500.00, "capacity": 200 },
    { "name": "VIP",      "price": 2000.00, "capacity": 50 }
  ]
}
```

Response 201 Created: объект события (как GET /events/{id}). Errors: 422 (дата в прошлом, пустые категории, capacity ≤ 0).

---

**PUT /events/{id}** Обновляет данные мероприятия. Поля `categories` обновляют существующие категории (по id внутри массива); не переданные категории не удаляются. Заголовок: `X-User-Id` (должен совпадать с `ownerId`).

Errors: 403 (не владелец), 404, 409 (попытка уменьшить capacity ниже проданных), 422 (дата в прошлом или в прошлом событие, нарушение BR-11).

---

**DELETE /events/{id}** Помечает событие как `Deleted`, удаляет Redis-ключи кэша (ec:event:{id}) и счётчиков (tc:counter:{categoryId}) для каждой категории. Заголовок: `X-User-Id`.

Response: 204 No Content. Errors: 403 (не владелец), 404, 409 (есть активные брони).

---

### Booking.Service (порт 5002)

**POST /bookings** Создаёт бронь.

Request body:

```json
{
  "eventId": "uuid",
  "ticketCategoryId": "uuid",
  "quantity": 2
}
```

Response 201:

```json
{
  "id": "uuid", "userId": "uuid",
  "eventId": "uuid", "ticketCategoryId": "uuid",
  "quantity": 2, "unitPrice": 500.00,
  "totalPrice": 1000.00, "status": "Confirmed",
  "createdAt": "2026-10-16T10:00:00Z"
}
```

Errors: 404 (событие/категория не найдены), 409 (недостаточно мест), 422 (событие в прошлом, нарушение BR-05).

---

**GET /bookings/{id}** Possible errors: 403 (чужая бронь), 404.

Response 200: объект брони (как POST /bookings).

---

**GET /bookings** Cписок броней текущего пользователя.

Query: `status` (Confirmed | Cancelled | Completed), `page`, `pageSize`. Response 200: `{ items: [...], page, pageSize, totalCount }`.

---

**DELETE /bookings/{id}** Отменяет бронь.

Response 200: обновлённый объект брони (status: Cancelled). Errors: 403 (чужая), 404, 409 (уже отменена), 422 (до события менее 24 ч).

---

**GET /bookings/report/{eventId}** Отчёт по мероприятию — только для владельца события (проверяется через HTTP-вызов в EventCatalog).

Response 200:

```json
{
  "eventId": "uuid", "eventTitle": "Rock Night",
  "categories": [
    { "categoryId": "uuid", "name": "Standard",
      "capacity": 200, "sold": 57, "available": 143,
      "revenue": 28500.00 }
  ],
  "totalRevenue": 28500.00, "totalSold": 57
}
```

*Поля `capacity` и `eventTitle` берутся из HTTP-вызова в EventCatalog (выполняется здесь же для проверки прав владельца); `available` читается из `tc:counter:{categoryId}` в Redis.*

---

### Notification.Service (порт 5003)

**GET /notifications** Query: `userId` (UUID, обязателен), `page`, `pageSize`.

Response 200:

```json
{
  "items": [
    { "id": "uuid", "bookingId": "uuid",
      "eventType": "BookingCreated",
      "message": "Бронь #... подтверждена",
      "sentAt": "2026-10-16T10:00:05Z" }
  ],
  "page": 1, "pageSize": 20, "totalCount": 3
}
```

## 4. Схема данных (PostgreSQL)

Каждый сервис использует **отдельный PostgreSQL-инстанс и отдельную БД**. Cross-database запросы запрещены. Все первичные ключи — `UUID` (генерация на стороне приложения через `Guid.NewGuid()`). Все таблицы используют именование `snake_case`.

### event\_catalog\_db

**Таблица `events`**

| Столбец | Тип | Ограничения | Описание |
| --- | --- | --- | --- |
| `id` | UUID | PK |  |
| `title` | VARCHAR(255) | NOT NULL |  |
| `description` | TEXT |  |  |
| `venue` | VARCHAR(500) | NOT NULL | Место проведения |
| `date` | TIMESTAMPTZ | NOT NULL | Дата и время начала (UTC) |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'Active' | Active / Deleted |
| `owner_id` | UUID | NOT NULL | X-User-Id создателя |
| `created_at` | TIMESTAMPTZ | DEFAULT NOW() |  |
| `updated_at` | TIMESTAMPTZ | DEFAULT NOW() | Обновлять триггером |

Индексы: `idx_events_date` ON `(date)` WHERE `status = 'Active'`; `idx_events_owner` ON `(owner_id)`.

**Таблица `ticket_categories`**

| Столбец | Тип | Ограничения | Описание |
| --- | --- | --- | --- |
| `id` | UUID | PK |  |
| `event_id` | UUID | NOT NULL, FK → events.id ON DELETE CASCADE |  |
| `name` | VARCHAR(100) | NOT NULL | Standard / VIP и т. д. |
| `price` | DECIMAL(12,2) | NOT NULL, CHECK > 0 | Текущая цена (цена продажи хранится в bookings) |
| `capacity` | INTEGER | NOT NULL, CHECK > 0 | Общее число мест |
| `created_at` | TIMESTAMPTZ | DEFAULT NOW() |  |
| `updated_at` | TIMESTAMPTZ | DEFAULT NOW() |  |

Индекс: `idx_ticket_categories_event` ON `(event_id)`.

---

### booking\_db

**Таблица `bookings`**

| Столбец | Тип | Ограничения | Описание |
| --- | --- | --- | --- |
| `id` | UUID | PK |  |
| `user_id` | UUID | NOT NULL | X-User-Id покупателя |
| `event_id` | UUID | NOT NULL | Денормализовано для отчётов |
| `ticket_category_id` | UUID | NOT NULL |  |
| `quantity` | INTEGER | NOT NULL, CHECK > 0 |  |
| `unit_price` | DECIMAL(12,2) | NOT NULL | Цена на момент бронирования |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'Confirmed' | Confirmed / Cancelled |
| `created_at` | TIMESTAMPTZ | DEFAULT NOW() |  |
| `updated_at` | TIMESTAMPTZ | DEFAULT NOW() |  |

Индексы: `idx_bookings_user` ON `(user_id)`; `idx_bookings_event` ON `(event_id)`; `idx_bookings_category` ON `(ticket_category_id)`.

Примечание: `event_id` не является FK — каждая БД изолирована. Это денормализованное поле для построения отчёта без кросс-БД запроса.

---

### notification\_db

**Таблица `notification_logs`**

| Столбец | Тип | Ограничения | Описание |
| --- | --- | --- | --- |
| `id` | UUID | PK |  |
| `user_id` | UUID | NOT NULL |  |
| `booking_id` | UUID | NOT NULL | Из сообщения RabbitMQ |
| `message_id` | UUID | NOT NULL, UNIQUE | UUID из RabbitMQ-сообщения; ключ идемпотентности |
| `event_type` | VARCHAR(50) | NOT NULL | BookingCreated / BookingCancelled |
| `message` | TEXT | NOT NULL | Сформированный текст уведомления |
| `sent_at` | TIMESTAMPTZ | DEFAULT NOW() |  |

Индекс: `idx_notification_logs_user` ON `(user_id, sent_at DESC)`; UNIQUE INDEX uq\_notification\_message ON (message\_id).

## 5. Redis

Один инстанс, два логических назначения: **кэш** (EventCatalog) и **атомарные счётчики** (EventCatalog инициализирует, Booking оперирует). Ключи изолированы префиксами.

### Кэш — EventCatalog.Service

| Ключ | Тип | TTL | Операции | Когда инвалидируется |
| --- | --- | --- | --- | --- |
| `ec:events:list:{hash}` | String (JSON) | 5 мин | GET / SET | POST, PUT, DELETE /events |
| `ec:event:{id}` | String (JSON) | 5 мин | GET / SET | PUT, DELETE /events/{id} |

`{hash}` — MD5 от строки query-параметров (`page=1&pageSize=20&dateFrom=...`).

Стратегия инвалидации: Cache-aside. При изменении события сначала пишем в PostgreSQL, затем удаляем Redis-ключ (`DEL`). Следующий запрос промажет по кэшу и прочитает из БД, после чего снова занесёт в кэш.

---

### Счётчики — совместно используются

| Ключ | Тип | TTL | Операции | Кто пишет |
| --- | --- | --- | --- | --- |
| `tc:counter:{categoryId}` | String (integer) | без TTL | SET / GET / DECRBY / INCRBY / DEL | EventCatalog (SET, DEL), Booking (DECRBY, INCRBY) |

**Инициализация** (EventCatalog при POST /events или при изменении capacity):

```
SET tc:counter:{categoryId} {capacity}  // при создании категории
// или при изменении capacity:
SET tc:counter:{categoryId} {новый_capacity - проданные}
```

**Бронирование** (Booking.Service — атомарная секция):

```
result = DECRBY tc:counter:{categoryId} {quantity}
IF result < 0:
    INCRBY tc:counter:{categoryId} {quantity}  // rollback
    → 409 Conflict
ELSE:
    → продолжить запись в PostgreSQL
```

Если ключ не существует → Booking.Service возвращает 404 (категория не найдена или не инициализирована).

**Отмена** (Booking.Service):

```
INCRBY tc:counter:{categoryId} {quantity}  // возврат мест
```

**Удаление мероприятия** (EventCatalog при DELETE):

```
// Для каждой категории события:
DEL tc:counter:{categoryId}
```

---

### Потенциальная несогласованность и восстановление

При падении Booking.Service **после** DECRBY, но **до** INSERT в PostgreSQL, счётчик будет занижен. При перезапуске сервиса рекомендуется выполнить сверку:

```
// Синхронизацию запускает EventCatalog.Service при старте:
// 1. читает все активные ticket_categories с capacity из event_catalog_db
// 2. для каждой категории вызывает Booking.Service GET /bookings/sold?categoryId={id}
//    чтобы получить суммарное количество проданных мест
// 3. пересчитывает счётчик:
foreach category in active_ticket_categories:
    sold = BookingService.GetSold(category.id)  // HTTP или прямой SQL при наличии shared-доступа
    SET tc:counter:{category.id} {category.capacity - sold}
```

## 6. RabbitMQ

### Топология

| Объект | Имя | Тип | Описание |
| --- | --- | --- | --- |
| Exchange | `ticketflow.events` | topic | Все события системы |
| Queue | `booking.notifications` | durable | Основная очередь Notification.Service |
| Queue | `booking.notifications.dlq` | durable | Dead Letter Queue |
| Binding | `booking.notifications` | routing key: `booking.#` | Все события бронирования |

DLQ-конфигурация очереди `booking.notifications`:

```
x-dead-letter-exchange: ""  (default exchange)
x-dead-letter-routing-key: "booking.notifications.dlq"
// x-message-ttl не нужен: DLQ-маршрут срабатывает при basicNack(requeue=false)
```

---

### Routing keys

| Routing key | Событие | Publisher |
| --- | --- | --- |
| `booking.created` | Бронь создана | Booking.Service |
| `booking.cancelled` | Бронь отменена | Booking.Service |

---

### Схемы сообщений

**BookingCreated**

```json
{
  "messageId": "uuid",
  "eventType": "BookingCreated",
  "occurredAt": "2026-10-16T10:00:00Z",
  "payload": {
    "bookingId": "uuid",
    "userId": "uuid",
    "eventId": "uuid",
    "eventTitle": "Rock Night",
    "ticketCategoryId": "uuid",
    "categoryName": "Standard",
    "quantity": 2,
    "unitPrice": 500.00,
    "totalPrice": 1000.00
  }
}
```

**BookingCancelled**

```json
{
  "messageId": "uuid",
  "eventType": "BookingCancelled",
  "occurredAt": "2026-10-16T11:30:00Z",
  "payload": {
    "bookingId": "uuid",
    "userId": "uuid",
    "eventId": "uuid",
    "eventTitle": "Rock Night",
    "ticketCategoryId": "uuid",
    "quantity": 2
  }
}
```

`messageId` — уникальный UUID каждого сообщения. Notification.Service сохраняет его в `notification_logs.message_id` (UNIQUE); при повторной доставке вставка отклоняется по дублирующему ключу. Проверять идемпотентность через `booking_id + event_type` ненадёжно: при сетевом сбое одно событие может прийти дважды.

---

### Retry-стратегия в Notification.Service

1. Получено сообщение → попытка обработки (сохранение в БД).
2. Успех → `basicAck`.
3. Ошибка → `basicNack` с `requeue: false` → сообщение уходит в DLQ.
4. Оператор/мониторинг анализирует DLQ и при необходимости повторно публикует.

Для практики: реализовать retry с exponential backoff через заголовки `x-retry-count` перед отправкой в DLQ (при первых 3 попытках — nack + requeue через отдельный delay exchange).

## 7. Сценарии взаимодействия

### Flow: создание брони (happy path)

1. **Client → Booking.Service** `POST /bookings` `{eventId, ticketCategoryId, quantity}`
2. **Booking.Service → EventCatalog.Service** `GET /events/{eventId}` — получение данных события и цены категории; заголовок `X-Correlation-ID` пробрасывается.
3. **EventCatalog.Service → Redis** `GET ec:event:{id}` → попадание в кэш (или SELECT из БД + SET в Redis).
4. **EventCatalog.Service → Booking.Service** `200 {event, categories: [{id, price, ...}]}`
5. **Booking.Service** — валидация:
   - событие существует и статус `Active`;
   - дата события > `NOW()` (дата в будущем);
   - категория найдена в ответе;
   - `quantity` ∈ \[1, 10\];
   - суммарные активные брони пользователя на это событие + `quantity` ≤ 10 (SELECT SUM из booking\_db).
6. **Booking.Service → Redis** `DECRBY tc:counter:{categoryId} {quantity}`
   - Результат ≥ 0 → переходим к шагу 7.
   - Результат < 0 → `INCRBY tc:counter:{categoryId} {quantity}` (откат) → вернуть `409 Conflict`.
7. **Booking.Service → booking\_db** `INSERT INTO bookings (id, user_id, event_id, ticket_category_id, quantity, unit_price, status) VALUES (...)` с `unit_price` из шага 4.
   - При ошибке INSERT → `INCRBY tc:counter:{categoryId} {quantity}` (откат Redis) → `500`.
8. **Booking.Service → RabbitMQ** publish `BookingCreated` на exchange `ticketflow.events` routing key `booking.created`.
9. **Booking.Service → Client** `201 Created {booking}`
10. **RabbitMQ → Notification.Service** доставка сообщения.
11. **Notification.Service → notification\_db** `INSERT INTO notification_logs (...)`

---

### Flow: отмена брони (happy path)

1. **Client → Booking.Service** `DELETE /bookings/{id}` заголовок `X-User-Id`
2. **Booking.Service → booking\_db** `SELECT * FROM bookings WHERE id = ? AND status = 'Confirmed'`
   - Не найдено → `404`.
3. **Booking.Service → EventCatalog.Service** `GET /events/{eventId}` — получение даты события для валидации 24-часового окна; `X-Correlation-ID` пробрасывается.
4. **Booking.Service** — валидация:
   - `booking.user_id == X-User-Id` → иначе `403 Forbidden`;
   - дата события > `NOW() + 24h` → иначе `422`.
5. **Booking.Service → booking\_db** `UPDATE bookings SET status = 'Cancelled', updated_at = NOW() WHERE id = ?`
6. **Booking.Service → Redis** `INCRBY tc:counter:{ticketCategoryId} {quantity}` (возврат мест).
7. **Booking.Service → RabbitMQ** publish `BookingCancelled`.
8. **Booking.Service → Client** `200 OK {booking: {..., status: 'Cancelled'}}`
9. **RabbitMQ → Notification.Service → notification\_db** аналогично созданию.

---

### Flow: просмотр каталога (кэш-попадание)

1. **Client → EventCatalog.Service** `GET /events?page=1&pageSize=20`
2. **EventCatalog.Service → Redis** `GET ec:events:list:{hash}` → HIT → вернуть кэшированный JSON.
3. **EventCatalog.Service → Client** `200` из Redis (без обращения к PostgreSQL).

### Flow: просмотр каталога (кэш-промах)

1. **Client → EventCatalog.Service** `GET /events?page=1&pageSize=20`
2. **EventCatalog.Service → Redis** `GET ec:events:list:{hash}` → MISS.
3. **EventCatalog.Service → event\_catalog\_db** SELECT с фильтром по дате, пагинацией.
4. Для поля `available` каждой категории: `GET tc:counter:{categoryId}` из Redis.
5. **EventCatalog.Service → Redis** `SET ec:events:list:{hash} {json} EX 300`
6. **EventCatalog.Service → Client** `200`

## 8. Инфраструктура

### Docker Compose — сервисы и порты

| Сервис | Image | Внешний порт | Внутренний порт | Назначение |
| --- | --- | --- | --- | --- |
| `event_catalog` | build: ./EventCatalog.Service | 5001 | 8080 | EventCatalog |
| `booking` | build: ./Booking.Service | 5002 | 8080 | Booking |
| `notification` | build: ./Notification.Service | 5003 | 8080 | Notification |
| `postgres_catalog` | postgres:16 | 5432 | 5432 | БД EventCatalog |
| `postgres_booking` | postgres:16 | 5433 | 5432 | БД Booking |
| `postgres_notification` | postgres:16 | 5434 | 5432 | БД Notification |
| `redis` | redis:7-alpine | 6379 | 6379 | Кэш + счётчики |
| `rabbitmq` | rabbitmq:3-management | 5672 / 15672 | 5672 / 15672 | AMQP / Management UI |

Все сервисы находятся в одной docker-сети `ticketflow-net` (driver: bridge). Имя контейнера используется как hostname для связи между сервисами (например, `event_catalog:8080`).

---

### Переменные окружения

**EventCatalog.Service**

```
CONNECTIONSTRINGS__DefaultConnection=Host=postgres_catalog;Port=5432;Database=event_catalog_db;Username=postgres;Password=postgres
REDIS__ConnectionString=redis:6379
ASPNETCORE_ENVIRONMENT=Development
SERVICE_NAME=EventCatalog
```

**Booking.Service**

```
CONNECTIONSTRINGS__DefaultConnection=Host=postgres_booking;Port=5432;Database=booking_db;Username=postgres;Password=postgres
REDIS__ConnectionString=redis:6379
RABBITMQ__Host=rabbitmq
RABBITMQ__Username=guest
RABBITMQ__Password=guest
RABBITMQ__Exchange=ticketflow.events
SERVICEURL__EventCatalog=http://event_catalog:8080
ASPNETCORE_ENVIRONMENT=Development
SERVICE_NAME=Booking
```

**Notification.Service**

```
CONNECTIONSTRINGS__DefaultConnection=Host=postgres_notification;Port=5432;Database=notification_db;Username=postgres;Password=postgres
RABBITMQ__Host=rabbitmq
RABBITMQ__Username=guest
RABBITMQ__Password=guest
RABBITMQ__Queue=booking.notifications
ASPNETCORE_ENVIRONMENT=Development
SERVICE_NAME=Notification
```

---

### Health checks

Рекомендуется добавить `/health` endpoint (через `Microsoft.Extensions.Diagnostics.HealthChecks`) в каждый сервис с проверкой:

- PostgreSQL (`AddNpgsql`)
- Redis (`AddRedis`) — только для EventCatalog и Booking
- RabbitMQ (`AddRabbitMQ`) — только для Booking и Notification

Docker Compose `healthcheck` для инфраструктурных сервисов гарантирует порядок старта: приложения поднимаются только после готовности БД, Redis и RabbitMQ (`depends_on: condition: service_healthy`).

---

### GitLab CI/CD

| Stage | Job | Команда |
| --- | --- | --- |
| build | build\_catalog | `dotnet build EventCatalog.Service` |
| build | build\_booking | `dotnet build Booking.Service` |
| build | build\_notification | `dotnet build Notification.Service` |
| test | test\_catalog | `dotnet test EventCatalog.Service.Tests --collect:"XPlat Code Coverage"` |
| test | test\_booking | `dotnet test Booking.Service.Tests --collect:"XPlat Code Coverage"` |
| test | test\_notification | `dotnet test Notification.Service.Tests --collect:"XPlat Code Coverage"` |
| docker | docker\_catalog | `docker build -t $CI_REGISTRY_IMAGE/event-catalog:$CI_COMMIT_SHA` + push |
| docker | docker\_booking | аналогично |
| docker | docker\_notification | аналогично |

Артефакт: coverage.xml из каждого test-job; опционально — загрузка отчёта в GitLab Code Coverage (через regex `Total\s+\|\s+(\d+\.?\d+)%`).

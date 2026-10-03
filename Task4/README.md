# Task4. Проектирование продажи ОСАГО

## Цель

Спроектировать оформление ОСАГО для пиковой нагрузки до **2 500 одновременных пользователей**, при этом предложение каждой страховой компании должно отображаться пользователю сразу после получения, а общий срок ожидания ограничен **60 секундами**.

## 1. osago-aggregator

Добавляется отдельный сервис `osago-aggregator`.

Ответственность:

- получать заявки на расчёт ОСАГО;
- отправлять заявку во все доступные страховые компании;
- хранить соответствие внутренних и внешних `request_id`;
- опрашивать API страховых компаний;
- публиковать каждое полученное предложение сразу после его получения;
- завершать workflow после получения всех ответов или достижения deadline 60 секунд.

## 2. Нужно ли собственное хранилище

**Да.**

`osago-aggregator` использует отдельную PostgreSQL schema/database.

Хранятся:

- `application_id`;
- insurer;
- внешний `request_id`;
- статус запроса;
- полученное предложение;
- число retry;
- `next_poll_at`;
- deadline;
- технические ошибки.

Хранилище необходимо, потому что workflow асинхронный и длится до 60 секунд. Оно также позволяет восстановить polling после рестарта pod.

## 3. Интеграция core-app ↔ osago-aggregator

Основное взаимодействие — **Kafka/Event Streaming**.

### Команда от core-app

```text
osago.application.requested
```

Минимальный contract:

```json
{
  "event_id": "uuid",
  "application_id": "uuid",
  "customer_id": "uuid",
  "vehicle": {},
  "created_at": "...",
  "deadline_at": "..."
}
```

### События от osago-aggregator

Каждое предложение публикуется отдельно:

```text
osago.offer.received
```

После завершения:

```text
osago.application.completed
```

При технической невозможности обработки:

```text
osago.application.failed
```

Kafka message key — `application_id`, чтобы сохранить порядок событий одной заявки.

Синхронный REST между `core-app` и `osago-aggregator` для основного workflow не требуется.

## 4. API web-приложения

### Создание заявки

```http
POST /api/v1/osago/applications
```

Ответ:

```http
202 Accepted
```

```json
{
  "applicationId": "uuid",
  "status": "PROCESSING"
}
```

### Текущее состояние

```http
GET /api/v1/osago/applications/{applicationId}
```

Возвращает уже полученные предложения и итоговый статус.

### Поток новых предложений

```http
GET /api/v1/osago/applications/{applicationId}/offers/stream
Accept: text/event-stream
```

Используется **Server-Sent Events (SSE)**.

SSE подходит лучше WebSocket, потому что после создания заявки основной realtime-трафик односторонний: `core-app → browser`.

Каждое предложение отправляется клиенту сразу после события `osago.offer.received`.

## 5. Несколько replicas core-app

SSE-соединение пользователя может находиться на одном экземпляре `core-app`, а Kafka-событие получить другой экземпляр.

Поэтому используется **Redis Pub/Sub backplane**:

```text
Kafka
  |
  v
core-app consumer
  |
  v
Redis Pub/Sub
  |
  +--> core-app replica A
  +--> core-app replica B
  +--> core-app replica N
          |
          v
         SSE
```

Consumer group Kafka гарантирует, что событие обрабатывается одним экземпляром `core-app`, после чего он публикует notification в Redis. Replica, которая держит SSE-соединение нужного пользователя, отправляет его в браузер.

Состояние полученных предложений сохраняется в БД `core-app`, поэтому потеря transient Redis-сообщения не приводит к потере бизнес-данных: клиент может восстановить состояние через `GET`.

## 6. Несколько replicas osago-aggregator

`osago-aggregator` масштабируется горизонтально.

Чтобы два pod не опрашивали одну и ту же заявку одновременно:

- Kafka partition key = `application_id`;
- polling jobs хранятся в PostgreSQL;
- worker захватывает работу через lease / `SELECT ... FOR UPDATE SKIP LOCKED`;
- обработка событий идемпотентна.

## 7. Паттерны отказоустойчивости

### osago-aggregator → страховые компании

Применяются **отдельно для каждой страховой компании**:

| Паттерн | Решение |
|---|---|
| Timeout | Ограничение времени каждого HTTP вызова; общий deadline workflow — 60 сек |
| Retry | Exponential backoff + jitter для временных ошибок; только для безопасных/idempotent операций |
| Circuit Breaker | Отдельный circuit breaker на каждого страховщика |
| Rate Limiting | Отдельный лимит запросов для каждого API страховой компании |

Дополнительно:

- idempotency key при создании заявки, если API страховщика поддерживает;
- retry не должен продлевать общий срок более 60 секунд;
- после открытия circuit breaker остальные страховщики продолжают обрабатываться независимо.

### Web → core-app

На public API используется Rate Limiting для защиты от аномального числа запросов.

Для SSE:

- heartbeat;
- reconnect;
- клиент повторно запрашивает текущее состояние после reconnect.

### Kafka

Используются:

- retry topic;
- DLQ;
- idempotent consumers;
- correlation по `application_id`.

## 8. Почему не ждать все страховые компании синхронно

Нельзя строить:

```text
Browser → core-app → osago-aggregator → 10 insurers → ждать всех → response
```

При 2 500 одновременных пользователей это создаёт десятки тысяч внешних запросов, долгоживущие synchronous requests и зависимость пользовательского latency от самого медленного страховщика.

В выбранной модели:

```text
POST → 202 Accepted

insurer A ответил → offer A сразу в SSE
insurer B ответил → offer B сразу в SSE
...
60 секунд → workflow завершён
```

Таким образом выполняется ключевое бизнес-требование: предложения отображаются по мере готовности.

## Файлы

- `InsureTech_C4_container_osago.drawio` — обновлённая C4 container diagram;
- `api-contracts.md` — краткие REST и event contracts.

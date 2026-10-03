# API и event contracts для ОСАГО

## REST: web → core-app

### POST /api/v1/osago/applications

Создаёт заявку.

Ответ:

```http
202 Accepted
```

```json
{
  "applicationId": "7af6...",
  "status": "PROCESSING",
  "deadlineAt": "2026-10-04T12:01:00Z"
}
```

### GET /api/v1/osago/applications/{applicationId}

Пример:

```json
{
  "applicationId": "7af6...",
  "status": "PROCESSING",
  "offers": [
    {
      "insurerId": "insurer-1",
      "price": 15200,
      "receivedAt": "..."
    }
  ]
}
```

### GET /api/v1/osago/applications/{applicationId}/offers/stream

Content-Type:

```text
text/event-stream
```

Пример события:

```text
event: offer
id: 91a1...
data: {"applicationId":"7af6...","insurerId":"insurer-2","price":14900}
```

## Kafka

### osago.application.requested

Producer: `core-app`

Consumer: `osago-aggregator`

Key:

```text
application_id
```

### osago.offer.received

Producer: `osago-aggregator`

Consumer: `core-app`

Одно событие на одно полученное предложение.

### osago.application.completed

Producer: `osago-aggregator`

Consumer: `core-app`

Причины завершения:

- все страховщики ответили;
- истёк deadline 60 секунд.

### Общие поля событий

```json
{
  "event_id": "uuid",
  "event_type": "...",
  "event_version": 1,
  "application_id": "uuid",
  "occurred_at": "..."
}
```

Consumers должны быть идемпотентными по `event_id`.

# API и event contracts для ОСАГО

## REST API: web → core-app

### POST /api/v1/osago/applications

Создаёт заявку на расчёт ОСАГО.

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

Возвращает текущее состояние заявки и уже полученные предложения.

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

## SSE: core-app → web

### GET /api/v1/osago/applications/{applicationId}/offers/stream

Content-Type:

```text
text/event-stream
```

Каждое новое предложение передаётся отдельным SSE-событием.

Пример:

```text
event: offer
id: 91a1...
data: {"applicationId":"7af6...","insurerId":"insurer-2","price":14900}
```

После reconnect клиент может получить актуальное состояние через REST `GET`.

## Kafka

### osago.application.requested

Producer:

```text
core-app
```

Consumer:

```text
osago-aggregator
```

Key:

```text
application_id
```

Минимальный payload:

```json
{
  "event_id": "uuid",
  "event_version": 1,
  "application_id": "uuid",
  "customer_id": "uuid",
  "vehicle": {},
  "created_at": "...",
  "deadline_at": "..."
}
```

### osago.offer.received

Producer:

```text
osago-aggregator
```

Consumer:

```text
core-app
```

Публикуется отдельно для каждого полученного предложения.

### osago.application.completed

Producer:

```text
osago-aggregator
```

Consumer:

```text
core-app
```

Workflow завершается, когда:

- получены ответы всех страховых компаний;
- либо достигнут deadline 60 секунд.

### osago.application.failed

Используется для технической ошибки, из-за которой обработка заявки не может быть продолжена.

## Общие требования к событиям

Все события содержат:

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

# Task3. Переход на Event-Driven архитектуру

## Цель

Устранить синхронные зависимости между `core-app`, `ins-product-aggregator` и `ins-comp-settlement`, которые становятся критичными при увеличении количества страховых компаний.

Основной подход — перейти на **Event Streaming через Kafka**, не меняя функциональную декомпозицию существующих сервисов.

## Архитектурное решение

### Продукты и тарифы

`ins-product-aggregator` больше не обслуживает периодические REST-запросы от `core-app` и `ins-comp-settlement` для синхронизации продуктов.

Вместо этого сервис самостоятельно получает данные страховых компаний, нормализует их и публикует изменения в Kafka:

```text
insurance.products.updated
```

Потребители:

- `core-app`;
- `ins-comp-settlement`.

Каждый сервис обновляет собственную локальную read model продуктов и тарифов.

Это уменьшает синхронную связанность сервисов и исключает повторные запросы одинаковых данных.

### Оформленные страховки

Ночной REST-запрос `ins-comp-settlement → core-app` заменяется событием:

```text
insurance.policy.issued
```

После оформления страховки `core-app` публикует событие, а `ins-comp-settlement` формирует собственный реестр на основе потока событий.

## Transactional Outbox

Паттерн **Transactional Outbox используется** в сервисах, которые должны атомарно сохранить бизнес-данные и инициировать публикацию события.

### core-app

В одной транзакции сохраняются:

1. оформленная страховка;
2. запись в `outbox`.

После commit отдельный publisher или CDC-механизм отправляет событие `insurance.policy.issued` в Kafka.

### ins-product-aggregator

Актуальное нормализованное состояние продуктов сохраняется вместе с записью `outbox`, после чего публикуется событие `insurance.products.updated`.

Локальное хранилище агрегатора используется для фиксации нормализованного состояния продуктов и поддержки Transactional Outbox. Функциональная ответственность сервиса при этом не меняется.

## Надёжность обработки событий

Для Event Streaming предусматриваются:

- at-least-once delivery;
- idempotent consumers;
- `event_id` для дедупликации;
- key = `product_id` для продуктовых событий;
- key = `policy_id` для событий оформления;
- retry topics / DLQ;
- versioning event contract.

## Результат

В целевой архитектуре исключаются следующие периодические синхронные зависимости:

```text
core-app → ins-product-aggregator
ins-comp-settlement → ins-product-aggregator
ins-comp-settlement → core-app
```

Они заменяются публикацией и потреблением событий через Kafka.

Подробный анализ исходных проблем и рисков находится в `problems-and-risks.md`.

Диаграмма решения:

```text
InsureTech_C4_container_event-driven.drawio
```

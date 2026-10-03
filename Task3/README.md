# Task3. Переход на Event-Driven архитектуру

## Цель

Устранить синхронные зависимости между `core-app`, `ins-product-aggregator` и `ins-comp-settlement`, которые становятся критичными при подключении новых страховых компаний.

Основной подход — **Event Streaming через Kafka** без изменения функциональной декомпозиции существующих сервисов.

## Ключевые изменения

### 1. Продукты и тарифы

**As-Is**

`core-app` каждые 15 минут и `ins-comp-settlement` раз в сутки синхронно вызывают `ins-product-aggregator`, который в рамках запроса обращается во все страховые компании.

Недостаток: доступность и latency внутренних сервисов напрямую зависят от внешних API.

**To-Be**

`ins-product-aggregator` самостоятельно обновляет данные страховых компаний и публикует изменения в Kafka.

Топик:

```text
insurance.products.updated
```

Потребители:

- `core-app`;
- `ins-comp-settlement`.

Каждый потребитель обновляет свою локальную read model продуктов и тарифов.

Таким образом, `core-app` и `ins-comp-settlement` больше не вызывают `ins-product-aggregator` для периодической синхронизации данных.

### 2. Оформленные страховки

**As-Is**

`ins-comp-settlement` раз в сутки вызывает REST API `core-app` и забирает все оформленные за день страховки.

**To-Be**

После успешного оформления страховки `core-app` публикует событие:

```text
insurance.policy.issued
```

`ins-comp-settlement` потребляет эти события и формирует собственный реестр оформленных страховок.

Это устраняет ночной batch-запрос в `core-app`.

## Transactional Outbox

Паттерн **Transactional Outbox применяется**.

### core-app

При оформлении страховки в одной локальной транзакции сохраняются:

1. оформленная страховка;
2. запись в `outbox`.

Отдельный publisher/CDC публикует событие `insurance.policy.issued` в Kafka.

Это исключает ситуацию:

```text
страховка сохранена в БД
        +
событие не опубликовано
```

### ins-product-aggregator

После получения и нормализации данных страховых компаний актуальное состояние продуктов сохраняется в локальное хранилище вместе с записью `outbox`. После commit событие публикуется в `insurance.products.updated`.

## Надёжность обработки

Для Event Streaming используются:

- at-least-once delivery;
- idempotent consumers;
- `event_id` для дедупликации;
- key = `product_id` для продуктовых событий;
- key = `policy_id` для событий оформления;
- retry topics / DLQ для сообщений, которые не удалось обработать;
- schema/version поля в event contract.

## Результат

Основные синхронные зависимости удалены:

```text
ins-product-aggregator ─X─REST─> core-app
ins-product-aggregator ─X─REST─> ins-comp-settlement
core-app               ─X─REST─> ins-comp-settlement
```

Вместо них:

```text
ins-product-aggregator
        |
        | insurance.products.updated
        v
      Kafka
       /  \
      v    v
core-app  ins-comp-settlement


core-app
   |
   | insurance.policy.issued
   v
 Kafka
   |
   v
ins-comp-settlement
```

Диаграмма: `InsureTech_C4_container_event-driven.drawio`.

Подробный анализ проблем и рисков: `problems-and-risks.md`.

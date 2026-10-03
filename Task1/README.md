# Task1. Проектирование технологической архитектуры InsureTech

## 1. Требования и решения

| Требование | Решение |
|---|---|
| Доступность 99,9% | Приложение и БД размещаются в трёх зонах доступности |
| RTO ≤ 45 минут | Автоматический failover, Kubernetes rescheduling, балансировка между AZ |
| RPO ≤ 15 минут | Репликация PostgreSQL + WAL + резервные копии + PITR |
| Рост нагрузки | Horizontal Pod Autoscaler + Cluster Autoscaler |
| Работа 24/7 | HA Kubernetes + HA PostgreSQL + health checks |
| Пользователи из разных регионов РФ | CDN для статического контента + Application Load Balancer |
| Объём данных около 50 GB | Шардирование не применяется |

## 2. Целевая архитектура To-Be

Основная схема:

- `InureTech_технологическая архитектура_to-be.drawio`
- превью: `diagrams/architecture.png`

Целевая цепочка обработки трафика:

`B2C/B2B → Cloud CDN / Application Load Balancer → Managed Kubernetes (3 AZ) → Managed PostgreSQL HA (3 AZ) → Backup/PITR`

Состав приложения на этом этапе не меняется:

- `core-app`;
- `client-info`;
- `ins-product-aggregator`;
- `ins-comp-settlement`;
- web frontend.

`ins-product-aggregator` продолжает интеграцию с пятью страховыми компаниями.

## 3. Kubernetes и масштабирование

Используется один Managed Kubernetes cluster с HA control plane, распределённый между тремя зонами доступности. Worker nodes также размещаются в разных AZ.

Основная стратегия масштабирования — горизонтальная:

- HPA масштабирует количество pod;
- Cluster Autoscaler масштабирует worker nodes;
- `topologySpreadConstraints` или `podAntiAffinity` распределяют replicas между AZ;
- для критичных workload используется `PodDisruptionBudget`.

Схема: `diagrams/scaling.png`.

Выбран один региональный HA-кластер вместо нескольких независимых кластеров: он закрывает требования доступности и отказоустойчивости при меньшей эксплуатационной сложности.

## 4. Входящий трафик и health checks

Для B2C статический контент web-приложения отдаётся через CDN, что уменьшает задержки для пользователей из разных регионов и снижает нагрузку на origin.

B2B-трафик поступает через Application Load Balancer. Ограничение партнёров по RPS относится к следующим заданиям и здесь не реализуется.

Проверки состояния:

- ALB выполняет health checks backend;
- `readinessProbe` определяет готовность pod принимать трафик;
- `livenessProbe` используется для контроля работоспособности контейнера;
- unhealthy backend исключается из балансировки.

Схема: `diagrams/health-failover.png`.

## 5. Отказоустойчивость и хранение данных

PostgreSQL разворачивается как HA-кластер из трёх hosts в разных AZ:

- один master;
- две replicas;
- streaming replication;
- automatic failover при недоступности master;
- WAL, автоматические резервные копии и PITR.

Это обеспечивает целевые значения **RPO ≤ 15 минут** и **RTO ≤ 45 минут**.

Схема: `diagrams/postgresql-ha.png`.

### Сценарии отказов

| Отказ | Реакция системы |
|---|---|
| Pod | Kubernetes перезапускает pod |
| Worker node | Pods запускаются на доступных nodes |
| Одна Availability Zone | ALB направляет трафик в оставшиеся AZ |
| PostgreSQL master | Replica повышается до master |

## 6. Шардирование

Шардирование не применяется. Текущий объём данных около 50 GB не требует горизонтального разделения PostgreSQL. На данном этапе sharding увеличил бы сложность разработки и эксплуатации без практической необходимости.

Существующее логическое разделение данных сервисов сохраняется; в Task1 изменяется инфраструктурная конфигурация БД.

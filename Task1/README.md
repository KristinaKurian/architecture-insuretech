# Task1. Проектирование технологической архитектуры InsureTech

## 1. Требования и принятые решения

| Требование | Решение |
|---|---|
| Доступность 99,9% | Размещение приложения и БД в трёх зонах доступности |
| RTO ≤ 45 минут | Автоматический failover, Kubernetes rescheduling, балансировка между AZ |
| RPO ≤ 15 минут | Репликация PostgreSQL + WAL + резервные копии + PITR |
| Рост нагрузки | Horizontal Pod Autoscaler + Cluster Autoscaler |
| Работа 24/7 | HA Kubernetes + HA PostgreSQL + health checks |
| Пользователи из разных регионов РФ | CDN для статического контента + единая точка входа через ALB |
| Объём данных около 50 GB | Шардирование не требуется |

## 2. Целевая архитектура To-Be

Основная схема находится в файле:

- `InureTech_технологическая архитектура_to-be.drawio`
- превью: `diagrams/architecture.png`

Целевая цепочка обработки трафика:

`B2C/B2B → Cloud CDN / Application Load Balancer → Managed Kubernetes (3 AZ) → Managed PostgreSQL HA (3 AZ) → Backup/PITR`

Состав бизнес-приложения сохраняется без декомпозиции монолита на этом этапе:

- `core-app`;
- `client-info`;
- `ins-product-aggregator`;
- `ins-comp-settlement`;
- web frontend.

`ins-product-aggregator` продолжает интеграцию с пятью страховыми компаниями.

## 3. Kubernetes и масштабирование

Используется **один Managed Kubernetes cluster с HA control plane**, распределённый между тремя зонами доступности. Worker nodes также размещаются в разных AZ.

Почему один региональный кластер, а не несколько независимых:

- этого достаточно для требования доступности 99,9%;
- отказ одной AZ не должен приводить к потере всего приложения;
- эксплуатация одного HA-кластера проще, чем нескольких независимых кластеров;
- требования RTO/RPO не требуют отдельного Kubernetes-кластера в каждой зоне.

Основная стратегия масштабирования — **горизонтальная**:

- HPA увеличивает/уменьшает число pod;
- Cluster Autoscaler изменяет число worker nodes;
- `topologySpreadConstraints` или `podAntiAffinity` распределяют replicas между AZ;
- для критичных workload используется `PodDisruptionBudget`.

Схема: `diagrams/scaling.png`.

## 4. Входящий трафик и health checks

### B2C

Статический контент web-приложения отдаётся через CDN. Это снижает нагрузку на origin и уменьшает влияние географического положения пользователя на время загрузки статических ресурсов.

### B2B

Партнёрский трафик поступает через Application Load Balancer. Ограничение партнёров по RPS будет прорабатываться в следующих заданиях, чтобы не смешивать задачи технологической отказоустойчивости и API governance.

### Проверки состояния

- ALB выполняет health checks backend;
- `readinessProbe` определяет, может ли pod принимать трафик;
- `livenessProbe` определяет, требуется ли перезапуск контейнера;
- unhealthy backend исключается из балансировки.

Схема: `diagrams/health-failover.png`.

## 5. Отказоустойчивость и хранение данных

PostgreSQL разворачивается как HA-кластер из трёх hosts в разных AZ:

- один master;
- две replicas;
- streaming replication;
- automatic failover при недоступности master.

Дополнительно используются:

- WAL;
- автоматические резервные копии;
- Point-in-Time Recovery (PITR).

Это позволяет спроектировать решение с целевыми характеристиками:

- **RPO ≤ 15 минут**;
- **RTO ≤ 45 минут**.

Схема: `diagrams/postgresql-ha.png`.

### Сценарии отказов

| Отказ | Реакция системы |
|---|---|
| Pod | Kubernetes перезапускает pod |
| Worker node | Pods запускаются на доступных nodes |
| Одна Availability Zone | ALB направляет трафик в оставшиеся AZ |
| PostgreSQL master | Replica повышается до master |

## 6. Решение по шардированию

**Шардирование не применяется.**

Текущий объём данных составляет около 50 GB. Для такого объёма HA PostgreSQL-кластера достаточно. Шардирование на данном этапе увеличит сложность транзакций, миграций, резервного копирования и эксплуатации без измеримой пользы.

Существующее логическое разделение данных сервисов сохраняется; в Task1 изменяется инфраструктурная конфигурация PostgreSQL.

## 7. Что показано на основной схеме

- B2C clients и B2B partners;
- Cloud CDN;
- Application Load Balancer и health checks;
- Managed Kubernetes в трёх AZ;
- replicas приложений в разных AZ;
- HPA и Cluster Autoscaler;
- Managed PostgreSQL HA в трёх AZ;
- streaming replication и automatic failover;
- Backup/WAL/PITR;
- интеграции `ins-product-aggregator` со страховыми компаниями;
- решение не использовать sharding.

## 8. Короткие архитектурные заметки для draw.io

**Kubernetes**  
Single regional Kubernetes cluster across 3 AZ. HA control plane and worker nodes distributed across availability zones.

**Scaling**  
HPA scales Pods. Cluster Autoscaler scales worker nodes. Replicas are distributed across AZ.

**Database**  
PostgreSQL HA: 3 hosts / 3 AZ, automatic failover, replication, backup and PITR. Target RPO ≤ 15 min, RTO ≤ 45 min.

**Sharding**  
Not required. Current data volume ≈ 50 GB.

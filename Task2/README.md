# Task2. Динамическое масштабирование контейнеров

## Цель

Настроить автоматическое горизонтальное масштабирование тестового приложения в Kubernetes на основании потребления оперативной памяти.

Используется тестовое приложение `scaletestapp`, которое предоставляет:

- `GET /` — возвращает идентификатор pod;
- `GET /metrics` — возвращает Prometheus-метрики;
- порт приложения — `8080`.

## Структура

```text
Task2/
├── deployment.yaml
├── service.yaml
├── hpa.yaml
├── locustfile.py
├── README.md
└── results/
    └── .gitkeep
```

После нагрузочного теста в `results/` необходимо добавить скриншоты или логи, подтверждающие изменение количества replicas.

---

## 1. Запуск Minikube

```bash
minikube start
kubectl cluster-info
kubectl get nodes
```

Проверить, что node находится в состоянии `Ready`.

---

## 2. Включение Metrics Server

```bash
minikube addons enable metrics-server
```

Проверка:

```bash
kubectl get pods -n kube-system | grep metrics-server
kubectl top nodes
```

Сразу после включения metrics-server команда `kubectl top` может некоторое время не возвращать значения — нужно дождаться появления метрик.

---

## 3. Deployment

Применить:

```bash
kubectl apply -f deployment.yaml
```

Проверить:

```bash
kubectl get deployments
kubectl get pods -l app=scaletestapp
```

Deployment запускается с одной replica.

Для контейнера установлены:

```yaml
resources:
  requests:
    memory: "10Mi"
    cpu: "10m"
  limits:
    memory: "30Mi"
    cpu: "100m"
```

`30Mi` — требуемый заданием memory limit.

`requests.memory` дополнительно задан, потому что HPA с `target.type: Utilization` рассчитывает процент потребления памяти относительно memory request.

---

## 4. Service

Применить:

```bash
kubectl apply -f service.yaml
```

Получить URL приложения:

```bash
minikube service scaletestapp --url
```

Пример:

```text
http://127.0.0.1:54321
```

Проверить приложение:

```bash
curl <SERVICE_URL>/
curl <SERVICE_URL>/metrics
```

При обращении к `/` приложение должно вернуть идентификатор pod.

---

## 5. Horizontal Pod Autoscaler

Применить:

```bash
kubectl apply -f hpa.yaml
```

Проверить:

```bash
kubectl get hpa
kubectl describe hpa scaletestapp
```

Настройки:

- `minReplicas: 1`;
- `maxReplicas: 10`;
- метрика — `memory`;
- целевая утилизация памяти — `80%`.

Наблюдение в реальном времени:

```bash
kubectl get hpa -w
```

В отдельном терминале:

```bash
kubectl get pods -w
```

---

## 6. Проверка метрик Kubernetes

Перед запуском нагрузки:

```bash
kubectl top pods
kubectl get hpa
```

Если в поле `TARGETS` HPA отображается `<unknown>/80%`, проверить:

```bash
kubectl top pods
kubectl get apiservice | grep metrics
kubectl describe hpa scaletestapp
```

---

## 7. Нагрузочное тестирование Locust

Установка:

```bash
python -m pip install locust
```

Получить адрес сервиса:

```bash
minikube service scaletestapp --url
```

Запустить Locust:

```bash
locust
```

Открыть:

```text
http://localhost:8089
```

В поле **Host** указать URL, который вернула команда:

```bash
minikube service scaletestapp --url
```

Для начала можно использовать:

- Number of users: `100`;
- Spawn rate: `10`.

Если HPA не начинает масштабирование, постепенно увеличить количество пользователей.

Во время теста наблюдать:

```bash
kubectl get hpa -w
```

```bash
kubectl get deployments,pods
```

```bash
kubectl top pods
```

---

## 8. Что должно произойти

До нагрузки:

```text
NAME           REFERENCE                 TARGETS   MINPODS   MAXPODS   REPLICAS
scaletestapp   Deployment/scaletestapp   .../80%   1         10        1
```

При росте memory utilization выше целевого значения HPA увеличивает `REPLICAS`.

Схематично:

```text
Locust
   |
   v
Service
   |
   v
Pod
   |
   | memory > 80% of request
   v
Metrics Server
   |
   v
HPA
   |
   v
Deployment
   |
   +---- Pod 1
   +---- Pod 2
   +---- ...
   +---- Pod N
```

После снижения нагрузки HPA постепенно уменьшает количество replicas. Scale-down не обязан происходить мгновенно: HPA использует окно стабилизации.

---

## 9. Доказательства для сдачи

В директорию `results/` добавить минимум два подтверждения.

### Вариант 1 — скриншоты

Например:

```text
results/
├── before-load.png
├── during-load.png
└── after-load.png
```

На скриншотах желательно показать:

1. до нагрузки — `1` replica;
2. при нагрузке — replicas стало больше `1`;
3. HPA показывает текущую memory utilization.

### Вариант 2 — логи

Можно сохранить вывод:

```bash
kubectl get hpa > results/hpa-before.txt
kubectl get pods >> results/hpa-before.txt
```

Во время нагрузки:

```bash
kubectl get hpa > results/hpa-during-load.txt
kubectl get pods >> results/hpa-during-load.txt
kubectl top pods >> results/hpa-during-load.txt
```

---

## 10. Полная последовательность команд

```bash
minikube start

minikube addons enable metrics-server

kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f hpa.yaml

kubectl get pods
kubectl get svc
kubectl get hpa

kubectl top pods

minikube service scaletestapp --url

locust
```

Для наблюдения:

```bash
kubectl get hpa -w
```

---

## Результат

Настроено динамическое горизонтальное масштабирование приложения:

- стартовое количество replicas — `1`;
- максимальное количество replicas — `10`;
- триггер масштабирования — memory utilization;
- target memory utilization — `80%`;
- memory limit контейнера — `30Mi`;
- метрики предоставляет Metrics Server;
- нагрузка генерируется Locust.

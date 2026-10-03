# Task2. Динамическое масштабирование контейнеров

## Цель

Настроить Horizontal Pod Autoscaler для тестового приложения `scaletestapp`, чтобы количество replicas автоматически изменялось в зависимости от потребления оперативной памяти.

Параметры задания:

- начальное количество replicas — `1`;
- memory limit — `30Mi`;
- target memory utilization — `80%`;
- максимальное количество replicas — `10`;
- порт приложения — `8081`.

## 1. Запуск кластера и Metrics Server

```bash
minikube start
minikube addons enable metrics-server
```

Проверка:

```bash
kubectl get nodes
kubectl get pods -n kube-system | grep metrics-server
kubectl top nodes
```

## 2. Развёртывание приложения

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f hpa.yaml
```

Проверка:

```bash
kubectl get deployments
kubectl get pods
kubectl get svc
kubectl get hpa
kubectl top pods
```

В `deployment.yaml` дополнительно задан `requests.memory`, потому что HPA с `target.type: Utilization` рассчитывает процент использования памяти относительно resource request.

## 3. Доступ к приложению

Получить URL сервиса:

```bash
minikube service scaletestapp --url
```

Проверить endpoints:

```bash
curl <SERVICE_URL>/
curl <SERVICE_URL>/metrics
```

На macOS также удобно использовать port-forward:

```bash
kubectl port-forward service/scaletestapp 8081:8080
```

После этого приложение доступно по адресу `http://127.0.0.1:8081`.

## 4. HPA

Конфигурация HPA:

- `minReplicas: 1`;
- `maxReplicas: 10`;
- resource metric — `memory`;
- `averageUtilization: 80`.

Наблюдение:

```bash
kubectl get hpa -w
```

В отдельном терминале:

```bash
kubectl get pods -w
```

## 5. Нагрузочное тестирование

Установка и запуск Locust:

```bash
python -m pip install locust
locust
```

Web UI:

```text
http://localhost:8089
```

В поле **Host** указывается адрес сервиса без завершающего `/`, например:

```text
http://127.0.0.1:8080
```

Для теста использовался сценарий из `locustfile.py` с запросами `GET /`.

Во время нагрузки контролировались:

```bash
kubectl get hpa
kubectl get pods
kubectl top pods
```

При превышении целевого уровня memory utilization HPA увеличивает количество replicas. После снижения нагрузки scale-down происходит не мгновенно из-за окна стабилизации HPA.

## 6. Результаты теста

В директории `results/` сохранены подтверждения работы autoscaling:

- `01-start-status.png` — состояние до нагрузки: одна replica;
- `02-locust-load.png` — активная нагрузка Locust;
- `03-hpa-scaled.png` — увеличение количества replicas;
- `04-hpa-events.png` — события HPA, подтверждающие rescale.

## 7. Особенность macOS ARM64

Официальный образ `ghcr.io/yandex-practicum/scaletestapp:latest` на момент выполнения теста не содержал manifest для `linux/arm64/v8`. Поэтому локальный запуск на Apple Silicon выполнялся через сборку образа из исходников `scaletestapp` внутри Minikube.

Пример локального workaround:

```bash
git clone https://github.com/Yandex-Practicum/scaletestapp.git
cd scaletestapp

eval $(minikube docker-env)
docker build -t scaletestapp:local .
```

Для локального теста image в Deployment временно заменялся на:

```yaml
image: scaletestapp:local
imagePullPolicy: Never
```

В сдаваемом `deployment.yaml` оставлена ссылка на официальный образ из условия задания.

## Результат

Настроено динамическое масштабирование приложения по потреблению памяти: HPA использует target `80%`, масштабирует Deployment от `1` до `10` replicas, а результаты нагрузочного теста зафиксированы в `results/`.

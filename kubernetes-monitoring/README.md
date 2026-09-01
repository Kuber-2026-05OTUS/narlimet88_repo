# Kubernetes Monitoring

Домашнее задание по настройке мониторинга приложения в Kubernetes с использованием Prometheus Operator.

## Что реализовано

В рамках задания:

- создан собственный Docker-образ nginx;
- nginx настроен на выдачу статистики через `/stub_status`;
- рядом с nginx запущен `nginx-prometheus-exporter`;
- создан Kubernetes Service для nginx и exporter;
- установлен `kube-prometheus-stack`, включающий Prometheus Operator;
- создан `ServiceMonitor`;
- Prometheus успешно обнаруживает exporter и собирает метрики;
- проверено наличие метрики `nginx_up = 1`.

Схема работы:

```text
nginx
  |
  | /stub_status
  v
nginx-prometheus-exporter
  |
  | :9113/metrics
  v
Service
  |
  v
ServiceMonitor
  |
  v
Prometheus Operator
  |
  v
Prometheus
```

## Структура каталога

```text
kubernetes-monitoring/
├── README.md
├── Dockerfile
├── nginx.conf
├── deployment.yaml
├── service.yaml
└── servicemonitor.yaml
```

## Docker image

Для nginx используется собственный Docker-образ.

### Dockerfile

```dockerfile
FROM nginx:1.27-alpine

COPY nginx.conf /etc/nginx/nginx.conf

RUN echo '<html><body><h1>OTUS Kubernetes Monitoring</h1></body></html>' \
    > /usr/share/nginx/html/index.html

EXPOSE 80
```

### nginx.conf

Nginx отдаёт основную страницу и предоставляет статистику через `/stub_status`.

```nginx
events {}

http {
    server {
        listen 80;

        location / {
            root /usr/share/nginx/html;
            index index.html;
        }

        location /stub_status {
            stub_status;
            access_log off;
        }
    }
}
```

## Сборка образа в Minikube

Образ собирается непосредственно внутри Minikube:

```bash
minikube image build -t otus-nginx-monitoring:1.0 .
```

Проверка:

```bash
minikube image ls | grep otus-nginx-monitoring
```

Пример:

```text
docker.io/library/otus-nginx-monitoring:1.0
```

## Установка Prometheus Operator

Для установки Prometheus Operator используется Helm chart `kube-prometheus-stack`.

Добавление репозитория:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Установка:

```bash
helm upgrade --install prometheus \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

Проверка Helm release:

```bash
helm list -n monitoring
```

Пример:

```text
NAME         NAMESPACE   STATUS     CHART
prometheus   monitoring  deployed   kube-prometheus-stack
```

Проверка Pod:

```bash
kubectl get pods -n monitoring
```

В namespace `monitoring` запускаются компоненты Prometheus Stack:

```text
alertmanager-prometheus-kube-prometheus-alertmanager-0
prometheus-grafana-...
prometheus-kube-prometheus-operator-...
prometheus-kube-state-metrics-...
prometheus-prometheus-kube-prometheus-prometheus-0
prometheus-prometheus-node-exporter-...
```

Проверка CRD Prometheus Operator:

```bash
kubectl get crd | grep monitoring.coreos.com
```

В том числе должны присутствовать:

```text
podmonitors.monitoring.coreos.com
prometheuses.monitoring.coreos.com
prometheusrules.monitoring.coreos.com
servicemonitors.monitoring.coreos.com
```

## Deployment

В одном Pod запускаются два контейнера:

- nginx;
- nginx-prometheus-exporter.

`deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-monitoring
  labels:
    app: nginx-monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-monitoring
  template:
    metadata:
      labels:
        app: nginx-monitoring
    spec:
      containers:
        - name: nginx
          image: otus-nginx-monitoring:1.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 80

        - name: nginx-exporter
          image: nginx/nginx-prometheus-exporter:1.4.2
          args:
            - --nginx.scrape-uri=http://127.0.0.1/stub_status
          ports:
            - name: metrics
              containerPort: 9113
```

Exporter получает статистику nginx через:

```text
http://127.0.0.1/stub_status
```

и отдаёт метрики в Prometheus-формате на порту `9113` по пути `/metrics`.

Применение:

```bash
kubectl apply -f deployment.yaml
```

Проверка:

```bash
kubectl get pods -l app=nginx-monitoring
```

Ожидаемый результат:

```text
READY   STATUS
2/2     Running
```

## Service

Для доступа к nginx и exporter создаётся Service.

`service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-monitoring
  labels:
    app: nginx-monitoring
spec:
  selector:
    app: nginx-monitoring
  ports:
    - name: http
      port: 80
      targetPort: http

    - name: metrics
      port: 9113
      targetPort: metrics
```

Применение:

```bash
kubectl apply -f service.yaml
```

Проверка:

```bash
kubectl get svc nginx-monitoring
```

Проверка EndpointSlice:

```bash
kubectl get endpointslice \
  -l kubernetes.io/service-name=nginx-monitoring
```

Service должен указывать на Pod приложения и предоставлять два порта:

```text
80
9113
```

## Проверка nginx-prometheus-exporter

Для локальной проверки exporter:

```bash
kubectl port-forward svc/nginx-monitoring 9113:9113
```

В другом терминале:

```bash
curl -s http://127.0.0.1:9113/metrics | grep '^nginx_'
```

Пример результата:

```text
nginx_connections_accepted 1
nginx_connections_active 1
nginx_connections_handled 1
nginx_connections_reading 0
nginx_connections_waiting 0
nginx_connections_writing 1
nginx_http_requests_total 1
nginx_up 1
```

Таким образом exporter успешно получает статистику nginx и преобразует её в Prometheus-формат.

## ServiceMonitor

Для автоматического обнаружения exporter Prometheus используется `ServiceMonitor`.

`servicemonitor.yaml`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nginx-monitoring
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: nginx-monitoring
  endpoints:
    - port: metrics
      path: /metrics
      interval: 15s
```

Label:

```text
release: prometheus
```

соответствует `serviceMonitorSelector` экземпляра Prometheus, созданного Helm release `prometheus`.

Применение:

```bash
kubectl apply -f servicemonitor.yaml
```

Проверка:

```bash
kubectl get servicemonitor nginx-monitoring
```

Проверка selector Prometheus:

```bash
kubectl -n monitoring get prometheus prometheus-kube-prometheus-prometheus \
  -o jsonpath='{.spec.serviceMonitorSelector}{"\n"}'
```

Результат:

```json
{"matchLabels":{"release":"prometheus"}}
```

Также Prometheus разрешено обнаруживать `ServiceMonitor` в других namespace:

```bash
kubectl -n monitoring get prometheus prometheus-kube-prometheus-prometheus \
  -o jsonpath='{.spec.serviceMonitorNamespaceSelector}{"\n"}'
```

Результат:

```json
{}
```

## Проверка конфигурации Prometheus

После создания `ServiceMonitor` Prometheus Operator автоматически добавляет соответствующий scrape job.

Проверка:

```bash
kubectl -n monitoring get secret prometheus-prometheus-kube-prometheus-prometheus \
  -o jsonpath='{.data.prometheus\.yaml\.gz}' \
  | base64 -d | gunzip | grep -n -A20 -B5 nginx-monitoring
```

В конфигурации присутствует job:

```text
job_name: serviceMonitor/default/nginx-monitoring/0
```

Для него настроены:

```text
namespace: default
metrics_path: /metrics
scrape_interval: 15s
port: metrics
```

## Проверка target в Prometheus

Для доступа к Prometheus:

```bash
kubectl -n monitoring port-forward \
  svc/prometheus-kube-prometheus-prometheus \
  9090:9090
```

В другом терминале проверяем target через API:

```bash
curl -s http://127.0.0.1:9090/api/v1/targets | python3 -c '
import sys, json

data=json.load(sys.stdin)

for t in data["data"]["activeTargets"]:
    if "nginx-monitoring" in t.get("scrapePool",""):
        print("scrapePool:", t.get("scrapePool"))
        print("scrapeUrl:", t.get("scrapeUrl"))
        print("health:", t.get("health"))
        print("lastError:", t.get("lastError"))
        print("labels:", t.get("labels"))
'
```

Получен активный target:

```text
scrapePool: serviceMonitor/default/nginx-monitoring/0
scrapeUrl: http://10.244.0.108:9113/metrics
health: up
lastError:
```

Target успешно обнаружен Prometheus и имеет состояние:

```text
health: up
```

## Проверка метрик через Prometheus

Проверяем метрику `nginx_up`:

```bash
curl -s \
  'http://127.0.0.1:9090/api/v1/query?query=nginx_up' \
  | python3 -m json.tool
```

Prometheus возвращает временной ряд со следующими labels:

```text
__name__: nginx_up
container: nginx-exporter
endpoint: metrics
job: nginx-monitoring
namespace: default
service: nginx-monitoring
```

Значение метрики:

```text
nginx_up = 1
```

Это подтверждает, что Prometheus успешно собирает метрики nginx через `nginx-prometheus-exporter`.

Дополнительно можно проверить счётчик HTTP-запросов:

```bash
curl -s \
  'http://127.0.0.1:9090/api/v1/query?query=nginx_http_requests_total' \
  | python3 -m json.tool
```

## Развёртывание приложения

После установки `kube-prometheus-stack` приложение и мониторинг применяются командами:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f servicemonitor.yaml
```

Проверка:

```bash
kubectl get pods -l app=nginx-monitoring
kubectl get svc nginx-monitoring
kubectl get servicemonitor nginx-monitoring
```

## Результат

В Kubernetes-кластере настроен полный цикл сбора метрик nginx:

```text
nginx
  |
  | /stub_status
  v
nginx-prometheus-exporter
  |
  | /metrics :9113
  v
Kubernetes Service
  |
  v
ServiceMonitor
  |
  v
Prometheus Operator
  |
  v
Prometheus
```

Проверено:

```text
Target health: up
nginx_up: 1
```

Prometheus Operator успешно обнаруживает приложение через `ServiceMonitor`, а Prometheus собирает метрики nginx.

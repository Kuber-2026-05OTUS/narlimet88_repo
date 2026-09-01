# Kubernetes Templating

Домашнее задание по теме **Helm, Helmfile и шаблонизация Kubernetes-манифестов**.

## Структура

```text
kubernetes-templating/
├── README.md
├── homework/
│   ├── Chart.yaml
│   ├── Chart.lock
│   ├── values.yaml
│   ├── charts/
│   │   └── redis-23.1.1.tgz
│   └── templates/
│       ├── NOTES.txt
│       ├── _helpers.tpl
│       ├── configmap.yaml
│       ├── deployment.yaml
│       ├── gateway.yaml
│       ├── httproute.yaml
│       ├── monitoring-rbac.yaml
│       ├── pvc.yaml
│       ├── service.yaml
│       └── serviceaccount.yaml
└── helmfile/
    ├── helmfile.yaml
    └── values/
        ├── kafka-dev.yaml
        └── kafka-prod.yaml
```

## Используемые инструменты

```text
Helm:      v4.2.4
Helmfile:  1.7.4
helm-diff: 3.15.12
```

---

# 1. Helm chart приложения

В каталоге `homework` создан Helm chart для приложения из предыдущих домашних заданий.

Основные параметры приложения вынесены в `values.yaml`:

- количество реплик;
- имя и tag образа;
- образ init-контейнера;
- порты приложения и Service;
- hostname для HTTPRoute;
- ServiceAccount;
- readinessProbe;
- PVC;
- ConfigMap;
- nodeSelector;
- RollingUpdate;
- параметры Gateway API;
- параметры Redis.

Repository и tag Docker image задаются отдельно.

Пример:

```yaml
image:
  repository: python
  tag: 3.12-alpine
```

Readiness probe может быть включена или выключена через `values.yaml`.

## Проверка Helm chart

Проверка синтаксиса:

```bash
cd kubernetes-templating/homework

helm lint .
```

Рендеринг Kubernetes-манифестов:

```bash
helm template homework .
```

Установка chart:

```bash
helm upgrade --install homework . \
  --namespace homework \
  --create-namespace
```

Проверка:

```bash
helm list -n homework

kubectl get pods -n homework
kubectl get svc -n homework
kubectl get pvc -n homework
kubectl get gateway -n homework
kubectl get httproute -n homework
```

## NOTES.txt

После установки Helm выводит адрес для обращения к приложению.

Приложение доступно через Gateway API по hostname:

```text
homework.otus
```

---

# 2. Redis dependency

В Helm chart добавлена зависимость от community chart Redis.

`Chart.yaml`:

```yaml
dependencies:
  - name: redis
    version: "23.1.1"
    repository: "oci://registry-1.docker.io/bitnamicharts"
    condition: redis.enabled
```

Зависимость загружается командой:

```bash
helm dependency update
```

Redis включается через `values.yaml`:

```yaml
redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: false
  master:
    persistence:
      enabled: false
```

Проверка Redis:

```bash
kubectl get pods -n homework
kubectl get statefulset -n homework
```

---

# 3. Helmfile

Для управления несколькими инсталляциями Kafka используется Helmfile.

Файл:

```text
helmfile/helmfile.yaml
```

Создаются два независимых Kafka-кластера:

| Namespace | Kafka | Brokers | Client | Inter-broker |
|-----------|-------|---------|--------|--------------|
| prod | 3.5.2 | 5 | SASL_PLAINTEXT | SASL_PLAINTEXT |
| dev | 4.0.0 | 1 | PLAINTEXT | PLAINTEXT |

---

# 4. Kafka PROD

Для prod используется Bitnami Kafka chart:

```yaml
chart: oci://registry-1.docker.io/bitnamicharts/kafka
version: 25.3.5
```

В chart отсутствует image с точной требуемой версией Kafka 3.5.2, поэтому версия Kafka задаётся отдельно:

```yaml
image:
  registry: docker.io
  repository: bitnamilegacy/kafka
  tag: 3.5.2-debian-12-r30
```

Используется legacy repository Bitnami, так как исторический образ Kafka 3.5.2 недоступен в основном repository `bitnami/kafka`.

Конфигурация:

```yaml
controller:
  replicaCount: 1
  controllerOnly: true

broker:
  replicaCount: 5

listeners:
  client:
    protocol: SASL_PLAINTEXT
  controller:
    protocol: SASL_PLAINTEXT
  interbroker:
    protocol: SASL_PLAINTEXT
```

`controllerOnly: true` используется для разделения controller и broker ролей.

В результате создаются:

```text
kafka-broker       5 replicas
kafka-controller   1 replica
```

Для локального Minikube увеличено время запуска liveness/readiness probes, поскольку одновременный запуск нескольких JVM Kafka занимает значительное время.

```yaml
livenessProbe:
  enabled: true
  initialDelaySeconds: 120
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 6

readinessProbe:
  enabled: true
  initialDelaySeconds: 60
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 6
```

## Проверка PROD

```bash
kubectl get sts -n prod
```

Результат:

```text
NAME               READY
kafka-broker       5/5
kafka-controller   1/1
```

Проверка Pod:

```bash
kubectl get pods -n prod
```

Результат:

```text
kafka-broker-0       1/1   Running
kafka-broker-1       1/1   Running
kafka-broker-2       1/1   Running
kafka-broker-3       1/1   Running
kafka-broker-4       1/1   Running
kafka-controller-0   1/1   Running
```

Проверка версии Kafka:

```bash
kubectl exec -n prod kafka-broker-0 -- \
  /opt/bitnami/kafka/bin/kafka-topics.sh --version
```

Результат:

```text
3.5.2
```

Проверка listener:

```bash
kubectl get cm -n prod kafka-broker-configuration -o yaml \
  | grep -Ei 'listener.security.protocol.map|inter.broker.listener.name|listeners='
```

Результат:

```text
listeners=CLIENT://:9092,INTERNAL://:9094
listener.security.protocol.map=CLIENT:SASL_PLAINTEXT,INTERNAL:SASL_PLAINTEXT,CONTROLLER:SASL_PLAINTEXT
advertised.listeners=CLIENT://advertised-address-placeholder:9092,INTERNAL://advertised-address-placeholder:9094
inter.broker.listener.name=INTERNAL
```

---

# 5. Kafka DEV

Для dev используется Bitnami Kafka chart:

```yaml
chart: oci://registry-1.docker.io/bitnamicharts/kafka
version: 32.4.3
```

Образ:

```yaml
image:
  registry: docker.io
  repository: bitnamilegacy/kafka
  tag: 4.0.0-debian-12-r10
```

Конфигурация:

```yaml
controller:
  replicaCount: 1
  controllerOnly: true

broker:
  replicaCount: 1

listeners:
  client:
    protocol: PLAINTEXT
  controller:
    protocol: PLAINTEXT
  interbroker:
    protocol: PLAINTEXT
```

Таким образом в namespace `dev` запускается один broker и один отдельный KRaft controller.

## Проверка DEV

```bash
kubectl get sts -n dev
```

Результат:

```text
NAME               READY
kafka-broker       1/1
kafka-controller   1/1
```

Проверка версии Kafka:

```bash
kubectl exec -n dev kafka-broker-0 -- \
  /opt/bitnami/kafka/bin/kafka-topics.sh --version
```

Результат:

```text
4.0.0
```

Проверка listener:

```bash
kubectl get cm -n dev kafka-broker-configuration -o yaml \
  | grep -Ei 'listener.security.protocol.map|inter.broker.listener.name|listeners='
```

Результат:

```text
advertised.listeners=CLIENT://advertised-address-placeholder:9092,INTERNAL://advertised-address-placeholder:9094
inter.broker.listener.name=INTERNAL
listener.security.protocol.map=CONTROLLER:PLAINTEXT,CLIENT:PLAINTEXT,INTERNAL:PLAINTEXT
listeners=CLIENT://:9092,INTERNAL://:9094
```

---

# 6. Запуск через Helmfile

Перейти в каталог:

```bash
cd kubernetes-templating/helmfile
```

Проверить конфигурацию:

```bash
helmfile lint
```

Посмотреть изменения:

```bash
helmfile diff
```

Применить конфигурацию:

```bash
helmfile apply
```

Проверить Helm releases:

```bash
helm list -n prod
helm list -n dev
```

Проверить Kafka:

```bash
kubectl get sts -n prod
kubectl get pods -n prod

kubectl get sts -n dev
kubectl get pods -n dev
```

Итоговая конфигурация:

```text
PROD
  namespace:       prod
  Kafka:           3.5.2
  brokers:         5
  controllers:     1
  client:          SASL_PLAINTEXT
  inter-broker:    SASL_PLAINTEXT

DEV
  namespace:       dev
  Kafka:           4.0.0
  brokers:         1
  controllers:     1
  client:          PLAINTEXT
  inter-broker:    PLAINTEXT
```

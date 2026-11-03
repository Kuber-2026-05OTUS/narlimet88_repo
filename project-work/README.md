# Дипломный проект OTUS

## Подготовка MVP инфраструктурной платформы для приложения

### Цель проекта

Развернуть инфраструктурную платформу на базе Kubernetes, обеспечивающую автоматическое развертывание приложения, HTTP-маршрутизацию, мониторинг и централизованный сбор логов.

## 1. Архитектура

Кластер Kubernetes развернут с помощью kubeadm.

| Узел | IP | Роль |
|------|----|------|
| k8s-cp1 | 192.168.122.5 | Control Plane |
| k8s-worker1 | 192.168.122.223 | Worker |
| k8s-worker2 | 192.168.122.158 | Worker |
| k8s-worker3 | 192.168.122.69 | Worker |

Версия Kubernetes: v1.35.9.

### Компоненты платформы

| Компонент | Назначение |
|-----------|------------|
| Kubernetes | Оркестрация контейнеров |
| Flannel | Сеть Pod |
| Helm | Управление установкой компонентов |
| Argo CD | GitOps и автоматическая синхронизация |
| Traefik | Gateway API и HTTP-маршрутизация |
| Prometheus | Сбор метрик |
| Grafana | Визуализация метрик и логов |
| Loki | Централизованное хранение логов |
| Grafana Alloy | Сбор логов контейнеров |

## 2. Структура проекта

    project-work/
    ├── application/
    │   ├── namespace.yaml
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   ├── nginx-config.yaml
    │   ├── metrics.yaml
    │   ├── gateway.yaml
    │   └── httproute.yaml
    ├── argocd/
    │   ├── values.yaml
    │   ├── project.yaml
    │   └── application.yaml
    ├── traefik/
    │   └── values.yaml
    ├── monitoring/
    │   └── values.yaml
    ├── loki/
    │   └── values.yaml
    └── alloy/
        └── values.yaml

## 3. Приложение

В namespace `diploma` развернут nginx.

- Deployment с двумя репликами.
- Service типа ClusterIP.
- Gateway и HTTPRoute.
- nginx-prometheus-exporter.
- ServiceMonitor.

### Проверка приложения

    kubectl get pods,svc -n diploma
    kubectl get gateway,httproute -n diploma
    kubectl get servicemonitor -n diploma

### Проверка HTTP-доступа

    curl -I http://192.168.122.223:30080/

Ожидаемый результат: HTTP 200.

## 4. GitOps

Для автоматического развертывания используется Argo CD.

Репозиторий:

https://github.com/Kuber-2026-05OTUS/narlimet88_repo

Ветка: `main`.

Путь приложения: `project-work/application`.

Изменения манифестов автоматически применяются Argo CD.

### Проверка

    kubectl get applications -n argocd

Ожидаемый статус: `Synced / Healthy`.

## 5. Мониторинг

Используется kube-prometheus-stack, включающий Prometheus и Grafana.

Prometheus собирает:

- Метрики всех четырех узлов Kubernetes.
- Метрики компонентов Kubernetes.
- Метрики nginx через nginx-prometheus-exporter.

### Проверенные результаты

| Метрика | Результат |
|---------|-----------|
| Node Exporter | 4/4 |
| nginx-exporter | 2/2 |
| Argo CD | Synced / Healthy |

### Проверка метрик

Количество доступных Node Exporter:

    count(up{job="node-exporter"} == 1)

Количество доступных nginx-exporter:

    count(up{namespace="diploma"} == 1)

Метрика работоспособности nginx:

    nginx_up{namespace="diploma"}

Ожидаемый результат: два значения `1`.

## 6. Централизованный сбор логов

Для сбора логов используются Grafana Alloy и Loki.

Grafana Alloy работает как DaemonSet на трех worker-узлах.

Логи контейнеров собираются через Kubernetes API и отправляются в Loki.

Адрес Loki внутри кластера:

    http://loki.logging.svc.cluster.local:3100

Loki подключен к Grafana как источник данных.

### Проверка логов

В Grafana открыть Explore, выбрать Loki и выполнить запрос:

    {namespace="diploma"}

Для просмотра логов nginx:

    {namespace="diploma", container="nginx"}

Для поиска ошибок:

    {namespace="diploma", container="nginx"} |= "error"

Проверено получение логов с обоих Pod приложения.

## 7. Версии компонентов

| Компонент | Версия |
|-----------|--------|
| Kubernetes | 1.35.9 |
| Argo CD Helm chart | 10.11.7 |
| Traefik Helm chart | 42.3.0 |
| kube-prometheus-stack | 92.17.1 |
| Loki Helm chart | 7.5.0 |
| Grafana Alloy Helm chart | 1.15.0 |

Примечание: при проверке Loki HTTP API запущенный контейнер сообщил версию 3.6.11, хотя Helm указывает APP VERSION 3.6.16.

## 8. Ограничения MVP

- В кластере отсутствует StorageClass.
- Prometheus, Alertmanager и Loki используют временное хранилище.
- При пересоздании Pod данные временного хранилища могут теряться.
- Loki работает в режиме SingleBinary с одной репликой.
- Grafana Alloy развернут на трех worker-узлах.
- Отказоустойчивое хранение данных не реализовано.
- Высокая доступность control-plane не реализована.

## 9. Результат

Реализована MVP инфраструктурной платформы Kubernetes.

Обеспечены:

1. Развертывание приложения через GitOps.
2. HTTP-доступ через Gateway API.
3. Мониторинг Kubernetes и приложения.
4. Централизованный сбор логов.
5. Просмотр метрик и логов через Grafana.

## 10. Online Boutique

В качестве основного приложения используется Google Online Boutique
из проекта microservices-demo.

Приложение развернуто в namespace `online-boutique` через Argo CD.

Состав приложения:
- 12 Kubernetes Deployment.
- 12 Kubernetes Service.
- 11 ServiceAccount.

Манифесты расположены в `project-work/online-boutique/`.

Argo CD Application: `online-boutique`.

### Проверка

    kubectl -n online-boutique get pods,deploy,svc
    kubectl -n argocd get application online-boutique

### HTTP-доступ

Online Boutique доступен через Traefik и Gateway API.

    curl -H 'Host: shop.local' http://192.168.122.223:30080/

Ожидаемый результат: HTTP 200.

Маршрутизация реализована через HTTPRoute с hostname `shop.local`.

## 11. CI/CD

Для CI используется GitHub Actions.

Workflow: `.github/workflows/ci.yaml`.

При изменениях в репозитории выполняется проверка Kubernetes-манифестов
с помощью kubeconform.

Для CD используется Argo CD:
- Отслеживает ветку `main`.
- Автоматически синхронизирует манифесты.
- Использует selfHeal и prune.

## 12. Мониторинг Online Boutique

Для приложения созданы:
- PrometheusRule с алертами.
- Grafana Dashboard через ConfigMap.

Файлы:
- `project-work/online-boutique/alerts.yaml`
- `project-work/online-boutique/dashboard.yaml`

Алерты:
- `OnlineBoutiqueFrontendDown`
- `OnlineBoutiquePodRestarts`

Панели Grafana:
- Frontend Available Replicas.
- Running Pods.
- Container Restarts 15m.

## 13. Автоматизация инфраструктуры

Для автоматизации Kubernetes используется Ansible.

Каталог: `project-work/ansible/`.

Основные Playbook:
- `prepare-nodes.yaml` — подготовка узлов.
- `install-kubernetes.yaml` — установка Kubernetes.
- `configure-containerd.yaml` — настройка containerd.
- `init-control-plane.yaml` — инициализация Control Plane.
- `install-flannel.yaml` — установка Flannel.
- `join-workers.yaml` — подключение Worker Nodes.
- `deploy-cluster.yaml` — общий сценарий развертывания.

Проверены синтаксис и последовательность задач Ansible.
Полное развертывание на чистых узлах пока не тестировалось.

## 14. Дополнительные ограничения

- Публичный HTTP-доступ из интернета не настроен.
- Приложение доступно внутри лабораторной сети.
- Полный Ansible-сценарий не проверен на чистых узлах.
- GitHub Actions выполняет валидацию манифестов, но не сборку образов.

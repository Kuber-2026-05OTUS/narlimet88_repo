# Установка MVP инфраструктурной платформы

## 1. Требования

Перед установкой необходимы:

- Kubernetes v1.35.9 (kubeadm)
- Рабочая сеть Pod (Flannel)
- kubectl с доступом к кластеру
- Helm
- Git
- Доступ к GitHub и Helm-репозиториям

Проверка:

    kubectl get nodes
    helm version

Все узлы должны иметь статус Ready.

## 2. Получение проекта

    git clone https://github.com/Kuber-2026-05OTUS/narlimet88_repo.git
    cd narlimet88_repo

Если репозиторий уже клонирован:

    git pull --ff-only

## 3. Подключение Helm-репозиториев

    helm repo add argo https://argoproj.github.io/argo-helm
    helm repo add traefik https://traefik.github.io/charts
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo add grafana https://grafana.github.io/helm-charts

    helm repo update

## 4. Установка Gateway API CRD

Traefik использует Gateway API.

Устанавливаем experimental CRD версии v1.5.1:

    kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.1/experimental-install.yaml

Проверка:

    kubectl get crd | grep gateway.networking.k8s.io

## 5. Установка Argo CD

    helm upgrade --install argocd argo/argo-cd \
      --version 10.11.7 \
      --namespace argocd \
      --create-namespace \
      -f project-work/argocd/values.yaml \
      --wait \
      --timeout 10m

Проверка:

    kubectl get pods -n argocd

## 6. Установка Traefik

    helm upgrade --install traefik traefik/traefik \
      --version 42.3.0 \
      --namespace traefik \
      --create-namespace \
      -f project-work/traefik/values.yaml \
      --wait \
      --timeout 10m

Проверка:

    kubectl get pods,svc -n traefik
    kubectl get gatewayclass

Ожидаемый результат: GatewayClass traefik Accepted=True.

HTTP NodePort: 30080.

## 7. Установка мониторинга

    helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
      --version 92.17.1 \
      --namespace monitoring \
      --create-namespace \
      -f project-work/monitoring/values.yaml \
      --wait \
      --timeout 10m

Проверка:

    kubectl get pods -n monitoring

## 8. Установка Loki

    helm upgrade --install loki grafana/loki \
      --version 7.5.0 \
      --namespace logging \
      --create-namespace \
      -f project-work/loki/values.yaml \
      --wait \
      --timeout 10m

Проверка:

    kubectl get pods -n logging
    kubectl get svc loki -n logging

## 9. Установка Grafana Alloy

    helm upgrade --install alloy grafana/alloy \
      --version 1.15.0 \
      --namespace logging \
      -f project-work/alloy/values.yaml \
      --wait \
      --timeout 10m

Проверка:

    kubectl get daemonset -n logging

Ожидаемый результат для текущего кластера: Alloy 3/3.

## 10. Развертывание приложения через Argo CD

Создаем AppProject и Application:

    kubectl apply -f project-work/argocd/project.yaml
    kubectl apply -f project-work/argocd/application.yaml

Argo CD автоматически создаст namespace diploma
и применит манифесты из project-work/application.

Проверка:

    kubectl get applications -n argocd
    kubectl get pods,svc -n diploma
    kubectl get gateway,httproute -n diploma

Ожидаемый статус приложения: Synced / Healthy.

### Развертывание Online Boutique

Основное приложение дипломного проекта — Google Online Boutique.

Создаем отдельное Argo CD Application:

    kubectl apply -f project-work/argocd/online-boutique.yaml

Argo CD автоматически создаст namespace online-boutique
и развернет микросервисы из project-work/online-boutique/.

Проверка:

    kubectl get application online-boutique -n argocd
    kubectl get pods,deploy,svc -n online-boutique
    kubectl get httproute -n online-boutique
    kubectl get prometheusrule -n online-boutique

Ожидаемый результат: 12 Deployment, все Pod Running,
Argo CD Application Synced / Healthy.

## 11. Проверка HTTP-доступа

В текущей лаборатории:

    curl -I http://192.168.122.223:30080/

Ожидаемый результат: HTTP 200.

Проверка Online Boutique через Traefik:

    curl -H 'Host: shop.local' http://192.168.122.223:30080/

Ожидаемый результат: HTTP 200.

Hostname shop.local задан в HTTPRoute приложения.
Публичный доступ из интернета не настроен.

При развертывании на других узлах необходимо использовать
доступный IP-адрес узла Kubernetes.

## 12. Доступ к Grafana

    kubectl -n monitoring port-forward \
      svc/monitoring-grafana 3000:80

Открыть с машины, на которой выполняется port-forward:

    http://localhost:3000

Имя пользователя: admin.

Получение пароля:

    kubectl -n monitoring get secret monitoring-grafana \
      -o jsonpath='{.data.admin-password}' | base64 -d
    echo

## 13. Проверка Prometheus

Открыть Prometheus через port-forward:

    kubectl -n monitoring port-forward \
      svc/monitoring-kube-prometheus-prometheus 9090:9090

Адрес:

    http://localhost:9090

Проверка четырех Node Exporter:

    count(up{job="node-exporter"} == 1)

Проверка двух nginx-exporter:

    count(up{namespace="diploma"} == 1)

## 14. Проверка Loki

В Grafana открыть Explore, выбрать источник Loki.

Запрос:

    {namespace="diploma"}

Ожидается получение логов двух экземпляров nginx.

## 15. Общая проверка платформы

    kubectl get nodes
    helm list -A
    kubectl get applications -n argocd
    kubectl get gatewayclass
    kubectl get gateway,httproute -n diploma
    kubectl get pods -A

## 16. Ограничения

Данная конфигурация предназначена для лабораторного MVP.

- Kubernetes-кластер создается отдельно.
- StorageClass отсутствует.
- Prometheus, Alertmanager и Loki используют временное хранилище.
- Loki работает с одной репликой.
- Отказоустойчивое хранение не реализовано.
- IP-адреса и NodePort соответствуют лабораторной среде.

## 17. Автоматизация Kubernetes через Ansible

Playbook расположены в каталоге `project-work/ansible/`.

Проверка доступности узлов:

    ansible k8s_cluster \
      -i project-work/ansible/inventory.ini \
      -m ping

Проверка установленного Kubernetes:

    ansible-playbook \
      -i project-work/ansible/inventory.ini \
      project-work/ansible/check-cluster.yaml

Проверка синтаксиса общего Playbook:

    ansible-playbook \
      -i project-work/ansible/inventory.ini \
      project-work/ansible/deploy-cluster.yaml \
      --syntax-check

Просмотр задач:

    ansible-playbook \
      -i project-work/ansible/inventory.ini \
      project-work/ansible/deploy-cluster.yaml \
      --list-tasks

Для развертывания нового кластера предусмотрен Playbook
`deploy-cluster.yaml`.

Внимание: полный сценарий проверен только на синтаксис.
Не запускайте его на действующем кластере без отдельной проверки.

## 18. Проверка CI/CD

CI реализован через GitHub Actions.

Файл workflow:

    .github/workflows/ci.yaml

При запуске workflow выполняется проверка Kubernetes-манифестов
с помощью kubeconform.

Страница GitHub Actions:

https://github.com/Kuber-2026-05OTUS/narlimet88_repo/actions

CD реализован через Argo CD.

Проверка состояния приложений:

    kubectl get applications -n argocd

Ожидаемый результат:

- diploma-app: Synced / Healthy.
- online-boutique: Synced / Healthy.

## 19. Проверка мониторинга Online Boutique

Проверка правил Prometheus:

    kubectl get prometheusrule -n online-boutique

Проверка Grafana Dashboard:

    kubectl get configmap -n online-boutique \
      -l grafana_dashboard=1

В Grafana открыть Dashboard `Online Boutique`.

Проверить графики:
- Frontend Available Replicas.
- Running Pods.
- Container Restarts 15m.

Логи Online Boutique доступны в Grafana Explore
через источник данных Loki.

LogQL:

    {namespace="online-boutique"}

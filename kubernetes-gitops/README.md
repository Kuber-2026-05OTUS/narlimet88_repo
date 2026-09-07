# Kubernetes GitOps

Домашнее задание по настройке GitOps-развертывания приложений в Kubernetes с использованием ArgoCD.

## Цель

В рамках работы необходимо:

1. Установить ArgoCD в Kubernetes-кластер.
2. Создать ArgoCD Project.
3. Создать Application для развертывания обычных Kubernetes-манифестов из Git.
4. Создать Application для развертывания Helm chart из Git.
5. Для Helm-приложения настроить:
   - автоматическую синхронизацию;
   - `selfHeal`;
   - `prune`;
   - переопределение параметров Helm chart.

## Структура

```text
kubernetes-gitops/
├── README.md
├── argocd-values.yaml
├── helm-app.yaml
├── network-app.yaml
└── project.yaml
```

## Окружение

Для выполнения домашнего задания использовался локальный одновузловой кластер Minikube.

```text
Kubernetes: v1.35.1
Minikube:   v1.38.1
ArgoCD:     v3.5.2
Helm chart: argo-cd 10.8.2
```

В методических материалах предполагается отдельная группа `infra`-нод.

Так как в Minikube используется только одна нода, для адаптации задания нода была помечена label:

```bash
kubectl label node minikube node-role=infra --overwrite
```

Проверка:

```bash
kubectl get nodes --show-labels
```

Также на ноде уже присутствует label:

```text
homework=true
```

который используется приложениями из предыдущих домашних заданий.

Отдельный taint для `infra` не создавался, так как в одновузловом кластере это повлияло бы на все остальные workload'ы.

---

# 1. Установка ArgoCD

Добавлен Helm repository:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

Для установки использовался файл `argocd-values.yaml`.

## argocd-values.yaml

```yaml
global:
  nodeSelector:
    node-role: infra

configs:
  params:
    server.insecure: true

server:
  service:
    type: ClusterIP

controller:
  nodeSelector:
    node-role: infra

repoServer:
  nodeSelector:
    node-role: infra

applicationSet:
  nodeSelector:
    node-role: infra

notifications:
  nodeSelector:
    node-role: infra

redis:
  nodeSelector:
    node-role: infra

dex:
  nodeSelector:
    node-role: infra
```

Перед установкой была выполнена проверка шаблонов:

```bash
helm template argocd argo/argo-cd \
  --version 10.8.2 \
  --namespace argocd \
  -f argocd-values.yaml > /tmp/argocd.yaml
```

Установка ArgoCD:

```bash
helm upgrade --install argocd argo/argo-cd \
  --version 10.8.2 \
  --namespace argocd \
  --create-namespace \
  -f argocd-values.yaml
```

Проверка:

```bash
kubectl get pods -n argocd
```

Все основные компоненты ArgoCD были успешно запущены:

```text
argocd-application-controller
argocd-applicationset-controller
argocd-dex-server
argocd-notifications-controller
argocd-redis
argocd-repo-server
argocd-server
```

---

# 2. Создание AppProject

Для домашних заданий создан отдельный ArgoCD Project `otus`.

## project.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: otus
  namespace: argocd
spec:
  description: OTUS Kubernetes homework project

  sourceRepos:
    - https://github.com/Kuber-2026-05OTUS/narlimet88_repo.git

  destinations:
    - namespace: homework
      server: https://kubernetes.default.svc
    - namespace: homeworkhelm
      server: https://kubernetes.default.svc

  clusterResourceWhitelist:
    - group: '*'
      kind: '*'

  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
```

Применение:

```bash
kubectl apply -f project.yaml
```

Проверка:

```bash
kubectl get appprojects -n argocd
```

```text
NAME      AGE
default   ...
otus      ...
```

---

# 3. Application из Kubernetes-манифестов

Первое приложение устанавливает манифесты из предыдущего домашнего задания:

```text
kubernetes-networks/
```

Приложение использует:

- repository: `narlimet88_repo`;
- branch: `main`;
- path: `kubernetes-networks`;
- namespace: `homework`;
- Project: `otus`;
- ручную синхронизацию.

## network-app.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kubernetes-networks
  namespace: argocd
spec:
  project: otus

  source:
    repoURL: https://github.com/Kuber-2026-05OTUS/narlimet88_repo.git
    targetRevision: main
    path: kubernetes-networks

  destination:
    server: https://kubernetes.default.svc
    namespace: homework

  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

Применение:

```bash
kubectl apply -f network-app.yaml
```

Проверка:

```bash
kubectl get applications -n argocd
```

Изначально приложение было `OutOfSync`.

Ручная синхронизация была запущена через Kubernetes API:

```bash
kubectl patch application kubernetes-networks -n argocd \
  --type merge \
  -p '{"operation":{"sync":{"syncStrategy":{"apply":{"force":false}}}}}'
```

## Конфликт со старым Deployment

В namespace `homework` уже существовал Deployment из предыдущего Helm-домашнего задания.

Он содержал другой immutable selector:

```text
spec.selector: field is immutable
```

Также Deployment принадлежал Helm release:

```text
meta.helm.sh/release-name: homework
meta.helm.sh/release-namespace: homework
```

ArgoCD не мог изменить immutable поле существующего Deployment.

Для устранения конфликта был удален только старый Deployment:

```bash
kubectl delete deployment homework -n homework
```

После этого была повторно выполнена ручная синхронизация.

ArgoCD создал Deployment заново из Git:

```text
Deployment homework/homework -> Synced : deployment.apps/homework created
```

Проверка ресурсов Application:

```bash
kubectl get application kubernetes-networks -n argocd \
  -o jsonpath='{range .status.resources[*]}{.kind}{" "}{.namespace}{"/"}{.name}{" -> "}{.status}{"\n"}{end}'
```

Результат:

```text
Namespace /homework -> Synced
Service homework/homework -> Synced
Deployment homework/homework -> Synced
Gateway homework/homework-gateway -> Synced
HTTPRoute homework/homework -> Synced
```

Итоговый статус:

```bash
kubectl get application kubernetes-networks -n argocd \
  -o custom-columns='NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status'
```

```text
NAME                  SYNC     HEALTH
kubernetes-networks   Synced   Healthy
```

Deployment:

```bash
kubectl get deployment homework -n homework
```

```text
NAME       READY   UP-TO-DATE   AVAILABLE
homework   3/3     3            3
```

Таким образом первое приложение разворачивается ArgoCD из обычных Kubernetes-манифестов, расположенных в Git repository.

---

# 4. Application из Helm chart

Второе приложение использует Helm chart из предыдущего домашнего задания:

```text
kubernetes-templating/homework
```

Chart поддерживает установку в произвольный namespace через:

```text
.Release.Namespace
```

Проверка:

```bash
grep -R "namespace:" \
  ../kubernetes-templating/homework/templates
```

Пример:

```text
namespace: {{ .Release.Namespace }}
```

Дополнительно chart был локально отрендерен для нового namespace:

```bash
helm template homework-helm \
  ../kubernetes-templating/homework \
  --namespace homeworkhelm \
  --set replicaCount=2
```

Шаблонизация завершилась успешно.

Для второго Application был создан отдельный namespace:

```text
homeworkhelm
```

Название используется в lowercase, так как Kubernetes namespace должен соответствовать требованиям DNS/RFC 1123.

---

## helm-app.yaml

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kubernetes-templating
  namespace: argocd
spec:
  project: otus

  source:
    repoURL: https://github.com/Kuber-2026-05OTUS/narlimet88_repo.git
    targetRevision: main
    path: kubernetes-templating/homework
    helm:
      parameters:
        - name: replicaCount
          value: "2"

  destination:
    server: https://kubernetes.default.svc
    namespace: homeworkhelm

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Применение:

```bash
kubectl apply -f helm-app.yaml
```

Здесь настроены:

```yaml
automated:
  prune: true
  selfHeal: true
```

Таким образом приложение автоматически синхронизируется с Git и восстанавливает desired state при ручном изменении ресурсов.

Также значение:

```yaml
replicaCount: 2
```

переопределяется непосредственно в ArgoCD Application.

---

# 5. Проверка Helm Application

После создания Application ArgoCD автоматически:

1. Получил repository.
2. Перешел в каталог Helm chart.
3. Выполнил Helm rendering.
4. Использовал:

```text
--namespace homeworkhelm
--set replicaCount=2
```

5. Создал namespace.
6. Создал ресурсы Helm chart.

Проверка:

```bash
kubectl get pods -n homeworkhelm
```

Результат:

```text
NAME                                   READY   STATUS
homework-68cd8f9cff-...                1/1     Running
homework-68cd8f9cff-...                1/1     Running
kubernetes-templating-redis-master-0   1/1     Running
```

Deployment:

```bash
kubectl get deployment homework -n homeworkhelm
```

```text
NAME       READY   UP-TO-DATE   AVAILABLE
homework   2/2     2            2
```

Redis:

```bash
kubectl get statefulset -n homeworkhelm
```

```text
NAME                                 READY
kubernetes-templating-redis-master   1/1
```

PVC:

```bash
kubectl get pvc -n homeworkhelm
```

```text
NAME       STATUS   CAPACITY   ACCESS MODES   STORAGECLASS
homework   Bound    1Gi        RWO            standard
```

Статус ArgoCD:

```bash
kubectl get application kubernetes-templating -n argocd \
  -o custom-columns='NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status'
```

```text
NAME                    SYNC     HEALTH
kubernetes-templating   Synced   Healthy
```

---

# 6. Проверка Self Heal

Для проверки `selfHeal` количество реплик Deployment было вручную изменено с двух до одной:

```bash
kubectl scale deployment homework \
  -n homeworkhelm \
  --replicas=1
```

После ручного изменения ArgoCD обнаружил отличие live state от desired state.

Application перешел в:

```text
OutOfSync
```

При этом в Application настроено:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

Проверка конфигурации:

```bash
kubectl get application kubernetes-templating -n argocd \
  -o jsonpath='{.spec.syncPolicy}{"\n"}'
```

Результат:

```json
{"automated":{"prune":true,"selfHeal":true},"syncOptions":["CreateNamespace=true"]}
```

ArgoCD автоматически запустил новую операцию синхронизации:

```text
operationState.phase: Running
```

После reconcile Deployment был автоматически восстановлен до значения из GitOps-конфигурации:

```bash
kubectl get deployment homework -n homeworkhelm
```

```text
NAME       READY   UP-TO-DATE   AVAILABLE
homework   2/2     2            2
```

Application снова перешел в:

```text
Synced / Healthy
```

Проверка:

```bash
kubectl get application kubernetes-templating -n argocd \
  -o custom-columns='NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status,PHASE:.status.operationState.phase'
```

Результат:

```text
NAME                    SYNC     HEALTH    PHASE
kubernetes-templating   Synced   Healthy   Succeeded
```

Таким образом работа `selfHeal` была проверена практически.

---

# 7. Итоговое состояние Applications

Проверка:

```bash
kubectl get applications -n argocd
```

Оба приложения находятся под управлением ArgoCD.

Первое приложение:

```text
kubernetes-networks
```

использует обычные Kubernetes-манифесты и ручную синхронизацию.

Второе приложение:

```text
kubernetes-templating
```

использует Helm chart и автоматическую GitOps-синхронизацию.

Итог:

```text
kubernetes-networks     Synced   Healthy
kubernetes-templating   Synced   Healthy
```

---

# 8. Доступ к Web UI ArgoCD

ArgoCD Server установлен как `ClusterIP`.

Для доступа к интерфейсу можно использовать port-forward:

```bash
kubectl port-forward \
  --address=0.0.0.0 \
  -n argocd \
  svc/argocd-server \
  8080:80
```

После этого интерфейс доступен по адресу:

```text
http://<IP Kubernetes host>:8080
```

Начальный пароль пользователя `admin` можно получить командой:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d
echo
```

---

# 9. Итог

В рамках домашнего задания:

- установлен ArgoCD;
- компоненты ArgoCD размещены на ноде с label `node-role=infra`;
- создан ArgoCD Project `otus`;
- Git repository разрешен в качестве source;
- настроено приложение из Kubernetes-манифестов;
- для первого приложения используется Manual Sync;
- приложение успешно развернуто в namespace `homework`;
- настроено приложение из Helm chart;
- Helm chart хранится в том же Git repository;
- второе приложение развернуто в отдельный namespace `homeworkhelm`;
- количество реплик переопределено через ArgoCD до `2`;
- включен Automatic Sync;
- включен `prune`;
- включен `selfHeal`;
- работа `selfHeal` проверена путем ручного изменения количества реплик;
- ArgoCD автоматически восстановил Deployment до desired state;
- оба Application находятся в состоянии `Synced / Healthy`.


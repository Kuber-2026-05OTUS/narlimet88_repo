# Kubernetes Security

Домашнее задание по теме:

`ServiceAccount и RBAC`

В рамках задания реализованы:

- ServiceAccount `monitoring`;
- доступ ServiceAccount `monitoring` к endpoint `/metrics`;
- запуск Deployment под ServiceAccount `monitoring`;
- ServiceAccount `cd`;
- выдача ServiceAccount `cd` роли `admin` в namespace `homework`;
- генерация токена ServiceAccount `cd` сроком на 24 часа;
- создание kubeconfig для ServiceAccount `cd`;
- проверка ограничения прав между namespace.

## Namespace

Все основные ресурсы приложения находятся в namespace:

```text
homework
```

## ServiceAccount monitoring

Создан ServiceAccount:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitoring
  namespace: homework
```

Применение:

```bash
kubectl apply -f monitoring-sa.yaml
```

## RBAC для monitoring

Для доступа к endpoint Kubernetes API `/metrics` создан ClusterRole:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
rules:
  - nonResourceURLs:
      - /metrics
    verbs:
      - get
```

И ClusterRoleBinding:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring

subjects:
  - kind: ServiceAccount
    name: monitoring
    namespace: homework

roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: monitoring
```

Применение:

```bash
kubectl apply -f monitoring-rbac.yaml
```

Проверка доступа:

```bash
kubectl auth can-i get /metrics \
  --as=system:serviceaccount:homework:monitoring
```

Ожидаемый результат:

```text
yes
```

Проверка отсутствия лишних прав:

```bash
kubectl auth can-i get pods \
  -n homework \
  --as=system:serviceaccount:homework:monitoring
```

Ожидаемый результат:

```text
no
```

## Deployment

Deployment изменён таким образом, чтобы Pod запускался под ServiceAccount `monitoring`.

В Pod template добавлено:

```yaml
spec:
  serviceAccountName: monitoring
```

Проверка:

```bash
kubectl get pods -n homework \
  -l app=homework \
  -o custom-columns=NAME:.metadata.name,SA:.spec.serviceAccountName
```

Ожидаемо все Pod используют:

```text
monitoring
```

Также можно проверить отдельный Pod:

```bash
POD=$(kubectl get pods -n homework \
  -l app=homework \
  -o jsonpath='{.items[0].metadata.name}')

kubectl get pod "$POD" -n homework \
  -o jsonpath='{.spec.serviceAccountName}'; echo
```

Ожидаемый результат:

```text
monitoring
```

## ServiceAccount cd

Создан ServiceAccount:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cd
  namespace: homework
```

Применение:

```bash
kubectl apply -f cd-sa.yaml
```

## Права admin для cd

ServiceAccount `cd` получил встроенную ClusterRole `admin`, ограниченную namespace `homework` через RoleBinding.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cd-admin
  namespace: homework

subjects:
  - kind: ServiceAccount
    name: cd
    namespace: homework

roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin
```

Применение:

```bash
kubectl apply -f cd-rolebinding.yaml
```

Проверка прав внутри namespace `homework`:

```bash
kubectl auth can-i get pods \
  -n homework \
  --as=system:serviceaccount:homework:cd

kubectl auth can-i create deployments.apps \
  -n homework \
  --as=system:serviceaccount:homework:cd

kubectl auth can-i delete services \
  -n homework \
  --as=system:serviceaccount:homework:cd
```

Ожидаемый результат:

```text
yes
yes
yes
```

Проверка отсутствия аналогичных прав в namespace `default`:

```bash
kubectl auth can-i get pods \
  -n default \
  --as=system:serviceaccount:homework:cd
```

Ожидаемый результат:

```text
no
```

Полный список разрешений:

```bash
kubectl auth can-i --list \
  -n homework \
  --as=system:serviceaccount:homework:cd
```

## Генерация токена

Для ServiceAccount `cd` генерируется токен сроком на 24 часа:

```bash
kubectl create token cd \
  -n homework \
  --duration=24h > token
```

Проверка:

```bash
ls -lh token
wc -c token
```

Файл `token` не добавляется в Git, так как содержит действующие credentials.

## Создание kubeconfig

Для ServiceAccount `cd` создаётся отдельный kubeconfig.

Получение параметров текущего кластера:

```bash
CLUSTER=$(kubectl config view --minify \
  -o jsonpath='{.contexts[0].context.cluster}')

SERVER=$(kubectl config view --minify \
  -o jsonpath='{.clusters[0].cluster.server}')

CA_FILE=$(kubectl config view --raw --minify \
  -o jsonpath='{.clusters[0].cluster.certificate-authority}')

CA_DATA=$(kubectl config view --raw --minify \
  -o jsonpath='{.clusters[0].cluster.certificate-authority-data}')
```

Получение CA:

```bash
if [ -n "$CA_DATA" ]; then
    CA_B64="$CA_DATA"
elif [ -n "$CA_FILE" ]; then
    CA_B64=$(base64 -w0 "$CA_FILE")
fi
```

Создание kubeconfig:

```bash
TOKEN=$(cat token)

cat > cd.kubeconfig <<EOF
apiVersion: v1
kind: Config

clusters:
- name: ${CLUSTER}
  cluster:
    server: ${SERVER}
    certificate-authority-data: ${CA_B64}

contexts:
- name: cd
  context:
    cluster: ${CLUSTER}
    namespace: homework
    user: cd

current-context: cd

users:
- name: cd
  user:
    token: ${TOKEN}
EOF
```

После создания:

```bash
unset TOKEN
unset CA_B64
unset CA_DATA
unset CA_FILE
```

## Проверка kubeconfig

Проверка подключения:

```bash
kubectl --kubeconfig=cd.kubeconfig get pods
```

Проверка прав:

```bash
kubectl --kubeconfig=cd.kubeconfig auth can-i get pods
kubectl --kubeconfig=cd.kubeconfig auth can-i create deployments
kubectl --kubeconfig=cd.kubeconfig auth can-i delete services
```

Ожидаемый результат:

```text
yes
yes
yes
```

Проверка namespace `default`:

```bash
kubectl --kubeconfig=cd.kubeconfig \
  auth can-i get pods -n default
```

Ожидаемый результат:

```text
no
```

Это подтверждает, что ServiceAccount `cd` имеет права `admin` в namespace `homework`, но не получает их в других namespace.

## Безопасное хранение credentials

Файлы:

```text
token
cd.kubeconfig
```

содержат рабочие credentials и поэтому исключены из Git.

`.gitignore`:

```gitignore
token
cd.kubeconfig
```

В репозитории хранится только безопасный пример:

```text
cd.kubeconfig.example
```

В нём токен заменён на:

```text
<SERVICE_ACCOUNT_TOKEN>
```

## Применение ресурсов

```bash
kubectl apply -f namespace.yaml
kubectl apply -f monitoring-sa.yaml
kubectl apply -f monitoring-rbac.yaml
kubectl apply -f cd-sa.yaml
kubectl apply -f cd-rolebinding.yaml

kubectl apply -f pvc.yaml
kubectl apply -f cm.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f gateway.yaml
kubectl apply -f httpRoute.yaml
```

## Финальная проверка

```bash
kubectl get sa -n homework
kubectl get clusterrole monitoring
kubectl get clusterrolebinding monitoring
kubectl get rolebinding cd-admin -n homework
kubectl get pods -n homework
```

Проверка ServiceAccount Deployment:

```bash
kubectl get pods -n homework \
  -l app=homework \
  -o custom-columns=NAME:.metadata.name,SA:.spec.serviceAccountName
```

Проверка `monitoring`:

```bash
kubectl auth can-i get /metrics \
  --as=system:serviceaccount:homework:monitoring
```

Проверка `cd`:

```bash
kubectl --kubeconfig=cd.kubeconfig auth can-i get pods
kubectl --kubeconfig=cd.kubeconfig auth can-i create deployments
kubectl --kubeconfig=cd.kubeconfig auth can-i get pods -n default
```

Ожидаемый результат:

```text
yes
yes
no
```

## Результат

В результате выполнения задания:

- создан ServiceAccount `monitoring`;
- `monitoring` получил доступ к `/metrics`;
- лишние права на Pod для `monitoring` отсутствуют;
- Deployment запускается под ServiceAccount `monitoring`;
- создан ServiceAccount `cd`;
- `cd` получил роль `admin` в namespace `homework`;
- права `cd` не распространяются на namespace `default`;
- создан токен ServiceAccount `cd` сроком на 24 часа;
- создан рабочий kubeconfig для ServiceAccount `cd`;
- проверена работа RBAC и namespace-scoped доступа.

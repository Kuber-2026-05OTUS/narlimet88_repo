# Kubernetes Operators

Домашнее задание по работе с Custom Resource Definition и Kubernetes Operator.

## Структура

```text
kubernetes-operators/
├── README.md
├── crd.yaml
├── mysql.yaml
├── operator-deployment.yaml
├── rbac.yaml
└── serviceaccount.yaml
```

## 1. Custom Resource Definition

Создан namespaced CRD со следующими параметрами:

- API Group: `otus.homework`
- Version: `v1`
- Kind: `MySQL`
- Plural: `mysqls`

CRD содержит обязательные поля:

- `image` — Docker image для MySQL;
- `database` — имя базы данных;
- `password` — пароль;
- `storage_size` — размер хранилища.

Применение CRD:

```bash
kubectl apply -f crd.yaml
```

Проверка:

```bash
kubectl get crd mysqls.otus.homework
kubectl api-resources | grep -i mysql
```

Результат:

```text
NAME                   CREATED AT
mysqls.otus.homework   ...

mysqls   mysql   otus.homework/v1   true   MySQL
```

## 2. ServiceAccount и RBAC

Для оператора создан ServiceAccount:

```text
mysql-operator
```

Также созданы:

- `ClusterRole mysql-operator`;
- `ClusterRoleBinding mysql-operator`.

ClusterRole предоставляет оператору полные права для работы с Kubernetes API.

Применение:

```bash
kubectl apply -f serviceaccount.yaml
kubectl apply -f rbac.yaml
```

Проверка:

```bash
kubectl get sa mysql-operator
kubectl get clusterrole mysql-operator
kubectl get clusterrolebinding mysql-operator

kubectl auth can-i \
  --as=system:serviceaccount:default:mysql-operator \
  '*' '*'
```

Результат:

```text
yes
```

## 3. MySQL Operator

Оператор запускается с использованием образа:

```text
roflmaoinmysoul/mysql-operator:1.0.0
```

Deployment использует ServiceAccount `mysql-operator`.

Применение:

```bash
kubectl apply -f operator-deployment.yaml
```

Проверка:

```bash
kubectl get deployment mysql-operator
kubectl get pods -l app=mysql-operator
```

Результат:

```text
NAME             READY   UP-TO-DATE   AVAILABLE
mysql-operator   1/1     1            1
```

Pod оператора:

```text
READY   STATUS    RESTARTS
1/1     Running   0
```

Проверка логов:

```bash
kubectl logs -l app=mysql-operator --tail=100
```

Оператор успешно проходит аутентификацию в Kubernetes API.

## 4. Custom Resource MySQL

Создан объект:

```yaml
apiVersion: otus.homework/v1
kind: MySQL
metadata:
  name: mysql
  namespace: default
spec:
  image: mysql:5.7
  database: otus
  password: otuspassword
  storage_size: 1Gi
```

Используется `mysql:5.7`, так как среда Minikube не предоставляет CPU-инструкции `x86-64-v2`, необходимые использованному при первоначальной проверке образу `mysql:8.0`.

Применение:

```bash
kubectl apply -f mysql.yaml
```

Проверка:

```bash
kubectl get mysql
kubectl get pods -l app=mysql
```

MySQL успешно запущен:

```text
NAME                     READY   STATUS    RESTARTS
mysql-6b9bb5c666-wtm7c   1/1     Running   0
```

## 5. Ресурсы, созданные оператором

После создания Custom Resource оператор автоматически создаёт:

- Deployment;
- Service;
- PersistentVolume;
- PersistentVolumeClaim.

Проверка Deployment:

```bash
kubectl get deployment mysql
```

Результат:

```text
NAME    READY   UP-TO-DATE   AVAILABLE
mysql   1/1     1            1
```

Проверка Service:

```bash
kubectl get svc mysql
```

Результат:

```text
NAME    TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)
mysql   ClusterIP   None         <none>        3306/TCP
```

Проверка PVC:

```bash
kubectl get pvc mysql-pvc
```

Результат:

```text
NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS
mysql-pvc   Bound    mysql-pv   1Gi        RWO            standard
```

Проверка PV:

```bash
kubectl get pv mysql-pv
```

Результат:

```text
NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM
mysql-pv   1Gi        RWO            Retain           Bound    default/mysql-pvc
```

В логах оператора создание ресурсов завершилось успешно:

```text
MySQL instance mysql and its children resources created!
Handler 'mysql_on_create' succeeded.
Creation is processed: 1 succeeded; 0 failed.
```

## 6. Проверка удаления Custom Resource

Удаляем объект MySQL:

```bash
kubectl delete -f mysql.yaml
```

Результат:

```text
mysql.otus.homework "mysql" deleted from default namespace
```

После удаления Custom Resource проверяем созданные оператором ресурсы:

```bash
kubectl get mysql
kubectl get deployment mysql
kubectl get svc mysql
kubectl get pvc mysql-pvc
kubectl get pv mysql-pv
```

Результат:

```text
No resources found in default namespace.

Error from server (NotFound): deployments.apps "mysql" not found
Error from server (NotFound): services "mysql" not found
Error from server (NotFound): persistentvolumeclaims "mysql-pvc" not found
Error from server (NotFound): persistentvolumes "mysql-pv" not found
```

Таким образом, при удалении Custom Resource оператор автоматически удаляет все созданные для него зависимые ресурсы.

В логах оператора:

```text
MySQL instance mysql and its children resources deleted!
Handler 'delete_object_make_backup' succeeded.
Deletion is processed: 1 succeeded; 0 failed.
```

## Итог

В рамках домашнего задания:

- создан namespaced CRD `mysqls.otus.homework`;
- зарегистрирован API `otus.homework/v1`;
- создан Custom Resource `MySQL`;
- реализованы обязательные поля `image`, `database`, `password`, `storage_size`;
- создан ServiceAccount для оператора;
- настроены ClusterRole и ClusterRoleBinding;
- оператор запущен с ServiceAccount `mysql-operator`;
- используется образ оператора `roflmaoinmysoul/mysql-operator:1.0.0`;
- создание объекта `MySQL` автоматически создаёт Deployment, Service, PV и PVC;
- MySQL успешно запускается;
- удаление объекта `MySQL` автоматически удаляет связанные Deployment, Service, PV и PVC.

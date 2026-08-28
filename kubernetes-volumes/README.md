# Kubernetes Volumes

Домашнее задание по теме:

`Volumes, StorageClass, PV, PVC`

В рамках задания реализованы:

- PersistentVolumeClaim;
- динамическое создание PersistentVolume через StorageClass по умолчанию;
- подключение PVC к Deployment;
- ConfigMap;
- монтирование ConfigMap как volume;
- доступ к файлу ConfigMap по HTTP URL `/conf/file`.

## Структура

```text
kubernetes-volumes/
├── README.md
├── namespace.yaml
├── deployment.yaml
├── service.yaml
├── gateway.yaml
├── httpRoute.yaml
├── pvc.yaml
└── cm.yaml
```

## Namespace

Все ресурсы приложения создаются в namespace:

```text
homework
```

Применение:

```bash
kubectl apply -f namespace.yaml
```

Проверка:

```bash
kubectl get namespace homework
```

## StorageClass

В Minikube используется StorageClass по умолчанию:

```bash
kubectl get sc
```

Пример:

```text
NAME                 PROVISIONER                RECLAIMPOLICY
standard (default)   k8s.io/minikube-hostpath   Delete
```

PVC использует StorageClass по умолчанию.

В `pvc.yaml` поле `storageClassName` не задаётся, поэтому Kubernetes автоматически выбирает default StorageClass.

## PersistentVolumeClaim

Создаётся PersistentVolumeClaim:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: homework
  namespace: homework

spec:
  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 1Gi
```

Применение:

```bash
kubectl apply -f pvc.yaml
```

Проверка PVC:

```bash
kubectl get pvc -n homework
```

Ожидаемый статус:

```text
STATUS
Bound
```

Проверка созданного PersistentVolume:

```bash
kubectl get pv
```

При использовании default StorageClass Minikube PV создаётся автоматически через provisioner:

```text
k8s.io/minikube-hostpath
```

## ConfigMap

Создаётся ConfigMap с файлом `file`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: homework
  namespace: homework

data:
  file: |
    hello from configmap
```

Применение:

```bash
kubectl apply -f cm.yaml
```

Проверка:

```bash
kubectl get configmap homework -n homework
kubectl describe configmap homework -n homework
```

## Deployment

Deployment запускает три реплики приложения.

Ранее используемый volume типа `emptyDir` заменён на PersistentVolumeClaim:

```yaml
volumes:
  - name: shared-data
    persistentVolumeClaim:
      claimName: homework
```

Этот volume используется одновременно init container и основным контейнером.

Init container монтирует его в:

```text
/init
```

Основной контейнер монтирует тот же volume в:

```text
/homework
```

Init container создаёт файл:

```text
/index.html
```

Основной контейнер запускает HTTP server на порту:

```text
8000
```

## Подключение ConfigMap

ConfigMap подключается как отдельный volume:

```yaml
volumes:
  - name: config
    configMap:
      name: homework
```

В основном контейнере он монтируется в:

```yaml
volumeMounts:
  - name: config
    mountPath: /homework/conf
```

В результате ключ ConfigMap:

```text
file
```

становится файлом:

```text
/homework/conf/file
```

## Readiness Probe

Для проверки готовности используется HTTP readiness probe:

```yaml
readinessProbe:
  httpGet:
    path: /index.html
    port: 8000
  initialDelaySeconds: 1
  periodSeconds: 5
```

## Применение Deployment

Проверка манифеста:

```bash
kubectl apply --dry-run=client -f deployment.yaml
```

Применение:

```bash
kubectl apply -f deployment.yaml
```

Проверка rollout:

```bash
kubectl rollout status deployment/homework -n homework
```

Проверка Pod:

```bash
kubectl get pods -n homework
```

Ожидаемый результат:

```text
READY   STATUS
1/1     Running
1/1     Running
1/1     Running
```

## Проверка подключённых volumes

Проверить volumes Deployment можно командой:

```bash
kubectl get deployment homework -n homework \
  -o jsonpath='{.spec.template.spec.volumes}' ; echo
```

Пример результата:

```text
[
  {
    "name":"shared-data",
    "persistentVolumeClaim":{
      "claimName":"homework"
    }
  },
  {
    "name":"config",
    "configMap":{
      "name":"homework"
    }
  }
]
```

Это подтверждает, что вместо `emptyDir` используется PVC, а ConfigMap подключён как отдельный volume.

## Проверка ConfigMap внутри Pod

Получим имя одного из Pod:

```bash
POD=$(kubectl get pods -n homework \
  -l app=homework \
  -o jsonpath='{.items[0].metadata.name}')
```

Проверяем файл:

```bash
kubectl exec -n homework $POD -- \
  cat /homework/conf/file
```

Ожидаемый результат:

```text
hello from configmap
```

## Проверка по HTTP

Так как HTTP server раздаёт директорию `/homework`, файл:

```text
/homework/conf/file
```

доступен по URL:

```text
/conf/file
```

Проверка внутри Pod:

```bash
kubectl exec -n homework $POD -- \
  wget -qO- http://127.0.0.1:8000/conf/file
```

Ожидаемый результат:

```text
hello from configmap
```

## Проверка через Gateway

Из предыдущего домашнего задания сохранены:

```text
service.yaml
gateway.yaml
httpRoute.yaml
```

Поэтому приложение также доступно через:

```text
homework.otus
```

Проверка:

```bash
curl http://homework.otus/conf/file
```

Ожидаемый результат:

```text
hello from configmap
```

Проверка основной страницы:

```bash
curl http://homework.otus/index.html
```

Ожидаемый результат:

```html
<html><body><h1>Hello from Kubernetes homework!</h1></body></html>
```

## Проверка PVC и PV

PVC:

```bash
kubectl get pvc -n homework
```

PV:

```bash
kubectl get pv
```

PVC должен находиться в состоянии:

```text
Bound
```

и быть связан с автоматически созданным PersistentVolume.

## Применение всех манифестов

```bash
kubectl apply -f namespace.yaml
kubectl apply -f pvc.yaml
kubectl apply -f cm.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f gateway.yaml
kubectl apply -f httpRoute.yaml
```

## Финальная проверка

```bash
kubectl get pods -n homework
kubectl get pvc -n homework
kubectl get pv
kubectl get configmap -n homework
kubectl get svc -n homework
kubectl get gateway -n homework
kubectl get httproute -n homework
```

Проверка содержимого ConfigMap:

```bash
curl http://homework.otus/conf/file
```

Ожидаемый ответ:

```text
hello from configmap
```

## Результат

В результате выполнения задания:

- создан PVC размером `1Gi`;
- PVC использует StorageClass по умолчанию;
- PersistentVolume создаётся автоматически;
- volume типа `emptyDir` заменён на PVC;
- init container и основной container используют persistent volume;
- создан ConfigMap;
- ConfigMap смонтирован в `/homework/conf`;
- файл `/homework/conf/file` доступен внутри контейнера;
- содержимое ConfigMap доступно по URL `/conf/file`;
- приложение продолжает работать через Service и Gateway API.

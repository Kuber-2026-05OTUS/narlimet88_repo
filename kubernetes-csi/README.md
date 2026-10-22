# Домашнее задание: Kubernetes CSI

## Цель

Развернуть S3-совместимое хранилище и подключить его к Kubernetes
через CSI-драйвер с динамическим выделением PersistentVolume.

## Окружение

- Kubernetes: Minikube v1.35.1
- S3-хранилище: MinIO
- CSI-драйвер: yandex-cloud/k8s-csi-s3 v0.43.7
- Mounter: GeeseFS
- StorageClass: csi-s3
- Bucket: otus-csi

Вместо Yandex Object Storage используется локальное S3-совместимое
хранилище MinIO.

## Развертывание

### 1. MinIO

Создать namespace:

```bash
kubectl create namespace minio
```

Создать Secret `minio-credentials` в namespace `minio`
с ключами `rootUser` и `rootPassword`.

Применить манифест:

```bash
kubectl apply -f minio.yaml
kubectl get pods,pvc -n minio
```

MinIO использует PVC `minio-data` размером 5Gi.

S3 API доступен внутри кластера:

`http://minio.minio.svc.cluster.local:9000`

Создать бакет `otus-csi` с помощью MinIO Client (`mc`).

### 2. CSI-драйвер

Репозиторий:

https://github.com/yandex-cloud/k8s-csi-s3

Установка:

```bash
git clone https://github.com/yandex-cloud/k8s-csi-s3.git
cd k8s-csi-s3/deploy/kubernetes

kubectl apply -f driver.yaml
kubectl apply -f csi-s3.yaml
kubectl apply -f provisioner.yaml
```

Создать Secret `csi-s3-secret` в namespace `kube-system`
со следующими ключами:

- `accessKeyID`
- `secretAccessKey`
- `endpoint`

Endpoint: `http://minio.minio.svc.cluster.local:9000`

Действующие ключи доступа не хранятся в Git.

### 3. StorageClass, PVC и Pod

Из каталога домашнего задания:

```bash
kubectl apply -f storageclass.yaml
kubectl apply -f pvc.yaml
kubectl apply -f pod.yaml
```

StorageClass использует:

- Provisioner: `ru.yandex.s3.csi`
- Mounter: `geesefs`
- Bucket: `otus-csi`

PVC `csi-s3-pvc` запрашивает 1Gi с режимом доступа ReadWriteMany.

Pod `csi-s3-test` монтирует PVC в `/data`
и записывает файл `/data/test.txt`.

## Проверка

Проверка компонентов CSI:

```bash
kubectl get csidrivers
kubectl get pods -n kube-system | grep csi
```

Проверка PVC:

```bash
kubectl get pvc csi-s3-pvc
kubectl get pv
```

Результат: PVC перешёл в статус `Bound`,
PV создан динамически драйвером `ru.yandex.s3.csi`.

Чтение файла из Pod:

```bash
kubectl exec csi-s3-test -- cat /data/test.txt
```

Результат:

```text
Hello from Kubernetes S3 CSI!
```

Проверка объекта в MinIO:

```bash
kubectl exec -n minio deployment/minio -- sh -c '
  mc alias set local http://127.0.0.1:9000 \
    "$MINIO_ROOT_USER" "$MINIO_ROOT_PASSWORD" >/dev/null &&
  mc ls --recursive local/otus-csi
'
```

Обнаружен объект:

```text
pvc-7593d7e3-470f-4afa-a960-37352c85a9f7/test.txt
```

## Результат

Реализована рабочая цепочка:

Pod → PVC → PV → CSI S3 → GeeseFS → MinIO

Подтверждены динамическое выделение PV, запись файла
через смонтированный том и наличие объекта в S3-хранилище.

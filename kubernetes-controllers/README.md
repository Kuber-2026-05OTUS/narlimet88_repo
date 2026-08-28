# Kubernetes Controllers Homework

Домашнее задание по настройке Kubernetes controllers, Deployment, стратегии обновления и проверок работоспособности приложения.

## Что реализовано

- Namespace `homework`
- Deployment в namespace `homework`
- 3 реплики приложения
- Init-контейнер, создающий файл `index.html`
- Общий том `emptyDir`
- HTTP-сервер на порту `8000`
- Readiness probe, проверяющая наличие `/homework/index.html`
- Стратегия обновления `RollingUpdate`
- Максимум 1 недоступный Pod во время обновления (`maxUnavailable: 1`)
- Допускается создание 1 дополнительного Pod во время обновления (`maxSurge: 1`)
- Pod запускаются только на нодах с меткой `homework=true`

## Запуск

Создать namespace:

```bash
kubectl apply -f namespace.yaml
```

Добавить метку `homework=true` на ноду Minikube:

```bash
kubectl label nodes minikube homework=true
```

Проверить наличие метки:

```bash
kubectl get nodes -L homework
```

Создать Deployment:

```bash
kubectl apply -f deployment.yaml
```

## Проверка Deployment

```bash
kubectl get deployments -n homework
```

Ожидаемый результат:

```text
NAME       READY   UP-TO-DATE   AVAILABLE
homework   3/3     3            3
```

Проверить запущенные Pod:

```bash
kubectl get pods -n homework -o wide
```

Должно быть запущено 3 Pod в состоянии `Running` и `Ready`.

## Проверка init-контейнера

Получить имя одного из Pod:

```bash
kubectl get pods -n homework
```

Посмотреть подробную информацию:

```bash
kubectl describe pod -n homework <pod-name>
```

Init-контейнер должен завершиться успешно:

```text
State:      Terminated
Reason:     Completed
Exit Code:  0
```

## Проверка readiness probe

Readiness probe проверяет наличие файла:

```text
/homework/index.html
```

Проверить конфигурацию Pod:

```bash
kubectl describe pod -n homework <pod-name>
```

В выводе должна присутствовать проверка:

```text
Readiness: exec [test -f /homework/index.html]
```

При наличии файла Pod находится в состоянии `Ready`.

## Проверка общего тома

Проверить наличие файла, созданного init-контейнером:

```bash
kubectl exec -n homework <pod-name> -- ls -la /homework
```

Проверить содержимое:

```bash
kubectl exec -n homework <pod-name> -- cat /homework/index.html
```

## Проверка RollingUpdate

Изменить версию образа:

```bash
kubectl set image deployment/homework \
  web=python:3.13-alpine \
  -n homework
```

Наблюдать за обновлением Pod:

```bash
kubectl get pods -n homework -w
```

Проверить завершение обновления:

```bash
kubectl rollout status deployment/homework -n homework
```

Ожидаемый результат:

```text
deployment "homework" successfully rolled out
```

Проверить ReplicaSet:

```bash
kubectl get rs -n homework
```

После успешного обновления новый ReplicaSet должен иметь 3 реплики, а старый — 0.

Пример:

```text
NAME                  DESIRED   CURRENT   READY
homework-xxxxxxxxxx   3         3         3
homework-yyyyyyyyyy   0         0         0
```

## Проверка размещения Pod на ноде

Deployment использует:

```yaml
nodeSelector:
  homework: "true"
```

Поэтому Pod могут быть запущены только на нодах с соответствующей меткой.

Проверить метку:

```bash
kubectl get nodes -L homework
```

Ожидаемый результат:

```text
NAME       STATUS   ROLES           HOMEWORK
minikube   Ready    control-plane   true
```

Проверить размещение Pod:

```bash
kubectl get pods -n homework -o wide
```

## Удаление ресурсов

Удалить Deployment:

```bash
kubectl delete -f deployment.yaml
```

Удалить namespace:

```bash
kubectl delete -f namespace.yaml
```

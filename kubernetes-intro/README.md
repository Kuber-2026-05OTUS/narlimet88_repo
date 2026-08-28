# Kubernetes Intro Homework

Домашнее задание по запуску локального Kubernetes-кластера и созданию Pod с init-контейнером.

## Что реализовано

- Namespace `homework`
- Pod в namespace `homework`
- Init-контейнер, создающий файл `index.html`
- Общий том `emptyDir`
- Основной контейнер с HTTP-сервером на порту `8000`
- Общий volume монтируется:
  - в `/init` у init-контейнера
  - в `/homework` у основного контейнера
- Перед завершением основного контейнера файл `/homework/index.html` удаляется через lifecycle hook `preStop`

## Запуск

Создать namespace:

```bash
kubectl apply -f namespace.yaml
```

Создать pod:

```bash
kubectl apply -f pod.yaml
```

Проверить состояние:

```bash
kubectl get pods -n homework
```

Ожидаемый результат:

```text
NAME       READY   STATUS    RESTARTS   AGE
homework   1/1     Running   0          ...
```

## Проверка init-контейнера

```bash
kubectl get pod homework -n homework \
  -o jsonpath='{.status.initContainerStatuses[0].state.terminated.reason}'
```

Ожидаемый результат:

```text
Completed
```

## Проверка файла

```bash
kubectl exec -n homework homework -- ls -la /homework
```

Проверить содержимое:

```bash
kubectl exec -n homework homework -- cat /homework/index.html
```

## Проверка веб-сервера

Выполнить port-forward:

```bash
kubectl port-forward -n homework pod/homework 8000:8000
```

В другом терминале:

```bash
curl http://localhost:8000
```

В ответ должна вернуться HTML-страница, созданная init-контейнером.

## Удаление ресурсов

```bash
kubectl delete -f pod.yaml
kubectl delete -f namespace.yaml
```

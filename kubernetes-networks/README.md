# Kubernetes Networks

Домашнее задание по теме «Сетевое взаимодействие Pod, сервисы».

В рамках задания настроены:

- Deployment с тремя репликами приложения;
- HTTP readiness probe;
- Service типа ClusterIP;
- Gateway API;
- Traefik в качестве Gateway API Controller;
- Gateway;
- HTTPRoute для домена `homework.otus`;
- URL Rewrite `/homepage` → `/index.html`.

## Структура

```text
kubernetes-networks/
├── README.md
├── namespace.yaml
├── deployment.yaml
├── service.yaml
├── gateway.yaml
└── httpRoute.yaml
```

## Namespace

Все ресурсы приложения создаются в namespace `homework`.

```bash
kubectl apply -f namespace.yaml
```

Проверка:

```bash
kubectl get namespace homework
```

## Deployment

Deployment запускает три реплики приложения.

Контейнер использует Python HTTP Server и раздаёт содержимое каталога `/homework` на порту `8000`.

Init container создаёт файл:

```text
/homework/index.html
```

Для проверки готовности контейнера используется HTTP readiness probe:

```yaml
readinessProbe:
  httpGet:
    path: /index.html
    port: 8000
  initialDelaySeconds: 1
  periodSeconds: 5
```

Deployment использует RollingUpdate:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```

Также Pod запускается только на ноде с label:

```text
homework=true
```

Для Minikube label можно установить командой:

```bash
kubectl label nodes minikube homework=true
```

Применение Deployment:

```bash
kubectl apply -f deployment.yaml
```

Проверка:

```bash
kubectl get deployment homework -n homework
kubectl get rs -n homework
kubectl get pods -n homework
```

Ожидаемый результат — три Pod в состоянии `Running` и `READY 1/1`.

## Service

Для доступа к Pod используется Service типа `ClusterIP`.

Service принимает соединения на порту `80` и перенаправляет их на порт `8000` контейнеров:

```text
Service :80
    |
    v
Pods :8000
```

Применение:

```bash
kubectl apply -f service.yaml
```

Проверка:

```bash
kubectl get svc -n homework
kubectl get endpoints -n homework
```

Service должен иметь тип:

```text
ClusterIP
```

и содержать endpoints всех трёх Pod приложения.

В новых версиях Kubernetes команда `kubectl get endpoints` может выводить предупреждение о переходе на EndpointSlice. Для просмотра EndpointSlice можно использовать:

```bash
kubectl get endpointslice -n homework
```

## Gateway API и Traefik

В качестве Gateway API Controller используется Traefik.

Проверка контроллера:

```bash
kubectl get pods -n traefik
kubectl get gatewayclass
```

GatewayClass должен быть принят контроллером:

```text
NAME      CONTROLLER                      ACCEPTED
traefik   traefik.io/gateway-controller   True
```

Также можно проверить наличие ресурсов Gateway API:

```bash
kubectl api-resources | grep -E 'GatewayClass|Gateway|HTTPRoute'
```

## Gateway

Gateway создаётся в namespace `homework` и использует GatewayClass `traefik`.

HTTP listener работает на порту `8000`.

Для HTTPRoute разрешено подключение из всех namespace:

```yaml
allowedRoutes:
  namespaces:
    from: All
```

Применение:

```bash
kubectl apply -f gateway.yaml
```

Проверка:

```bash
kubectl get gateway -n homework
kubectl describe gateway homework-gateway -n homework
```

Gateway должен иметь состояния:

```text
Accepted:    True
Programmed:  True
```

## HTTPRoute

HTTPRoute обрабатывает запросы для hostname:

```text
homework.otus
```

и направляет их в Service:

```text
homework:80
```

Применение:

```bash
kubectl apply -f httpRoute.yaml
```

Проверка:

```bash
kubectl get httproute -n homework
kubectl describe httproute homework -n homework
```

HTTPRoute должен иметь состояния:

```text
Accepted:      True
ResolvedRefs:  True
```

После подключения HTTPRoute у Gateway должен отображаться подключённый маршрут:

```text
Attached Routes: 1
```

Проверить можно командой:

```bash
kubectl describe gateway homework-gateway -n homework
```

## URL Rewrite

Дополнительное задание реализовано с помощью фильтра `URLRewrite`.

Запрос:

```text
http://homework.otus/homepage
```

переписывается в:

```text
/index.html
```

Конфигурация HTTPRoute:

```yaml
matches:
  - path:
      type: Exact
      value: /homepage

filters:
  - type: URLRewrite
    urlRewrite:
      path:
        type: ReplaceFullPath
        replaceFullPath: /index.html
```

После rewrite запрос направляется в Service `homework`.

## Доступ к приложению

Traefik используется как Service типа `LoadBalancer`.

Проверка:

```bash
kubectl get svc -n traefik
```

В Minikube для получения внешнего адреса LoadBalancer при необходимости запускается:

```bash
minikube tunnel
```

Команда должна продолжать работать в отдельном терминале.

После получения `EXTERNAL-IP` Traefik необходимо добавить запись для `homework.otus` в `/etc/hosts`.

Пример:

```text
10.111.211.148 homework.otus
```

IP необходимо брать из текущего значения `EXTERNAL-IP`:

```bash
kubectl get svc -n traefik
```

Проверка DNS:

```bash
getent hosts homework.otus
```

## Проверка приложения

Проверка стандартного маршрута:

```bash
curl http://homework.otus/index.html
```

Ожидаемый ответ:

```html
<html><body><h1>Hello from Kubernetes homework!</h1></body></html>
```

Проверка дополнительного задания:

```bash
curl http://homework.otus/homepage
```

Ожидаемый ответ:

```html
<html><body><h1>Hello from Kubernetes homework!</h1></body></html>
```

Таким образом:

```text
http://homework.otus/index.html
              |
              v
        HTTPRoute
              |
              v
       Service :80
              |
              v
        Pods :8000
```

Для `/homepage` дополнительно выполняется rewrite:

```text
http://homework.otus/homepage
              |
              v
     /homepage -> /index.html
              |
              v
        Service :80
              |
              v
        Pods :8000
```

## Применение всех манифестов

Манифесты приложения можно применить последовательно:

```bash
kubectl apply -f namespace.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f gateway.yaml
kubectl apply -f httpRoute.yaml
```

Проверка всех ресурсов:

```bash
kubectl get deployment,pods,svc -n homework
kubectl get gateway,httproute -n homework
```

## Результат

В результате выполнения задания:

- приложение работает в трёх Pod;
- готовность Pod проверяется HTTP readiness probe по `/index.html`;
- Service типа ClusterIP распределяет запросы между Pod;
- Traefik используется как Gateway API Controller;
- Gateway принимает HTTP-трафик;
- HTTPRoute маршрутизирует запросы для `homework.otus`;
- `http://homework.otus/index.html` возвращает страницу приложения;
- дополнительное задание реализует rewrite `/homepage` → `/index.html`;
- `http://homework.otus/homepage` также возвращает страницу приложения.

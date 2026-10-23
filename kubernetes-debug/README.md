# Домашнее задание №13 — Диагностика Kubernetes

## 1. Запуск distroless Nginx

Создан Pod с образом kyos0109/nginx-distroless:latest.

Манифест: nginx.yaml

Команда запуска:

    kubectl apply -f kubernetes-debug/nginx.yaml

Проверка:

    kubectl get pods -o wide

Результат: Pod nginx-distroless запущен в статусе Running.

## 2. Ephemeral debug container

Для диагностики запущен эфемерный контейнер с образом nicolaka/netshoot:

    kubectl debug -it pod/nginx-distroless --image=nicolaka/netshoot --target=nginx --container=debugger -- bash

Проверка процессов:

    kubectl exec nginx-distroless -c debugger -- ps aux

Результат:
- PID 1 — nginx master process
- PID 7 — nginx worker process

## 3. Доступ к файловой системе Nginx

Через /proc/1/root получен доступ к файловой системе основного контейнера:

    kubectl exec nginx-distroless -c debugger -- ls -la /proc/1/root/etc/nginx

Результат сохранён в nginx-filesystem.txt.

## 4. Анализ сетевого трафика

В debug-контейнере запущен tcpdump:

    kubectl exec nginx-distroless -c debugger -- tcpdump -nn -i any -e port 80

Для генерации трафика отправлены HTTP-запросы:

    kubectl exec nginx-distroless -c debugger -- curl http://127.0.0.1/

Результат сохранён в tcpdump.txt.

Зафиксированы TCP-соединения, HTTP GET-запросы и ответы HTTP 200 OK.

## 5. Диагностика Kubernetes-ноды

Создан отладочный Pod для ноды minikube:

    kubectl debug node/minikube -it --image=nicolaka/netshoot -- bash

Файловая система ноды доступна через /host.

Найден каталог логов Nginx:

    /host/var/log/pods/default_nginx-distroless_<POD_UID>/nginx/

Файл 0.log является символической ссылкой на Docker JSON-log.

Логи прочитаны непосредственно из:

    /host/var/lib/docker/containers/<CONTAINER_ID>/<CONTAINER_ID>-json.log

Результат сохранён в nginx-node-logs.txt.

Получено 15 записей HTTP-запросов с кодом ответа 200.

## 6. Дополнительное задание — strace

Через ephemeral debug container выполнена трассировка системных вызовов Nginx.

Master process (PID 1):

    kubectl exec nginx-distroless -c debugger -- timeout 15 strace -tt -p 1 -e trace=all

Обнаружен rt_sigsuspend() — ожидание сигнала.

Worker process (PID 7):

    kubectl exec nginx-distroless -c debugger -- timeout 10 strace -tt -p 7 -e trace=network,read,write,writev,epoll_wait,epoll_pwait

Во время трассировки отправлено 10 HTTP-запросов.

Обнаружены epoll_wait(), accept4(), recvfrom(), writev(),
sendfile(), write(), setsockopt().

Результаты:
- nginx-strace.txt
- nginx-worker-strace.txt
- strace-analysis.md

## Вывод

Проведена диагностика distroless-контейнера Nginx без установки
инструментов внутрь основного контейнера.

С помощью kubectl debug, tcpdump и strace исследованы процессы,
файловая система, сетевой трафик и системные вызовы Nginx.

Через debug Pod ноды получен доступ к логам контейнера
непосредственно из файловой системы Kubernetes-ноды.

# Анализ системных вызовов Nginx

## Master process (PID 1)

Команда: kubectl exec nginx-distroless -c debugger -- timeout 15 strace -tt -p 1 -e trace=all

Результат: nginx-strace.txt

Обнаружен системный вызов rt_sigsuspend().

Master-процесс Nginx ожидает сигналы и управляет worker-процессами.
Непосредственной обработкой HTTP-запросов он не занимается.

## Worker process (PID 7)

Команда: kubectl exec nginx-distroless -c debugger -- timeout 10 strace -tt -p 7 -e trace=network,read,write,writev,epoll_wait,epoll_pwait

Результат: nginx-worker-strace.txt

Во время трассировки отправлено 10 HTTP-запросов на 127.0.0.1:80.

Обнаруженные системные вызовы:

- epoll_wait() — ожидание сетевых событий.
- accept4() — принятие входящего TCP-соединения.
- recvfrom() — получение HTTP GET-запроса.
- writev() — отправка HTTP-заголовков с кодом 200 OK.
- sendfile() — передача тела HTTP-ответа размером 612 байт.
- write() — запись информации о запросе в access log.
- setsockopt() — настройка TCP_NODELAY.

## Вывод

Master-процесс Nginx управляет рабочими процессами и ожидает сигналы.

Worker-процесс принимает HTTP-запросы, отправляет ответы
и записывает события в журнал.

Ephemeral debug container позволил выполнить strace без установки
дополнительных инструментов в distroless-контейнер Nginx.
